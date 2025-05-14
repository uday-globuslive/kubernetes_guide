# Kubernetes on Google Cloud (GKE)

## Introduction to Google Kubernetes Engine (GKE)

Google Kubernetes Engine (GKE) is a managed Kubernetes service offered by Google Cloud Platform (GCP) that allows users to deploy, manage, and scale containerized applications on the cloud using Google's infrastructure. As the birthplace of Kubernetes, GCP offers a highly optimized and well-integrated Kubernetes experience.

## Key Features of GKE

### Fully Managed Kubernetes

- Automated control plane management
- Automatic updates and patches
- Google SRE team monitoring
- Simplified operations
- Node auto-repair and health monitoring

### Deployment Options

#### Standard Clusters

- User-managed node pools
- Full control over cluster configuration
- Regional or zonal deployment options
- Support for various machine types

#### Autopilot Clusters

- Fully managed node infrastructure
- Pay only for pods, not nodes
- Automatic resource optimization
- Simplified operations model
- Enhanced security by default

### Advanced Networking

- VPC-native clusters
- Network policies
- Private clusters
- Cloud Load Balancing integration
- Cloud Service Mesh support
- Cloud CDN integration

### Security Features

- GKE Sandbox (gVisor)
- Binary Authorization
- Vulnerability scanning
- Automatic node upgrades
- Private GKE clusters
- Workload Identity
- Confidential computing

### Observability and Monitoring

- Cloud Monitoring integration
- Cloud Logging integration
- Error reporting
- Integrated dashboards
- Managed Prometheus (Preview)

### Scalability

- Node auto-provisioning
- Horizontal Pod Autoscaling
- Vertical Pod Autoscaling
- Cluster Autoscaler
- Multi-cluster support
- Multi-regional deployments

## Getting Started with GKE

### Setting Up GKE with Google Cloud CLI

1. **Install Google Cloud SDK**

```bash
# Install Google Cloud SDK
curl https://sdk.cloud.google.com | bash

# Restart shell
exec -l $SHELL

# Initialize gcloud
gcloud init
```

2. **Create a GKE Cluster**

```bash
# Standard cluster
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-standard-4

# Autopilot cluster
gcloud container clusters create-auto my-autopilot-cluster \
  --region us-central1
```

3. **Get Credentials for kubectl**

```bash
gcloud container clusters get-credentials my-cluster --zone us-central1-a
```

4. **Verify Connection**

```bash
kubectl get nodes
```

### Setting Up GKE with Terraform

```hcl
provider "google" {
  project = "my-project-id"
  region  = "us-central1"
}

resource "google_container_cluster" "primary" {
  name     = "my-gke-cluster"
  location = "us-central1"
  
  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Network settings
  network    = "default"
  subnetwork = "default"
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-node-pool"
  location   = "us-central1"
  cluster    = google_container_cluster.primary.name
  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    
    # Google recommends custom service accounts with minimum permissions
    service_account = google_service_account.default.email

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}

resource "google_service_account" "default" {
  account_id   = "gke-service-account"
  display_name = "GKE Service Account"
}
```

## Working with GKE

### Node Pools

Node pools allow you to have different types of nodes within the same cluster. This is useful for workloads with different resource requirements.

```bash
# Add a node pool
gcloud container node-pools create high-memory-pool \
  --cluster my-cluster \
  --zone us-central1-a \
  --num-nodes 2 \
  --machine-type e2-highmem-8
```

### Autoscaling Configuration

```bash
# Enable autoscaling for a node pool
gcloud container clusters update my-cluster \
  --zone us-central1-a \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 10 \
  --node-pool default-pool

# Configure Horizontal Pod Autoscaler
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=80
```

### Upgrading GKE Clusters

```bash
# List available versions
gcloud container get-server-config --zone us-central1-a

# Upgrade control plane
gcloud container clusters upgrade my-cluster \
  --zone us-central1-a \
  --master

# Upgrade node pool
gcloud container clusters upgrade my-cluster \
  --zone us-central1-a \
  --node-pool default-pool
```

### Private Clusters

Private clusters use internal IP addresses for nodes, enhancing security.

