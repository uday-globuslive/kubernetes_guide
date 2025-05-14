# Persistent Volumes in Kubernetes

## Introduction

Persistent Volumes (PVs) represent the cornerstone of stateful workloads in Kubernetes, providing a storage abstraction that outlives the lifecycle of individual pods. Unlike regular volumes, which are tightly coupled to pods, Persistent Volumes exist independently in the cluster, allowing data to persist even when pods are rescheduled, deleted, or moved between nodes.

This chapter explores the Persistent Volume architecture in Kubernetes, including PersistentVolumes (PVs), PersistentVolumeClaims (PVCs), Storage Classes, and dynamic provisioning. We'll examine how these components work together to provide durable storage for cloud-native applications.

## Persistent Volume Architecture

### The Storage Abstraction Model

Kubernetes implements a storage model that abstracts away the details of how storage is provisioned, attached, and managed through several key components:

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│                    Kubernetes Cluster                     │
│                                                           │
│  ┌───────────────┐   request   ┌───────────────────────┐  │
│  │      Pod      │─────────────▶ PersistentVolumeClaim │  │
│  └───────┬───────┘             └────────────┬──────────┘  │
│          │                                  │             │
│          │ mounts                           │ binds to    │
│          │ volume                           │             │
│          ▼                                  ▼             │
│  ┌───────────────┐             ┌────────────────────────┐ │
│  │Volume (in Pod)│◀────────────┤   PersistentVolume    │ │
│  └───────────────┘  provides   └────────────┬──────────┘ │
│                                             │            │
│                                  created by │            │
│                                             ▼            │
│                               ┌──────────────────────────┐│
│                               │      StorageClass        ││
│                               └────────────┬─────────────┘│
│                                            │              │
│                                uses        │              │
│                                            ▼              │
│                               ┌──────────────────────────┐│
│                               │   Volume Provisioner     ││
│                               └──────────────────────────┘│
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. PersistentVolume (PV)

A PersistentVolume is a cluster-wide resource that represents a piece of storage provisioned by an administrator or dynamically via a StorageClass. PVs have a lifecycle independent of any pod that uses them.

#### 2. PersistentVolumeClaim (PVC)

A PersistentVolumeClaim is a request for storage by a user. It's similar to a pod in that pods consume node resources and PVCs consume PV resources. PVCs can request specific size and access modes.

#### 3. StorageClass

StorageClasses provide a way to describe different "classes" of storage offerings in a cluster. Different classes might map to quality-of-service levels, backup policies, or arbitrary policies determined by cluster administrators.

#### 4. Volume Provisioner

Provisioners are responsible for creating the actual storage based on PVC requests. These can be internal (in-tree) provisioners or external provisioners implemented via Container Storage Interface (CSI) drivers.

## PersistentVolumes

### Creating PersistentVolumes

PersistentVolumes can be provisioned statically by a cluster administrator or dynamically using StorageClasses.

#### Static Provisioning Example

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

In the example above, we're creating a local PV backed by a hostPath. In production environments, you would typically use network storage like NFS, cloud volumes, or other storage solutions.

#### Other PV Examples

**NFS PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-server.example.com
    path: "/exports"
```

**AWS EBS PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-aws-ebs
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

**Azure Disk PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-azure-disk
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  azureDisk:
    cachingMode: ReadOnly
    diskName: azure-managed-disk
    diskURI: /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Compute/disks/<disk-name>
    kind: Managed
```

### PV Access Modes

PersistentVolumes support different access modes, which define how the volume can be mounted:

| Access Mode | Description |
|-------------|-------------|
| `ReadWriteOnce` (RWO) | The volume can be mounted as read-write by a single node |
| `ReadOnlyMany` (ROX) | The volume can be mounted read-only by many nodes |
| `ReadWriteMany` (RWX) | The volume can be mounted as read-write by many nodes |
| `ReadWriteOncePod` (RWOP) | The volume can be mounted as read-write by a single pod (Kubernetes 1.22+) |

