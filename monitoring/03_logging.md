# Logging in Kubernetes with ELK Stack

## Introduction

Logging is a critical aspect of operating applications in Kubernetes environments. This guide covers comprehensive approaches to collecting, processing, and analyzing logs from Kubernetes clusters using the Elastic Stack. We'll explore various deployment patterns, configuration options, and best practices to implement an effective logging solution that provides meaningful insights into your containerized applications.

## Kubernetes Logging Architecture

### Understanding Kubernetes Log Sources

When dealing with Kubernetes environments, logs come from various sources:

1. **Container Logs**: Application logs from containerized applications
2. **Kubernetes Control Plane Logs**: API server, scheduler, controller-manager
3. **Node Logs**: kubelet, container runtime, system services
4. **Audit Logs**: Records of API server requests for security and compliance

### Default Kubernetes Logging Mechanism

Kubernetes has a basic logging system:

- Container stdout/stderr are captured by the container runtime
- kubelet redirects these logs to files on nodes in `/var/log/containers/`
- `kubectl logs` provides basic access to these files

However, this basic mechanism has limitations:
- No log aggregation across nodes
- No log rotation (except by kubelet, which is limited)
- No search or advanced analysis capabilities
- Logs are lost when pods are deleted

## ELK Stack Architecture for Kubernetes Logging

A typical ELK Stack deployment for Kubernetes logging consists of:

1. **Log Collection**: Filebeat, Fluent Bit, or Fluentd deployed as DaemonSets
2. **Log Processing**: Logstash for parsing, enrichment, and transformation
3. **Log Storage**: Elasticsearch for efficient storage and search
4. **Visualization**: Kibana for dashboards and exploration

### Sample Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Node 1    │     │   Node 2    │     │   Node 3    │
│  ┌───────┐  │     │  ┌───────┐  │     │  ┌───────┐  │
│  │Pod A  │  │     │  │Pod C  │  │     │  │Pod E  │  │
│  └───────┘  │     │  └───────┘  │     │  └───────┘  │
│  ┌───────┐  │     │  ┌───────┐  │     │  ┌───────┐  │
│  │Pod B  │  │     │  │Pod D  │  │     │  │Pod F  │  │
│  └───────┘  │     │  └───────┘  │     │  └───────┘  │
│      ▼      │     │      ▼      │     │      ▼      │
│  ┌───────┐  │     │  ┌───────┐  │     │  ┌───────┐  │
│  │Filebeat│  │     │  │Filebeat│  │     │  │Filebeat│  │
│  └───┬───┘  │     │  └───┬───┘  │     │  └───┬───┘  │
└──────┼──────┘     └──────┼──────┘     └──────┼──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────────────────────────────────────────┐
│                    Logstash                      │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                 Elasticsearch                    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                     Kibana                       │
└──────────────────────────────────────────────────┘
```

## Deploying Log Collectors in Kubernetes

### Filebeat DaemonSet Configuration

Filebeat is lightweight and efficient for collecting logs from Kubernetes nodes.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.10.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-client.logging.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        securityContext:
          runAsUser: 0
          # For Filebeat to read from /var/log and /var/lib/docker/containers
          privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
```

#### Filebeat Configuration ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    processors:
      - add_cloud_metadata: {}
      - add_host_metadata: {}
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
      - drop_event:
          when:
            and:
              - contains:
                  kubernetes.namespace: "kube-system"
              - not:
                  contains:
                    message: "error"

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}']
      index: "kubernetes-%{[agent.version]}-%{+yyyy.MM.dd}"
```

### Fluent Bit Configuration

Fluent Bit is a lighter alternative to Fluentd with a smaller memory footprint.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:1.9.7
        volumeMounts:
        - name: config-volume
          mountPath: /fluent-bit/etc/
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        resources:
          limits:
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 64Mi
      volumes:
      - name: config-volume
        configMap:
          name: fluent-bit-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### Fluent Bit ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |-
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch-client.logging.svc.cluster.local
        Port            9200
        Index           kubernetes_cluster
        Type            _doc
        Logstash_Format On
        Logstash_Prefix kubernetes
        Time_Key        @timestamp
        Replace_Dots    On
        Retry_Limit     False

  parsers.conf: |-
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On
```

