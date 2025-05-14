# Kubernetes Persistent Volumes: Complete Guide

## Introduction to Kubernetes Storage

Kubernetes provides robust storage management capabilities to ensure that applications have access to persistent storage regardless of the underlying infrastructure. The Kubernetes storage system is designed to address several key challenges:

1. **Pod Ephemerality**: Pods can be created and destroyed at any time, but data often needs to persist beyond a pod's lifecycle
2. **Infrastructure Abstraction**: Applications should work the same way regardless of the underlying storage provider
3. **Cross-Node Accessibility**: Storage may need to follow pods as they move between nodes
4. **Resource Management**: Storage needs to be allocated, tracked, and managed like other cluster resources

This guide explores Kubernetes' persistent storage options, focusing on the concepts, implementation patterns, and best practices for managing stateful workloads in production environments.

## Core Concepts

### Storage Abstractions

Kubernetes storage is built around several key abstractions:

#### Volumes

At the most basic level, Kubernetes volumes mount storage into pods, ensuring data is available to all containers in the pod. Unlike Docker volumes, Kubernetes volumes have a lifetime coupled to the pod's lifetime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
  - name: processor
    image: data-processor:1.0
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    emptyDir: {} # Simple ephemeral volume
```

#### Persistent Volumes (PV)

Persistent Volumes are cluster resources that provide storage which exists independently of pods. They are provisioned by cluster administrators or dynamically by the system.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-storage
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /storage/data
    server: nfs-server.example.com
```

#### Persistent Volume Claims (PVC)

PVCs are requests for storage by users. They act as an intermediary between pods and PVs, allowing users to request specific sizes and access modes without needing to know the details of the underlying storage implementation.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

#### Storage Classes

StorageClasses define different "classes" of storage offerings with various performance characteristics, allowing dynamic provisioning of PVs based on user requirements.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
```

### Access Modes

Kubernetes supports different access modes for volumes, which determine how the storage can be mounted by the nodes:

- **ReadWriteOnce (RWO)**: Volume can be mounted as read-write by a single node
- **ReadOnlyMany (ROX)**: Volume can be mounted read-only by many nodes
- **ReadWriteMany (RWX)**: Volume can be mounted as read-write by many nodes
- **ReadWriteOncePod (RWOP)**: Volume can be mounted as read-write by a single pod (added in Kubernetes 1.22)

Not all storage providers support all access modes. For example, AWS EBS volumes only support ReadWriteOnce, while NFS can support all three traditional access modes.

### Volume Modes

Kubernetes supports two volume modes:

- **Filesystem**: The default mode where the volume is mounted into pods as a directory
- **Block**: The volume is presented as a raw block device to pods

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block  # Using block mode instead of Filesystem
  resources:
    requests:
      storage: 10Gi
```

### Reclaim Policies

When a PVC is deleted, the reclaim policy of the PV determines what happens to the underlying storage:

- **Retain**: The PV remains available but in a "released" state, requiring manual reclamation
- **Delete**: The PV and associated storage are deleted automatically
- **Recycle**: The volume's contents are scrubbed before making it available again (deprecated)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: critical-data-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # Data remains after PVC deletion
  # ... other specifications
```

## Static vs Dynamic Provisioning

### Static Provisioning

In static provisioning, cluster administrators pre-create PVs that users can claim:

1. Admin creates the storage resource (e.g., an AWS EBS volume)
2. Admin creates a PV representing that storage
3. User creates a PVC to claim the PV
4. Kubernetes binds the PVC to a matching PV

```yaml
# Step 1: Admin creates a PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-storage
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: my-manually-created-disk
    fsType: ext4

# Step 2: User creates a PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi  # Must match or be smaller than the PV
```

### Dynamic Provisioning

Dynamic provisioning automates the creation of storage resources and PVs:

1. Admin creates StorageClasses that define storage options
2. User creates a PVC specifying a StorageClass
3. Storage is automatically provisioned and a PV is created

```yaml
# Step 1: Admin creates a StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true

# Step 2: User creates a PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd  # References the StorageClass
  resources:
    requests:
      storage: 30Gi
```

### Default Storage Class

Kubernetes allows setting a default StorageClass. When a PVC doesn't specify a StorageClass, the default is used:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

## Volume Types and CSI

### Built-in Volume Types

Kubernetes has several built-in volume types for different storage providers:

- **awsElasticBlockStore**: AWS EBS volumes
- **azureDisk**: Azure Disk volumes
- **gcePersistentDisk**: GCP Persistent Disks
- **nfs**: Network File System mounts
- **hostPath**: Files or directories from the host node
- **configMap**: ConfigMap content as volumes
- **secret**: Secret content as volumes
- **emptyDir**: Empty directory for temporary storage

### Container Storage Interface (CSI)

CSI is an API standard that allows storage providers to develop plugins once and have them work across container orchestrators. CSI drivers expand Kubernetes' storage capabilities beyond the built-in providers.

#### CSI Driver Structure

A CSI driver consists of:

1. **Controller Plugin**: Handles volume creation/deletion, attaching/detaching
2. **Node Plugin**: Manages mounting/unmounting volumes on nodes

#### Deploying CSI Drivers

Most CSI drivers are deployed as pods in the cluster:

```yaml
# Example of deploying AWS EBS CSI Driver
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ebs-csi-controller-sa
  namespace: kube-system
