# Building an Observability Platform for Kubernetes Microservices

## Introduction to Microservices Observability

Observability in a microservices architecture is the ability to understand the internal state of your system by examining its outputs. In a distributed system running on Kubernetes, observability extends beyond simple monitoring to provide deep insights into complex and often ephemeral systems. This project guide walks through building a comprehensive observability platform that integrates metrics, logs, and traces to create a holistic view of your microservices ecosystem.

## The Three Pillars of Observability

### Metrics
- Numerical data points collected over time
- Measure system and application performance
- Used for alerting, trend analysis, and capacity planning
- Examples: CPU usage, request count, error rates, latency

### Logs
- Time-stamped records of discrete events
- Provide detailed context for troubleshooting
- Capture specific actions and errors
- Examples: Application logs, system logs, audit logs

### Traces
- End-to-end journey of requests through distributed systems
- Visualize request flow across service boundaries
- Identify performance bottlenecks
- Examples: API call traces, database query traces, service dependencies

## Project Architecture Overview

### High-Level Components

```
Kubernetes Cluster
│
├── Metrics Collection and Storage
│   ├── Prometheus (Collection and Storage)
│   ├── Thanos (Long-term Storage)
│   └── Grafana (Visualization)
│
├── Log Management
│   ├── Fluent Bit (Collection)
│   ├── Elasticsearch (Storage)
│   └── Kibana (Visualization)
│
├── Distributed Tracing
│   ├── OpenTelemetry (Instrumentation)
│   ├── Jaeger (Processing and Storage)
│   └── Jaeger UI (Visualization)
│
└── Unified Dashboard
    └── Grafana (Cross-data Integration)
```

### Key Technology Choices

- **Metrics Stack**: Prometheus, Thanos, Grafana
- **Logging Stack**: Fluent Bit, Elasticsearch, Kibana
- **Tracing Stack**: OpenTelemetry, Jaeger
- **Service Mesh**: Istio (optional for enhanced observability)
- **Alerting**: Alertmanager, PagerDuty integration
- **Dashboarding**: Grafana, Kibana

## Implementation Steps

### 1. Setting Up the Metrics Stack

#### Deploy Prometheus Operator

Use the kube-prometheus-stack Helm chart to install Prometheus Operator, which provides:

- Prometheus instances
- ServiceMonitor and PodMonitor CRDs
- Default alerting rules
- Grafana with pre-configured dashboards
- Node exporter for hardware metrics
- kube-state-metrics for Kubernetes object metrics

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.resources.requests.memory=2Gi \
  --set prometheus.prometheusSpec.resources.requests.cpu=500m
```

#### Configure ServiceMonitors

Create ServiceMonitor resources to scrape metrics from your services:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
  namespaceSelector:
    matchNames:
    - default
```

#### Add Thanos for Long-term Storage

Integrate Thanos with Prometheus for long-term metrics storage and high availability:

```bash
# Install Thanos components
helm install thanos bitnami/thanos \
  --namespace monitoring \
  --set objstoreConfig="type: S3\nconfig:\n  bucket: thanos\n  endpoint: minio.default.svc.cluster.local:9000\n  access_key: minioadmin\n  secret_key: minioadmin\n  insecure: true" \
  --set query.replicaCount=2 \
  --set bucketweb.enabled=true \
  --set compactor.enabled=true

# Configure Prometheus to use Thanos sidecar
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.thanos.baseImage=quay.io/thanos/thanos \
  --set prometheus.prometheusSpec.thanos.version=v0.24.0 \
  --set prometheus.prometheusSpec.thanos.objectStorageConfig.key=objstore.yml \
  --set prometheus.prometheusSpec.thanos.objectStorageConfig.name=thanos-objstore-config
```

### 2. Implementing the Logging Stack

#### Deploy Elasticsearch

Install Elasticsearch using the Elastic Cloud on Kubernetes (ECK) operator:

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.3.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.3.0/operator.yaml

