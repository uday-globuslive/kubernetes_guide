# Docker Compose

## Introduction

Docker Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application's services, networks, and volumes. Then, with a single command, you create and start all the services from your configuration.

This guide covers Docker Compose fundamentals, configuration, and best practices, with a focus on how it relates to the ELK stack and Kubernetes.

## Docker Compose Fundamentals

### Installation

Before using Docker Compose, you need to install it:

```bash
# On Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker-compose-plugin

# On MacOS with Homebrew
brew install docker-compose

# Using pip
pip install docker-compose
```

### Basic Commands

The most common Docker Compose commands:

```bash
# Create and start containers
docker compose up

# Start in detached mode (background)
docker compose up -d

# Stop containers
docker compose down

# Stop and remove containers, networks, images, and volumes
docker compose down --volumes --rmi all

# View logs
docker compose logs

# Follow logs
docker compose logs -f

# Execute command in a container
docker compose exec service_name command

# List containers
docker compose ps

# Show resource usage
docker compose top
```

## Docker Compose File

A Docker Compose file is a YAML file that defines services, networks, and volumes:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    restart: always
    depends_on:
      - app
  
  app:
    build: ./app
    environment:
      - NODE_ENV=production
    restart: always
```

### Key Components

#### Services

Services define the containers that your application uses:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
    ports:
      - "9200:9200"
      - "9300:9300"
```

#### Networks

Networks define how containers communicate with each other:

```yaml
services:
  elasticsearch:
    networks:
      - elk

networks:
  elk:
    driver: bridge
```

#### Volumes

Volumes allow data to persist beyond the lifecycle of a container:

```yaml
services:
  elasticsearch:
    volumes:
      - esdata:/usr/share/elasticsearch/data

volumes:
  esdata:
    driver: local
```

## Docker Compose for ELK Stack

### Basic ELK Stack Setup

Here's a complete example of a docker-compose.yml file for the ELK stack:

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=es-docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q '\"status\":\"green\"\\|\"status\":\"yellow\"'"]
      interval: 30s
      timeout: 10s
      retries: 5
  
  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.1
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9600"]
      interval: 30s
      timeout: 10s
      retries: 5
  
  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.1
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    networks:
      - elk
    depends_on:
      - elasticsearch
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5601/api/status"]
      interval: 30s
      timeout: 10s
      retries: 5

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.8.1
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
    networks:
      - elk
    depends_on:
      - elasticsearch
      - logstash

networks:
  elk:
    driver: bridge

volumes:
  esdata:
    driver: local
```

### Exploring Key Configuration Elements

#### Elasticsearch Configuration

```yaml
elasticsearch:
  environment:
    - node.name=elasticsearch
    - cluster.name=es-docker-cluster
    - discovery.type=single-node
    - bootstrap.memory_lock=true
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  ulimits:
    memlock:
      soft: -1
      hard: -1
```

- `discovery.type=single-node`: Configures Elasticsearch to run as a single-node cluster
- `bootstrap.memory_lock=true`: Locks the memory to prevent swapping
- `ES_JAVA_OPTS`: Sets the JVM heap size
- `memlock: soft/hard: -1`: Disables memory limits

#### Logstash Configuration

You need to create pipeline configuration files:

```bash
mkdir -p logstash/config logstash/pipeline
```

Create `logstash/config/logstash.yml`:

```yaml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]
```

Create `logstash/pipeline/logstash.conf`:

```
input {
  beats {
    port => 5044
  }
  tcp {
    port => 5000
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

#### Filebeat Configuration

Create `filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
- type: container
  paths:
    - /var/lib/docker/containers/*/*.log

filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

processors:
- add_docker_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]
```

## Docker Compose vs Kubernetes

While Docker Compose is great for development and small deployments, Kubernetes offers more robust features for production:

| Feature | Docker Compose | Kubernetes |
|---------|---------------|------------|
| Scaling | Limited (docker-compose scale) | Advanced (horizontal pod autoscaling) |
| Self-healing | Limited (restart policies) | Advanced (health checks, auto-restart) |
| Load balancing | Basic | Advanced (services, ingress) |
| Rolling updates | Basic | Advanced (deployment strategies) |
| Volume management | Basic | Advanced (persistent volume claims) |
| Service discovery | Via Docker network | Advanced (DNS, services) |

### When to Use Docker Compose vs Kubernetes

- **Use Docker Compose when**:
  - You're in development or testing
  - You have a simple application with few services
  - You need a quick and easy setup
  - You're running on a single host

- **Use Kubernetes when**:
  - You're in production
  - You need high availability
  - You need horizontal scaling
  - You have complex deployment requirements
  - You're running across multiple hosts

## Converting Docker Compose to Kubernetes

You can convert a Docker Compose file to Kubernetes manifests using tools like:

- **Kompose**: A conversion tool to go from Docker Compose to Kubernetes

```bash
# Install Kompose
curl -L https://github.com/kubernetes/kompose/releases/download/v1.26.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose

# Convert docker-compose.yml to Kubernetes manifests
kompose convert -f docker-compose.yml
```

Example Kubernetes manifests generated from the ELK stack Docker Compose:

```yaml
# elasticsearch-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
        env:
        - name: node.name
          value: elasticsearch
        - name: cluster.name
          value: es-docker-cluster
        - name: discovery.type
          value: single-node
        - name: bootstrap.memory_lock
          value: "true"
        - name: ES_JAVA_OPTS
          value: -Xms512m -Xmx512m
        ports:
        - containerPort: 9200
        - containerPort: 9300
        volumeMounts:
        - name: esdata
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: esdata
        persistentVolumeClaim:
          claimName: esdata
```

## Best Practices

### Docker Compose Best Practices

1. **Version Control**: Keep your docker-compose.yml under version control
2. **Environment Variables**: Use .env files to store environment variables
3. **Service Dependencies**: Use depends_on to control startup order
4. **Health Checks**: Implement health checks for services
5. **Named Volumes**: Use named volumes instead of bind mounts for persistent data
6. **Networks**: Create custom networks for better isolation

### Production Considerations

1. **Resource Limits**: Set resource limits for each service
2. **Logging**: Configure proper logging
3. **Monitoring**: Add monitoring services like Prometheus and Grafana
4. **Backup**: Implement backup strategies for volumes
5. **Security**: Secure your services with proper network isolation and secrets management

```yaml
services:
  elasticsearch:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

## Conclusion

Docker Compose is a powerful tool for defining and running multi-container applications. For the ELK stack, it provides a convenient way to set up development environments and small deployments. However, for production environments, especially those requiring high availability and scalability, Kubernetes is the recommended approach. Understanding Docker Compose is still valuable, as it forms the foundation for container orchestration concepts that are extended in Kubernetes.