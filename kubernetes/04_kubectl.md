# Kubernetes API and kubectl

## Introduction

The Kubernetes API is the foundation of the Kubernetes control plane, enabling administrators and applications to interact with the cluster. kubectl is the command-line tool that communicates with the Kubernetes API, allowing users to manage Kubernetes resources. This guide covers the Kubernetes API architecture, kubectl usage, and practical techniques for managing Elastic Stack deployments.

## Kubernetes API Architecture

The Kubernetes API server is the central component of Kubernetes that exposes the RESTful API. It processes and validates API requests and updates the corresponding objects in etcd.

### API Structure

The Kubernetes API is organized into groups, versions, and resources:

```
/api/v1/namespaces/{namespace}/{resource}
/apis/{group}/{version}/namespaces/{namespace}/{resource}
```

Examples:
- `/api/v1/namespaces/default/pods` - Core API, lists pods in the default namespace
- `/apis/apps/v1/namespaces/elastic/deployments` - Apps API group, lists deployments in the elastic namespace

### API Access and Authentication

Access to the Kubernetes API can be controlled via:

1. **Authentication**: Identifies the user making the request
   - Client certificates
   - Bearer tokens
   - OpenID Connect tokens
   - Webhook token authentication

2. **Authorization**: Determines what actions a user can perform
   - RBAC (Role-Based Access Control)
   - ABAC (Attribute-Based Access Control)
   - Node authorization
   - Webhook authorization

3. **Admission Control**: Validates and modifies requests
   - Validating admission webhooks
   - Mutating admission webhooks

## kubectl Overview

kubectl is the official command-line interface for Kubernetes. It allows you to:

- Create, update, delete, and inspect resources
- Debug applications
- View logs
- Execute commands in containers
- Forward ports
- Scale applications

### Installation

Install kubectl on various platforms:

```bash
# macOS with Homebrew
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows with Chocolatey
choco install kubernetes-cli
```

### Configuration

kubectl uses a configuration file (`kubeconfig`) to store information about clusters, users, and authentication:

```bash
# Default location
~/.kube/config

# Specify a custom config file
kubectl --kubeconfig=/path/to/config get pods

# View current configuration
kubectl config view

# Set a different context
kubectl config use-context my-cluster
```

Creating a kubeconfig file manually:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: elastic-cluster
  cluster:
    server: https://kubernetes.example.com:6443
    certificate-authority-data: <base64-encoded-ca-cert>
users:
- name: elastic-admin
  user:
    client-certificate-data: <base64-encoded-client-cert>
    client-key-data: <base64-encoded-client-key>
contexts:
- name: elastic-context
  context:
    cluster: elastic-cluster
    user: elastic-admin
    namespace: elastic
current-context: elastic-context
```

## Basic kubectl Commands

### Getting Information

```bash
# List all pods in the current namespace
kubectl get pods

# List all pods in all namespaces
kubectl get pods --all-namespaces

# Get detailed information about a specific pod
kubectl describe pod <pod-name>

# Get pods with more information
kubectl get pods -o wide

# Output in YAML format
kubectl get pod <pod-name> -o yaml

# Output in JSON format
kubectl get pod <pod-name> -o json

# List resources with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Sort resources by field
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### Creating and Updating Resources

```bash
# Create resources from a file
kubectl apply -f elasticsearch.yaml

# Create resources from multiple files
kubectl apply -f elasticsearch.yaml -f kibana.yaml

# Create resources from a directory
kubectl apply -f manifests/

# Create resources from a URL
kubectl apply -f https://raw.githubusercontent.com/elastic/cloud-on-k8s/master/config/samples/elasticsearch/elasticsearch.yaml

# Edit a resource
kubectl edit deployment kibana
```

### Deleting Resources

```bash
# Delete a pod
kubectl delete pod <pod-name>

# Delete resources using a file
kubectl delete -f elasticsearch.yaml

# Delete all pods in a namespace
kubectl delete pods --all -n elastic

# Delete resources with a label selector
kubectl delete pods -l app=elasticsearch

# Force deletion (use with caution)
kubectl delete pod <pod-name> --grace-period=0 --force
```

### Working with Namespaces

