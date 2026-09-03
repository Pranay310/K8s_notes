# Phase 3 — Database in the Cluster

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** run Postgres inside Kubernetes with storage that survives the pod being destroyed, and credentials kept out of the manifests.

**Outcome:** Postgres running as `postgres-0`, reachable by the name `postgres`, with a 1Gi disk that persisted through a deliberate pod deletion.

---

## The problem this phase solves

Everything deployed so far has been disposable. Kill a frontend pod and a fresh one takes its place — nothing lost, because it holds nothing.

A database is different. It has data, and that data has to outlive the container holding it. Kubernetes has specific machinery for this, and it's what the whole phase is about.

---

## Files

### 1. `k8s/postgres-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_DB: store_app
```

A Secret is a place to keep values that shouldn't be scattered through other files.

**Be honest about what it does.** Secrets are stored **base64-encoded, which is encoding, not encryption.** Anyone who can read Secrets in the cluster can read the password with one command. Real protection comes from restricting who can read them (RBAC, Phase 9) and encrypting the cluster's datastore. For a local learning cluster this is fine — just don't mistake it for security.

**`stringData` vs `data`:** `stringData` accepts plain text and Kubernetes encodes it. `data` requires base64 by hand. Always prefer `stringData` when writing manifests.

**Still a plain-text password sitting in a file that could reach git.** The real-world answers are Sealed Secrets, External Secrets Operator, or a vault. Worth knowing now; out of scope here.

### 2. `k8s/postgres-statefulset.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U postgres -d store_app"]
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U postgres -d store_app"]
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              memory: 512Mi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

---

## What each part does

### Two objects, one file

The `---` separates them. Common practice when things belong together and are always applied as a unit.

### StatefulSet, not Deployment

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | random (`backend-7d4f-x9k2`) | stable (`postgres-0`) |
| Pods are | interchangeable | distinct identities |
| Storage | shared or none | one disk per pod, reattached on restart |
| Startup order | all at once | one at a time, in order |

A Deployment assumes its pods are clones — kill one, get another, nobody notices. A database can't work that way. The StatefulSet guarantees the replacement pod gets the **same name and the same disk**.

### Headless Service — `clusterIP: None`

A normal Service gets its own address and spreads incoming traffic across all matching pods. For a single database that's not wanted — the name `postgres` should point straight at the one pod.

`clusterIP: None` does exactly that: no load balancing, the name resolves directly to pod IPs.

### `volumeClaimTemplates` — the storage request

This is the heart of the phase. It says: *give this pod 1GB of disk that outlives the pod.*

Three pieces of vocabulary, easiest understood as a chain:

- **PVC (PersistentVolumeClaim)** — the request. "I need 1GB."
- **PV (PersistentVolume)** — the actual storage that gets handed over.
- **StorageClass** — the rules for how storage gets created. kind ships one called `standard` that just uses disk on the node.

The claim is named after the pod: `data-postgres-0`. That naming is how the same disk finds its way back to the same pod.

`ReadWriteOnce` means one node can mount it at a time — normal for a database, and the reason a database can't be casually scaled to multiple replicas this way.

### `PGDATA` pointing one level down

```yaml
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata
```

Small but necessary. A freshly mounted volume arrives containing a `lost+found` directory, and Postgres refuses to initialize into a directory that isn't empty. Pointing it at a subfolder sidesteps this.

Without it: `initdb: error: directory "/var/lib/postgresql/data" exists but is not empty`.

### `envFrom: secretRef`

Pulls **every** key in the Secret in as an environment variable, rather than listing them one at a time. The Postgres image reads `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` on first startup to create the database.

### Probes — the Phase 0 idea, now in Kubernetes

```yaml
readinessProbe:   # can it serve traffic? → controls whether the Service sends anything here
livenessProbe:    # is it wedged?         → restarts the pod
```

Both run `pg_isready`, but the timings differ deliberately:

- Readiness starts after 5s, checks every 5s — get it into rotation quickly.
- **Liveness waits 30s** before its first check. A database can be slow to start, and a liveness probe firing during startup would kill the pod mid-boot, over and over.

That gap between `initialDelaySeconds` values is the whole reason liveness and readiness are separate settings.

### `requests` vs `limits`

- **requests** — what the scheduler reserves when choosing a node. Guaranteed.
- **limits** — the hard ceiling. Exceed the memory limit and the container is killed (OOMKilled).

Deliberately triggering that is a Phase 7 exercise.

---

## Applying and verifying

```bash
kubectl apply -f k8s/postgres-secret.yaml
kubectl apply -f k8s/postgres-statefulset.yaml
kubectl get pods -w
```

`-w` watches and keeps printing changes. Ctrl+C when it reaches `1/1 Running`.

```
postgres-0   0/1     Pending             0     0s
postgres-0   0/1     ContainerCreating   0     4s
postgres-0   0/1     Running             0    44s
postgres-0   1/1     Running             0    49s
```

