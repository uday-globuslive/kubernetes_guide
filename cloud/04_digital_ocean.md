# DigitalOcean Kubernetes (DOKS)

## Introduction

DigitalOcean Kubernetes Service (DOKS) is a managed Kubernetes offering that allows developers to deploy containerized applications without the complexity of managing the underlying infrastructure. This guide covers everything you need to know about using DOKS, from setup to advanced configurations, focusing on its unique features and integration with the broader DigitalOcean ecosystem.

## Overview of DigitalOcean Kubernetes

### Key Features of DOKS

DigitalOcean Kubernetes offers several advantages that make it attractive for small to medium-sized businesses and developer teams:

1. **Simplicity**: Streamlined setup and management compared to other Kubernetes distributions
2. **Cost-effectiveness**: Transparent pricing with no hidden costs
3. **Fully managed control plane**: Free, managed Kubernetes control plane
4. **Integration**: Seamless integration with other DigitalOcean services like Volumes, Load Balancers, and Container Registry
5. **Global Availability**: Multiple region options worldwide
6. **Automatic updates**: Regular, managed Kubernetes version updates

### Architecture

DOKS follows a standard Kubernetes architecture with some DigitalOcean-specific components:

1. **Control Plane**:
   - Fully managed by DigitalOcean
   - Highly available across multiple availability zones (in supported regions)
   - Free of charge (compared to some other cloud providers that charge for control plane nodes)

2. **Worker Nodes**:
   - Run on DigitalOcean Droplets (VMs)
   - Available in various sizes and configurations
   - Support for regular or optimized CPU/memory configurations

3. **Networking**:
   - Uses a VPC-native approach
   - DigitalOcean Cloud Controller Manager for integrating with DigitalOcean networking
   - Cilium as the default CNI (Container Network Interface)

4. **Storage**:
   - Integration with DigitalOcean Volumes (Block Storage)
   - CSI (Container Storage Interface) driver for dynamic provisioning

## Getting Started with DOKS

### Creating a DOKS Cluster

You can create a DOKS cluster through the web UI or using the doctl CLI:

#### Using the DigitalOcean Control Panel (Web UI)

1. Navigate to the Kubernetes section in the DigitalOcean Control Panel
2. Click "Create Cluster"
3. Select a Kubernetes version
4. Choose a datacenter region
5. Configure your node pool(s)
6. Select any additional features (VPC, auto-scaling, etc.)
7. Name your cluster
8. Click "Create Cluster"

#### Using the doctl CLI

First, install and configure doctl:

```bash
# Install doctl (macOS example)
brew install doctl

# Authenticate with DigitalOcean
doctl auth init
```

Then create a Kubernetes cluster:

```bash
# List available Kubernetes versions
doctl kubernetes options versions

# Create a cluster
doctl kubernetes cluster create my-cluster \
  --region nyc1 \
  --version 1.28.2-do.0 \
  --count 3 \
  --size s-2vcpu-4gb \
  --tag demo,kubernetes,production
```

### Connecting to Your Cluster

After your cluster is created, you can connect to it using kubectl:

```bash
# Download kubectl config and set as current context
doctl kubernetes cluster kubeconfig save my-cluster

# Verify connection
kubectl get nodes
```

### Exploring the Cluster

Check the components running in your cluster:

```bash
# View all pods across namespaces
kubectl get pods -A

# Check node status
kubectl get nodes -o wide

# View cluster details
doctl kubernetes cluster get my-cluster
```

## Node Management

### Node Pool Configuration

DOKS organizes nodes into node pools, which are groups of nodes with the same configuration:

```bash
# List existing node pools
doctl kubernetes cluster node-pool list my-cluster

# Create a new node pool
doctl kubernetes cluster node-pool create my-cluster \
  --name worker-pool \
  --count 2 \
  --size s-4vcpu-8gb \
  --tag workload,backend \
  --label workload=backend

# Delete a node pool
doctl kubernetes cluster node-pool delete my-cluster worker-pool
```

