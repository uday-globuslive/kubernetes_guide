# Pulumi for Kubernetes

Pulumi is an Infrastructure as Code (IaC) platform that allows you to define cloud resources using general-purpose programming languages like TypeScript, JavaScript, Python, Go, C#, and Java. This guide covers how to use Pulumi to manage Kubernetes resources and infrastructure.

## Introduction to Pulumi

Unlike declarative tools like Terraform or Kubernetes YAML files, Pulumi enables you to use familiar programming languages and techniques to define infrastructure. This approach provides several advantages:

- Use loops, conditionals, functions, and classes
- Access the full ecosystem of each language (npm, PyPI, etc.)
- Leverage existing IDEs, testing frameworks, and tools
- Share and reuse code with standard package managers

## Pulumi vs. Other IaC Tools

| Feature | Pulumi | Terraform | Helm | kubectl + YAML |
|---------|--------|-----------|------|----------------|
| **Language** | TypeScript, Python, Go, etc. | HCL | YAML + Go templates | YAML |
| **Programming Model** | Imperative code with declarative results | Declarative | Template-based declarative | Declarative |
| **State Management** | Built-in service or self-hosted | State files | N/A | N/A |
| **Community Resources** | Growing | Extensive | Extensive | Very extensive |
| **Native Kubernetes Support** | Yes | Via provider | Yes | Yes |
| **Multi-cloud Support** | Yes | Yes | No | No |
| **Learning Curve** | Knowledge of programming language + Pulumi | HCL-specific | YAML + Go templates | YAML |

## Installing Pulumi

### CLI Installation

```bash
# MacOS
brew install pulumi

# Windows (with Chocolatey)
choco install pulumi

# Linux
curl -fsSL https://get.pulumi.com | sh
```

### Initial Setup

```bash
# Login to Pulumi service (state management)
pulumi login

# Or use local state
pulumi login file://~/.pulumi

# Initialize a new project
mkdir k8s-project && cd k8s-project
pulumi new kubernetes-typescript
```

## Basic Pulumi Concepts

### Project Structure

A typical Pulumi project for Kubernetes includes:

```
my-k8s-project/
  ├── Pulumi.yaml           # Project metadata
  ├── Pulumi.dev.yaml       # Dev stack configuration
  ├── Pulumi.prod.yaml      # Prod stack configuration
  ├── index.ts              # Main program (TypeScript example)
  ├── package.json          # Dependencies
  ├── tsconfig.json         # TypeScript config
  └── node_modules/         # Dependencies
```

### Core Concepts

1. **Project**: A directory containing Pulumi code
2. **Stack**: An isolated deployment environment (dev, staging, prod)
3. **Resource**: An infrastructure component (pod, deployment, service, etc.)
4. **Provider**: A plugin that interacts with cloud or service APIs
5. **State**: Information about deployed resources

## Kubernetes with Pulumi

### Simple Example: Deploying a Nginx Pod and Service

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Create a namespace
const namespace = new k8s.core.v1.Namespace("app-namespace", {
    metadata: { name: "demo" }
});

// Create a Deployment
const appLabels = { app: "nginx" };
const deployment = new k8s.apps.v1.Deployment("nginx", {
    metadata: { 
        namespace: namespace.metadata.name,
        labels: appLabels 
    },
    spec: {
        selector: { matchLabels: appLabels },
        replicas: 2,
        template: {
            metadata: { labels: appLabels },
            spec: {
                containers: [{
                    name: "nginx",
                    image: "nginx:latest",
                    ports: [{ name: "http", containerPort: 80 }],
                    resources: {
                        requests: {
                            cpu: "100m",
                            memory: "128Mi"
                        },
                        limits: {
                            cpu: "500m",
                            memory: "256Mi"
                        }
                    }
                }]
            }
        }
    }
});

