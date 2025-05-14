# Autoscaling Strategies

Autoscaling is a critical feature in Kubernetes that allows your applications to handle varying loads efficiently. This guide covers different autoscaling strategies, their implementation, and best practices for optimizing resource utilization and application performance.

## Introduction to Kubernetes Autoscaling

Kubernetes offers several autoscaling options that operate at different levels of the infrastructure:

1. **Horizontal Pod Autoscaler (HPA)**: Scales the number of pod replicas
2. **Vertical Pod Autoscaler (VPA)**: Adjusts CPU and memory resources for pods
3. **Cluster Autoscaler (CA)**: Scales the number of nodes in the cluster

Each autoscaler serves different purposes and can work together to provide comprehensive scaling capabilities.

## Horizontal Pod Autoscaler (HPA)

The Horizontal Pod Autoscaler automatically scales the number of pods in a deployment, replicaset, or statefulset based on observed CPU utilization, memory usage, or custom metrics.

### How HPA Works

1. HPA periodically checks metrics from the Metrics API
2. It calculates the desired number of replicas based on current vs. target metric values
3. It updates the replica count to meet the target utilization

```
                                   ┌──────────────┐
                                   │              │
                 ┌────────────────►│  Metrics     │
                 │                 │  Server      │
                 │                 │              │
┌──────────────┐ │                 └──────────────┘
│              │ │                        ▲
│  Horizontal  │ │                        │
│  Pod         │ │                        │ Metrics
│  Autoscaler  │ │                        │
│              │ │                        │
└──────────────┘ │                 ┌──────────────┐
        │        │                 │              │
        │        │  Scale          │  Application │
        │        │  Replicas       │  Pods        │
        └────────┴────────────────►│              │
                                   └──────────────┘
```

### Basic HPA Example

To create an HPA based on CPU utilization:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

This HPA will maintain CPU utilization at around 80% by scaling between 2 and 10 replicas.

### Multi-Metric HPA

You can configure HPA to consider multiple metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Custom and External Metrics

HPA can scale based on custom or external metrics from your applications or infrastructure:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-processor-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-processor
  minReplicas: 1
  maxReplicas: 30
  metrics:
  - type: Pods
    pods:
      metric:
        name: messages_per_second
      target:
        type: AverageValue
        averageValue: 100
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue_name: "workqueue"
      target:
        type: AverageValue
        averageValue: 30
```

This HPA scales based on:
- A custom metric measuring messages processed per second
- An external metric tracking messages in a queue

### Setting Up Metrics Server

HPA requires the Metrics Server to be installed for basic CPU and memory metrics:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify installation:

```bash
kubectl get deployment metrics-server -n kube-system
```

### HPA Behavior Configuration

In Kubernetes 1.18+, you can configure scaling behavior to prevent thrashing:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 4
        periodSeconds: 60
      selectPolicy: Max
```

This configuration:
- Waits 5 minutes before considering scale down (stabilization window)
- Scales down by at most 10% of current replicas per minute
- Scales up immediately (no stabilization window)
- Scales up by either 100% of current replicas or 4 pods per minute, whichever is greater

## Vertical Pod Autoscaler (VPA)

The Vertical Pod Autoscaler automatically adjusts the CPU and memory resource requests and limits of containers based on their usage patterns.

### How VPA Works

1. VPA collects resource usage metrics from pods
2. It analyzes usage patterns and recommends optimal CPU and memory settings
3. It can apply changes automatically or provide recommendations only

```
   ┌──────────────┐
   │              │◄───┐
   │  Recommender │    │
   │              │    │
   └──────────────┘    │
           │           │
           ▼           │
   ┌──────────────┐    │   ┌──────────────┐
   │              │    │   │              │
   │  Admission   │    └───┤   Metrics    │
   │  Controller  │        │   Server     │
   │              │        │              │
   └──────────────┘        └──────────────┘
           │                      ▲
           ▼                      │
   ┌──────────────┐               │
   │              │               │
   │  Pods with   ├───────────────┘
   │  adjusted    │
   │  resources   │
   │              │
   └──────────────┘
```

### Installing VPA

To use VPA, you need to install its components:

```bash
# Clone the VPA repository
git clone https://github.com/kubernetes/autoscaler.git

# Navigate to the VPA directory
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh
```

