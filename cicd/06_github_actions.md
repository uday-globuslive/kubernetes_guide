# GitHub Actions for Kubernetes

This guide covers how to use GitHub Actions to build, test, and deploy applications to Kubernetes clusters, including best practices, common patterns, and real-world examples.

## Table of Contents

1. [Introduction to GitHub Actions](#introduction-to-github-actions)
2. [Setting Up GitHub Actions for Kubernetes](#setting-up-github-actions-for-kubernetes)
3. [Building and Publishing Container Images](#building-and-publishing-container-images)
4. [Kubernetes Deployment Workflows](#kubernetes-deployment-workflows)
5. [Testing in GitHub Actions](#testing-in-github-actions)
6. [Security Scanning](#security-scanning)
7. [Advanced Deployment Patterns](#advanced-deployment-patterns)
8. [Environment Management](#environment-management)
9. [Secrets Management](#secrets-management)
10. [Optimizing GitHub Actions](#optimizing-github-actions)
11. [Real-World Examples](#real-world-examples)
12. [Best Practices](#best-practices)

## Introduction to GitHub Actions

GitHub Actions is a CI/CD platform integrated directly into GitHub repositories. It allows you to automate your software development workflows with a wide range of capabilities for building, testing, and deploying applications.

### Key Concepts

- **Workflows**: YAML files defined in the `.github/workflows` directory of your repository
- **Jobs**: A set of steps that execute on the same runner
- **Steps**: Individual tasks that run commands or actions
- **Actions**: Reusable units of code that can be shared across workflows
- **Runners**: Servers that run your workflows (GitHub-hosted or self-hosted)
- **Events**: Specific activities that trigger workflow runs

### Benefits for Kubernetes Deployments

- **Tight Integration with Source Code**: Workflows are stored alongside application code
- **Built-in Secrets Management**: Secure handling of credentials
- **Matrix Builds**: Test across multiple configurations
- **Reusable Actions**: Community-maintained actions for common tasks
- **Self-Hosted Runners**: Run workflows in your own infrastructure
- **GitHub Marketplace**: Discover and use actions created by the community

## Setting Up GitHub Actions for Kubernetes

### Basic Workflow Structure

```yaml
# .github/workflows/kubernetes-deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        
      - name: Configure Kubernetes context
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
          
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f kubernetes/deployment.yaml
          kubectl rollout status deployment/my-app -n default
```

### Self-Hosted Runners

For secure access to private Kubernetes clusters, you can set up self-hosted runners:

```yaml
# .github/workflows/self-hosted-deploy.yml
name: Deploy with Self-Hosted Runner

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f kubernetes/deployment.yaml
          kubectl rollout status deployment/my-app -n default
```

### Using Multiple Environments

```yaml
# .github/workflows/multi-environment.yml
name: Multi-Environment Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes (Staging)
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}
          
      - name: Deploy to Staging
        run: |
          kubectl apply -f kubernetes/staging/

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes (Production)
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}
          
      - name: Deploy to Production
        run: |
          kubectl apply -f kubernetes/production/
```

## Building and Publishing Container Images

### Docker Build and Push

```yaml
# .github/workflows/build-push.yml
name: Build and Push

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: user/app:latest,user/app:${{ github.sha }}
          cache-from: type=registry,ref=user/app:latest
          cache-to: type=inline
```

### Building for Multiple Architectures

```yaml
# .github/workflows/multi-arch-build.yml
name: Multi-Arch Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: user/app:latest,user/app:${{ github.sha }}
```

### Using GitHub Container Registry

```yaml
# .github/workflows/ghcr-push.yml
name: Build and Push to GHCR

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=ref,event=branch
            type=sha,format=long
      
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Kubernetes Deployment Workflows

### Basic Deployment

```yaml
# .github/workflows/k8s-deploy.yml
name: Kubernetes Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Deploy to Kubernetes
        run: |
          # Replace image tag in deployment file
          sed -i "s|image: user/app:.*|image: user/app:${{ github.sha }}|" kubernetes/deployment.yaml
          
          # Apply Kubernetes manifests
          kubectl apply -f kubernetes/deployment.yaml
          kubectl apply -f kubernetes/service.yaml
          
          # Wait for deployment to complete
          kubectl rollout status deployment/my-app -n default
```

### Using kustomize

```yaml
# .github/workflows/kustomize-deploy.yml
name: Kustomize Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup kustomize
        uses: imranismail/setup-kustomize@v1
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Update kustomization.yaml
        run: |
          cd kubernetes/overlays/production
          kustomize edit set image user/app=user/app:${{ github.sha }}
      
      - name: Deploy with kustomize
        run: |
          kustomize build kubernetes/overlays/production | kubectl apply -f -
          kubectl rollout status deployment/my-app -n production
```

### Using Helm

```yaml
# .github/workflows/helm-deploy.yml
name: Helm Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=3 \
            --atomic \
            --timeout 5m
```

## Testing in GitHub Actions

### Unit and Integration Testing

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm test
      
      - name: Run integration tests
        run: npm run test:integration
```

### End-to-End Testing with Kubernetes

```yaml
# .github/workflows/e2e-test.yml
name: End-to-End Tests

on:
  push:
    branches: [ main ]

jobs:
  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build test image
        uses: docker/build-push-action@v3
        with:
          context: .
          push: false
          load: true
          tags: my-app:test
      
      - name: Set up kind
        uses: engineerd/setup-kind@v0.5.0
        with:
          version: v0.17.0
      
      - name: Load image to kind
        run: |
          kind load docker-image my-app:test
      
      - name: Deploy to kind cluster
        run: |
          # Replace image in deployment
          sed -i 's|image: .*|image: my-app:test|' kubernetes/test/deployment.yaml
          
          # Apply Kubernetes manifests
          kubectl apply -f kubernetes/test/
          kubectl wait --for=condition=available --timeout=300s deployment/my-app -n default
      
      - name: Run end-to-end tests
        run: |
          # Get service URL
          export SERVICE_URL=$(kubectl get svc my-app -n default -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          
          # Run tests against the deployed service
          npm run test:e2e
```

## Security Scanning

### Container Image Scanning

```yaml
# .github/workflows/image-scan.yml
name: Container Image Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Build image
        run: |
          docker build -t my-app:${{ github.sha }} .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:${{ github.sha }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

### Kubernetes Manifest Validation

```yaml
# .github/workflows/k8s-validation.yml
name: Kubernetes Validation

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Validate Kubernetes manifests
        uses: instrumenta/kubeval-action@master
        with:
          files: kubernetes/
      
      - name: Scan for misconfigurations
        uses: bridgecrewio/checkov-action@master
        with:
          directory: kubernetes/
          framework: kubernetes
```

### Secret Scanning

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scanning

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: TruffleHog OSS
        uses: trufflesecurity/trufflehog-actions-scanning@master
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified
```

## Advanced Deployment Patterns

### Canary Deployments

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy-canary:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Deploy Canary (10% traffic)
        run: |
          # Update canary deployment with new image
          sed -i "s|image: user/app:.*|image: user/app:${{ github.sha }}|" kubernetes/canary/deployment.yaml
          
          # Apply canary deployment and service
          kubectl apply -f kubernetes/canary/
          kubectl rollout status deployment/my-app-canary -n default
      
      - name: Wait for validation period
        run: sleep 300  # Wait 5 minutes for metrics collection
      
      - name: Check canary metrics
        id: check-metrics
        run: |
          # Logic to check error rates, latency, etc.
          # This could be a script that queries Prometheus or other monitoring system
          ERROR_RATE=$(curl -s http://monitoring-api/query?metric=error_rate_canary)
          echo "ERROR_RATE=$ERROR_RATE" >> $GITHUB_OUTPUT
          if [[ $(echo "$ERROR_RATE < 0.01" | bc) -eq 1 ]]; then
            echo "CANARY_HEALTHY=true" >> $GITHUB_OUTPUT
          else
            echo "CANARY_HEALTHY=false" >> $GITHUB_OUTPUT
          fi
      
      - name: Promote Canary to Production
        if: steps.check-metrics.outputs.CANARY_HEALTHY == 'true'
        run: |
          # Update production deployment with new image
          sed -i "s|image: user/app:.*|image: user/app:${{ github.sha }}|" kubernetes/production/deployment.yaml
          kubectl apply -f kubernetes/production/
          kubectl rollout status deployment/my-app -n default
          
          # Remove canary deployment
          kubectl delete -f kubernetes/canary/
      
      - name: Rollback Canary
        if: steps.check-metrics.outputs.CANARY_HEALTHY == 'false'
        run: |
          kubectl delete -f kubernetes/canary/
          echo "Canary deployment failed metrics check and was rolled back"
          exit 1
```

### Blue-Green Deployments

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deployment

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Determine current environment
        id: current-env
        run: |
          CURRENT_ENV=$(kubectl get service my-app-service -n default -o jsonpath='{.spec.selector.environment}')
          if [ "$CURRENT_ENV" == "blue" ]; then
            echo "NEW_ENV=green" >> $GITHUB_OUTPUT
            echo "OLD_ENV=blue" >> $GITHUB_OUTPUT
          else
            echo "NEW_ENV=blue" >> $GITHUB_OUTPUT
            echo "OLD_ENV=green" >> $GITHUB_OUTPUT
          fi
      
      - name: Deploy new environment
        run: |
          # Create deployment for new environment
          cat kubernetes/deployment.yaml | \
            sed "s|image: user/app:.*|image: user/app:${{ github.sha }}|" | \
            sed "s|environment: .*|environment: ${{ steps.current-env.outputs.NEW_ENV }}|" | \
            kubectl apply -f -
          
          # Wait for new deployment to be ready
          kubectl rollout status deployment/my-app-${{ steps.current-env.outputs.NEW_ENV }} -n default
      
      - name: Test new deployment
        id: test
        run: |
          # Run tests against the new deployment
          # This would typically be a script calling your APIs and checking responses
          echo "TESTS_PASSED=true" >> $GITHUB_OUTPUT
      
      - name: Switch traffic
        if: steps.test.outputs.TESTS_PASSED == 'true'
        run: |
          # Update service to point to new deployment
          kubectl patch service my-app-service -n default -p \
            '{"spec":{"selector":{"environment":"${{ steps.current-env.outputs.NEW_ENV }}"}}}'
          
          # Wait for confirmation from manual approval
          sleep 30
      
      - name: Clean up old environment
        if: steps.test.outputs.TESTS_PASSED == 'true'
        run: |
          # Delete old deployment
          kubectl delete deployment my-app-${{ steps.current-env.outputs.OLD_ENV }} -n default
```

## Environment Management

### Environment-specific Configurations

```yaml
# .github/workflows/environment-deploy.yml
name: Environment Deployment

on:
  push:
    branches:
      - main
      - 'release/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    # Determine target environment based on branch
    environment: ${{ github.ref == 'refs/heads/main' && 'staging' || 'production' }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Load environment variables
        id: env-vars
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "ENV_NAME=staging" >> $GITHUB_OUTPUT
            echo "REPLICAS=2" >> $GITHUB_OUTPUT
          else
            echo "ENV_NAME=production" >> $GITHUB_OUTPUT
            echo "REPLICAS=5" >> $GITHUB_OUTPUT
          fi
      
      - name: Deploy to environment
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace ${{ steps.env-vars.outputs.ENV_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=${{ steps.env-vars.outputs.REPLICAS }} \
            --set environment=${{ steps.env-vars.outputs.ENV_NAME }} \
            --values ./helm/my-app/values-${{ steps.env-vars.outputs.ENV_NAME }}.yaml
```

### Managing Multiple Environments

```yaml
# .github/workflows/multi-stage-deploy.yml
name: Multi-Stage Deployment

on:
  push:
    branches:
      - main
      - staging
      - production

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - id: set-env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=development" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/production" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          fi
  
  deploy:
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets[format('{0}_KUBECONFIG', needs.determine-environment.outputs.environment)] }}
      
      - name: Deploy to environment
        run: |
          kubectl apply -k kubernetes/overlays/${{ needs.determine-environment.outputs.environment }}
          kubectl rollout status deployment/my-app -n ${{ needs.determine-environment.outputs.environment }}
```

## Secrets Management

### Using GitHub Secrets

```yaml
# .github/workflows/secrets-deploy.yml
name: Deploy with Secrets

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Create Kubernetes Secrets
        run: |
          # Create Secret from GitHub Secrets
          kubectl create secret generic app-secrets \
            --namespace default \
            --from-literal=api-key=${{ secrets.API_KEY }} \
            --from-literal=database-password=${{ secrets.DB_PASSWORD }} \
            --dry-run=client -o yaml | kubectl apply -f -
      
      - name: Deploy application
        run: |
          kubectl apply -f kubernetes/deployment.yaml
          kubectl rollout status deployment/my-app -n default
```

### Using External Secret Stores

```yaml
# .github/workflows/external-secrets.yml
name: Deploy with External Secrets

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-west-2
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Deploy External Secrets Operator resources
        run: |
          # Create SecretStore and ExternalSecret resources
          kubectl apply -f kubernetes/external-secrets/
      
      - name: Deploy application
        run: |
          kubectl apply -f kubernetes/deployment.yaml
          kubectl rollout status deployment/my-app -n default
```

## Optimizing GitHub Actions

### Caching Dependencies

```yaml
# .github/workflows/caching.yml
name: Build with Caching

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Cache Docker layers
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: user/app:${{ github.sha }}
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new
      
      # Temp fix for growing cache: https://github.com/docker/build-push-action/issues/252
      - name: Move cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
# Reusable workflow for Kubernetes deployments
name: Reusable Kubernetes Deployment

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      namespace:
        required: true
        type: string
      manifest-path:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      kubeconfig:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.kubeconfig }}
      
      - name: Update image tag
        run: |
          sed -i "s|image: user/app:.*|image: user/app:${{ inputs.image-tag }}|" ${{ inputs.manifest-path }}
      
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f ${{ inputs.manifest-path }} -n ${{ inputs.namespace }}
          kubectl rollout status deployment/my-app -n ${{ inputs.namespace }}
```

Using the reusable workflow:

```yaml
# .github/workflows/call-deploy.yml
name: Deploy to Environments

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.tag.outputs.tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Generate tag
        id: tag
        run: echo "tag=${GITHUB_SHA}" >> $GITHUB_OUTPUT
      
      # Build and push image steps...
  
  deploy-staging:
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      namespace: staging
      manifest-path: kubernetes/staging/deployment.yaml
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}
  
  deploy-production:
    needs: [build, deploy-staging]
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      namespace: production
      manifest-path: kubernetes/production/deployment.yaml
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}
```

## Real-World Examples

### Microservices CI/CD Pipeline

```yaml
# .github/workflows/microservice-pipeline.yml
name: Microservice Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint code
        run: npm run lint
      
      - name: Run unit tests
        run: npm test
      
      - name: Run integration tests
        run: npm run test:integration
  
  scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run code security scan
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript
      
      - name: Run dependency check
        run: npm audit --production
  
  build:
    runs-on: ubuntu-latest
    needs: [test, scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.tag.outputs.tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Generate tag
        id: tag
        run: echo "tag=${GITHUB_SHA}" >> $GITHUB_OUTPUT
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: |
            user/app:latest
            user/app:${{ steps.tag.outputs.tag }}
          cache-from: type=registry,ref=user/app:latest
          cache-to: type=inline
  
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}
      
      - name: Deploy to staging
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace staging \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --set replicaCount=2 \
            --set environment=staging \
            --set ingress.host=staging.myapp.com \
            --atomic \
            --timeout 5m
  
  e2e-tests:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: staging
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run end-to-end tests
        env:
          API_URL: https://staging.myapp.com
        run: npm run test:e2e
  
  deploy-production:
    runs-on: ubuntu-latest
    needs: e2e-tests
    environment: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure Kubernetes
        uses: azure/k8s-set-context@v2
        with:
          kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}
      
      - name: Deploy to production
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace production \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --set replicaCount=5 \
            --set environment=production \
            --set ingress.host=myapp.com \
            --atomic \
            --timeout 5m
```

## Best Practices

### GitHub Actions Best Practices

1. **Use Specific Action Versions**: Pin actions to specific versions for stability.
   ```yaml
   uses: actions/checkout@v3  # Good
   ```
   Instead of:
   ```yaml
   uses: actions/checkout@master  # Bad - can break unexpectedly
   ```

2. **Implement CI/CD Stages**: Build pipelines with distinct stages that can fail early.

3. **Cache Dependencies**: Use caching to speed up workflows.

4. **Secure Secrets Management**: Never hardcode secrets, use GitHub Secrets.

5. **Use Self-Hosted Runners for Sensitive Operations**: Especially when accessing private infrastructure.

6. **Implement Proper Environment Protection Rules**: Require approvals for production deployments.

7. **Create Reusable Workflows**: DRY principle applies to CI/CD too.

8. **Implement Matrix Builds**: Test across multiple configurations efficiently.
   ```yaml
   strategy:
     matrix:
       node-version: [14.x, 16.x, 18.x]
       os: [ubuntu-latest, windows-latest]
   ```

9. **Use Job Dependencies**: Define clear flow with `needs` to ensure proper sequence.

10. **Add Workflow Status Badges**: Show build status in your README.md.
    ```markdown
    ![Build Status](https://github.com/username/repo/actions/workflows/main.yml/badge.svg)
    ```

### Kubernetes Deployment Best Practices

1. **Use Declarative Manifests**: Kustomize or Helm charts for environment management.

2. **Implement Progressive Delivery**: Use canary or blue-green deployments for reduced risk.

3. **Ensure Idempotent Deployments**: Deployments should be repeatable without side effects.

4. **Include Post-Deployment Validation**: Verify your deployments work as expected.

5. **Configure Resource Limits**: Always define CPU and memory limits for deployments.

6. **Implement Health Checks**: Use readiness and liveness probes in Kubernetes.

7. **Manage Secrets Securely**: Use dedicated secret management solutions.

8. **Version Control Everything**: All manifests and scripts should be in version control.

9. **Document Pipeline Behavior**: Ensure team members understand the workflow.

10. **Implement Monitoring and Observability**: Know when deployments succeed or fail.

## Summary

GitHub Actions provides a powerful and flexible platform for implementing CI/CD pipelines for Kubernetes deployments. By leveraging its deep integration with GitHub repositories, built-in security features, and extensive marketplace of community-contributed actions, you can create efficient, secure, and automated workflows for your applications.

Key advantages of using GitHub Actions for Kubernetes deployments include:

1. **Seamless Integration with Source Code**: Workflows live alongside your application code
2. **Built-in Security**: Secret management and environment protection rules
3. **Extensibility**: Rich ecosystem of community-built actions and workflows
4. **Flexibility**: Support for complex deployment strategies and patterns
5. **Reduced Infrastructure Maintenance**: GitHub-hosted runners require no maintenance

Whether you're deploying a simple application or a complex microservices architecture, GitHub Actions can be configured to meet your needs, providing a streamlined path from code to production.