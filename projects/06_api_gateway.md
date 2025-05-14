# Kubernetes API Gateway

Building and deploying a robust API Gateway on Kubernetes is essential for modern microservices architectures. This project guide walks through the complete implementation of an API Gateway architecture on Kubernetes, covering different gateway options, configurations, security patterns, and operational best practices.

## Introduction to API Gateways on Kubernetes

### What is an API Gateway?

An API Gateway serves as a single entry point for all client API calls to your application, routing requests to appropriate microservices, handling cross-cutting concerns like authentication, rate limiting, and providing a unified interface to clients.

Key responsibilities include:
- Request routing
- API composition
- Protocol translation
- Authentication and authorization
- Rate limiting and quotas
- Caching
- Monitoring and analytics
- Request/response transformation

### API Gateway Architecture in Kubernetes

```
                                  ┌──────────────────────────────────────────────────┐
                                  │                Kubernetes Cluster                │
                                  │                                                  │
                                  │                                                  │
                                  │                                                  │
┌─────────────┐                   │   ┌───────────┐      ┌──────────────────────┐   │
│             │                   │   │           │      │                      │   │
│   Clients   │────────►│         │   │ Ingress   │      │ API Gateway Service  │   │
│             │          Load     ├───►           ├─────►│                      │   │
└─────────────┘         Balancer │   │ Controller │      │ (Ambassador, Kong,   │   │
                                  │   │           │      │  Traefik, etc.)      │   │
                                  │   └───────────┘      └──────────┬───────────┘   │
                                  │                                  │               │
                                  │                                  │               │
                                  │   ┌──────────┐  ┌──────────┐  ┌─▼────────┐      │
                                  │   │          │  │          │  │          │      │
                                  │   │ Service A│  │ Service B│  │ Service C│      │
                                  │   │          │  │          │  │          │      │
                                  │   └──────────┘  └──────────┘  └──────────┘      │
                                  │                                                  │
                                  │                                                  │
                                  └──────────────────────────────────────────────────┘
```

### Kubernetes API Gateway Options

Several API Gateways are widely used in Kubernetes environments:

1. **Kong**: Feature-rich API gateway built on NGINX
2. **Ambassador/Emissary**: Control plane for Envoy focused on Kubernetes
3. **Istio Gateway**: Part of the Istio service mesh
4. **NGINX Ingress Controller**: Configurable as an API Gateway
5. **Traefik**: Cloud-native edge router with API gateway features
6. **Gloo**: API Gateway built on Envoy proxy
7. **AWS API Gateway + AWS Load Balancer Controller**: For AWS environments

Each has different strengths. This guide will focus primarily on Kong, Ambassador, and Istio Gateway, representing different approaches to API Gateway implementation.

## Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured to access your cluster
- Helm 3
- Kubernetes Ingress controller (if not using a Gateway with its own ingress)
- Domain name for API Gateway (optional but recommended)

## Implementation

### Step 1: Choose Your API Gateway Solution

Let's evaluate the top options to help you choose:

| Gateway | Key Strengths | When to Use |
|---------|--------------|-------------|
| Kong | Comprehensive plugin ecosystem, flexible auth, traditional API Management features | Complex API management requirements, need for many plugins |
| Ambassador/Emissary | Developer-centric, decentralized configuration, Kubernetes-native | Microservices environments, teams manage their own services |
| Istio Gateway | Integrated with service mesh, advanced traffic management | Already using Istio service mesh, need advanced networking features |
| NGINX Ingress | Lightweight, familiar configuration, widely deployed | Simpler requirements, NGINX experience on team |
| Traefik | Auto-discovery, multiple providers, middleware | Cloud-native environments, need for automatic service discovery |

For this guide, we'll implement each of the top three solutions to demonstrate different approaches.

### Step 2: Install and Configure Kong API Gateway

Kong is a powerful, plugin-oriented API gateway built on NGINX.

#### Installing Kong via Helm

