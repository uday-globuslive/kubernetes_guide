# Network Policies

Network Policies are Kubernetes resources that control the flow of network traffic between pods, namespaces, and external endpoints. They act as a firewall, allowing you to implement the principle of least privilege for network communication.

## Introduction to Network Policies

By default, all pods in a Kubernetes cluster can communicate with any other pod and external endpoints without restriction. Network Policies introduce restrictions on this communication, providing a way to:

- Isolate applications from each other
- Restrict traffic to/from specific namespaces
- Limit outbound traffic to certain endpoints
- Implement multi-tier application segmentation
- Enforce regulatory and security requirements

Network Policies are namespace-scoped resources that select pods and define rules which specify what traffic is allowed to and from those pods.

## Prerequisites

Not all Kubernetes clusters support Network Policies by default. You need:

1. A network plugin that supports Network Policies:
   - Calico
   - Cilium
   - Antrea
   - Weave Net
   - Kube-router
   - Amazon VPC CNI (with additional components)
   - Azure CNI
   - Google Kubernetes Engine with NetworkPolicy enabled

2. Proper RBAC permissions to create NetworkPolicy resources

To check if your cluster supports Network Policies:

```bash
# Create a test policy
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Check if it was created successfully
kubectl get networkpolicy test-policy

# Clean up the test policy
kubectl delete networkpolicy test-policy
```

## Network Policy Structure

A Network Policy consists of:

1. **Pod selector**: Identifies the pods to which the policy applies
2. **Policy types**: Specifies whether the policy applies to ingress, egress, or both
3. **Ingress rules**: Defines allowed incoming traffic
4. **Egress rules**: Defines allowed outgoing traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
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
          role: backend
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
      port: 5000
```

## Pod Selectors

The `podSelector` field determines which pods the NetworkPolicy applies to.

### Select All Pods in a Namespace

```yaml
spec:
  podSelector: {}  # Empty selector matches all pods
```

### Select Specific Pods

```yaml
spec:
  podSelector:
    matchLabels:
      app: api
      tier: backend
```

### Advanced Selection with Expressions

```yaml
spec:
  podSelector:
    matchExpressions:
    - key: app
      operator: In
      values: ["api", "worker"]
    - key: environment
      operator: NotIn
      values: ["test"]
```

## Basic Network Policy Types

### Default Deny All Ingress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### Default Deny All Egress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Default Deny All Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow All Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-traffic
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}  # Empty rule allows all ingress traffic
  egress:
  - {}  # Empty rule allows all egress traffic
```

## Ingress Policy Rules

Ingress rules control incoming traffic to the selected pods.

### Allow Traffic from Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Traffic from Multiple Sources (OR Condition)

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  - podSelector:
      matchLabels:
        app: monitoring
```

This allows traffic from pods with either `app: frontend` OR `app: monitoring`.

### Allow Traffic from Specific Namespaces

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        purpose: production
```

### Allow Traffic from Specific Pods in Specific Namespaces (AND Condition)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        purpose: production
    podSelector:
      matchLabels:
        app: frontend
```

This allows traffic only from pods with `app: frontend` in namespaces labeled `purpose: production`.

### Allow Traffic from Specific IP Blocks

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 172.17.0.0/16
      except:
      - 172.17.1.0/24
```

### Allow Traffic to Specific Ports

```yaml
ingress:
- ports:
  - protocol: TCP
    port: 8080
  - protocol: TCP
    port: 8443
```

### Combine Multiple Conditions

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  - namespaceSelector:
      matchLabels:
        purpose: monitoring
  ports:
  - protocol: TCP
    port: 8080
```

## Egress Policy Rules

Egress rules control outgoing traffic from the selected pods.

### Allow Traffic to Specific Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-to-db
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

### Allow Traffic to Specific Namespaces

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        purpose: services
```

### Allow Traffic to External Services

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
```

### Allow DNS Resolution

This is critical for most applications to function properly:

```yaml
egress:
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

