# Upgrading and Maintenance in Kubernetes

## Introduction

Maintaining a Kubernetes cluster requires careful planning and execution of regular upgrades, patches, and maintenance activities. As Kubernetes evolves rapidly with new features, security patches, and bug fixes, organizations need a structured approach to keeping their clusters up-to-date while minimizing disruption to running workloads.

This guide covers best practices, strategies, and techniques for successfully managing Kubernetes cluster maintenance, including upgrading the control plane, worker nodes, and applications, with a focus on minimizing downtime and ensuring system reliability.

## Understanding Kubernetes Upgrade Basics

### Kubernetes Release Cycle

Kubernetes follows a structured release schedule:

- **Minor releases**: Approximately every 3-4 months (e.g., 1.26, 1.27, 1.28)
- **Patch releases**: Approximately every 2-4 weeks (e.g., 1.27.1, 1.27.2)
- **Support window**: Each minor version is supported for approximately 12-14 months
- **Version skew policy**: Control plane components must be within 1 minor version of each other

### Version Skew Policy

The Kubernetes version skew policy defines the supported version differences between components:

1. **kube-apiserver**: No other components should be newer than the API server
2. **kubelet**: Can be up to 2 minor versions older than the API server
3. **kube-controller-manager, kube-scheduler, and cloud-controller-manager**: Can be up to 1 minor version older than the API server
4. **kubectl**: Can be 1 minor version newer or older than the API server

### Determining When to Upgrade

Key considerations for timing upgrades:

1. **Security patches**: Prioritize upgrades that include critical security fixes
2. **End-of-support timeline**: Plan upgrades before versions reach end-of-support
3. **Feature requirements**: Evaluate if new features are needed for your workloads
4. **Stability concerns**: Allow new releases to stabilize before adopting in production
5. **Organizational change windows**: Align with existing maintenance schedules

## Preparing for Cluster Upgrades

### Pre-Upgrade Checklist

Before starting any upgrade, complete this essential checklist:

1. **Review the Kubernetes release notes** for potential breaking changes
2. **Verify cluster health** through monitoring and basic checks
3. **Back up critical components**, especially etcd
4. **Ensure sufficient resources** are available during the upgrade
5. **Check compatibility** of CNI plugins, CSI drivers, and other components
6. **Update your upgrade plan** based on current cluster architecture
7. **Notify stakeholders** about the planned upgrade window

### Cluster Health Verification

Run these commands to verify your cluster's health before upgrading:

```bash
# Check overall cluster status
kubectl get nodes
kubectl get pods --all-namespaces

# Check control plane health
kubectl get pods -n kube-system
kubectl get componentstatuses

# Verify that all nodes are ready
kubectl get nodes | grep -v "Ready"

# Check for any pods not in Running or Completed state
kubectl get pods --all-namespaces | grep -v "Running\|Completed"

# Verify that the API server is healthy
curl -k https://<API_SERVER_IP>:6443/healthz

# Check for any events that might indicate problems
kubectl get events --sort-by='.lastTimestamp' --all-namespaces
```

### Backing Up Cluster State

Create comprehensive backups before upgrading:

1. **etcd backup**: Most critical component to back up

```bash
# For Kubernetes installed with kubeadm
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-$(date +%Y-%m-%d-%H-%M-%S).db

# Verify the backup
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /tmp/etcd-backup-$(date +%Y-%m-%d-%H-%M-%S).db
```

2. **Kubernetes resource backup**: Use a tool like Velero or simple kubectl exports

```bash
# Export all resources using kubectl
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl -n $ns get -o yaml \
    configmaps,secrets,deployments,statefulsets,daemonsets,replicasets,services,ingress,pvc > k8s-backup-$ns-$(date +%Y-%m-%d-%H-%M-%S).yaml
done

# Using Velero for backups
velero backup create pre-upgrade-backup --include-namespaces '*'
```

### Resource Planning

Ensure sufficient resources are available for a safe upgrade:

1. **Node capacity**: Maintain at least 10-20% spare capacity for workload migrations
2. **Control plane resources**: Temporary spike during upgrades
3. **Network bandwidth**: Required for pulling new container images
4. **Storage performance**: Needed for data migrations and backups

### Document Upgrade Plan

Create a detailed upgrade plan document:

