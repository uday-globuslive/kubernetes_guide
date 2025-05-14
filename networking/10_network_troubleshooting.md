# Advanced Network Troubleshooting in Kubernetes

## Introduction

Kubernetes networking presents unique challenges due to its multi-layered architecture, involving container networking interfaces (CNIs), service discovery, load balancing, and network policies. When network issues arise, they can be difficult to diagnose and resolve due to the complex interactions between these components. This guide presents a comprehensive approach to troubleshooting Kubernetes networking problems, from common issues to advanced diagnostic techniques.

## Understanding Kubernetes Network Architecture

Before diving into troubleshooting, it's essential to understand the Kubernetes networking model:

### Kubernetes Networking Layers

1. **Pod Networking**: Communication between containers within a pod and between pods
2. **Service Networking**: Service discovery and load balancing
3. **Ingress/Egress**: External traffic entering and leaving the cluster
4. **Network Policies**: Security rules that control pod communication
5. **DNS**: Service discovery via cluster DNS

### Key Components in the Network Path

A typical network request in Kubernetes traverses multiple components:

```
External Client → Ingress Controller → Service → Endpoints → Pod → Container
```

Each layer can introduce potential issues:

![Kubernetes Network Flow](https://mermaid.ink/img/pako:eNp1kc9uwjAMxl8l8qkgHnDdsaduOwASJ04pB5OYNqJpU-UPQojw7k1bChPb8Pf5-2zFPkKpCSKs1TZQQtg4VlXyTt97R6IZKUljbVq6yCaFqGptVSoaeWJiyKjCrfZXLrQmZatQO8PaPpLc_hrB0WhvRqOQR--_OGJDnUpRBhvez8PLtvH6KXJeRcz4f2UjYvL-k1SJpTlwspzkq0ZlNIZMvEjtJGrBPB3MWaMa64T_bsrGB67BA-mcyjWKCdmYAwjOTpezxWJ_hUfbVdY1tnd2T2hxMO7jkn2YzN1-sXh07GHY3G52z7N9Msue9t3u5X_2Jk6x0UykJPmB2CZEiDGCCD3YPrRlgz8QMUJRGy0RPZZcXIDg5a8?type=png)

## Essential Networking Concepts for Troubleshooting

### Network Addressing in Kubernetes

| Network Type | Description | Address Range (Example) |
|--------------|-------------|------------------------|
| Node Network | Physical or virtual network where cluster nodes reside | 10.0.0.0/24 |
| Pod Network | Network for pod-to-pod communication (defined by CNI) | 10.244.0.0/16 |
| Service Network | Virtual IP range for Kubernetes services | 10.96.0.0/12 |
| External Network | Networks outside the cluster | Various |

### Container Network Interface (CNI)

The CNI plugin implements pod networking. Common options include:

- **Calico**: BGP-based networking with policy support
- **Cilium**: eBPF-based networking with advanced observability
- **Flannel**: Simple overlay network
- **Weave Net**: Mesh overlay network

Understanding your CNI's architecture and logging mechanisms is crucial for troubleshooting.

## Diagnostic Methodology

### Systematic Networking Troubleshooting Approach

Follow this structured methodology when facing Kubernetes networking issues:

1. **Define the Problem**
   - Identify affected components
   - Determine frequency and impact
   - Document expected vs. actual behavior

2. **Understand the Network Path**
   - Map the complete communication path
   - Identify all components involved

3. **Isolate the Issue**
   - Test each networking layer systematically
   - Move from simple to complex tests

4. **Gather Evidence**
   - Collect logs from all relevant components
   - Capture network traffic if possible
   - Document configuration settings

5. **Analyze Findings**
   - Look for error patterns
   - Compare with known issues
   - Check recent changes

6. **Implement Solution**
   - Apply targeted fixes
   - Validate the solution
   - Document for future reference

## Essential Troubleshooting Tools

### Built-in Kubernetes Tools

```bash
# Describe pod networking details
kubectl describe pod <pod-name> -n <namespace>

# Check endpoints for a service
kubectl get endpoints <service-name> -n <namespace>

# View events related to networking
kubectl get events -n <namespace> --field-selector type=Warning

# Access pod for network diagnostics
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```

### Network Diagnostic Tools for Pods

Create a network diagnostics pod that has the necessary tools:

```yaml
# network-debug.yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
  namespace: default
spec:
  containers:
  - name: network-debug
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
```

Deploy and use the diagnostic pod:

```bash
kubectl apply -f network-debug.yaml
kubectl exec -it network-debug -- /bin/bash

# Inside the pod, use tools like:
ping <target-ip>
traceroute <target-host>
dig <service-name>.<namespace>.svc.cluster.local
tcpdump -i any -n port 80
curl -v <service-url>
```

### CNI-Specific Tools

Depending on your CNI, use specific troubleshooting tools:

**Calico:**
```bash
# Check Calico node status
kubectl exec -n kube-system calico-node-xxxxx -- calico-node -felix-live

# View BGP peer information
calicoctl node status
```

**Cilium:**
```bash
# Install Cilium CLI
cilium status
cilium connectivity test
```

## Common Networking Issues and Solutions

### Pod-to-Pod Communication Problems

**Issue 1: Pods cannot communicate across nodes**

Symptoms:
- Pods on the same node can communicate
- Pods on different nodes cannot reach each other

Diagnosis:
```bash
# From source pod
kubectl exec -it <source-pod> -n <namespace> -- ping <destination-pod-ip>

# Check CNI logs
kubectl logs -n kube-system <cni-pod-name>

# Check node networking
kubectl debug node/<node-name> -it --image=ubuntu
```

Potential Solutions:
- Verify network plugin configuration
- Check for MTU mismatches between nodes
- Ensure correct routing between node networks
- Inspect network policies that might be blocking traffic

**Issue 2: Intermittent pod connectivity**

Symptoms:
- Random timeouts between services
- Connections occasionally fail

Diagnosis:
```bash
# Continuous connectivity testing
while true; do kubectl exec <pod> -- curl -m 1 <service>.<namespace>.svc.cluster.local; sleep 1; done

# Check for packet loss
kubectl exec <pod> -- ping -c 100 <destination-ip> | grep loss

# Monitor network interfaces
kubectl debug node/<node-name> -it --image=ubuntu -- sh -c "apt update && apt install -y iftop && iftop"
```

Potential Solutions:
- Scale up CNI controller pods
- Increase resources for CNI components
- Check for network congestion or rate limiting
- Verify node health (CPU/memory pressure)

### Service Resolution Problems

**Issue: Services cannot be resolved or accessed**

Symptoms:
- `curl: (6) Could not resolve host` errors
- Timeouts when accessing service by name
- Direct pod IP works but service IP fails

Diagnosis:
```bash
# Check if DNS is working
kubectl exec -it <pod-name> -- nslookup kubernetes.default.svc.cluster.local

# Verify service and endpoints
kubectl get svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Test service connectivity using IP
kubectl exec -it <pod-name> -- curl -v <service-cluster-ip>

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

Potential Solutions:
- Restart CoreDNS/kube-dns pods
- Check for mismatched labels between service selector and pod labels
- Verify kube-proxy is running on all nodes
- Ensure pods are in Ready state

### Network Policy Issues

**Issue: Traffic being blocked unexpectedly**

Symptoms:
- Connections timeout despite correct routing
- Some pods can communicate while others cannot

Diagnosis:
```bash
# List all network policies
kubectl get networkpolicy --all-namespaces

# Test connectivity with temporary bypass pod (no policy)
kubectl run temp-bypass --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl exec -it temp-bypass -- curl <destination-service>

# Generate traffic and observe logs (Calico example)
kubectl exec -it -n calico-system <calico-pod> -- tail -f /var/log/calico/felix.log
```

Potential Solutions:
- Review all network policies affecting the namespace
- Check for implicit deny policies
- Verify policy selectors match intended pods
- Add explicit allow rules for necessary traffic

### Ingress/External Access Problems

**Issue: External traffic not reaching services**

Symptoms:
- Internal access works but external fails
- HTTP errors when accessing via Ingress

Diagnosis:
```bash
# Check Ingress configuration
kubectl describe ingress <ingress-name> -n <namespace>

# Verify Ingress controller logs
kubectl logs -n <ingress-namespace> -l app=ingress-nginx

# Test connectivity directly to NodePort
curl -v http://<node-ip>:<node-port>

# Check service from inside the cluster
kubectl exec -it network-debug -- curl -v <service-name>.<namespace>.svc.cluster.local
```

Potential Solutions:
- Check TLS certificate configuration
- Verify Ingress controller has proper permissions
- Ensure backend service and endpoints are healthy
- Check for firewall rules blocking external traffic

## Advanced Diagnostic Techniques

### Packet Capture and Analysis

Capture network traffic to identify specific issues:

```bash
# Install tcpdump in container (if not available)
kubectl exec -it <pod-name> -- apt-get update && apt-get install -y tcpdump

# Capture traffic on a specific port
kubectl exec -it <pod-name> -- tcpdump -i any -n port 80 -w /tmp/capture.pcap

# Copy capture file to local machine for analysis
kubectl cp <pod-name>:/tmp/capture.pcap ./capture.pcap

# Analyze with Wireshark or tcpdump
tcpdump -r capture.pcap -n
```

### Using eBPF for Deep Network Insights

eBPF tools provide kernel-level visibility into network traffic:

```bash
# Deploy bpftrace for eBPF tracing
kubectl apply -f https://raw.githubusercontent.com/iovisor/kubectl-trace/master/examples/kubectl-trace-pod.yaml

# Trace TCP connections (run on host)
bpftrace -e 'tracepoint:syscalls:sys_enter_connect { printf("PID: %d, Process: %s\n", pid, comm); }'

# Using Hubble with Cilium
hubble observe --namespace <namespace> --protocol http
```

### Service Mesh Observability

If using a service mesh like Istio or Linkerd:

```bash
# Istio proxy logs
kubectl logs <pod-name> -c istio-proxy -n <namespace>

# Check Envoy configuration
istioctl proxy-config all <pod-name>.<namespace>

# Linkerd diagnostics
linkerd diagnostics proxy -n <namespace> <resource-name>
```

## Kubernetes DNS Deep Dive

### CoreDNS Troubleshooting

CoreDNS is the default DNS server in Kubernetes. Common DNS issues include:

**Issue: DNS resolution failures or slowness**

Diagnosis:
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from a pod
kubectl exec -it <pod-name> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod-name> -- dig +short kubernetes.default.svc.cluster.local

# Check CoreDNS configuration
kubectl get configmap -n kube-system coredns -o yaml
```

Potential Solutions:
- Scale up CoreDNS deployment
- Optimize CoreDNS configuration with caching
- Check for network policies blocking DNS traffic (port 53)
- Verify kubelet DNS configuration

Example CoreDNS optimization:
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
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## CNI-Specific Troubleshooting

### Calico

**Check Calico component status:**
```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
```

**View BGP peering information:**
```bash
# Install calicoctl if needed
kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml

# Check node status
kubectl exec -it -n kube-system calicoctl -- calicoctl node status
```

**Troubleshoot specific issues:**
```bash
# Check IP pools
kubectl exec -it -n kube-system calicoctl -- calicoctl get ippool -o wide

# View BGP configuration
kubectl exec -it -n kube-system calicoctl -- calicoctl get bgpconfig -o yaml

# Debug policy issues
kubectl exec -it -n kube-system calicoctl -- calicoctl get networkpolicy -n <namespace> -o yaml
```

### Cilium

**Check Cilium component status:**
```bash
# Install Cilium CLI
cilium status

# Verify connectivity
cilium connectivity test
```

**Debug specific issues:**
```bash
# Check endpoint status
cilium endpoint list

# View service details
cilium service list

# Debug policies
cilium policy get

# Enable debug mode
kubectl -n kube-system exec ds/cilium -- cilium config set debug=true
```

### Flannel

**Check Flannel pods:**
```bash
kubectl get pods -n kube-system -l app=flannel
```

**Debug Flannel issues:**
```bash
# View logs
kubectl logs -n kube-system -l app=flannel

# Check Flannel network configuration
kubectl get cm -n kube-system kube-flannel-cfg -o yaml

# Verify subnet allocation
kubectl debug node/<node-name> -it --image=ubuntu -- cat /run/flannel/subnet.env
```

## Troubleshooting Scenarios

### Scenario 1: Pod Cannot Reach External Services

**Problem**: Applications inside pods cannot access external APIs or services.

**Step 1**: Verify basic connectivity
```bash
# Test DNS resolution of external domains
kubectl exec -it <pod-name> -- nslookup google.com

# Test outbound connectivity
kubectl exec -it <pod-name> -- curl -v https://httpbin.org/get
```

**Step 2**: Check for network policies restricting egress
```bash
# List network policies
kubectl get networkpolicy --all-namespaces

# Look for policies with egress rules
kubectl get networkpolicy -n <namespace> -o yaml | grep -A 20 egress:
```

**Step 3**: Verify node networking configuration
```bash
# Check node's outbound connectivity
kubectl debug node/<node-name> -it --image=ubuntu -- curl -v https://httpbin.org/get

# Verify NAT configuration
kubectl debug node/<node-name> -it --image=ubuntu -- iptables -t nat -S
```

**Step 4**: Look for proxy configurations
```bash
# Check for proxy environment variables
kubectl exec -it <pod-name> -- env | grep -i proxy

# Verify HTTP_PROXY doesn't have typos
```

**Solution**: Common fixes include:
- Adding appropriate egress network policies
- Configuring proper proxy settings
- Ensuring node networking allows outbound traffic
- Setting up correct DNS forwarding in CoreDNS

### Scenario 2: Service Intermittently Unavailable

**Problem**: A service is occasionally unreachable with timeout errors.

**Step 1**: Check service and endpoint health
```bash
# Monitor endpoints over time
watch kubectl get endpoints <service-name> -n <namespace>

# Check pod readiness status
kubectl get pods -n <namespace> -l app=<app-label> -o wide
```

**Step 2**: Analyze service connection patterns
```bash
# Create test script
cat <<EOF > test.sh
#!/bin/bash
while true; do
  echo -n "\$(date): "
  curl -s -m 2 <service-name>.<namespace>.svc.cluster.local || echo "Failed"
  sleep 1
done
EOF

# Run in debug pod
kubectl exec -it network-debug -- bash -c "$(cat test.sh)"
```

**Step 3**: Check for resource constraints
```bash
# Look for pod evictions or resource limits
kubectl describe pods -n <namespace> -l app=<app-label> | grep -E 'OOM|Evicted'

# View resource usage
kubectl top pods -n <namespace>
```

**Step 4**: Examine kube-proxy functionality
```bash
# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Verify iptables rules for the service
kubectl debug node/<node-name> -it --image=ubuntu -- iptables-save | grep <service-cluster-ip>
```

**Solution**: Common fixes include:
- Increasing pod resources or replicas
- Adding pod anti-affinity rules to spread across nodes
- Optimizing readiness probe configuration
- Restarting kube-proxy if iptables rules are corrupted

### Scenario 3: Cross-Namespace Communication Blocked

**Problem**: Services in one namespace cannot communicate with services in another namespace.

**Step 1**: Test DNS resolution across namespaces
```bash
# Test DNS resolution for cross-namespace service
kubectl exec -it <pod-name> -n <source-namespace> -- nslookup <service-name>.<target-namespace>.svc.cluster.local
```

**Step 2**: Check for namespace isolation policies
```bash
# Look for network policies that isolate namespaces
kubectl get networkpolicy -A -o yaml | grep -A 10 namespaceSelector
```

**Step 3**: Verify service configuration
```bash
# Check that service is exposed correctly
kubectl get svc <service-name> -n <target-namespace>

# Confirm endpoints exist
kubectl get endpoints <service-name> -n <target-namespace>
```

**Step 4**: Test direct IP connectivity
```bash
# Get service cluster IP
SERVICE_IP=$(kubectl get svc <service-name> -n <target-namespace> -o jsonpath='{.spec.clusterIP}')

# Test direct IP connectivity
kubectl exec -it <pod-name> -n <source-namespace> -- curl -v $SERVICE_IP:<port>
```

**Solution**: Common fixes include:
- Adding namespace-specific allow rules in network policies
- Ensuring DNS is properly configured for cross-namespace resolution
- Creating custom network policies to allow specific cross-namespace traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cross-namespace
  namespace: target-namespace
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: source-namespace
  policyTypes:
  - Ingress
```

## Network Policy Examples

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow Intra-Namespace Communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
  policyTypes:
  - Ingress
```

### Allow Specific External Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  egress:
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
  policyTypes:
  - Egress
```

### Database Access Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress: []
  policyTypes:
  - Ingress
  - Egress
```

## Best Practices for Network Resilience

### Health Checks and Readiness Probes

Configure appropriate readiness and liveness probes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resilient-app
spec:
  containers:
  - name: app
    image: your-app:1.0
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3
```

### Connection Pooling

Implement connection pooling to prevent service overload:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-connection-config
data:
  db.conf: |
    max_connections=100
    min_connections=5
    max_idle_time=300
```

### Circuit Breaking

Use service mesh capabilities for circuit breaking:

```yaml
# Istio circuit breaker example
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
spec:
  host: my-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 5
      interval: 5s
      baseEjectionTime: 30s
```

### Graceful Service Termination

Ensure pods terminate gracefully:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: your-app:1.0
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10 && /app/shutdown.sh"]
```

## Monitoring and Observability for Networking

### Prometheus Metrics for Network Monitoring

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-network-metrics
  namespace: monitoring
data:
  prometheus.yml: |
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
      - job_name: 'kube-proxy'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_k8s_app]
            regex: kube-proxy
            action: keep
      - job_name: 'cni-metrics'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_k8s_app]
            regex: calico-node|cilium|flannel
            action: keep
```

### Grafana Dashboard for Network Visibility

Set up dedicated dashboards for network monitoring:

1. **Pod Network Dashboard**: Track pod network metrics
   - Bytes transmitted/received per pod
   - Connection rate per service
   - TCP retransmissions
   - DNS query latency

2. **Service Mesh Dashboard**: If using a service mesh
   - Request volume
   - Success rate
   - Latency percentiles
   - mTLS coverage

3. **CNI Dashboard**: CNI-specific metrics
   - IP allocation statistics
   - BGP session status (for Calico)
   - Policy enforcement metrics
   - Control plane health

### Distributed Tracing for Network Paths

Implement OpenTelemetry for distributed tracing:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: opentelemetry-config
data:
  otel-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch:
        timeout: 1s
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
```

Instrument your application with OpenTelemetry:

```java
// Java example
Tracer tracer = GlobalOpenTelemetry.getTracer("service-name");
Span span = tracer.spanBuilder("http-request")
    .setAttribute("http.method", "GET")
    .setAttribute("http.url", url)
    .startSpan();
try (Scope scope = span.makeCurrent()) {
    // Make the HTTP request
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR);
    throw e;
} finally {
    span.end();
}
```

## Kubernetes Networking in Production

### High-Performance CNI Configuration

Optimize CNI for production workloads:

```yaml
# Calico performance optimization example
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: true
    nodeAddressAutodetectionV4:
      firstFound: true
  typhaMetricsPort: 9091
  prometheusMetricsEnabled: true
  componentResources:
  - componentName: typha
    resourceRequirements:
      requests:
        cpu: 200m
        memory: 128Mi
      limits:
        cpu: 1000m
        memory: 512Mi
  - componentName: calico-node
    resourceRequirements:
      requests:
        cpu: 250m
        memory: 64Mi
      limits:
        cpu: 1000m
        memory: 500Mi
