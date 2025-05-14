# ELK Stack for Kubernetes Monitoring

## Introduction

The ELK Stack (Elasticsearch, Logstash, and Kibana) along with Beats has become a standard for comprehensive monitoring and observability in Kubernetes environments. While we covered logging with ELK in detail in the [logging chapter](./03_logging.md), this chapter focuses on using the full ELK Stack as a complete monitoring solution for Kubernetes clusters, integrating metrics, logs, and traces.

## ELK Stack Components for Kubernetes Monitoring

### Core Components

1. **Elasticsearch**: Distributed search and analytics engine that stores all monitoring data
2. **Logstash**: Data processing pipeline for log enrichment and transformation
3. **Kibana**: Visualization platform for exploring, analyzing, and creating dashboards
4. **Beats**: Lightweight data shippers for collecting various types of operational data
   - Filebeat: Log collection
   - Metricbeat: Metric collection
   - Heartbeat: Uptime monitoring
   - Packetbeat: Network traffic analysis
   - APM Server: Application performance monitoring

## Unified Monitoring Architecture

The ELK Stack can provide unified monitoring by collecting and correlating different types of telemetry:

```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                    │
│                                                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  │
│  │ Metrics │   │   Logs  │   │ Traces  │   │ Network │  │
│  └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘  │
│       │             │             │             │       │
│  ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐  │
│  │Metricbeat│  │ Filebeat │  │ APM Agent│  │Packetbeat│  │
│  └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘  │
└───────┼─────────────┼─────────────┼─────────────┼───────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌───────────────────────────────────────────────────────┐
│                     Logstash                          │
│             (Filtering & Enrichment)                  │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│                  Elasticsearch                         │
│                  (Data Storage)                        │
└───────────────────────┬───────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│                     Kibana                             │
│            (Visualization & Alerting)                  │
└───────────────────────────────────────────────────────┘
```

## Setting Up the ELK Stack for Kubernetes Monitoring

### Option 1: ECK (Elastic Cloud on Kubernetes)

ECK is the official Kubernetes operator for Elasticsearch and Kibana, providing:

- Simplified deployment and management
- High availability configurations
- Seamless upgrades
- Secure by default
- Auto-scaling capabilities

#### 1. Install the ECK Operator

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.8.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/operator.yaml
```

#### 2. Deploy Elasticsearch

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: monitoring
spec:
  version: 8.10.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
              cpu: 2
  http:
    tls:
      selfSignedCertificate:
        disabled: true
```

#### 3. Deploy Kibana

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: monitoring
spec:
  version: 8.10.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  http:
    tls:
      selfSignedCertificate:
        disabled: true
```

#### 4. Deploy Beats for Monitoring

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.10.0
  elasticsearchRef:
    name: elasticsearch
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
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```

### Option 2: Helm Chart Deployment

For environments without Elastic Cloud on Kubernetes, Helm provides a flexible deployment option:

#### 1. Add Elastic Helm Repository

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

#### 2. Install Elasticsearch

```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace monitoring \
  --create-namespace \
  --set replicas=3 \
  --set minimumMasterNodes=2 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=1Gi \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=2Gi
```

#### 3. Install Kibana

```bash
helm install kibana elastic/kibana \
  --namespace monitoring \
  --set elasticsearchHosts="http://elasticsearch-master:9200"
```

#### 4. Install Metricbeat

```bash
helm install metricbeat elastic/metricbeat \
  --namespace monitoring \
  --set daemonset.enabled=true \
  --set deployment.enabled=true \
  --set elasticsearchHosts="http://elasticsearch-master:9200"
```

## Configuring Metrics Collection with Metricbeat

Metricbeat collects system and Kubernetes metrics for comprehensive monitoring.

### Key Metricbeat Modules for Kubernetes

1. **Kubernetes Module**: Collects metrics from Kubernetes components
2. **System Module**: Collects host-level metrics
3. **Docker Module**: Collects container metrics
4. **Prometheus Module**: Collects metrics from Prometheus endpoints (for apps instrumented with Prometheus)

### Sample Metricbeat Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-config
  namespace: monitoring
