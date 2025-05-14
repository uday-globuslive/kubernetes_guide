# Docker Networking

## Introduction

Docker networking is a critical component for containerized applications, allowing containers to communicate with each other and with external networks. Understanding Docker networking is essential for deploying and managing Elasticsearch, Logstash, Kibana, and Beats in both standalone Docker environments and Kubernetes.

This guide explores Docker networking concepts, types of networks, configuration options, and best practices, with a focus on the ELK stack.

## Networking Fundamentals

### Docker Network Architecture

Docker uses a pluggable architecture for networking that supports various drivers. The basic components include:

- **Network Namespaces**: Isolated network stacks that provide containers with their own interfaces, routing tables, and firewall rules
- **Virtual Ethernet Devices**: Pairs of virtual interfaces that connect containers to the host
- **Linux Bridge**: A virtual switch that forwards packets between connected interfaces
- **iptables Rules**: NAT and firewall rules for container connectivity and security

### Container Network Model (CNM)

Docker implements the Container Network Model, which consists of:

1. **Sandbox**: Network namespace with interfaces
2. **Endpoint**: Virtual network interface connecting a container to a network
3. **Network**: A group of endpoints that can communicate with each other

## Network Types

Docker provides several built-in network drivers:

### Bridge Networks

The default network type for containers. Containers on the same bridge network can communicate with each other.

```bash
# Create a bridge network
docker network create elk-network --driver bridge

# Run a container on the network
docker run --network=elk-network --name=elasticsearch docker.elastic.co/elasticsearch/elasticsearch:8.8.1
```

### Host Network

Containers share the host's network namespace, providing better performance but less isolation.

```bash
# Run with host networking
docker run --network=host docker.elastic.co/elasticsearch/elasticsearch:8.8.1
```

### None Network

Containers have no external network connectivity.

```bash
# Run with no networking
docker run --network=none alpine sh
```

### Overlay Networks

Enables communication between containers across multiple Docker hosts, essential for Docker Swarm.

```bash
# Create an overlay network (in Swarm mode)
docker network create --driver overlay --attachable elk-overlay-network
```

### Macvlan Networks

Assigns a MAC address to containers, making them appear as physical devices on the network.

```bash
# Create a macvlan network
docker network create --driver macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 elk-macvlan-network
```

## Network Configuration

### Port Mapping

Expose container ports to the host to enable external access:

```bash
# Map port 9200 in the container to port 9200 on the host
docker run -p 9200:9200 docker.elastic.co/elasticsearch/elasticsearch:8.8.1
```

### DNS Configuration

Docker manages DNS resolution for containers:

```bash
# Set custom DNS servers
docker run --dns=8.8.8.8 --dns=8.8.4.4 alpine ping example.com

# Set DNS search domains
docker run --dns-search=example.com alpine ping db
```

### Network Aliases

Create network aliases for service discovery:

```bash
# Create a container with a network alias
docker run --network=elk-network --network-alias=es docker.elastic.co/elasticsearch/elasticsearch:8.8.1

# Connect to the container using the alias
docker run --network=elk-network alpine ping es
```

## ELK Stack Networking

### Elasticsearch Networking

Elasticsearch uses two ports:
- 9200 (HTTP): REST API
- 9300 (TCP): Node-to-node communication

```bash
# Run Elasticsearch with proper network settings
docker run -d \
  --name elasticsearch \
  --network elk-network \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  docker.elastic.co/elasticsearch/elasticsearch:8.8.1
```

### Cluster Communication

For multi-node Elasticsearch clusters, additional network settings are required:

```bash
# Node 1
docker run -d \
  --name es01 \
  --network elk-network \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "node.name=es01" \
  -e "cluster.name=es-cluster" \
  -e "discovery.seed_hosts=es02,es03" \
  -e "cluster.initial_master_nodes=es01,es02,es03" \
  -e "network.host=0.0.0.0" \
  docker.elastic.co/elasticsearch/elasticsearch:8.8.1

# Node 2
docker run -d \
  --name es02 \
  --network elk-network \
  -p 9201:9200 \
  -p 9301:9300 \
  -e "node.name=es02" \
  -e "cluster.name=es-cluster" \
  -e "discovery.seed_hosts=es01,es03" \
  -e "cluster.initial_master_nodes=es01,es02,es03" \
  -e "network.host=0.0.0.0" \
  docker.elastic.co/elasticsearch/elasticsearch:8.8.1
```

### Logstash Networking

Logstash requires various ports depending on the inputs configured:

```bash
docker run -d \
  --name logstash \
  --network elk-network \
  -p 5044:5044 \
  -p 5000:5000/tcp \
  -p 5000:5000/udp \
  -p 9600:9600 \
  docker.elastic.co/logstash/logstash:8.8.1
```

### Kibana Networking

Kibana uses port 5601 for its web interface:

```bash
docker run -d \
  --name kibana \
  --network elk-network \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.8.1
```

### Beats Networking

Filebeat needs access to log files and Elasticsearch/Logstash:

```bash
docker run -d \
  --name filebeat \
  --network elk-network \
  --user root \
  -v "/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  -v "/var/run/docker.sock:/var/run/docker.sock:ro" \
  docker.elastic.co/beats/filebeat:8.8.1
```

## Network Inspection and Troubleshooting

### Inspecting Networks

```bash
# List all networks
docker network ls

# Inspect a network
docker network inspect elk-network

# List containers connected to a network
docker network inspect -f '{{range .Containers}}{{.Name}} {{end}}' elk-network
```

### Container Network Information

```bash
# Get IP address of a container
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' elasticsearch

# Show all network settings
docker inspect -f '{{json .NetworkSettings}}' elasticsearch | jq
```

