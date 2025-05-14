# Alert Management in Kubernetes

## Introduction

Effective alert management is a critical component of any Kubernetes monitoring strategy. This chapter explores how to implement a comprehensive alerting system for Kubernetes environments, covering alert definition, notification channels, alert routing, and handling alert fatigue. We'll focus on best practices for designing actionable alerts that enable teams to respond quickly to issues while avoiding unnecessary noise.

## Alerting Architecture for Kubernetes

A robust alerting system for Kubernetes typically involves these key components:

1. **Alert Sources**: Systems that generate alerts (Prometheus, Elasticsearch, etc.)
2. **Alert Processors**: Systems that handle alert routing, deduplication, and grouping (Alertmanager, Elasticsearch Alerting)
3. **Notification Channels**: Methods to notify teams (email, Slack, PagerDuty, etc.)
4. **Response Systems**: Tools and processes for handling alerts (incident management, runbooks)

```
┌──────────────────────────────────────────────────┐
│                Kubernetes Cluster                │
│                                                  │
│  ┌────────────┐   ┌────────────┐   ┌───────────┐ │
│  │ Prometheus │   │    Loki    │   │ ELK Stack │ │
│  └──────┬─────┘   └──────┬─────┘   └─────┬─────┘ │
└─────────┼──────────────┬─┼─────────────┬─┼───────┘
          │              │ │             │ │
          ▼              │ │             │ │
┌─────────────────┐      │ │             │ │
│  Alertmanager   │◄─────┘ │             │ │
└────────┬────────┘        │             │ │
         │                 │             │ │
         │                 │             │ │
┌────────▼─────────────────▼─────────────▼─┴───────┐
│                                                  │
│               Alert Distribution                 │
│                                                  │
│  ┌─────────┐  ┌────────┐  ┌────────┐  ┌───────┐  │
│  │  Email  │  │ Slack  │  │PagerDuty│  │ OpsGenie│ │
│  └─────────┘  └────────┘  └────────┘  └───────┘  │
│                                                  │
└─────────────────────────┬────────────────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Response Teams  │
                 └─────────────────┘
```

## Implementing Prometheus Alerting for Kubernetes

Prometheus and Alertmanager provide a powerful combination for Kubernetes alerting.

### 1. Defining Alert Rules in Prometheus

Alert rules in Prometheus are defined using PromQL and typically deployed as ConfigMaps in Kubernetes.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alert-rules
  namespace: monitoring
data:
  alert-rules.yaml: |
    groups:
    - name: kubernetes-system
      rules:
      - alert: KubernetesNodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="true"} == 0
        for: 15m
        labels:
          severity: critical
          category: infrastructure
        annotations:
          summary: "Node {{ $labels.node }} not ready"
          description: "Node {{ $labels.node }} has been unready for more than 15 minutes"
          runbook_url: "https://example.com/runbooks/node-not-ready"
      
      - alert: KubernetesPodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 5
        for: 15m
        labels:
          severity: warning
          category: application
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping ({{ $value }} restarts in 15 minutes)"
          runbook_url: "https://example.com/runbooks/pod-crash-looping"
      
      - alert: KubernetesContainerOOMKilled
        expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
        for: 5m
        labels:
          severity: warning
          category: application
        annotations:
          summary: "Container {{ $labels.container }} in pod {{ $labels.namespace }}/{{ $labels.pod }} OOM killed"
          description: "Container {{ $labels.container }} in pod {{ $labels.namespace }}/{{ $labels.pod }} was OOM killed recently"
          runbook_url: "https://example.com/runbooks/container-oom-killed"
