# Ingress Controllers in Kubernetes

## Introduction to Ingress

In Kubernetes, an Ingress is an API object that manages external access to services within a cluster, typically HTTP and HTTPS. Unlike a simple Service resource that exposes a single service using NodePort or LoadBalancer, Ingress provides more sophisticated capabilities like path-based routing, name-based virtual hosting, TLS termination, and load balancing.

## Ingress vs. Service

Before diving into Ingress controllers, let's understand why they're necessary:

| Feature | Service (LoadBalancer) | Ingress |
|---------|----------------------|---------|
| L4 vs L7 | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Cost | One load balancer per service | Single load balancer for multiple services |
| Routing | No path-based routing | Supports path and host-based routing |
| TLS | Limited TLS support | Native TLS termination |
| Features | Limited | Advanced features like rewriting, redirects, etc. |

## Ingress Architecture

An Ingress system in Kubernetes consists of two main components:

1. **Ingress Resource**: A Kubernetes API object that defines the routing rules
2. **Ingress Controller**: A controller that reads the Ingress Resource and processes the rules

### Ingress Resource Definition

An Ingress resource defines rules for routing traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
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
  - host: elasticsearch.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-client
            port:
              number: 9200
  tls:
  - hosts:
    - kibana.example.com
    - elasticsearch.example.com
    secretName: elk-tls-cert
```

## Common Ingress Controllers

### NGINX Ingress Controller

The NGINX Ingress Controller is one of the most popular controllers due to its feature-rich capabilities, performance, and community support.

#### Installation with Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

#### NGINX Ingress Configuration for ELK Stack

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  annotations:
    # Basic settings
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Proxy settings for Elasticsearch and Kibana
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    
    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
    
    # Basic Auth for Elasticsearch (optional)
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  tls:
  - hosts:
    - kibana.example.com
    - elasticsearch.example.com
    secretName: elk-tls-cert
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
  - host: elasticsearch.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-client
            port:
              number: 9200
```

### Traefik Ingress Controller

Traefik is a modern HTTP reverse proxy and load balancer that integrates with Kubernetes.

#### Installation with Helm

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace
```

#### Traefik Ingress Configuration for ELK Stack

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    
    # Middleware for security headers
    traefik.ingress.kubernetes.io/router.middlewares: default-security-headers@kubernetescrd
spec:
  tls:
  - hosts:
    - kibana.example.com
    - elasticsearch.example.com
    secretName: elk-tls-cert
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
  - host: elasticsearch.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-client
            port:
              number: 9200
```

And define a middleware for security headers:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: security-headers
spec:
  headers:
    frameDeny: true
    browserXssFilter: true
    contentTypeNosniff: true
    stsSeconds: 31536000
    stsIncludeSubdomains: true
```

### AWS ALB Ingress Controller

The AWS Application Load Balancer (ALB) Ingress Controller is designed specifically for Kubernetes clusters running on AWS.

#### Installation

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=your-cluster-name \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### ALB Ingress Configuration for ELK Stack

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/certificate-id
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
  - host: elasticsearch.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-client
            port:
              number: 9200
```

## Security Considerations for ELK Stack Ingress

### 1. TLS Encryption

Always secure your Elasticsearch and Kibana endpoints with TLS certificates:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
spec:
  tls:
  - hosts:
    - kibana.example.com
    - elasticsearch.example.com
    secretName: elk-tls-cert
  # rest of ingress definition...
```

Create the TLS secret:

```bash
kubectl create secret tls elk-tls-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

Or use cert-manager for automated certificate management:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: elk-cert
  namespace: default
spec:
  secretName: elk-tls-cert
  dnsNames:
  - kibana.example.com
  - elasticsearch.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### 2. Authentication

Add additional authentication layers to protect your ELK stack:

#### Basic Authentication

```yaml
# Create auth secret
kubectl create secret generic basic-auth \
  --from-file=auth=./auth

# In Ingress NGINX
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```

#### OAuth2 Proxy

For more advanced authentication, use OAuth2 Proxy with providers like Google, GitHub, or Okta:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://$host/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://$host/oauth2/start?rd=$escaped_request_uri"
```

### 3. Network Policies

Restrict traffic to your ELK components with network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
      role: client
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ingress-nginx
    ports:
    - protocol: TCP
      port: 9200
```

