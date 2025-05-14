# Tekton Pipelines

This guide explores Tekton, a powerful and flexible Kubernetes-native CI/CD solution for building, testing, and deploying applications.

## Table of Contents

1. [Introduction to Tekton](#introduction-to-tekton)
2. [Core Concepts](#core-concepts)
3. [Installing Tekton](#installing-tekton)
4. [Building Basic Pipelines](#building-basic-pipelines)
5. [Working with Tasks](#working-with-tasks)
6. [Pipeline Resources](#pipeline-resources)
7. [Triggers and Events](#triggers-and-events)
8. [Security and Authentication](#security-and-authentication)
9. [Integration with Kubernetes](#integration-with-kubernetes)
10. [Advanced Patterns](#advanced-patterns)
11. [Tekton Dashboard](#tekton-dashboard)
12. [Tekton CLI](#tekton-cli)
13. [Best Practices](#best-practices)
14. [Real-World Examples](#real-world-examples)

## Introduction to Tekton

Tekton is a powerful and flexible Kubernetes-native open-source framework for creating continuous integration and delivery (CI/CD) systems. It abstracts the underlying details of CI/CD workflows and offers a standardized set of building blocks that can be combined to create delivery pipelines that run on Kubernetes.

### Key Features

- **Kubernetes Native**: Built specifically for Kubernetes and fully leverages its capabilities
- **Declarative**: Define pipelines using Kubernetes custom resources
- **Serverless**: No central CI/CD server to maintain
- **Reusable Components**: Tasks and pipelines can be shared and reused across teams
- **Cloud Native**: Designed according to cloud-native principles
- **Language Agnostic**: Works with any language, framework, or cloud
- **Extensible**: Plugin system for adding new capabilities

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  Tekton Components                   │    │
│  │                                                      │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │    │
│  │  │ Pipelines  │  │  Triggers  │  │  Chains    │     │    │
│  │  └────────────┘  └────────────┘  └────────────┘     │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │    │
│  │  │  Results   │  │ Dashboard  │  │    Hub     │     │    │
│  │  └────────────┘  └────────────┘  └────────────┘     │    │
│  │                                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────┐  ┌─────────────────────────┐      │
│  │ Pipeline Resources  │  │     TaskRuns/PipelineRuns│      │
│  │ (Tasks, Pipelines)  │  │     (Execution)          │      │
│  └─────────────────────┘  └─────────────────────────┘      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Application Pods                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Tasks

Tasks are the building blocks of Tekton pipelines. A Task defines a series of steps that run sequentially to achieve a specific goal, such as building an application, running tests, or deploying to a Kubernetes cluster.

```yaml
# Example Task
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-app
spec:
  params:
    - name: repository
      type: string
      description: The git repository to clone
    - name: revision
      type: string
      description: The git revision to checkout
      default: main
  steps:
    - name: git-clone
      image: alpine/git:v2.26.2
      script: |
        git clone $(params.repository) /workspace/source
        cd /workspace/source
        git checkout $(params.revision)
    - name: build
      image: node:14
      workingDir: /workspace/source
      script: |
        npm install
        npm run build
```

### TaskRun

A TaskRun is a specific execution of a Task, providing the inputs, parameters, and runtime information to run the Task.

```yaml
# Example TaskRun
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-app-run-1
spec:
  taskRef:
    name: build-app
  params:
    - name: repository
      value: https://github.com/example/app.git
    - name: revision
      value: main
```

### Pipelines

Pipelines define a series of Tasks that are executed to form a complete CI/CD workflow. Tasks in a Pipeline can run sequentially, in parallel, or a combination of both.

```yaml
# Example Pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-test-deploy
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
      default: main
    - name: image-name
      type: string
  tasks:
    - name: build
      taskRef:
        name: build-app
      params:
        - name: repository
          value: $(params.repository)
        - name: revision
          value: $(params.revision)
    - name: test
      runAfter:
        - build
      taskRef:
        name: run-tests
      params:
        - name: repository
          value: $(params.repository)
        - name: revision
          value: $(params.revision)
    - name: build-image
      runAfter:
        - test
      taskRef:
        name: build-docker-image
      params:
        - name: repository
          value: $(params.repository)
        - name: image-name
          value: $(params.image-name)
    - name: deploy
      runAfter:
        - build-image
      taskRef:
        name: deploy-to-kubernetes
      params:
        - name: image-name
          value: $(params.image-name)
```

### PipelineRun

A PipelineRun is a specific execution of a Pipeline, providing the inputs, parameters, and runtime information to run the Pipeline.

```yaml
# Example PipelineRun
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: build-test-deploy-run-1
spec:
  pipelineRef:
    name: build-test-deploy
  params:
    - name: repository
      value: https://github.com/example/app.git
    - name: revision
      value: feature-branch
    - name: image-name
      value: example/app:latest
```

### Workspaces

Workspaces define shared storage volumes that Tasks can use for inputs or outputs. They allow data to be shared between steps of a Task or between different Tasks in a Pipeline.

```yaml
# Example Task with Workspace
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-with-workspace
spec:
  workspaces:
    - name: source
      description: The git source code to build
  steps:
    - name: build
      image: node:14
      workingDir: $(workspaces.source.path)
      script: |
        npm install
        npm run build
```

```yaml
# Example Pipeline with Workspace
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-test-with-workspace
spec:
  workspaces:
    - name: shared-workspace
      description: The workspace for sharing data between tasks
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
    - name: build
      runAfter:
        - fetch-source
      taskRef:
        name: build-with-workspace
      workspaces:
        - name: source
          workspace: shared-workspace
```

## Installing Tekton

### Prerequisites

- A Kubernetes cluster (v1.17 or higher)
- kubectl installed and configured
- Kubernetes cluster admin permissions

### Installing Tekton Pipelines

```bash
# Install Tekton Pipelines
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Verify the installation
kubectl get pods --namespace tekton-pipelines
```

### Installing Tekton Triggers

```bash
# Install Tekton Triggers
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

# Verify the installation
kubectl get pods --namespace tekton-pipelines
```

### Installing Tekton Dashboard

```bash
# Install Tekton Dashboard
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Access the dashboard
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

### Installing Tekton CLI (tkn)

```bash
# For Linux
curl -LO https://github.com/tektoncd/cli/releases/download/v0.28.0/tkn_0.28.0_Linux_x86_64.tar.gz
tar xvzf tkn_0.28.0_Linux_x86_64.tar.gz -C /tmp/
sudo mv /tmp/tkn /usr/local/bin/

# For macOS
brew install tektoncd-cli
```

## Building Basic Pipelines

### Hello World Task

```yaml
# hello-world-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-world
spec:
  steps:
    - name: say-hello
      image: ubuntu
      command:
        - echo
      args:
        - "Hello, World!"
```

```bash
# Apply the Task
kubectl apply -f hello-world-task.yaml

# Run the Task
kubectl create -f - <<EOF
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-world-run-1
spec:
  taskRef:
    name: hello-world
EOF

# Check the status
kubectl get taskrun hello-world-run-1
```

### Simple Build Pipeline

```yaml
# git-clone-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
      default: main
  workspaces:
    - name: output
      description: The git repo will be cloned into this workspace
  steps:
    - name: clone
      image: alpine/git:v2.26.2
      script: |
        git clone $(params.repository) $(workspaces.output.path)
        cd $(workspaces.output.path)
        git checkout $(params.revision)
```

```yaml
# build-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: npm-build
spec:
  workspaces:
    - name: source
      description: The source code to build
  steps:
    - name: build
      image: node:14
      workingDir: $(workspaces.source.path)
      script: |
        npm install
        npm run build
```

```yaml
# build-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-pipeline
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
      default: main
  workspaces:
    - name: shared-workspace
      description: Workspace shared between tasks
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      params:
        - name: repository
          value: $(params.repository)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: shared-workspace
    - name: build
      runAfter:
        - fetch-source
      taskRef:
        name: npm-build
      workspaces:
        - name: source
          workspace: shared-workspace
```

```yaml
# pipeline-run.yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: build-pipeline-run-1
spec:
  pipelineRef:
    name: build-pipeline
  params:
    - name: repository
      value: https://github.com/example/app.git
    - name: revision
      value: main
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: build-pvc
```

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: build-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Working with Tasks

### Using Task Parameters

```yaml
# parametrized-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: parametrized-task
spec:
  params:
    - name: greeting
      type: string
      description: The greeting to display
      default: "Hello"
    - name: name
      type: string
      description: The name to greet
      default: "World"
    - name: sleep-time
      type: string
      description: The time to sleep in seconds
      default: "1"
    - name: flags
      type: array
      description: Flags to pass to the echo command
      default: ["-n"]
  steps:
    - name: greet
      image: ubuntu
      command:
        - echo
      args:
        - $(params.flags)
        - "$(params.greeting), $(params.name)!"
    - name: sleep
      image: ubuntu
      command:
        - sleep
      args:
        - $(params.sleep-time)
```

### Task Results

Tasks can produce results that can be consumed by other tasks in a pipeline.

```yaml
# calculate-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: calculate
spec:
  params:
    - name: a
      type: string
    - name: b
      type: string
  results:
    - name: sum
      description: The sum of a and b
    - name: product
      description: The product of a and b
  steps:
    - name: calculate
      image: ubuntu
      script: |
        #!/usr/bin/env bash
        sum=$(($(params.a) + $(params.b)))
        product=$(($(params.a) * $(params.b)))
        
        echo -n $sum > $(results.sum.path)
        echo -n $product > $(results.product.path)
```

```yaml
# use-results-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: use-results
spec:
  params:
    - name: a
      type: string
      default: "5"
    - name: b
      type: string
      default: "7"
  tasks:
    - name: calculate-values
      taskRef:
        name: calculate
      params:
        - name: a
          value: $(params.a)
        - name: b
          value: $(params.b)
    - name: display-results
      runAfter:
        - calculate-values
      taskRef:
        name: display
      params:
        - name: message
          value: "Sum: $(tasks.calculate-values.results.sum), Product: $(tasks.calculate-values.results.product)"
```

### Using Conditions

You can make task execution conditional based on previous task results.

```yaml
# condition-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: check-status
spec:
  params:
    - name: status-code
      type: string
  results:
    - name: status
      description: Whether the status is success or failure
  steps:
    - name: check
      image: ubuntu
      script: |
        #!/usr/bin/env bash
        if [ $(params.status-code) -eq 200 ]; then
          echo -n "success" > $(results.status.path)
        else
          echo -n "failure" > $(results.status.path)
        fi
```

```yaml
# conditional-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: conditional-pipeline
spec:
  params:
    - name: status-code
      type: string
      default: "200"
  tasks:
    - name: check-status
      taskRef:
        name: check-status
      params:
        - name: status-code
          value: $(params.status-code)
    - name: on-success
      when:
        - input: $(tasks.check-status.results.status)
          operator: in
          values: ["success"]
      taskRef:
        name: echo-success
    - name: on-failure
      when:
        - input: $(tasks.check-status.results.status)
          operator: in
          values: ["failure"]
      taskRef:
        name: echo-failure
```

### Task Retries

You can configure a task to retry if it fails.

```yaml
# retry-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-with-retries
spec:
  tasks:
    - name: sometimes-fails
      retries: 3
      taskRef:
        name: flaky-task
```

### Timeouts

You can set a timeout for a task or pipeline to ensure it doesn't run indefinitely.

```yaml
# timeout-task.yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: task-with-timeout
spec:
  taskRef:
    name: long-running-task
  timeout: "10m"
```

## Pipeline Resources

### Using Workspaces for Storage

```yaml
# workspace-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: workspace-pipeline
spec:
  workspaces:
    - name: shared-data
      description: Workspace for sharing data between tasks
  tasks:
    - name: write-data
      taskRef:
        name: write-to-file
      workspaces:
        - name: output
          workspace: shared-data
    - name: read-data
      runAfter:
        - write-data
      taskRef:
        name: read-from-file
      workspaces:
        - name: input
          workspace: shared-data
```

### Different Workspace Types

```yaml
# Different workspace configurations in PipelineRun

# Using PVC
workspaces:
  - name: shared-data
    persistentVolumeClaim:
      claimName: my-pvc

# Using ConfigMap
workspaces:
  - name: shared-data
    configMap:
      name: my-config

# Using Secret
workspaces:
  - name: shared-data
    secret:
      secretName: my-secret

# Using emptyDir
workspaces:
  - name: shared-data
    emptyDir: {}

# Using volumeClaimTemplate (creates a new PVC for each run)
workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

## Triggers and Events

### TriggerTemplates

TriggerTemplates define what resources to create when a trigger is invoked.

```yaml
# trigger-template.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: build-template
spec:
  params:
    - name: gitRepository
      description: The git repository to clone
    - name: gitRevision
      description: The git revision to check out
    - name: imageTag
      description: The image tag to build
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: build-pipeline-run-
      spec:
        pipelineRef:
          name: build-pipeline
        params:
          - name: repository
            value: $(tt.params.gitRepository)
          - name: revision
            value: $(tt.params.gitRevision)
          - name: image-tag
            value: $(tt.params.imageTag)
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi
```

### TriggerBindings

TriggerBindings extract values from events to pass to TriggerTemplates.

```yaml
# trigger-binding.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
spec:
  params:
    - name: gitRepository
      value: $(body.repository.url)
    - name: gitRevision
      value: $(body.head_commit.id)
    - name: imageTag
      value: $(body.head_commit.id)
```

### EventListeners

EventListeners listen for events and process them according to the specified triggers.

```yaml
# event-listener.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      bindings:
        - ref: github-push-binding
      template:
        ref: build-template
```

### Interceptors

Interceptors can be used to filter and modify events before triggering resources.

```yaml
# github-interceptor.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener-with-interceptor
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: "github"
          params:
            - name: "secretRef"
              value:
                secretName: github-secret
                secretKey: secretToken
            - name: "eventTypes"
              value: ["push"]
      bindings:
        - ref: github-push-binding
      template:
        ref: build-template
```

## Security and Authentication

### Service Accounts

Service accounts define permissions for running Tasks and Pipelines.

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pipeline-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
subjects:
  - kind: ServiceAccount
    name: pipeline-sa
roleRef:
  kind: Role
  name: pipeline-role
  apiGroup: rbac.authorization.k8s.io
```

### Using Service Accounts with Tasks

```yaml
# task-with-sa.yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: task-with-sa
spec:
  taskRef:
    name: deploy-task
  serviceAccountName: pipeline-sa
```

### Secrets for Authentication

```yaml
# docker-registry-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

```yaml
# use-docker-secret.yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-and-push
spec:
  taskRef:
    name: kaniko-build
  serviceAccountName: pipeline-sa
```

```yaml
# service-account-with-secrets.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-sa
secrets:
  - name: docker-registry-secret
```

## Integration with Kubernetes

### Deploying to Kubernetes

```yaml
# deploy-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-to-kubernetes
spec:
  params:
    - name: image
      type: string
    - name: namespace
      type: string
      default: default
  steps:
    - name: deploy
      image: bitnami/kubectl:latest
      script: |
        # Update deployment to use the new image
        cat <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
          namespace: $(params.namespace)
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: my-app
          template:
            metadata:
              labels:
                app: my-app
            spec:
              containers:
              - name: my-app
                image: $(params.image)
                ports:
                - containerPort: 8080
        EOF
        
        # Wait for deployment to be ready
        kubectl rollout status deployment/my-app -n $(params.namespace) --timeout=2m
```

### Using Helm for Deployment

```yaml
# helm-deploy-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: helm-deploy
spec:
  params:
    - name: release-name
      type: string
    - name: chart-name
      type: string
    - name: values-file
      type: string
      default: "values.yaml"
    - name: namespace
      type: string
      default: default
  workspaces:
    - name: source
      description: The workspace containing the Helm chart and values
  steps:
    - name: deploy
      image: alpine/helm:3.8.0
      script: |
        cd $(workspaces.source.path)
        helm upgrade --install $(params.release-name) $(params.chart-name) \
          -f $(params.values-file) \
          -n $(params.namespace) \
          --create-namespace \
          --wait \
          --timeout 2m
```

### Interacting with Kubernetes API

```yaml
# kubernetes-api-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: interact-with-k8s-api
spec:
  steps:
    - name: list-pods
      image: bitnami/kubectl:latest
      script: |
        # List all pods
        kubectl get pods -A
        
        # Get details of a specific deployment
        kubectl describe deployment my-app -n default
```

## Advanced Patterns

### Parallel Execution

```yaml
# parallel-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: parallel-pipeline
spec:
  params:
    - name: repository
      type: string
  workspaces:
    - name: shared-workspace
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: repository
          value: $(params.repository)
      workspaces:
        - name: output
          workspace: shared-workspace
    - name: lint
      runAfter:
        - clone
      taskRef:
        name: lint-code
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: test-unit
      runAfter:
        - clone
      taskRef:
        name: unit-test
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: test-integration
      runAfter:
        - clone
      taskRef:
        name: integration-test
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: build
      runAfter:
        - lint
        - test-unit
        - test-integration
      taskRef:
        name: build-app
      workspaces:
        - name: source
          workspace: shared-workspace
```

### Fan-in/Fan-out Pattern

```yaml
# fan-in-fan-out-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: fan-in-fan-out-pipeline
spec:
  params:
    - name: repository
      type: string
  workspaces:
    - name: shared-workspace
  tasks:
    # Fan-out: One task creates multiple parallel tasks
    - name: prepare
      taskRef:
        name: prepare-tasks
      params:
        - name: repository
          value: $(params.repository)
      workspaces:
        - name: output
          workspace: shared-workspace
      
    # Parallel tasks
    - name: task-1
      runAfter:
        - prepare
      taskRef:
        name: build-component-1
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: task-2
      runAfter:
        - prepare
      taskRef:
        name: build-component-2
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: task-3
      runAfter:
        - prepare
      taskRef:
        name: build-component-3
      workspaces:
        - name: source
          workspace: shared-workspace
    
    # Fan-in: All parallel tasks feed into one final task
    - name: finalize
      runAfter:
        - task-1
        - task-2
        - task-3
      taskRef:
        name: assemble-components
      workspaces:
        - name: source
          workspace: shared-workspace
```

### Custom Task

Custom Tasks allow you to extend Tekton with your own task controllers.

```yaml
# custom-task-example.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-with-custom-task
spec:
  tasks:
    - name: regular-task
      taskRef:
        name: simple-task
    - name: custom-task
      taskRef:
        apiVersion: example.dev/v1alpha1
        kind: Example
        name: my-example-task
```

## Tekton Dashboard

The Tekton Dashboard is a web-based UI for Tekton Pipelines, allowing you to view and manage PipelineRuns, TaskRuns, and their logs.

### Key Features

- View all PipelineRuns and TaskRuns
- Monitor real-time logs
- Visualize pipeline execution
- Manually trigger PipelineRuns
- Import resources from Git repositories

### Installation and Access

As shown earlier in the installation section, you can access the dashboard with:

```bash
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

Then, open a browser and navigate to `http://localhost:9097`.

## Tekton CLI

The Tekton CLI (`tkn`) provides command-line access to Tekton resources.

### Key Commands

```bash
# List all pipelines
tkn pipeline list

# Describe a pipeline
tkn pipeline describe build-pipeline

# List all pipelineruns
tkn pipelinerun list

# Get logs from a pipelinerun
tkn pipelinerun logs build-pipeline-run-1 -f

# List all tasks
tkn task list

# Describe a task
tkn task describe build-app

# Delete a pipelinerun
tkn pipelinerun delete build-pipeline-run-1

# Start a pipeline
tkn pipeline start build-pipeline \
    -p repository=https://github.com/example/app.git \
    -p revision=main \
    -w name=shared-workspace,claimName=build-pvc
```

## Best Practices

### Resource Management

1. **Right-size containers**: Specify resource requests and limits for your Task steps.

```yaml
steps:
  - name: build
    image: golang:1.17
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
```

2. **Clean up resources**: Use Finally tasks to clean up resources after a Pipeline run.

```yaml
finally:
  - name: cleanup
    taskRef:
      name: cleanup-task
```

### Reusability

1. **Use Task parameters**: Make your Tasks configurable with parameters.

2. **Create reusable Task libraries**: Build common Tasks that can be shared across teams.

3. **Use Tekton Hub**: Leverage the community-maintained Tasks in Tekton Hub.

```bash
# Install Task from Tekton Hub
tkn hub install task git-clone
```

### Pipeline Design

1. **Keep Tasks focused**: Each Task should do one thing well.

2. **Use clear naming conventions**: Name your Tasks and Pipelines descriptively.

3. **Use workspaces for sharing data**: Avoid emptyDir for critical data.

4. **Structure pipelines logically**: Group related tasks and structure your pipeline for parallelism where possible.

5. **Implement proper error handling**: Use `finally` blocks and conditions to handle failures gracefully.

### Security

1. **Use dedicated service accounts**: Create specific service accounts with minimal permissions.

2. **Manage secrets properly**: Use Kubernetes secrets for sensitive information.

3. **Scan container images**: Include security scanning in your pipelines.

4. **Review pipeline permissions**: Regularly audit your pipeline's RBAC permissions.

## Real-World Examples

### Complete CI/CD Pipeline

```yaml
# ci-cd-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ci-cd-pipeline
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
      default: main
    - name: image-registry
      type: string
    - name: image-name
      type: string
    - name: image-tag
      type: string
      default: latest
  workspaces:
    - name: shared-workspace
    - name: docker-config
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      params:
        - name: repository
          value: $(params.repository)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: shared-workspace
    
    - name: run-tests
      runAfter:
        - fetch-source
      taskRef:
        name: run-tests
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: security-scan
      runAfter:
        - fetch-source
      taskRef:
        name: security-scan
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: build-image
      runAfter:
        - run-tests
        - security-scan
      taskRef:
        name: kaniko-build
      params:
        - name: IMAGE
          value: $(params.image-registry)/$(params.image-name):$(params.image-tag)
        - name: CONTEXT
          value: $(workspaces.source.path)
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: docker-config
          workspace: docker-config
    
    - name: deploy-dev
      runAfter:
        - build-image
      taskRef:
        name: helm-deploy
      params:
        - name: release-name
          value: $(params.image-name)
        - name: chart-path
          value: $(workspaces.source.path)/deploy/helm
        - name: namespace
          value: development
        - name: values-file
          value: values-dev.yaml
        - name: set-values
          value: image.repository=$(params.image-registry)/$(params.image-name),image.tag=$(params.image-tag)
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: integration-tests
      runAfter:
        - deploy-dev
      taskRef:
        name: run-integration-tests
      params:
        - name: namespace
          value: development
        - name: app-url
          value: http://$(params.image-name).development.svc.cluster.local
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: deploy-prod
      runAfter:
        - integration-tests
      taskRef:
        name: helm-deploy
      params:
        - name: release-name
          value: $(params.image-name)
        - name: chart-path
          value: $(workspaces.source.path)/deploy/helm
        - name: namespace
          value: production
        - name: values-file
          value: values-prod.yaml
        - name: set-values
          value: image.repository=$(params.image-registry)/$(params.image-name),image.tag=$(params.image-tag)
      workspaces:
        - name: source
          workspace: shared-workspace
  
  finally:
    - name: notify
      taskRef:
        name: send-notification
      params:
        - name: message
          value: "Pipeline for $(params.repository) completed"
```

### Microservices Build and Deploy

```yaml
# microservices-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: microservices-pipeline
spec:
  params:
    - name: repository
      type: string
    - name: revision
      type: string
    - name: image-registry
      type: string
    - name: image-tag
      type: string
  workspaces:
    - name: shared-workspace
    - name: docker-config
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      params:
        - name: repository
          value: $(params.repository)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: shared-workspace
    
    # Build and deploy service A
    - name: build-service-a
      runAfter:
        - fetch-source
      taskRef:
        name: kaniko-build
      params:
        - name: IMAGE
          value: $(params.image-registry)/service-a:$(params.image-tag)
        - name: CONTEXT
          value: $(workspaces.source.path)/service-a
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: docker-config
          workspace: docker-config
    
    - name: deploy-service-a
      runAfter:
        - build-service-a
      taskRef:
        name: k8s-deploy
      params:
        - name: namespace
          value: default
        - name: deployment
          value: service-a
        - name: image
          value: $(params.image-registry)/service-a:$(params.image-tag)
    
    # Build and deploy service B
    - name: build-service-b
      runAfter:
        - fetch-source
      taskRef:
        name: kaniko-build
      params:
        - name: IMAGE
          value: $(params.image-registry)/service-b:$(params.image-tag)
        - name: CONTEXT
          value: $(workspaces.source.path)/service-b
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: docker-config
          workspace: docker-config
    
    - name: deploy-service-b
      runAfter:
        - build-service-b
      taskRef:
        name: k8s-deploy
      params:
        - name: namespace
          value: default
        - name: deployment
          value: service-b
        - name: image
          value: $(params.image-registry)/service-b:$(params.image-tag)
    
    # Run integration tests after both services are deployed
    - name: integration-tests
      runAfter:
        - deploy-service-a
        - deploy-service-b
      taskRef:
        name: run-integration-tests
      workspaces:
        - name: source
          workspace: shared-workspace
```

## Summary

Tekton provides a powerful, flexible, and Kubernetes-native framework for building CI/CD systems. By leveraging Tekton's components like Tasks, Pipelines, Triggers, and more, you can create robust, reusable, and scalable CI/CD workflows that are seamlessly integrated with your Kubernetes environment.

Key benefits of Tekton include:

1. **Kubernetes-native**: Fully integrated with Kubernetes
2. **Declarative**: Configuration as code using YAML
3. **Modular**: Reusable components
4. **Serverless**: No central server to manage
5. **Extensible**: Support for custom tasks and extensions
6. **Community-supported**: Growing ecosystem and active community

Whether you're building a simple deployment pipeline or a complex CI/CD system for microservices, Tekton provides the tools and flexibility to meet your needs.