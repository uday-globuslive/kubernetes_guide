# Building Docker Images

This chapter provides a comprehensive guide to building Docker images, covering Dockerfile syntax, best practices, multi-stage builds, optimization techniques, and advanced topics for creating efficient and secure container images.

## Introduction to Docker Images

### What is a Docker Image?

A Docker image is a lightweight, standalone, executable package that includes everything needed to run an application:

- Code
- Runtime
- System libraries
- Environment variables
- Configuration files

Images follow a layered architecture where each layer represents a change to the filesystem.

### Image vs Container

Understanding the relationship between images and containers:

- **Image**: Read-only template used to create containers (like a class)
- **Container**: Running instance of an image (like an object)
- **Registry**: Repository for storing and distributing images

```
┌─────────────────┐
│   Image         │ ──▶ docker run ──▶ Container 1
│   (Template)    │ ──▶ docker run ──▶ Container 2
└─────────────────┘ ──▶ docker run ──▶ Container 3
```

### Image Layers

Docker images consist of multiple read-only layers:

```
┌─────────────────┐
│ Application     │ # Latest changes (your code)
├─────────────────┤
│ Dependencies    │ # Libraries and packages
├─────────────────┤
│ OS Files        │ # Operating system files
├─────────────────┤
│ Base Layer      │ # Base image layer
└─────────────────┘
```

Each instruction in a Dockerfile creates a new layer. When you update an image, only the changed layers need to be rebuilt and transferred.

## Dockerfile Basics

### Dockerfile Overview

A Dockerfile is a text document containing commands to build an image:

```dockerfile
# Comment
INSTRUCTION arguments
```

Example of a basic Dockerfile:

```dockerfile
FROM node:14
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

### Essential Dockerfile Instructions

#### FROM

Specifies the base image:

```dockerfile
FROM ubuntu:20.04
FROM node:14-alpine
FROM scratch  # Empty image for binaries
```

#### WORKDIR

Sets the working directory:

```dockerfile
WORKDIR /app
# All subsequent commands will run in /app
```

#### COPY and ADD

Copy files from host to image:

```dockerfile
# Copy local files to the image
COPY package.json .
COPY src/ /app/src/

# ADD can also extract archives and fetch URLs
ADD https://example.com/file.tar.gz /tmp/
```

Difference between COPY and ADD:
- COPY: Simple file/directory copy 
- ADD: Same as COPY plus URL downloads and automatic tar extraction

#### RUN

Executes commands during build:

```dockerfile
# Shell form
RUN apt-get update && apt-get install -y curl

# Exec form
RUN ["apt-get", "update"]
```

#### CMD and ENTRYPOINT

Defines default command and parameters:

```dockerfile
# CMD provides defaults for executing container
CMD ["npm", "start"]

# ENTRYPOINT configures container to run as executable
ENTRYPOINT ["node", "app.js"]

# Combined usage
ENTRYPOINT ["node"]
CMD ["app.js"]  # Can be overridden when running container
```

Differences:
- CMD: Provides default arguments, can be easily overridden
- ENTRYPOINT: Configures container as executable, harder to override

#### ENV

Sets environment variables:

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000 DEBUG=false
```

#### EXPOSE

Documents container ports:

```dockerfile
EXPOSE 80
EXPOSE 443
```

Note: EXPOSE is only documentation. You still need `-p` when running.

#### VOLUME

Creates a mount point:

```dockerfile
VOLUME /data
VOLUME ["/data", "/logs"]
```

#### USER

Sets the user for subsequent commands:

```dockerfile
# Create user
RUN useradd -ms /bin/bash appuser

# Switch to that user
USER appuser
```

#### ARG

Defines build-time variables:

```dockerfile
ARG VERSION=latest
FROM node:${VERSION}

ARG USER_HOME=/home/user
```

## Building Images

### Basic Build Command

```bash
docker build -t myapp:1.0 .
```

- `-t` or `--tag`: Name and optionally tag the image
- `.`: Build context (directory containing Dockerfile)

### Build with Custom Dockerfile

```bash
docker build -f Dockerfile.dev -t myapp:dev .
```

### Build Arguments

Pass build-time variables:

```bash
docker build --build-arg VERSION=16 -t myapp:16 .
```

### Building from Git Repository

```bash
docker build -t myapp https://github.com/user/repo.git#branch
```

### Building without Context

```bash
docker build -t myapp - < Dockerfile
```

## Dockerfile Best Practices

### Reduce Layers

Combine commands to reduce layers:

```dockerfile
# Bad: Three layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good: One layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Leverage Build Cache

Order instructions from least to most likely to change:

```dockerfile
FROM node:14

# Less frequently changing layers first
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install

