# Terraform with Kubernetes

## Introduction to Terraform

Terraform is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp that allows users to define and provision infrastructure resources using a declarative configuration language. Terraform automates the creation, modification, and deletion of infrastructure across multiple cloud providers and services, including Kubernetes clusters.

Key features that make Terraform particularly well-suited for Kubernetes environments include:

- **Provider Ecosystem**: Native support for major Kubernetes platforms (AWS EKS, Azure AKS, Google GKE)
- **Declarative Language**: HCL (HashiCorp Configuration Language) for defining resources
- **State Management**: Tracking of resource states for incremental changes
- **Plan and Apply**: Preview changes before applying them
- **Modularity**: Reusable components for common infrastructure patterns
- **Extensibility**: Custom providers and modules for specialized needs

This guide explores how to use Terraform to provision, manage, and configure Kubernetes clusters and the resources running on them.

## Terraform Basics for Kubernetes

### Core Concepts

#### Providers

Providers are plugins that interact with cloud platforms, infrastructure services, or other APIs. For Kubernetes, the primary providers are:

```hcl
# Kubernetes provider for interacting with existing clusters
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "my-context"
}

# Helm provider for deploying Helm charts
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

# Cloud provider for creating the cluster itself
provider "aws" {
  region = "us-west-2"
}
```

#### Resources

Resources represent infrastructure objects like clusters, nodes, and Kubernetes resources:

```hcl
# Creating a Kubernetes namespace
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = "elastic-system"
    labels = {
      environment = var.environment
    }
  }
}
```

#### Data Sources

Data sources allow retrieving information about existing infrastructure:

```hcl
# Retrieving information about an existing Kubernetes Service
data "kubernetes_service" "elasticsearch" {
  metadata {
    name      = "elasticsearch"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }
}
```

#### Variables and Outputs

Variables provide flexibility, while outputs expose information about provisioned resources:

```hcl
# Variable definitions
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

# Output definitions
output "elasticsearch_endpoint" {
  value       = data.kubernetes_service.elasticsearch.status[0].load_balancer[0].ingress[0].hostname
  description = "Elasticsearch service endpoint"
}
```

### Directory Structure

A well-organized Terraform project for Kubernetes might look like:

```
terraform-k8s/
├── main.tf               # Main configurations
├── providers.tf          # Provider configurations
├── variables.tf          # Input variable definitions
├── outputs.tf            # Output definitions
├── terraform.tfvars      # Variable values
├── modules/              # Reusable modules
│   ├── eks-cluster/      # EKS cluster module
│   ├── k8s-elasticsearch/ # Elasticsearch on Kubernetes module
│   └── k8s-monitoring/   # Monitoring stack module
└── environments/         # Environment-specific configurations
    ├── dev/
    ├── staging/
    └── production/
```

### Terraform Workflow for Kubernetes

The standard Terraform workflow applies for Kubernetes projects:

1. **Initialize**: `terraform init` 
   - Downloads providers 
   - Sets up backend for state storage

2. **Plan**: `terraform plan`
   - Shows changes to be made
   - Validates configurations

3. **Apply**: `terraform apply`
   - Creates, updates, or deletes resources
   - Updates state file

4. **Destroy**: `terraform destroy`
   - Removes all resources managed by Terraform
   - Useful for ephemeral environments

## Creating Kubernetes Clusters with Terraform

### AWS EKS Cluster

Using Terraform to provision an Amazon EKS cluster:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"

  name = "eks-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 18.0"

  cluster_name    = "${var.environment}-eks-cluster"
  cluster_version = "1.22"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Managed node groups
  eks_managed_node_groups = {
    general = {
      desired_size = 2
      min_size     = 1
      max_size     = 3

      instance_types = ["m5.large"]
      capacity_type  = "ON_DEMAND"
      
      labels = {
        Environment = var.environment
        Role        = "general"
      }

      tags = {
        Environment = var.environment
        Terraform   = "true"
      }
    }
  }

  # Enable EKS add-ons
  cluster_addons = {
    coredns = {
      resolve_conflicts = "OVERWRITE"
    }
    kube-proxy = {}
    vpc-cni = {
      resolve_conflicts = "OVERWRITE"
    }
  }
}

