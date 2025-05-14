# Kubernetes Components

## Introduction

Kubernetes is an open-source container orchestration platform designed to automate the deployment, scaling, and management of containerized applications. Understanding the core components of Kubernetes is essential for effectively deploying and managing the Elastic Stack in a containerized environment.

This guide provides a detailed overview of Kubernetes architecture and its key components, explaining how they work together to form a robust platform for container orchestration.

## Architecture Overview

Kubernetes follows a master-worker architecture, consisting of a control plane (master) that manages the worker nodes. The control plane makes global decisions about the cluster, while the worker nodes host the applications.

```
┌─────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                          │
│                                                                 │
│  ┌─────────────────────────┐      ┌─────────────────────────┐   │
│  │     Control Plane       │      │       Worker Node       │   │
│  │                         │      │                         │   │
│  │  ┌─────────────────┐    │      │  ┌─────────────────┐    │   │
│  │  │  API Server     │    │      │  │      Kubelet    │    │   │
│  │  └─────────────────┘    │      │  └─────────────────┘    │   │
│  │                         │      │                         │   │
│  │  ┌─────────────────┐    │      │  ┌─────────────────┐    │   │
│  │  │  Scheduler      │    │      │  │    Kube-proxy   │    │   │
│  │  └─────────────────┘    │      │  └─────────────────┘    │   │
│  │                         │      │                         │   │
│  │  ┌─────────────────┐    │      │  ┌─────────────────┐    │   │
│  │  │ Controller      │    │      │  │     Container   │    │   │
│  │  │ Manager         │    │      │  │     Runtime     │    │   │
│  │  └─────────────────┘    │      │  └─────────────────┘    │   │
│  │                         │      │                         │   │
│  │  ┌─────────────────┐    │      │  ┌─────────────────┐    │   │
│  │  │     etcd        │    │      │  │   Application   │    │   │
│  │  │                 │    │      │  │     Pods        │    │   │
│  │  └─────────────────┘    │      │  └─────────────────┘    │   │
│  └─────────────────────────┘      └─────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Control Plane Components

The control plane manages the Kubernetes cluster and is responsible for making global decisions such as scheduling, responding to events, and implementing changes to maintain the desired state.

### API Server

The Kubernetes API server is the front end of the control plane, exposing the Kubernetes API. It processes RESTful API requests, validates them, and updates the corresponding objects in etcd.

Key features:
- Authentication and authorization
- API object validation and persistence
- RESTful API endpoint for cluster operations
- Serves as the gateway to the cluster

```yaml
# Example API interaction using kubectl
kubectl get pods -n kube-system
```

### etcd

etcd is a distributed key-value store that serves as Kubernetes' backing store for all cluster data. It provides a reliable way to store data across a cluster of machines.

Key features:
- Consistent and highly-available key-value store
- Stores the entire configuration and state of the cluster
- Uses the Raft consensus algorithm for data replication
- Sensitive to disk I/O performance

```bash
# Example: Interacting with etcd directly (not usually necessary)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only
```

### Scheduler

The Kubernetes scheduler assigns newly created pods to nodes based on resource requirements, policies, and constraints.

Key features:
- Monitors the API server for unassigned pods
- Considers resource requirements, hardware/software constraints, and data locality
- Ranks available nodes and binds the pod to an optimal node
- Respects taints, tolerations, and node affinity rules

```yaml
# Example: Pod with node affinity
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/role
            operator: In
            values:
            - elasticsearch
  containers:
  - name: elasticsearch
    image: elasticsearch:8.8.1
```

### Controller Manager

The controller manager runs controller processes that regulate the state of the cluster. Each controller is a separate process, but they are compiled into a single binary and run in a single process.

Key controller examples:
- **Node Controller**: Notices and responds when nodes go down
- **Replication Controller**: Maintains the correct number of pods for each replica set
- **Endpoints Controller**: Populates the Endpoints object (joins Services & Pods)
- **Service Account & Token Controllers**: Create default accounts and API access tokens

```yaml
# Example: Deployment controller managing replica count
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  replicas: 3
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
```

### Cloud Controller Manager

The cloud controller manager lets you link your cluster to your cloud provider's API. It separates components that interact with the cloud platform from components that only interact with your cluster.

Cloud-specific controllers:
- **Node Controller**: Checks the cloud provider to determine if a node has been deleted
- **Route Controller**: Sets up routes in the cloud
- **Service Controller**: Creates, updates, and deletes cloud provider load balancers

## Node Components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

### Kubelet

The kubelet is an agent that runs on each node and ensures containers are running in a Pod. It takes a set of PodSpecs and ensures the described containers are running and healthy.

Key features:
- Communicates with the API server
- Manages container lifecycle
- Performs node and pod health checks
- Manages pod volumes and secrets
- Reports node and pod status to the master

```bash
# Example: Check kubelet status
systemctl status kubelet
```

### Kube-proxy

Kube-proxy is a network proxy that runs on each node and implements part of the Kubernetes Service concept. It maintains network rules that allow network communication to your Pods.

Key features:
- Watches the API server for Service and Endpoint changes
- Maintains network rules for pod accessibility
- Performs connection forwarding
- Implements service abstraction

```yaml
# Example: Service that kube-proxy will manage
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
  type: ClusterIP
