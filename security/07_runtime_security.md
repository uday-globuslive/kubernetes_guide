# Runtime Security in Kubernetes

## Introduction

Runtime security in Kubernetes focuses on protecting containerized applications while they are executing. Unlike build-time security, which addresses vulnerabilities before deployment, runtime security deals with threats that emerge during application execution. This document explores the principles, tools, and best practices for implementing comprehensive runtime security in Kubernetes environments.

## Runtime Security Fundamentals

### The Runtime Security Landscape

Runtime security encompasses multiple layers of protection:

1. **Container Runtime Security**: Securing the container runtime (containerd, CRI-O)
2. **Kubernetes Pod Security**: Enforcing security policies at the pod level
3. **Node Security**: Protecting the underlying host systems
4. **Behavioral Monitoring**: Detecting anomalous activity
5. **Runtime Vulnerability Management**: Identifying vulnerabilities in running containers

## Container Runtime Security

### Container Runtime Interface (CRI)

The Container Runtime Interface (CRI) is the standard API between Kubernetes and container runtimes:

```
kubelet → CRI → containerd/CRI-O → OCI runtime (runc, kata, etc.)
```

Common container runtimes in Kubernetes:

| Runtime | Description | Security Features |
|---------|-------------|-------------------|
| containerd | Industry-standard container runtime | Seccomp, AppArmor, namespace isolation |
| CRI-O | Lightweight runtime for Kubernetes | OCI-compatible, reduced attack surface |
| Docker Engine | Legacy runtime (via dockershim, deprecated) | Various security options |
| gVisor | Application kernel for containers | Strong isolation through syscall interception |
| Kata Containers | Hardware-virtualized containers | VM-level isolation for containers |

### Container Runtime Configuration

Example of a secure containerd configuration in `/etc/containerd/config.toml`:

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  # Enable seccomp by default
  [plugins."io.containerd.grpc.v1.cri".containerd]
    default_runtime_name = "runc"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
        BinaryName = "runc"
        # Enable seccomp support
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options.SecurityOpt]
          seccomp = "/etc/containerd/seccomp.json"
```

## Pod-Level Security Controls

### Pod Security Context

Security contexts define privilege and access control settings for pods and containers:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: secure-container
    image: nginx:1.25.1-alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

### Pod Security Standards

Kubernetes offers three built-in security profiles:

1. **Privileged**: Unrestricted policy, provides maximum flexibility
2. **Baseline**: Minimally restrictive policy, prevents known privilege escalations
3. **Restricted**: Heavily restricted policy, follows security best practices

Applying Pod Security Standards at namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## System Call Filtering

### Seccomp Profiles

Secure Computing Mode (seccomp) restricts the syscalls a container can make to the host kernel:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom-seccomp.json
  containers:
  - name: secure-container
    image: nginx:1.25.1-alpine
```

Example of a custom seccomp profile (`custom-seccomp.json`):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_ARM64"
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "bind",
        "brk",
        "close",
        "connect",
        "exit",
        "exit_group",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "fstat",
        "futex",
        "getdents64",
        "getpid",
        "ioctl",
        "mmap",
        "nanosleep",
        "openat",
        "read",
        "recvfrom",
        "rt_sigaction",
        "rt_sigprocmask",
        "sendto",
        "set_robust_list",
        "sigaltstack",
        "socket",
        "stat",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### AppArmor Profiles

AppArmor provides Mandatory Access Control (MAC) for processes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/secure-container: localhost/custom-nginx
spec:
  containers:
  - name: secure-container
    image: nginx:1.25.1-alpine
```

Example AppArmor profile (`/etc/apparmor.d/custom-nginx`):

```
#include <tunables/global>