```

### Scaling Network Policies

Deploy network policies using GitOps for consistency:

```yaml
# FluxCD network policy management
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: network-policies
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/your-org/network-policies
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: network-policies
  namespace: flux-system
spec:
  interval: 10m
  path: ./policies
  prune: true
  sourceRef:
    kind: GitRepository
    name: network-policies
```

### Load Testing Network Performance

Deploy a network performance testing framework:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: network-perf-test
spec:
  template:
    spec:
      containers:
      - name: iperf
        image: networkstatic/iperf3
        command:
        - /bin/sh
        - -c
        - |
          iperf3 -s &
          sleep 5
          for node in $(dig +short iperf.default.svc.cluster.local); do
            echo "Testing performance to $node..."
            iperf3 -c $node -t 30 -J > /results/$(date +%s)-$node.json
          done
          sleep infinity
        volumeMounts:
        - name: results
          mountPath: /results
      volumes:
      - name: results
        emptyDir: {}
      restartPolicy: Never
```

## Conclusion

Effective Kubernetes network troubleshooting requires a systematic approach, deep understanding of the networking stack, and familiarity with diagnostic tools. By following the methodologies outlined in this guide and leveraging the provided examples, you can identify and resolve network issues more efficiently in your Kubernetes clusters.

Remember these key principles:

1. **Understand the full network path** before diving into troubleshooting
2. **Isolate issues methodically** by testing each layer
3. **Use appropriate diagnostic tools** for visibility
4. **Document solutions** for faster resolution in the future
5. **Monitor network health proactively** to catch issues early

With practice, you'll develop intuition about common failure patterns and be able to resolve complex networking issues with greater confidence.

## Additional Resources

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Kubernetes Network Policy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Debugging DNS Resolution in Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Service Mesh Interface Specification](https://smi-spec.io/)
- [eBPF for Kubernetes Networking](https://cilium.io/blog/2021/05/20/cilium-110-cni-mesh/)