### Autoscaling Node Pools

DOKS supports horizontal autoscaling of node pools:

```bash
# Create an autoscaling node pool
doctl kubernetes cluster node-pool create my-cluster \
  --name autoscale-pool \
  --count 1 \
  --size s-2vcpu-4gb \
  --tag autoscale \
  --auto-scale \
  --min-nodes 1 \
  --max-nodes 5
```

Configure the Kubernetes Horizontal Pod Autoscaler (HPA) to work with node autoscaling:

```yaml
# example-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: example-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: example-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the HPA configuration:
```bash
kubectl apply -f example-hpa.yaml
```

### Node Upgrades

DOKS allows for upgrading Kubernetes versions and replacing nodes:

```bash
# View available upgrades
doctl kubernetes cluster get my-cluster

# Upgrade a cluster
doctl kubernetes cluster upgrade my-cluster --version 1.28.2-do.0

# Replace nodes in a node pool
doctl kubernetes cluster node-pool recycle my-cluster worker-pool --node-ids node-id-1,node-id-2
```

## Storage and Persistent Data

### DigitalOcean Volumes Integration

DOKS integrates with DigitalOcean Volumes through the CSI driver, which is automatically installed:

```yaml
# example-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-volume
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: do-block-storage
```

Apply the PVC configuration:
```bash
kubectl apply -f example-pvc.yaml
```

Using the volume in a deployment:

```yaml
# example-deployment-with-volume.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-volume
```

### Storage Classes

DOKS provides several storage classes:

1. **do-block-storage** (default): Standard SSD-backed block storage
2. **do-block-storage-retain**: Same as default but with "Retain" reclaim policy
3. **do-block-storage-xfs**: Uses XFS as the filesystem instead of ext4

Create a custom storage class:

```yaml
# premium-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-storage
provisioner: dobs.csi.digitalocean.com
parameters:
  type: premium  # This is a hypothetical parameter
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### Volume Snapshots

DOKS supports volume snapshots:

```yaml
# volume-snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: do-snapshot
driver: dobs.csi.digitalocean.com
deletionPolicy: Delete
```

Create a snapshot:

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-volume-snapshot
spec:
  volumeSnapshotClassName: do-snapshot
  source:
    persistentVolumeClaimName: data-volume
```

## Networking

### Load Balancers

DOKS automatically integrates with DigitalOcean Load Balancers:

```yaml
# load-balancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-protocol: "http"
    service.beta.kubernetes.io/do-loadbalancer-algorithm: "round_robin"
    service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
    service.beta.kubernetes.io/do-loadbalancer-certificate-id: "your-certificate-id"
    service.beta.kubernetes.io/do-loadbalancer-redirect-http-to-https: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    name: http
  - port: 443
    targetPort: 80
    name: https
  selector:
    app: example-app
```

### Network Policies

Configure network policies in DOKS to control traffic flow between pods:

```yaml
# example-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-traffic
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

Apply the network policy:
```bash
kubectl apply -f example-network-policy.yaml
```

### Ingress Controllers

Install and configure an Ingress controller like NGINX:

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/do/deploy.yaml
```

Configure an Ingress resource:

```yaml
# example-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
  tls:
  - hosts:
    - example.com
    secretName: example-tls
```

## Integrating with DigitalOcean Services

### Container Registry

Use DigitalOcean Container Registry with your DOKS cluster:

```bash
# Create a container registry
doctl registry create my-registry

# Log in to the registry
doctl registry login

# Create a Kubernetes secret for registry access
doctl registry kubernetes-manifest | kubectl apply -f -
```

Reference the registry in your deployments:

```yaml
# registry-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: registry-app
  template:
    metadata:
      labels:
        app: registry-app
    spec:
      containers:
      - name: app
        image: registry.digitalocean.com/my-registry/my-app:latest
      imagePullSecrets:
      - name: registry-my-registry