data:
  metricbeat.yml: |-
    metricbeat.modules:
    - module: kubernetes
      enabled: true
      metricsets:
        - container
        - node
        - pod
        - system
        - volume
        - state_node
        - state_deployment
        - state_replicaset
        - state_pod
        - state_container
      period: 10s
      host: "${NODE_NAME}"
      hosts: ["https://${NODE_NAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
      # If using RBAC
      add_metadata: true
      
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
      process.include_top_n:
        by_cpu: 5
        by_memory: 5
        
    - module: prometheus
      period: 10s
      hosts: ["kube-state-metrics.kube-system.svc.cluster.local:8080"]
      metrics_path: /metrics
      
    processors:
      - add_cloud_metadata:
      - add_host_metadata:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          default_indexers.enabled: true
          default_matchers.enabled: true
          
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOSTS}']
      username: '${ELASTICSEARCH_USERNAME}'
      password: '${ELASTICSEARCH_PASSWORD}'
```

## Setting Up APM for Application Tracing

Application Performance Monitoring (APM) provides code-level visibility into application performance.

### 1. Deploy APM Server

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-server
  namespace: monitoring
spec:
  version: 8.10.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  config:
    apm-server:
      rum:
        enabled: true
  http:
    tls:
      selfSignedCertificate:
        disabled: true
```

### 2. Instrument Applications

Instrument your applications using the Elastic APM agents available for various languages:

**Java Example (using Spring Boot)**

Add the APM agent to your application deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  namespace: default
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: spring-app
        image: my-spring-app:1.0.0
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server-apm-http.monitoring.svc.cluster.local:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "spring-app"
        - name: ELASTIC_APM_APPLICATION_PACKAGES
          value: "com.example"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        volumeMounts:
        - name: elastic-apm-agent
          mountPath: /elastic-apm-agent
      volumes:
      - name: elastic-apm-agent
        emptyDir: {}
      initContainers:
      - name: elastic-java-agent
        image: docker.elastic.co/observability/apm-agent-java:1.34.1
        command: ['cp', '-v', '/usr/agent/elastic-apm-agent.jar', '/elastic-apm-agent']
        volumeMounts:
        - name: elastic-apm-agent
          mountPath: /elastic-apm-agent
```

## Centralized Logging with Filebeat

While detailed in our logging chapter, integrating Filebeat with your monitoring solution creates a unified observability platform.

### Key Filebeat Modules for Kubernetes

- Kubernetes module for container logs
- System module for node logs
- Audit module for Kubernetes audit logs
- Specific modules for apps like Nginx, MySQL, Elasticsearch, etc.

### Enriching Logs with Kubernetes Metadata

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: monitoring
data:
  filebeat.yml: |-
    filebeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true
          hints.default_config:
            type: container
            paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log
          templates:
            - condition:
                equals:
                  kubernetes.namespace: kube-system
              config:
                - type: container
                  paths:
                    - /var/log/containers/*${data.kubernetes.container.id}.log
                  exclude_lines: ["^\\s+[\\-`('.|_]"]
                  
    processors:
      - add_cloud_metadata: 
      - add_host_metadata:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
      - add_fields:
          target: ''
          fields:
            cluster_name: 'production-cluster'
            
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOSTS}']
      username: '${ELASTICSEARCH_USERNAME}'
      password: '${ELASTICSEARCH_PASSWORD}'
      index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"
