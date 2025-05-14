# Service Types and Discovery

Kubernetes Services enable network access to a set of Pods. They provide stable endpoints and load balancing for applications running in your cluster. This guide covers the different types of Services and how service discovery works in Kubernetes.

## What is a Kubernetes Service?

A Service is an abstraction that defines a logical set of Pods and a policy to access them. Services enable loose coupling between dependent Pods by providing a stable networking endpoint.

Key functions of Services:
- Provide stable IP addresses and DNS names
- Load balance traffic across matching Pods
- Enable service discovery within and outside the cluster
- Support different access patterns (internal only, external only, etc.)
- Allow application scaling without breaking dependencies

## Service Selectors

Services identify which Pods to route traffic to using label selectors:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp  # Selects Pods with label app=MyApp
  ports:
  - port: 80    # Port the service listens on
    targetPort: 8080  # Port the traffic is routed to on the Pod
```

## Service Types

Kubernetes offers several types of Services to accommodate different access patterns:

### ClusterIP (Default)

Exposes the Service on a cluster-internal IP only, making it only reachable from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # This is actually the default
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

#### Features:
- Only accessible within the cluster
- Gets a stable internal cluster IP
- Provides load balancing across Pods
- Automatically registered in the cluster DNS

#### Use cases:
- Internal microservices communication
- Backend services that shouldn't be exposed externally
- Databases and caches

### NodePort

Exposes the Service on each Node's IP at a static port. Creates a ClusterIP Service and routes external traffic to it.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80          # Internal cluster port
    targetPort: 8080  # Container port
    nodePort: 30080   # External port (must be 30000-32767)
```

If you don't specify a `nodePort`, Kubernetes will automatically assign one from the configured range.

#### Features:
- Accessible from outside the cluster
- Each node proxies the specified port to your Service
- Automatically creates a ClusterIP Service
- Usable even if you don't have a load balancer

#### Use cases:
- Development environments
- Testing external access
- When load balancer services aren't available
- When you need a fixed external port

### LoadBalancer

Exposes the Service externally using a cloud provider's load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  # Optional: restrict which IPs can access this service
  loadBalancerSourceRanges:
  - 192.168.0.0/24
```

#### Features:
- Creates an external load balancer in the cloud provider
- Automatically creates NodePort and ClusterIP Services
- Routes external traffic to NodePort Service
- Provides static external IP (eventually, may take some time to provision)

#### Use cases:
- Production applications that need external access
- HTTP/HTTPS public services
- When high availability and automatic scaling are required for external access

### ExternalName

Maps a Service to a DNS name, not to selectors. Acts as a CNAME record.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
```

#### Features:
- No proxy is created, just a DNS CNAME record
- No ports or selectors are defined
- Traffic is sent to the external service, not proxied

#### Use cases:
- External databases
- Services in different namespaces or clusters
- Services in different cloud regions or providers
- Migration scenarios

### Headless Services

Services that don't need load balancing or a single Service IP.

#### With Selectors:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None  # Makes it a headless service
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

#### Without Selectors:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  clusterIP: None
```
With manual endpoints:
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service  # Must match the service name
subsets:
  - addresses:
      - ip: 192.168.0.1
      - ip: 192.168.0.2
    ports:
      - port: 9376
```

#### Features:
- No load balancing or proxying
- Returns multiple A records (one for each Pod) from DNS queries
- Essential for StatefulSets to get stable DNS names per Pod

#### Use cases:
- StatefulSets with individual Pod DNS entries
- When clients need to connect to specific Pods
- Service discovery for peer-to-peer communication

## Service Discovery

There are two primary ways Kubernetes implements service discovery:

### Environment Variables

When a Pod is run, Kubernetes injects environment variables for each active Service:

For a Service named `redis-master` exposed on port 6379:
```
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

**Limitation**: The Pod must be created after the Service for the variables to be available.

### DNS

The more flexible and common approach is DNS-based service discovery:

- Every Service gets a DNS entry in the format: `<service-name>.<namespace>.svc.cluster.local`
- Pods can look up the Service by name in the same namespace: `<service-name>`
- Or by fully qualified domain name: `<service-name>.<namespace>.svc.cluster.local`

Features of DNS-based service discovery:
- Works across namespaces
- No ordering dependency (works regardless of creation order)
- Provides A records for headless Services (pointing to Pod IPs)
- Provides SRV records for named ports

Example query from a Pod:
```bash
# Simple lookup in the same namespace
nslookup redis-master

# Cross-namespace lookup
nslookup redis-master.other-namespace.svc.cluster.local
```

## Port Configuration

Services can expose multiple ports for different protocols:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9100
    targetPort: metrics  # Can reference a named port in the Pod
```

Named ports in Pods:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
    - name: metrics
      containerPort: 9100
```

## Session Affinity

By default, Services route each connection to a randomly selected Pod. You can enable session affinity to make a client's connections go to the same Pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
  ports:
  - port: 80
    targetPort: 8080
```

Options:
- `None` (default): Random distribution
- `ClientIP`: Routes based on client's IP address

## External Traffic Policy

Controls how external traffic is routed:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

Options:
- `Cluster` (default): Can route traffic to Pods on any node
- `Local`: Only routes to Pods on the node receiving the request (preserves client IP, may cause uneven traffic distribution)

## Internal Traffic Policy

Controls how internal traffic is routed within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  internalTrafficPolicy: Local
  ports:
  - port: 80
    targetPort: 8080