# Output the kubeconfig command
output "configure_kubectl" {
  value       = "aws eks update-kubeconfig --region ${var.region} --name ${module.eks.cluster_id}"
  description = "Command to configure kubectl"
}
```

### Azure AKS Cluster

Creating an Azure Kubernetes Service (AKS) cluster:

```hcl
resource "azurerm_resource_group" "aks_rg" {
  name     = "${var.environment}-aks-rg"
  location = var.location
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "${var.environment}-aks-cluster"
  location            = azurerm_resource_group.aks_rg.location
  resource_group_name = azurerm_resource_group.aks_rg.name
  dns_prefix          = "${var.environment}-aks"
  kubernetes_version  = var.kubernetes_version

  default_node_pool {
    name       = "default"
    node_count = var.node_count
    vm_size    = "Standard_D2s_v3"
    os_disk_size_gb = 30
    
    tags = {
      Environment = var.environment
    }
  }

  identity {
    type = "SystemAssigned"
  }

  addon_profile {
    oms_agent {
      enabled                    = true
      log_analytics_workspace_id = azurerm_log_analytics_workspace.logs.id
    }
    
    kube_dashboard {
      enabled = false
    }
  }

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}

# Create Log Analytics workspace for monitoring
resource "azurerm_log_analytics_workspace" "logs" {
  name                = "${var.environment}-aks-logs"
  location            = azurerm_resource_group.aks_rg.location
  resource_group_name = azurerm_resource_group.aks_rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Output the kubeconfig
output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
```

### Google GKE Cluster

Creating a Google Kubernetes Engine (GKE) cluster:

```hcl
resource "google_container_cluster" "primary" {
  name     = "${var.environment}-gke-cluster"
  location = var.region
  
  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  # Enable Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

resource "google_container_node_pool" "general_nodes" {
  name       = "${google_container_cluster.primary.name}-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = var.node_count

  node_config {
    preemptible  = var.preemptible
    machine_type = var.machine_type

    # Google recommends custom service accounts that have cloud-platform scope and permissions granted via IAM Roles.
    service_account = google_service_account.default.email
    oauth_scopes    = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    labels = {
      environment = var.environment
    }

    # Enable workload identity on this node pool
    workload_metadata_config {
      node_metadata = "GKE_METADATA_SERVER"
    }
  }
}

resource "google_service_account" "default" {
  account_id   = "${var.environment}-gke-sa"
  display_name = "GKE Service Account"
}

resource "google_project_iam_member" "service_account" {
  project = var.project_id
  role    = "roles/container.nodeServiceAccount"
  member  = "serviceAccount:${google_service_account.default.email}"
}

# Create VPC network for GKE
resource "google_compute_network" "vpc" {
  name                    = "${var.environment}-vpc"
  auto_create_subnetworks = false
}

# Create subnet for GKE
resource "google_compute_subnetwork" "subnet" {
  name          = "${var.environment}-subnet"
  region        = var.region
  network       = google_compute_network.vpc.name
  ip_cidr_range = "10.0.0.0/16"
  
  secondary_ip_range {
    range_name    = "pod-range"
    ip_cidr_range = "10.1.0.0/16"
  }
  
  secondary_ip_range {
    range_name    = "service-range"
    ip_cidr_range = "10.2.0.0/16"
  }
}
```

## Managing Kubernetes Resources with Terraform

Once a Kubernetes cluster is provisioned, Terraform can manage the resources within it.

### Namespaces and RBAC

Setting up namespaces and role-based access control:

```hcl
# Create a namespace for the Elastic Stack
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = "elastic-system"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

# Create a service account for Elasticsearch
resource "kubernetes_service_account" "elasticsearch" {
  metadata {
    name      = "elasticsearch"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }
}

# Create a cluster role for Elasticsearch
resource "kubernetes_cluster_role" "elasticsearch" {
  metadata {
    name = "elasticsearch-role"
  }

  rule {
    api_groups = [""]
    resources  = ["pods", "nodes", "namespaces"]
    verbs      = ["get", "list"]
  }
}

# Bind the role to the service account
resource "kubernetes_cluster_role_binding" "elasticsearch" {
  metadata {
    name = "elasticsearch-binding"
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = kubernetes_cluster_role.elasticsearch.metadata[0].name
  }

  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.elasticsearch.metadata[0].name
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }
}
```

### ConfigMaps and Secrets

Managing configuration and secrets:

```hcl
# Create a ConfigMap for Elasticsearch
resource "kubernetes_config_map" "elasticsearch_config" {
  metadata {
    name      = "elasticsearch-config"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }

  data = {
    "elasticsearch.yml" = <<-EOT
      cluster.name: ${var.cluster_name}
      network.host: 0.0.0.0
      discovery.zen.minimum_master_nodes: ${min(var.elasticsearch_master_nodes, 2)}
      bootstrap.memory_lock: true
      xpack.security.enabled: ${var.xpack_security_enabled}
      xpack.monitoring.collection.enabled: true
    EOT
    
    "jvm.options" = <<-EOT
      -Xms${var.elasticsearch_jvm_memory}
      -Xmx${var.elasticsearch_jvm_memory}
    EOT
  }
}

# Create a Secret for Elasticsearch credentials
resource "kubernetes_secret" "elasticsearch_credentials" {
  metadata {
    name      = "elasticsearch-credentials"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }

  data = {
    username = var.elasticsearch_username
    password = var.elasticsearch_password
  }

  type = "Opaque"
}
```

### Deployments and StatefulSets

Deploying containerized applications:

```hcl
# Create a StatefulSet for Elasticsearch
resource "kubernetes_stateful_set" "elasticsearch" {
  metadata {
    name      = "elasticsearch"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
    labels = {
      app = "elasticsearch"
    }
  }

  spec {
    replicas = var.elasticsearch_nodes
    
    selector {
      match_labels = {
        app = "elasticsearch"
      }
    }
    
    service_name = "elasticsearch"
    
    template {
      metadata {
        labels = {
          app = "elasticsearch"
        }
      }
      
      spec {
        service_account_name = kubernetes_service_account.elasticsearch.metadata[0].name
        
        init_container {
          name    = "fix-permissions"
          image   = "busybox:1.32"
          command = ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          
          volume_mount {
            name       = "data"
            mount_path = "/usr/share/elasticsearch/data"
          }
          
          security_context {
            run_as_user = 0
          }
        }
        
        container {
          name  = "elasticsearch"
          image = "docker.elastic.co/elasticsearch/elasticsearch:${var.elasticsearch_version}"
          
          resources {
            limits = {
              cpu    = var.elasticsearch_cpu_limit
              memory = var.elasticsearch_memory_limit
            }
            requests = {
              cpu    = var.elasticsearch_cpu_request
              memory = var.elasticsearch_memory_request
            }
          }
          
          port {
            container_port = 9200
            name           = "http"
          }
          
          port {
            container_port = 9300
            name           = "transport"
          }
          
          env {
            name = "node.name"
            value_from {
              field_ref {
                field_path = "metadata.name"
              }
            }
          }
          
          env {
            name  = "discovery.seed_hosts"
            value = "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
          }
          
          env {
            name  = "cluster.initial_master_nodes"
            value = "elasticsearch-0,elasticsearch-1,elasticsearch-2"
          }
          
          env {
            name  = "ES_JAVA_OPTS"
            value = "-Xms${var.elasticsearch_jvm_memory} -Xmx${var.elasticsearch_jvm_memory}"
          }
          
          volume_mount {
            name       = "config"
            mount_path = "/usr/share/elasticsearch/config/elasticsearch.yml"
            sub_path   = "elasticsearch.yml"
          }
          
          volume_mount {
            name       = "config"
            mount_path = "/usr/share/elasticsearch/config/jvm.options.d/jvm.options"
            sub_path   = "jvm.options"
          }
          
          volume_mount {
            name       = "data"
            mount_path = "/usr/share/elasticsearch/data"
          }
        }
        
        volume {
          name = "config"
          config_map {
            name = kubernetes_config_map.elasticsearch_config.metadata[0].name
          }
        }
      }
    }
    
    volume_claim_template {
      metadata {
        name = "data"
      }
      
      spec {
        access_modes = ["ReadWriteOnce"]
        
        resources {
          requests = {
            storage = var.elasticsearch_storage_size
          }
        }
        
        storage_class_name = var.storage_class_name
      }
    }
  }
}
```

### Services and Ingress

Exposing applications within and outside the cluster:

```hcl
# Create a Service for Elasticsearch
resource "kubernetes_service" "elasticsearch" {
  metadata {
    name      = "elasticsearch"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
    labels = {
      app = "elasticsearch"
    }
  }
  
  spec {
    selector = {
      app = "elasticsearch"
    }
    
    port {
      name        = "http"
      port        = 9200
      target_port = 9200
    }
    
    port {
      name        = "transport"
      port        = 9300
      target_port = 9300
    }
    
    type = "ClusterIP"
  }
}

# Create a Service for Kibana
resource "kubernetes_service" "kibana" {
  metadata {
    name      = "kibana"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
    labels = {
      app = "kibana"
    }
  }
  
  spec {
    selector = {
      app = "kibana"
    }
    
    port {
      port        = 5601
      target_port = 5601
    }
    
    type = "ClusterIP"
  }
}

# Create an Ingress for Kibana
resource "kubernetes_ingress_v1" "kibana" {
  metadata {
    name      = "kibana"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
    
    annotations = {
      "kubernetes.io/ingress.class"                = "nginx"
      "nginx.ingress.kubernetes.io/ssl-redirect"   = "true"
      "nginx.ingress.kubernetes.io/backend-protocol" = "HTTP"
    }
  }
  
  spec {
    rule {
      host = "kibana.${var.domain_name}"
      
      http {
        path {
          path = "/"
          
          backend {
            service {
              name = kubernetes_service.kibana.metadata[0].name
              port {
                number = 5601
              }
            }
          }
        }
      }
    }
    
    tls {
      hosts       = ["kibana.${var.domain_name}"]
      secret_name = "kibana-tls"
    }
  }
}
```

## Helm Integration

Terraform can deploy applications using Helm through the Helm provider.

### Basic Helm Chart Deployment

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "elasticsearch" {
  name       = "elasticsearch"
  repository = "https://helm.elastic.co"
  chart      = "elasticsearch"
  version    = "7.17.1"
  namespace  = kubernetes_namespace.elastic_system.metadata[0].name
  
  values = [
    <<-EOT
    replicas: 3
    minimumMasterNodes: 2
    
    esJavaOpts: "-Xmx1g -Xms1g"
    
    resources:
      requests:
        cpu: "1000m"
        memory: "2Gi"
      limits:
        cpu: "2000m"
        memory: "2Gi"
    
    volumeClaimTemplate:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi
      storageClassName: ${var.storage_class_name}
    EOT
  ]
  
  set {
    name  = "antiAffinity"
    value = "soft"
  }
  
  set {
    name  = "image"
    value = "docker.elastic.co/elasticsearch/elasticsearch"
  }
  
  set {
    name  = "imageTag"
    value = var.elasticsearch_version
  }
  
  depends_on = [
    kubernetes_namespace.elastic_system
  ]
}

resource "helm_release" "kibana" {
  name       = "kibana"
  repository = "https://helm.elastic.co"
  chart      = "kibana"
  version    = "7.17.1"
  namespace  = kubernetes_namespace.elastic_system.metadata[0].name
  
  values = [
    <<-EOT
    elasticsearchHosts: "http://elasticsearch-master:9200"
    
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
    
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
      hosts:
        - host: kibana.${var.domain_name}
          paths:
            - path: /
      tls:
        - secretName: kibana-tls
          hosts:
            - kibana.${var.domain_name}
    EOT
  ]
  
  set {
    name  = "image"
    value = "docker.elastic.co/kibana/kibana"
  }
  
  set {
    name  = "imageTag"
    value = var.elasticsearch_version
  }
  
  depends_on = [
    helm_release.elasticsearch
  ]
}
```

### Custom Helm Charts

For more complex deployments, you can use custom Helm charts:

```hcl
resource "helm_release" "elk_stack" {
  name      = "elk"
  chart     = "${path.module}/charts/elk-stack"
  namespace = kubernetes_namespace.elk.metadata[0].name
  
  values = [
    file("${path.module}/values/${var.environment}.yaml")
  ]
  
  set {
    name  = "global.environment"
    value = var.environment
  }
  
  set {
    name  = "elasticsearch.replicas"
    value = var.elasticsearch_replicas
  }
  
  set {
    name  = "kibana.ingress.host"
    value = "kibana.${var.domain_name}"
  }
}
```

## Terraform Modules for Kubernetes

Creating reusable modules enhances productivity and ensures consistent deployment patterns.

### Example: ELK Stack Module

```hcl
# modules/elk-stack/main.tf

variable "namespace" {
  description = "Kubernetes namespace for the ELK Stack"
  type        = string
  default     = "elastic-system"
}

variable "elasticsearch_version" {
  description = "Elasticsearch version"
  type        = string
  default     = "7.17.1"
}

variable "elasticsearch_replicas" {
  description = "Number of Elasticsearch replicas"
  type        = number
  default     = 3
}

variable "storage_class_name" {
  description = "Storage class name for Elasticsearch PVCs"
  type        = string
}

variable "domain_name" {
  description = "Domain name for ingress"
  type        = string
}

resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = var.namespace
  }
}

module "elasticsearch" {
  source = "../elasticsearch"
  
  namespace           = kubernetes_namespace.elastic_system.metadata[0].name
  version             = var.elasticsearch_version
  replicas            = var.elasticsearch_replicas
  storage_class_name  = var.storage_class_name
}

module "kibana" {
  source = "../kibana"
  
  namespace           = kubernetes_namespace.elastic_system.metadata[0].name
  version             = var.elasticsearch_version
  elasticsearch_host  = module.elasticsearch.service_name
  domain_name         = var.domain_name
  
  depends_on = [
    module.elasticsearch
  ]
}

module "logstash" {
  source = "../logstash"
  
  namespace           = kubernetes_namespace.elastic_system.metadata[0].name
  version             = var.elasticsearch_version
  elasticsearch_host  = module.elasticsearch.service_name
  
  depends_on = [
    module.elasticsearch
  ]
}

output "elasticsearch_service" {
  value = module.elasticsearch.service_name
}

output "kibana_url" {
  value = "https://kibana.${var.domain_name}"
}
```

### Example: Usage of the ELK Stack Module

```hcl
# main.tf
module "elk_stack_dev" {
  source = "./modules/elk-stack"
  
  namespace             = "elastic-dev"
  elasticsearch_version = "7.17.1"
  elasticsearch_replicas = 1
  storage_class_name    = "standard"
  domain_name           = "dev.example.com"
}

module "elk_stack_prod" {
  source = "./modules/elk-stack"
  
  namespace             = "elastic-prod"
  elasticsearch_version = "7.17.1"
  elasticsearch_replicas = 3
  storage_class_name    = "premium-ssd"
  domain_name           = "example.com"
}
```

## State Management with Terraform

Properly managing the state file is critical for team collaboration and production environments.

### Remote State Storage

Using AWS S3 for remote state storage:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "elk-stack/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

Using Azure Storage for remote state:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "terraformstate12345"
    container_name       = "tfstate"
    key                  = "elk-stack.terraform.tfstate"
  }
}
```

Using Google Cloud Storage for remote state:

```hcl
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "elk-stack"
  }
}
```

### State Locking

State locking prevents concurrent operations that could corrupt the state:

- **AWS**: DynamoDB table for locking
- **Azure**: Blob lease for locking
- **GCP**: GCS object locking

```hcl
# AWS DynamoDB Table for State Locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Terraform Best Practices for Kubernetes

