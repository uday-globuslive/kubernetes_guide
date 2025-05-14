# GitLab CI for Kubernetes

This guide covers how to use GitLab CI/CD to build, test, and deploy applications to Kubernetes clusters, including best practices, common patterns, and real-world examples.

## Table of Contents

1. [Introduction to GitLab CI](#introduction-to-gitlab-ci)
2. [Setting Up GitLab CI for Kubernetes](#setting-up-gitlab-ci-for-kubernetes)
3. [Building Container Images](#building-container-images)
4. [Deploying to Kubernetes](#deploying-to-kubernetes)
5. [Advanced Deployment Strategies](#advanced-deployment-strategies)
6. [Environment Management](#environment-management)
7. [Security Scanning](#security-scanning)
8. [GitLab's Kubernetes Integration](#gitlabs-kubernetes-integration)
9. [Auto DevOps](#auto-devops)
10. [Performance Optimization](#performance-optimization)
11. [Best Practices](#best-practices)
12. [Real-World Examples](#real-world-examples)

## Introduction to GitLab CI

GitLab CI/CD is a powerful continuous integration and delivery platform built into GitLab. It provides a comprehensive solution for automating the building, testing, and deployment of applications to Kubernetes environments.

### Key Concepts

- **Pipelines**: A collection of jobs organized into stages
- **Jobs**: Smallest unit of work, executed by runners
- **Stages**: Groups of jobs that run sequentially
- **Runners**: Agents that execute the jobs
- **.gitlab-ci.yml**: Configuration file that defines the pipeline structure

### Benefits for Kubernetes Deployments

- **Built-in Container Registry**: Easily push and pull container images
- **Kubernetes Integration**: Native Kubernetes integration
- **Environment Management**: Track deployments across environments
- **Auto DevOps**: One-click Kubernetes deployments
- **Review Apps**: Deploy dynamic environments for merge requests
- **Security Scanning**: Built-in vulnerability scanning
- **Scalable Runners**: Scale CI/CD infrastructure as needed

## Setting Up GitLab CI for Kubernetes

### Basic Pipeline Structure

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  DOCKER_HOST: tcp://docker:2375

build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

test:
  stage: test
  image: node:18
  script:
    - npm install
    - npm test
  only:
    - main

deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app -n default
  only:
    - main
```

### Configuring Kubernetes Credentials

In GitLab, there are several ways to configure Kubernetes credentials:

1. **Using Environment Variables**

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app -n default
  only:
    - main
```

2. **Using GitLab's Kubernetes Integration**

Set up a Kubernetes cluster in GitLab's UI (Operations > Kubernetes), then use the provided context:

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app -n default
  only:
    - main
```

3. **Using GitLab Agent for Kubernetes**

For secure connections to Kubernetes clusters, use the GitLab Kubernetes Agent:

```yaml
# .gitlab/agents/my-agent/config.yaml
ci_access:
  projects:
    - id: group/project

# .gitlab-ci.yml
deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - kubectl config use-context ci:${CI_PROJECT_PATH}:my-agent
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app -n default
  only:
    - main
```

## Building Container Images

### Using Docker-in-Docker

```yaml
build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
    DOCKER_HOST: tcp://docker:2375
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

### Using Kaniko

Kaniko allows building container images without the Docker daemon:

```yaml
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.8.0-debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor 
      --context ${CI_PROJECT_DIR} 
      --dockerfile ${CI_PROJECT_DIR}/Dockerfile 
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA 
      --destination $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

### Multi-Architecture Builds

```yaml
build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
    DOCKER_HOST: tcp://docker:2375
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker run --privileged --rm tonistiigi/binfmt --install all
  script:
    - docker buildx create --use --name multi-arch-builder
    - docker buildx build --platform linux/amd64,linux/arm64 
      -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA 
      -t $CI_REGISTRY_IMAGE:latest 
      --push .
  only:
    - main
```

## Deploying to Kubernetes

### Basic Deployment

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update the deployment image
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/deployment.yaml
    # Apply the Kubernetes manifests
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl apply -f kubernetes/service.yaml
    # Wait for deployment to complete
    - kubectl rollout status deployment/my-app -n default
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
```

### Using Helm

```yaml
deploy:
  stage: deploy
  image: alpine/helm:3.9.0
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update values file with current image tag
    - sed -i "s|tag: .*|tag: $CI_COMMIT_SHA|" helm/my-app/values.yaml
    # Deploy with Helm
    - helm upgrade --install my-app ./helm/my-app 
      --namespace default 
      --set image.repository=$CI_REGISTRY_IMAGE 
      --set image.tag=$CI_COMMIT_SHA 
      --atomic 
      --timeout 5m
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
```

### Using Kustomize

```yaml
deploy:
  stage: deploy
  image:
    name: registry.gitlab.com/gitlab-org/cluster-integration/kubectl:1.23.3
    entrypoint: [""]
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update kustomization with current image tag
    - cd kubernetes/overlays/production
    - kustomize edit set image $CI_REGISTRY_IMAGE=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    # Apply with kustomize
    - kubectl apply -k .
    - kubectl rollout status deployment/my-app -n default
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
```

## Advanced Deployment Strategies

### Multi-Stage Deployments

```yaml
stages:
  - build
  - test
  - deploy-staging
  - test-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  DOCKER_HOST: tcp://docker:2375

build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

test:
  stage: test
  image: node:18
  script:
    - npm install
    - npm test
  only:
    - main

deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/staging/deployment.yaml
    - kubectl apply -f kubernetes/staging/
    - kubectl rollout status deployment/my-app -n staging
  environment:
    name: staging
    url: https://staging.myapp.example.com
  only:
    - main

test-staging:
  stage: test-staging
  image: node:18
  script:
    - npm install
    - npm run test:e2e
  variables:
    APP_URL: https://staging.myapp.example.com
  only:
    - main

deploy-production:
  stage: deploy-production
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/production/deployment.yaml
    - kubectl apply -f kubernetes/production/
    - kubectl rollout status deployment/my-app -n production
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main
```

### Canary Deployments

```yaml
deploy-canary:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update the canary deployment image
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/canary/deployment.yaml
    # Apply the canary deployment
    - kubectl apply -f kubernetes/canary/
    - kubectl rollout status deployment/my-app-canary -n default
  environment:
    name: production-canary
    url: https://myapp.example.com
  only:
    - main

test-canary:
  stage: test
  image: alpine:3.16
  script:
    - apk add --no-cache curl
    # Run some tests against the canary deployment
    - |
      for i in {1..10}; do
        status_code=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.example.com/health)
        if [ $status_code -ne 200 ]; then
          echo "Canary deployment failed with status code $status_code"
          exit 1
        fi
        sleep 5
      done
  only:
    - main

promote-to-production:
  stage: deploy-production
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update the production deployment with the canary image
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/production/deployment.yaml
    # Apply the production deployment
    - kubectl apply -f kubernetes/production/
    - kubectl rollout status deployment/my-app -n default
    # Remove canary
    - kubectl delete -f kubernetes/canary/
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main
```

### Blue-Green Deployments

```yaml
deploy-green:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Determine current environment (blue or green)
    - |
      CURRENT_ENV=$(kubectl get service my-app-service -n default -o jsonpath='{.spec.selector.environment}' || echo "blue")
      if [ "$CURRENT_ENV" == "blue" ]; then
        NEW_ENV="green"
      else
        NEW_ENV="blue"
      fi
      echo "Current environment is $CURRENT_ENV, deploying to $NEW_ENV"
    # Deploy new environment
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/deployment.yaml
    - sed -i "s|environment: .*|environment: $NEW_ENV|" kubernetes/deployment.yaml
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app-$NEW_ENV -n default
  environment:
    name: production-new
    url: https://my-app-new.example.com
  only:
    - main

switch-traffic:
  stage: deploy-production
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Determine new environment
    - |
      CURRENT_ENV=$(kubectl get service my-app-service -n default -o jsonpath='{.spec.selector.environment}')
      if [ "$CURRENT_ENV" == "blue" ]; then
        NEW_ENV="green"
      else
        NEW_ENV="blue"
      fi
      echo "Switching traffic from $CURRENT_ENV to $NEW_ENV"
    # Update service to point to new deployment
    - kubectl patch service my-app-service -n default -p "{\"spec\":{\"selector\":{\"environment\":\"$NEW_ENV\"}}}"
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main

cleanup-old-environment:
  stage: cleanup
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Determine old environment
    - |
      CURRENT_ENV=$(kubectl get service my-app-service -n default -o jsonpath='{.spec.selector.environment}')
      if [ "$CURRENT_ENV" == "blue" ]; then
        OLD_ENV="green"
      else
        OLD_ENV="blue"
      fi
      echo "Removing old environment: $OLD_ENV"
    # Delete old deployment
    - kubectl delete deployment my-app-$OLD_ENV -n default
  environment:
    name: production-cleanup
  when: manual
  only:
    - main
```

## Environment Management

### Dynamic Environments

```yaml
stages:
  - build
  - test
  - review
  - staging
  - production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  DOCKER_HOST: tcp://docker:2375

build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - branches

test:
  stage: test
  image: node:18
  script:
    - npm install
    - npm test
  only:
    - branches

review:
  stage: review
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Create namespace if it doesn't exist
    - kubectl create namespace review-$CI_COMMIT_REF_SLUG --dry-run=client -o yaml | kubectl apply -f -
    # Generate deployment from template
    - sed -i "s|__CI_REGISTRY_IMAGE__|$CI_REGISTRY_IMAGE|g" kubernetes/review/deployment.yaml
    - sed -i "s|__CI_COMMIT_SHA__|$CI_COMMIT_SHA|g" kubernetes/review/deployment.yaml
    - sed -i "s|__CI_ENVIRONMENT_SLUG__|$CI_ENVIRONMENT_SLUG|g" kubernetes/review/deployment.yaml
    - sed -i "s|__CI_ENVIRONMENT_HOSTNAME__|$CI_ENVIRONMENT_HOSTNAME|g" kubernetes/review/ingress.yaml
    # Apply Kubernetes manifests
    - kubectl apply -f kubernetes/review/ -n review-$CI_COMMIT_REF_SLUG
    - kubectl rollout status deployment/$CI_ENVIRONMENT_SLUG -n review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop_review
  only:
    - branches
  except:
    - main

stop_review:
  stage: review
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - kubectl delete namespace review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
  only:
    - branches
  except:
    - main
```

### Environment-specific Configuration

```yaml
.deploy_template: &deploy_definition
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Process environment-specific configuration
    - envsubst < kubernetes/overlays/$CI_ENVIRONMENT_NAME/kustomization.template.yaml > kubernetes/overlays/$CI_ENVIRONMENT_NAME/kustomization.yaml
    # Apply with kustomize
    - kubectl apply -k kubernetes/overlays/$CI_ENVIRONMENT_NAME
    - kubectl rollout status deployment/my-app -n $CI_ENVIRONMENT_NAME

deploy_staging:
  stage: staging
  image: bitnami/kubectl:1.23
  <<: *deploy_definition
  variables:
    APP_REPLICAS: 2
    APP_RESOURCES_CPU: "500m"
    APP_RESOURCES_MEMORY: "512Mi"
  environment:
    name: staging
    url: https://staging.myapp.example.com
  only:
    - main

deploy_production:
  stage: production
  image: bitnami/kubectl:1.23
  <<: *deploy_definition
  variables:
    APP_REPLICAS: 5
    APP_RESOURCES_CPU: "1000m"
    APP_RESOURCES_MEMORY: "1Gi"
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main
```

## Security Scanning

### Container Scanning

```yaml
container_scanning:
  stage: test
  image: 
    name: registry.gitlab.com/gitlab-org/security-products/container-scanning:5
    entrypoint: [""]
  variables:
    # Set the container registry image to scan
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    # Enable Docker-in-Docker
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
    DOCKER_HOST: tcp://docker:2375
  services:
    - docker:20.10.12-dind
  script:
    - /analyzer run
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
  only:
    - main
```

### SAST (Static Application Security Testing)

```yaml
sast:
  stage: test
  image: 
    name: registry.gitlab.com/gitlab-org/security-products/sast:latest
    entrypoint: [""]
  variables:
    SAST_DEFAULT_ANALYZERS: "eslint,nodejs-scan"
    SAST_EXCLUDED_PATHS: "node_modules,test"
  script:
    - /analyzer run
  artifacts:
    reports:
      sast: gl-sast-report.json
  only:
    - main
```

### Dependency Scanning

```yaml
dependency_scanning:
  stage: test
  image: 
    name: registry.gitlab.com/gitlab-org/security-products/dependency-scanning:latest
    entrypoint: [""]
  script:
    - /analyzer run
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  only:
    - main
```

### Kubernetes Manifest Scanning

```yaml
kubernetes_scanning:
  stage: test
  image: 
    name: registry.gitlab.com/gitlab-org/security-products/kube-scan:latest
    entrypoint: [""]
  script:
    - /analyzer run
  artifacts:
    reports:
      kubernetes_scanning: gl-kubernetes-scanning-report.json
  only:
    - main
```

## GitLab's Kubernetes Integration

### Connecting a Kubernetes Cluster

In GitLab, you can connect a Kubernetes cluster in two ways:

1. **Using the UI**: Navigate to Operations > Kubernetes, add a cluster, and provide credentials.
2. **Using the GitLab Kubernetes Agent**: A more secure approach for connecting Kubernetes clusters.

### Setting Up the GitLab Kubernetes Agent

1. Create an agent configuration file in your repository:

```yaml
# .gitlab/agents/my-agent/config.yaml
ci_access:
  projects:
    - id: group/project
```

2. Register the agent in GitLab UI (Infrastructure > Kubernetes clusters).

3. Install the agent in your Kubernetes cluster:

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install my-agent gitlab/gitlab-agent \
  --namespace gitlab-agent \
  --create-namespace \
  --set image.tag=v15.7.0 \
  --set config.token=<agent-token> \
  --set config.kasAddress=wss://kas.gitlab.com
```

4. Use the agent in your CI/CD pipelines:

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl:1.23
  script:
    - kubectl config use-context ci:${CI_PROJECT_PATH}:my-agent
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl rollout status deployment/my-app -n default
  only:
    - main
```

### Using GitLab's Auto DevOps with Kubernetes

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml

variables:
  # Override Auto DevOps variables as needed
  POSTGRES_ENABLED: "false"
  REPLICAS: 3
  KUBERNETES_NAMESPACE: my-app-production
  STAGING_ENABLED: "true"
  CANARY_ENABLED: "true"
  PRODUCTION_MANUAL_DEPLOYMENT: "true"
```

## Auto DevOps

### Enabling Auto DevOps

Auto DevOps provides a pre-configured CI/CD pipeline for Kubernetes deployments.

1. **Project Level**: Enable in Settings > CI/CD > Auto DevOps
2. **Via .gitlab-ci.yml**:

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml
```

### Customizing Auto DevOps

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml

variables:
  # Disable built-in PostgreSQL
  POSTGRES_ENABLED: "false"
  # Set custom Kubernetes namespace
  KUBERNETES_NAMESPACE: my-app
  # Enable staging environment
  STAGING_ENABLED: "true"
  # Set number of production replicas
  REPLICAS: 3
  # Use custom Helm chart
  AUTO_DEVOPS_CHART: .gitlab/charts/my-custom-chart
  # Custom Helm values
  AUTO_DEVOPS_CHART_REPOSITORY: https://charts.example.com/
  AUTO_DEVOPS_CHART_REPOSITORY_USERNAME: $CHART_REPOSITORY_USERNAME
  AUTO_DEVOPS_CHART_REPOSITORY_PASSWORD: $CHART_REPOSITORY_PASSWORD

# Override specific Auto DevOps jobs
production:
  stage: production
  extends: .auto-deploy
  script:
    - auto-deploy deploy
    # Add custom post-deployment tasks
    - kubectl apply -f kubernetes/additional-resources.yaml
  environment:
    name: production
    url: https://myapp.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

### Using a Custom Dockerfile

By default, Auto DevOps uses Herokuish buildpacks. To use your own Dockerfile:

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml

variables:
  # Use custom Dockerfile
  AUTO_DEVOPS_BUILD_IMAGE_CNB_ENABLED: "false"
```

### Using a Custom Helm Chart

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml

variables:
  # Use custom Helm chart
  AUTO_DEVOPS_CHART: .gitlab/charts/my-custom-chart
```

## Performance Optimization

### Caching Dependencies

```yaml
variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"

.node-cache:
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - .npm/
    policy: pull-push

test:
  stage: test
  image: node:18
  extends: .node-cache
  script:
    - npm ci --cache .npm --prefer-offline
    - npm test
  only:
    - main
```

### Using Docker Layer Caching

```yaml
build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
    DOCKER_HOST: tcp://docker:2375
    # Enable Docker BuildKit
    DOCKER_BUILDKIT: 1
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build 
      --cache-from $CI_REGISTRY_IMAGE:latest 
      --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA 
      --tag $CI_REGISTRY_IMAGE:latest 
      --build-arg BUILDKIT_INLINE_CACHE=1 .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

### Using GitLab Runners on Kubernetes

```yaml
# .gitlab/runners/values.yaml
gitlabUrl: https://gitlab.example.com/
runnerRegistrationToken: "your-registration-token"
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "gitlab-runners"
        image = "ubuntu:20.04"
        privileged = true
        [[runners.kubernetes.volumes.pvc]]
          name = "docker-cache"
          mount_path = "/var/lib/docker"
# Installation:
# helm install gitlab-runner -f .gitlab/runners/values.yaml gitlab/gitlab-runner --namespace gitlab-runners
```

## Best Practices

### CI/CD Best Practices

1. **Use Pipeline Caching**: Cache dependencies between pipeline runs.

2. **Group Jobs into Stages**: Organize jobs in logical stages for clarity.

3. **Use Variables**: Define variables for reusable values.

```yaml
variables:
  APP_NAME: "my-app"
  DOCKER_DRIVER: overlay2
```

4. **Use Job Templates**: Create reusable job templates with YAML anchors.

```yaml
.deploy_template: &deploy_template
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    - kubectl apply -f kubernetes/

deploy_staging:
  <<: *deploy_template
  environment:
    name: staging
```

5. **Use Include Files**: Break down large pipeline configurations.

```yaml
include:
  - local: ci/build.gitlab-ci.yml
  - local: ci/test.gitlab-ci.yml
  - local: ci/deploy.gitlab-ci.yml
```

6. **Secure Secrets**: Store sensitive data in GitLab CI/CD variables.

7. **Optimize Docker Image Building**: Use cache and multi-stage builds.

8. **Implement Parallel Jobs**: Speed up pipelines with concurrent jobs.

```yaml
test:
  parallel: 3
  script:
    - npm test -- --split=$CI_NODE_INDEX/$CI_NODE_TOTAL
```

### Kubernetes Deployment Best Practices

1. **Use Namespaces**: Separate resources by environment or project.

```yaml
deploy:
  script:
    - kubectl create namespace $CI_ENVIRONMENT_SLUG --dry-run=client -o yaml | kubectl apply -f -
    - kubectl apply -f kubernetes/ -n $CI_ENVIRONMENT_SLUG
```

2. **Implement Resource Limits**: Set CPU and memory limits for deployments.

```yaml
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

3. **Use Helm or Kustomize**: Manage environment-specific configurations.

4. **Implement Progressive Delivery**: Use canary or blue-green deployments.

5. **Add Health Checks**: Include readiness and liveness probes.

```yaml
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

6. **Use GitLab Environments**: Track deployments and enable rollbacks.

```yaml
deploy:
  environment:
    name: production
    url: https://myapp.example.com
```

7. **Implement Rollback Strategies**: Prepare for deployment failures.

```yaml
deploy:
  script:
    - kubectl apply -f kubernetes/
    - kubectl rollout status deployment/app -n default --timeout=300s || kubectl rollout undo deployment/app -n default
```

8. **Scan Kubernetes Manifests**: Check for security and best practices.

```yaml
kube_lint:
  image: garethr/kubeval
  script:
    - kubeval --strict kubernetes/*.yaml
```

## Real-World Examples

### Microservices Application

```yaml
stages:
  - build
  - test
  - scan
  - deploy-staging
  - test-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  DOCKER_HOST: tcp://docker:2375

build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

unit_tests:
  stage: test
  image: node:18
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
  script:
    - npm ci
    - npm test
  only:
    - main

integration_tests:
  stage: test
  image: node:18
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
  services:
    - name: mongo:4.4
      alias: mongodb
  variables:
    MONGODB_URI: "mongodb://mongodb:27017/test"
  script:
    - npm ci
    - npm run test:integration
  only:
    - main

container_scanning:
  stage: scan
  image:
    name: registry.gitlab.com/gitlab-org/security-products/container-scanning:5
    entrypoint: [""]
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  services:
    - docker:20.10.12-dind
  script:
    - /analyzer run
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
  only:
    - main

deploy_staging:
  stage: deploy-staging
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update the staging deployment image
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/staging/deployment.yaml
    # Apply the Kubernetes manifests
    - kubectl apply -f kubernetes/staging/
    - kubectl rollout status deployment/my-app -n staging
  environment:
    name: staging
    url: https://staging.myapp.example.com
  only:
    - main

e2e_tests:
  stage: test-staging
  image: node:18
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
  script:
    - npm ci
    - npm run test:e2e
  variables:
    APP_URL: https://staging.myapp.example.com
  only:
    - main

performance_tests:
  stage: test-staging
  image: grafana/k6:latest
  script:
    - k6 run load-tests/performance.js
  variables:
    K6_BROWSER_ENABLED: "true"
    APP_URL: https://staging.myapp.example.com
  only:
    - main

deploy_production:
  stage: deploy-production
  image: bitnami/kubectl:1.23
  script:
    - echo "$KUBE_CONFIG" > kubeconfig.yaml
    - export KUBECONFIG=kubeconfig.yaml
    # Update the production deployment image
    - sed -i "s|image: $CI_REGISTRY_IMAGE:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" kubernetes/production/deployment.yaml
    # Apply the Kubernetes manifests
    - kubectl apply -f kubernetes/production/
    - kubectl rollout status deployment/my-app -n production
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main
```

### Microservices with Helm and GitLab Agent

```yaml
stages:
  - build
  - test
  - scan
  - deploy-staging
  - test-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  DOCKER_HOST: tcp://docker:2375

.setup_helm:
  before_script:
    - curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update

build:
  stage: build
  image: docker:20.10.12
  services:
    - docker:20.10.12-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHA -f services/api/Dockerfile services/api
    - docker build -t $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA -f services/frontend/Dockerfile services/frontend
    - docker push $CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
  only:
    - main

test_api:
  stage: test
  image: node:18
  cache:
    key: api-$CI_COMMIT_REF_SLUG
    paths:
      - services/api/node_modules/
  script:
    - cd services/api
    - npm ci
    - npm test
  only:
    - main

test_frontend:
  stage: test
  image: node:18
  cache:
    key: frontend-$CI_COMMIT_REF_SLUG
    paths:
      - services/frontend/node_modules/
  script:
    - cd services/frontend
    - npm ci
    - npm test
  only:
    - main

scan_api:
  stage: scan
  image:
    name: registry.gitlab.com/gitlab-org/security-products/container-scanning:5
    entrypoint: [""]
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHA
  services:
    - docker:20.10.12-dind
  script:
    - /analyzer run
  artifacts:
    reports:
      container_scanning: gl-api-container-scanning-report.json
  only:
    - main

scan_frontend:
  stage: scan
  image:
    name: registry.gitlab.com/gitlab-org/security-products/container-scanning:5
    entrypoint: [""]
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
  services:
    - docker:20.10.12-dind
  script:
    - /analyzer run
  artifacts:
    reports:
      container_scanning: gl-frontend-container-scanning-report.json
  only:
    - main

deploy_staging:
  stage: deploy-staging
  image: registry.gitlab.com/gitlab-org/cluster-integration/kubernetes-agent/cli:stable
  extends: .setup_helm
  script:
    - kubectl config use-context ci:${CI_PROJECT_PATH}:my-agent
    # Update Helm values with new image tags
    - |
      cat > staging-values.yaml << EOF
      api:
        image:
          repository: $CI_REGISTRY_IMAGE/api
          tag: $CI_COMMIT_SHA
      frontend:
        image:
          repository: $CI_REGISTRY_IMAGE/frontend
          tag: $CI_COMMIT_SHA
      EOF
    # Deploy with Helm
    - helm upgrade --install my-app ./helm/my-app -f ./helm/my-app/values-staging.yaml -f staging-values.yaml --namespace staging
    # Wait for deployment to complete
    - kubectl rollout status deployment/my-app-api -n staging
    - kubectl rollout status deployment/my-app-frontend -n staging
  environment:
    name: staging
    url: https://staging.myapp.example.com
  only:
    - main

test_staging:
  stage: test-staging
  image: node:18
  script:
    - npm install -g newman
    - newman run tests/postman/api-tests.json --environment tests/postman/staging-env.json
  variables:
    APP_URL: https://staging.myapp.example.com
  only:
    - main

deploy_production:
  stage: deploy-production
  image: registry.gitlab.com/gitlab-org/cluster-integration/kubernetes-agent/cli:stable
  extends: .setup_helm
  script:
    - kubectl config use-context ci:${CI_PROJECT_PATH}:my-agent
    # Update Helm values with new image tags
    - |
      cat > production-values.yaml << EOF
      api:
        image:
          repository: $CI_REGISTRY_IMAGE/api
          tag: $CI_COMMIT_SHA
      frontend:
        image:
          repository: $CI_REGISTRY_IMAGE/frontend
          tag: $CI_COMMIT_SHA
      EOF
    # Deploy with Helm
    - helm upgrade --install my-app ./helm/my-app -f ./helm/my-app/values-production.yaml -f production-values.yaml --namespace production
    # Wait for deployment to complete
    - kubectl rollout status deployment/my-app-api -n production
    - kubectl rollout status deployment/my-app-frontend -n production
  environment:
    name: production
    url: https://myapp.example.com
  when: manual
  only:
    - main
```

## Summary

GitLab CI/CD provides a comprehensive platform for deploying applications to Kubernetes, with features like built-in container registry, Kubernetes integration, and Auto DevOps. By leveraging GitLab's powerful capabilities, you can build robust CI/CD pipelines that automate the building, testing, and deployment of your applications to Kubernetes environments.

Key benefits of using GitLab CI for Kubernetes deployments include:

1. **Integrated Platform**: Container registry, CI/CD, and Kubernetes management in one place
2. **Flexible Pipeline Configuration**: Custom pipelines with stages, jobs, and includes
3. **Environment Management**: Track deployments across environments with URLs and history
4. **Security Scanning**: Built-in container, SAST, and dependency scanning
5. **Auto DevOps**: Pre-configured CI/CD for Kubernetes deployments
6. **GitLab Kubernetes Agent**: Secure connection to Kubernetes clusters

By following the best practices and examples in this guide, you can implement efficient, secure, and automated CI/CD pipelines for your Kubernetes applications using GitLab CI.