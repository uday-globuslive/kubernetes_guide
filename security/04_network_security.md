# Network Security in Kubernetes

## Introduction

Network security in Kubernetes is a critical aspect of maintaining a secure ELK Stack deployment. This document covers Kubernetes Network Policies, service meshes, ingress security, and other network security considerations when running Elasticsearch, Logstash, and Kibana in Kubernetes environments. Properly configured network security ensures that your ELK Stack components communicate securely and are protected from unauthorized access.

## Kubernetes Network Policies

Network Policies are Kubernetes resources that control the traffic flow between pods, acting as a firewall for your applications.

### Basic Network Policy Concepts

- **Ingress**: Controls incoming traffic to selected pods
- **Egress**: Controls outgoing traffic from selected pods
- **Selectors**: Determine which pods the policy applies to
- **Rules**: Define allowed traffic patterns

### Example: Elasticsearch Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
  namespace: elk-stack
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    # Allow Kibana to connect to Elasticsearch
    - podSelector:
        matchLabels:
          app: kibana
    # Allow Logstash to connect to Elasticsearch
    - podSelector:
        matchLabels:
          app: logstash
    ports:
    - protocol: TCP
      port: 9200  # HTTP API
    - protocol: TCP
      port: 9300  # Transport
  egress:
  - to:
    # Allow communication between Elasticsearch nodes
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9300
```

### Example: Kibana Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kibana-network-policy
  namespace: elk-stack
spec:
  podSelector:
    matchLabels:
      app: kibana
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow access from ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 5601
  egress:
  # Allow access to Elasticsearch
  - to:
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9200
```

### Example: Logstash Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: logstash-network-policy
  namespace: elk-stack
spec:
  podSelector:
    matchLabels:
      app: logstash
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow incoming logs from application pods
  - from:
    - namespaceSelector:
        matchLabels:
          logs: enabled
    ports:
    - protocol: TCP
      port: 5044  # Beats
    - protocol: TCP
      port: 9600  # API
  egress:
  # Allow connection to Elasticsearch
  - to:
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9200
```

### Default Deny Policy

It's a best practice to implement a default deny policy for namespaces containing sensitive applications like Elasticsearch:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: elk-stack
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Network Security with Service Meshes

Service meshes like Istio, Linkerd, or Consul provide advanced network security capabilities beyond basic Network Policies.

### Istio Security Features for ELK Stack

#### Mutual TLS (mTLS)

Enforce mutual TLS authentication between all ELK components:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: elk-peer-auth
  namespace: elk-stack
spec:
  mtls:
    mode: STRICT
```

#### Authorization Policy

Control service-to-service communication with fine-grained policies:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: elasticsearch-authz
  namespace: elk-stack
