# Phase 4 — Backend in the Cluster

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** run the Express API in Kubernetes with two replicas, config supplied from outside, migrations gated before startup, and the seed run exactly once.

**Outcome:** two backend pods running, all four tables created by the migration, the admin seeded, and `/readyz` returning `ready` — proving the backend found Postgres by service name.

---

## Files

### 1. `k8s/backend-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  JWT_EXPIRES_IN: "7d"
  CLIENT_URL: "http://store.local"
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
type: Opaque
stringData:
  DATABASE_URL: "postgresql://postgres:postgres@postgres:5432/store_app?schema=public"
  JWT_SECRET: "dev_secret_at_least_16_chars_long"
```

**ConfigMap vs Secret** — same mechanism, different sensitivity. The split is mostly about who's allowed to read them: in a real cluster, more people can see ConfigMaps than Secrets. Neither is encrypted.

**The host in `DATABASE_URL` is `postgres`** — the Service name from Phase 3. Three `postgres` in a row in that string: username, password, service name.

`POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` are required by the app's zod schema and already exist in `postgres-secret` from Phase 3, so that Secret gets pulled in rather than duplicated.

### 2. `k8s/backend-deployment.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      initContainers:
        - name: migrate
          image: store-backend:v1
          imagePullPolicy: IfNotPresent
          command: ["npm", "run", "migrate:deploy"]
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: backend-secret
            - secretRef:
                name: postgres-secret
      containers:
        - name: backend
          image: store-backend:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: backend-secret
            - secretRef:
                name: postgres-secret
          readinessProbe:
            httpGet:
              path: /readyz
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 20
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
```

### 3. `k8s/seed-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: seed-admin
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: seed
          image: store-backend:v1
          imagePullPolicy: IfNotPresent
          command: ["npm", "run", "seed"]
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: backend-secret
            - secretRef:
                name: postgres-secret
