# 🚀 Kubernetes HPA with Custom Metrics using Prometheus

This guide walks through setting up **Horizontal Pod Autoscaler (HPA)** using **custom metrics** from Prometheus.

---

## 📌 Step 1: Install Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

---

## 📌 Step 2: Deploy NGINX with Metrics

### 🔹 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
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

### 🔹 Deployment + Service

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
        - "--nginx.scrape-uri=http://localhost/nginx_status"
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
  - name: http
    port: 80
    targetPort: 80
  - name: metrics
    port: 9113
    targetPort: 9113
  type: ClusterIP
```

---

## 📌 Step 3: Verify Metrics

```bash
kubectl port-forward svc/nginx-service 9113:9113
```

Open:

```
http://localhost:9113/metrics
```

You should see:

```
nginx_up 1
```

---

## 📌 Step 4: Create ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-service-monitor
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

## 📌 Step 5: Validate in Prometheus

* Go to **Prometheus UI**
* Check **Targets**
* Query:

```
nginx_http_requests_total
```

---

## 📌 Step 6: Install Prometheus Adapter

```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring
```

---

## 📌 Step 7: Configure Adapter Rules

Edit config:

```bash
kubectl edit configmap -n monitoring prometheus-adapter
```

Add:

```yaml
rules:
  custom:
  - seriesQuery: 'nginx_http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    name:
      matches: "nginx_http_requests_total"
      as: "nginx_requests_per_second"
    metricsQuery: 'sum(rate(nginx_http_requests_total{<<.LabelMatchers>>}[1m])) by (namespace,pod)'
```

Restart adapter:

```bash
kubectl rollout restart deployment prometheus-adapter -n monitoring
```

---

## 📌 Step 8: Verify Custom Metrics API

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq
```

---

## 📌 Step 9: Create HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: nginx_requests_per_second
      target:
        type: AverageValue
        averageValue: 10
```

---

## 📌 Step 10: Monitor Scaling

```bash
kubectl get hpa -w
```

---

## 📌 Step 11: Generate Load

```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- \
/bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx-service; done"
```

---

## 🎯 Summary

✔ Prometheus scrapes metrics via ServiceMonitor
✔ Exporter converts `/nginx_status` → `/metrics`
✔ Adapter exposes metrics to Kubernetes API
✔ HPA scales based on custom metric

---

## ⚠️ Common Issues

* ServiceMonitor label mismatch
* Wrong Prometheus service URL in adapter
* Metrics not visible in Prometheus
* Adapter not restarted after config change

---

## 🚀 Outcome

You now have:

```
NGINX → Exporter → Prometheus → Adapter → HPA
```

Fully working **custom metrics autoscaling pipeline** 🔥