# More frequently changing layers last
COPY . .
RUN npm run build
```

### Use .dockerignore

Create a `.dockerignore` file to exclude files from the build context:

```
node_modules
npm-debug.log
Dockerfile
.dockerignore
.git
.gitignore
README.md
tests
```

Benefits:
- Faster builds (smaller build context)
- Prevent sensitive files from being included
- Avoid unnecessary cache invalidation

### Use Specific Tags

Prefer specific version tags over latest:

```dockerfile
# Avoid
FROM node:latest

# Better
FROM node:14.17.0-alpine3.13
```

### Minimal Base Images

Choose smaller base images:

```dockerfile
# Large: ~300MB
FROM node:14

# Medium: ~110MB
FROM node:14-slim

# Small: ~40MB
FROM node:14-alpine
```

### Non-Root User

Run as non-root user for security:

```dockerfile
FROM node:14-alpine

# Create app directory and user
RUN mkdir -p /app && \
    addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser && \
    chown -R appuser:appuser /app

WORKDIR /app
USER appuser

# Continue with remaining instructions
```

### Clean Up

Remove unnecessary files in the same layer they're created:

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### One Container, One Concern

Each container should have a single responsibility:

- Web application container
- Database container
- Cache container

## Multi-Stage Builds

### Multi-Stage Build Concept

Multi-stage builds allow using multiple FROM statements in a Dockerfile. Each FROM instruction can use a different base image and begins a new stage. You can selectively copy artifacts from one stage to another, leaving behind everything you don't need.

Benefits:
- Smaller final images
- Separation of build and runtime dependencies
- Improved security by not including build tools in final image

### Basic Multi-Stage Example

```dockerfile
# Build stage
FROM node:14 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Advanced Multi-Stage Example

More complex example with multiple stages:

```dockerfile
# Stage 1: Download and verify dependencies
FROM alpine:3.14 AS downloader
RUN apk add --no-cache curl
WORKDIR /downloads
RUN curl -O https://example.com/package.tar.gz && \
    sha256sum package.tar.gz | grep ^e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

# Stage 2: Build the application
FROM golang:1.17 AS builder
WORKDIR /app
COPY --from=downloader /downloads/package.tar.gz /tmp/
RUN tar -xzf /tmp/package.tar.gz -C /tmp && \
    cp -r /tmp/package/* /app/
COPY *.go .
RUN go build -o myapp .

# Stage 3: Final image
FROM alpine:3.14
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/myapp /usr/local/bin/
USER nobody
ENTRYPOINT ["myapp"]
```

### Using Stage Outputs

Copy files from named stages:

```dockerfile
FROM golang:1.17 AS builder
# ... build stuff ...

FROM ubuntu:20.04 AS tester
COPY --from=builder /app/myapp .
RUN ./test_suite.sh

FROM alpine:3.14
COPY --from=builder /app/myapp .
# Note: we only copy from builder, not tester
```

### Stage Selection

Build just one stage:

```bash
docker build --target builder -t myapp:build .
```

This is useful for debugging or creating different images from the same Dockerfile.

## Image Optimization Techniques

### Minimize Layer Size

Techniques for smaller layers:

1. Use smaller base images (alpine, slim, distroless)
2. Clean up in the same layer you install packages
3. Remove build dependencies
4. Compress files before adding to image

Example:

```dockerfile
FROM alpine:3.14

RUN apk add --no-cache --virtual .build-deps \
        gcc \
        musl-dev \
        python3-dev \
    && pip install --no-cache-dir cython \
    && pip install --no-cache-dir numpy \
    && apk del .build-deps
```

### Layer Caching Strategies

Optimize for cache hits:

```dockerfile
# Copy only package files first
COPY package.json package-lock.json ./
RUN npm install

# Then copy the rest
COPY . .
```

### Squashing Layers

Reduce number of layers in final image:

```bash
docker build --squash -t myapp:squashed .
```

Note: This requires experimental features to be enabled in the Docker daemon.

### Analyze Image Size

Inspect what's taking up space:

```bash
docker image history myapp:latest
docker image history --no-trunc myapp:latest
```

For deeper analysis, use tools like dive:

```bash
docker run --rm -it wagoodman/dive:latest myapp:latest
```

## Specialized Dockerfile Examples

### Node.js Application

```dockerfile
FROM node:14-alpine

# Set working directory
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY . .

# Set non-root user
USER node

# Expose port and define command
EXPOSE 3000
CMD ["node", "app.js"]
```

### Python Web Application

```dockerfile
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m appuser
USER appuser

# Expose port and define command
EXPOSE 5000
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:5000"]
```

### Go Application

```dockerfile
# Build stage
FROM golang:1.17-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# Final stage
FROM alpine:3.14

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/app .
COPY --from=builder /app/config.yaml .

EXPOSE 8080
CMD ["./app"]
```

