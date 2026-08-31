# Kubernetes Notes: Deployments (What, Why, How Framework)

---

## 1. What is a Deployment?

A **Deployment** is a higher-level Kubernetes controller that provides declarative updates for Pods and ReplicaSets. Instead of manually creating individual Pods, a Deployment lets you describe a desired state (e.g., "keep 3 replicas of `nginx:1.25` running"), and the controller automatically handles Pod lifecycle management.

### Key Capabilities:
* **Self-Healing:** Automatically replaces Pods that crash or live on failed nodes.
* **Declarative Scaling:** Scale up or down with a single command or YAML update.
* **Rolling Updates:** Incrementally update Pod instances with zero downtime.
* **Rollbacks:** Revert to previous working versions if an update fails.

---

## 2. Hierarchy & Mental Model

### The Factory Analogy
* **Deployment (Factory Manager):** Coordinates the overall release process, rollout strategy, and desired replica count.
* **ReplicaSet (Floor Supervisor):** Automatically generated and managed by the Deployment to ensure the exact number of Pods are running.
* **Pods (Factory Workers):** The individual units running the application containers.

```
Deployment (nginx-deployment)
   └── ReplicaSet (nginx-deployment-77f9859f7b)
          ├── Pod (nginx-deployment-77f9859f7b-1)
          ├── Pod (nginx-deployment-77f9859f7b-2)
          └── Pod (nginx-deployment-77f9859f7b-3)
```

---

## 3. Anatomy of a Deployment Manifest (`YAML`)

Save the following specification as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
  labels:
    app: web-app
spec:
  # Desired number of Pod copies
  replicas: 3

  # Selector links the Deployment to its Pods (MUST match template.metadata.labels)
  selector:
    matchLabels:
      app: web-app

  # Pod Template: The blueprint used to create Pod instances
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web-container
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

---

## 4. Field-by-Field Reference

| Field | Description |
| :--- | :--- |
| `apiVersion: apps/v1` | The API group and version for workload controllers like Deployments and DaemonSets. |
| `spec.replicas` | Total number of identical Pods to run concurrently. |
| `spec.selector.matchLabels` | Defines which Pods belong to this Deployment. Must match `template.metadata.labels`. |
| `spec.template` | The embedded Pod specification. Contains metadata (labels) and container configurations. |
| `resources.requests` | Guaranteed minimum compute resources allocated to each container. |
| `resources.limits` | Maximum upper limit of CPU and RAM allowed before throttling or termination. |

---

## 5. Deployment Lifecycle & Operational Commands

| Action | Command | Purpose |
| :--- | :--- | :--- |
| **Apply / Create** | `kubectl apply -f deployment.yaml` | Create or update a Deployment declaratively. |
| **List Deployments** | `kubectl get deployments` *(alias: `kubectl get deploy`)* | Check desired vs. current vs. available replicas. |
| **View ReplicaSets** | `kubectl get rs` | Inspect the active and historical ReplicaSets. |
| **Scale Replicas** | `kubectl scale deployment web-app-deployment --replicas=5` | Adjust Pod count imperatively. |
| **Update Image** | `kubectl set image deployment/web-app-deployment web-container=nginx:1.26-alpine` | Trigger a rolling update to a new image version. |
| **Track Rollout** | `kubectl rollout status deployment/web-app-deployment` | Monitor real-time status of a rolling upgrade. |
| **Check History** | `kubectl rollout history deployment/web-app-deployment` | View past revisions of the Deployment. |
| **Rollback Revision** | `kubectl rollout undo deployment/web-app-deployment` | Roll back to the immediately preceding revision. |
| **Rollback to Specific Rev** | `kubectl rollout undo deployment/web-app-deployment --to-revision=2` | Revert to a specific revision from history. |
| **Restart Deployment** | `kubectl rollout restart deployment/web-app-deployment` | Gracefully restart all pods one-by-one. |
| **Delete Deployment** | `kubectl delete deployment web-app-deployment` | Tear down the Deployment, ReplicaSet, and all Pods. |