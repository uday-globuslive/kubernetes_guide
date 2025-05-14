# DaemonSets

A DaemonSet ensures that all (or some) nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

## Use Cases for DaemonSets

DaemonSets are perfect for running background services on every node:

- **Node Monitoring**: Running log collection agents (like Fluentd or Filebeat)
- **Node Metrics Collection**: Running Prometheus node exporters or metrics agents
- **Storage Daemons**: Running distributed storage agents (like Ceph or GlusterFS)
- **Network Plugins**: CNI plugins like Calico or Flannel often use DaemonSets
- **Security Agents**: Running security scanners or compliance tools
- **Load Balancers**: Running proxy services (like HAProxy or Envoy) on each node
- **Node Maintenance**: Running maintenance scripts or backup agents

## DaemonSet vs. Deployment

Understanding when to use a DaemonSet instead of a Deployment:

| Feature | DaemonSet | Deployment |
|---------|-----------|------------|
| Purpose | Run a Pod on every node (or selected nodes) | Run multiple replicas of a Pod, anywhere in the cluster |
| Scaling | Automatic scaling based on node count | Manual or auto-scaling based on metrics |
| Pod placement | One per node (by default) | Placed by scheduler for optimal distribution |
| Use case | Node-level operations | Application workloads |

## Basic DaemonSet Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # This toleration is to have the daemonset runnable on master nodes
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

## Creating and Managing DaemonSets

### Creating a DaemonSet

```bash
kubectl apply -f daemon-set.yaml
```

### Checking DaemonSet Status

```bash
# Get basic information
kubectl get daemonset -n kube-system

# Detailed information
kubectl describe daemonset fluentd-elasticsearch -n kube-system
```

The output shows:
- Desired, current, ready, up-to-date, and available Pod counts
- Node selector and selector details
- Events related to the DaemonSet

### Updating a DaemonSet

Similar to Deployments, DaemonSets support updating Pods via rollout:

```bash
# Update image
kubectl set image daemonset/fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n kube-system

# Check rollout status
kubectl rollout status daemonset/fluentd-elasticsearch -n kube-system
```

### Rolling Back a DaemonSet

```bash
# View rollout history
kubectl rollout history daemonset/fluentd-elasticsearch -n kube-system

# Rollback to previous revision
kubectl rollout undo daemonset/fluentd-elasticsearch -n kube-system

# Rollback to specific revision
kubectl rollout undo daemonset/fluentd-elasticsearch -n kube-system --to-revision=2
```

### Deleting a DaemonSet

```bash
kubectl delete daemonset fluentd-elasticsearch -n kube-system
```

## DaemonSet Scheduling

### Running on Specific Nodes

Use `nodeSelector` to run the DaemonSet on specific nodes:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
```

### Using Node Affinity

Node affinity provides more flexible node selection:

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
```

### Using Tolerations

Tolerations allow Pods to be scheduled on nodes with matching taints:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node.kubernetes.io/unschedulable
        operator: Exists
        effect: NoSchedule
```

Common taint effects:
- `NoSchedule`: No new pods without matching toleration will be scheduled
- `PreferNoSchedule`: System will try to avoid scheduling pods without matching toleration
- `NoExecute`: Existing pods without matching toleration will be evicted

## DaemonSet Update Strategies

### RollingUpdate (Default)

Updates Pods one at a time:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

Parameters:
- `maxUnavailable`: Maximum number of Pods that can be unavailable during the update (absolute number or percentage)

### OnDelete

Only updates Pods when they are manually deleted:

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

## Advanced DaemonSet Patterns

### Init Containers with DaemonSets

Run setup operations before the main container:

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: init-ds
        image: busybox:1.28
        command: ['sh', '-c', 'echo The daemon is preparing && sleep 5']
```

### Multiple Containers in a DaemonSet Pod

Run sidecars alongside the main container:

```yaml
containers:
- name: main-daemon
  image: main-daemon:1.0
- name: sidecar-logger
  image: logger:1.0
```

### Host Networking

Access host network directly:

```yaml
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
```

### Host PID and IPC Namespaces

Access host processes and IPC:

```yaml
spec:
  template:
    spec:
      hostPID: true
      hostIPC: true
```

### Host Port Mapping

Expose container port on the host:

```yaml
ports:
- containerPort: 80
  hostPort: 9001
```

## Real-World DaemonSet Examples

### Monitoring Agent: Prometheus Node Exporter

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
        - --collector.netclass.ignored-devices=^(veth.*)$
        ports:
        - containerPort: 9100
          protocol: TCP
        resources:
          limits:
            cpu: 250m
            memory: 180Mi
          requests:
            cpu: 102m
            memory: 180Mi
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
          mountPropagation: HostToContainer
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