### Java Spring Boot Application

```dockerfile
# Build stage
FROM maven:3.8-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src/ /app/src/
RUN mvn package -DskipTests

# Run stage
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Static Website with Nginx

```dockerfile
# Build stage for Node.js site
FROM node:14-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html

# Copy custom nginx config if needed
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Advanced Image Building

### ARG and ENV Interaction

Using build arguments to set environment variables:

```dockerfile
# Define build argument with default
ARG NODE_ENV=production

# Set environment variable from build argument
ENV NODE_ENV=${NODE_ENV}

# Later in the file, use the environment variable
RUN if [ "$NODE_ENV" = "development" ]; then \
        npm install; \
    else \
        npm ci --only=production; \
    fi
```

Building with different environments:

```bash
docker build -t myapp:prod .
docker build --build-arg NODE_ENV=development -t myapp:dev .
```

### BuildKit Features

Docker BuildKit provides advanced features:

```dockerfile
# syntax=docker/dockerfile:1.3

# Mount cache for package managers
FROM node:14-alpine
WORKDIR /app
COPY package.json .
RUN --mount=type=cache,target=/root/.npm \
    npm install

# Mounting secrets during build
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm install

# Using SSH during build for private repos
RUN --mount=type=ssh \
    git clone git@github.com:user/private-repo.git
```

Building with BuildKit:

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Build with secret
docker build --secret id=npmrc,src=.npmrc -t myapp .

# Build with SSH agent
docker build --ssh default -t myapp .
```

### Healthchecks

Add container health monitoring:

```dockerfile
FROM nginx:alpine

COPY app/ /usr/share/nginx/html/
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

EXPOSE 80
```

### Using ONBUILD

Create template images that run commands when used as a base:

```dockerfile
# Dockerfile for template image
FROM node:14-alpine
WORKDIR /app
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
ONBUILD COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

Child image using this template:

```dockerfile
# This will trigger the ONBUILD commands
FROM my-node-template
# No need to specify COPY or npm install
```

### Using SHELL

Change the default shell:

```dockerfile
# Default shell is ["/bin/sh", "-c"]
# Change to bash
SHELL ["/bin/bash", "-c"]

# Now RUN commands use bash
RUN source ~/.bashrc && echo $PATH
```

## Image Security

### Security Scanning

Scan images for vulnerabilities:

```bash
# Using Docker Scan
docker scan myapp:latest

# Using Trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest
```

### Distroless Images

Use minimal images without package managers or shells:

```dockerfile
# Build stage
FROM golang:1.17 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app

# Final stage uses distroless
FROM gcr.io/distroless/static
COPY --from=builder /app/app /
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

### Non-Root Users

Always run as a non-root user:

```dockerfile
FROM node:14-alpine

WORKDIR /app
COPY . .
RUN npm ci --only=production

# Create and use non-root user
RUN addgroup -g 1001 appuser && \
    adduser -u 1001 -G appuser -s /bin/sh -D appuser && \
    chown -R appuser:appuser /app

USER appuser
CMD ["node", "index.js"]
```

### Pinning Package Versions

Explicitly specify versions for reproducibility and security:

```dockerfile
FROM ubuntu:20.04

