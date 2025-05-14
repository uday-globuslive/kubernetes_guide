# Authentication and Authorization in Kubernetes

Securing access to your Kubernetes cluster is fundamental to protecting your containerized applications and data. This chapter explores the authentication and authorization mechanisms in Kubernetes, providing a comprehensive guide to implementing robust security controls.

## Table of Contents

- [Kubernetes Security Model](#kubernetes-security-model)
- [Authentication Methods](#authentication-methods)
- [Authorization Mechanisms](#authorization-mechanisms)
- [Service Accounts](#service-accounts)
- [Kubernetes API Access Security](#kubernetes-api-access-security)
- [Authentication and Authorization for Cloud-Managed Kubernetes](#authentication-and-authorization-for-cloud-managed-kubernetes)
- [Security Best Practices](#security-best-practices)
- [Practical Implementation Examples](#practical-implementation-examples)
- [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)
- [Advanced Configuration](#advanced-configuration)

## Kubernetes Security Model

Kubernetes implements a defense-in-depth approach to security:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Security Layers                       │
│                                                                     │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐     │
│  │               │     │               │     │               │     │
│  │Authentication │────▶│ Authorization │────▶│ Admission     │     │
│  │               │     │               │     │ Control       │     │
│  └───────────────┘     └───────────────┘     └───────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The security flow consists of:

1. **Authentication**: Verifies the identity of users or systems
2. **Authorization**: Determines what authenticated users can do
3. **Admission Control**: Validates and modifies requests

This chapter focuses on the first two aspects, while admission controllers will be covered separately.

## Authentication Methods

Kubernetes supports multiple authentication strategies, which can be used simultaneously:

### X.509 Client Certificates

Kubernetes can authenticate users with TLS client certificates:

```bash
# Generate a private key
openssl genrsa -out john.key 2048

# Create a certificate signing request (CSR)
openssl req -new -key john.key -out john.csr -subj "/CN=john/O=devops"

# Sign the CSR with the Kubernetes CA
openssl x509 -req -in john.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out john.crt -days 365
```

Configure kubectl to use the certificate:

```bash
kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key

kubectl config set-context john-context \
  --cluster=kubernetes \
  --user=john
```

Pros:
- Widely supported by all Kubernetes distributions
- Doesn't require additional infrastructure
- Strong cryptographic identity

Cons:
- Certificate management overhead
- Limited expiration and revocation options
- No easy integration with corporate identity systems

### Static Token File

Kubernetes API server can be configured to use a token file:

```
# /etc/kubernetes/auth/tokens.csv
token123,user1,1000,"devops,admin"
token456,user2,2000,"developers"
```

Configure API server with `--token-auth-file=/etc/kubernetes/auth/tokens.csv`

Use the token:

```bash
kubectl config set-credentials user1 --token=token123
```

Pros:
- Simple to set up
- Easy to understand

Cons:
- Plaintext tokens
- Requires API server restart to update
- No token expiration
- Not suitable for production

### OpenID Connect (OIDC)

Integrate with identity providers like Google, Azure AD, Okta, etc.:

API server configuration:

```bash
kube-apiserver \
  --oidc-issuer-url=https://accounts.google.com \
  --oidc-client-id=kubernetes \
  --oidc-username-claim=email \
  --oidc-groups-claim=groups
```

Configure kubectl with OIDC:

```bash
kubectl config set-credentials oidc-user \
  --auth-provider=oidc \
  --auth-provider-arg=idp-issuer-url=https://accounts.google.com \
  --auth-provider-arg=client-id=kubernetes \
  --auth-provider-arg=client-secret=secret \
  --auth-provider-arg=refresh-token=refresh_token \
  --auth-provider-arg=id-token=id_token
```

Pros:
- Integration with enterprise identity providers
- Token expiration and revocation
- Centralized user management
- Multi-factor authentication support

Cons:
- Complex configuration
- Requires external identity provider
- Token size limitations

### Webhook Token Authentication

Delegate authentication to an external service:

API server configuration:

```bash
kube-apiserver \
  --authentication-token-webhook-config-file=/etc/kubernetes/webhook-config.yaml
```

Webhook configuration:

```yaml
# webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
- name: auth-service
  cluster:
    server: https://auth.example.com/authenticate
users:
- name: apiserver
  user:
    client-certificate: /etc/kubernetes/pki/webhook-client.crt
    client-key: /etc/kubernetes/pki/webhook-client.key
current-context: auth-webhook
contexts:
- context:
    cluster: auth-service
    user: apiserver
  name: auth-webhook
```

Pros:
- Fully customizable authentication logic
- Integration with any identity system
- Dynamic authentication policies

Cons:
- Requires developing and maintaining a service
- Performance overhead
- Single point of failure if webhook service is down

### Authentication Proxy

Configure the API server behind an authenticating proxy:

```bash
kube-apiserver \
  --requestheader-client-ca-file=/etc/kubernetes/pki/proxy-ca.crt \
  --requestheader-allowed-names=front-proxy-client \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User
```

Pros:
- Leverage existing enterprise proxies
- Offload authentication complexity
- Support for additional features like single sign-on

Cons:
- Additional infrastructure component
- More complex architecture
- Secure header handling required

### Bootstrap Tokens

Used primarily for cluster bootstrap and joining nodes:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcdef
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: abcdef
  token-secret: 1234567890abcdef
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: system:bootstrappers:worker
```

Pros:
- Purpose-built for node bootstrap
- Time-limited tokens
- Minimal privilege by default

Cons:
- Limited use cases
- Not suitable for regular user authentication

## Authorization Mechanisms

Once a user is authenticated, Kubernetes determines what actions they can perform using several authorization modes:

### RBAC (Role-Based Access Control)

RBAC is the recommended authorization mode in Kubernetes:

```yaml
# Role for read-only access to pods in a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Bind the role to users:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

For cluster-wide permissions, use ClusterRole and ClusterRoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

Pros:
- Fine-grained control
- Namespace scoping
- Separation of concerns
- Native Kubernetes primitive

Cons:
- Complex for large organizations
- No deny rules (only allow)
- Learning curve for administrators

### ABAC (Attribute-Based Access Control)

ABAC uses policies based on attributes:

Policy file example:
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "*", "resource": "pods", "apiGroup": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "bob", "namespace": "projectA", "resource": "*", "apiGroup": "*"}}
```

API server configuration:
```bash
kube-apiserver \
  --authorization-mode=ABAC \
  --authorization-policy-file=/etc/kubernetes/abac-policy.json
```

Pros:
- Flexible policies based on any attribute
- Simple policy language

Cons:
- Static file requires API server restart
- No native tooling for management
- Less standardized than RBAC
- Legacy approach in Kubernetes

### Webhook

Delegate authorization decisions to an external service:

API server configuration:
```bash
kube-apiserver \
  --authorization-mode=Webhook \
  --authorization-webhook-config-file=/etc/kubernetes/webhook-config.yaml
```

Webhook config:
```yaml
apiVersion: v1
kind: Config
clusters:
- name: authorization-webhook
  cluster:
    server: https://auth.example.com/authorize
users:
- name: apiserver
  user:
    client-certificate: /etc/kubernetes/pki/apiserver-client.crt
    client-key: /etc/kubernetes/pki/apiserver-client.key
current-context: webhook
contexts:
- context:
    cluster: authorization-webhook
    user: apiserver
  name: webhook
```

Pros:
- Fully customizable authorization logic
- Integration with external policy engines
- Dynamic policy updates without API server restart

Cons:
- External dependency
- Latency concerns
- Development and maintenance overhead

### Node Authorization

Special-purpose authorizer for kubelet access:

```bash
kube-apiserver \
  --authorization-mode=Node,RBAC
```

Pros:
- Automatic secure kubelet authorization
- Prevents privilege escalation via node credentials

Cons:
- Limited to node authorization use case

### AlwaysAllow/AlwaysDeny

Simple authorizers for testing:

```bash
# Allow everything (insecure)
kube-apiserver --authorization-mode=AlwaysAllow

# Deny everything (debugging)
kube-apiserver --authorization-mode=AlwaysDeny
```

Pros:
- Simplicity for testing
- No configuration required

Cons:
- Not suitable for production (especially AlwaysAllow)
- No granular control

### Multiple Authorization Modes

Kubernetes can be configured with multiple authorization modes:

```bash
kube-apiserver \
  --authorization-mode=Node,RBAC,Webhook
```

The request is checked by each configured authorizer in order. If any authorizer approves or denies the request, that decision is returned immediately.

## Service Accounts

Service accounts provide identities for pods running in your cluster:

### Service Account Basics

Create a service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
```

Reference it in a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: my-app:1.0.0
```

### Default Service Account

Each namespace has a default service account:

```bash
kubectl get serviceaccounts default -n default
```

Pods use this service account if none is specified:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-pod
spec:
  containers:
  - name: app
    image: my-app:1.0.0
  # serviceAccountName is omitted, so 'default' is used
```

### Token Management

Service account tokens are automatically mounted:

```bash
# Examine the mounted token
kubectl exec -it my-app-pod -- ls -l /var/run/secrets/kubernetes.io/serviceaccount/
```

In Kubernetes 1.24+, service account tokens are short-lived and automatically rotated. For long-lived tokens:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-token
  annotations:
    kubernetes.io/service-account.name: my-app
type: kubernetes.io/service-account-token
```

### Assigning Permissions to Service Accounts

Use RBAC to grant permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: app-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

### Accessing the API from Pods

Using service account tokens to access the API:

```python
import requests
import os

# Service account token is automatically mounted
with open('/var/run/secrets/kubernetes.io/serviceaccount/token', 'r') as file:
    token = file.read()

# API server CA certificate
ca_cert = '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'

# Kubernetes API server endpoint
api_server = 'https://kubernetes.default.svc'

# Get pods in current namespace
namespace = open('/var/run/secrets/kubernetes.io/serviceaccount/namespace').read()
url = f'{api_server}/api/v1/namespaces/{namespace}/pods'

headers = {'Authorization': f'Bearer {token}'}
response = requests.get(url, headers=headers, verify=ca_cert)
pods = response.json()
```

### Service Account Token Volume Projection

For more control over token properties:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-projection-pod
spec:
  containers:
  - name: app
    image: my-app:1.0.0
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
      readOnly: true
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: my-app
```

## Kubernetes API Access Security

### Securing the API Server

API server security best practices:

1. **Restrict Network Access**:
   ```bash
   kube-apiserver \
     --bind-address=secure.internal.ip \
     --advertise-address=secure.internal.ip
   ```

2. **TLS Encryption**:
   ```bash
   kube-apiserver \
     --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
     --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
     --client-ca-file=/etc/kubernetes/pki/ca.crt
   ```

3. **Disable Anonymous Access**:
   ```bash
   kube-apiserver \
     --anonymous-auth=false
   ```

4. **Enable Auditing**:
   ```bash
   kube-apiserver \
     --audit-log-path=/var/log/kubernetes/audit.log \
     --audit-log-maxage=30 \
     --audit-log-maxbackup=10 \
     --audit-log-maxsize=100 \
     --audit-policy-file=/etc/kubernetes/audit-policy.yaml
   ```

### Kubectl Authentication

Configure kubectl for different authentication methods:

1. **Certificate Authentication**:
   ```bash
   kubectl config set-credentials user \
     --client-certificate=user.crt \
     --client-key=user.key
   ```

2. **Token Authentication**:
   ```bash
   kubectl config set-credentials user \
     --token=bearer_token
   ```

3. **Basic Authentication**:
   ```bash
   kubectl config set-credentials user \
     --username=user \
     --password=password
   ```

4. **OIDC Authentication**:
   ```bash
   kubectl config set-credentials oidc-user \
     --auth-provider=oidc \
     --auth-provider-arg=idp-issuer-url=https://oidc.example.com \
     --auth-provider-arg=client-id=kubernetes \
     --auth-provider-arg=refresh-token=refresh_token \
     --auth-provider-arg=id-token=id_token
   ```

Configure kubectl context:

```bash
kubectl config set-context my-context \
  --cluster=my-cluster \
  --user=my-user \
  --namespace=my-namespace
```

Activate the context:

```bash
kubectl config use-context my-context
```

### API Request Authentication Flow

1. Client makes API request with credentials
2. API server authenticates the request using configured authenticators
3. User identity is extracted (username, UID, groups)
4. API server authorizes the request using configured authorizers
5. Admission controllers validate and potentially modify the request
6. Request is processed if it passes all checks

## Authentication and Authorization for Cloud-Managed Kubernetes

### AWS EKS

AWS EKS uses IAM for authentication:

```bash
# Update kubeconfig for EKS
aws eks update-kubeconfig --name my-cluster --region us-west-2
```

Map IAM users/roles to Kubernetes RBAC:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/EksAdminRole
      username: admin
      groups:
        - system:masters
    - rolearn: arn:aws:iam::123456789012:role/DevTeamRole
      username: developer
      groups:
        - developers
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/alice
      username: alice
      groups:
        - system:masters
```

### Google GKE

GKE integrates with Google IAM:

```bash
# Update kubeconfig for GKE
gcloud container clusters get-credentials my-cluster --zone us-central1-a
```

IAM roles map to predefined Kubernetes roles:
- Basic: provides read-only access
- Developer: provides edit access
- Admin: provides admin access

Custom IAM role bindings:

```bash
# Grant admin role to a user
gcloud projects add-iam-policy-binding project-id \
  --member=user:alice@example.com \
  --role=container.admin
```

### Azure AKS

AKS integrates with Azure AD:

```bash
# Update kubeconfig for AKS
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

Configure Azure AD integration during cluster creation:

```bash
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-aad \
  --aad-admin-group-object-ids <AAD-GROUP-ID>
```

Create RBAC bindings for Azure AD users:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: contoso-admin
subjects:
- kind: Group
  name: "00000000-0000-0000-0000-000000000000" # AAD Group Object ID
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## Security Best Practices

### Least Privilege Principle

Grant only the permissions needed:

```yaml
# Instead of using cluster-admin:
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: accounting
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Regular Credential Rotation

Rotate credentials regularly:

```bash
# Generate new certificate for user
openssl genrsa -out john-new.key 2048
openssl req -new -key john-new.key -out john-new.csr -subj "/CN=john/O=devops"
openssl x509 -req -in john-new.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out john-new.crt -days 365

# Update kubeconfig with new credentials
kubectl config set-credentials john \
  --client-certificate=john-new.crt \
  --client-key=john-new.key
```

### Audit Existing RBAC Permissions

Regularly review RBAC permissions:

```bash
# List all clusterrolebindings
kubectl get clusterrolebindings

# Check who has cluster-admin
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# View effective permissions for a user
kubectl auth can-i --list --as=jane
```

### Namespace Isolation

Use namespaces for security isolation:

```yaml
# Create namespaced role and binding
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-access
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-access
  namespace: team-a
subjects:
- kind: Group
  name: team-a
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-access
  apiGroup: rbac.authorization.k8s.io
```

### Restrict Pod Access to Service Accounts

Limit service account token access:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-app:1.0.0
```

### Regularly Review Authentication Configurations

Audit API server authentication configuration:

```bash
# Check kube-apiserver configuration
kubectl get pods -n kube-system kube-apiserver-control-plane-node -o yaml | grep "\--authorization-mode"
```

## Practical Implementation Examples

### Setting up RBAC for a Development Team

Create role for developers:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

Create role binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: development-developers
  namespace: development
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Implementing OIDC Authentication with Keycloak

Configure Keycloak:
1. Create a new realm (e.g., "kubernetes")
2. Create a client with:
   - Client ID: kubernetes
   - Access Type: confidential
   - Valid Redirect URIs: http://localhost:8000
3. Create groups and assign users

API server configuration:

```bash
kube-apiserver \
  --oidc-issuer-url=https://keycloak.example.com/auth/realms/kubernetes \
  --oidc-client-id=kubernetes \
  --oidc-username-claim=preferred_username \
  --oidc-groups-claim=groups \
  --oidc-username-prefix="oidc:" \
  --oidc-groups-prefix="oidc:"
```

Configure kubectl:

```bash
kubectl config set-credentials oidc-user \
  --auth-provider=oidc \
  --auth-provider-arg=idp-issuer-url=https://keycloak.example.com/auth/realms/kubernetes \
  --auth-provider-arg=client-id=kubernetes \
  --auth-provider-arg=client-secret=your-client-secret \
  --auth-provider-arg=refresh-token=refresh_token \
  --auth-provider-arg=id-token=id_token
```

Create RBAC binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-cluster-admins
subjects:
- kind: Group
  name: "oidc:cluster-admins"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Implementing Service Account for CI/CD Pipeline

Create dedicated service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-pipeline
  namespace: ci
```

Create role with deployment permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch"]
```

Bind role to service account:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-pipeline
  namespace: ci
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

Create long-lived token:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ci-pipeline-token
  namespace: ci
  annotations:
    kubernetes.io/service-account.name: ci-pipeline
type: kubernetes.io/service-account-token
```

Retrieve the token:

```bash
kubectl get secret ci-pipeline-token -n ci -o jsonpath='{.data.token}' | base64 --decode
```

## Common Pitfalls and Solutions

### Excessive Permissions

Problem: Users or service accounts have too many permissions

Solution:
1. Regularly audit permissions:
   ```bash
   kubectl get clusterrolebindings
   ```
2. Replace cluster-wide roles with namespace-scoped roles
3. Apply least privilege principle
4. Use role aggregation for easier management

### Missing RBAC Configurations

Problem: Users can't access resources they need

Solution:
1. Check authorization with impersonation:
   ```bash
   kubectl auth can-i get pods --as=jane
   ```
2. Create appropriate roles and bindings
3. Verify group memberships
4. Check namespaces

### Service Account Token Exposure

Problem: Service account tokens accessible to attackers

Solution:
1. Disable automatic token mounting:
   ```yaml
   automountServiceAccountToken: false
   ```
2. Use projected volume with limited lifetime
3. Implement pod security policies
4. Rotate service account tokens

### Certificate Expiration

Problem: Client certificates expire

Solution:
1. Set up monitoring for certificate expiration
2. Implement automated certificate rotation
3. Use OIDC or other authentication methods with better expiration handling
4. Document certificate renewal procedures

### Complex RBAC Structure

Problem: RBAC becomes unmanageable with many roles

Solution:
1. Use ClusterRoles for common permissions
2. Aggregate roles:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: monitoring
     aggregationRule:
       clusterRoleSelectors:
       - matchLabels:
           rbac.example.com/aggregate-to-monitoring: "true"
   rules: []
   ```
3. Implement naming conventions
4. Use tools to visualize RBAC relationships

## Advanced Configuration

### Webhook Authentication Example

Create a webhook authentication service:

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type AuthResponse struct {
    APIVersion string `json:"apiVersion"`
    Kind       string `json:"kind"`
    Status     struct {
        Authenticated bool     `json:"authenticated"`
        User          UserInfo `json:"user,omitempty"`
    } `json:"status"`
}

type UserInfo struct {
    Username string   `json:"username"`
    UID      string   `json:"uid"`
    Groups   []string `json:"groups"`
}

func authHandler(w http.ResponseWriter, r *http.Request) {
    var requestBody map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&requestBody); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Extract the token from the request
    spec, ok := requestBody["spec"].(map[string]interface{})
    if !ok {
        http.Error(w, "Invalid request format", http.StatusBadRequest)
        return
    }
    
    token, ok := spec["token"].(string)
    if !ok {
        http.Error(w, "Token not found", http.StatusBadRequest)
        return
    }
    
    // Validate the token (simplified example)
    var response AuthResponse
    response.APIVersion = "authentication.k8s.io/v1beta1"
    response.Kind = "TokenReview"
    
    // In a real implementation, you would validate the token
    // against your authentication system
    if token == "valid-token" {
        response.Status.Authenticated = true
        response.Status.User = UserInfo{
            Username: "webhook-user",
            UID:      "1234",
            Groups:   []string{"system:authenticated", "developers"},
        }
    } else {
        response.Status.Authenticated = false
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func main() {
    http.HandleFunc("/authenticate", authHandler)
    log.Fatal(http.ListenAndServeTLS(":8443", "server.crt", "server.key", nil))
}
```

### Implementing Advanced RBAC Patterns

Role with aggregation labels:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
```

Role for specific resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

### Implementing Custom Authorizers

Custom authorizer using the webhook mode:

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type SubjectAccessReviewSpec struct {
    User                  string                 `json:"user"`
    Groups                []string               `json:"groups"`
    ResourceAttributes    *ResourceAttributes    `json:"resourceAttributes,omitempty"`
    NonResourceAttributes *NonResourceAttributes `json:"nonResourceAttributes,omitempty"`
}

type ResourceAttributes struct {
    Namespace   string `json:"namespace,omitempty"`
    Verb        string `json:"verb"`
    Group       string `json:"group"`
    Version     string `json:"version"`
    Resource    string `json:"resource"`
    Subresource string `json:"subresource,omitempty"`
    Name        string `json:"name,omitempty"`
}

type NonResourceAttributes struct {
    Path string `json:"path"`
    Verb string `json:"verb"`
}

type AuthorizationResponse struct {
    APIVersion string `json:"apiVersion"`
    Kind       string `json:"kind"`
    Status     struct {
        Allowed bool   `json:"allowed"`
        Reason  string `json:"reason,omitempty"`
    } `json:"status"`
}

func authorizeHandler(w http.ResponseWriter, r *http.Request) {
    var requestBody map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&requestBody); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    var response AuthorizationResponse
    response.APIVersion = "authorization.k8s.io/v1beta1"
    response.Kind = "SubjectAccessReview"
    
    // Extract the spec
    spec, ok := requestBody["spec"].(map[string]interface{})
    if !ok {
        response.Status.Allowed = false
        response.Status.Reason = "Invalid request format"
    } else {
        // Extract user and resource information
        user, _ := spec["user"].(string)
        resourceAttrs, hasResourceAttrs := spec["resourceAttributes"].(map[string]interface{})
        
        // Implement your authorization logic here
        // This is a simple example that allows all requests from a specific user
        if user == "admin-user" {
            response.Status.Allowed = true
            response.Status.Reason = "Admin user is allowed all actions"
        } else if hasResourceAttrs {
            // Check resource-specific permissions
            resource, _ := resourceAttrs["resource"].(string)
            namespace, _ := resourceAttrs["namespace"].(string)
            verb, _ := resourceAttrs["verb"].(string)
            
            // Example: Allow read-only access to pods in default namespace
            if resource == "pods" && namespace == "default" && (verb == "get" || verb == "list" || verb == "watch") {
                response.Status.Allowed = true
                response.Status.Reason = "Read-only access to pods in default namespace"
            } else {
                response.Status.Allowed = false
                response.Status.Reason = "Permission denied by custom authorizer"
            }
        } else {
            response.Status.Allowed = false
            response.Status.Reason = "No resource attributes provided"
        }
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func main() {
    http.HandleFunc("/authorize", authorizeHandler)
    log.Fatal(http.ListenAndServeTLS(":8444", "server.crt", "server.key", nil))
}
```

## Summary

Kubernetes authentication and authorization are fundamental aspects of cluster security. By implementing proper controls, following the principle of least privilege, and regularly auditing access, you can create a secure foundation for your Kubernetes deployments.

Key takeaways:
- Choose authentication methods appropriate for your organization
- Implement RBAC with fine-grained permissions
- Use service accounts properly for workloads
- Regularly review and audit access controls
- Follow cloud provider best practices for managed Kubernetes

In the next chapter, we'll explore RBAC in depth, covering advanced configurations and management techniques.

## Further Reading

- [Kubernetes Authentication Documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Authorization Documentation](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Service Accounts Documentation](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)