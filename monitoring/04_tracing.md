# Distributed Tracing in Kubernetes with Elastic APM

## Introduction

Distributed tracing is an essential observability technique for understanding the behavior of applications in a microservices architecture. This guide explores implementing distributed tracing in Kubernetes environments using Elastic APM, as well as integration with other tracing systems like Jaeger and OpenTelemetry. We'll cover instrumentation strategies, data collection, analysis, and visualization of traces to provide deep insights into application performance and issues.

## Understanding Distributed Tracing

### Core Concepts

Distributed tracing follows a request as it traverses through multiple services in a distributed system:

1. **Trace**: A representation of a transaction or request flow through the entire system
2. **Span**: An operation or work unit within a trace, representing a single step in the process
3. **Context Propagation**: The mechanism to maintain trace context across service boundaries
4. **Service Map**: A visualization of service dependencies derived from trace data

### The W3C Trace Context Standard

The W3C Trace Context standard defines HTTP headers for propagating trace context across service boundaries:

- `traceparent`: Contains trace ID, parent span ID, trace flags (e.g., sampling)
- `tracestate`: Vendor-specific trace information

Example:
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b9c7c989f97918e1-01
tracestate: elastic=1
```

## Elastic APM Architecture in Kubernetes

### Components of Elastic APM

Elastic APM consists of several components working together:

1. **APM Agents**: Libraries integrated with applications to collect trace data
2. **APM Server**: Receives, processes, and sends trace data to Elasticsearch
3. **Elasticsearch**: Stores and indexes trace data
4. **Kibana APM UI**: Visualizes and analyzes trace data

### Deploying APM Server in Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apm-server
  namespace: elastic-system
  labels:
    app: apm-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apm-server
  template:
    metadata:
      labels:
        app: apm-server
    spec:
      containers:
      - name: apm-server
        image: docker.elastic.co/apm/apm-server:8.10.0
        ports:
        - containerPort: 8200
          name: http
        resources:
          limits:
            memory: 512Mi
            cpu: 500m
          requests:
            memory: 256Mi
            cpu: 100m
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "https://elasticsearch-es-http.elastic-system.svc:9200"
        - name: ELASTICSEARCH_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-es-elastic-user
              key: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-es-elastic-user
              key: elastic
        volumeMounts:
        - name: config
          mountPath: /usr/share/apm-server/apm-server.yml
          subPath: apm-server.yml
        - name: elastic-certificates
          mountPath: /usr/share/apm-server/config/certs
      volumes:
      - name: config
        configMap:
          name: apm-server-config
          defaultMode: 0600
      - name: elastic-certificates
        secret:
          secretName: elasticsearch-es-http-ca-internal
```

### APM Server ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apm-server-config
  namespace: elastic-system
data:
  apm-server.yml: |-
    apm-server:
      host: "0.0.0.0:8200"
      rum:
        enabled: true
      kibana:
        enabled: true
        host: "https://kibana-kb-http.elastic-system.svc:5601"
        username: "${ELASTICSEARCH_USERNAME}"
        password: "${ELASTICSEARCH_PASSWORD}"

    output.elasticsearch:
      hosts: ["${ELASTICSEARCH_HOSTS}"]
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl:
        certificate_authorities: ["/usr/share/apm-server/config/certs/ca.crt"]
        verification_mode: certificate

    logging.level: info
    logging.to_files: false
```

### APM Server Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apm-server
  namespace: elastic-system
  labels:
    app: apm-server
spec:
  ports:
  - port: 8200
    targetPort: 8200
    protocol: TCP
    name: http
  selector:
    app: apm-server
```

## Instrumenting Applications for Distributed Tracing

### Java Application with Elastic APM Agent

For a Java application deployed in Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-service
  namespace: application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-service
  template:
    metadata:
      labels:
        app: java-service
    spec:
      containers:
      - name: java-service
        image: my-java-service:latest
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server.elastic-system.svc:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "java-service"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        - name: ELASTIC_APM_APPLICATION_PACKAGES
          value: "com.example"
        - name: JAVA_TOOL_OPTIONS
          value: "-javaagent:/elastic-apm-agent.jar"
        volumeMounts:
        - name: elastic-apm-agent
          mountPath: /elastic-apm-agent.jar
      volumes:
      - name: elastic-apm-agent
        configMap:
          name: elastic-apm-agent
