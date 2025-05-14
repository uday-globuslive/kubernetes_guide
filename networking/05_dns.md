# DNS in Kubernetes

## Introduction to DNS in Kubernetes

DNS (Domain Name System) is a critical component in Kubernetes that enables service discovery and communication between applications. It allows pods to find and connect to services using human-readable names instead of IP addresses. This is particularly important for dynamic environments like Kubernetes, where pods and services can be created, destroyed, or scaled at any time, resulting in constantly changing IP addresses.

## CoreDNS Overview

CoreDNS is the default DNS server for Kubernetes since version 1.13. It is a flexible, extensible DNS server that can serve as the Kubernetes cluster DNS. CoreDNS functions as a DNS server and also provides additional features through plugins.

### Key Features of CoreDNS

- **Modularity**: Functionality is implemented through plugins
- **Flexibility**: Configurable through a simple text file (Corefile)
- **High Performance**: Designed for speed and efficiency
- **Integration**: Native integration with Kubernetes

## How DNS Works in Kubernetes

### DNS Service Discovery Flow

1. A pod is created and assigned a hostname and an IP address
2. Kubernetes creates DNS records for services and pods
3. Pods query the cluster DNS server (CoreDNS)
4. CoreDNS resolves the name to an IP address
5. The pod connects to the service or pod using the IP address

### DNS Records in Kubernetes

Kubernetes DNS creates several types of records:

#### 1. Services

For a service named `elasticsearch` in namespace `elastic-system`:

- **A/AAAA record**: `elasticsearch.elastic-system.svc.cluster.local` → Service IP
- **SRV records**: `_elasticsearch._tcp.elastic-system.svc.cluster.local` → Service port information

#### 2. Pods

For a pod with IP `10.244.1.10` in namespace `elastic-system`:

- **A/AAAA record**: `10-244-1-10.elastic-system.pod.cluster.local` → Pod IP

### DNS Names Format

The fully qualified domain name (FQDN) follows this pattern:

```
<service-name>.<namespace>.svc.cluster.local
```

For example:
- `elasticsearch-client.elastic-system.svc.cluster.local`
- `kibana.elastic-system.svc.cluster.local`
- `logstash.elastic-system.svc.cluster.local`

## DNS Configuration for ELK Stack

### Elasticsearch Service DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-client
  namespace: elastic-system
  labels:
    app: elasticsearch
    role: client
spec:
  selector:
    app: elasticsearch
    role: client
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  - name: transport
    port: 9300
    targetPort: 9300
```

With this service, other pods can connect to Elasticsearch using:
- Within the same namespace: `elasticsearch-client`
- From other namespaces: `elasticsearch-client.elastic-system.svc.cluster.local`

### Kibana Service DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elastic-system
  labels:
    app: kibana
spec:
  selector:
    app: kibana
  ports:
  - name: http
    port: 5601
    targetPort: 5601
```

Other pods can connect to Kibana using:
- Within the same namespace: `kibana`
- From other namespaces: `kibana.elastic-system.svc.cluster.local`

### Logstash Service DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elastic-system
  labels:
    app: logstash
spec:
  selector:
    app: logstash
  ports:
  - name: beats
    port: 5044
    targetPort: 5044
  - name: http
    port: 8080
    targetPort: 8080
```

### Connecting ELK Components Using DNS

#### Kibana to Elasticsearch

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elastic-system
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0.0.0.0"
    elasticsearch.hosts:
      - http://elasticsearch-client:9200
    # For cross-namespace, use FQDN:
    # elasticsearch.hosts:
    #   - http://elasticsearch-client.elastic-system.svc.cluster.local:9200
```

#### Logstash to Elasticsearch

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elastic-system
data:
  logstash.yml: |
    http.host: "0.0.0.0"
  logstash.conf: |
    output {
      elasticsearch {
        hosts => ["http://elasticsearch-client:9200"]
        # For cross-namespace, use FQDN:
        # hosts => ["http://elasticsearch-client.elastic-system.svc.cluster.local:9200"]
        index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
      }
    }
```

#### Filebeat to Logstash

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
    output.logstash:
      hosts: ["logstash.elastic-system.svc.cluster.local:5044"]
```

## DNS for Multi-Cluster ELK Stack

For larger deployments, you might have multiple Elasticsearch clusters that need to communicate with each other. This can be facilitated using Kubernetes DNS and proper configuration.

### Cross-Cluster Search

First, ensure both clusters can reach each other. Then configure cross-cluster search using the DNS names:

```yaml
POST _cluster/settings
{
  "persistent": {
    "cluster.remote.production-cluster.seeds": [
      "elasticsearch-master.production.svc.cluster.local:9300"
    ],
    "cluster.remote.staging-cluster.seeds": [
      "elasticsearch-master.staging.svc.cluster.local:9300"
    ]
  }
}
```

## Pod DNS Configuration

### Pod DNS Policy

The `dnsPolicy` field in the Pod spec determines how DNS resolution works for that Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-client
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
  dnsPolicy: ClusterFirst
```

Available policies:

- **ClusterFirst** (default): Queries that don't match the cluster domain are forwarded to upstream nameserver
- **Default**: Pod inherits the name resolution from the node
- **None**: All DNS settings are provided in the `dnsConfig` field
- **ClusterFirstWithHostNet**: For pods running with hostNetwork

### Custom DNS Settings

You can override specific DNS settings using the `dnsConfig` field:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-custom-dns
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 10.254.0.10
    searches:
      - elastic-system.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "5"
      - name: timeout
        value: "3"
```

