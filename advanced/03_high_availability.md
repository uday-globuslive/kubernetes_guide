# High Availability Patterns in Kubernetes

## Introduction

High Availability (HA) is a critical aspect of production-grade Kubernetes deployments, especially for mission-critical applications like the ELK Stack. HA refers to a system's ability to remain operational and accessible during component failures or planned maintenance. This guide explores various high availability patterns in Kubernetes and how to apply them to ELK Stack deployments to ensure resilience, uptime, and reliability.

## Core Kubernetes HA Concepts

### Control Plane High Availability

The Kubernetes control plane consists of several key components:

- **kube-apiserver**: The API server that serves as the frontend for the Kubernetes control plane
- **etcd**: A distributed key-value store for all cluster data
- **kube-scheduler**: Component that assigns workloads to nodes
- **kube-controller-manager**: Runs controller processes
- **cloud-controller-manager**: Integrates with underlying cloud provider APIs

For a highly available control plane:

```yaml
# Example etcd deployment for HA (simplified)
apiVersion: v1
kind: Pod
metadata:
  name: etcd-member-0
  namespace: kube-system
spec:
  containers:
  - name: etcd
    image: k8s.gcr.io/etcd:3.5.0
    command:
    - etcd
    - --advertise-client-urls=https://10.0.1.10:2379
    - --initial-advertise-peer-urls=https://10.0.1.10:2380
    - --initial-cluster=etcd-0=https://10.0.1.10:2380,etcd-1=https://10.0.1.11:2380,etcd-2=https://10.0.1.12:2380
    - --initial-cluster-state=new
    # Additional parameters...
```

### Node High Availability

To ensure node-level high availability:

1. **Multi-zone deployment**: Distribute nodes across multiple availability zones
2. **Node auto-repair**: Automatically repair failed nodes
3. **Node auto-scaling**: Dynamically adjust the number of nodes based on workload

### Workload High Availability

For workload high availability:

1. **Pod replicas**: Run multiple replicas of each pod
2. **Anti-affinity rules**: Distribute pods across nodes and zones
3. **PodDisruptionBudgets**: Limit voluntary disruptions
4. **Readiness/liveness probes**: Detect and replace unhealthy pods

## HA Patterns for ELK Stack Components

### Elasticsearch High Availability

Elasticsearch is designed for distributed operation but requires proper configuration:

#### Multi-Node Cluster Architecture

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-master
  namespace: elastic
spec:
  serviceName: elasticsearch-master
  replicas: 3  # Minimum of 3 for quorum
  selector:
    matchLabels:
      app: elasticsearch
      role: master
  template:
    metadata:
      labels:
        app: elasticsearch
        role: master
    spec:
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
                - master
            topologyKey: kubernetes.io/hostname
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        env:
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: cluster.name
          value: elasticsearch
        - name: discovery.seed_hosts
          value: "elasticsearch-master-0.elasticsearch-master,elasticsearch-master-1.elasticsearch-master,elasticsearch-master-2.elasticsearch-master"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2"
        - name: node.roles
          value: "master"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        resources:
          limits:
            cpu: 1
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 1Gi
```

#### Zone-Aware Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: elastic
spec:
  # ...other fields...
  template:
    spec:
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
            topologyKey: topology.kubernetes.io/zone
```

#### Shard Allocation Awareness

```yaml
# Configure in elasticsearch.yml or via API
cluster.routing.allocation.awareness.attributes: zone
node.attr.zone: us-east-1a  # Set per node based on actual zone
```

### Kibana High Availability

Kibana instances should be deployed with redundancy and load balancing:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elastic
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - kibana
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.8.0
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "https://elasticsearch-master:9200"
        # Additional configuration...
```

### Logstash High Availability

Logstash should be deployed with redundancy and proper input/output configurations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elastic
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - logstash
              topologyKey: kubernetes.io/hostname
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.8.0
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/logstash/config
        - name: pipeline-volume
          mountPath: /usr/share/logstash/pipeline
      volumes:
      - name: config-volume
        configMap:
          name: logstash-config
      - name: pipeline-volume
        configMap:
          name: logstash-pipeline
```

## Advanced HA Patterns

### Active-Active Configuration

For services like Kibana and Logstash, implement active-active setups:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elastic
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
  sessionAffinity: ClientIP  # Sticky sessions if needed
  type: LoadBalancer
```

### Active-Standby Configuration

For components that don't support active-active, implement active-standby:

```yaml
# Example leader election configuration (simplified)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-processor
  namespace: elastic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-processor
  template:
    metadata:
      labels:
        app: custom-processor
    spec:
      containers:
      - name: processor
        image: custom-processor:1.0
        command:
        - /bin/sh
        - -c
        - |
          if [ "$(curl -s http://localhost:4040/election/status)" == "leader" ]; then
            /app/process
          else
            /app/standby
          fi