Not all volume types support all access modes. For example:

| Volume Plugin | ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod |
|---------------|:-------------:|:------------:|:-------------:|:----------------:|
| AWS EBS       | ✓             | ✗            | ✗             | ✓                |
| Azure Disk    | ✓             | ✗            | ✗             | ✓                |
| Azure File    | ✓             | ✓            | ✓             | ✓                |
| GCE PD        | ✓             | ✗            | ✗             | ✓                |
| NFS           | ✓             | ✓            | ✓             | ✓                |
| iSCSI         | ✓             | ✓            | ✗             | ✗                |
| CephFS        | ✓             | ✓            | ✓             | ✓                |
| Ceph RBD      | ✓             | ✓            | ✗             | ✓                |

### PV Reclaim Policies

When a PersistentVolumeClaim is deleted, the PersistentVolume that was bound to it can be handled in different ways according to its reclaim policy:

| Reclaim Policy | Description |
|----------------|-------------|
| `Retain` | The PV is kept with its data intact. It's considered "released" but not available for another claim until manually reclaimed by an administrator. |
| `Delete` | The PV and its associated storage are automatically deleted when the PVC is deleted. |
| `Recycle` | *(Deprecated)* Basic scrub (`rm -rf /thevolume/*`) is performed before making the volume available again. |

Setting the reclaim policy:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # ... storage backend details
```

### PV Status

PersistentVolumes can be in one of the following phases:

| Phase | Description |
|-------|-------------|
| `Available` | Free resource not yet bound to a claim |
| `Bound` | Volume is bound to a claim |
| `Released` | Claim was deleted, but the resource is not yet reclaimed by the cluster |
| `Failed` | Volume has failed its automatic reclamation |

## PersistentVolumeClaims

### Creating PersistentVolumeClaims

PersistentVolumeClaims (PVCs) are requests for storage by users.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
```

When a PVC is created, Kubernetes looks for a PV that satisfies the claim based on:
- Storage capacity (must be at least as large as requested)
- Access modes (must match what is requested)
- StorageClass (if specified)
- Selectors (if specified)

### PVC Selectors

You can use selectors to bind to specific PVs based on labels:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  selector:
    matchLabels:
      environment: production
  storageClassName: ""  # Empty string must be explicitly set to use selector
```

### Using PVCs in Pods

Once a PVC is created and bound to a PV, it can be used by pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: "/var/www/html"
          name: app-data
  volumes:
    - name: app-data
      persistentVolumeClaim:
        claimName: my-app-data
```

### PVC Status

PVCs can be in various states:

| Status | Description |
|--------|-------------|
| `Pending` | The claim has been accepted but not yet bound to a PV, either because a matching PV doesn't exist or dynamic provisioning is in progress |
| `Bound` | The claim is bound to a PV |
| `Lost` | The underlying PV has been lost |

## Storage Classes and Dynamic Provisioning

### Understanding StorageClass

StorageClasses enable dynamic provisioning of PersistentVolumes based on the storage requirements specified in PVCs.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

Key fields in a StorageClass:

| Field | Description |
|-------|-------------|
| `provisioner` | The volume plugin to use for provisioning |
| `parameters` | Parameters specific to the provisioner |
| `reclaimPolicy` | What happens to volumes when PVCs are deleted (defaults to `Delete`) |
| `allowVolumeExpansion` | Whether the volumes can be expanded after creation |
| `volumeBindingMode` | When volume binding and dynamic provisioning should occur |

### Common Provisioners

| Provider | Provisioner | Notes |
|----------|-------------|-------|
| AWS | `kubernetes.io/aws-ebs` | EBS volumes |
| Azure | `kubernetes.io/azure-disk` | Azure disks |
| GCP | `kubernetes.io/gce-pd` | GCE persistent disks |
| Local | `kubernetes.io/no-provisioner` | For pre-created local volumes |
| OpenStack | `kubernetes.io/cinder` | OpenStack Cinder volumes |
| vSphere | `kubernetes.io/vsphere-volume` | vSphere volumes |

