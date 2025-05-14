# Infrastructure as Code for Kubernetes

## Introduction to Infrastructure as Code

Infrastructure as Code (IaC) is a methodology that allows developers and operations teams to manage and provision infrastructure through machine-readable definition files rather than through physical hardware configuration or interactive configuration tools. It brings the same versioning, testing, and automated deployment practices from application development to infrastructure management.

In the context of Kubernetes, IaC enables:

- **Declarative infrastructure definitions**: Explicitly defining the desired state of your Kubernetes clusters
- **Version-controlled infrastructure**: Tracking changes to infrastructure over time
- **Repeatable deployments**: Ensuring consistent environments across development, testing, and production
- **Automated provisioning**: Reducing manual intervention and human error
- **Self-documenting systems**: Infrastructure definitions serve as documentation

This shift from manual, imperative infrastructure management to automated, declarative infrastructure as code is a fundamental aspect of modern DevOps practices and cloud-native architectures.

## Core Principles of IaC

### Declarative vs. Imperative Approaches

#### Declarative Approach

The declarative approach focuses on defining the desired end state of the infrastructure without specifying the step-by-step process to achieve it.

```yaml
# Declarative example with YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
        resources:
          limits:
            memory: 2Gi
```

Advantages:
- **Idempotency**: Running the same configuration multiple times results in the same state
- **Self-documenting**: The configuration describes the infrastructure state
- **Easier troubleshooting**: Comparing desired state with actual state
- **Aligns with Kubernetes philosophy**: Kubernetes itself is declarative

#### Imperative Approach

The imperative approach specifies the exact sequence of commands to achieve the desired state.

```bash
# Imperative example with CLI commands
kubectl create namespace elastic
kubectl create deployment elasticsearch --image=docker.elastic.co/elasticsearch/elasticsearch:7.15.0 -n elastic
kubectl scale deployment elasticsearch --replicas=3 -n elastic
kubectl set resources deployment elasticsearch --limits=memory=2Gi -n elastic
```

Advantages:
- **Procedural clarity**: Explicit steps can be easier to understand for newcomers
- **Immediate feedback**: Each command provides immediate results
- **Direct control**: Precise control over the execution order

### Immutability vs. Mutability

#### Immutable Infrastructure

Immutable infrastructure treats infrastructure elements as disposable and replaceable. Instead of updating existing resources, new resources are deployed with the updated configuration.

Benefits:
- **Predictability**: Known state with each deployment
- **Rollback simplicity**: Return to previous versions by redeploying
- **Reduced configuration drift**: Eliminates incremental changes
- **Consistency**: All instances have identical configurations

#### Mutable Infrastructure

Mutable infrastructure allows for in-place updates and modifications to existing resources.

Benefits:
- **Resource efficiency**: Avoids creating new resources for small changes
- **Faster updates**: Some changes can be applied more quickly
- **State preservation**: Maintains ongoing state and connections

In Kubernetes, both approaches are used:
- **Immutable**: Recreating pods with new configurations
- **Mutable**: Updating ConfigMaps that applications read dynamically

### Infrastructure Versioning

Version controlling infrastructure code offers numerous benefits:

- **Change history**: Track when and why infrastructure changed
- **Collaboration**: Multiple team members can work on infrastructure
- **Auditability**: Review and approve infrastructure changes
- **Rollback capability**: Return to a working state if issues arise

Best practices:
- **Branch protection**: Require code reviews for infrastructure changes
- **CI validation**: Automatically validate infrastructure code
- **Small, focused changes**: Make incremental changes easier to review
- **Meaningful commit messages**: Document the purpose of changes

## IaC Tools for Kubernetes

### Terraform

Terraform is a popular open-source IaC tool that enables users to define infrastructure as code across multiple cloud providers and services. It follows a declarative approach using HashiCorp Configuration Language (HCL).

#### Terraform Basics

**Directory Structure**:
```
kubernetes-infra/
├── main.tf           # Main configuration file
├── variables.tf      # Variable definitions
├── outputs.tf        # Output definitions
├── providers.tf      # Provider configurations
└── modules/          # Reusable modules
    └── eks-cluster/  # Example module for EKS
```

**Provider Configuration**:
```hcl
# providers.tf
provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "aws" {
  region = "us-west-2"
}
```