### Modularization

Break infrastructure into logical modules:

- **Networking**: VPC, subnets, security groups
- **Cluster**: Kubernetes cluster and node pools
- **Applications**: Application deployments, services, ingress
- **Monitoring**: Prometheus, Grafana, alerting

### Environments and Workspaces

Manage different environments using workspaces:

```bash
# Create workspace for development
terraform workspace new dev

# Create workspace for production
terraform workspace new prod

# Select workspace
terraform workspace select dev

# Apply configurations for selected workspace
terraform apply -var-file=dev.tfvars
```

### Secret Management

Avoid storing sensitive information in Terraform files:

1. **Environment Variables**:
```bash
export TF_VAR_elasticsearch_password="secure-password"
```

2. **Secret Management Services**:
```hcl
data "aws_secretsmanager_secret_version" "elasticsearch_credentials" {
  secret_id = "elasticsearch/credentials"
}

locals {
  credentials = jsondecode(data.aws_secretsmanager_secret_version.elasticsearch_credentials.secret_string)
}

resource "kubernetes_secret" "elasticsearch_credentials" {
  metadata {
    name      = "elasticsearch-credentials"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }

  data = {
    username = local.credentials.username
    password = local.credentials.password
  }

  type = "Opaque"
}
```

3. **External Secret Operators**:
```hcl
resource "kubernetes_manifest" "elasticsearch_external_secret" {
  manifest = {
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ExternalSecret"
    metadata = {
      name      = "elasticsearch-credentials"
      namespace = kubernetes_namespace.elastic_system.metadata[0].name
    }
    spec = {
      refreshInterval = "1h"
      secretStoreRef = {
        name = "aws-secretsmanager"
        kind = "ClusterSecretStore"
      }
      target = {
        name           = "elasticsearch-credentials"
        creationPolicy = "Owner"
      }
      data = [
        {
          secretKey = "username"
          remoteRef = {
            key      = "elasticsearch/credentials"
            property = "username"
          }
        },
        {
          secretKey = "password"
          remoteRef = {
            key      = "elasticsearch/credentials"
            property = "password"
          }
        }
      ]
    }
  }
}
```

