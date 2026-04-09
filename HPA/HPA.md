# 📈 Horizontal Pod Autoscaler (HPA) in Kubernetes

This document explains the **Horizontal Pod Autoscaler (HPA)** in Kubernetes, how it works, and how to set it up in a local cluster like Minikube.

---

## 🧠 What is HPA?

The **Horizontal Pod Autoscaler (HPA)** is a Kubernetes object that automatically scales the number of pods in a deployment based on observed metrics such as CPU or memory usage.

---

## 📊 How HPA Works

HPA relies on the **Metrics Server** to fetch resource usage data.

### 🔁 Metrics Flow

```
Container → cAdvisor → Kubelet → Metrics Server → HPA
```

* **cAdvisor** collects container-level metrics
* **Kubelet** exposes node and pod metrics
* **Metrics Server** aggregates these metrics across the cluster
* **HPA** uses these metrics to make scaling decisions

---

## 🔗 Metrics API

The Metrics Server exposes metrics via the Kubernetes API:

```
/apis/metrics.k8s.io/v1beta1/
```

---

## ☁️ Managed vs Local Clusters

* **Managed clusters (e.g., AKS)**
  Metrics Server is installed by default ✅

* **Local clusters (e.g., Minikube)**
  You must enable it manually ❗

---

## 🚀 Enable Metrics Server in Minikube

```bash
minikube addons enable metrics-server
```

⏳ It may take a few seconds to become ready.

---

## 🔍 Verify Metrics Server

### Check if the pod is running

```bash
kubectl get pods -n kube-system | grep metrics-server
```

---

### Test Metrics API directly

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq '.'
```

---

### Verify using `kubectl top`

```bash
kubectl top nodes
kubectl top pods
```

If these commands return metrics, the setup is successful ✅

---

# 🧱 Kubernetes Resources

Below are the required Kubernetes manifests:

---

## 📦 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-app
  template:
    metadata:
      labels:
        app: hpa-app
    spec:
      containers:
        - name: hpa-container
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

---

## 🌐 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-service
spec:
  selector:
    app: hpa-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

## 📈 Horizontal Pod Autoscaler (HPA)

This is a full-featured HPA configuration using `autoscaling/v2`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: default
  labels:
    app: my-app
    environment: production
  annotations:
    description: "HPA for my application"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app

  minReplicas: 1
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
      selectPolicy: Max

    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
```

---

## 📌 Notes

* `autoscaling/v2` provides advanced scaling behavior configuration
* `resources.requests` in the deployment is **mandatory** for HPA to work
* HPA will not function without Metrics Server

---

## 🔍 Check HPA Status

```bash
kubectl get hpa
```

---

## 📊 Example HPA Status Output

```yaml
status:
  observedGeneration: 1
  lastScaleTime: "2024-01-01T00:00:00Z"
  currentReplicas: 1
  desiredReplicas: 3
  currentMetrics:
    - type: Resource
      resource:
        name: cpu
        current:
          averageUtilization: 75
          averageValue: "750m"
```

---

## 🎯 Summary

* HPA automatically scales pods based on resource usage
* Metrics Server is required for HPA to function
* Minikube requires manual enablement of Metrics Server
* Proper resource requests must be defined in the deployment

---

## 🚀 Next Steps

* Generate load to test scaling
* Monitor scaling behavior using `kubectl get hpa -w`
* Integrate with Prometheus for advanced metrics

---
## 🔥 Load Testing (Trigger HPA Scaling)

To verify that the Horizontal Pod Autoscaler (HPA) is working correctly, you need to generate load on your application.

You can use a temporary BusyBox pod to continuously send requests to your service.

---

### 🚀 Run Load Generator

```bash
kubectl run -i --tty load-generator \
  --rm \
  --image=busybox:1.28 \
  --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://my-app-service; done"
```

---

### 🧠 What this does

* Creates a temporary pod named `load-generator`
* Sends continuous HTTP requests to your service (`my-app-service`)
* Generates CPU load on your application
* Automatically deletes the pod when stopped

---

### 📈 Observe Scaling

In another terminal, monitor HPA behavior:

```bash
kubectl get hpa -w
```

You should see something like:

```text
NAME          REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
my-app-hpa    Deployment/my-app      75%/50%   1         10        3
```

---

### 🔍 Monitor Pods

```bash
kubectl get pods
```

You will notice the number of pods increasing as load rises 🚀

---

### 🛑 Stop Load Test

Press:

```text
Ctrl + C
```

This will stop the load generator and delete the pod.

---

## 🎯 Expected Behavior

* CPU usage increases 📈
* HPA detects higher utilization
* New pods are created automatically
* When load decreases, pods scale down

---

```bash
kubectl get hpa --watch
```
