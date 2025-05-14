# Docker Installation and Setup

This chapter provides a comprehensive guide to installing and setting up Docker across various operating systems, configuring common settings, and preparing your environment for container development.

## Understanding Docker Editions

Docker offers different editions to meet various needs:

### Docker Engine

Docker Engine is the core open-source container runtime:

- Community-supported container platform
- Available for Linux, Windows, and macOS
- Foundation for all Docker deployments
- Free and open-source

### Docker Desktop

Docker Desktop is an easy-to-install application for Mac and Windows:

- Includes Docker Engine, Docker CLI, Docker Compose, and more
- Provides a GUI for managing containers
- Simplified Kubernetes integration
- Volume mounting and file sharing with host
- Available in both free and subscription tiers

## System Requirements

### Linux Requirements

For Linux installations:

- 64-bit kernel version 3.10 or higher
- systemd or another init system
- 4GB of RAM (minimum)
- Supported distributions:
  - Ubuntu 20.04 (LTS) or later
  - Debian 10 or later
  - Fedora 32 or later
  - CentOS/RHEL 7/8/9
  - SUSE Linux Enterprise 15 or later

### Windows Requirements

For Windows installations:

- Windows 10 64-bit: Pro, Enterprise, or Education (Build 18362 or later)
- Windows 11 64-bit: Pro, Enterprise, or Education
- WSL 2 enabled (preferred) or Hyper-V enabled
- 4GB of RAM (minimum)
- CPU with hardware virtualization support

### macOS Requirements

For macOS installations:

- macOS 11 (Big Sur) or later
- Intel or Apple Silicon processor
- 4GB of RAM (minimum)
- VirtualBox not running (conflicts with hypervisor)

## Installing Docker on Linux

### Ubuntu Installation

Install Docker on Ubuntu using the repository:

```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Verify installation
sudo docker run hello-world
```

### CentOS/RHEL Installation

Install Docker on CentOS/RHEL using the repository:

```bash
# Install required packages
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
sudo docker run hello-world
```

### Debian Installation

Install Docker on Debian:

```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's GPG key
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Verify installation
sudo docker run hello-world
```

### Installation Script Method

Docker provides a convenience script for testing and development:

```bash
# Download and run installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group (optional)
sudo usermod -aG docker $USER
```

> **Note**: The convenience script is not recommended for production environments.

## Installing Docker on Windows

### Docker Desktop for Windows