## Named Ports

You can refer to named ports defined in your pod specifications:

```yaml
# Pod definition with named ports
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    app: api
spec:
  containers:
  - name: api
    image: api:1.0
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```

Then reference these named ports in the NetworkPolicy:

```yaml
ingress:
- ports:
  - protocol: TCP
    port: http  # Refers to the named port
  - protocol: TCP
    port: https
```

## Common Use Cases

### Three-Tier Web Application

```yaml
# Frontend policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  # Allow DNS
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
---
# Backend policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
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
---
# Database policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Allow DNS
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
```

### Microservices Architecture

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: microservice-auth
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: auth-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: user-db
    ports:
    - protocol: TCP
      port: 5432
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
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: microservice-gateway
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: service
    ports:
    - protocol: TCP
      port: 8080
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
```

### Cross-Namespace Communication

```yaml
# Allow traffic from the "web" namespace to the "api" namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-web-to-api
  namespace: api
spec:
  podSelector: {}  # Apply to all pods in the "api" namespace
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: web
```

### Allow Monitoring

```yaml
# Allow Prometheus to scrape metrics from all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus
  namespace: default
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090  # Metrics port
```

## Advanced Network Policy Patterns

### Using Multiple Network Policies

NetworkPolicies are additive. If multiple policies select the same pod, the pod is restricted by the union of those policies' ingress/egress rules.

```yaml
# First policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Second policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-monitoring
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

With these two policies, the API pods allow traffic from frontend pods on port 8080 AND from monitoring namespace pods on port 9090.

### Applying Policies to Namespace Scope

```yaml
# Apply baseline policy to dev namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dev-namespace-baseline
  namespace: dev
spec:
  podSelector: {}  # Selects all pods in the namespace
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: development
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          purpose: development
  # Allow DNS
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
```

### Combining Network Policy Types

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    - ipBlock:
        cidr: 10.10.0.0/16
    ports:
    - protocol: TCP