## Advanced Ingress Patterns for ELK Stack

### 1. Path-Based Routing

Route different paths to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
spec:
  rules:
  - host: elk.example.com
    http:
      paths:
      - path: /kibana
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-client
            port:
              number: 9200
```

With rewrite to remove the path prefix:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: elk.example.com
    http:
      paths:
      - path: /kibana(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
```

### 2. Multi-Tenant ELK Stack

For a multi-tenant setup, you can route different hosts to different ELK stacks:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tenant-elk-ingress
spec:
  rules:
  - host: team-a-kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: team-a-kibana
            port:
              number: 5601
  - host: team-b-kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: team-b-kibana
            port:
              number: 5601
```

### 3. Canary Deployments

Use canary deployments to test new versions of ELK components:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana-v2
            port:
              number: 5601
```

## Performance Tuning for ELK Stack Ingress

### 1. Buffering and Timeouts

Elasticsearch queries and bulk uploads may take longer than default timeouts:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

### 2. Keep-Alive Connections

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "32"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"
    nginx.ingress.kubernetes.io/upstream-keepalive-requests: "100"
```

### 3. SSL Optimization

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-session-cache: "true"
    nginx.ingress.kubernetes.io/ssl-session-tickets: "false"
```

## Troubleshooting Ingress for ELK Stack

### Common Issues and Solutions

#### 1. 502 Bad Gateway

Possible causes and solutions:
- **Elasticsearch not ready**: Check Elasticsearch health and readiness
- **Proxy timeout**: Increase timeouts in the Ingress annotations
- **Memory pressure**: Elasticsearch may be OOM, check resources

#### 2. 403 Forbidden

Possible causes and solutions:
- **CORS issues**: Add CORS annotations to your Ingress
- **Network policies**: Check if network policies are blocking traffic
- **Authentication**: Ensure authentication is correctly configured

#### 3. TLS Errors

Possible causes and solutions:
- **Invalid certificate**: Check certificate expiration and validity
- **Mismatched hostname**: Ensure the certificate matches the hostname
- **TLS version mismatch**: Check TLS version compatibility

### Debugging Commands

```bash
# Check Ingress status
kubectl get ingress elk-ingress -o yaml

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check Ingress controller events
kubectl get events -n ingress-nginx

# Test connectivity from inside the cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -v http://elasticsearch-client:9200
```

## Monitoring Ingress for ELK Stack

### Prometheus Metrics

Most Ingress controllers expose Prometheus metrics. Configure your Ingress controller to expose them:

```yaml
# For NGINX Ingress
controller:
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "10254"
```

### Metricbeat Configuration

Configure Metricbeat to collect metrics from your Ingress controller:

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
      hosts: ["${NGINX_INGRESS_METRICS_SERVICE_HOST}:${NGINX_INGRESS_METRICS_SERVICE_PORT}"]
      metrics_path: /metrics
```

### Visualizing in Kibana

Create dashboards in Kibana to monitor your Ingress controller:
- Request rate and latency
- Error rates
- Connection statistics
- TLS handshake time

## Best Practices for ELK Stack Ingress

1. **Use TLS**: Always encrypt traffic to your ELK stack

2. **Authentication**: Implement strong authentication for Elasticsearch and Kibana

3. **Rate Limiting**: Protect your ELK stack from overwhelming traffic

4. **Monitoring**: Monitor your Ingress performance and error rates

5. **Readiness Probes**: Ensure Elasticsearch and Kibana are ready before sending traffic

6. **Horizontal Scaling**: Scale your Ingress controller based on traffic patterns

7. **Regular Updates**: Keep your Ingress controller updated for security and features

8. **Backup Ingress Configurations**: Include Ingress resources in your backup strategy

9. **Documentation**: Document your Ingress configuration and security measures

10. **Testing**: Regularly test your Ingress configuration, especially after changes

## Conclusion

Ingress controllers are a critical component in exposing your ELK stack securely and efficiently to users and applications. By choosing the right Ingress controller and properly configuring it with appropriate security, performance, and monitoring settings, you can ensure reliable access to your Elasticsearch and Kibana instances while protecting them from unauthorized access and overload. The examples and best practices in this guide should help you implement a robust Ingress solution for your ELK stack on Kubernetes.