```

### Node.js Application with Elastic APM Agent

For a Node.js application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-service
  namespace: application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-service
  template:
    metadata:
      labels:
        app: nodejs-service
    spec:
      containers:
      - name: nodejs-service
        image: my-nodejs-service:latest
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server.elastic-system.svc:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "nodejs-service"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        - name: ELASTIC_APM_TRANSACTION_SAMPLE_RATE
          value: "1.0"
```

### Python Application with Elastic APM Agent

For a Python application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-service
  namespace: application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: python-service
  template:
    metadata:
      labels:
        app: python-service
    spec:
      containers:
      - name: python-service
        image: my-python-service:latest
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server.elastic-system.svc:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "python-service"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        - name: ELASTIC_APM_TRANSACTION_SAMPLE_RATE
          value: "1.0"
```

### .NET Application with Elastic APM Agent

For a .NET application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-service
  namespace: application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dotnet-service
  template:
    metadata:
      labels:
        app: dotnet-service
    spec:
      containers:
      - name: dotnet-service
        image: my-dotnet-service:latest
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server.elastic-system.svc:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "dotnet-service"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        - name: ELASTIC_APM_TRANSACTION_SAMPLE_RATE
          value: "1.0"
        - name: CORECLR_ENABLE_PROFILING
          value: "1"
        - name: CORECLR_PROFILER
          value: "{FA65FE15-F085-4681-9B20-95E04F6C03CC}"
        - name: CORECLR_PROFILER_PATH
          value: "/elastic-apm-agent/libelastic_apm_profiler.so"
        volumeMounts:
        - name: elastic-apm-agent
          mountPath: /elastic-apm-agent
      volumes:
      - name: elastic-apm-agent
        configMap:
          name: elastic-apm-dotnet-agent
```

## Service Mesh Integration with Tracing

Service meshes like Istio and Linkerd can automatically capture distributed trace information.

### Istio Tracing Configuration

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100
        zipkin:
          address: apm-server.elastic-system:8200
  components:
    pilot:
      k8s:
        env:
          - name: PILOT_TRACE_SAMPLING
            value: "100"
```

### Configure Envoy to Send Traces to Elastic APM

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: apm-tracing
  namespace: istio-system
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: ANY
      listener:
        filterChain:
          filter:
            name: "envoy.http_connection_manager"
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager"
          tracing:
            provider:
              name: envoy.zipkin
              typed_config:
                "@type": "type.googleapis.com/envoy.config.trace.v2.ZipkinConfig"
                collector_cluster: outbound|8200||apm-server.elastic-system.svc.cluster.local
                collector_endpoint: "/api/v1/spans"
                trace_id_128bit: true
                shared_span_context: false
```

## OpenTelemetry Integration

OpenTelemetry is an open standard for telemetry data, including distributed tracing.

### Deploy OpenTelemetry Collector

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-conf
  namespace: monitoring
data:
  otel-collector-config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 1000
        spike_limit_mib: 200

    exporters:
      zipkin:
        endpoint: "http://apm-server.elastic-system.svc:8200/api/v1/spans"
        format: json-v2

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [zipkin]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.72.0
        args:
          - "--config=/conf/otel-collector-config.yaml"
        ports:
        - containerPort: 4317 # OTLP gRPC
        - containerPort: 4318 # OTLP HTTP
        volumeMounts:
        - name: otel-collector-config-vol
          mountPath: /conf
      volumes:
        - name: otel-collector-config-vol
          configMap:
            name: otel-collector-conf
            items:
              - key: otel-collector-config
                path: otel-collector-config.yaml
```

### OpenTelemetry Collector Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  ports:
  - name: otlp-grpc
    port: 4317
    protocol: TCP
    targetPort: 4317
  - name: otlp-http
    port: 4318
    protocol: TCP
    targetPort: 4318
  selector:
    app: otel-collector
```

### Instrumenting an Application with OpenTelemetry