```markdown
# Kubernetes Upgrade Plan

## Upgrade Details
- Current version: 1.27.3
- Target version: 1.28.2
- Planned start time: 2023-10-15 01:00 UTC
- Estimated completion time: 2023-10-15 05:00 UTC
- Change ticket: CHG0012345

## Pre-Upgrade Tasks
1. [ ] Review release notes for 1.28.0, 1.28.1, and 1.28.2
2. [ ] Complete pre-upgrade checklist
3. [ ] Create etcd backup
4. [ ] Create Velero backup of all namespaces
5. [ ] Notify stakeholders of maintenance window
6. [ ] Scale up critical workloads for redundancy during node upgrades
7. [ ] Verify compatibility of CNI, CSI, and other components

## Control Plane Upgrade
1. [ ] Upgrade primary control plane node
2. [ ] Verify control plane health
3. [ ] Upgrade additional control plane nodes one by one
4. [ ] Verify all control plane components are upgraded

## Worker Node Upgrade
1. [ ] Cordon and drain first worker node group
2. [ ] Upgrade worker nodes in group
3. [ ] Uncordon nodes and verify workload scheduling
4. [ ] Repeat for remaining worker node groups

## Post-Upgrade Verification
1. [ ] Verify cluster health (nodes, pods, services)
2. [ ] Run test deployments
3. [ ] Verify monitoring and logging systems
4. [ ] Check CNI functionality
5. [ ] Validate persistent volume operations

## Rollback Plan
1. If control plane upgrade fails:
   - Restore etcd from backup
   - Reinstall previous Kubernetes version on control plane

2. If worker node upgrade fails:
   - Stop the upgrade process
   - Rebuild worker nodes with previous version
   - Reschedule workloads

## Contact Information
- Primary operator: Jane Smith (jane.smith@example.com, +1-555-123-4567)
- Secondary operator: John Doe (john.doe@example.com, +1-555-765-4321)
- Escalation contact: Sarah Johnson (sarah.johnson@example.com, +1-555-987-6543)
```

## Performing Kubernetes Control Plane Upgrades

### Managed Kubernetes Service Upgrades

For managed Kubernetes services like GKE, EKS, or AKS:

#### Google Kubernetes Engine (GKE)

```bash
# Check available upgrades
gcloud container clusters get-upgrades my-cluster --region us-central1

# Upgrade the control plane only
gcloud container clusters upgrade my-cluster --region us-central1 \
  --master --no-node-pool-upgrade

# Upgrade specific node pool
gcloud container clusters upgrade my-cluster --region us-central1 \
  --node-pool=default-pool
```

#### Amazon EKS

```bash
# Check current version
aws eks describe-cluster --name my-cluster --query "cluster.version"

# Update cluster version
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.28

# Check upgrade status
aws eks describe-update \
  --name my-cluster \
  --update-id <update-id-from-previous-command>
```

#### Azure Kubernetes Service (AKS)

```bash
# Check available upgrades
az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster

# Upgrade cluster
az aks upgrade \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --kubernetes-version 1.28.2

# Check upgrade progress
az aks show --resource-group myResourceGroup --name myAKSCluster --output table
```

### Self-Managed Cluster Upgrades with kubeadm

For clusters managed with kubeadm:

#### Control Plane Upgrade

```bash
# On the primary control plane node
# 1. Update kubeadm
apt-get update && apt-get install -y kubeadm=1.28.2-00

# 2. Check upgrade plan
kubeadm upgrade plan

# 3. Apply the upgrade
kubeadm upgrade apply v1.28.2

# 4. Upgrade kubelet and kubectl
apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00
systemctl daemon-reload
systemctl restart kubelet

# Verify the upgrade
kubectl get nodes
```

#### Secondary Control Plane Nodes

Repeat for each additional control plane node:

```bash
# 1. Update kubeadm
apt-get update && apt-get install -y kubeadm=1.28.2-00

# 2. Upgrade the node
kubeadm upgrade node

# 3. Upgrade kubelet and kubectl
apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00
systemctl daemon-reload
systemctl restart kubelet
```

### Upgrading etcd

If you need to upgrade etcd separately:

```bash
# 1. Check current etcd version
ETCDCTL_API=3 etcdctl version

# 2. Create a backup
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-$(date +%Y-%m-%d-%H-%M-%S).db

# 3. Update etcd
# For kubeadm-managed clusters, etcd is updated as part of the control plane upgrade
# For externally managed etcd, follow the official etcd upgrade documentation
```