```bash
# Create a private cluster
gcloud container clusters create private-cluster \
  --zone us-central1-a \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.0/28 \
  --network my-network \
  --subnetwork my-subnet
```

## Integrating GKE with Other Google Cloud Services

### Cloud Storage

```yaml
# Pod using Google Cloud Storage via Workload Identity
apiVersion: v1
kind: Pod
metadata:
  name: storage-app
  annotations:
    iam.gke.io/gcp-service-account: gcs-user@my-project.iam.gserviceaccount.com
spec:
  containers:
  - name: storage-app
    image: gcr.io/my-project/storage-app:latest
  serviceAccountName: gcs-k8s-sa
```

### Cloud SQL

```yaml
# Deployment with Cloud SQL proxy sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-cloudsql
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: application
        image: gcr.io/my-project/my-app:latest
      - name: cloudsql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.23.0
        command:
          - "/cloud_sql_proxy"
          - "-instances=my-project:us-central1:my-db=tcp:5432"
        securityContext:
          runAsNonRoot: true
```

### Cloud Pub/Sub

```yaml
# Deployment with Pub/Sub subscription
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pubsub-subscriber
spec:
  replicas: 3
  selector:
    matchLabels:
      app: subscriber
  template:
    metadata:
      labels:
        app: subscriber
    spec:
      containers:
      - name: subscriber
        image: gcr.io/my-project/pubsub-app:latest
        env:
        - name: PUBSUB_SUBSCRIPTION
          value: "projects/my-project/subscriptions/my-subscription"
```

### Cloud IAM and Workload Identity

Workload Identity allows Kubernetes service accounts to act as Google Cloud service accounts.

```bash
# Enable Workload Identity for a cluster
gcloud container clusters update my-cluster \
  --zone us-central1-a \
  --workload-pool=my-project.svc.id.goog

# Create a Kubernetes service account
kubectl create serviceaccount app-sa

# Create IAM policy binding
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[default/app-sa]" \
  gcp-sa@my-project.iam.gserviceaccount.com

# Annotate the Kubernetes service account
kubectl annotate serviceaccount app-sa \
  iam.gke.io/gcp-service-account=gcp-sa@my-project.iam.gserviceaccount.com
```

## Advanced GKE Features

### GKE Config Sync (GitOps)

Config Sync is part of GKE's Config Management, enabling GitOps workflows.

```yaml
# configsync.yaml
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  clusterName: my-cluster
  git:
    syncRepo: https://github.com/my-org/kubernetes-configs
    syncBranch: main
    secretType: none
  sourceFormat: unstructured
```

### GKE Multi-cluster Gateway

Multi-cluster Gateway provides unified ingress across multiple GKE clusters.

```yaml
# multi-cluster-service.yaml
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: my-service
  namespace: default
spec:
  template:
    spec:
      selector:
        app: my-app
      ports:
      - name: web
        protocol: TCP
        port: 80
        targetPort: 8080

# multi-cluster-ingress.yaml
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: my-ingress
  namespace: default
spec:
  template:
    spec:
      backend:
        serviceName: my-service
        servicePort: 80
```

### Binary Authorization

Binary Authorization ensures only trusted container images are deployed.

```yaml
# Pod with Binary Authorization attestation
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  annotations:
    alpha.image-policy.k8s.io/break-glass: "true"  # Only for emergencies
spec:
  containers:
  - name: secure-app
    image: gcr.io/my-project/secure-app:signed
```

### Cloud Service Mesh (Anthos Service Mesh)

Anthos Service Mesh provides a managed service mesh based on Istio.

```bash
# Enable ASM for a cluster
gcloud container clusters update my-cluster \
  --zone us-central1-a \
  --enable-mesh-certificates

# Install ASM
gcloud beta container fleet mesh enable

gcloud container fleet memberships register my-cluster \
  --project=my-project \
  --gke-cluster=us-central1-a/my-cluster \
  --enable-workload-identity

gcloud beta container fleet mesh update \
  --management automatic \
  --memberships my-cluster
```

## Best Practices for GKE

### Security Best Practices