### Validation and Testing

Implement validation and testing for Terraform code:

1. **Terraform Validate**:
```bash
terraform validate
```

2. **Unit Tests with Terratest**:
```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestElasticStackDeployment(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/elk-stack",
        Vars: map[string]interface{}{
            "namespace": "test-elastic",
            "elasticsearch_replicas": 1,
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    elasticsearchService := terraform.Output(t, terraformOptions, "elasticsearch_service")
    assert.Equal(t, "elasticsearch", elasticsearchService)
}
```

3. **Linting with TFLint**:
```bash
tflint --enable-rule=terraform_deprecated_syntax
```

4. **Security Scanning with tfsec**:
```bash
tfsec .
```

### CI/CD Integration

Implementing CI/CD for Terraform and Kubernetes:

```yaml
# GitHub Actions workflow
name: Terraform CI/CD

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/**'
  pull_request:
    branches: [ main ]
    paths:
    - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./terraform
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.0.0
    
    - name: Terraform Format
      run: terraform fmt -check
    
    - name: TFLint
      uses: terraform-linters/setup-tflint@v1
      with:
        tflint_version: v0.29.0
    
    - name: Run TFLint
      run: tflint --enable-rule=terraform_deprecated_syntax
    
    - name: tfsec
      uses: aquasecurity/tfsec-action@v1.0.0
    
    - name: Terraform Init
      run: terraform init -backend=false
    
    - name: Terraform Validate
      run: terraform validate
    
    - name: Terraform Plan
      if: github.event_name == 'pull_request'
      run: terraform plan -var-file=dev.tfvars
      
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve -var-file=dev.tfvars
```

