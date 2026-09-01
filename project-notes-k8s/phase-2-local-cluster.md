# Phase 2 — Local Kubernetes Cluster

**Project:** Store Rating Platform (React + Vite / Express + Prisma / PostgreSQL)
**Goal of this phase:** get a working cluster on the laptop, with the app images available inside it, and confirm it can actually run something.

**Outcome:** 3-node cluster running, `store-app` namespace as the default, both images loaded onto every node, and a throwaway test pod reached successfully.

---

## What a "local cluster" actually is

kind stands for **K**ubernetes **in** **D**ocker. Each node is a Docker container that behaves like a machine in a real cluster.

This matters more than it sounds, and explains most of the quirks in this phase:

- Nothing inside is reachable from the browser unless a port is deliberately published — exactly like Compose.
- Each node has its own separate image storage. It cannot see the images on the laptop's Docker.

Everything unusual in the config file below exists to fake, on one laptop, what real infrastructure provides.

---

## Tools

```bash
kind version
kubectl version --client
```

Two separate things:

- **kind** — creates and destroys clusters. Used a handful of times.
- **kubectl** — talks to a running cluster. Used constantly from here on.

Before creating the cluster, confirm ports 80 and 443 are free, because the cluster will claim them:

```bash
sudo ss -tulpn | grep -E ':80 |:443 '
```

Empty output is what's wanted.

---

## Cluster config — `kind-config.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: store-app
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-config.yaml
```

### Line by line

**`kind: Cluster` / `apiVersion:`** — every Kubernetes-style file starts with what type of thing it is and which version of the format. Confusingly, `kind:` here means "type of object", and the tool is also called kind. Every manifest from Phase 3 onward has these same two fields.

**`name: store-app`** — prefixes the node container names (`store-app-control-plane`, `store-app-worker`, `store-app-worker2`) and creates the kubeconfig context `kind-store-app`. Needed for `kind load --name store-app` and `kind delete cluster --name store-app`.

**`nodes:`** — three entries, three Docker containers. **One control plane, two workers.**

- **Control plane** runs the cluster's brain: the API server (what kubectl talks to), etcd (the datastore for all cluster state), the scheduler (picks which node each pod goes on), and the controller-manager (loops that make reality match what was asked for).
- **Workers** run application pods.

By default the control plane refuses ordinary app pods, so everything deployed lands on the two workers.

**Why two workers:** pods visibly spread across nodes. Scaling the frontend to 3 replicas in Phase 5 and running `kubectl get pods -o wide` shows them distributed, which makes scheduling concrete. It also allows experimenting with node affinity and draining a node later.

**Cost:** each node is roughly 500MB–1GB of RAM. Dropping to one worker breaks nothing in the plan if the laptop struggles.

**`kubeadmConfigPatches` / `node-labels: "ingress-ready=true"`** — attaches a label to the control-plane node when it registers. The ingress-nginx install manifest for kind requires a node with exactly this label. **Without it, the ingress controller sits `Pending` forever in Phase 6** with no obvious cause — nothing looks broken, the pod just never gets scheduled. Annoying enough to be worth getting right upfront.

The `- |` is YAML syntax for "keep this block as literal multi-line text".

**`extraPortMappings`** — the piece that makes the cluster reachable at all. Since nodes are Docker containers, host port 80 has to be published to the control-plane container's port 80, the same idea as `-p 8080:80` in Compose.

The Phase 6 path becomes:

```
browser → localhost:80 → control-plane container:80 → ingress-nginx → Service → pod
```

The mapping sits on the control plane because that's where ingress-nginx will run (thanks to the label). 443 is mapped for the TLS experiment later.

Without these mappings, everything would have to go through `kubectl port-forward`, which skips Services and Ingress entirely — and so can't test the routing this project is trying to learn.

> **This whole file is a kind concern, not a Kubernetes one.** Real clusters have real machines with real IPs and a cloud load balancer in front. Nothing in the application manifests from Phase 3 onward references any of this.

---

## Orientation

```bash
kubectl get nodes
kubectl get pods -A
```

`-A` means all namespaces — show everything, not just my stuff.

Expected: three nodes, all `Ready`.

```
NAME                      STATUS   ROLES           AGE   VERSION
store-app-control-plane   Ready    control-plane   46m   v1.36.1
store-app-worker          Ready    <none>          46m   v1.36.1
store-app-worker2         Ready    <none>          46m   v1.36.1
```

`get pods -A` shows about a dozen pods nobody created — that's the cluster running itself. Worth recognising these names, since they turn up in error messages:

| Pod | What it does |
|---|---|
| **etcd** | the database holding everything the cluster knows |
| **kube-apiserver** | the front door — every kubectl command talks to this |
| **kube-scheduler** | decides which node each new pod goes on |
| **controller-manager** | loops that make actual state match desired state |
| **coredns** | the phone book — turns a name like `postgres` into an address |
| **kube-proxy** | makes network traffic reach the right pod |
| **kindnet** | networking layer, specific to kind |

**CoreDNS being just a pod** explains a lot later — when something reports "cannot resolve host", this is the thing involved.

---

## Namespace

```bash
kubectl create namespace store-app
kubectl config set-context --current --namespace=store-app
kubectl config view --minify | grep namespace
```

A namespace is a folder for grouping things, so the app's objects don't mix with the cluster's own. Setting it as the default avoids typing `-n store-app` on every command for the rest of the project.

It's also the cleanest reset button: `kubectl delete namespace store-app` removes everything at once and re-applying from scratch proves nothing was left in a weird manual state.

---

## Loading images — the step that's easy to miss

```bash
kind load docker-image store-backend:v1 --name store-app
kind load docker-image store-frontend:v1 --name store-app
```

The images live in the laptop's Docker. The cluster nodes are separate containers with their own image storage and no access to it. `kind load` copies them across, onto **every** node — because any node might be asked to run the pod.

Verify:

```bash
docker exec store-app-worker crictl images | grep store
```

```
docker.io/library/store-backend     v1   7397b4e4d8fa3   199MB
docker.io/library/store-frontend    v1   c4a8df5c4895a   21.4MB
```

**This is the first thing to check when a pod reports `ErrImagePull` or `ImagePullBackOff` in Phase 4.** The usual cause is either forgetting to load after a rebuild, or forgetting `imagePullPolicy: IfNotPresent` — without it Kubernetes tries to fetch from Docker Hub, where these images don't exist.

Two habits worth keeping:

- **Always tag with real versions, never `latest`.** With `latest` there's no way to tell which build a pod is running, and rolling updates behave confusingly.
- **Rebuild → reload → restart.** Editing source changes nothing until the image is rebuilt *and* re-loaded into the cluster. Same immutability rule as Phase 1, now with an extra step.

---

## Problem hit: docker group permissions

`docker exec` failed with `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`, forcing `sudo` on every command.

The group had been added correctly:

```bash
getent group docker     # docker:x:124:pranay
```

But `id -nG | grep docker` returned nothing. **Linux reads group membership at login**, so an existing session doesn't see the change. Closing the terminal isn't enough — the fix is logging out of the desktop session entirely and back in (or rebooting).

`newgrp docker` gives a working shell immediately, but only for that one terminal — a patch, not a fix.

**Why this mattered enough to stop and fix:** kind and kubectl store their settings in the home folder. Running `sudo kind` writes as root while `kubectl` reads as the normal user, producing confusing "cluster not found" errors where nothing is actually wrong.

**Security note:** membership in the `docker` group is effectively full admin access on the machine, since any part of the filesystem can be mounted into a container. Normal for a personal dev laptop; not appropriate on a shared machine.

---

## Smoke test

Prove the cluster runs pods before deploying anything real:

```bash
kubectl run nginx-test --image=nginx:alpine
kubectl get pods -o wide
```

`-o wide` adds columns, including which node the pod landed on — a worker, confirming the control plane refuses ordinary pods.

```
NAME         READY   STATUS              RESTARTS   AGE   NODE
nginx-test   0/1     ContainerCreating   0          0s    store-app-worker2
```

`ContainerCreating` means it's still pulling and starting.

The **READY** column is worth understanding early: `1/1` means one container in the pod, one ready. `0/1` means the container exists but isn't ready — which in Phase 4 will be a readiness probe deciding the backend can't serve traffic yet.

```bash
kubectl port-forward pod/nginx-test 8888:80
# other terminal:
curl localhost:8888
kubectl delete pod nginx-test
```

**Why `port-forward` matters:** it opens a tunnel from the laptop straight into one pod, skipping Services and Ingress entirely. When something breaks later, it answers "is the pod itself working?" without routing muddying the picture. It becomes the most-used debugging tool from here on.

---

## Commands learned

```bash
kind create cluster --config kind-config.yaml
kind get clusters
kind delete cluster --name store-app
kind load docker-image <image>:<tag> --name store-app

