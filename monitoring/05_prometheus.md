# Prometheus

Prometheus is a powerful open-source monitoring and alerting system designed for reliability and scalability. In Kubernetes environments, Prometheus has become the de facto standard for metrics collection and monitoring. This guide covers everything from basic Prometheus setup to advanced configurations in Kubernetes.

## Introduction to Prometheus

Prometheus is an open-source systems monitoring and alerting toolkit that was originally built at SoundCloud. It's now a graduated project in the Cloud Native Computing Foundation (CNCF), alongside Kubernetes. Prometheus's key strengths include:

- Multi-dimensional data model with time series data identified by metric name and key-value pairs
- Flexible query language (PromQL) for slicing and dicing data
- No reliance on distributed storage; single server nodes are autonomous
- Time series collection via a pull model over HTTP
- Push is supported via an intermediary gateway
- Service discovery or static configuration to identify targets
- Multiple modes of graphing and dashboarding support

### Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                            Prometheus Server                            │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │              │  │              │  │              │  │              │ │
│  │ Retrieval    │  │ Storage      │  │  PromQL     │  │  HTTP Server │ │
│  │ (Scraping)   │  │ (TSDB)       │  │  Engine     │  │              │ │
│  │              │  │              │  │              │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                                         │
└────────────────────────────────▲────────────────────────────────────────┘
                                 │
                                 │ Scrape metrics
                                 │
┌────────────────────────────────┼────────────────────────────────────────┐
│                                │                                        │
│                                │                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │              │  │              │  │              │  │              │ │
│  │ Kubernetes   │  │ Node         │  │ Exporters    │  │ Service      │ │
│  │ API Server   │  │ (kubelet)    │  │ (e.g. node)  │  │ Pods         │ │
│  │              │  │              │  │              │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                                         │
│                        Kubernetes Cluster                               │
└─────────────────────────────────────────────────────────────────────────┘

```

## Deploying Prometheus in Kubernetes

### Using Prometheus Operator

Prometheus Operator is the recommended way to deploy Prometheus in Kubernetes. It creates, configures, and manages Prometheus servers and related monitoring components.

#### Installing with Helm

```bash
# Add the Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack (includes Prometheus, Alertmanager, Grafana, and exporters)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
```

#### Verifying the Installation

```bash
# Check if Prometheus pods are running
kubectl get pods -n monitoring

# List the services created
kubectl get svc -n monitoring

# Access Prometheus UI (port-forward)
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Now you can access Prometheus at http://localhost:9090
```

### Manual Deployment (without Operator)

If you prefer a more direct approach without the operator, you can deploy Prometheus using regular Kubernetes resources:

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

Create the Prometheus deployment and service:

```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.36.0
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention.time=15d"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
```

Create the necessary RBAC resources:

```yaml
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

Apply these manifest files:

```bash
kubectl apply -f prometheus-config.yaml
kubectl apply -f prometheus-rbac.yaml
kubectl apply -f prometheus-deployment.yaml
```

## Prometheus Configuration

### Core Concepts

#### Metrics and Labels

Prometheus collects metrics, which are time-series data points identified by:
- A metric name (e.g., `http_requests_total`)
- Labels, which are key-value pairs (e.g., `{method="GET", endpoint="/api/v1/users"}`)

For example: 
```
http_requests_total{method="GET", endpoint="/api/v1/users", status="200"} 934
```

#### Data Types

Prometheus supports four data types:

1. **Counter**: Cumulative metric that only increases (e.g., number of requests)
2. **Gauge**: Metric that can go up and down (e.g., memory usage)
3. **Histogram**: Samples observations and counts them in configurable buckets (e.g., request durations)
4. **Summary**: Similar to histogram but also provides percentiles

### Scrape Configuration

Prometheus works by "scraping" HTTP endpoints that expose metrics. The scrape configuration defines the targets to be scraped.

#### Static Targets

```yaml
scrape_configs:
  - job_name: 'example-job'
    static_configs:
      - targets: ['example.com:8080', 'example.org:8080']
```

#### Service Discovery

Kubernetes service discovery allows Prometheus to automatically find and scrape targets:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

#### Relabeling

Relabeling allows transforming the metadata from service discovery:

```yaml
relabel_configs:
  # Keep only pods with the prometheus.io/scrape annotation set to true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  
  # Use the pod's custom metrics path if defined
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
    
  # Extract the pod namespace and name as labels
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: pod
```

