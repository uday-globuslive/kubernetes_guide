# Docker CLI Fundamentals

This chapter provides a comprehensive guide to the Docker Command Line Interface (CLI), covering essential commands, workflows, and best practices for working with containers and images.

## Docker CLI Overview

The Docker CLI is the primary interface for interacting with Docker. It communicates with the Docker daemon to perform operations on containers, images, networks, and volumes.

### CLI Structure

Docker CLI commands follow a consistent structure:

```
docker [OPTIONS] COMMAND [SUBCOMMAND] [ARGS...]
```

- **docker**: The base command
- **OPTIONS**: Global options like `-v` (verbose) or `--config`
- **COMMAND**: The main command (e.g., `run`, `build`, `pull`)
- **SUBCOMMAND**: Some commands have subcommands
- **ARGS**: Arguments specific to the command

### Getting Help

Access built-in help for commands:

```bash
# General help
docker --help

# Command-specific help
docker run --help

# Subcommand help
docker container ls --help
```

### Command Categories

Docker commands are organized into management categories:

- **Container Commands**: Managing container lifecycle
- **Image Commands**: Working with container images
- **Network Commands**: Managing container networks
- **Volume Commands**: Working with persistent storage
- **System Commands**: System-wide operations and information
- **Docker Compose**: Multi-container management

## Container Management

### Running Containers

The `docker run` command creates and starts a container:

```bash
# Basic syntax
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

# Run a container in detached mode
docker run -d --name webserver nginx

# Run with interactive terminal
docker run -it --name alpine alpine sh

# Run and remove after exit
docker run --rm alpine echo "Hello, World!"

# Run with specific user
docker run -u 1000:1000 alpine id

# Run with resource limits
docker run --memory=512m --cpus=0.5 nginx

# Run with port mapping
docker run -p 8080:80 nginx

# Run with volume mount
docker run -v /host/path:/container/path nginx

# Run with environment variables
docker run -e DB_HOST=mydb -e DB_PORT=5432 myapp
```

Key `run` options:

| Option | Description |
|--------|-------------|
| `-d, --detach` | Run in background |
| `-i, --interactive` | Keep STDIN open |
| `-t, --tty` | Allocate a pseudo-TTY |
| `--name` | Assign a name |
| `-p, --publish` | Publish container ports |
| `-v, --volume` | Bind mount a volume |
| `-e, --env` | Set environment variables |
| `--rm` | Remove when stopped |
| `--restart` | Restart policy |
| `--network` | Connect to network |

### Container Lifecycle Management

Commands to manage container state:

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Start a container
docker start container_id_or_name

# Stop a container
docker stop container_id_or_name

# Restart a container
docker restart container_id_or_name

# Pause a container
docker pause container_id_or_name

# Unpause a container
docker unpause container_id_or_name

# Kill a container
docker kill container_id_or_name

# Remove a container
docker rm container_id_or_name

# Remove all stopped containers
docker container prune
```

### Container Inspection and Logs

Commands to monitor and debug containers:

```bash
# View container logs
docker logs container_id_or_name

# Follow log output
docker logs -f container_id_or_name

# Show last N lines
docker logs --tail 100 container_id_or_name

# Show container details
docker inspect container_id_or_name

# Show running processes
docker top container_id_or_name

# Show resource usage statistics
docker stats [container_id_or_name]

# Show changes to container filesystem
docker diff container_id_or_name
```

### Executing Commands in Containers

Run commands in running containers:

```bash
# Execute command in running container
docker exec container_id_or_name command

# Interactive shell in running container
docker exec -it container_id_or_name sh

# Run as specific user
docker exec -u root container_id_or_name whoami
```

### Container Resource Management

Monitor and manage container resources:

```bash
# Update container resource limits
docker update --memory=1g --cpus=1 container_id_or_name

# View container resource usage
docker stats container_id_or_name
```

### Container Restart Policies

Set restart behavior for containers:

```bash
docker run --restart=always nginx      # Always restart
docker run --restart=on-failure:5 nginx # Restart up to 5 times on failure
docker run --restart=unless-stopped nginx # Restart unless explicitly stopped
```

## Image Management

### Pulling Images

Download images from registries:

```bash
# Pull latest tag
docker pull nginx

# Pull specific tag
docker pull nginx:1.21