This configuration can be useful for:
- Setting custom DNS servers
- Adding domain searches
- Configuring DNS options

## Common DNS Issues in ELK Stack Deployments

### 1. Resolution Failures

**Symptoms:**
- Unable to connect to Elasticsearch from Kibana
- Logstash can't send data to Elasticsearch
- Error logs showing "unknown host"

**Possible Causes:**
- CoreDNS pods not running properly
- Network policies blocking DNS traffic
- Misconfigured DNS policy or settings

**Troubleshooting:**
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from a pod
kubectl run dnsutils --image=gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 --rm -it -- nslookup elasticsearch-client.elastic-system.svc.cluster.local
```

### 2. Slow DNS Resolution

**Symptoms:**
- Elasticsearch cluster formation is slow
- Kibana takes a long time to connect to Elasticsearch
- General slowness in service communication

**Possible Causes:**
- Insufficient CoreDNS resources
- High DNS query volume
- DNS caching issues

**Solutions:**
- Scale CoreDNS deployment:
  ```bash
  kubectl scale deployment coredns -n kube-system --replicas=3
  ```
- Optimize CoreDNS with appropriate caching:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: coredns
    namespace: kube-system
  data:
    Corefile: |
      .:53 {
          errors
          health {
             lameduck 5s
          }
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
             pods insecure
             fallthrough in-addr.arpa ip6.arpa
             ttl 30
          }
          cache 30
          prometheus :9153
          forward . /etc/resolv.conf
          loop
          reload
          loadbalance
      }
  ```

## Advanced DNS Configurations for ELK Stack

### 1. Split DNS for Cross-Environment Communication

For communicating between development and production ELK stacks:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    prod.elastic.example.com:53 {
      forward . 10.0.0.10
    }
    dev.elastic.example.com:53 {
      forward . 10.0.1.10
    }
```

### 2. External DNS Integration

For automatically creating external DNS records for your ELK services:

```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: elasticsearch-external
spec:
  endpoints:
  - dnsName: elasticsearch.example.com
    recordType: A
    targets:
    - 192.168.1.10
  - dnsName: kibana.example.com
    recordType: A
    targets:
    - 192.168.1.11
```

### 3. DNS-Based Service Mesh Discovery

If using a service mesh like Istio, you can configure DNS for cross-mesh discovery:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-elasticsearch
spec:
  hosts:
  - es-central.example.com
  location: MESH_EXTERNAL
  ports:
  - number: 9200
    name: http
    protocol: HTTP
  resolution: DNS
```

## DNS Monitoring for ELK Stack

### Monitoring CoreDNS with Metricbeat

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
spec:
  type: metricbeat
  version: 7.15.0
  config:
    metricbeat.modules:
    - module: prometheus
      period: 10s
      hosts: ["${COREDNS_SERVICE_HOST}:9153"]
      metrics_path: /metrics
```

### Using Elastic APM for DNS-Related Issues

Configure your applications to use Elastic APM for tracking DNS resolution times:

```java
import co.elastic.apm.api.ElasticApm;
import co.elastic.apm.api.Span;

public class ElasticsearchClient {
    public void connect(String hostname) {
        Span span = ElasticApm.currentTransaction().startSpan();
        span.setName("DNS resolution");
        span.setType("dns");
        try {
            // DNS resolution happens here
            InetAddress.getByName(hostname);
        } finally {
            span.end();
        }
    }
}
```

### DNS Query Logging

Enable query logging in CoreDNS for debugging:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log
        errors
        # other plugins...
    }
```

## Best Practices for DNS in ELK Stack Deployments

### 1. Use Short DNS Names Within Namespaces

When services are in the same namespace, use short names:

```yaml
elasticsearch.hosts: ["http://elasticsearch-client:9200"]
```

This reduces DNS queries and improves resolution time.

### 2. Use FQDNs for Cross-Namespace Communication

For services in different namespaces, always use fully qualified domain names:

```yaml
elasticsearch.hosts: ["http://elasticsearch-client.elastic-system.svc.cluster.local:9200"]
```

### 3. Configure Appropriate TTLs

For stable services like Elasticsearch master nodes, increase TTL to reduce DNS queries:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 60
        }
        # other plugins...
    }
```

### 4. Scale CoreDNS According to Cluster Size

For large ELK deployments with many pods, scale CoreDNS appropriately:

```bash
kubectl scale deployment coredns -n kube-system --replicas=4
```

### 5. Use Headless Services for Direct Pod Communication

For Elasticsearch node-to-node communication, use headless services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-discovery
  namespace: elastic-system
spec:
  clusterIP: None
  selector:
    app: elasticsearch
    role: master
  ports:
  - port: 9300
    name: transport
```

### 6. Configure DNS Probes

Add liveness and readiness probes based on DNS resolution:

```yaml
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - nslookup elasticsearch-client.elastic-system.svc.cluster.local
  initialDelaySeconds: 10
  periodSeconds: 20
```

## Conclusion

DNS in Kubernetes is a fundamental service that enables reliable communication between components of the ELK Stack. By understanding how DNS works, configuring it properly, and following best practices, you can ensure that Elasticsearch, Logstash, Kibana, and Beats components can discover and communicate with each other efficiently. This is especially critical in dynamic environments where pods and services are constantly being created, updated, and destroyed.

Proper DNS configuration contributes to the overall resilience and performance of your ELK Stack deployment on Kubernetes, enabling smooth operation even as the deployment scales or changes over time.