## Setting Up Log Processing with Logstash

Logstash adds powerful processing capabilities to the logging pipeline.

### Logstash Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.10.0
        resources:
          limits:
            memory: 1Gi
            cpu: 1000m
          requests:
            cpu: 100m
            memory: 512Mi
        ports:
        - containerPort: 5044
          name: filebeat
        - containerPort: 9600
          name: monitoring
        env:
        - name: LS_JAVA_OPTS
          value: "-Xmx512m -Xms512m"
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch-client.logging.svc.cluster.local:9200
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/logstash/pipeline
        - name: logstash-config-volume
          mountPath: /usr/share/logstash/config/logstash.yml
          subPath: logstash.yml
      volumes:
      - name: config-volume
        configMap:
          name: logstash-pipeline
      - name: logstash-config-volume
        configMap:
          name: logstash-config
```

### Logstash Pipeline Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipeline
  namespace: logging
data:
  logstash.conf: |-
    input {
      beats {
        port => 5044
      }
    }

    filter {
      if [kubernetes] {
        mutate {
          add_field => { "[@metadata][index]" => "kubernetes-%{+YYYY.MM.dd}" }
        }

        # Parse JSON log messages for specific applications
        if [kubernetes][container_name] == "my-app" {
          json {
            source => "message"
            target => "parsed_json"
            skip_on_invalid_json => true
          }
          if "_jsonparsefailure" in [tags] {
            mutate {
              remove_tag => ["_jsonparsefailure"]
            }
          } else {
            mutate {
              rename => { "[parsed_json][log_level]" => "log_level" }
              rename => { "[parsed_json][message]" => "log_message" }
              rename => { "[parsed_json][timestamp]" => "app_timestamp" }
              remove_field => ["message", "parsed_json"]
            }
          }
        }

        # Extract important fields from system logs
        if [kubernetes][namespace] == "kube-system" {
          grok {
            match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log_level} %{GREEDYDATA:log_message}" }
            overwrite => [ "message" ]
          }
          date {
            match => [ "timestamp", "ISO8601" ]
            target => "@timestamp"
            remove_field => [ "timestamp" ]
          }
        }

        # Add severity field for easier filtering
        if [log_level] in ["ERROR", "FATAL", "error", "fatal"] {
          mutate {
            add_field => { "severity" => "error" }
          }
        } else if [log_level] in ["WARN", "WARNING", "warn", "warning"] {
          mutate {
            add_field => { "severity" => "warning" }
          }
        } else if [log_level] in ["INFO", "info"] {
          mutate {
            add_field => { "severity" => "info" }
          }
        } else if [log_level] in ["DEBUG", "debug", "TRACE", "trace"] {
          mutate {
            add_field => { "severity" => "debug" }
          }
        } else {
          mutate {
            add_field => { "severity" => "unknown" }
          }
        }
      }
    }

    output {
      elasticsearch {
        hosts => ["${ELASTICSEARCH_HOSTS}"]
        index => "%{[@metadata][index]}"
        manage_template => false
      }
    }
```

### Logstash Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: logging
data:
  logstash.yml: |-
    http.host: "0.0.0.0"
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch-client.logging.svc.cluster.local:9200"]
    queue.type: persisted
    queue.max_bytes: 1024mb
    pipeline.workers: 2
    pipeline.batch.size: 125
    pipeline.batch.delay: 5
```

### Logstash Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: logging
  labels:
    app: logstash
spec:
  ports:
  - port: 5044
    targetPort: 5044
    protocol: TCP
    name: filebeat
  - port: 9600
    targetPort: 9600
    protocol: TCP
    name: monitoring
  selector:
    app: logstash
```

## Deploying Elasticsearch for Log Storage