## Worker Node Upgrades

### Upgrading Worker Nodes with Minimal Disruption

#### Node Drain and Upgrade Process

For each worker node:

```bash
# 1. Cordon the node to prevent new pods from being scheduled
kubectl cordon node-01

# 2. Drain the node (evicts pods to other nodes)
kubectl drain node-01 --ignore-daemonsets --delete-emptydir-data

# 3. Upgrade the node's kubelet
# On the node itself:
apt-get update && apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00
systemctl daemon-reload
systemctl restart kubelet

# 4. Verify the upgrade
kubectl get node node-01

# 5. Uncordon the node to allow pod scheduling
kubectl uncordon node-01
```

#### Using Node Upgrades in Managed Services

For managed services, use platform-specific commands:

```bash
# GKE
gcloud container clusters upgrade my-cluster \
  --node-pool=default-pool \
  --region us-central1

# EKS with managed node groups
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup

# AKS
az aks nodepool upgrade \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --kubernetes-version 1.28.2
```

### Node Upgrade Strategies

#### Rolling Upgrades

Upgrade nodes one by one to minimize disruption:

```bash
# Example script for rolling upgrades
#!/bin/bash
NODES=$(kubectl get nodes -l role=worker -o jsonpath='{.items[*].metadata.name}')
for node in $NODES; do
  echo "Upgrading $node..."
  
  # Cordon node
  kubectl cordon $node
  
  # Drain node
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data
  
  # Wait for pods to fully migrate
  sleep 60
  
  # Perform upgrade on node
  # Example: SSH to node and run upgrade commands
  ssh $node "apt-get update && apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00 && systemctl daemon-reload && systemctl restart kubelet"
  
  # Wait for node to become ready
  until kubectl get node $node | grep -w "Ready"; do
    echo "Waiting for $node to become ready..."
    sleep 10
  done
  
  # Uncordon node
  kubectl uncordon $node
  
  # Wait before proceeding to next node
  echo "Upgraded $node successfully. Waiting before next node upgrade..."
  sleep 300
done
```

#### Blue-Green Upgrades

Create a new node pool with the updated version, then migrate workloads:

```bash
# 1. Create new node pool (GKE example)
gcloud container node-pools create new-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --node-version=1.28.2 \
  --num-nodes=3

# 2. Cordon old node pool to prevent new scheduling
kubectl cordon -l cloud.google.com/gke-nodepool=old-pool

# 3. Migrate workloads by recreating deployments or using pod anti-affinity
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-nodepool
                operator: In
                values:
                - new-pool
      containers:
      - name: sample-app
        image: sample-app:latest
EOF

# 4. Verify workloads are on new nodes
kubectl get pods -o wide

# 5. Delete old node pool when migration is complete
gcloud container node-pools delete old-pool \
  --cluster=my-cluster \
  --region=us-central1
```

#### Surge Upgrades

Increase capacity temporarily during upgrades:

```yaml
# Example kOps configuration for surge upgrades
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  name: nodes
spec:
  machineType: t3.large
  maxSize: 15
  minSize: 10
  role: Node
  subnets:
  - us-east-1a
  - us-east-1b
  - us-east-1c
  cloudLabels:
    k8s.io/cluster-autoscaler/enabled: ""
    k8s.io/cluster-autoscaler/my-cluster: ""
  # Surge configuration
  rollingUpdate:
    maxSurge: 3
    maxUnavailable: 0
```

### Managing PodDisruptionBudgets

Use PodDisruptionBudgets (PDBs) to ensure service availability during upgrades:

```yaml
# Example PDB ensuring at least 2 replicas of an app remain available
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-service
```

Alternatively, use maxUnavailable:

```yaml
# Example PDB limiting unavailable replicas to 1
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: critical-service
```

Check existing PDBs before node maintenance:

```bash
# List all PDBs across the cluster
kubectl get poddisruptionbudgets --all-namespaces

# Check if draining a node would violate any PDBs
kubectl drain node-01 --ignore-daemonsets --dry-run=server
```

## Application Upgrades and Maintenance

### Deployment Upgrade Strategies

#### Rolling Updates

The default Kubernetes deployment strategy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: web-app:v2.0.0
        ports:
        - containerPort: 80