// Create a Service
const service = new k8s.core.v1.Service("nginx", {
    metadata: { 
        namespace: namespace.metadata.name,
        labels: appLabels 
    },
    spec: {
        type: "ClusterIP",
        ports: [{ port: 80, targetPort: "http" }],
        selector: appLabels
    }
});

// Export the Service name and IP
export const serviceName = service.metadata.name;
export const serviceIP = service.spec.clusterIP;
```

### Deploying the Example

```bash
# Preview changes
pulumi preview

# Deploy resources
pulumi up

# Review outputs
pulumi stack output serviceIP

# Destroy resources
pulumi destroy
```

## Advanced Pulumi Patterns for Kubernetes

### Working with Multiple Clusters

Pulumi can target different Kubernetes clusters by configuring the provider:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Use the current kubeconfig context
const defaultProvider = new k8s.Provider("default");

// Use a specific kubeconfig context
const stagingProvider = new k8s.Provider("staging", {
    context: "staging-cluster"
});

// Use an explicit kubeconfig file
const prodProvider = new k8s.Provider("prod", {
    kubeconfig: process.env.PROD_KUBECONFIG
});

// Deploy to default cluster
const defaultNamespace = new k8s.core.v1.Namespace("default-ns", {
    metadata: { name: "demo-default" }
}, { provider: defaultProvider });

// Deploy to staging cluster
const stagingNamespace = new k8s.core.v1.Namespace("staging-ns", {
    metadata: { name: "demo-staging" }
}, { provider: stagingProvider });

// Deploy to production cluster
const prodNamespace = new k8s.core.v1.Namespace("prod-ns", {
    metadata: { name: "demo-prod" }
}, { provider: prodProvider });
```

### Using Component Resources

Component resources help you group related resources together:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Define a component for a web application
class WebAppComponent extends pulumi.ComponentResource {
    public readonly service: k8s.core.v1.Service;
    public readonly deployment: k8s.apps.v1.Deployment;

    constructor(name: string, args: {
        namespace: pulumi.Input<string>,
        image: string,
        port: number,
        replicas: number,
    }, opts?: pulumi.ComponentResourceOptions) {
        super("my:components:WebApp", name, {}, opts);

        const labels = { app: name };
        
        // Create a Deployment
        this.deployment = new k8s.apps.v1.Deployment(`${name}-deployment`, {
            metadata: { 
                namespace: args.namespace,
                labels: labels 
            },
            spec: {
                selector: { matchLabels: labels },
                replicas: args.replicas,
                template: {
                    metadata: { labels: labels },
                    spec: {
                        containers: [{
                            name: name,
                            image: args.image,
                            ports: [{ containerPort: args.port }]
                        }]
                    }
                }
            }
        }, { parent: this });
        
        // Create a Service
        this.service = new k8s.core.v1.Service(`${name}-service`, {
            metadata: { 
                namespace: args.namespace,
                labels: labels 
            },
            spec: {
                type: "ClusterIP",
                ports: [{ port: args.port, targetPort: args.port }],
                selector: labels
            }
        }, { parent: this });
        
        this.registerOutputs({
            deployment: this.deployment,
            service: this.service,
        });
    }
}

// Create a namespace
const namespace = new k8s.core.v1.Namespace("apps", {
    metadata: { name: "applications" }
});

// Create web applications using the component
const frontend = new WebAppComponent("frontend", {
    namespace: namespace.metadata.name,
    image: "nginx:latest",
    port: 80,
    replicas: 3,
});

const backend = new WebAppComponent("backend", {
    namespace: namespace.metadata.name,
    image: "myapp/backend:v1.2.3",
    port: 8080,
    replicas: 2,
});