For a Java application using OpenTelemetry:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-otel-service
  namespace: application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-otel-service
  template:
    metadata:
      labels:
        app: java-otel-service
    spec:
      containers:
      - name: java-otel-service
        image: my-java-service:latest
        env:
        - name: OTEL_SERVICE_NAME
          value: "java-otel-service"
        - name: OTEL_TRACES_EXPORTER
          value: "otlp"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector.monitoring.svc:4317"
        - name: OTEL_PROPAGATORS
          value: "tracecontext,baggage,b3multi"
        - name: JAVA_TOOL_OPTIONS
          value: "-javaagent:/opentelemetry-javaagent.jar"
        volumeMounts:
        - name: otel-agent
          mountPath: /opentelemetry-javaagent.jar
      volumes:
      - name: otel-agent
        configMap:
          name: otel-java-agent
```

## Jaeger Integration with Elastic APM

You can set up Jaeger to send traces to Elastic APM, benefiting from both systems.

### Jaeger All-in-One Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.42
        ports:
        - containerPort: 16686 # UI
        - containerPort: 14268 # Collector HTTP
        - containerPort: 14250 # Collector gRPC
        env:
        - name: SPAN_STORAGE_TYPE
          value: "elasticsearch"
        - name: ES_SERVER_URLS
          value: "https://elasticsearch-es-http.elastic-system.svc:9200"
        - name: ES_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-es-elastic-user
              key: elastic
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-es-elastic-user
              key: elastic
        - name: ES_TLS_ENABLED
          value: "true"
        - name: ES_TLS_CA
          value: "/certificates/ca.crt"
        volumeMounts:
        - name: elastic-certificates
          mountPath: /certificates
      volumes:
      - name: elastic-certificates
        secret:
          secretName: elasticsearch-es-http-ca-internal
```

### Jaeger Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
  - port: 16686
    targetPort: 16686
    name: query-http
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
  - port: 14268
    targetPort: 14268
    name: collector-http
  - port: 14250
    targetPort: 14250
    name: collector-grpc
```

## Analyzing Trace Data

### Creating Visualizations in Kibana

1. **Service Maps**: Understand service dependencies and performance
2. **Trace Analytics**: Identify slow requests and errors
3. **Span Timeline**: View detailed span information
4. **Custom Dashboards**: Create tailored views of trace data

### Example Custom APM Dashboard in Kibana

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-apm-dashboard
  namespace: elastic-system
data:
  apm-dashboard.ndjson: |-
    {
      "type": "dashboard",
      "id": "apm-service-overview",
      "attributes": {
        "title": "APM Service Overview",
        "hits": 0,
        "description": "Overview of service performance from APM traces",
        "panelsJSON": "[{\"version\":\"7.10.0\",\"gridData\":{\"x\":0,\"y\":0,\"w\":24,\"h\":15,\"i\":\"1\"},\"panelIndex\":\"1\",\"embeddable\":{\"title\":\"Transaction Duration\",\"type\":\"lens\"},\"title\":\"Transaction Duration\"},{\"version\":\"7.10.0\",\"gridData\":{\"x\":24,\"y\":0,\"w\":24,\"h\":15,\"i\":\"2\"},\"panelIndex\":\"2\",\"embeddable\":{\"title\":\"Error Rate\",\"type\":\"lens\"},\"title\":\"Error Rate\"},{\"version\":\"7.10.0\",\"gridData\":{\"x\":0,\"y\":15,\"w\":48,\"h\":20,\"i\":\"3\"},\"panelIndex\":\"3\",\"embeddable\":{\"title\":\"Transaction Response Time Distribution\",\"type\":\"lens\"},\"title\":\"Transaction Response Time Distribution\"}]",
        "optionsJSON": "{\"hidePanelTitles\":false,\"useMargins\":true}",
        "version": 1,
        "timeRestore": false,
        "kibanaSavedObjectMeta": {
          "searchSourceJSON": "{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"
        }
      }
    }
```

## Setting Up Alerting for Trace Data

### Creating APM Transaction Duration Alert

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apm-alerts
  namespace: elastic-system