**Resource Definition**:
```hcl
# main.tf
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = "elastic-system"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_deployment" "elasticsearch" {
  metadata {
    name      = "elasticsearch"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
    labels = {
      app = "elasticsearch"
    }
  }

  spec {
    replicas = var.elasticsearch_replicas

    selector {
      match_labels = {
        app = "elasticsearch"
      }
    }

    template {
      metadata {
        labels = {
          app = "elasticsearch"
        }
      }

      spec {
        container {
          image = "docker.elastic.co/elasticsearch/elasticsearch:${var.elasticsearch_version}"
          name  = "elasticsearch"

          resources {
            limits = {
              cpu    = "1000m"
              memory = "2Gi"
            }
            requests = {
              cpu    = "500m"
              memory = "1Gi"
            }
          }
        }
      }
    }
  }
}
```

**Variables**:
```hcl
# variables.tf
variable "environment" {
  description = "Environment name (e.g., dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "elasticsearch_version" {
  description = "Elasticsearch version"
  type        = string
  default     = "7.15.0"
}

variable "elasticsearch_replicas" {
  description = "Number of Elasticsearch replicas"
  type        = number
  default     = 3
}
```

#### Terraform Workflow

1. **Initialize**: `terraform init` downloads providers and modules
2. **Plan**: `terraform plan` shows changes that will be made
3. **Apply**: `terraform apply` applies the changes
4. **Destroy**: `terraform destroy` removes all resources

#### Terraform Modules

Modules allow for reusable, composable infrastructure components:

```hcl
# modules/elastic-stack/main.tf
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = var.namespace
  }
}

resource "kubernetes_deployment" "elasticsearch" {
  # Elasticsearch configuration
}

resource "kubernetes_deployment" "kibana" {
  # Kibana configuration
}

resource "kubernetes_deployment" "logstash" {
  # Logstash configuration
}
```

Usage:
```hcl
# main.tf
module "elastic_stack_prod" {
  source = "./modules/elastic-stack"
  
  namespace            = "elastic-prod"
  elasticsearch_version = "7.15.0"
  elasticsearch_replicas = 5
}

module "elastic_stack_dev" {
  source = "./modules/elastic-stack"
  
  namespace            = "elastic-dev"
  elasticsearch_version = "7.15.0"
  elasticsearch_replicas = 1
}
```

#### Terraform for Kubernetes Cluster Provisioning

Example: Provisioning an EKS cluster
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 18.0"

  cluster_name    = "my-eks-cluster"
  cluster_version = "1.22"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      desired_size = 2
      min_size     = 1
      max_size     = 3

      instance_types = ["m5.large"]
      capacity_type  = "ON_DEMAND"
    }
  }
}
```

### Pulumi

Pulumi is an infrastructure as code tool that allows you to use familiar programming languages (TypeScript, JavaScript, Python, Go, .NET) instead of domain-specific languages.

#### Pulumi Basics

**Directory Structure**:
```
pulumi-k8s/
├── index.ts         # Main program file (TypeScript)
├── package.json     # Node.js package file
├── tsconfig.json    # TypeScript configuration
└── Pulumi.yaml      # Pulumi project file
```

**Project Configuration**:
```yaml
# Pulumi.yaml
name: elk-infrastructure
runtime: nodejs
description: ELK Stack on Kubernetes using Pulumi
```

**TypeScript Definition**:
```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

const config = new pulumi.Config();
const environment = config.require("environment");
const esVersion = config.get("esVersion") || "7.15.0";
const replicas = config.getNumber("replicas") || 3;

// Create a namespace
const namespace = new k8s.core.v1.Namespace("elastic-system", {
    metadata: {
        name: `elastic-system-${environment}`,
        labels: {
            environment: environment,
            "managed-by": "pulumi"
        }
    }
});

// Create an Elasticsearch deployment
const elasticsearch = new k8s.apps.v1.Deployment("elasticsearch", {
    metadata: {
        name: "elasticsearch",
        namespace: namespace.metadata.name
    },
    spec: {
        replicas: replicas,
        selector: {
            matchLabels: {
                app: "elasticsearch"
            }
        },
        template: {
            metadata: {
                labels: {
                    app: "elasticsearch"
                }
            },
            spec: {
                containers: [{
                    name: "elasticsearch",
                    image: `docker.elastic.co/elasticsearch/elasticsearch:${esVersion}`,
                    resources: {
                        limits: {
                            cpu: "1000m",
                            memory: "2Gi"
                        },
                        requests: {
                            cpu: "500m",
                            memory: "1Gi"
                        }
                    }
                }]
            }
        }
    }
});