spec:
  selector:
    matchLabels:
      app: elasticsearch
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/elk-stack/sa/kibana"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/elasticsearch/*"]
  - from:
    - source:
        principals: ["cluster.local/ns/elk-stack/sa/logstash"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/elasticsearch/*"]
```

### Setting Up Linkerd for ELK Stack

Linkerd is a lightweight service mesh that can be used to secure ELK Stack communications:

```bash
# Install Linkerd CLI
curl -sL run.linkerd.io/install | sh

# Add Linkerd annotations to ELK Stack deployments
kubectl get deploy -n elk-stack -o yaml | linkerd inject - | kubectl apply -f -

# View mesh metrics
linkerd -n elk-stack stat deploy
```

## TLS Termination and Ingress Security

### Secure Kibana Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: elk-stack
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - kibana.example.com
    secretName: kibana-tls
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

### Creating TLS Certificates with cert-manager

```yaml
# First install cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.10.0/cert-manager.yaml

# Create a ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

## Configuring Elasticsearch Transport Security

Transport Layer Security between Elasticsearch nodes is essential in Kubernetes deployments:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elk-stack
data:
  elasticsearch.yml: |
    cluster.name: elk-k8s
    network.host: 0.0.0.0
    
    # Node-to-node encryption
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    
    # HTTP encryption
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

### Generate Elasticsearch Certificates

```bash
# Generate CA
elasticsearch-certutil ca --out /tmp/elastic-stack-ca.p12 --pass ""

# Generate certificates
elasticsearch-certutil cert --ca /tmp/elastic-stack-ca.p12 --ca-pass "" \
  --name elasticsearch --out /tmp/elastic-certificates.p12 --pass ""

# Create Kubernetes secret
kubectl create secret generic elastic-certificates \
  --from-file=/tmp/elastic-certificates.p12 \
  --namespace elk-stack
```

## Securing Elasticsearch Client Access

### Using Client Certificates

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kibana-elasticsearch-credentials
  namespace: elk-stack
type: Opaque
data:
  kibana.keystore.p12: <base64-encoded-keystore>
  kibana.truststore.p12: <base64-encoded-truststore>
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk-stack
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch:9200"]
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    elasticsearch.ssl.certificate: "/usr/share/kibana/config/certs/kibana.crt"
    elasticsearch.ssl.key: "/usr/share/kibana/config/certs/kibana.key"
    elasticsearch.ssl.verificationMode: certificate
```

## Kubernetes CNI Network Security

The Container Network Interface (CNI) plugin you choose affects your network security capabilities.

### Calico Network Security

Calico is a popular CNI that offers advanced network policy features:

```yaml
# More detailed Calico network policy for Elasticsearch
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
  namespace: elk-stack
spec:
  selector: app == 'elasticsearch'
  types:
  - Ingress
  - Egress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == 'kibana' || app == 'logstash'
    destination:
      ports:
      - 9200
      - 9300
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: app == 'elasticsearch'
      ports:
      - 9300
```

### Cilium Network Security

Cilium is a CNI that provides eBPF-based networking and security:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: elasticsearch-cilium-policy
  namespace: elk-stack
spec:
  endpointSelector:
    matchLabels:
      app: elasticsearch
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: kibana
    - matchLabels:
        app: logstash
    toPorts:
    - ports:
      - port: "9200"
        protocol: TCP
      - port: "9300"
        protocol: TCP
  egress:
  - toEndpoints:
    - matchLabels:
        app: elasticsearch
    toPorts:
    - ports:
      - port: "9300"
        protocol: TCP
```

## Encrypting Data in Transit

### Configuring Filebeat to Use TLS with Logstash

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      
    output.logstash:
      hosts: ["logstash.elk-stack.svc.cluster.local:5044"]
      ssl.certificate_authorities: ["/etc/filebeat/certs/ca.crt"]
      ssl.certificate: "/etc/filebeat/certs/filebeat.crt"
      ssl.key: "/etc/filebeat/certs/filebeat.key"
```

### Configuring Logstash to Accept TLS Connections

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk-stack
data:
  logstash.conf: |
    input {
      beats {
        port => 5044
        ssl => true
        ssl_certificate => "/etc/logstash/certs/logstash.crt"
        ssl_key => "/etc/logstash/certs/logstash.key"
        ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
        ssl_verify_mode => "force_peer"
      }
    }
    
    output {
      elasticsearch {
        hosts => ["https://elasticsearch:9200"]
        ssl => true
        ssl_certificate_verification => true
        cacert => "/etc/logstash/certs/ca.crt"
        user => "${ELASTICSEARCH_USERNAME}"
        password => "${ELASTICSEARCH_PASSWORD}"
      }
    }
```

## Network Policy Testing and Troubleshooting

### Testing Network Connectivity

```bash
# Create a temporary debugging pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash

# Test connections to Elasticsearch
curl -v telnet://elasticsearch.elk-stack.svc.cluster.local:9200
```

### Verifying Network Policies

```bash
# Check if network policies are enforced
kubectl get networkpolicies -n elk-stack

# Check if your CNI supports network policies
kubectl get pods -n kube-system | grep -i cni

# View NetworkPolicy details
kubectl describe networkpolicy elasticsearch-network-policy -n elk-stack
```

## Network Security Best Practices for ELK on Kubernetes

1. **Defense in Depth**:
   - Implement network policies at multiple levels
   - Use service meshes for additional security layers
   - Secure both pod-to-pod and external communications

2. **Principle of Least Connectivity**:
   - Only allow necessary communication paths
   - Implement default deny policies and explicitly allow required traffic
   - Regularly audit network connections

3. **Encryption Everywhere**:
   - Use TLS for all ELK component communications
   - Implement mutual TLS where possible
   - Rotate certificates regularly

4. **Regular Security Audits**:
   - Use tools like `kube-hunter` and `kube-bench` to identify weaknesses
   - Perform regular penetration testing
   - Audit network policy configurations

5. **Logging and Monitoring**:
   - Enable network flow logs
   - Monitor for unusual traffic patterns
   - Set up alerts for policy violations or unauthorized connection attempts

## Network Security Tools

### kube-hunter

kube-hunter is a tool for finding security weaknesses in Kubernetes clusters:

```bash
# Run kube-hunter
kubectl run kube-hunter --restart=Never --image=aquasec/kube-hunter:latest -- --pod
kubectl logs kube-hunter
```

### Cilium Network Policy Editor

For Cilium users, there's a web-based editor for Cilium network policies:

```bash
# Install the Cilium Network Policy Editor
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/master/examples/kubernetes/addons/network-policy-editor/editor-service.yaml

# Access the editor
kubectl port-forward -n kube-system svc/network-policy-editor 8080:80
```

## Conclusion

Network security is a fundamental aspect of running a secure ELK Stack on Kubernetes. By implementing Kubernetes Network Policies, configuring TLS for all communications, and following security best practices, you can create a robust security posture for your Elasticsearch, Logstash, and Kibana deployments. Remember that network security should be part of a comprehensive security strategy that includes pod security, authentication, authorization, and regular security audits.

## References

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Network Security](https://docs.projectcalico.org/security/calico-network-policy)
- [Istio Security](https://istio.io/latest/docs/concepts/security/)
- [Elastic Stack Security Features](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)
- [Securing Kubernetes Clusters](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)