data:
  transaction-duration-alert.json: |-
    {
      "alert": {
        "name": "High Transaction Duration",
        "tags": ["APM", "Performance"],
        "alertTypeId": ".es-query",
        "consumer": "apm",
        "schedule": {
          "interval": "5m"
        },
        "enabled": true,
        "params": {
          "size": 100,
          "index": [
            "apm-*-transaction*"
          ],
          "timeField": "@timestamp",
          "esQuery": "{\"query\":{\"bool\":{\"filter\":[{\"range\":{\"transaction.duration.us\":{\"gt\":500000}}},{\"term\":{\"service.name\":\"critical-service\"}}]}}}",
          "thresholdComparator": ">",
          "threshold": [
            10
          ]
        },
        "actions": [
          {
            "id": "slack-connector",
            "group": "threshold met",
            "params": {
              "message": "High transaction durations detected in critical-service. {{context.hits.length}} transactions exceeded 500ms."
            }
          }
        ]
      }
    }
```

### Creating APM Error Rate Alert

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apm-error-alerts
  namespace: elastic-system
data:
  error-rate-alert.json: |-
    {
      "alert": {
        "name": "High Error Rate",
        "tags": ["APM", "Errors"],
        "alertTypeId": ".es-query",
        "consumer": "apm",
        "schedule": {
          "interval": "5m"
        },
        "enabled": true,
        "params": {
          "size": 100,
          "index": [
            "apm-*-error*"
          ],
          "timeField": "@timestamp",
          "esQuery": "{\"query\":{\"bool\":{\"filter\":[{\"term\":{\"service.name\":\"payment-service\"}}]}}}",
          "thresholdComparator": ">",
          "threshold": [
            5
          ]
        },
        "actions": [
          {
            "id": "slack-connector",
            "group": "threshold met",
            "params": {
              "message": "High error rate detected in payment-service. {{context.hits.length}} errors in the last 5 minutes."
            }
          }
        ]
      }
    }
```

## Advanced Trace Analysis

### Correlating Traces with Logs and Metrics

Correlating traces with logs and metrics provides a complete view of application behavior:

1. **Log Correlation**: View related logs for a specific trace
2. **Metric Correlation**: Connect trace performance with system metrics

Example Elasticsearch query to find logs related to a trace:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "trace.id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
          }
        },
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ]
    }
  }
}
```

### Root Cause Analysis Using Tracing

To identify the root cause of performance issues:

1. Find slow transactions in APM UI
2. Examine individual spans to locate bottlenecks
3. Analyze span attributes and context for additional information
4. Correlate with logs for specific error messages
5. Use service map to identify problematic dependencies

## Sampling Strategies

### Configuring Head-Based Sampling

Head-based sampling (deciding at the beginning of a trace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apm-server-config
  namespace: elastic-system
data:
  apm-server.yml: |-
    apm-server:
      sampling:
        keep_unsampled: false
        reservoir:
          size: 1000
          interval: "1s"
```

### Using Tail-Based Sampling with OpenTelemetry

Tail-based sampling (deciding after trace completion):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-tail-sampling
  namespace: monitoring
data:
  otel-collector-config: |-
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      tail_sampling:
        decision_wait: 5s
        num_traces: 100
        expected_new_traces_per_sec: 10
        policies:
          [
            {
              name: error-policy,
              type: status_code,
              status_code: {status_codes: [ERROR]}
            },
            {
              name: latency-policy,
              type: latency,
              latency: {threshold_ms: 500}
            },
            {
              name: rate-policy,
              type: probabilistic,
              probabilistic: {sampling_percentage: 10}
            }
          ]

    exporters:
      zipkin:
        endpoint: "http://apm-server.elastic-system.svc:8200/api/v1/spans"
        format: json-v2

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [tail_sampling]
          exporters: [zipkin]
```

## Custom Instrumentation

### Manual Span Creation in Java

Example of manually creating spans in a Java application:

```java
import co.elastic.apm.api.ElasticApm;
import co.elastic.apm.api.Span;

