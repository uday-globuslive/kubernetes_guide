# Secret Management in Kubernetes

## Introduction

Secret management is a critical aspect of running secure Elasticsearch, Logstash, and Kibana (ELK) deployments in Kubernetes. This guide explores various approaches to managing secrets in Kubernetes environments, including native Kubernetes Secrets, external secret stores, encryption strategies, and best practices specific to ELK Stack deployments.

## Kubernetes Native Secrets

Kubernetes Secrets are objects that store sensitive information like passwords, OAuth tokens, and SSH keys. By default, Secrets are stored unencrypted in the cluster's etcd database.

### Creating Basic Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elastic-credentials
  namespace: elk-stack
type: Opaque
stringData:
  username: elastic
  password: strong-password-here
```

### Using Secrets in Elasticsearch Deployments

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elk-stack
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
        image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
        env:
        - name: ELASTIC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elastic-credentials
              key: password
        - name: ELASTIC_USERNAME
          valueFrom:
            secretKeyRef:
              name: elastic-credentials
              key: username
```

### Using Secrets in Volume Mounts

For configuration files containing sensitive data:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk-stack
spec:
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
        image: docker.elastic.co/kibana/kibana:8.10.0
        volumeMounts:
        - name: kibana-config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
          readOnly: true
        - name: kibana-certificates
          mountPath: /usr/share/kibana/config/certs
          readOnly: true
      volumes:
      - name: kibana-config
        configMap:
          name: kibana-config
      - name: kibana-certificates
        secret:
          secretName: kibana-certificates
```

## Encrypting Kubernetes Secrets at Rest

By default, Kubernetes Secrets are stored unencrypted in etcd. Here's how to enable encryption at rest:

### Creating an Encryption Configuration

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

### Enabling Encryption in the API Server

```bash
# Update the kube-apiserver configuration
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Add the following args
- --encryption-provider-config=/etc/kubernetes/encryption/encryption-config.yaml
```

### Verifying Secret Encryption

```bash
# Check if a secret is encrypted
ETCDCTL_API=3 etcdctl get /registry/secrets/elk-stack/elastic-credentials --prefix | hexdump -C
```

## External Secret Management Solutions

### HashiCorp Vault

HashiCorp Vault is a widely used external secret management solution.

#### Installing Vault on Kubernetes

```bash
# Add the Hashicorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install Vault
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true"
```

#### Configuring Vault for ELK Secrets

```bash
# Port-forward to Vault
kubectl port-forward vault-0 8200:8200 -n vault

# Create a policy for ELK Secrets
vault policy write elk-policy - <<EOF
path "secret/data/elk/*" {
  capabilities = ["read", "list"]
}
EOF

# Create a Kubernetes authentication method
vault auth enable kubernetes

# Configure Kubernetes authentication
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert="$(cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"

# Create a role for ELK components
vault write auth/kubernetes/role/elk \
  bound_service_account_names=elk-sa \
  bound_service_account_namespaces=elk-stack \
  policies=elk-policy \
  ttl=1h
```

#### Storing ELK Secrets in Vault

```bash
# Store Elasticsearch credentials
vault kv put secret/elk/elasticsearch \
  username=elastic \
  password=password123

# Store Kibana encryption key
vault kv put secret/elk/kibana \
  encryptionKey=$(openssl rand -base64 32)
```

### Using External Secrets Operator

External Secrets Operator is a Kubernetes operator that integrates external secret management systems.

#### Installing External Secrets Operator

```bash
# Add the External Secrets Operator Helm repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Install External Secrets Operator
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

#### Configuring External Secrets for Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: elk-stack
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "elk"
```

#### Creating an External Secret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: elasticsearch-credentials
  namespace: elk-stack
spec:
  refreshInterval: "15m"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: elastic-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: elk/elasticsearch
      property: username
  - secretKey: password
    remoteRef:
      key: elk/elasticsearch
      property: password
```

## Sealed Secrets