---
# ... RBAC Configuration ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-csi-controller
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ebs-csi-controller
  template:
    metadata:
      labels:
        app: ebs-csi-controller
    spec:
      serviceAccountName: ebs-csi-controller-sa
      containers:
      - name: ebs-plugin
        image: amazon/aws-ebs-csi-driver:v1.5.0
        args:
          - --endpoint=$(CSI_ENDPOINT)
          - --logtostderr
          - --v=5
        env:
          - name: CSI_ENDPOINT
            value: unix:///var/lib/csi/sockets/pluginproxy/csi.sock
        # ... additional configuration ...
```

#### Using CSI Volumes

Once a CSI driver is deployed, it's used through StorageClasses:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
```

### Common CSI Drivers

- **AWS EBS CSI Driver**: For AWS Elastic Block Store
- **Azure Disk CSI Driver**: For Azure Disk
- **GCE PD CSI Driver**: For Google Compute Engine Persistent Disk
- **Ceph RBD/CephFS CSI Driver**: For Ceph storage
- **Portworx CSI Driver**: For cloud-native storage
- **NetApp Trident**: For NetApp storage systems
- **OpenEBS CSI Driver**: For container attached storage

## Advanced Volume Features

### Volume Snapshots

CSI volume snapshots allow point-in-time copies of volumes:

```yaml
# VolumeSnapshotClass definition
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete

# Creating a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: database-data

# Creating a PVC from a snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-restore
spec:
  dataSource:
    name: database-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Volume Cloning

Volume cloning creates a duplicate of an existing PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  dataSource:
    name: source-pvc
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Volume Expansion

Many StorageClasses support volume expansion, allowing PVCs to be resized:

```yaml
# Enable volume expansion in StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-sc
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
allowVolumeExpansion: true

# To expand a volume, edit the PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  storageClassName: expandable-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi  # Original size was 10Gi
```

### Volume Topology

Volume topology ensures volumes are provisioned in the same zone/region as the pods that will use them:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topology-aware-sc
provisioner: kubernetes.io/gce-pd
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: pd-standard
```

### Raw Block Volumes

For applications that need direct access to block devices:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: block-user
spec:
  containers:
    - name: block-user
      image: debian
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
```

## Stateful Applications in Kubernetes

### StatefulSets

StatefulSets are designed for applications that require stable network identities and persistent storage:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 3
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
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

### Key StatefulSet Features

1. **Stable Network Identity**: Each pod gets a predictable hostname (e.g., postgres-0, postgres-1)
2. **Ordered Deployment**: Pods are created and updated in order
3. **Stable Storage**: Each replica gets its own PVC through volumeClaimTemplates
4. **Deletion Protection**: PVCs are not deleted when pods are deleted

### Databases and Messaging Systems

Database clusters typically use StatefulSets with dedicated storage per instance:

```yaml
# Example: MongoDB ReplicaSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: "mongodb"
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:4.4
        command:
        - mongod
        - "--replSet"
        - rs0
        - "--bind_ip"
        - "0.0.0.0"
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

### Headless Services

Headless services are often used with StatefulSets to provide DNS entries for each pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None  # Headless service
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

With this service, each pod is accessible via `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.

## Multi-Container Pod Storage Patterns

### Shared Volume

Multiple containers within a pod can share volumes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo $(date) >> /data/output.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["/bin/sh", "-c", "while true; do cat /data/output.txt; sleep 10; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

### Sidecar Pattern

Sidecars can process or enhance the data of the main container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-processor
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  - name: log-collector
    image: fluentd
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true  # Only needs read access
  volumes:
  - name: logs
    emptyDir: {}
```

### Init Container with Data

Init containers can prepare data before the main container starts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-prep
spec:
  initContainers:
  - name: data-downloader
    image: busybox
    command: ['wget', '-O', '/data/application-data.tar.gz', 'http://storage/application-data.tar.gz']
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

## Backup and Disaster Recovery

### Volume Snapshots for Backup

Use CSI snapshots for application-consistent backups:

```yaml
# Create a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-data-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: mysql-data

# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
spec:
  dataSource:
    name: mysql-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Backup Tools

Several Kubernetes-native backup tools are available:

- **Velero**: Backs up Kubernetes resources and volumes
- **Kasten K10**: Application-centric data management platform
- **Stash**: Backup and recovery solution for Kubernetes
- **Triliovault**: Data protection for Kubernetes

Example Velero backup:

```bash
# Install Velero
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.2.0 --bucket velero-backups --secret-file ./credentials