## Real-World Applications of Terraform with Kubernetes

### Complete ELK Stack Deployment

A comprehensive example of deploying the ELK Stack:

```hcl
module "elk_stack" {
  source = "./modules/elk-stack"
  
  # General settings
  namespace   = "elastic-system"
  environment = "production"
  domain_name = "example.com"
  
  # Elasticsearch settings
  elasticsearch = {
    version     = "7.17.1"
    replicas    = 3
    heap_size   = "2g"
    storage_size = "100Gi"
    storage_class = "premium-ssd"
    node_affinity = {
      required = [{
        key      = "kubernetes.io/arch"
        operator = "In"
        values   = ["amd64"]
      }]
    }
  }
  
  # Kibana settings
  kibana = {
    version   = "7.17.1"
    replicas  = 2
    resources = {
      limits = {
        cpu    = "1000m"
        memory = "1Gi"
      }
      requests = {
        cpu    = "500m"
        memory = "512Mi"
      }
    }
    ingress = {
      enabled     = true
      host        = "kibana.example.com"
      path        = "/"
      tls_enabled = true
      annotations = {
        "cert-manager.io/cluster-issuer" = "letsencrypt-prod"
      }
    }
  }
  
  # Logstash settings
  logstash = {
    version   = "7.17.1"
    replicas  = 2
    heap_size = "1g"
    pipelines = {
      main = {
        config  = file("${path.module}/pipelines/main.conf")
        workers = 2
      }
      beats = {
        config  = file("${path.module}/pipelines/beats.conf")
        workers = 2
      }
    }
  }
  
  # Filebeat settings
  filebeat = {
    version = "7.17.1"
    enabled = true
    config  = file("${path.module}/configs/filebeat.yml")
  }
  
  # Security settings
  security = {
    tls_enabled   = true
    create_certs  = true
    authentication = {
      enabled      = true
      admin_user   = "admin"
      admin_password_secret = "elastic-admin-password"
    }
  }
  
  # Monitoring settings
  monitoring = {
    prometheus_enabled  = true
    grafana_dashboards = true
    alerts = {
      email = {
        enabled    = true
        recipients = ["alerts@example.com"]
      }
      slack = {
        enabled   = true
        webhook   = "https://hooks.slack.com/services/TXXXXXXXX/YYYYYYYYYY/ZZZZZZZZZZZZZZZZZZZZZZZZ"
        channel   = "#elk-alerts"
      }
    }
  }
}
```

