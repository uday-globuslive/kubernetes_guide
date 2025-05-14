# Helm Deep Dive

Helm is the package manager for Kubernetes, enabling you to define, install, and upgrade complex Kubernetes applications. This guide covers Helm architecture, chart creation, templating, repositories, deployment strategies, and best practices.

## Introduction to Helm

Helm simplifies Kubernetes application deployment by packaging resources into a single unit called a chart. Similar to how apt, yum, or homebrew help manage software on Linux and macOS, Helm manages Kubernetes applications.

### Key Concepts

- **Chart**: A Helm package containing Kubernetes resource definitions
- **Release**: An instance of a chart deployed to a Kubernetes cluster
- **Repository**: A collection of charts available for installation
- **Values**: Configuration parameters to customize chart installation
- **Templates**: YAML files with Go templating used to generate Kubernetes manifests

### Evolution of Helm

- **Helm 2**: Client-server architecture with Tiller running in the cluster
- **Helm 3**: Tiller removed, improved security model, release management per namespace

### When to Use Helm

Helm is ideal for:
- Complex applications with multiple Kubernetes resources
- Shared applications deployed across teams or clusters
- Applications requiring customization for different environments
- Standardizing deployment patterns
- Managing application lifecycle with rollbacks

## Installing Helm

Install Helm 3 on different platforms:

```bash
# macOS with Homebrew
brew install helm

# Windows with Chocolatey
choco install kubernetes-helm

# Linux with snap
sudo snap install helm --classic

# Linux with script
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Verify the installation:

```bash
helm version
```

## Helm Architecture

Helm 3 uses a client-only architecture with three main components:

1. **Helm Client**: CLI tool for chart management
2. **Chart**: Collection of files describing Kubernetes resources
3. **Kubernetes API**: Used directly to deploy resources

### Helm Chart Structure

A typical chart directory structure:

```
mychart/
├── Chart.yaml           # Metadata about the chart
├── values.yaml          # Default configuration values
├── templates/           # Templates that generate Kubernetes manifests
│   ├── _helpers.tpl     # Template helper functions
│   ├── deployment.yaml  # Deployment template
│   ├── service.yaml     # Service template
│   ├── ingress.yaml     # Ingress template
│   └── NOTES.txt        # Usage notes displayed after installation
├── charts/              # Dependencies (subcharts)
├── .helmignore          # Patterns to ignore when packaging
└── README.md            # Documentation
```

### Chart.yaml

The Chart.yaml file contains metadata about a chart:

```yaml
apiVersion: v2                # Helm chart API version
name: nginx                   # Chart name
version: 1.2.3                # Chart version (SemVer 2)
kubeVersion: ">=1.16.0"       # Compatible Kubernetes versions
description: NGINX web server # Chart description
type: application             # application or library
appVersion: "1.20.1"          # App version contained in the chart
dependencies:                 # Chart dependencies
  - name: mariadb
    version: 9.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mariadb.enabled
maintainers:
  - name: John Doe
    email: john@example.com
    url: https://example.com
icon: https://example.com/icon.png
home: https://example.com/project
sources:
  - https://github.com/example/project
annotations:
  category: "Web Server"
```

## Working with Helm Charts

### Finding Charts

Search for available Helm charts:

```bash
# Search Artifact Hub
helm search hub wordpress

# Search repositories you've added
helm search repo stable

# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Installing Charts

Install a chart with default values:

```bash
helm install my-release bitnami/wordpress
```

With custom values:

```bash
# Using a values file
helm install my-release bitnami/wordpress -f values.yaml

# Overriding specific values
helm install my-release bitnami/wordpress --set mariadb.auth.password=password
```

With multiple values files (applied from left to right):

```bash
helm install my-release bitnami/wordpress -f global-values.yaml -f environment-values.yaml -f override-values.yaml
```

### Managing Releases

List all releases:

```bash
helm list
helm list -n custom-namespace
helm list -A  # All namespaces
```

Check release status:

```bash
helm status my-release
```

Upgrade a release:

```bash
helm upgrade my-release bitnami/wordpress -f new-values.yaml
```

Rollback to a previous revision:

```bash
# View revision history
helm history my-release

# Rollback to revision 2
helm rollback my-release 2
```

