# Kubernetes Notes: Pods (What, Why, How Framework)

---

## 1. What is a Pod?

A **Pod** is the smallest and most basic deployable unit in Kubernetes. Kubernetes does not run containers directly on nodes; instead, it always encapsulates one or more containers inside a Pod.

### Key Characteristics:
* **Single-container Pods:** The most common pattern (1 container per Pod).
* **Multi-container Pods:** Used for tightly coupled helper containers (e.g., sidecars, proxies, log shippers) that need to share resources.
* **Shared Context:** Containers within the same Pod share:
  * The same **Network Namespace** (they share an IP address and communicate via `localhost`).
  * The same **Storage Volumes** (mounted data accessible across containers in the Pod).

---

## 2. Pod Mental Model & Analogy

### The Pea Pod / Pod of Whales
* **The Pod:** The outer protective shell.
* **Containers:** The individual peas inside the shell.
* **Nodes:** The garden bed hosting multiple pea pods.

---

## 3. How to Create a Pod

### Method 1: Imperative CLI (Quick testing & debugging)

```bash
# Create and run an Nginx pod directly
kubectl run my-first-pod --image=nginx:alpine --port=80
```

---

### Method 2: Declarative YAML (Production Standard)

Save the following specification as `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-web-pod
  labels:
    app: web
    env: development
spec:
  containers:
    - name: nginx-container
      image: nginx:alpine
      ports:
        - containerPort: 80
```

Apply the manifest:
```bash
kubectl apply -f pod.yaml
```

---

## 4. Pod Lifecycle & Operational Commands

| Task | Command | Description |
| :--- | :--- | :--- |
| **List Pods** | `kubectl get pods` | Check status (`Running`, `Pending`, `CrashLoopBackOff`) and readiness. |
| **Watch Pods** | `kubectl get pods -w` | Stream real-time status changes in your terminal. |
| **Inspect Details** | `kubectl describe pod my-web-pod` | View detailed configuration, conditions, and recent cluster events (crucial for debugging). |
| **View Logs** | `kubectl logs my-web-pod` | Stream stdout/stderr console logs from the container. |
| **Follow Logs** | `kubectl logs -f my-web-pod` | Follow application logs in real-time (`tail -f`). |
| **Interactive Shell** | `kubectl exec -it my-web-pod -- sh` | Access an interactive terminal inside the running container. |
| **Port Forwarding** | `kubectl port-forward pod/my-web-pod 8080:80` | Forward local port `8080` to container port `80` (accessible at `http://localhost:8080`). |
| **Delete Pod** | `kubectl delete pod my-web-pod` | Terminate and remove the pod from the cluster. |

---

## 5. Important Considerations for Production

* **Pods are Ephemeral (Mortal):** Pods are designed to be temporary. If a Pod crashes or a Node fails, a standalone Pod will not automatically heal or recreate itself.
* **Deployments over Pods:** In real-world production environments, never create bare Pods directly. Always manage them using higher-level abstractions like **Deployments**, **StatefulSets**, or **DaemonSets** to get self-healing, rolling updates, and scaling.