# Stateful Applications Best Practices in Kubernetes

## Introduction

Deploying stateful applications in Kubernetes can be challenging due to their data persistence requirements and complex lifecycle management needs. Unlike stateless applications, which can be easily scaled and replaced, stateful applications require careful handling of data, stable network identities, and ordered deployment and scaling. This chapter explores best practices, patterns, and techniques for successfully running stateful applications in Kubernetes environments.

## Understanding Stateful Workloads in Kubernetes

### Characteristics of Stateful Applications

Stateful applications have specific characteristics that make them different from stateless workloads:

1. **Data persistence**: Require data to survive pod restarts and rescheduling
2. **Ordering and dependencies**: Often have strict startup and shutdown sequences
3. **Stable network identities**: Need predictable hostnames and DNS entries
4. **Non-interchangeable instances**: Each instance may have a unique role and configuration
5. **Complex clustering**: Often form clusters with leader election, replication, etc.

Common examples of stateful applications include:
- Databases (MySQL, PostgreSQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Distributed caches (Redis, Memcached)
- Search engines (Elasticsearch)
- File and object storage systems (MinIO, Ceph)

### StatefulSet vs. Deployment

Understanding the differences between StatefulSets and Deployments is crucial for managing stateful applications:

| Feature | StatefulSet | Deployment |
|---------|-------------|------------|
| Pod Identity | Stable, predictable names with ordered indices | Random, replaceable identities |
| DNS Names | Stable DNS entries per pod | Service-level DNS only |
| Volume Attachment | Persistent volumes tied to specific pods | Random PV assignment |
| Scaling Order | Sequential (ordered) | Parallel (unordered) |
| Update Strategy | Ordered update with configurable partition | Random, replaceable pods |
| Deletion | Graceful, ordered termination | Parallel termination |

### Persistent Volume Architecture

Kubernetes persistent volumes provide the foundation for stateful applications:

```
┌───────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                       │
│                                                           │
│  ┌─────────────────┐      references     ┌─────────────┐  │
│  │  StatefulSet    │─────────────────────▶ PodTemplate │  │
│  └─────────────────┘                     └──────┬──────┘  │
│                                                 │         │
│                                                 ▼         │
│  ┌─────────────────┐                     ┌─────────────┐  │
│  │  Service (Headless) │                 │ Pod         │  │
│  └─────────────────┘                     └──────┬──────┘  │
│           ▲                                     │         │
│           │                                     │         │
│           │         provides DNS                │         │
│           └─────────────────────────────────────┘         │
│                                                           │
│  ┌─────────────────┐                     ┌─────────────┐  │
│  │ VolumeClaimTemplate │                 │ PVC         │  │
│  └─────────────────┘                     └──────┬──────┘  │
│           ▲                                     │         │
│           │                                     │         │
│           │     references                      │ bound to│
│           └─────────────────────────────────────┘         │
│                                                 │         │
│                                                 ▼         │
│                                         ┌─────────────┐   │
│                                         │ PV          │   │
│                                         └─────────────┘   │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

## StatefulSet Fundamentals

### Basic StatefulSet Configuration

Here's a basic StatefulSet configuration for a simple stateful application:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  # Headless service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
```

### Headless Services for StatefulSets

A headless service (when `clusterIP: None`) is crucial for StatefulSets as it provides DNS entries for each pod:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For example, with the StatefulSet above:
- web-0.nginx.default.svc.cluster.local
- web-1.nginx.default.svc.cluster.local
- web-2.nginx.default.svc.cluster.local

This allows pods to directly communicate with specific instances, which is critical for clustering.

### StatefulSet Pod Management Policies

StatefulSets offer two pod management policies:

1. **OrderedReady** (default): Sequential creation and termination
   ```yaml
   spec:
     podManagementPolicy: OrderedReady
   ```

2. **Parallel**: Parallel creation and termination (but pods still get ordered names)
   ```yaml
   spec:
     podManagementPolicy: Parallel
   ```

Choosing the right policy depends on your application's requirements:
- Use OrderedReady for applications that require strict startup/shutdown order
- Use Parallel for independent instances that can start in any order

### StatefulSet Update Strategies

StatefulSets support different update strategies:

1. **RollingUpdate** (default): Updates pods one at a time, in reverse order
   ```yaml
   spec:
     updateStrategy:
       type: RollingUpdate
       rollingUpdate:
         partition: 0  # All pods will be updated
   ```

2. **OnDelete**: Only update pods when they are manually deleted
   ```yaml
   spec:
     updateStrategy:
       type: OnDelete
   ```

3. **Partitioned Updates**: Update only pods with ordinals >= partition
   ```yaml
   spec:
     updateStrategy:
       type: RollingUpdate
       rollingUpdate:
         partition: 2  # Only pods with index 2 or higher will update
   ```

## Database Deployments in Kubernetes

### Single-Instance Database Patterns

For simpler use cases, a single database instance can be deployed with a StatefulSet:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
      terminationGracePeriodSeconds: 60
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

### Primary-Replica Database Patterns

For databases that support primary-replica replication:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
    role: replica
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
            echo role=replica >> /mnt/conf.d/server-id.cnf
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check if MySQL is ready to accept connections.
            command: ["mysql", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}", "-e", "SELECT 1"]
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
      - name: xtrabackup
        image: percona/percona-xtrabackup:8.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Determine if we need to bootstrap as primary or secondary.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          if [[ $ordinal -eq 0 ]]; then
            # Backup from primary.
            trap "pkill -f 'xtrabackup'; exit 0" SIGTERM
            while true; do
              xtrabackup --backup --stream=xbstream --socket=/var/run/mysqld/mysqld.sock --user=root --password=$MYSQL_ROOT_PASSWORD | ncat --listen --keep-open --send-only 3307
              sleep 2
            done
          else
            # This container just sleeps on secondary.
            trap "exit 0" SIGTERM
            while true; do
              sleep 30
            done
          fi
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
          limits:
            memory: 500Mi
            cpu: 300m
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
data:
  primary.cnf: |
    # Primary configuration
    [mysqld]
    log-bin
    binlog_format=ROW
  replica.cnf: |
    # Replica configuration
    [mysqld]
    super-read-only
    log_bin = mysql-bin
    relay_log = mysql-relay-bin
    log_slave_updates = ON
    read_only = ON
```

### Database Clustering with Custom Resources

Modern Kubernetes deployments often use operators with Custom Resources for database management:

```yaml
# Using PostgreSQL Operator (Zalando)
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: acid-postgresql-cluster
spec:
  teamId: "acid"
  volume:
    size: 10Gi
    storageClass: fast-ssd
  numberOfInstances: 3
  users:
    # database owner
    zalando:
    - superuser
    - createdb
    # application user
    app_user: []
  databases:
    app_db: app_user
  postgresql:
    version: "13"
    parameters:
      shared_buffers: "32MB"
      max_connections: "100"
      log_statement: "all"
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 1Gi
```

Operators handle complex tasks like:
- Automatic failover
- Backup and restore
- Configuration management
- Rolling updates
- Performance tuning

## Messaging and Queue Systems

### Kafka Deployment Best Practices

Deploying Apache Kafka in Kubernetes requires careful planning:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  labels:
    app: kafka
spec:
  ports:
  - port: 9092
    name: kafka
  clusterIP: None
  selector:
    app: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  selector:
    matchLabels:
      app: kafka
  serviceName: kafka-headless
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        app: kafka
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - kafka
            topologyKey: kubernetes.io/hostname
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.0.0
        ports:
        - containerPort: 9092
          name: kafka
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: HOST_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_HEAP_OPTS
          value: -Xms1G -Xmx1G
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper:2181"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless:9092"
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_DEFAULT_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_MIN_INSYNC_REPLICAS
          value: "2"
        - name: KAFKA_AUTO_CREATE_TOPICS_ENABLE
          value: "false"
        - name: KAFKA_LOG_DIRS
          value: /var/lib/kafka/data
        volumeMounts:
        - name: data
          mountPath: /var/lib/kafka/data
        resources:
          requests:
            memory: 2Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
        livenessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 60
          timeoutSeconds: 5
        readinessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 50Gi
```

Key considerations for Kafka:
- Use anti-affinity to distribute brokers across nodes
- Ensure sufficient storage performance (high IOPS)
- Set appropriate replication factors
- Configure heap size based on available memory
- Use node selectors or taints/tolerations for dedicated nodes
- Consider using an operator for complex management tasks

### RabbitMQ Clustering

For RabbitMQ clusters:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-headless
  labels:
    app: rabbitmq
spec:
  ports:
  - port: 5672
    name: amqp
  - port: 15672
    name: http
  clusterIP: None
  selector:
    app: rabbitmq
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  selector:
    matchLabels:
      app: rabbitmq
  serviceName: rabbitmq-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      containers:
      - name: rabbitmq
        image: rabbitmq:3.9-management
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/bash
              - -c
              - |
                until rabbitmqctl node_health_check; do
                  sleep 1
                done
                if [[ "$HOSTNAME" != "rabbitmq-0" ]]; then
                  rabbitmqctl stop_app
                  rabbitmqctl reset
                  rabbitmqctl join_cluster rabbit@rabbitmq-0.rabbitmq-headless
                  rabbitmqctl start_app
                fi
                rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all", "ha-sync-mode":"automatic"}'
        env:
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: erlang-cookie
        - name: RABBITMQ_NODENAME
          value: rabbit@$(HOSTNAME).rabbitmq-headless
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        ports:
        - containerPort: 5672
          name: amqp
        - containerPort: 15672
          name: http
        volumeMounts:
        - name: data
          mountPath: /var/lib/rabbitmq
        resources:
          requests:
            memory: 512Mi
            cpu: 300m
          limits:
            memory: 1Gi
            cpu: 500m
        livenessProbe:
          exec:
            command: ["rabbitmqctl", "status"]
          initialDelaySeconds: 60
          periodSeconds: 60
          timeoutSeconds: 10
        readinessProbe:
          exec:
            command: ["rabbitmqctl", "status"]
          initialDelaySeconds: 20
          periodSeconds: 60
          timeoutSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

## Caching Systems in Kubernetes

### Redis Standalone Deployment

For a single Redis instance:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        command:
        - redis-server
        - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: 512Mi
            cpu: 100m
          limits:
            memory: 1Gi
            cpu: 300m
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
      volumes:
      - name: config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  redis.conf: |
    dir /data
    appendonly yes
    save 900 1
    save 300 10
    save 60 10000
    maxmemory 800mb
    maxmemory-policy allkeys-lru
```

### Redis Sentinel for High Availability

For high availability with Redis Sentinel:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    name: redis
  clusterIP: None
  selector:
    app: redis
---
apiVersion: v1
kind: Service
metadata:
  name: redis-sentinel
  labels:
    app: redis-sentinel
spec:
  ports:
  - port: 26379
    name: sentinel
  clusterIP: None
  selector:
    app: redis-sentinel
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate redis server-id from pod ordinal index.
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          # If index is 0, run as master, otherwise as replica.
          if [[ $ordinal -eq 0 ]]; then
            redis-server /etc/redis/master.conf
          else
            redis-server /etc/redis/replica.conf --slaveof redis-0.redis 6379
          fi
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: 512Mi
            cpu: 100m
          limits:
            memory: 1Gi
            cpu: 300m
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sentinel
spec:
  serviceName: redis-sentinel
  replicas: 3
  selector:
    matchLabels:
      app: redis-sentinel
  template:
    metadata:
      labels:
        app: redis-sentinel
    spec:
      containers:
      - name: sentinel
        image: redis:6.2
        command:
        - redis-sentinel
        - /etc/redis/sentinel.conf
        ports:
        - containerPort: 26379
          name: sentinel
        volumeMounts:
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: 128Mi
            cpu: 50m
          limits:
            memory: 256Mi
            cpu: 100m
      volumes:
      - name: config
        configMap:
          name: redis-sentinel-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  master.conf: |
    dir /data
    appendonly yes
    save 900 1
    save 300 10
    save 60 10000
    maxmemory 800mb
    maxmemory-policy allkeys-lru
  replica.conf: |
    dir /data
    appendonly yes
    save 900 1
    save 300 10
    save 60 10000
    maxmemory 800mb
    maxmemory-policy allkeys-lru
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-sentinel-config
data:
  sentinel.conf: |
    port 26379
    sentinel monitor mymaster redis-0.redis 6379 2
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 60000
    sentinel parallel-syncs mymaster 1
```

### Redis Cluster Mode

For Redis Cluster:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    save 900 1
    save 300 10
    save 60 10000
    maxmemory 800mb
    maxmemory-policy allkeys-lru
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  clusterIP: None
  selector:
    app: redis-cluster
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:6.2
        command: ["redis-server", "/conf/redis.conf"]
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: data
          mountPath: /data
        - name: conf
          mountPath: /conf
        resources:
          requests:
            memory: 512Mi
            cpu: 100m
          limits:
            memory: 1Gi
            cpu: 300m
      - name: redis-cluster-init
        image: redis:6.2
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Wait for all pods to be ready
          until [[ "$(kubectl get pods -l app=redis-cluster -o jsonpath='{.items[*].status.containerStatuses[0].ready}' | tr ' ' '\n' | grep -c true)" == "6" ]]; do
            echo "Waiting for all Redis pods to be ready..."
            sleep 5
          done
          # Check if the cluster is already initialized
          if redis-cli -h redis-cluster-0.redis-cluster cluster info | grep -q 'cluster_state:ok'; then
            echo "Redis Cluster already initialized...exiting"
            exit 0
          fi
          # Initialize the cluster
          echo "Initializing Redis Cluster"
          redis-cli --cluster create \
            redis-cluster-0.redis-cluster:6379 \
            redis-cluster-1.redis-cluster:6379 \
            redis-cluster-2.redis-cluster:6379 \
            redis-cluster-3.redis-cluster:6379 \
            redis-cluster-4.redis-cluster:6379 \
            redis-cluster-5.redis-cluster:6379 \
            --cluster-replicas 1 \
            --cluster-yes
        volumeMounts:
        - name: conf
          mountPath: /conf
      volumes:
      - name: conf
        configMap:
          name: redis-cluster-config
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

## Storage Optimization Strategies

### Storage Performance Tuning

Different storage types offer varying performance characteristics:

| Storage Type | Performance | Use Case | Cost |
|--------------|-------------|----------|------|
| SSD/NVMe | High IOPS, low latency | Databases, high-performance caches | Higher |
| HDD | High throughput, higher latency | Log storage, backups, archives | Lower |
| Memory-backed | Highest performance | Volatile caches, temporary storage | Highest |

Implement storage classes that target specific workloads:

```yaml
# High-performance storage for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# Balanced storage for general workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# High-capacity, lower-performance storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: capacity-hdd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Filesystem Selection for Stateful Apps

Different filesystems offer different performance characteristics:

| Filesystem | Advantages | Disadvantages | Best For |
|------------|------------|---------------|----------|
| ext4 | Mature, well-tested | Limited scalability | General-purpose workloads |
| XFS | Better for large files, journaling | Higher memory usage | Large databases, logging |
| ZFS | Data integrity, compression | Higher CPU overhead | Data integrity critical apps |
| Btrfs | Snapshots, compression | Less mature in some environments | Applications needing snapshots |

Example of configuring the filesystem in the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: xfs  # XFS filesystem
allowVolumeExpansion: true
```

### Volume Expansion and Data Migration

For expanding volumes:

```yaml
# First, enable volume expansion in the StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
allowVolumeExpansion: true

# Then, edit the PVC to request more storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi  # Updated from 10Gi
  storageClassName: expandable-storage
```

For data migration between volumes:

```yaml
# Use a helper pod to migrate data
apiVersion: v1
kind: Pod
metadata:
  name: data-migration
spec:
  containers:
  - name: migrator
    image: debian:bullseye-slim
    command:
    - bash
    - "-c"
    - |
      echo "Starting data migration..."
      rsync -av /source/ /destination/
      echo "Data migration completed"
    volumeMounts:
    - name: source-data
      mountPath: /source
    - name: destination-data
      mountPath: /destination
  volumes:
  - name: source-data
    persistentVolumeClaim:
      claimName: source-pvc
  - name: destination-data
    persistentVolumeClaim:
      claimName: destination-pvc
  restartPolicy: Never
```

## High Availability Patterns

### Multi-AZ Deployment Strategies

For high availability across availability zones:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 3
  # Other settings...
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - mongodb
            topologyKey: "topology.kubernetes.io/zone"
```

### Backup and Restore Integration

Integrate backups with stateful applications:

```yaml
# Using Velero with pre-backup hooks
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  annotations:
    pre.hook.backup.velero.io/container: postgres
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "pg_dump -U postgres -d mydb > /backup/db.sql"]'
    post.hook.backup.velero.io/container: postgres
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo Backup completed"]'
spec:
  containers:
  - name: postgres
    image: postgres:13
    # Other settings...
