# Image Security in Kubernetes

## Introduction

Container image security is a critical component of Kubernetes security. Since containers are built from images that include the application code, dependencies, and sometimes configuration, securing these images is essential to maintaining a secure Kubernetes environment. This document covers best practices for securing container images, scanning for vulnerabilities, and implementing proper controls throughout the image lifecycle.

## Image Security Fundamentals

### The Container Image Supply Chain

The container image supply chain includes:

1. **Base Images**: The foundation of your container
2. **Dependencies**: Libraries and packages added to your image
3. **Application Code**: Your custom application code
4. **Build Process**: How the image is built and published
5. **Distribution**: How images are stored and distributed
6. **Runtime**: Where images are executed as containers

Each stage presents security risks that must be addressed.

## Building Secure Container Images

### Minimal Base Images

Use minimal, purpose-built base images to reduce the attack surface:

```dockerfile
# Avoid general-purpose images
FROM ubuntu:latest  # NOT RECOMMENDED - large attack surface

# Prefer minimal images
FROM alpine:3.18  # BETTER - smaller but still has package manager

# Best: Distroless images when possible
FROM gcr.io/distroless/static-debian11  # BEST - minimal attack surface
```

### Best Practices for Dockerfile Security

1. **Use specific image tags** rather than `latest` to ensure reproducibility:

   ```dockerfile
   FROM nginx:1.25.1-alpine  # Specific version
   ```

2. **Run containers as non-root users**:

   ```dockerfile
   # Create a user
   RUN addgroup -S appgroup && adduser -S appuser -G appgroup
   
   # Set working directory and ownership
   WORKDIR /app
   COPY --chown=appuser:appgroup . .
   
   # Switch to non-root user
   USER appuser
   ```

3. **Multi-stage builds** to reduce image size and attack surface:

   ```dockerfile
   # Build stage
   FROM golang:1.20-alpine AS builder
   WORKDIR /app
   COPY . .
   RUN CGO_ENABLED=0 go build -o myapp
   
   # Final stage
   FROM alpine:3.18
   RUN adduser -D appuser
   USER appuser
   COPY --from=builder /app/myapp /usr/local/bin/
   CMD ["myapp"]
   ```

4. **Remove unnecessary tools** and development dependencies:

   ```dockerfile
   # Install dependencies, use them, then remove unneeded packages
   RUN apk add --no-cache build-base python3-dev && \
       pip install --no-cache-dir -r requirements.txt && \
       apk del build-base python3-dev
   ```

5. **Set appropriate filesystem permissions**:

   ```dockerfile
   # Make application directory with correct permissions
   RUN mkdir -p /app && \
       chown -R appuser:appgroup /app && \
       chmod -R 750 /app
   ```

## Image Vulnerability Scanning

### Tools for Image Scanning

1. **Trivy**: A comprehensive, easy-to-use vulnerability scanner
   ```bash
   # Scan an image with Trivy
   trivy image nginx:1.25.1-alpine
   ```

2. **Clair**: CoreOS's vulnerability static analyzer for containers
   ```bash
   # Using clairctl
   clairctl analyze --addr http://clair:6060 --format json nginx:1.25.1-alpine
   ```

3. **Anchore Engine**: Deep analysis of container images
   ```bash
   # Using Anchore CLI
   anchore-cli image add docker.io/library/nginx:1.25.1-alpine
   anchore-cli image wait docker.io/library/nginx:1.25.1-alpine
   anchore-cli image vulns docker.io/library/nginx:1.25.1-alpine
   ```

4. **Snyk**: Developer-centric security tool with free and commercial options

### Integrating Scanning into CI/CD

Example GitHub Actions workflow for image scanning:

```yaml
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
        run: docker build -t my-app:${{ github.sha }} .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

## Container Image Registries

### Secure Registry Best Practices

1. **Use private registries** for all production images
2. **Implement access controls** with role-based permissions
3. **Enable vulnerability scanning** in your registry
4. **Enforce image signing** for authenticity verification
5. **Set retention policies** to manage old image versions

### Container Registry Security Features

Common features in enterprise-grade registries:

| Feature | Description |
|---------|-------------|
| Authentication | Multi-factor authentication, OIDC/SAML integration |
| Authorization | RBAC for image access and management |
| Vulnerability Scanning | Automated scanning and reporting |
| Image Signing | Digital signatures to verify authenticity |
| Policy Enforcement | Admission controllers to enforce security policies |
| Retention Policies | Auto-cleanup of unused or old images |

## Image Admission Controllers

### Preventing Vulnerable Images in Kubernetes

1. **OPA Gatekeeper**: Define policies for image admission

   Example policy to require images from trusted registry:
   ```yaml
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: K8sTrustedRegistry
   metadata:
     name: require-trusted-registry
   spec:
     match:
       kinds:
         - apiGroups: [""]
           kinds: ["Pod"]
       namespaces: ["default", "app"]
     parameters:
       registries: ["gcr.io/my-org", "registry.example.com"]
   ```

2. **Kyverno**: Policy engine designed for Kubernetes

   Example policy to verify image signatures:
   ```yaml
   apiVersion: kyverno.io/v1
   kind: ClusterPolicy
   metadata:
     name: verify-image-signatures
   spec:
     validationFailureAction: enforce
     rules:
     - name: check-signatures
       match:
         resources:
           kinds:
           - Pod
       validate:
         message: "Image signature verification failed"
         image:
           verifyImages:
           - image: "registry.example.com/*"
             key: "cosign.pub"
             repository: "registry.example.com/signatures"
   ```

## Image Signing and Attestation

### Cosign for Image Signing

```bash
# Generate a keypair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key registry.example.com/my-app:1.0.0

