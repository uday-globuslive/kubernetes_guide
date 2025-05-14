# Docker Registry and Distribution

A Docker registry is a storage and content delivery system containing named Docker images, available in different tagged versions. This guide covers Docker registry concepts, setup options, and best practices.

## Registry Concepts

### Images and Tags

Docker images are stored in registries with a naming convention:

```
registry-host:port/namespace/repository:tag
```

Examples:
- `docker.io/library/ubuntu:20.04` (public image on Docker Hub)
- `registry.example.com:5000/myteam/myapp:v1.2.3` (private registry)

If registry-host is omitted, Docker Hub (`docker.io`) is assumed.

### Registry Types

1. **Public Registries**:
   - Docker Hub: The default public registry
   - GitHub Container Registry (GHCR)
   - Amazon ECR Public Gallery
   - Google Container Registry
   - Azure Container Registry (public repositories)
   - Quay.io

2. **Private Registries**:
   - Self-hosted Docker Registry
   - Amazon Elastic Container Registry (ECR)
   - Google Container Registry (GCR)
   - Azure Container Registry (ACR)
   - JFrog Artifactory
   - Nexus Repository
   - Harbor

## Docker Hub

Docker Hub is Docker's official cloud-based registry service for storing and retrieving container images.

### Using Docker Hub

```bash
# Login to Docker Hub
docker login

# Pull an image
docker pull nginx:latest

# Tag an image
docker tag myapp:latest username/myapp:latest

# Push an image
docker push username/myapp:latest
```

### Docker Hub Features

- **Official Images**: Curated base images maintained by Docker
- **Verified Publishers**: Images from trusted software vendors
- **Automated Builds**: Build images automatically from source code repositories
- **Teams and Organizations**: Manage access to private repositories
- **Webhooks**: Trigger actions when images are pushed

## Setting Up a Private Registry

### Docker Registry (Open Source)

Running the official Docker registry:

```bash
# Run a local registry
docker run -d -p 5000:5000 --name registry registry:2

# Tag and push to local registry
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest

# Pull from local registry
docker pull localhost:5000/myapp:latest
```

### Securing a Private Registry

#### Adding TLS

Create a certificate:
```bash
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt
```

Run registry with TLS:
```bash
docker run -d -p 5000:5000 --name registry \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

#### Adding Authentication

Create password file:
```bash
mkdir auth
docker run --entrypoint htpasswd registry:2 -Bbn username password > auth/htpasswd
```

Run registry with authentication:
```bash
docker run -d -p 5000:5000 --name registry \
  -v $(pwd)/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

Login to secure registry:
```bash
docker login registry.example.com:5000
```

### Registry Storage Backends

Configure the registry to use different storage backends:

```yaml
# config.yml
version: 0.1
storage:
  s3:
    accesskey: awsaccesskey
    secretkey: awssecretkey
    region: us-west-1
    bucket: docker-registry
```

Run with custom configuration:
```bash
docker run -d -p 5000:5000 --name registry \
  -v $(pwd)/config.yml:/etc/docker/registry/config.yml \
  registry:2
```

Available storage drivers:
- filesystem (default)
- s3 (AWS S3)
- azure (Azure Blob Storage)
- gcs (Google Cloud Storage)
- swift (OpenStack Swift)
- oss (Alibaba Cloud OSS)

## Harbor: Enterprise Registry

Harbor is an open source container image registry that secures artifacts with policies and role-based access control:

```bash
# Download Harbor installer
wget https://github.com/goharbor/harbor/releases/download/v2.4.1/harbor-offline-installer-v2.4.1.tgz
tar xzvf harbor-offline-installer-v2.4.1.tgz

# Configure Harbor
cd harbor
cp harbor.yml.tmpl harbor.yml
# Edit harbor.yml to set hostname, certificate paths, and passwords

# Install Harbor
./install.sh
```

Harbor features:
- Role-based access control
- Image vulnerability scanning 
- Content signing and validation
- Replication between registries
- Extensible API and web UI
- LDAP/AD integration
- Project quota management

## Registry Mirroring

Configure a registry as a pull-through cache:

```yaml
# config.yml for mirror
version: 0.1
proxy:
  remoteurl: https://registry-1.docker.io
```

Docker daemon configuration to use mirror:
```json
{
  "registry-mirrors": ["https://my-mirror.example.com:5000"]
}
```

## Best Practices

### Image Tagging Strategies

Effective tagging strategies to manage images:

1. **Semantic Versioning**:
   - `myapp:1.2.3` (specific version)
   - `myapp:1.2` (latest patch version)
   - `myapp:1` (latest minor version)
   - `myapp:latest` (latest version)

2. **Git-based Tagging**:
   - `myapp:git-commit-hash`
   - `myapp:branch-name`
   - `myapp:pr-number`

3. **Environment/Stage Tagging**:
   - `myapp:dev`
   - `myapp:staging`
   - `myapp:prod`

4. **Date-based Tagging**:
   - `myapp:20230415`
   - `myapp:2023-04-15`

### Image Cleanup and Retention

Strategies to manage the lifecycle of images:

1. **Registry garbage collection**:
   ```bash
   # For Docker Registry
   docker exec -it registry bin/registry garbage-collect /etc/docker/registry/config.yml
   ```

2. **Automated cleanup policies**:
   - Harbor: Configure retention policies per project
   - ECR: Lifecycle policies based on age or count
   - GCR: Container Analysis API for cleanup

3. **Tagging expiration**:
   - Add expiration policies based on tags
   - Automatically delete development or PR-based images after a certain period

4. **Client-side cleanup**:
   ```bash
   # Remove dangling images
   docker image prune
   
   # Remove all unused images
   docker image prune -a
   
   # Remove images older than 24h
   docker image prune -a --filter "until=24h"
   ```

### Registry High Availability

For production environments, consider:

1. **Load balancing**:
   - Use multiple registry instances behind a load balancer
   - Configure sticky sessions for push operations

2. **Shared storage**:
   - Use S3 or another distributed storage backend

3. **Distributed cache**:
   - Configure Redis for cache sharing

4. **Monitoring and health checks**:
   - Implement health checks for registry instances
   - Set up alerts for registry errors

## CI/CD Integration

Integrate Docker registries with CI/CD pipelines:

### GitHub Actions Example

```yaml
name: Build and Push

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKER_HUB_USERNAME }}
        password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: username/myapp:latest,username/myapp:${{ github.sha }}
```

### GitLab CI Example

```yaml
build:
  stage: build
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
```

## Troubleshooting

Common registry issues and solutions:

### Authentication Issues

```bash
# Error: unauthorized: authentication required
docker login registry.example.com
```

### Certificate Issues

```bash
# Error: x509: certificate signed by unknown authority
# Add registry to Docker daemon insecure registries
# In /etc/docker/daemon.json:
{
  "insecure-registries": ["registry.example.com:5000"]
}
```

### Push/Pull Timeouts

```bash
# Increase timeouts in daemon.json
{
  "registry-mirrors": [],
  "insecure-registries": [],
  "debug": true,
  "experimental": false,
  "registry-timeout": 60
}
```

### Storage Capacity Issues

Monitor and expand storage:
```bash
# Check registry disk usage
docker exec registry df -h

# For S3 storage, check bucket metrics
aws s3api list-objects --bucket my-registry-bucket --query "[sum(Contents[].Size), length(Contents[])]"
```

By understanding Docker registry concepts and implementing these best practices, you can create a robust container image distribution system tailored to your organization's needs, ensuring security, reliability, and efficiency.