```

### Managed Databases

Connect to a DigitalOcean Managed Database:

```bash
# Create a managed database
doctl databases create my-postgres --engine postgresql --region nyc1 --num-nodes 1 --size db-s-2vcpu-4gb

# Get connection details
doctl databases connection my-postgres --format Certificate,Host,Name,Password,Port,User
```

Create a Kubernetes secret with database credentials:

```yaml
# database-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
data:
  host: <base64-encoded-host>
  port: <base64-encoded-port>
  database: <base64-encoded-database>
  username: <base64-encoded-username>
  password: <base64-encoded-password>
```

Use the secret in your application:

```yaml
# database-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-app
  template:
    metadata:
      labels:
        app: db-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: port
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: database
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
```

### Spaces (Object Storage)

Integrate with DigitalOcean Spaces using the S3-compatible API:

```bash
# Configure credentials for Spaces
export SPACES_ACCESS_KEY_ID=your_access_key
export SPACES_SECRET_ACCESS_KEY=your_secret_key

# Create a Kubernetes secret for Spaces access
kubectl create secret generic spaces-credentials \
  --from-literal=access-key=$SPACES_ACCESS_KEY_ID \
  --from-literal=secret-key=$SPACES_SECRET_ACCESS_KEY
```

Deploy a pod that can access Spaces:

```yaml
# spaces-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: spaces-example
spec:
  containers:
  - name: spaces-client
    image: amazon/aws-cli:latest
    command: ["sleep", "infinity"]
    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: spaces-credentials
          key: access-key
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: spaces-credentials
          key: secret-key
    - name: AWS_DEFAULT_REGION
      value: "nyc3"
    - name: AWS_ENDPOINT_URL
      value: "https://nyc3.digitaloceanspaces.com"
```

Use the AWS CLI to interact with Spaces:

```bash
# Execute commands in the pod
kubectl exec -it spaces-example -- aws s3 ls s3://my-bucket --endpoint-url $AWS_ENDPOINT_URL
```

## Monitoring and Logging

### DigitalOcean Monitoring

DOKS includes basic monitoring through the DigitalOcean control panel, but you can enhance it with additional tools:

#### Prometheus and Grafana Setup

1. Install Prometheus Operator:

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus Stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

2. Access the Grafana dashboard:

```bash
# Forward port to Grafana
kubectl port-forward svc/prometheus-grafana -n monitoring 8080:80

# Get the default credentials
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Logging Solutions

Set up centralized logging with EFK (Elasticsearch, Fluentd, Kibana) stack:

```bash
# Add Elastic Helm repository
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=512Mi

# Install Fluentd
helm install fluentd stable/fluentd \
  --namespace logging \
  --set output.host=elasticsearch-master.logging.svc.cluster.local \
  --set output.port=9200

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts=http://elasticsearch-master.logging:9200
```

## Security Best Practices

### Securing Your DOKS Cluster

1. **Restrict access to your cluster**:

```bash
# Add firewall rules to your cluster
doctl kubernetes cluster update my-cluster --update-kubeconfig
```

2. **Enable and configure RBAC**:

```yaml
# restricted-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "services", "deployments", "jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-binding
  namespace: development
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

3. **Implement Pod Security Standards**:

```yaml
# pod-security.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

4. **Use Private Container Registry**:

```bash
# Make your container registry private
doctl registry configure-docker
```

5. **Network Policies for microsegmentation**:

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Securing Secrets

Use Sealed Secrets or an external secrets manager:

```bash
# Install Sealed Secrets
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets

# Install kubeseal CLI
# Example for macOS
brew install kubeseal

# Seal a secret
kubectl create secret generic my-secret --dry-run=client --from-literal=key=value -o yaml | \
  kubeseal --controller-name=sealed-secrets --controller-namespace=default > sealed-secret.yaml
```

## Cost Optimization

### Optimizing DOKS Costs

1. **Right-size your clusters**:

```bash
# Update node pool to more appropriate size
doctl kubernetes cluster node-pool update my-cluster worker-pool --size s-2vcpu-2gb
```