```

## Creating Comprehensive Dashboards in Kibana

### Essential Dashboards for Kubernetes Monitoring

1. **Cluster Overview Dashboard**
   - Cluster state and health
   - Node count and status
   - Pod count and status
   - Deployment status
   - Resource utilization overview

2. **Node Performance Dashboard**
   - CPU, memory, disk, network usage per node
   - Node conditions
   - Container runtime metrics
   - System load metrics

3. **Pod Performance Dashboard**
   - Pod resource usage
   - Container restart count
   - Pod ready status
   - Container status by phase
   - Top resource-consuming pods

4. **Application Performance Dashboard**
   - Request rates
   - Error rates
   - Latency metrics
   - Service health
   - Endpoint metrics

### Sample Dashboard JSON

```json
{
  "attributes": {
    "title": "Kubernetes Cluster Overview",
    "hits": 0,
    "description": "",
    "panelsJSON": "[{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":0,\"y\":0,\"w\":24,\"h\":15,\"i\":\"1\"},\"panelIndex\":\"1\",\"embeddableConfig\":{\"title\":\"Kubernetes Node Count\"}},{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":24,\"y\":0,\"w\":24,\"h\":15,\"i\":\"2\"},\"panelIndex\":\"2\",\"embeddableConfig\":{\"title\":\"Pod Status\"}},{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":0,\"y\":15,\"w\":24,\"h\":15,\"i\":\"3\"},\"panelIndex\":\"3\",\"embeddableConfig\":{\"title\":\"CPU Usage by Node\"}},{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":24,\"y\":15,\"w\":24,\"h\":15,\"i\":\"4\"},\"panelIndex\":\"4\",\"embeddableConfig\":{\"title\":\"Memory Usage by Node\"}},{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":0,\"y\":30,\"w\":48,\"h\":15,\"i\":\"5\"},\"panelIndex\":\"5\",\"embeddableConfig\":{\"title\":\"Top Pods by CPU\"}},{\"version\":\"8.5.0\",\"type\":\"visualization\",\"gridData\":{\"x\":0,\"y\":45,\"w\":48,\"h\":15,\"i\":\"6\"},\"panelIndex\":\"6\",\"embeddableConfig\":{\"title\":\"Top Pods by Memory\"}}]",
    "timeRestore": false,
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"
    }
  },
  "type": "dashboard"
}
```

## Alerting and Anomaly Detection

### Setting Up Alerting Rules

Alerting is critical for proactive monitoring. Set up alerts for:

1. **Infrastructure issues**
   - Node CPU/Memory usage above 80%
   - Pod restart count
   - Disk space below 10%

2. **Application issues**
   - Error rate spikes
   - Latency above thresholds
   - Reduced traffic patterns

3. **Security issues**
   - Unauthorized access attempts
   - Privileged container creation
   - Changes to security contexts

### Sample Alert Rule for High Node CPU Usage

```json
{
  "trigger": {
    "schedule": { "interval": "1m" }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["metricbeat-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                { "term": { "metricset.name": "node" } },
                { "term": { "metricset.module": "kubernetes" } },
                { "range": { "@timestamp": { "gte": "now-5m" } } }
              ]
            }
          },
          "aggs": {
            "nodes": {
              "terms": { "field": "kubernetes.node.name", "size": 100 },
              "aggs": {
                "cpu_usage": {
                  "max": { "field": "kubernetes.node.cpu.usage.nanocores" }
                },
                "cpu_capacity": {
                  "max": { "field": "kubernetes.node.cpu.allocatable.cores" }
                },
                "cpu_pct": {
                  "bucket_script": {
                    "buckets_path": {
                      "usage": "cpu_usage",
                      "capacity": "cpu_capacity"
                    },
                    "script": "params.usage / (params.capacity * 1000000000) * 100"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.nodes.buckets.stream().anyMatch(node -> node.cpu_pct.value > 80)",
      "lang": "painless"
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "High CPU utilization on Kubernetes nodes",
        "body": "One or more nodes have high CPU usage (>80%)."
      }
    }
  }
}
```

### Machine Learning for Anomaly Detection

Elasticsearch's machine learning capabilities can detect anomalies without manual thresholds:

1. **Set up ML jobs for common metrics**
   - Node resource usage patterns
   - Pod scaling patterns
   - Request rate patterns

2. **Example ML job for pod restarts**

```json
{
  "job_id": "pod_restarts",
  "description": "Detect unusual pod restart patterns",
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "function": "count",
        "partition_field_name": "kubernetes.namespace"
      }
    ],
    "influencers": [
      "kubernetes.namespace",
      "kubernetes.pod.name"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  }
}
```

## Implementing Log-Metric-Trace Correlation

The power of the ELK Stack comes from correlating different data sources for complete context.

### 1. Use Common Fields for Correlation

Ensure all monitoring data includes:
- `service.name`
- `kubernetes.namespace`
- `kubernetes.pod.name`
- `kubernetes.node.name`
- `trace.id` (when available)

### 2. Create Correlation Views in Kibana

```
GET _search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "kubernetes.pod.name": "my-app-pod-123456" } },
        { "range": { "@timestamp": { "gte": "now-15m", "lte": "now" } } }
      ]
    }
  },
  "sort": [
    { "@timestamp": { "order": "asc" } }
  ],
  "_source": [
    "@timestamp",
    "data_stream.type",
    "message",
    "log.level",
    "event.dataset"
  ]
}
```

## Security and Access Control

### Implementing Role-Based Access Control

1. **Create role mappings for different teams**
   - Development team: View logs and metrics for their namespaces
   - Operations team: View all cluster metrics and logs
   - Security team: Access to audit logs and security events

2. **Example role definition for Dev team**

```json
{
  "role": {
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["filebeat-*", "metricbeat-*"],
        "privileges": ["read"],
        "query": "{\"term\": {\"kubernetes.namespace\": \"dev\"}}"
      }
    ]
  }
}
```

## Performance Tuning and Scaling

### Optimize Elasticsearch Performance

1. **Shard allocation strategy**
   - Use ILM (Index Lifecycle Management) 
   - Implement hot-warm-cold architecture
   - Right-size shards (aim for 20-40GB per shard)

2. **Sample ILM Policy**

```json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Filebeat and Metricbeat Performance Tips

