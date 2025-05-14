# Kubernetes RBAC: Role-Based Access Control

## Introduction to RBAC in Kubernetes

Role-Based Access Control (RBAC) is a method of regulating access to resources based on the roles of individual users within an organization. In Kubernetes, RBAC is a core security mechanism that determines who can access what resources and what actions they can perform on those resources. RBAC was added to Kubernetes in version 1.6 and has been the standard authorization model since version 1.8.

Kubernetes RBAC is designed around a few key concepts:

- **Subjects**: The entities that can request access (users, groups, service accounts)
- **Resources**: The API objects that need to be accessed (pods, services, deployments, etc.)
- **Verbs**: The operations that can be performed on resources (get, list, create, update, delete, etc.)
- **Roles and ClusterRoles**: Collections of rules that define permissions
- **RoleBindings and ClusterRoleBindings**: Assignment of roles to subjects

## RBAC Components

### Subjects: Users, Groups, and Service Accounts

#### Users

Kubernetes doesn't have an internal user database. Instead, it relies on external identity providers or client certificates:

- **Client certificates**: X.509 certificates signed by the cluster's certificate authority
- **External identity providers**: OIDC, LDAP, or other authentication mechanisms
- **Token-based authentication**: Webhook or bearer token authentication

Example user entry in a kubeconfig file:

```yaml
users:
- name: alice
  user:
    client-certificate: /path/to/alice.crt
    client-key: /path/to/alice.key
```

#### Groups

Groups are sets of users that are referenced together for RBAC bindings. Kubernetes doesn't manage groups directly; they come from the authentication system:

- **system:authenticated**: All authenticated users
- **system:unauthenticated**: Requests that aren't authenticated
- **system:masters**: Super-admin group with full cluster access
- **system:serviceaccounts**: All service accounts in the system
- **system:serviceaccounts:[namespace]**: All service accounts in a specific namespace

#### Service Accounts

Service accounts are Kubernetes API objects designed for processes running in pods:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
```

Pods can be associated with service accounts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: app-service-account
  containers:
  - name: main-container
    image: my-app:1.0
```

### Resources and API Groups

Kubernetes organizes its API into API groups, which helps with organizing and versioning. Some examples:

- **core** (also called ""): Includes pods, services, configmaps, secrets
- **apps**: Includes deployments, statefulsets, daemonsets
- **rbac.authorization.k8s.io**: Includes roles, rolebindings, clusterroles, clusterrolebindings
- **networking.k8s.io**: Includes ingresses, network policies

Resources can be specified at different levels of granularity:

- **All resources in an API group**: `apiGroups: ["apps"]`
- **Specific resource types**: `resources: ["deployments", "statefulsets"]`
- **Specific resource instances**: Using `resourceNames: ["my-deployment"]`
- **Subresources**: `resources: ["pods/log", "pods/exec"]`

### Verbs (Operations)

Verbs define the operations that can be performed on resources:

- **get**: Read a single resource
- **list**: Read a collection of resources
- **watch**: Stream updates to resources
- **create**: Create a new resource
- **update**: Modify an existing resource
- **patch**: Partially modify a resource
- **delete**: Remove a resource
- **deletecollection**: Remove a collection of resources

## Roles and ClusterRoles

### Role

A Role is a namespace-scoped set of permissions. It can only grant access to resources within the same namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### ClusterRole

A ClusterRole is similar to a Role but has cluster-wide scope. It can be used to grant access to:

- Cluster-scoped resources (like nodes)
- Non-resource endpoints (like /healthz)
- Resources across all namespaces

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### Default ClusterRoles

Kubernetes comes with several predefined ClusterRoles:

- **cluster-admin**: Superuser access to perform any action on any resource
- **admin**: Full access within a namespace
- **edit**: Read/write access to most resources in a namespace
- **view**: Read-only access to most resources in a namespace

## RoleBindings and ClusterRoleBindings

### RoleBinding

A RoleBinding grants the permissions defined in a Role to a user, group, or service account. It applies only to the namespace it's defined in.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: jenkins
  namespace: ci
roleRef:
  kind: Role # This must be Role or ClusterRole
  name: pod-reader # This must match the name of the Role or ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

A ClusterRoleBinding grants permissions across the entire cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding to a ClusterRole

You can use a RoleBinding to bind a ClusterRole to a namespace. This allows reusing ClusterRoles for namespace-level permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole # Note: using ClusterRole here
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Common RBAC Patterns

