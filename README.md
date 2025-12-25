# Kubernetes Horizontal Pod Autoscaler (HPA) Setup Guide

A comprehensive guide to implementing Horizontal Pod Autoscaler in Kubernetes with Metrics Server, resource management, and hands-on examples.

## 📋 Table of Contents

- [Overview](#overview)
- [Key Concepts](#key-concepts)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Installation & Setup](#installation--setup)
- [Resource Management](#resource-management)
- [Understanding Scaling](#understanding-scaling)
- [Testing & Verification](#testing--verification)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## 🎯 Overview

This repository demonstrates how to configure and deploy Horizontal Pod Autoscaler (HPA) in Kubernetes. You'll learn to automatically scale your applications based on CPU utilization while understanding the underlying mechanisms that make it work.

### What You'll Learn

- How Horizontal Pod Autoscaler works
- Setting up and configuring Metrics Server
- Understanding cAdvisor's role in metrics collection
- Implementing CPU-based autoscaling
- Configuring resource requests and limits properly
- Distinguishing between pod scaling and cluster scaling

## 🧠 Key Concepts

### Horizontal Pod Autoscaler (HPA)

HPA automatically adjusts the number of pod replicas based on observed resource metrics:

- **CPU utilization** (most common)
- **Memory usage**
- **Custom metrics** (via custom metrics API)

HPA continuously monitors these metrics and scales pods up or down to maintain optimal performance and resource utilization.

### Metrics Server

The Metrics Server is a cluster-wide aggregator of resource usage data that:

- Collects CPU and memory metrics from all nodes and pods
- Provides the metrics API that HPA queries
- Enables `kubectl top` commands

**Without Metrics Server, HPA cannot function.**

```bash
# These commands require Metrics Server
kubectl top pods
kubectl top nodes
```

### cAdvisor (Container Advisor)

- Integrated into kubelet on every node
- Collects real-time container resource usage statistics
- Monitors CPU, memory, filesystem, and network metrics
- Serves as the data source for Metrics Server

**Metrics Flow:**

```
cAdvisor (on node) → Kubelet → Metrics Server → HPA Controller → Scale Decision
```

## ⚙️ Prerequisites

Before you begin, ensure you have:

- A running Kubernetes cluster (minikube, kind, Docker Desktop, or cloud provider)
- `kubectl` command-line tool installed and configured
- Basic understanding of Kubernetes pods, deployments, and services
- Cluster admin permissions (for Metrics Server installation)

## 📂 Repository Structure

```
.
├── deployment.yml      # Application deployment with resource configuration
├── service.yml         # Kubernetes service to expose the application
├── hpa.yml            # HPA configuration with scaling rules
├── stress-test.yml    # (Optional) CPU stress test pod
└── README.md          # This documentation
```

## 🚀 Installation & Setup

### Step 1: Install Metrics Server

Deploy the latest Metrics Server to your cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### For Local Development Clusters

If using **minikube**, **kind**, or **Docker Desktop**, you need to allow insecure TLS connections:

```bash
# Edit the Metrics Server deployment
kubectl edit deployment metrics-server -n kube-system
```

Add the `--kubelet-insecure-tls` flag under the container args:

```yaml
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls  # Add this line
```

Restart the deployment:

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

#### Verify Metrics Server Installation

Wait for the metrics server pod to be ready:

```bash
kubectl get deployment metrics-server -n kube-system
```

Test metrics collection:

```bash
# View node resource usage
kubectl top nodes

# View pod resource usage
kubectl top pods -A
```

If you see CPU and memory values, Metrics Server is working correctly.

### Step 2: Deploy Your Application

Apply the deployment and service manifests:

```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

Verify the deployment:

```bash
# Check if pods are running
kubectl get pods

# Check deployment status
kubectl get deployment

# Check service
kubectl get service
```

### Step 3: Configure Horizontal Pod Autoscaler

Apply the HPA configuration:

```bash
kubectl apply -f hpa.yml
```

Verify HPA is active:

```bash
kubectl get hpa
```

Expected output:

```
NAME      REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app-hpa   Deployment/myapp   15%/50%   2         10        2          30s
```

The `TARGETS` column shows: `current_cpu_utilization / target_cpu_utilization`

## 📐 Resource Management

### Understanding Requests vs Limits

Resource configuration is crucial for HPA to function correctly:

```yaml
resources:
  requests:
    cpu: "200m"        # 0.2 CPU cores
    memory: "256Mi"
  limits:
    cpu: "500m"        # 0.5 CPU cores maximum
    memory: "512Mi"
```

#### Requests

- **Guaranteed resources** allocated to the container
- Used by the Kubernetes scheduler for pod placement
- **HPA uses CPU requests as the baseline** for calculating utilization percentage
- Should match typical workload requirements

#### Limits

- **Maximum resources** a container can consume
- Container is throttled (CPU) or killed (memory) if limit is exceeded
- Protects cluster from resource exhaustion
- Should account for traffic spikes

### CPU Measurement Units

- `1000m` = 1 CPU core = 1000 millicores
- `500m` = 0.5 CPU core (half a core)
- `200m` = 0.2 CPU core (20% of one core)

### Memory Measurement Units

- `Ki` = Kibibytes (1024 bytes)
- `Mi` = Mebibytes (1024 Ki)
- `Gi` = Gibibytes (1024 Mi)

## 📊 Understanding Scaling

### HPA Scaling Formula

HPA calculates desired replicas using this formula:

```
Desired Replicas = ceil(Current Replicas × (Current Metric Value / Target Metric Value))
```

#### Example Calculation

Given:
- Current replicas: **2**
- Current CPU usage: **80%** of requested CPU
- Target CPU: **50%** of requested CPU

```
Desired Replicas = ceil(2 × (80 / 50))
                 = ceil(2 × 1.6)
                 = ceil(3.2)
                 = 4 replicas
```

HPA will scale up from 2 to 4 pods.

### Scaling Behavior

**Scale Up:**
- Triggered when metric exceeds target for a sustained period
- Default: evaluates every 15 seconds
- Adds pods quickly to handle load

**Scale Down:**
- More conservative to prevent thrashing
- Default: waits 5 minutes after scale-down before another scale-down
- Removes pods gradually

### HPA Configuration Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Target 50% CPU utilization
```

### Pod Scaling vs Cluster Scaling

| Aspect | Horizontal Pod Autoscaling (HPA) | Cluster Autoscaling |
|--------|----------------------------------|---------------------|
| **What it scales** | Number of pods | Number of nodes |
| **Managed by** | Kubernetes HPA controller | Cloud provider autoscaler |
| **Triggers** | Pod resource metrics | Unschedulable pods / low utilization |
| **Speed** | Fast (seconds to minutes) | Slower (minutes) |
| **Resource** | Uses existing cluster capacity | Adds/removes virtual machines |
| **Scope** | Application-level | Infrastructure-level |

**Important:** HPA does NOT add nodes to your cluster. If all nodes are full, pods will remain pending until:
- Cluster Autoscaler adds nodes (cloud environments)
- Manual node addition (on-premises)
- Other pods are terminated

## 🧪 Testing & Verification

### Monitor HPA Status

Watch HPA in real-time:

```bash
kubectl get hpa -w
```

### View Detailed HPA Information

```bash
kubectl describe hpa app-hpa
```

### Generate CPU Load (Manual Test)

Create a stress test pod:

```bash
kubectl run stress-test --image=busybox --restart=Never -- /bin/sh -c "while true; do :; done"
```

Or exec into an existing pod:

```bash
kubectl exec -it <pod-name> -- sh
# Inside the pod
while true; do :; done
```

Watch the scaling behavior:

```bash
# Terminal 1: Watch HPA
kubectl get hpa -w

# Terminal 2: Watch pods
kubectl get pods -w

# Terminal 3: Monitor resource usage
watch kubectl top pods
```

### Using Load Testing Tools

For more realistic testing, use a load generator:

```bash
# Example with Apache Bench (if service is exposed)
ab -n 10000 -c 100 http://<service-url>/
```

### Verify Metrics Collection

Check current resource usage:

```bash
# Node-level metrics
kubectl top nodes

# Pod-level metrics
kubectl top pods

# Specific namespace
kubectl top pods -n default
```

## 🔧 Troubleshooting

### HPA Shows "unknown" for Metrics

**Problem:** `kubectl get hpa` shows `<unknown>/50%`

**Solutions:**

1. Ensure resource requests are defined in deployment
2. Wait 1-2 minutes for metrics to be collected
3. Verify Metrics Server is running:
   ```bash
   kubectl get pods -n kube-system | grep metrics-server
   ```

### Metrics Server Not Working

**Problem:** `kubectl top nodes` returns an error

**Solutions:**

1. Check Metrics Server logs:
   ```bash
   kubectl logs -n kube-system deployment/metrics-server
   ```

2. Verify kubelet ports are accessible (10250)

3. For local clusters, ensure `--kubelet-insecure-tls` flag is set

### HPA Not Scaling

**Problem:** Load increases but HPA doesn't create new pods

**Possible causes:**

1. **Already at maxReplicas:** Check `kubectl get hpa`
2. **No available resources:** Check `kubectl describe nodes`
3. **Insufficient node capacity:** May need cluster autoscaling
4. **Metric below threshold:** Current usage might not exceed target

### Pods in Pending State

**Problem:** HPA creates pods but they remain pending

**Solution:** Check node resources:

```bash
kubectl describe nodes
kubectl get pods -o wide
```

You may need to add nodes or adjust resource requests.

## 📚 References

### Official Documentation

- [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server Repository](https://github.com/kubernetes-sigs/metrics-server)
- [Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

### Related Topics

- [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [Custom Metrics API](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics)

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📝 License

This project is open source and available under the [MIT License](LICENSE).

---

**Need Help?**

- Open an issue in this repository
- Check Kubernetes Slack: [#kubernetes-users](https://kubernetes.slack.com/)
- Stack Overflow: Tag `kubernetes` and `horizontal-pod-autoscaler`

---

Made with ❤️ for the Kubernetes community
