# Grafana

Grafana is a powerful open-source visualization and analytics platform that allows you to query, visualize, alert on, and understand your metrics, logs, and traces. In Kubernetes environments, Grafana is commonly used to create dashboards for monitoring cluster health, application performance, and business metrics. This guide covers everything from basic Grafana setup to advanced configurations in Kubernetes.

## Introduction to Grafana

Grafana is the leading tool for creating observability dashboards. It connects to a wide variety of data sources, including Prometheus, Elasticsearch, InfluxDB, MySQL, PostgreSQL, and many more. Key features include:

- Multi-dimensional data visualization
- Customizable dashboards with various panels (graphs, tables, heatmaps, etc.)
- Alerting engine with notification channels
- User management and authentication
- Annotations for event correlation
- Variable templates for dynamic dashboards
- Plugin architecture for extending functionality

### Grafana Architecture in Kubernetes

```
┌───────────────────────────────────────────────┐
│                Kubernetes Cluster             │
│                                               │
│  ┌─────────────────┐      ┌─────────────────┐ │
│  │                 │      │                 │ │
│  │   Grafana Pods  │      │  Data Sources   │ │
│  │                 │      │                 │ │
│  │ ┌─────────────┐ │      │ ┌─────────────┐ │ │
│  │ │  Grafana    │ │      │ │ Prometheus  │ │ │
│  │ │  Server     │◄┼──────┼─┤             │ │ │
│  │ │             │ │      │ │             │ │ │
│  │ └─────────────┘ │      │ └─────────────┘ │ │
│  │                 │      │                 │ │
│  │ ┌─────────────┐ │      │ ┌─────────────┐ │ │
│  │ │  Grafana    │ │      │ │ Loki        │ │ │
│  │ │  Database   │ │      │ │             │ │ │
│  │ │  (SQLite)   │ │      │ │             │ │ │
│  │ └─────────────┘ │      │ └─────────────┘ │ │
│  │                 │      │                 │ │
│  └─────────────────┘      └─────────────────┘ │
│                                               │
│  ┌─────────────────┐      ┌─────────────────┐ │
│  │   Ingress       │      │   Persistent    │ │
│  │   Controller    │      │   Storage       │ │
│  └─────────────────┘      └─────────────────┘ │
└───────────────────────────────────────────────┘
```

## Deploying Grafana in Kubernetes

### Using Helm

The simplest way to deploy Grafana in Kubernetes is using Helm:

```bash
# Add the Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminPassword=admin \
  --set datasources."datasources\.yaml".apiVersion=1 \
  --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
  --set datasources."datasources\.yaml".datasources[0].type=prometheus \
  --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local \
  --set datasources."datasources\.yaml".datasources[0].access=proxy \
  --set datasources."datasources\.yaml".datasources[0].isDefault=true
```

### Manual Deployment with YAML

If you prefer more control, deploy Grafana using YAML manifests:

```yaml
# grafana-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |-
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local
      access: proxy
      isDefault: true
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-providers
  namespace: monitoring
data:
  dashboards.yaml: |-
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-settings
  namespace: monitoring
data:
  grafana.ini: |-
    [server]
    root_url = https://grafana.example.com
    [analytics]
    check_for_updates = true
    [security]
    admin_user = admin
    [auth.anonymous]
    enabled = false
    [unified_alerting]
    enabled = true
```

```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
        - name: grafana
          image: grafana/grafana:9.3.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http-grafana
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /robots.txt
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 2
          livenessProbe:
            failureThreshold: 3
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 3000
            timeoutSeconds: 1
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 256Mi
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-storage
            - mountPath: /etc/grafana/provisioning/datasources
              name: grafana-datasources
              readOnly: false
            - mountPath: /etc/grafana/provisioning/dashboards
              name: grafana-dashboards-providers
              readOnly: false
            - mountPath: /var/lib/grafana/dashboards
              name: grafana-dashboards
              readOnly: false
            - mountPath: /etc/grafana/grafana.ini
              name: grafana-settings
              subPath: grafana.ini
              readOnly: true
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-storage
        - name: grafana-datasources
          configMap:
            defaultMode: 420
            name: grafana-datasources
        - name: grafana-dashboards-providers
          configMap:
            defaultMode: 420
            name: grafana-dashboards-providers
        - name: grafana-dashboards
          emptyDir: {}
        - name: grafana-settings
          configMap:
            defaultMode: 420
            name: grafana-settings
```