### Logging Agent: Filebeat

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
  labels:
    app: filebeat
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
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.15.1
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch.logging.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
```

### Network Plugin: Calico

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoExecute
        operator: Exists
      serviceAccountName: calico-node
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      initContainers:
      - name: install-cni
        image: calico/cni:v3.20.2
        command: ["/install-cni.sh"]
        env:
        - name: CNI_CONF_NAME
          value: "10-calico.conflist"
        - name: CNI_NETWORK_CONFIG
          valueFrom:
            configMapKeyRef:
              name: calico-config
              key: cni_network_config
        volumeMounts:
        - mountPath: /host/opt/cni/bin
          name: cni-bin-dir
        - mountPath: /host/etc/cni/net.d
          name: cni-net-dir
      containers:
      - name: calico-node
        image: calico/node:v3.20.2
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        - name: FELIX_TYPHAK8SSERVICENAME
          value: "calico-typha"
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"
        - name: CLUSTER_TYPE
          value: "k8s,bgp"
        - name: IP_AUTODETECTION_METHOD
          value: "interface=eth*"
        - name: IP6_AUTODETECTION_METHOD
          value: "interface=eth*"
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: 250m
        volumeMounts:
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
        - mountPath: /run/calico
          name: run-calico
        - mountPath: /var/run/calico
          name: var-run-calico
        - mountPath: /var/lib/calico
          name: var-lib-calico
        - mountPath: /var/run/nodeagent
          name: policysync
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: run-calico
        hostPath:
          path: /run/calico
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
      - name: policysync
        hostPath:
          path: /var/run/nodeagent
          type: DirectoryOrCreate
      - name: cni-bin-dir
        hostPath:
          path: /opt/cni/bin
      - name: cni-net-dir
        hostPath:
          path: /etc/cni/net.d
```

## Best Practices

1. **Set resource limits and requests**

```yaml
resources:
  limits:
    memory: 200Mi
  requests:
    cpu: 100m
    memory: 200Mi
```

2. **Use node affinity instead of nodeSelector for complex selection**

3. **Set update strategy carefully**

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 10%
```

4. **Add appropriate tolerations**

```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300
```

5. **Set proper termination grace period**

```yaml
terminationGracePeriodSeconds: 60
```

6. **Add readiness/liveness probes**

```yaml
livenessProbe:
  httpGet:
    path: /metrics
    port: 9100
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /metrics
    port: 9100
  initialDelaySeconds: 5
  periodSeconds: 10
```

7. **Use security contexts**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
```

8. **Minimize hostPath usage** (where possible)

9. **Label DaemonSets properly** for organizational clarity

```yaml
metadata:
  labels:
    app: node-exporter
    component: monitoring
    tier: infrastructure
```

10. **Use ConfigMaps/Secrets for configuration**

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/config
volumes:
- name: config-volume
  configMap:
    name: daemon-config
```

## Troubleshooting DaemonSets

### Common Issues

1. **Pods not scheduled on all nodes**
   - Check node selectors and affinities
   - Verify tolerations match node taints
   - Check if nodes are cordoned or unschedulable

2. **Pods in CrashLoopBackOff**
   - Check container logs: `kubectl logs <pod-name> -n <namespace>`
   - Verify container arguments and environment variables
   - Check if required host paths exist

3. **Pods not starting due to resource constraints**
   - Check node resources: `kubectl describe node <node-name>`
   - Adjust resource requests/limits

4. **Update not proceeding**
   - Check update strategy
   - Verify maxUnavailable setting
   - Look for PodDisruptionBudgets that may block updates

### Useful Commands

```bash
# Check DaemonSet status
kubectl get ds -n <namespace>

# Detailed information
kubectl describe ds <daemonset-name> -n <namespace>

# Check Pods created by DaemonSet
kubectl get pods -n <namespace> -l <label-selector>

# Check logs
kubectl logs -n <namespace> <pod-name>

# Check events for troubleshooting
kubectl get events -n <namespace> --field-selector involvedObject.name=<daemonset-name>
```

## Comparison with Other Kubernetes Resources

| Feature | DaemonSet | StatefulSet | Deployment |
|---------|-----------|------------|------------|
| Purpose | One Pod per node | Stateful applications | Stateless applications |
| Naming | No special naming | Predictable names (app-0, app-1) | Random suffixes |
| Scaling | Based on node count | Manual, ordered scaling | Manual or auto-scaling |
| Ordering | No guaranteed ordering | Strict ordering | No guaranteed ordering |
| Storage | Typically local or hostPath | Persistent volume per Pod | Shared or no storage |
| Network identity | Typically uses host network | Stable network identity | No stable identity |
| Use cases | System services, agents | Databases, distributed systems | Web applications, APIs |

DaemonSets provide a powerful way to ensure that specific functionality is available on every node in your Kubernetes cluster. By understanding when and how to use them effectively, you can implement robust logging, monitoring, networking, and other critical infrastructure components.