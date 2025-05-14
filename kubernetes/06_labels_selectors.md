# Labels, Selectors, and Annotations in Kubernetes

## Introduction to Labels and Selectors

Labels and selectors are fundamental organizational concepts in Kubernetes. They provide a flexible way to organize and select subsets of objects, forming the backbone of dynamic resource management and service discovery.

## Labels

Labels are key-value pairs attached to Kubernetes objects. They are a non-hierarchical, arbitrary way of identifying and organizing resources. For the ELK stack, labels help organize and identify Elasticsearch, Logstash, and Kibana components.

### Label Format

- Keys can have optional prefixes with a DNS subdomain format (e.g., `elastic.co/component`)
- Keys without prefixes must be valid DNS labels (lowercase alphanumeric, '-')
- Values must be 63 characters or less, with valid characters being alphanumeric, '_', '-', and '.'

### Example Labels for ELK Stack

```yaml
metadata:
  labels:
    app: elasticsearch
    component: elasticsearch
    role: master
    cluster: production
    environment: prod
    tier: data
    version: 7.15.0
```

## Label Use Cases for ELK Stack

### 1. Component Identification

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  labels:
    app: elasticsearch
    component: elk
    role: data
spec:
  selector:
    matchLabels:
      app: elasticsearch
      role: data
  template:
    metadata:
      labels:
        app: elasticsearch
        component: elk
        role: data
```

### 2. Release Versioning

```yaml
metadata:
  labels:
    app.kubernetes.io/name: elasticsearch
    app.kubernetes.io/instance: elk-production
    app.kubernetes.io/version: 7.15.0
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: elk-stack
    app.kubernetes.io/managed-by: helm
```

This follows the [recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) from Kubernetes.

### 3. Environment Segregation

```yaml
metadata:
  labels:
    environment: production
    region: us-west
    availability-zone: us-west-2a
```

### 4. Team Ownership

```yaml
metadata:
  labels:
    team: logging
    owner: logging-team
    cost-center: infra-123
```

## Label Selectors

Label selectors are used to identify and select sets of objects based on their labels. There are two types of selectors:

### Equality-Based Selectors

Match objects based on label value equality or inequality:

```yaml
# Selects all resources with label role=master
selector:
  role: master

# Selects all resources with label role!=client
selector:
  role: '!client'
```

### Set-Based Selectors

Match objects based on a set of values:

```yaml
selector:
  matchExpressions:
    - key: role
      operator: In
      values:
      - master
      - data
    - key: environment
      operator: NotIn
      values:
      - dev
    - key: tier
      operator: Exists
```

## Practical Use of Selectors in ELK Stack

### 1. Service Selection

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-discovery
  labels:
    app: elasticsearch
    role: master
spec:
  selector:
    app: elasticsearch
    role: master
  ports:
  - port: 9300
    name: transport
  clusterIP: None
```

This service targets only Elasticsearch master nodes.

### 2. NetworkPolicy Targeting

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: kibana
    ports:
    - protocol: TCP
      port: 9200
  - from:
    - podSelector:
        matchLabels:
          app: logstash
    ports:
    - protocol: TCP
      port: 9200
```

This network policy restricts access to Elasticsearch pods to only Kibana and Logstash pods.

### 3. Pod Affinity and Anti-Affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - elasticsearch
        - key: role
          operator: In
          values:
          - data
      topologyKey: kubernetes.io/hostname
```

This ensures Elasticsearch data nodes are scheduled on different hosts.

### 4. Node Selection

```yaml
nodeSelector:
  elasticsearch-data: "true"
```

This simple node selector ensures pods are scheduled only on nodes labeled for Elasticsearch data.

More sophisticated node selection with node affinity:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: In
          values:
          - elasticsearch-optimized
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 10
      preference:
        matchExpressions:
        - key: instance-type
          operator: In
          values:
          - highio
          - highmem
```

## Annotations

Annotations are similar to labels but are used for non-identifying metadata that tools and libraries can leverage. While labels are used for selection, annotations are designed for attaching non-identifying metadata.

### Annotation Format

- Annotations follow the same format requirements as label keys
- Annotation values can be arbitrary strings (not limited to 63 characters)

### Common Annotations in ELK Stack Deployments

```yaml
metadata:
  annotations:
    kubernetes.io/description: "Elasticsearch master node for production ELK stack"
    co.elastic.logs/enabled: "true"
    co.elastic.metrics/module: "elasticsearch"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9114"
    fluentbit.io/parser: "json"
    scheduler.alpha.kubernetes.io/critical-pod: ""
    seccomp.security.alpha.kubernetes.io/pod: "docker/default"