### Elasticsearch Statefulset

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: logging
spec:
  serviceName: elasticsearch-data
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
      role: data
  template:
    metadata:
      labels:
        app: elasticsearch
        role: data
    spec:
      containers:
      - name: elasticsearch-data
        image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: cluster.name
          value: k8s-logs
        - name: discovery.seed_hosts
          value: elasticsearch-master.$(NAMESPACE).svc.cluster.local
        - name: node.roles
          value: data
        - name: ES_JAVA_OPTS
          value: -Xms1g -Xmx1g
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 1Gi
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 50Gi
```

## Configuring Kibana for Log Visualization

### Kibana Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.10.0
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch-client.logging.svc.cluster.local:9200
        ports:
        - containerPort: 5601
          name: http
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 60
          timeoutSeconds: 30
```

### Kibana Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    targetPort: 5601
    protocol: TCP
    name: http
  selector:
    app: kibana
```

### Kibana Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
```

## Index Patterns and Dashboards

### Creating Index Patterns in Kibana

Once logs are flowing into Elasticsearch, you need to create Index Patterns in Kibana:

1. Navigate to Kibana → Management → Stack Management → Index Patterns
2. Create a new pattern matching your indices (e.g., `kubernetes-*`)
3. Set the Time Filter field to `@timestamp`

### Sample Dashboard Configuration

You can create pre-configured dashboards in Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-dashboards
  namespace: logging
data:
  k8s-logs-dashboard.ndjson: |-
    {
      "type": "dashboard",
      "attributes": {
        "title": "Kubernetes Logs Overview",
        "hits": 0,
        "description": "Overview of Kubernetes logs",
        "panelsJSON": "[{\"version\":\"7.10.0\",\"gridData\":{\"x\":0,\"y\":0,\"w\":24,\"h\":15,\"i\":\"1\"},\"panelIndex\":\"1\",\"embeddable\":{\"title\":\"Log Counts by Namespace\",\"type\":\"visualization\"},\"title\":\"Log Counts by Namespace\"},{\"version\":\"7.10.0\",\"gridData\":{\"x\":24,\"y\":0,\"w\":24,\"h\":15,\"i\":\"2\"},\"panelIndex\":\"2\",\"embeddable\":{\"title\":\"Log Levels Distribution\",\"type\":\"visualization\"},\"title\":\"Log Levels Distribution\"},{\"version\":\"7.10.0\",\"gridData\":{\"x\":0,\"y\":15,\"w\":48,\"h\":33,\"i\":\"3\"},\"panelIndex\":\"3\",\"embeddable\":{\"title\":\"Recent Logs\",\"type\":\"search\"},\"title\":\"Recent Logs\"}]",
        "optionsJSON": "{\"hidePanelTitles\":false,\"useMargins\":true}",
        "version": 1,
        "timeRestore": false,
        "kibanaSavedObjectMeta": {
          "searchSourceJSON": "{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"
        }
      }
    }
```

## Advanced Logging Patterns

### Structured Logging

Encourage applications to use structured logging formats (especially JSON):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: app
data:
  log-config.xml: |-
    <configuration>
      <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
          <includeContext>false</includeContext>
          <customFields>{"app":"my-application","env":"${ENV}"}</customFields>
        </encoder>
      </appender>
      <root level="INFO">
        <appender-ref ref="STDOUT" />
      </root>
    </configuration>
```

### Multi-line Log Handling

Configure Filebeat to properly handle multi-line log entries (like stack traces):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
      multiline:
        pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
        negate: true
        match: after
```

### Log Retention and Lifecycle Management

Use Index Lifecycle Management (ILM) to manage log retention:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-ilm
  namespace: logging
data:
  ilm-policy.json: |-
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
            "min_age": "7d",
            "actions": {
              "set_priority": {
                "priority": 0
              }
            }
          },
          "delete": {
            "min_age": "30d",
            "actions": {
              "delete": {}
            }
          }
        }
      }
    }
```

## Security Considerations