1. Download Docker Desktop for Windows from [Docker Hub](https://hub.docker.com/editions/community/docker-ce-desktop-windows/)
2. Double-click the installer and follow the instructions
3. When prompted, select either WSL 2 backend or Hyper-V backend
4. Allow the installer to log you out if requested
5. Log back in and start Docker Desktop
6. Verify installation by running:

```powershell
docker run hello-world
```

### WSL 2 Backend Configuration

For optimal performance on Windows, configure the WSL 2 backend:

1. Enable WSL 2 by running as Administrator:

```powershell
# Enable WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Enable Virtual Machine Platform
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Restart your computer
```

2. Download and install the [WSL 2 Linux kernel update package](https://aka.ms/wsl2kernel)

3. Set WSL 2 as default:

```powershell
wsl --set-default-version 2
```

4. Configure Docker Desktop to use WSL 2 backend in Settings

## Installing Docker on macOS

### Docker Desktop for Mac

1. Download Docker Desktop for Mac from [Docker Hub](https://hub.docker.com/editions/community/docker-ce-desktop-mac/)
2. Drag the Docker.app to your Applications folder
3. Open Docker.app from Applications
4. When prompted, provide your administrator password
5. Verify installation by running:

```bash
docker run hello-world
```

### Apple Silicon Configuration

For Macs with Apple Silicon:

1. Download the Apple Silicon version of Docker Desktop
2. Installation is the same as Intel version
3. Enable Rosetta 2 emulation for compatibility with x86 container images:

```bash
softwareupdate --install-rosetta
```

4. Configure resource limits appropriate for Apple Silicon in Docker Desktop settings

## Post-Installation Configuration

### Managing Docker as a Non-Root User

On Linux, add your user to the Docker group to avoid using sudo:

```bash
# Create docker group (if it doesn't exist)
sudo groupadd docker

# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply the new group membership
newgrp docker

# Verify you can run docker without sudo
docker run hello-world
```

### Configuring Docker to Start on Boot

Configure Docker to start automatically on system boot:

```bash
# On systemd-based systems (Ubuntu, Debian, CentOS 7+)
sudo systemctl enable docker

# On SysVinit systems
sudo chkconfig docker on
```

### Storage Driver Configuration

Docker supports different storage drivers with different characteristics:

| Storage Driver | Description | Pros | Cons |
|----------------|-------------|------|------|
| overlay2 | Uses OverlayFS | Fast, stable, recommended | Requires kernel 4.0+ |
| devicemapper | Uses Device Mapper | Good for older kernels | Slower than overlay2 |
| btrfs | Uses Btrfs filesystem | Snapshots, fast | Requires Btrfs filesystem |
| zfs | Uses ZFS filesystem | Advanced features | High memory requirements |
| vfs | Simple filesystem | Works everywhere | Very slow, no layers |

Configure the storage driver in `/etc/docker/daemon.json`:

```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

### Registry Mirror Configuration

Configure registry mirrors to improve pull performance:

```json
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://registry-1.docker.io"
  ]
}
```

### Logging Configuration

Configure container logging drivers:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Available log drivers:

- json-file (default)
- local
- syslog
- journald
- splunk
- awslogs
- fluentd
- gcplogs

### Resource Limits

Configure resource limits for Docker daemon:

```json
{
  "default-shm-size": "64M",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

### Docker Daemon Configuration

Full example of `/etc/docker/daemon.json`:

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "registry-mirrors": [
    "https://mirror.gcr.io"
  ],
  "bip": "172.17.0.1/16",
  "default-address-pools": [
    {"base": "172.18.0.0/16", "size": 24}
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "insecure-registries": [],
  "data-root": "/var/lib/docker",
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "metrics-addr": "0.0.0.0:9323"
}
```

### Applying Configuration Changes

After changing daemon.json, restart Docker:

```bash
# On systemd-based systems
sudo systemctl restart docker

# On macOS and Windows
Restart Docker Desktop application
```

## Docker Compose Installation

Docker Compose is a tool for defining and running multi-container applications:

### Linux Installation

```bash
# Download the current stable release
sudo curl -L "https://github.com/docker/compose/releases/download/v2.15.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Apply executable permissions
sudo chmod +x /usr/local/bin/docker-compose

# Create a symbolic link
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Verify installation
docker-compose --version
```

### Windows/macOS Installation

Docker Compose comes pre-installed with Docker Desktop.

### Compose Plugin for Docker CLI (v2)

Newer Docker versions support Compose as a plugin:

```bash
# Using Compose plugin
docker compose up -d
```

## Docker in Production

### Security Best Practices

Secure your Docker installation:

1. **Keep Docker updated**:
   ```bash
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   ```

2. **Run containers with least privilege**:
   ```bash
   docker run --user 1000:1000 --cap-drop=ALL --security-opt=no-new-privileges alpine
   ```

3. **Use Docker Bench for Security**:
   ```bash
   docker run -it --net host --pid host --userns host --cap-add audit_control \
     -v /var/lib:/var/lib \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v /etc:/etc \
     -v /usr/bin/containerd:/usr/bin/containerd \
     -v /usr/bin/runc:/usr/bin/runc \
     docker/docker-bench-security
   ```

4. **Scan images for vulnerabilities**:
   ```bash
   docker scan mysql:latest
   ```

### Resource Limits for Containers

Always set resource limits for containers:

```bash
docker run -d --name app --memory="512m" --cpus="1.0" nginx
```

### Monitoring and Logging

Configure monitoring tools:

1. **Enable Docker's metrics endpoint**:
   ```json
   {
     "metrics-addr": "0.0.0.0:9323"
   }
   ```

2. **Configure Prometheus for Docker metrics**:
   ```yaml
   scrape_configs:
     - job_name: 'docker'
       static_configs:
         - targets: ['host:9323']
   ```

3. **Use Fluentd for centralized logging**:
   ```json
   {
     "log-driver": "fluentd",
     "log-opts": {
       "fluentd-address": "fluentdhost:24224"
     }
   }
   ```

## Network Configuration

### Default Bridge Network

Configure the default bridge network:

```json
{
  "bip": "172.17.0.1/16",
  "fixed-cidr": "172.17.0.0/16",
  "mtu": 1500
}
```

### Custom Bridge Networks

Create custom bridge networks for better isolation:

```bash
docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 app-network
```

### Host and Macvlan Networks

For advanced networking:

```bash
# Host network (use host's network stack)
docker run --network host nginx

# Macvlan network (assign MAC address to container)
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 macvlan-net

docker run --network macvlan-net nginx
```

## Volume Configuration

### Default Docker Volumes

Volumes are stored in:
- Linux: `/var/lib/docker/volumes`
- Windows: `C:\ProgramData\Docker\volumes`
- macOS: In the VM used by Docker Desktop

### Bind Mounts vs Volumes

```bash
# Docker volume
docker volume create my-vol
docker run -v my-vol:/app/data nginx

# Bind mount
docker run -v /host/path:/container/path nginx
```

### Volume Drivers

For distributed storage:

```bash
# Create a volume using the local driver
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/share \
  nfs-volume
```

## Docker Context

Docker context allows managing multiple Docker endpoints:

```bash
# Create a new context for a remote Docker host
docker context create remote-server --docker "host=ssh://user@remote-host"

# List available contexts
docker context ls

# Switch to a different context
docker context use remote-server

# Run commands against the active context
docker ps
```

## Troubleshooting Installation Issues

### Docker Service Not Starting

If Docker won't start:

```bash
# Check Docker service status
sudo systemctl status docker

# View Docker logs
sudo journalctl -u docker

# Check for conflicting software
sudo lsof -i :2375
sudo lsof -i :2376
```

### Permission Denied Errors

Fix socket permission issues:

```bash
# Check if your user is in the docker group
groups

# If not, add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Docker Desktop WSL 2 Issues

Troubleshoot WSL 2 backend problems:

```powershell
# Check WSL version
wsl -l -v

# Update WSL kernel
wsl --update

# Convert existing Linux distributions to WSL 2
wsl --set-version Ubuntu-20.04 2
```

### Registry Connection Issues

Fix registry connection problems:

```bash
# Check DNS resolution
nslookup registry-1.docker.io

# Test connectivity
curl -v https://registry-1.docker.io/v2/

# Configure Docker to use alternate DNS
echo '{
  "dns": ["8.8.8.8", "8.8.4.4"]
}' | sudo tee /etc/docker/daemon.json

sudo systemctl restart docker
```

## Uninstalling Docker

### Uninstall on Linux

```bash
# Ubuntu/Debian
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# CentOS/RHEL
sudo yum remove docker-ce docker-ce-cli containerd.io
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

### Uninstall on Windows

1. Open Settings > Apps > Apps & features
2. Find Docker Desktop and click Uninstall
3. Delete Docker Desktop data directory (optional):
   ```
   %ProgramData%\Docker
   ```

### Uninstall on macOS

1. Open Applications folder
2. Drag Docker.app to Trash
3. Delete additional data (optional):
   ```bash
   rm -rf ~/Library/Group\ Containers/group.com.docker
   rm -rf ~/Library/Containers/com.docker.docker
   rm -rf ~/.docker
   ```

## Version Management

### Downgrading Docker

Downgrade to a specific version:

```bash
# List available versions
apt-cache madison docker-ce

# Install specific version
sudo apt-get install docker-ce=5:20.10.12~3-0~ubuntu-focal docker-ce-cli=5:20.10.12~3-0~ubuntu-focal containerd.io
```

### Pinning Docker Version

Prevent automatic upgrades:

```bash
# Ubuntu/Debian
sudo apt-mark hold docker-ce docker-ce-cli containerd.io

# CentOS/RHEL
sudo yum versionlock docker-ce docker-ce-cli containerd.io
```

## Docker in CI/CD Environments

### GitHub Actions Example

Use Docker in GitHub Actions:

```yaml
name: Docker CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          push: false
          tags: myapp:latest
```

### GitLab CI Example

Use Docker in GitLab CI:

```yaml
build:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t myapp:latest .
    - docker run myapp:latest test
```

### Jenkins Pipeline Example

Use Docker in Jenkins:

```groovy
pipeline {
    agent {
        docker {
            image 'node:14'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'npm test'
            }
        }
    }
}
```

## Conclusion

Docker installation varies across different platforms, but the core functionality remains consistent. Whether you're running Docker on Linux, Windows, or macOS, understanding the proper installation and configuration ensures a smooth experience managing containers.

After setting up Docker, focus on security practices, resource management, and integration with your development workflow to get the most out of containerization.

In the next chapter, we'll explore the Docker CLI fundamentals to help you start working with containers and images efficiently.

## Further Reading

- [Official Docker Installation Documentation](https://docs.docker.com/engine/install/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Docker Configuration Reference](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
- [Docker in Production Best Practices](https://docs.docker.com/config/containers/start-containers-automatically/)