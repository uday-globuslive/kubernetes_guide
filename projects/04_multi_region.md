# Multi-Region Kubernetes with Global DNS

Building a globally distributed Kubernetes infrastructure with multi-region deployment capabilities and global DNS management is critical for applications requiring high availability, disaster recovery, and low-latency global access. This project guide walks through the complete implementation of a multi-region Kubernetes architecture with global DNS routing.

## Introduction

### Benefits of Multi-Region Kubernetes

- **High Availability**: Survive regional outages and maintain uptime
- **Geographic Redundancy**: Protect against natural disasters or region-wide failures
- **Low Latency**: Serve users from the closest region
- **Compliance**: Meet data residency requirements
- **Scalability**: Distribute load across multiple regions
- **Disaster Recovery**: Maintain operations even if an entire region fails

### Architecture Overview

Our architecture will consist of:

1. **Multiple Kubernetes Clusters**: One in each geographic region
2. **Global Load Balancing**: Using global DNS with health checks and routing policies
3. **Data Synchronization**: For stateful components
4. **Centralized Monitoring**: To observe all regions
5. **GitOps Deployment**: For consistent application deployment across regions

```
                           ┌───────────────────┐
                           │  Global DNS/GLB   │
                           │  (Route53/Cloud   │
                           │      DNS/etc)     │
                           └─────────┬─────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
  ┌────────▼─────────┐     ┌─────────▼────────┐     ┌─────────▼────────┐
  │  Kubernetes      │     │  Kubernetes      │     │  Kubernetes      │
  │  Region: us-east │     │  Region: eu-west │     │  Region: ap-east │
  └────────┬─────────┘     └─────────┬────────┘     └─────────┬────────┘
           │                         │                         │
           │                         │                         │
  ┌────────▼─────────┐     ┌─────────▼────────┐     ┌─────────▼────────┐
  │  Regional        │     │  Regional        │     │  Regional        │
  │  Services        │     │  Services        │     │  Services        │
  └────────┬─────────┘     └─────────┬────────┘     └─────────┬────────┘
           │                         │                         │
           └─────────────────────────┼─────────────────────────┘
                                     │
                           ┌─────────▼─────────┐
                           │  Data             │
                           │  Synchronization  │
                           └───────────────────┘
```

## Prerequisites

- Multiple cloud provider accounts (AWS, GCP, Azure, etc.) or a single provider with multiple regions
- Domain name with DNS provider that supports global load balancing
- Terraform or other infrastructure as code tool
- kubectl, helm, and other Kubernetes tools
- Git repository for configuration management

## Project Implementation

### Step 1: Set Up Regional Kubernetes Clusters

We'll create three Kubernetes clusters in different regions using Terraform.

#### AWS EKS Clusters

```hcl
# main.tf
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast_1"
  region = "ap-southeast-1"
}

module "eks_us_east_1" {
  providers = {
    aws = aws.us_east_1
  }

  source          = "terraform-aws-modules/eks/aws"
  version         = "18.0.0"
  cluster_name    = "multi-region-east"
  cluster_version = "1.23"

  vpc_id     = module.vpc_us_east_1.vpc_id
  subnet_ids = module.vpc_us_east_1.private_subnets

  eks_managed_node_groups = {
    default = {
      min_size     = 2
      max_size     = 5
      desired_size = 3
      instance_types = ["t3.large"]
    }
  }
}

# Repeat similar configuration for eu_west_1 and ap_southeast_1
# with corresponding modules for vpc_eu_west_1 and vpc_ap_southeast_1
```

#### GKE Clusters

```hcl
# For GCP, you would use:
provider "google" {
  alias   = "us_central1"
  project = var.project_id
  region  = "us-central1"
}

provider "google" {
  alias   = "europe_west1"
  project = var.project_id
  region  = "europe-west1"
}

provider "google" {
  alias   = "asia_east1"
  project = var.project_id
  region  = "asia-east1"
}

module "gke_us_central1" {
  providers = {
    google = google.us_central1
  }

  source     = "terraform-google-modules/kubernetes-engine/google"
  project_id = var.project_id
  name       = "multi-region-central"
  region     = "us-central1"
  
  network    = module.vpc.network_name
  subnetwork = module.vpc.subnets["us-central1"].name
  
  node_pools = [
    {
      name         = "default-node-pool"
      machine_type = "e2-standard-2"
      min_count    = 2
      max_count    = 5
      auto_scaling = true
    }
  ]
}

# Repeat similar configuration for europe_west1 and asia_east1
```