### Namespace Admin

Grant full control over a namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin
  namespace: team-a
subjects:
- kind: User
  name: team-a-lead
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: team-a-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

### Read-Only Users

Grant read-only access to a namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-viewers
  namespace: team-a
subjects:
- kind: Group
  name: team-a-viewers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

### Application-Specific Permissions

Create a role for a specific application's needs:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""] 
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-updater-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: config-updater
  namespace: default
roleRef:
  kind: Role
  name: configmap-updater
  apiGroup: rbac.authorization.k8s.io
```

### Aggregated ClusterRoles

Create a ClusterRole that extends an existing one:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-view: "true"
rules:
- apiGroups: [""] 
  resources: ["endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-view-role
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-view: "true"
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
```

## RBAC for Service Accounts

### Default Service Account

Each namespace has a default service account, but it has minimal permissions. For security, avoid using the default service account for applications with specific permission needs.

### Creating Dedicated Service Accounts

Best practice is to create dedicated service accounts for each application:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-role
rules:
- apiGroups: [""] 
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-role
  apiGroup: rbac.authorization.k8s.io
```

### Service Account Tokens

Kubernetes automatically mounts service account tokens into pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus-sa
  containers:
  - name: prometheus
    image: prom/prometheus:v2.30.0
    # Token is automatically mounted at /var/run/secrets/kubernetes.io/serviceaccount
```

In Kubernetes 1.21+, you can disable automatic token mounting or specify token properties:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-auto-token-sa
audience: my-audience
---
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  serviceAccountName: no-auto-token-sa
  automountServiceAccountToken: false
  containers:
  - name: main
    image: busybox
```

## Implementing Least Privilege

### Start with Minimum Permissions

Build roles with the minimum permissions needed:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# No permissions for other resources
```

### Use ResourceNames to Restrict Access to Specific Instances

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: specific-configmaps-editor
rules:
- apiGroups: [""] 
  resources: ["configmaps"]
  resourceNames: ["app-config", "app-env"]
  verbs: ["get", "update"]
```

### Avoid Overly Permissive Roles

Don't use wildcards for resources or verbs unless absolutely necessary:

```yaml
# Avoid this type of role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: overly-permissive-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

## Verifying and Testing RBAC

### Can-I API

Use the `can-i` subcommand to check permissions:

```bash
# Check if current user can create pods
kubectl auth can-i create pods

# Check if bob can list secrets in namespace dev
kubectl auth can-i list secrets --as=bob --namespace=dev

# Check if service account can delete deployments
kubectl auth can-i delete deployments --as=system:serviceaccount:default:myapp-sa
```

### Impersonation for Testing

Use impersonation to test permissions:

```bash
# List pods as user bob
kubectl get pods --as=bob

# Create deployment as service account
kubectl create -f deployment.yaml --as=system:serviceaccount:default:deployment-sa
```

### Access Matrix

Use tools like `kubectl-access-matrix` to analyze RBAC permissions:

```bash
# Install the plugin
kubectl krew install access-matrix

# Generate access matrix for default namespace
kubectl access-matrix --namespace=default

# Check specific service account
kubectl access-matrix --sa=myapp --namespace=default
```

## RBAC with External Authentication

### OpenID Connect (OIDC) Integration

Use OIDC providers like Google, GitHub, or Azure AD:

```yaml
# API server configuration
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=your-client-id.apps.googleusercontent.com
--oidc-username-claim=email
--oidc-groups-claim=groups
```

Kubeconfig configuration:

```yaml
users:
- name: jane@example.com
  user:
    auth-provider:
      name: oidc
      config:
        client-id: your-client-id.apps.googleusercontent.com
        client-secret: your-client-secret
        id-token: eyJh...
        idp-issuer-url: https://accounts.google.com
        refresh-token: Gl...
```

### Groups from External Providers

Map OIDC groups to Kubernetes RBAC bindings:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-cluster-admins
subjects:
- kind: Group
  name: oidc:cluster-admins@example.com  # Group from OIDC provider
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## RBAC Best Practices

### Namespace Separation

Use namespaces to isolate teams and applications, with appropriate RBAC for each:

```yaml
# Create namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    team: a
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-b
  labels:
    team: b

# Create team-specific RBAC
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-access
  namespace: team-a
subjects:
- kind: Group
  name: team-a@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

### Regular Auditing

Regularly review and audit RBAC configurations:

```bash
# List all cluster roles
kubectl get clusterroles