### Alerting Configuration

Prometheus can be configured to trigger alerts based on metric conditions:

```yaml
# alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alert-rules
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
  - name: example-alerts
    rules:
    - alert: HighRequestLatency
      expr: http_request_duration_seconds{quantile="0.95"} > 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High request latency on {{ $labels.instance }}"
        description: "Request latency for {{ $labels.job }} is above 1s (current value: {{ $value }}s)"
        
    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.01
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate is above 1% (current value: {{ $value | humanizePercentage }})"
```

Apply this with:

```bash
kubectl apply -f alert-rules.yaml
```

### Custom Scrape Configurations

#### Example: Adding Redis Monitoring

1. Deploy Redis exporter:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:v1.43.0
        env:
        - name: REDIS_ADDR
          value: "redis:6379"
        ports:
        - containerPort: 9121
---
apiVersion: v1
kind: Service
metadata:
  name: redis-exporter
  namespace: monitoring
  labels:
    app: redis-exporter
spec:
  selector:
    app: redis-exporter
  ports:
  - port: 9121
    targetPort: 9121
```

2. Create a ServiceMonitor to scrape the Redis exporter:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-exporter
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: redis-exporter
  endpoints:
  - port: 9121
    interval: 30s
```

## PromQL: Prometheus Query Language

PromQL is the powerful query language used to select and aggregate time series data in Prometheus.

### Basic Queries

```
# Simple metric selection
http_requests_total

# Filtering by label
http_requests_total{method="GET"}

# Negation
http_requests_total{method!="GET"}

# Multiple filters (AND)
http_requests_total{method="GET", status="200"}

# Regular expressions
http_requests_total{path=~"/api/v1/.*"}
http_requests_total{path!~"/api/v1/.*"}
```

### Range Vectors and Functions

```
# Range vector: last 5 minutes of data
http_requests_total[5m]

# Rate: calculate per-second rate over a time window
rate(http_requests_total[5m])

# irate: instant rate for volatile metrics
irate(http_requests_total[5m])

# Increase: total increase over a time window
increase(http_requests_total[1h])
```

### Aggregation Operators

```
# Sum of requests across all instances
sum(http_requests_total)

# Average request duration by path
avg by(path) (http_request_duration_seconds)

# 95th percentile response time
histogram_quantile(0.95, sum(rate(http_request_duration_bucket[5m])) by (le, path))

# Count number of instances
count(up{job="kubernetes-nodes"})

# Max memory usage across all pods
max by(pod) (container_memory_usage_bytes)
```

### Advanced Examples

```
# Error ratio calculation
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# CPU usage by namespace
sum by(namespace) (rate(container_cpu_usage_seconds_total[5m]))

# Top 5 memory-consuming pods
topk(5, sum by(pod) (container_memory_usage_bytes))

# Node disk utilization
100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)
```

## Prometheus Exporters

Exporters are agents that convert metrics from various systems and applications into Prometheus format.

### Common Exporters

1. **Node Exporter**: System metrics (CPU, memory, disk, network)
   ```bash
   helm install node-exporter prometheus-community/prometheus-node-exporter \
     --namespace monitoring
   ```

2. **Blackbox Exporter**: Probes endpoints over HTTP, HTTPS, DNS, TCP, and ICMP
   ```bash
   helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
     --namespace monitoring
   ```

3. **MySQL Exporter**: MySQL database metrics
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mysql-exporter
     namespace: monitoring
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mysql-exporter
     template:
       metadata:
         labels:
           app: mysql-exporter
       spec:
         containers:
         - name: mysql-exporter
           image: prom/mysqld-exporter:v0.14.0
           ports:
           - containerPort: 9104
           env:
           - name: DATA_SOURCE_NAME
             valueFrom:
               secretKeyRef:
                 name: mysql-credentials
                 key: connection-string
   ```

4. **Elasticsearch Exporter**: Elasticsearch metrics
   ```bash
   helm install elasticsearch-exporter prometheus-community/prometheus-elasticsearch-exporter \
     --namespace monitoring \
     --set es.uri=http://elasticsearch:9200
   ```

### Writing Custom Exporters

Create custom exporters when needed for applications without built-in Prometheus support:

```python
# Example Python exporter using prometheus_client library
from prometheus_client import start_http_server, Counter, Gauge
import random
import time

# Create metrics
REQUEST_COUNT = Counter('app_requests_total', 'Total app requests', ['endpoint'])
RANDOM_VALUE = Gauge('app_random_value', 'Random value example')