#### AKS Clusters

```hcl
# For Azure, you would use:
provider "azurerm" {
  alias = "east_us"
  features {}
}

provider "azurerm" {
  alias = "west_europe"
  features {}
}

provider "azurerm" {
  alias = "east_asia"
  features {}
}

module "aks_east_us" {
  providers = {
    azurerm = azurerm.east_us
  }
  
  source              = "Azure/aks/azurerm"
  resource_group_name = azurerm_resource_group.east_us.name
  kubernetes_version  = "1.23"
  prefix              = "multi-region-east"
  
  default_node_pool = {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D2_v2"
  }
}

# Repeat similar configuration for west_europe and east_asia
```

### Step 2: Set Up Global DNS with Health Checks

#### AWS Route 53 with Health Checks

```hcl
resource "aws_route53_health_check" "us_east" {
  fqdn              = aws_lb.us_east_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "us-east-health-check"
  }
}

resource "aws_route53_health_check" "eu_west" {
  fqdn              = aws_lb.eu_west_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "eu-west-health-check"
  }
}

resource "aws_route53_health_check" "ap_southeast" {
  fqdn              = aws_lb.ap_southeast_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "ap-southeast-health-check"
  }
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = "www.example.com"
  type    = "WEIGHTED"
  
  health_check_id = aws_route53_health_check.us_east.id
  set_identifier  = "us-east"
  ttl             = 60
  records         = [aws_lb.us_east_1.dns_name]
  weight          = 100
}

# Repeat similar records for eu_west and ap_southeast
```

#### GCP Cloud DNS with Load Balancing

```hcl
# Global load balancer with GCP
resource "google_compute_global_address" "default" {
  name = "global-app-ip"
}

resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-rule"
  target     = google_compute_target_http_proxy.default.self_link
  port_range = "80"
  ip_address = google_compute_global_address.default.address
}

resource "google_compute_health_check" "default" {
  name               = "health-check"
  check_interval_sec = 5
  timeout_sec        = 5
  http_health_check {
    port         = 80
    request_path = "/health"
  }
}

resource "google_compute_backend_service" "us_central1" {
  name        = "backend-us-central1"
  protocol    = "HTTP"
  timeout_sec = 10
  health_checks = [google_compute_health_check.default.self_link]
  backend {
    group = google_compute_instance_group_manager.us_central1.instance_group
  }
}

# Repeat for other regions and complete the load balancer configuration
```

#### Azure Traffic Manager

```hcl
resource "azurerm_traffic_manager_profile" "global" {
  name                   = "global-app-tm"
  resource_group_name    = azurerm_resource_group.global.name
  traffic_routing_method = "Performance"

  dns_config {
    relative_name = "app"
    ttl           = 60
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 30
    timeout_in_seconds           = 10
    tolerated_number_of_failures = 3
  }
}

resource "azurerm_traffic_manager_endpoint" "east_us" {
  name                = "east-us"
  resource_group_name = azurerm_resource_group.global.name
  profile_name        = azurerm_traffic_manager_profile.global.name
  type                = "externalEndpoints"
  target              = azurerm_public_ip.east_us.fqdn
  endpoint_location   = "East US"
}

# Repeat for other regions
```

### Step 3: Configure Kubernetes for Multi-Region

#### Set Up Cluster Contexts

Create a script to manage multiple clusters:

```bash
#!/bin/bash
# setup-kubeconfigs.sh

# Get kubeconfigs from each provider
aws eks update-kubeconfig --name multi-region-east --region us-east-1 --alias eks-us-east
aws eks update-kubeconfig --name multi-region-west --region eu-west-1 --alias eks-eu-west
aws eks update-kubeconfig --name multi-region-ap --region ap-southeast-1 --alias eks-ap-southeast

# List available contexts
kubectl config get-contexts

# Create a function to run a command on all clusters
kubectlall() {
  for ctx in eks-us-east eks-eu-west eks-ap-southeast; do
    echo "Context: $ctx"
    kubectl --context=$ctx "$@"
    echo "-----"
  done
}

# Export the function
export -f kubectlall
```

