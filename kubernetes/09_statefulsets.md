# StatefulSets

StatefulSets are a Kubernetes workload API object used to manage stateful applications. Unlike Deployments, StatefulSets provide guarantees about the ordering and uniqueness of Pods, which is important for applications that require stable network identities, persistent storage, and ordered deployment and scaling.

## When to Use StatefulSets

Use StatefulSets when you need one or more of the following:

- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

Common workloads requiring StatefulSets:
- Databases (MySQL, PostgreSQL, MongoDB)
- Distributed systems (Elasticsearch, Kafka, ZooKeeper)
- Queuing and streaming applications
- Any application with leader-follower architecture

## StatefulSet Features

### Stable Network Identity

StatefulSets create Pods with predictable naming patterns and stable hostnames based on their ordinal index:
- `<statefulset-name>-0`
- `<statefulset-name>-1`
- `<statefulset-name>-2`
- ...

Each Pod gets a stable DNS name: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

### Persistent Storage

Each Pod can have its own persistent volume that follows it even when rescheduled to another node.

### Ordered Deployment and Scaling

Pods are created, updated, and deleted in strict sequential order, starting from lowest ordinal index (0).

### Headless Service

StatefulSets require a Headless Service for network identity:
- A Service with `clusterIP: None`
- Allows direct DNS lookups for individual Pods

## Basic StatefulSet Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  # Headless service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"  # Must match the headless service name
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:  # PVC template for each Pod
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
```

## StatefulSet Components

### Pod Identity

Each Pod in a StatefulSet derives its identity from:
- Ordinal index (a zero-based integer)
- Stable network identity based on StatefulSet name and ordinal
- Stable storage identity based on the ordinal

### Headless Service

The headless service allows direct network access to individual Pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
spec:
  clusterIP: None  # Headless service definition
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

### Volume Claim Templates

The `volumeClaimTemplates` create a PersistentVolumeClaim (PVC) for each Pod:

```yaml
volumeClaimTemplates:
- metadata:
    name: mongodb-data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "standard"
    resources:
      requests:
        storage: 10Gi
```

When a Pod is (re)scheduled, the same PVC is mounted, ensuring data persistence.

## Managing StatefulSets

### Creating a StatefulSet

```bash
kubectl apply -f mongodb-statefulset.yaml
```

Pods are created in order (0, 1, 2, ...), and each must be Running and Ready before the next is created.

### Scaling a StatefulSet

```bash
kubectl scale statefulset mongodb --replicas=5
```

Scaling up creates new Pods in order (N, N+1, N+2, ...).
Scaling down removes Pods in reverse order (N-1, N-2, ...).

### Updating a StatefulSet

StatefulSets support update strategies similar to Deployments:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate  # or OnDelete
    rollingUpdate:
      partition: 2  # Optional: Only update Pods with ordinal >= 2
```

Update strategies:
- `RollingUpdate`: Kubernetes updates Pods automatically in reverse order (N-1, N-2, ..., 0)
- `OnDelete`: Kubernetes updates Pods only when you delete them manually

### Deleting a StatefulSet

```bash
# Delete the StatefulSet but keep Pods
kubectl delete statefulset mongodb --cascade=false

# Delete the StatefulSet and its Pods
kubectl delete statefulset mongodb
```

Pods are terminated in reverse order, and deletion is blocked until a Pod is completely stopped.

## Advanced StatefulSet Patterns

### Leader Election

Many distributed systems need a leader. StatefulSets make this easier:

```yaml
# The "0" Pod is often designated as the initial leader
env:
- name: REPLICA_ROLE
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: IS_LEADER
  value: $(REPLICA_ROLE) == 'mongodb-0'
```

### Init Containers for Bootstrapping

Use init containers to check for existing data or perform bootstrapping:

```yaml
initContainers:
- name: init-mongodb
  image: mongo:4.4
  command:
  - bash
  - "-c"
  - |
    set -ex
    # Determine if this is a new Pod with no data
    [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
    ordinal=${BASH_REMATCH[1]}
    # Only initialize from previous Pod if not the first Pod
    if [[ $ordinal -gt 0 ]]; then
      until mongo --host=mongodb-$(($ordinal-1)).$SERVICE_NAME --eval "print('ok')"; do
        sleep 1
      done
      # Clone data from previous Pod if needed
    fi
  volumeMounts:
  - name: mongodb-data
    mountPath: /data/db
```

### Pod Management Policy

Control whether Pods are created and terminated in parallel:

```yaml
spec:
  podManagementPolicy: OrderedReady  # Default
  # podManagementPolicy: Parallel    # Create/delete Pods in parallel
```

- `OrderedReady`: Sequential Pod management (default)
- `Parallel`: Parallel Pod launch and termination (for independent StatefulSet Pods)

## Real-World Examples