```bash
# Create a namespace
kubectl create namespace elastic

# List all namespaces
kubectl get namespaces

# Set default namespace for kubectl
kubectl config set-context --current --namespace=elastic

# View resources across all namespaces
kubectl get pods --all-namespaces
```

## Advanced kubectl Techniques

### Filtering and Selecting Resources

```bash
# Get pods by label selector
kubectl get pods -l app=elasticsearch

# Get pods with complex label selectors
kubectl get pods -l 'app in (elasticsearch,kibana)',tier=production

# Get pods by field selector
kubectl get pods --field-selector status.phase=Running
```

### Managing Deployments

```bash
# Scale a deployment
kubectl scale deployment elasticsearch --replicas=3

# Set image for a deployment
kubectl set image deployment/kibana kibana=docker.elastic.co/kibana/kibana:8.8.2

# Rollout status
kubectl rollout status deployment/elasticsearch

# Pause a rollout
kubectl rollout pause deployment/elasticsearch

# Resume a rollout
kubectl rollout resume deployment/elasticsearch

# Rollback to previous version
kubectl rollout undo deployment/elasticsearch

# Rollback to a specific revision
kubectl rollout undo deployment/elasticsearch --to-revision=2

# View rollout history
kubectl rollout history deployment/elasticsearch
```

### Debugging with kubectl

```bash
# Get logs from a pod
kubectl logs elasticsearch-0

# Get logs from a specific container in a pod
kubectl logs elasticsearch-0 -c init-sysctl

# Stream logs
kubectl logs -f elasticsearch-0

# Get logs for all containers in a pod
kubectl logs elasticsearch-0 --all-containers

# Get previous logs (if container has restarted)
kubectl logs elasticsearch-0 --previous

# Execute commands in containers
kubectl exec -it elasticsearch-0 -- bash

# Run a one-time command
kubectl exec elasticsearch-0 -- cat /usr/share/elasticsearch/config/elasticsearch.yml

# Get a shell into a running container
kubectl exec -it elasticsearch-0 -- /bin/bash
```

### Port Forwarding

```bash
# Forward local port 9200 to pod port 9200
kubectl port-forward elasticsearch-0 9200:9200

# Forward to a service
kubectl port-forward svc/elasticsearch 9200:9200

# Forward to a deployment
kubectl port-forward deployment/kibana 5601:5601

# Forward to multiple ports
kubectl port-forward elasticsearch-0 9200:9200 9300:9300

# Forward with address binding
kubectl port-forward --address 0.0.0.0 elasticsearch-0 9200:9200
```

### Working with Configuration

```bash
# Create a ConfigMap
kubectl create configmap elasticsearch-config --from-file=elasticsearch.yml

# Create a Secret
kubectl create secret generic elastic-credentials \
  --from-literal=username=elastic \
  --from-literal=password=changeme

# View ConfigMap
kubectl get configmap elasticsearch-config -o yaml

# View Secret (base64 encoded)
kubectl get secret elastic-credentials -o yaml

# Decode a Secret value
kubectl get secret elastic-credentials -o jsonpath='{.data.password}' | base64 --decode
```

## kubectl for Elastic Stack Management

### Managing Elasticsearch Clusters

```bash
# Deploy Elasticsearch with a YAML manifest
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elastic
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
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: elasticsearch-storage
      resources:
        requests:
          storage: 10Gi
EOF

# Create a service for Elasticsearch
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    name: http
  - port: 9300
    name: transport
  clusterIP: None
EOF

# Check Elasticsearch pods
kubectl get pods -l app=elasticsearch -n elastic

# Get Elasticsearch cluster health
kubectl exec -it elasticsearch-0 -n elastic -- curl -XGET "localhost:9200/_cluster/health?pretty"
```

### Deploying Kibana

```bash
# Create Kibana deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elastic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.8.1
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
        ports:
        - containerPort: 5601
          name: http
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 60
          timeoutSeconds: 5
EOF

# Create Kibana service
cat <<EOF | kubectl apply -f -
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
  type: ClusterIP
EOF

# Create an Ingress for Kibana
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: elastic
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - kibana.example.com
    secretName: kibana-tls
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
EOF

# Check Kibana deployment
kubectl get deployment,pods,svc,ingress -l app=kibana -n elastic
```