# Pull from custom registry
docker pull registry.example.com/myapp:latest

# Pull from Docker Hub with authentiation
docker login
docker pull myusername/private-image
```

### Listing Images

View local images:

```bash
# List all images
docker images

# List with specific format
docker images --format "{{.Repository}}:{{.Tag}}: {{.Size}}"

# List dangling images
docker images --filter "dangling=true"
```

### Image Tags and Management

Work with image tags:

```bash
# Tag an image
docker tag nginx:latest mynginx:v1

# Remove an image
docker rmi nginx:latest

# Remove dangling images
docker image prune

# Remove all unused images
docker image prune -a
```

### Image Inspection

Examine image details:

```bash
# Show image information
docker inspect nginx:latest

# Show image history (layers)
docker history nginx:latest

# See image manifest
docker manifest inspect nginx:latest
```

### Saving and Loading Images

Transfer images without a registry:

```bash
# Save image to tar file
docker save -o nginx.tar nginx:latest

# Load image from tar file
docker load -i nginx.tar
```

### Pushing Images to Registries

Share images via registries:

```bash
# Login to registry
docker login [registry-url]

# Push image to registry
docker push myusername/myapp:latest

# Push to custom registry
docker push registry.example.com/myapp:latest
```

## Network Management

### Network Basics

Docker networks provide isolation between container environments:

```bash
# List networks
docker network ls

# Network types: bridge, host, overlay, macvlan, none

# Inspect a network
docker network inspect bridge
```

### Creating and Managing Networks

Work with custom networks:

```bash
# Create a bridge network
docker network create my-network

# Create with subnet and gateway
docker network create --subnet=172.18.0.0/16 --gateway=172.18.0.1 my-network

# Connect container to network
docker network connect my-network container_name

# Disconnect container from network
docker network disconnect my-network container_name

# Remove network
docker network rm my-network

# Remove all unused networks
docker network prune
```

### Network Drivers

Docker supports multiple network drivers:

| Driver | Description | Use Case |
|--------|-------------|----------|
| bridge | Default isolated network | Development |
| host | Uses host's network stack | Performance |
| overlay | Multi-host networking | Swarm, Kubernetes |
| macvlan | Assigns MAC address to container | Legacy apps |
| none | No networking | Maximum isolation |

Example of different network modes:

```bash
# Default bridge network
docker run -d nginx

# Host network mode
docker run --network=host nginx

# No networking
docker run --network=none nginx
```

## Volume Management

### Volume Basics

Docker volumes provide persistent storage for containers:

```bash
# List volumes
docker volume ls

# Create a volume
docker volume create my-data

# Inspect a volume
docker volume inspect my-data

# Remove a volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune
```

### Using Volumes with Containers

Mount volumes to containers:

```bash
# Mount a named volume
docker run -v my-data:/app/data nginx

# Bind mount a host directory
docker run -v $(pwd):/app/code nginx

# Read-only mount
docker run -v my-data:/app/data:ro nginx

# Mount with tmpfs (memory)
docker run --tmpfs /app/temp nginx
```

### Volume Drivers

Support for external storage systems:

```bash
# Create NFS volume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  nfs-volume

# Use volume with specific driver
docker run -v nfs-volume:/app/data nginx
```

## System Commands

### System Information

Commands to view Docker system info:

```bash
# Show system-wide information
docker info

# Show version info
docker version

# Show disk usage
docker system df

# Detailed disk usage
docker system df -v
```

### Cleanup Commands

Maintain system resources:

```bash
# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune -a

# Remove all unused volumes
docker volume prune

# Remove all unused networks
docker network prune

# Remove all unused objects (comprehensive cleanup)
docker system prune

# Include volumes in system prune
docker system prune -a --volumes
```

### Events and Monitoring

Monitor Docker events:

```bash
# Show real-time events
docker events

# Filter events
docker events --filter 'type=container'

# Show events for specific time period
docker events --since '2023-01-01' --until '2023-01-02'
```

## Docker Compose Commands

Docker Compose manages multi-container applications:

```bash
# Start services defined in docker-compose.yml
docker-compose up

# Start in detached mode
docker-compose up -d

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# View logs
docker-compose logs

# Scale a service
docker-compose up -d --scale web=3

# List containers
docker-compose ps