1. **Adjust batch size and flush interval**
2. **Filter unnecessary logs/metrics at collection time**
3. **Use multiple output workers for higher throughput**

```yaml
# Sample Filebeat optimization
filebeat.yml: |-
  filebeat:
    config.modules:
      reload.enabled: false
    
    # Increase throughput
    registry:
      flush: 5s
    
  output.elasticsearch:
    hosts: ['${ELASTICSEARCH_HOSTS}']
    worker: 4
    bulk_max_size: 1024
    
  # Reduce CPU usage 
  processors:
    - drop_fields:
        fields: ["agent.ephemeral_id", "ecs.version"]
    - include_fields:
        fields: ["@timestamp", "message", "log.*", "kubernetes.*"]
```

## Integration with Service Mesh and API Gateway

### Istio Integration

When using service mesh like Istio, integrate with the ELK Stack:

```yaml
# Metricbeat configuration for Istio metrics
metricbeat.modules:
- module: prometheus
  period: 10s
  hosts: ["istio-telemetry.istio-system.svc.cluster.local:42422"]
  metrics_path: /metrics
  namespace: istio
```

## Real-World Example: Complete Monitoring Stack

This comprehensive example ties all components together:

1. **ECK Operator for managing Elasticsearch, Kibana and Beats**
2. **Metricbeat for infrastructure and application metrics**
3. **Filebeat for log collection**
4. **APM server and agents for distributed tracing**
5. **Heartbeat for uptime monitoring**
6. **Packetbeat for network monitoring**

This integrated stack provides:
- Complete visibility across all components
- Correlation between metrics, logs, and traces
- Unified querying and visualization in Kibana
- Centralized alerting and anomaly detection

## Troubleshooting Common Issues

### Common Elasticsearch Issues

1. **High Memory Usage**
   - Check JVM heap size (should be 50% of container memory)
   - Review shard allocation
   - Check for large queries or aggregations

2. **Slow Indexing**
   - Review refresh interval (increase for write-heavy workloads)
   - Check disk I/O (use SSD for production)
   - Consider disabling refresh during bulk indexing

### Metricbeat Issues

1. **High CPU Usage**
   - Increase collection interval (from 10s to 30s)
   - Reduce enabled modules
   - Filter unnecessary metrics

2. **Missing Kubernetes Metrics**
   - Check RBAC permissions
   - Verify access to kubelet API
   - Check network policies

## Best Practices and Recommendations

1. **Start small and scale**
   - Begin with core metrics and logs
   - Add APM and other components as needed

2. **Use ECK when possible**
   - Simplifies management
   - Handles upgrades and scaling
   - Manages security configurations

3. **Implement proper resource planning**
   - Size Elasticsearch properly (CPU, memory, disk)
   - Use autoscaling policies
   - Monitor resource usage trends

4. **Create actionable dashboards**
   - Focus on key metrics
   - Organize by user persona (developers, operators, business)
   - Include context and documentation

5. **Implement proper retention policies**
   - Match retention to data value
   - Consider compliance requirements
   - Use ILM for automated management

6. **Secure your stack**
   - Enable TLS for all components
   - Implement authentication and authorization
   - Follow least privilege principles

## Conclusion

The ELK Stack provides a powerful, flexible monitoring solution for Kubernetes environments. By integrating metrics, logs, and traces, it delivers comprehensive observability into your containerized applications and infrastructure. With proper setup, configuration, and tuning, the ELK Stack can scale to handle monitoring for clusters of any size while providing actionable insights through visualization and alerting.

## References

- [Elastic Cloud on Kubernetes Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)
- [Metricbeat Kubernetes Module](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-kubernetes.html)
- [APM Java Agent Documentation](https://www.elastic.co/guide/en/apm/agent/java/current/index.html)
- [Elasticsearch Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Kibana Dashboard Documentation](https://www.elastic.co/guide/en/kibana/current/dashboard.html)
- [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html)