export const namespaceName = namespace.metadata.name;
export const elasticsearchService = elasticsearch.metadata.name;
```

#### Pulumi Workflow

1. **Initialize**: `pulumi new kubernetes-typescript` creates a new project
2. **Configure**: `pulumi config set environment dev` sets configuration
3. **Preview**: `pulumi preview` shows changes
4. **Update**: `pulumi up` applies changes
5. **Destroy**: `pulumi destroy` removes resources

#### Pulumi for Kubernetes Cluster Provisioning

Example: Provisioning an AKS cluster
```typescript
import * as azure from "@pulumi/azure";
import * as azuread from "@pulumi/azuread";
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Create an Azure Resource Group
const resourceGroup = new azure.core.ResourceGroup("k8s-rg", {
    location: "East US",
});

// Create an AD service principal
const adApp = new azuread.Application("k8s-app");
const adSp = new azuread.ServicePrincipal("k8s-sp", {
    applicationId: adApp.applicationId,
});
const adSpPassword = new azuread.ServicePrincipalPassword("k8s-sp-pwd", {
    servicePrincipalId: adSp.id,
    endDate: "2099-01-01T00:00:00Z",
});

// Create an AKS cluster
const k8sCluster = new azure.containerservice.KubernetesCluster("k8s", {
    resourceGroupName: resourceGroup.name,
    location: resourceGroup.location,
    defaultNodePool: {
        name: "default",
        nodeCount: 2,
        vmSize: "Standard_D2_v2",
    },
    dnsPrefix: `${pulumi.getStack()}-kube`,
    linuxProfile: {
        adminUsername: "adminuser",
        sshKey: {
            keyData: "ssh-rsa ...",
        },
    },
    servicePrincipal: {
        clientId: adApp.applicationId,
        clientSecret: adSpPassword.value,
    },
});

// Export the kubeconfig
export const kubeconfig = k8sCluster.kubeConfigRaw;
```

### CDK for Kubernetes (cdk8s)

CDK for Kubernetes (cdk8s) is an open-source software development framework for defining Kubernetes applications using familiar programming languages.

#### cdk8s Basics

**Directory Structure**:
```
cdk8s-app/
├── main.ts                # Main application file
├── package.json           # Node.js package file
├── tsconfig.json          # TypeScript configuration
└── cdk8s.yaml             # cdk8s configuration file
```

**Project Configuration**:
```yaml
# cdk8s.yaml
language: typescript
app: npm run build
imports:
  - k8s
```

**TypeScript Definition**:
```typescript
// main.ts
import { Construct } from 'constructs';
import { App, Chart } from 'cdk8s';
import * as k8s from 'cdk8s-plus-24';

class ElasticStack extends Chart {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    // Create a namespace
    const namespace = new k8s.Namespace(this, 'elastic-system', {
      metadata: {
        name: 'elastic-system',
      },
    });

    // Create an Elasticsearch deployment
    const elasticsearch = new k8s.Deployment(this, 'elasticsearch', {
      metadata: {
        namespace: 'elastic-system',
      },
      containers: [{
        image: 'docker.elastic.co/elasticsearch/elasticsearch:7.15.0',
        name: 'elasticsearch',
        resources: {
          limits: {
            cpu: k8s.Cpu.millis(1000),
            memory: k8s.Size.gibibytes(2),
          },
          requests: {
            cpu: k8s.Cpu.millis(500),
            memory: k8s.Size.gibibytes(1),
          },
        },
      }],
      replicas: 3,
    });

    // Create a service for Elasticsearch
    const elasticService = new k8s.Service(this, 'elasticsearch-svc', {
      metadata: {
        namespace: 'elastic-system',
      },
      ports: [{ port: 9200, targetPort: 9200 }],
      selector: elasticsearch,
    });
  }
}

const app = new App();
new ElasticStack(app, 'elastic-stack');
app.synth();
```

#### cdk8s Workflow

1. **Initialize**: `cdk8s init typescript-app` creates a new project
2. **Develop**: Write your infrastructure code
3. **Synthesize**: `cdk8s synth` generates Kubernetes YAML
4. **Apply**: `kubectl apply -f dist/` applies the YAML

### Helm Charts

Helm is a package manager for Kubernetes that allows you to define, install, and upgrade Kubernetes applications. Helm Charts are packages of pre-configured Kubernetes resources.

#### Helm Chart Structure

```
elasticsearch/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Template helpers
└── charts/             # Dependent charts
```

**Chart.yaml**:
```yaml
apiVersion: v2
name: elasticsearch
description: Elasticsearch Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "7.15.0"
```

**values.yaml**:
```yaml
# Default values for elasticsearch
replicaCount: 3