### Network Troubleshooting

Tools available inside containers:

```bash
# Install network tools in a container
docker exec elasticsearch apt-get update && apt-get install -y iproute2 iputils-ping net-tools

# Check network interfaces
docker exec elasticsearch ip addr

# Test connectivity to another container
docker exec elasticsearch ping logstash

# Check open ports
docker exec elasticsearch netstat -tuln
```

## Network Security

### Network Isolation

Use separate networks to isolate different application tiers:

```bash
# Create separate networks
docker network create elk-frontend
docker network create elk-backend

# Connect Kibana to frontend
docker run --network=elk-frontend --name=kibana docker.elastic.co/kibana/kibana:8.8.1

# Connect Elasticsearch to backend
docker run --network=elk-backend --name=elasticsearch docker.elastic.co/elasticsearch/elasticsearch:8.8.1

# Connect Kibana to both networks to access Elasticsearch
docker network connect elk-backend kibana
```

### Network Policies

Control traffic between containers using iptables rules:

```bash
# Block all outgoing traffic from a container
docker exec elasticsearch iptables -A OUTPUT -j DROP

# Allow only specific connections
docker exec elasticsearch iptables -A OUTPUT -p tcp --dport 9200 -j ACCEPT
```

## Docker Networking vs Kubernetes Networking

While Docker and Kubernetes networking share concepts, they differ in implementation:

| Feature | Docker | Kubernetes |
|---------|--------|------------|
| Default network | Bridge | Cluster network |
| Service discovery | DNS or links | Services |
| Load balancing | Manual | Automatic |
| Network policies | Manual iptables | Network Policies API |
| Cross-host networking | Overlay | CNI plugins |
| IP per container | Optional | Mandatory |

### Key Differences

1. **Network Model**: 
   - Docker: Container Network Model (CNM)
   - Kubernetes: Container Network Interface (CNI)

2. **Service Discovery**:
   - Docker: DNS resolution within networks
   - Kubernetes: Service objects with DNS names

3. **Load Balancing**:
   - Docker: Requires external tools
   - Kubernetes: Built-in load balancing

4. **Network Policies**:
   - Docker: Manual configuration
   - Kubernetes: Declarative policies

## Advanced Topics

### Docker Network Performance

Performance considerations:

1. **Host Mode**: Fastest performance (no network isolation)
2. **Bridge Mode**: Good balance between isolation and performance
3. **Overlay Networks**: Additional overhead for cross-host communication

Benchmarking example:

```bash
# Install tools
docker run --name=netperf alpine apk add --no-cache iperf3

# Run in host mode (benchmark)
docker run --network=host --name=iperf-server -d alpine iperf3 -s
docker run --network=host --rm alpine iperf3 -c localhost

# Run in bridge mode (benchmark)
docker run --network=bridge --name=iperf-server -d alpine iperf3 -s
docker run --network=bridge --rm alpine iperf3 -c iperf-server
```

### Custom Network Plugins

For specialized needs, Docker supports custom network drivers:

```bash
# Install a network plugin
docker plugin install purestorage/docker-plugin:latest

# Create a network using the plugin
docker network create --driver=purestorage/docker-plugin mynetwork
```

## Best Practices

1. **Use Custom Bridge Networks**: Always create custom bridge networks instead of using the default bridge.

2. **Network Segmentation**: Separate different application tiers with different networks.

3. **Expose Only Necessary Ports**: Minimize exposed ports to reduce attack surface.

4. **Use Fixed IP Addressing**: For critical services, consider fixed IP addresses.

5. **Monitor Network Usage**: Use tools like `docker stats` to monitor container network usage.

6. **Document Network Architecture**: Maintain documentation of your container network architecture.

7. **Use Host Mode Wisely**: Use host networking mode only when necessary for performance.

8. **Regular Security Audits**: Regularly audit your container network configurations for security issues.

## ELK Stack Network Architecture Example

Here's a comprehensive example of a network architecture for the ELK stack:

```
                      ┌─────────────────────┐
                      │     User Access     │
                      └─────────────────────┘
                                ▲
                                │
                                ▼
┌───────────────────────────────────────────────────────┐
│                     Frontend Network                   │
└───────────────────────────────────────────────────────┘
                                ▲
                                │
                                ▼
                        ┌───────────────┐
                        │    Kibana     │
                        └───────────────┘
                                ▲
                                │
                                ▼
┌───────────────────────────────────────────────────────┐
│                     Backend Network                    │
└───────────────────────────────────────────────────────┘
        ▲                       ▲                      ▲
        │                       │                      │
        ▼                       ▼                      ▼
┌───────────────┐      ┌───────────────┐     ┌───────────────┐
│ Elasticsearch  │ ◄─► │   Logstash    │ ◄── │    Filebeat   │
│  (Cluster)     │     │               │     │               │
└───────────────┘      └───────────────┘     └───────────────┘
```

Implementation with Docker Compose:

```yaml
version: '3.8'

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
    networks:
      - backend
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node

  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.1
    networks:
      - backend
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.1
    networks:
      - frontend
      - backend
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.8.1
    networks:
      - backend
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch
      - logstash
```

## Conclusion

Docker networking provides a flexible and powerful system for connecting containers. For the ELK stack, proper network configuration is crucial for secure and efficient operation. While Docker networking offers many features, Kubernetes extends these capabilities with a more robust, declarative approach that's better suited for production environments.

Understanding the networking concepts presented in this guide will help you effectively deploy and manage the ELK stack in both Docker and Kubernetes environments, ensuring your applications can communicate securely and efficiently.