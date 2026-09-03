# Phase 5 — Frontend in the Cluster

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** run the nginx-served static build in Kubernetes.

**Outcome:** frontend pods running, page served, SPA routing working. API calls still fail — nothing routes `/api` to the backend yet.

This is the shortest phase in the project, and that's the point.

---

## Why this phase is so small

Compare what the frontend needs against the backend:

| | Backend | Frontend |
|---|---|---|
| ConfigMap | yes | **none** |
| Secret | yes | **none** |
| initContainer | yes (migrations) | **none** |
| Job | yes (seed) | **none** |
| Database connection | yes | **none** |
| Distinct probes | `/readyz` vs `/healthz` | same endpoint for both |

**Nothing to configure.** That's the Phase 0 decision paying off: because the frontend has no backend address baked into it — it just requests the relative path `/api/v1` — there is no per-environment setting to inject.

Had `VITE_API_URL=http://localhost:3000` been baked into the build, this phase would have required a rebuild for the cluster, and the image would no longer be the same artifact that ran in Compose.

---

## `k8s/frontend-deployment.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: store-frontend:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 128Mi
```

### Notes on the choices

**Both probes hit the same `/healthz`.** For a static file server there's no meaningful difference between "alive" and "ready" — nginx either serves files or it doesn't. There's no external dependency to check, so the readiness/liveness split that mattered so much for the backend collapses here.

That endpoint comes from the `nginx.conf` written in Phase 1.

**Much smaller resources.** nginx serving pre-built static files barely uses anything — 50m CPU and 64Mi memory versus the backend's 100m/128Mi.

**ClusterIP again.** Still not reachable from outside the cluster. Ingress will be the single front door for both frontend and backend in Phase 6.

**`imagePullPolicy: IfNotPresent`** for the same reason as the backend — the image came from `kind load`, not a registry.

---

## Applying and verifying

```bash
kubectl apply -f k8s/frontend-deployment.yaml
kubectl get pods -o wide
```

`-o wide` is worth reading here. With 2 backend and 2 frontend replicas, the pods should be **spread across both worker nodes**. The scheduler does this without being asked — it prefers not to stack replicas of the same thing on one node, so losing a node doesn't take out everything at once.

```bash
kubectl port-forward svc/frontend 8080:80
```

Then in another terminal or the browser:

```bash
curl localhost:8080/healthz     # ok
```

**Browser check:** open `http://localhost:8080`, navigate to a route, then refresh. Staying on that route rather than 404ing proves the nginx SPA fallback (`try_files $uri $uri/ /index.html`) is working.

**API calls still fail** with the same 405 seen in Compose, for the same reason: nginx has no `/api` rule, falls through to the SPA fallback, and refuses to serve a static file for a POST. That gap closes in Phase 6.

---

## The scaling experiment

```bash
kubectl scale deploy/frontend --replicas=4
kubectl get pods -o wide
kubectl scale deploy/frontend --replicas=2
```

Four pods appear across two nodes within seconds.

**This is only safe because the pods hold nothing.** Each one is an identical copy of the same static files; it doesn't matter which one a request reaches.

Try the same on Postgres and you'd get four databases, four separate disks, and no way to know which one holds the data. That's the Deployment/StatefulSet distinction made concrete — and worth doing once precisely because the frontend makes the *safe* case obvious.

---

## Key takeaways

1. **A well-designed frontend image needs no Kubernetes configuration at all.** No ConfigMap, no Secret, no env vars. If it needed them, something was baked in at build time that shouldn't have been.
2. **The readiness/liveness split only matters when there are dependencies to check.** A static file server has none, so both probes can point at the same endpoint.
3. **Stateless means trivially scalable.** `kubectl scale` is instant and safe here, and would be destructive for the database.
4. **The scheduler spreads replicas across nodes by default** — a free resilience property, visible with `-o wide`.
5. **The same image ran in Compose and now in Kubernetes, unchanged.** That's "build once, deploy anywhere" actually happening, rather than just being a slogan.

---

## Next: Phase 6 — Ingress

The routing gap deliberately left open since Phase 1 finally closes:

- Install ingress-nginx (the `ingress-ready=true` label from the Phase 2 cluster config is what lets it schedule)
- Add `store.local` to `/etc/hosts`
- One Ingress resource routing `/api` → backend Service and `/` → frontend Service
- The app works end to end at `http://store.local`, on the same origin, with no CORS involved

Host ports 80 and 443 were mapped to the control-plane node back in Phase 2 specifically so this would be reachable from the browser.