```

### Cross-Zone Redundancy

Implement cross-zone redundancy for all ELK components:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: elastic
spec:
  # ...other fields...
  template:
    spec:
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
            topologyKey: topology.kubernetes.io/zone
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-east-1a
                - us-east-1b
                - us-east-1c
```

## Enhancing Availability with PodDisruptionBudgets

PodDisruptionBudgets (PDBs) ensure that a minimum number of pods remain available during voluntary disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: elasticsearch-master-pdb
  namespace: elastic
spec:
  minAvailable: 2  # Always keep at least 2 master nodes running
  selector:
    matchLabels:
      app: elasticsearch
      role: master
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: elasticsearch-data-pdb
  namespace: elastic
spec:
  maxUnavailable: 1  # Only allow one data node to be unavailable at a time
  selector:
    matchLabels:
      app: elasticsearch
      role: data
```

## StatefulSet Guarantees for Data Nodes

StatefulSets provide important guarantees for stateful applications like Elasticsearch:

- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: elastic
spec:
  serviceName: elasticsearch-data
  replicas: 6
  podManagementPolicy: Parallel  # Allow parallel pod management for faster operations
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods
  # ...rest of the StatefulSet definition...
```

## Readiness and Liveness Probes

Implement proper health checks for all ELK components:

### Elasticsearch Readiness Probe

```yaml
readinessProbe:
  httpGet:
    scheme: HTTPS
    path: /_cluster/health?local=true
    port: 9200
  initialDelaySeconds: 90
  periodSeconds: 10
  failureThreshold: 3
```

### Kibana Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /api/status
    port: 5601
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
```

### Logstash Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 9600
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
```

## Network Redundancy

### Service Mesh Integration

Integrate with service mesh solutions like Istio for advanced traffic management:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: elasticsearch-destinationrule
  namespace: elastic
spec:
  host: elasticsearch-master
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Multiple Ingress Routes

Create redundant ingress paths for services like Kibana:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: elastic
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
spec:
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
```

## Persistent Storage HA Patterns

### Storage Replication

Ensure data redundancy at the storage level:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-ha-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
  # For GCP or other providers that support it
  replication-type: regional-pd  # Regional persistent disk
```

### Snapshot and Backup Strategies

Implement automated snapshot strategies for disaster recovery:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: elasticsearch-snapshot
  namespace: elastic
spec:
  schedule: "0 2 * * *"  # Every day at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot-creator
            image: curlimages/curl:7.80.0
            command:
            - /bin/sh
            - -c
            - |
              # Create repository if it doesn't exist
              curl -X PUT "elasticsearch-master:9200/_snapshot/kubernetes_backup" -H 'Content-Type: application/json' -d'
              {
                "type": "s3",
                "settings": {
                  "bucket": "elasticsearch-backups",
                  "region": "us-east-1"
                }
              }
              '
              
              # Create snapshot
              curl -X PUT "elasticsearch-master:9200/_snapshot/kubernetes_backup/snapshot_$(date +%Y%m%d)" -H 'Content-Type: application/json' -d'
              {
                "indices": "*",
                "ignore_unavailable": true,
                "include_global_state": true
              }
              '
          restartPolicy: OnFailure
```

## Multi-Cluster Resilience

### Cross-Cluster Replication for Elasticsearch

Set up cross-cluster replication for geographically distributed resilience:

```yaml
# Establish cross-cluster connection
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.datacenter2.seeds": [
      "elasticsearch-master-0.elasticsearch-master.elastic-dc2.svc.cluster.local:9300",
      "elasticsearch-master-1.elasticsearch-master.elastic-dc2.svc.cluster.local:9300"
    ]
  }
}

# Set up cross-cluster replication
PUT /follower_index/_ccr/follow
{
  "remote_cluster": "datacenter2",
  "leader_index": "leader_index"
}
```

### Global Load Balancing

Implement global load balancing for multi-region access:

```yaml
# Using AWS Global Accelerator (sample CLI commands)
aws globalaccelerator create-accelerator --name elk-accelerator --region us-west-2

aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::111122223333:accelerator/1234abcd-abcd-1234-abcd-1234abcdefgh \
  --port-ranges FromPort=443,ToPort=443 \
  --protocol TCP

aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::111122223333:accelerator/1234abcd-abcd-1234-abcd-1234abcdefgh/listener/0123vxyz \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations EndpointId=eipalloc-abcdefg1234567890,Weight=128
```

## Autoscaling for High Availability

### Horizontal Pod Autoscaler (HPA)

Automatically scale pods based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  minReplicas: 3
  maxReplicas: 10
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
        averageUtilization: 75
```