```

### Automated Failover Solutions

Implement automated failover for high availability:

```yaml
# Using Patroni for PostgreSQL HA
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: patroni
spec:
  serviceName: patroni
  replicas: 3
  template:
    spec:
      containers:
      - name: patroni
        image: patroni/patroni:latest
        env:
        - name: PATRONI_KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: PATRONI_KUBERNETES_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: PATRONI_KUBERNETES_PORTS
          value: '[{"name": "postgresql", "port": 5432}]'
        - name: PATRONI_SUPERUSER_USERNAME
          value: postgres
        - name: PATRONI_SUPERUSER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: patroni-secret
              key: password
        - name: PATRONI_REPLICATION_USERNAME
          value: replicator
        - name: PATRONI_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: patroni-secret
              key: replication-password
        ports:
        - containerPort: 8008
          name: patroni
        - containerPort: 5432
          name: postgresql
        volumeMounts:
        - name: data
          mountPath: /home/postgres/pgdata
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

## Pod Lifecycle Management

### Init Containers for Stateful Apps

Init containers are especially useful for stateful applications:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
spec:
  # Other settings...
  template:
    spec:
      initContainers:
      - name: check-seed-nodes
        image: busybox:1.28
        command:
        - /bin/sh
        - -c
        - |
          echo "Checking for seed nodes..."
          until nslookup cassandra-0.cassandra; do
            echo "Waiting for cassandra-0.cassandra to be available..."
            sleep 2
          done
          echo "Seed nodes are available."
      - name: configure-cassandra
        image: cassandra:3.11
        command:
        - /bin/sh
        - -c
        - |
          # Calculate node ID from pod name
          ORDINAL=${HOSTNAME##*-}
          
          # Configure cassandra.yaml
          cp /etc/cassandra/cassandra.yaml /config/cassandra.yaml
          sed -i "s/seeds: \"127.0.0.1\"/seeds: \"cassandra-0.cassandra\"/" /config/cassandra.yaml
          sed -i "s/listen_address: localhost/listen_address: ${POD_IP}/" /config/cassandra.yaml
          sed -i "s/rpc_address: localhost/rpc_address: ${POD_IP}/" /config/cassandra.yaml
          sed -i "s/endpoint_snitch: SimpleSnitch/endpoint_snitch: GossipingPropertyFileSnitch/" /config/cassandra.yaml
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: config
          mountPath: /config
      containers:
      - name: cassandra
        image: cassandra:3.11
        # Other settings...
        volumeMounts:
        - name: data
          mountPath: /var/lib/cassandra
        - name: config
          mountPath: /etc/cassandra
      volumes:
      - name: config
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

### Graceful Termination for Stateful Apps

Ensure graceful shutdown with proper termination periods:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  # Other settings...
  template:
    spec:
      terminationGracePeriodSeconds: 120  # Allow 2 minutes for shutdown
      containers:
      - name: mysql
        image: mysql:8.0
        # Other settings...
        lifecycle:
          preStop:
            exec:
              command: 
              - /bin/sh
              - -c
              - |
                echo "Starting graceful shutdown of MySQL..."
                # Perform a clean shutdown
                mysqladmin -u root -p${MYSQL_ROOT_PASSWORD} shutdown
                echo "MySQL shutdown complete"
```

### Pod Disruption Budgets

Prevent too many pods from being unavailable at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2  # Always keep at least 2 instances running
  selector:
    matchLabels:
      app: postgres
```

## Security for Stateful Applications

### Secret Management for Databases

Manage database credentials securely:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded "admin"
  password: cGFzc3dvcmQxMjM=  # base64 encoded "password123"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  # Other settings...
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
```

For more secure credential handling, consider using solutions like:
- Kubernetes External Secrets
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault

### Network Policies for Stateful Apps

Restrict traffic to stateful applications:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: application
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - protocol: TCP
      port: 9091  # Metrics endpoint
```

### Volume Encryption

Enable encryption for sensitive data:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
  # Optional: specify KMS key ID
  kmsKeyId: "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
```

## Scaling Stateful Applications

### Horizontal Scaling Considerations

When scaling stateful applications, consider these patterns:

1. **Read Replicas**: Scale reads while maintaining a single write endpoint
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-read
   spec:
     selector:
       app: mysql
       role: replica
     ports:
     - port: 3306
   ```

2. **Sharding**: Distributing data across multiple instances
   ```yaml
   # Example of simple sharding with multiple StatefulSets
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mongodb-shard0
   spec:
     # Shard 0 configuration
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mongodb-shard1
   spec:
     # Shard 1 configuration
   ```

3. **Connection Pooling**: Efficiently manage database connections
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pgbouncer
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: pgbouncer
     template:
       metadata:
         labels:
           app: pgbouncer
       spec:
         containers:
         - name: pgbouncer
           image: edoburu/pgbouncer:1.15.0
           env:
           - name: DB_USER
             valueFrom:
               secretKeyRef:
                 name: pgbouncer-secret
                 key: username
           - name: DB_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: pgbouncer-secret
                 key: password
           - name: DB_HOST
             value: "postgresql"
           - name: DB_PORT
             value: "5432"
           - name: POOL_MODE
             value: "transaction"
           - name: MAX_CLIENT_CONN
             value: "1000"
           - name: DEFAULT_POOL_SIZE
             value: "100"
           ports:
           - containerPort: 5432
   ```

### Vertical Scaling with Resource Management

For vertical scaling, manage resources appropriately:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  # Other settings...
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:13
        resources:
          requests:
            memory: 2Gi
            cpu: 500m
          limits:
            memory: 4Gi
            cpu: 1000m
```

For applications that need to change resource allocations:

1. Set up Vertical Pod Autoscaler (VPA) for automatic scaling:
   ```yaml
   apiVersion: autoscaling.k8s.io/v1
   kind: VerticalPodAutoscaler
   metadata:
     name: postgres-vpa
   spec:
     targetRef:
       apiVersion: "apps/v1"
       kind: StatefulSet
       name: postgres
     updatePolicy:
       updateMode: "Auto"
     resourcePolicy:
       containerPolicies:
       - containerName: postgres
         minAllowed:
           cpu: 100m
           memory: 1Gi
         maxAllowed:
           cpu: 2000m
           memory: 8Gi
   ```

2. For manual vertical scaling, update StatefulSet resources and perform a rolling restart

### Auto-Scaling Stateful Applications

While stateful applications are traditionally more challenging to auto-scale, you can implement certain patterns:

1. **Prometheus-based custom metrics**:
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: kafka-consumer-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: kafka-consumer
     minReplicas: 3
     maxReplicas: 10
     metrics:
     - type: Object
       object:
         metric:
           name: kafka_consumer_lag
         describedObject:
           apiVersion: apps/v1
           kind: StatefulSet
           name: kafka
         target:
           type: Value
           value: 100
   ```

2. **Scheduled scaling** for predictable loads:
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: scale-up-morning
   spec:
     schedule: "0 8 * * 1-5"  # Weekdays at 8am
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: kubectl
               image: bitnami/kubectl:latest
               command:
               - /bin/sh
               - -c
               - kubectl scale statefulset redis --replicas=5
             restartPolicy: OnFailure
   ```

## Monitoring and Observability

### Key Metrics for Stateful Applications

Monitor these essential metrics for stateful applications:

1. **Database-specific metrics**:
   - Query performance
   - Transaction rates
   - Connection counts
   - Buffer usage
   - Lock contention

2. **Storage metrics**:
   - IOPS utilization
   - Storage latency
   - Disk usage
   - Read/write ratios

3. **Application metrics**:
   - Request latency
   - Error rates
   - Throughput
   - Connection pool utilization

### Prometheus Integration for Databases

Example of Prometheus monitoring for PostgreSQL:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-metrics
  labels:
    app: postgres
spec:
  ports:
  - port: 9187
    name: metrics
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  # Other settings...
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:13
        # PostgreSQL container settings...
      - name: postgres-exporter
        image: wrouesnel/postgres_exporter:v0.9.0
        env:
        - name: DATA_SOURCE_NAME
          value: "postgresql://postgres:${POSTGRES_PASSWORD}@localhost:5432/postgres?sslmode=disable"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 9187
          name: metrics
```

### Health Probes for Stateful Applications

Implement appropriate health checks:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  # Other settings...
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        # Other settings...
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["mysql", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}", "-e", "SELECT 1"]
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          exec:
            command: ["mysqladmin", "ping", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 30  # Allow up to 5 minutes (30 x 10s) for startup
```

## Disaster Recovery and Backup Strategies

### Regular Backup Procedures

For MySQL databases:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 1 * * *"  # Daily at 1:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-backup
            image: mysql:8.0
            command:
            - /bin/bash
            - -c
            - |
              # Back up MySQL database
              mysqldump -h mysql-0.mysql --all-databases -u root -p${MYSQL_ROOT_PASSWORD} | gzip > /backup/mysql-$(date +%Y%m%d).sql.gz
              
              # Keep only the last 7 backups
              ls -tp /backup/mysql-*.sql.gz | tail -n +8 | xargs -I {} rm -- {}
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: mysql-backup-pvc
          restartPolicy: OnFailure
```

### Volume Snapshots for Quick Recovery

Use CSI snapshots for point-in-time recovery:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-data-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: postgres-data-postgres-0
```

### Multi-Region Disaster Recovery

For critical applications, implement cross-region replication:

```yaml
# Using AWS S3 cross-region replication for backups
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup-to-s3
spec:
  schedule: "0 2 * * *"  # Daily at 2:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: amazon/aws-cli:latest
            command:
            - /bin/bash
            - -c
            - |
              # Back up database
              pg_dump -h postgres-0.postgres -U postgres -d mydb | gzip > /backup/postgres-$(date +%Y%m%d).sql.gz
              
              # Upload to S3 (will be replicated cross-region via S3 replication)
              aws s3 cp /backup/postgres-$(date +%Y%m%d).sql.gz s3://my-database-backups/
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            emptyDir: {}
          restartPolicy: OnFailure
```

## Conclusion

Running stateful applications in Kubernetes requires careful planning and proper implementation of storage, networking, and scaling strategies. By following the best practices outlined in this chapter, you can successfully deploy, manage, and scale stateful applications in Kubernetes while ensuring high availability, performance, and reliability.

Key takeaways for stateful application management in Kubernetes:

1. Use StatefulSets for applications that require stable network identities and ordered deployment
2. Select appropriate storage solutions based on performance, reliability, and cost requirements
3. Implement proper backup and disaster recovery procedures
4. Configure health checks and monitoring for early problem detection
5. Design for high availability with multi-zone deployments and automated failover
6. Consider using operators for complex stateful applications
7. Implement security best practices for protecting sensitive data
8. Plan scaling strategies based on application architecture and requirements

As Kubernetes continues to mature, managing stateful applications becomes increasingly streamlined. By leveraging the patterns and practices in this chapter, you can confidently run even the most demanding stateful workloads on Kubernetes.

## References

- [Kubernetes StatefulSets Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [Database Operators for Kubernetes](https://operatorhub.io/category/Database)
- [Zalando PostgreSQL Operator](https://github.com/zalando/postgres-operator)
- [Vitess: MySQL Sharding for Kubernetes](https://vitess.io/)
- [Kubernetes SIG Storage](https://github.com/kubernetes/community/tree/master/sig-storage)
- [Patterns for Resilient Architecture](https://kubernetes.io/blog/2018/04/27/patterns-for-resilient-architecture-part-1/)