- Use Workload Identity instead of node-level service account keys
- Enable VPC Service Controls for additional isolation
- Implement Binary Authorization for image validation
- Use private clusters to minimize exposure
- Enable network policies to control pod-to-pod communication
- Regularly update cluster and node versions
- Use Pod Security Policies or GKE Sandbox for workload isolation

### Reliability Best Practices

- Use regional clusters for high availability
- Implement proper resource requests and limits
- Set up appropriate node auto-provisioning
- Use Pod Disruption Budgets for controlled maintenance
- Implement health checks and readiness probes
- Use multi-region deployments for critical applications
- Configure appropriate backups for stateful workloads

### Cost Optimization

- Use Autopilot for simplicity and cost efficiency
- Implement Cluster Autoscaler for dynamic capacity
- Use committed use discounts for predictable workloads
- Consider spot VMs for fault-tolerant workloads
- Use GKE usage metering for cost allocation
- Implement resource quotas to limit consumption
- Optimize container images for faster startup and lower resource usage

### Performance Optimization

- Choose appropriate machine types for workloads
- Use node auto-provisioning for specialized hardware needs
- Configure cluster and pod autoscaling thresholds appropriately
- Use regional persistent disks for critical storage
- Implement efficient container image caching
- Use Google Cloud CDN for edge caching
- Consider GPU or TPU nodes for ML/AI workloads

## Monitoring and Logging for GKE

### Cloud Monitoring Integration

```yaml
# ConfigMap for Prometheus configuration to scrape custom metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
```

### Cloud Logging Integration

```yaml
# Fluentd ConfigMap for custom log parsing
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <match kubernetes.var.log.containers.**>
      @type rewrite_tag_filter
      <rule>
        key log
        pattern /ERROR|WARN|FATAL/
        tag high_priority
      </rule>
    </match>
```

### Custom Dashboards

Cloud Monitoring allows creating custom dashboards for GKE metrics.

```terraform
resource "google_monitoring_dashboard" "gke_dashboard" {
  dashboard_json = <<EOF
{
  "displayName": "GKE Cluster Dashboard",
  "gridLayout": {
    "columns": "2",
    "widgets": [
      {
        "title": "CPU Usage",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"kubernetes.io/container/cpu/core_usage_time\" resource.type=\"k8s_container\"",
                "aggregation": {
                  "alignmentPeriod": "60s",
                  "perSeriesAligner": "ALIGN_RATE"
                }
              }
            }
          }]
        }
      }
    ]
  }
}
EOF
}
```

## Troubleshooting GKE

### Common Issues and Solutions

#### Cluster Creation Fails

- Check quota limits
- Verify VPC and subnet configurations
- Ensure IAM permissions are correct

```bash
# Check for quota issues
gcloud compute project-info describe --project my-project
```

#### Nodes Not Joining Cluster

- Check node network connectivity
- Verify node service account permissions
- Check node VM health

```bash
# SSH to problematic node
gcloud compute ssh gke-my-cluster-default-pool-12345678-abcd

# Check kubelet logs
sudo journalctl -u kubelet
```

#### Pod Scheduling Issues

- Check resource constraints
- Verify node selectors and affinities
- Check for taint/toleration issues

```bash
# Describe the pod to see scheduling events
kubectl describe pod problematic-pod

# Check available resources
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Diagnostic Tools

```bash
# GKE Workload Metrics
gcloud container clusters update my-cluster \
  --zone us-central1-a \
  --enable-master-metrics \
  --enable-master-logs

# Enable GKE System Logs
gcloud container clusters update my-cluster \
  --zone us-central1-a \
  --logging-service logging.googleapis.com

# Run diagnostic checks using kubectl
kubectl get componentstatuses
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Conclusion

Google Kubernetes Engine offers a fully managed Kubernetes experience with tight integration to other Google Cloud services. From small startups to large enterprises, GKE provides the flexibility, security, and reliability needed for modern cloud-native applications. By following the best practices and leveraging GKE's advanced features, organizations can build robust, secure, and scalable container-based applications.

## Further Resources

- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [GKE Tutorials](https://cloud.google.com/kubernetes-engine/docs/tutorials)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [Google Cloud Architecture Center](https://cloud.google.com/architecture)