```

---

## What each part does

### Deployment, not StatefulSet

The backend holds no data, so its pods genuinely are interchangeable. Random names are fine, and `replicas: 2` is safe — which it wasn't for Postgres, where two replicas would mean two databases on two separate disks with no way to know which held the data.

**This is the practical difference between the two object types**, and it comes down to one question: does this thing own data?

### `imagePullPolicy: IfNotPresent` — essential

Without it, Kubernetes tries to download `store-backend:v1` from Docker Hub, where it doesn't exist, and the pod fails with `ErrImagePull`. This tells it to use the local copy loaded with `kind load`.

**Together with `kind load`, this is the #1 cause of failed pods in a kind cluster.** The two go hand in hand: load the image, and tell Kubernetes not to look elsewhere for it.

### initContainer — migrations as a gate

An initContainer runs to completion **before** the main container starts. If it fails, the pod never starts at all.

So migrations always finish before the app serves a single request. This is the Phase 0 decision finally paying off: migrations aren't a startup side effect that races with itself across replicas — they're a gate.

**Caveat:** with 2 replicas, both pods run the init container. Prisma handles concurrent migrations safely using a lock, so this works, but it's slightly wasteful. A cleaner design uses a separate Job for migrations too. Worth knowing; not worth doing on a learning cluster.

### Job — run once, not per replica

A Job runs a container until it succeeds, then stops. Unlike a Deployment, it isn't meant to stay running. `backoffLimit: 3` retries up to three times before giving up.

This is what `seedAdmin()` became after being pulled out of app startup in Phase 0. With 2 replicas, startup seeding would have meant two pods racing to create the same admin row.

### Three `envFrom` entries

All three sources merge into one set of environment variables. Later entries win on conflicts.

`envFrom` pulls in **every** key from the source, rather than listing them one at a time.

### Probes — the Phase 0 endpoints finally used

```yaml
readinessProbe:  path: /readyz    # checks the database
livenessProbe:   path: /healthz   # checks nothing but the process
```

The difference now has real consequences:

- A database blip makes `/readyz` fail → pods are **pulled out of the Service**, no traffic sent, no restart.
- A wedged process makes `/healthz` fail → pod is **killed and restarted**.

Had the database check been in `/healthz`, a 30-second database blip would have restarted every backend pod at once — turning a brief blip into a restart storm. That's the reason the two endpoints were built separately back in Phase 0.

### ClusterIP Service

`ClusterIP` means reachable **from inside the cluster only**. Nothing outside can address the backend directly.

That's deliberate: Ingress will be the single front door (Phase 6), which keeps the attack surface to one hardened entry point. It's the same reasoning that removed the `ports:` block from the Compose database.

---

## Applying

```bash
kubectl apply -f k8s/backend-config.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl get pods -w
```

Pod states to expect, in order:

| State | Meaning |
|---|---|
| `Init:0/1` | the migration initContainer is running |
| `PodInitializing` | init finished, main container starting |
| `Running 0/1` | process up, readiness probe not passing yet |
| `Running 1/1` | ready, receiving traffic |

Then, once the backend pods are ready:

```bash
kubectl apply -f k8s/seed-job.yaml
kubectl logs job/seed-admin
```

Result:

```
NAME                       READY   STATUS      RESTARTS   AGE
backend-5d7f5b4f4f-mdb4m   1/1     Running     0          2m6s
backend-5d7f5b4f4f-zcwm7   1/1     Running     0          2m6s
postgres-0                 1/1     Running     4          2d13h
seed-admin-dzwrv           0/1     Completed   0          41s
```

Migrations created all four tables (`User`, `Store`, `Rating`, `_prisma_migrations`), and the seed logged `Admin ready`.

---

## Reading the output — things that look wrong but aren't

**`seed-admin-dzwrv  0/1  Completed`**

`0/1` looks like a failure. It isn't. The container finished and exited, so zero are *running*. **`Completed` is the success state for a Job.** A failed Job shows `Error` or `BackoffLimitExceeded`.

**`Defaulted container "backend" out of: backend, migrate (init)`**

kubectl noting the pod has two containers and it picked the main one. To read the migration's output — which is how a pod stuck at `Init:0/1` gets debugged:

```bash
kubectl logs <pod-name> -c migrate
```

**`port-forward` "not working"**

```bash
kubectl port-forward svc/backend 3000:3000
curl localhost:3000/healthz     # Failed to connect
```

Not a Kubernetes problem — terminal mechanics. `port-forward` runs in the foreground and holds the tunnel open. **Getting the prompt back means the tunnel already stopped.** It needs its own terminal, or backgrounding:

```bash
kubectl port-forward svc/backend 3000:3000 &
sleep 2
curl localhost:3000/healthz
kill %1
```

**No request logs appearing**

Nothing wrong — the app has no request logger. The only `console.log` runs at startup. Express writes nothing per-request unless middleware is added.

**`postgres-0 ... RESTARTS 4`**

Worth investigating rather than ignoring — most likely the liveness probe failing during laptop sleep or resource pressure:

```bash
kubectl describe pod postgres-0 | tail -20
```

The Events section says what happened. A recurring liveness failure suggests raising `timeoutSeconds`.

---

## `port-forward svc/` vs `port-forward pod/`

| | Reaches | Use for |
|---|---|---|
| `port-forward pod/<name>` | one specific pod | "is this pod itself working?" |
| `port-forward svc/<name>` | the Service, spread across replicas | closer to real traffic |

Both skip Ingress entirely, which is what makes them useful for isolating where a problem is.

---

## The rebuild loop

Changing source code requires the **full** loop — this is Phase 1's immutability rule with two extra steps:

```bash
docker build -t store-backend:v2 ./backend
kind load docker-image store-backend:v2 --name store-app
kubectl set image deploy/backend backend=store-backend:v2 migrate=store-backend:v2
kubectl rollout status deploy/backend
```

**Note the new tag `v2`.** Rebuilding as `v1` and re-applying does nothing — Kubernetes compares the image name, sees no change, and correctly decides there's nothing to roll out. This is exactly why `latest` is a bad habit: with it, there is no way to express "this is a different build."

---

## Commands learned

```bash
kubectl apply -f <file>
kubectl get pods -w
kubectl logs deploy/backend                    # picks one pod
kubectl logs <pod> -c migrate                  # a specific container in a pod
kubectl logs job/seed-admin
kubectl logs -f -l app=backend --prefix        # follow all pods with a label
kubectl describe pod <name>                    # Events section explains failures
kubectl port-forward svc/backend 3000:3000
kubectl set image deploy/backend backend=store-backend:v2
kubectl rollout status deploy/backend
```

`-l app=backend` selects by label; `--prefix` tags each line with its pod name.

---

## Key takeaways

1. **Deployment vs StatefulSet comes down to one question: does it own data?** No data → interchangeable pods, safe to scale, random names fine.
2. **`imagePullPolicy: IfNotPresent` is mandatory with `kind load`.** Otherwise Kubernetes hunts for the image on Docker Hub.
3. **initContainers are gates.** They run to completion before the app starts, and their failure stops the pod entirely. The right home for migrations.
4. **Jobs run once.** The right home for seeding, which must not happen per replica.
5. **`0/1 Completed` is success for a Job**, not a failure.
6. **Readiness pulls a pod from the Service; liveness restarts it.** Only readiness should check dependencies — that's why `/readyz` hits the database and `/healthz` doesn't.
7. **ClusterIP means cluster-internal only.** The backend has no public address by design; Ingress is the single front door.
8. **`port-forward` must keep running.** A returned prompt means the tunnel is gone.
9. **Source change → rebuild → `kind load` → new tag → rollout.** Skipping any step means the cluster keeps running the old code, silently.
10. **`kubectl logs <pod> -c <container>`** is how init containers get debugged. Without `-c`, kubectl shows the main container only.

---

## Next: Phase 5 — Frontend

Much simpler — no database, no secrets, no migrations:

- **Deployment + ClusterIP Service**, nginx serving static files
- **No ConfigMap and no Secret at all** — the payoff from Phase 0, since the frontend has no backend address baked in, there is nothing to configure
- Both probes hit `/healthz`; for a static file server there's no meaningful difference between alive and ready
- **`kubectl scale deploy/frontend --replicas=4`** — trivially safe here, and impossible for Postgres, which makes the Deployment/StatefulSet distinction concrete
- API calls will still fail with the same 405 as in Compose. Nothing routes `/api` to the backend yet — that's Phase 6.