### MongoDB Replica Set

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
  labels:
    app: mongodb
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-svc
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: mongodb
        image: mongo:4.4
        command:
        - mongod
        - --bind_ip_all
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: mongodb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        readinessProbe:
          exec:
            command:
            - mongo
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          timeoutSeconds: 5
      - name: mongo-sidecar
        image: cvallance/mongo-k8s-sidecar
        env:
        - name: MONGO_SIDECAR_POD_LABELS
          value: "app=mongodb"
        - name: KUBE_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: KUBERNETES_MONGO_SERVICE_NAME
          value: mongodb-svc
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### Kafka Cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - port: 9092
    name: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:6.2.0
        ports:
        - containerPort: 9092
          name: kafka
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['pod-index']
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper-0.zookeeper-headless.default.svc.cluster.local:2181,zookeeper-1.zookeeper-headless.default.svc.cluster.local:2181,zookeeper-2.zookeeper-headless.default.svc.cluster.local:2181"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(MY_POD_NAME).kafka-headless.$(MY_POD_NAMESPACE).svc.cluster.local:9092"
        volumeMounts:
        - name: kafka-data
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 20Gi
```

## Elasticsearch Cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  clusterIP: None
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    name: rest
  - port: 9300
    name: inter-node
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
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
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "es-cluster-0,es-cluster-1,es-cluster-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        ports:
        - containerPort: 9200
          name: rest
        - containerPort: 9300
          name: inter-node
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

## Best Practices

1. **Size StatefulSets appropriately**
   - Start with the minimum necessary replicas
   - Consider resource requirements for each Pod

2. **Use anti-affinity for high availability**
   ```yaml
   spec:
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
                   - mongodb
               topologyKey: "kubernetes.io/hostname"
   ```

3. **Set appropriate termination grace periods**
   ```yaml
   spec:
     template:
       spec:
         terminationGracePeriodSeconds: 60  # Default is 30
   ```

4. **Implement health checks**
   ```yaml
   readinessProbe:
     exec:
       command:
       - mongo
       - --eval
       - "db.adminCommand('ping')"
     initialDelaySeconds: 5
     timeoutSeconds: 5
   livenessProbe:
     exec:
       command:
       - mongo
       - --eval
       - "db.adminCommand('ping')"
     initialDelaySeconds: 30
     timeoutSeconds: 5
   ```

5. **Set resource requests and limits**
   ```yaml
   resources:
     requests:
       memory: "1Gi"
       cpu: "500m"
     limits:
       memory: "2Gi"
       cpu: "1"
   ```

6. **Plan for backup and recovery**
   - Set up regular backups of persistent volumes
   - Document restore procedures
   - Test the restore process

7. **Consider topology spread constraints**
   ```yaml
   topologySpreadConstraints:
   - maxSkew: 1
     topologyKey: topology.kubernetes.io/zone
     whenUnsatisfiable: DoNotSchedule
     labelSelector:
       matchLabels:
         app: mongodb
   ```

8. **Implement proper security**
   - Use secrets for sensitive configurations
   - Restrict network access with NetworkPolicies
   - Configure appropriate RBAC

## Troubleshooting StatefulSets

### Common Issues

1. **Pods stuck in Pending state**
   - Check for available PersistentVolumes
   - Verify storage class exists
   - Check node resources

2. **Pods in CrashLoopBackOff**
   - Check container logs: `kubectl logs <pod-name>`
   - Verify if volume mounts are correct
   - Check container startup commands

3. **Networking issues between Pods**
   - Verify headless service configuration
   - Test DNS resolution: `kubectl exec <pod-name> -- nslookup <service-name>`
   - Check NetworkPolicies

4. **Update/scaling problems**
   - Inspect StatefulSet events: `kubectl describe statefulset <name>`
   - Check pod-specific events: `kubectl describe pod <pod-name>`
   - Verify update strategy settings

### Useful Commands

```bash
# Check StatefulSet status
kubectl get statefulset mongodb -o wide

# Describe a StatefulSet
kubectl describe statefulset mongodb

# Check PVCs created by StatefulSet
kubectl get pvc -l app=mongodb

# Check individual Pod DNS
kubectl exec mongodb-0 -- nslookup mongodb-0.mongodb-svc

# Force delete a stuck Pod (use with caution)
kubectl delete pod mongodb-0 --force --grace-period=0
```

## Migrating from Deployments to StatefulSets

If you need to convert a Deployment to a StatefulSet:

1. Export your Deployment YAML:
   ```bash
   kubectl get deployment my-app -o yaml > my-app-deployment.yaml
   ```

2. Convert the YAML:
   - Change `kind: Deployment` to `kind: StatefulSet`
   - Add `serviceName` field
   - Add `volumeClaimTemplates` if needed
   - Remove any Deployment-specific fields (e.g., `progressDeadlineSeconds`)

3. Create a headless service

4. Apply the StatefulSet and migrate data

StatefulSets provide the foundation for running stateful applications in Kubernetes. By understanding their features and best practices, you can effectively deploy, scale, and maintain complex distributed systems with data persistence requirements.