#### Install Common Infrastructure

Deploy common infrastructure components on all clusters:

```bash
# Install Cert-Manager
kubectlall apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml

# Install Prometheus for monitoring
kubectlall apply -f https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/manifests/setup/0namespace-namespace.yaml
kubectlall apply -f https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/manifests/setup/0prometheusrules-crd.yaml
kubectlall apply -f https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/manifests/setup/0servicemonitors-crd.yaml
kubectlall apply -f https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/manifests/prometheus-operator-deployment.yaml

# Install an ingress controller on each cluster
kubectlall apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

### Step 4: Set Up Data Synchronization for Stateful Components

#### Database Replication Strategy

For multi-region stateful components, we can use:

1. **Regional Databases with Global Replication**
2. **Global Database Services** (like Amazon Aurora Global Database, Azure Cosmos DB, or Google Cloud Spanner)

Example for PostgreSQL with replication across regions:

```yaml
# primary-db.yaml for US East region
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
spec:
  serviceName: postgres-primary
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      role: primary
  template:
    metadata:
      labels:
        app: postgres
        role: primary
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres
    role: primary
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

```yaml
# replica-db.yaml for other regions
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-replica
spec:
  serviceName: postgres-replica
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      role: replica
  template:
    metadata:
      labels:
        app: postgres
        role: replica
    spec:
      containers:
      - name: postgres
        image: postgres:14
        command:
        - bash
        - "-c"
        - |
          rm -rf /var/lib/postgresql/data/*
          until pg_basebackup -h postgres-primary.default.svc.cluster.local -D /var/lib/postgresql/data -U replicator -v -P -R; do
            echo "Waiting for primary to connect..."
            sleep 10
          done
          echo "host replication replicator 0.0.0.0/0 md5" >> /var/lib/postgresql/data/pg_hba.conf
          exec postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
spec:
  selector:
    app: postgres
    role: replica
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

#### Stateless Application Strategy

For stateless applications, we'll use GitOps to ensure consistency across regions:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: web-application:1.0.0
        env:
        - name: REGION
          valueFrom:
            configMapKeyRef:
              name: region-config
              key: region
        - name: DB_HOST
          value: postgres-replica  # Will use local replica in each region
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

Create region-specific ConfigMaps:

```bash
# US East region
kubectl --context=eks-us-east create configmap region-config --from-literal=region=us-east-1

# EU West region
kubectl --context=eks-eu-west create configmap region-config --from-literal=region=eu-west-1

# AP Southeast region
kubectl --context=eks-ap-southeast create configmap region-config --from-literal=region=ap-southeast-1
```

### Step 5: Implement GitOps for Multi-Region Deployment

We'll use ArgoCD for multi-region GitOps:

```yaml
# Install ArgoCD in each region
kubectlall apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create an app-of-apps pattern for managing applications
cat <<EOF > app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/multi-region-apps.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Apply this to each cluster
kubectlall apply -f app-of-apps.yaml
```

In your Git repository, create an apps directory with application definitions:

```yaml
# apps/web-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/multi-region-apps.git
    targetRevision: HEAD
    path: web-application
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

And create the web-application directory with your Kubernetes resources.

### Step 6: Set Up Centralized Monitoring

We'll use Prometheus and Grafana for monitoring, with a central instance that scrapes metrics from all regions:

```yaml
# central-prometheus-values.yaml
prometheus:
  additionalScrapeConfigs:
    - job_name: 'federate-us-east'
      scrape_interval: 15s
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job!=""}'
      static_configs:
        - targets:
          - 'prometheus-us-east.example.com'
          
    - job_name: 'federate-eu-west'
      scrape_interval: 15s
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job!=""}'
      static_configs:
        - targets:
          - 'prometheus-eu-west.example.com'
          
    - job_name: 'federate-ap-southeast'
      scrape_interval: 15s
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job!=""}'
      static_configs:
        - targets:
          - 'prometheus-ap-southeast.example.com'

grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
          
  dashboards:
    default:
      multi-region-overview:
        json: |
          {
            "annotations": {...},
            "editable": true,
            "gnetId": null,
            "graphTooltip": 0,
            "id": 1,
            "links": [],
            "panels": [
              {
                "aliasColors": {},
                "bars": false,
                "dashLength": 10,
                "dashes": false,
                "datasource": "Prometheus",
                "fill": 1,
                "fillGradient": 0,
                "gridPos": {
                  "h": 8,
                  "w": 12,
                  "x": 0,
                  "y": 0
                },
                "hiddenSeries": false,
                "id": 2,
                "legend": {
                  "avg": false,
                  "current": false,
                  "max": false,
                  "min": false,
                  "show": true,
                  "total": false,
                  "values": false
                },
                "lines": true,
                "linewidth": 1,
                "nullPointMode": "null",
                "options": {
                  "dataLinks": []
                },
                "percentage": false,
                "pointradius": 2,
                "points": false,
                "renderer": "flot",
                "seriesOverrides": [],
                "spaceLength": 10,
                "stack": false,
                "steppedLine": false,
                "targets": [
                  {
                    "expr": "sum(rate(http_requests_total[5m])) by (region)",
                    "refId": "A"
                  }
                ],
                "thresholds": [],
                "timeFrom": null,
                "timeRegions": [],
                "timeShift": null,
                "title": "HTTP Requests by Region",
                "tooltip": {
                  "shared": true,
                  "sort": 0,
                  "value_type": "individual"
                },
                "type": "graph",
                "xaxis": {
                  "buckets": null,
                  "mode": "time",
                  "name": null,
                  "show": true,
                  "values": []
                },
                "yaxes": [
                  {
                    "format": "short",
                    "label": null,
                    "logBase": 1,
                    "max": null,
                    "min": null,
                    "show": true
                  },
                  {
                    "format": "short",
                    "label": null,
                    "logBase": 1,
                    "max": null,
                    "min": null,
                    "show": true
                  }
                ],
                "yaxis": {
                  "align": false,
                  "alignLevel": null
                }
              }
            ],
            "refresh": "10s",
            "schemaVersion": 22,
            "style": "dark",
            "tags": [],
            "templating": {
              "list": []
            },
            "time": {
              "from": "now-6h",
              "to": "now"
            },
            "timepicker": {},
            "timezone": "",
            "title": "Multi-Region Overview",
            "uid": "multi-region",
            "version": 1
          }
```

### Step 7: Implement Global Health Checking

Create a health check service to monitor your application in each region:

```yaml
# health-check-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-checker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-checker
  template:
    metadata:
      labels:
        app: health-checker
    spec:
      containers:
      - name: health-checker
        image: curlimages/curl:7.83.1
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            curl -s http://web-application/health | grep -q "ok" && echo "Health check passed" || echo "Health check failed"
            sleep 10
          done
---
apiVersion: v1
kind: Service
metadata:
  name: health-checker
spec:
  selector:
    app: health-checker
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Step 8: Set Up Disaster Recovery Procedure

Create a disaster recovery plan as a Kubernetes Job that can be triggered manually:

```yaml
# dr-plan.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: dr-failover
spec:
  template:
    spec:
      containers:
      - name: dr-failover
        image: amazon/aws-cli:latest
        command:
        - /bin/sh
        - -c
        - |
          # Update Route53 to point all traffic to surviving regions
          aws route53 change-resource-record-sets \
            --hosted-zone-id ${HOSTED_ZONE_ID} \
            --change-batch '{
              "Changes": [
                {
                  "Action": "UPSERT",
                  "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "WEIGHTED",
                    "SetIdentifier": "eu-west",
                    "Weight": 100,
                    "ResourceRecords": [
                      {"Value": "eu-west-nlb.example.com"}
                    ],
                    "TTL": 60
                  }
                },
                {
                  "Action": "UPSERT",
                  "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "WEIGHTED",
                    "SetIdentifier": "ap-southeast",
                    "Weight": 100,
                    "ResourceRecords": [
                      {"Value": "ap-southeast-nlb.example.com"}
                    ],
                    "TTL": 60
                  }
                },
                {
                  "Action": "UPSERT",
                  "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "WEIGHTED",
                    "SetIdentifier": "us-east",
                    "Weight": 0,
                    "ResourceRecords": [
                      {"Value": "us-east-nlb.example.com"}
                    ],
                    "TTL": 60
                  }
                }
              ]
            }'
          
          # Notify team
          curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"DR failover completed: Traffic redirected away from us-east"}' \
            ${SLACK_WEBHOOK_URL}
        env:
        - name: HOSTED_ZONE_ID
          valueFrom:
            secretKeyRef:
              name: dr-secrets
              key: hosted-zone-id
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: dr-secrets
              key: slack-webhook-url
      restartPolicy: Never
  backoffLimit: 2