```

### 2. Configuring Alertmanager

Alertmanager handles alert deduplication, grouping, silencing, and routing to notification channels.

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
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alerts@example.com'
      smtp_auth_username: 'alerts@example.com'
      smtp_auth_password: 'password'
      slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
      
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'team-default'
      routes:
      - match:
          severity: critical
        receiver: 'team-critical'
        continue: true
      - match:
          category: infrastructure
        receiver: 'team-infrastructure'
      - match:
          category: application
        receiver: 'team-application'
    
    receivers:
    - name: 'team-default'
      email_configs:
      - to: 'team@example.com'
    
    - name: 'team-critical'
      pagerduty_configs:
      - service_key: '<pagerduty-service-key>'
    
    - name: 'team-infrastructure'
      slack_configs:
      - channel: '#infrastructure-alerts'
        send_resolved: true
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
            *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
    
    - name: 'team-application'
      slack_configs:
      - channel: '#application-alerts'
        send_resolved: true
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
            *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
    
    inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'cluster', 'service']
```

### 3. Deploying with Prometheus Operator

The Prometheus Operator simplifies the deployment and management of Prometheus, Alertmanager, and alert rules.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-system-rules
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
  - name: kubernetes-system
    rules:
    - alert: KubernetesNodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 15m
      labels:
        severity: critical
        category: infrastructure
      annotations:
        summary: "Node {{ $labels.node }} not ready"
        description: "Node {{ $labels.node }} has been unready for more than 15 minutes"
        runbook_url: "https://example.com/runbooks/node-not-ready"
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
  namespace: monitoring
  labels:
    alertmanager: main
spec:
  replicas: 3
  version: v0.25.0
  externalUrl: https://alertmanager.example.com
  configSecret: alertmanager-config