```

#### Blue-Green Deployments

Using service switching to target different deployments:

```yaml
# Blue deployment (current version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: blue
  template:
    metadata:
      labels:
        app: web-app
        version: blue
    spec:
      containers:
      - name: web-app
        image: web-app:v1.0.0
        ports:
        - containerPort: 80
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: green
  template:
    metadata:
      labels:
        app: web-app
        version: green
    spec:
      containers:
      - name: web-app
        image: web-app:v2.0.0
        ports:
        - containerPort: 80
---
# Service targeting blue deployment
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
    version: blue  # Change to green when ready to switch
  ports:
  - port: 80
    targetPort: 80
```

#### Canary Deployments

Gradually shifting traffic to the new version:

```yaml
# Main deployment (stable version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web-app
      version: stable
  template:
    metadata:
      labels:
        app: web-app
        version: stable
    spec:
      containers:
      - name: web-app
        image: web-app:v1.0.0
        ports:
        - containerPort: 80
---
# Canary deployment (new version with limited traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-canary
spec:
  replicas: 1  # Only 10% of the pods
  selector:
    matchLabels:
      app: web-app
      version: canary
  template:
    metadata:
      labels:
        app: web-app
        version: canary
    spec:
      containers:
      - name: web-app
        image: web-app:v2.0.0
        ports:
        - containerPort: 80
---
# Service targeting both deployments for weighted traffic
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app  # Selects pods from both deployments
  ports:
  - port: 80
    targetPort: 80
```

For more fine-grained traffic control, use a service mesh like Istio:

```yaml
# Istio VirtualService for precise traffic splitting
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-app
spec:
  hosts:
  - web-app
  http:
  - route:
    - destination:
        host: web-app-stable
        subset: stable
      weight: 90
    - destination:
        host: web-app-canary
        subset: canary
      weight: 10
```

### Stateful Application Maintenance

#### StatefulSet Updates

Update StatefulSets with controlled strategies:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  updateStrategy:
    type: RollingUpdate  # or OnDelete
    rollingUpdate:
      partition: 0  # Update all pods; set higher to update only pods >= that index
  serviceName: database
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: database
        image: postgres:15.2
        ports:
        - containerPort: 5432
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

Use the partition parameter for controlled rolling updates:

```bash
# Update only pods with index 2 and higher (in a 3-pod StatefulSet, only pod 2)
kubectl patch statefulset database -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Update the image
kubectl set image statefulset/database database=postgres:15.3

# Check the status
kubectl rollout status statefulset/database

# Update all remaining pods by setting partition to 0
kubectl patch statefulset database -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

#### Database Maintenance

For database maintenance, use operators when available:

```yaml
# Example of a PostgreSQL upgrade using the Zalando operator
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: postgres-cluster
spec:
  teamId: "data-team"
  volume:
    size: 10Gi
  numberOfInstances: 3
  postgresql:
    version: "15"  # Upgrade from previous version
    parameters:
      shared_buffers: "1GB"
      max_connections: "100"
  patroni:
    initdb:
      encoding: "UTF8"
      locale: "en_US.UTF-8"
    pg_hba:
    - host all all 0.0.0.0/0 md5
```

### Configuration Updates

#### ConfigMap and Secret Updates

Update ConfigMaps and Secrets safely:

```bash
# Update a ConfigMap
kubectl create configmap app-config --from-file=config.properties -o yaml --dry-run=client | kubectl apply -f -

# Update a Secret
kubectl create secret generic app-secret --from-literal=api-key=NEW_KEY -o yaml --dry-run=client | kubectl apply -f -
```

Handle configuration changes in applications:

1. **Automated reload** with applications that support it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    metadata:
      annotations:
        checksum/config: "${SHA256_HASH}"  # Add during CI/CD pipeline
    spec:
      containers:
      - name: web-app
        image: web-app:v1.0.0
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        lifecycle:
          postStart:
            exec:
              command: ["sh", "-c", "watch -n 5 'kill -HUP 1'"]  # Simple reload approach
      volumes:
      - name: config-volume
        configMap:
          name: app-config
```

2. **Immutable configurations** with new pods:

```bash
# Create a new ConfigMap with a versioned name
kubectl create configmap app-config-v2 --from-file=config.properties