# Create namespace
kubectl create namespace logging

# Create Elasticsearch cluster
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: logging
spec:
  version: 8.2.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: standard
  http:
    tls:
      selfSignedCertificate:
        disabled: true
EOF
```

#### Install Kibana

Deploy Kibana to visualize and search logs:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: logging
spec:
  version: 8.2.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  http:
    tls:
      selfSignedCertificate:
        disabled: true
EOF
```

#### Deploy Fluent Bit

Use Fluent Bit to collect container logs and send them to Elasticsearch:

```bash
# Add Fluent Bit Helm repo
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --set config.outputs="[OUTPUT]\n    Name            es\n    Match           *\n    Host            elasticsearch-es-http.logging.svc.cluster.local\n    Port            9200\n    Index           kubernetes_cluster\n    Type            _doc\n    HTTP_User       elastic\n    HTTP_Passwd     `kubectl get secret elasticsearch-es-elastic-user -n logging -o jsonpath='{.data.elastic}' | base64 --decode`\n    tls             off\n    tls.verify      off\n    Logstash_Format On\n    Logstash_Prefix kubernetes\n    Retry_Limit     False\n    Replace_Dots    On\n    Trace_Error     On"
```

### 3. Setting Up Distributed Tracing

#### Deploy Jaeger

Install Jaeger for distributed tracing:

```bash
# Create namespace
kubectl create namespace tracing

# Add Jaeger Helm repo
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Install Jaeger
helm install jaeger jaegertracing/jaeger \
  --namespace tracing \
  --set provisionDataStore.cassandra=false \
  --set provisionDataStore.elasticsearch=true \
  --set storage.type=elasticsearch \
  --set storage.elasticsearch.host=elasticsearch-es-http.logging.svc.cluster.local \
  --set storage.elasticsearch.port=9200 \
  --set storage.elasticsearch.user=elastic \
  --set storage.elasticsearch.password=`kubectl get secret elasticsearch-es-elastic-user -n logging -o jsonpath='{.data.elastic}' | base64 --decode`
```

#### Deploy OpenTelemetry Collector

Install the OpenTelemetry Collector to receive, process and export telemetry data:

```bash
# Add OpenTelemetry Helm repo
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Install OpenTelemetry Collector
helm install opentelemetry-collector open-telemetry/opentelemetry-collector \
  --namespace tracing \
  --set mode=deployment \
  --set config.receivers.jaeger.protocols.grpc.endpoint=0.0.0.0:14250 \
  --set config.receivers.otlp.protocols.grpc.endpoint=0.0.0.0:4317 \
  --set config.processors.batch={} \
  --set config.exporters.jaeger.endpoint=jaeger-collector.tracing.svc.cluster.local:14250 \
  --set config.service.pipelines.traces.receivers[0]=otlp \
  --set config.service.pipelines.traces.processors[0]=batch \
  --set config.service.pipelines.traces.exporters[0]=jaeger
```

### 4. Configuring Application Instrumentation

#### Metrics Instrumentation

For a Node.js application, you can use the prom-client library:

```javascript
const express = require('express');
const client = require('prom-client');

// Create a Registry to register metrics
const register = new client.Registry();

// Add default metrics (GC, memory, etc)
client.collectDefaultMetrics({ register });

// Create custom metrics
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [register]
});

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path', 'status'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
  registers: [register]
});

const app = express();

// Middleware to track requests
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    httpRequestsTotal.inc({
      method: req.method,
      path: req.path,
      status: res.statusCode
    });
    httpRequestDuration.observe(
      {
        method: req.method,
        path: req.path,
        status: res.statusCode
      },
      duration / 1000
    );
  });
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(3000);
```

#### Logging Instrumentation

Use structured logging with context information:

