# Docker Security

Security is a critical aspect of container deployments. This guide covers Docker security fundamentals, best practices, and tools to help you build and run secure containers.

## Docker Security Architecture

Docker's security architecture consists of several components:

- **Linux namespaces**: Isolate container processes from the host and each other
- **Control groups (cgroups)**: Limit resources available to containers
- **Linux capabilities**: Restrict privileged operations
- **Seccomp profiles**: Filter system calls available to containers
- **AppArmor/SELinux**: Mandatory Access Control systems

## Container Isolation

Containers provide process-level isolation, not VM-level isolation:

- Containers share the host kernel
- Root in container is (by default) root on the host
- Container breakout is possible with misconfigurations

## Non-Root Containers

A critical security best practice is running containers as non-root users:

```dockerfile
FROM ubuntu:20.04

# Create a non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory permissions
WORKDIR /app
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

CMD ["./app"]
```

## Reducing Container Privileges

### Use the `--security-opt` flag

```bash
# Run with a custom seccomp profile
docker run --security-opt seccomp=/path/to/seccomp.json nginx

# Run with no new privileges flag
docker run --security-opt no-new-privileges nginx
```

### Drop Linux capabilities

```bash
# Run with minimal capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

Common capabilities to consider:
- `NET_BIND_SERVICE`: Bind to ports below 1024
- `CHOWN`: Change file ownership
- `DAC_OVERRIDE`: Bypass file read/write/execute permission checks
- `SETFCAP`: Set file capabilities
- `SYS_PTRACE`: Trace processes (needed for debugging tools)

## Docker Content Trust (DCT)

Docker Content Trust provides the ability to verify image publisher identity and ensure image integrity:

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Sign and push an image
docker push mycompany/myimage:1.0
```

## Image Security

### Minimize Image Size

Use minimal base images:
- Alpine-based images
- Distroless images
- Scratch-based images

```dockerfile
# Instead of full Ubuntu
FROM ubuntu:20.04

# Use smaller Alpine
FROM alpine:3.14

# Or even smaller distroless
FROM gcr.io/distroless/static-debian11
```

### Multi-stage Builds

Use multi-stage builds to reduce final image size and attack surface:

```dockerfile
# Build stage
FROM golang:1.17 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Final minimal stage
FROM alpine:3.14
COPY --from=builder /app/myapp /usr/local/bin/
USER nobody
ENTRYPOINT ["myapp"]
```

### Image Scanning

Scan images for vulnerabilities before deployment:

```bash
# Using Docker Scout
docker scout quickview myimage:latest

# Using Trivy
trivy image myimage:latest

# Using Clair
docker run -p 5432:5432 -d --name db arminc/clair-db
docker run -p 6060:6060 --link db:postgres -d --name clair arminc/clair-local-scan
```

## Runtime Security

### Read-only Filesystem

```bash
docker run --read-only nginx
```

Add temporary folders if needed:
```bash
docker run --read-only --tmpfs /tmp nginx
```

### Memory and CPU Limits

```bash
docker run --memory=512m --cpus=0.5 nginx
```

### PID Limits

```bash
docker run --pids-limit=100 nginx
```

## Network Security

### Use User-defined Networks

```bash
# Create a custom network
docker network create --driver bridge app-network

# Run containers in that network
docker run --network app-network --name db postgres
docker run --network app-network --name web nginx
```

### Restrict External Access

```bash
# Don't publish ports unnecessarily
docker run --network app-network redis
```

### Use Network Isolation

```bash
# Create isolated networks for different application tiers
docker network create frontend
docker network create backend

# Web server accessible from outside
docker run --network frontend -p 80:80 nginx

# Database only accessible from backend network
docker run --network backend postgres
```

## Secret Management

Avoid hardcoding secrets in Docker images:

```bash
# Using Docker secrets (Swarm mode)
docker secret create db_password /path/to/password.txt
docker service create --secret db_password postgres

# Environment variables (less secure)
docker run -e DB_PASSWORD=mypassword postgres

# Bind mount secrets at runtime (development)
docker run -v /path/to/secrets:/run/secrets nginx
```

## Docker Daemon Security

### Secure the Docker daemon socket

The Docker socket is a significant potential attack vector:

```bash
# Don't do this in production
docker run -v /var/run/docker.sock:/var/run/docker.sock nginx
```

### TLS for Docker daemon

Configure Docker to use TLS:
```bash
# Server (daemon) configuration
dockerd --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=server-cert.pem \
  --tlskey=server-key.pem \
  -H=0.0.0.0:2376

# Client configuration
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=cert.pem \
  --tlskey=key.pem \
  -H=tcp://server:2376 info
```

## Security Audit and Compliance

### Docker Bench Security

Docker Bench is a script that checks for common best practices around deploying Docker containers in production:

```bash
docker run -it --net host --pid host --userns host --cap-add audit_control \
    -v /var/lib:/var/lib \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /usr/lib/systemd:/usr/lib/systemd \
    -v /etc:/etc --label docker_bench_security \
    docker/docker-bench-security
```

### CIS Docker Benchmark

The Center for Internet Security (CIS) provides a benchmark for Docker security best practices:

- Installation and configuration
- Docker daemon configuration
- Docker daemon files
- Container images and build file
- Container runtime
- Docker security operations
- Docker enterprise configuration

## Security Tools Ecosystem

- **Falco**: Runtime security monitoring 
- **Anchore**: Image scanning and policy enforcement
- **Aqua Security**: Container security platform
- **Sysdig Secure**: Container security and forensics
- **NeuVector**: Layer 7 container firewall

## Security Best Practices Checklist

1. ✅ Run containers as non-root users
2. ✅ Use minimal base images
3. ✅ Apply the principle of least privilege (capabilities, seccomp)
4. ✅ Scan images for vulnerabilities
5. ✅ Sign and verify images
6. ✅ Use read-only containers
7. ✅ Set resource limits for containers
8. ✅ Secure the Docker daemon socket
9. ✅ Use user-defined networks
10. ✅ Manage secrets securely
11. ✅ Apply security patches regularly
12. ✅ Implement runtime security monitoring
13. ✅ Audit Docker security regularly

By applying these security measures, you can significantly reduce the attack surface of your Docker containers and create a more secure containerized environment.