2. **Use autoscaling effectively**:

```bash
# Enable autoscaling on a node pool
doctl kubernetes cluster node-pool update my-cluster worker-pool \
  --auto-scale \
  --min-nodes 1 \
  --max-nodes 5
```

3. **Implement Kubernetes pod autoscaling**:

```yaml
# resource-efficient-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cost-efficient-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
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
        value: 100
        periodSeconds: 15
```

4. **Use the DigitalOcean Container Registry**:

```bash
# Create a registry
doctl registry create my-registry --subscription-tier basic
```

5. **Implement Resource Quotas**:

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "20"
```

## Backup and Disaster Recovery

### Backup Solutions

Use Velero for Kubernetes backup:

```bash
# Install Velero CLI
# Example for macOS
brew install velero

# Set up Spaces credentials for Velero
export SPACES_ACCESS_KEY_ID=your_access_key
export SPACES_SECRET_ACCESS_KEY=your_secret_key

# Install Velero with DigitalOcean Spaces
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.5.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true \
  --backup-location-config region=nyc3,s3ForcePathStyle=true,s3Url=https://nyc3.digitaloceanspaces.com
```

Create a backup:

```bash
# Backup entire cluster
velero backup create all-cluster-backup --include-cluster-resources=true
```

Backup a specific namespace:

```bash
# Backup a specific namespace
velero backup create production-backup --include-namespaces=production
```

### Disaster Recovery Strategies

1. **Regular backups**:

```bash
# Schedule regular backups
velero schedule create daily-backup --schedule="0 1 * * *" --include-cluster-resources=true
```

2. **Restore from backup**:

```bash
# List available backups
velero backup get

# Restore from a backup
velero restore create --from-backup daily-backup-20231012010000
```

3. **Multi-region strategy**:

```bash
# Create clusters in different regions
doctl kubernetes cluster create backup-cluster \
  --region sfo3 \
  --version 1.28.2-do.0 \
  --count 2 \
  --size s-2vcpu-4gb
```

## Advanced Cluster Management

### Multi-cluster Management

Manage multiple DOKS clusters effectively:

```bash
# List all your clusters
doctl kubernetes cluster list

# Create a merged kubeconfig file
KUBECONFIG=~/.kube/config:~/kubeconfig-cluster1:~/kubeconfig-cluster2 kubectl config view --flatten > ~/.kube/merged-config
export KUBECONFIG=~/.kube/merged-config

# Switch between contexts
kubectl config use-context do-nyc1-my-cluster
kubectl config use-context do-sfo3-backup-cluster
```

### CI/CD Integration

Integrate DOKS with CI/CD pipelines:

Example GitHub Actions workflow:

```yaml
# .github/workflows/deploy-to-doks.yml
name: Deploy to DOKS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Install doctl
      uses: digitalocean/action-doctl@v2
      with:
        token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
    
    - name: Build container image
      run: docker build -t registry.digitalocean.com/my-registry/my-app:${GITHUB_SHA::8} .
    
    - name: Log in to DigitalOcean Container Registry
      run: doctl registry login --expiry-seconds 600
    
    - name: Push image to DigitalOcean Container Registry
      run: docker push registry.digitalocean.com/my-registry/my-app:${GITHUB_SHA::8}
    
    - name: Update deployment file
      run: |
        sed -i 's|registry.digitalocean.com/my-registry/my-app:.*|registry.digitalocean.com/my-registry/my-app:'"${GITHUB_SHA::8}"'|' deployment.yaml
    
    - name: Save KUBECONFIG
      run: doctl kubernetes cluster kubeconfig save my-cluster
    
    - name: Deploy to DigitalOcean Kubernetes
      run: kubectl apply -f deployment.yaml
```

### Cluster Autoscaler

Configure and optimize cluster autoscaler:

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.23.0
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=digitalocean
        - --nodes=1:5:worker-pool
        env:
        - name: DO_TOKEN
          valueFrom:
            secretKeyRef:
              name: digitalocean
              key: access-token
```