# Start server
if __name__ == '__main__':
    # Start up the server to expose metrics
    start_http_server(8000)
    # Generate some metrics
    while True:
        REQUEST_COUNT.labels(endpoint='/api/v1/users').inc()
        RANDOM_VALUE.set(random.random() * 100)
        time.sleep(1)
```

### Using the Prometheus Operator Pattern

With the operator, you can create ServiceMonitor resources to define scrape configurations:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-application
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
```

## Recording Rules and Alerts

### Recording Rules

Recording rules allow you to precompute frequently needed or computationally expensive expressions:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  namespace: monitoring
spec:
  groups:
  - name: cpu-usage
    interval: 30s
    rules:
    - record: instance:node_cpu:avg_usage
      expr: avg by(instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))
    
    - record: namespace:container_cpu:usage_seconds_rate5m
      expr: sum by(namespace) (rate(container_cpu_usage_seconds_total[5m]))
```

### Alert Rules

Alert rules define conditions that trigger alerts:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-apps
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
  - name: kubernetes-apps
    rules:
    - alert: KubePodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total{job="kube-state-metrics"}[15m]) * 60 * 5 > 0
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting {{ printf \"%.2f\" $value }} times / 5 minutes."
        
    - alert: KubePodNotReady
      expr: sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown"}) > 0
      for: 1h
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready state for more than 1 hour."
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready state for more than 1 hour."
```

### Alertmanager Configuration

Alertmanager handles alerts sent by Prometheus:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  configSecret: alertmanager-config
---
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXX'
    
    route:
      group_by: ['alertname', 'job', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
      - match:
          severity: warning
        receiver: 'slack-notifications'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.message" . }}'
    
    - name: 'pagerduty-critical'
      pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        send_resolved: true
```

## Federation and Long-Term Storage

### Prometheus Federation

Federation allows a Prometheus server to scrape selected time series from another Prometheus server:

```yaml
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="kubernetes-apiservers"}'
        - '{job="kubernetes-nodes"}'
        - '{job="kubernetes-pods"}'
    static_configs:
      - targets:
        - 'prometheus-1:9090'
        - 'prometheus-2:9090'
```

### Remote Storage Integration

For long-term storage, Prometheus can integrate with remote storage solutions:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  serviceAccountName: prometheus
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  remoteWrite:
  - url: "http://thanos-receive:19291/api/v1/receive"
  - url: "http://victoria-metrics:8428/api/v1/write"
  remoteRead:
  - url: "http://thanos-query:9090/api/v1/read"
    readRecent: true
```

### Thanos Integration

Thanos is a set of components that adds high availability, long-term storage, and global query features to Prometheus:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  serviceAccountName: prometheus
  thanos:
    image: quay.io/thanos/thanos:v0.24.0
    version: v0.24.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

Thanos configuration secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: monitoring
type: Opaque
stringData:
  thanos.yaml: |
    type: S3
    config:
      bucket: thanos
      endpoint: minio.monitoring.svc:9000
      access_key: minio
      secret_key: minio123
      insecure: true
```

## Using Prometheus in Production

### Resource Requirements

Properly size Prometheus resources based on metrics volume and retention period:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
    limits:
      memory: 4Gi
      cpu: 1000m
  retention: 15d
  retentionSize: 10GB
```

### High Availability Setup

For high availability, deploy multiple Prometheus instances:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  serviceAccountName: prometheus
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: prometheus
            operator: In
            values:
            - prometheus
        topologyKey: kubernetes.io/hostname
```

### Security Considerations

Secure your Prometheus installation:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus
  web:
    tlsConfig:
      cert:
        secret:
          name: prometheus-tls
          key: tls.crt
      keySecret:
        name: prometheus-tls
        key: tls.key
```

### Performance Tuning

Optimize Prometheus performance:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  resources:
    requests:
      memory: 4Gi
      cpu: 1
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
  walCompression: true
  enableAdminAPI: false
  evaluationInterval: 30s
  scrapeInterval: 30s
  retention: 7d
```

## Advanced Use Cases

### Monitoring Custom Applications

1. Add Prometheus metrics to your application:

```go
// Go example using the Prometheus client library
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total number of HTTP requests",
    }, []string{"method", "endpoint", "status"})
    
    httpRequestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request latencies in seconds",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "endpoint"})
)

