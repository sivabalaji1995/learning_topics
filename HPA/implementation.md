- Refer this blog for prometheus implementation

https://medium.com/@muppedaanvesh/a-hands-on-guide-to-kubernetes-monitoring-using-prometheus-grafana-%EF%B8%8F-b0e00b1ae039

┌─────────────────────────────────────────────────────────────────────────────┐
│                              COMPLETE WORKFLOW                               │
└─────────────────────────────────────────────────────────────────────────────┘

Step 1: INSTALL PROMETHEUS
┌─────────────────────────────────────────────────────────────────────────────┐
│ $ helm install prometheus prometheus-community/kube-prometheus-stack        │
│                                                                              │
│ Prometheus now runs in your cluster and can scrape metrics                   │
└─────────────────────────────────────────────────────────────────────────────┘

Step 2: APPLICATION EXPOSES METRICS
┌─────────────────────────────────────────────────────────────────────────────┐
│ Your application pods expose /metrics endpoint with http_requests_total     │
│                                                                              │
│ Pod-1: http://10.244.1.5:8080/metrics → http_requests_total 1000            │
│ Pod-2: http://10.244.2.8:8080/metrics → http_requests_total 2000            │
└─────────────────────────────────────────────────────────────────────────────┘

Step 3: PROMETHEUS SCRAPES DIRECTLY (No Adapter Needed!)
┌─────────────────────────────────────────────────────────────────────────────┐
│ Prometheus scrapes each pod every 15 seconds:                                │
│ GET http://10.244.1.5:8080/metrics → Stores http_requests_total             │
│ GET http://10.244.2.8:8080/metrics → Stores http_requests_total             │
│                                                                              │
│ Prometheus now has raw counter data                                          │
└─────────────────────────────────────────────────────────────────────────────┘

Step 4: INSTALL PROMETHEUS ADAPTER (For HPA, NOT for Prometheus!)
┌─────────────────────────────────────────────────────────────────────────────┐
│ $ helm install prometheus-adapter prometheus-community/prometheus-adapter   │
│                                                                              │
│ Adapter registers itself with Kubernetes API under custom.metrics.k8s.io    │
│ Adapter is configured with seriesQuery to calculate RPS                      │
└─────────────────────────────────────────────────────────────────────────────┘

Step 5: CONFIGURE ADAPTER WITH SERIES QUERY
┌─────────────────────────────────────────────────────────────────────────────┐
│ # prometheus-adapter-values.yaml                                             │
│ rules:                                                                       │
│   custom:                                                                    │
│   - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'               │
│     metricsQuery: |                                                          │
│       sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)      │
│     name:                                                                    │
│       as: "http_requests_per_second"                                         │
│                                                                              │
│ $ helm upgrade --install prometheus-adapter -f values.yaml                   │
└─────────────────────────────────────────────────────────────────────────────┘

Step 6: CREATE HPA (Uses Adapter-Exposed Metric)
┌─────────────────────────────────────────────────────────────────────────────┐
│ apiVersion: autoscaling/v2                                                   │
│ kind: HorizontalPodAutoscaler                                                │
│ spec:                                                                        │
│   metrics:                                                                   │
│   - type: Pods                                                               │
│     pods:                                                                    │
│       metric:                                                                │
│         name: http_requests_per_second  # ← From adapter                     │
│       target:                                                                │
│         averageValue: "500"                                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Step 7: HPA SCALING LOOP (What happens at runtime)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  HPA ──▶ Asks Kubernetes API: "What's the current RPS for my pods?"         │
│         GET /apis/custom.metrics.k8s.io/.../http_requests_per_second         │
│                                                                              │
│  API ───▶ Forwards request to Prometheus Adapter                             │
│                                                                              │
│  Adapter ─▶ Queries Prometheus:                                              │
│            sum(rate(http_requests_total{namespace="default"}[1m])) by (pod)  │
│                                                                              │
│  Prometheus ─▶ Calculates rate() and returns:                                │
│                {pod-1: 425 RPS, pod-2: 398 RPS}                              │
│                                                                              │
│  Adapter ─▶ Formats as Kubernetes metric and returns to API                 │
│                                                                              │
│  API ─────▶ Returns to HPA: "Average RPS = 411.5"                           │
│                                                                              │
│  HPA ─────▶ Calculates: 411.5 / 500 = 0.82 → No scaling needed              │
│                                                                              │
│  (If RPS increases to 750): 750 / 500 = 1.5 → Scale from 2 to 3 pods         │
└─────────────────────────────────────────────────────────────────────────────┘



## Setup Checklist

- [ ] 1. Install Prometheus using Helm (e.g., kube-prometheus-stack)
- [ ] 2. Ensure your application pods expose /metrics with http_requests_total counter
- [ ] 3. Verify Prometheus is scraping your pods (check Targets page)
- [ ] 4. Install Prometheus Adapter using Helm
- [ ] 5. Create values.yaml with seriesQuery to calculate RPS
- [ ] 6. Upgrade adapter with the values.yaml configuration
- [ ] 7. Verify custom metrics API is available (kubectl api-versions | grep custom.metrics)
- [ ] 8. Create HPA targeting your deployment with metric name http_requests_per_second
- [ ] 9. Monitor HPA with kubectl get hpa --watch