```javascript
const winston = require('winston');
const { format } = winston;

const logger = winston.createLogger({
  level: 'info',
  format: format.combine(
    format.timestamp(),
    format.json()
  ),
  defaultMeta: { service: 'my-service' },
  transports: [
    new winston.transports.Console()
  ]
});

// Middleware to add request ID to logs
app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] || uuid.v4();
  req.requestId = requestId;
  res.setHeader('x-request-id', requestId);
  
  // Add request context to all log entries within this request
  logger.defaultMeta = { 
    ...logger.defaultMeta, 
    requestId,
    userId: req.user?.id,
    ip: req.ip
  };
  
  next();
});

// Log API requests
app.use((req, res, next) => {
  logger.info('API request', {
    method: req.method,
    path: req.path,
    query: req.query,
    headers: req.headers
  });
  next();
});
```

#### Tracing Instrumentation

Use OpenTelemetry for distributed tracing in a Node.js application:

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { MongoDBInstrumentation } = require('@opentelemetry/instrumentation-mongodb');

// Set up the tracer provider
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
  }),
});

// Configure span processor and exporter
const otlpExporter = new OTLPTraceExporter({
  url: 'opentelemetry-collector.tracing.svc.cluster.local:4317',
});
const spanProcessor = new BatchSpanProcessor(otlpExporter);
provider.addSpanProcessor(spanProcessor);

// Register instrumentations
registerInstrumentations({
  tracerProvider: provider,
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
    new MongoDBInstrumentation(),
  ],
});

// Register the provider
provider.register();
```

### 5. Creating a Unified Observability Dashboard

#### Configure Grafana Data Sources

Add Elasticsearch and Jaeger as data sources in Grafana:

```bash
# Get Grafana password
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode

# Port-forward Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Then navigate to http://localhost:3000 and add:

1. Elasticsearch data source:
   - URL: http://elasticsearch-es-http.logging.svc.cluster.local:9200
   - Basic auth credentials: elastic / [password]
   - Index name: kubernetes-*
   - Time field: @timestamp

2. Jaeger data source:
   - URL: http://jaeger-query.tracing.svc.cluster.local:16686

#### Create a Multi-Data Dashboard

Create a dashboard that combines metrics, logs, and traces using Grafana's mixed data sources feature:

```bash
# Example dashboard JSON configuration
cat <<EOF > observability-dashboard.json
{
  "dashboard": {
    "title": "Microservices Observability",
    "panels": [
      {
        "title": "Service Health Overview",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 }
      },
      {
        "title": "Request Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 1 }
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5.*\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 1 }
      },
      {
        "title": "System Metrics",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 9 }
      },
      {
        "title": "CPU Usage",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"default\"}[5m])) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 10 }
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(container_memory_working_set_bytes{namespace=\"default\"}) by (pod)",
            "legendFormat": "{{pod}}"
          }
        ],
        "gridPos": { "h": 8, "w": 8, "x": 8, "y": 10 }
      },
      {
        "title": "Network I/O",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(container_network_receive_bytes_total{namespace=\"default\"}[5m])) by (pod)",
            "legendFormat": "{{pod}} RX"
          },
          {
            "expr": "sum(rate(container_network_transmit_bytes_total{namespace=\"default\"}[5m])) by (pod)",
            "legendFormat": "{{pod}} TX"
          }
        ],
        "gridPos": { "h": 8, "w": 8, "x": 16, "y": 10 }
      },
      {
        "title": "Logs",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 18 }
      },
      {
        "title": "Error Logs",
        "type": "logs",
        "datasource": "Elasticsearch",
        "targets": [
          {
            "query": "kubernetes.namespace: default AND log: *error*",
            "refId": "A"
          }
        ],
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 19 }
      },
      {
        "title": "Traces",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 27 }
      },
      {
        "title": "Service Performance",
        "type": "jaeger-panel",
        "datasource": "Jaeger",
        "targets": [
          {
            "queryType": "search",
            "service": "my-service",
            "operation": "HTTP GET",
            "limit": 20
          }
        ],
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 28 }
      }
    ],
    "refresh": "10s",
    "time": { "from": "now-1h", "to": "now" }
  }
}
EOF

# Import the dashboard using Grafana API
curl -X POST -H "Content-Type: application/json" -d @observability-dashboard.json http://admin:$GRAFANA_PASSWORD@localhost:3000/api/dashboards/db
```

