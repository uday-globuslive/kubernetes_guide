# Microservices Application Deployment

This chapter provides a practical guide to deploying a complete microservices application on Kubernetes. We'll walk through the design, implementation, and deployment of a robust microservices architecture using best practices for scalability, resilience, and maintainability.

## Table of Contents

- [Project Overview](#project-overview)
- [System Architecture](#system-architecture)
- [Prerequisites](#prerequisites)
- [Infrastructure Setup](#infrastructure-setup)
- [Application Implementation](#application-implementation)
- [Deployment Configuration](#deployment-configuration)
- [Service Communication](#service-communication)
- [API Gateway](#api-gateway)
- [Authentication and Authorization](#authentication-and-authorization)
- [Observability](#observability)
- [CI/CD Pipeline](#cicd-pipeline)
- [Scaling and Resilience](#scaling-and-resilience)
- [Performance Testing](#performance-testing)
- [Common Issues and Solutions](#common-issues-and-solutions)
- [Future Improvements](#future-improvements)

## Project Overview

### Scenario

We'll build an e-commerce platform consisting of:

- **User Service**: Customer account management
- **Catalog Service**: Product listings and details
- **Cart Service**: Shopping cart management
- **Order Service**: Order processing
- **Payment Service**: Payment processing
- **Notification Service**: Email and SMS notifications
- **Review Service**: Product reviews
- **Search Service**: Product search functionality
- **Analytics Service**: Business metrics and analytics

### Technical Stack

- **Container Runtime**: Docker
- **Orchestration**: Kubernetes
- **Programming Languages**: Go, Node.js, Python
- **Databases**: PostgreSQL, MongoDB, Redis
- **Message Broker**: Kafka
- **API Gateway**: Kong
- **Service Mesh**: Istio
- **Observability**: Prometheus, Grafana, Jaeger, Fluentd
- **CI/CD**: GitHub Actions, ArgoCD

## System Architecture

Our microservices architecture follows these design principles:

- **Service Independence**: Each service maintains its own data store
- **API-Driven Communication**: Clear API contracts between services
- **Async Communication**: Event-driven patterns for eventual consistency
- **Defense in Depth**: Multiple security layers
- **Observable by Default**: Comprehensive monitoring

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Kubernetes Cluster                            │
│                                                                         │
│  ┌───────────────┐                                                      │
│  │ External Load │                                                      │
│  │ Balancer      │                                                      │
│  └───────┬───────┘                                                      │
│          │                                                              │
│          ▼                                                              │
│  ┌───────────────┐     ┌────────────────┐                               │
│  │ API Gateway   │────▶│ Authentication │                               │
│  │ (Kong)        │     │ Service        │                               │
│  └───────┬───────┘     └────────────────┘                               │
│          │                                                              │
│          │                                                              │
│  ┌───────┴───────┬──────────────┬───────────────┬──────────────┐        │
│  ▼               ▼              ▼               ▼              ▼        │
│┌─────────┐  ┌─────────┐   ┌─────────┐    ┌─────────┐     ┌─────────┐    │
││  User   │  │ Catalog │   │  Cart   │    │  Order  │     │ Payment │    │
││ Service │  │ Service │   │ Service │    │ Service │     │ Service │    │
│└────┬────┘  └────┬────┘   └────┬────┘    └────┬────┘     └────┬────┘    │
│     │            │             │               │               │        │
│     │            │             │               │               │        │
│┌────▼────┐  ┌────▼────┐   ┌────▼────┐    ┌────▼────┐     ┌────▼────┐    │
││  User   │  │ Catalog │   │  Cart   │    │  Order  │     │ Payment │    │
││   DB    │  │   DB    │   │   DB    │    │   DB    │     │ Gateway │    │
│└─────────┘  └─────────┘   └─────────┘    └─────────┘     └─────────┘    │
│                                                                         │
│                                                                         │
│  ┌───────────────┐     ┌────────────────┐    ┌────────────────┐         │
│  │ Notification  │     │ Search Service │    │ Review Service │         │
│  │ Service       │     │ (Elasticsearch)│    │                │         │
│  └───────┬───────┘     └────────┬───────┘    └────────┬───────┘         │
│          │                      │                     │                 │
│          │                      │                     │                 │
│  ┌───────▼───────┐     ┌────────▼───────┐    ┌────────▼───────┐         │
│  │ Message Queue │     │ Search Index   │    │ Review DB      │         │
│  │ (Kafka)       │     │                │    │                │         │
│  └───────────────┘     └────────────────┘    └────────────────┘         │
│                                                                         │
│  ┌───────────────┐     ┌────────────────┐    ┌────────────────┐         │
│  │ Prometheus    │     │ Grafana        │    │ Jaeger         │         │
│  │ (Metrics)     │     │ (Dashboards)   │    │ (Tracing)      │         │
│  └───────────────┘     └────────────────┘    └────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before beginning deployment:

1. **Kubernetes Cluster**: Set up a production-grade cluster
2. **Namespace**: Create a dedicated namespace
3. **RBAC**: Configure appropriate permissions
4. **CI/CD Tools**: Set up CI/CD pipeline tooling
5. **Container Registry**: Configure for storing container images
6. **Domain Names**: Register domains for your services
7. **TLS Certificates**: Provision certificates for secure communication

```bash
# Create namespace
kubectl create namespace ecommerce

# Set context to use this namespace by default
kubectl config set-context --current --namespace=ecommerce

# Create service account
kubectl create serviceaccount ecommerce-admin -n ecommerce

# Create ClusterRoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ecommerce-admin
subjects:
- kind: ServiceAccount
  name: ecommerce-admin
  namespace: ecommerce
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

## Infrastructure Setup

### Helm and Operators

We'll use Helm charts for most deployments:

```bash
# Initialize Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add elastic https://helm.elastic.co
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo add kong https://charts.konghq.com
helm repo update
```

### Database Setup

Deploy the databases needed by our services:

1. **PostgreSQL for User and Order Services**:

```yaml
# postgres-values.yaml
architecture: replication
auth:
  postgresPassword: "your-postgres-pwd"
  replicationPassword: "your-repl-pwd"
primary:
  persistence:
    size: 10Gi
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
readReplicas:
  replicaCount: 2
  persistence:
    size: 10Gi
metrics:
  enabled: true
```

```bash
helm install postgres bitnami/postgresql -f postgres-values.yaml
```

2. **MongoDB for Catalog and Review Services**:

```yaml
# mongodb-values.yaml
architecture: replicaset
auth:
  rootPassword: "your-mongo-pwd"
replicaCount: 3
persistence:
  size: 10Gi
metrics:
  enabled: true
```

```bash
helm install mongodb bitnami/mongodb -f mongodb-values.yaml
```

3. **Redis for Cart Service**:

```yaml
# redis-values.yaml
architecture: replication
auth:
  password: "your-redis-pwd"
master:
  persistence:
    size: 8Gi
replica:
  replicaCount: 2
  persistence:
    size: 8Gi
metrics:
  enabled: true
```

```bash
helm install redis bitnami/redis -f redis-values.yaml
```

### Message Broker Setup

Deploy Kafka for async communications:

```yaml
# kafka-values.yaml
replicaCount: 3
deleteTopicEnable: true
autoCreateTopicsEnable: true
allowPlaintextListener: false
metrics:
  kafka:
    enabled: true
  jmx:
    enabled: true
zookeeper:
  enabled: true
  replicaCount: 3
  persistence:
    size: 8Gi
persistence:
  size: 10Gi
```

```bash
helm install kafka bitnami/kafka -f kafka-values.yaml
```

### Elasticsearch for Search Service

```yaml
# elasticsearch-values.yaml
replicas: 3
minimumMasterNodes: 2
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"
volumeClaimTemplate:
  resources:
    requests:
      storage: 100Gi
```

```bash
helm install elasticsearch elastic/elasticsearch -f elasticsearch-values.yaml
```

### Service Mesh Setup

Install Istio for advanced network features:

```bash
# Set up Istio with sidecar injection
istioctl install --set profile=demo -y

# Enable automatic sidecar injection for our namespace
kubectl label namespace ecommerce istio-injection=enabled
```

## Application Implementation

Let's examine each microservice implementation:

### User Service (Go)

The User service handles user management and authentication:

```go
// main.go
package main

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"
    "os"
    
    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    // Additional fields omitted for brevity
}

var db *sql.DB

func main() {
    // Database setup
    connStr := os.Getenv("DB_CONNECTION_STRING")
    var err error
    db, err = sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // Router setup
    r := mux.NewRouter()
    r.HandleFunc("/users", getUsers).Methods("GET")
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    r.HandleFunc("/users", createUser).Methods("POST")
    // Additional routes omitted
    
    // Metrics and health check
    r.HandleFunc("/health", healthCheck).Methods("GET")
    r.HandleFunc("/metrics", metricsHandler).Methods("GET")
    
    // Start the server
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    log.Printf("User service listening on port %s", port)
    log.Fatal(http.ListenAndServe(":"+port, r))
}

// Handler implementations omitted for brevity
```

**Dockerfile**:

```dockerfile
FROM golang:1.19-alpine AS build

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o userservice

FROM alpine:3.16
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=build /app/userservice .
EXPOSE 8080
CMD ["./userservice"]
```

### Catalog Service (Node.js)

The Catalog service manages product information:

```javascript
// index.js
const express = require('express');
const mongoose = require('mongoose');
const { MongoClient } = require('mongodb');
const app = express();
const port = process.env.PORT || 8080;

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

// Product schema
const productSchema = new mongoose.Schema({
  name: String,
  description: String,
  price: Number,
  category: String,
  imageUrl: String,
  inventory: Number
});

const Product = mongoose.model('Product', productSchema);

// Middleware
app.use(express.json());

// Routes
app.get('/products', async (req, res) => {
  try {
    const products = await Product.find();
    res.json(products);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.get('/products/:id', async (req, res) => {
  try {
    const product = await Product.findById(req.params.id);
    if (!product) return res.status(404).json({ error: 'Product not found' });
    res.json(product);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// More routes omitted for brevity

// Health check
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Start the server
app.listen(port, () => {
  console.log(`Catalog service listening on port ${port}`);
});

// Publish product updates to Kafka
function publishProductUpdate(product) {
  // Kafka producer code omitted for brevity
}
```

**Dockerfile**:

```dockerfile
FROM node:16-alpine

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 8080
CMD ["node", "index.js"]
```

### Cart Service (Python)

The Cart service manages shopping carts with Redis:

```python
# app.py
from flask import Flask, request, jsonify
import redis
import json
import os
import prometheus_client
from prometheus_client import Counter, Histogram
import time

app = Flask(__name__)
# Redis connection
redis_host = os.environ.get('REDIS_HOST', 'localhost')
redis_port = os.environ.get('REDIS_PORT', 6379)
redis_password = os.environ.get('REDIS_PASSWORD', '')
redis_client = redis.Redis(
    host=redis_host,
    port=redis_port,
    password=redis_password,
    decode_responses=True
)

# Prometheus metrics
REQUEST_COUNT = Counter('cart_request_count', 'App Request Count',
                         ['method', 'endpoint', 'http_status'])
REQUEST_LATENCY = Histogram('cart_request_latency_seconds', 'Request latency',
                         ['method', 'endpoint'])

@app.route('/carts/<user_id>', methods=['GET'])
def get_cart(user_id):
    start_time = time.time()
    cart_key = f"cart:{user_id}"
    
    try:
        cart_data = redis_client.get(cart_key)
        if not cart_data:
            cart = {'user_id': user_id, 'items': []}
        else:
            cart = json.loads(cart_data)
        
        status_code = 200
        response = jsonify(cart)
    except Exception as e:
        status_code = 500
        response = jsonify({'error': str(e)})
    
    # Record metrics
    REQUEST_COUNT.labels(request.method, request.path, status_code).inc()
    REQUEST_LATENCY.labels(request.method, request.path).observe(time.time() - start_time)
    
    return response, status_code

@app.route('/carts/<user_id>/items', methods=['POST'])
def add_to_cart(user_id):
    # Implementation omitted for brevity
    pass

# Additional routes omitted for brevity

@app.route('/metrics')
def metrics():
    return prometheus_client.generate_latest()

@app.route('/health')
def health():
    try:
        redis_client.ping()
        return jsonify({'status': 'ok'})
    except:
        return jsonify({'status': 'error', 'message': 'Redis connection failed'}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

**Dockerfile**:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080
CMD ["python", "app.py"]
```

## Deployment Configuration

Let's look at the Kubernetes manifests for each service:

### User Service Deployment

```yaml
# user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: example.com/ecommerce/user-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: connection-string
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
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

### Database Secrets

```yaml
# postgres-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
data:
  connection-string: cG9zdGdyZXM6Ly91c2VyOnBhc3N3b3JkQHBvc3RncmVzOjU0MzIvdXNlcnM=  # Base64 encoded
---
# mongodb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials
type: Opaque
data:
  uri: bW9uZ29kYjovL3VzZXI6cGFzc3dvcmRAbW9uZ29kYjoyNzAxNy9jYXRhbG9n # Base64 encoded
---
# redis-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-credentials
type: Opaque
data:
  password: eW91ci1yZWRpcy1wd2Q= # Base64 encoded
```

### ConfigMaps for Configuration

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-config
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  METRICS_ENABLED: "true"
  TRACING_ENABLED: "true"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
data:
  JWT_EXPIRATION: "3600"
  PASSWORD_HASH_ITERATIONS: "10000"
```

## Service Communication

### Service Discovery

Services communicate using Kubernetes service discovery:

```yaml
# Example of service-to-service communication
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### Istio for Advanced Networking

Add Istio VirtualService and DestinationRule for traffic management:

```yaml
# virtual-services.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 90
    - destination:
        host: user-service
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1024
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Secure Communication with mTLS

Enable mutual TLS between services:

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT
```

## API Gateway

Kong API Gateway provides a unified entry point:

```yaml
# kong-gateway.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-gateway
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/strip-path: "true"
    konghq.com/plugins: rate-limiting,cors,bot-detection
spec:
  rules:
  - host: api.ecommerce-example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80
      - path: /carts
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

Add Kong plugins for additional functionality:

```yaml
# kong-plugins.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
config:
  minute: 60
  limit_by: ip
  policy: local
plugin: rate-limiting
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: cors
config:
  origins:
  - https://ecommerce-example.com
  methods:
  - GET
  - POST
  - PUT
  - DELETE
  - OPTIONS
  headers:
  - Authorization
  - Content-Type
  - Accept
  credentials: true
  max_age: 3600
plugin: cors
```

## Authentication and Authorization

### JWT Authentication Service

Implement secure authentication:

```yaml
# jwt-auth.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth
config:
  secret_is_base64: false
  key_claim_name: kid
  claims_to_verify:
  - exp
plugin: jwt
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: web-app
  annotations:
    kubernetes.io/ingress.class: kong
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: mobile-app
  annotations:
    kubernetes.io/ingress.class: kong
---
apiVersion: configuration.konghq.com/v1
kind: JWTAuth
metadata:
  name: web-app-jwt
consumer_ref: web-app
algorithm: RS256
key: "web-app-key"
rsa_public_key: |
  -----BEGIN PUBLIC KEY-----
  MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqD0AoLOK3JXUt1bwQGLh
  ... [truncated for brevity]
  AIDAQAB
  -----END PUBLIC KEY-----
```

### Role-Based Access Control (RBAC)

Implement RBAC for fine-grained permissions:

```yaml
# rbac-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rbac-config
data:
  admin-roles: |-
    {
      "admin": ["users:read", "users:write", "products:read", "products:write", "orders:read", "orders:write"],
      "customer-service": ["users:read", "orders:read", "products:read"],
      "inventory-manager": ["products:read", "products:write"]
    }
```

## Observability

### Prometheus and Grafana

Set up monitoring:

```yaml
# prometheus-values.yaml
serviceMonitorSelector:
  matchLabels:
    release: prometheus
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack'
      routes:
      - match:
          alertname: Watchdog
        receiver: 'null'
    receivers:
    - name: 'null'
    - name: 'slack'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
```

```bash
helm install prometheus prometheus-community/prometheus -f prometheus-values.yaml
```

Create a ServiceMonitor for your services:

```yaml
# service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ecommerce-services
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: ecommerce
  endpoints:
  - port: http-metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - ecommerce
```

### Distributed Tracing with Jaeger

Deploy Jaeger for tracing:

```yaml
# jaeger.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
        username: elastic
        password: changeme
  ingress:
    enabled: true
    hosts:
    - jaeger.ecommerce-example.com
```

### Centralized Logging with EFK Stack

```yaml
# fluentd-values.yaml
configMaps:
  useDefaults:
    outputs.conf: false
    general.conf: false

extraConfigMaps:
  outputs.conf: |
    <match **>
      @type elasticsearch
      host elasticsearch
      port 9200
      logstash_format true
      logstash_prefix fluentd
      <buffer>
        flush_interval 5s
      </buffer>
    </match>
```

```bash
helm install fluentd bitnami/fluentd -f fluentd-values.yaml
```

## CI/CD Pipeline

### GitHub Actions Workflow

Create a CI/CD workflow:

```yaml
# .github/workflows/ci-cd.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.17
      
    - name: Build
      run: |
        cd user-service
        go build -v ./...
      
    - name: Test
      run: |
        cd user-service
        go test -v ./...
        
    - name: Run linters
      uses: golangci/golangci-lint-action@v2
      with:
        working-directory: user-service
    
    - name: Build and push Docker image
      if: github.event_name == 'push'
      uses: docker/build-push-action@v2
      with:
        context: ./user-service
        push: true
        tags: |
          example.com/ecommerce/user-service:latest
          example.com/ecommerce/user-service:${{ github.sha }}

  deploy-dev:
    needs: build-and-test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Kustomize
      uses: imranismail/setup-kustomize@v1
      
    - name: Update Kubernetes resources
      run: |
        cd deploy/overlays/dev
        kustomize edit set image example.com/ecommerce/user-service:${{ github.sha }}
        
    - name: Commit changes
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add deploy/overlays/dev
        git commit -m "Update user-service image to ${{ github.sha }}"
        git push
```

### ArgoCD Application Deployment

Use GitOps with ArgoCD:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/ecommerce-k8s.git
    targetRevision: HEAD
    path: deploy/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Kustomize for Environment Management

Organize deployments with Kustomize:

```
deploy/
├── base/
│   ├── kustomization.yaml
│   ├── user-service.yaml
│   ├── catalog-service.yaml
│   └── ...
├── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    └── prod/
        ├── kustomization.yaml
        └── patches/
```

Base kustomization.yaml:

```yaml
# deploy/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- user-service.yaml
- catalog-service.yaml
- cart-service.yaml
- order-service.yaml
- payment-service.yaml
- notification-service.yaml
- review-service.yaml
- search-service.yaml
- kong-gateway.yaml
```

Dev environment customization:

```yaml
# deploy/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

namespace: ecommerce-dev

patches:
- path: patches/replicas-patch.yaml
- path: patches/resource-limits-patch.yaml

configMapGenerator:
- name: global-config
  behavior: merge
  literals:
  - ENVIRONMENT=development
  - LOG_LEVEL=debug
```

```yaml
# deploy/overlays/dev/patches/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
spec:
  replicas: 2
```

## Scaling and Resilience

### Pod Disruption Budgets

Ensure high availability during cluster operations:

```yaml
# pdbs.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: user-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: catalog-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: catalog-service
```

### Circuit Breaking with Istio

Implement circuit breaking for resilience:

```yaml
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
```

### Retry and Timeout Policies

Add retries and timeouts:

```yaml
# retry-policy.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
    timeout: 5s
```

## Performance Testing

### Load Testing with K6

Create a k6 load test:

```javascript
// load-test.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '5m', target: 100 }, // Ramp up to 100 users
    { duration: '10m', target: 100 }, // Stay at 100 users
    { duration: '5m', target: 0 }, // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must complete within 500ms
    'http_req_duration{name:get-products}': ['p(95)<400'],
    'http_req_duration{name:get-cart}': ['p(95)<300'],
  },
};

const BASE_URL = 'https://api.ecommerce-example.com';

export default function () {
  // Get product catalog
  let productsResponse = http.get(`${BASE_URL}/products`, {
    tags: { name: 'get-products' },
  });
  check(productsResponse, {
    'products status 200': (r) => r.status === 200,
    'products response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  // Get user cart
  let cartResponse = http.get(`${BASE_URL}/carts/user123`, {
    tags: { name: 'get-cart' },
    headers: { 'Authorization': `Bearer ${__ENV.USER_TOKEN}` },
  });
  check(cartResponse, {
    'cart status 200': (r) => r.status === 200,
    'cart response time < 300ms': (r) => r.timings.duration < 300,
  });
  
  sleep(1);
}
```

### Chaos Testing with Chaos Mesh

Implement chaos testing:

```yaml
# network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-service-delay
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - ecommerce
    labelSelectors:
      app: payment-service
  delay:
    latency: '200ms'
    correlation: '25'
    jitter: '50ms'
  duration: '5m'
  scheduler:
    cron: '@every 30m'
```

## Common Issues and Solutions

### Database Connection Pooling

Add connection pooling for databases:

```yaml
# postgresql-connection-pooling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
      - name: pgbouncer
        image: edoburu/pgbouncer:1.17.0
        ports:
        - containerPort: 5432
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: connection-string
        - name: POOL_MODE
          value: transaction
        - name: MAX_CLIENT_CONN
          value: "1000"
        - name: DEFAULT_POOL_SIZE
          value: "50"
        livenessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer
spec:
  selector:
    app: pgbouncer
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

### Memory Leaks

Monitor for memory leaks:

```yaml
# prometheus-rule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: memory-leak-alert
spec:
  groups:
  - name: memory-alerts
    rules:
    - alert: PossibleMemoryLeak
      expr: sum(rate(process_resident_memory_bytes[10m])) by (pod) > 0.01
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "Possible memory leak in {{ $labels.pod }}"
        description: "Pod {{ $labels.pod }} memory usage has been steadily increasing for 30 minutes"
```

### Network Issues

Diagnose network problems with network policies:

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-traffic
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: ecommerce
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: ecommerce
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-access
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          database-client: postgres
    ports:
    - protocol: TCP
      port: 5432
```

## Future Improvements

### Feature Flagging

Implement feature flagging with ConfigMaps:

```yaml
# feature-flags.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  NEW_CHECKOUT_FLOW: "true"
  RECOMMENDATIONS_ENGINE: "false"
  SOCIAL_LOGIN: "true"
  INVENTORY_TRACKING: "true"
```

### A/B Testing with Istio

Use Istio for A/B testing:

```yaml
# ab-testing.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: checkout-service
spec:
  hosts:
  - checkout-service
  http:
  - match:
    - headers:
        x-test-group:
          exact: "A"
    route:
    - destination:
        host: checkout-service
        subset: original
  - match:
    - headers:
        x-test-group:
          exact: "B"
    route:
    - destination:
        host: checkout-service
        subset: variation
  - route:
    - destination:
        host: checkout-service
        subset: original
      weight: 90
    - destination:
        host: checkout-service
        subset: variation
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: checkout-service
spec:
  host: checkout-service
  subsets:
  - name: original
    labels:
      version: v1
  - name: variation
    labels:
      version: v2
```

### Blue-Green Deployments

Implement blue-green deployments:

```yaml
# blue-green.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: catalog-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: catalog-service
  template:
    metadata:
      labels:
        app: catalog-service
    spec:
      containers:
      - name: catalog-service
        image: example.com/ecommerce/catalog-service:1.0.0
        ports:
        - containerPort: 8080
  strategy:
    blueGreen:
      activeService: catalog-service-active
      previewService: catalog-service-preview
      autoPromotionEnabled: false
```

### Global Cache Layer

Add a global caching layer:

```yaml
# redis-cache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis-cache
  template:
    metadata:
      labels:
        app: redis-cache
    spec:
      containers:
      - name: redis
        image: redis:6.2-alpine
        ports:
        - containerPort: 6379
        args: ["--requirepass", "$(REDIS_PASSWORD)"]
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-cache-credentials
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cache
spec:
  selector:
    app: redis-cache
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

## Conclusion

In this project, we've built a complete microservices-based e-commerce platform on Kubernetes. Key takeaways include:

1. **Architecture Matters**: Well-designed microservices architecture enables scalability and maintainability
2. **Observability First**: Comprehensive monitoring is critical for production systems
3. **Security in Layers**: From network policies to authentication, security must be built in
4. **Automation is Essential**: CI/CD and GitOps streamline deployment and reduce errors
5. **Resilience by Design**: Circuit breakers, retries, and chaos testing build robust systems

By following these patterns and practices, you can deploy scalable, resilient microservices applications on Kubernetes in production environments.

## Further Reading

- [Kubernetes Patterns](https://www.oreilly.com/library/view/kubernetes-patterns/9781492050278/)
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
- [Istio in Action](https://www.manning.com/books/istio-in-action)
- [SRE Workbook](https://sre.google/workbook/table-of-contents/)