# Update the deployment to use the new ConfigMap
kubectl set volume deployment/web-app --add --name=config-volume --mount-path=/app/config --configmap-name=app-config-v2
```

## Maintenance Windows and Scheduling

### Planning Maintenance Windows

Best practices for maintenance windows:

1. **Schedule during low-traffic periods**
2. **Communicate window details in advance**
3. **Set realistic time frames** including buffer for unexpected issues
4. **Stagger maintenance across zones** for multi-region deployments

Sample maintenance schedule:

```markdown
# Q4 2023 Kubernetes Maintenance Schedule

| Date | Time (UTC) | Activity | Components | Impact |
|------|------------|----------|------------|--------|
| Oct 15 | 01:00-05:00 | Kubernetes Upgrade | All clusters | Rolling node restarts |
| Oct 22 | 02:00-04:00 | Patch Updates | Worker nodes | Minimal application impact |
| Nov 12 | 01:00-03:00 | Network Maintenance | CNI components | Brief connection resets |
| Dec 10 | 01:00-06:00 | Major Version Upgrade | All clusters | Potential 5-minute API server outage |
```

### Automating Maintenance Activities

Use tools like Kured (Kubernetes Reboot Daemon) for automated node maintenance:

```yaml
# Kured deployment for automated node reboots
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kured
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: kured
  template:
    metadata:
      labels:
        name: kured
    spec:
      serviceAccountName: kured
      containers:
        - name: kured
          image: kubereboot/kured:1.13.0
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true 
          env:
            - name: KURED_NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          command:
            - /usr/bin/kured
            - --reboot-command=/bin/systemctl reboot
            - --period=1h
            - --reboot-sentinel=/var/run/reboot-required
            - --time-zone=UTC
            - --start-time=0:00
            - --end-time=6:00
            - --lockfile-annotation=kured.io/reboot-in-progress
```

## Handling Failed Upgrades

### Detecting Upgrade Failures

Common signs of upgrade failures:

1. **Control plane component failures**:
   - API server unavailable
   - Scheduler or controller-manager not functioning
   - etcd data inconsistency

2. **Worker node failures**:
   - Nodes not joining the cluster
   - Kubelet service failing
   - Container runtime issues

3. **Network failures**:
   - Pod-to-pod communication issues
   - Service discovery problems
   - DNS resolution failures

### Troubleshooting Common Upgrade Issues

#### API Server Issues

```bash
# Check API server logs
kubectl logs -n kube-system kube-apiserver-controlplane -p

# Check API server status on the node
systemctl status kube-apiserver

# Verify certificate validity
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 Validity
```

#### etcd Issues

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd member list
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

#### Kubelet Issues

```bash
# Check kubelet status
systemctl status kubelet

# Check kubelet logs
journalctl -u kubelet -f

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml
```

### Rollback Procedures

#### Rolling Back Control Plane

For kubeadm-managed clusters:

```bash
# 1. Downgrade kubeadm
apt-get install -y kubeadm=1.27.3-00

# 2. Roll back the upgrade
kubeadm upgrade plan
kubeadm upgrade apply v1.27.3

# 3. Downgrade kubelet and kubectl
apt-get install -y kubelet=1.27.3-00 kubectl=1.27.3-00
systemctl daemon-reload
systemctl restart kubelet
```

#### Restoring etcd from Backup

If etcd data is corrupted:

```bash
# Stop the API server and etcd
systemctl stop kube-apiserver
systemctl stop etcd

# Restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /path/to/etcd-backup.db \
  --data-dir /var/lib/etcd-restore \
  --name control-plane-01 \
  --initial-cluster control-plane-01=https://10.0.0.1:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://10.0.0.1:2380

# Update etcd configuration to use the restored data
mv /var/lib/etcd /var/lib/etcd-old
mv /var/lib/etcd-restore /var/lib/etcd

# Restart etcd and API server
systemctl start etcd
systemctl start kube-apiserver
```

#### Rolling Back Worker Nodes

Restore worker nodes to the previous version:

```bash
# 1. Downgrade kubelet
apt-get install -y kubelet=1.27.3-00

# 2. Restart the kubelet
systemctl daemon-reload
systemctl restart kubelet

# 3. Verify the node version
kubectl get node node-01 -o jsonpath='{.status.nodeInfo.kubeletVersion}'
```

## Operational Best Practices

### Documentation and Record Keeping

Maintain detailed records of all maintenance activities:

```markdown
# Kubernetes Upgrade Record

