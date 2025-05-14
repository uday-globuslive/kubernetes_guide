# Kubernetes on AWS (EKS)

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it easy to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane. This chapter provides a comprehensive guide to using EKS effectively, from basic setup to advanced operational considerations.

## Table of Contents

- [EKS Architecture](#eks-architecture)
- [Setting Up an EKS Cluster](#setting-up-an-eks-cluster)
- [Node Management](#node-management)
- [Networking on EKS](#networking-on-eks)
- [Storage Options](#storage-options)
- [Security Best Practices](#security-best-practices)
- [Cost Optimization](#cost-optimization)
- [Observability and Monitoring](#observability-and-monitoring)
- [AWS Service Integrations](#aws-service-integrations)
- [Advanced EKS Features](#advanced-eks-features)
- [Common Operational Tasks](#common-operational-tasks)
- [Troubleshooting EKS](#troubleshooting-eks)
- [Production Checklist](#production-checklist)

## EKS Architecture

Amazon EKS provides a managed Kubernetes control plane that spans multiple AWS availability zones for high availability:

### Managed Control Plane

- **Highly Available**: Runs across multiple AZs
- **Managed etcd**: AWS handles etcd cluster setup, backups, and scaling
- **Managed API Servers**: Load-balanced Kubernetes API endpoints
- **Managed Scheduler & Controller Manager**: Core K8s components
- **Automatic Upgrades**: Minor version updates with upgrade windows

### Worker Node Architecture

- **Self-Managed Nodes**: EC2 instances you manage
- **Managed Node Groups**: EC2 instances managed by AWS
- **Fargate Profiles**: Serverless compute for pods

```
┌─────────────────────────────────────────────────────────────────┐
│                    Amazon EKS Architecture                      │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  AWS-Managed Control Plane                │  │
│  │                                                           │  │
│  │    ┌─────────┐      ┌─────────┐      ┌─────────┐         │  │
│  │    │ API     │      │etcd     │      │Controller│        │  │
│  │    │ Server  │      │Cluster  │      │Manager   │        │  │
│  │    └─────────┘      └─────────┘      └─────────┘         │  │
│  │                                                           │  │
│  │    ┌─────────┐                                            │  │
│  │    │Scheduler│                                            │  │
│  │    └─────────┘                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                               │                                  │
│                               │                                  │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │                     Worker Nodes                          │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │ Self-Managed │  │Managed Node  │  │ Fargate      │     │  │
│  │  │ Nodes (EC2)  │  │Group (EC2)   │  │ (Serverless) │     │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Networking Model

- **VPC CNI Plugin**: Provides native AWS VPC networking for pods
- **Pods get IPs from VPC subnets**: Direct integration with VPC networking
- **Security Groups for Pods**: Network policy implementation
- **Load Balancer Integration**: AWS Load Balancers for Services

### API Flow

When you interact with an EKS cluster, this is the flow:

1. **Authentication**: AWS IAM authenticates your request
2. **API Call**: Your request reaches the EKS control plane
3. **Kubernetes Processing**: Standard Kubernetes API handling occurs
4. **Worker Node Interaction**: Changes propagate to worker nodes

## Setting Up an EKS Cluster

### Prerequisites

- AWS CLI installed and configured
- eksctl command-line tool installed
- kubectl command-line tool installed
- IAM permissions to create EKS clusters and related resources

### Using eksctl (Recommended)

eksctl is the official CLI for Amazon EKS that simplifies cluster creation:

```bash
# Create a basic cluster
eksctl create cluster --name my-cluster --region us-west-2 --node-type t3.medium --nodes 3

# Create cluster with a config file
eksctl create cluster -f cluster-config.yaml
```

Example cluster configuration file:

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-eks-cluster
  region: us-west-2
  version: "1.24"

vpc:
  cidr: "10.0.0.0/16"
  clusterEndpoints:
    privateAccess: true
    publicAccess: true

managedNodeGroups:
  - name: managed-ng-1
    instanceType: m5.large
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 80
    privateNetworking: true
    availabilityZones: ["us-west-2a", "us-west-2b", "us-west-2c"]
    labels: 
      role: worker
    tags:
      nodegroup-type: standard
    iam:
      withAddonPolicies:
        albIngress: true
        externalDNS: true
        certManager: true
        autoScaler: true

fargate:
  profiles:
    - name: fp-default
      selectors:
        - namespace: default
        - namespace: kube-system
```

### Using AWS Console

You can also create a cluster through the AWS Management Console:

1. Open the EKS console
2. Click "Add cluster" → "Create"
3. Configure cluster settings:
   - Name, Kubernetes version
   - IAM role for service account
   - VPC and subnet configuration
   - Logging options
4. Create/select cluster IAM role
5. Configure networking
6. Configure logging
7. Review and create
8. Add node groups or Fargate profiles

### Getting kubectl Access

After creating your cluster, configure kubectl:

```bash
# Update kubeconfig for your cluster
aws eks update-kubeconfig --name my-cluster --region us-west-2

# Verify connectivity
kubectl get nodes
```

## Node Management

EKS offers multiple options for worker nodes:

### Managed Node Groups

Managed node groups automate the provisioning and lifecycle of EC2 instances:

```bash
# Create a managed node group with eksctl
eksctl create nodegroup \
  --cluster=my-cluster \
  --region=us-west-2 \
  --name=my-mng \
  --node-type=t3.large \
  --nodes=3 \
  --nodes-min=2 \
  --nodes-max=5 \
  --ssh-access \
  --ssh-public-key=~/.ssh/id_rsa.pub
```

Benefits of managed node groups:
- Automated node provisioning and updates
- Managed scaling
- Simplified security patching
- Health monitoring and auto-replacement
- Graceful node updates and terminations

### Self-Managed Nodes

Self-managed nodes provide more control but require more management:

```bash
# Create a self-managed node group
eksctl create nodegroup \
  --cluster=my-cluster \
  --region=us-west-2 \
  --name=my-self-managed-ng \
  --node-type=t3.large \
  --nodes=3 \
  --node-private-networking \
  --managed=false
```

When to use self-managed nodes:
- Custom AMIs required
- GPU workloads with specific drivers
- Specialized kernel settings
- Extended instance types not supported by managed nodes
- Custom bootstrap scripts

### Fargate

AWS Fargate provides serverless Kubernetes pods:

```bash
# Create a Fargate profile
eksctl create fargateprofile \
  --cluster my-cluster \
  --name my-fargate-profile \
  --namespace default \
  --labels app=serverless
```

Fargate characteristics:
- No nodes to manage
- Pod-level isolation
- Pay-per-pod pricing model
- Limited to specific instance sizes
- No DaemonSets
- No privileged containers
- No hostNetwork, hostPort

### Node Autoscaling

Set up Cluster Autoscaler for automatic node scaling:

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
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --v=4
```

## Networking on EKS

EKS networking leverages AWS VPC infrastructure:

### VPC Considerations

Design your VPC for EKS:
- Sufficient IP address space (CIDR planning)
- Public and private subnets
- NAT gateways for private subnet outbound traffic
- Subnet tagging for EKS

Example VPC setup with eksctl:

```yaml
# vpc-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cluster
  region: us-west-2

vpc:
  id: "vpc-12345"  # Use existing VPC
  subnets:
    private:
      us-west-2a:
        id: "subnet-12345"
      us-west-2b: 
        id: "subnet-67890"
    public:
      us-west-2a:
        id: "subnet-abcdef"
      us-west-2b:
        id: "subnet-ghijkl"
```

### AWS VPC CNI

The AWS VPC CNI plugin provides:
- AWS VPC IP addresses to pods
- ENI (Elastic Network Interface) configuration
- Security group integration

Important CNI configurations:
```bash
# Enable prefix assignment (more IPs per node)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Enable custom networking (use different subnet for pods)
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# Configure secondary IP ranges
kubectl set env daemonset aws-node -n kube-system WARM_IP_TARGET=5
```

### Service Types and Load Balancers

EKS integrates with AWS load balancers:

1. **Classic Load Balancer (CLB)**: Legacy option
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     type: LoadBalancer
     ports:
     - port: 80
     selector:
       app: my-app
   ```

2. **Network Load Balancer (NLB)**: For TCP/UDP traffic
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-nlb-service
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-type: nlb
   spec:
     type: LoadBalancer
     ports:
     - port: 80
     selector:
       app: my-app
   ```

3. **Application Load Balancer (ALB)**: For HTTP/HTTPS traffic (with AWS Load Balancer Controller)
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: my-ingress
     annotations:
       kubernetes.io/ingress.class: alb
       alb.ingress.kubernetes.io/scheme: internet-facing
   spec:
     rules:
     - host: example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: my-service
               port:
                 number: 80
   ```

### AWS Load Balancer Controller

Install the AWS Load Balancer Controller for Ingress resources:

```bash
# Install AWS Load Balancer Controller using Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Security Groups for Pods

Enable security groups for pods to apply AWS Security Group rules directly to pods:

```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: allow-db-access
spec:
  podSelector:
    matchLabels:
      app: web
  securityGroups:
    groupIds:
      - sg-12345 # Security group ID allowing access to RDS
```

## Storage Options

EKS supports multiple storage options for your workloads:

### Amazon EBS CSI Driver

The EBS CSI driver enables Kubernetes StorageClasses backed by Amazon EBS volumes:

```bash
# Install EBS CSI Driver
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --role-name EKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Enable the EBS CSI driver as an addon
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/EKS_EBS_CSI_DriverRole \
  --force
```

Create a StorageClass for EBS volumes:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
```

Create a PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi
```

### Amazon EFS CSI Driver

For ReadWriteMany volumes, use Amazon EFS:

```bash
# Install EFS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Create a StorageClass for EFS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345
  directoryPerms: "700"
```

### FSx for Lustre CSI Driver

For high-performance workloads:

```bash
# Install FSx for Lustre CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-fsx-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

## Security Best Practices

### IAM Roles for Service Accounts (IRSA)

IRSA provides fine-grained IAM permissions to pods:

```bash
# Create IAM role for service account
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Use the service account in a pod
apiVersion: v1
kind: Pod
metadata:
  name: s3-reader
spec:
  serviceAccountName: s3-reader
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ['sleep', '3600']
```

### Network Security

1. **Private API Endpoint**: Limit API server access
   ```bash
   eksctl utils update-cluster-endpoints \
     --name=my-cluster \
     --private-access=true \
     --public-access=false \
     --approve
   ```

2. **Security Groups**: Restrict inbound/outbound traffic

3. **Network Policies**: Use Calico or security groups for pods

### Pod Security

1. **Pod Security Standards**: Enforce pod security with admission controllers
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: restricted
     labels:
       pod-security.kubernetes.io/enforce: restricted
       pod-security.kubernetes.io/audit: restricted
       pod-security.kubernetes.io/warn: restricted
   ```

2. **Security Contexts**: Set permissions at container level
   ```yaml
   securityContext:
     runAsUser: 1000
     runAsGroup: 3000
     fsGroup: 2000
     runAsNonRoot: true
     readOnlyRootFilesystem: true
   ```

### Secrets Management

1. **AWS Secrets Manager**: Use External Secrets Operator
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: database-credentials
   spec:
     refreshInterval: 1h
     secretStoreRef:
       name: aws-secretsmanager
       kind: ClusterSecretStore
     target:
       name: database-credentials
     data:
     - secretKey: username
       remoteRef:
         key: prod/db/credentials
         property: username
     - secretKey: password
       remoteRef:
         key: prod/db/credentials
         property: password
   ```

2. **AWS KMS**: Encrypt Kubernetes secrets
   ```bash
   # Enable KMS encryption for secrets
   eksctl utils enable-secrets-encryption \
     --cluster=my-cluster \
     --key-arn=arn:aws:kms:region:account:key/key-id
   ```

### Audit Logging

Enable AWS CloudTrail and Kubernetes audit logs:

```bash
# Enable control plane logging
eksctl utils update-cluster-logging \
  --cluster=my-cluster \
  --enable-types=api,audit,authenticator,controllerManager,scheduler
```

## Cost Optimization

### Node Right-Sizing

Choose appropriate instances:
- Use EC2 Spot Instances for non-critical workloads
- Deploy with Managed Node Groups for simpler management
- Consider Graviton (ARM) instances for cost savings

```yaml
# Mixed instance types in a nodegroup
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-west-2
managedNodeGroups:
  - name: spot-ng
    instanceTypes: ["m5.large", "m5a.large", "m5n.large", "m5ad.large"]
    spot: true
    minSize: 2
    maxSize: 5
```

### Spot Instances

Use Spot Instances effectively:
- Set up multiple instance types
- Configure correct interruption handling
- Implement node selectors for appropriate workloads

```yaml
# Pod tolerating spot instances
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "kubernetes.amazonaws.com/spot"
    operator: "Exists"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "kubernetes.amazonaws.com/lifecycle"
            operator: In
            values:
            - spot
```

### Cluster Autoscaler

Set up Cluster Autoscaler to scale nodes based on demand:

```bash
# Install Cluster Autoscaler with Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set 'autoDiscovery.clusterName'=my-cluster \
  --set 'awsRegion'=us-west-2
```

### Horizontal Pod Autoscaler

Scale pods based on CPU/memory/custom metrics:

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
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Karpenter (next-gen autoscaling)

Karpenter provides more flexible autoscaling:

```bash
# Install Karpenter with Helm
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.create=true \
  --set serviceAccount.name=karpenter \
  --set clusterName=my-cluster \
  --set clusterEndpoint=$(aws eks describe-cluster --name my-cluster --query "cluster.endpoint" --output text) \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile
```

Create a Karpenter Provisioner:

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64", "arm64"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  provider:
    subnetSelector:
      karpenter.sh/discovery: "true"
    securityGroupSelector:
      karpenter.sh/discovery: "true"
    instanceProfile: KarpenterNodeInstanceProfile
  ttlSecondsAfterEmpty: 30
```

## Observability and Monitoring

### Container Insights

Enable AWS CloudWatch Container Insights:

```bash
# Using Fluent Bit for CloudWatch Logs
ClusterName=my-cluster
RegionName=us-west-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -
```

### Prometheus and Grafana

Install Prometheus and Grafana:

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace

# Install Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set service.type=LoadBalancer
```

### AWS Distro for OpenTelemetry (ADOT)

Install ADOT for full observability:

```bash
# Install ADOT Operator
kubectl apply -f https://github.com/aws-observability/aws-otel-collector/releases/latest/download/adot-operator.yaml

# Create a collector instance
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: my-collector
spec:
  mode: deployment
  config: |
    receivers:
      otlp:
        protocols:
          http:
            endpoint: 0.0.0.0:4318
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch:
        timeout: 1s
        send_batch_size: 50
    exporters:
      awsxray:
      awsemf:
        namespace: EKS
        log_group_name: '/aws/eks/my-cluster/application'
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsemf]
```

## AWS Service Integrations

### IAM Authentication

AWS IAM authenticates users to EKS:

```bash
# Add an IAM user or role to your cluster
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --arn arn:aws:iam::123456789012:user/admin \
  --username admin \
  --group system:masters
```

### AWS Load Balancer Controller

For Ingress and Service resources:

```yaml
# Application Load Balancer Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/certificate-id
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### ExternalDNS

Automatically create Route53 DNS records:

```bash
# Install ExternalDNS with Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set aws.zoneType=public \
  --set txtOwnerId=my-cluster \
  --set policy=sync \
  --set domainFilters[0]=example.com

# Use with a Service
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.example.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
```

### AWS App Mesh

Set up service mesh for microservices:

```bash
# Install App Mesh controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade -i appmesh-controller eks/appmesh-controller \
  --namespace kube-system \
  --set region=us-west-2 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=appmesh-controller
```

### AWS X-Ray

Implement distributed tracing:

```yaml
# X-Ray daemon as a DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xray-daemon
spec:
  selector:
    matchLabels:
      app: xray-daemon
  template:
    metadata:
      labels:
        app: xray-daemon
    spec:
      containers:
      - name: xray-daemon
        image: amazon/aws-xray-daemon
        ports:
        - containerPort: 2000
          hostPort: 2000
          protocol: UDP
```

## Advanced EKS Features

### EKS Anywhere

EKS Anywhere extends EKS to on-premises:

```bash
# Install eksctl anywhere
curl -o /usr/local/bin/eksctl-anywhere -LO "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/1/artifacts/eks-a/v0.12.0/linux/amd64/eksctl-anywhere"
chmod +x /usr/local/bin/eksctl-anywhere

# Create a cluster spec
eksctl anywhere generate clusterconfig cluster-name \
  --provider vmware > vmware-cluster.yaml

# Create the cluster
eksctl anywhere create cluster -f vmware-cluster.yaml
```

### EKS Distro

Use EKS Distro for consistent Kubernetes environments:

```bash
# Pull EKS Distro images
docker pull public.ecr.aws/eks-distro/kubernetes/kube-apiserver:v1.23.13-eks-1-23-9
```

### EKS with AWS Bottlerocket

Use Bottlerocket as a container-optimized OS:

```yaml
# Bottlerocket nodes with eksctl
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-west-2
nodeGroups:
  - name: ng-bottlerocket
    instanceType: m5.large
    desiredCapacity: 2
    ami: auto-bottlerocket
    amiFamily: Bottlerocket
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

## Common Operational Tasks

### Cluster Updates

Update your EKS cluster version:

```bash
# Check available versions
aws eks describe-addon-versions --kubernetes-version 1.24

# Update cluster version
eksctl upgrade cluster --name=my-cluster --version=1.24 --approve
```

### Node Updates

Update your worker nodes:

```bash
# Update managed node group
eksctl upgrade nodegroup \
  --name=managed-ng-1 \
  --cluster=my-cluster \
  --kubernetes-version=1.24
```

### Scaling

Scale your cluster up or down:

```bash
# Scale managed node group
eksctl scale nodegroup \
  --cluster=my-cluster \
  --name=managed-ng-1 \
  --nodes=5 \
  --nodes-min=2 \
  --nodes-max=10
```

## Troubleshooting EKS

### Connectivity Issues

Check cluster connectivity:

```bash
# Verify AWS CLI connectivity
aws eks describe-cluster --name my-cluster --region us-west-2

# Test kubectl connectivity
kubectl get nodes

# Check AWS IAM Authenticator config
aws eks get-token --cluster-name my-cluster --region us-west-2
```

### Node Issues

Diagnose worker node problems:

```bash
# Check node status
kubectl get nodes -o wide

# View node details
kubectl describe node <node-name>

# Check kubelet logs
ssh ec2-user@<node-ip>
sudo journalctl -u kubelet

# Check for pending pods
kubectl get pods --all-namespaces | grep Pending
```

### Control Plane Logs

View EKS control plane logs:

```bash
# Enable control plane logging if not already enabled
aws eks update-cluster-config \
    --name my-cluster \
    --region us-west-2 \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# View logs in CloudWatch
# Navigate to CloudWatch > Log groups > /aws/eks/my-cluster/*
```

## Production Checklist

### High Availability

Ensure your cluster is highly available:

- ✅ Control plane spans multiple AZs (automatic with EKS)
- ✅ Worker nodes distributed across multiple AZs
- ✅ Critical applications use Pod anti-affinity
- ✅ Persistent data properly backed up
- ✅ Use node lifecycle hooks for graceful termination

### Security

Implement security best practices:

- ✅ Private API endpoint enabled
- ✅ IRSA used for all AWS service integrations
- ✅ Pod security standards enforced
- ✅ Network policies or security groups for pods configured
- ✅ Audit logging enabled
- ✅ Secrets encrypted with KMS
- ✅ Node security groups properly configured
- ✅ Regular vulnerability scanning of container images

### Monitoring

Ensure comprehensive monitoring:

- ✅ Container Insights enabled
- ✅ Prometheus for metrics collection
- ✅ Alerting configured for critical issues
- ✅ Log aggregation implemented
- ✅ Distributed tracing set up (X-Ray or ADOT)
- ✅ Dashboards created for key metrics

### Scalability

Prepare for scaling:

- ✅ Cluster Autoscaler or Karpenter configured
- ✅ Horizontal Pod Autoscaler implemented
- ✅ Resource requests and limits defined
- ✅ Pod Disruption Budgets configured for critical services
- ✅ Node and cluster scaling tested

### Disaster Recovery

Plan for disaster recovery:

- ✅ etcd backups enabled (automatic in EKS)
- ✅ Application state backed up
- ✅ Recovery procedures documented and tested
- ✅ Multi-region strategy considered for critical workloads

## Conclusion

Amazon EKS provides a robust, fully-managed Kubernetes platform that lets you focus on your applications rather than cluster management. By leveraging AWS integrations and following best practices, you can build highly available, secure, and cost-effective Kubernetes deployments.

As you progress with EKS, continue to explore advanced features like Fargate, Karpenter, IRSA, and service integrations to optimize your Kubernetes experience on AWS.

## Further Reading

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EKS Workshop](https://www.eksworkshop.com/)
- [AWS Containers Blog](https://aws.amazon.com/blogs/containers/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)