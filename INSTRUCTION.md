# TodoApp Kubernetes Deployment

## Overview

This project deploys the TodoApp application to Kubernetes using:

- Kubernetes Deployment
- RollingUpdate deployment strategy
- Horizontal Pod Autoscaler (HPA)
- CPU and memory resource requests and limits

All application resources are deployed to the `mateapp` namespace.

---

## 1. Prerequisites

Before deploying the application, make sure that:

- A Kubernetes cluster is running.
- `kubectl` is installed and configured.
- The `mateapp` namespace exists.

Check the Kubernetes connection:

```bash
kubectl cluster-info
```

Check the namespace:

```bash
kubectl get namespace mateapp
```

If the namespace does not exist, create it:

```bash
kubectl create namespace mateapp
```

---

## 2. Application Deployment

Apply the Deployment manifest:

```bash
kubectl apply -f deployment.yml
```

Apply the Horizontal Pod Autoscaler:

```bash
kubectl apply -f hpa.yml
```

Check the Deployment:

```bash
kubectl get deployment -n mateapp
```

Check the Pods:

```bash
kubectl get pods -n mateapp
```

Check the HPA:

```bash
kubectl get hpa -n mateapp
```

A successful initial deployment should have two running Pods because the Deployment starts with two replicas.

---

## 3. Deployment Configuration

The Deployment is configured with:

```yaml
replicas: 2
```

Two replicas are used as the initial number of Pods because the application should remain available even if one Pod becomes unavailable.

The HPA also uses two as its minimum number of replicas, so autoscaling will never reduce the application below two Pods.

---

## 4. RollingUpdate Strategy

The Deployment uses the following strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

### Why RollingUpdate?

`RollingUpdate` allows a new version of the application to be deployed gradually instead of stopping all existing Pods at once.

This helps keep the application available during updates.

### Why `maxUnavailable: 0`?

```yaml
maxUnavailable: 0
```

This means that Kubernetes should not intentionally make an existing Pod unavailable during the update.

Since the application starts with two replicas, keeping the existing replicas available provides better availability during a rollout.

### Why `maxSurge: 1`?

```yaml
maxSurge: 1
```

This allows Kubernetes to temporarily create one additional Pod during an update.

For example:

```text
Before update:
2 old Pods

During update:
2 old Pods + 1 new Pod

After update:
2 new Pods
```

This provides a reasonable balance between application availability and additional resource consumption.

---

## 5. Resource Requests and Limits

The application container uses:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### CPU request

```yaml
cpu: "100m"
```

The application requests 0.1 CPU core.

This is appropriate for a lightweight Todo application and gives Kubernetes a baseline amount of CPU to reserve for the container.

### CPU limit

```yaml
cpu: "500m"
```

The container can use up to 0.5 CPU core when additional CPU capacity is available.

This prevents a single Pod from consuming an excessive amount of CPU.

### Memory request

```yaml
memory: "128Mi"
```

The application requests 128 MiB of memory.

This provides a reasonable baseline for a lightweight application.

### Memory limit

```yaml
memory: "256Mi"
```

The container is limited to 256 MiB of memory.

This prevents uncontrolled memory consumption by an individual Pod.

### Why requests and limits are important

Resource requests help Kubernetes schedule Pods on nodes with enough available resources.

Resource limits prevent a container from consuming more than the configured amount of CPU and memory.

The CPU and memory requests are also important for HPA because utilization percentages are calculated relative to the configured resource requests.

---

## 6. Horizontal Pod Autoscaler

The application uses a Horizontal Pod Autoscaler with:

```yaml
minReplicas: 2
maxReplicas: 5
```

The HPA can therefore maintain between 2 and 5 Pods.

### Why minimum 2?

Two Pods provide basic redundancy and availability.

If one Pod becomes unavailable, another Pod can continue serving the application.

### Why maximum 5?

Five Pods provide enough room for the application to handle increased load while preventing unlimited scaling and uncontrolled resource consumption.

---

## 7. HPA Metrics

The HPA uses both CPU and memory metrics:

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### CPU target

