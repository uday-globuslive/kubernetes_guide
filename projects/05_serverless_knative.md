# Serverless on Kubernetes with Knative

Building a serverless platform on Kubernetes using Knative enables you to deploy scalable, event-driven applications that automatically scale down to zero when not in use. This project guide walks through the complete implementation of a serverless architecture with Knative, covering installation, configuration, deployment patterns, event sources, and real-world use cases.

## Introduction to Serverless on Kubernetes

### What is Knative?

Knative is an open-source platform that extends Kubernetes to provide a serverless experience. It consists of two main components:

1. **Knative Serving**: For deploying and automatically scaling containerized applications
2. **Knative Eventing**: For building event-driven architectures with loosely coupled services

Knative provides:
- Automatic scaling based on traffic, including scale-to-zero
- Request-based autoscaling
- Revision tracking for deployments
- Event-driven architecture capabilities
- Integration with various event sources (Kafka, CloudEvents, etc.)

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             Kubernetes Cluster                           │
│                                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌──────────────────────────┐  │
│  │   Ingress     │   │    Knative    │   │  Knative Serving         │  │
│  │  Controller   ├───┤    Gateway    ├───┤                          │  │
│  └───────────────┘   └───────────────┘   │  ┌──────────────────┐    │  │
│                                          │  │ Revision 1 (v1)   │    │  │
│  ┌───────────────────────────────────┐   │  │                  │    │  │
│  │  Knative Eventing                 │   │  │ ┌──────┐ ┌──────┐│    │  │
│  │                                   │   │  │ │Pod   │ │Pod   ││    │  │
│  │  ┌────────────┐ ┌────────────┐   │   │  │ └──────┘ └──────┘│    │  │
│  │  │   Event    │ │   Event    │   │   │  └──────────────────┘    │  │
│  │  │  Source    │ │   Broker   │   │   │                          │  │
│  │  └────────────┘ └────────────┘   │   │  ┌──────────────────┐    │  │
│  │         │              │         │   │  │ Revision 2 (v2)   │    │  │
│  │         ▼              ▼         │   │  │                  │    │  │
│  │  ┌────────────┐ ┌────────────┐   │   │  │ ┌──────┐ ┌──────┐│    │  │
│  │  │  Trigger   │ │    Sink    │   │   │  │ │Pod   │ │Pod   ││    │  │
│  │  └────────────┘ └────────────┘   │   │  │ └──────┘ └──────┘│    │  │
│  │         │              │         │   │  └──────────────────┘    │  │
│  └─────────┼──────────────┼─────────┘   └──────────────────────────┘  │
│            │              │                                            │
│            ▼              ▼                                            │
│  ┌────────────────────────────────────┐  ┌────────────────────────┐   │
│  │       Serverless Services          │  │   Autoscaler           │   │
│  │                                    │  │                        │   │
│  │  ┌──────────┐       ┌──────────┐  │  │  ┌──────────────────┐  │   │
│  │  │ Service A│       │ Service B│  │  │  │                  │  │   │
│  │  │          │◄─────►│          │  │  │  │   Metrics &      │  │   │
│  │  └──────────┘       └──────────┘  │  │  │  Scaling Logic   │  │   │
│  │                                    │  │  │                  │  │   │
│  └────────────────────────────────────┘  │  └──────────────────┘  │   │
│                                          └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Project Prerequisites

- Kubernetes cluster 1.18+ (EKS, GKE, AKS, or local cluster like kind/minikube)
- kubectl configured to access your cluster
- Helm 3.0+
- About 4 vCPUs and 8GB of memory available in your cluster

## Implementation Steps

### Step 1: Setting Up the Kubernetes Cluster

Start with a properly sized Kubernetes cluster. For this guide, we'll use a local kind cluster, but you can adapt these steps for cloud providers.

```bash
# Create a kind cluster with extra resources
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
  extraPortMappings:
  - containerPort: 31080
    hostPort: 80
    protocol: TCP
  - containerPort: 31443
    hostPort: 443
    protocol: TCP
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
EOF

kind create cluster --config kind-config.yaml --name knative-demo
```

For cloud providers, use appropriate sizing:

```bash
# For EKS
eksctl create cluster \
  --name=knative-cluster \
  --region=us-west-2 \
  --node-type=t3.large \
  --nodes=3 \
  --nodes-min=3 \
  --nodes-max=5

# For GKE
gcloud container clusters create knative-cluster \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=5

# For AKS
az aks create \
  --resource-group myResourceGroup \
  --name knative-cluster \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5
```

### Step 2: Installing Knative Serving

Install Knative Serving components:

```bash
# Install Knative CRDs
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.7.0/serving-crds.yaml

# Install Knative core
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.7.0/serving-core.yaml

# Install Knative Serving with Kourier networking layer
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.7.0/kourier.yaml

# Configure Knative Serving to use Kourier
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'

# Set up DNS (for local development)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.7.0/serving-default-domain.yaml
```

Verify the installation:

```bash
# Check if all pods are running
kubectl get pods -n knative-serving

# Check Knative Services
kubectl get ksvc -A
```

### Step 3: Installing Knative Eventing

Install Knative Eventing components:

```bash
# Install Knative Eventing CRDs
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.7.0/eventing-crds.yaml

# Install Knative Eventing core
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.7.0/eventing-core.yaml

# Install default in-memory channel (for development)
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.7.0/in-memory-channel.yaml

# Install MT Channel Based Broker
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.7.0/mt-channel-broker.yaml
```

Verify the installation:

```bash
# Check if all pods are running
kubectl get pods -n knative-eventing

# Check Knative Eventing resources
kubectl get broker,trigger,channel -A
```

### Step 4: Configure Knative for Production

Customize Knative for production workloads:

#### Autoscaling Configuration

```bash
# Configure global autoscaling (scale from 1-10 pods, scale to zero after 30s)
kubectl patch configmap/config-autoscaler \
  --namespace knative-serving \
  --type merge \
  --patch '{
    "data": {
      "min-scale": "1",
      "max-scale": "10",
      "scale-to-zero-grace-period": "30s",
      "stable-window": "60s",
      "enable-scale-to-zero": "true",
      "target": "100",
      "container-concurrency-target-default": "100"
    }
  }'

# Configure request-based autoscaling
kubectl patch configmap/config-autoscaler \
  --namespace knative-serving \
  --type merge \
  --patch '{
    "data": {
      "container-concurrency-target-percentage": "70",
      "panic-window-percentage": "10.0",
      "panic-threshold-percentage": "200.0"
    }
  }'
```

#### Networking Configuration

```bash
# Configure reasonable defaults for timeouts
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{
    "data": {
      "httpprotocol": "Enabled",
      "autocreate-cluster-domain-claims": "true",
      "domain-template": "{{.Name}}.{{.Namespace}}.{{.Domain}}",
      "read-timeout": "10s",
      "write-timeout": "10s"
    }
  }'
```

#### Resource Optimization for Scale to Zero

```bash
# Configure more aggressive scale-to-zero
kubectl patch configmap/config-autoscaler \
  --namespace knative-serving \
  --type merge \
  --patch '{
    "data": {
      "scale-to-zero-pod-retention-period": "30s"
    }
  }'
```

### Step 5: Deploy Your First Serverless Application

Create a simple serverless Hello World service:

```yaml
# hello-world.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-world
  namespace: default
spec:
  template:
    metadata:
      name: hello-world-v1
      annotations:
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "5"
        autoscaling.knative.dev/target: "50"
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Knative Serverless"
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 64Mi
```

Deploy and test the service:

```bash
# Apply the YAML
kubectl apply -f hello-world.yaml

# Watch the deployment (it should scale to zero if idle)
kubectl get ksvc hello-world --watch

# Get the URL to access the service
SERVICE_URL=$(kubectl get ksvc hello-world -o jsonpath='{.status.url}')
echo $SERVICE_URL

# Test the service (this will cause it to scale from 0 to 1)
curl $SERVICE_URL

# Watch the pods scale up and then back down to zero
kubectl get pods -l serving.knative.dev/service=hello-world --watch
```

### Step 6: Implementing Canary Deployments

Knative makes it easy to implement canary or blue-green deployments:

```yaml
# hello-canary.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-canary
  namespace: default
spec:
  template:
    metadata:
      name: hello-canary-v2
      annotations:
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "5"
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Knative Serverless v2"
  traffic:
  - revisionName: hello-canary-v1
    percent: 80
  - revisionName: hello-canary-v2
    percent: 20
```

To deploy this canary update:

```bash
# First, deploy v1
kubectl apply -f hello-world.yaml

# Modify the service name to hello-canary and apply
kubectl apply -f hello-canary.yaml

# Update to create the canary v2 and split traffic
kubectl apply -f hello-canary.yaml

# Test distribution of requests
for i in {1..10}; do curl $(kubectl get ksvc hello-canary -o jsonpath='{.status.url}'); echo; done
```

### Step 7: Setting Up Knative Eventing

Create a Broker to handle events:

```yaml
# default-broker.yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: default
spec:
  config:
    apiVersion: v1
    kind: ConfigMap
    name: config-br-default-channel
    namespace: knative-eventing
```

Deploy a simple event consumer:

```yaml
# event-display.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-display
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-display
  template:
    metadata:
      labels:
        app: event-display
    spec:
      containers:
      - name: event-display
        image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: event-display
spec:
  selector:
    app: event-display
  ports:
  - port: 80
    targetPort: 8080
```

Create a Trigger to connect events to the service:

```yaml
# trigger.yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: event-display-trigger
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.samples.helloworld
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: event-display
```

Apply these configurations:

```bash
kubectl apply -f default-broker.yaml
kubectl apply -f event-display.yaml
kubectl apply -f trigger.yaml
```

### Step 8: Create Event Sources

Let's create a simple cron job source to generate events:

```yaml
# cronjob-source.yaml
apiVersion: sources.knative.dev/v1
kind: CronJobSource
metadata:
  name: heartbeat-cron
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Hello from Knative!"}'
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
```

Apply the CronJobSource:

```bash
kubectl apply -f cronjob-source.yaml
```

Verify events are flowing:

```bash
# Get pods
kubectl get pods

# Watch logs of the event display deployment
kubectl logs -l app=event-display -c event-display --follow
```

### Step 9: Implement a Complete Serverless Workflow

Let's build a more sophisticated event-driven workflow:

1. An API service receives HTTP requests
2. It emits events to a broker
3. Multiple services react to these events
4. Results are aggregated and stored

#### 1. Create an Order Service

```yaml
# order-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Order Service"
```

#### 2. Create a Payment Processor

```yaml
# payment-processor.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: payment-processor
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Payment Processor"
```

#### 3. Create an Inventory Service

```yaml
# inventory-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inventory-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Inventory Service"
```

#### 4. Create Triggers for Each Service

```yaml
# triggers.yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: payment-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: payment-processor
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: inventory-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: inventory-service
```

Apply all of these resources:

```bash
kubectl apply -f order-service.yaml
kubectl apply -f payment-processor.yaml
kubectl apply -f inventory-service.yaml
kubectl apply -f triggers.yaml
```

### Step 10: Monitoring and Observability

Set up monitoring for your serverless workloads:

#### Installing Prometheus and Grafana

```bash
# Add Prometheus helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.persistentVolume.enabled=false

# Add Grafana helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=false \
  --set adminPassword=admin
```

#### Configure Knative Metrics Collection

```bash
# Install Knative monitoring
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.7.0/monitoring-metrics-prometheus.yaml
```

#### Create Grafana Dashboards

Import Knative Dashboards into Grafana:

1. Get the Grafana admin password:
   ```bash
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
   ```

2. Port-forward Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/grafana 3000:80
   ```

3. Open Grafana at http://localhost:3000 and import dashboards for Knative Serving and Eventing (available on Grafana.com).

### Step 11: Scaling and Performance Testing

Test how your serverless application scales under load:

```bash
# Install a load testing tool
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: hey
        image: williamyeh/hey
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting load test..."
          hey -z 2m -c 50 http://hello-world.default.svc.cluster.local
          echo "Load test completed"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF
```

Monitor how your services scale during the load test:

```bash
# Watch the autoscaling in action
kubectl get pods -l serving.knative.dev/service=hello-world --watch
```

### Step 12: Production-Ready Configuration

Enhance your deployment with production settings:

#### Configure Revision Timeouts

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: production-service
spec:
  template:
    metadata:
      annotations:
        # Timeout for requests
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "10"
        # Target 50 concurrent requests per pod
        autoscaling.knative.dev/target: "50"
    spec:
      # Set request timeouts
      timeoutSeconds: 300
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "Production Service"
        # Set up liveness and readiness probes
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
```

#### Configure Horizontal Pod Autoscaler (HPA) Metrics

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hpa-service
spec:
  template:
    metadata:
      annotations:
        # Use CPU-based autoscaling
        autoscaling.knative.dev/class: "hpa.autoscaling.knative.dev"
        autoscaling.knative.dev/metric: "cpu"
        autoscaling.knative.dev/target: "80"
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "10"
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 500m
```

#### Set Up Private Container Registry Secret

```bash
# Create a Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

Modify your Knative Service to use the private registry:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: private-image-service
spec:
  template:
    spec:
      containers:
      - image: your-private-registry.com/app:latest
      imagePullSecrets:
      - name: regcred
```

## Advanced Scenarios and Use Cases

### Use Case 1: Event-Driven Microservices Architecture

Let's implement a simplified e-commerce system:

![Event-Driven Architecture](https://i.imgur.com/jXrVdpP.png)

```yaml
# e-commerce-services.yaml
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: order-api
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Order API"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Payment Service"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: inventory-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Inventory Service"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: shipping-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Shipping Service"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: notification-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Notification Service"
```

Set up the event triggers:

```yaml
# e-commerce-triggers.yaml
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: payment-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: payment-service
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: inventory-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: inventory-service
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: shipping-trigger
spec:
  broker: default
  filter:
    attributes:
      type: payment.completed
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: shipping-service
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: notification-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.status.changed
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: notification-service
```

### Use Case 2: Scheduled Data Processing

Implement a recurring data processing job:

```yaml
# data-processor.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: data-processor
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "5"
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Data Processor"
---
apiVersion: sources.knative.dev/v1
kind: CronJobSource
metadata:
  name: data-processor-cron
spec:
  schedule: "0 * * * *"  # Run hourly
  data: '{"operation": "process-data"}'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: data-processor
```

### Use Case 3: Event-Driven CI/CD Pipeline

Implement a serverless CI/CD system:

```yaml
# ci-cd-pipeline.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: git-webhook-receiver
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Git Webhook Receiver"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: build-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Build Service"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: test-runner
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Test Runner"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: deployer
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Deployer"
---
# Triggers
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: build-trigger
spec:
  broker: default
  filter:
    attributes:
      type: git.push
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: build-service
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: test-trigger
spec:
  broker: default
  filter:
    attributes:
      type: build.completed
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: test-runner
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: deploy-trigger
spec:
  broker: default
  filter:
    attributes:
      type: test.passed
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: deployer
```

## Performance Tuning and Best Practices

### 1. Cold Start Optimization

Reduce cold start times with these strategies:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: optimized-service
spec:
  template:
    metadata:
      annotations:
        # Keep 1 instance always running to avoid cold starts for critical services
        autoscaling.knative.dev/minScale: "1"
        
        # Use a smaller, optimized container image
        # Consider distroless or Alpine-based images
    spec:
      containers:
      - image: gcr.io/distroless/nodejs:14
        # Set resource limits appropriately
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 1000m
            memory: 256Mi
        # Set environment variables before startup
        env:
        - name: NODE_OPTIONS
          value: "--max-old-space-size=128"
```

### 2. Resource Configuration

Properly size your containers:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: resource-optimized
spec:
  template:
    spec:
      containers:
      - image: my-app-image
        resources:
          requests:
            # Set reasonable requests based on actual needs
            cpu: 100m     # 0.1 CPU cores
            memory: 128Mi
          limits:
            # Set limits to avoid noisy neighbors
            cpu: 500m     # 0.5 CPU cores
            memory: 256Mi
        # Set readiness probe for faster scaling
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 1
          periodSeconds: 2
          timeoutSeconds: 1
