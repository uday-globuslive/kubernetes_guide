# Kubernetes Pods and Containers

## Introduction to Pods

In Kubernetes, a Pod is the smallest and most basic deployable unit that can be created and managed. A Pod represents a single instance of a running process in your cluster and encapsulates one or more containers.

Think of a Pod as a logical host for your containers - containers within a Pod share the same network namespace, IP address, and storage volumes. This makes communication between containers in the same Pod straightforward since they can reach each other via `localhost`.

## Pod Characteristics

- **Atomicity**: Pods are created, scheduled, and retired as a unit. All containers in a Pod are deployed to the same node.
- **Networking**: Containers within a Pod share the same network namespace, meaning they share the same IP address and port space.
- **Storage**: Pods can specify a set of shared storage volumes that all containers within the Pod can access.
- **Lifecycle**: Pods are ephemeral by nature - they're not designed to run forever. When a Pod is terminated, it's not automatically rescheduled.

## Pod Structure

A basic Pod manifest looks like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-data
  labels:
    component: elasticsearch
    role: data
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    ports:
    - containerPort: 9200
      name: http
    - containerPort: 9300
      name: transport
    env:
    - name: ES_JAVA_OPTS
      value: "-Xms2g -Xmx2g"
    - name: node.roles
      value: "data"
    - name: discovery.seed_hosts
      value: "elasticsearch-master"
    - name: cluster.name
      value: "elk-cluster"
    volumeMounts:
    - name: data
      mountPath: /usr/share/elasticsearch/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: elasticsearch-data-pvc
```

## Multi-Container Pods

While many Pods include only a single container, multi-container Pods are a common pattern in Elasticsearch deployments. Some common patterns include:

### Sidecar Pattern

A sidecar container extends and enhances the main container. For example, logging agents like Filebeat can be added as sidecars to Elasticsearch Pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-with-filebeat
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    # elasticsearch container config...
  - name: filebeat
    image: docker.elastic.co/beats/filebeat:7.15.0
    volumeMounts:
    - name: elasticsearch-logs
      mountPath: /usr/share/elasticsearch/logs
    - name: filebeat-config
      mountPath: /usr/share/filebeat/filebeat.yml
      subPath: filebeat.yml
  volumes:
  - name: elasticsearch-logs
    emptyDir: {}
  - name: filebeat-config
    configMap:
      name: filebeat-config
```

### Ambassador Pattern

The ambassador pattern uses a proxy container to interface with external services. For instance, you could use an ambassador container to handle TLS termination for Elasticsearch:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-ambassador
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    # elasticsearch container config...
  - name: tls-proxy
    image: envoyproxy/envoy:v1.18.3
    ports:
    - containerPort: 443
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/envoy/certs
    - name: envoy-config
      mountPath: /etc/envoy/envoy.yaml
      subPath: envoy.yaml
  volumes:
  - name: tls-certs
    secret:
      secretName: elasticsearch-tls
  - name: envoy-config
    configMap:
      name: envoy-config
```

### Adapter Pattern

The adapter pattern standardizes output from the main container. For Elasticsearch, you might use an adapter to transform logs into a specific format:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-adapter
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    # elasticsearch container config...
  - name: log-transformer
    image: custom/log-transformer:1.0
    volumeMounts:
    - name: elasticsearch-logs
      mountPath: /usr/share/elasticsearch/logs
  volumes:
  - name: elasticsearch-logs
    emptyDir: {}
```

## Pod Lifecycle

Pods follow a defined lifecycle:

1. **Pending**: The Pod has been accepted by the Kubernetes cluster, but one or more containers have not been set up and made ready to run.
2. **Running**: The Pod has been bound to a node, and all the containers have been created.
3. **Succeeded**: All containers in the Pod have successfully terminated and will not be restarted.
4. **Failed**: All containers in the Pod have terminated, and at least one container has terminated in failure.
5. **Unknown**: The state of the Pod could not be determined.

## Init Containers

Init containers run before the main application containers start and can handle setup tasks. They're particularly useful in Elasticsearch deployments for tasks like increasing the `vm.max_map_count` setting:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-data
spec:
  initContainers:
  - name: increase-vm-max-map-count
    image: busybox
    command: ["sysctl", "-w", "vm.max_map_count=262144"]
    securityContext:
      privileged: true
  - name: fix-permissions
    image: busybox
    command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
    volumeMounts:
    - name: data
      mountPath: /usr/share/elasticsearch/data
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    # elasticsearch container config...
```

## Pod Resource Management

Properly configuring resources is critical for Elasticsearch performance in Kubernetes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-master
spec:
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    resources:
      requests:
        memory: "4Gi"
        cpu: "1"
      limits:
        memory: "4Gi"
        cpu: "2"
```

Best practices for Elasticsearch resource management:

- Set memory requests equal to limits to avoid OOM kills
- Allocate enough CPU for JVM garbage collection
- Reserve resources for the operating system (don't allocate 100% of node resources)

## Pod Security Context

Security contexts define privilege and access control settings for Pods and containers. For Elasticsearch, you typically need to:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-security
spec:
  securityContext:
    fsGroup: 1000
    runAsUser: 1000
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

## Pod Affinity and Anti-Affinity

Affinity rules are crucial for Elasticsearch to ensure high availability:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elasticsearch-data
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: component
            operator: In
            values:
            - elasticsearch
          - key: role
            operator: In
            values:
            - data
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
```

This configuration ensures that Elasticsearch data nodes are scheduled on different physical hosts, improving resilience.

## Pod Disruption Budgets

To ensure high availability during voluntary disruptions like cluster upgrades, use Pod Disruption Budgets:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: elasticsearch-pdb
spec:
  minAvailable: 2  # or maxUnavailable: 1
  selector:
    matchLabels:
      component: elasticsearch
      role: data
```

## Best Practices for ELK Stack Pods

1. **Resource Requests and Limits**: Always set appropriate resource requests and limits for Elasticsearch, Logstash, and Kibana. Elasticsearch is memory-intensive, so proper memory allocation is critical.

2. **Persistent Storage**: Use persistent volumes for Elasticsearch data to prevent data loss during Pod restarts.

3. **Pod Anti-Affinity**: Configure anti-affinity rules to distribute Elasticsearch nodes across different hosts.

4. **Init Containers**: Use init containers to handle prerequisites like kernel parameter tuning.

5. **Health Checks**: Implement proper liveness and readiness probes to ensure healthy pods.

6. **Security Contexts**: Run containers with non-root users and apply the principle of least privilege.

7. **Pod Topology Spread Constraints**: Use topology spread constraints to distribute Elasticsearch nodes across failure domains.

## Troubleshooting Pods

Common issues with Elasticsearch Pods in Kubernetes include:

- **OOM Kills**: Elasticsearch exceeding memory limits
- **CrashLoopBackOff**: Incorrectly configured Elasticsearch settings
- **Pod eviction**: Nodes running out of resources
- **Readiness probe failures**: Elasticsearch taking too long to start

To troubleshoot:

```bash
# View Pod status
kubectl get pods -n elastic-system

# View detailed Pod information
kubectl describe pod elasticsearch-master-0 -n elastic-system

# View Pod logs
kubectl logs elasticsearch-master-0 -n elastic-system

# View previous container logs (if container has restarted)
kubectl logs elasticsearch-master-0 -n elastic-system --previous
```

## Conclusion

Understanding Pods and their configuration is essential for deploying the ELK Stack on Kubernetes. By properly configuring resource constraints, security contexts, affinity rules, and using patterns like sidecars, you can create resilient and efficient Elasticsearch, Logstash, and Kibana deployments.