### Deploying Logstash

```bash
# Create ConfigMap for Logstash pipeline
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipelines
  namespace: elastic
data:
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }
    
    filter {
      if [kubernetes] {
        mutate {
          add_field => { "[@metadata][index]" => "k8s-logs-%{+YYYY.MM.dd}" }
        }
      } else {
        mutate {
          add_field => { "[@metadata][index]" => "logs-%{+YYYY.MM.dd}" }
        }
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch:9200"]
        index => "%{[@metadata][index]}"
      }
    }
EOF

# Deploy Logstash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elastic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.8.1
        ports:
        - containerPort: 5044
          name: beats
        - containerPort: 9600
          name: monitoring
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/logstash/pipeline/logstash.conf
          subPath: logstash.conf
        env:
        - name: LS_JAVA_OPTS
          value: "-Xmx512m -Xms512m"
      volumes:
      - name: config-volume
        configMap:
          name: logstash-pipelines
          items:
            - key: logstash.conf
              path: logstash.conf
EOF

# Create Logstash service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elastic
spec:
  selector:
    app: logstash
  ports:
  - port: 5044
    name: beats
    targetPort: 5044
  type: ClusterIP
EOF

# Check Logstash deployment
kubectl get deployment,pods,svc -l app=logstash -n elastic
```

### Deploying Filebeat

```bash
# Create ConfigMap for Filebeat configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: elastic
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: \${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
    
    output.logstash:
      hosts: ["logstash:5044"]
EOF

# Deploy Filebeat as a DaemonSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elastic
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.1
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
EOF

# Create a service account for Filebeat
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elastic
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: elastic
EOF

# Check Filebeat deployment
kubectl get ds,pods -l app=filebeat -n elastic
```

## Working with Elastic Cloud on Kubernetes (ECK)

ECK is the official operator for managing Elastic Stack resources on Kubernetes.

### Installing ECK

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.8.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/operator.yaml

# Verify installation
kubectl get pods -n elastic-system
```

### Deploying Elasticsearch with ECK

```bash
# Create an Elasticsearch cluster
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  version: 8.8.1
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: elasticsearch-storage
  - name: data
    count: 3
    config:
      node.roles: ["data", "ingest"]
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
        storageClassName: elasticsearch-storage
EOF

# Check Elasticsearch resources
kubectl get elasticsearch -n elastic
kubectl get pods -l elasticsearch.k8s.elastic.co/cluster-name=elasticsearch -n elastic

# Get Elasticsearch password
kubectl get secret elasticsearch-es-elastic-user -n elastic -o jsonpath='{.data.elastic}' | base64 --decode
```

### Deploying Kibana with ECK

```bash
# Create a Kibana instance
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic
spec:
  version: 8.8.1
  count: 1
  elasticsearchRef:
    name: elasticsearch
  http:
    tls:
      selfSignedCertificate:
        disabled: true
EOF

# Check Kibana resources
kubectl get kibana -n elastic
kubectl get pods -l kibana.k8s.elastic.co/name=kibana -n elastic

# Create an Ingress for Kibana
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: elastic
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - kibana.example.com
    secretName: kibana-tls
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana-kb-http
            port:
              number: 5601
EOF
```

### Deploying APM Server with ECK

```bash
# Create an APM Server
cat <<EOF | kubectl apply -f -
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-server
  namespace: elastic
spec:
  version: 8.8.1
  count: 1
  elasticsearchRef:
    name: elasticsearch
  kibanaRef:
    name: kibana
EOF