public class CustomService {
    public void performOperation() {
        Span span = ElasticApm.currentSpan()
            .startSpan("custom-operation", "custom", "operation");
        try {
            // Custom operation code
            span.addLabel("operation.type", "data-processing");
            span.addLabel("operation.size", "large");
            
            // Create a child span
            Span childSpan = span.startSpan("child-operation", "custom", "child");
            try {
                // Child operation code
            } finally {
                childSpan.end();
            }
        } finally {
            span.end();
        }
    }
}
```

### Custom Span Attributes in Node.js

Example of adding custom attributes in a Node.js application:

```javascript
const apm = require('elastic-apm-node');

async function processOrder(orderId) {
  const span = apm.startSpan('process-order', 'custom', 'order-processing');
  if (span) {
    span.setLabel('order.id', orderId);
    span.setLabel('order.priority', 'high');
    
    try {
      // Process order code
      return await orderService.process(orderId);
    } catch (error) {
      span.setLabel('error.message', error.message);
      throw error;
    } finally {
      span.end();
    }
  } else {
    // No active transaction
    return await orderService.process(orderId);
  }
}
```

## Real User Monitoring (RUM)

### Configuring RUM for Elastic APM

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apm-server-rum-config
  namespace: elastic-system
data:
  apm-server.yml: |-
    apm-server:
      host: "0.0.0.0:8200"
      rum:
        enabled: true
        event_rate:
          limit: 300
          lru_size: 1000
        allow_origins: ['*']
        library_pattern: "node_modules|bower_components|~\\/vendor"
        exclude_from_grouping: "^/webpack"
        source_mapping:
          enabled: true
          cache:
            expiration: 5m
          index_pattern: "apm-*-sourcemap*"
```

### JavaScript RUM Agent Integration

Add the RUM agent to your web application:

```html
<script>
  elasticApm.init({
    serviceName: 'frontend-app',
    serverUrl: 'https://apm-server.example.com:8200',
    environment: 'production',
    distributedTracing: true,
    distributedTracingOrigins: ['https://api.example.com'],
    transactionSampleRate: 0.5
  })
</script>
```

## Troubleshooting Tracing Issues

### Common Problems and Solutions

1. **Missing Traces**
   - Verify agent configuration
   - Check network connectivity to APM Server
   - Ensure correct sampling rate

2. **Incomplete Traces**
   - Check for missing context propagation
   - Verify all services are instrumented
   - Look for timeouts in service calls

3. **High Cardinality**
   - Limit custom attributes
   - Use structured attribute naming
   - Apply appropriate sampling rate

### Debugging APM Agent Issues

```bash
# Enable debug logging for Java agent
-Delastic.apm.log_level=DEBUG

# Enable debug logging for Node.js agent
ELASTIC_APM_LOG_LEVEL=debug

# Enable debug logging for Python agent
ELASTIC_APM_LOG_LEVEL=debug

# Check APM Server logs
kubectl logs -n elastic-system -l app=apm-server
```

## Best Practices for Distributed Tracing

1. **Consistent Service Naming**
   - Use clear, consistent service names
   - Include environment information
   - Consider hierarchical naming for microservices

2. **Appropriate Sampling**
   - Use higher rates in development
   - Lower rates in production (1-10%)
   - Consider differential sampling based on endpoint importance

3. **Context Propagation**
   - Use standard headers (W3C Trace Context)
   - Propagate context across all communication channels
   - Preserve baggage items for business context

4. **Performance Considerations**
   - Monitor agent overhead
   - Adjust batch sizes and flush intervals
   - Limit span attributes in high-throughput services

5. **Security and Privacy**
   - Avoid logging sensitive data in spans
   - Apply appropriate access controls to trace data
   - Consider PII implications of trace data

## Conclusion

Distributed tracing with Elastic APM provides deep insights into the behavior and performance of applications in Kubernetes environments. By implementing proper instrumentation, effective sampling strategies, and integrating with other observability tools, you can create a comprehensive view of your system's behavior. This enables faster troubleshooting, more informed optimization, and ultimately more reliable applications.

## References

- [Elastic APM Documentation](https://www.elastic.co/guide/en/apm/guide/current/index.html)
- [Elastic APM Agents](https://www.elastic.co/guide/en/apm/agent/index.html)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/latest/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)