```bash
# Add Kong Helm repository
helm repo add kong https://charts.konghq.com
helm repo update

# Create namespace
kubectl create namespace kong

# Install Kong with Helm
helm install kong kong/kong -n kong \
  --set ingressController.installCRDs=false \
  --set admin.enabled=true \
  --set admin.http.enabled=true \
  --set proxy.type=LoadBalancer
```

#### Verify Installation

```bash
# Check if pods are running
kubectl get pods -n kong

# Get the Kong proxy service
kubectl get service -n kong kong-kong-proxy

# Store the Kong Gateway IP/hostname for later use
export KONG_GATEWAY=$(kubectl get svc -n kong kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Kong Gateway available at: $KONG_GATEWAY"
```

#### Configure a Basic Service and Route

Create a test backend service first:

```yaml
# echo-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 80
```

Apply this to your cluster:

```bash
kubectl apply -f echo-service.yaml
```

Now, configure Kong to route to this service:

```yaml
# kong-echo-api.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-echo-api.yaml
```

Test the endpoint:

```bash
curl http://$KONG_GATEWAY/echo
```

#### Configure Kong Plugins

Kong's strength is its extensive plugin ecosystem. Let's add rate limiting and key authentication:

```yaml
# kong-plugins.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
config:
  minute: 5
  policy: local
plugin: rate-limiting
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: key-auth
plugin: key-auth
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: rate-limiting, key-auth
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply these changes:

```bash
kubectl apply -f kong-plugins.yaml
```

Create a consumer and key:

```yaml
# kong-consumer.yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: example-user
  annotations:
    kubernetes.io/ingress.class: kong
username: example-user
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: example-user
credentials:
- example-key
```

Apply the consumer configuration:

```bash
kubectl apply -f kong-consumer.yaml
```

Test with authentication:

```bash
# This should fail
curl http://$KONG_GATEWAY/echo

# This should work
curl http://$KONG_GATEWAY/echo -H 'apikey: example-key'
```

### Step 3: Install and Configure Ambassador/Emissary API Gateway

Ambassador (now called Emissary Ingress) is a Kubernetes-native API Gateway built on the Envoy proxy.

#### Installing Ambassador via Helm

```bash
# Add Emissary-ingress Helm repository
helm repo add datawire https://app.getambassador.io
helm repo update

# Create namespace
kubectl create namespace ambassador

# Install Ambassador
helm install ambassador datawire/emissary-ingress -n ambassador \
  --set service.type=LoadBalancer
```

#### Verify Installation

```bash
# Check pods
kubectl get pods -n ambassador