### VPA Usage Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: redis-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: redis
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
```

This VPA:
- Automatically adjusts resources for the redis deployment
- Sets minimum and maximum boundaries for resources
- Applies to all containers in the pod (using wildcarded containerName)

### VPA Modes

VPA supports three different modes:

1. **Auto**: Automatically updates pod resources (requires pod restart)
   ```yaml
   updatePolicy:
     updateMode: "Auto"
   ```

2. **Initial**: Only sets resources when pods are first created
   ```yaml
   updatePolicy:
     updateMode: "Initial"
   ```

3. **Off**: Provides recommendations but doesn't apply them
   ```yaml
   updatePolicy:
     updateMode: "Off"
   ```

### VPA Recommendations

You can view VPA recommendations:

```bash
kubectl describe vpa redis-vpa
```

Example output:
```
Status:
  Conditions:
    Last Transition Time:  2022-03-14T12:00:00Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  redis
      Lower Bound:
        Cpu:     100m
        Memory:  50Mi
      Target:
        Cpu:     150m
        Memory:  200Mi
      Upper Bound:
        Cpu:     300m
        Memory:  300Mi
```

### VPA Limitations

- VPA requires pod restarts to apply new resource settings in "Auto" mode
- It cannot be used together with HPA for the same metric (e.g., CPU)
- It may conflict with node resource constraints

## Cluster Autoscaler (CA)

The Cluster Autoscaler automatically adjusts the size of your Kubernetes cluster when:
- Pods fail to schedule due to insufficient resources
- Nodes are underutilized and their pods can be rescheduled elsewhere

### How CA Works

1. CA regularly checks for unschedulable pods
2. If it finds pods that can't be scheduled due to resource constraints, it adds nodes
3. It also looks for underutilized nodes and removes them if all pods can be rescheduled

```
             ┌──────────────┐
             │              │
             │  Cluster     │
             │  Autoscaler  │
             │              │
             └──────────────┘
                 │      ▲
     Check for   │      │  Scale up or
    unschedulable│      │  down nodes
         pods    │      │
                 ▼      │
┌─────────────────────────────────────┐
│           Kubernetes Cluster         │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐
│  │         │  │         │  │         │
│  │ Node 1  │  │ Node 2  │  │ Node 3  │
│  │         │  │         │  │         │
│  └─────────┘  └─────────┘  └─────────┘
│                                     │
└─────────────────────────────────────┘
```

### Setting Up Cluster Autoscaler

Configuration depends on your cloud provider:

#### AWS EKS

```bash
# Create IAM policy for CA
cat > cluster-autoscaler-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EOF

# Create policy
aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
    --cluster=my-cluster \
    --namespace=kube-system \
    --name=cluster-autoscaler \
    --attach-policy-arn=arn:aws:iam::123456789012:policy/AmazonEKSClusterAutoscalerPolicy \
    --override-existing-serviceaccounts \
    --approve

# Deploy Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Add cluster name annotation
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
    cluster-autoscaler.kubernetes.io/safe-to-evict="false"

# Set your cluster name
kubectl -n kube-system set image deployment.apps/cluster-autoscaler \
    cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.0
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

Add `--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster` to the command args.

#### GKE

GKE has built-in support for Cluster Autoscaler:

```bash
gcloud container clusters update my-cluster \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=10 \
    --zone=us-central1-a
```

#### AKS

AKS also has built-in support:

```bash
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 10
```

### Configure CA Parameters

You can adjust the behavior of Cluster Autoscaler:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --scan-interval=30s
        - --scale-down-delay-after-add=10m
        - --scale-down-delay-after-delete=10m
        - --scale-down-delay-after-failure=3m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
```

Key parameters:
- `scan-interval`: How often CA checks for scaling needs
- `scale-down-unneeded-time`: How long a node must be underutilized before it's considered for scale down
- `scale-down-utilization-threshold`: Threshold for node underutilization (0.5 = 50%)
- `scale-down-delay-after-add`: Wait time after adding nodes before considering scale down

### Testing Cluster Autoscaler

Create a deployment that exceeds current cluster capacity:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ca-test
spec:
  replicas: 50
  selector:
    matchLabels:
      app: ca-test
  template:
    metadata:
      labels:
        app: ca-test
    spec:
      containers:
      - name: ca-test
        image: nginx
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
```

Verify that CA adds nodes:

```bash
kubectl get nodes --watch
```

## Combining Autoscaling Approaches

For a comprehensive autoscaling strategy, you can combine different autoscalers:

### HPA + CA

