# Kubernetes Architecture

Kubernetes (K8s) is a sophisticated orchestration system for containerized applications. This chapter provides a comprehensive overview of Kubernetes architecture, explaining how its components work together to enable scalable, resilient, and manageable container deployments.

## Table of Contents

- [Architectural Overview](#architectural-overview)
- [Control Plane Components](#control-plane-components)
- [Node Components](#node-components)
- [Kubernetes Objects and Resources](#kubernetes-objects-and-resources)
- [Communication Pathways](#communication-pathways)
- [High Availability Architecture](#high-availability-architecture)
- [Networking Architecture](#networking-architecture)
- [Storage Architecture](#storage-architecture)
- [Security Architecture](#security-architecture)
- [Extending Kubernetes](#extending-kubernetes)
- [Architectural Considerations](#architectural-considerations)

## Architectural Overview

Kubernetes follows a master-worker architecture pattern, consisting of a control plane (master) and worker nodes:

```
┌────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                              │
│                                                                     │
│  ┌─────────────────────────────────┐   ┌───────────────────────┐   │
│  │        CONTROL PLANE            │   │       WORKER NODES    │   │
│  │                                 │   │                       │   │
│  │ ┌─────────────┐ ┌─────────────┐ │   │  ┌─────────────────┐  │   │
│  │ │ API Server  │ │ Scheduler   │ │   │  │    Node 1       │  │   │
│  │ └─────────────┘ └─────────────┘ │   │  │ ┌─────┐ ┌─────┐ │  │   │
│  │                                 │   │  │ │Pod 1│ │Pod 2│ │  │   │
│  │ ┌─────────────┐ ┌─────────────┐ │   │  │ └─────┘ └─────┘ │  │   │
│  │ │ Controller  │ │   etcd      │ │   │  └─────────────────┘  │   │
│  │ │  Manager    │ │             │ │   │                       │   │
│  │ └─────────────┘ └─────────────┘ │   │  ┌─────────────────┐  │   │
│  │                                 │   │  │    Node 2       │  │   │
│  └─────────────────────────────────┘   │  │ ┌─────┐ ┌─────┐ │  │   │
│                                        │  │ │Pod 3│ │Pod 4│ │  │   │
│                                        │  │ └─────┘ └─────┘ │  │   │
│                                        │  └─────────────────┘  │   │
│                                        └───────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

Key characteristics of this architecture:

1. **Distributed System**: Multiple components working together
2. **Declarative Model**: Users describe the desired state
3. **Controller Pattern**: Background reconciliation loops
4. **API-Driven**: All interactions go through the API
5. **Loosely Coupled**: Components interact through well-defined interfaces

## Control Plane Components

The control plane is the brain of Kubernetes, responsible for making global decisions about the cluster. It consists of several components that work together:

### API Server (kube-apiserver)

The API server is the front-end interface to the Kubernetes control plane:

- **Central Communication Hub**: All other components communicate through it
- **RESTful API**: Exposes Kubernetes API
- **Authentication & Authorization**: Validates and authorizes requests
- **Data Validation**: Validates API objects
- **RESTful Storage Interface**: Mediates interactions with etcd

```
┌─────────────────────────────────────────────────────────┐
│                    kube-apiserver                        │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│  │ REST API   │  │ Auth/Authz │  │ Object Validation  │ │
│  └────────────┘  └────────────┘  └────────────────────┘ │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│  │ Admission  │  │ etcd       │  │ Conversion/Versioned│ │
│  │ Control    │  │ Interface  │  │ API Support        │ │
│  └────────────┘  └────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### etcd

etcd is a distributed key-value store that serves as Kubernetes' primary datastore:

- **Consistent & Highly-Available**: Uses Raft consensus algorithm
- **Source of Truth**: Stores all cluster data (state, configuration, metadata)
- **Watch Mechanism**: Enables components to respond to state changes
- **Strong Consistency**: Guarantees data integrity
- **Security Features**: TLS, authentication, encryption at rest

### Scheduler (kube-scheduler)

The scheduler assigns newly created pods to nodes:

- **Resource Requirements**: Considers CPU, memory, etc.
- **Constraints**: Honors node selectors, affinity rules, taints, tolerations
- **Topology Awareness**: Understands racks, zones, regions
- **Scoring System**: Ranks nodes by suitability
- **Extensible**: Supports custom scheduling logic

Scheduling process:
```
1. Pod Creation ─┐
                 │
                 ▼
2. Filtering (Predicates) ─┐
   - Resource requirements │
   - Node selectors        │
   - Taints/tolerations    │
                          │
                          ▼
3. Scoring (Priorities) ──┐
   - Least loaded node    │
   - Topology spread      │
   - Affinity/anti-affinity
                          │
                          ▼
4. Node Selection ────────┐
                          │
                          ▼
5. Binding to Node
```

### Controller Manager (kube-controller-manager)

The controller manager runs controller processes that regulate the state of the system:

- **Node Controller**: Monitors node health
- **Replication Controller**: Ensures the correct number of pods
- **Endpoints Controller**: Populates endpoints objects
- **Service Account & Token Controllers**: Create accounts and API tokens
- **And many others**: Daemon, Job, Namespace controllers, etc.

Each controller follows a simple pattern:
```
┌────────────────────┐
│    Controller      │
│                    │
│  ┌──────────────┐  │
│  │  Observe     │  │
│  │  current     │──┼──▶ Watch API objects
│  │  state       │  │
│  └──────────────┘  │
│         │          │
│         ▼          │
│  ┌──────────────┐  │
│  │  Determine   │  │
│  │  differences │  │
│  └──────────────┘  │
│         │          │
│         ▼          │
│  ┌──────────────┐  │
│  │  Take action │──┼──▶ Update API objects
│  │  to reconcile│  │
│  └──────────────┘  │
└────────────────────┘
```

### Cloud Controller Manager (cloud-controller-manager)

Cloud Controller Manager integrates with cloud providers:

- **Node Controller**: Updates node status with cloud provider info
- **Route Controller**: Sets up routes in cloud infrastructure
- **Service Controller**: Manages cloud load balancers
- **Volume Controller**: Creates, attaches, and mounts cloud storage

## Node Components

Worker nodes are the machines that run containerized applications. Each node contains:

### Kubelet

The kubelet is an agent that runs on each node:

- **Pod Lifecycle Management**: Creates, monitors, and terminates containers
- **API Server Communication**: Reports node and pod status
- **Container Runtime Interface (CRI)**: Interfaces with container runtime
- **Volume Management**: Mounts volumes for pods
- **Health Monitoring**: Ensures containers are running

```
┌────────────────────────────────────────────────┐
│                   Kubelet                       │
│                                                │
│  ┌─────────────┐  ┌───────────────────────┐   │
│  │ Pod         │  │ Node Status Reporting │   │
│  │ Lifecycle   │  │                       │   │
│  └─────────────┘  └───────────────────────┘   │
│                                                │
│  ┌─────────────┐  ┌───────────────────────┐   │
│  │ Volume      │  │ Container Runtime     │   │
│  │ Management  │  │ Interface (CRI)       │   │
│  └─────────────┘  └───────────────────────┘   │
│                                                │
│  ┌─────────────┐  ┌───────────────────────┐   │
│  │ cAdvisor    │  │ Image/Secret Pull     │   │
│  │ Monitoring  │  │                       │   │
│  └─────────────┘  └───────────────────────┘   │
└────────────────────────────────────────────────┘
```

### Container Runtime

The container runtime is the software responsible for running containers:

- **Image Pulling**: Fetches container images
- **Container Execution**: Runs containers
- **Resource Isolation**: Uses namespaces, cgroups
- **Networking Setup**: Configures container network interfaces
- **Storage Provisioning**: Handles container storage

Common container runtimes:
- **containerd**: Lightweight runtime (default in most Kubernetes distributions)
- **CRI-O**: Kubernetes-optimized runtime
- **Docker**: Still supported via dockershim (deprecated) or cri-dockerd

### kube-proxy

kube-proxy maintains network rules on nodes:

- **Service Abstraction**: Implements Kubernetes Service concept
- **Connection Forwarding**: Routes traffic to appropriate pods
- **Load Balancing**: Distributes traffic across pods
- **IP Tables/IPVS**: Manipulates host network rules

Implementation modes:
- **IPTables Mode**: Uses iptables rules for traffic redirection
- **IPVS Mode**: Linux IP Virtual Server for more efficient load balancing
- **Userspace Mode**: Legacy mode, rarely used

## Kubernetes Objects and Resources

Kubernetes uses various objects to represent the state of the system:

### Core Objects

- **Pod**: Basic unit of deployment containing one or more containers
- **Service**: Network abstraction for pod access
- **Volume**: Storage abstraction
- **Namespace**: Logical partitioning of resources

### Controllers

- **Deployment**: Manages pod rollout and updates
- **StatefulSet**: For stateful applications
- **DaemonSet**: Ensures a pod runs on each node
- **Job/CronJob**: For batch and scheduled tasks
- **ReplicaSet**: Ensures a specific number of pod replicas

### Configuration

- **ConfigMap**: Non-confidential configuration
- **Secret**: Confidential information
- **ResourceQuota**: Limits resources in a namespace
- **HorizontalPodAutoscaler**: Automatically scales pod replicas

Resource relationships:
```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  Deployment   │────▶│  ReplicaSet   │────▶│     Pod       │
└───────────────┘     └───────────────┘     └───────────────┘
                                                   │
┌───────────────┐                                  │
│ ConfigMap/    │                                  │
│ Secret        │◀─────────────────────────────────┘
└───────────────┘
        ▲
        │
┌───────────────┐     ┌───────────────┐
│   Volume      │◀────│    Service    │
└───────────────┘     └───────────────┘
```

## Communication Pathways

Kubernetes has several communication paths:

### Internal Cluster Communication

1. **Control Plane to Node**:
   - API server to kubelets (HTTPS)
   - API server to nodes, pods, services (HTTPS/HTTP)

2. **Control Plane Internal**:
   - etcd to API server (HTTPS)
   - Controller Manager & Scheduler to API server (HTTPS)

3. **Node-Level**:
   - Kubelet to API server (HTTPS)
   - Kubelet to container runtime (CRI/Unix sockets)
   - Kube-proxy to API server (HTTPS)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Control Plane                            │
│                                                                 │
│   ┌──────────┐      ┌────────────┐      ┌─────────────┐        │
│   │ etcd     │◀────▶│ API Server │◀────▶│ Controller  │        │
│   └──────────┘      └─────┬──────┘      │ Manager     │        │
│                           │             └─────────────┘        │
│                           │                                    │
│                           │             ┌─────────────┐        │
│                           └────────────▶│ Scheduler   │        │
│                                         └─────────────┘        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                HTTPS       │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Node                                   │
│                                                                 │
│   ┌──────────┐      ┌────────────┐      ┌─────────────┐        │
│   │ kubelet  │◀────▶│ Container  │      │ kube-proxy  │        │
│   └────┬─────┘      │ Runtime    │      └─────────────┘        │
│        │            └────────────┘                             │
│        │                                                       │
│        │            ┌────────────┐                             │
│        └───────────▶│ Pods       │                             │
│                     └────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

### External Communication

- **User to Kubernetes API**: kubectl, client libraries, web UIs
- **External to Services**: Ingress controllers, LoadBalancer services
- **Node to External Services**: Image pulling, external APIs

## High Availability Architecture

Production Kubernetes clusters implement high availability:

### Control Plane HA

- **Multiple API Servers**: Load balanced
- **etcd Cluster**: Multi-node (typically 3, 5, or 7 nodes)
- **Multiple Controller Managers**: Active-standby mode
- **Multiple Schedulers**: Active-standby mode

```
┌──────────────────────────────────────────────────────────────────┐
│                      HA Control Plane                            │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                     │
│  │ etcd-1   │   │ etcd-2   │   │ etcd-3   │                     │
│  └──────────┘   └──────────┘   └──────────┘                     │
│                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│  │ API Server-1 │ │ API Server-2 │ │ API Server-3 │             │
│  └──────────────┘ └──────────────┘ └──────────────┘             │
│                                                                  │
│  ┌──────────────────┐           ┌─────────────────┐             │
│  │ Controller       │◀─Locks────▶│ Controller     │             │
│  │ Manager-1(active)│           │ Manager-2(standby)            │
│  └──────────────────┘           └─────────────────┘             │
│                                                                  │
│  ┌──────────────────┐           ┌─────────────────┐             │
│  │ Scheduler-1      │◀─Locks────▶│ Scheduler-2    │             │
│  │ (active)         │           │ (standby)       │             │
│  └──────────────────┘           └─────────────────┘             │
└──────────────────────────────────────────────────────────────────┘
```

### Worker Node HA

- **Multiple Nodes**: Across availability zones if possible
- **Node Auto-Repair**: Automatically replace failed nodes
- **Pod Distribution**: Spread using pod anti-affinity
- **Pod Disruption Budgets**: Ensure minimum replica count during disruptions

## Networking Architecture

Kubernetes networking has several key components:

### Pod Networking

Every pod gets a unique IP address in a flat networking space:

- **Container Network Interface (CNI)**: Plugin architecture for networking
- **Pod-to-Pod Communication**: Direct communication between pods
- **Network Policies**: Firewall rules for pod traffic

### Service Networking

Services provide stable endpoints for pods:

- **ClusterIP**: Internal-only virtual IP
- **NodePort**: Exposes service on each node's IP
- **LoadBalancer**: Provisions external load balancer
- **ExternalName**: DNS CNAME record

```
┌──────────────────────────────────────────────────────────────────┐
│                      Kubernetes Networking                       │
│                                                                  │
│  ┌──────────┐                             ┌──────────┐           │
│  │ Pod A    │                             │ Pod B    │           │
│  │ IP: 10.1.1.2                          │ IP: 10.1.2.3         │
│  └────┬─────┘                             └────┬─────┘           │
│       │                                        │                 │
│       │                                        │                 │
│       └─────────────┐               ┌──────────┘                 │
│                     ▼               ▼                            │
│               ┌─────────────────────────────┐                    │
│               │       Pod Network           │                    │
│               └────────────┬────────────────┘                    │
│                            │                                     │
│                            │                                     │
│                    ┌───────┴──────┐                              │
│                    │              │                              │
│                    ▼              ▼                              │
│           ┌─────────────┐ ┌────────────────┐                     │
│           │ Service A   │ │ Service B      │                     │
│           │ ClusterIP   │ │ LoadBalancer   │                     │
│           └──────┬──────┘ └────────┬───────┘                     │
│                  │                 │                             │
│                  │                 │                             │
│         ┌────────┘                 └──────────┐                  │
│         │                                     │                  │
│         ▼                                     ▼                  │
│  ┌─────────────┐                     ┌──────────────────┐        │
│  │ Internal    │                     │ External         │        │
│  │ Clients     │                     │ Clients          │        │
│  └─────────────┘                     └──────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### Ingress

Ingress controllers provide HTTP/HTTPS routing into the cluster:

- **Ingress Resources**: Define routing rules
- **Ingress Controllers**: Implementations (Nginx, HAProxy, etc.)
- **SSL Termination**: Manage certificates and encryption
- **Path-Based Routing**: Route traffic by URL path
- **Host-Based Routing**: Route traffic by hostname

## Storage Architecture

Kubernetes abstracts storage through several layers:

### Storage Concepts

- **Volumes**: Pod-level storage abstraction
- **PersistentVolumes (PV)**: Cluster-level storage resource
- **PersistentVolumeClaims (PVC)**: Request for storage
- **StorageClasses**: Automated PV provisioning
- **Container Storage Interface (CSI)**: Plugin system for storage providers

```
┌──────────────────────────────────────────────────────────────────┐
│                     Kubernetes Storage                           │
│                                                                  │
│  ┌────────────────┐         ┌────────────────┐                   │
│  │ StorageClass   │───┐     │ StorageClass   │                   │
│  │ (Standard)     │   │     │ (Fast)         │                   │
│  └────────────────┘   │     └────────────────┘                   │
│                       ▼                                          │
│               ┌───────────────┐                                  │
│               │ Persistent    │                                  │
│               │ Volume        │                                  │
│               └───────┬───────┘                                  │
│                       │                                          │
│                       │Claims                                    │
│                       ▼                                          │
│               ┌───────────────┐                                  │
│               │ Persistent    │                                  │
│               │ Volume Claim  │                                  │
│               └───────┬───────┘                                  │
│                       │Mounts                                    │
│                       ▼                                          │
│               ┌───────────────┐                                  │
│               │     Pod       │                                  │
│               │               │                                  │
│               └───────────────┘                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Volume Types

Kubernetes supports various volume types:

- **emptyDir**: Temporary space in a pod
- **hostPath**: Mount from node filesystem
- **nfs/cifs/iscsi**: Network storage protocols
- **Cloud Volumes**: EBS, Azure Disk, GCE PD
- **CSI Volumes**: Storage from third-party providers via CSI

## Security Architecture

Kubernetes security is multi-layered:

### Authentication

- **Client Certificates**: TLS client certs
- **Bearer Tokens**: JWT/OIDC tokens
- **Authentication Webhook**: External auth services
- **Service Accounts**: For in-cluster workloads

### Authorization

- **RBAC**: Role-Based Access Control
- **Node Authorization**: Special authorizer for kubelet
- **Webhook**: External authorization services
- **ABAC**: Attribute-Based Access Control (legacy)

```
┌──────────────────────────────────────────────────────────┐
│                Kubernetes Security                       │
│                                                          │
│  ┌─────────────┐     ┌─────────────┐    ┌─────────────┐  │
│  │             │     │             │    │             │  │
│  │Authentication────▶│Authorization────▶│Admission    │  │
│  │             │     │             │    │Control      │  │
│  └─────────────┘     └─────────────┘    └─────────────┘  │
│                                                          │
│  ┌─────────────┐     ┌─────────────┐    ┌─────────────┐  │
│  │ Secret      │     │ Network     │    │ Pod/Container│  │
│  │ Management  │     │ Policies    │    │ Security     │  │
│  └─────────────┘     └─────────────┘    └─────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Network Security

- **Network Policies**: Pod-level firewall rules
- **TLS**: Encryption for control plane communication
- **API Server**: Secure endpoints and authentication
- **Private Clusters**: Limit external access

### Pod/Container Security

- **Pod Security Standards**: Define security profiles for pods
- **Security Contexts**: Set container permissions
- **Seccomp/AppArmor/SELinux**: Kernel security features
- **Image Scanning**: Check for vulnerabilities
- **Secrets Management**: Protect sensitive data

## Extending Kubernetes

Kubernetes is designed to be extensible:

### API Extensions

- **Custom Resources (CRDs)**: Define new resource types
- **API Aggregation**: Run additional API servers

### Custom Controllers

- **Operators**: Application-specific controllers
- **Admission Controllers**: Validate/modify API requests
- **Initializers**: Process resources before they become visible

```
┌──────────────────────────────────────────────────────────┐
│               Extending Kubernetes                       │
│                                                          │
│  ┌─────────────┐     ┌─────────────┐    ┌─────────────┐  │
│  │ Custom      │     │ Custom      │    │ Webhooks    │  │
│  │ Resources   │────▶│ Controllers │◀───│             │  │
│  └─────────────┘     └─────────────┘    └─────────────┘  │
│                                                          │
│  ┌─────────────┐     ┌─────────────┐    ┌─────────────┐  │
│  │ API         │     │ Device      │    │ Scheduler   │  │
│  │ Aggregation │     │ Plugins     │    │ Extensions  │  │
│  └─────────────┘     └─────────────┘    └─────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Plugins

- **Container Runtimes**: Via CRI
- **Network Plugins**: Via CNI
- **Storage Plugins**: Via CSI
- **Device Plugins**: For specialized hardware
- **Scheduler Extensions**: Custom scheduling logic

## Architectural Considerations

When working with Kubernetes, consider these architectural factors:

### Scalability Limits

- **Node Counts**: Practical limits (1000-5000 nodes)
- **Pod Density**: Pods per node (typically 100-250)
- **Control Plane Sizing**: CPU/memory requirements
- **etcd Performance**: Storage and network requirements

### Failure Domains

- **Node Failures**: Pods rescheduled automatically
- **Zone Failures**: Multi-zone deployments
- **Region Failures**: Multi-region considerations
- **Control Plane Failures**: HA control plane

### Operational Complexity

- **Management Overhead**: Expertise required
- **Observability**: Monitoring, logging, tracing
- **Backup and Recovery**: etcd backups, disaster recovery
- **Upgrade Procedures**: Version management

## Summary

Kubernetes architecture is a complex but well-structured system that enables container orchestration at scale. Its key strengths include:

- **Declarative Configuration**: Describe what, not how
- **Self-Healing**: Automatically handles failures
- **Scaling**: Horizontal scaling of applications
- **Service Discovery**: Automatic service registration and discovery
- **Load Balancing**: Distribute traffic across pods
- **Storage Orchestration**: Automated provisioning and attachment
- **Automated Rollouts/Rollbacks**: Controlled deployment strategies
- **Extensibility**: Adapt to different needs

Understanding this architecture is essential for effectively deploying, operating, and troubleshooting Kubernetes environments.

## Further Reading

- [Kubernetes Architecture Documentation](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Design Principles](https://kubernetes.io/docs/concepts/architecture/principles/)
- [etcd Documentation](https://etcd.io/docs/)