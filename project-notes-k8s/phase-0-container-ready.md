# Phase 0 — Making the App Container-Ready

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** change nothing about what the app *does*, and everything about how it *gets its configuration and reports its health* — so it can survive being packed into a container and scheduled by Kubernetes.

Nothing in this phase involves Docker or Kubernetes. That's deliberate. Every problem fixed here would otherwise show up later disguised as a Kubernetes problem, which is a much worse place to debug it.

---

## The one idea behind the whole phase

> **The application must not know anything about where it is running.**

Locally, `localhost:5432` is the database. In Kubernetes, it's `postgres.store-app.svc.cluster.local`. In production it might be a managed RDS endpoint. The app must not care. Everything environment-specific comes in from outside, at runtime.

This is the [Twelve-Factor App](https://12factor.net/config) config principle, and it's the reason all the changes below exist.

---

## Steps taken

### 1. Centralized and validated environment config

**File:** `src/config/env.ts`

- All env vars parsed through a single zod schema, exported as `config`.
- `process.exit(1)` on a validation failure.
- `PORT` uses `z.coerce.number()` so it isn't a string.
- `NODE_ENV` restricted to an enum.
- `JWT_SECRET` given a `.min(16)`.
- `dotenv` only loads when `NODE_ENV !== "production"`.
- Removed the duplicate `dotenv.config()` from `app.ts`.

**Why:** In a container, a missing or malformed env var is the single most common failure. Without validation, a missing `DATABASE_URL` surfaces as a confusing Prisma error on the third request instead of an obvious crash at boot. **Fail fast and loudly at startup** — Kubernetes will restart the pod, you'll see `CrashLoopBackOff`, and the logs will name the exact missing key.

Loading `.env` in production is wrong because Kubernetes injects real environment variables from ConfigMaps and Secrets. There is no `.env` file in the image.

**Rule that came out of this:** only `env.ts` may touch `process.env`. Everywhere else imports `config`.

---

### 2. Bound the server to `0.0.0.0`

**File:** `src/index.ts`

```ts
app.listen(config.PORT, "0.0.0.0", () => { ... });
```

**Why:** Binding to `127.0.0.1` means "only accept connections originating inside this network namespace." A container *is* its own network namespace, so the port would be published but nothing could reach it — and the Kubernetes readiness probe would fail forever with no useful error.

Verified with `ss -tulpn | grep :3000`, which showed `0.0.0.0:3000` rather than `127.0.0.1:3000`.

---

### 3. Used the validated config for the port

**Before:** `const PORT = process.env.PORT || 5000;`
**After:** `config.PORT`

**Why:** The old line bypassed validation entirely and introduced a second, contradictory default (5000 vs 3000). Two sources of truth for the same value is a bug waiting for a bad day.

---

### 4. Added separate liveness and readiness endpoints

```ts
app.get("/healthz", (_req, res) => res.status(200).send("ok"));

app.get("/readyz", async (_req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    res.status(200).send("ready");
  } catch {
    res.status(503).send("not ready");
  }
});
```

**Why they are different — this is the part worth remembering:**

| Probe | Question it answers | What Kubernetes does on failure |
|---|---|---|
| **Liveness** (`/healthz`) | Is the process wedged? | **Kills and restarts the pod** |
| **Readiness** (`/readyz`) | Can it serve traffic right now? | **Removes the pod from the Service** — no restart |

Putting the DB check in `/healthz` is a classic self-inflicted outage: a 30-second database blip would cause Kubernetes to kill *every* backend pod simultaneously, turning a brief blip into a full restart storm. Liveness must only test the process itself. Readiness tests dependencies.

---

### 5. Added graceful shutdown on SIGTERM

```ts
const shutdown = (signal: string) => {
  server.close(async () => {
    await prisma.$disconnect();
    process.exit(0);
  });
  server.closeIdleConnections();
  setTimeout(() => { server.closeAllConnections(); process.exit(1); }, 10_000).unref();
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT",  () => shutdown("SIGINT"));
```

**Why:** During any rolling update, scale-down, or node drain, Kubernetes sends `SIGTERM` and waits ~30s before `SIGKILL`. Node.js does **not** handle `SIGTERM` by default in a container — the process is simply killed, dropping every in-flight request. Every deployment would produce user-visible errors.

`closeIdleConnections()` matters because `server.close()` waits for keep-alive connections to end on their own; an open browser tab is enough to hang shutdown indefinitely. The `unref()`'d timeout is the force-quit backstop.

**Related discovery:** `npm start` adds a process layer that does not reliably forward signals to the child. Hence the Phase 1 rule: **container `CMD` must be `["node", "dist/index.js"]`, never `["npm", "start"]`.**

---

### 6. Removed seeding from application startup

`seedAdmin()` was being awaited inside `startServer()`. It now lives in its own entrypoint, `src/seed.ts`, exposed as an npm script.

```json
"scripts": {
  "build": "prisma generate && tsc",
  "start": "node dist/index.js",
  "migrate:deploy": "prisma migrate deploy",
  "seed": "node dist/seed.js"
}
```

**Why:** With 3 replicas, startup seeding means three pods racing to create the same admin row. More fundamentally, **a container should do exactly one job.** Serving traffic and mutating schema/data are different jobs with different lifecycles.

In Phase 4 these become:
- `migrate:deploy` → an **initContainer** (blocks app start until schema is ready)
- `seed` → a **Job** (runs once, then exits)

The old `catch` block also logged the failure without exiting, leaving a zombie process that Kubernetes would consider healthy. Removed.

---

### 7. Made the frontend use a relative API base URL

**`src/api/axios.ts`:**
```ts
baseURL: import.meta.env.VITE_API_URL ?? "/api/v1"
```

**`vite.config.ts`:**
```ts
server:  { proxy: { "/api": "http://localhost:3000" } },
preview: { proxy: { "/api": "http://localhost:3000" } },
```

**Why this is the most important change in Phase 0:**

Vite substitutes `import.meta.env.VITE_*` at **build time**. The value is welded into the JavaScript bundle as a literal string. A Kubernetes env var **cannot** change it later — env vars reach running processes, and the bundle is a static file being served by nginx.

Bake in `http://localhost:3000` and you need a separate image per environment, which breaks the core container principle: **build once, deploy anywhere.**

With a relative path, the browser resolves `/api/v1` against whatever origin served the page, and a reverse proxy decides where that actually goes. The identical bundle works in all three environments.

---

### 8. Housekeeping

- Fixed the `.env` mismatch — `POSTGRES_DB=rating_store` vs `store_app` in `DATABASE_URL`. In Kubernetes, `DATABASE_URL` gets **composed from** the `POSTGRES_*` values, so this would have silently pointed at a nonexistent database.
- Removed deprecated `baseUrl` from `tsconfig.app.json` (unnecessary since TS 4.1; `paths` resolves relative to the tsconfig).
- Confirmed the matching `resolve.alias` exists in `vite.config.ts` — tsconfig paths only affect type-checking, and the Docker build runs `vite build` in a clean container where editor-only conveniences don't exist.
- `dist/` removed from git and added to `.gitignore` — stale committed build artifacts cause genuinely baffling container bugs.
- `.env` in `.gitignore`; `.env.example` committed as the reference for writing ConfigMaps and Secrets.
- Picked a single package manager and deleted the other lockfile.
- **Pending:** if the image is Alpine-based, `schema.prisma` needs `binaryTargets = ["native", "linux-musl-openssl-3.0.x"]`. Simpler to use `node:22-slim` and skip it — Alpine + Prisma has a long history of papercuts.

---

## The concept: reverse proxy / same-origin architecture

The frontend requests `/api/v1/store`. The browser sends it to the origin that served the page. A proxy forwards anything matching `/api` to the backend.

```
DEV / PREVIEW
browser ──▶ localhost:4173/api/v1/store    (Vite proxy)
                    │
                    ▼
            localhost:3000/api/v1/store    (Express)

KUBERNETES
browser ──▶ store.local/api/v1/store       (Ingress)
                    │  path rule /api
                    ▼
            backend Service :3000          (Express pod)

browser ──▶ store.local/                   (Ingress)
                    │  path rule /
                    ▼
            frontend Service :80           (nginx pod)
```

Different machinery, **identical shape**. The browser only ever knows one address.

**Analogy:** a hotel front desk. You call one number and ask for room service; you never learn room service has its own extension.

**Forward vs reverse proxy:**
- *Forward* proxy sits in front of **you**, hiding you from servers (a VPN).
- *Reverse* proxy sits in front of **servers**, hiding them from you (this).

### Why we do it, ranked

1. **Portability** — one image, config injected at runtime. The reason this matters most is that the alternative is undiscoverable until Phase 5, when the frontend pod serves a bundle that keeps calling `localhost:3000` and it can't be fixed without a rebuild.
2. **Simplicity** — same origin means CORS, preflight requests, and cookie `SameSite` problems simply stop existing.
3. **Security** — the backend never needs to be publicly reachable, so its Service can stay `ClusterIP` (cluster-internal only). One hardened entry point for TLS, rate limiting, WAF.

### What it does *not* do

A reverse proxy hides **where** the backend lives, not **what it does**. Anyone can still `curl store.local/api/v1/store`. JWT middleware and RBAC checks are what protect the data. Treating a proxy as an authorization layer is a classic and dangerous mistake.

---

## Verification performed

**Backend** — started with env vars supplied inline, no `.env` involvement:

```bash
npm run build
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/store_app?schema=public" \
POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres POSTGRES_DB=store_app \
JWT_SECRET=dev_secret_at_least_16_chars PORT=3000 NODE_ENV=production \
node dist/index.js
```

Checks: `/healthz` → 200, `/readyz` → 200, a real API call succeeds, `ss -tulpn | grep :3000` shows `0.0.0.0`.

**Signal handling:**
```bash
node dist/index.js          # terminal 1
kill -TERM $(lsof -t -i :3000)   # terminal 2
```
Expected: shutdown log, clean exit in under a second.

**Frontend** — `npm run build && npm run preview`, then log in and exercise real API calls at `localhost:4173`. Requests appear in the network tab as `localhost:4173/api/v1/...` and succeed. That proves the relative base URL survived the build with no backend address inside it.

---

## Useful commands learned

```bash
ss -tulpn | grep :3000        # what's listening on a port (modern netstat)
ss -tulpn | grep node         # all node listeners — catches orphans
lsof -t -i :3000              # just the PID
kill -TERM <pid>              # graceful stop, the signal k8s sends
kill -9 <pid>                 # force — what k8s sends after the grace period

node dist/index.js > /tmp/backend.log 2>&1 &   # background with logs to file
tail -f /tmp/backend.log
jobs / fg / kill %1                             # job control
```

---

## Key takeaways

1. **Config comes from the environment, never the code.** One validated module, fail fast at boot.
2. **Bind `0.0.0.0`, not `127.0.0.1`.** A container is its own network namespace.
3. **Liveness ≠ readiness.** Liveness restarts the pod; readiness only pulls it from the load balancer. Never check dependencies in liveness.
4. **Handle SIGTERM.** Node ignores it by default, and every rolling update depends on it. Don't let `npm` sit between the signal and your process.
5. **One container, one job.** Migrations and seeding are separate entrypoints, not startup side effects.
6. **Build-time vs runtime config is the frontend trap.** Vite bakes `VITE_*` into the bundle permanently. Relative URLs + a reverse proxy is the escape.
7. **Build once, deploy anywhere.** If an environment needs a different image, the config is in the wrong place.
8. **If it only works because of your laptop, it's broken.** Local shell exports, editor path resolution, a `.env` that happens to be there — all absent in the container.

---

## Next: Phase 1 — Dockerize

- Multi-stage `Dockerfile` for backend (build: `prisma generate` + `tsc`; runtime: slim, non-root, `CMD ["node","dist/index.js"]`)
- Multi-stage `Dockerfile` for frontend (build: `vite build`; runtime: `nginx:alpine` with SPA fallback `try_files $uri /index.html`)
- `.dockerignore` in both (`node_modules`, `dist`, `.env`, `.git`)
- Prove it with `docker compose up` before touching Kubernetes — if it fails in Compose it will fail in Kubernetes, and Compose is far faster to debug.