Uninstall a release:

```bash
helm uninstall my-release
```

## Creating Your Own Helm Charts

### Create a New Chart

```bash
helm create mychart
```

This command creates a chart with a standard directory structure and example templates.

### Chart Values

The values.yaml file defines default configuration values:

```yaml
# Default values for mychart
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""  # Defaults to chart appVersion

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: []
  tls: []

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

### Templating Basics

Templates use Go templating with the sprig library to generate Kubernetes manifests.

#### Variables and Functions

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

#### Flow Control

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

#### Helper Functions

Define reusable functions in _helpers.tpl:

```yaml
{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### Testing Templates

Render templates locally without installing:

```bash
# Render all templates
helm template my-release ./mychart

# Render specific template
helm template my-release ./mychart -s templates/deployment.yaml

# Render with custom values
helm template my-release ./mychart -f values.yaml

# Only output specific resources
helm template my-release ./mychart | grep -A20 "kind: Deployment"
```

Validate generated manifests:

```bash
helm lint ./mychart
```

Test a chart installation locally:

```bash
helm install my-release ./mychart --dry-run
```

### Packaging and Sharing Charts

Package a chart into an archive:

```bash
helm package ./mychart
```

Create a chart repository:

```bash
# Generate an index file
helm repo index --url https://example.com/charts .

# Update the index with a new chart
helm repo index --url https://example.com/charts --merge index.yaml .
```

## Advanced Helm Features

### Subchart Dependencies

Define chart dependencies in Chart.yaml:

```yaml
dependencies:
  - name: mariadb
    version: 9.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mariadb.enabled
    tags:
      - database
  - name: redis
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

Manage dependencies:

```bash
# Update dependencies
helm dependency update ./mychart

# List dependencies
helm dependency list ./mychart

# Rebuild charts/ directory
helm dependency build ./mychart
```

Override subchart values in parent values.yaml:

```yaml
# Parent chart values
mariadb:
  auth:
    rootPassword: "my-root-password"
    database: "my-database"
  primary:
    persistence:
      size: 10Gi
```

### Conditional Template Rendering

Use `if`, `with`, `range` for conditional logic:

```yaml
{{- if .Values.rbac.create }}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "mychart.fullname" . }}
rules:
  {{- toYaml .Values.rbac.rules | nindent 2 }}
{{- end }}
```

Use conditions to skip entire files:

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### Schemas for Values Validation

Define a JSON Schema in values.schema.json:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  }
}
```

Validate values against the schema:

```bash
helm install --dry-run my-release ./mychart
```

### Library Charts

Create reusable functionality with library charts:

```yaml
# Chart.yaml
apiVersion: v2
name: my-library
type: library
version: 0.1.0
```

Use library chart functions:

```yaml
# Application chart's Chart.yaml
dependencies:
  - name: my-library
    version: 0.1.0
    repository: https://example.com/charts

# Application chart's template
{{- include "my-library.labels" . }}
```

### Post-Render Operations

Process rendered manifests with external tools:

```bash
helm install my-release ./mychart --post-renderer ./my-post-renderer.sh
```

Example post-renderer script:

```bash
#!/bin/bash
# my-post-renderer.sh
cat <&0 | kustomize build -f -
```

### Helm Hooks

Control the lifecycle of chart installation with hooks:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-db-init
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: db-init
        image: "{{ .Values.initImage.repository }}:{{ .Values.initImage.tag | default .Chart.AppVersion }}"
        command: ["./init-db.sh"]
        env:
          - name: DB_HOST
            value: {{ .Values.database.host }}
      restartPolicy: Never
  backoffLimit: 0
```

Common hook annotations:
- `helm.sh/hook`: Define when to run (pre-install, post-install, pre-delete, etc.)
- `helm.sh/hook-weight`: Set execution order (lower executes first)
- `helm.sh/hook-delete-policy`: Manage hook resource cleanup

## Deploying Charts in CI/CD Pipelines

### GitLab CI/CD Example

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - deploy

helm-lint:
  stage: lint
  image: alpine/helm:3.7.1
  script:
    - helm lint ./charts/myapp

deploy-dev:
  stage: deploy
  image: alpine/helm:3.7.1
  script:
    - helm upgrade --install my-release ./charts/myapp
      --namespace dev
      --create-namespace
      --set environment=dev
      --set replicaCount=1
      --set image.tag=${CI_COMMIT_SHA}
      --atomic
      --timeout 5m
  environment:
    name: development
  only:
    - develop
```

