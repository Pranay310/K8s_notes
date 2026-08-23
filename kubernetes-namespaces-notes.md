# Kubernetes Notes: Namespaces (What, Why, How Framework)

---

## 1. What is a Namespace?

A **Namespace** provides a mechanism for isolating groups of resources within a single physical Kubernetes cluster.

### The Apartment Analogy
* **The Cluster:** Think of the entire physical/virtual cluster (nodes, CPU, memory, networking) as a **large apartment building**.
* **Namespaces:** Each namespace is an **individual apartment** inside that building.
* **Objects (Pods, Services, Deployments):** These are the furniture and rooms inside each apartment.
* Residents in Apartment `A` (Team Alpha) and Apartment `B` (Team Beta) both share the building's underlying utilities (cluster power, water, structure), but have private spaces and can name their rooms identically without conflict.

---

## 2. Why Use Namespaces?

Namespaces are essential for multi-tenant, multi-team, and multi-environment management.

| Benefit | Explanation |
| :--- | :--- |
| **Avoid Name Collisions** | Two different teams or apps can each create a Pod or Service named `backend-api` without naming clashes, provided they reside in different namespaces. |
| **Environment Isolation** | Separate environments like `dev`, `qa`, `staging`, and `prod` within the same cluster to keep workloads organized and prevent unintended changes. |
| **Resource Quotas (`ResourceQuota`)** | Enforce hard caps on total CPU, Memory, or Pod counts per namespace so one faulty deployment cannot starve other teams of cluster resources. |
| **Security & Access Control (RBAC)** | Restrict user and service account permissions to a specific namespace, preventing unauthorized modifications to production or system components. |

### Built-in Default Namespaces

When a Kubernetes cluster is initialized, the following namespaces are created automatically:

* `default` — The fallback namespace for any resource created without an explicit `--namespace` / `-n` flag or metadata declaration.
* `kube-system` — Reserved for Kubernetes core control plane and system workloads (e.g., `kube-dns`/`coredns`, networking plugins, API proxies).
* `kube-public` — A publicly readable namespace used for bootstrap cluster information (e.g., cluster info ConfigMaps).
* `kube-node-lease` — Holds heartbeat lease objects for each node to determine cluster availability efficiently.

---

## 3. How to Use Namespaces (Practical Guide)

### A. Creating Namespaces

#### Imperative (CLI):
```bash
kubectl create namespace staging
```

#### Declarative (YAML):
Save as `staging-namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    managed-by: platform-team
```
Apply using:
```bash
kubectl apply -f staging-namespace.yaml
```

---

### B. Viewing & Inspecting Namespaces

```bash
# List all namespaces in the cluster
kubectl get namespaces
# (Short alias: kubectl get ns)

# Describe a namespace to view labels, status, and active quotas
kubectl describe namespace staging
```

---

### C. Deploying & Managing Resources in a Namespace

```bash
# 1. Run a test pod inside the 'staging' namespace
kubectl run test-web --image=nginx:alpine -n staging

# 2. List pods running in the 'staging' namespace
kubectl get pods -n staging

# 3. List pods across ALL namespaces at once
kubectl get pods -A
# or
kubectl get pods --all-namespaces

# 4. Apply any YAML file into a target namespace directly
kubectl apply -f deployment.yaml -n staging

# 5. Delete a pod from a specific namespace
kubectl delete pod test-web -n staging
```

---

### D. Changing Default Namespace Context

To avoid typing `-n staging` on every single command, set the current context namespace:

```bash
# Switch default context namespace to 'staging'
kubectl config set-context --current --namespace=staging

# Verify current namespace context
kubectl config view --minify | grep namespace:
```

---

### E. Deleting a Namespace

> ⚠️ **Caution:** Deleting a namespace cascades and deletes **all resources** (Pods, Services, Secrets, Deployments) inside it!

```bash
kubectl delete namespace staging
```

---

## 4. Key Rules & Best Practices

1. **Not all resources are namespaced:**
   * **Namespaced resources:** Pods, Deployments, Services, ConfigMaps, Secrets, PVCs.
   * **Cluster-wide (non-namespaced) resources:** Nodes, PersistentVolumes (PV), StorageClasses, ClusterRoles, Namespaces themselves.
   * *Check if a resource is namespaced:*
     ```bash
     kubectl api-resources --namespaced=true
     kubectl api-resources --namespaced=false
     ```

2. **Cross-Namespace Communication:**
   * Pods across namespaces can still communicate by default using Kubernetes Full Qualified Domain Name (FQDN):
     ```
     <service-name>.<namespace-name>.svc.cluster.local
     ```
   * Example: A service named `db` in `staging` is reachable at `http://db.staging.svc.cluster.local:5432`.