image:
  repository: docker.elastic.co/elasticsearch/elasticsearch
  tag: 7.15.0
  pullPolicy: IfNotPresent

resources:
  limits:
    cpu: 1000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi

service:
  type: ClusterIP
  port: 9200
```

**templates/deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "elasticsearch.fullname" . }}
  labels:
    {{- include "elasticsearch.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "elasticsearch.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "elasticsearch.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 9200
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

#### Helm Workflow

1. **Create**: `helm create elasticsearch` creates a chart template
2. **Package**: `helm package elasticsearch` packages the chart
3. **Install**: `helm install elastic-release ./elasticsearch` installs the chart
4. **Upgrade**: `helm upgrade elastic-release ./elasticsearch` upgrades the release
5. **Uninstall**: `helm uninstall elastic-release` removes the release

#### Helm Charts for Complex Applications

Helm excels at managing complex applications with multiple components:

```yaml
# ELK Stack Umbrella Chart values.yaml
elasticsearch:
  enabled: true
  replicaCount: 3
  # ... other Elasticsearch settings

kibana:
  enabled: true
  replicaCount: 1
  elasticsearchUrl: http://elastic-release-elasticsearch:9200
  # ... other Kibana settings

logstash:
  enabled: true
  replicaCount: 2
  # ... other Logstash settings
```

### Kustomize

Kustomize is a configuration customization tool built into kubectl that allows you to customize YAML configurations without templates.

#### Kustomize Basics

**Base Configuration**:
```
base/
├── kustomization.yaml
├── deployment.yaml
├── service.yaml
└── configmap.yaml
```

**kustomization.yaml**:
```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: elasticsearch
```

**Overlay for Production**:
```
overlays/
└── production/
    ├── kustomization.yaml
    └── deployment-patch.yaml
```

**Production Kustomization**:
```yaml
# overlays/production/kustomization.yaml
resources:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml

namespace: elasticsearch-prod

commonLabels:
  environment: production
```

**Deployment Patch**:
```yaml
# overlays/production/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: elasticsearch
        resources:
          limits:
            memory: "4Gi"
            cpu: "2000m"
          requests:
            memory: "2Gi"
            cpu: "1000m"
```

#### Kustomize Workflow

1. **Create Base**: Define base Kubernetes manifests
2. **Create Overlays**: Define environment-specific overlays
3. **Preview**: `kubectl kustomize overlays/production` shows the generated YAML
4. **Apply**: `kubectl apply -k overlays/production` applies the configuration

## Integrating IaC with CI/CD

### Automated Infrastructure Deployment

Integrating IaC with CI/CD pipelines automates the deployment and validation of infrastructure changes:

**GitHub Actions Example**:
```yaml
name: Deploy Infrastructure

on:
  push:
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
    
    - name: Terraform Init
      run: terraform init
    
    - name: Terraform Format
      run: terraform fmt -check
    
    - name: Terraform Validate
      run: terraform validate
    
    - name: Terraform Plan
      run: terraform plan -out=tfplan
    
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

### Infrastructure Testing

IaC code should be tested like application code:

1. **Static Analysis**:
   - Detect syntax errors and security issues
   - Tools: terraform-lint, tfsec, checkov

2. **Unit Testing**:
   - Test individual resources or modules
   - Tools: Terratest, KUTTL (Kubernetes Test Tool)

3. **Integration Testing**:
   - Test interactions between components
   - Tools: Terratest, InSpec, Goss

4. **Compliance Testing**:
   - Ensure infrastructure meets security and compliance requirements
   - Tools: Open Policy Agent (OPA), Conftest

**Example Terratest**:
```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestElasticsearchDeployment(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/elasticsearch",
        Vars: map[string]interface{}{
            "environment": "test",
            "replicas": 1,
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    namespaceName := terraform.Output(t, terraformOptions, "namespace_name")
    assert.Equal(t, "elastic-system-test", namespaceName)
}
```

### GitOps for Infrastructure