### GitHub Actions Example

```yaml
# .github/workflows/deploy.yml
name: Deploy Helm Chart

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Helm Lint
        uses: stefanprodan/kube-tools@v1
        with:
          helm: 3.7.1
          command: |
            helm lint ./charts/myapp

  deploy:
    needs: lint
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Helm
        uses: azure/setup-helm@v1
        with:
          version: 'v3.7.1'
      - name: Set up kubeconfig
        uses: azure/k8s-set-context@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      - name: Deploy
        run: |
          helm upgrade --install my-release ./charts/myapp \
            --namespace prod \
            --create-namespace \
            --set environment=prod \
            --set image.tag=${{ github.sha }} \
            --atomic \
            --timeout 5m
```

### ArgoCD Integration

Define an Application in ArgoCD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/myapp.git
    targetRevision: HEAD
    path: charts/myapp
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: latest
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Real-World Examples

### Multi-Environment Setup

Structure for different environments:

```
mychart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
```

Common base configuration in values.yaml:

```yaml
replicaCount: 1
image:
  repository: myapp
  tag: latest
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

Environment-specific overrides:

```yaml
# values-dev.yaml
replicaCount: 1
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
ingress:
  enabled: true
  host: dev.example.com

# values-prod.yaml
replicaCount: 3
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
ingress:
  enabled: true
  host: example.com
  tls:
    enabled: true
```

Deploy to different environments:

```bash
# Development
helm upgrade --install myapp ./mychart -f values-dev.yaml --namespace dev

# Production
helm upgrade --install myapp ./mychart -f values-prod.yaml --namespace prod
```

### Microservices Application

A more complex example for a microservices application:

```
microservices/
├── Chart.yaml
├── values.yaml
├── templates/
├── charts/
│   ├── frontend/
│   ├── api-gateway/
│   ├── auth-service/
│   ├── user-service/
│   ├── product-service/
│   └── common-lib/
```

Parent Chart.yaml:

```yaml
apiVersion: v2
name: microservices
description: A microservice application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: frontend
    version: 0.1.0
    condition: frontend.enabled
  - name: api-gateway
    version: 0.1.0
  - name: auth-service
    version: 0.1.0
  - name: user-service
    version: 0.1.0
  - name: product-service
    version: 0.1.0
  - name: common-lib
    version: 0.1.0
    repository: file://./charts/common-lib
    import-values:
      - defaults
```

Parent values.yaml:

```yaml
global:
  environment: development
  domain: example.com
  images:
    registry: docker.io/mycompany
    tag: latest
    pullPolicy: IfNotPresent
  ingress:
    className: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
  database:
    host: postgres.database.svc.cluster.local
    port: 5432
  redis:
    host: redis-master.cache.svc.cluster.local
    port: 6379

frontend:
  enabled: true
  replicaCount: 2
  service:
    type: ClusterIP
    port: 80

api-gateway:
  replicaCount: 2
  service:
    type: ClusterIP
    port: 8080

auth-service:
  replicaCount: 2
  database:
    name: auth_db
    user: auth_user

user-service:
  replicaCount: 2
  database:
    name: user_db
    user: user_service

product-service:
  replicaCount: 2
  database:
    name: product_db
    user: product_service
```

## Helm Best Practices

### Design Principles

1. **Chart Organization**
   - Keep charts simple and focused
   - Use subcharts for complex applications
   - Create library charts for shared logic

2. **Versioning**
   - Use semantic versioning
   - Keep appVersion and chart version separate
   - Document version changes in CHANGELOG.md

3. **Values**
   - Provide sensible defaults
   - Document each value with comments
   - Group related values together
   - Use a consistent naming convention

### Security Practices

1. **Avoid sensitive data in values.yaml**
   - Use Kubernetes Secrets
   - Pass sensitive values via command line `--set`
   - Use external secret management tools

2. **Set proper RBAC**
   - Include RBAC templates
   - Follow least privilege principle
   - Make RBAC creation optional

3. **Container security**
   - Set security contexts
   - Configure non-root user
   - Add Pod security policies
   - Set resource limits

Example of good security settings:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
```