```yaml
# grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-storage
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```yaml
# grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  ports:
    - port: 80
      targetPort: http-grafana
      protocol: TCP
      name: http
  selector:
    app: grafana
  type: ClusterIP
```

```yaml
# grafana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              name: http
  tls:
  - hosts:
    - grafana.example.com
    secretName: grafana-tls
```

Apply these manifests:

```bash
kubectl apply -f grafana-configmap.yaml
kubectl apply -f grafana-pvc.yaml
kubectl apply -f grafana-deployment.yaml
kubectl apply -f grafana-service.yaml
kubectl apply -f grafana-ingress.yaml
```

### Accessing Grafana

After deploying Grafana, you can access it by:

```bash
# Port forwarding (for development/testing)
kubectl port-forward -n monitoring svc/grafana 3000:80

# Get admin password (if using Helm)
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Visit http://localhost:3000 in your browser (or use your Ingress URL if configured).

## Configuring Grafana

### Data Sources

Grafana supports many data sources. The most common in Kubernetes environments are:

#### Prometheus

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |-
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local
      access: proxy
      isDefault: true
      jsonData:
        timeInterval: 15s
        queryTimeout: 60s
        httpMethod: POST
```

#### Loki (for logs)

```yaml
- name: Loki
  type: loki
  url: http://loki-gateway.monitoring.svc.cluster.local
  access: proxy
  jsonData:
    maxLines: 1000
```

#### Tempo (for traces)

```yaml
- name: Tempo
  type: tempo
  url: http://tempo.monitoring.svc.cluster.local:3100
  access: proxy
  jsonData:
    httpMethod: GET
    serviceMap:
      datasourceUid: 'prometheus'
```

#### Adding Multiple Data Sources

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |-
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local
      access: proxy
      isDefault: true
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local
      access: proxy
      jsonData:
        maxLines: 1000
    - name: MySQL
      type: mysql
      url: mysql:3306
      database: grafana
      user: grafana
      secureJsonData:
        password: $MYSQL_PASSWORD
      jsonData:
        maxOpenConns: 100
        maxIdleConns: 100
        connMaxLifetime: 14400
```

### Dashboard Provisioning

Automatically provision dashboards by adding JSON definitions to ConfigMaps:

```yaml
# grafana-dashboards-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-kubernetes
  namespace: monitoring
data:
  kubernetes-cluster-monitoring.json: |-
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
      "gnetId": 315,
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
            "dataLinks": []
          },
          "percentage": false,
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(container_cpu_usage_seconds_total{image!=\"\"}[5m])) by (namespace)",
              "refId": "A"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "CPU Usage by Namespace",
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
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "Prometheus",
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 0
          },
          "hiddenSeries": false,
          "id": 3,
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
            "dataLinks": []
          },
          "percentage": false,
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(container_memory_working_set_bytes{image!=\"\"}) by (namespace)",
              "refId": "A"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Memory Usage by Namespace",
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
              "format": "bytes",
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
      "refresh": "10s",
      "schemaVersion": 22,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "Kubernetes Cluster Monitoring",
      "uid": "kubernetes-cluster",
      "version": 1
    }
```

Add the dashboard provider:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-providers
  namespace: monitoring
data:
  dashboards.yaml: |-
    apiVersion: 1
    providers:
    - name: 'kubernetes'
      orgId: 1
      folder: 'Kubernetes'
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards/kubernetes
```

Update the Grafana deployment to mount these ConfigMaps:

```yaml
spec:
  template:
    spec:
      containers:
        - name: grafana
          # ...
          volumeMounts:
            # ...
            - mountPath: /var/lib/grafana/dashboards/kubernetes
              name: grafana-dashboards-kubernetes
      volumes:
        # ...
        - name: grafana-dashboards-kubernetes
          configMap:
            name: grafana-dashboards-kubernetes
