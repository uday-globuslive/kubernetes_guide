# Setting up a Kubernetes Cluster

## Introduction

This guide walks through the process of setting up a Kubernetes cluster suitable for deploying the Elastic Stack. A properly configured Kubernetes cluster is essential for running Elasticsearch, Kibana, and other Elastic Stack components in a production environment.

We'll cover several deployment options, from local development clusters to production-grade setups across various cloud providers and on-premises environments. For each option, we'll provide detailed instructions, configuration examples, and best practices.

## Cluster Requirements for Elastic Stack

Before setting up a Kubernetes cluster for the Elastic Stack, it's important to understand the resource requirements:

### Minimum Requirements

- **Kubernetes version**: 1.19 or later
- **Nodes**: At least 3 worker nodes for production (1 for development)
- **CPU**: Minimum 4 cores per node for production workloads
- **Memory**: Minimum 8GB RAM per node
- **Storage**: Fast SSD storage with adequate IOPS
- **Network**: Low-latency networking between nodes

### Recommended Resources

| Component | Nodes | CPU/Node | RAM/Node | Storage/Node |
|-----------|-------|----------|----------|--------------|
| Development | 1-3 | 2 cores | 4GB | 20GB |
| Small Production | 3 | 4 cores | 16GB | 100GB |
| Medium Production | 5+ | 8 cores | 32GB | 200GB+ |
| Large Production | 7+ | 16+ cores | 64GB+ | 500GB+ |

## Local Development Clusters

For local development and testing, there are several options to run a Kubernetes cluster on your local machine.

### Minikube

Minikube is a tool that makes it easy to run Kubernetes locally. It runs a single-node Kubernetes cluster inside a VM on your laptop.

#### Installation

```bash
# macOS with Homebrew
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Windows with Chocolatey
choco install minikube
```

#### Starting Minikube

```bash
# Start with adequate resources for Elastic Stack
minikube start --cpus=4 --memory=8g --disk-size=50g

# Enable necessary addons
minikube addons enable ingress
minikube addons enable storage-provisioner
minikube addons enable metrics-server

# Verify setup
minikube status
kubectl get nodes
```

#### Minikube Dashboard

```bash
# Access the Kubernetes dashboard
minikube dashboard
```

### Kind (Kubernetes IN Docker)

Kind runs Kubernetes clusters using Docker containers as nodes, making it lightweight and suitable for CI environments.

#### Installation

```bash
# macOS or Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-$(uname)-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Windows with Chocolatey
choco install kind
```

#### Creating a Cluster

Create a multi-node Kind cluster with a configuration file:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
- role: worker
  extraMounts:
  - hostPath: /path/to/elasticsearch/data
    containerPath: /var/lib/elasticsearch
- role: worker
  extraMounts:
  - hostPath: /path/to/elasticsearch/data2
    containerPath: /var/lib/elasticsearch
```

Start the cluster:

```bash
kind create cluster --name elastic-cluster --config kind-config.yaml

# Verify setup
kubectl get nodes
```

### Docker Desktop

Docker Desktop for Mac and Windows comes with Kubernetes support built-in.

#### Enabling Kubernetes

1. Open Docker Desktop
2. Go to Preferences/Settings
3. Select the "Kubernetes" tab
4. Check "Enable Kubernetes"
5. Click "Apply & Restart"

```bash
# Verify setup
kubectl get nodes
```

## Cloud-Based Kubernetes Services

For production deployments, managed Kubernetes services from cloud providers offer the best reliability, scalability, and ease of management.

### Amazon EKS (Elastic Kubernetes Service)

#### Setting up EKS with eksctl

Install eksctl:

```bash
# macOS with Homebrew
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Create a cluster configuration file:

```yaml
# eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: elastic-cluster
  region: us-west-2
  version: "1.24"

nodeGroups:
  - name: elasticsearch
    instanceType: m5.2xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 100
    volumeType: gp3
    labels:
      role: elasticsearch
    taints:
      dedicated: elasticsearch:NoSchedule
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
    iam:
      withAddonPolicies:
        autoScaler: true
  
  - name: kibana-logstash
    instanceType: m5.xlarge
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 50
    volumeType: gp3
    labels:
      role: kibana-logstash

managedNodeGroups:
  - name: system-nodes
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 4

addons:
  - name: aws-ebs-csi-driver
    version: latest
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
```

Create the cluster:

```bash
eksctl create cluster -f eks-cluster.yaml

# Alternatively, use the CLI directly
eksctl create cluster \
  --name elastic-cluster \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --with-oidc \
  --ssh-access \
  --ssh-public-key my-key \
  --managed
```