# Check APM Server resources
kubectl get apmserver -n elastic
kubectl get pods -l apm.k8s.elastic.co/name=apm-server -n elastic
```

## kubectl Productivity Tips

### Aliases and Shortcuts

Create useful kubectl aliases to save time:

```bash
# Add to your ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kns='kubectl config set-context --current --namespace'
alias ke='kubectl get events --sort-by=.metadata.creationTimestamp'
alias kp='kubectl get pods'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kx='kubectl exec -it'
alias ka='kubectl apply -f'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgss='kubectl get statefulsets'
```

### Auto-completion

Enable kubectl command completion:

```bash
# Bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
# If you use an alias for kubectl
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >>~/.zshrc
# If you use an alias for kubectl
echo 'alias k=kubectl' >>~/.zshrc
echo 'complete -F __start_kubectl k' >>~/.zshrc
```

### Using kubectl Plugins with krew

Krew is a plugin manager for kubectl, providing additional functionality:

```bash
# Install krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Add krew to your PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Install useful plugins
kubectl krew install ctx        # Switch contexts
kubectl krew install ns         # Switch namespaces
kubectl krew install neat       # Clean up YAML/JSON output
kubectl krew install resource-capacity  # Show resource usage
kubectl krew install view-secret  # Decode secrets
kubectl krew install access-matrix  # Show RBAC access matrix
kubectl krew install tree       # Show hierarchical object relationships
```

### Context Switching

Tools for managing multiple clusters:

```bash
# Install kubectx and kubens
# macOS with Homebrew
brew install kubectx

# Using krew
kubectl krew install ctx
kubectl krew install ns

# Using kubectx to switch contexts
kubectx my-cluster

# Using kubens to switch namespaces
kubens elastic
```

### Custom Output Formats

Customize kubectl output for better readability:

```bash
# Output as a table with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Output just the names of pods
kubectl get pods -o name

# Get the IP of a pod
kubectl get pod elasticsearch-0 -o jsonpath='{.status.podIP}'

# Get all container images in use
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | tr -s '[[:space:]]' '\n' | sort | uniq

# Format with jsonpath and custom templates
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Multi-cluster Management

Techniques for managing multiple Kubernetes clusters:

```bash
# Merge kubeconfig files
KUBECONFIG=~/.kube/config:~/new-cluster-config kubectl config view --flatten > ~/.kube/merged_config
export KUBECONFIG=~/.kube/merged_config

# List all contexts
kubectl config get-contexts

# Switch between clusters
kubectl config use-context cluster-1

# Execute commands against multiple clusters
for ctx in $(kubectl config get-contexts -o name); do
  kubectl --context=$ctx get nodes
done
```

## Troubleshooting with kubectl

### Diagnosing Pod Issues

```bash
# Check pod status
kubectl get pod elasticsearch-0 -n elastic

# Describe pod for events and conditions
kubectl describe pod elasticsearch-0 -n elastic

# Check logs
kubectl logs elasticsearch-0 -n elastic

# Check previous container logs if restarting
kubectl logs elasticsearch-0 -n elastic --previous

# Check init container logs
kubectl logs elasticsearch-0 -n elastic -c init-sysctl

# Get pod details
kubectl get pod elasticsearch-0 -n elastic -o yaml
```

### Debugging Pod Networking

```bash
# Deploy a debug pod
kubectl run debug --image=nicolaka/netshoot -it --rm -- bash

# Test DNS resolution
kubectl exec -it debug -- nslookup elasticsearch.elastic.svc.cluster.local

# Test connectivity to a service
kubectl exec -it debug -- curl -v http://elasticsearch:9200
```

### Analyzing Resource Usage

```bash
# Get node resource usage
kubectl top nodes

# Get pod resource usage
kubectl top pods -n elastic

# Get container resource usage
kubectl top pods -n elastic --containers
```

### Common kubectl Troubleshooting Commands

```bash
# Check API server connectivity
kubectl cluster-info

# Validate YAML syntax
kubectl apply --validate -f elasticsearch.yaml --dry-run=client

# Check resource metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# Check statefulset status
kubectl get statefulsets elasticsearch -n elastic
kubectl describe statefulset elasticsearch -n elastic

# Check persistent volume claims
kubectl get pvc -n elastic
kubectl describe pvc elasticsearch-data-elasticsearch-0 -n elastic

# Check services
kubectl get svc -n elastic
kubectl describe svc elasticsearch -n elastic

# Generate a support bundle (for ECK)
kubectl eck-diagnostics -n elastic
```

## Conclusion

The Kubernetes API and kubectl are powerful tools for managing Elastic Stack deployments in Kubernetes environments. By mastering kubectl commands and workflows, you can efficiently deploy, manage, and troubleshoot Elasticsearch, Kibana, Logstash, and Beats.

In the next section, we'll explore Elastic Cloud on Kubernetes (ECK) in more detail, focusing on advanced features and customization options.