```

### Authentication and Authorization

Configure authentication methods in Grafana:

#### Basic Authentication

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin-credentials
  namespace: monitoring
type: Opaque
stringData:
  admin-user: admin
  admin-password: "StrongPassword123!"
```

Update the Grafana ConfigMap:

```yaml
grafana.ini: |-
  [security]
  admin_user = ${GF_SECURITY_ADMIN_USER}
  admin_password = ${GF_SECURITY_ADMIN_PASSWORD}
```

Update the Deployment:

```yaml
containers:
  - name: grafana
    # ...
    env:
      - name: GF_SECURITY_ADMIN_USER
        valueFrom:
          secretKeyRef:
            name: grafana-admin-credentials
            key: admin-user
      - name: GF_SECURITY_ADMIN_PASSWORD
        valueFrom:
          secretKeyRef:
            name: grafana-admin-credentials
            key: admin-password
```

#### LDAP Authentication

```yaml
grafana.ini: |-
  [auth.ldap]
  enabled = true
  config_file = /etc/grafana/ldap.toml
  allow_sign_up = true
```

Create LDAP ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-ldap-config
  namespace: monitoring
data:
  ldap.toml: |-
    [[servers]]
    host = "ldap.example.org"
    port = 389
    use_ssl = false
    start_tls = false
    bind_dn = "cn=admin,dc=example,dc=org"
    bind_password = "admin_password"
    search_filter = "(cn=%s)"
    search_base_dns = ["dc=example,dc=org"]
    
    [servers.attributes]
    name = "givenName"
    surname = "sn"
    username = "cn"
    member_of = "memberOf"
    email = "email"
    
    [[servers.group_mappings]]
    group_dn = "cn=admins,dc=example,dc=org"
    org_role = "Admin"
    
    [[servers.group_mappings]]
    group_dn = "cn=editors,dc=example,dc=org"
    org_role = "Editor"
    
    [[servers.group_mappings]]
    group_dn = "*"
    org_role = "Viewer"
```

#### OAuth Authentication

```yaml
grafana.ini: |-
  [auth.github]
  enabled = true
  allow_sign_up = true
  client_id = YOUR_GITHUB_CLIENT_ID
  client_secret = YOUR_GITHUB_CLIENT_SECRET
  scopes = user:email,read:org
  auth_url = https://github.com/login/oauth/authorize
  token_url = https://github.com/login/oauth/access_token
  api_url = https://api.github.com/user
  team_ids =
  allowed_organizations = your-github-org
```

## Creating Effective Dashboards

### Dashboard Structure

A well-organized dashboard typically includes:

1. **Overview panels**: High-level metrics and status
2. **Detailed metrics**: Specific performance indicators
3. **Resource usage**: CPU, memory, network, etc.
4. **Error rates and logs**: Problems and exceptions
5. **Business metrics**: Application-specific KPIs

### Panel Types

Grafana offers various panel types for different visualization needs:

#### Time Series

Best for showing data over time:

```
avg by(pod) (rate(container_cpu_usage_seconds_total{namespace="default"}[5m]))
```

#### Stat Panel

For single value displays:

```
sum(kube_pod_container_status_running{namespace="default"})
```

#### Gauge

For values in a specific range:

```
sum(kube_pod_container_status_running{namespace="default"}) / sum(kube_pod_container_status_ready{namespace="default"}) * 100
```

#### Bar Gauge

For comparing multiple values:

```
sum by(pod) (container_memory_usage_bytes{namespace="default"})
```

#### Table

For displaying structured data:

```
sum by(pod, container) (container_memory_usage_bytes{namespace="default"})
```

#### Logs Panel

For displaying log data from sources like Loki:

```
{namespace="default"} |= "error"
```

### Variables and Templates

Make dashboards dynamic with variables:

#### Creating a Namespace Variable

```
Name: namespace
Label: Namespace
Query: label_values(kube_pod_info, namespace)
Include All: true
```

Use the variable in queries:

```
sum by(pod) (rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m]))
```

#### Chained Variables

Create dependent variables:

```
# First variable
Name: namespace
Label: Namespace
Query: label_values(kube_pod_info, namespace)