This is the most common combination:
- HPA scales the pods based on metrics
- CA ensures enough nodes are available for the pods

```yaml
# HPA for a web application
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

With Cluster Autoscaler configured, it will automatically add nodes if the scaled pods can't fit on existing nodes.

### VPA + CA

Use VPA for right-sizing pods and CA for ensuring node capacity:

```yaml
# VPA for a database
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: database
  updatePolicy:
    updateMode: "Auto"
```

This approach is useful for:
- Stateful applications that can't easily scale horizontally
- Batch jobs that need right-sized resources
- Applications with unpredictable resource usage

### HPA + VPA with Different Metrics

You can use HPA and VPA together if they target different metrics:

```yaml
# HPA based on custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
```

```yaml
# VPA for right-sizing memory
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      controlledResources: ["memory"]
```

Here, HPA scales based on custom metric `requests_per_second`, while VPA only adjusts memory resources.

## Advanced Autoscaling Strategies

### Predictive Scaling

While Kubernetes doesn't natively support predictive scaling, you can implement it using custom solutions:

1. Collect metrics over time
2. Analyze patterns (e.g., time-of-day patterns, weekly patterns)
3. Pre-scale before anticipated load increases

Example tools:
- Prometheus + KEDA for metric-based predictions
- Custom controllers that implement predictive algorithms

### KEDA (Kubernetes Event-Driven Autoscaling)

KEDA extends Kubernetes autoscaling with event-based scaling for event-driven workloads:

```bash
# Install KEDA
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.8.0/keda-2.8.0.yaml
```

Example of using KEDA with RabbitMQ:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: consumer
  pollingInterval: 15
  cooldownPeriod: 30
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: rabbitmq
    metadata:
      queueName: tasks
      host: amqp://guest:guest@rabbitmq:5672
      queueLength: "50"
```

This will scale the consumer deployment based on the number of messages in the RabbitMQ queue.

### Proportional Autoscaling

Scale related services proportionally:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  triggers:
  - type: kubernetes-workload
    metadata:
      workloadName: backend
      namespace: default
      podSelector: 'app=backend'
      threshold: '0.7'
```

This scales the frontend in proportion to the backend (with 0.7 ratio).

### Multi-dimensional Scaling

Use multiple metrics to make scaling decisions:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
```

The HPA will scale based on whichever metric requires the highest replica count.

## Real-World Autoscaling Examples

### Web Application with Autoscaling

```yaml
# Frontend deployment with HPA
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: my-frontend:latest
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        ports:
        - containerPort: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 60
```

### Batch Processing with KEDA

```yaml
# Batch job processor scaled based on queue length
apiVersion: apps/v1
kind: Deployment
metadata:
  name: job-processor
spec:
  replicas: 0  # KEDA will manage replicas
  selector:
    matchLabels:
      app: job-processor
  template:
    metadata:
      labels:
        app: job-processor
    spec:
      containers:
      - name: processor
        image: job-processor:latest
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1Gi
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: job-processor-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: job-processor
  pollingInterval: 15
  cooldownPeriod: 30
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: job-processor
      topic: jobs-to-process
      lagThreshold: "50"
```

### Database with VPA

```yaml
# Database StatefulSet with VPA
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: postgres
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 500m
        memory: 1Gi
      maxAllowed:
        cpu: 4
        memory: 8Gi
```

## Best Practices

### Performance Testing for Autoscaling

1. **Establish baselines**: Determine normal resource usage patterns
2. **Load test**: Generate realistic load to test scaling triggers
3. **Step pattern testing**: Increase load in steps to test scaling responsiveness 
4. **Spike testing**: Suddenly increase load to test how quickly scaling responds
5. **Endurance testing**: Test with sustained load to observe long-term behavior

### Optimizing HPA Settings