Many CSI drivers also act as provisioners for their storage systems.

### Dynamic Provisioning

Dynamic provisioning removes the need for cluster administrators to pre-provision storage. When a user creates a PVC that requests a StorageClass, a new PV is automatically created and bound to the PVC.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast
```

When this PVC is created, the `fast` StorageClass will automatically provision a 100Gi volume that matches the requested access modes.

### Default StorageClass

Clusters can designate a default StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

When a PVC is created without specifying a StorageClass, the default StorageClass is used.

### VolumeBindingMode

StorageClasses support two binding modes:

1. **Immediate (default)**: Volume binding and dynamic provisioning occurs as soon as the PVC is created.

2. **WaitForFirstConsumer**: Volume binding and provisioning are delayed until a pod using the PVC is created, which helps with topology constraints.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### Topology-Aware Provisioning

For cloud providers, you can specify zones where volumes should be created:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-pd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

## Advanced PV Features

### Volume Expansion

If the StorageClass allows volume expansion (`allowVolumeExpansion: true`), you can resize PVCs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi  # Original size
  storageClassName: expandable
```

To expand the volume, edit the PVC and increase the requested storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi  # Increased size
  storageClassName: expandable
```

**Important Notes on Volume Expansion:**
- Only certain volume types support expansion
- Some volume types may require the pod to be restarted
- You can only increase the size, not decrease it
- Underlying file system may need to be expanded separately

### PV/PVC Cloning

Some storage providers support cloning an existing PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clone-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: source-pvc
    kind: PersistentVolumeClaim
  storageClassName: csi-storage-class
```

### Volume Snapshots

Some storage providers support snapshots via the Volume Snapshot API:

```yaml
# Define a VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: csi.example.com
deletionPolicy: Delete

# Create a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: my-pvc

# Create a PVC from a snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  storageClassName: csi-storage-class
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### PV Data Protection

To prevent accidental data loss, you can use PodDisruptionBudgets and finalizers:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-stateful-app
```

## Practical PV/PVC Usage Patterns

### Deployments with PersistentVolumes

Using PVCs with Deployments:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-data
        persistentVolumeClaim:
          claimName: nginx-pvc
```

**Note:** When using PVCs with Deployments where `replicas > 1`, ensure the PVC's access mode is `ReadWriteMany` if multiple pods need to write to the volume simultaneously.

### StatefulSets with PersistentVolumes

StatefulSets provide stable identity and storage for stateful applications:

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
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

StatefulSets use `volumeClaimTemplates` to create a unique PVC for each pod replica. When a pod is rescheduled, it maintains the same PVC, ensuring data persistence.

### Shared Storage with ReadWriteMany

When multiple pods need to share the same data:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reader-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: reader
  template:
    metadata:
      labels:
        app: reader
    spec:
      containers:
      - name: reader
        image: busybox
        command: ["sh", "-c", "while true; do cat /data/file.txt; sleep 10; done"]
        volumeMounts:
        - name: shared-volume
          mountPath: /data
      volumes:
      - name: shared-volume
        persistentVolumeClaim:
          claimName: shared-data
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: writer-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: writer
  template:
    metadata:
      labels:
        app: writer
    spec:
      containers:
      - name: writer
        image: busybox
        command: ["sh", "-c", "while true; do date >> /data/file.txt; sleep 5; done"]
        volumeMounts:
        - name: shared-volume
          mountPath: /data
      volumes:
      - name: shared-volume
        persistentVolumeClaim:
          claimName: shared-data
```

### Database with Local Storage

For high-performance databases that need local storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: local-storage
---
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
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: db-data
```

## Best Practices

### Capacity Planning

1. **Right-size your volumes**:
   - Don't over-provision, but include room for growth
   - Consider the growth rate of your data

2. **Set appropriate resource quotas**:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-a
spec:
  hard:
    persistentvolumeclaims: 10
    requests.storage: 500Gi
```