# Second variable (depends on namespace)
Name: pod
Label: Pod
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
```

### Annotations

Add event markers to dashboards:

```yaml
# Annotation definition
Name: Deployments
Datasource: Prometheus
Query: sum by(kubernetes_pod_name) (changes(kube_deployment_status_replicas_updated{namespace="$namespace"}[5m])) > 0
```

### Alerts

Configure alerts to notify on important conditions:

```
Name: High CPU Usage
Rule: avg by(pod) (rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m])) > 0.8
Evaluation interval: 1m
For: 5m
Notifications: Send to Slack channel #alerts
```

## Essential Kubernetes Dashboards

### Node Exporter Dashboard

This dashboard shows system-level metrics from all nodes:

```bash
# Import dashboard via Grafana UI
# Dashboard ID: 1860
```

Key metrics to monitor:
- CPU usage and load
- Memory usage and swap
- Disk space and I/O
- Network traffic and errors

### Kubernetes Cluster Overview

Monitor cluster-wide metrics:

```bash
# Import dashboard via Grafana UI
# Dashboard ID: 315
```

Key metrics to monitor:
- Pod status and count
- Deployment status
- Resource usage by namespace
- Node status

### Application-Specific Dashboards

Create custom dashboards for your applications:

```yaml
{
  "panels": [
    {
      "title": "Request Rate",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{app=\"my-app\"}[5m])) by (method, path)"
        }
      ]
    },
    {
      "title": "Error Rate",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{app=\"my-app\", status=~\"5..\"}[5m])) / sum(rate(http_requests_total{app=\"my-app\"}[5m]))"
        }
      ]
    },
    {
      "title": "Response Time",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{app=\"my-app\"}[5m])) by (le, path))"
        }
      ]
    }
  ]
}
```

## Advanced Grafana Features

### Unified Alerting System

Grafana's unified alerting system provides a centralized place to manage alerts:

```yaml
grafana.ini: |-
  [unified_alerting]
  enabled = true
  [alerting]
  enabled = false
```

Creating an alert rule:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-alert-rules
  namespace: monitoring
data:
  cpu-alert.yaml: |-
    apiVersion: 1
    groups:
      - name: cpu-usage
        folder: Kubernetes
        interval: 1m
        rules:
          - name: high-cpu-usage
            condition: $B
            data:
              - refId: A
                relativeTimeRange:
                  from: 600
                  to: 0
                datasourceUid: prometheusUID
                expr: avg by(pod) (rate(container_cpu_usage_seconds_total{namespace="default"}[5m]))
              - refId: B
                relativeTimeRange:
                  from: 0
                  to: 0
                datasourceUid: "__expr__"
                expr: $A > 0.8
            noDataState: NoData
            execErrState: Error
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High CPU usage for pod {{ $labels.pod }}"
              description: "Pod {{ $labels.pod }} has high CPU usage of {{ $values.A }}"
```

### Dashboard Variables

Use template variables to make dashboards flexible:

```
// Global variable for time range
$__from - Start of the current time range (in milliseconds)
$__to - End of the current time range (in milliseconds)

// Custom variables
$namespace - Selected namespace
$pod - Selected pod
```

Query examples with variables:

```
// Filter by namespace
sum by(pod) (container_memory_usage_bytes{namespace="$namespace"})

// Filter by pod
container_memory_usage_bytes{namespace="$namespace", pod="$pod"}

// Time range variable
rate(http_requests_total[$__rate_interval])
```

### Plugin System

Extend Grafana with plugins:

#### Installing plugins via Helm