### Multi-Cluster, Multi-Region Deployment

Managing Kubernetes across multiple clusters and regions:

```hcl
locals {
  regions = {
    us_east = {
      region = "us-east-1"
      cidr   = "10.0.0.0/16"
      azs    = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
    us_west = {
      region = "us-west-2"
      cidr   = "10.1.0.0/16"
      azs    = ["us-west-2a", "us-west-2b", "us-west-2c"]
    }
    eu_west = {
      region = "eu-west-1"
      cidr   = "10.2.0.0/16"
      azs    = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
    }
  }
}

# Create a VPC in each region
module "vpc" {
  for_each = local.regions
  
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.environment}-vpc-${each.key}"
  cidr = each.value.cidr
  
  azs             = each.value.azs
  private_subnets = [for i, az in each.value.azs : cidrsubnet(each.value.cidr, 8, i)]
  public_subnets  = [for i, az in each.value.azs : cidrsubnet(each.value.cidr, 8, i + 10)]
  
  enable_nat_gateway = true
  single_nat_gateway = true
  
  tags = {
    Environment = var.environment
    Region      = each.key
    Terraform   = "true"
  }
  
  providers = {
    aws = aws.${each.key}
  }
}

# Create an EKS cluster in each region
module "eks" {
  for_each = local.regions
  
  source  = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "${var.environment}-eks-${each.key}"
  cluster_version = "1.22"
  
  vpc_id     = module.vpc[each.key].vpc_id
  subnet_ids = module.vpc[each.key].private_subnets
  
  eks_managed_node_groups = {
    general = {
      desired_size = 3
      min_size     = 2
      max_size     = 5
      
      instance_types = ["m5.large"]
      capacity_type  = "ON_DEMAND"
      
      labels = {
        Environment = var.environment
        Region      = each.key
      }
    }
  }
  
  providers = {
    aws = aws.${each.key}
  }
}

# Deploy ELK Stack to each cluster
module "elk_stack" {
  for_each = module.eks
  
  source = "./modules/elk-stack"
  
  namespace   = "elastic-system"
  environment = var.environment
  domain_name = "${each.key}.example.com"
  
  elasticsearch = {
    version     = "7.17.1"
    replicas    = 3
    heap_size   = "2g"
    storage_size = "100Gi"
    storage_class = "gp2"
  }
  
  kibana = {
    version   = "7.17.1"
    replicas  = 2
    ingress = {
      enabled     = true
      host        = "kibana.${each.key}.example.com"
      tls_enabled = true
    }
  }
  
  # Configure cross-cluster replication for disaster recovery
  cross_cluster_replication = {
    enabled = true
    remote_clusters = [for k, v in module.elk_stack : {
      name     = k
      seeds    = ["elasticsearch-${k}.elastic-system.svc:9300"]
    } if k != each.key]
  }
  
  providers = {
    kubernetes = kubernetes.${each.key}
    helm       = helm.${each.key}
  }
  
  depends_on = [
    module.eks
  ]
}
```