Set up storage for Elasticsearch:

```bash
# Install the EBS CSI driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Create a storage class for Elasticsearch
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "3000"
  throughput: "125"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

### Google GKE (Google Kubernetes Engine)

#### Setting up GKE

Install Google Cloud SDK:

```bash
# macOS with Homebrew
brew install --cask google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

Create a GKE cluster:

```bash
# Enable the GKE API
gcloud services enable container.googleapis.com

# Create a cluster with regional availability, node auto-provisioning and auto-scaling
gcloud container clusters create elastic-cluster \
  --region us-central1 \
  --num-nodes 1 \
  --node-locations us-central1-a,us-central1-b,us-central1-c \
  --machine-type n2-standard-4 \
  --disk-type pd-ssd \
  --disk-size 100 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 3 \
  --enable-autoprovisioning \
  --min-cpu 4 \
  --max-cpu 32 \
  --min-memory 16 \
  --max-memory 128 \
  --enable-ip-alias \
  --enable-autorepair \
  --enable-autoupgrade \
  --scopes cloud-platform

# Get credentials to use with kubectl
gcloud container clusters get-credentials elastic-cluster --region us-central1

# Verify the cluster
kubectl get nodes
```

Create node pools for Elasticsearch:

```bash
gcloud container node-pools create elasticsearch \
  --cluster elastic-cluster \
  --region us-central1 \
  --num-nodes 1 \
  --node-locations us-central1-a,us-central1-b,us-central1-c \
  --machine-type n2-standard-8 \
  --disk-type pd-ssd \
  --disk-size 200 \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 6 \
  --node-taints dedicated=elasticsearch:NoSchedule \
  --node-labels role=elasticsearch

# Create a node pool for Kibana and Logstash
gcloud container node-pools create kibana-logstash \
  --cluster elastic-cluster \
  --region us-central1 \
  --num-nodes 1 \
  --node-locations us-central1-a,us-central1-b,us-central1-c \
  --machine-type n2-standard-4 \
  --disk-type pd-ssd \
  --disk-size 50 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 4 \
  --node-labels role=kibana-logstash
```

Set up storage for Elasticsearch:

```bash
# Create a storage class for Elasticsearch
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

# Set it as the default storage class
kubectl patch storageclass elasticsearch-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Microsoft AKS (Azure Kubernetes Service)

#### Setting up AKS

Install Azure CLI:

```bash
# macOS with Homebrew
brew install azure-cli

# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Create an AKS cluster:

```bash
# Login to Azure
az login

# Create a resource group
az group create --name elastic-cluster-rg --location eastus

# Create AKS cluster
az aks create \
  --resource-group elastic-cluster-rg \
  --name elastic-cluster \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --node-vm-size Standard_DS4_v2 \
  --network-plugin azure \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 6 \
  --zones 1 2 3

# Get credentials for kubectl
az aks get-credentials --resource-group elastic-cluster-rg --name elastic-cluster

# Verify the cluster
kubectl get nodes
```

Create node pools for Elasticsearch:

```bash
az aks nodepool add \
  --resource-group elastic-cluster-rg \
  --cluster-name elastic-cluster \
  --name elasticsearch \
  --node-count 3 \
  --node-vm-size Standard_DS4_v2 \
  --node-taints dedicated=elasticsearch:NoSchedule \
  --labels role=elasticsearch \
  --mode User \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 6 \
  --zones 1 2 3

# Create a node pool for Kibana and Logstash
az aks nodepool add \
  --resource-group elastic-cluster-rg \
  --cluster-name elastic-cluster \
  --name kibanals \
  --node-count 2 \
  --node-vm-size Standard_DS3_v2 \
  --labels role=kibana-logstash \
  --mode User \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 4 \
  --zones 1 2 3
```

Set up storage for Elasticsearch:

```bash
# Create a storage class for Elasticsearch
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-storage
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
  kind: Managed
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

## On-Premises Kubernetes Clusters

For organizations that prefer to run Kubernetes on their own infrastructure, there are several options for setting up a cluster.

### kubeadm

kubeadm is the official Kubernetes cluster bootstrapping tool, ideal for on-premises deployments.

#### Prerequisites

On all nodes:

```bash
# Install container runtime (containerd)
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Install kubeadm, kubelet, and kubectl
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Initialize the Control Plane

On the master node:

```bash
# Initialize the control plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint="<LOAD_BALANCER_IP>:6443" --upload-certs

# Set up kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico network plugin
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Get the join command for worker nodes
kubeadm token create --print-join-command
```