# Execute command in service container
docker-compose exec web sh
```

## Advanced CLI Features

### Formatting Output

Customize command output:

```bash
# Format container list
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"

# Format image list
docker images --format "{{.Repository}}:{{.Tag}} ({{.Size}})"

# Format JSON output
docker inspect --format '{{json .Config}}' container_name | jq
```

Available format directives depend on the command.

### Filtering Results

Filter command results:

```bash
# Filter containers by status
docker ps -a --filter "status=exited"

# Filter containers by label
docker ps --filter "label=environment=test"

# Filter images by reference
docker images --filter=reference="nginx:*"

# Multiple filters (AND logic)
docker ps --filter "status=running" --filter "name=web"
```

### Context Management

Work with multiple Docker hosts:

```bash
# List contexts
docker context ls

# Create a new context
docker context create my-remote-host --docker "host=ssh://user@host"

# Use a specific context
docker context use my-remote-host

# Run command with specific context
docker --context=my-remote-host ps
```

## Docker CLI Workflows

### Container Development Workflow

Typical local development workflow:

```bash
# 1. Pull a base image
docker pull python:3.9

# 2. Run interactive container for development
docker run -it --name dev -v $(pwd):/app python:3.9 bash

# 3. Test application
docker exec -it dev python test.py

# 4. Stop and clean up when done
docker stop dev
docker rm dev
```

### Debugging Workflow

Troubleshoot container issues:

```bash
# 1. Check container status
docker ps -a

# 2. View logs
docker logs problematic-container

# 3. Inspect container
docker inspect problematic-container

# 4. Execute diagnostic commands
docker exec -it problematic-container sh

# 5. Check resource usage
docker stats problematic-container
```

### Image Publishing Workflow

Create and share container images:

```bash
# 1. Build image locally
docker build -t myapp:latest .

# 2. Test the image
docker run --rm myapp:latest

# 3. Tag for registry
docker tag myapp:latest username/myapp:latest

# 4. Login to registry
docker login

# 5. Push to registry
docker push username/myapp:latest
```

## Docker CLI Best Practices

### Resource Management

Prevent containers from consuming excessive resources:

```bash
# Set memory limits
docker run --memory=512m nginx

# Set CPU limits
docker run --cpus=0.5 nginx

# Set both with restart on OOM
docker run --memory=512m --cpus=0.5 --restart=on-failure nginx
```

### Container Naming Conventions

Use consistent naming for easier management:

```bash
# Use descriptive names with prefix for environment
docker run --name prod-web-1 nginx
docker run --name dev-db-1 postgres

# Use labels for additional metadata
docker run --label env=prod --label app=web nginx
```

### Security Recommendations

Run containers with minimal privileges:

```bash
# Run as non-root user
docker run -u 1000:1000 nginx

# Drop all capabilities and add only necessary ones
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Read-only filesystem with writable directories
docker run --read-only --tmpfs /tmp nginx
```

### Resource Cleanup

Maintain system resources:

```bash
# Use --rm flag for temporary containers
docker run --rm alpine echo "Hello"

# Create cleanup cron job for pruning
# Example crontab entry:
# 0 0 * * * docker system prune -f
```

## Scripting with Docker CLI

### Using Docker in Shell Scripts

Integrate Docker commands in shell scripts:

```bash
#!/bin/bash
# Example script to backup all container volumes

# Get list of running containers
CONTAINERS=$(docker ps -q)

for CONTAINER in $CONTAINERS; do
  # Get container name
  NAME=$(docker inspect --format '{{.Name}}' $CONTAINER | sed 's/\///')
  
  # Create backup directory
  mkdir -p ./backups/$NAME
  
  # Get volumes
  VOLUMES=$(docker inspect --format '{{range .Mounts}}{{if eq .Type "volume"}}{{.Name}}{{printf " "}}{{end}}{{end}}' $CONTAINER)
  
  for VOLUME in $VOLUMES; do
    echo "Backing up volume $VOLUME from container $NAME"
    docker run --rm -v $VOLUME:/source -v $(pwd)/backups/$NAME:/backup alpine tar -czf /backup/$VOLUME.tar.gz -C /source .
  done