### GitOps with Terraform and Flux

Implementing GitOps for Kubernetes using Terraform and Flux:

```hcl
# Install Flux using Terraform
resource "kubernetes_namespace" "flux_system" {
  metadata {
    name = "flux-system"
  }
}

resource "helm_release" "flux" {
  name       = "flux"
  repository = "https://charts.fluxcd.io"
  chart      = "flux2"
  namespace  = kubernetes_namespace.flux_system.metadata[0].name
  
  set {
    name  = "git.url"
    value = "ssh://git@github.com/example/gitops-repo.git"
  }
  
  set {
    name  = "git.branch"
    value = "main"
  }
  
  set {
    name  = "git.path"
    value = "clusters/${var.environment}"
  }
  
  set {
    name  = "syncGarbageCollection.enabled"
    value = "true"
  }
}

# Deploy the Flux manifests using the Kubernetes provider
resource "kubernetes_manifest" "flux_git_repository" {
  manifest = {
    apiVersion = "source.toolkit.fluxcd.io/v1beta1"
    kind       = "GitRepository"
    metadata = {
      name      = "flux-system"
      namespace = "flux-system"
    }
    spec = {
      interval = "1m0s"
      ref = {
        branch = "main"
      }
      url = "https://github.com/example/gitops-repo.git"
    }
  }
  
  depends_on = [
    helm_release.flux
  ]
}

resource "kubernetes_manifest" "flux_kustomization" {
  manifest = {
    apiVersion = "kustomize.toolkit.fluxcd.io/v1beta1"
    kind       = "Kustomization"
    metadata = {
      name      = "flux-system"
      namespace = "flux-system"
    }
    spec = {
      interval = "10m0s"
      path     = "./clusters/${var.environment}"
      prune    = true
      sourceRef = {
        kind = "GitRepository"
        name = "flux-system"
      }
      validation = "client"
    }
  }
  
  depends_on = [
    kubernetes_manifest.flux_git_repository
  ]
}
```