kubectl get nodes -o wide
kubectl get pods -A
kubectl config get-contexts
kubectl config set-context --current --namespace=store-app
kubectl run <name> --image=<image>
kubectl port-forward pod/<name> 8888:80
kubectl delete pod <name>

docker exec store-app-worker crictl images | grep store
```

`crictl` is the tool for inspecting containers on a Kubernetes node — the node-level equivalent of `docker images`.

---

## Key takeaways

1. **kind nodes are Docker containers.** They have their own image storage and are unreachable unless ports are deliberately mapped. Both facts drive the whole config file.
2. **`kind load` is a required step, not an optional one.** The cluster cannot see local images. Forgetting it after a rebuild is a recurring source of `ErrImagePull`.
3. **The `ingress-ready=true` label must exist before Phase 6**, or the ingress controller never gets scheduled and nothing obviously looks wrong.
4. **Host ports 80/443 must be mapped at cluster creation.** They can't be added later without recreating the cluster — which is cheap, since kind clusters are disposable.
5. **Set the default namespace once** and save `-n store-app` on hundreds of commands.
6. **The control plane won't run app pods** by default. Seeing pods land only on workers is correct behaviour, not a scheduling failure.
7. **Fix the docker group before doing anything else.** Mixing root and non-root breaks kubeconfig in confusing ways.
8. **`port-forward` isolates the pod from the routing.** The first question when something breaks is always "is the pod itself fine?"
9. **Clusters are disposable.** Deleting one with unknown config and starting fresh took about a minute and was cheaper than auditing it.

---

## Next: Phase 3 — Database in the cluster

Postgres with storage that survives the pod being deleted, and credentials kept out of the manifests:

- **Secret** for the credentials
- **StatefulSet** rather than Deployment — a database isn't interchangeable, it has data
- **volumeClaimTemplates** requesting disk that outlives the pod
- **Headless Service** (`clusterIP: None`) so the name `postgres` points at the one pod rather than load-balancing
- **The test that proves it:** create a table, delete the pod, wait for the replacement, confirm the table is still there