profile custom-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/openssl>

  # Allow network access
  network inet tcp,
  network inet udp,

  # Nginx needs to read its configuration
  /etc/nginx/** r,

  # Allow reading web content
  /var/www/** r,

  # Logs
  /var/log/nginx/*.log w,

  # Deny all other file access
  deny /** w,
  deny /etc/nginx/** w,
}
```

## Runtime Threat Detection

### Runtime Security Tools

1. **Falco**: Cloud-native runtime security tool

   Example Falco rule to detect unauthorized process execution:
   ```yaml
   - rule: Unauthorized Process Execution
     desc: Detects execution of unauthorized processes in containers
     condition: container and not user_known_shell_spawn_process and proc.name in (curl, wget, bash, sh)
     output: "Unauthorized process executed in container (user=%user.name command=%proc.cmdline container=%container.id)"
     priority: WARNING
   ```

2. **Aqua Security**: Commercial container security platform

3. **Sysdig Secure**: Security monitoring and forensics

4. **Cilium**: eBPF-based networking and security

### Running Falco in Kubernetes

```yaml
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
      hostPID: true
      hostNetwork: true
      serviceAccountName: falco
      containers:
      - name: falco
        image: falcosecurity/falco:0.34.1
        securityContext:
          privileged: true
        volumeMounts:
        - name: dev-fs
          mountPath: /dev
        - name: proc-fs
          mountPath: /host/proc
          readOnly: true
        - name: boot-fs
          mountPath: /host/boot
          readOnly: true
        - name: lib-modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr-fs
          mountPath: /host/usr
          readOnly: true
        - name: etc-fs
          mountPath: /host/etc
          readOnly: true
      volumes:
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
      - name: etc-fs
        hostPath:
          path: /etc
```

## Admission Controllers for Runtime Security

### OPA Gatekeeper

Open Policy Agent (OPA) Gatekeeper enforces policies at admission time:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoPrivilegedContainers
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
```

### Kyverno

Policy management designed specifically for Kubernetes:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-capabilities
spec:
  validationFailureAction: enforce
  rules:
  - name: drop-all-capabilities
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Containers must drop ALL capabilities and may only add specific ones"
      pattern:
        spec:
          containers:
          - securityContext:
              capabilities:
                drop:
                - ALL
```

## Advanced Runtime Protection

### Linux Security Modules (LSMs)

Kubernetes supports multiple Linux Security Modules:

| LSM | Description | When to Use |
|-----|-------------|-------------|
| AppArmor | Mandatory Access Control | General-purpose security profiles |
| SELinux | Label-based access control | Environments requiring strict access control |
| seccomp | System call filtering | Limiting container capabilities |

### eBPF for Runtime Security

Extended Berkeley Packet Filter (eBPF) provides low-level kernel observability:

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "file-monitoring"
spec:
  kprobes:
  - call: "security_file_permission"
    return: false
    syscall: false
    args:
    - index: 0
      type: "file"
    selectors:
    - matchBinaries:
      - operator: "In"
        values:
        - "/usr/bin/curl"
        - "/usr/bin/wget"
      matchCapabilities:
      - operator: "In"
        values:
        - "CAP_NET_ADMIN"
```

## Container Sandboxing

### gVisor

gVisor provides a user-space kernel for running containers:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

Using gVisor in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: sandbox-container
    image: nginx:1.25.1-alpine
```

### Kata Containers

Kata Containers run containers inside lightweight VMs:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

Using Kata Containers:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kata-pod
spec:
  runtimeClassName: kata
  containers:
  - name: kata-container
    image: nginx:1.25.1-alpine
```

## Privileged Container Mitigation

### Detecting and Preventing Privileged Containers

OPA Gatekeeper constraint to prevent privileged containers:

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivileged
        
        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          c.securityContext.privileged
          msg := sprintf("Privileged container is not allowed: %v", [c.name])
        }
        
        violation[{"msg": msg}] {
          c := input.review.object.spec.initContainers[_]
          c.securityContext.privileged
          msg := sprintf("Privileged init container is not allowed: %v", [c.name])
        }
```

## Implementing Runtime Security Monitoring

### Kubernetes Audit Logging

Configure audit policy in Kubernetes API server:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at the Metadata level
  - level: Metadata
    # Don't generate audit events for these high-volume resources unless accessing secrets
    omitStages:
      - "RequestReceived"
    resources:
    - group: ""
      resources: ["endpoints", "services", "configmaps"]
      verbs: ["get", "list", "watch"]
  
  # Log pod exec/attach/port-forward at the Request level
  - level: Request
    resources:
    - group: ""
      resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
      verbs: ["create", "update", "patch", "delete"]
  
  # Log all other resources at Metadata level
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

### Falco Alert Integration with Slack

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: security
data:
  falco.yaml: |-
    program_output:
      enabled: true
      program: "curl -s -X POST --data-binary @- https://hooks.slack.com/services/XXXX/YYYY/ZZZZ"
```

## Container Forensics

### Investigation Steps

1. **Identify the problematic container**:
   ```bash
   kubectl get pods -A
   kubectl describe pod <pod-name> -n <namespace>
   ```

2. **Capture container information**:
   ```bash
   # Capture container logs
   kubectl logs <pod-name> -n <namespace> > pod-logs.txt
   
   # Get container image details
   kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'
   ```

3. **Execute forensics commands in the container**:
   ```bash
   kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
   # Inside the container
   ps aux
   netstat -tulpn
   ls -la /proc/self/fd/
   ```

4. **Collect host-level forensics**:
   ```bash
   # On the node
   sudo crictl inspect <container-id>
   sudo ls -la /var/log/containers/
   sudo journalctl -u kubelet
   ```

## Comprehensive Runtime Security Strategy

### Defense-in-Depth Approach

1. **Secure the build pipeline**:
   - Scan images for vulnerabilities
   - Follow secure Dockerfile practices
   - Sign images

2. **Admission controls**:
   - Enforce Pod Security Standards
   - Implement OPA Gatekeeper or Kyverno
   - Verify image signatures

3. **Container hardening**:
   - Apply security contexts
   - Use seccomp/AppArmor profiles
   - Drop capabilities
   - Read-only filesystems

4. **Runtime protection**:
   - Deploy Falco or similar runtime monitoring
   - Configure audit logging
   - Implement network policies

5. **Regular security drills**:
   - Practice container forensics
   - Test incident response
   - Update security policies

### Example: Complete Security-Focused Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-hardened-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/custom-profile
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myregistry.example.com/secure-app:1.0.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
      privileged: false
      procMount: Default
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: data
      mountPath: /data
      readOnly: true
  volumes:
  - name: tmp
    emptyDir: {}
  - name: data
    configMap:
      name: app-data
  automountServiceAccountToken: false
```

## Best Practices Summary

1. **Adopt Pod Security Standards** in all namespaces
2. **Drop ALL capabilities** and add back only what's needed
3. **Use runtime monitoring tools** like Falco
4. **Implement admission control** with OPA Gatekeeper or Kyverno
5. **Apply seccomp profiles** to restrict syscalls
6. **Run as non-root** with minimum privileges
7. **Use read-only root filesystems** where possible
8. **Consider sandboxed runtimes** for high-security requirements
9. **Enable audit logging** for forensics and compliance
10. **Regularly update security policies** as threats evolve

## Conclusion

Runtime security in Kubernetes requires a multi-layered approach that combines proactive controls, runtime monitoring, and incident response capabilities. By implementing the security controls described in this document, organizations can significantly reduce the risk of container compromise and limit the potential impact of security incidents. Remember that runtime security is just one component of a comprehensive Kubernetes security strategy, which should also include build-time security, network security, and secure cluster configuration.

## Additional Resources

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Falco Documentation](https://falco.org/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/)
- [Kyverno Documentation](https://kyverno.io/docs/)
- [Seccomp Security Profiles](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [AppArmor in Kubernetes](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [gVisor Documentation](https://gvisor.dev/docs/)
- [Kata Containers Documentation](https://katacontainers.io/docs/)