done
```

### Docker CLI in CI/CD Pipelines

Example Jenkins pipeline script:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker tag myapp:${BUILD_NUMBER} myapp:latest'
            }
        }
        
        stage('Test') {
            steps {
                sh 'docker run --rm myapp:latest npm test'
            }
        }
        
        stage('Deploy') {
            steps {
                sh 'docker push myapp:${BUILD_NUMBER}'
                sh 'docker push myapp:latest'
            }
        }
    }
    
    post {
        always {
            sh 'docker rmi myapp:${BUILD_NUMBER} myapp:latest || true'
        }
    }
}
```

## Docker CLI Environment Variables

Docker CLI recognizes several environment variables:

| Variable | Description |
|----------|-------------|
| DOCKER_HOST | Daemon socket to connect to |
| DOCKER_TLS_VERIFY | Use TLS with daemon |
| DOCKER_CERT_PATH | Path to TLS certificates |
| DOCKER_CONFIG | Path to client config |
| DOCKER_CONTEXT | Default context to use |
| DOCKER_API_VERSION | API version to use |
| DOCKER_BUILDKIT | Enable BuildKit for builds |

Example usage:

```bash
# Connect to remote Docker host
export DOCKER_HOST=ssh://user@remote-host
docker ps

# Use specific API version
export DOCKER_API_VERSION=1.41
docker version
```

## CLI Configuration File

Configure Docker CLI behavior with `~/.docker/config.json`:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "base64-encoded-credentials"
    }
  },
  "proxies": {
    "default": {
      "httpProxy": "http://proxy.example.com:8080",
      "httpsProxy": "https://proxy.example.com:8080",
      "noProxy": "localhost,127.0.0.1"
    }
  },
  "credsStore": "desktop",
  "plugins": {
    "cli-plugins": {
      "buildx": {
        "enabled": true
      }
    }
  },
  "experimental": true,
  "buildkit": true
}
```

## CLI Plugins

Docker CLI supports plugins to extend functionality:

```bash
# List available plugins
docker --help | grep -A10 Commands

# List buildx builders
docker buildx ls

# Use buildx to build multi-platform images
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Use scan plugin
docker scan nginx:latest
```

## Troubleshooting Docker CLI

### Common Error Scenarios

| Error | Possible Cause | Solution |
|-------|---------------|----------|
| "Cannot connect to the Docker daemon" | Daemon not running | Start Docker service |
| "permission denied" | User not in docker group | Add user to docker group |
| "port is already allocated" | Port conflict | Use different port or stop conflicting service |
| "no space left on device" | Disk full | Prune unused objects or expand disk |
| "image not found" | Image doesn't exist locally | Pull the image first |

### Diagnostic Commands

Commands to help troubleshoot issues:

```bash
# Check Docker service status
sudo systemctl status docker

# View Docker daemon logs
sudo journalctl -u docker

# Check Docker info
docker info

# Verify Docker connectivity
docker version

# Test Docker functionality
docker run --rm hello-world
```

## Docker CLI vs Docker API

The Docker CLI is a client that communicates with the Docker API:

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ Docker CLI  │──────▶│ Docker API  │──────▶│ Docker      │
│ (Client)    │       │ (REST API)  │       │ Daemon      │
└─────────────┘       └─────────────┘       └─────────────┘
```

Direct API interaction examples:

```bash
# Get list of containers via API
curl --unix-socket /var/run/docker.sock http://localhost/v1.41/containers/json

# Create a container via API
curl -X POST --unix-socket /var/run/docker.sock \
  -H "Content-Type: application/json" \
  -d '{"Image":"nginx", "ExposedPorts": {"80/tcp": {}}}' \
  http://localhost/v1.41/containers/create
```

## Conclusion

The Docker CLI is a powerful tool for managing the complete lifecycle of containers, images, networks, and volumes. Mastering these commands enables you to efficiently work with containerized applications from development to production.

As you become more comfortable with the CLI, you'll discover more advanced features and patterns to integrate Docker into your development workflow, CI/CD pipelines, and production environments.

In the next chapter, we'll explore building Docker images, which is a critical skill for creating customized container environments for your applications.

## Further Reading

- [Docker CLI Documentation](https://docs.docker.com/engine/reference/commandline/cli/)
- [Docker Command Cheat Sheet](https://docs.docker.com/get-started/docker_cheatsheet.pdf)
- [Docker REST API Documentation](https://docs.docker.com/engine/api/)
- [Docker CLI Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security Documentation](https://docs.docker.com/engine/security/)