### Network Policies

Restrict network access to your logging components:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: logging-network-policy
  namespace: logging
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: logstash
    ports:
    - protocol: TCP
      port: 9200
  - from:
    - podSelector:
        matchLabels:
          app: kibana
    ports:
    - protocol: TCP
      port: 9200
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9300
```

### TLS for Secure Communication

Secure communication between logging components:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-certificates
  namespace: logging
type: Opaque
data:
  elasticsearch.keystore.p12: <base64-encoded-keystore>
  elasticsearch.truststore.p12: <base64-encoded-truststore>
```

### User Authentication

Set up authentication for Kibana and Elasticsearch:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-credentials
  namespace: logging
type: Opaque
data:
  username: ZWxhc3RpYw==  # 'elastic' base64 encoded
  password: Y2hhbmdlbWU=  # 'changeme' base64 encoded
```

## Monitoring the Logging Stack

### Filebeat for Monitoring Elasticsearch

Monitor your logging stack itself:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-monitoring-config
  namespace: logging
data:
  filebeat.yml: |-
    filebeat.modules:
    - module: elasticsearch
      server:
        enabled: true
      gc:
        enabled: true
      audit:
        enabled: true
      slowlog:
        enabled: true
    
    - module: logstash
      log:
        enabled: true
      slowlog:
        enabled: true
    
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOSTS}']
      index: "filebeat-monitoring-%{+yyyy.MM.dd}"
```

### Metricbeat for Logging Stack Metrics

Collect metrics from your logging components:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-monitoring-config
  namespace: logging
data:
  metricbeat.yml: |-
    metricbeat.modules:
    - module: elasticsearch
      metricsets:
        - node
        - node_stats
        - index
        - index_recovery
        - index_summary
        - shard
        - ml_job
      period: 10s
      hosts: ["http://elasticsearch-client:9200"]
    
    - module: logstash
      metricsets:
        - node
        - node_stats
      period: 10s
      hosts: ["http://logstash:9600"]
    
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOSTS}']
      index: "metricbeat-monitoring-%{+yyyy.MM.dd}"
```

## Troubleshooting

### Common Issues and Solutions

1. **No logs appearing in Elasticsearch**
   - Check Filebeat/Fluent Bit pod logs
   - Verify network connectivity to Logstash/Elasticsearch
   - Check resource constraints on logging components

2. **Missing Kubernetes metadata**
   - Verify RBAC permissions
   - Check Kubernetes API connectivity

3. **High resource usage**
   - Adjust batch sizes and flush intervals
   - Implement filters to reduce log volume
   - Scale Elasticsearch cluster

### Debugging Commands

```bash
# Check Filebeat status
kubectl exec -it -n logging $(kubectl get pods -n logging -l app=filebeat -o name | head -n 1) -- filebeat test config -c /etc/filebeat.yml

# Verify Elasticsearch cluster health
kubectl exec -it -n logging $(kubectl get pods -n logging -l app=elasticsearch,role=client -o name | head -n 1) -- curl http://localhost:9200/_cluster/health?pretty

# Follow Logstash logs
kubectl logs -f -n logging $(kubectl get pods -n logging -l app=logstash -o name | head -n 1)
```

## Best Practices for Kubernetes Logging

1. **Use Structured Logging**
   - Encourage JSON log formats
   - Include correlation IDs across microservices
   - Include relevant metadata (service name, version, etc.)

2. **Implement Log Levels Properly**
   - DEBUG for detailed troubleshooting
   - INFO for normal operations
   - WARN for potential issues
   - ERROR for failures
   - Follow a consistent standard across all applications

3. **Handle Volume Appropriately**
   - Filter out noisy logs
   - Sample high-volume logs
   - Implement rate limiting in production
   - Consider separate indices for high-volume services

4. **Resource Planning**
   - Size Elasticsearch cluster appropriately
   - Monitor storage usage and growth
   - Implement ILM for log retention
   - Consider hot-warm-cold architecture for large deployments

5. **Security Best Practices**
   - Encrypt data in transit with TLS
   - Implement role-based access control
   - Anonymize sensitive data before logging
   - Regular auditing of access to logs

## Scaling Considerations

### Horizontal Scaling

For larger clusters, scale out your logging infrastructure:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: logging
spec:
  replicas: 5  # Increase replicas
```