### 6. Setting Up Alerts and Notifications

#### Configure Alertmanager

Set up alert rules and notification channels in Alertmanager:

```bash
# Create ConfigMap for Alertmanager configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
    route:
      group_by: ['alertname', 'job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-notifications'
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] Monitoring Event Notification'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
            *Description:* {{ .Annotations.description }}
            *Details:*
            {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
            {{ end }}
          {{ end }}
    - name: 'pagerduty-notifications'
      pagerduty_configs:
      - service_key: '0123456789abcdef0123456789abcdef'
        send_resolved: true
EOF

# Update Prometheus Operator to use the ConfigMap
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set alertmanager.config.enabled=false \
  --set alertmanager.configMapOverrideName=alertmanager-config
```

#### Create PrometheusRules

Define alert rules for your applications:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
  - name: application.rules
    rules:
    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{status=~"5.*"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: High error rate detected
        description: "{{ $labels.service }} has a high error rate: {{ $value | humanizePercentage }}"
    - alert: SlowResponseTime
      expr: http_request_duration_seconds{quantile="0.9"} > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: Slow response time detected
        description: "{{ $labels.service }} has a 90th percentile response time > 1s: {{ $value }}s"
    - alert: HighCPUUsage
      expr: sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod) / sum(kube_pod_container_resource_limits_cpu_cores{namespace="default"}) by (pod) > 0.8
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: High CPU usage detected
        description: "Pod {{ $labels.pod }} is using more than 80% of its CPU limit for over 15 minutes: {{ $value | humanizePercentage }}"
    - alert: HighMemoryUsage
      expr: sum(container_memory_working_set_bytes{namespace="default"}) by (pod) / sum(kube_pod_container_resource_limits_memory_bytes{namespace="default"}) by (pod) > 0.8
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: High memory usage detected
        description: "Pod {{ $labels.pod }} is using more than 80% of its memory limit for over 15 minutes: {{ $value | humanizePercentage }}"
    - alert: PodCrashLooping
      expr: increase(kube_pod_container_status_restarts_total{namespace="default"}[1h]) > 5
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: Pod is crash looping
        description: "Pod {{ $labels.pod }} has restarted more than 5 times in the last hour"
EOF
```

### 7. Integrating with Service Mesh (Optional)

#### Install Istio

Add Istio for enhanced tracing and metrics collection:

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*

# Install Istio with default profile
./bin/istioctl install --set profile=default -y

# Create namespace for demo application
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled

# Install Prometheus and Grafana add-ons if not already installed
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
kubectl apply -f samples/addons/kiali.yaml
```

#### Deploy Sample Application with Istio

Deploy a sample application to demonstrate Istio's observability features:

```bash
# Deploy BookInfo sample application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo

# Generate traffic
for i in $(seq 1 100); do
  curl -s -o /dev/null "http://$(kubectl -n istio-ingress get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/productpage"
  sleep 0.5
done
```

#### View Kiali Dashboard

Access the Kiali dashboard to visualize service mesh topology:

