# Kubernetes Security Best Practices

## Introduction

Securing Kubernetes environments requires a comprehensive approach that addresses threats at multiple layers of the stack. This document provides a collection of best practices to help organizations build and maintain secure Kubernetes deployments. These recommendations are based on industry standards, real-world experience, and security frameworks such as the CIS Kubernetes Benchmark and NIST guidelines.

## Security Architecture Overview

### Defense in Depth

A robust Kubernetes security architecture implements multiple layers of protection:

![Kubernetes Security Layers](https://mermaid.ink/img/pako:eNp1kU1uwjAQha8y8pIVxA22LFhUldJKVVlQNl4YwhQ8JHbkH4QQ9-_YQQrQwmZm3nvzPE_sERLNERII51tNMktbx6xi7_RrdURTKLVnklfW5v7BMOlWaMXMmivtj7rKFDwrhbYm6NtIcvtrBCeF7qRy6KXzX9SojGtTrCvo8LwuTveNpw-qYGUgo_uF0EAjhw-azSPflYmyzNmikFYqk9yXqCyMwq5_licF2WhN_kEyjAUtyTHqYEPSaDofTXpnFvbb6jP1re1r2217eCZ2rNzrOfswkav-ZNQdB7EY1s3lbfY8OIX98OG5P3zpvf_d92GJ2jhqyblsQwjDeQJJpbUOIWA9SlxikyP_oAYChjmKEoJDltcvb7iZWsE?type=png)

1. **Infrastructure Layer**: Physical and cloud provider security
2. **Cluster Layer**: Kubernetes API server, etcd, controllers, and node security
3. **Container Layer**: Image security, runtime protection, and isolation
4. **Application Layer**: Code security, dependencies, and configurations
5. **Data Layer**: Secrets management, encryption, and access controls
6. **Network Layer**: Network policies, service mesh, and ingress/egress controls

## Cluster Security Fundamentals

### Secure Cluster Configuration

1. **API Server Configuration**:
   ```yaml
   # /etc/kubernetes/apiserver.yaml (sample)
   apiVersion: v1
   kind: Pod
   metadata:
     name: kube-apiserver
     namespace: kube-system
   spec:
     containers:
     - name: kube-apiserver
       image: registry.k8s.io/kube-apiserver:v1.28.3
       command:
       - kube-apiserver
       - --anonymous-auth=false
       - --authorization-mode=Node,RBAC
       - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
       - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
       - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
       - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
       - --audit-log-path=/var/log/kubernetes/audit.log
       - --audit-log-maxage=30
       - --audit-log-maxbackup=10
       - --audit-log-maxsize=100
   ```

2. **etcd Security**: Protect your cluster's data store
   ```yaml
   # etcd configuration
   apiVersion: v1
   kind: Pod
   metadata:
     name: etcd
     namespace: kube-system
   spec:
     containers:
     - name: etcd
       image: registry.k8s.io/etcd:3.5.6-0
       command:
       - etcd
       - --cert-file=/etc/kubernetes/pki/etcd/server.crt
       - --key-file=/etc/kubernetes/pki/etcd/server.key
       - --client-cert-auth=true
       - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
       - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
       - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
       - --peer-client-cert-auth=true
       - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
       - --auto-tls=false
       - --peer-auto-tls=false
   ```

3. **Kubelet Configuration**:
   ```yaml
   # /var/lib/kubelet/config.yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   authentication:
     anonymous:
       enabled: false
     webhook:
       enabled: true
     x509:
       clientCAFile: /etc/kubernetes/pki/ca.crt
   authorization:
     mode: Webhook
   protectKernelDefaults: true
   readOnlyPort: 0
   tlsCertFile: /etc/kubernetes/pki/kubelet.crt
   tlsPrivateKeyFile: /etc/kubernetes/pki/kubelet.key
   ```

### Secure Cluster Operations

1. **Regular Updates**:
   ```bash
   # Check current Kubernetes version
   kubectl version --short
   
   # Plan cluster upgrade (example for kubeadm-based clusters)
   kubeadm upgrade plan
   
   # Upgrade kubeadm
   apt-get update && apt-get install -y kubeadm=1.28.3-00
   
   # Apply the upgrade
   kubeadm upgrade apply v1.28.3
   
   # Upgrade kubelet and kubectl
   apt-get update && apt-get install -y kubelet=1.28.3-00 kubectl=1.28.3-00
   systemctl daemon-reload
   systemctl restart kubelet
   ```

2. **Monitor API Server Access**:
   ```bash
   # Check who has cluster-admin access
   kubectl get clusterrolebindings -o json | jq '.items[] | 
     select(.roleRef.name=="cluster-admin") | 
     {name: .metadata.name, subjects: .subjects}'
   ```

3. **Audit Logging Configuration**:
   ```yaml
   # /etc/kubernetes/audit-policy.yaml
   apiVersion: audit.k8s.io/v1
   kind: Policy
   rules:
   - level: Metadata
     resources:
     - group: ""
       resources: ["secrets"]
   - level: RequestResponse
     resources:
     - group: ""
       resources: ["pods"]
   - level: Request
     verbs: ["create", "update", "patch", "delete"]
   - level: Metadata
     omitStages:
     - "RequestReceived"
   ```

## Workload Security

### Pod Security Standards

Implement Pod Security Standards to secure your workloads:

```yaml
# Apply Pod Security Standards to namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Security Context Configuration

Apply security contexts to limit container privileges:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
    fsGroup: 3000
  containers:
  - name: app
    image: myapp:1.0.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      runAsUser: 10001
      runAsGroup: 10001
      readOnlyRootFilesystem: true
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
```

### Container Security Hardening

1. **Minimal Base Images**:
   ```dockerfile
   # Use distroless images when possible
   FROM gcr.io/distroless/java17-debian11:nonroot
   COPY --chown=nonroot:nonroot myapp.jar /app/myapp.jar
   WORKDIR /app
   USER nonroot
   CMD ["myapp.jar"]
   ```

2. **Multi-stage Builds**:
   ```dockerfile
   # Build stage
   FROM golang:1.20-alpine AS builder
   WORKDIR /app
   COPY . .
   RUN CGO_ENABLED=0 go build -o /app/server

   # Final stage
   FROM alpine:3.18
   RUN addgroup -S appgroup && adduser -S appuser -G appgroup
   USER appuser
   COPY --from=builder /app/server /app/server
   CMD ["/app/server"]
   ```

## Network Security

### Network Policies

Default deny-all policy to enforce zero-trust networking:

```yaml
# Default deny all ingress and egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Allow specific traffic with explicit policies:

```yaml
# Allow specific ingress and egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### TLS Everywhere

1. **Service Mesh for mTLS**:
   ```yaml
   # Istio example for enabling mTLS
   apiVersion: security.istio.io/v1beta1
   kind: PeerAuthentication
   metadata:
     name: default
     namespace: production
   spec:
     mtls:
       mode: STRICT
   ```

2. **Secure Ingress**:
   ```yaml
   # Ingress with TLS configuration
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: secure-ingress
     namespace: production
     annotations:
       kubernetes.io/ingress.class: nginx
       nginx.ingress.kubernetes.io/ssl-redirect: "true"
   spec:
     tls:
     - hosts:
       - api.example.com
       secretName: api-tls-cert
     rules:
     - host: api.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: api-service
               port:
                 number: 80
   ```

## Authentication and Authorization

### RBAC Best Practices

1. **Principle of Least Privilege**:
   ```yaml
   # Specific role for pod management only
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: production
     name: pod-manager
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   ---
   # Bind role to specific service account
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: pod-manager-binding
     namespace: production
   subjects:
   - kind: ServiceAccount
     name: deployment-manager
     namespace: production
   roleRef:
     kind: Role
     name: pod-manager
     apiGroup: rbac.authorization.k8s.io
   ```

2. **Time-bound Access**:
   ```bash
   # Generate short-lived kubeconfig with limited privileges
   kubectl create token deployment-manager --duration=2h
   ```

3. **Regularly Audit RBAC**:
   ```bash
   # List all cluster roles and their permissions
   kubectl get clusterrole -o custom-columns="ROLE:.metadata.name,RESOURCE:.rules[*].resources,VERBS:.rules[*].verbs"
   
   # Check who has specific permissions
   kubectl auth can-i create pods --as=system:serviceaccount:production:deployment-manager -n production
   ```

## Secret Management

### Secret Protection

1. **Encrypt Secrets at Rest**:
   ```yaml
   # /etc/kubernetes/encryption-config.yaml
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

2. **Use External Secret Stores**:
   ```yaml
   # External Secrets Operator with AWS Secrets Manager
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: database-credentials
     namespace: production
   spec:
     refreshInterval: "15m"
     secretStoreRef:
       name: aws-secretsmanager
       kind: ClusterSecretStore
     target:
       name: database-credentials
       creationPolicy: Owner
     data:
     - secretKey: username
       remoteRef:
         key: production/db-creds
         property: username
     - secretKey: password
       remoteRef:
         key: production/db-creds
         property: password
   ```

3. **Mount Secrets as Files**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secure-app
   spec:
     containers:
     - name: app
       image: myapp:1.0.0
       volumeMounts:
       - name: config-volume
         mountPath: /etc/secrets
         readOnly: true
     volumes:
     - name: config-volume
       secret:
         secretName: app-credentials
         defaultMode: 0400
   ```

## Threat Detection and Response

### Runtime Security Monitoring

1. **Deploy Falco for Runtime Monitoring**:
   ```yaml
   # Falco DaemonSet
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: falco
     namespace: security
   spec:
     selector:
       matchLabels:
         app: falco
     template:
       metadata:
         labels:
           app: falco
       spec:
         hostNetwork: true
         hostPID: true
         hostIPC: true
         containers:
         - name: falco
           image: falcosecurity/falco:0.34.1
           securityContext:
             privileged: true
           volumeMounts:
           - mountPath: /host/var/run/docker.sock
             name: docker-socket
           - mountPath: /host/dev
             name: dev-fs
           - mountPath: /host/proc
             name: proc-fs
             readOnly: true
           - mountPath: /host/boot
             name: boot-fs
             readOnly: true
           - mountPath: /host/lib/modules
             name: lib-modules
             readOnly: true
           - mountPath: /host/usr
             name: usr-fs
             readOnly: true
           - mountPath: /etc/falco
             name: falco-config
         volumes:
         - name: docker-socket
           hostPath:
             path: /var/run/docker.sock
         - name: dev-fs
           hostPath:
             path: /dev
         - name: proc-fs
           hostPath:
             path: /proc
         - name: boot-fs
           hostPath:
             path: /boot
         - name: lib-modules
           hostPath:
             path: /lib/modules
         - name: usr-fs
           hostPath:
             path: /usr
         - name: falco-config
           configMap:
             name: falco-config
   ```

2. **Custom Falco Rules**:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: falco-config
     namespace: security
   data:
     falco_rules.yaml: |-
       - rule: Unauthorized Process Launched
         desc: Detect unauthorized processes running in containers
         condition: container and not user_known_shell_spawn_process and proc.name in (nc, ncat, netcat, wget, curl, bash, sh, ssh)
         output: Unauthorized process launched in container (user=%user.name command=%proc.cmdline container=%container.id)
         priority: WARNING
       - rule: Container Shell Access
         desc: Detect shell access to containers
         condition: container and proc.name = bash
         output: Shell access to container (user=%user.name container=%container.id image=%container.image.repository)
         priority: NOTICE
   ```

### Incident Response

1. **Service Account Token Invalidation**:
   ```bash
   # Delete service account to invalidate all tokens
   kubectl delete serviceaccount compromised-account -n production
   
   # Create a new service account
   kubectl create serviceaccount replacement-account -n production
   
   # Update deployments to use the new service account
   kubectl set serviceaccount deployment/app replacement-account -n production
   ```

2. **Pod Isolation During Investigation**:
   ```yaml
   # Network policy to isolate suspicious pod
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: isolate-suspicious-pod
     namespace: production
   spec:
     podSelector:
       matchLabels:
         app: suspicious-app
     policyTypes:
     - Ingress
     - Egress
   ```

3. **Forensic Container**:
   ```yaml
   # Pod for forensic investigation
   apiVersion: v1
   kind: Pod
   metadata:
     name: forensic-tools
     namespace: production
   spec:
     nodeName: <compromised-node-name>
     hostPID: true
     hostNetwork: true
     containers:
     - name: forensic
       image: alpine:latest
       command: ["/bin/sh", "-c", "apk add --no-cache curl procps lsof && sleep 3600"]
       securityContext:
         privileged: true
       volumeMounts:
       - name: host-fs
         mountPath: /host
     volumes:
     - name: host-fs
       hostPath:
         path: /
   ```

## Supply Chain Security

### Secure CI/CD Pipeline

1. **Image Scanning in CI/CD**:
   ```yaml
   # GitHub Actions workflow example
   name: Container Security Pipeline
   
   on:
     push:
       branches: [ main ]
     pull_request:
       branches: [ main ]
   
   jobs:
     security-scan:
       runs-on: ubuntu-latest
       steps:
         - name: Checkout code
           uses: actions/checkout@v3
         
         - name: Build image
           run: docker build -t app:${{ github.sha }} .
         
         - name: Scan for vulnerabilities
           uses: aquasecurity/trivy-action@master
           with:
             image-ref: app:${{ github.sha }}
             format: sarif
             output: trivy-results.sarif
             severity: 'CRITICAL,HIGH'
   ```

2. **Sign and Verify Container Images**:
   ```bash
   # Sign image with cosign
   cosign sign --key cosign.key myregistry.example.com/app:latest
   
   # Verify image signature
   cosign verify --key cosign.pub myregistry.example.com/app:latest
   ```

3. **Software Bill of Materials (SBOM)**:
   ```bash
   # Generate SBOM with syft
   syft myregistry.example.com/app:latest -o spdx-json > app-sbom.json
   
   # Attach SBOM to image
   cosign attach sbom --sbom app-sbom.json myregistry.example.com/app:latest
   ```

### Secure Deployment Workflows

1. **GitOps with Sealed Secrets**:
   ```yaml
   # SealedSecret for secure GitOps deployment
   apiVersion: bitnami.com/v1alpha1
   kind: SealedSecret
   metadata:
     name: database-credentials
     namespace: production
   spec:
     encryptedData:
       username: AgBy8hCM8...truncated
       password: AgCtr85s3...truncated
   ```

2. **Admission Control Policies**:
   ```yaml
   # OPA Gatekeeper policy to enforce image signing
   apiVersion: templates.gatekeeper.sh/v1beta1
   kind: ConstraintTemplate
   metadata:
     name: signedimages
   spec:
     crd:
       spec:
         names:
           kind: SignedImages
         validation:
           openAPIV3Schema:
             properties:
               registry:
                 type: string
     targets:
       - target: admission.k8s.gatekeeper.sh
         rego: |
           package signedimages
           
           violation[{"msg": msg}] {
             container := input.review.object.spec.containers[_]
             not startswith(container.image, input.parameters.registry)
             msg := sprintf("Container image %v comes from untrusted registry", [container.image])
           }
   ```

## Compliance Automation

### Automated Security Checks

1. **Kube-bench for CIS Benchmarks**:
   ```yaml
   # Run kube-bench as a Job
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: kube-bench
     namespace: security
   spec:
     template:
       spec:
         hostPID: true
         containers:
         - name: kube-bench
           image: aquasec/kube-bench:latest
           command: ["kube-bench", "--json", "/tmp/results.json"]
           volumeMounts:
           - name: var-lib-kubelet
             mountPath: /var/lib/kubelet
             readOnly: true
           - name: etc-systemd
             mountPath: /etc/systemd
             readOnly: true
           - name: etc-kubernetes
             mountPath: /etc/kubernetes
             readOnly: true
         restartPolicy: Never
         volumes:
         - name: var-lib-kubelet
           hostPath:
             path: /var/lib/kubelet
         - name: etc-systemd
           hostPath:
             path: /etc/systemd
         - name: etc-kubernetes
           hostPath:
             path: /etc/kubernetes
   ```

2. **Popeye for Kubernetes Sanitization**:
   ```yaml
   # Run Popeye as a CronJob
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: popeye-scan
     namespace: security
   spec:
     schedule: "0 1 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             serviceAccountName: popeye-sa
             containers:
             - name: popeye
               image: derailed/popeye:latest
               args:
               - -o
               - json
               - -f
               - /tmp/popeye-report.json
             restartPolicy: OnFailure
   ```

### Continuous Compliance Monitoring

1. **Prometheus Compliance Metrics**:
   ```yaml
   # Prometheus recording rules for compliance metrics
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: compliance-metrics
     namespace: monitoring
   spec:
     groups:
     - name: kubernetes.compliance
       interval: 30s
       rules:
       - record: compliance:pods_without_security_context:ratio
         expr: sum(kube_pod_security_context{securityContext="true"} / count(kube_pod_info)) * 100
       - record: compliance:containers_with_privileged:count
         expr: count(kube_pod_container_status_running{container_privileged="true"})
       - record: compliance:pods_without_resource_limits:count
         expr: count(kube_pod_container_info) unless count(kube_pod_container_resource_limits)
   ```

2. **Security Dashboards**:
   ```yaml
   # Grafana dashboard ConfigMap
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: security-dashboard
     namespace: monitoring
   data:
     security-dashboard.json: |
       {
         "annotations": {
           "list": []
         },
         "editable": true,
         "panels": [
           {
             "datasource": "Prometheus",
             "fieldConfig": {
               "defaults": {
                 "color": {
                   "mode": "thresholds"
                 },
                 "thresholds": {
                   "mode": "absolute",
                   "steps": [
                     { "color": "red", "value": null },
                     { "color": "yellow", "value": 50 },
                     { "color": "green", "value": 90 }
                   ]
                 }
               }
             },
             "title": "Security Context Coverage",
             "type": "gauge"
           }
         ],
         "title": "Kubernetes Security Dashboard"
       }
   ```

## Putting It All Together

### Multi-Layer Security Implementation

A comprehensive Kubernetes security implementation includes:

1. **Secure Infrastructure**: Properly configured cloud provider security and access controls
2. **Hardened Clusters**: CIS-compliant Kubernetes configuration
3. **Secure Workloads**: Pod Security Standards and security contexts
4. **Network Security**: Default-deny network policies and mTLS
5. **RBAC and Authentication**: Least privilege and secure access
6. **Secret Management**: Encrypted secrets and proper mounting
7. **Runtime Protection**: Behavioral monitoring and threat detection
8. **Supply Chain Security**: Image scanning, signing, and verification
9. **Compliance Automation**: Regular checks and automated reporting

### Security Maturity Model

| Level | Description | Sample Controls |
|-------|-------------|-----------------|
| Level 1: Basic | Fundamental security controls | RBAC, Network Policies, Secrets Management |
| Level 2: Standard | Enhanced protection mechanisms | Pod Security Standards, TLS, Image Scanning |
| Level 3: Advanced | Comprehensive security stack | Runtime Monitoring, Automated Compliance, mTLS |
| Level 4: Mature | Full security lifecycle | Supply Chain Security, Threat Hunting, Automated Response |

### Implementation Roadmap

1. **Phase 1: Foundation**
   - Implement secure cluster configuration
   - Deploy basic network policies
   - Configure RBAC with least privilege
   - Set up secure secret management

2. **Phase 2: Enhanced Protection**
   - Apply Pod Security Standards
   - Implement image scanning in CI/CD
   - Configure TLS for all services
   - Deploy admission controllers

3. **Phase 3: Runtime Security**
   - Implement runtime security monitoring
   - Set up automated compliance checks
   - Enable audit logging and monitoring
   - Develop incident response procedures

4. **Phase 4: Advanced Security**
   - Implement service mesh with mTLS
   - Deploy image signing and verification
   - Build comprehensive security dashboards
   - Establish continuous security improvement

## Best Practices Summary

1. **Never Trust, Always Verify**: Implement zero-trust principles across all layers
2. **Defense in Depth**: Deploy multiple, redundant security controls
3. **Least Privilege**: Grant minimal access required for system functionality
4. **Default Deny**: Block all traffic, then selectively allow only what's needed
5. **Secure Defaults**: Start with hardened configurations and security baselines
6. **Automate Security**: Implement security as code and continuous scanning
7. **Isolate Workloads**: Use namespaces, network policies, and other isolation mechanisms
8. **Encrypt Everything**: Use encryption for data at rest and in transit
9. **Validate Input**: Filter and sanitize all user inputs in applications
10. **Monitor and Alert**: Implement comprehensive monitoring and alerting
11. **Keep Updated**: Regularly update and patch all components
12. **Security Awareness**: Train developers and operators on security best practices

## Conclusion

Kubernetes security is a continuous journey that requires attention to multiple layers of the stack. By implementing the best practices outlined in this document, organizations can significantly reduce their attack surface and enhance their security posture. Remember that security is not a one-time effort but an ongoing process of improvement, monitoring, and adaptation to new threats and vulnerabilities.

## Additional Resources

- [Kubernetes Security Documentation](https://kubernetes.io/docs/concepts/security/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Application Container Security Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)
- [Kubernetes Security Best Practices (CNCF)](https://www.cncf.io/blog/2019/01/14/9-kubernetes-security-best-practices-everyone-must-follow/)
- [Kubernetes Security Cheat Sheet](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Aqua Security Kubernetes Security Guide](https://www.aquasec.com/cloud-native-academy/kubernetes-101/kubernetes-security-best-practices-10-steps-to-securing-k8s/)
- [Snyk Kubernetes Security Guide](https://snyk.io/learn/kubernetes-security/)