# Get the Gateway address
export AMBASSADOR_GATEWAY=$(kubectl get svc -n ambassador ambassador -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ambassador Gateway available at: $AMBASSADOR_GATEWAY"
```

#### Configure Ambassador Mapping

Ambassador uses Custom Resources called `Mapping` to define routes:

```yaml
# ambassador-echo-api.yaml
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: echo-mapping
spec:
  hostname: "*"
  prefix: /echo/
  service: echo
  rewrite: ""
```

Apply the mapping:

```bash
kubectl apply -f ambassador-echo-api.yaml
```

Test the endpoint:

```bash
curl http://$AMBASSADOR_GATEWAY/echo/
```

#### Configure Authentication with Ambassador

Ambassador supports various authentication mechanisms. Let's implement API key authentication:

```yaml
# ambassador-auth.yaml
apiVersion: getambassador.io/v3alpha1
kind: Filter
metadata:
  name: api-key-filter
spec:
  type: external
  config:
    auth_service: "auth-service"
    proto: http
    timeout_ms: 5000
    status_on_error:
      code: 503
---
apiVersion: getambassador.io/v3alpha1
kind: FilterPolicy
metadata:
  name: api-key-policy
spec:
  rules:
  - host: "*"
    path: /echo/
    filters:
    - name: api-key-filter
```

Deploy a simple auth service:

```yaml
# auth-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: curlimages/curl:7.82.0
        command: ["/bin/sh", "-c"]
        args:
          - |
            cat > app.py << 'EOF'
            from flask import Flask, request, jsonify
            import os
            
            app = Flask(__name__)
            
            VALID_API_KEYS = ["valid-api-key-1", "valid-api-key-2"]
            
            @app.route('/', methods=['GET'])
            def auth():
                api_key = request.headers.get('X-Api-Key', '')
                if api_key in VALID_API_KEYS:
                    return '', 200
                return jsonify({"error": "Invalid API key"}), 401
            
            if __name__ == '__main__':
                app.run(host='0.0.0.0', port=3000)
            EOF
            pip install flask
            python app.py
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - port: 80
    targetPort: 3000
```

Apply these configurations:

```bash
kubectl apply -f auth-service.yaml
kubectl apply -f ambassador-auth.yaml
```

Test the authenticated API:

```bash
# This should fail
curl http://$AMBASSADOR_GATEWAY/echo/

# This should work
curl http://$AMBASSADOR_GATEWAY/echo/ -H 'X-Api-Key: valid-api-key-1'
```

### Step 4: Install and Configure Istio Gateway

Istio provides a Gateway component that integrates with its service mesh capabilities.

#### Installing Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Add istioctl to your path
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with default profile
istioctl install --set profile=default -y

# Label namespace for Istio injection
kubectl label namespace default istio-injection=enabled
```

#### Create an Istio Gateway and Virtual Service

```yaml
# istio-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: api-gateway
spec:
  selector:
    istio: ingressgateway # Use the default Istio ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-vs
spec:
  hosts:
  - "*"
  gateways:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: /echo
    rewrite:
      uri: /
    route:
    - destination:
        host: echo
        port:
          number: 80
```

Apply the gateway configuration:

```bash
kubectl apply -f istio-gateway.yaml
```

#### Get the Istio Ingress Gateway address

```bash
export ISTIO_GATEWAY=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Istio Gateway available at: $ISTIO_GATEWAY"
```

Test the endpoint:

```bash
curl http://$ISTIO_GATEWAY/echo
```

#### Configure Authentication with Istio

Istio provides robust authentication options. Let's set up JWT authentication:

```yaml
# istio-auth.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.13/security/tools/jwt/samples/jwks.json"
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
        paths: ["/echo"]
```

Apply the authentication configuration:

```bash
kubectl apply -f istio-auth.yaml
```

Test with JWT token:

```bash
# Get a sample token for testing
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.13/security/tools/jwt/samples/demo.jwt -s)

# This should fail
curl http://$ISTIO_GATEWAY/echo

# This should work
curl --header "Authorization: Bearer $TOKEN" http://$ISTIO_GATEWAY/echo
```

## Step 5: Implementing Advanced API Gateway Features

Let's implement some advanced features that are common requirements for API gateways:

### Rate Limiting

#### Rate Limiting with Kong

```yaml
# kong-rate-limit.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: advanced-rate-limiting
config:
  minute: 10
  hour: 100
  limit_by: ip
  policy: local
  hide_client_headers: false
plugin: rate-limiting
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: advanced-rate-limiting
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-rate-limit.yaml
```

#### Rate Limiting with Istio

```yaml
# istio-rate-limit.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-local-ratelimit
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 10
                tokens_per_fill: 10
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: 'true'
```

Apply this configuration:

```bash
kubectl apply -f istio-rate-limit.yaml
```

### API Versioning and Canary Deployments

Let's deploy two versions of our service and implement canary routing:

```yaml
# echo-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-v2
  labels:
    app: echo
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      version: v2
  template:
    metadata:
      labels:
        app: echo
        version: v2
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        env:
        - name: VERSION
          value: "v2"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo-v2
spec:
  selector:
    app: echo
    version: v2
  ports:
  - port: 80
    targetPort: 80
```

Apply this to create the v2 service:

```bash
kubectl apply -f echo-v2.yaml
```

#### Canary with Istio

```yaml
# istio-canary.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-vs
spec:
  hosts:
  - "*"
  gateways:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: /echo
      headers:
        x-api-version:
          exact: v2
    rewrite:
      uri: /
    route:
    - destination:
        host: echo-v2
        port:
          number: 80
  - match:
    - uri:
        prefix: /echo
    rewrite:
      uri: /
    route:
    - destination:
        host: echo
        port:
          number: 80
        weight: 90
    - destination:
        host: echo-v2
        port:
          number: 80
        weight: 10
```

Apply the canary configuration:

```bash
kubectl apply -f istio-canary.yaml
```

Test the canary routing:

```bash
# Regular request (90% v1, 10% v2)
curl http://$ISTIO_GATEWAY/echo

# Force v2 with header
curl http://$ISTIO_GATEWAY/echo -H 'x-api-version: v2'
```

### Request/Response Transformation

#### Kong Transformation

```yaml
# kong-transform.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-transformer
config:
  add:
    headers:
    - X-Consumer-ID:$(uuid)
    - X-Request-ID:$(uuid)
    - X-Api-Version:v1
plugin: request-transformer
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: response-transformer
config:
  add:
    headers:
    - X-Response-Time:$(current_timestamp)
    - X-Api-Gateway:Kong
    json:
    - api_version:v1
plugin: response-transformer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: request-transformer, response-transformer
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply these transformations:

```bash
kubectl apply -f kong-transform.yaml
```

### API Documentation

Let's add API documentation with Kong's Swagger plugin:

```yaml
# kong-swagger.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: swagger-docs
plugin: swagger-ui
config:
  url: https://petstore.swagger.io/v2/swagger.json
  uiVersion: 3.x
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-docs
  annotations:
    konghq.com/plugins: swagger-docs
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /api-docs
        pathType: Prefix
        backend:
          service:
            name: kong-kong-proxy
            port:
              number: 80
```

Apply the docs configuration:

```bash
kubectl apply -f kong-swagger.yaml
```

Access the API docs at:

```
http://$KONG_GATEWAY/api-docs
```

## Step 6: Monitoring and Observability

### Installing Prometheus and Grafana

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Add Prometheus helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring

# Install Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminPassword=admin
```

### Kong Monitoring

Kong exposes Prometheus metrics that can be scraped. Configure a ServiceMonitor:

```yaml
# kong-monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kong-servicemonitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kong
  namespaceSelector:
    matchNames:
    - kong
  endpoints:
  - port: metrics
    interval: 15s
```

Apply this configuration:

```bash
kubectl apply -f kong-monitoring.yaml
```

### Grafana Dashboard for Kong

Get the Grafana admin password:

```bash
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

Port-forward Grafana:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Access Grafana at http://localhost:3000 and import the Kong dashboard (ID: 7424).

### Logging with Elasticsearch, Fluentd, and Kibana (EFK Stack)

```bash
# Install EFK stack
kubectl create namespace logging

# Add Elastic helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=512Mi

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging

# Install Fluentd
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch-rbac.yaml
```

## Step 7: Security Best Practices

### TLS Encryption

Let's secure our API Gateway with TLS:

```yaml
# First, create a self-signed certificate for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout api-gateway.key -out api-gateway.crt \
  -subj "/CN=api-gateway.example.com"

# Create a TLS secret
kubectl create secret tls api-gateway-tls \
  --key api-gateway.key \
  --cert api-gateway.crt
```

#### TLS with Kong

```yaml
# kong-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress-tls
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  tls:
  - secretName: api-gateway-tls
    hosts:
    - api-gateway.example.com
  rules:
  - host: api-gateway.example.com
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-tls.yaml
```

#### TLS with Istio

```yaml
# istio-tls.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: api-gateway-tls
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-gateway-tls
    hosts:
    - "api-gateway.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-vs-tls
spec:
  hosts:
  - "api-gateway.example.com"
  gateways:
  - api-gateway-tls
  http:
  - match:
    - uri:
        prefix: /echo
    rewrite:
      uri: /
    route:
    - destination:
        host: echo
        port:
          number: 80
```

Apply this configuration:

```bash
kubectl apply -f istio-tls.yaml
```

### OAuth2 Authentication

Let's implement OAuth2 with Kong:

```yaml
# kong-oauth2.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: oauth2
plugin: oauth2
config:
  enable_authorization_code: true
  mandatory_scope: true
  provision_key: "my-oauth2-provision-key"
  token_expiration: 86400
  scopes:
  - "read"
  - "write"
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: oauth2-client
username: oauth2-client
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: oauth2-client
credentials:
- oauth2-client
---
apiVersion: secretgen.k8s.io/v1
kind: OAuth2Credential
metadata:
  name: oauth2-client
  labels:
    konghq.com/consumer: oauth2-client
clientId: "my-client-id"
clientSecret: "my-client-secret"
redirectUris:
- https://example.com/callback
name: "My OAuth2 Client"
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress-oauth2
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: oauth2
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo-secure
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-oauth2.yaml
```

### CORS Configuration

```yaml
# kong-cors.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: cors
plugin: cors
config:
  origins:
  - https://example.com
  - https://app.example.com
  methods:
  - GET
  - POST
  - PUT
  - DELETE
  headers:
  - Authorization
  - Content-Type
  - Accept
  credentials: true
  max_age: 3600
  preflight_continue: false
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress-cors
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: cors
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo-cors
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-cors.yaml
```

## Step 8: Advanced Patterns

### API Composition (Aggregation)

Let's create a composite API that combines responses from multiple services:

```yaml
# First, create a few more echo services with different data
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-users
  labels:
    app: echo-users
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-users
  template:
    metadata:
      labels:
        app: echo-users
    spec:
      containers:
      - name: echo
        image: mendhak/http-https-echo:26
        env:
        - name: ECHO_DATA
          value: '{"users": [{"id": 1, "name": "User 1"}, {"id": 2, "name": "User 2"}]}'
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo-users
spec:
  selector:
    app: echo-users
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-products
  labels:
    app: echo-products
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-products
  template:
    metadata:
      labels:
        app: echo-products
    spec:
      containers:
      - name: echo
        image: mendhak/http-https-echo:26
        env:
        - name: ECHO_DATA
          value: '{"products": [{"id": 1, "name": "Product 1"}, {"id": 2, "name": "Product 2"}]}'
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo-products
spec:
  selector:
    app: echo-products
  ports:
  - port: 80
    targetPort: 80
EOF
```

Deploy an API composition service:

```yaml
# api-composer.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-composer
  labels:
    app: api-composer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-composer
  template:
    metadata:
      labels:
        app: api-composer
    spec:
      containers:
      - name: composer
        image: node:16-alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            # Install dependencies
            npm init -y
            npm install express axios lodash
            
            # Create app.js
            cat > app.js << 'EOF'
            const express = require('express');
            const axios = require('axios');
            const _ = require('lodash');
            
            const app = express();
            const port = 3000;
            
            app.get('/composite', async (req, res) => {
              try {
                // Get data from multiple services
                const [usersResponse, productsResponse] = await Promise.all([
                  axios.get('http://echo-users'),
                  axios.get('http://echo-products')
                ]);
                
                // Extract data from responses
                const userData = usersResponse.data.data ? JSON.parse(usersResponse.data.data) : {};
                const productData = productsResponse.data.data ? JSON.parse(productsResponse.data.data) : {};
                
                // Combine the data
                const result = {
                  users: userData.users || [],
                  products: productData.products || [],
                  metadata: {
                    userCount: (userData.users || []).length,
                    productCount: (productData.products || []).length,
                    timestamp: new Date().toISOString()
                  }
                };
                
                res.json(result);
              } catch (error) {
                console.error('Error:', error.message);
                res.status(500).json({ error: 'Failed to fetch data from services' });
              }
            });
            
            app.listen(port, () => {
              console.log(`API composer listening at http://localhost:${port}`);
            });
            EOF
            
            # Run the app
            node app.js
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api-composer
spec:
  selector:
    app: api-composer
  ports:
  - port: 80
    targetPort: 3000
```

Apply this configuration:

```bash
kubectl apply -f api-composer.yaml
```

Now, expose this via the API gateway:

```yaml
# kong-composer.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: composer-ingress
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /composite
        pathType: Prefix
        backend:
          service:
            name: api-composer
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-composer.yaml
```

Test the composite API:

```bash
curl http://$KONG_GATEWAY/composite
```

### Circuit Breaking

Istio provides built-in circuit breaking capabilities:

```yaml
# istio-circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo-circuit-breaker
spec:
  host: echo
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
```

Apply the circuit breaker configuration:

```bash
kubectl apply -f istio-circuit-breaker.yaml
```

### GraphQL API Gateway

Let's set up a simple GraphQL API gateway:

```yaml
# graphql-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graphql-gateway
  labels:
    app: graphql-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: graphql-gateway
  template:
    metadata:
      labels:
        app: graphql-gateway
    spec:
      containers:
      - name: graphql
        image: node:16-alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            # Install dependencies
            npm init -y
            npm install apollo-server graphql axios
            
            # Create index.js
            cat > index.js << 'EOF'
            const { ApolloServer, gql } = require('apollo-server');
            const axios = require('axios');
            
            // GraphQL schema
            const typeDefs = gql`
              type User {
                id: Int
                name: String
              }
              
              type Product {
                id: Int
                name: String
              }
              
              type Query {
                users: [User]
                products: [Product]
                user(id: Int!): User
                product(id: Int!): Product
              }
            `;
            
            // Resolvers
            const resolvers = {
              Query: {
                users: async () => {
                  const response = await axios.get('http://echo-users');
                  const data = response.data.data ? JSON.parse(response.data.data) : {};
                  return data.users || [];
                },
                products: async () => {
                  const response = await axios.get('http://echo-products');
                  const data = response.data.data ? JSON.parse(response.data.data) : {};
                  return data.products || [];
                },
                user: async (_, { id }) => {
                  const response = await axios.get('http://echo-users');
                  const data = response.data.data ? JSON.parse(response.data.data) : {};
                  return (data.users || []).find(user => user.id === id);
                },
                product: async (_, { id }) => {
                  const response = await axios.get('http://echo-products');
                  const data = response.data.data ? JSON.parse(response.data.data) : {};
                  return (data.products || []).find(product => product.id === id);
                }
              }
            };
            
            // Create Apollo Server
            const server = new ApolloServer({ typeDefs, resolvers });
            
            server.listen().then(({ url }) => {
              console.log(`GraphQL server running at ${url}`);
            });
            EOF
            
            # Run the server
            node index.js
        ports:
        - containerPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: graphql-gateway
spec:
  selector:
    app: graphql-gateway
  ports:
  - port: 80
    targetPort: 4000
```

Apply this configuration:

```bash
kubectl apply -f graphql-gateway.yaml
```

Expose the GraphQL API through Kong:

```yaml
# kong-graphql.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: graphql-ingress
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /graphql
        pathType: Prefix
        backend:
          service:
            name: graphql-gateway
            port:
              number: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-graphql.yaml
```

Test the GraphQL API:

```bash
# Simple query
curl -X POST http://$KONG_GATEWAY/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ users { id name } products { id name } }"}'
```

## Step 9: Operational Best Practices

### High Availability Configuration

For production environments, ensure high availability:

```yaml
# kong-ha.yaml (example for Kong)
apiVersion: helm.sh/v1
kind: HelmRelease
metadata:
  name: kong
  namespace: kong
spec:
  chart:
    spec:
      chart: kong
      version: 2.8.0
      sourceRef:
        kind: HelmRepository
        name: kong
  values:
    replicaCount: 3
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
    postgresql:
      enabled: true
      replication:
        enabled: true
        readReplicas: 2
    proxy:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 2
          memory: 1Gi
    ingressController:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 1
          memory: 512Mi
```

### Backup and Restore

For Kong, back up the PostgreSQL database:

```bash
# Create a backup job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kong-db-backup
  namespace: kong
spec:
  schedule: "0 2 * * *"  # Every day at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:13
            command:
            - /bin/bash
            - -c
            - |
              PGPASSWORD=$POSTGRES_PASSWORD pg_dump -h kong-postgresql -U postgres kong > /backup/kong-$(date +%Y%m%d-%H%M%S).sql
              echo "Backup completed"
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: kong-postgresql
                  key: postgres-password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: kong-backups-pvc
EOF
```

### Scaling Strategies

Configure HPA for Kong components:

```yaml
# kong-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kong-proxy-hpa
  namespace: kong
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kong-kong-proxy
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Apply this configuration:

```bash
kubectl apply -f kong-hpa.yaml
```

### Configuration Management with GitOps

Implement GitOps workflow for API Gateway configuration using Flux:

```bash
# Install Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=api-gateway-gitops \
  --branch=main \
  --path=./clusters/production \
  --personal
```

Create a Git repository structure for your API gateway configuration:

```
api-gateway-gitops/
├── clusters/
│   └── production/
│       ├── kong/
│       │   ├── kustomization.yaml
│       │   ├── release.yaml
│       │   └── values.yaml
│       └── apis/
│           ├── kustomization.yaml
│           ├── echo-api.yaml
│           ├── composite-api.yaml
│           └── auth-config.yaml
└── apis/
    ├── base/
    │   ├── echo-api.yaml
    │   └── auth-config.yaml
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Maintenance and Troubleshooting

### Debugging API Gateway Issues

Common troubleshooting commands:

```bash
# Check Kong's status
kubectl get pods -n kong
kubectl logs -n kong deployment/kong-kong

# Check Kong's configuration
kubectl exec -it -n kong deployment/kong-kong -- kong config

# Check Istio Gateway status
kubectl get gateways -A
kubectl get virtualservices -A

# Check Envoy proxy configuration
istioctl proxy-config routes $(kubectl get pod -l app=echo -o jsonpath='{.items[0].metadata.name}')
```

### Monitoring Gateway Health

```bash
# Create Prometheus alerts for gateway health
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-gateway-alerts
  namespace: monitoring
spec:
  groups:
  - name: api-gateway.rules
    rules:
    - alert: ApiGatewayHighErrorRate
      expr: sum(rate(kong_http_status{code=~"5.."}[5m])) / sum(rate(kong_http_status[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "API Gateway high error rate"
        description: "API Gateway error rate is above 5% for 5 minutes"
    - alert: ApiGatewayHighLatency
      expr: histogram_quantile(0.95, sum(rate(kong_latency_bucket[5m])) by (le)) > 500
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "API Gateway high latency"
        description: "API Gateway P95 latency is above 500ms for 5 minutes"
    - alert: ApiGatewayDown
      expr: count(up{job="kong"} == 0) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "API Gateway down"
        description: "API Gateway is down"
EOF
```

## Conclusion

In this comprehensive project, we've implemented and explored various API Gateway solutions for Kubernetes:

1. **Installation and setup** of Kong, Ambassador, and Istio Gateway
2. **Basic configuration** for routing and service discovery
3. **Authentication and security** with various methods
4. **Advanced features** like rate limiting, canary deployments, and request transformation
5. **Monitoring and observability** with Prometheus, Grafana, and EFK
6. **Security best practices** including TLS, OAuth2, and CORS
7. **Advanced patterns** like API composition and GraphQL integration
8. **Operational best practices** for high availability and maintenance

By following this guide, you've created a production-ready API Gateway architecture on Kubernetes that can handle complex routing requirements, secure your APIs, and provide detailed monitoring and observability.

Key takeaways:
- Choose the API Gateway that best fits your specific requirements
- Implement proper security from the beginning
- Use monitoring and observability tools to gain insights into API traffic
- Follow best practices for configuration management and high availability
- Consider advanced patterns for sophisticated API requirements

This foundation provides a robust starting point for your API management needs, and you can extend it with additional features and integrations as your requirements evolve.