```

### Step 9: Test Application Deployments and Failover

Create a script to deploy, test, and verify failover for your multi-region application:

```bash
#!/bin/bash
# test-deployment-and-failover.sh

# Deploy application to all regions
kubectlall apply -f web-application.yaml

# Wait for deployments to be ready
for ctx in eks-us-east eks-eu-west eks-ap-southeast; do
  echo "Waiting for deployment in $ctx..."
  kubectl --context=$ctx rollout status deployment/web-application
done

# Test connectivity to each region
for ctx in eks-us-east eks-eu-west eks-ap-southeast; do
  echo "Testing connectivity in $ctx..."
  INGRESS_IP=$(kubectl --context=$ctx get svc nginx-ingress-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  curl -s -H "Host: www.example.com" http://$INGRESS_IP/health
done

# Test global DNS
echo "Testing global DNS..."
curl -s https://www.example.com/health

# Simulate failure in us-east region
echo "Simulating failure in us-east region..."
kubectl --context=eks-us-east scale deployment web-application --replicas=0

# Wait for health checks to detect failure
echo "Waiting for health checks to detect failure (30 seconds)..."
sleep 30

# Test global DNS again to verify failover
echo "Testing global DNS after failover..."
curl -s https://www.example.com/health

# Restore us-east region
echo "Restoring us-east region..."
kubectl --context=eks-us-east scale deployment web-application --replicas=3

# Wait for the deployment to be ready
kubectl --context=eks-us-east rollout status deployment/web-application

echo "Test completed."
```

### Step 10: Implement Security Best Practices

Ensure consistent security across all regions:

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
  name: allow-web-traffic
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53 # DNS
```

Apply this to all clusters:

```bash
kubectlall apply -f network-policy.yaml
```

## Multi-Region Best Practices

### 1. Data Management

- **Application Data**: Use a global database or regional databases with replication
- **User Sessions**: Store in distributed caches or token-based auth (JWT)
- **Static Content**: Use a CDN for global distribution

### 2. Performance Considerations

- **Latency Awareness**: Serve users from the closest region
- **Regional Resources**: Size regional clusters appropriately for regional traffic
- **Database Proximity**: Ensure applications connect to the closest database replica

### 3. Cost Optimization

- **Regional Scaling**: Scale each region based on regional traffic patterns
- **Resource Allocation**: Rightsize resources in each region
- **Reserved Instances**: Use reserved instances for baseline capacity

### 4. Operational Considerations

- **Deployment Strategy**: Roll out changes gradually across regions
- **Monitoring**: Monitor both global and regional metrics
- **Runbooks**: Create runbooks for region failure and recovery

## Troubleshooting

### Common Issues and Solutions

1. **DNS Propagation Delays**
   - Issue: DNS changes can take time to propagate globally
   - Solution: Use low TTL values (60 seconds) during initial setup and failover tests

2. **Data Synchronization Lag**
   - Issue: Replica databases may lag behind the primary
   - Solution: Monitor replication lag and implement retry logic in applications

3. **Cluster Authentication**
   - Issue: Managing multiple kubeconfig files
   - Solution: Use a kubeconfig manager or organize contexts with clear naming

4. **Inconsistent Application Versions**
   - Issue: Different versions running in different regions
   - Solution: Use GitOps with synchronized deployments

5. **Cross-Region Networking Issues**
   - Issue: Connectivity problems between regions
   - Solution: Use health checks and circuit breakers to handle regional failures

## Conclusion

A multi-region Kubernetes architecture with global DNS provides high availability and geographic redundancy for your applications. By implementing proper data synchronization, centralized monitoring, and automated failover, you can ensure your applications remain available even during regional outages.

This project demonstrates how to:
- Set up Kubernetes clusters in multiple regions
- Configure global DNS with health checks
- Implement data synchronization strategies
- Deploy applications consistently with GitOps
- Monitor the entire system centrally
- Implement and test disaster recovery procedures

With these components in place, your application will be resilient to regional failures and provide low-latency access to users around the world.