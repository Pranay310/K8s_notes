# Phase 1 — Dockerizing the Stack

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** produce two images that build reproducibly and run correctly, and prove it with Docker Compose *before* Kubernetes is involved.

**Outcome:** both images build; the backend starts, reaches Postgres by service name, runs migrations as a separate step, serves requests, and shuts down cleanly on SIGTERM; the frontend serves the SPA with working client-side routing.

---

## Why Compose before Kubernetes

Compose is a **debugging accelerator**, not a deliverable. Kubernetes never reads `docker-compose.yml`.

It isolates exactly one question: **are my images correct?**

- Breaks in Compose → the fault is in the Dockerfile, env vars, or app code.
- Works in Compose but not Kubernetes → the fault is in the YAML.

That split is valuable because a broken pod gives a much noisier signal — you'd be debugging images, Services, Secrets, probes, and DNS simultaneously.

The other reason is loop speed. `docker compose up --build` takes seconds. The Kubernetes loop is rebuild → `kind load` → `kubectl apply` → wait for rollout → `describe` → `logs`, which is minutes.

**It paid for itself.** Compose caught, in seconds each: a missing `seed.ts`, a wrongly-named Prisma config, missing `COPY` lines in the Dockerfile, and a missing datasource URL. Every one of those would have surfaced in Kubernetes as `CrashLoopBackOff` with no obvious cause.

### Concepts that transfer directly

| Compose | Kubernetes |
|---|---|
| service name as hostname (`db`) | Service DNS (`postgres.store-app.svc.cluster.local`) |
| `environment:` | ConfigMap + Secret |
| named volume | PersistentVolumeClaim |
| `healthcheck:` | liveness / readiness probes |
| `depends_on: service_healthy` | initContainer that waits |
| `ports: "8080:80"` | Service + Ingress |
| `compose run --rm backend npm run migrate:deploy` | Job |

Most teams running Kubernetes in production keep a Compose file for local dev. The risk is drift — if the Compose env and the ConfigMap diverge, Compose stops being a trustworthy signal.

---

## Base image decision: `node:22-slim`, not Alpine

Debian-based, so no musl/OpenSSL surprises with Prisma, and no `binaryTargets` needed in `schema.prisma`. Alpine saves ~40MB and costs an afternoon. Alpine *is* used for the frontend runtime, where nothing Prisma-related is involved.

---

## Backend image

### `.dockerignore` (write this first)

```
node_modules
dist
.env
.env.*
!.env.example
.git
*.log
```

Without it Docker uploads `node_modules` and `.env` into the build context — slow, and a secret leak.

### `Dockerfile`

```dockerfile
# ---------- deps ----------
FROM node:22-slim AS deps
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends openssl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY package*.json ./
RUN npm ci

# ---------- build ----------
FROM deps AS build
WORKDIR /app
COPY prisma ./prisma
COPY prisma.config.mjs ./
COPY tsconfig.json ./
COPY src ./src
RUN npx prisma generate
RUN npx tsc

# ---------- runtime ----------
FROM node:22-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production

RUN apt-get update && apt-get install -y --no-install-recommends openssl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
COPY --from=build /app/prisma.config.mjs ./
COPY --from=build /app/src/generated ./dist/generated

USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### What each decision is doing

**Multi-stage build.** The runtime image never contains TypeScript, source files, or dev dependencies. Smaller image, smaller attack surface.

**`npm ci` before copying source.** Docker caches layers in order. Dependencies change rarely; source changes constantly. This ordering means editing a controller doesn't re-download `node_modules`.

**`COPY --from=build /app/src/generated ./dist/generated` — the Prisma gotcha.** `prisma generate` writes *both* TypeScript files and native query-engine binaries (`.so.node`) into `src/generated/prisma`. `tsc` compiles the TS and ignores the binaries. Without this copy, the container starts fine and dies on the first query with an engine-not-found error.

**`CMD ["node", ...]` — exec form, no npm.** Exec form makes Node PID 1, so it receives SIGTERM directly. The Phase 0 graceful shutdown only works this way. `npm` in between does not reliably forward signals.

**`USER node`.** Don't run as root. In Kubernetes this pairs with `runAsNonRoot: true` in the security context.

**`openssl` installed.** Prisma's query engine links against it.

---

## Frontend image

### `nginx.conf`

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    location /healthz {
        access_log off;
        return 200 "ok\n";
    }

    # SPA fallback — required for React Router
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**`try_files` is mandatory.** Without it, refreshing on `/admin/stores` returns 404 — nginx looks for a file at that path and there isn't one. Every unmatched route must fall back to `index.html` so React Router can handle it client-side.

**Caching strategy:** Vite emits content-hashed filenames under `/assets/`, so those can be cached for a year immutably. `index.html` must never be cached, or browsers keep loading the old bundle after a deploy.

**No `/api` proxy here.** Deliberate — see the routing gap section below.

### `Dockerfile`

```dockerfile
# ---------- build ----------
FROM node:22-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---------- runtime ----------
FROM nginx:1.27-alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