The CPU target is:

```yaml
averageUtilization: 70
```

This means the HPA considers average CPU utilization relative to the CPU requests.

For example, with a CPU request of `100m`, sustained utilization around 70% corresponds to approximately `70m` of CPU per Pod.

### Memory target

The memory target is:

```yaml
averageUtilization: 80
```

This means the HPA monitors memory usage relative to the configured memory requests.

Using both metrics allows the application to react to different types of workload.

For example:

```text
CPU-intensive workload
        ↓
CPU utilization increases
        ↓
HPA can increase the number of Pods
```

or:

```text
Memory-intensive workload
        ↓
Memory utilization increases
        ↓
HPA can increase the number of Pods
```

The HPA keeps the number of replicas within the configured range of 2 to 5.

---

## 8. Metrics Server Requirement

The Kubernetes cluster must provide CPU and memory metrics for HPA to work correctly.

Check whether resource metrics are available:

```bash
kubectl top pods -n mateapp
```

If this command returns CPU and memory usage, the metrics are available.

If it reports that metrics are unavailable, the cluster needs a working Metrics Server before HPA can make scaling decisions.

Check HPA status with:

```bash
kubectl get hpa -n mateapp
```

For detailed information:

```bash
kubectl describe hpa todoapp-hpa -n mateapp
```

---

## 9. Accessing the Application

The Deployment itself does not expose the application outside the Kubernetes cluster.

For local testing, the application can be accessed using `kubectl port-forward`:

```bash
kubectl port-forward deployment/todoapp 8080:8080 -n mateapp
```

Then open the application in a browser:

```text
http://localhost:8080
```

The connection works as follows:

```text
Browser
   |
   | localhost:8080
   v
kubectl port-forward
   |
   v
TodoApp Pod
   |
   | port 8080
   v
TodoApp
```

The `port-forward` command must remain running while the application is being accessed.

Press `Ctrl+C` to stop port forwarding.

### Important

`kubectl port-forward` is intended for local access and testing. It does not create a Kubernetes Service and does not provide permanent external access.

---

## 10. Verification Commands

Get all resources in the namespace:

```bash
kubectl get all -n mateapp
```

Get Pods:

```bash
kubectl get pods -n mateapp
```

Get the Deployment:

```bash
kubectl get deployment todoapp -n mateapp
```

Get the HPA:

```bash
kubectl get hpa todoapp-hpa -n mateapp
```

Check resource usage:

```bash
kubectl top pods -n mateapp
```

Describe the Deployment:

```bash
kubectl describe deployment todoapp -n mateapp
```

Describe the HPA:

```bash
kubectl describe hpa todoapp-hpa -n mateapp
```

View Pod logs:

```bash
kubectl logs <pod-name> -n mateapp
```

---

## 11. Updating the Application

After changing the Deployment manifest, apply it again:

```bash
kubectl apply -f deployment.yml
```

Check the rollout:

```bash
kubectl rollout status deployment/todoapp -n mateapp
```

Check the Pods during the rollout:

```bash
kubectl get pods -n mateapp
```

View rollout history:

```bash
kubectl rollout history deployment/todoapp -n mateapp
```

If necessary, a previous version can be restored with:

```bash
kubectl rollout undo deployment/todoapp -n mateapp
```

---

## 12. Removing the Application

Delete the HPA:

```bash
kubectl delete -f hpa.yml
```

Delete the Deployment:

```bash
kubectl delete -f deployment.yml
```

The namespace can be removed separately if it is no longer required:

```bash
kubectl delete namespace mateapp
```

---

## 13. Expected Final State

After deployment, the main resources should be:

```text
Namespace: mateapp

Deployment:
  todoapp
  replicas: 2

HPA:
  todoapp-hpa
  minReplicas: 2
  maxReplicas: 5
  CPU target: 70%
  Memory target: 80%
```

The application can be tested locally with:

```bash
kubectl port-forward deployment/todoapp 8080:8080 -n mateapp
```

and accessed at:

```text
http://localhost:8080
```
