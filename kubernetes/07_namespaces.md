# Namespaces and Resource Quotas in Kubernetes

## Introduction to Namespaces

Namespaces provide a mechanism for isolating groups of resources within a single Kubernetes cluster. They are especially valuable in multi-tenant environments where multiple teams or projects share cluster resources. For ELK Stack deployments, namespaces offer logical separation between components and can isolate different environments.

## Namespace Fundamentals

### What Are Namespaces?

Namespaces are a way to divide cluster resources between multiple users, teams, or applications. They provide a scope for names - names of resources must be unique within a namespace, but not across namespaces.

### Default Namespaces

Kubernetes creates several namespaces out of the box:

- **default**: The default namespace for resources when no namespace is specified
- **kube-system**: System components created by Kubernetes
- **kube-public**: Resources that should be publicly readable throughout the cluster
- **kube-node-lease**: Node lease objects for node heartbeats

### Creating Namespaces

Namespaces can be created via a YAML file:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elastic-system
  labels:
    name: elastic-system
    environment: production
```

Or directly with kubectl:

```bash
kubectl create namespace elastic-system
```

### Listing Namespaces

```bash
kubectl get namespaces
```

### Deleting Namespaces

```bash
kubectl delete namespace elastic-system
```

**Warning**: Deleting a namespace will delete all resources within that namespace!

## Namespaces in ELK Stack Deployments

### Common ELK Stack Namespace Strategies

#### Scenario 1: Component-Based Namespaces

Separate namespaces for each component of the ELK stack:

```bash
kubectl create namespace elasticsearch
kubectl create namespace logstash
kubectl create namespace kibana
kubectl create namespace beats
```

This approach provides component-level isolation, allows different teams to manage different components, and enables granular resource allocation.

#### Scenario 2: Environment-Based Namespaces

Separate namespaces for different environments:

```bash
kubectl create namespace elk-production
kubectl create namespace elk-staging
kubectl create namespace elk-development
```

This approach logically separates environments, prevents accidental cross-environment resource access, and allows for different configuration policies per environment.

#### Scenario 3: Tenant-Based Namespaces

Separate namespaces for different teams or applications that need their own ELK stack:

```bash
kubectl create namespace team-a-logging
kubectl create namespace team-b-logging
kubectl create namespace security-logging
```

This approach enables multi-tenancy, allows isolation of sensitive data, and provides separate resource quotas per tenant.

### Deploying ELK Stack to a Namespace

You can specify a namespace in your Kubernetes manifests:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-master
  namespace: elastic-system
```

Or deploy directly to a namespace:

```bash
kubectl apply -f elasticsearch.yaml -n elastic-system
```

### Cross-Namespace Communication

Resources in different namespaces can communicate with each other using fully qualified domain names (FQDNs):

```
<service-name>.<namespace>.svc.cluster.local
```

Example for Elasticsearch:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: logstash
data:
  logstash.yml: |
    output.elasticsearch:
      hosts: ["elasticsearch-master.elasticsearch.svc.cluster.local:9200"]
```

## Resource Quotas

Resource Quotas provide constraints that limit aggregate resource consumption per namespace. They can limit the quantity of objects that can be created in a namespace as well as the total amount of compute resources consumed.

### Types of Resource Quotas

#### Compute Resource Quotas

Limit CPU and memory resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: elasticsearch-quota
  namespace: elastic-system
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 100Gi
    limits.cpu: "40"
    limits.memory: 200Gi
```

#### Storage Resource Quotas

Limit storage resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: elasticsearch-storage-quota
  namespace: elastic-system
spec:
  hard:
    persistentvolumeclaims: "30"
    requests.storage: "1Ti"
    elastic-storage.storageclass.storage.k8s.io/requests.storage: "500Gi"
```

#### Object Count Quotas

Limit the number of objects by type:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: elk-object-quota
  namespace: elastic-system
spec:
  hard:
    configmaps: "20"
    secrets: "20"
    services: "15"
    services.loadbalancers: "3"
    pods: "50"
    statefulsets.apps: "10"
    deployments.apps: "10"
```

### Viewing Resource Quotas

```bash
kubectl get resourcequota -n elastic-system
kubectl describe resourcequota elasticsearch-quota -n elastic-system
```

### Resource Quotas for ELK Stack Components

#### Elasticsearch Quota Example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: elasticsearch-quota
  namespace: elasticsearch
spec:
  hard:
    # Compute resources
    requests.cpu: "16"
    requests.memory: 64Gi
    limits.cpu: "32"
    limits.memory: 64Gi
    
    # Storage resources
    persistentvolumeclaims: "12"
    requests.storage: "2Ti"
    
    # Object counts
    pods: "12"
    services: "5"
    secrets: "10"
    configmaps: "10"
```

#### Kibana Quota Example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: kibana-quota
  namespace: kibana
spec:
  hard:
    # Compute resources
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    
    # Object counts
    pods: "4"
    services: "2"
    secrets: "5"
    configmaps: "5"
```