Sealed Secrets allow you to encrypt your secrets for safe storage in Git repositories.

### Installing Sealed Secrets Controller

```bash
# Add the Sealed Secrets Helm repository
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

# Install Sealed Secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system
```

### Creating Sealed Secrets for ELK Stack

```bash
# Install kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.22.0/kubeseal-0.22.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.22.0-linux-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/

# Create a regular Secret YAML file
cat > elasticsearch-credentials.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: elastic-credentials
  namespace: elk-stack
type: Opaque
stringData:
  username: elastic
  password: ChangeMeToAStrongPassword
EOF

# Seal the Secret
kubeseal --format yaml < elasticsearch-credentials.yaml > sealed-elasticsearch-credentials.yaml

# Apply the Sealed Secret
kubectl apply -f sealed-elasticsearch-credentials.yaml
```

## Managing TLS Certificates for ELK Stack

### Using cert-manager

cert-manager is a popular Kubernetes add-on for certificate management.

#### Installing cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

#### Creating a Self-Signed Issuer for ELK

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: elk-selfsigned-issuer
  namespace: elk-stack
spec:
  selfSigned: {}
```

#### Creating Certificates for Elasticsearch

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: elasticsearch-certs
  namespace: elk-stack
spec:
  secretName: elasticsearch-certs
  duration: 8760h  # 1 year
  renewBefore: 720h  # 30 days
  subject:
    organizations:
      - Elastic
  commonName: elasticsearch.elk-stack.svc.cluster.local
  dnsNames:
  - elasticsearch
  - elasticsearch.elk-stack
  - elasticsearch.elk-stack.svc
  - elasticsearch.elk-stack.svc.cluster.local
  - "*.elasticsearch.elk-stack.svc.cluster.local"
  issuerRef:
    name: elk-selfsigned-issuer
    kind: Issuer
```

### Creating CA and Certificates Manually

For cases where cert-manager isn't available:

```bash
# Generate CA key and certificate
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 -out ca.crt -subj "/CN=Elastic CA"

# Generate Elasticsearch certificate
openssl genrsa -out elasticsearch.key 2048
openssl req -new -key elasticsearch.key -out elasticsearch.csr -subj "/CN=elasticsearch.elk-stack.svc.cluster.local"
openssl x509 -req -in elasticsearch.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out elasticsearch.crt -days 365 -sha256

# Create Kubernetes secrets
kubectl create secret generic elasticsearch-certs \
  --from-file=ca.crt \
  --from-file=elasticsearch.key \
  --from-file=elasticsearch.crt \
  -n elk-stack
```

## Secret Management in Elastic Cloud on Kubernetes (ECK)

ECK manages many secrets automatically:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elk-stack
spec:
  version: 8.10.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
  http:
    tls:
      selfSignedCertificate:
        disabled: false  # ECK will generate TLS certificates
```

### Adding Custom Secrets to ECK

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elk-stack
spec:
  version: 8.10.0
  nodeSets:
  - name: default
    count: 3
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: CUSTOM_ENV_VAR
            valueFrom:
              secretKeyRef:
                name: custom-secret
                key: value
```

## Secret Rotation in ELK Stack

### Rotating Elasticsearch User Passwords

```bash
# Create a new secret with updated password
kubectl create secret generic elastic-credentials-new \
  --from-literal=username=elastic \
  --from-literal=password=new-strong-password \
  -n elk-stack

# Update the StatefulSet to use the new secret
kubectl patch statefulset elasticsearch \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/valueFrom/secretKeyRef/name", "value": "elastic-credentials-new"}]' \
  -n elk-stack

# Restart pods one by one
kubectl rollout restart statefulset/elasticsearch -n elk-stack
```

### Automating Certificate Rotation with cert-manager

cert-manager automatically handles certificate rotation based on the `renewBefore` setting in the Certificate resource.

## ELK Stack Secret Management Best Practices