func main() {
    // Instrument the handlers
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(r.Method, "/"))
        defer timer.ObserveDuration()
        
        // Your handler logic here
        time.Sleep(time.Duration(100) * time.Millisecond)
        
        w.WriteHeader(http.StatusOK)
        httpRequestsTotal.WithLabelValues(r.Method, "/", "200").Inc()
    })
    
    // Expose the Prometheus metrics
    http.Handle("/metrics", promhttp.Handler())
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

2. Create a ServiceMonitor for your application:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: web
    interval: 15s
    path: /metrics
```

### Monitoring External Systems

Set up the Blackbox Exporter to probe external systems:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus-blackbox-exporter
  endpoints:
  - port: http
    interval: 30s
    scrapeTimeout: 10s
    path: /probe
    params:
      module: [http_2xx]
      target:
      - https://example.com
      - https://api.example.com/health
    relabelings:
    - sourceLabels: [__param_target]
      targetLabel: instance
    - sourceLabels: [__param_target]
      targetLabel: __param_target
    - targetLabel: __address__
      replacement: prometheus-blackbox-exporter:9115
```

### Multi-Cluster Monitoring

Set up a global Prometheus cluster to monitor multiple Kubernetes clusters:

1. Thanos sidecar in each cluster:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    prometheus: k8s
spec:
  replicas: 2
  serviceAccountName: prometheus
  thanos:
    image: quay.io/thanos/thanos:v0.24.0
    version: v0.24.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

2. Thanos components in the global cluster:

```yaml
# Thanos Query
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:v0.24.0
        args:
        - query
        - --log.level=info
        - --query.replica-label=replica
        - --query.auto-downsampling
        - --store=thanos-store:10901
        - --store=dnssrv+_grpc._tcp.thanos-sidecar.monitoring.svc.cluster.local
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
```

## Troubleshooting Prometheus

### Common Issues and Solutions

1. **High memory usage**:
   - Reduce retention period
   - Increase memory limits
   - Use recording rules to reduce query load
   - Implement federation or remote storage

2. **Slow queries**:
   - Use recording rules for complex queries
   - Optimize label cardinality
   - Review service discovery configurations

3. **Missed scrapes**:
   - Increase scrape timeout
   - Check for network issues
   - Review target load and resource usage

4. **Alertmanager not receiving alerts**:
   - Check alertmanager configuration
   - Verify Prometheus can reach Alertmanager
   - Check Alert rules configuration

### Debugging Tools

```bash
# Check Prometheus logs
kubectl logs -n monitoring prometheus-prometheus-0

# Check targets and their health
kubectl port-forward -n monitoring prometheus-prometheus-0 9090
# Then visit http://localhost:9090/targets

# Check alertmanager connections
# Visit http://localhost:9090/config

# Check ServiceMonitor configuration
kubectl get servicemonitors -A -o yaml

# Check if metrics endpoints are accessible
kubectl exec -n monitoring prometheus-prometheus-0 -- curl -s http://SERVICE:PORT/metrics | head
```

## Best Practices

1. **Cardinality Control**:
   - Limit the number of labels used with high-cardinality values
   - Avoid using user IDs, emails, or unbounded values as labels
   - Use recording rules to aggregate high-cardinality metrics

2. **Resource Planning**:
   - Size Prometheus based on metrics volume and retention
   - Consider the ~1-2 bytes per sample rule of thumb
   - Monitor Prometheus itself with a separate instance

3. **Query Optimization**:
   - Use recording rules for frequently used and complex queries
   - Limit the time range in queries
   - Prefer rate() over increase() for counters

4. **Reliability**:
   - Use multiple Prometheus replicas
   - Implement remote storage for long-term data
   - Set up cross-cluster federation

5. **Security**:
   - Restrict access to Prometheus UI
   - Use TLS for all connections
   - Apply RBAC controls strictly

6. **Alert Design**:
   - Alert on symptoms, not causes
   - Set appropriate thresholds to avoid alert fatigue
   - Include clear instructions in alert annotations

## Conclusion

Prometheus has become the de facto standard for monitoring Kubernetes and cloud-native applications due to its reliability, scalability, and integration with the Kubernetes ecosystem. By understanding its key components and properly configuring them, you can build a comprehensive monitoring solution for your applications and infrastructure.

As your Kubernetes environment grows, consider enhancing Prometheus with components like Thanos or using managed solutions to further scale your monitoring capabilities while focusing on deriving value from the metrics you collect.