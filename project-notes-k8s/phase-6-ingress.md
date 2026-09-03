# Phase 6 — Ingress: Closing the Routing Gap

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** put one front door in front of both the frontend and backend, so the browser sees a single origin.

**Outcome:** the whole app works end to end at `http://store.local` — register, log in, browse, rate. First time in the project the full stack runs in the cluster.

---

## The gap being closed

Since Phase 1 there's been a hole: the frontend requests `/api/v1/...`, the browser sends that to whatever origin served the page, and in Compose nothing was listening for it. The request hit nginx, fell through to the SPA fallback, and returned 405.

Every environment needs something in front doing this split. This phase supplies the Kubernetes version.

| Environment | Router |
|---|---|
| `npm run dev` | Vite dev proxy |
| `npm run preview` | Vite preview proxy |
| Compose | *nothing — the gap* |
| **Kubernetes** | **Ingress** |

---

## What Ingress is

Two separate things, easy to confuse:

- **Ingress** (the YAML) — a set of routing rules. Just data. On its own it does nothing.
- **Ingress controller** (the pod) — the thing that reads those rules and actually routes traffic. Here it's nginx.

Writing an Ingress resource with no controller installed produces no error and no effect. The rules sit there unread.

---

## 1. Install the controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

It installs into its own `ingress-nginx` namespace, so every command about it needs `-n ingress-nginx` — the default namespace is `store-app`.

### Problem hit: scheduled on the wrong node

```bash
kubectl get pods -n ingress-nginx -o wide
# ingress-nginx-controller-...  Running  store-app-worker2   ← wrong
```

**Why this matters:** host ports 80 and 443 were mapped to the **control-plane** container back in Phase 2. Traffic to `localhost:80` arrives there. With the controller on worker2, nothing answers.

The label was present and correct:

```bash
kubectl get nodes --show-labels | grep ingress-ready
# store-app-control-plane   ...,ingress-ready=true,...
```

But the Deployment wasn't asking for it:

```bash
kubectl get deploy -n ingress-nginx ingress-nginx-controller -o yaml | grep -A5 nodeSelector
#   nodeSelector:
#     kubernetes.io/os: linux        ← no ingress-ready
```

**Cause:** the manifest pulled from `main` wasn't the kind-specific build, so it lacked the kind scheduling rules. (The `hostPort` settings *were* present — only the node selection was missing.)

**Fix:**

```bash
kubectl patch deploy -n ingress-nginx ingress-nginx-controller \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}]}}}}'
```

Two pieces:

- **`nodeSelector`** — only run me on a node with this label.
- **`tolerations`** — and I'm allowed on the control plane despite its rule against ordinary pods. That rule is a **taint**; without a matching toleration the pod would sit `Pending` forever.

Patching a Deployment triggers a rollout automatically, so the replacement landed on the control plane.

### `hostPort` — the piece that makes it reachable

```yaml
ports:
- containerPort: 80
  hostPort: 80
- containerPort: 443
  hostPort: 443
```

`hostPort` binds the pod directly to that port on its node. Combined with the Phase 2 `extraPortMappings`, the full chain is:

```
browser → localhost:80 → control-plane container:80 → controller pod → Service → app pods
```

**`EXTERNAL-IP: <pending>` on the controller Service is normal here and can be ignored.** A `LoadBalancer` Service expects a cloud provider to hand it a public IP, and there isn't one locally. It doesn't matter, because `hostPort` bypasses that path entirely.

---

## 2. Hostname

```bash
echo "127.0.0.1 store.local" | sudo tee -a /etc/hosts
```

Nothing clever — a local text file the browser checks before doing real DNS. It just says `store.local` means this machine.

---

## 3. `k8s/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-app
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
    - host: store.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

In plain terms: a request for `store.local` whose path starts with `/api` goes to the backend on 3000; everything else goes to the frontend on 80.

**Order matters.** `/` matches everything, so `/api` must come first.

**`pathType: Prefix`** means "starts with", so `/api` also matches `/api/v1/auth/login`.

**`ingressClassName: nginx`** picks which controller handles these rules. Only one is installed here, but a cluster can run several.

---

## Verification

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress

curl http://store.local/healthz        # ok            ← frontend pod (nginx)
curl http://store.local/api/v1/store   # Unauthorized  ← backend pod (Express)
```

**The `Unauthorized` is the better result of the two.** It's a JSON error from the app's own auth middleware, which means the request travelled the entire chain and reached Express. A 405 or an HTML page would have meant routing failed.

---

## Problem hit: cookies rejected by the browser

Login succeeded; every authenticated request afterwards returned 401.

The cookie-setting code:

```ts
res.cookie("token", token, {
    httpOnly: true,
    secure: config.NODE_ENV === "production",
    sameSite: config.NODE_ENV === "production" ? "none" : "lax",
});
```

The ConfigMap sets `NODE_ENV: "production"`, so this produced `secure: true, sameSite: "none"`.

- **`secure: true`** means "only send over HTTPS." The site is plain HTTP, so the browser dropped it.
- **`SameSite=None` without `Secure` is invalid**, and browsers reject the cookie outright.

It had always worked in development because `NODE_ENV` was unset there.

### The real lesson

**`NODE_ENV === "production"` was standing in for "am I behind HTTPS?"** Those are different questions. They only looked identical because dev happened to be HTTP and production happened to be HTTPS.

The moment a third combination appeared — **production-mode code on plain HTTP** — the assumption broke.

> Config should describe what is actually true about the environment, not be inferred from a label.

### Fix

`config/env.ts`:

```ts
COOKIE_SECURE: z.string().default("false"),
```

`auth.controller.ts` (both `login` and `logout`):

```ts
const isSecure = config.COOKIE_SECURE === "true";