## Upgrade Details
- Date: 2023-10-15
- Performed by: Jane Smith
- Kubernetes version: 1.27.3 to 1.28.2
- Duration: 3 hours 15 minutes

## Pre-Upgrade Checks
- [x] Cluster health verified
- [x] etcd backup created: /backups/etcd-2023-10-15-010015.db
- [x] Velero backup created: pre-upgrade-2023-10-15

## Upgrade Steps
1. Control plane upgraded:
   - control-plane-01: Completed at 01:45 UTC
   - control-plane-02: Completed at 02:10 UTC
   - control-plane-03: Completed at 02:35 UTC

2. Worker nodes upgraded:
   - worker-group-1 (nodes 1-5): Completed at 03:25 UTC
   - worker-group-2 (nodes 6-10): Completed at 04:15 UTC

## Issues Encountered
- Worker node 7 failed to join after upgrade
  - Issue: Incompatible containerd version
  - Resolution: Upgraded containerd from 1.6.15 to 1.6.19

## Post-Upgrade Verification
- [x] All nodes reporting correct version
- [x] Sample workloads deployed successfully
- [x] CNI functionality verified
- [x] Storage operations tested

## Follow-up Actions
- Update monitoring system to be compatible with 1.28
- Schedule testing of new features:
  - Pod scheduling readiness gates
  - CEL in ValidatingAdmissionPolicy
```

### Monitoring During and After Upgrades

Set up specialized monitoring for upgrade periods:

1. **Prometheus alerts**:

```yaml
# Example Prometheus alert rules for upgrades
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: upgrade-monitoring
  namespace: monitoring
spec:
  groups:
  - name: upgrade.rules
    rules:
    - alert: KubernetesVersionMismatch
      expr: count(count by (kubernetes_version) (kubernetes_build_info{job="kubelet"} != "")) > 1
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Multiple Kubernetes versions detected"
        description: "Cluster is running different versions of Kubernetes components. This could indicate a stalled upgrade."
    
    - alert: NodeNotReadyDuringUpgrade
      expr: (kube_node_spec_taint{key="node.kubernetes.io/not-ready"} unless on(node) kube_node_spec_taint{key="node-role.kubernetes.io/master"}) > 0
      for: 15m
      labels:
        severity: critical
      annotations:
        summary: "Node not ready during upgrade"
        description: "The node {{ $labels.node }} has been in a not-ready state for more than 15 minutes during the upgrade window."
    
    - alert: HighPodDisruptionDuringUpgrade
      expr: round(sum by (node) (delta(kube_pod_status_ready{condition="false"}[10m])) / sum by (node) (kube_pod_status_ready)) * 100 > 20
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High pod disruption during upgrade"
        description: "More than 20% of pods became not-ready in a short period on node {{ $labels.node }}."
```

2. **Grafana dashboard** for upgrade monitoring:

```json
{
  "dashboard": {
    "id": null,
    "title": "Kubernetes Upgrade Monitoring",
    "tags": ["kubernetes", "upgrade"],
    "panels": [
      {
        "title": "Node Versions",
        "type": "gauge",
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        },
        "fieldConfig": {
          "defaults": {
            "mappings": [],
            "thresholds": {
              "mode": "percentage",
              "steps": [
                { "color": "red", "value": 0 },
                { "color": "yellow", "value": 50 },
                { "color": "green", "value": 100 }
              ]
            }
          }
        },
        "targets": [
          {
            "expr": "sum(kube_node_info{kubelet_version=~\"v1.28.*\"}) / count(kube_node_info) * 100",
            "refId": "A"
          }
        ]
      },
      {
        "title": "Node Status",
        "type": "table",
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        },
        "targets": [
          {
            "expr": "kube_node_info{job=\"kube-state-metrics\"} * on(node) group_left(status) (kube_node_status_condition{job=\"kube-state-metrics\",condition=\"Ready\"} != 0)",
            "refId": "A",
            "instant": true
          }
        ],
        "transformations": [
          {
            "id": "organize",
            "options": {
              "excludeByName": {
                "Time": true,
                "Value": true,
                "__name__": true,
                "container_runtime_version": true,
                "job": true,
                "instance": true
              },
              "renameByName": {
                "kernel_version": "Kernel",
                "kubelet_version": "Kubelet",
                "kubeproxy_version": "Kube-Proxy",
                "node": "Node",
                "os_image": "OS",
                "status": "Ready"
              }
            }
          }
        ]
      }
    ]
  }
}
```

### Upgrade Validation Tests

Create automated tests to verify successful upgrades:

```yaml
# Job to run post-upgrade validation tests
apiVersion: batch/v1
kind: Job
metadata:
  name: upgrade-validation
  namespace: kube-system
