# Security Response and Incident Handling in Kubernetes

## Introduction

Effective security incident response is critical for Kubernetes environments. Despite the best preventive measures, security incidents will occur, and organizations must be prepared to detect, respond to, and recover from them. This document outlines a comprehensive approach to security incident response in Kubernetes environments, including preparation, detection, containment, investigation, remediation, and lessons learned.

## Kubernetes Security Incident Response Framework

### Incident Response Lifecycle

A complete Kubernetes incident response lifecycle includes:

1. **Preparation**: Establish tools, processes, and team readiness
2. **Detection**: Identify potential security incidents
3. **Containment**: Limit the scope and impact of the incident
4. **Investigation**: Understand what happened, how, and why
5. **Remediation**: Resolve the incident and restore normal operations
6. **Lessons Learned**: Improve security posture based on the incident

![Incident Response Lifecycle](https://mermaid.ink/img/pako:eNptksFuwjAMhl_F8qkgHnCcuEx7gJ22E9rFOZjErU3TJMoLQojw7k2hFCR8sv_f_-zYR6hJIUJdEwlKHAMZx5s6Ye7kp0RHLiM9SFZJl8jQsrG1Sk2JJE-GjNZdLfMIPipDwbEg6MdYCffLJC68as07DNL7V5Ywo_eeimDFy3q43jceP1nOZcSc71dWIybvP0jpUJojx7VR_oFJaXTIJ69SOYmixM3U8qhQTXXCv4Oy9oDr4INmOVUxkC0OINV-XC4ms49zulVWWVfb79YHBuFRufdz8W6yV_3FsB8GcRg22eVtPpxcpsPo47NfvPQ-_-57r_WhdUKkJPmB2G4rwRhAhHrILrR5g38Q0UIdSZSIHjNZ_b63_lo_?type=png)

## Preparation Phase

### Establish an Incident Response Team

Define roles and responsibilities for Kubernetes security incidents:

| Role | Responsibilities |
|------|------------------|
| Incident Commander | Coordinates the overall response effort |
| Kubernetes Administrator | Manages cluster operations during an incident |
| Security Analyst | Investigates and analyzes security threats |
| Application Owner | Provides application-specific expertise |
| Communications Lead | Manages internal and external communications |
| Legal/Compliance | Addresses regulatory and legal requirements |

### Create an Incident Response Plan

Document the incident response process specifically for Kubernetes:

```markdown
# Kubernetes Incident Response Plan

## 1. Incident Classification Criteria
- Severity levels: Critical, High, Medium, Low
- Kubernetes-specific criteria: cluster compromise, application compromise, data breach, etc.

## 2. Response Procedures
- Detection and reporting procedures
- Initial assessment and triage
- Containment strategies
- Investigation techniques
- Remediation steps
- Post-incident activities

## 3. Communication Plan
- Internal escalation path
- External communication guidelines
- Customer notification process (if applicable)

## 4. Documentation Requirements
- Incident timeline
- Actions taken
- Evidence collection
- Lessons learned
```

### Deploy Monitoring and Detection Tools

1. **Falco for Runtime Security Monitoring**:
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
         serviceAccountName: falco
         hostNetwork: true
         hostPID: true
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

2. **Audit Logging Configuration**:
   ```yaml
   # /etc/kubernetes/audit-policy.yaml
   apiVersion: audit.k8s.io/v1
   kind: Policy
   rules:
   - level: Metadata
     resources:
     - group: ""
       resources: ["secrets", "configmaps"]
     omitStages: ["RequestReceived"]
   - level: Request
     resources:
     - group: ""
       resources: ["pods"]
     verbs: ["create", "update", "patch", "delete"]
   - level: RequestResponse
     resources:
     - group: "rbac.authorization.k8s.io"
       resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
     verbs: ["create", "update", "patch", "delete"]
   - level: RequestResponse
     users: ["system:serviceaccount:kube-system:default"]
     resources:
     - group: ""
       resources: ["secrets"]
     verbs: ["get", "create", "update", "patch", "delete"]
   - level: Metadata
     omitStages: ["RequestReceived"]
   ```

3. **Prometheus AlertManager Rules**:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: kubernetes-security-alerts
     namespace: monitoring
   spec:
     groups:
     - name: kubernetes-security
       rules:
       - alert: PodRunningAsRoot
         expr: count(kube_pod_container_info{container!=""} unless on(namespace,pod,container) kube_pod_container_status_running{container!=""} == 0 or on(namespace,pod,container) kube_pod_container_security_context_runasnonroot{runAsNonRoot="false"} > 0)
         for: 5m
         labels:
           severity: warning
         annotations:
           summary: "Container running as root detected"
           description: "Container in pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is running as root."
       - alert: PrivilegedContainer
         expr: count(kube_pod_container_status_running{container!=""} * on(namespace,pod,container) kube_pod_container_security_context{container!="", privileged="true"}) > 0
         for: 5m
         labels:
           severity: critical
         annotations:
           summary: "Privileged container detected"
           description: "Privileged container {{ $labels.container }} in pod {{ $labels.pod }} in namespace {{ $labels.namespace }} detected."
   ```

4. **Event Logging with Fluentd**:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: fluentd-config
     namespace: logging
   data:
     fluent.conf: |
       <source>
         @type tail
         path /var/log/containers/*.log
         pos_file /var/log/fluentd-containers.log.pos
         time_format %Y-%m-%dT%H:%M:%S.%NZ
         tag kubernetes.*
         format json
         read_from_head true
       </source>
       
       <filter kubernetes.**>
         @type kubernetes_metadata
         kubernetes_url https://kubernetes.default.svc
         bearer_token_file /var/run/secrets/kubernetes.io/serviceaccount/token
         ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
       </filter>
       
       <match kubernetes.var.log.containers.**kube-apiserver**.log>
         @type elasticsearch
         host elasticsearch.logging
         port 9200
         logstash_format true
         logstash_prefix k8s-audit
         <buffer>
           @type file
           path /var/log/fluentd-buffers/kubernetes.audit
           flush_mode interval
           flush_interval 5s
         </buffer>
       </match>
   ```

### Conduct Tabletop Exercises

Regularly practice responding to simulated Kubernetes security incidents:

```markdown
# Kubernetes Security Incident Tabletop Exercise

## Scenario: Compromised Container
A container in the production namespace appears to be running unusual processes and making unexpected network connections.

## Timeline:
1. 0:00 - Alert triggered: "Unexpected process execution in container"
2. 0:05 - Initial assessment and team assembly
3. 0:15 - Containment decision point
4. 0:30 - Investigation process
5. 0:45 - Remediation planning
6. 1:00 - Recovery operations
7. 1:15 - Lessons learned discussion

## Questions for Discussion:
1. Who would be notified first?
2. What immediate steps would be taken?
3. How would we isolate the affected workload?
4. What evidence should be collected?
5. How do we determine the scope of the compromise?
6. What is our remediation strategy?
7. How do we verify the threat has been eliminated?
```

## Detection Phase

### Security Event Sources

Critical sources of security events in Kubernetes environments:

1. **API Server Audit Logs**: Records all API server requests
2. **Node-Level Logs**: System logs, container runtime logs, kubelet logs
3. **Application Logs**: Container stdout/stderr and application-specific logs
4. **Network Logs**: Service mesh telemetry, CNI logs, ingress controller logs
5. **Cloud Provider Logs**: AWS CloudTrail, Azure Activity Logs, GCP Audit Logs
6. **Security Tool Alerts**: Falco alerts, container vulnerability notifications
7. **Monitoring System Metrics**: Unusual resource usage patterns

### Key Detection Scenarios

| Scenario | Detection Method | Sample Alert |
|----------|------------------|-------------|
| Privilege Escalation | Falco, audit logs | User attempted to create ClusterRoleBinding granting excessive permissions |
| Unusual API Access | Audit logs, anomaly detection | Unusual pattern of API calls from service account |
| Container Breakout | Falco, host monitoring | Container process accessing host namespace |
| Crypto Mining | Resource monitoring | Unusual CPU pattern consistent with mining |
| Data Exfiltration | Network monitoring | Abnormal egress traffic patterns |
| Credential Theft | Secret access logs | Multiple sequential secret accesses |

### Sample Falco Rules for Detection

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: security
data:
  custom_rules.yaml: |-
    - rule: Terminal Shell in Container
      desc: A shell was spawned in a container
      condition: container and shell_procs and evt.type = execve and container.image.repository != "exception-image"
      output: Shell spawned in a container (user=%user.name container=%container.id image=%container.image.repository pod=%k8s.pod.name ns=%k8s.ns.name)
      priority: WARNING
    
    - rule: Kubectl Exec Activity
      desc: Detect kubectl exec operations
      condition: evt.type=execve and proc.name=kubectl and proc.args contains "exec"
      output: kubectl exec detected (user=%user.name command=%proc.cmdline)
      priority: NOTICE
    
    - rule: Sensitive Mount in Container
      desc: Sensitive host paths mounted in container
      condition: container and sensitive_mount
      output: Sensitive directory mounted in container (user=%user.name container=%container.id image=%container.image.repository pod=%k8s.pod.name mounts=%container.mounts)
      priority: WARNING
      
    - rule: Container Namespace Change
      desc: Detect namespace changes within containers that could be escape attempts
      condition: evt.type=setns and container
      output: Namespace change operation in container (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name container=%container.id image=%container.image.repository pid=%proc.pid command=%proc.cmdline)
      priority: CRITICAL
```

## Containment Phase

### Immediate Response Actions

1. **Isolate Compromised Pods**:
   ```yaml
   # Network Policy to isolate pod
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: isolate-compromised-pod
     namespace: production
   spec:
     podSelector:
       matchLabels:
         app: compromised-app
         id: compromised-pod-id
     policyTypes:
     - Ingress
     - Egress
   ```

2. **Quarantine Compromised Nodes**:
   ```bash
   # Cordon the compromised node
   kubectl cordon compromised-node-name
   
   # Drain the node (force if necessary)
   kubectl drain compromised-node-name --force --ignore-daemonsets --delete-emptydir-data
   ```

3. **Prevent New Deployments**:
   ```yaml
   # Admission Policy to block new deployments
   apiVersion: admissionregistration.k8s.io/v1
   kind: ValidatingWebhookConfiguration
   metadata:
     name: emergency-freeze
   webhooks:
   - name: freeze.example.com
     admissionReviewVersions: ["v1"]
     clientConfig:
       service:
         name: admission-webhook
         namespace: security
         path: "/validate"
       caBundle: <base64-encoded-ca-bundle>
     rules:
     - apiGroups: ["", "apps"]
       apiVersions: ["v1"]
       operations: ["CREATE", "UPDATE"]
       resources: ["pods", "deployments", "statefulsets", "daemonsets"]
       scope: "Namespaced"
     failurePolicy: Fail
     sideEffects: None
   ```

4. **Revoke Compromised Credentials**:
   ```bash
   # Delete compromised service account
   kubectl delete serviceaccount compromised-sa -n compromised-namespace
   
   # Delete all secrets for the service account
   kubectl get secrets -n compromised-namespace --field-selector type=kubernetes.io/service-account-token -o custom-columns="NAME:.metadata.name,SA:.metadata.annotations.kubernetes\.io/service-account\.name" | grep "compromised-sa" | awk '{print $1}' | xargs kubectl delete secret -n compromised-namespace
   ```

### Forensic Container Deployment

Deploy a forensic container for investigation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: forensic-pod
  namespace: security
spec:
  # Schedule on the same node as the compromised pod
  nodeName: <compromised-node>
  hostPID: true
  hostNetwork: true
  hostIPC: true
  containers:
  - name: forensic-tools
    image: forensics-toolkit:latest
    securityContext:
      privileged: true
    command: ["/bin/sh", "-c", "sleep 14400"]
    volumeMounts:
    - name: forensic-volume
      mountPath: /forensic
    - name: host-root
      mountPath: /host
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: forensic-volume
    emptyDir: {}
  - name: host-root
    hostPath:
      path: /
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
```

### Evidence Collection

Script for collecting Kubernetes evidence:

```bash
#!/bin/bash
# Kubernetes Incident Evidence Collection Script

# Set timestamp for evidence file naming
TIMESTAMP=$(date +%Y%m%d%H%M%S)
EVIDENCE_DIR="/forensic/evidence-$TIMESTAMP"
mkdir -p $EVIDENCE_DIR

# Collect cluster-level information
echo "Collecting cluster information..."
kubectl version > $EVIDENCE_DIR/k8s-version.txt
kubectl get nodes -o yaml > $EVIDENCE_DIR/nodes.yaml
kubectl get namespaces -o yaml > $EVIDENCE_DIR/namespaces.yaml
kubectl get clusterroles,clusterrolebindings -o yaml > $EVIDENCE_DIR/cluster-rbac.yaml

# Collect namespace-specific information
NAMESPACE="compromised-namespace"
echo "Collecting information from namespace $NAMESPACE..."
kubectl get all -n $NAMESPACE -o yaml > $EVIDENCE_DIR/${NAMESPACE}-resources.yaml
kubectl get configmaps,secrets -n $NAMESPACE -o yaml > $EVIDENCE_DIR/${NAMESPACE}-configs.yaml
kubectl get roles,rolebindings -n $NAMESPACE -o yaml > $EVIDENCE_DIR/${NAMESPACE}-rbac.yaml
kubectl get networkpolicies -n $NAMESPACE -o yaml > $EVIDENCE_DIR/${NAMESPACE}-networkpolicies.yaml

# Collect pod-specific information
POD_NAME="compromised-pod"
echo "Collecting information for pod $POD_NAME in namespace $NAMESPACE..."
kubectl get pod $POD_NAME -n $NAMESPACE -o yaml > $EVIDENCE_DIR/${POD_NAME}.yaml
kubectl logs $POD_NAME -n $NAMESPACE --all-containers > $EVIDENCE_DIR/${POD_NAME}-logs.txt
kubectl logs $POD_NAME -n $NAMESPACE --all-containers --previous > $EVIDENCE_DIR/${POD_NAME}-previous-logs.txt 2>/dev/null

# Collect pod events
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME > $EVIDENCE_DIR/${POD_NAME}-events.txt

# Collect node-level evidence
NODE_NAME=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
echo "Collecting evidence from node $NODE_NAME..."

# If we're running in a privileged forensic container on the node
if [ -d "/host" ]; then
  # Capture container filesystem
  CONTAINER_ID=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d'/' -f3)
  CONTAINER_ROOT="/host/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots"
  
  # Create filesystem snapshot
  mkdir -p $EVIDENCE_DIR/container-fs
  find $CONTAINER_ROOT -name "*$CONTAINER_ID*" -exec cp -r {} $EVIDENCE_DIR/container-fs \; 2>/dev/null
  
  # Get process list from node
  ps -ef > $EVIDENCE_DIR/node-processes.txt
  
  # Get network connections
  netstat -tupan > $EVIDENCE_DIR/node-connections.txt
  
  # Get loaded kernel modules
  lsmod > $EVIDENCE_DIR/node-kernel-modules.txt
  
  # Collect important logs
  cp -r /host/var/log $EVIDENCE_DIR/host-logs
fi

# Create tar archive of evidence
cd /forensic
tar -czf evidence-$TIMESTAMP.tar.gz evidence-$TIMESTAMP
echo "Evidence collection complete. Archive saved to /forensic/evidence-$TIMESTAMP.tar.gz"
```

## Investigation Phase

### Analyzing Container Compromise

1. **Examine Container Processes**:
   ```bash
   # In forensic container
   PID=$(crictl inspect $CONTAINER_ID | jq '.info.pid')
   nsenter -t $PID -n netstat -tupan
   nsenter -t $PID -m ls -la /proc/*/exe 2>/dev/null | grep deleted
   ```

2. **Analyze Container Filesystem**:
   ```bash
   # Look for unexpected binaries
   nsenter -t $PID -m find / -type f -perm -u=x -not -path "*/proc/*" -not -path "*/sys/*" -not -path "*/dev/*" -ls
   
   # Check for modified files
   nsenter -t $PID -m find / -type f -mtime -1 -not -path "*/proc/*" -not -path "*/sys/*" -not -path "*/dev/*" -ls
   ```

3. **Examine Network Connections**:
   ```bash
   # Check established connections
   nsenter -t $PID -n ss -tupan
   
   # Look for unusual listening ports
   nsenter -t $PID -n ss -lnpt
   ```

### Kubernetes API Audit Log Analysis

Script to parse API audit logs for suspicious activity:

```python
#!/usr/bin/env python3
import json
import sys
import re
from datetime import datetime, timedelta

def parse_audit_log(log_file, lookback_hours=24):
    # Define patterns of suspicious activity
    suspicious_patterns = [
        {
            "name": "Privilege escalation",
            "pattern": lambda e: (e["objectRef"].get("resource") in ["clusterroles", "clusterrolebindings", "roles", "rolebindings"] and 
                                e["verb"] in ["create", "update", "patch"])
        },
        {
            "name": "Secret access",
            "pattern": lambda e: (e["objectRef"].get("resource") == "secrets" and 
                                 e["verb"] in ["get", "watch", "list"])
        },
        {
            "name": "Pod exec",
            "pattern": lambda e: (e["objectRef"].get("subresource") == "exec" and
                                 e["objectRef"].get("resource") == "pods")
        },
        {
            "name": "ConfigMap modification",
            "pattern": lambda e: (e["objectRef"].get("resource") == "configmaps" and
                                 e["verb"] in ["create", "update", "patch", "delete"])
        },
        {
            "name": "ServiceAccount creation",
            "pattern": lambda e: (e["objectRef"].get("resource") == "serviceaccounts" and
                                 e["verb"] == "create")
        }
    ]
    
    # Calculate cutoff time
    cutoff_time = datetime.now() - timedelta(hours=lookback_hours)
    
    suspicious_events = []
    
    with open(log_file, 'r') as f:
        for line in f:
            try:
                event = json.loads(line)
                
                # Skip events older than lookback period
                event_time = datetime.strptime(event["requestReceivedTimestamp"], 
                                             "%Y-%m-%dT%H:%M:%SZ")
                if event_time < cutoff_time:
                    continue
                
                # Check event against suspicious patterns
                for pattern in suspicious_patterns:
                    if pattern["pattern"](event):
                        suspicious_events.append({
                            "type": pattern["name"],
                            "timestamp": event["requestReceivedTimestamp"],
                            "user": event.get("user", {}).get("username", "unknown"),
                            "sourceIPs": event.get("sourceIPs", []),
                            "userAgent": event.get("userAgent", ""),
                            "resource": event.get("objectRef", {}).get("resource", ""),
                            "name": event.get("objectRef", {}).get("name", ""),
                            "namespace": event.get("objectRef", {}).get("namespace", ""),
                            "verb": event.get("verb", "")
                        })
                        break
                        
            except json.JSONDecodeError:
                continue
    
    return suspicious_events

def generate_report(events):
    if not events:
        print("No suspicious events detected in the specified time period.")
        return
    
    print(f"Found {len(events)} suspicious events:")
    print("-" * 80)
    
    # Group events by type
    event_types = {}
    for event in events:
        if event["type"] not in event_types:
            event_types[event["type"]] = []
        event_types[event["type"]].append(event)
    
    # Print summary by type
    for event_type, events_list in event_types.items():
        print(f"\n{event_type}: {len(events_list)} events")
        print("-" * 40)
        
        for event in events_list:
            print(f"Time: {event['timestamp']}")
            print(f"User: {event['user']}")
            print(f"Source IPs: {', '.join(event['sourceIPs'])}")
            print(f"Action: {event['verb']} {event['resource']} {event['namespace']}/{event['name']}")
            print(f"User Agent: {event['userAgent']}")
            print("-" * 30)

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python audit_analyzer.py <audit_log_file> [lookback_hours]")
        sys.exit(1)
    
    log_file = sys.argv[1]
    lookback_hours = 24
    if len(sys.argv) >= 3:
        lookback_hours = int(sys.argv[2])
    
    events = parse_audit_log(log_file, lookback_hours)
    generate_report(events)
```

### Timeline Construction

Create a detailed event timeline using available logs:

```markdown
# Incident Timeline

## Initial Access
- 2023-06-15 14:23:45 UTC: First suspicious login from unrecognized IP address 
  - User: system:serviceaccount:default:app-service-account
  - API Server audit log #123456

## Discovery and Lateral Movement
- 2023-06-15 14:25:13 UTC: Service account used to list pods across namespaces
  - User: system:serviceaccount:default:app-service-account
  - API Server audit log #123457

- 2023-06-15 14:27:01 UTC: Secret accessed in production namespace
  - Resource: secrets/database-credentials
  - User: system:serviceaccount:default:app-service-account
  - API Server audit log #123458

## Privilege Escalation
- 2023-06-15 14:30:22 UTC: Attempt to create ClusterRoleBinding
  - Name: compromised-admin-access
  - User: system:serviceaccount:default:app-service-account
  - API Server audit log #123459

## Persistence
- 2023-06-15 14:32:45 UTC: New deployment created in kube-system namespace
  - Name: monitoring-service (malicious backdoor)
  - User: kubernetes-admin
  - API Server audit log #123460

## Actions on Objectives
- 2023-06-15 14:35:10 UTC: Data exfiltration attempt detected
  - Pod: monitoring-service
  - Destination: 198.51.100.24:8080
  - Network log #789012
```

## Remediation Phase

### Removing Compromised Resources

1. **Delete Malicious Workloads**:
   ```bash
   # Delete compromised deployments
   kubectl delete deployment malicious-deployment -n compromised-namespace
   
   # Delete compromised pods
   kubectl delete pod malicious-pod -n compromised-namespace
   
   # Remove compromised service accounts
   kubectl delete serviceaccount compromised-sa -n compromised-namespace
   ```

2. **Revoke Excessive Permissions**:
   ```bash
   # Remove malicious role bindings
   kubectl delete clusterrolebinding malicious-binding
   kubectl delete rolebinding malicious-binding -n compromised-namespace
   ```

3. **Rotate Compromised Secrets**:
   ```bash
   # Delete and recreate affected secrets
   kubectl delete secret database-credentials -n production
   kubectl create secret generic database-credentials \
     --from-literal=username=newuser \
     --from-literal=password=newpassword \
     -n production
   ```

### Restore from Known-Good State

Restore affected workloads from trusted sources:

```yaml
# Apply verified configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: restored-deployment
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: restored-app
  template:
    metadata:
      labels:
        app: restored-app
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: trusted-registry.example.com/app:verified-version
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
```

### Apply Additional Security Controls

Implement enhanced security measures post-incident:

```yaml
# Apply stricter Pod Security Standard
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Add network policy to limit egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: limit-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432

---
# Enable additional audit logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods", "pods/exec", "services", "endpoints", "secrets", "configmaps"]
  verbs: ["create", "update", "patch", "delete"]
```

## Recovery Phase

### Verification Checklist

Create a verification checklist to ensure recovery is complete:

```markdown
# Kubernetes Incident Recovery Verification Checklist

## Cluster Integrity
- [ ] API server configurations verified
- [ ] etcd data integrity confirmed
- [ ] Control plane components restarted
- [ ] Node status checked
- [ ] Cluster version and patches verified

## Workload Security
- [ ] All affected workloads rebuilt from trusted sources
- [ ] All container images verified and scanned
- [ ] All pods running with proper security contexts
- [ ] Resource limits applied to all workloads
- [ ] Pod Security Standards enforced

## Access Controls
- [ ] All compromised credentials rotated
- [ ] Service account permissions audited
- [ ] RBAC policies reviewed and tightened
- [ ] Authentication mechanisms verified
- [ ] Sensitive operations require additional authorization

## Network Security
- [ ] Network policies applied to all namespaces
- [ ] Egress traffic restricted
- [ ] Service mesh mTLS enabled
- [ ] Ingress controllers configured securely
- [ ] External endpoints secured

## Monitoring and Detection
- [ ] Security monitoring tools operational
- [ ] Alerts configured for suspicious activities
- [ ] Audit logging enabled
- [ ] Log aggregation functioning
- [ ] Incident detection capabilities verified

## Documentation
- [ ] Incident timeline documented
- [ ] Affected systems inventoried
- [ ] Recovery actions documented
- [ ] Incident report completed
- [ ] Lessons learned identified
```

### Security Posture Improvement

Post-incident security enhancements:

1. **Implement GitOps for Continuous Security Verification**:
   ```yaml
   # FluxCD configuration for security controls
   apiVersion: source.toolkit.fluxcd.io/v1beta2
   kind: GitRepository
   metadata:
     name: security-controls
     namespace: flux-system
   spec:
     interval: 1m
     url: https://github.com/example/security-controls
     ref:
       branch: main
   ---
   apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
   kind: Kustomization
   metadata:
     name: security-policies
     namespace: flux-system
   spec:
     interval: 10m
     path: ./policies
     prune: true
     sourceRef:
       kind: GitRepository
       name: security-controls
   ```

2. **Implement Continuous Compliance Scanning**:
   ```yaml
   # Kube-bench CronJob for CIS benchmark scanning
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: kube-bench-scan
     namespace: security
   spec:
     schedule: "0 2 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             hostPID: true
             containers:
             - name: kube-bench
               image: aquasec/kube-bench:latest
               command:
               - /bin/sh
               - -c
               - kube-bench run --json | tee /var/log/kube-bench-results-$(date +%Y%m%d).json
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
               - name: results
                 mountPath: /var/log
             restartPolicy: OnFailure
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
             - name: results
               persistentVolumeClaim:
                 claimName: security-scan-results
   ```

## Lessons Learned Phase

### Incident Retrospective Template

```markdown
# Kubernetes Security Incident Retrospective

## Incident Summary
- **Incident ID**: INC-2023-06-15-001
- **Date and Time**: 2023-06-15 14:23:45 UTC to 2023-06-16 09:45:23 UTC
- **Duration**: 19 hours, 22 minutes
- **Severity**: Critical
- **Status**: Resolved
- **Incident Commander**: Jane Smith

## Incident Description
Brief description of the incident, including the attack vector, affected components, and impact.

## Timeline
| Time | Event | Actor | Details |
|------|-------|-------|---------|
| 2023-06-15 14:23:45 | Initial detection | Monitoring system | Unusual API server activity detected |
| 2023-06-15 14:35:00 | Incident declared | SOC Analyst | Escalated to security team |
| 2023-06-15 15:10:00 | Containment started | Incident Response Team | Isolated affected pods |
| ... | ... | ... | ... |
| 2023-06-16 09:45:23 | Recovery complete | Operations team | Verification checklist completed |

## Root Cause Analysis
Detailed analysis of how the incident occurred, including the initial entry point, exploitation methods, and progression.

## Impact Assessment
- **Services Affected**: List of affected services
- **Data Compromise**: Assessment of any data exposure or loss
- **Performance Impact**: Description of any performance degradation
- **Business Impact**: Description of business impacts (financial, reputational, etc.)

## Actions Taken
Detailed list of all response actions taken during the incident.

## What Went Well
1. Item 1
2. Item 2
3. Item 3

## What Could Be Improved
1. Item 1
2. Item 2
3. Item 3

## Action Items
| Item | Description | Owner | Due Date | Status |
|------|-------------|-------|----------|--------|
| 1 | Implement network policies in all namespaces | Network Team | 2023-06-30 | In Progress |
| 2 | Enhance monitoring for privileged containers | Security Team | 2023-07-15 | Not Started |
| 3 | Update incident response playbook | IR Team | 2023-07-01 | Not Started |

## Preventative Measures
Specific technical and procedural controls to prevent similar incidents in the future.
```

### Security Control Improvements

Based on lessons learned, implement technical improvements:

```yaml
# OPA Gatekeeper policy to prevent privilege escalation
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spspallowprivilegeescalationcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPAllowPrivilegeEscalationContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspallowprivilegeescalationcontainer
        
        violation[{"msg": msg, "details": {}}] {
          c := input_containers[_]
          c.securityContext.allowPrivilegeEscalation
          msg := sprintf("Privilege escalation is not allowed: %v", [c.name])
        }
        
        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }
        
        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }
```

## Building a Comprehensive Security Response Program

### Developing Response Playbooks

Create specific playbooks for common Kubernetes security incidents:

```markdown
# Kubernetes Incident Response Playbook: Container Escape

## Overview
This playbook addresses container escape incidents, where a container breakout allows access to the host or other containers.

## Detection
**Indicators of Compromise**:
- Processes running outside expected namespaces
- Unexpected host filesystem access
- Tampering with kubelet or container runtime
- Host-level network connections from container processes

**Detection Tools**:
- Falco rules for namespace violations
- Host monitoring agents
- Kubernetes audit logs

## Response Procedure

### 1. Initial Assessment (T+0 to T+15 minutes)
- [ ] Confirm the alert is not a false positive
- [ ] Identify the compromised container and host
- [ ] Assemble the incident response team
- [ ] Establish communication channels

### 2. Containment (T+15 to T+30 minutes)
- [ ] Isolate the affected node:
  ```bash
  kubectl cordon <node-name>
  kubectl drain <node-name> --force --ignore-daemonsets --delete-emptydir-data
  ```
- [ ] If possible, collect process list before termination:
  ```bash
  ssh <node-name> "ps -ef > /tmp/process-list.txt"
  ```
- [ ] Stop the container runtime on the affected node:
  ```bash
  ssh <node-name> "systemctl stop containerd"
  ```

### 3. Evidence Collection (T+30 to T+60 minutes)
- [ ] Deploy forensic pod to collect evidence (see template)
- [ ] Capture memory dump if possible
- [ ] Create disk image of affected container and host
- [ ] Collect relevant logs:
  ```bash
  ssh <node-name> "journalctl -u kubelet > /tmp/kubelet.log"
  ssh <node-name> "journalctl -u containerd > /tmp/containerd.log"
  ```

### 4. Investigation (T+1 to T+4 hours)
- [ ] Analyze how the escape occurred
- [ ] Determine the extent of post-escape activities
- [ ] Identify affected resources
- [ ] Establish attack timeline
- [ ] Document findings

### 5. Eradication (T+4 to T+6 hours)
- [ ] Remove compromised workloads
- [ ] Rebuild affected node from clean image
- [ ] Reset node certificates:
  ```bash
  kubeadm init phase upload-certs --upload-certs
  kubeadm init phase kubelet-finalize all
  ```

### 6. Recovery (T+6 to T+10 hours)
- [ ] Return node to service with enhanced security
  ```bash
  kubectl uncordon <node-name>
  ```
- [ ] Deploy workloads with additional security controls
- [ ] Implement additional monitoring
- [ ] Verify recovery with security testing

### 7. Post-Incident (T+1 to T+7 days)
- [ ] Conduct incident retrospective
- [ ] Update documentation
- [ ] Implement preventative measures
- [ ] Share lessons learned
```

### Building a Security Incident Response Team (SIRT)

Structure and responsibilities for a Kubernetes-focused SIRT:

```markdown
# Kubernetes Security Incident Response Team

## Team Structure

### Core Team
- **Incident Commander**: Coordinates response activities and decisions
- **Kubernetes Administrator**: Manages cluster operations and configuration
- **Security Analyst**: Investigates threats and performs forensics
- **Application Owner**: Provides expertise on affected applications
- **Communications Lead**: Manages stakeholder communications

### Extended Team
- **Executive Sponsor**: Provides authority and resources
- **Legal Counsel**: Addresses legal and compliance requirements
- **Cloud Provider Expert**: Assists with cloud infrastructure issues
- **External Security Consultant**: Provides specialized expertise as needed

## Roles and Responsibilities

### Incident Commander
- Leads the incident response process
- Makes critical decisions
- Assigns tasks and tracks progress
- Ensures proper incident documentation
- Reports to stakeholders

### Kubernetes Administrator
- Executes technical containment actions
- Performs cluster configuration changes
- Assists with evidence collection
- Implements recovery procedures
- Validates cluster health post-incident

### Security Analyst
- Analyzes alerts and logs
- Performs forensic investigation
- Identifies compromised resources
- Develops indicators of compromise
- Recommends security improvements

### Application Owner
- Provides application-specific knowledge
- Assesses impact on application services
- Assists with application recovery
- Validates application functionality
- Implements application-level security controls

### Communications Lead
- Manages internal communications
- Drafts external notifications
- Coordinates with public relations
- Documents incident for reporting
- Ensures appropriate information sharing

## Activation Process

1. **Detection**: Initial alert or report received
2. **Triage**: Security Analyst assesses severity and scope
3. **Activation**: Incident Commander assembles appropriate team members
4. **Notification**: Team members notified via established channels
5. **Initiation**: Initial response call and handoff from detection to response

## Training and Readiness

1. Regular tabletop exercises
2. Technical training on Kubernetes security
3. Incident response simulations
4. Tool familiarity workshops
5. Post-incident reviews and improvements
```

## Conclusion

Effective security incident response in Kubernetes environments requires a well-prepared team, established processes, and appropriate tools. By following the framework outlined in this document, organizations can develop the capability to detect, respond to, and recover from Kubernetes security incidents while minimizing their impact. Remember that incident response is a continuous improvement process—each incident provides an opportunity to learn and enhance your security posture.

## Additional Resources

- [Kubernetes Security Response Guide](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [NIST Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)
- [Cloud Native Security Incident Response](https://www.cncf.io/blog/2020/12/17/cloud-native-security-incident-response/)
- [Falco Runtime Security](https://falco.org/docs/)
- [Kubernetes Audit Logging](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)
- [Forensics for Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)
- [SANS Incident Handler's Handbook](https://www.sans.org/white-papers/33901/)