#### Logstash Quota Example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: logstash-quota
  namespace: logstash
spec:
  hard:
    # Compute resources
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    
    # Object counts
    pods: "6"
    services: "3"
    secrets: "10"
    configmaps: "10"
```

## LimitRanges

While ResourceQuotas set limits on the total resources in a namespace, LimitRanges set constraints on individual containers and pods.

### Types of Limits

- Default resource limits for containers
- Minimum and maximum resource requirements
- Storage request limits
- CPU limits ratio

### LimitRange for Elasticsearch Containers

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: elasticsearch-limits
  namespace: elasticsearch
spec:
  limits:
  - type: Container
    default:
      cpu: "2"
      memory: 4Gi
    defaultRequest:
      cpu: "1"
      memory: 2Gi
    min:
      cpu: "500m"
      memory: 1Gi
    max:
      cpu: "4"
      memory: 8Gi
```

This LimitRange ensures that:
- All containers have a default limit of 2 CPU cores and 4GB memory
- All containers have a default request of 1 CPU core and 2GB memory
- No container can request less than 500m CPU or 1GB memory
- No container can request more than 4 CPU cores or 8GB memory

## Best Practices for Namespaces and Resource Quotas with ELK Stack

### Namespace Best Practices

1. **Logical Grouping**: Group related resources in the same namespace (e.g., keep all Elasticsearch components in one namespace)

2. **Environment Isolation**: Use separate namespaces for different environments (production, staging, development)

3. **Admin Access**: Restrict administrative access to critical namespaces like production

4. **Namespace Conventions**: Establish naming conventions for namespaces

5. **Cross-Namespace References**: Minimize cross-namespace references where possible, but use FQDNs when necessary

6. **Service Mesh Integration**: Consider using a service mesh for more sophisticated cross-namespace traffic controls

### Resource Quota Best Practices

1. **Start with Monitoring**: Before enforcing quotas, monitor resource usage to establish baselines

2. **Buffer for Spikes**: Include a buffer in quotas for unexpected spikes in resource usage

3. **Separate Critical Components**: Use separate quotas for critical components like Elasticsearch master nodes

4. **Document Quotas**: Document resource quotas and their rationale

5. **Regular Review**: Regularly review and adjust quotas based on actual usage patterns

6. **Coordinate with Autoscaling**: Ensure resource quotas don't conflict with horizontal and vertical pod autoscaling

## Real-World Example: Multi-Tenant ELK Stack

For a multi-tenant logging platform where multiple teams have their own Elasticsearch clusters:

### Namespace Structure

```bash
# Shared platform services
kubectl create namespace logging-platform

# Team namespaces
kubectl create namespace team-a-logging
kubectl create namespace team-b-logging
kubectl create namespace team-c-logging
```

### Platform-Level Services

Deploy central Kibana and monitoring to the platform namespace:

```yaml
# In logging-platform namespace
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging-platform
spec:
  # Kibana configuration
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-monitoring
  namespace: logging-platform
spec:
  # Monitoring Elasticsearch configuration
```

### Team-Level Elasticsearch Clusters

Each team gets their own Elasticsearch cluster with resource quotas:

```yaml
# In team-a-logging namespace
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a-logging
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 32Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    persistentvolumeclaims: "6"
    requests.storage: "500Gi"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: team-a-logging
spec:
  # Team A Elasticsearch configuration
```

### Cross-Namespace Access

Allow Kibana in the platform namespace to access team Elasticsearch clusters:

```yaml
# In logging-platform namespace
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-elasticsearch-config
  namespace: logging-platform
data:
  kibana.yml: |
    elasticsearch.hosts:
      - https://elasticsearch.team-a-logging.svc.cluster.local:9200
      - https://elasticsearch.team-b-logging.svc.cluster.local:9200
      - https://elasticsearch.team-c-logging.svc.cluster.local:9200
```

### Network Policies for Namespace Isolation

Restrict access between namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-a-elasticsearch-policy
  namespace: team-a-logging
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: logging-platform
      podSelector:
        matchLabels:
          app: kibana
  - from:
    - namespaceSelector:
        matchLabels:
          name: team-a-logging
```

## Monitoring Namespace Resource Usage

To effectively manage namespaces and resource quotas, monitor resource usage:

```bash
# Get resource usage across namespaces
kubectl top pod --all-namespaces

# Get resource usage in the elasticsearch namespace
kubectl top pod -n elasticsearch

# View resource quota utilization
kubectl describe resourcequota elasticsearch-quota -n elasticsearch
```

For more detailed monitoring, implement a monitoring solution like Prometheus and Grafana, or use the Elastic Stack's Metricbeat with the Kubernetes module enabled to monitor Kubernetes resource usage.

## Conclusion

Namespaces and resource quotas are powerful tools for organizing and controlling resource usage in Kubernetes clusters. For ELK Stack deployments, they enable logical separation of components, isolation between environments, and efficient resource allocation. By following best practices for namespace organization and resource quotas, you can create reliable, efficient, and secure ELK Stack deployments on Kubernetes.