```

## Practical Annotation Use Cases

### 1. Documentation and Description

```yaml
metadata:
  annotations:
    kubernetes.io/description: "Elasticsearch data node with hot tier configuration"
    team.elastic.co/owner: "data-platform-team"
    team.elastic.co/contact: "slack:#elk-support"
```

### 2. Build and Release Information

```yaml
metadata:
  annotations:
    build.elastic.co/commit: "ab3f4c5d6e7"
    build.elastic.co/branch: "release-7.15"
    build.elastic.co/timestamp: "2023-04-12T15:30:00Z"
    deployment.elastic.co/initiated-by: "jenkins-ci"
```

### 3. Configuration Management

```yaml
metadata:
  annotations:
    config.elasticsearch.io/jvm-heap: "8g"
    config.elasticsearch.io/node-roles: "data,ingest"
    config.elasticsearch.io/settings-checksum: "a1b2c3d4e5f6"
```

### 4. Ingress Controller Configuration

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### 5. Configuring Prometheus Monitoring

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "9200"
```

## Best Practices for Labels and Annotations

### Labeling Strategy

1. **Consistency**: Use consistent labeling schemes across your ELK stack deployments
2. **Governance**: Document labeling standards for your organization
3. **Minimalism**: Use only the labels you need for selection and organization
4. **Hierarchy**: Consider using DNS-prefixed labels for hierarchical organization

### Label Scheme Examples

#### Elasticsearch Nodes

```yaml
app: elasticsearch          # Application name
component: elk-stack        # Component within the broader system
role: master|data|ingest    # Role within the Elasticsearch cluster
cluster: production         # Cluster identifier
tier: hot|warm|cold|frozen  # Data tier (for Elasticsearch data nodes)
environment: prod|staging   # Deployment environment
region: us-west             # Geographical location
```

#### Kibana and Logstash

```yaml
app: kibana|logstash        # Application name
component: elk-stack        # Component within the broader system
environment: prod|staging   # Deployment environment
region: us-west             # Geographical location
```

### Annotation Best Practices

1. **Non-identifying Metadata**: Use annotations for metadata that doesn't identify objects
2. **Automation**: Leverage annotations for automation and tooling integration
3. **Documentation**: Document system configurations and relationships
4. **Size Limitations**: Remember that very large annotations can impact etcd performance

## Labels vs. Annotations: When to Use Which

Use **Labels** when you need to:
- Select objects using selectors
- Organize resources for querying
- Specify scheduling constraints
- Target specific deployment scenarios

Use **Annotations** when you need to:
- Store build/release information
- Configure third-party tools
- Document resources
- Store checksum/configuration references
- Add descriptive information
- Configure controllers (like ingress)

## Practical Examples for ELK Stack Deployment

### Complete Elasticsearch StatefulSet with Labels and Annotations

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  labels:
    app: elasticsearch
    component: elk-stack
    role: data
    tier: hot
    environment: production
  annotations:
    kubernetes.io/description: "Elasticsearch hot data nodes for production"
    co.elastic.logs/enabled: "true"
    co.elastic.metrics/module: "elasticsearch"
    config.elasticsearch.io/jvm-heap: "16g"
spec:
  selector:
    matchLabels:
      app: elasticsearch
      role: data
      tier: hot
  serviceName: elasticsearch-data
  replicas: 3
  template:
    metadata:
      labels:
        app: elasticsearch
        component: elk-stack
        role: data
        tier: hot
        environment: production
      annotations:
        co.elastic.logs/enabled: "true"
        co.elastic.metrics/module: "elasticsearch"
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
        env:
        - name: node.roles
          value: "data"
        - name: cluster.name
          value: "elk-production"
        # other configuration...
```

### Kibana Deployment with Labels and Annotations

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  labels:
    app: kibana
    component: elk-stack
    environment: production
  annotations:
    kubernetes.io/description: "Kibana dashboard for production ELK stack"
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
        component: elk-stack
        environment: production
      annotations:
        co.elastic.logs/enabled: "true"
        co.elastic.metrics/module: "kibana"
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.15.0
        # configuration...
```

## Conclusion

Labels, selectors, and annotations are powerful tools for organizing and managing your ELK stack on Kubernetes. By following consistent labeling schemes and leveraging annotations effectively, you can create maintainable, scalable, and discoverable Elasticsearch, Logstash, and Kibana deployments that integrate well with Kubernetes' dynamic scheduling and service discovery mechanisms.