export const frontendServiceName = frontend.service.metadata.name;
export const backendServiceName = backend.service.metadata.name;
```

### Working with Helm Charts

Pulumi supports deploying Helm charts:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Deploy Prometheus using Helm
const prometheus = new k8s.helm.v3.Chart("prometheus", {
    chart: "prometheus",
    version: "15.5.3",
    fetchOpts: {
        repo: "https://prometheus-community.github.io/helm-charts"
    },
    namespace: "monitoring",
    values: {
        server: {
            persistentVolume: {
                size: "100Gi"
            }
        },
        alertmanager: {
            enabled: true,
            persistentVolume: {
                size: "20Gi"
            }
        }
    }
});

// Deploy Grafana using Helm
const grafana = new k8s.helm.v3.Chart("grafana", {
    chart: "grafana",
    version: "6.24.1",
    fetchOpts: {
        repo: "https://grafana.github.io/helm-charts"
    },
    namespace: "monitoring",
    values: {
        adminPassword: "strongPassword123",
        persistence: {
            enabled: true,
            size: "10Gi"
        },
        datasources: {
            "datasources.yaml": {
                apiVersion: 1,
                datasources: [{
                    name: "Prometheus",
                    type: "prometheus",
                    url: "http://prometheus-server.monitoring.svc.cluster.local",
                    access: "proxy",
                    isDefault: true
                }]
            }
        }
    }
});

export const grafanaService = grafana.getResourceProperty("v1/Service", "monitoring/grafana", "metadata");
```

### Using YAML Manifests

You can import existing YAML manifests and manage them with Pulumi:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";
import * as fs from "fs";
import * as yaml from "js-yaml";

// Load a single YAML file
const fileData = fs.readFileSync("manifests/deployment.yaml", "utf8");
const parsed = yaml.load(fileData) as any;

// Create the resource from YAML
const fromYaml = new k8s.yaml.ConfigFile("from-yaml", {
    file: "manifests/deployment.yaml"
});

// Load multiple YAML files from a directory
const baseDir = "manifests";
const files = fs.readdirSync(baseDir).filter(file => 
    file.endsWith(".yaml") || file.endsWith(".yml")
);

for (const file of files) {
    const configFile = new k8s.yaml.ConfigFile(`config-${file}`, {
        file: `${baseDir}/${file}`
    });
}
```

### Working with CRDs

Pulumi can work with Custom Resource Definitions (CRDs):

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Apply a CRD to the cluster
const certManagerCRD = new k8s.yaml.ConfigFile("cert-manager-crd", {
    file: "https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml"
});

// Create a certificate using the CRD
const certificate = new k8s.apiextensions.CustomResource("example-com", {
    apiVersion: "cert-manager.io/v1",
    kind: "Certificate",
    metadata: {
        name: "example-com",
        namespace: "default"
    },
    spec: {
        secretName: "example-com-tls",
        issuerRef: {
            name: "letsencrypt-prod",
            kind: "ClusterIssuer"
        },
        dnsNames: [
            "example.com",
            "www.example.com"
        ]
    }
}, { dependsOn: certManagerCRD });
```

## Creating a Complete Kubernetes Infrastructure

### Creating a Full EKS Cluster with Pulumi

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";
import * as k8s from "@pulumi/kubernetes";

// Create a VPC for the EKS cluster
const vpc = new awsx.ec2.Vpc("eks-vpc", {
    cidrBlock: "10.0.0.0/16",
    subnetStrategy: awsx.ec2.SubnetAllocationStrategy.Auto,
    numberOfAvailabilityZones: 3,
    tags: {
        Name: "eks-vpc",
        Environment: "production"
    }
});

// Create an EKS cluster
const cluster = new eks.Cluster("production", {
    vpcId: vpc.id,
    subnetIds: vpc.privateSubnetIds,
    instanceType: "t3.large",
    desiredCapacity: 3,
    minSize: 2,
    maxSize: 5,
    nodeRootVolumeSize: 50,
    encryptRootBlockDevice: true,
    tags: {
        Environment: "production"
    }
});

// Export the cluster's kubeconfig
export const kubeconfig = cluster.kubeconfig;

// Create a Kubernetes provider using the EKS cluster
const provider = new k8s.Provider("eks-provider", {
    kubeconfig: kubeconfig
});