```

### Container Runtime

The container runtime is the software responsible for running containers. Kubernetes supports several container runtimes such as Docker, containerd, CRI-O, and others that implement the Kubernetes Container Runtime Interface (CRI).

Common container runtimes:
- **containerd**: A lightweight, high-performance container runtime
- **CRI-O**: A lightweight runtime specifically for Kubernetes
- **Docker Engine**: The traditional container runtime (uses containerd under the hood)

## Addons

Addons are pods and services that implement cluster features. These are managed by Kubernetes resources like Deployments and typically run in the kube-system namespace.

### DNS

DNS is a critical addon that serves DNS records for Kubernetes services, enabling service discovery.

Key features:
- Provides cluster-wide DNS resolution
- Allows service discovery by name
- Supports DNS-based service load balancing
- Implemented by CoreDNS (previously kube-dns)

```yaml
# Example: Creating a service with a DNS record
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-service
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
```

After creation, pods can access this service at `elasticsearch-service.default.svc.cluster.local`

### Dashboard

The Kubernetes Dashboard is a web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them.

```bash
# Deploy Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create service account for dashboard access
kubectl create serviceaccount dashboard-admin
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin
```

### Container Resource Monitoring

Container Resource Monitoring records generic time-series metrics about containers in a central database and provides a UI for browsing that data.

Common monitoring solutions:
- **Prometheus**: Monitoring and alerting toolkit
- **Grafana**: Visualization and analytics platform
- **Elastic Stack**: Logging and monitoring platform

```yaml
# Example: Deploying Prometheus for Kubernetes monitoring
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.42.0
        ports:
        - containerPort: 9090
```

### Cluster-level Logging

Cluster-level logging mechanisms save container logs to a central log store with search/browsing interface. The Elastic Stack (ELK) is a popular solution for this purpose.

```yaml
# Example: Filebeat DaemonSet for logging
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.1
```

## Kubernetes API Objects

Kubernetes API objects are persistent entities that represent the state of your cluster. They describe what containerized applications are running, what resources they consume, and the policies around how they behave.

### Fundamental Objects

#### Pod

The smallest deployable unit in Kubernetes, representing a single instance of a running process in a cluster. Pods contain one or more containers that share storage and network resources.

```yaml
# Basic Pod example
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-pod
  labels:
    app: elasticsearch
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
    ports:
    - containerPort: 9200
    env:
    - name: ES_JAVA_OPTS
      value: "-Xms512m -Xmx512m"
    - name: discovery.type
      value: single-node
```

#### Service

An abstraction that defines a logical set of Pods and a policy by which to access them. Services enable network access to a set of Pods, providing stable endpoints.

Types of Services:
- **ClusterIP**: Internal-only IP accessible within the cluster
- **NodePort**: Exposes the service on each node's IP at a static port
- **LoadBalancer**: Exposes the service externally using a cloud provider's load balancer
- **ExternalName**: Maps the service to a DNS name

```yaml
# Example Service
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
  type: ClusterIP
```

#### Volume

A directory accessible to containers in a Pod. Kubernetes volumes have an explicit lifetime - the same as the Pod that encloses them.

Types of Volumes:
- **emptyDir**: Empty directory that starts with Pod; shares Pod lifetime
- **hostPath**: Mounts file or directory from the host node's filesystem
- **persistentVolumeClaim**: Mounts a PersistentVolume into a Pod
- **configMap/secret**: Provides Pods with configuration data or sensitive information

```yaml
# Example Pod with a volume
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-pod
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
    volumeMounts:
    - name: elasticsearch-data
      mountPath: /usr/share/elasticsearch/data
  volumes:
  - name: elasticsearch-data
    persistentVolumeClaim:
      claimName: elasticsearch-pvc
```

#### Namespace

Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces.

```yaml
# Creating a namespace
apiVersion: v1
kind: Namespace
metadata:
  name: elastic-system
```

### Higher-level Abstractions

#### Deployment

A Deployment provides declarative updates for Pods and ReplicaSets. It manages the deployment and scaling of a set of Pods, and provides rollback and rollout functionality.

```yaml
# Example Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.8.1
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch:9200
```

#### StatefulSet

StatefulSet is used to manage stateful applications, providing guarantees about the ordering and uniqueness of Pods. It maintains a sticky identity for each of its Pods.

Key features:
- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

```yaml
# Example StatefulSet for Elasticsearch
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
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
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: cluster.name
          value: elasticsearch-cluster
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