The states in order: **Pending** (waiting for a node and for storage to be provisioned) → **ContainerCreating** (pulling the image, mounting the disk) → **Running 0/1** (process started, readiness probe not passing yet) → **Running 1/1** (ready).

That gap between `Running 0/1` and `Running 1/1` is the readiness probe doing its job.

```bash
kubectl get statefulset,svc,pvc,pv
```

```
statefulset.apps/postgres    1/1
service/postgres             ClusterIP   None   5432/TCP
persistentvolumeclaim/data-postgres-0   Bound   pvc-e02b93f5...   1Gi   RWO   standard
persistentvolume/pvc-e02b93f5...        1Gi   RWO   Delete   Bound   store-app/data-postgres-0
```

The claim is `Bound`, meaning it found real storage. `CLUSTER-IP: None` confirms the headless Service.

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d store_app -c '\l'
```

`store_app` exists — the Secret values were read correctly at first startup.

---

## The test that proves the phase

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d store_app -c 'CREATE TABLE survived (id int);'
kubectl delete pod postgres-0
kubectl get pods -w
kubectl exec -it postgres-0 -- psql -U postgres -d store_app -c '\dt'
```

Result: the `survived` table was still there.

Three things worth noticing in that run:

**The pod returned in 6 seconds**, versus 49 the first time. The first startup had to initialize a fresh database; the replacement just reattached the existing disk.

**Same name — `postgres-0`.** A Deployment would have produced a new random name. Stable identity is how the storage gets matched back.

**The PVC stayed `Bound` the whole time.** Deleting a pod does not touch its storage claim. That's deliberate — losing data by accident is far easier than recovering it, so Kubernetes errs strongly toward keeping it. Removing the data requires deleting the PVC explicitly.

> **The pod is disposable. The data is not.** That is the entire point of this phase.

---

## Service DNS

```bash
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup postgres
```

```
Name:   postgres.store-app.svc.cluster.local
Address: 10.244.1.2
```

The `NXDOMAIN` lines that appear alongside it are **not errors**. Looking up a short name makes the cluster try several full versions in order until one works:

1. `postgres.store-app.svc.cluster.local` ← this namespace — **found**
2. `postgres.svc.cluster.local` — not found
3. `postgres.cluster.local` — not found

Every attempt gets printed, misses included.

The shape of the full name is the thing to remember:

```
postgres  .  store-app  .  svc  .  cluster.local
service      namespace     type    cluster suffix
```

Because the backend runs in the same namespace, it can just say `postgres` and the rest is filled in automatically.

So the connection string in Phase 4 becomes:

```
postgresql://postgres:postgres@postgres:5432/store_app?schema=public
```

Three `postgres` in a row, which reads strangely — they are the **username**, the **password**, and the **service name**.

The progression across the whole project:

| Where | Host |
|---|---|
| Laptop | `localhost` |
| Compose | `db` |
| Kubernetes | `postgres` |

Exactly why Phase 0 removed every hardcoded host from the code.

---

## Commands learned

```bash
kubectl apply -f <file>
kubectl get pods -w                          # watch state changes live
kubectl get statefulset,svc,pvc,pv           # several types in one query
kubectl exec -it postgres-0 -- psql -U postgres -d store_app -c '\dt'
kubectl delete pod postgres-0                # it comes back
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup postgres
```

`--rm` deletes the throwaway pod when it finishes. A `couldn't attach to pod` warning on a short-lived pod is a harmless timing hiccup — it finished before kubectl could attach, so kubectl read the logs instead.

---

## Key takeaways

1. **StatefulSet for anything with data.** Stable name, stable disk, ordered startup. Deployment for everything else.
2. **The PVC outlives the pod on purpose.** Deleting a pod never deletes its storage; that requires deleting the claim.
3. **Headless Service (`clusterIP: None`) when the name should point at a specific pod**, not spread traffic across many.
4. **Secrets are encoded, not encrypted.** They keep values out of manifests, not out of reach.
5. **Liveness needs a longer `initialDelaySeconds` than readiness**, or a slow-starting database gets killed mid-boot in a loop.
6. **A mounted volume isn't empty** — hence `PGDATA` pointing at a subfolder.
7. **`Running 0/1` is not broken.** The process is up; the readiness probe hasn't passed yet.
8. **Service name replaces hostname.** `localhost` → `db` → `postgres`, with the code unchanged throughout.

---

## Next: Phase 4 — Backend

- **Deployment** — the backend is stateless, so pods can be interchangeable
- **ConfigMap** for non-secret settings, **Secret** for `JWT_SECRET` and `DATABASE_URL`
- **Liveness on `/healthz`, readiness on `/readyz`** — the endpoints built in Phase 0 finally get used
- **initContainer** running `prisma migrate deploy` — blocks the app from starting until the schema is ready
- **Job** running the seed — once, not per replica
- **ClusterIP Service** on port 3000 — reachable inside the cluster only, since Ingress will be the single front door