RUN apt-get update && apt-get install -y \
    curl=7.68.0-1ubuntu2 \
    nginx=1.18.0-0ubuntu1 \
    && rm -rf /var/lib/apt/lists/*
```

### Using COPY Instead of ADD

Prefer COPY for predictability:

```dockerfile
# Avoid
ADD https://example.com/file.tar.gz /tmp/

# Better approach
RUN curl -fsSL https://example.com/file.tar.gz -o /tmp/file.tar.gz && \
    tar -xzf /tmp/file.tar.gz -C /tmp && \
    rm /tmp/file.tar.gz
```

### Using .dockerignore for Sensitive Files

Prevent inclusion of sensitive files:

```
# .dockerignore
.env
.git
credentials/
*.key
*.pem
```

## Working with Container Registries

### Pushing Images to Docker Hub

```bash
# Login to Docker Hub
docker login

# Tag image
docker tag myapp:latest username/myapp:latest

# Push image
docker push username/myapp:latest
```

### Using Private Registries

```bash
# Login to private registry
docker login registry.example.com

# Tag for private registry
docker tag myapp:latest registry.example.com/myapp:latest

# Push to private registry
docker push registry.example.com/myapp:latest
```

### Multi-Architecture Images

Build images for multiple platforms:

```bash
# Create and use builder with BuildKit
docker buildx create --name mybuilder --use

# Build and push for multiple architectures
docker buildx build --platform linux/amd64,linux/arm64 \
  -t username/myapp:latest --push .
```

## Docker Image Versioning

### Tagging Strategies

Common tagging patterns:

1. **Semantic Versioning**:
   ```bash
   docker build -t myapp:1.0.0 .
   docker tag myapp:1.0.0 myapp:1.0
   docker tag myapp:1.0.0 myapp:1
   docker tag myapp:1.0.0 myapp:latest
   ```

2. **Git-Based**:
   ```bash
   docker build -t myapp:$(git rev-parse --short HEAD) .
   ```

3. **Date-Based**:
   ```bash
   docker build -t myapp:$(date +%Y%m%d) .
   ```

4. **Environment**:
   ```bash
   docker build -t myapp:dev .
   docker build -t myapp:staging .
   docker build -t myapp:prod .
   ```

### Image Labels

Add metadata to images:

```dockerfile
FROM node:14-alpine

LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="Example application" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.created="2023-01-01T00:00:00Z" \
      org.opencontainers.image.authors="dev@example.com" \
      org.opencontainers.image.url="https://github.com/example/myapp" \
      org.opencontainers.image.vendor="Example Corp" \
      org.opencontainers.image.licenses="MIT"
```

View labels:

```bash
docker inspect --format '{{json .Config.Labels}}' myapp:latest | jq
```

## Automated Image Building

### GitHub Actions

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      
      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v3
        with:
          images: username/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=ref,event=branch
      
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - build

docker-build:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        TAG="latest"
      else
        TAG="$CI_COMMIT_REF_SLUG"
      fi
    - docker build -t $CI_REGISTRY_IMAGE:$TAG .
    - docker push $CI_REGISTRY_IMAGE:$TAG
```

### Docker Hub Automated Builds

Configure Docker Hub repository to automatically build when changes are pushed to GitHub or Bitbucket.

## Debugging Image Builds

### Interactive Builds

Debug build failures:

```bash
# Start container from last successful stage
docker build --target=build -t myapp:debug .
docker run -it myapp:debug sh

# Or directly enter build process
docker build --progress=plain -t myapp .
```

### Build Arguments for Debugging

```dockerfile
FROM alpine:3.14

ARG DEBUG=false

RUN if [ "$DEBUG" = "true" ]; then \
        echo "Installing debug tools" && \
        apk add --no-cache curl vim procps; \
    fi

# Rest of Dockerfile
```

Build with debugging:

```bash
docker build --build-arg DEBUG=true -t myapp:debug .
```

### Inspecting Build Cache

```bash
# Show layers
docker history myapp:latest

# Use dive tool for deeper analysis
docker run --rm -it wagoodman/dive myapp:latest
```

## Best Practices Checklist

1. **Base Image Selection**:
   - [ ] Use official images when possible
   - [ ] Choose minimal base images (alpine, slim, distroless)
   - [ ] Pin specific version tags

2. **Layer Optimization**:
   - [ ] Minimize layer count
   - [ ] Order instructions from least to most frequently changing
   - [ ] Combine related commands into a single RUN instruction

3. **Security**:
   - [ ] Run as non-root user
   - [ ] Scan images for vulnerabilities
   - [ ] Use .dockerignore to prevent including sensitive files
   - [ ] Set appropriate file permissions

4. **Maintainability**:
   - [ ] Include meaningful comments
   - [ ] Use ARG for build-time customization
   - [ ] Add labels for metadata
   - [ ] Document exposed ports

5. **Multi-Stage Builds**:
   - [ ] Separate build and runtime environments
   - [ ] Only copy necessary artifacts to final image
   - [ ] Use appropriate base images for each stage

6. **Package Management**:
   - [ ] Clear package manager caches
   - [ ] Pin package versions
   - [ ] Remove build dependencies

7. **Configuration**:
   - [ ] Use environment variables for configuration
   - [ ] Implement healthchecks
   - [ ] Set appropriate defaults

## Conclusion

Building efficient Docker images is both an art and a science. By following the best practices outlined in this chapter, you can create images that are:

- **Smaller**: Reducing storage costs and transfer times
- **Faster**: Building and deploying more quickly
- **More secure**: Minimizing attack surface and vulnerabilities
- **More maintainable**: Making updates and debugging easier

As you continue your container journey, these image building skills will form the foundation for creating reliable, scalable, and secure containerized applications.

In the next chapter, we'll explore how to orchestrate multiple containers with Docker Compose, allowing you to define and manage multi-container applications.

## Further Reading

- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Build Command Reference](https://docs.docker.com/engine/reference/commandline/build/)
- [Multi-Stage Builds](https://docs.docker.com/develop/develop-images/multistage-build/)
- [BuildKit Documentation](https://docs.docker.com/develop/develop-images/build_enhancements/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)