#### Join Worker Nodes

On each worker node, run the join command obtained from the master:

```bash
sudo kubeadm join <LOAD_BALANCER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Set up Storage for Elasticsearch

For on-premises clusters, you can use different storage options:

1. **Local Storage**:

```yaml
# Create a storage class for local SSDs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
# For each node, create a PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-data-0
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disk/elasticsearch-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
```

2. **Rook-Ceph for Distributed Storage**:

```bash
# Clone the Rook repository
git clone --single-branch --branch v1.10.3 https://github.com/rook/rook.git

# Deploy the Rook operator
cd rook/deploy/examples
kubectl create -f crds.yaml
kubectl create -f common.yaml
kubectl create -f operator.yaml

# Create a Ceph cluster
kubectl create -f cluster.yaml

# Create a storage class for Elasticsearch
kubectl create -f csi/rbd/storageclass.yaml
```

### Kubespray

Kubespray is a composition of Ansible playbooks for deploying Kubernetes with advanced configurations.

#### Setting up with Kubespray

```bash
# Clone Kubespray
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# Install dependencies
pip install -r requirements.txt

# Copy the sample inventory
cp -rfp inventory/sample inventory/mycluster

# Update the inventory with your servers
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and modify the inventory as needed
vim inventory/mycluster/hosts.yaml

# Deploy Kubernetes
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```

## Post-Installation Setup

After setting up the Kubernetes cluster, there are several additional components you should configure for a production-ready Elastic Stack deployment.

### Installing Helm

Helm is a package manager for Kubernetes that simplifies application deployment:

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### Setting up Ingress Controller

Nginx Ingress Controller for external access:

```bash
# Add the Nginx Ingress Controller repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install the Nginx Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux
```

### Setting up Certificate Manager

Cert-Manager for TLS certificate management:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml

# Create a ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

### Setting up Monitoring

Prometheus and Grafana for monitoring:

```bash
# Add the Prometheus repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus Operator with Grafana
helm install prometheus prometheus-community/kube-prometheus-stack \
  --set prometheus.prometheusSpec.serviceMonitorSelector.matchLabels.release=prometheus \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### Setting up Logging

Fluent Bit for log collection:

```bash
# Add the Fluent Bit repository
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --set backend.type=es \
  --set backend.es.host=elasticsearch-master \
  --set backend.es.port=9200
```

## Preparing for Elastic Stack Deployment

Before deploying the Elastic Stack, set up the necessary resources in your Kubernetes cluster.

### Create Namespace

```bash
kubectl create namespace elastic
```

### Set Resource Quotas

```yaml
# elastic-resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: elastic-quota
  namespace: elastic
spec:
  hard:
    requests.cpu: "16"
    requests.memory: 64Gi
    limits.cpu: "32"
    limits.memory: 128Gi
    persistentvolumeclaims: "20"
    pods: "50"
    services: "20"
```

Apply the resource quota:

```bash
kubectl apply -f elastic-resource-quota.yaml
```

### Set Network Policies

```yaml
# elastic-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elastic-network-policy
  namespace: elastic
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/instance: elastic
    ports:
    - protocol: TCP
      port: 9200
    - protocol: TCP
      port: 9300
  egress:
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/instance: elastic
```

Apply the network policy:

```bash
kubectl apply -f elastic-network-policy.yaml
```

### Prepare for Persistent Volumes

Verify the storage class is ready:

```bash
kubectl get storageclass
```

## Cluster Validation

Before deploying the Elastic Stack, validate that your Kubernetes cluster is properly configured.

### Validate Node Setup

```bash
# Verify nodes are healthy
kubectl get nodes
kubectl describe nodes

# Check pods in kube-system
kubectl get pods -n kube-system
```

### Validate Storage

```bash
# Create a test PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: elasticsearch-storage
EOF

# Check the PVC status
kubectl get pvc test-pvc

# Cleanup
kubectl delete pvc test-pvc
```

### Validate Network

```bash
# Create a test pod
kubectl run nginx --image=nginx

# Create a service
kubectl expose pod nginx --port=80 --type=ClusterIP

# Test connectivity
kubectl run busybox --rm -it --image=busybox -- wget -O- nginx

# Cleanup
kubectl delete pod nginx
kubectl delete service nginx
```

## Conclusion

You now have a Kubernetes cluster ready for deploying the Elastic Stack. The cluster is configured with the necessary resources, storage, networking, and security policies to run Elasticsearch, Kibana, and other components in a production environment.

In the next section, we'll explore how to use kubectl, the Kubernetes command-line tool, to interact with your cluster and deploy applications.