// Deploy the Kubernetes dashboard
const dashboard = new k8s.helm.v3.Chart("kubernetes-dashboard", {
    chart: "kubernetes-dashboard",
    version: "5.2.0",
    fetchOpts: {
        repo: "https://kubernetes.github.io/dashboard/"
    },
    namespace: "kubernetes-dashboard",
    values: {
        "extraArgs": ["--token-ttl=0"]
    }
}, { provider });

// Deploy the AWS Load Balancer Controller
const albController = new k8s.helm.v3.Chart("aws-load-balancer-controller", {
    chart: "aws-load-balancer-controller",
    version: "1.4.1",
    fetchOpts: {
        repo: "https://aws.github.io/eks-charts"
    },
    namespace: "kube-system",
    values: {
        clusterName: cluster.eksCluster.name,
        serviceAccount: {
            create: true,
            annotations: {
                "eks.amazonaws.com/role-arn": cluster.instanceRoles[0].arn
            }
        }
    }
}, { provider });

// Deploy Prometheus and Grafana for monitoring
const monitoring = new k8s.helm.v3.Chart("prometheus-stack", {
    chart: "kube-prometheus-stack",
    version: "36.2.0",
    fetchOpts: {
        repo: "https://prometheus-community.github.io/helm-charts"
    },
    namespace: "monitoring",
    values: {
        grafana: {
            persistence: {
                enabled: true,
                size: "10Gi"
            },
            adminPassword: "strongP@ssw0rd"
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
                                    storage: "50Gi"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}, { provider });

// Export useful outputs
export const clusterName = cluster.eksCluster.name;
export const dashboardUrl = pulumi.interpolate`https://kubernetes-dashboard.example.com`;
export const grafanaUrl = pulumi.interpolate`https://grafana.monitoring.example.com`;
```

## Multi-cloud Kubernetes Deployment

Deploying applications across multiple cloud providers:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Configure providers for different clusters
const awsProvider = new k8s.Provider("aws-cluster", {
    kubeconfig: process.env.AWS_KUBECONFIG
});

const gkeProvider = new k8s.Provider("gke-cluster", {
    kubeconfig: process.env.GKE_KUBECONFIG
});

const azureProvider = new k8s.Provider("aks-cluster", {
    kubeconfig: process.env.AKS_KUBECONFIG
});

// Define a function to deploy the same app to multiple clusters
function deployApp(name: string, provider: k8s.Provider, replicas: number) {
    const labels = { app: name };
    
    // Create a namespace
    const namespace = new k8s.core.v1.Namespace(`${name}-ns`, {
        metadata: { name }
    }, { provider });
    
    // Create a deployment
    const deployment = new k8s.apps.v1.Deployment(`${name}-deployment`, {
        metadata: {
            namespace: namespace.metadata.name,
            labels
        },
        spec: {
            replicas,
            selector: { matchLabels: labels },
            template: {
                metadata: { labels },
                spec: {
                    containers: [{
                        name,
                        image: "nginx:latest",
                        ports: [{ containerPort: 80 }]
                    }]
                }
            }
        }
    }, { provider });
    
    // Create a service
    const service = new k8s.core.v1.Service(`${name}-service`, {
        metadata: {
            namespace: namespace.metadata.name,
            labels
        },
        spec: {
            type: "LoadBalancer",
            ports: [{ port: 80, targetPort: 80 }],
            selector: labels
        }
    }, { provider });
    
    return {
        namespace: namespace.metadata.name,
        service: service.status.loadBalancer.ingress[0].ip
    };
}

// Deploy to multiple clouds
const awsDeployment = deployApp("multi-cloud-app", awsProvider, 3);
const gkeDeployment = deployApp("multi-cloud-app", gkeProvider, 3);
const aksDeployment = deployApp("multi-cloud-app", azureProvider, 3);

// Export the endpoints
export const awsEndpoint = awsDeployment.service;
export const gkeEndpoint = gkeDeployment.service;
export const aksEndpoint = aksDeployment.service;
```

## GitOps with Pulumi

Implementing GitOps workflows with Pulumi:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";

// Create the ArgoCD namespace
const namespace = new k8s.core.v1.Namespace("argocd", {
    metadata: {
        name: "argocd"
    }
});

// Deploy ArgoCD using Helm
const argocd = new k8s.helm.v3.Chart("argocd", {
    chart: "argo-cd",
    version: "4.2.2",
    fetchOpts: {
        repo: "https://argoproj.github.io/argo-helm"
    },
    namespace: namespace.metadata.name,
    values: {
        server: {
            extraArgs: ["--insecure"],
            service: {
                type: "LoadBalancer"
            }
        },
        configs: {
            repositories: {
                "app-repo": {
                    url: "https://github.com/example/app-manifests.git",
                    type: "git"
                }
            }
        }
    }
});

// Create an Argo CD Application CR
const application = new k8s.apiextensions.CustomResource("example-app", {
    apiVersion: "argoproj.io/v1alpha1",
    kind: "Application",
    metadata: {
        name: "example-app",
        namespace: namespace.metadata.name
    },
    spec: {
        project: "default",
        source: {
            repoURL: "https://github.com/example/app-manifests.git",
            targetRevision: "HEAD",
            path: "overlays/production"
        },
        destination: {
            server: "https://kubernetes.default.svc",
            namespace: "example"
        },
        syncPolicy: {
            automated: {
                prune: true,
                selfHeal: true
            },
            syncOptions: ["CreateNamespace=true"]
        }
    }
}, { dependsOn: argocd });

// Output the ArgoCD URL
export const argoCDUrl = argocd.getResourceProperty("v1/Service", "argocd/argocd-server", "status").apply(
    status => `https://${status.loadBalancer.ingress[0].ip}`
);
```

## CI/CD Integration

Integrating Pulumi into CI/CD pipelines using GitHub Actions:

```yaml
# .github/workflows/pulumi.yml
name: Pulumi
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  preview:
    name: Preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - uses: actions/setup-node@v1
        with:
          node-version: 14.x
          
      - name: Install dependencies
        run: npm install
        
      - uses: pulumi/actions@v3
        with:
          command: preview
          stack-name: dev
          comment-on-pr: true
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
          
  update:
    name: Update
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v2
      
      - uses: actions/setup-node@v1
        with:
          node-version: 14.x
          
      - name: Install dependencies
        run: npm install
        
      - uses: pulumi/actions@v3
        with:
          command: up
          stack-name: dev
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

## Testing Pulumi Programs

### Unit Testing with Mocks

```typescript
// __tests__/infrastructure.test.ts
import * as pulumi from "@pulumi/pulumi";
import { WebAppComponent } from "../components/webapp";

// Create a mock for testing
const mocks = {
    newResource: (args: pulumi.runtime.MockResourceArgs) => {
        return {
            id: `${args.name}-id`,
            state: args.inputs,
        };
    },
    call: (args: pulumi.runtime.MockCallArgs) => {
        return args.inputs;
    },
};

// Test the WebApp component
pulumi.runtime.setMocks(mocks);

describe("WebApp Component", function() {
    let webapp: WebAppComponent;
    
    beforeAll(async function() {
        // Create an instance of the component
        webapp = new WebAppComponent("test", {
            namespace: "test-ns",
            image: "nginx:latest",
            port: 80,
            replicas: 2
        });
    });
    
    test("creates a deployment", function(done) {
        pulumi.all([webapp.deployment.spec]).apply(([spec]) => {
            expect(spec.replicas).toBe(2);
            expect(spec.template.spec.containers[0].image).toBe("nginx:latest");
            done();
        });
    });
    
    test("creates a service", function(done) {
        pulumi.all([webapp.service.spec]).apply(([spec]) => {
            expect(spec.ports[0].port).toBe(80);
            expect(spec.type).toBe("ClusterIP");
            done();
        });
    });
});
```

## Best Practices

### Organization

1. **Use a consistent project structure**
   ```
   project/
     ├── index.ts                 # Main entry point
     ├── cluster.ts               # Cluster definition
     ├── components/              # Reusable components
     │   ├── webapp.ts
     │   └── database.ts
     ├── applications/            # Application definitions
     │   ├── frontend.ts
     │   └── backend.ts
     ├── __tests__/               # Unit tests
     └── Pulumi.yaml              # Project config
   ```

2. **Use stack configuration for environment-specific values**
   ```typescript
   const config = new pulumi.Config();
   const replicas = config.getNumber("replicas") || 2;
   const environment = pulumi.getStack();
   ```

3. **Create reusable component resources**
   - Encapsulate related resources
   - Standardize deployments
   - Enable composition

4. **Store sensitive data securely**
   ```bash
   # Set a secret in the configuration
   pulumi config set --secret dbPassword supersecretpassword
   ```

   ```typescript
   // Read the secret in code
   const config = new pulumi.Config();
   const dbPassword = config.requireSecret("dbPassword");
   ```

### Performance

1. **Use parallelism when deploying many resources**
   ```
   pulumi up --parallel 10
   ```

2. **Consider resource dependencies**
   - Explicitly define dependencies with `dependsOn`
   - Use output properties to create implicit dependencies

3. **Optimize Helm chart usage**
   - Use specific versions
   - Cache charts locally when possible
   - Only set necessary values

### Security

1. **Follow least privilege principle**
   - Use IAM roles with minimal permissions
   - Set up RBAC correctly for Kubernetes

2. **Secure sensitive configuration**
   - Use Pulumi secrets for passwords and tokens
   - Leverage cloud provider secret stores

3. **Implement resource policies**
   ```typescript
   // Define a policy that requires all pods to have resource limits
   new PolicyPack("kubernetes", {
       policies: [{
           name: "resource-limits-required",
           description: "Requires all pods to have CPU and memory limits",
           enforcementLevel: "mandatory",
           validateResource: (args) => {
               if (args.type === "kubernetes:core/v1:Pod") {
                   const pod = args.resource;
                   for (const container of pod.spec.containers) {
                       if (!container.resources || !container.resources.limits) {
                           return {
                               diagnostics: [{
                                   message: "Container must have resource limits"
                               }]
                           };
                       }
                   }
               }
               return { diagnostics: [] };
           }
       }]
   });
   ```

## Troubleshooting Common Issues

### State Management

1. **State conflicts**
   - Use `pulumi refresh` to reconcile state with reality
   - One stack update at a time to avoid conflicts

2. **Resource not found**
   - Check if it exists in the `pulumi stack`
   - Use `pulumi refresh` to update state

3. **Permission errors**
   - Verify cloud provider credentials
   - Check KUBECONFIG permissions

### Deployment Issues

1. **Dependencies not resolving**
   - Explicitly use `dependsOn` for resource dependencies
   - Check for circular dependencies

2. **Provider configuration**
   - Ensure provider configuration is correct
   - Check cluster connectivity with `kubectl`

3. **Resource property errors**
   - Check property names and types
   - Use TypeScript for type checking

```bash
# View the current stack
pulumi stack

# Refresh the stack state
pulumi refresh

# View detailed logs
pulumi up --verbose

# Export the stack to diagnose issues
pulumi stack export > stack.json
```

## Conclusion

Pulumi offers a powerful, flexible approach to managing Kubernetes infrastructure using familiar programming languages. By leveraging general purpose languages, you can build reusable components, implement robust testing, and integrate seamlessly with your existing development workflows.

The ability to manage both cloud infrastructure and Kubernetes resources with the same tool creates a consistent deployment experience from infrastructure provisioning to application deployment, enabling true infrastructure as code across your entire stack.