## Conclusion

Terraform is a powerful tool for managing Kubernetes infrastructure across multiple environments and cloud providers. By using Terraform with Kubernetes, you gain:

- **Declarative Infrastructure**: Define your entire infrastructure stack in code
- **Reproducibility**: Create consistent environments from development to production
- **Modularity**: Reuse common patterns across teams and projects
- **Multi-Cloud Support**: Deploy to AWS, Azure, GCP, and other providers with the same configuration
- **Lifecycle Management**: Manage the full lifecycle of resources from creation to deletion
- **Integration with CI/CD**: Automate infrastructure provisioning and application deployment

The most effective Terraform implementations for Kubernetes follow these principles:

1. **Start with Cluster Provisioning**: Use Terraform to create and manage the Kubernetes cluster itself
2. **Use Modules for Reusability**: Create reusable modules for common infrastructure patterns
3. **Implement State Management**: Store state remotely and enable locking for team collaboration
4. **Manage Environments Separately**: Use workspaces or separate state files for different environments
5. **Integrate with GitOps**: Combine Terraform for infrastructure with tools like Flux or ArgoCD for applications
6. **Implement Testing and Validation**: Test infrastructure code before applying it to production
7. **Handle Secrets Securely**: Use secret management solutions instead of storing sensitive data in Terraform files

By applying these practices, you can build robust, scalable, and secure Kubernetes infrastructure that evolves with your organization's needs.