The runtime image has no Node, no npm, no source — static files and nginx, ~50MB.

---

## Compose file

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: store_app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d store_app"]
      interval: 5s
      retries: 10

  backend:
    build: ./backend
    image: store-backend:v1
    environment:
      NODE_ENV: production
      PORT: "3000"
      DATABASE_URL: postgresql://postgres:postgres@db:5432/store_app?schema=public
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: store_app
      JWT_SECRET: dev_secret_at_least_16_chars
      JWT_EXPIRES_IN: 7d
      CLIENT_URL: http://localhost:8080
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: ./frontend
    image: store-frontend:v1
    ports:
      - "8080:80"
    depends_on:
      - backend

volumes:
  pgdata:
```

**`@db:5432`, not `@localhost:5432`.** Compose gives each service a DNS name matching its key. This is a direct preview of Kubernetes Service discovery — in Phase 3 it becomes `@postgres:5432` for exactly the same reason.

**`db` has no `ports:` block.** Nothing outside the Compose network needs to reach it. This matches the ClusterIP Postgres Service coming in Phase 3, and it avoids colliding with a local Postgres install on 5432.

---

## Startup sequence

```bash
docker compose build
docker compose up -d db
docker compose run --rm backend npm run migrate:deploy
docker compose run --rm backend npm run seed
docker compose up -d
```

Migrations run as a **separate one-off container**, not on app boot — rehearsing the Phase 0 split. Order matters and Kubernetes will enforce the same one in Phase 4: schema first (initContainer), then data (Job), then serve traffic.

---

## Problems hit, and what they taught

### 1. Compose plugin not installed

`docker.io` from Ubuntu's repos ships the engine only. Compose V2 is a separate plugin from Docker's own repository.

Also added the user to the `docker` group to drop `sudo` — important before kind, which stores cluster config in the home directory and breaks when root and non-root Docker are mixed. (Note: the `docker` group is effectively root access. Fine on a dev laptop, not on a shared server.)

### 2. Port collisions — 5432, then 8080

`sudo ss -tulpn | grep :5432` showed `docker-proxy`, meaning other **containers** held the ports: an old `pern-postgres` from another project, and — unexpectedly — an existing kind cluster mapping host 8080. Deleted the old cluster (`kind delete cluster`) to start fresh; stopped the stray Postgres.

**The underlying lesson:** host ports are a scarce global resource, and containers that publish them collide. Inside Kubernetes every pod gets its own IP and can bind whatever port it likes without coordination. That's *why* Postgres will be ClusterIP with no host port, and only Ingress touches the host.

### 3. Half-started stacks holding ports

Failed `up` attempts leave containers `Created` or `Up` and still holding ports. **Habit: `docker compose down` before `up` whenever something failed mid-start.** Same discipline as `kubectl delete namespace` + re-apply when a deployment gets into a weird state.

### 4. `Cannot find module '/app/dist/seed.js'`

`src/seed.ts` had been discussed in Phase 0 but never written. Wrote it, then had to **rebuild the image**.

**Key realization: the image is immutable.** Editing `src/` on the laptop changes nothing inside a running container. Every source change needs a rebuild. Same rule in Kubernetes, plus a `kind load` and a rollout — which is exactly why catching these in Compose matters.

### 5. Prisma 7 config — three stacked problems

The error was `The datasource.url property is required in your Prisma config file`.

Cause a: **`datasource db` in `schema.prisma` had no `url`.** Prisma 7 moved it to a config file.

Cause b: **the file was named `prisma7.config.ts`.** Prisma looks for `prisma.config.{ts,js,mjs}`. It was never being loaded.

Cause c: **the file wasn't in the image.** The Dockerfile copied `prisma/`, `tsconfig.json`, and `src/` — but not the config.

Also removed `import "dotenv/config"` from it, since `dotenv` is a devDependency stripped by `npm ci --omit=dev`.

**Renamed to `prisma.config.mjs`** (plain ESM, not TypeScript) for two reasons: the runtime image has no TypeScript loader, and an initContainer in Phase 4 has no shell to improvise in. Config files that need compiling before they can be read are a trap in slim production images.

```js
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: process.env.DATABASE_URL },
});
```

**The debugging pattern that solved it:** the build output showed no `COPY prisma.config.mjs` step. When something behaves as though a file is missing, *verify it's in the image* rather than assuming:

```bash
docker compose run --rm backend ls -la
```

That is the Compose equivalent of `kubectl exec -it <pod> -- ls -la`.

---

## The routing gap (deliberate, unresolved)

`POST http://localhost:8080/api/v1/auth/register` → **405 Not Allowed**.

The frontend requests a relative `/api/v1/...`, so the browser sends it to the origin serving the page — nginx on 8080. nginx has no `/api` rule, falls through to `try_files ... /index.html`, and refuses to serve a static file for a POST. Hence 405 rather than 404. **The request never reaches the backend.**