```

## Implementing Elasticsearch Alerting

For environments using the ELK Stack, Elasticsearch provides powerful alerting capabilities.

### 1. Elasticsearch Watches for Infrastructure Monitoring

```json
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["metricbeat-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                },
                {
                  "term": {
                    "kubernetes.node.name": {
                      "value": "{{ctx.metadata.node_name}}"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "cpu_usage": {
              "avg": {
                "field": "system.cpu.total.pct"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.cpu_usage.value > 0.8"
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["ops@example.com"],
        "subject": "High CPU Alert for node {{ctx.metadata.node_name}}",
        "body": {
          "html": "<p>Node {{ctx.metadata.node_name}} is experiencing high CPU usage: {{ctx.payload.aggregations.cpu_usage.value}}%</p>"
        }
      }
    },
    "slack_message": {
      "slack": {
        "account": "monitoring",
        "message": {
          "from": "ElasticSearch Alert",
          "to": ["#alerts"],
          "text": "Node {{ctx.metadata.node_name}} is experiencing high CPU usage: {{ctx.payload.aggregations.cpu_usage.value}}%"
        }
      }
    }
  },
  "metadata": {
    "node_name": "worker-1",
    "severity": "warning"
  }
}
```

### 2. Alerting on Container Logs

```json
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["filebeat-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "match_phrase": {
                    "message": "error"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                },
                {
                  "term": {
                    "kubernetes.namespace": "production"
                  }
                }
              ]
            }
          },
          "aggs": {
            "error_count": {
              "value_count": {
                "field": "_id"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.error_count.value > 10"
    }
  },
  "actions": {
    "slack_notification": {
      "slack": {
        "account": "monitoring",
        "message": {
          "from": "ELK Alerts",
          "to": ["#production-alerts"],
          "text": "High error rate detected in production namespace: {{ctx.payload.aggregations.error_count.value}} errors in the last 5 minutes.",
          "attachments": {
            "color": "danger",
            "title": "View in Kibana",
            "title_link": "https://kibana.example.com/app/discover#/?_g=(refreshInterval:(pause:!t,value:0),time:(from:now-5m,to:now))&_a=(columns:!(kubernetes.pod.name,message),index:'filebeat-*',interval:auto,query:(language:kuery,query:'message:error AND kubernetes.namespace:production'),sort:!('@timestamp',desc))"
          }
        }
      }
    }
  },
  "metadata": {
    "severity": "warning",
    "application": "production"
  }
}
```

## Advanced Alert Design Patterns

### 1. Multi-threshold Alerts

Implement multiple severity levels based on thresholds:

```yaml
- alert: NodeCPUUsageHigh
  expr: |
    (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100 > 80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High CPU usage on {{ $labels.instance }}"
    description: "CPU usage is above 80% (currently at {{ $value }}%)"

- alert: NodeCPUUsageCritical
  expr: |
    (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100 > 95
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Critical CPU usage on {{ $labels.instance }}"
    description: "CPU usage is above 95% (currently at {{ $value }}%)"
```

### 2. Rate of Change Alerts

Alert on the rate of change rather than absolute values:

```yaml
- alert: PodReplicasIncreasing
  expr: |
    delta(kube_deployment_status_replicas{deployment=~".*api"}[10m]) > 5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Unusual scale-up of {{ $labels.deployment }}"
    description: "Deployment {{ $labels.deployment }} has increased by more than 5 replicas in 10 minutes"
```

### 3. Prediction-Based Alerts

Alert on predicted future problems:

```yaml
- alert: DiskSpaceFillingUp
  expr: |
    predict_linear(node_filesystem_free_bytes[6h], 24 * 3600) < 0
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Disk on {{ $labels.instance }} predicted to fill in 24h"
    description: "Based on recent usage patterns, disk is predicted to run out of space within 24 hours"
```

## Notification Channels and Integrations

### 1. Email Notifications

Email remains a foundational notification channel:

```yaml
receivers:
- name: 'email-notifications'
  email_configs:
  - to: 'team@example.com'
    send_resolved: true
    headers:
      subject: '{{ template "email.subject" . }}'
    html: '{{ template "email.html" . }}'
    require_tls: true
```

### 2. Slack Integration

Slack provides real-time notifications with rich formatting:

```yaml
receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXX'
    channel: '#alerts'
    send_resolved: true
    icon_url: 'https://avatars3.githubusercontent.com/u/3380462'
    title: '{{ template "slack.title" . }}'
    text: '{{ template "slack.text" . }}'
    actions:
    - type: button
      text: 'Runbook 📝'
      url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
    - type: button
      text: 'Dashboard 📊'
      url: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
    - type: button
      text: 'Silence 🔕'
      url: '{{ template "slack.silence-url" . }}'
```

### 3. PagerDuty Integration

PagerDuty is ideal for critical alerts requiring immediate action:

```yaml
receivers:
- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: '<pagerduty-service-key>'
    description: '{{ template "pagerduty.description" . }}'
    client: 'Prometheus Alertmanager'
    client_url: '{{ template "pagerduty.client-url" . }}'
    details:
      firing: '{{ template "pagerduty.firing" . }}'
      num_firing: '{{ .Alerts.Firing | len }}'
      resolved: '{{ template "pagerduty.resolved" . }}'
      num_resolved: '{{ .Alerts.Resolved | len }}'
      link: '{{ template "pagerduty.link" . }}'
```

### 4. Microsoft Teams Integration

For organizations using Microsoft Teams:

```yaml
receivers:
- name: 'teams-notifications'
  webhook_configs:
  - url: 'https://outlook.office.com/webhook/TEAM-ID/IncomingWebhook/WEBHOOK-ID'
    send_resolved: true
```

## Creating Custom Alert Templates

Alertmanager and Elasticsearch both support custom templates for notifications:

### Alertmanager Custom Templates

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-templates
  namespace: monitoring
data:
  template.tmpl: |
    {{ define "slack.title" -}}
      [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
    {{- end }}

    {{ define "slack.text" -}}
      {{ range .Alerts }}
        *Alert:* {{ .Annotations.summary }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}
        *Description:* {{ .Annotations.description }}
        *Details:*
          {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
          {{ end }}
      {{ end }}
    {{- end }}

    {{ define "email.subject" -}}
      [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
    {{- end }}

    {{ define "email.html" -}}
    <!DOCTYPE html>
    <html>
    <head>
      <style type="text/css">
        body {
          font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
          color: #333333;
        }
        table {
          width: 100%;
          border-collapse: collapse;
        }
        th, td {
          padding: 8px;
          text-align: left;
          border-bottom: 1px solid #ddd;
        }
        tr:nth-child(even) {
          background-color: #f2f2f2;
        }
        .severity-critical {
          color: #cc0000;
          font-weight: bold;
        }
        .severity-warning {
          color: #ff9900;
          font-weight: bold;
        }
      </style>
    </head>
    <body>
      <h2>{{ .CommonLabels.alertname }}</h2>
      <p><strong>Status:</strong> {{ .Status | toUpper }}</p>
      
      <table>
        <tr>
          <th>Alert</th>
          <th>Severity</th>
          <th>Description</th>
          <th>Started</th>
        </tr>
        {{ range .Alerts }}
        <tr>
          <td>{{ .Annotations.summary }}</td>
          <td class="severity-{{ .Labels.severity }}">{{ .Labels.severity }}</td>
          <td>{{ .Annotations.description }}</td>
          <td>{{ .StartsAt.Format "2006-01-02 15:04:05" }}</td>
        </tr>
        {{ end }}
      </table>
      
      <p>
        <a href="{{ .ExternalURL }}">View in Alertmanager</a>
        {{ if .CommonAnnotations.runbook_url }}
        | <a href="{{ .CommonAnnotations.runbook_url }}">View Runbook</a>
        {{ end }}
      </p>
    </body>
    </html>
    {{- end }}
```

## Managing Alert Fatigue

Alert fatigue occurs when teams receive too many alerts, leading to important notifications being ignored.

### 1. Alert Filtering and Grouping

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  # Only send critical alerts to PagerDuty
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty'
    continue: true
```

### 2. Implementing Time-based Routing

Route alerts differently based on time of day:

```yaml
route:
  routes:
  - match_re:
      severity: critical|warning
    routes:
    - match:
        timeperiod: business-hours
      receiver: slack-team
    - match:
        timeperiod: non-business-hours
      receiver: pagerduty-oncall

time_intervals:
- name: business-hours
  time_intervals:
  - weekdays: ['monday:friday']
    times:
    - start_time: '09:00'
      end_time: '17:00'
- name: non-business-hours
  time_intervals:
  - weekdays: ['monday:friday']
    times:
    - start_time: '00:00'
      end_time: '09:00'
    - start_time: '17:00'
      end_time: '24:00'
  - weekdays: ['saturday', 'sunday']
```

### 3. Alert Silencing and Inhibition

Temporarily silence or inhibit alerts:

```yaml
# Silence during planned maintenance
silences:
- comment: "Planned maintenance - ignore disk space alerts"
  creator: "admin"
  matchers:
  - name: "alertname"
    value: "DiskSpaceCritical"
    isRegex: false
  - name: "instance"
    value: "node-01"
    isRegex: false
  startsAt: "2023-10-01T22:00:00Z"
  endsAt: "2023-10-02T02:00:00Z"

# Inhibit lower severity alerts when a critical alert is firing
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  equal: ['alertname', 'instance']
```

## Implementing Alert Runbooks

Runbooks provide guidance for handling alerts, improving response time and consistency.

### 1. Basic Runbook Structure

```yaml
- alert: KubernetesNodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 15m
  labels:
    severity: critical
    category: infrastructure
  annotations:
    summary: "Node {{ $labels.node }} not ready"
    description: "Node {{ $labels.node }} has been unready for more than 15 minutes"
    runbook_url: "https://example.com/runbooks/node-not-ready"
```

### 2. Markdown Runbook Example

```markdown
# Node Not Ready Runbook

## Alert Description
This alert fires when a Kubernetes node has been in the NotReady state for more than 15 minutes.

## Impact
- Pods scheduled on this node may be unavailable
- New pods may not be scheduled on this node
- Services with pods on this node may have reduced capacity

## Investigation Steps

1. Check the node status and events:
   ```bash
   kubectl describe node <node-name>
   ```

2. Check kubelet logs on the affected node:
   ```bash
   ssh <node-name>
   journalctl -u kubelet -n 500
   ```

3. Common issues to look for:
   - Disk pressure (check `df -h`)
   - Memory pressure (check `free -m`)
   - Network connectivity issues
   - Kubelet service failed

## Resolution

### If disk pressure:
1. Clean up unused Docker images:
   ```bash
   docker system prune -af
   ```

### If memory pressure:
1. Identify memory-intensive processes:
   ```bash
   ps aux --sort=-%mem | head -n 10
   ```
2. Consider restarting problematic services or the node itself

### If kubelet service failed:
1. Restart the kubelet service:
   ```bash
   systemctl restart kubelet
   ```
2. Check logs after restart:
   ```bash
   journalctl -u kubelet -f
   ```

## Escalation
If unable to resolve within 30 minutes, escalate to the infrastructure team.
```

## Measuring Alert Effectiveness

Track and improve alert quality with these metrics:

### 1. Key Alerting Metrics

1. **Alert Volume**: Total number of alerts fired per day/week
2. **Alert Distribution**: Breakdown by severity, service, and alert type
3. **MTTA (Mean Time to Acknowledge)**: Average time to acknowledge alerts
4. **MTTR (Mean Time to Resolve)**: Average time to resolve alerted issues
5. **False Positive Rate**: Percentage of alerts requiring no action

### 2. Implementing Alert Quality Feedback

Create a feedback loop for alert improvement:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-feedback-form
  namespace: monitoring
data:
  feedback-template.json: |
    {
      "fields": [
        {
          "title": "Alert Name",
          "type": "string",
          "required": true
        },
        {
          "title": "Was this alert actionable?",
          "type": "boolean",
          "required": true
        },
        {
          "title": "Did the alert provide sufficient context?",
          "type": "boolean",
          "required": true
        },
        {
          "title": "Was the severity appropriate?",
          "type": "select",
          "options": ["Too high", "Appropriate", "Too low"],
          "required": true
        },
        {
          "title": "Comments",
          "type": "text",
          "required": false
        }
      ]
    }
```

## Alerts for Specific Kubernetes Components

### 1. Control Plane Alerts

```yaml
- alert: KubernetesAPIServerDown
  expr: up{job="apiserver"} == 0
  for: 5m
  labels:
    severity: critical
    component: control-plane
  annotations:
    summary: "Kubernetes API server is down"
    description: "API server on {{ $labels.instance }} has been down for more than 5 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-api-server-down"

- alert: KubernetesControllerManagerDown
  expr: up{job="kube-controller-manager"} == 0
  for: 5m
  labels:
    severity: critical
    component: control-plane
  annotations:
    summary: "Kubernetes controller manager is down"
    description: "Controller manager on {{ $labels.instance }} has been down for more than 5 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-controller-manager-down"

- alert: KubernetesSchedulerDown
  expr: up{job="kube-scheduler"} == 0
  for: 5m
  labels:
    severity: critical
    component: control-plane
  annotations:
    summary: "Kubernetes scheduler is down"
    description: "Scheduler on {{ $labels.instance }} has been down for more than 5 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-scheduler-down"
```

### 2. Node Alerts

```yaml
- alert: KubernetesNodeDiskPressure
  expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
  for: 5m
  labels:
    severity: warning
    component: node
  annotations:
    summary: "Node {{ $labels.node }} is under disk pressure"
    description: "Node {{ $labels.node }} has reported disk pressure for more than 5 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-node-disk-pressure"

- alert: KubernetesNodeMemoryPressure
  expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
  for: 5m
  labels:
    severity: warning
    component: node
  annotations:
    summary: "Node {{ $labels.node }} is under memory pressure"
    description: "Node {{ $labels.node }} has reported memory pressure for more than 5 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-node-memory-pressure"
```

### 3. Pod and Container Alerts

```yaml
- alert: KubernetesPodNotScheduled
  expr: kube_pod_status_phase{phase="Pending"} == 1
  for: 1h
  labels:
    severity: warning
    component: pod
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not scheduled"
    description: "Pod has been in pending state for more than 1 hour"
    runbook_url: "https://example.com/runbooks/kubernetes-pod-not-scheduled"

- alert: KubernetesContainerHighCPUUsage
  expr: |
    sum by(namespace, pod, container) (
      rate(container_cpu_usage_seconds_total{container!=""}[5m])
    ) > 0.8
  for: 15m
  labels:
    severity: warning
    component: container
  annotations:
    summary: "Container {{ $labels.container }} in {{ $labels.namespace }}/{{ $labels.pod }} high CPU usage"
    description: "Container {{ $labels.container }} has been using more than 80% CPU for 15 minutes"
    runbook_url: "https://example.com/runbooks/kubernetes-container-high-cpu-usage"
```

## Integrating with Incident Management Systems

### 1. PagerDuty Integration for Incident Creation

```yaml
receivers:
- name: 'pagerduty-incidents'
  pagerduty_configs:
  - routing_key: '<pagerduty-routing-key>'
    description: '{{ template "pagerduty.description" . }}'
    details:
      severity: '{{ .CommonLabels.severity }}'
      summary: '{{ .CommonAnnotations.summary }}'
      component: '{{ .CommonLabels.component }}'
      cluster: '{{ .CommonLabels.cluster }}'
      custom_details:
        firing_alerts: '{{ .Alerts.Firing | len }}'
```

### 2. ServiceNow Integration for Ticket Creation

```yaml
receivers:
- name: 'servicenow-tickets'
  webhook_configs:
  - url: 'https://example.service-now.com/api/now/table/incident'
    http_config:
      basic_auth:
        username: 'servicenow_user'
        password: 'servicenow_password'
    send_resolved: true
    max_alerts: 0
    custom_payload:
      short_description: '{{ template "servicenow.short_description" . }}'
      description: '{{ template "servicenow.description" . }}'
      impact: '{{ template "servicenow.impact" . }}'
      urgency: '{{ template "servicenow.urgency" . }}'
      assignment_group: 'kubernetes-support'
      cmdb_ci: 'kubernetes-cluster'
```

## Best Practices for Alert Management

### 1. Alert Design Principles

1. **Actionable**: Every alert should be actionable
2. **Relevant**: Alerts should be relevant to their recipients
3. **Clear**: Alert messages should be clear and informative
4. **Contextual**: Include all necessary context for troubleshooting
5. **Prioritized**: Clearly indicate severity and urgency

### 2. Alert Technical Implementation

1. **Use appropriate timing**:
   - `for` duration should match the criticality of the issue
   - Avoid flapping alerts with proper thresholds

2. **Include proper metadata**:
   - Severity, component, service, team
   - Links to dashboards, runbooks, and documentation

3. **Implement proper grouping**:
   - Group related alerts to reduce noise
   - Use inhibition rules to prevent alert storms

### 3. Alert Operations

1. **Regular alert review**:
   - Review alert triggers and thresholds quarterly
   - Analyze alert patterns and adjust as needed

2. **Alert testing**:
   - Test new alerts before deployment
   - Verify they're triggered when expected

3. **Runbook maintenance**:
   - Keep runbooks updated and accessible
   - Add common resolutions based on alert history

## Conclusion

A well-designed alert management system is essential for maintaining the reliability of Kubernetes environments. By implementing appropriate alerting tools, designing meaningful alerts, establishing effective notification channels, and managing alert fatigue, teams can respond quickly to issues while avoiding unnecessary noise. Remember that alert management is an ongoing process that should evolve with your infrastructure and applications.

## References

- [Prometheus Alerting Documentation](https://prometheus.io/docs/alerting/latest/overview/)
- [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Elasticsearch Alerting Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-getting-started.html)
- [PagerDuty Integration Guide](https://www.pagerduty.com/docs/guides/prometheus-integration-guide/)
- [Google SRE Book: Practical Alerting](https://sre.google/sre-book/practical-alerting/)
- [Kubernetes Monitoring Guide](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)