# Create a backup
velero backup create daily-backup --include-namespaces app-namespace

# Restore from backup
velero restore create --from-backup daily-backup
```

### Cross-Cluster Data Migration

Migrate data between clusters using snapshots or backup tools:

```bash
# Backup from source cluster
velero backup create migration-backup --include-namespaces app-namespace

# Restore to target cluster
# First, switch kubectl context to the target cluster
velero restore create --from-backup migration-backup
```

## Security Considerations

### Encryption at Rest

Configure encryption at the storage provider level:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-ebs
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
```

### Access Control

Use Kubernetes RBAC to control access to storage resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: app-namespace
  name: pvc-manager
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pvc-manager-binding
  namespace: app-namespace
subjects:
- kind: User
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pvc-manager
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Context

Control how pods access volumes through security contexts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    fsGroup: 2000     # All processes get this supplementary group
    runAsUser: 1000   # Run as non-root user
    runAsGroup: 3000  # Run as non-root group
  containers:
  - name: secure-container
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: secure-pvc
```

## Storage Performance Optimization

### Storage Classes for Different Performance Tiers

Create multiple StorageClasses for different performance requirements:

```yaml
# High-performance storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none

# Cost-effective storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

### Volume Mount Options

Optimize mount options for specific workloads:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-optimized
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  mountOptions:
    - hard
    - noatime
    - nodiratime
    - rsize=1048576
    - wsize=1048576
  nfs:
    server: nfs-server.example.com
    path: /exports
```

### Local Volumes for High Performance

For applications requiring extremely low latency:

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
```

## Monitoring and Troubleshooting

### Monitoring PV and PVC Status

Check the status of volumes and claims:

```bash
# List all PVs and their status
kubectl get pv

# List all PVCs and their status
kubectl get pvc --all-namespaces

# Get detailed information about a PV
kubectl describe pv <pv-name>

# Get detailed information about a PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

### Common Issues and Solutions

1. **PVC Stuck in Pending State**
   - Check that the StorageClass exists
   - Verify the storage provisioner is running
   - Look for capacity, access mode, or topology constraints

2. **Volume Mount Failures**
   - Check node kubelet logs
   - Verify the PV is correctly bound to the PVC
   - Check for filesystem or permission issues

3. **No Default StorageClass**
   - Create a StorageClass and mark it as default

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

### Debugging StorageClass Issues

```bash
# Check storage provisioner pods
kubectl get pods -n kube-system | grep csi

# Review provisioner logs
kubectl logs -n kube-system <provisioner-pod-name>

# Check storage class parameters
kubectl describe storageclass <storageclass-name>
```

## Best Practices

### Resource Management

1. **Establish Storage Quotas**:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-a
spec:
  hard:
    persistentvolumeclaims: "10"
    requests.storage: "500Gi"
    ssd-storage.storageclass.storage.k8s.io/requests.storage: "200Gi"
```

2. **Use Storage Request Limits**: Request only what you need

3. **Monitor Storage Usage**: Implement monitoring for storage utilization

### Production Readiness

1. **Implement High Availability**: Ensure storage systems are redundant

2. **Use Proper Reclaim Policies**: Use `Retain` for critical data

3. **Plan for Backups**: Regular backups of important volumes

4. **Document Storage Requirements**: For each application

### Application Storage Design

1. **Separate Data Types**: Use different volumes for different data types

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: static-content
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: static-content
    persistentVolumeClaim:
      claimName: static-content-pvc
  - name: logs
    persistentVolumeClaim:
      claimName: log-storage-pvc
```

2. **Right-Size Applications**: Understand storage requirements before deployment

3. **Implement Graceful Handling of Storage Issues**: Applications should handle storage errors

## Cloud-Specific Storage Options

### AWS

1. **EBS Volumes**: Block storage for single-node access

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "10"
```

2. **EFS**: File storage for multi-node access

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-a1b2c3d4
  directoryPerms: "700"
```

### Azure

1. **Azure Disk**: Block storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: managed
```

2. **Azure Files**: File storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-file
provisioner: file.csi.azure.com
parameters:
  skuName: Standard_LRS
```

### Google Cloud

1. **Google Persistent Disk**: Block storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gce-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
```

2. **Google Filestore**: File storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-enterprise
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: enterprise
  network: default
```

## Conclusion

Kubernetes provides a flexible, robust system for managing persistent storage that can meet the needs of diverse applications. By understanding the core concepts and best practices around PVs, PVCs, StorageClasses, and related features, you can design storage solutions that balance performance, durability, and cost-effectiveness.

As stateful workloads become more common in Kubernetes environments, mastering these storage concepts becomes increasingly important for building resilient, production-grade applications. Whether you're running databases, file servers, or other stateful services, Kubernetes provides the primitives needed to manage their storage requirements effectively.