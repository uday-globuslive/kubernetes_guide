# Scheduling and Resource Management in Kubernetes

## Introduction

Effective scheduling and resource management are critical components of a well-functioning Kubernetes cluster. The Kubernetes scheduler determines which nodes should run which pods, taking into account resource requirements, hardware/software constraints, and workload characteristics. This document provides a comprehensive overview of Kubernetes scheduling mechanisms, resource management strategies, and advanced configuration options to optimize workload placement.

## The Kubernetes Scheduler

The Kubernetes scheduler is responsible for watching newly created pods and assigning them to nodes. It makes these decisions based on a complex set of factors, including pod resource requirements, node capacity, and various constraints and policies.

### Scheduling Process

The scheduler performs its work through a two-step process:

1. **Filtering**: Identifies the set of nodes where it's feasible to schedule the pod
2. **Scoring**: Ranks the remaining nodes to find the most suitable placement

![Kubernetes Scheduling Process](https://mermaid.ink/img/pako:eNp1kstuwjAQRX_F8qpF4gO67FCpXbQSFULqphviYYjAkWPLj7SIf-_YgQSSspnJPXPHc8f2ESKNCAJYvZLCWdIRw6q2Xfbfd9QYY6VVrHLO5SfDJptRIqGVFtY3yUwG1kpIdNIAF0lyu3SBI2JOssMk4eMF_6SRipNR5cHEDofk49h4fhIZqwAF3ncsM0uF_SKRe5UHJZ7jyJemojRSUqQQrpRG6gDkO-NeDX3KlP_O26eRLCUStHYXiPe6Qh-0BnXCuVYa2aEPvNDd6XV36BZ4jZXhXL9lGy0sdq54XRfnLnYzH3vD-PcwtLv90_gw-pje79YPe7h79Bs_8hy3jBXl_EKcjwjQiwACRnxqOx3xGwImEIXUgiEGLKj8B0u0hVc?type=png)

### Key Scheduler Components

1. **kube-scheduler**: Core component that assigns nodes to pods
2. **PriorityClass**: Defines pod priority relative to other pods
3. **Node affinity/anti-affinity**: Controls pod placement based on node properties
4. **Pod affinity/anti-affinity**: Controls pod placement relative to other pods
5. **Taints and tolerations**: Allow nodes to repel certain pods
6. **Node selectors**: Simple node selection constraints

## Resource Management Fundamentals

### Resource Requests and Limits

Kubernetes uses resource requests and limits to manage compute resources:

- **Requests**: Resources guaranteed to be available to the container
- **Limits**: Maximum resources the container can use

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Resource Units

Kubernetes uses specific units for different resource types:

**CPU Resources**:
- 1 CPU unit = 1 physical or virtual core
- Specified in millicores (m): 500m = 0.5 CPU
- Minimum: 1m (0.001 CPU)

**Memory Resources**:
- Specified using standard SI or binary suffixes
- Examples: Mi (mebibytes), Gi (gibibytes), M (megabytes), G (gigabytes)
- 1 Mi = 1024 Ki, 1 M = 1000 K

**Ephemeral Storage**:
- For container filesystem use
- Uses same units as memory
- Example: "2Gi" ephemeral storage

### Quality of Service (QoS) Classes

Kubernetes assigns QoS classes to pods based on resource configurations:

1. **Guaranteed**:
   - Every container has identical requests and limits
   - Highest priority, least likely to be evicted
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "500m"
     limits:
       memory: "128Mi"
       cpu: "500m"
   ```

2. **Burstable**:
   - At least one container has resource requests, but they don't match limits
   - Medium priority
   ```yaml
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"
       cpu: "500m"
   ```

3. **BestEffort**:
   - No resource requests or limits specified
   - Lowest priority, first to be evicted under resource pressure
   ```yaml
   resources: {}  # No requests or limits
   ```

## Basic Scheduling Techniques

### Node Selectors

The simplest way to constrain pod placement:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    hardware: high-memory
  containers:
  - name: gpu-container
    image: nvidia/cuda:11.0-base
```

To label a node:
```bash
kubectl label nodes node1 hardware=high-memory
```

### Node Name

Explicitly specify a node:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: specific-node-pod
spec:
  nodeName: worker-01
  containers:
  - name: app
    image: nginx
```

### Pod Scheduling Lifecycle

1. **Pod Creation**: Client submits Pod spec to API server
2. **Filtering**: Scheduler filters out unsuitable nodes
3. **Scoring**: Scheduler ranks filtered nodes
4. **Assignment**: Pod assigned to highest-scoring node
5. **Binding**: API server records the binding
6. **Execution**: Kubelet on selected node creates containers

## Advanced Scheduling

### Node Affinity

Provides more expressive syntax than nodeSelector:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
```

Node affinity types:
- **requiredDuringSchedulingIgnoredDuringExecution**: Hard requirement
- **preferredDuringSchedulingIgnoredDuringExecution**: Soft preference, with weights
- **requiredDuringSchedulingRequiredDuringExecution**: (Planned feature) Will force rescheduling when node labels change

Operators include:
- `In`: Value exists in the specified list
- `NotIn`: Value doesn't exist in the specified list
- `Exists`: Key exists
- `DoesNotExist`: Key doesn't exist
- `Gt`: Value is greater than specified value (numeric)
- `Lt`: Value is less than specified value (numeric)

### Pod Affinity and Anti-Affinity

Control pod placement relative to other pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-example
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - frontend
          topologyKey: kubernetes.io/hostname
  containers:
  - name: app
    image: nginx
```

The `topologyKey` is crucial - it defines the scope of the affinity rule:
- `kubernetes.io/hostname`: Same node
- `topology.kubernetes.io/zone`: Same availability zone
- `topology.kubernetes.io/region`: Same region

### Taints and Tolerations

Taints are properties of nodes that repel pods, while tolerations allow pods to schedule onto nodes with matching taints:

**Adding a taint to a node:**
```bash
kubectl taint nodes node1 key=value:effect
```

Where `effect` can be:
- `NoSchedule`: No new pods (without tolerations) will be scheduled
- `PreferNoSchedule`: System tries to avoid placing pods without tolerations
- `NoExecute`: Existing pods without tolerations will be evicted

**Pod with tolerations:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerating-pod
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
  - key: "key2"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 3600  # Optional, only for NoExecute
  containers:
  - name: app
    image: nginx
```

Common use cases:
- Dedicated nodes for specific workloads
- Nodes with specialized hardware
- Gradual node draining for maintenance

### Pod Priority and Preemption

Priority allows certain pods to be scheduled ahead of other pods and enables lower-priority pods to be evicted when needed:

**Define a PriorityClass:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority pods"
```

**Assign priority to a pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

Priority classes allow the scheduler to:
- Schedule higher-priority pods before lower-priority ones
- Preempt (evict) lower-priority pods to make room for higher-priority ones
- Provide ordering in the scheduling queue

## Resource Management Strategies

### Resource Quotas

Resource quotas limit the total resource consumption in a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "25"
    configmaps: "20"
    persistentvolumeclaims: "10"
    services: "15"
```

To check quota usage:
```bash
kubectl describe quota compute-quota -n team-a
```

### Limit Ranges

Limit ranges set default, minimum, and maximum constraints for resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: 50m
      memory: 64Mi
  - type: Pod
    max:
      cpu: "4"
      memory: 4Gi
```

Key benefits:
- Enforces resource constraints in a namespace
- Sets defaults for pods that don't specify their own resources
- Prevents users from creating unreasonably large or small resource allocations

### Vertical Pod Autoscaler (VPA)

VPA automatically adjusts resource requests and limits based on usage:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 10m
        memory: 50Mi
      maxAllowed:
        cpu: "1"
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
```

VPA modes:
- `Off`: Only recommendations, no automatic updates
- `Initial`: Updates at pod creation only
- `Auto`: Updates at pod creation and restarts pods when needed

### Horizontal Pod Autoscaler (HPA)

HPA automatically adjusts pod count based on observed metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

HPA can scale on:
- CPU and memory usage
- Custom metrics
- External metrics

### Cluster Autoscaler

Automatically adjusts the number of nodes in the cluster:

```yaml
# Example for GKE cluster autoscaler configuration
apiVersion: autoscaling.gke.io/v1
kind: ClusterAutoscaler
metadata:
  name: cluster-autoscaler
spec:
  clusterName: my-cluster
  minNodes: 3
  maxNodes: 10
  scaleDownUtilizationThreshold: 0.5
  scaleDownUnneededTime: 10m
  podPriorityThreshold: -10
```

Key features:
- Scales up when pods can't be scheduled due to resource constraints
- Scales down when nodes are underutilized
- Respects pod constraints like node affinity
- Supports multiple node groups and zones

## Advanced Resource Management

### Extended Resources and Device Plugins

Manage specialized hardware like GPUs and FPGAs:

```yaml
# Pod requesting a GPU
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
```

Deploy a device plugin (NVIDIA example):
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - image: nvidia/k8s-device-plugin:v0.12.2
        name: nvidia-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
```

### CPU Management Policies

Control CPU allocation for pods:

```yaml
# kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cpuManagerPolicy: static
reservedSystemCPUs: "0,1"
```

Pod using exclusive CPUs:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-exclusive-pod
spec:
  containers:
  - name: cpu-intensive-app
    image: nginx
    resources:
      requests:
        cpu: 2
      limits:
        cpu: 2  # Must be a whole number and equal to request
```

CPU management policies:
- `none` (default): Conventional CFS quota-based allocation
- `static`: Allows exclusive CPU allocation for Guaranteed QoS pods

### Memory Management

Control memory allocation and avoid OOM (Out of Memory) kills:

```yaml
# kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
memoryManagerPolicy: Static
reservedMemory:
  - numaNode: 0
    limits:
      memory: 1Gi
```

Pod requesting NUMA-aligned memory:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-pod
spec:
  containers:
  - name: memory-demo
    image: nginx
    resources:
      requests:
        memory: 2Gi
      limits:
        memory: 2Gi  # Must be equal to request for guaranteed allocation
```

### Topology Manager

Coordinate NUMA-aware resource allocation:

```yaml
# kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
topologyManagerPolicy: best-effort
```

Topology manager policies:
- `none`: No topology alignment
- `best-effort`: Prefer NUMA alignment, but don't reject pods
- `restricted`: Prefer NUMA alignment, reject pods that can't be aligned
- `single-numa-node`: Require all resources from a single NUMA node

## Scheduling Patterns and Best Practices

### Pattern: High Availability Workload Placement

Distribute pods across failure domains:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-app
spec:
  replicas: 6
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: topology.kubernetes.io/zone
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx
```

This ensures:
- Pods are distributed across different zones (required)
- Pods prefer to run on different nodes (preferred)

### Pattern: Co-location for Data Locality

Place related services together for performance:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
  labels:
    app: data-processor
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
  containers:
  - name: processor
    image: data-processor:v1
```

This ensures:
- Data processor pods run on the same node as cache pods
- Reduces network latency between services

### Pattern: Dedicated Nodes for Specialized Workloads

Create purpose-specific node pools:

```yaml
# Taint nodes with specialized hardware
kubectl taint nodes node-with-gpu dedicated=gpu:NoSchedule

# Deploy workload that requires GPU
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training-job
spec:
  replicas: 3
  template:
    spec:
      tolerations:
      - key: dedicated
        operator: Equal
        value: gpu
        effect: NoSchedule
      containers:
      - name: ml-container
        image: tensorflow/tensorflow:latest-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
```

This ensures:
- GPU nodes are reserved for GPU workloads
- Regular workloads won't be scheduled on expensive GPU nodes

### Best Practices for Resource Requests

1. **Accurate Resource Estimation**:
   - Start with realistic estimates
   - Use Vertical Pod Autoscaler in recommendation mode
   - Tune based on actual observed usage

2. **Set Both Requests and Limits**:
   - Requests ensure minimum resource availability
   - Limits prevent resource hogging
   - Consider setting equal values for critical workloads (Guaranteed QoS)

3. **Memory-to-CPU Ratio**:
   - Different workload types have different needs
   - Java apps: Higher memory-to-CPU ratio 
   - Batch processing: CPU-intensive 
   - Databases: Memory-intensive

4. **Resource Headroom**:
   - Always leave 10-20% capacity for system components
   - Set appropriate resource requests for all system pods

### Best Practices for Scheduling

1. **Use Default Namespace Resource Quotas**:
   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: default-quota
     namespace: default
   spec:
     hard:
       requests.cpu: "1"
       requests.memory: 1Gi
       limits.cpu: "2"
       limits.memory: 2Gi
       pods: "10"
   ```

2. **Set Default Resource Requests with LimitRange**:
   ```yaml
   apiVersion: v1
   kind: LimitRange
   metadata:
     name: default-limit-range
   spec:
     limits:
     - default:
         cpu: 100m
         memory: 256Mi
       defaultRequest:
         cpu: 50m
         memory: 128Mi
       type: Container
   ```

3. **Apply Pod Disruption Budgets for Critical Services**:
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: api-pdb
   spec:
     minAvailable: 2  # or maxUnavailable: 1
     selector:
       matchLabels:
         app: api
   ```

4. **Use Appropriate Topology Spread Constraints**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: spread-pod
   spec:
     topologySpreadConstraints:
     - maxSkew: 1
       topologyKey: topology.kubernetes.io/zone
       whenUnsatisfiable: DoNotSchedule
       labelSelector:
         matchLabels:
           app: web
     - maxSkew: 1
       topologyKey: kubernetes.io/hostname
       whenUnsatisfiable: ScheduleAnyway
       labelSelector:
         matchLabels:
           app: web
     containers:
     - name: web
       image: nginx
   ```

## Advanced Scheduling Scenarios

### Topology Spread Constraints

Evenly distribute pods across topology domains:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: topology-spread-pod
  labels:
    app: web
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
  containers:
  - name: web
    image: nginx
```

Key settings:
- `maxSkew`: Maximum difference between domain with most pods and domain with least pods
- `topologyKey`: The key that defines a domain (zone, node, etc.)
- `whenUnsatisfiable`: What to do when constraint can't be met (`DoNotSchedule` or `ScheduleAnyway`)

### Descheduler

Automatically rebalance pod distribution:

```yaml
apiVersion: kubedescheduler.io/v1alpha1
kind: Descheduler
metadata:
  name: descheduler-policy
  namespace: kube-system
spec:
  strategies:
    RemoveDuplicates:
      enabled: true
    RemovePodsViolatingInterPodAntiAffinity:
      enabled: true
    RemovePodsViolatingNodeAffinity:
      enabled: true
      params:
        nodeAffinityType:
        - requiredDuringSchedulingIgnoredDuringExecution
    LowNodeUtilization:
      enabled: true
      params:
        nodeResourceUtilizationThresholds:
          thresholds:
            cpu: 20
            memory: 20
            pods: 20
          targetThresholds:
            cpu: 50
            memory: 50
            pods: 50
    RemovePodsViolatingTopologySpreadConstraint:
      enabled: true
```

The descheduler helps:
- Rebalance pods after node additions/removals
- Fix violations of pod constraints due to changes in the environment
- Optimize cluster utilization

### Specialized Hardware and Resource Types

Handle workloads with special requirements:

```yaml
# For FPGA workloads
apiVersion: v1
kind: Pod
metadata:
  name: fpga-processing
spec:
  containers:
  - name: fpga-app
    image: fpga-image:latest
    resources:
      limits:
        acme.com/fpga: 1
```

Define custom resources:
```bash
# Add custom resource to a node
kubectl patch node worker-1 -p '{"status":{"capacity":{"example.com/custom-resource":"5"}}}'
```

### Multi-Scheduler Setup

Deploy a secondary scheduler for specialized workloads:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: custom-scheduler
  template:
    metadata:
      labels:
        component: custom-scheduler
    spec:
      containers:
      - command:
        - kube-scheduler
        - --address=0.0.0.0
        - --leader-elect=false
        - --scheduler-name=custom-scheduler
        image: registry.k8s.io/kube-scheduler:v1.27.0
        name: custom-scheduler
```

Use the custom scheduler for a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: custom-scheduler
  containers:
  - name: app
    image: nginx
```

## Resource Optimization Techniques

### Rightsizing Workloads

Optimize resource allocation:

```yaml
# Start with conservative estimates
apiVersion: v1
kind: Pod
metadata:
  name: rightsized-app
spec:
  containers:
  - name: app
    image: my-app:latest
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

Use VPA to collect data and adjust:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"  # Recommendation mode
```

Analyze recommendations:
```bash
kubectl describe vpa app-vpa
```

### Overcommitment Strategies

Define safe overcommitment ratios:

```yaml
# Namespace with high overcommitment
apiVersion: v1
kind: ResourceQuota
metadata:
  name: overcommitted-quota
  namespace: batch-jobs
spec:
  hard:
    requests.cpu: "50"    # Physical cores: 20
    requests.memory: 100Gi  # Physical memory: 64Gi
    limits.cpu: "100"
    limits.memory: 200Gi
```

Appropriate for:
- Batch workloads with variable usage patterns
- Dev/test environments
- Non-critical services

Not recommended for:
- Production databases
- Critical API services
- Real-time processing systems

### Resource Reclamation

Recover resources from idle workloads:

```yaml
# Configure kubelet for better memory management
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
memoryThrottlingFactor: 0.9
evictionHard:
  memory.available: "5%"
  nodefs.available: "10%"
evictionSoft:
  memory.available: "10%"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m"
  nodefs.available: "2m"
evictionMaxPodGracePeriod: 120
```

Implement low priority workloads with preemption:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: reclaim-priority
value: -10
globalDefault: false
description: "Preemptible workloads that can be evicted when resources are needed"
```

## Real-World Scheduling Scenarios

### Case Study 1: Mixed Workload Cluster

Problem: Run both critical stateful applications and batch jobs on the same cluster.

Solution:
```yaml
# Label nodes
kubectl label nodes node1 node2 node3 workload-type=critical
kubectl label nodes node4 node5 node6 workload-type=batch

# Taint batch nodes
kubectl taint nodes node4 node5 node6 workload=batch:NoSchedule

# Critical workload deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 6
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - critical
      containers:
      - name: app
        image: critical-app:v1
        resources:
          requests:
            cpu: 1
            memory: 2Gi
          limits:
            cpu: 2
            memory: 4Gi

# Batch job
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    spec:
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - batch
      containers:
      - name: job
        image: batch-job:v1
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
      restartPolicy: Never
```

### Case Study 2: Multi-Tenant Environment

Problem: Provide fair resource allocation in a shared cluster for multiple teams.

Solution:
```yaml
# Create namespace with quotas for each team
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
---
# Enforce default limits
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    type: Container
---
# Prioritize critical team workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: team-a-critical
value: 1000000
globalDefault: false
description: "Critical workloads for Team A"
---
# Deploy team app with priority
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-a-app
  namespace: team-a
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: team-a-critical
      containers:
      - name: app
        image: team-a-app:v1
```

### Case Study 3: Advanced GPU Scheduling

Problem: Efficiently share expensive GPU resources among multiple projects.

Solution:
```yaml
# Define time-sharing priority classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-training-high
value: 900000
globalDefault: false
description: "High priority GPU training jobs"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-training-medium
value: 800000
globalDefault: false
description: "Medium priority GPU training jobs"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-inference
value: 700000
globalDefault: false
description: "GPU inference workloads"
---
# Define GPU training job
apiVersion: batch/v1
kind: Job
metadata:
  name: model-training
  namespace: ml-workloads
spec:
  template:
    spec:
      priorityClassName: gpu-training-high
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: gpu-type
                operator: In
                values:
                - nvidia-v100
                - nvidia-a100
      containers:
      - name: training
        image: tensorflow/tensorflow:latest-gpu
        resources:
          limits:
            nvidia.com/gpu: 2
            cpu: 8
            memory: 32Gi
          requests:
            nvidia.com/gpu: 2
            cpu: 4
            memory: 16Gi
        command:
        - python
        - /app/train.py
      restartPolicy: Never
```

## Conclusion

Effective scheduling and resource management in Kubernetes requires understanding the interplay between resource requirements, constraints, and workload characteristics. By leveraging the rich set of scheduling features and following best practices, you can optimize your cluster utilization, ensure workload reliability, and maintain performance predictability.

Key takeaways from this guide:

1. Set appropriate resource requests and limits for all workloads
2. Use affinity/anti-affinity rules to control workload placement
3. Implement taints and tolerations for specialized nodes
4. Apply resource quotas and limit ranges at the namespace level
5. Leverage autoscaling features (HPA, VPA, CA) for efficient resource usage
6. Consider node topology for performance-sensitive applications
7. Use priority classes to ensure critical workloads get resources first
8. Regularly monitor and adjust your resource allocation based on actual usage

By mastering these concepts, you can build a robust scheduling strategy that meets your specific application requirements and optimizes your Kubernetes infrastructure.

## Additional Resources

- [Kubernetes Scheduler Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Managing Resources in Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Advanced Scheduling and Node Selection](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Resource Quotas and Limits](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)