```

### 3. Autoscaling Configuration

Fine-tune autoscaling for your workloads:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: autoscale-optimized
spec:
  template:
    metadata:
      annotations:
        # Scale based on CPU rather than concurrency for CPU-intensive workloads
        autoscaling.knative.dev/class: "hpa.autoscaling.knative.dev"
        autoscaling.knative.dev/metric: "cpu"
        autoscaling.knative.dev/target: "70"
        
        # For concurrency-based scaling (default)
        # autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        # autoscaling.knative.dev/metric: "concurrency"
        # autoscaling.knative.dev/target: "50"
        
        # Set scale down delay to avoid rapid scale down then back up
        autoscaling.knative.dev/scaleDownDelay: "15m"
        
        # Set window for autoscaler to smooth out traffic spikes
        autoscaling.knative.dev/window: "60s"
    spec:
      containers:
      - image: my-app-image
```

### 4. Event Delivery Optimization

For reliable event delivery:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: production-broker
  namespace: default
  annotations:
    # Use Kafka for production
    eventing.knative.dev/broker.class: Kafka
spec:
  # Kafka broker config
  config:
    apiVersion: v1
    kind: ConfigMap
    name: kafka-broker-config
    namespace: knative-eventing
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: reliable-trigger
  annotations:
    # Set delivery timeout 
    eventing.knative.dev/delivery.timeout: "30m"
spec:
  broker: production-broker
  filter:
    attributes:
      type: important.event
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-processor
  delivery:
    retry: 5
    backoffPolicy: exponential
    backoffDelay: "PT0.5S"
```

## Monitoring and Troubleshooting

### Common Issues and Solutions

#### 1. Cold Start Latency

**Issue**: Services take too long to start when scaling from zero.

**Solutions**:
- Set `minScale` to 1 for critical services
- Use smaller, optimized container images
- Pre-warm containers with readiness probes
- Optimize application startup time

#### 2. Service Not Scaling

**Issue**: Service is not scaling properly under load.

**Solutions**:
- Check that autoscaler is properly configured
- Verify concurrency settings match your application's capabilities
- Look for errors in autoscaler logs
- Ensure your application properly handles concurrent requests

```bash
# Check autoscaler logs
kubectl logs -n knative-serving deploy/autoscaler
```

#### 3. Event Delivery Issues

**Issue**: Events are not being delivered to subscribers.

**Solutions**:
- Check broker and trigger configurations
- Verify event filtering matches what sources are sending
- Look at broker logs for delivery failures
- Ensure subscribers are available and can process events

```bash
# Check broker logs
kubectl logs -n knative-eventing deploy/mt-broker-controller

# Check ingress logs
kubectl logs -n knative-eventing deploy/mt-broker-ingress

# Check filter logs
kubectl logs -n knative-eventing deploy/mt-broker-filter
```

## Security Considerations

Enhance the security of your Knative deployment:

### Network Policies

Restrict communication between services:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-service-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: secure-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: authorized-client
    ports:
    - protocol: TCP
      port: 8080
```

### RBAC for Knative Resources

Restrict who can create and manage serverless workloads:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: knative-deployer
rules:
- apiGroups: ["serving.knative.dev"]
  resources: ["services", "configurations", "routes", "revisions"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: knative-deployer-binding
  namespace: default
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: knative-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Secure Secrets Access

Securely provide credentials to your services:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: dXNlcm5hbWU=  # base64 encoded "username"
  password: cGFzc3dvcmQ=  # base64 encoded "password"
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: secure-service
spec:
  template:
    spec:
      containers:
      - image: my-secure-app
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

## Conclusion

In this project guide, we've implemented a complete serverless platform on Kubernetes using Knative. We've covered:

1. **Setting up Knative Serving and Eventing** to provide serverless capabilities
2. **Implementing auto-scaling** including scale-to-zero functionality
3. **Creating event-driven architectures** with various event sources and sinks
4. **Monitoring and observability** for serverless workloads
5. **Performance optimization** for real-world production workloads
6. **Security considerations** for serverless applications

The serverless approach with Knative provides:
- Efficient resource utilization with scale-to-zero
- Event-driven architecture enabling loose coupling
- Lower operational overhead for managing services
- Fast iteration and deployment cycles
- Cost optimization through precise scaling

By following this guide, you've created a production-ready serverless platform that can handle various workloads while maintaining high efficiency and scalability.