# Verify an image
cosign verify --key cosign.pub registry.example.com/my-app:1.0.0
```

### Software Bill of Materials (SBOM)

An SBOM is a formal record of the components used in building software, providing transparency about dependencies:

```bash
# Generate SBOM with Syft
syft alpine:latest -o spdx-json > alpine-sbom.json

# Attach SBOM to image with Cosign
cosign attach sbom --sbom alpine-sbom.json alpine:latest

# Verify and retrieve SBOM
cosign verify-attestation --key cosign.pub alpine:latest
```

## Runtime Image Security

### Controlling What Can Run

1. **Pod Security Standards**: Enforce baseline security practices
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: restricted-ns
     labels:
       pod-security.kubernetes.io/enforce: restricted
   ```

2. **Image Pull Policies**: Control image deployment behavior
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secure-pod
   spec:
     containers:
     - name: app
       image: registry.example.com/my-app:1.0.0
       imagePullPolicy: Always  # Always pull the latest of the specified tag
   ```

3. **Read-Only Root Filesystem**: Prevent runtime filesystem modifications
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: readonly-pod
   spec:
     containers:
     - name: app
       image: registry.example.com/my-app:1.0.0
       securityContext:
         readOnlyRootFilesystem: true
   ```

## Real-World Implementation

### Implementing Image Security in an Enterprise Environment

1. **Define Image Security Policy**:
   - Approved base images and registries
   - Vulnerability severity thresholds
   - Required security annotations/metadata
   - SBOM and signature requirements

2. **Automate Security Checks**:
   - CI/CD pipeline integration
   - Pre-build dependency scanning
   - Post-build image scanning
   - Signature verification in production

3. **Establish Continuous Monitoring**:
   - Regular vulnerability rescanning
   - Runtime security monitoring
   - Event-driven rescanning on new CVE discovery

### Example: End-to-End Secure Image Workflow

1. **Development**:
   - Developer creates Dockerfile with security best practices
   - Pre-commit hooks check for obvious security issues

2. **CI/CD Pipeline**:
   - Build image with proper tagging
   - Scan for vulnerabilities
   - Fail build if critical/high vulnerabilities found
   - Sign image and generate attestations

3. **Registry Storage**:
   - Store in secure registry with access controls
   - Enable registry vulnerability scanning

4. **Deployment**:
   - Admission controller verifies registry source
   - Validate image signatures
   - Check vulnerability status
   - Apply Pod Security Context

5. **Runtime**:
   - Apply runtime security monitoring
   - Schedule regular rescans
   - Track CVE databases for new vulnerabilities

## Best Practices Summary

1. **Use minimal base images** to reduce attack surface
2. **Build with the principle of least privilege** (non-root users, minimal capabilities)
3. **Version pin everything** - base images, dependencies, application versions
4. **Scan early and often** for vulnerabilities
5. **Implement image signing** for provenance and verification
6. **Store SBOMs** for dependency transparency
7. **Use private registries** with access controls
8. **Implement admission controls** to enforce policies
9. **Keep images updated** to address new vulnerabilities
10. **Automate the security process** via CI/CD integration

## Conclusion

Image security is a critical foundation for Kubernetes security. By implementing proper controls at each stage of the image lifecycle—from building and scanning to signing, distributing, and running—organizations can significantly reduce their attack surface and minimize the risk of compromise. A well-designed image security strategy, combined with other Kubernetes security measures, provides a strong defense-in-depth approach to container security.

## Additional Resources

- [Container Security Best Practices](https://github.com/docker/docker-bench-security)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/latest/)
- [Cosign Project](https://github.com/sigstore/cosign)
- [NIST Container Security Guidelines](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)
- [Kubernetes Image Policy Webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)
- [Open Container Initiative (OCI) Standards](https://opencontainers.org/)