```

Options:
- `Cluster` (default): Can route traffic to Pods on any node
- `Local`: Only routes to Pods on the node originating the request

## Service Topology

Allows routing traffic based on node topology:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  topologyKeys:
  - "kubernetes.io/hostname"
  - "topology.kubernetes.io/zone"
  - "topology.kubernetes.io/region"
  - "*"
  ports:
  - port: 80
    targetPort: 8080
```

This prioritizes routing traffic to Pods:
1. On the same node
2. In the same zone
3. In the same region
4. Anywhere in the cluster

## Multi-Port Services

Services can expose multiple ports, with each port potentially targeting different container ports:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: admin
    port: 8888
    targetPort: admin-port
```

Each port must have a name if more than one port is defined.

## Service without Selectors

Services can be created without selectors, allowing manual endpoint configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  ports:
  - port: 5432
    targetPort: 5432
```

Then create matching Endpoints:
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-database  # Must match service name
subsets:
  - addresses:
      - ip: 192.168.0.10
      - ip: 192.168.0.11
    ports:
      - port: 5432
```

Use cases:
- External databases
- Services in different namespaces
- Services across clusters
- Services during migration

## EndpointSlices

EndpointSlices are a more scalable alternative to Endpoints:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: web-service-abc
  labels:
    kubernetes.io/service-name: web-service
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    nodeName: node-1
    topology:
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```

Features:
- More scalable for services with many endpoints
- Tracked by topology (hostname, zone, region)
- Support for multiple address types (IPv4, IPv6, FQDN)
- Automatic slicing for large services

## Real-World Service Configurations

### Web Application with Multiple Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
spec:
  type: ClusterIP
  selector:
    app: backend
    component: api
  ports:
  - name: http
    port: 8000
    targetPort: api-port
  - name: metrics
    port: 9102
    targetPort: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: backend-db
spec:
  clusterIP: None  # Headless service for database
  selector:
    app: backend
    component: database
  ports:
  - port: 5432
    targetPort: 5432
```

### Microservices Architecture

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: microservices
spec:
  selector:
    app: auth
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: microservices
spec:
  selector:
    app: orders
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: grpc
    port: 9090
    targetPort: grpc
---
apiVersion: v1
kind: Service
metadata:
  name: payment-gateway
  namespace: microservices
spec:
  type: ExternalName
  externalName: payments.example.com
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-canary
  namespace: microservices
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
spec:
  type: LoadBalancer
  selector:
    app: frontend
    version: canary
  ports:
  - port: 80
    targetPort: 8080
```

## Best Practices

1. **Name your ports** when defining multiple ports on a Service
   ```yaml
   ports:
   - name: http
     port: 80
   - name: https
     port: 443
   ```

2. **Use specific selectors** to target the right Pods
   ```yaml
   selector:
     app: frontend
     tier: web
     version: v1
   ```

3. **Add appropriate annotations** for cloud-specific load balancer configuration
   ```yaml
   metadata:
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/cert-id"
       service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
   ```

4. **Set resource-specific timeout values** for Provider load balancers
   ```yaml
   metadata:
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
   ```

5. **Restrict access with loadBalancerSourceRanges** for security
   ```yaml
   spec:
     loadBalancerSourceRanges:
     - 192.168.0.0/24
     - 10.0.0.0/8
   ```

6. **Use ExternalTrafficPolicy: Local** when client IP preservation is needed
   ```yaml
   spec:
     externalTrafficPolicy: Local
   ```

7. **Create separate Services for different access patterns**
   - Internal-only APIs use ClusterIP
   - Public endpoints use LoadBalancer

8. **Leverage headless Services for StatefulSet workloads**
   ```yaml
   spec:
     clusterIP: None
   ```

9. **Use ExternalName for service abstraction** during migrations
   ```yaml
   spec:
     type: ExternalName
     externalName: legacy-service.example.com
   ```

10. **Consider topology awareness** for multi-zone clusters
    ```yaml
    spec:
      topologyKeys:
      - "kubernetes.io/hostname"
      - "topology.kubernetes.io/zone"
    ```

## Troubleshooting Services

### Common Issues

1. **Service not routing traffic to Pods**
   - Check selector matches Pod labels
   - Verify Pods are running and ready
   - Confirm targetPort matches containerPort

2. **Cannot access a Service**
   - For ClusterIP: ensure you're connecting from within the cluster
   - For NodePort: check if any network policies are blocking access
   - For LoadBalancer: verify security groups/firewall rules

3. **Service external IP stays in pending**
   - For LoadBalancer services, confirm your cloud provider integration is working
   - For on-premises clusters, ensure a load balancer implementation is installed

4. **DNS resolution not working**
   - Verify kube-dns/CoreDNS is running
   - Check if Pod has correct DNS policy and config

### Diagnostic Commands

```bash
# List all services
kubectl get services

# Describe a service
kubectl describe service <service-name>

# Check if endpoints are created
kubectl get endpoints <service-name>

# View endpoint slices
kubectl get endpointslices

# Verify labels on pods match service selector
kubectl get pods --selector=app=<app-name> --show-labels

# Check DNS resolution from a test pod
kubectl run -it --rm debug --image=alpine --restart=Never -- sh -c "nslookup <service-name>"

# Test service connectivity
kubectl run -it --rm debug --image=alpine --restart=Never -- sh -c "wget -qO- <service-name>"

# Check if kube-proxy is running on all nodes
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

Kubernetes Services provide the critical networking infrastructure that enables reliable communication between components of your applications. Understanding the different service types and their capabilities allows you to design robust networking solutions for your containerized workloads.