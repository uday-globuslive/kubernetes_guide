# ArgoCD: GitOps Continuous Delivery for Kubernetes

## Introduction to ArgoCD

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes that automates the deployment of applications to target environments. It serves as a Kubernetes controller that continuously monitors running applications and compares their live state against the desired state specified in a Git repository. When it detects a deviation, ArgoCD can automatically or manually sync the application back to its desired state.

As a core component of the GitOps methodology, ArgoCD enables:

- **Declarative deployment**: All application configuration is defined declaratively in Git
- **Version control**: All changes are tracked, auditable, and reversible
- **Automated synchronization**: Continuous deployment from Git to Kubernetes
- **Self-healing**: Automatic correction of configuration drift
- **Multi-cluster management**: Deployment to multiple environments from a single source

## ArgoCD Architecture

ArgoCD follows a client-server architecture composed of several key components:

```
┌─────────────────────────────────────────────────────────┐
│                    ArgoCD Server                         │
│                                                         │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │ API Server   │  │ Repository│  │ Application      │  │
│  │ (gRPC/REST)  │  │ Server    │  │ Controller       │  │
│  └──────┬───────┘  └─────┬─────┘  └──────────┬───────┘  │
│         │                │                   │          │
│         ▼                ▼                   ▼          │
│  ┌──────────────────────────────────────────────────┐   │
│  │                  Redis Cache                      │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │                │                   │
         ▼                ▼                   ▼
┌─────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Git         │  │ Kubernetes API  │  │ Kubernetes      │
│ Repository  │  │ Server          │  │ Cluster         │
└─────────────┘  └─────────────────┘  └─────────────────┘
```

### Core Components

- **API Server**: Exposes the ArgoCD API for the UI, CLI, and other clients
- **Repository Server**: Maintains a local cache of Git repositories and generates Kubernetes manifests
- **Application Controller**: Monitors applications and compares their current state with the desired state
- **Redis**: Serves as a cache and notification bus for the other components

### Additional Components

- **Dex**: Optional OpenID Connect (OIDC) provider for SSO integration
- **ApplicationSet Controller**: Manages the creation, management, and deletion of Argo CD Applications across multiple clusters

## Installing ArgoCD

### Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured to access your cluster
- Helm (optional, for Helm-based installation)

### Installation Methods

#### Using the Official Manifests

```bash
# Create namespace
kubectl create namespace argocd

# Apply ArgoCD installation manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Using Helm

```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

### Post-Installation Setup

#### Accessing the ArgoCD API Server

```bash
# Port-forward the ArgoCD API server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

#### Getting the Initial Admin Password

```bash
# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### Installing the ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

## ArgoCD Core Concepts

### Applications

An Application in ArgoCD represents a deployed application instance in a target Kubernetes cluster. It defines the source of the application's configuration (e.g., a Git repository) and the destination (e.g., a Kubernetes cluster and namespace).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticsearch
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/elk-stack.git
    targetRevision: HEAD
    path: elasticsearch
  destination:
    server: https://kubernetes.default.svc
    namespace: elastic-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Projects

Projects provide a logical grouping of applications and define constraints and permissions for those applications, enabling multi-tenancy within ArgoCD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: elk-stack
  namespace: argocd
