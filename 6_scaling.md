# Kubernetes Notes: Scaling (HPA, VPA, and Cluster Autoscaler)

---

## 1. What is Scaling in Kubernetes?

Scaling adjusts cluster capacity to handle traffic changes, workload demand, and resource consumption. It operates across two primary dimensions:
* **Workload Level (Pod Scaling):** Adjusting Pod replica counts or container resource allocations.
* **Infrastructure Level (Node Scaling):** Adding or removing physical/virtual worker machines.

---

## 2. The Three Dimensions of Scaling

| Type | Full Name | How It Scales | Best For |
| :--- | :--- | :--- | :--- |
| **HPA** | Horizontal Pod Autoscaler | Adds/removes **Pod replicas** (out/in) | Stateless web apps, APIs, variable user traffic |
| **VPA** | Vertical Pod Autoscaler | Increases/decreases **CPU & RAM limits** (up/down) | Databases, batch jobs, monolithic apps |
| **CA / Karpenter** | Cluster Autoscaler | Adds/removes **Nodes** (worker machines) | When Pods are `Pending` due to full cluster capacity |

---

## 3. Manual Scaling

You can manually modify the replica count of a Deployment using imperative commands or declarative manifests [cite: 3]:

* **Imperative CLI:**
  ```bash
  kubectl scale deployment web-app-deployment --replicas=5
  ```
* **Declarative YAML:** Update `replicas` inside `deployment.yaml` and apply via `kubectl apply -f deployment.yaml` [cite: 3].

---

## 4. Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of Pod replicas based on observed CPU utilization or other custom metrics.

* **HPA Formula:**
  $$	ext{Desired Replicas} = \left\lceil 	ext{Current Replicas} 	imes \left( rac{	ext{Current Metric Value}}{	ext{Target Metric Value}} 
ight) 
ight
ceil$$

* **Imperative Setup:**
  ```bash
  kubectl autoscale deployment web-app-deployment --min=2 --max=10 --cpu-percent=70
  ```

* **Declarative HPA Manifest (`hpa.yaml`):**
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: web-app-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: web-app-deployment
    minReplicas: 2
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
  ```

> **Prerequisite:** Containers must have `resources.requests` defined, and the cluster must run the Metrics Server.

---

## 5. Vertical Pod Autoscaler (VPA) & Cluster Autoscaler (CA)

* **Vertical Pod Autoscaler (VPA):** Adjusts CPU and memory requests/limits for existing Pods over time to prevent over- or under-provisioning.
* **Cluster Autoscaler (CA):** Monitors `Pending` pods caused by insufficient cluster node resources and provisions new virtual machines from the cloud provider.