#### DaemonSet

DaemonSet ensures that a copy of a Pod runs on all (or some) nodes in a cluster. As nodes are added to the cluster, Pods are added to them.

Use cases:
- Cluster storage daemons (e.g., Ceph)
- Log collection (e.g., Filebeat, Fluentd)
- Node monitoring (e.g., Prometheus Node Exporter)

```yaml
# Example DaemonSet for Filebeat
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### Job and CronJob

Jobs create Pods that run until successful completion. CronJobs create Jobs on a time-based schedule.

```yaml
# Example CronJob for Elasticsearch index cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: elasticsearch-cleanup
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: curator
            image: elasticsearch-curator:5.8.4
            args:
            - "--config"
            - "/etc/curator/config.yml"
            - "/etc/curator/actions.yml"
            volumeMounts:
            - name: config
              mountPath: /etc/curator
          volumes:
          - name: config
            configMap:
              name: curator-config
          restartPolicy: OnFailure
```

#### ConfigMap and Secret

ConfigMap and Secret provide a way to inject configuration data into Pods. ConfigMaps store non-confidential data while Secrets store sensitive information.

```yaml
# Example ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
data:
  elasticsearch.yml: |
    cluster.name: my-cluster
    node.name: ${HOSTNAME}
    network.host: 0.0.0.0
    discovery.seed_hosts: ["elasticsearch-0.elasticsearch", "elasticsearch-1.elasticsearch"]
    cluster.initial_master_nodes: ["elasticsearch-0", "elasticsearch-1"]

# Example Secret
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-credentials
type: Opaque
data:
  username: ZWxhc3RpYw==  # elastic (base64 encoded)
  password: Y2hhbmdlbWU=  # changeme (base64 encoded)
```

## Component Interactions

Understanding how Kubernetes components interact is essential for troubleshooting and optimization.

### Deployment Flow

1. **User creates a Deployment**: User applies a YAML file with `kubectl apply -f deployment.yaml`
2. **API Server**: Validates and stores the Deployment in etcd
3. **Deployment Controller**: Notices the new Deployment and creates a ReplicaSet
4. **ReplicaSet Controller**: Creates Pod objects as specified
5. **Scheduler**: Assigns nodes to the Pods
6. **Kubelet**: Creates and manages the containers on the assigned node

### Service Discovery Flow

1. **User creates a Service**: User applies a YAML file with `kubectl apply -f service.yaml`
2. **API Server**: Validates and stores the Service in etcd
3. **Endpoints Controller**: Creates Endpoints that point to Pods matching the Service selector
4. **Kube-proxy**: Configures iptables rules on each node to direct traffic
5. **CoreDNS**: Creates DNS records for the Service

## Elastic Stack on Kubernetes

The Kubernetes components described above work together to provide a robust platform for deploying the Elastic Stack.

### Elasticsearch Architecture

Elasticsearch on Kubernetes is typically deployed as a StatefulSet, with each Pod having stable network identity and persistent storage.

```yaml
# Elasticsearch StatefulSet (simplified)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
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
```

### Kibana Deployment

Kibana is typically deployed as a Deployment with a Service to expose it.

```yaml
# Kibana Deployment (simplified)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.8.1
```

### Logstash and Beats

Logstash is typically deployed as a StatefulSet or Deployment, while Beats (like Filebeat) are deployed as DaemonSets to run on every node.

```yaml
# Filebeat DaemonSet (simplified)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.1
```

## Advanced Kubernetes Features

### Custom Resource Definitions (CRDs)

CRDs extend the Kubernetes API, allowing you to define custom resources. The Elastic Cloud on Kubernetes (ECK) operator uses CRDs to manage Elastic Stack resources.

```yaml
# Example Elasticsearch CRD with ECK
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 8.8.1
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
```

### Operators

Operators are application-specific controllers that extend Kubernetes to automate the management of complex applications. The ECK operator manages Elasticsearch, Kibana, and other Elastic Stack components.

```bash
# Install ECK Operator
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/operator.yaml
```

### Admission Controllers

Admission controllers intercept requests to the Kubernetes API server before object persistence, allowing for validation and modification of objects.

Common admission controllers:
- **PodSecurityPolicy**: Controls security-sensitive aspects of Pod specification
- **ResourceQuota**: Enforces constraints on resource consumption
- **LimitRanger**: Sets default resource limits

### Helm

Helm is a package manager for Kubernetes that simplifies application deployment. The Elastic Stack can be deployed using Helm charts.

```bash
# Add Elastic Helm repository
helm repo add elastic https://helm.elastic.co

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch
```

## Conclusion

Understanding Kubernetes components is essential for effectively deploying and managing the Elastic Stack in a containerized environment. Each component plays a specific role in ensuring that applications run reliably and can be scaled as needed.

In the next section, we'll explore how to set up a Kubernetes cluster suitable for running the Elastic Stack.