1. **Use Strong Access Controls**:
   - Apply RBAC to limit which users and pods can access secrets
   - Use namespaces to isolate secrets by application

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: secret-manager
     namespace: elk-stack
   rules:
   - apiGroups: [""]
     resources: ["secrets"]
     verbs: ["get", "list", "watch"]
     resourceNames: ["elastic-credentials"]
   ```

2. **Encrypt Elasticsearch Communications**:
   - Enable TLS for all Elasticsearch traffic
   - Use client certificates for service-to-service communication

3. **Minimize Secret Exposure**:
   - Mount secrets as files, not environment variables when possible
   - Use subPath volume mounts to mount individual files not entire secrets

4. **Audit Secret Access**:
   - Enable Kubernetes audit logs for secret access
   - Monitor for suspicious access patterns

   ```yaml
   apiVersion: audit.k8s.io/v1
   kind: Policy
   rules:
   - level: Metadata
     resources:
     - group: ""
       resources: ["secrets"]
   ```

5. **Implement Secret Rotation**:
   - Rotate credentials regularly (every 30-90 days)
   - Automate rotation where possible

6. **Consider Ephemeral Volumes for Secrets**:
   - Use memory-backed volumes for highly sensitive secrets
   
   ```yaml
   volumes:
   - name: elastic-credentials
     secret:
       secretName: elastic-credentials
     defaultMode: 0400
   ```

## Tools for Secret Management

### Helm Secrets

Helm Secrets is a plugin that helps manage secrets for Helm charts.

```bash
# Install Helm Secrets plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt secrets
helm secrets enc elk/secrets.yaml

# Deploy with encrypted secrets
helm secrets install elk-stack ./elk-chart -f elk/secrets.yaml
```

### Sops (Secrets OPerationS)

Mozilla SOPS is a tool for encrypting and decrypting configuration files.

```bash
# Install SOPS
curl -LO https://github.com/mozilla/sops/releases/download/v3.7.3/sops-v3.7.3.linux
chmod +x sops-v3.7.3.linux
sudo mv sops-v3.7.3.linux /usr/local/bin/sops

# Encrypt a secrets file
sops --encrypt --age age1ut9drk02l5sa8arcyrrltzkw4xrh6nkk9s8gef46pm28vqlj9f0srg5cl3 elk-secrets.yaml > elk-secrets.enc.yaml

# Decrypt the file
sops --decrypt elk-secrets.enc.yaml
```

## Monitoring Secret Access

### Falco for Runtime Security

Falco can monitor and alert on suspicious access to secrets:

```yaml
# falco-rules.yaml
- rule: Secret_file_read
  desc: Detect attempts to read Kubernetes secret files 
  condition: >
    open_read and
    container and
    openat_file matches "secrets/elk-stack/*" and
    not proc.name in (elasticsearch, logstash, kibana, filebeat)
  output: >
    Secret file read detected (user=%user.name file=%fd.name command=%proc.cmdline)
  priority: WARNING
```

### OPA Gatekeeper Policies

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8snoexternalserviceaccounts
spec:
  crd:
    spec:
      names:
        kind: K8sNoExternalServiceAccounts
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoexternalserviceaccounts
        violation[{"msg": msg}] {
          input.review.object.kind == "Pod"
          input.review.object.spec.automountServiceAccountToken == true
          not input_allowed_namespace
          msg := "Service accounts should not be auto-mounted in this namespace"
        }
        
        input_allowed_namespace {
          input.review.object.metadata.namespace == "kube-system"
        }
```

## Conclusion

Effective secret management is crucial for securing ELK Stack deployments in Kubernetes. By utilizing Kubernetes native features like Secrets and encryption at rest, along with external tools like HashiCorp Vault or Sealed Secrets, you can significantly enhance the security posture of your ELK Stack infrastructure. Remember to implement best practices like secret rotation, strong access controls, and monitoring to maintain a robust security posture throughout the lifecycle of your applications.

## References

- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Elastic Cloud on Kubernetes Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)
- [Falco Security](https://falco.org/docs/)