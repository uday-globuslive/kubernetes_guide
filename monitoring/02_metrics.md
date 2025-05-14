# Metrics Collection in Kubernetes

## Introduction

Metrics collection is a fundamental aspect of monitoring Kubernetes clusters and the applications running on them, including Elastic Stack components. This guide covers various approaches to collecting, storing, and analyzing metrics from Kubernetes clusters and ELK Stack deployments, with a focus on Prometheus-based monitoring and integration with Elasticsearch for unified observability.

## Metrics Types in Kubernetes Environment

Kubernetes environments generate several types of metrics:

1. **Infrastructure Metrics**: CPU, memory, disk, and network metrics from nodes
2. **Kubernetes Metrics**: Pod, deployment, and service metrics
3. **Container Metrics**: Resource usage from individual containers
4. **Application Metrics**: Custom metrics exposed by applications
5. **ELK Stack Metrics**: Specific metrics from Elasticsearch, Logstash, Kibana

## Prometheus for Kubernetes Metrics Collection

Prometheus is the de facto standard for metrics collection in Kubernetes environments.

### Installing Prometheus with Helm

```bash
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack (includes Prometheus, Alertmanager, and Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.service.type=ClusterIP
```

### Key Components of Prometheus Stack

1. **Prometheus Server**: Collects and stores metrics
2. **Alertmanager**: Handles alerts and notifications
3. **kube-state-metrics**: Generates metrics about Kubernetes objects
4. **node-exporter**: Collects node-level metrics
5. **Grafana**: Visualization platform for metrics

### Custom Prometheus Configuration for ELK Stack

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    
    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
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
    
    - job_name: 'elasticsearch'
      metrics_path: '/_prometheus/metrics'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_label_app]
        action: keep
        regex: elk-stack;elasticsearch
      - source_labels: [__address__]
        action: replace
        regex: ([^:]+)(:\d+)?
        replacement: $1:9200
        target_label: instance
      
    - job_name: 'kibana'
      metrics_path: '/api/reporting/stats'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_label_app]
        action: keep
        regex: elk-stack;kibana
      - source_labels: [__address__]
        action: replace
        regex: ([^:]+)(:\d+)?
        replacement: $1:5601
        target_label: instance
```

## Configuring Elasticsearch for Metrics Exposure

Elasticsearch can expose metrics in Prometheus format using the Prometheus exporter plugin.

### Installing the Elasticsearch Prometheus Exporter

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elk-stack
data:
  elasticsearch.yml: |
    cluster.name: elk-cluster
    network.host: 0.0.0.0
    
    # Enable X-Pack monitoring
    xpack.monitoring.enabled: true
    
    # Enable Prometheus exporter
    xpack.monitoring.exporters.my_prometheus:
      type: prometheus
      use_all_nodes: true
```

### Configuring Kibana for Metrics Exposure

Kibana also provides metrics that can be collected by Prometheus.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk-stack
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    
    # Enable monitoring
    monitoring.enabled: true
    monitoring.ui.container.elasticsearch.enabled: true
    
    # Enable reporting metrics
    xpack.reporting.enabled: true
    xpack.reporting.kibanaServer.hostname: "localhost"
```

## Metricbeat for Kubernetes Monitoring

Metricbeat is Elastic's solution for metrics collection, which integrates natively with Elasticsearch.

### Installing Metricbeat for Kubernetes Monitoring

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: elk-stack
spec:
  type: metricbeat
  version: 8.10.0
  elasticsearchRef:
    name: elasticsearch
  kibanaRef:
    name: kibana
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - container
        - node
        - pod
        - system
        - volume
      period: 10s
      host: "${NODE_NAME}"
      hosts: ["https://${NODE_NAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
    - module: kubernetes
      metricsets:
        - apiserver
        - event
      period: 10s
      host: "${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
      ssl.verification_mode: "none"
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
    - module: elasticsearch
      period: 10s
      hosts: ["http://elasticsearch:9200"]
      metricsets:
        - node
        - node_stats
        - cluster_stats
        - index
        - index_recovery
        - index_summary
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        default_indexers.enabled: false
        default_matchers.enabled: false
        indexers:
          - node:
          - pod:
        matchers:
          - fields:
              lookup_fields: ["kubernetes.node.name"]
  daemonSet:
    podTemplate:
      spec:
        serviceAccount: metricbeat
        automountServiceAccountToken: true
        terminationGracePeriodSeconds: 30
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```