# List all cluster role bindings
kubectl get clusterrolebindings

# List role bindings across all namespaces
kubectl get rolebindings --all-namespaces

# Check who has cluster-admin access
kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
```

### Use Groups

Manage access through groups rather than individual users:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-view
subjects:
- kind: Group
  name: developers@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

### Avoid Unnecessary Permissions

Minimize ServiceAccount permissions, especially for third-party applications:

```yaml
# Restricted role for an external monitoring agent
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
- apiGroups: [""] 
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
- nonResourceURLs: ["/metrics", "/healthz"]
  verbs: ["get"]
```

## Advanced RBAC Topics

### Aggregated ClusterRoles

Create composable ClusterRoles using the aggregation feature:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-metrics-reader
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["custom.metrics.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### RBAC for API Extensions

Define roles for custom resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-operator
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheuses", "alertmanagers", "servicemonitors"]
  verbs: ["*"]
```

### API Request Validation

Use validating admission webhooks alongside RBAC for fine-grained control:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: resource-validations
webhooks:
- name: pod-policy.example.com
  clientConfig:
    service:
      namespace: validation-system
      name: validation-service
      path: "/validate-pods"
    caBundle: <base64-encoded-ca-cert>
  rules:
  - apiGroups: [""] 
    apiVersions: ["v1"]
    resources: ["pods"]
    operations: ["CREATE", "UPDATE"]
    scope: "Namespaced"
```

## Troubleshooting RBAC

### Common RBAC Issues

1. **Missing permissions**: Subject can't perform actions on resources
2. **Binding to wrong role**: Using the wrong role in a binding
3. **Namespace confusion**: Using RoleBinding for ClusterRole across namespaces
4. **Incorrect subject references**: Wrong naming in subject specifications

### Checking Existing Permissions

Useful commands for troubleshooting:

```bash
# Check what permissions a user has
kubectl auth can-i --list --as=jane

# Check role details
kubectl describe clusterrole admin

# View RoleBinding in a specific namespace
kubectl get rolebinding -n development -o yaml

# Check which ServiceAccount is used by a pod
kubectl get pod mypod -o jsonpath='{.spec.serviceAccountName}'

# View API requests for debugging (API server flag)
--audit-log-path=/var/log/kubernetes/audit.log
```

### Verbose Authorization Logging

Enable detailed logging for authorization decisions:

```
# API server flag
--v=5
```

### Correcting RBAC Issues

Example fixes for common problems:

```yaml
# Fix: Add missing permission to a role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-role
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
- apiGroups: ["apps"] # Added missing rule
  resources: ["replicasets"]
  verbs: ["get", "list"]

# Fix: Correct subject reference
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: User
  name: jane@example.com # Fixed user reference
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## RBAC in Production Environments

### GitOps for RBAC

Manage RBAC as code using GitOps principles:

```yaml
# File: rbac/team-a/rolebindings.yaml in Git repository
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-developers
  namespace: team-a
subjects:
- kind: Group
  name: team-a-devs@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

Then use CI/CD or tools like Flux/ArgoCD to apply these configurations.

### Automating RBAC Management

Use tools to generate RBAC configurations:

```bash
# Example using open-source RBAC Manager
cat <<EOF | kubectl apply -f -
apiVersion: rbacmanager.reactiveops.io/v1beta1
kind: RBACDefinition
metadata:
  name: organization-rbac
rbacBindings:
  - name: admin-team
    subjects:
      - kind: Group
        name: admin-team@example.com
    clusterRoleBindings:
      - clusterRole: cluster-admin
  - name: dev-team
    subjects:
      - kind: Group
        name: developers@example.com
    roleBindings:
      - clusterRole: edit
        namespace: development
      - clusterRole: view
        namespace: production
EOF
```

### Managing Service Account Tokens

Use token volume projection for better control over tokens:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-projection-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: token-volume
  volumes:
  - name: token-volume
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: my-audience
```

## Conclusion

RBAC is a crucial component of Kubernetes security. By properly implementing RBAC, you can create a secure, least-privilege environment for your cluster users and applications. Key takeaways include:

- Use namespaces and RBAC together for team and application isolation
- Follow the principle of least privilege by granting minimal permissions
- Leverage service accounts for application workloads
- Regularly audit and review RBAC configurations
- Automate RBAC management using GitOps and tooling
- Integrate with external identity providers for enterprise deployments

A well-designed RBAC system balances security with usability, providing appropriate access to resources while protecting the cluster from unauthorized operations.