spec:
  template:
    spec:
      serviceAccountName: upgrade-validator
      containers:
      - name: validator
        image: upgrade-test:v1.0
        command: ["python", "/scripts/validate_upgrade.py"]
        env:
        - name: TARGET_VERSION
          value: "v1.28.2"
        - name: KUBE_CONFIG
          value: "/var/run/secrets/kubernetes.io/serviceaccount/token"
      restartPolicy: Never
  backoffLimit: 3
```

Example validation script:

```python
#!/usr/bin/env python3
# validate_upgrade.py

import os
import sys
import time
import requests
import json
from kubernetes import client, config

def main():
    # Load Kubernetes configuration
    config.load_incluster_config()
    
    # Initialize API clients
    v1 = client.CoreV1Api()
    version_client = client.VersionApi()
    
    # Check Kubernetes version
    target_version = os.environ.get("TARGET_VERSION", "v1.28.2")
    version = version_client.get_code()
    if not version.git_version.startswith(target_version):
        print(f"Error: Expected Kubernetes {target_version}, but found {version.git_version}")
        sys.exit(1)
    
    # Check node versions
    nodes = v1.list_node()
    outdated_nodes = []
    for node in nodes.items:
        node_version = node.status.node_info.kubelet_version
        if not node_version.startswith(target_version):
            outdated_nodes.append(f"{node.metadata.name}: {node_version}")
    
    if outdated_nodes:
        print(f"Error: Found nodes with outdated versions: {outdated_nodes}")
        sys.exit(1)
    
    # Test API functionality
    apis_to_test = [
        ("/api/v1/namespaces", "List namespaces"),
        ("/apis/apps/v1/deployments", "List deployments"),
        ("/apis/batch/v1/jobs", "List jobs")
    ]
    
    for api_path, description in apis_to_test:
        try:
            response = requests.get(
                f"https://kubernetes.default.svc{api_path}",
                headers={
                    "Authorization": f"Bearer {open('/var/run/secrets/kubernetes.io/serviceaccount/token').read()}"
                },
                verify="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt",
                timeout=10
            )
            response.raise_for_status()
            print(f"✅ API Test Passed: {description}")
        except Exception as e:
            print(f"❌ API Test Failed: {description} - {str(e)}")
            sys.exit(1)
    
    # Test pod creation
    test_pod_name = f"test-pod-{int(time.time())}"
    test_pod = client.V1Pod(
        metadata=client.V1ObjectMeta(name=test_pod_name),
        spec=client.V1PodSpec(
            containers=[
                client.V1Container(
                    name="test-container",
                    image="nginx:stable"
                )
            ],
            restart_policy="Never"
        )
    )
    
    try:
        v1.create_namespaced_pod("default", test_pod)
        print("✅ Pod Creation Test: Successfully created test pod")
        
        # Wait for pod to be ready
        for _ in range(30):  # 30 * 2 seconds = 60 seconds timeout
            pod = v1.read_namespaced_pod(test_pod_name, "default")
            if pod.status.phase == "Running":
                print("✅ Pod Running Test: Test pod is running")
                break
            time.sleep(2)
        else:
            print("❌ Pod Running Test: Timed out waiting for pod to run")
            sys.exit(1)
        
        # Clean up
        v1.delete_namespaced_pod(test_pod_name, "default")
        print("✅ Pod Deletion Test: Successfully deleted test pod")
        
    except Exception as e:
        print(f"❌ Pod Creation Test Failed: {str(e)}")
        sys.exit(1)
    
    print("✅ All upgrade validation tests passed!")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

## Future-Proofing Maintenance Strategies

### Adopting GitOps for Cluster Management

Use GitOps tools like Flux or ArgoCD to manage cluster configuration:

```yaml
# Example Flux configuration for managing Kubernetes version
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cluster-infrastructure
  namespace: flux-system
spec:
  interval: 1h
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: coredns
    namespace: kube-system
  - apiVersion: apps/v1
    kind: Deployment
    name: metrics-server
    namespace: kube-system
```

### Cluster API for Declarative Management

Leverage Cluster API for managing Kubernetes clusters:

```yaml
# Example Cluster API configuration
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: my-cluster
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
spec:
  replicas: 3
  version: v1.28.2
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: my-cluster-control-plane
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
      controllerManager:
        extraArgs:
          cloud-provider: aws
    initConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
        kubeletExtraArgs:
          cloud-provider: aws
    joinConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
        kubeletExtraArgs:
          cloud-provider: aws
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-cluster-workers
spec:
  clusterName: my-cluster
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
      pool: worker-pool
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: my-cluster
        pool: worker-pool
    spec:
      version: v1.28.2
      clusterName: my-cluster
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: my-cluster-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: my-cluster-workers
```

### Continuous Upgrade Testing with CI/CD

Implement continuous testing of upgrades in CI/CD pipelines:

```yaml
# Example GitHub Actions workflow for testing upgrades
name: Test Kubernetes Upgrade

on:
  schedule:
    - cron: '0 2 * * 0'  # Weekly on Sunday at 2 AM
  workflow_dispatch:
    inputs:
      from_version:
        description: 'Kubernetes version to upgrade from'
        required: true
        default: 'v1.27.3'
      to_version:
        description: 'Kubernetes version to upgrade to'
        required: true
        default: 'v1.28.2'

jobs:
  test-upgrade:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        
      - name: Set up kind cluster
        uses: helm/kind-action@v1.5.0
        with:
          node_image: kindest/node:${{ github.event.inputs.from_version || 'v1.27.3' }}
          cluster_name: upgrade-test
          
      - name: Deploy test workloads
        run: |
          kubectl apply -f ./test/workloads/
          
      - name: Wait for workloads to be ready
        run: |
          kubectl wait --for=condition=ready pods --all -n default --timeout=300s
          
      - name: Upgrade kind cluster
        run: |
          kind upgrade cluster --name upgrade-test --image kindest/node:${{ github.event.inputs.to_version || 'v1.28.2' }}
          
      - name: Verify cluster version
        run: |
          kubectl version --short
          
      - name: Verify workloads after upgrade
        run: |
          kubectl wait --for=condition=ready pods --all -n default --timeout=300s
          ./test/verify-workloads.sh
          
      - name: Run end-to-end tests
        run: |
          cd ./test
          go test -v ./e2e
          
      - name: Create upgrade report
        if: always()
        run: |
          echo "# Kubernetes Upgrade Test Report" > upgrade-report.md
          echo "Upgrade from ${{ github.event.inputs.from_version || 'v1.27.3' }} to ${{ github.event.inputs.to_version || 'v1.28.2' }}" >> upgrade-report.md
          echo "Date: $(date)" >> upgrade-report.md
          echo "## Results" >> upgrade-report.md
          echo "- Cluster upgrade: ${{ job.status == 'success' ? 'Success' : 'Failure' }}" >> upgrade-report.md
          
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: upgrade-test-results
          path: |
            upgrade-report.md
            test-results/
```

## Conclusion

Effective Kubernetes maintenance and upgrades are essential for maintaining secure, reliable, and high-performing clusters. By following structured approaches to planning, executing, and validating upgrades, organizations can minimize disruption while keeping their Kubernetes infrastructure current.

Key takeaways:

1. **Plan thoroughly**: Understand version requirements, compatibility, and resource needs
2. **Document everything**: Maintain detailed records of all maintenance activities
3. **Test before production**: Validate upgrades in non-production environments first
4. **Minimize disruption**: Use strategies like rolling updates and PodDisruptionBudgets
5. **Monitor constantly**: Implement specialized monitoring during upgrade windows
6. **Be prepared for failures**: Have well-tested rollback procedures ready
7. **Automate where possible**: Use tools like kured, Cluster API, and GitOps
8. **Continuously improve**: Learn from each upgrade to refine processes

By implementing these practices, organizations can maintain a healthy Kubernetes ecosystem while minimizing the operational burden of regular upgrades and maintenance.

## Additional Resources

- [Kubernetes Version Skew Policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Backing up etcd clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Cluster API Documentation](https://cluster-api.sigs.k8s.io/introduction.html)
- [Flux Documentation](https://fluxcd.io/docs/)
- [Kured: Kubernetes Reboot Daemon](https://github.com/kubereboot/kured)
- [Kubernetes Release Notes](https://kubernetes.io/docs/setup/release/notes/)