```bash
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --set "plugins={grafana-piechart-panel,grafana-worldmap-panel}"
```

#### Manual plugin installation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  template:
    spec:
      containers:
        - name: grafana
          # ...
          env:
            - name: GF_INSTALL_PLUGINS
              value: "grafana-piechart-panel,grafana-worldmap-panel"
```

### Grafana API

Automate Grafana operations using its API:

```bash
# Create API key for automation
GRAFANA_API_KEY=$(curl -X POST -H "Content-Type: application/json" -d '{"name":"automation", "role": "Admin"}' http://admin:admin@grafana:3000/api/auth/keys | jq -r '.key')

# Create dashboard using API
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d @dashboard.json \
  http://grafana:3000/api/dashboards/db
```

## High Availability and Production Setup

### Scaling Grafana

For production environments, consider a more robust setup:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    # ...
```

### External Database

Use an external database instead of the built-in SQLite:

```yaml
grafana.ini: |-
  [database]
  type = mysql
  host = mysql:3306
  name = grafana
  user = grafana
  password = ${GF_DATABASE_PASSWORD}
  max_idle_conn = 100
  max_open_conn = 100
  conn_max_lifetime = 14400
```

### External Image Storage

Store rendered images externally:

```yaml
grafana.ini: |-
  [external_image_storage]
  provider = s3
  
  [external_image_storage.s3]
  bucket = grafana-images
  region = us-west-2
  access_key = ${GF_S3_ACCESS_KEY}
  secret_key = ${GF_S3_SECRET_KEY}
```

### Session Management

Configure session management for high availability:

```yaml
grafana.ini: |-
  [session]
  provider = redis
  provider_config = addr=redis:6379,db=0,pool_size=100,prefix=grafana:
  cookie_secure = true
  cookie_name = grafana_session
  session_life_time = 86400
```

### Load Balancing and Ingress

Configure an Ingress with sticky sessions:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "grafana_session"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              name: http
  tls:
  - hosts:
    - grafana.example.com
    secretName: grafana-tls
```

## Integration with Other Tools

### Prometheus and Alertmanager

Connect Grafana alerts to Alertmanager:

```yaml
grafana.ini: |-
  [unified_alerting.alertmanager]
  enabled = true
  external_url = http://alertmanager-operated.monitoring.svc.cluster.local:9093
```

### Loki for Logs

Configure Loki data source for log visualization:

```yaml
datasources.yaml: |-
  apiVersion: 1
  datasources:
  - name: Loki
    type: loki
    url: http://loki-gateway.monitoring.svc.cluster.local
    access: proxy
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: "trace_id"
          matcherRegex: "traceID=(\\w+)"
          url: "http://tempo:3100/tempo/trace/$${__value.raw}"
```

### Tempo for Tracing

Configure Tempo for distributed tracing:

```yaml
datasources.yaml: |-
  apiVersion: 1
  datasources:
  - name: Tempo
    type: tempo
    url: http://tempo.monitoring.svc.cluster.local:3100
    access: proxy
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['instance', 'pod', 'namespace']
        mappedTags: [{ key: 'service.name', value: 'service' }]
        mapTagNamesEnabled: false
        spanStartTimeShift: '1h'
        spanEndTimeShift: '1h'
        filterByTraceID: true
        filterBySpanID: false
```

### Prometheus Metrics Browser

Use Explore mode to browse metrics:

```
# Create a dashboard link to Explore
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-links
  namespace: monitoring
data:
  links.yaml: |-
    apiVersion: 1
    links:
    - name: Explore Prometheus
      type: link
      icon: external link
      url: /explore?left=["now-1h","now","Prometheus",{"expr":""},{"mode":"Metrics"},{"ui":[true,true,true,"none"]}]
      target: _blank
```

## Security Considerations

### Secure Grafana Configuration

Harden your Grafana installation:

```yaml
grafana.ini: |-
  [security]
  # Disable creation of admin user on first start of grafana
  disable_initial_admin_creation = true
  
  # Disable gravatar profile images
  disable_gravatar = true
  
  # Disable user signup
  allow_sign_up = false
  
  # Only allow admins to create organizations
  allow_org_create = false
  
  # Auto-assign org role
  auto_assign_org_role = Viewer
  
  # Set cookie secure flag
  cookie_secure = true
  
  # Set cookie same site policy
  cookie_samesite = lax
  
  # Enforce strict transport security
  strict_transport_security = true
  
  # Enforce strict transport security max age
  strict_transport_security_max_age_seconds = 86400
  
  # Enforce strict transport security preload
  strict_transport_security_preload = true
  
  # Enforce strict transport security subdomains
  strict_transport_security_subdomains = true
  
  # Hide version on login page
  hide_version = true
```

### Kubernetes RBAC

Restrict access to Grafana resources:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: grafana-clusterrole
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: grafana-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: grafana-clusterrole
subjects:
- kind: ServiceAccount
  name: grafana
  namespace: monitoring
```

### Network Policies

Restrict network access to Grafana:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: grafana-network-policy
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app: grafana
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090  # Prometheus
    - protocol: TCP
      port: 3100  # Tempo
    - protocol: TCP
      port: 3000  # Loki
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 3306  # MySQL
```

## Troubleshooting

### Common Issues and Solutions

#### Database Connection Issues

```bash
# Check logs for database connection errors
kubectl logs -n monitoring deployment/grafana

# Verify database connectivity
kubectl exec -it -n monitoring deployment/grafana -- env PGPASSWORD=password psql -h postgres -U grafana -c "SELECT 1;"
```

#### Dashboard Loading Issues

```bash
# Check for data source connection problems
kubectl exec -it -n monitoring deployment/grafana -- curl -s http://prometheus-server:9090/api/v1/query?query=up

# Verify dashboard provisioning
kubectl exec -it -n monitoring deployment/grafana -- ls -la /var/lib/grafana/dashboards
```

#### Authentication Problems

```bash
# Test LDAP configuration
kubectl exec -it -n monitoring deployment/grafana -- grafana-cli admin ldap-sync
```

#### Plugin Installation Failures

```bash
# Check plugin installation logs
kubectl logs -n monitoring deployment/grafana | grep "plugin"

# Verify plugin directory permissions
kubectl exec -it -n monitoring deployment/grafana -- ls -la /var/lib/grafana/plugins
```

### Performance Optimization

#### Optimize Queries

- Use recording rules in Prometheus for complex queries
- Add caching to slow data sources
- Limit the time range for dashboard queries

```yaml
grafana.ini: |-
  [dataproxy]
  timeout = 30
  keep_alive_seconds = 30
  
  [dashboards]
  min_refresh_interval = 10s
```

#### Resource Configuration

Adjust resources based on usage patterns:

```yaml
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

## Best Practices

1. **Dashboard Organization**
   - Use folders to organize dashboards by application, team, or purpose
   - Add tags for easier searching
   - Include documentation within dashboards

2. **Consistent Visualization**
   - Standardize on colors, units, and thresholds
   - Use sensible value ranges on graphs
   - Include legends with clear labels

3. **Effective Alerting**
   - Alert on symptoms, not causes
   - Set proper thresholds to avoid alert fatigue
   - Include clear resolution instructions

4. **Performance**
   - Optimize panel queries
   - Set appropriate refresh intervals
   - Use template variables for efficient filtering

5. **Security**
   - Implement proper authentication and authorization
   - Regularly update Grafana and plugins
   - Apply the principle of least privilege

## Conclusion

Grafana is a powerful visualization platform that provides deep insights into your Kubernetes environment and applications. By following this guide, you've learned how to deploy, configure, and use Grafana effectively in Kubernetes.

With properly configured dashboards and alerts, you'll gain comprehensive visibility into the health and performance of your systems, enabling you to proactively manage your infrastructure and applications.

As you continue to build and refine your observability stack, remember that effective dashboards should tell a story about your systems, highlighting what's normal, what's important, and what requires attention.