```bash
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

Then navigate to http://localhost:20001/kiali

## Advanced Configurations

### Implementing SLO Monitoring

Set up Service Level Objective (SLO) monitoring using Prometheus:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
  - name: slo.rules
    rules:
    # Availability SLO: 99.9% of requests should be successful (non-5xx responses)
    - record: availability:ratio_rate5m
      expr: sum(rate(http_requests_total{status!~"5.*"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)
    - record: availability:ratio_rate30m
      expr: sum(rate(http_requests_total{status!~"5.*"}[30m])) by (service) / sum(rate(http_requests_total[30m])) by (service)
    - record: availability:ratio_rate1h
      expr: sum(rate(http_requests_total{status!~"5.*"}[1h])) by (service) / sum(rate(http_requests_total[1h])) by (service)
    - record: availability:ratio_rate6h
      expr: sum(rate(http_requests_total{status!~"5.*"}[6h])) by (service) / sum(rate(http_requests_total[6h])) by (service)
    # Latency SLO: 95% of requests should be processed in less than 300ms
    - record: latency:ratio_rate5m
      expr: sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (service) / sum(rate(http_request_duration_seconds_count[5m])) by (service)
    - record: latency:ratio_rate30m
      expr: sum(rate(http_request_duration_seconds_bucket{le="0.3"}[30m])) by (service) / sum(rate(http_request_duration_seconds_count[30m])) by (service)
    - record: latency:ratio_rate1h
      expr: sum(rate(http_request_duration_seconds_bucket{le="0.3"}[1h])) by (service) / sum(rate(http_request_duration_seconds_count[1h])) by (service)
    - record: latency:ratio_rate6h
      expr: sum(rate(http_request_duration_seconds_bucket{le="0.3"}[6h])) by (service) / sum(rate(http_request_duration_seconds_count[6h])) by (service)
    # Error Budget Alert (Alert if error budget spent is over 75% in last hour)
    - alert: ErrorBudgetBurn
      expr: |
        (
          availability:ratio_rate1h < 0.999
          and
          availability:ratio_rate5m < 0.999
        ) or (
          latency:ratio_rate1h < 0.95
          and
          latency:ratio_rate5m < 0.95
        )
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Error budget burn detected"
        description: "Service {{ $labels.service }} is burning through its error budget too fast."
EOF
```

### Implementing Log-based Metrics

Create metrics from log data using Elasticsearch and Grafana:

1. Create an Elasticsearch index pattern in Kibana (kubernetes-*)

2. Create a Grafana dashboard with Elasticsearch as a data source:

```grafana
ESQUERY: kubernetes.namespace: "default" AND log: *error*
GROUP BY: kubernetes.pod.name
INTERVAL: 10m
METRIC: count()
```

### Cross-Service Correlation

Correlate metrics, logs, and traces using a common identifier (e.g., request ID):

1. Ensure your applications propagate a `x-request-id` header

2. Add this ID to logs and traces

3. Create a Grafana dashboard with drill-down capabilities:
   - From metrics panel: Select time range with anomaly
   - To logs panel: Filter logs by time range
   - To trace panel: View traces for specific request IDs identified in logs

## Project Outcomes and Success Metrics

### Key Performance Indicators

1. **Mean Time to Detection (MTTD)**: Time to detect issues
2. **Mean Time to Resolution (MTTR)**: Time to resolve issues
3. **Alert Accuracy**: Percentage of true positive alerts
4. **SLO Compliance**: Meeting defined SLOs

### Operational Improvement Metrics

1. **Reduced Downtime**: Total downtime compared to baseline
2. **Decreased Incident Frequency**: Number of incidents over time
3. **Improved Developer Productivity**: Time saved in debugging
4. **Enhanced System Understanding**: Better visibility into system behavior

## Conclusion

A comprehensive observability platform combines metrics, logs, and traces to provide a unified view of your Kubernetes-based microservices architecture. By implementing these components, you'll gain deep insights into your applications' behavior, enabling faster troubleshooting, proactive issue detection, and data-driven decision making.

This project guide has walked through the complete implementation of such a platform, from setting up the infrastructure components to configuring application instrumentation and creating unified dashboards. By customizing this approach to your specific needs and gradually expanding its capabilities, you can create an observability solution that evolves with your system and continues to provide value as your microservices architecture grows in complexity.