## Troubleshooting Common Issues

### Connectivity Issues

If you can't connect to your cluster:

```bash
# Regenerate your kubeconfig
doctl kubernetes cluster kubeconfig save my-cluster

# Check if your cluster is running
doctl kubernetes cluster get my-cluster

# Check if nodes are running
kubectl get nodes
```

### Pod Scheduling Issues

If pods are stuck in Pending state:

```bash
# Check the pod description for events
kubectl describe pod <pod-name>

# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check for taints on nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Persistent Volume Issues

If PVCs are stuck in Pending state:

```bash
# Check the PVC status
kubectl describe pvc <pvc-name>

# Check StorageClass
kubectl get storageclass

# Verify CSI driver is running
kubectl get pods -n kube-system | grep csi
```

## Migration Strategies

### Migrating to DOKS

Steps to migrate an existing Kubernetes workload to DOKS:

1. **Export resources from existing cluster**:

```bash
# Export all resources (excluding system resources)
NAMESPACES=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v -E 'kube-.*|default')
for NS in $NAMESPACES; do
  kubectl -n $NS get -o yaml \
    deployment,statefulset,service,configmap,secret,pvc > $NS-export.yaml
done
```

2. **Prepare for data migration**:

```bash
# Install Velero on source cluster
velero install --provider aws --bucket migration-backup --secret-file ./credentials

# Create a backup
velero backup create migration-backup --include-namespaces=<namespace>
```

3. **Restore to DOKS cluster**:

```bash
# Install Velero on DOKS
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.5.0 \
  --bucket migration-backup \
  --secret-file ./credentials-velero

# Restore from backup
velero restore create --from-backup migration-backup
```

## Use Cases and Examples

### Microservices Architecture on DOKS

Example of deploying a microservices architecture:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
---
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: microservices
spec:
  replicas: 3
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
        image: registry.digitalocean.com/my-registry/frontend:latest
        ports:
        - containerPort: 80
---
# frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: microservices
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: frontend
---
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: microservices
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: registry.digitalocean.com/my-registry/backend:latest
        ports:
        - containerPort: 8080
---
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: microservices
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: backend
```

### Web Application with Database

Example of a web application connected to a DigitalOcean Managed Database:

```yaml
# webapp-with-db.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: webapp
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded "admin"
  password: cGFzc3dvcmQ=  # base64 encoded "password"
  host: ZGItcG9zdGdyZXMtbnljMS00Ny5kYi5kaWdpdGFsb2NlYW4uY29t  # example host
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: registry.digitalocean.com/my-registry/webapp:latest
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: host
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: webapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: webapp
```

## Conclusion

DigitalOcean Kubernetes Service offers a streamlined, cost-effective managed Kubernetes solution that's particularly well-suited for small to medium-sized teams and businesses. Its integration with other DigitalOcean services, straightforward pricing model, and user-friendly experience make it an attractive option for organizations looking to deploy containerized applications without the overhead of managing complex infrastructure.

Key advantages of DOKS include:

1. **Simplicity**: Easy setup and management compared to self-managed Kubernetes
2. **Cost-effectiveness**: Transparent pricing with no control plane charges
3. **Seamless integration**: Works well with other DigitalOcean services
4. **Developer-friendly**: Designed for developer productivity
5. **Global availability**: Deploy in multiple regions worldwide

By following the best practices outlined in this guide, you can effectively leverage DOKS to build scalable, reliable, and efficient containerized applications.

## Additional Resources

- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [doctl CLI Documentation](https://docs.digitalocean.com/reference/doctl/)
- [DigitalOcean Cloud Control Panel](https://cloud.digitalocean.com/kubernetes/clusters)
- [Kubernetes Official Documentation](https://kubernetes.io/docs/home/)
- [CNCF Kubernetes Community](https://www.cncf.io/projects/kubernetes/)