1. **Set appropriate metric targets**:
   - CPU: 50-80% for most workloads
   - Memory: 70-90% (memory doesn't scale down as efficiently)

2. **Configure behavior settings**:
   - Use longer stabilization windows for scale down (3-5 minutes)
   - Use shorter stabilization windows for scale up (0-1 minute)
   - Limit scale down rate to avoid service disruption

3. **Choose the right metrics**:
   - Use application-specific metrics when possible
   - Consider queue length, request rate, or error rate metrics

### Optimizing VPA Settings

1. **Start in recommendation mode**:
   ```yaml
   updatePolicy:
     updateMode: "Off"
   ```

2. **Set reasonable min/max boundaries**:
   ```yaml
   containerPolicies:
   - containerName: '*'
     minAllowed:
       cpu: 50m
       memory: 64Mi
     maxAllowed:
       cpu: 2
       memory: 4Gi
   ```

3. **Use Initial mode for stateful applications**:
   ```yaml
   updatePolicy:
     updateMode: "Initial"
   ```

### Optimizing Cluster Autoscaler

1. **Set appropriate buffer capacity**:
   - Keep some spare capacity to handle sudden increases in load
   - Use `--scale-down-utilization-threshold=0.5` for moderate buffer

2. **Configure scale-down delays**:
   - Use longer delays after scaling up (`--scale-down-delay-after-add=10m`)
   - Use shorter delays after scaling down (`--scale-down-delay-after-delete=5m`)

3. **Use node groups for different workloads**:
   - Separate node groups for CPU-intensive vs. memory-intensive workloads
   - Use node labels and affinity rules to place pods appropriately

### General Autoscaling Tips

1. **Use resource requests and limits consistently**
   - All containers should have requests specified
   - Set realistic limits based on actual usage

2. **Be aware of pod disruption during scaling**
   - Use PodDisruptionBudgets to maintain availability
   - Set appropriate terminationGracePeriodSeconds

   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: frontend-pdb
   spec:
     minAvailable: 2  # or use maxUnavailable
     selector:
       matchLabels:
         app: frontend
   ```

3. **Monitor autoscaling events**
   - Log autoscaling decisions
   - Set up alerts for scaling failures

   ```bash
   kubectl get events --sort-by='.lastTimestamp' | grep -i "horizontalpodautoscaler\|verticalpodautoscaler\|clusterautoscaler"
   ```

4. **Consider cost implications**
   - Balance performance against cost
   - Use spot/preemptible instances when appropriate
   - Set reasonable maxReplicas

## Troubleshooting Autoscaling

### Common HPA Issues

1. **HPA not scaling up**
   - Check if metrics are being collected
     ```bash
     kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods"
     ```
   - Verify target utilization isn't set too high
   - Check if maxReplicas is reached

2. **HPA scaling too aggressively**
   - Adjust stabilization window
     ```yaml
     behavior:
       scaleUp:
         stabilizationWindowSeconds: 300
     ```
   - Increase metric target value
   - Limit scaling rate with policies

3. **Custom metrics not working**
   - Verify custom metrics adapter is installed
   - Check metrics availability
     ```bash
     kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
     ```

### Common VPA Issues

1. **VPA recommendations not appearing**
   - Check VPA is correctly targeting the workload
   - Verify metrics server is working
   - Give VPA time to collect usage data (24+ hours)

2. **Pods not updating with new resource recommendations**
   - Confirm updateMode is set to "Auto"
   - Check if pods are being recreated

3. **Resource recommendations too conservative**
   - Use longer history window
   - Adjust resource policy settings

### Common Cluster Autoscaler Issues

1. **Cluster not scaling up**
   - Check for scheduling errors in events
     ```bash
     kubectl get events | grep -i "scale up"
     ```
   - Verify CA has appropriate permissions
   - Check if node groups have appropriate max size

2. **Cluster not scaling down**
   - Look for pods preventing node deletion
   - Check scale-down delay settings
   - Verify utilization threshold settings

3. **Scaling taking too long**
   - Optimize node provisioning time
   - Adjust scan-interval settings
   - Consider using node pools with pre-created instances

### Debugging Autoscaling

Use these commands for troubleshooting:

```bash
# Check HPA status
kubectl describe hpa <hpa-name>

# Check VPA status and recommendations
kubectl describe vpa <vpa-name>

# Check cluster autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler

# View all scaling events
kubectl get events --sort-by='.lastTimestamp' | grep -i "scaling"
```

## Conclusion

Kubernetes autoscaling provides powerful mechanisms to efficiently manage application resources. By combining different autoscaling strategies—horizontal, vertical, and cluster autoscaling—you can build a comprehensive scaling solution tailored to your application's needs.

Key takeaways:
- HPA is best for stateless applications with variable load
- VPA works well for applications with unpredictable resource needs
- Cluster Autoscaler ensures infrastructure matches application demands
- Combined approaches provide the most comprehensive solution

Start with simple configurations and metrics, monitor behavior, and refine your autoscaling strategy over time based on real-world performance and cost data.