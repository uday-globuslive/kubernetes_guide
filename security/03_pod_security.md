# Pod Security in Kubernetes

## Introduction

Pod Security is a critical aspect of Kubernetes security that focuses on securing the pod runtime environment. Since pods are the basic building blocks in Kubernetes where applications run, securing them is essential for overall cluster security. This document covers Pod Security Standards, Pod Security Admission, and best practices for implementing pod-level security in an ELK Stack deployment on Kubernetes.

## Pod Security Standards

Kubernetes defines three levels of security profiles:

1. **Privileged**: Unrestricted policy, providing the widest possible level of permissions.
2. **Baseline**: Minimally restrictive policy that prevents known privilege escalations.
3. **Restricted**: Heavily restricted policy, following current pod hardening best practices.

### Example: Pod Security Standard Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elk-stack
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Pod Security Context

The Pod Security Context allows you to define privilege and access control settings for pods and containers.

### Key Security Context Settings

- `runAsUser`: Specifies the user ID (UID) to run the processes in containers
- `runAsGroup`: Specifies the group ID (GID) to run the processes
- `fsGroup`: Controls the ownership of volume filesystems
- `seccompProfile`: Controls Secure Computing Mode profiles
- `allowPrivilegeEscalation`: Controls whether processes can gain more privileges than their parent process
- `readOnlyRootFilesystem`: Makes the root filesystem read-only

### Example: Elasticsearch Pod with Security Context

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
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsNonRoot: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 1Gi
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
```

## Pod Security Admission Controller

Kubernetes 1.25+ uses the built-in Pod Security Admission (PSA) controller to enforce Pod Security Standards.

### Configuration Options

- **enforce**: Violations will cause the pod to be rejected
- **audit**: Violations will be recorded in the audit log
- **warn**: Users will receive warnings for violations

### Configuring PSA for ELK Stack

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: ["kube-system"]
```

## Security Policies with OPA Gatekeeper

Open Policy Agent (OPA) Gatekeeper provides more advanced policy capabilities beyond the built-in Pod Security Admission controller.

### Example: OPA Constraint for Elasticsearch Pods

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRestrictedPod
metadata:
  name: elasticsearch-pod-restrictions
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["StatefulSet", "Deployment"]
    namespaces:
      - "elk-stack"
  parameters:
    allowedHostPaths: []
    allowedCapabilities: []
    requiredDropCapabilities: ["ALL"]
    runAsUser:
      rule: "MustRunAsNonRoot"
    volumes:
      - "configMap"
      - "emptyDir"
      - "persistentVolumeClaim"
      - "secret"
```

## Securing ELK Stack Pods

### Elasticsearch Security Recommendations

1. Run Elasticsearch as a non-root user (typically uid 1000)
2. Use a read-only root filesystem
3. Drop all Linux capabilities
4. Apply resource limits and requests
5. Use seccomp profiles to limit system calls

### Example: Complete Elasticsearch with Enhanced Security

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
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
          privileged: false
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 1Gi
        env:
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: discovery.type
          value: single-node  # For production use appropriate discovery settings
        - name: bootstrap.memory_lock
          value: "true"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: elasticsearch-data
          mountPath: /usr/share/elasticsearch/data
        - name: elasticsearch-config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
          readOnly: true
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### Kibana Security Recommendations

1. Run as a non-root user
2. Use a read-only root filesystem
3. Apply resource limits
4. Drop all Linux capabilities

### Example: Kibana with Security Context

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
  replicas: 1
  template:
    metadata:
      labels:
        app: kibana
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.10.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 500Mi
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
          name: http
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: kibana-config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
          readOnly: true
      volumes:
      - name: tmp
        emptyDir: {}
      - name: kibana-config
        configMap:
          name: kibana-config
```

## Testing and Validating Pod Security

### Using kubeaudit

[kubeaudit](https://github.com/Shopify/kubeaudit) is a command-line tool to audit Kubernetes clusters for security issues.

```bash
# Install kubeaudit
curl -L https://github.com/Shopify/kubeaudit/releases/download/v0.17.0/kubeaudit_0.17.0_linux_amd64.tar.gz | tar xz
sudo mv kubeaudit /usr/local/bin

# Audit all resources in the elk-stack namespace
kubeaudit all -n elk-stack
```

### Using kube-bench

[kube-bench](https://github.com/aquasecurity/kube-bench) checks for CIS Kubernetes Benchmark compliance.

```bash
# Run kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View the results
kubectl logs $(kubectl get pods -l app=kube-bench -o name)
```

## Security Scanning with Trivy

[Trivy](https://github.com/aquasecurity/trivy) can scan container images for vulnerabilities.

```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan Elasticsearch image
trivy image docker.elastic.co/elasticsearch/elasticsearch:8.10.0
```

## Best Practices for Pod Security in ELK Stack

1. **Principle of Least Privilege**:
   - Run all ELK components as non-root users
   - Use a restricted Pod Security Standard
   - Drop all Linux capabilities and only add back those that are required

2. **Resource Management**:
   - Set appropriate CPU and memory limits for all ELK components
   - Monitor resource usage to fine-tune these limits

3. **Image Security**:
   - Use official Elastic images
   - Implement a vulnerability scanning process
   - Consider using a private registry with image signing

4. **Runtime Security**:
   - Enable seccomp profiles (RuntimeDefault at minimum)
   - Use read-only root filesystems where possible
   - Mount temporary directories as emptyDir volumes

5. **Update Strategy**:
   - Implement a regular patching schedule
   - Monitor CVE databases for vulnerabilities in ELK components

## Conclusion

Securing pods in your Kubernetes cluster is an essential part of deploying the ELK Stack securely. By implementing Pod Security Standards, using appropriate security contexts, and following best practices, you can significantly reduce the attack surface of your Elasticsearch, Logstash, and Kibana deployments.

## References

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Elastic Security Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/)