```

## Best Practices

1. **Start with a default deny policy**
   - Create policies to deny all ingress and egress traffic in each namespace
   - Then add targeted policies to allow necessary traffic

2. **Test policies in a non-production environment first**
   - NetworkPolicies can easily break application communication
   - Use a staging environment to validate policies

3. **Always allow DNS access**
   - Most applications need DNS resolution
   - Include a rule allowing UDP port 53 to the cluster's DNS service

4. **Use labels effectively**
   - Create consistent labeling schemes for pods and namespaces
   - Use role-based labels (frontend, backend, db) for easy policy targeting

5. **Document traffic flows**
   - Create a matrix of required communications
   - Document expected traffic patterns for each application component

6. **Use namespaceSelectors for coarse-grained control**
   - Organize applications into logical namespaces
   - Use namespace selectors for broader rules

7. **Use podSelectors for fine-grained control**
   - Apply specific rules to individual application components
   - Combine with namespaceSelectors for precision

8. **Include logging/monitoring tools**
   - Ensure network policies don't prevent metrics collection
   - Allow traffic to/from logging infrastructure

9. **Start specific and broaden as needed**
   - Begin with specific, targeted policies
   - Expand rules incrementally as issues arise

10. **Regular review and audit**
    - Periodically review and update network policies
    - Remove obsolete rules as applications evolve

## Troubleshooting Network Policies

### Common Issues

1. **Policy not taking effect**
   - CNI plugin might not support NetworkPolicies
   - RBAC permissions might be insufficient
   - Policy may not be selecting the intended pods

2. **Unexpected blocking of traffic**
   - Multiple policies may conflict
   - Default deny policies may need exceptions
   - DNS traffic might be blocked

3. **One-way communication**
   - Remember to consider both ingress and egress
   - Both sides need appropriate policies

### Debugging Steps

1. **Verify network policy support**
   ```bash
   # Create a test policy and confirm it appears
   kubectl create -f test-policy.yaml
   kubectl get networkpolicy
   ```

2. **Check pod labels match selectors**
   ```bash
   kubectl get pods --show-labels
   ```

3. **Test basic connectivity**
   ```bash
   # Create a temporary debug pod
   kubectl run temp-pod --rm -it --image=nicolaka/netshoot -- /bin/bash
   
   # Then inside the pod
   ping <service-name>
   curl <service-name>:<port>
   ```

4. **Examine policy details**
   ```bash
   kubectl describe networkpolicy <policy-name>
   ```

5. **Check if multiple policies are applied**
   ```bash
   kubectl get networkpolicy --selector=app=<app-name>
   ```

6. **Test with temporary permissive policy**
   ```bash
   # Create a temporary policy allowing all traffic
   kubectl apply -f allow-all-temp.yaml
   
   # Test if the application works
   # Then remove the temporary policy
   kubectl delete -f allow-all-temp.yaml
   ```

7. **Check CNI plugin logs**
   ```bash
   kubectl logs -n kube-system -l k8s-app=calico-node
   # or
   kubectl logs -n kube-system -l k8s-app=cilium
   ```

## Network Policy Limitations

1. **No Cluster-wide policies**
   - NetworkPolicies are namespace-scoped
   - For cluster-wide policies, third-party solutions like Calico's GlobalNetworkPolicy are needed

2. **Limited protocol support**
   - TCP, UDP, and SCTP are supported
   - ICMP (ping) cannot be explicitly controlled

3. **No stateful inspection**
   - NetworkPolicies do not maintain connection state
   - Each direction of traffic needs separate rules

4. **No support for layer 7 filtering**
   - Cannot filter based on HTTP paths, methods, etc.
   - Service mesh solutions like Istio are needed for layer 7 policies

5. **Complex policy composition**
   - Policies are additive and can be complex to reason about
   - Multiple overlapping policies can be difficult to debug

## Extended Network Policy Features with CNI Plugins

Different CNI plugins offer extended network policy capabilities beyond the Kubernetes standard:

### Calico

Calico offers advanced network policy features:

- **GlobalNetworkPolicy**: Cluster-wide policies
- **NetworkSet**: Reusable collections of IP addresses/CIDRs
- **Layer 7 filtering**: HTTP methods, paths, headers
- **ICMP type filtering**: Control specific ICMP traffic types

Example of a Calico GlobalNetworkPolicy:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-external-egress
spec:
  selector: all()
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      selector: has(kubernetes.io/metadata.name)
  - action: Deny
    destination:
      notSelector: has(kubernetes.io/metadata.name)
```

### Cilium

Cilium provides identity-based security with extensive features:

- **CiliumNetworkPolicy**: Extended network policies
- **Layer 7 filtering**: HTTP, Kafka, gRPC, etc.
- **DNS policies**: Filter by domain names
- **TLS-aware policies**: Filter based on TLS identity
- **Encryption**: Transparent encryption

Example of a Cilium Network Policy with L7 rules:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "api-l7-policy"
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/public"
```

### Antrea

Antrea provides extended network policy features:

- **ClusterNetworkPolicy**: Cluster-wide policies
- **Tier**: Hierarchical policy organization
- **Applied-to**: Direct application to specific endpoints

Example of an Antrea ClusterNetworkPolicy:

```yaml
apiVersion: security.antrea.tanzu.vmware.com/v1alpha1
kind: ClusterNetworkPolicy
metadata:
  name: cluster-default-deny
spec:
  priority: 1
  tier: securityops
  appliedTo:
  - namespaceSelector: {}
  ingress:
  - action: Drop
    from:
    - podSelector: {}
```

Network Policies are a fundamental security feature in Kubernetes that enable you to implement the principle of least privilege for network communication. By carefully designing policies for your applications, you can significantly reduce the attack surface and improve the security posture of your Kubernetes clusters.