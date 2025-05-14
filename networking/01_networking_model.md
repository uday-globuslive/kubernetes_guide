# Kubernetes Networking Model

Kubernetes networking represents one of the platform's most powerful but complex aspects. This chapter provides a comprehensive exploration of the Kubernetes networking model, including its core concepts, implementation details, and practical considerations for production deployments.

## Table of Contents

- [Kubernetes Networking Fundamentals](#kubernetes-networking-fundamentals)
- [The Four Networking Problems](#the-four-networking-problems)
- [Container Network Interface (CNI)](#container-network-interface-cni)
- [Pod Networking](#pod-networking)
- [Service Networking](#service-networking)
- [Network Policies](#network-policies)
- [DNS in Kubernetes](#dns-in-kubernetes)
- [Ingress and Egress](#ingress-and-egress)
- [NetworkPolicy Implementation](#networkpolicy-implementation)
- [Performance Considerations](#performance-considerations)
- [Troubleshooting Network Issues](#troubleshooting-network-issues)
- [CNI Plugin Comparison](#cni-plugin-comparison)
- [Advanced Topics](#advanced-topics)

## Kubernetes Networking Fundamentals

The Kubernetes networking model creates a flat network in which every pod can communicate with every other pod without NAT. This fundamental principle underpins all Kubernetes networking implementations.

### Key Requirements

The Kubernetes networking model imposes the following requirements:

1. **Pods on a node can communicate with all pods on all nodes without NAT**
2. **Agents on a node (e.g., kubelet) can communicate with all pods on that node**
3. **Pods in the host network can communicate with all pods on all nodes**

### IP-per-Pod Model

Every pod in Kubernetes receives its own unique IP address:

- Pods are treated similarly to VMs or physical hosts on a network
- Containers within a pod share the pod's network namespace
- Containers within a pod can communicate via localhost
- All containers in all pods can communicate without NAT

```
┌──────────────────────────────┐    ┌──────────────────────────────┐
│           Node 1             │    │           Node 2             │
│                              │    │                              │
│  ┌───────────┐  ┌───────────┐│    │  ┌───────────┐  ┌───────────┐│
│  │  Pod A    │  │  Pod B    ││    │  │  Pod C    │  │  Pod D    ││
│  │           │  │           ││    │  │           │  │           ││
│  │ IP: 10.1.1.2 │ IP: 10.1.1.3 │    │ IP: 10.1.2.2 │ IP: 10.1.2.3 │
│  └───────────┘  └───────────┘│    │  └───────────┘  └───────────┘│
│                              │    │                              │
└──────────────┬───────────────┘    └──────────────┬───────────────┘
               │                                    │
               │        Cluster Network             │
               └────────────────┬──────────────────┘
```

In this model:
- Pod A can communicate directly with Pod C on a different node
- Containers within Pod A share the same IP address and can communicate via localhost
- Pod networking is the foundation for all other Kubernetes networking concepts

## The Four Networking Problems

Kubernetes networking addresses four distinct problems:

### 1. Container-to-Container Communication

Containers within the same pod share the same network namespace:

```
┌─────────────────────────────────┐
│             Pod A               │
│                                 │
│   ┌──────────┐   ┌──────────┐   │
│   │Container1│   │Container2│   │
│   │          │   │          │   │
│   └──────────┘   └──────────┘   │
│         │             │         │
│         └─────┬───────┘         │
│               │                 │
│          localhost              │
│                                 │
└─────────────────────────────────┘
```

Containers within a pod:
- Share the same IP address
- Can communicate via localhost
- Share the same port space (ports must be unique within a pod)
- Share the same network namespace, including network interfaces

This is handled by Kubernetes through the use of the Pause container (or "sandbox" container), which creates the network namespace shared by all containers in the pod.

### 2. Pod-to-Pod Communication

Pods on the same node communicate through the node's internal bridge network:

```
┌───────────────────────────────────────────────────┐
│                     Node 1                        │
│                                                   │
│  ┌───────────┐           ┌───────────┐            │
│  │  Pod A    │           │  Pod B    │            │
│  │10.1.1.2   │◄─────────►│10.1.1.3   │            │
│  └─────┬─────┘           └─────┬─────┘            │
│        │                       │                  │
│        └─────────┬─────────────┘                  │
│                  │                                │
│            ┌─────▼─────┐                          │
│            │  Bridge   │                          │
│            │  cbr0     │                          │
│            └─────┬─────┘                          │
│                  │                                │
└──────────────────┼────────────────────────────────┘
                   │
               Node Network
```

Pods on different nodes must communicate across the node network:

```
┌────────────────────┐     ┌────────────────────┐
│      Node 1        │     │      Node 2        │
│                    │     │                    │
│  ┌─────────────┐   │     │   ┌─────────────┐  │
│  │   Pod A     │   │     │   │   Pod C     │  │
│  │  10.1.1.2   │   │     │   │  10.1.2.2   │  │
│  └──────┬──────┘   │     │   └──────┬──────┘  │
│         │          │     │          │         │
│  ┌──────▼──────┐   │     │   ┌──────▼──────┐  │
│  │    cbr0     │   │     │   │    cbr0     │  │
│  └──────┬──────┘   │     │   └──────┬──────┘  │
│         │          │     │          │         │
│  ┌──────▼──────┐   │     │   ┌──────▼──────┐  │
│  │   eth0      │◄──┼─────┼──►│   eth0      │  │
│  │  172.16.1.2 │   │     │   │  172.16.1.3 │  │
│  └─────────────┘   │     │   └─────────────┘  │
└────────────────────┘     └────────────────────┘
           │                        │
           └────────────┬───────────┘
                        │
                   Cluster Network
```

This is implemented by CNI plugins, which provide various mechanisms to establish this connectivity.

### 3. Pod-to-Service Communication

Services provide stable endpoints for accessing pods:

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│                 Service (ClusterIP)                      │
│                 10.96.0.10:80                            │
│                       │                                  │
│                       │                                  │
│           ┌───────────▼───────────┐                      │
│           │                       │                      │
│  ┌────────▼─────┐      ┌─────────▼────┐     ┌───────────▼────┐
│  │   Pod A      │      │   Pod B      │     │   Pod C       │
│  │ 10.1.1.2:8080 │      │ 10.1.1.3:8080 │     │ 10.1.2.2:8080  │
│  └──────────────┘      └──────────────┘     └────────────────┘
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

Services facilitate:
- Load balancing across pods
- Service discovery via DNS
- Stable IP addresses and ports regardless of pod changes
- Different access types (ClusterIP, NodePort, LoadBalancer)

This is implemented by kube-proxy, which sets up the necessary iptables or IPVS rules to direct traffic to the appropriate pods.

### 4. External-to-Service Communication

External traffic can access services through:

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  External Traffic                                             │
│        │                                                      │
│        │                                                      │
│        │                                                      │
│  ┌─────▼───────────────────────────────┐                      │
│  │ Service (LoadBalancer/NodePort)     │                      │
│  │                                     │                      │
│  └─────────────┬───────────────────────┘                      │
│                │                                              │
│                │                                              │
│  ┌─────────────▼──────┐   ┌─────────────────────┐            │
│  │     Pod A          │   │      Pod B          │            │
│  │                    │   │                     │            │
│  └────────────────────┘   └─────────────────────┘            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

External access is facilitated by:
- NodePort Services (expose on every node's IP)
- LoadBalancer Services (provision external load balancer)
- Ingress resources (HTTP/HTTPS routing)

This is implemented by a combination of kube-proxy, cloud provider integration, and ingress controllers.

## Container Network Interface (CNI)

The Container Network Interface provides a standard way for container runtimes to configure network interfaces for containers.

### CNI Basics

CNI is:
- A specification for configuring network interfaces in Linux containers
- A set of versioned libraries for writing plugins
- A collection of plugins implementing various network types

The CNI contract is simple:
1. Runtime creates network namespace
2. Runtime identifies network the container should join
3. Runtime hands namespace to CNI plugin
4. Plugin configures container networking
5. Plugin returns result or error to runtime

### CNI Configuration

A typical CNI configuration:

```json
{
  "cniVersion": "0.4.0",
  "name": "k8s-pod-network",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    }
  ]
}
```

This configuration is typically stored in `/etc/cni/net.d/` on each node.

### CNI Plugin Types

CNI plugins are categorized into:

1. **Main network plugins**: Configure pod network interfaces
   - Calico
   - Cilium
   - Flannel
   - Weave Net

2. **IPAM plugins**: Manage IP address allocation
   - host-local
   - dhcp
   - calico-ipam

3. **Meta plugins**: Provide additional functionality
   - Multus: Enables multiple networks per pod
   - Whereabouts: IP address management for multi-cluster environments

### CNI Implementation in Kubernetes

In Kubernetes:
1. Kubelet invokes CNI plugins when a pod is created or deleted
2. The `--network-plugin=cni` flag tells kubelet to use CNI plugins
3. Kubelet looks in `/etc/cni/net.d/` for configuration
4. Kubelet executes plugins found in `/opt/cni/bin/`

Example flow:
```
1. Pod created by API server
2. Kubelet creates pod sandbox (pause container)
3. Kubelet calls CNI plugin with ADD command
4. CNI plugin sets up network (interfaces, routes, etc.)
5. CNI plugin allocates IP address
6. Kubelet starts application containers in the pod
```

## Pod Networking

Pod networking implements the core Kubernetes networking model where each pod gets its own IP address.

### Pod Network Implementation

Pod networking is typically implemented through:

1. **Virtual Ethernet Pairs**: Connect pod namespace to host namespace
2. **Bridge Network**: Node-local bridge connects pods on the same node
3. **Overlay or Underlay Network**: Provides cross-node pod connectivity
4. **Routing Tables**: Direct traffic between nodes

Example pod networking setup:

```
┌─────────────────────────────────────────────────────────────┐
│                          Node                               │
│                                                             │
│  ┌───────────┐                      ┌───────────┐           │
│  │   Pod A   │                      │   Pod B   │           │
│  │           │                      │           │           │
│  │  eth0     │                      │  eth0     │           │
│  └─────┬─────┘                      └─────┬─────┘           │
│        │                                  │                 │
│  veth0 │                            veth1 │                 │
├────────┼──────────────────────────────────┼─────────────────┤
│        │                                  │                 │
│  ┌─────▼──────────────────────────────────▼─────┐           │
│  │                   cbr0                       │           │
│  │                Bridge Network                │           │
│  └─────────────────────┬─────────────────────────┘           │
│                        │                                    │
│                        │                                    │
│  ┌─────────────────────▼─────────────────────────┐           │
│  │                   eth0                        │           │
│  │               Node Interface                  │           │
│  └───────────────────────────────────────────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Pod-to-Pod Communication Within a Node

When Pod A communicates with Pod B on the same node:

1. Traffic leaves Pod A through its network interface (`eth0`)
2. Traffic traverses the virtual ethernet pair (veth0) to the node
3. Traffic travels through the bridge network (cbr0)
4. Traffic enters Pod B via its virtual ethernet interface

### Pod-to-Pod Communication Across Nodes

When Pod A communicates with Pod C on a different node:

1. Traffic leaves Pod A through its network interface
2. Node's routing table directs traffic to the appropriate node
3. Traffic arrives at the destination node
4. Node's bridge directs traffic to the correct pod

This cross-node communication can be implemented using:

1. **Layer 2 (Switching)**: Pods share a single Layer 2 network
2. **Layer 3 (Routing)**: Routing between pod networks on different nodes
3. **Overlay Networks**: Encapsulate pod traffic inside node network packets

### Common Pod Networking Implementations

**Flannel**:
- Uses VXLAN (Virtual Extensible LAN) encapsulation by default
- Can use host-gw (direct routing) for better performance
- Simple configuration, limited policy support

**Calico**:
- Uses BGP for routing pod traffic between nodes
- No encapsulation by default (pure Layer 3)
- Excellent NetworkPolicy support
- Can use VXLAN or IPinIP encapsulation when needed

**Cilium**:
- Uses eBPF (extended Berkeley Packet Filter) for high-performance networking
- Advanced NetworkPolicy support
- Can replace kube-proxy with eBPF
- Provides L7 policy and visibility

**Weave Net**:
- Uses a mesh overlay network between nodes
- Encryption support
- Works well in environments where other solutions face challenges

## Service Networking

Services provide stable endpoints for accessing pods, regardless of pod lifecycle or location changes.

### Service Types

Kubernetes offers several service types:

1. **ClusterIP** (default):
   - Virtual IP only accessible within the cluster
   - Allows pods to communicate with the service

2. **NodePort**:
   - Extends ClusterIP
   - Exposes the service on a static port on every node's IP
   - Port range: 30000-32767 by default

3. **LoadBalancer**:
   - Extends NodePort
   - Provisions an external load balancer (in cloud environments)
   - Routes external traffic to the service

4. **ExternalName**:
   - Maps the service to a DNS name
   - No proxying, just DNS CNAME records

```yaml
# ClusterIP service example
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

```yaml
# NodePort service example
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
  - port: 80
    targetPort: 9376
    nodePort: 30007
```

```yaml
# LoadBalancer service example
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

```yaml
# ExternalName service example
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
```

### Kube-Proxy Implementation Modes

The kube-proxy component is responsible for implementing the service abstraction:

1. **Userspace Mode** (legacy):
   - Watches for service and endpoint changes
   - Opens a port on the node
   - Proxies connections to backend pods
   - Inefficient due to multiple packet traversals

2. **IPTables Mode** (default):
   - Uses iptables rules to redirect traffic
   - Random pod selection for load balancing
   - More efficient than userspace mode
   - Limited to 5000-10000 services due to iptables performance

3. **IPVS Mode**:
   - Uses Linux IPVS (IP Virtual Server)
   - More efficient for large clusters
   - Supports multiple load balancing algorithms
   - Requires the IPVS kernel modules

Configuration example:

```yaml
# kube-proxy config
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"  # Round robin algorithm
```

### Service Implementation Details

A ClusterIP service works as follows:

1. Service is created with a virtual IP from the service CIDR
2. kube-proxy observes the service creation
3. kube-proxy sets up rules to direct traffic from the service IP to backend pods
4. kube-proxy updates rules when pods or services change

iptables rules example (simplified):

```
# Capture traffic to service IP
-A PREROUTING -d 10.96.0.10/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-XXX

# Load balance to backends (for a service with 2 pods)
-A KUBE-SVC-XXX -m statistic --mode random --probability 0.5 -j KUBE-SEP-1
-A KUBE-SVC-XXX -j KUBE-SEP-2

# Direct to specific pod IPs
-A KUBE-SEP-1 -s 10.1.1.2/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-1 -p tcp -m tcp -j DNAT --to-destination 10.1.1.2:8080
-A KUBE-SEP-2 -s 10.1.1.3/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-2 -p tcp -m tcp -j DNAT --to-destination 10.1.1.3:8080
```

IPVS settings example:

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP  10.96.0.10:80 rr
  -> 10.1.1.2:8080   Masq    1      0          0
  -> 10.1.1.3:8080   Masq    1      0          0
```

### Service Discovery

Kubernetes provides two methods for service discovery:

1. **Environment Variables**:
   - Kubelet sets environment variables for active services
   - Format: `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT`
   - Limited to services created before a pod is created

2. **DNS**:
   - CoreDNS provides DNS resolution for services
   - Format: `<service-name>.<namespace>.svc.cluster.local`
   - Automatic for all services

DNS record types:
- A/AAAA records for ClusterIP services
- SRV records for named ports
- CNAME records for ExternalName services

## Network Policies

NetworkPolicies provide firewall-like capabilities to control traffic between pods.

### Basic NetworkPolicy Concepts

By default, all pods can communicate with all other pods. NetworkPolicies restrict this communication:

- NetworkPolicies are applied to pods via label selectors
- Policies are additive (multiple policies can apply to the same pod)
- They specify ingress and/or egress rules
- If no policies select a pod, all traffic is allowed
- If any policy selects a pod, all non-allowed traffic is denied

```yaml
# Basic NetworkPolicy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

This policy:
- Applies to pods with label `app: nginx`
- Allows incoming traffic from pods with label `app: frontend`
- Only allows traffic to TCP port 80

### Ingress and Egress Policies

NetworkPolicies can control both incoming and outgoing traffic:

```yaml
# Combined ingress and egress policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: app
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: monitoring
    ports:
    - protocol: TCP
      port: 8086
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

This policy:
- Applies to database pods (`role: db`)
- Allows ingress from application pods (`role: app`) to port 5432
- Allows egress to monitoring pods (`role: monitoring`) on port 8086
- Allows egress to DNS on ports 53 TCP/UDP

### Selectors in NetworkPolicies

NetworkPolicies support several types of selectors:

1. **podSelector**: Select pods in the same namespace
   ```yaml
   from:
   - podSelector:
       matchLabels:
         role: frontend
   ```

2. **namespaceSelector**: Select all pods in matching namespaces
   ```yaml
   from:
   - namespaceSelector:
       matchLabels:
         purpose: development
   ```

3. **Combined selectors**: Pod must match both selectors (AND logic)
   ```yaml
   from:
   - namespaceSelector:
       matchLabels:
         purpose: production
     podSelector:
       matchLabels:
         role: monitoring
   ```

4. **Multiple rules**: Pod can match any rule (OR logic)
   ```yaml
   from:
   - podSelector:
       matchLabels:
         role: frontend
   - podSelector:
       matchLabels:
         role: monitoring
   ```

5. **ipBlock**: Select traffic from IP ranges
   ```yaml
   from:
   - ipBlock:
       cidr: 172.17.0.0/16
       except:
       - 172.17.1.0/24
   ```

### Default Deny Policies

To create a default deny policy for a namespace:

```yaml
# Default deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}  # Matches all pods
  policyTypes:
  - Ingress
```

```yaml
# Default deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}  # Matches all pods
  policyTypes:
  - Egress
```

### Default Allow Policies

To create a default allow policy for a namespace:

```yaml
# Default allow all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}  # Matches all pods
  ingress:
  - {}  # Empty rule allows all ingress traffic
  policyTypes:
  - Ingress
```

```yaml
# Default allow all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}  # Matches all pods
  egress:
  - {}  # Empty rule allows all egress traffic
  policyTypes:
  - Egress
```

## DNS in Kubernetes

DNS is a critical component for service discovery in Kubernetes.

### CoreDNS Overview

CoreDNS is the default DNS server in Kubernetes since v1.13:

- Deployed as a pod in the `kube-system` namespace
- Uses ConfigMap for configuration
- Exposes as service `kube-dns`
- Provides DNS records for services and pods

Default CoreDNS configuration:

```
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

### DNS Records for Services

Services get the following DNS records:

1. **Regular service**:
   - A/AAAA record: `<service-name>.<namespace>.svc.cluster.local`
   - Points to the ClusterIP of the service

2. **Headless service** (ClusterIP: None):
   - A/AAAA records for each pod backing the service
   - Format: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

3. **Named ports**:
   - SRV records: `_<port-name>._<port-protocol>.<service-name>.<namespace>.svc.cluster.local`
   - Points to the named port

Example service DNS records:

```
# Regular service (ClusterIP: 10.96.0.10)
my-service.default.svc.cluster.local. IN A 10.96.0.10

# Headless service with 3 pods
nginx-0.nginx-headless.default.svc.cluster.local. IN A 10.1.1.2
nginx-1.nginx-headless.default.svc.cluster.local. IN A 10.1.1.3
nginx-2.nginx-headless.default.svc.cluster.local. IN A 10.1.2.2

# SRV record for named port
_http._tcp.my-service.default.svc.cluster.local. IN SRV 0 10 80 my-service.default.svc.cluster.local.
```

### DNS Records for Pods

Pod DNS records follow this format:

```
<pod-ip-address>.<namespace>.pod.cluster.local
```

Example: `10-1-1-2.default.pod.cluster.local` for a pod with IP 10.1.1.2 in the default namespace.

Pod DNS records are not created by default. To enable them, set the `pods` option in the CoreDNS configuration:

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
   pods insecure
   ...
}
```

### DNS Policy for Pods

Pods can have the following DNS policies:

1. **Default**: Inherits the DNS configuration from the node
2. **ClusterFirst**: Queries the cluster DNS server first (default policy)
3. **ClusterFirstWithHostNet**: Like ClusterFirst but for pods using hostNetwork
4. **None**: Requires explicit DNS configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-example
spec:
  dnsPolicy: "ClusterFirst"
  # Custom DNS configuration when using dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "5"
  containers:
  - name: dns-example
    image: nginx
```

### DNS Resolution Flow

When a pod resolves a DNS name:

1. The container's DNS configuration points to the Pod's DNS server (typically 169.254.25.10)
2. CoreDNS receives the query
3. For service names, CoreDNS returns the service ClusterIP
4. For external domains, CoreDNS forwards to upstream DNS servers
5. The response is returned to the pod

### Common DNS Issues and Troubleshooting

1. **5-second delays**: Often caused by improperly configured search domains
2. **DNS resolution failures**: Verify CoreDNS pods are running
3. **Intermittent failures**: Check for CoreDNS resources (memory, CPU)
4. **Custom domain resolution**: Configure proper forwarding in CoreDNS

Troubleshooting commands:

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from a pod
kubectl run -it --rm debug --image=alpine -- nslookup kubernetes.default
```

## Ingress and Egress

Ingress and egress controllers manage traffic entering and leaving the cluster.

### Ingress Resources

Ingress resources define rules for external HTTP/HTTPS access to services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 80
  tls:
  - hosts:
    - example.com
    secretName: example-tls
```

This Ingress:
- Routes traffic to `example.com/app1` to Service1
- Routes traffic to `example.com/app2` to Service2
- Configures TLS using the certificate in the `example-tls` Secret

### Ingress Controllers

Ingress controllers implement the Ingress resources:

1. **Nginx Ingress Controller**:
   - Most popular implementation
   - Highly configurable
   - Extensive annotations

2. **Contour**:
   - Uses Envoy proxy
   - Focus on performance
   - Support for advanced features like WebSockets

3. **Traefik**:
   - Easy configuration
   - Built-in metrics
   - Let's Encrypt integration

4. **HAProxy Ingress**:
   - High performance
   - Enterprise-grade features
   - Advanced traffic management

5. **AWS ALB Ingress Controller**:
   - Uses AWS Application Load Balancer
   - Native integration with AWS services
   - Support for AWS WAF and Shield

Example Nginx Ingress Controller installation:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/cloud/deploy.yaml
```

### Advanced Ingress Features

Modern ingress controllers support advanced features:

1. **Path-based routing**:
   ```yaml
   paths:
   - path: /api/v1
     pathType: Prefix
     backend:
       service:
         name: api-v1
         port:
           number: 80
   ```

2. **Host-based routing**:
   ```yaml
   rules:
   - host: api.example.com
     http:
       paths:
       - path: /
         pathType: Prefix
         backend:
           service:
             name: api-service
             port:
               number: 80
   ```

3. **TLS termination**:
   ```yaml
   tls:
   - hosts:
     - example.com
     secretName: example-tls
   ```

4. **Rate limiting** (Nginx annotation):
   ```yaml
   metadata:
     annotations:
       nginx.ingress.kubernetes.io/limit-rps: "10"
   ```

5. **Session affinity** (Nginx annotation):
   ```yaml
   metadata:
     annotations:
       nginx.ingress.kubernetes.io/affinity: "cookie"
       nginx.ingress.kubernetes.io/session-cookie-name: "route"
   ```

6. **URL rewriting** (Nginx annotation):
   ```yaml
   metadata:
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /$2
   spec:
     rules:
     - host: example.com
       http:
         paths:
         - path: /api(/|$)(.*)
           pathType: Prefix
   ```

### Ingress Traffic Flow

Traffic flow through an Ingress controller:

1. External request arrives at load balancer
2. Load balancer routes to Ingress controller pods
3. Ingress controller applies routing rules
4. Traffic is directed to backend service
5. Service routes to pods

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│External Client│────▶│  Load Balancer│────▶│Ingress        │
└───────────────┘     │ (NodePort or  │     │Controller Pods│
                      │ LoadBalancer) │     └───────┬───────┘
                      └───────────────┘             │
                                                    │
                                                    ▼
                      ┌───────────────┐     ┌───────────────┐
                      │   Backend     │     │  Kubernetes   │
                      │     Pods      │◀────│   Service     │
                      └───────────────┘     └───────────────┘
```

### Egress Traffic Control

Controlling outbound traffic from pods:

1. **Network Policies**: Restrict outbound connections to specific destinations
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: egress-control
   spec:
     podSelector:
       matchLabels:
         app: web
     policyTypes:
     - Egress
     egress:
     - to:
       - ipBlock:
           cidr: 10.0.0.0/24
       - namespaceSelector:
           matchLabels:
             project: database
     - to:
       - ipBlock:
           cidr: 0.0.0.0/0
           except:
           - 10.0.0.0/8
       ports:
       - protocol: TCP
         port: 443
   ```

2. **Egress Gateways**: Channel outbound traffic through a central point
   - Envoy as an egress proxy
   - Istio Egress Gateway
   - Third-party solutions like NGINXaaS or Traffic Director

Example Istio Egress Gateway configuration:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - external-service.example.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-external-service-through-egress-gateway
spec:
  hosts:
  - external-service.example.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 80
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: external-service.example.com
        port:
          number: 80
```

## NetworkPolicy Implementation

NetworkPolicies are implemented by CNI plugins, not by Kubernetes itself.

### CNI Plugin Support for NetworkPolicy

Not all CNI plugins support NetworkPolicies:

| CNI Plugin | NetworkPolicy Support | Implementation Method |
|------------|----------------------|------------------------|
| Calico     | Full                 | iptables or eBPF       |
| Cilium     | Full+ (enhanced)     | eBPF                   |
| Weave Net  | Full                 | iptables               |
| Antrea     | Full                 | Open vSwitch           |
| Flannel    | None (native)        | N/A                    |
| Kube-router | Full                | iptables               |

For Flannel, you can add NetworkPolicy support by:
1. Running Calico in policy-only mode with Flannel networking
2. Using kube-router for policy enforcement with Flannel networking

### How NetworkPolicies Are Enforced

Using Calico as an example:

1. **Agent Deployment**: Calico deploys an agent (Felix) on each node
2. **Policy Watching**: The agent watches for NetworkPolicy resources
3. **Rule Translation**: NetworkPolicies are translated to iptables/eBPF rules
4. **Rule Application**: Rules are applied on each node
5. **Traffic Enforcement**: Traffic is allowed or denied based on rules

iptables implementation (simplified):

```
# Default deny incoming traffic to pod with IP 10.1.1.2
iptables -A FORWARD -d 10.1.1.2/32 -j DROP

# Allow traffic from pod with label app=frontend (IP 10.1.1.3) to pod with IP 10.1.1.2
iptables -A FORWARD -s 10.1.1.3/32 -d 10.1.1.2/32 -j ACCEPT
```

eBPF implementation:
- Programs attached to network interfaces
- More efficient than iptables
- More powerful (can make decisions based on application-layer information)
- Lower overhead for large numbers of rules

### Testing NetworkPolicy Effectiveness

To verify NetworkPolicy enforcement:

1. **Create test pods**:
   ```bash
   kubectl run pod-a --image=nginx
   kubectl run pod-b --image=busybox --rm -it -- wget -q -O- pod-a
   ```

2. **Apply NetworkPolicy**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: test-policy
   spec:
     podSelector:
       matchLabels:
         run: pod-a
     ingress: []  # Empty ingress rules = deny all
   ```

3. **Test connectivity after policy**:
   ```bash
   kubectl run pod-c --image=busybox --rm -it -- wget -q -O- --timeout=5 pod-a
   # Should fail with a timeout
   ```

4. **Add an allow rule and test again**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: test-policy
   spec:
     podSelector:
       matchLabels:
         run: pod-a
     ingress:
     - from:
       - podSelector:
           matchLabels:
             run: pod-b
   ```

   ```bash
   kubectl run pod-b --image=busybox --rm -it -- wget -q -O- pod-a
   # Should succeed
   kubectl run pod-c --image=busybox --rm -it -- wget -q -O- --timeout=5 pod-a
   # Should still fail
   ```

## Performance Considerations

Networking performance is critical for Kubernetes applications.

### CNI Performance Factors

When selecting a CNI plugin, consider these performance factors:

1. **Encapsulation Overhead**:
   - VXLAN adds ~50 bytes of overhead per packet
   - Direct routing (no encapsulation) has better performance
   - Consider MTU settings to avoid fragmentation

2. **Network Hops**:
   - More hops = higher latency
   - Overlay networks typically add more hops
   - Node-to-node routing minimizes hops

3. **iptables vs. eBPF**:
   - iptables performance degrades with rule count
   - eBPF provides better performance at scale
   - For large clusters, prefer eBPF-based solutions

4. **Cross-node Performance**:
   - Host network throughput limits pod-to-pod performance
   - Avoid unnecessary cross-node traffic when possible
   - Consider node anti-affinity for communicating pods

### Performance Optimization Techniques

1. **MTU Optimization**:
   - Increase MTU to reduce overhead (if network supports jumbo frames)
   - Ensure consistent MTU across all network components
   - Example Calico configuration:
     ```yaml
     apiVersion: projectcalico.org/v3
     kind: IPPool
     metadata:
       name: default-ipv4-ippool
     spec:
       cidr: 10.1.0.0/16
       blockSize: 26
       ipipMode: Always
       vxlanMode: Never
       natOutgoing: true
       nodeSelector: all()
     ---
     apiVersion: projectcalico.org/v3
     kind: FelixConfiguration
     metadata:
       name: default
     spec:
       mtu: 9000
     ```

2. **Direct Routing**:
   - Use BGP (Calico) or direct routing where possible
   - Avoid encapsulation when not needed
   - Example Calico configuration for direct routing:
     ```yaml
     apiVersion: projectcalico.org/v3
     kind: IPPool
     metadata:
       name: default-ipv4-ippool
     spec:
       cidr: 10.1.0.0/16
       ipipMode: Never
       vxlanMode: Never
     ```

3. **Service Proxy Optimization**:
   - Use IPVS mode for kube-proxy
   - Consider eBPF-based replacements (Cilium, Calico eBPF)
   - Example kube-proxy configuration:
     ```yaml
     apiVersion: kubeproxy.config.k8s.io/v1alpha1
     kind: KubeProxyConfiguration
     mode: "ipvs"
     ipvs:
       scheduler: "rr"
       syncPeriod: "30s"
     ```

4. **Host Network for Performance-critical Pods**:
   - Use `hostNetwork: true` for pods that need maximum performance
   - Bypass CNI overhead completely
   - Example:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: high-performance-pod
     spec:
       hostNetwork: true
       containers:
       - name: networking-intensive
         image: myapp:1.0
     ```

5. **Network Topology Awareness**:
   - Use topology keys to keep related pods on the same node/rack/zone
   - Example topology spread constraint:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: app-pod
     spec:
       topologySpreadConstraints:
       - maxSkew: 1
         topologyKey: kubernetes.io/hostname
         whenUnsatisfiable: DoNotSchedule
         labelSelector:
           matchLabels:
             app: my-app
       containers:
       - name: app
         image: myapp:1.0
     ```

### Benchmarking Tools

To evaluate Kubernetes networking performance:

1. **iperf3**: Network throughput testing
   ```bash
   # Start server pod
   kubectl run iperf3-server --image=networkstatic/iperf3 --command -- iperf3 -s
   
   # Start client pod and run test
   kubectl run iperf3-client --rm -it --image=networkstatic/iperf3 --command -- iperf3 -c iperf3-server
   ```

2. **netperf**: Network latency and throughput
   ```bash
   # Start server pod
   kubectl run netperf-server --image=netperf/netperf -- netserver -D
   
   # Start client pod and run test
   kubectl run netperf-client --rm -it --image=netperf/netperf -- netperf -H netperf-server
   ```

3. **ping**: Basic latency testing
   ```bash
   kubectl run pingtest --rm -it --image=busybox -- ping -c 10 <target-pod-ip>
   ```

4. **sockperf**: Socket performance testing
   ```bash
   # Start server pod
   kubectl run sockperf-server --image=docker.io/burlachenkok/sockperf -- sockperf server
   
   # Start client pod and run test
   kubectl run sockperf-client --rm -it --image=docker.io/burlachenkok/sockperf -- sockperf ping-pong -i sockperf-server --tcp -m 1024 -t 10
   ```

## Troubleshooting Network Issues

Kubernetes networking issues can be complex to troubleshoot.

### Common Network Issues

1. **Pod-to-Pod Communication Failures**:
   - Pod cannot communicate with another pod
   - NetworkPolicy blocking traffic
   - CNI configuration issues

2. **Service Resolution Failures**:
   - Pods cannot resolve service DNS names
   - CoreDNS configuration issues
   - kube-proxy not functioning correctly

3. **External Access Issues**:
   - Service not accessible from outside
   - Ingress controller not configured correctly
   - Load balancer provisioning failures

4. **Performance Problems**:
   - High latency between pods
   - CNI plugin overhead
   - Network congestion

### Troubleshooting Tools

Essential tools for network troubleshooting:

1. **kubectl**:
   ```bash
   # Check pod status
   kubectl get pods -o wide
   
   # Check service details
   kubectl get svc my-service -o yaml
   
   # View endpoint details
   kubectl get endpoints my-service
   ```

2. **Debugging pods**:
   ```bash
   # Create a debugging pod
   kubectl run debug --image=nicolaka/netshoot --rm -it -- bash
   
   # Test connectivity
   ping <pod-ip>
   curl <service-name>
   nslookup kubernetes.default
   ```

3. **tcpdump**:
   ```bash
   # Capture traffic on a pod
   kubectl exec -it <pod-name> -- tcpdump -i eth0 -n
   
   # Capture specific traffic
   kubectl exec -it <pod-name> -- tcpdump -i eth0 -n "port 80"
   ```

4. **Network policy checker**:
   ```bash
   # Use a tool like netassert or kube-networkpolicy-verification
   kubectl run policy-test --image=danihodovic/netassert --rm -it -- netassert verify \
     --namespace default --source-pod my-pod --target-pod other-pod --port 80
   ```

### Troubleshooting Methodology

A structured approach to network troubleshooting:

1. **Identify the scope**:
   - Pod-to-pod within same node?
   - Pod-to-pod across nodes?
   - Pod-to-service?
   - External-to-service?

2. **Test connectivity layers**:
   - IP connectivity: `ping <ip>`
   - Port connectivity: `nc -zv <ip> <port>`
   - Application connectivity: `curl http://<ip>:<port>`

3. **Check configuration**:
   - NetworkPolicies: `kubectl get networkpolicy`
   - Service configuration: `kubectl get svc <service-name> -o yaml`
   - Endpoints: `kubectl get endpoints <service-name>`
   - CNI configuration: `kubectl get cm -n kube-system calico-config -o yaml`

4. **Examine logs**:
   - CNI plugin logs: `kubectl logs -n kube-system -l k8s-app=calico-node`
   - kube-proxy logs: `kubectl logs -n kube-system -l k8s-app=kube-proxy`
   - CoreDNS logs: `kubectl logs -n kube-system -l k8s-app=kube-dns`

5. **Check for MTU mismatches**:
   ```bash
   # On node
   ip link show
   
   # In pod
   kubectl exec -it <pod-name> -- ip link show eth0
   ```

6. **Verify CNI plugin status**:
   ```bash
   # For Calico
   kubectl get pods -n kube-system -l k8s-app=calico-node
   calicoctl node status
   
   # For Cilium
   kubectl -n kube-system exec ds/cilium -- cilium status
   ```

### Troubleshooting Examples

1. **Pod cannot reach service**:
   ```bash
   # Check if the service has endpoints
   kubectl get endpoints my-service
   
   # Check if kube-proxy is running
   kubectl get pods -n kube-system -l k8s-app=kube-proxy
   
   # Check kube-proxy logs
   kubectl logs -n kube-system -l k8s-app=kube-proxy
   
   # Check iptables rules (on node)
   sudo iptables-save | grep my-service
   ```

2. **DNS resolution issues**:
   ```bash
   # Check CoreDNS pods
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   
   # Check CoreDNS service
   kubectl get svc -n kube-system kube-dns
   
   # Test DNS from pod
   kubectl run debug --image=busybox --rm -it -- nslookup kubernetes.default
   
   # Check CoreDNS logs
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

3. **NetworkPolicy blocking traffic**:
   ```bash
   # List NetworkPolicies
   kubectl get networkpolicy
   
   # Check if policy applies to your pods
   kubectl get pods --selector=app=my-app -o wide
   kubectl get networkpolicy my-policy -o yaml
   
   # Temporarily bypass by adding a label to the pod
   kubectl label pod my-pod bypass-policy=true
   ```

## CNI Plugin Comparison

Different CNI plugins have strengths and weaknesses for various use cases.

### Feature Comparison

| Feature | Calico | Cilium | Flannel | Weave Net | Antrea |
|---------|--------|--------|---------|-----------|--------|
| NetworkPolicy | ✅ | ✅ (enhanced) | ❌ | ✅ | ✅ |
| Encryption | ✅ | ✅ | ❌ | ✅ | ✅ |
| eBPF | ✅ | ✅ | ❌ | ❌ | ❌ |
| IPv6 | ✅ | ✅ | ✅ | ✅ | ✅ |
| Without encapsulation | ✅ | ✅ | ✅ | ❌ | ✅ |
| L7 Policy | ❌ | ✅ | ❌ | ❌ | ❌ |
| Multi-cluster | ✅ | ✅ | ❌ | ✅ | ❌ |
| Windows support | ✅ | ❌ | ✅ | ❌ | ✅ |

### Performance Characteristics

| CNI Plugin | Throughput | Latency | CPU Overhead | Memory Usage | Scale Limit |
|------------|------------|---------|--------------|--------------|-------------|
| Calico (direct) | High | Low | Low | Medium | High |
| Calico (IPIP) | Medium | Medium | Medium | Medium | High |
| Cilium (eBPF) | High | Low | Low | Medium-High | Very High |
| Flannel (VXLAN) | Medium | Medium | Medium | Low | Medium |
| Weave Net | Medium | Medium | Medium | Medium | Medium |
| Antrea | High | Low | Medium | Medium | High |

### CNI Selection Criteria

Choose a CNI plugin based on your requirements:

1. **Network Policy needs**:
   - Basic policies: Calico, Weave, Antrea
   - Advanced L7 policies: Cilium
   - No policies: Flannel

2. **Performance requirements**:
   - Highest performance: Calico (direct), Cilium (eBPF)
   - Moderate performance: Flannel, Weave, Calico (IPIP/VXLAN)

3. **Deployment environment**:
   - Cloud: All CNI plugins
   - On-premises with direct L2: Calico (direct), Cilium (direct)
   - On-premises without direct L2: Calico (IPIP), Cilium (VXLAN), Flannel, Weave

4. **Operational complexity**:
   - Simple: Flannel
   - Moderate: Calico, Weave
   - More complex: Cilium, Antrea

5. **Advanced features**:
   - Multi-cluster networking: Cilium, Calico
   - Encryption: Cilium, Calico, Weave
   - Observability: Cilium

### CNI Installation Examples

1. **Calico**:
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```

2. **Cilium**:
   ```bash
   helm repo add cilium https://helm.cilium.io/
   helm install cilium cilium/cilium --namespace kube-system
   ```

3. **Flannel**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

4. **Weave Net**:
   ```bash
   kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   ```

5. **Antrea**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/build/yamls/antrea.yml
   ```

## Advanced Topics

### Kubernetes Service Mesh Integration

Service meshes add advanced traffic management, security, and observability:

1. **Istio**:
   - Advanced traffic management
   - Service-to-service authentication
   - Rich telemetry and monitoring
   - Works alongside CNI

   ```bash
   # Install Istio
   istioctl install --set profile=default
   
   # Label namespace for automatic sidecar injection
   kubectl label namespace default istio-injection=enabled
   ```

2. **Linkerd**:
   - Lightweight service mesh
   - Automatic mTLS
   - Easy to install and use
   - Works with any CNI

   ```bash
   # Install Linkerd
   linkerd install | kubectl apply -f -
   
   # Label namespace for automatic proxy injection
   kubectl label namespace default linkerd.io/inject=enabled
   ```

### Multi-cluster Networking

Connect Kubernetes clusters for cross-cluster communication:

1. **Cilium Cluster Mesh**:
   ```bash
   # In cluster1
   cilium clustermesh enable --service-type NodePort
   cilium clustermesh status --wait
   
   # Connect clusters
   cilium clustermesh connect --destination-context cluster2
   ```

2. **Istio Multi-cluster**:
   ```yaml
   # Create multi-cluster service
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: remote-service
   spec:
     hosts:
     - remote-service.remote-namespace.global
     location: MESH_INTERNAL
     ports:
     - number: 80
       name: http
       protocol: HTTP
     resolution: DNS
     addresses:
     - 240.0.0.1
     endpoints:
     - address: cluster2-istio-gateway.example.com
       ports:
         http: 15443
   ```

### IPv6 in Kubernetes

Enable IPv6 support in Kubernetes:

1. **Cluster configuration**:
   ```yaml
   # Enable IPv6 in kubeadm
   apiVersion: kubeadm.k8s.io/v1beta2
   kind: ClusterConfiguration
   networking:
     serviceSubnet: "fd00::/108"
     podSubnet: "fd01::/48"
   featureGates:
     IPv6DualStack: true
   ```

2. **CNI configuration (Calico example)**:
   ```yaml
   apiVersion: projectcalico.org/v3
   kind: IPPool
   metadata:
     name: default-ipv6-ippool
   spec:
     cidr: fd01::/48
     natOutgoing: true
     nodeSelector: all()
   ```

3. **Dual-stack services**:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: dual-stack-service
   spec:
     ipFamilyPolicy: PreferDualStack
     ipFamilies:
     - IPv4
     - IPv6
     selector:
       app: MyApp
     ports:
     - port: 80
   ```

### Network Service Mesh (NSM)

NSM provides more flexible networking options:

```yaml
# Example NSM configuration
apiVersion: networkservicemesh.io/v1
kind: NetworkService
metadata:
  name: secure-intranet
spec:
  payload: IP
  matches:
  - source:
      selector:
        app: web
    route:
    - destination:
        selector:
          service: vpn-gateway
```

### Custom CNI Chaining

Chain multiple CNI plugins for combined functionality:

```json
{
  "cniVersion": "0.3.1",
  "name": "custom-network",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "__KUBERNETES_NODE_NAME__",
      "ipam": {
        "type": "calico-ipam"
      }
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true},
      "bandwidth": {
        "ingressRate": 1000000,
        "ingressBurst": 1000000,
        "egressRate": 1000000,
        "egressBurst": 1000000
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true},
      "snat": true
    }
  ]
}
```

## Summary

Kubernetes networking provides a robust foundation for container communication while offering flexibility through CNI plugins, service abstractions, and network policies. Understanding these components helps you design, implement, and troubleshoot network architectures that meet your application requirements.

Key takeaways:
- Kubernetes uses an IP-per-pod model
- Services provide stable endpoints for pods
- CNI plugins implement pod networking
- NetworkPolicies provide security controls
- Various plugins offer different performance/feature tradeoffs

By mastering these concepts, you can build scalable, secure, and high-performance Kubernetes network architectures.

## Further Reading

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Network Policy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)