GitOps extends the principles of IaC by using Git as the single source of truth for infrastructure:

1. **Repository Structure**:
```
gitops-repo/
├── clusters/
│   ├── production/
│   │   ├── infrastructure/
│   │   │   ├── networking.yaml
│   │   │   └── storage.yaml
│   │   └── applications/
│   │       ├── elastic-stack/
│   │       │   ├── elasticsearch.yaml
│   │       │   ├── kibana.yaml
│   │       │   └── logstash.yaml
│   │       └── monitoring/
│   └── staging/
├── .flux.yaml
└── README.md
```

2. **Flux or ArgoCD Integration**:
```yaml
# .flux.yaml
version: 1
commandUpdated:
  generators:
    - command: kustomize build .
  updaters:
    - containerImage:
        command: >-
          kustomize edit set image
          ${IMAGE_NAME}=${IMAGE_NAME}:${IMAGE_TAG}
```

3. **Automated Synchronization**:
   - Flux/ArgoCD monitors the Git repository
   - Changes to configuration automatically apply to the cluster
   - Drift is automatically corrected

## Infrastructure as Code Best Practices

### Modular and Reusable Components

Create reusable modules for common infrastructure patterns:

```hcl
# Terraform ELK Stack module
module "elk_stack" {
  source = "./modules/elk-stack"
  
  environment = "production"
  elasticsearch_nodes = 3
  kibana_replicas = 2
  logstash_pipelines = {
    "main" = {
      config = file("${path.module}/pipelines/main.conf")
    }
  }
}
```

### Environment Management

Manage different environments using configuration files or directories:

**Terraform Workspaces**:
```bash
terraform workspace new production
terraform workspace select production
terraform apply -var-file=production.tfvars
```

**Environment-Specific Variables**:
```hcl
# production.tfvars
elasticsearch_nodes = 5
elasticsearch_memory = "4Gi"
kibana_replicas = 3
```

### State Management

Properly manage the state of your infrastructure:

**Terraform Remote State**:
```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state-bucket"
    key    = "elk-stack/terraform.tfstate"
    region = "us-west-2"
    dynamodb_table = "terraform-locks"
  }
}
```

**State Locking**:
- Prevent concurrent modifications
- Essential for team environments
- Supported by most cloud storage backends

### Security in IaC

Implement security best practices:

1. **Secret Management**:
   - Never store secrets in version control
   - Use services like HashiCorp Vault, AWS Secrets Manager
   - Inject secrets at deployment time

```hcl
# Secure handling of sensitive data
resource "kubernetes_secret" "elasticsearch_credentials" {
  metadata {
    name = "elasticsearch-credentials"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }

  data = {
    username = var.elasticsearch_username
    password = var.elasticsearch_password
  }

  type = "Opaque"
}
```

2. **Infrastructure Security**:
   - Define network policies and security groups
   - Use RBAC for Kubernetes resources
   - Limit service account permissions

```hcl
resource "kubernetes_network_policy" "elasticsearch_network_policy" {
  metadata {
    name = "elasticsearch-network-policy"
    namespace = kubernetes_namespace.elastic_system.metadata[0].name
  }

  spec {
    pod_selector {
      match_labels = {
        app = "elasticsearch"
      }
    }

    ingress {
      from {
        pod_selector {
          match_labels = {
            app = "kibana"
          }
        }
      }
      
      ports {
        port = "9200"
        protocol = "TCP"
      }
    }

    policy_types = ["Ingress"]
  }
}
```

3. **Automated Security Scanning**:
   - Scan IaC code for security issues
   - Integrate security checks into CI/CD pipelines

```yaml
# Security scanning in CI/CD
- name: Security Scan
  run: |
    docker run --rm -v $(pwd):/src aquasec/tfsec:latest /src
```

### Documentation and Tagging

Maintain comprehensive documentation and use tags for better organization:

```hcl
# Using tags/labels for organization
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = "elastic-system"
    labels = {
      environment = var.environment
      team = "platform"
      cost-center = "infrastructure"
      application = "elk-stack"
      managed-by = "terraform"
    }
  }
}
```

## Real-World IaC Examples

### ELK Stack Deployment with Terraform