### Vertical Scaling

Allocate more resources to components as needed:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: logging
spec:
  template:
    spec:
      containers:
      - name: elasticsearch-data
        resources:
          limits:
            cpu: 2000m    # Increased CPU
            memory: 4Gi   # Increased memory
          requests:
            cpu: 1000m
            memory: 2Gi
        env:
        - name: ES_JAVA_OPTS
          value: -Xms2g -Xmx2g  # Increased heap size
```

### Sharding Strategy

Optimize Elasticsearch performance with proper sharding:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-template
  namespace: logging
data:
  index-template.json: |-
    {
      "index_patterns": ["kubernetes-*"],
      "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "refresh_interval": "5s"
      },
      "mappings": {
        "properties": {
          "@timestamp": { "type": "date" },
          "kubernetes": {
            "properties": {
              "namespace_name": { "type": "keyword" },
              "pod_name": { "type": "keyword" },
              "container_name": { "type": "keyword" }
            }
          },
          "log_level": { "type": "keyword" },
          "message": { "type": "text" }
        }
      }
    }
```

## Integrating with Log Analytics

### Setting Up Alerting

Create alerts for critical log events:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-alerting
  namespace: logging
data:
  error-alert.json: |-
    {
      "trigger": {
        "schedule": {
          "interval": "1m"
        }
      },
      "input": {
        "search": {
          "request": {
            "search_type": "query_then_fetch",
            "indices": ["kubernetes-*"],
            "body": {
              "size": 0,
              "query": {
                "bool": {
                  "must": [
                    {
                      "match": {
                        "severity": "error"
                      }
                    },
                    {
                      "range": {
                        "@timestamp": {
                          "gte": "now-5m"
                        }
                      }
                    }
                  ]
                }
              }
            }
          }
        }
      },
      "condition": {
        "compare": {
          "ctx.payload.hits.total": {
            "gt": 10
          }
        }
      },
      "actions": {
        "send_email": {
          "email": {
            "to": "ops@example.com",
            "subject": "High number of errors detected",
            "body": "More than 10 errors detected in the last 5 minutes"
          }
        }
      }
    }
```

### Log Anomaly Detection with Machine Learning

Set up Machine Learning jobs for anomaly detection:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-ml
  namespace: logging
data:
  api-error-detection.json: |-
    {
      "description": "Detect unusual error rates in API service",
      "analysis_config": {
        "bucket_span": "15m",
        "detectors": [
          {
            "detector_description": "count(severity=error) partitionfield=kubernetes.namespace_name",
            "function": "count",
            "by_field_name": "kubernetes.namespace_name",
            "custom_rules": [
              {
                "actions": ["skip_result"],
                "conditions": [
                  {"applies_to": "actual", "operator": "lt", "value": 5}
                ]
              }
            ]
          }
        ],
        "influencers": [
          "kubernetes.namespace_name",
          "kubernetes.pod_name"
        ]
      },
      "data_description": {
        "time_field": "@timestamp",
        "time_format": "epoch_ms"
      },
      "model_plot_config": {
        "enabled": true
      }
    }
```

## Conclusion

Implementing a robust logging solution with the ELK Stack in Kubernetes provides valuable insights into the behavior and health of your applications. By following the deployment patterns, configuration options, and best practices outlined in this guide, you can build a scalable, secure, and effective logging infrastructure that enhances your ability to monitor, troubleshoot, and analyze your Kubernetes workloads.

## References

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Filebeat Documentation](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Logstash Documentation](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Fluent Bit Documentation](https://docs.fluentbit.io/manual/)
- [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Elastic Cloud on Kubernetes](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)