The backend itself is fine, proven directly:

```bash
curl -X POST localhost:3000/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Test User Name Long","email":"t@example.com","password":"Passw0rd!","address":"somewhere"}'
```

### Why it wasn't fixed

Every environment needs something in front routing `/api` to the backend and everything else to the frontend:

| Environment | Router | Status |
|---|---|---|
| `npm run dev` | Vite dev proxy | configured |
| `npm run preview` | Vite preview proxy | configured |
| **Compose** | **nothing** | **this gap** |
| Kubernetes | Ingress | Phase 6 |

A `proxy_pass http://backend:3000` in `nginx.conf` would fix it instantly — but `backend` is a *Compose service name*. Baking it into the nginx image couples that image to Compose, breaking the rule held since Phase 0: **the frontend image knows nothing about where the backend lives.**

The trade was: make Compose feature-complete at the cost of coupling, or accept a gap that Phase 6 closes properly. Took the second. Compose already answered its one question — the images are correct.

---

## Verification checklist

```bash
docker compose ps                        # all three up, db healthy
curl localhost:3000/healthz              # ok
curl localhost:3000/readyz               # ready  ← the meaningful one
curl localhost:8080/healthz              # ok, from nginx
docker compose exec db psql -U postgres -d store_app -c '\dt'
```

**`/readyz` returning `ready` is the milestone of this phase.** It proves three things at once: the Prisma query engine loaded correctly inside the container, the backend resolved `db` via internal DNS, and it authenticated against Postgres. Those were the two most likely failure points.

Graceful shutdown:

```bash
docker compose stop backend    # should return in ~1s
```

A 10-second hang means SIGTERM isn't reaching Node — check the `CMD` is exec form.

Frontend: open `http://localhost:8080`, navigate to a route, refresh. Staying on that route (not 404) proves the SPA fallback works.

---

## Useful commands

```bash
docker compose logs -f backend           # follow live
docker compose logs --tail=50 backend
docker compose logs --since 5m backend
docker compose exec db psql -U postgres -d store_app -c '\dt'
docker compose run --rm backend ls -la   # inspect image contents
docker compose down                      # always, before re-up
docker container prune                   # clear stopped containers
sudo ss -tulpn | grep :5432              # who holds a port (sudo for process name)
```

Kubernetes equivalents are nearly identical:

```bash
kubectl logs -f deploy/backend
kubectl logs --tail=50 deploy/backend
kubectl logs --previous deploy/backend   # ← no Compose equivalent
kubectl exec -it <pod> -- ls -la
```

`--previous` is the single most useful command in Phase 7: when a pod is in `CrashLoopBackOff`, it shows why the *last* attempt died, since the current container is usually too young to have logged anything.

**Logging convention:** all of this works because the app logs to stdout/stderr rather than a file. The runtime captures the streams; collection is somebody else's job. Any logger added later must keep writing to stdout.

---

## Key takeaways

1. **Compose answers one question — "are my images correct?" — and answers it in seconds.** Reaching Kubernetes with untrusted images means debugging two layers at once.
2. **The image is immutable.** Source edits require a rebuild. There is no live reload in a production container.
3. **Layer order is cache strategy.** Dependencies before source, or every code change re-installs `node_modules`.
4. **Multi-stage keeps build tools out of runtime.** Smaller image, smaller attack surface.
5. **`CMD` must be exec form and must not go through `npm`,** or SIGTERM never reaches the process and graceful shutdown silently doesn't happen.
6. **Generated native binaries are not compiled output.** `tsc` ignores Prisma's engine `.so.node` files — they need their own `COPY`.
7. **If a config file isn't `COPY`'d, it doesn't exist.** Read the build output; a missing step is visible there.
8. **Prefer plain `.js`/`.mjs` config over `.ts` in production images.** No loader required, works in an initContainer with no shell.
9. **Service names, not `localhost`.** `@db:5432` in Compose; `@postgres:5432` in Kubernetes. Same idea, and the whole reason Phase 0 removed hardcoded hosts.
10. **Don't publish ports nothing external needs.** Fewer collisions, and it matches the ClusterIP shape coming next.
11. **Tag images with real versions, never `latest`.** With `latest` you can't tell which build a pod is running, and rolling updates behave confusingly.

---

## Next: Phase 2 — Local cluster

- `kind create cluster` with a config: 1 control-plane + 2 workers, `ingress-ready=true` label, host 80/443 mapped through to the control plane
- Create the `store-app` namespace and set it as the default context namespace
- `kind load docker-image store-backend:v1 --name store-app` — the cluster's nodes are separate containers with their own image stores and no access to the local Docker daemon
- Verify with `docker exec store-app-worker crictl images | grep store` — first thing to check on `ErrImagePull`
- Smoke test with a throwaway nginx pod and `kubectl port-forward` before deploying anything real