```hcl
module "elk_stack" {
  source = "./modules/elk-stack"
  
  # General settings
  namespace = "elastic-system"
  environment = "production"
  
  # Elasticsearch settings
  elasticsearch = {
    version = "7.15.0"
    replicas = 3
    heap_size = "1g"
    storage_size = "100Gi"
    storage_class = "premium-ssd"
    node_affinity = {
      required = [{
        key = "kubernetes.io/arch"
        operator = "In"
        values = ["amd64"]
      }]
    }
  }
  
  # Kibana settings
  kibana = {
    version = "7.15.0"
    replicas = 2
    resources = {
      limits = {
        cpu = "500m"
        memory = "1Gi"
      }
      requests = {
        cpu = "200m"
        memory = "512Mi"
      }
    }
  }
  
  # Logstash settings
  logstash = {
    version = "7.15.0"
    replicas = 2
    pipelines = {
      main = {
        config = file("${path.module}/pipelines/main.conf")
      }
      http = {
        config = file("${path.module}/pipelines/http.conf")
      }
    }
  }
  
  # Security settings
  security = {
    tls_enabled = true
    create_certs = true
    authentication = {
      enabled = true
      admin_user = "admin"
      admin_password_secret = "elastic-admin-password"
    }
  }
  
  # Monitoring settings
  monitoring = {
    prometheus_enabled = true
    grafana_dashboards = true
  }
}
```

### Multi-Environment Kubernetes Clusters with Pulumi

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as eks from "@pulumi/eks";
import * as k8s from "@pulumi/kubernetes";

const config = new pulumi.Config();
const environment = config.require("environment");
const nodeCount = config.getNumber("nodeCount") || 3;
const nodeSize = config.get("nodeSize") || "t3.medium";

// Create an EKS cluster
const cluster = new eks.Cluster(`${environment}-cluster`, {
    vpcId: config.get("vpcId"),
    subnetIds: config.requireObject<string[]>("subnetIds"),
    instanceType: nodeSize,
    desiredCapacity: nodeCount,
    minSize: nodeCount,
    maxSize: nodeCount * 2,
    storageClasses: "gp2",
    deployDashboard: false,
});

// Create a Kubernetes provider instance using the EKS cluster's kubeconfig
const provider = new k8s.Provider(`${environment}-k8s-provider`, {
    kubeconfig: cluster.kubeconfig,
});

// Deploy monitoring stack
const monitoringNamespace = new k8s.core.v1.Namespace("monitoring", {
    metadata: {
        name: "monitoring",
    },
}, { provider });

// Deploy Prometheus operator using Helm
const prometheusOperator = new k8s.helm.v3.Chart("prometheus-operator", {
    chart: "kube-prometheus-stack",
    version: "19.0.3",
    namespace: monitoringNamespace.metadata.name,
    fetchOpts: {
        repo: "https://prometheus-community.github.io/helm-charts",
    },
    values: {
        grafana: {
            persistence: {
                enabled: true,
                size: "10Gi",
            },
            adminPassword: config.getSecret("grafanaPassword"),
        },
        prometheus: {
            prometheusSpec: {
                retention: "15d",
                storageSpec: {
                    volumeClaimTemplate: {
                        spec: {
                            storageClassName: "gp2",
                            accessModes: ["ReadWriteOnce"],
                            resources: {
                                requests: {
                                    storage: "50Gi",
                                },
                            },
                        },
                    },
                },
            },
        },
    },
}, { provider });

// Export the cluster's kubeconfig and endpoint
export const kubeconfig = cluster.kubeconfig;
export const clusterName = cluster.eksCluster.name;
export const clusterEndpoint = cluster.eksCluster.endpoint;
```

### GitOps-Driven Infrastructure

```yaml
# flux-system/gotk-sync.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/example/infrastructure.git
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: infrastructure
  validation: client
```

## Conclusion

Infrastructure as Code has revolutionized how we manage Kubernetes infrastructure, bringing software engineering practices to infrastructure provisioning and management. By treating infrastructure as code, teams can achieve greater consistency, reliability, and automation in their Kubernetes deployments.

Key takeaways:
- **Declarative definitions** simplify infrastructure management
- **Version control** provides change history and collaboration
- **Automation** reduces manual effort and human error
- **Multiple tools** suit different requirements and preferences
- **Integration with CI/CD** enables automated deployment and testing
- **GitOps** extends IaC with continuous reconciliation

For ELK Stack deployments on Kubernetes, IaC offers particular benefits in managing complex distributed systems across multiple environments. By applying these techniques and best practices, teams can build robust, scalable, and secure logging infrastructure that can evolve with their organization's needs.