spec:
  description: ELK Stack Project
  sourceRepos:
  - 'https://github.com/example/elk-stack.git'
  destinations:
  - namespace: 'elastic-*'
    server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
  roles:
  - name: developer
    description: Developer role
    policies:
    - p, proj:elk-stack:developer, applications, get, elk-stack/*, allow
    - p, proj:elk-stack:developer, applications, sync, elk-stack/*, allow
```

### Repositories

Repositories are the sources from which ArgoCD retrieves application manifests. ArgoCD supports various repository types:

- Git repositories (HTTPS or SSH)
- Helm repositories
- OCI registries with Helm charts

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/example/elk-stack.git
  username: git-user
  password: git-password
```

### Sync Options and Strategies

ArgoCD provides various options for how applications are synchronized:

- **Manual sync**: Changes are applied only when triggered by a user
- **Automated sync**: Changes are applied automatically when detected
- **Prune resources**: Remove resources that no longer exist in Git
- **Self-heal**: Automatically correct drift even if caused by direct changes to the cluster

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
```

## Working with ArgoCD

### Creating Applications

#### Using the CLI

```bash
# Login to ArgoCD
argocd login localhost:8080

# Create an application
argocd app create elasticsearch \
  --repo https://github.com/example/elk-stack.git \
  --path elasticsearch \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace elastic-system \
  --sync-policy automated
```

#### Using the Web UI

1. Navigate to the ArgoCD UI
2. Click "New App"
3. Fill in the application details:
   - Application Name: `elasticsearch`
   - Project: `default`
   - Repository URL: `https://github.com/example/elk-stack.git`
   - Path: `elasticsearch`
   - Destination: `https://kubernetes.default.svc`
   - Namespace: `elastic-system`
4. Configure sync policy options as needed
5. Click "Create"

#### Using Kubernetes Manifests

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kibana
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/elk-stack.git
    targetRevision: HEAD
    path: kibana
  destination:
    server: https://kubernetes.default.svc
    namespace: elastic-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply with kubectl:

```bash
kubectl apply -f kibana-application.yaml
```

### Managing Application Lifecycle

#### Syncing Applications

```bash
# Sync an application
argocd app sync elasticsearch

# Sync with specific resources
argocd app sync elasticsearch --resource apps:Deployment:elasticsearch

# Dry-run to see what would be applied
argocd app sync elasticsearch --dry-run
```

#### Viewing Application Status

```bash
# Get application details
argocd app get elasticsearch

# Get application history
argocd app history elasticsearch

# View specific revision details
argocd app get elasticsearch --revision 1
```

#### Rollback to Previous Versions

```bash
# List application history
argocd app history elasticsearch

# Rollback to a specific version
argocd app rollback elasticsearch 2
```

### Health Assessment

ArgoCD continuously monitors the health of deployed applications:

- **Healthy**: Application is fully synced and all resources are healthy
- **Degraded**: Application has issues (e.g., pods not ready)
- **Progressing**: Application is in the process of becoming healthy
- **Suspended**: Application deployment is paused
- **Missing**: Resources are missing from the cluster

```bash
# Check application health
argocd app get elasticsearch --output health
```

## Configuration Management Tools Integration

ArgoCD supports various configuration management tools:

### Helm

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticsearch-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm.elastic.co
    chart: elasticsearch
    targetRevision: 7.15.0
    helm:
      values: |
        replicas: 3
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: elastic-system
```

### Kustomize

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elk-stack-kustomize
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/elk-stack.git
    targetRevision: HEAD
    path: overlay/production
    kustomize:
      namePrefix: prod-
      images:
      - docker.elastic.co/elasticsearch/elasticsearch:7.15.0
  destination:
    server: https://kubernetes.default.svc
    namespace: elastic-system
```

### JsonNet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticsearch-jsonnet
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/elk-stack.git
    targetRevision: HEAD
    path: jsonnet
    directory:
      jsonnet:
        libs:
        - vendor/elasticsearch
        extVars:
        - name: replicas
          value: "3"
  destination:
    server: https://kubernetes.default.svc
    namespace: elastic-system
```

## Advanced Features

### ApplicationSets

ApplicationSets enable automated creation of ArgoCD Applications across multiple clusters or from template patterns.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: elasticsearch-clusters
  namespace: argocd
spec:
  generators:
  - clusters: {}  # Use all connected clusters
  template:
    metadata:
      name: 'elasticsearch-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/elk-stack.git
        targetRevision: HEAD
        path: elasticsearch
      destination:
        server: '{{server}}'
        namespace: elastic-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Types of generators:
- **List**: Static list of parameters
- **Cluster**: Target multiple clusters
- **Git**: Generate from directories or files in a Git repository
- **Matrix**: Combine multiple generators
- **SCM Provider**: Generate from GitHub, GitLab, or Bitbucket repositories

### Resource Hooks

Resource hooks allow executing specific actions during the sync process:

- **PreSync**: Execute before the sync
- **Sync**: Execute during the sync
- **PostSync**: Execute after the sync
- **SyncFail**: Execute when the sync fails

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: elasticsearch-index-backup
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: backup
        image: curl:latest
        command: ["curl", "-X", "POST", "http://elasticsearch:9200/_snapshot/my_backup/snapshot_1"]
      restartPolicy: Never
  backoffLimit: 1
```

Hook policies:
- **HookSucceeded**: Delete after successful execution
- **HookFailed**: Delete after failed execution
- **BeforeHookCreation**: Delete before a new hook is created

### Secrets Management

ArgoCD provides several approaches for managing secrets:

#### Bitnami Sealed Secrets

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: elasticsearch-credentials
  namespace: elastic-system
spec:
  encryptedData:
    username: AgBy8hCe2...truncated
    password: AgBy8hCe2...truncated
```

#### AWS KMS / Vault Plugin

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticsearch-with-secrets
spec:
  # ... other fields
  source:
    plugin:
      name: argocd-vault-plugin
      env:
      - name: AVP_TYPE
        value: vault
      - name: VAULT_ADDR
        value: http://vault.vault.svc:8200
      - name: AVP_SECRET_PATH
        value: secret/data/elasticsearch
```

#### External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: elasticsearch-credentials
  namespace: elastic-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: elasticsearch-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: secret/data/elasticsearch
      property: username
  - secretKey: password
    remoteRef:
      key: secret/data/elasticsearch
      property: password
```

### Multi-Tenancy

ArgoCD supports multi-tenancy through Projects, which provide isolation and access controls:

1. **Namespace Isolation**: Restrict applications to specific namespaces
2. **Source Repo Restriction**: Limit which repositories can be used
3. **Role-Based Access Control**: Define who can create/sync applications
4. **Resource Whitelisting/Blacklisting**: Control which Kubernetes resources can be deployed

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-logging
  namespace: argocd
spec:
  description: Logging platform team
  sourceRepos:
  - 'https://github.com/logging-team/*'
  destinations:
  - namespace: 'logging-*'
    server: 'https://kubernetes.default.svc'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
  clusterResourceWhitelist:
  - group: 'rbac.authorization.k8s.io'
    kind: 'ClusterRole'
  - group: 'rbac.authorization.k8s.io'
    kind: 'ClusterRoleBinding'
  roles:
  - name: admin
    description: Admin role
    policies:
    - p, proj:team-logging:admin, applications, *, team-logging/*, allow
    groups:
    - logging-admins
  - name: developer
    description: Developer role
    policies:
    - p, proj:team-logging:developer, applications, get, team-logging/*, allow
    - p, proj:team-logging:developer, applications, sync, team-logging/*, allow
    groups:
    - logging-developers
```

## Authentication and Authorization

### SSO Integration

ArgoCD can integrate with various SSO providers through Dex or direct OIDC:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: aabbccddeeff00112233
          clientSecret: $dex.github.clientSecret
          orgs:
          - name: my-github-org
```

### RBAC Configuration

ArgoCD uses a custom RBAC model with policies:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Built-in policy that grants admin-level access to the 'admins' group
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    p, role:admin, logs, *, *, allow
    
    # DevOps team can sync any app
    p, role:devops, applications, sync, */*, allow
    
    # Developers can only view apps
    p, role:developer, applications, get, */*, allow
    
    # Group membership
    g, admin-user, role:admin
    g, devops-user, role:devops
    g, dev-user, role:developer
```

### API Tokens

ArgoCD supports API tokens for automation:

```bash
# Generate a token for automation
argocd account generate-token --account pipeline-automation --expires-in 0
```

## Monitoring and Observability

### Metrics

ArgoCD exposes Prometheus metrics at `/metrics` on port 8082:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
```

Common metrics include:
- `argocd_app_info`: Application information
- `argocd_app_sync_status`: Sync status by application
- `argocd_app_health_status`: Health status by application
- `argocd_cluster_info`: Managed cluster information
- `argocd_app_reconcile`: Reconciliation performance metrics

### Logging

ArgoCD uses structured JSON logging that can be integrated with logging platforms:

```bash
# Get controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f

# Get API server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f

# Get repo server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -f
```

### Notifications

ArgoCD can send notifications about application events:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticsearch
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: elk-alerts
    notifications.argoproj.io/subscribe.on-sync-failed.slack: elk-alerts
spec:
  # ... application spec
```

Configuration for notification services:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    username: ArgoCD Notifications
  template.app-sync-status: |
    message: |
      Application {{.app.metadata.name}} sync status: {{.app.status.sync.status}}
      Health status: {{.app.status.health.status}}
  trigger.on-sync-succeeded: |
    - when: app.status.sync.status == 'Synced'
      send: [app-sync-status]
  trigger.on-sync-failed: |
    - when: app.status.sync.status == 'OutOfSync'
      send: [app-sync-status]
```

## Performance Tuning

### Scaling ArgoCD

For larger deployments, consider:

1. **Increase controller resources**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-application-controller
spec:
  template:
    spec:
      containers:
      - name: argocd-application-controller
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
```

2. **Adjust controller settings**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  controller.repo.server.timeout.seconds: "300"
  controller.status.processors: "20"
  controller.operation.processors: "10"
```

3. **Shard the controller** (for very large installations):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  application.namespaces: ns1,ns2,ns3
```

Then deploy multiple controller instances with different namespace assignments.

### Repository Server Optimization

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  reposerver.parallelism.limit: "10"
  reposerver.timeout.seconds: "300"
```

### Redis Scaling

For large installations, consider scaling Redis:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-redis
spec:
  template:
    spec:
      containers:
      - name: redis
        resources:
          requests:
            memory: 512Mi
            cpu: 250m
          limits:
            memory: 1Gi
            cpu: 500m
```

## Real-World ArgoCD Implementation for ELK Stack

### Complete ELK Stack with ArgoCD

Let's implement a complete ELK Stack (Elasticsearch, Logstash, Kibana) managed by ArgoCD:

#### 1. Repository Structure

```
elk-stack-gitops/
├── base/
│   ├── elasticsearch/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── logstash/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── kibana/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── patches/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    └── production/
        ├── kustomization.yaml
        └── patches/
```

#### 2. ArgoCD Projects and Applications

Create an ArgoCD project for the ELK Stack:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: elk-stack
  namespace: argocd
spec:
  description: ELK Stack Logging Platform
  sourceRepos:
  - 'https://github.com/example/elk-stack-gitops.git'
  destinations:
  - namespace: 'logging-*'
    server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
```

Create an ApplicationSet for each environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: elk-environments
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: development
        namespace: logging-dev
      - env: staging
        namespace: logging-staging
      - env: production
        namespace: logging-prod
  template:
    metadata:
      name: 'elk-{{env}}'
      namespace: argocd
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      project: elk-stack
      source:
        repoURL: https://github.com/example/elk-stack-gitops.git
        targetRevision: HEAD
        path: overlays/{{env}}
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

#### 3. Elasticsearch Configuration

Example Elasticsearch StatefulSet for high availability:

```yaml
# base/elasticsearch/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
        env:
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: cluster.name
          value: elasticsearch-cluster
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 30Gi
```

#### 4. Environment-Specific Customizations

Example production environment customization:

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base/elasticsearch
- ../../base/logstash
- ../../base/kibana

namespace: logging-prod

patchesStrategicMerge:
- patches/elasticsearch-resources.yaml
- patches/kibana-resources.yaml

images:
- name: docker.elastic.co/elasticsearch/elasticsearch
  newTag: 7.15.0
- name: docker.elastic.co/logstash/logstash
  newTag: 7.15.0
- name: docker.elastic.co/kibana/kibana
  newTag: 7.15.0
```

```yaml
# overlays/production/patches/elasticsearch-resources.yaml
apiVersion: apps/v1
kind: StatefulSet
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
            cpu: 2000m
            memory: 4Gi
          requests:
            cpu: 1000m
            memory: 2Gi
        env:
        - name: ES_JAVA_OPTS
          value: "-Xms2g -Xmx2g"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 100Gi
```

#### 5. Progressive Delivery with ArgoCD

Configure progressive delivery using a multi-stage approach:

1. **Development**: Automatic syncs with every commit
2. **Staging**: Syncs after development tests pass
3. **Production**: Manual approval with blue/green deployment

```yaml
# Development environment with automated sync
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elk-development
  namespace: argocd
spec:
  project: elk-stack
  source:
    repoURL: https://github.com/example/elk-stack-gitops.git
    targetRevision: HEAD
    path: overlays/development
  destination:
    server: https://kubernetes.default.svc
    namespace: logging-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# Production environment with manual sync
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elk-production
  namespace: argocd
spec:
  project: elk-stack
  source:
    repoURL: https://github.com/example/elk-stack-gitops.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: logging-prod
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    # No automated sync for production
```

### Integration with CI Pipeline

1. **CI Pipeline** (GitHub Actions Example):

```yaml
name: Build and Deploy ELK Stack

on:
  push:
    branches: [ main ]
    paths:
    - 'base/**'
    - 'overlays/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Kustomize
      uses: imranismail/setup-kustomize@v1
    
    - name: Validate Kustomize Builds
      run: |
        for env in development staging production; do
          echo "Validating $env environment"
          kustomize build overlays/$env > /dev/null
        done
        
    - name: Validate Kubernetes Resources
      run: |
        for env in development staging production; do
          echo "Validating $env environment"
          kustomize build overlays/$env | kubectl apply --dry-run=client -f -
        done
  
  deploy-dev:
    needs: validate
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Install ArgoCD CLI
      run: |
        curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
        sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
        rm argocd-linux-amd64
    
    - name: Login to ArgoCD
      run: |
        argocd login ${{ secrets.ARGOCD_SERVER }} --username ${{ secrets.ARGOCD_USERNAME }} --password ${{ secrets.ARGOCD_PASSWORD }} --insecure
    
    - name: Sync Development Application
      run: |
        argocd app sync elk-development --prune
        
    - name: Wait for Development Sync
      run: |
        argocd app wait elk-development --health
```

## Troubleshooting ArgoCD

### Common Issues and Resolutions

#### Application Stuck in "Progressing" State

1. **Check application logs**:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller | grep elasticsearch
```

2. **Check Kubernetes events**:
```bash
kubectl get events -n logging-prod
```

3. **Force application refresh**:
```bash
argocd app get elasticsearch --refresh
```

#### Authentication Issues

1. **Reset admin password**:
```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$mivhwttXM0U5eBrZGtAG8.VSRL1l9cZNAmaSaqotIzXRBRwID1NT.",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```
This resets the password to "admin"

2. **Check RBAC configuration**:
```bash
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

#### Repository Connection Issues

1. **Verify repository credentials**:
```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository
```

2. **Check repository server logs**:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -f
```

3. **Test repository connection**:
```bash
argocd repo list
```

### Diagnostic Procedures

#### Health Check

```bash
# Check ArgoCD component health
kubectl get pods -n argocd

# Check ArgoCD service status
kubectl get svc -n argocd

# Check persistent volumes
kubectl get pvc -n argocd
```

#### Database Backup and Restore

ArgoCD stores its data in Redis. To backup:

```bash
# Port-forward to Redis
kubectl port-forward svc/argocd-redis 6379:6379 -n argocd

# Backup Redis data
redis-cli -p 6379 --rdb argocd-backup.rdb
```

To restore:

```bash
# Copy backup to Redis pod
kubectl cp argocd-backup.rdb argocd/argocd-redis-5b6967fdfc-abcde:/data/

# Restart Redis pod to load data
kubectl rollout restart deployment argocd-redis -n argocd
```

## Best Practices and Recommendations

### Project Structure

1. **Organize by application**: Keep related components together
2. **Use a consistent layout**: Maintain consistent directory structures
3. **Separate application logic from configuration**: Use base/overlay pattern
4. **Document configuration options**: Add README files in key directories

### Security Considerations

1. **Use Projects for isolation**: Restrict access to specific repositories and namespaces
2. **Implement RBAC**: Use fine-grained access controls
3. **Secure secrets**: Never store plain-text secrets in Git
4. **Enable SSO**: Integrate with your organization's identity provider
5. **Run security scans**: Validate manifests for security issues

### Performance Optimization

1. **Right-size controller resources**: Adjust based on number of applications
2. **Optimize repository polling**: Use webhooks where possible
3. **Implement caching**: Use Git repository caching
4. **Tune reconciliation periods**: Adjust based on your update frequency

### Operational Recommendations

1. **Use Infrastructure as Code** for ArgoCD itself
2. **Backup ArgoCD regularly**
3. **Monitor ArgoCD components**
4. **Implement progressive delivery** across environments
5. **Maintain clear documentation** of workflows and procedures

## Conclusion

ArgoCD has established itself as a leading GitOps continuous delivery tool for Kubernetes, providing a robust, declarative approach to application deployment and management. By leveraging ArgoCD's capabilities, organizations can implement consistent, scalable, and secure deployment workflows for their Kubernetes applications.

For ELK Stack deployments, ArgoCD offers powerful features that enable reliable management of complex distributed systems, from development environments to large-scale production deployments. By adopting GitOps principles through ArgoCD, teams can achieve faster deployments, improved reliability, and better collaboration between development and operations.

As Kubernetes continues to evolve, ArgoCD remains at the forefront of GitOps tooling, continuously adding features and capabilities that align with the broader ecosystem and emerging best practices in cloud-native application delivery.