### Vertical Pod Autoscaler (VPA)

Automatically adjust resource requests:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: elasticsearch-data-vpa
  namespace: elastic
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: elasticsearch
      minAllowed:
        memory: "1Gi"
        cpu: "500m"
      maxAllowed:
        memory: "4Gi"
        cpu: "2"
```

### Cluster Autoscaler

Scale the Kubernetes cluster itself based on workload:

```yaml
# GKE Example (applied via gcloud command)
gcloud container clusters update elk-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=10 \
  --zone=us-central1-a
```

## Chaos Engineering for HA Testing

### Pod Chaos Tests

Test resilience by intentionally disrupting pods:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: elasticsearch-data-failure
  namespace: elastic
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
    - elastic
    labelSelectors:
      "app": "elasticsearch"
      "role": "data"
  duration: "30s"
  scheduler:
    cron: "@every 10m"
```

### Network Chaos Tests

Test resilience to network issues:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: elasticsearch-network-delay
  namespace: elastic
spec:
  action: delay
  mode: one
  selector:
    namespaces:
    - elastic
    labelSelectors:
      "app": "elasticsearch"
  delay:
    latency: "90ms"
    correlation: "25"
    jitter: "30ms"
  duration: "60s"
  scheduler:
    cron: "@every 30m"
```

## HA Monitoring and Alerts

### Prometheus Monitoring

Set up comprehensive monitoring for HA components:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: elasticsearch-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: elasticsearch
  endpoints:
  - port: metrics
    interval: 15s
    scrapeTimeout: 10s
```

### Alerting Rules

Create alerts for HA-related issues:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: elasticsearch-ha-alerts
  namespace: monitoring
spec:
  groups:
  - name: elasticsearch.rules
    rules:
    - alert: ElasticsearchClusterRed
      expr: elasticsearch_cluster_health_status{color="red"} == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Elasticsearch cluster is RED"
        description: "Elasticsearch cluster has been RED for more than 5 minutes."
    - alert: ElasticsearchInstanceDown
      expr: up{job="elasticsearch"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Elasticsearch instance is down"
        description: "Elasticsearch instance has been down for more than 5 minutes."
    - alert: KibanaInstanceDown
      expr: up{job="kibana"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Kibana instance is down"
        description: "Kibana instance has been down for more than 5 minutes."
```

## Best Practices for ELK Stack HA

1. **Always deploy multi-node Elasticsearch clusters**:
   - Minimum 3 dedicated master nodes for quorum
   - Distribute data nodes across zones/regions
   - Configure appropriate shard allocation

2. **Implement proper node roles**:
   - Separate master, data, and coordinating roles
   - Consider dedicated ingest and machine learning nodes

3. **Use StatefulSets with proper ordinal indices**:
   - Ensure stable network identities
   - Maintain persistent storage association

4. **Configure appropriate resource requests and limits**:
   - Prevent resource starvation
   - Account for JVM heap and off-heap memory

5. **Implement thorough health checks**:
   - Readiness probes that check actual service health
   - Liveness probes with appropriate failure thresholds

6. **Use Pod anti-affinity rules**:
   - Ensure pods are distributed across nodes
   - Enforce zone-level anti-affinity for critical components

7. **Configure PodDisruptionBudgets**:
   - Prevent accidental cluster failures during maintenance
   - Set appropriate availability constraints

8. **Implement proper backup strategies**:
   - Elasticsearch snapshots
   - Volume snapshots
   - Regular backup testing

9. **Monitor and alert on HA-related metrics**:
   - Cluster health status
   - Node availability
   - Replication lag
   - Shard distribution

10. **Regular chaos testing**:
    - Simulate node failures
    - Test network partitions
    - Validate recovery procedures

## Conclusion

High availability in Kubernetes for ELK Stack deployments requires careful planning and implementation across multiple layers. By applying these patterns and best practices, you can build resilient Elasticsearch, Logstash, and Kibana deployments that maintain availability even during failures, maintenance, and unexpected events. Remember that true high availability is achieved through a combination of redundancy, automated recovery, proper monitoring, and regular testing.

## Additional Resources

- [Kubernetes High Availability Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Elasticsearch Resilience in Production](https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability.html)
- [Kibana High Availability Guide](https://www.elastic.co/guide/en/kibana/current/production.html)
- [Logstash Resiliency](https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html)
- [Chaos Mesh for Kubernetes](https://chaos-mesh.org/docs/)
- [Elastic Cloud on Kubernetes (ECK) Best Practices](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-operator-best-practices.html)