3. **Monitor storage usage**:
   - Track PV utilization
   - Set up alerts for volumes nearing capacity

### Security

1. **Use appropriate access modes**:
   - Avoid ReadWriteMany unless truly needed
   - Consider using ReadWriteOncePod for strict isolation

2. **Set filesystem group and permissions**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-pod
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: secure-pvc
```

3. **Encryption for sensitive data**:
   - Use CSI drivers that support encryption
   - Consider using volume encryption at the storage level

### High Availability

1. **Use redundant storage backends**:
   - Cloud provider replicated volumes
   - Distributed storage systems (Ceph, GlusterFS)

2. **Data backup strategies**:
   - Regular volume snapshots
   - Backup to external storage
   - Test restore procedures

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: daily-backup
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup-job
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: k8s-snapshotter:v1.0
            command: ["/bin/sh", "-c", "create-snapshot --pvc=data-pvc --snapshot-class=daily-backup"]
          restartPolicy: OnFailure
```

### Performance

1. **Choose the right storage type**:
   - SSD for high IOPS workloads
   - Local storage for latency-sensitive applications
   - HDD for archive data

2. **Consider storage location**:
   - Use topology-aware StorageClasses
   - Place storage close to compute nodes

3. **Cache effectively**:
   - Use memory-backed emptyDir volumes for cache
   - Consider using PVs for read caches

### Troubleshooting Common Issues

#### 1. PVC stuck in "Pending" state

**Check StorageClass:**
```bash
kubectl get storageclass
```

**Check PV availability:**
```bash
kubectl get pv
```

**Check PVC events:**
```bash
kubectl describe pvc <pvc-name>
```

Common causes:
- No matching PV available
- StorageClass doesn't exist
- Dynamic provisioner is failing
- Insufficient resources

#### 2. Unable to delete PV

Check if the PV is protected by a finalizer:
```bash
kubectl get pv <pv-name> -o yaml
```

Look for the `kubernetes.io/pv-protection` finalizer. The PV might be still in use by a pod.

#### 3. Volume mount failures

Check pod events:
```bash
kubectl describe pod <pod-name>
```

Common causes:
- Wrong access mode
- Permission issues
- PV/PVC mismatch

#### 4. Volume performance issues

Monitor storage I/O:
```bash
kubectl exec -it <pod-name> -- dd if=/dev/zero of=/mount-path/test bs=1M count=100 oflag=dsync
```

## Future Trends

### CSI Migration

Kubernetes is migrating in-tree volume plugins to Container Storage Interface (CSI) drivers:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com  # CSI driver instead of kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
```

### Improved Data Services

Future Kubernetes releases will enhance data management capabilities:
- Better snapshot management
- Cross-namespace PVC access
- Improved volume metrics
- Enhanced volume health monitoring

### Kubernetes Data Gravity

As more stateful workloads move to Kubernetes, we'll see:
- Specialized storage operators
- Enhanced database-specific storage drivers
- Tighter integration with data pipelines

## Conclusion

Persistent Volumes provide the foundation for running stateful workloads in Kubernetes. By abstracting storage providers, offering flexible provisioning models, and providing features like snapshots and cloning, PVs enable data-intensive applications to run reliably in container environments.

Understanding how to effectively use PersistentVolumes, PersistentVolumeClaims, and StorageClasses is essential for any production Kubernetes deployment. By following best practices for capacity planning, security, high availability, and performance, you can ensure your application's data remains safe, accessible, and performant.

As Kubernetes continues to evolve, its storage capabilities are becoming increasingly sophisticated, making it suitable for ever more complex data workloads.

## References

- [Kubernetes Persistent Volumes Documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes Storage Classes Documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Container Storage Interface (CSI) Documentation](https://kubernetes-csi.github.io/docs/)
- [Kubernetes Volume Snapshots Documentation](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Kubernetes StatefulSets Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)