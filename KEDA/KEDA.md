# KEDA-Based Autoscaling with Prometheus

## 📌 Overview

KEDA (Kubernetes Event-Driven Autoscaler) is a lightweight component that enables event-driven autoscaling in Kubernetes. It allows you to scale applications based on external metrics such as Prometheus, Kafka, RabbitMQ, and more.

In this setup, we use Prometheus metrics to scale an NGINX deployment using KEDA.

---

## 🚀 Install KEDA

You can install KEDA using Helm:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda --namespace keda --create-namespace
```

Verify installation:

```bash
kubectl get all -n keda
```

---

## 🧱 Step 1: Create NGINX Deployment with Metrics Exporter

We deploy an NGINX application along with a sidecar container that exposes metrics in Prometheus format.

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  labels:
    app: nginx
data:
  nginx.conf: |
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /nginx_status {
            stub_status;
            allow all;
        }
    }
```

---

### Deployment + Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
        resources:
          limits:
            cpu: "500m"
            memory: "256Mi"
          requests:
            cpu: "250m"
            memory: "128Mi"

      - name: nginx-prometheus-exporter
        image: nginx/nginx-prometheus-exporter:latest
        args:
        - "-nginx.scrape-uri=http://localhost/nginx_status"
        ports:
        - containerPort: 9113

      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http
  - protocol: TCP
    port: 9113
    targetPort: 9113
    name: metrics
  type: ClusterIP
```

---

## 📊 Step 2: Configure ServiceMonitor

A ServiceMonitor allows Prometheus to discover and scrape metrics from the NGINX pods.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-servicemonitor
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: nginx
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
  namespaceSelector:
    matchNames:
    - default
```

---

## ⚡ Step 3: Create KEDA ScaledObject

The ScaledObject is the core KEDA resource that defines how your application should scale.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-scaledobject
spec:
  scaleTargetRef:
    name: nginx

  minReplicaCount: 1
  maxReplicaCount: 10

  pollingInterval: 15
  cooldownPeriod: 30

  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
      metricName: nginx_requests_per_second
      threshold: "80"
      query: |
        sum(rate(nginx_http_requests_total{namespace!="",pod!=""}[1m])) by (pod)
```

---

## 🔄 How It Works

1. NGINX exposes basic stats using `stub_status`.
2. The sidecar exporter converts these stats into Prometheus metrics.
3. Prometheus scrapes the metrics using the ServiceMonitor.
4. KEDA queries Prometheus at regular intervals.
5. Based on the defined threshold, KEDA scales the deployment.
6. KEDA automatically creates and manages an HPA (Horizontal Pod Autoscaler).

---

## ✅ Key Benefits of KEDA

* No need for Prometheus Adapter
* Simplified autoscaling configuration
* Supports multiple event sources
* Automatically manages HPA

---

## 🎯 Conclusion

KEDA simplifies autoscaling by directly integrating with Prometheus and other event sources. By using ScaledObjects, you can define flexible and powerful scaling rules without the complexity of custom metrics adapters.

This approach is more efficient, easier to manage, and closer to real-world production use cases compared to traditional HPA setups.