### Creating RBAC for Metricbeat

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: elk-stack
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metricbeat
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - events
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - nodes/stats
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: elk-stack
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
```

## Kube Metrics Server

Kubernetes Metrics Server is required for horizontal pod autoscaling based on custom metrics.

### Installing Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Configuring Horizontal Pod Autoscaler for Elasticsearch

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-data
  namespace: elk-stack
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  minReplicas: 3
  maxReplicas: 10
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
        averageUtilization: 75
```

## Custom Metrics with Prometheus Adapter

Prometheus Adapter allows using Prometheus metrics for Horizontal Pod Autoscaler.

### Installing Prometheus Adapter

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --set prometheus.port=80
```

### Configuring Prometheus Adapter for Elasticsearch Metrics

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: '{__name__="elasticsearch_cluster_health_active_shards",namespace="elk-stack"}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_shards"
        as: "es_active_shards"
      metricsQuery: sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)
```

### Using Custom Metrics in HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-custom
  namespace: elk-stack
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: es_active_shards
      target:
        type: AverageValue
        averageValue: 50
```

## Integrating Prometheus with Elasticsearch

You can send Prometheus metrics to Elasticsearch for unified analysis.

### Using Prometheus Elasticsearch Exporter

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-es-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-es-exporter
  template:
    metadata:
      labels:
        app: prometheus-es-exporter
    spec:
      containers:
      - name: exporter
        image: justwatch/elasticsearch_exporter:1.5.0
        args:
        - '--es.uri=http://elasticsearch.elk-stack.svc:9200'
        - '--es.all'
        - '--es.indices'
        - '--es.timeout=30s'
        ports:
        - containerPort: 9114
          name: http
```

### Creating a Service for the Exporter

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-es-exporter
  namespace: monitoring
  labels:
    app: prometheus-es-exporter
spec:
  selector:
    app: prometheus-es-exporter
  ports:
  - port: 9114
    targetPort: 9114
    name: http
```

### Configuring Prometheus to Scrape the Exporter

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-es-exporter
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus-es-exporter
  endpoints:
  - port: http
    interval: 15s
```

## Grafana Dashboards for ELK Stack

Grafana can visualize metrics from Prometheus for monitoring ELK Stack.