### Operational Practices

1. **Health Checking**
   - Include readiness/liveness probes
   - Add startup probes for slow-starting applications

2. **Resource Management**
   - Set appropriate resource requests/limits
   - Make them configurable via values
   - Set sensible defaults

3. **Labels and Annotations**
   - Use consistent labeling scheme
   - Include recommended labels
   - Add useful annotations for discovery

4. **Helpers for consistency**
   - Create helper templates for common elements
   - Standardize naming conventions
   - Use consistent indentation

5. **Documentation**
   - Provide comprehensive README.md
   - Include example values
   - Document chart behavior
   - Write useful NOTES.txt

### Templating Best Practices

1. **Indentation**
   - Use functions like `nindent`, `indent`
   - Be consistent with spaces

2. **Logic Organization**
   - Keep logic simple in templates
   - Move complex logic to helper functions
   - Use `define` for reusable blocks

3. **Comments**
   - Add comments to explain complex sections
   - Document helper functions
   - Include examples

4. **Error Handling**
   - Use functions like `required`
   - Provide meaningful error messages
   - Validate inputs

5. **Configuration**
   - Use defaults where possible
   - Provide examples in values.yaml
   - Make values validate with schema

### Testing

1. **Validate templates**
   - Use `helm lint` for basic validation
   - Use `helm template` to render manifests
   - Add tests in `tests/` directory

2. **CI Integration**
   - Run helm lint in CI
   - Test chart installation in CI
   - Validate against schemas

3. **Test Templates**
   ```yaml
   # templates/tests/test-connection.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: "{{ include "mychart.fullname" . }}-test-connection"
     annotations:
       "helm.sh/hook": test
     labels:
       {{- include "mychart.labels" . | nindent 4 }}
   spec:
     containers:
       - name: wget
         image: busybox
         command: ['wget']
         args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
     restartPolicy: Never
   ```

### Performance

1. **Chart Size**
   - Use .helmignore to exclude unnecessary files
   - Keep chart size manageable

2. **Caching**
   - Package charts for faster deployment
   - Use a chart repository for distribution

3. **Rendering Speed**
   - Avoid excessive templating logic
   - Use named templates for large repeated sections

## Troubleshooting Common Issues

### Installation Failures

1. **Template rendering errors**
   ```bash
   # Use --debug to see rendered templates
   helm install --debug --dry-run my-release ./mychart
   ```

2. **Missing values**
   ```bash
   # Check if required values are set
   helm install --dry-run my-release ./mychart --set required.value=foo
   ```

3. **Kubernetes validation errors**
   ```bash
   # Validate against server
   helm install --dry-run my-release ./mychart
   
   # Detailed validation
   helm template my-release ./mychart | kubectl apply --validate=true --dry-run=client -f -
   ```

### Upgrade Issues

1. **Resource conflicts**
   ```bash
   # Force resource update
   helm upgrade --force my-release ./mychart
   ```

2. **Value changes not applying**
   ```bash
   # Check current values
   helm get values my-release
   
   # Reset values
   helm upgrade my-release ./mychart --reset-values --set new.value=true
   ```

3. **Stuck deployments**
   ```bash
   # Increase timeout
   helm upgrade --timeout 10m my-release ./mychart
   
   # Use atomic to rollback on failure
   helm upgrade --atomic my-release ./mychart
   ```

### Debugging Templates

1. **Print debug information**
   ```yaml
   {{ printf "Value: %v" .Values.something | fail }}
   ```

2. **Check object structure**
   ```yaml
   {{ .Values | toYaml | nindent 2 }}
   ```

3. **Extract generated YAML**
   ```bash
   helm template my-release ./mychart > rendered.yaml
   ```

## Conclusion

Helm has become the de facto standard for packaging and deploying applications on Kubernetes. By mastering Helm's templating language, chart organization, and deployment patterns, you can create reusable, maintainable packages for any Kubernetes application.

The best Helm charts strike a balance between flexibility and simplicity, allowing users to customize what they need while providing sensible defaults. Following the practices outlined in this guide will help you develop robust charts that can be used across teams and environments with confidence.