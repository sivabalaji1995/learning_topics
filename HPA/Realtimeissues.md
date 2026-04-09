1) The HPA min count will overwritten the replica count in the deployment manifest file


Real-World Production Challenges

Challenge 1: Metric Inaccuracy and Lag

The Problem:
Metrics Server provides 15-60 second old data, but your traffic changes in milliseconds. This lag causes undershoot (too few pods) or overshoot (too many).

Real Example:

text
12:00:00 - Traffic spikes to 10,000 RPS
12:00:05 - Metrics Server reports "we're at 5,000 RPS" (lagging)
12:00:05 - HPA calculates: need 5 pods (but actually need 10)
12:00:30 - Finally sees 10,000 RPS, now needs 10 pods
12:01:30 - New pods are running (90 seconds later)
12:02:00 - Traffic drops to 1,000 RPS (already missed the spike)


# 1. Reduce metric collection interval (modify metrics-server)
args:
- --metric-resolution=10s  # Default is 60s
- --kubelet-preferred-address-types=InternalIP


The args block in your YAML needs to be added to the Metrics Server deployment. You cannot do this by just applying a YAML file; you have to edit the live deployment.

Step-by-step guide:

Run the following command to open the Metrics Server deployment in your default text editor (usually vi or nano):

bash
kubectl -n kube-system edit deployment metrics-server
You will see the YAML definition. Locate the spec.template.spec.containers section.

Find the args section under the metrics-server container. Add or modify the following flags:

spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --metric-resolution=15s   # Default is 60s, makes metrics fresher
        - --kubelet-preferred-address-types=InternalIP # Improves reliability
        - --kubelet-insecure-tls   # Bypasses TLS issues (common in some setups)

# 2. Use predictive scaling with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: predictive-hpa
spec:
  metrics:
  - type: External
    external:
      metric:
        name: "queue_depth_forecast"  # Use predicted values
      target:
        type: AverageValue
        averageValue: "100"

Real fix: Implement hybrid scaling - keep a minimum buffer pool plus HPA.

# Buffer pool approach
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: buffer-pool-hpa
spec:
  minReplicas: 5  # Always keep 5 warm pods
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30  # Scale at lower threshold
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30  # Faster reaction