### Example Grafana Dashboard for Elasticsearch

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "panels": [
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {},
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 2,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.5.7",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "elasticsearch_cluster_health_status{instance=~\"$instance\",cluster=~\"$cluster\"}",
          "interval": "",
          "legendFormat": "Cluster Status",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Cluster Health Status",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "refresh": "5s",
  "schemaVersion": 27,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": [
      {
        "allValue": null,
        "current": {
          "selected": false,
          "text": "All",
          "value": "$__all"
        },
        "datasource": "Prometheus",
        "definition": "label_values(elasticsearch_cluster_health_status, instance)",
        "description": null,
        "error": null,
        "hide": 0,
        "includeAll": true,
        "label": "Instance",
        "multi": false,
        "name": "instance",
        "options": [],
        "query": {
          "query": "label_values(elasticsearch_cluster_health_status, instance)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "selected": false,
          "text": "All",
          "value": "$__all"
        },
        "datasource": "Prometheus",
        "definition": "label_values(elasticsearch_cluster_health_status, cluster)",
        "description": null,
        "error": null,
        "hide": 0,
        "includeAll": true,
        "label": "Cluster",
        "multi": false,
        "name": "cluster",
        "options": [],
        "query": {
          "query": "label_values(elasticsearch_cluster_health_status, cluster)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Elasticsearch Monitoring",
  "version": 1
}
```

## ElasticSearch Operator (ECK) and Metrics

Elastic Cloud on Kubernetes (ECK) provides integrated monitoring capabilities.

### Metrics Collection with ECK

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elk-stack
spec:
  version: 8.10.0
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: elasticsearch-monitor  # Separate ES cluster for monitoring
    logs:
      elasticsearchRefs:
      - name: elasticsearch-monitor
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
      xpack.monitoring.enabled: true
      xpack.monitoring.collection.enabled: true
  - name: data
    count: 3
    config:
      node.roles: ["data"]
      xpack.monitoring.enabled: true
      xpack.monitoring.collection.enabled: true
```

## Setting Up Alerts

### Prometheus AlertManager Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXX'
    
    route:
      group_by: ['alertname', 'job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'slack-notifications'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitoring-alerts'
        send_resolved: true
        title: '{{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

### Creating Prometheus Alert Rules for Elasticsearch

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: elasticsearch-alerts
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
  - name: elasticsearch
    rules:
    - alert: ElasticsearchClusterRed
      expr: elasticsearch_cluster_health_status{color="red"} == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Elasticsearch cluster health is RED"
        description: "Elasticsearch cluster {{ $labels.cluster }} health is RED for 5 minutes."
    
    - alert: ElasticsearchClusterYellow
      expr: elasticsearch_cluster_health_status{color="yellow"} == 1
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Elasticsearch cluster health is YELLOW"
        description: "Elasticsearch cluster {{ $labels.cluster }} health is YELLOW for 15 minutes."
    
    - alert: ElasticsearchHeapUsageHigh
      expr: (elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Elasticsearch heap usage is high"
        description: "Elasticsearch node {{ $labels.instance }} heap usage is above 85% for 10 minutes."
```

## Collecting JVM Metrics for Elasticsearch

JVM metrics are critical for Elasticsearch performance monitoring.

### JVM Metrics in Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: elasticsearch-jvm
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: elasticsearch
  endpoints:
  - port: metrics
    path: /_prometheus/metrics
    interval: 15s
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'elasticsearch_jvm_.*'
      action: keep
```

## End-to-End Monitoring Solution for ELK on Kubernetes

A complete monitoring stack for ELK on Kubernetes should include:

1. **Metrics Collection**:
   - Prometheus for infrastructure and Kubernetes metrics
   - Metricbeat for detailed ELK Stack metrics
   - Custom exporters for specialized metrics

2. **Storage**:
   - Prometheus for short-term storage and alerting
   - Elasticsearch for long-term storage and analysis

3. **Visualization**:
   - Grafana for infrastructure and Kubernetes dashboards
   - Kibana for ELK Stack monitoring

4. **Alerting**:
   - Prometheus AlertManager for infrastructure alerts
   - Elasticsearch Alerting for application-specific alerts

### Deploying a Complete Solution

```yaml
# Example Monitoring Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
# Prometheus Operator with Grafana, AlertManager
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 5m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: 44.2.1
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  values:
    prometheus:
      prometheusSpec:
        retention: 15d
        resources:
          requests:
            cpu: 200m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        serviceMonitorSelectorNilUsesHelmValues: false
        serviceMonitorSelector: {}
        serviceMonitorNamespaceSelector: {}
        podMonitorSelectorNilUsesHelmValues: false
        podMonitorSelector: {}
        podMonitorNamespaceSelector: {}

    grafana:
      enabled: true
      adminPassword: admin-password-change-me
      persistence:
        enabled: true
        size: 10Gi
      dashboardProviders:
        dashboardproviders.yaml:
          apiVersion: 1
          providers:
          - name: 'elasticsearch'
            orgId: 1
            folder: 'Elasticsearch'
            type: file
            disableDeletion: false
            editable: true
            options:
              path: /var/lib/grafana/dashboards/elasticsearch

---
# Metricbeat for ELK Stack metrics
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: elk-stack
spec:
  type: metricbeat
  version: 8.10.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: elasticsearch
      period: 10s
      hosts: ["http://elasticsearch:9200"]
      metricsets:
        - node
        - node_stats
        - cluster_stats
        - index
    - module: kibana
      period: 10s
      hosts: ["http://kibana:5601"]
      metricsets:
        - stats
```

## Best Practices for Metrics Collection in Kubernetes

1. **Resource Considerations**:
   - Limit resources used by monitoring components
   - Set appropriate scrape intervals to balance accuracy and overhead
   - Use federation for large clusters

2. **Storage Optimization**:
   - Configure appropriate retention periods
   - Use downsampling for long-term storage
   - Consider remote write to Elasticsearch for historical data

3. **Metric Naming and Organization**:
   - Follow Prometheus naming conventions
   - Group related metrics with consistent labels
   - Document custom metrics

4. **Security Considerations**:
   - Secure metrics endpoints with TLS
   - Implement proper RBAC for monitoring components
   - Consider network policies to restrict access to metrics endpoints

5. **High Availability**:
   - Deploy redundant monitoring components
   - Distribute monitoring across nodes
   - Ensure monitoring system failure doesn't affect production services

## Conclusion

Effective metrics collection is essential for maintaining the health and performance of Elasticsearch, Logstash, and Kibana running on Kubernetes. By combining the power of Prometheus for real-time monitoring and alerting with Elasticsearch for long-term storage and analysis, you can create a comprehensive observability solution for your ELK Stack deployment. The tools and configurations provided in this guide should help you establish a robust metrics collection system that enables proactive management of your Kubernetes-based ELK infrastructure.

## References

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Metricbeat Documentation](https://www.elastic.co/guide/en/beats/metricbeat/current/index.html)
- [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Elastic Cloud on Kubernetes Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)