res.cookie("token", token, {
    httpOnly: true,
    secure: isSecure,
    sameSite: isSecure ? "none" : "lax",
    maxAge: durationToMs(config.JWT_EXPIRES_IN),
});
```

`k8s/backend-config.yaml`:

```yaml
data:
  COOKIE_SECURE: "false"
```

Adding TLS later means flipping that one value to `"true"` and rolling out — no code change.

**Note:** with Ingress putting both halves on one origin, `sameSite: "none"` isn't needed at all — that setting exists for cross-origin cookies. `"lax"` is correct and safer. Tying it to `isSecure` keeps it sensible if the origins are ever split again.

### Follow-on: only half the change landed

After rebuilding, the cookie was *still* wrong. Inspecting the compiled code inside the image:

```bash
docker run --rm store-backend:v2 grep -n "sameSite\|COOKIE_SECURE\|isSecure" dist/controllers/auth.controller.js
```

```
39: const isSecure = config.COOKIE_SECURE === "true";
70:     secure: isSecure,
71:     sameSite: config.NODE_ENV === "production" ? "none" : "lax",   ← missed
```

The `secure:` line had been updated; the `sameSite:` line hadn't.

**This debugging technique is worth keeping.** `docker run --rm <image> grep ...` reads the compiled code inside the image directly, answering "is my change actually in there?" in one command without deploying anything.

The complementary check, for whether the *cluster* is running that image:

```bash
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

Between the two, "my change isn't working" resolves into one of three specific questions: is it in the source, is it in the image, or is it in the cluster?

### Checking the cookie without a browser

```bash
curl -i -X POST http://store.local/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"...","password":"..."}' | grep -i set-cookie
```

curl accepts cookie combinations browsers reject, which makes it useful for reading the raw header — and a reminder that "curl works" doesn't mean "the browser will."

`-c file` saves cookies, `-b file` sends them back, for testing an authenticated flow end to end.

---

## The end result

```
browser → store.local:80 → control-plane container → ingress controller
        → path starts with /api? → backend Service  → 1 of 2 backend pods
        → otherwise             → frontend Service → 1 of 2 frontend pods
```

**And notice what isn't happening:** no CORS errors, no preflight requests, no cross-origin cookie problems. The browser sees one origin.

That's the same-origin architecture chosen back in Phase 0, with Ingress now playing the role Vite's proxy plays in development. Same shape, different machinery, and the frontend bundle unchanged across all of them.

---

## Commands learned

```bash
kubectl wait --for=condition=ready pod --selector=... --timeout=120s
kubectl get pods -n ingress-nginx -o wide
kubectl get nodes --show-labels | grep ingress-ready
kubectl patch deploy -n ingress-nginx <name> --type=merge -p '{...}'
kubectl get ingress
kubectl describe ingress store-app
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller   # logs every routed request
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
docker run --rm <image> grep -n "pattern" dist/path/to/file.js
```

---

## Key takeaways

1. **Ingress is rules; the controller is the thing that acts on them.** Rules with no controller fail silently.
2. **The controller must run where the host ports are mapped.** `nodeSelector` + `tolerations` put it on the control plane; `hostPort` binds it to port 80 there.
3. **A taint repels pods; a toleration is permission to ignore that.** Without one, a control-plane placement just sits `Pending`.
4. **`EXTERNAL-IP: <pending>` is expected locally** and doesn't indicate a problem.
5. **Path order matters** — specific before general, since `/` matches everything.
6. **Don't infer environment facts from `NODE_ENV`.** "Am I in production?" and "am I on HTTPS?" are different questions, and conflating them breaks the moment a third combination appears.
7. **`SameSite=None` requires `Secure`.** Browsers reject the pair silently; curl doesn't, so curl succeeding proves nothing about the browser.
8. **When behaviour doesn't match the source, check the image, then the cluster.** Two commands narrow it down immediately.
9. **A partial edit compiles fine and deploys fine.** TypeScript can't catch a line you forgot to change — only reading the built output can.

---

## Next: Phase 7 — Day-2 operations

Now that there's a working system, break it deliberately:

- Rolling updates and `kubectl rollout undo`
- Force `CrashLoopBackOff` (wrong DB password), `ImagePullBackOff` (typo'd tag), `Pending` (impossible resource request), and OOMKilled (memory limit set too low)
- Read each one with `kubectl describe` and `kubectl logs --previous`
- Install metrics-server, add an HPA, load-test the ratings endpoint and watch replicas scale
