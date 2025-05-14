# Disaster Recovery in Kubernetes

## Introduction

Disaster Recovery (DR) is an essential aspect of operating production Kubernetes clusters. A well-designed DR strategy ensures business continuity by enabling rapid recovery from catastrophic failures, data corruption, infrastructure outages, or other disruptive events. This document provides a comprehensive guide to implementing disaster recovery practices for Kubernetes environments, covering strategies, tools, and best practices to protect your applications and data.

## Understanding Disaster Recovery Concepts

### Disaster Recovery Fundamentals

Disaster Recovery involves planning and implementing procedures to recover critical IT infrastructure and applications after a disruptive event. For Kubernetes, this means:

1. **Protecting cluster state and configuration**
2. **Preserving application data and configurations**
3. **Ensuring rapid restoration of services**
4. **Minimizing downtime and data loss**

### Key Metrics

Two critical metrics define your disaster recovery strategy:

1. **Recovery Time Objective (RTO)**: Maximum acceptable downtime
   - How quickly must services be restored after a disaster?
   - Ranges from seconds (high availability) to hours/days (basic recovery)

2. **Recovery Point Objective (RPO)**: Maximum acceptable data loss
   - How much data can you afford to lose?
   - Ranges from zero (synchronous replication) to hours/days (daily backups)

![RTO and RPO Diagram](https://mermaid.ink/img/pako:eNp1kctuwjAQRX9l5GWLeIB1V5X6ACEIIZ4bLExiQBbYkR8gVPHvHTtACaiyyTi-Z-6MTxAYQgDQGrpOeSRn41x56_ijroilw-mQnbd5HsxkklUqV0Kt2RY5L6lXSqtshjRsHVvU3CbJe8_dPk6SD2HmyYbkxrKfDKH0vF79UE6HlkXVxVsOtx_aHF9YRrpADb9nLCGzvH0RGq11nImXmFJpHDqRvGpdKR9BxFU4a8QSUlhRB-cTjKfzxXI-PJSvk7OsblhjzQ71K1zcwv1h7T2u41E_HEeDW3jYH5_nSZI-TaeP-xv2lBBrWpO35LoYAawxQADVR3-c9wgvECBC4YxFCGApRfUPLn2Izw?type=png)

### DR Strategies for Kubernetes

There are several DR strategies for Kubernetes, each with different trade-offs:

1. **Backup and Restore**
   - Regular backups of cluster resources and data
   - Restoration to a new cluster when disaster occurs
   - Highest RPO/RTO, but simplest to implement

2. **Pilot Light**
   - Minimal standby environment with critical components
   - Core infrastructure remains running but scaled down
   - Medium RPO/RTO, moderate complexity

3. **Warm Standby**
   - Fully functional replica environment at reduced capacity
   - Constantly updated with data from primary environment
   - Lower RPO/RTO, higher complexity and cost

4. **Hot Standby (Multi-Cluster)**
   - Fully redundant cluster running in parallel with primary
   - Synchronized data and configuration
   - Lowest RPO/RTO, highest complexity and cost

| Strategy | RTO | RPO | Complexity | Cost |
|----------|-----|-----|------------|------|
| Backup & Restore | Hours/Days | Hours/Days | Low | Low |
| Pilot Light | Hours | Minutes/Hours | Medium | Medium |
| Warm Standby | Minutes | Minutes | High | High |
| Hot Standby | Seconds/Minutes | Seconds | Very High | Very High |

## Backup and Restore Strategy

### What to Back Up

A comprehensive Kubernetes backup strategy should include:

1. **Kubernetes Resources**
   - Deployments, StatefulSets, ConfigMaps, Secrets, etc.
   - Custom Resource Definitions (CRDs) and their instances
   - RBAC configurations (Roles, RoleBindings, etc.)

2. **Persistent Data**
   - PersistentVolumes (PVs) and their content
   - Database dumps for stateful applications
   - Object storage data

3. **Cluster Configuration**
   - etcd data
   - Kubernetes control plane configuration
   - Cluster infrastructure settings (networking, security, etc.)

### Kubernetes Resource Backup Tools

#### Velero

[Velero](https://velero.io/) (formerly Heptio Ark) is a widely-used open-source tool for backing up and restoring Kubernetes resources and persistent volumes.

Installation:
```bash
# Install Velero CLI
brew install velero  # macOS
# or
wget https://github.com/vmware-tanzu/velero/releases/download/v1.10.0/velero-v1.10.0-linux-amd64.tar.gz  # Linux

# Install Velero in the cluster with AWS S3 storage
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.6.0 \
  --bucket velero-backups \
  --backup-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true \
  --use-restic
```

Creating backups:
```bash
# Backup entire cluster
velero backup create full-cluster-backup --include-cluster-resources

# Backup specific namespace
velero backup create app-backup --include-namespaces app

# Backup with volume snapshots
velero backup create app-with-volumes --include-namespaces app --snapshot-volumes
```

Restore operations:
```bash
# List available backups
velero backup get

# Restore entire backup
velero restore create --from-backup full-cluster-backup

# Restore to different namespace
velero restore create --from-backup app-backup --namespace-mappings app:app-restore
```

YAML configuration for scheduled backups:
```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-app-backup
  namespace: velero
spec:
  schedule: "0 0 * * *"  # Daily at midnight
  template:
    includedNamespaces:
    - app
    - app-config
    includedResources:
    - deployments
    - services
    - configmaps
    - secrets
    - persistentvolumeclaims
    - persistentvolumes
    labelSelector:
      matchLabels:
        app: critical
    snapshotVolumes: true
    storageLocation: default
    volumeSnapshotLocations:
    - default
    ttl: 720h  # 30 days
```

#### K8up

[K8up](https://k8up.io/) is another backup operator for Kubernetes that uses Restic for data backup:

```yaml
# Install K8up operator (using Helm)
helm repo add k8up-io https://k8up-io.github.io/k8up
helm install k8up k8up-io/k8up

# Example Schedule resource
apiVersion: k8up.io/v1
kind: Schedule
metadata:
  name: app-backup-schedule
spec:
  backend:
    repoPasswordSecretRef:
      name: backup-repo-password
      key: password
    s3:
      endpoint: s3.amazonaws.com
      bucket: k8up-backup
      accessKeyIDSecretRef:
        name: backup-credentials
        key: access-key-id
      secretAccessKeySecretRef:
        name: backup-credentials
        key: secret-access-key
  backup:
    schedule: "0 1 * * *"  # Daily at 1 AM
    keepJobs: 14
  check:
    schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
    keepJobs: 2
  prune:
    schedule: "0 5 * * 0"  # Weekly on Sunday at 5 AM
    keepJobs: 2
    retention:
      keepLast: 5
      keepDaily: 14
      keepWeekly: 5
      keepMonthly: 12
      keepYearly: 2
```

### etcd Backup

The etcd database contains the entire state of your Kubernetes cluster. Backing it up is critical:

```bash
# Get etcd pod name
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Create etcd snapshot
kubectl exec -it $ETCD_POD -n kube-system -- etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot-$(date +%Y%m%d-%H%M%S).db

# Copy snapshot to local machine
kubectl cp kube-system/$ETCD_POD:/var/lib/etcd/snapshot-$(date +%Y%m%d-%H%M%S).db ./etcd-snapshot.db
```

For automated etcd backups, create a CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')
              BACKUP_FILE="etcd-snapshot-$(date +%Y%m%d-%H%M%S).db"
              kubectl exec -it $ETCD_POD -n kube-system -- etcdctl --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key \
                snapshot save /var/lib/etcd/$BACKUP_FILE
              kubectl exec -it $ETCD_POD -n kube-system -- tar czf /var/lib/etcd/$BACKUP_FILE.tar.gz /var/lib/etcd/$BACKUP_FILE
              kubectl cp kube-system/$ETCD_POD:/var/lib/etcd/$BACKUP_FILE.tar.gz /backups/$BACKUP_FILE.tar.gz
              # Upload to object storage (example: AWS S3)
              aws s3 cp /backups/$BACKUP_FILE.tar.gz s3://my-etcd-backups/
            volumeMounts:
            - name: backup-volume
              mountPath: /backups
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          serviceAccountName: etcd-backup-sa
          restartPolicy: OnFailure
```

### Database Backups

For databases running in Kubernetes, you'll need specialized backup approaches:

#### PostgreSQL Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: app
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:14
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_FILE="postgres-$(date +%Y%m%d-%H%M%S).sql.gz"
              pg_dump -h postgres-service -U postgres -d appdb | gzip > /backups/$BACKUP_FILE
              # Upload to cloud storage
              aws s3 cp /backups/$BACKUP_FILE s3://db-backups/postgres/
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backups
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
```

#### MongoDB Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: app
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mongodb-backup
            image: mongo:5
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_DIR="/backups/mongodb-$(date +%Y%m%d-%H%M%S)"
              mkdir -p $BACKUP_DIR
              mongodump --host=mongodb-service --username=admin --password=$MONGO_PASSWORD --authenticationDatabase=admin --out=$BACKUP_DIR
              tar czf $BACKUP_DIR.tar.gz $BACKUP_DIR
              rm -rf $BACKUP_DIR
              # Upload to cloud storage
              aws s3 cp $BACKUP_DIR.tar.gz s3://db-backups/mongodb/
            env:
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backups
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
          restartPolicy: OnFailure
```

### Backup Validation

Regular backup validation is crucial:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-validation
  namespace: backup-system
spec:
  schedule: "0 4 * * 0"  # Weekly on Sunday at 4 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: validation-container
            image: custom/backup-validator:latest
            command:
            - /bin/sh
            - -c
            - |
              # List recent backups
              velero backup get --output json > /tmp/backups.json
              
              # Get most recent backup
              LATEST_BACKUP=$(jq -r '.items | sort_by(.metadata.creationTimestamp) | last | .metadata.name' /tmp/backups.json)
              
              # Create temporary namespace for restore test
              kubectl create namespace backup-validation-test
              
              # Restore backup to test namespace
              velero restore create --from-backup $LATEST_BACKUP --namespace-mappings app:backup-validation-test
              
              # Wait for restore to complete
              while [[ $(velero restore get --output json | jq -r '.items[] | select(.metadata.name=="$LATEST_BACKUP-validation") | .status.phase') != "Completed" ]]; do
                sleep 10
              done
              
              # Run validation tests
              /opt/validate-restore.sh backup-validation-test
              
              # Clean up
              kubectl delete namespace backup-validation-test
              
              # Send results
              if [ $? -eq 0 ]; then
                echo "Backup validation successful" | mail -s "Backup Validation: SUCCESS" admin@example.com
              else
                echo "Backup validation failed" | mail -s "Backup Validation: FAILURE" admin@example.com
              fi
          restartPolicy: OnFailure
```

## Disaster Recovery Implementation

### Pilot Light DR Strategy

In a Pilot Light setup, you maintain a minimal standby environment in a secondary region:

```yaml
# Terraform configuration for Pilot Light DR (simplified)
provider "aws" {
  region = "us-west-2"  # Secondary region
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ... VPC configuration
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  cluster_name = "dr-eks-cluster"
  cluster_version = "1.25"
  # Minimal node configuration for core services
  node_groups = {
    core = {
      desired_capacity = 2
      max_capacity     = 5
      min_capacity     = 2
      instance_types   = ["m5.large"]
    }
  }
}

# S3 Cross-Region Replication for backups
resource "aws_s3_bucket" "dr_backup_bucket" {
  bucket = "dr-backup-bucket"
  region = "us-west-2"
}

resource "aws_s3_bucket_replication_configuration" "backup_replication" {
  bucket = aws_s3_bucket.primary_backup_bucket.id
  
  rule {
    status = "Enabled"
    
    destination {
      bucket = aws_s3_bucket.dr_backup_bucket.arn
    }
  }
}
```

### Warm Standby DR Strategy

A Warm Standby has a fully functional but scaled-down replica of your primary cluster:

```yaml
# Terraform configuration for Warm Standby DR (simplified)
provider "aws" {
  region = "us-west-2"  # Secondary region
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ... VPC configuration similar to primary
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  cluster_name = "dr-eks-cluster"
  cluster_version = "1.25"
  # Similar configuration to primary but with smaller node groups
  node_groups = {
    app = {
      desired_capacity = 2  # Scaled down
      max_capacity     = 10
      min_capacity     = 2
      instance_types   = ["m5.large"]
    }
    data = {
      desired_capacity = 1  # Scaled down
      max_capacity     = 5
      min_capacity     = 1
      instance_types   = ["r5.large"]
    }
  }
}

# Continuous data replication (example for PostgreSQL)
resource "kubernetes_deployment" "postgres_replica" {
  metadata {
    name = "postgres-replica"
    namespace = "app"
  }
  spec {
    replicas = 1
    template {
      spec {
        containers {
          name = "postgres"
          image = "postgres:14"
          env {
            name = "POSTGRES_PASSWORD"
            value_from {
              secret_key_ref {
                name = "postgres-credentials"
                key = "password"
              }
            }
          }
          # ... other configuration
        }
      }
    }
  }
}
```

### Hot Standby DR Strategy

A Hot Standby maintains a full-scale replica of your production environment:

```yaml
# ArgoCD Application to deploy to both clusters
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-cluster-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# Second Application targeting DR cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-cluster-app-dr
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://dr-cluster-api-endpoint
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

For database replication in Hot Standby:

```yaml
# PostgreSQL with streaming replication
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: postgres-cluster
spec:
  teamId: "app"
  volume:
    size: 100Gi
  numberOfInstances: 3
  users:
    app: []
  databases:
    appdb: app
  postgresql:
    version: "14"
    parameters:
      shared_buffers: "1GB"
      wal_level: "logical"
      max_wal_senders: "10"
  patroni:
    synchronous_mode: true
  # Cross-region replication
  standby:
    s3_wal_path: "s3://postgres-wal-archive/cluster/"
```

## DR Testing and Validation

### Regular DR Testing

Implement regular DR testing with automation:

```yaml
# Scheduled DR test job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dr-test
  namespace: dr-testing
spec:
  schedule: "0 2 15 * *"  # 2 AM on the 15th of each month
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: dr-test-sa
          containers:
          - name: dr-test
            image: custom/dr-test:latest
            command:
            - /bin/sh
            - -c
            - |
              # 1. Create isolated test namespace
              kubectl create namespace dr-test-$(date +%Y%m%d)
              
              # 2. Restore latest backup to test namespace
              velero restore create --from-backup latest-backup --namespace-mappings app:dr-test-$(date +%Y%m%d)
              
              # 3. Validate restored applications
              /opt/validate-apps.sh dr-test-$(date +%Y%m%d)
              
              # 4. Test database restoration
              /opt/validate-db.sh dr-test-$(date +%Y%m%d)
              
              # 5. Generate test report
              /opt/generate-report.sh > /reports/dr-test-$(date +%Y%m%d).html
              
              # 6. Clean up
              kubectl delete namespace dr-test-$(date +%Y%m%d)
            volumeMounts:
            - name: reports-volume
              mountPath: /reports
          volumes:
          - name: reports-volume
            persistentVolumeClaim:
              claimName: dr-test-reports-pvc
          restartPolicy: OnFailure
```

### Game Days

Schedule regular "Game Days" to practice DR scenarios:

1. **Prepare a runbook** with specific DR scenarios to test:

```markdown
# Disaster Recovery Game Day Runbook

## Scenario 1: Complete Cluster Failure

### Objective
Test the recovery process when the entire primary cluster becomes unavailable.

### Procedure
1. Simulate cluster failure:
   ```bash
   # For managed clusters (e.g., GKE)
   gcloud container clusters update primary-cluster --no-enable-ip-alias
   
   # For self-managed clusters
   ssh master-node "systemctl stop kubelet"
   ```

2. Initiate DR procedure:
   ```bash
   # Activate DR cluster
   ./scripts/activate-dr-cluster.sh
   
   # Update DNS to point to DR cluster
   ./scripts/update-dns.sh
   
   # Verify application availability
   ./scripts/verify-applications.sh
   ```

3. Measure key metrics:
   - Time to detect failure: __________
   - Time to activate DR cluster: __________
   - Time to restore service: __________
   - RTO achieved: __________
   - RPO achieved: __________

4. Restore primary cluster:
   ```bash
   # Restore primary cluster
   gcloud container clusters update primary-cluster --enable-ip-alias
   
   # Verify primary cluster health
   ./scripts/verify-primary-cluster.sh
   
   # Failback to primary
   ./scripts/failback-to-primary.sh
   ```

## Scenario 2: Database Corruption

### Objective
Test recovery from database corruption in a critical service.

### Procedure
1. Simulate database corruption:
   ```bash
   kubectl exec -it postgres-0 -n app -- psql -U postgres -c "DROP TABLE critical_data;"
   ```

2. Detect and assess the impact:
   ```bash
   ./scripts/verify-database-integrity.sh
   ```

3. Initiate recovery procedure:
   ```bash
   # Restore from latest backup
   ./scripts/restore-database.sh
   
   # Verify data integrity
   ./scripts/verify-data-integrity.sh
   ```

4. Measure key metrics:
   - Time to detect corruption: __________
   - Time to restore database: __________
   - Data loss (if any): __________
   - RPO achieved: __________
```

## Multi-Region and Multi-Cluster Strategies

### Global Load Balancing

Implement global load balancing for multi-region traffic distribution:

```yaml
# GCP global load balancer example with Terraform
resource "google_compute_global_address" "default" {
  name = "global-app-ip"
}

resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-rule"
  target     = google_compute_target_http_proxy.default.id
  port_range = "80"
  ip_address = google_compute_global_address.default.address
}

resource "google_compute_target_http_proxy" "default" {
  name    = "target-proxy"
  url_map = google_compute_url_map.default.id
}

resource "google_compute_url_map" "default" {
  name            = "url-map"
  default_service = google_compute_backend_service.default.id
}

resource "google_compute_backend_service" "default" {
  name        = "backend-service"
  port_name   = "http"
  protocol    = "HTTP"
  timeout_sec = 10
  
  backend {
    group = google_compute_region_instance_group_manager.us_central1.instance_group
  }
  
  backend {
    group = google_compute_region_instance_group_manager.us_east1.instance_group
  }
  
  health_checks = [google_compute_health_check.default.id]
}
```

### Cross-Cluster Service Communication

For services spanning multiple clusters:

```yaml
# Service Mesh implementation (Istio) for cross-cluster services
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
spec:
  profile: default
  meshConfig:
    accessLogFile: /dev/stdout
    enableAutoMtls: true
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
  values:
    global:
      meshID: multi-cluster-mesh
      multiCluster:
        clusterName: cluster-1
      network: network-1

---
# Multicluster Gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-cluster-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
    hosts:
    - "*.local"
```

### Multi-Cluster Data Services

For databases spanning multiple clusters:

```yaml
# CockroachDB multi-region deployment
apiVersion: crdb.cockroachlabs.com/v1alpha1
kind: CrdbCluster
metadata:
  name: cockroachdb
spec:
  dataStore:
    pvc:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: "100Gi"
        volumeMode: Filesystem
  nodes: 9
  tlsEnabled: true
  image:
    name: cockroachdb/cockroach:v22.1.10
  cockroachDBVersion: v22.1.10
  ingress:
    ui:
      annotations:
        kubernetes.io/ingress.class: nginx
  additionalLabels:
    app: cockroachdb
  additionalAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "_status/vars"
    prometheus.io/port: "8080"
  regions:
  - name: us-east1
    nodes: 3
  - name: us-central1
    nodes: 3
  - name: us-west1
    nodes: 3
```

## Automated Failover

### Automated Detection

Implement monitoring for automated failure detection:

```yaml
# Prometheus AlertManager configuration
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-health-rules
  namespace: monitoring
spec:
  groups:
  - name: cluster-health
    rules:
    - alert: ClusterUnhealthy
      expr: sum(up{job="apiserver"}) < 3
      for: 5m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Cluster API server unhealthy"
        description: "Less than 3 API servers are up for more than 5 minutes."
        runbook_url: "https://runbooks.example.com/cluster-failure"
```

### Automated Failover Procedure

Automate failover with a script triggered by monitoring alerts:

```yaml
# Kubernetes Job for automated failover
apiVersion: batch/v1
kind: Job
metadata:
  name: cluster-failover
  namespace: dr
spec:
  template:
    spec:
      serviceAccountName: dr-failover-sa
      containers:
      - name: failover-controller
        image: custom/failover-controller:latest
        command: ["/failover.sh"]
        env:
        - name: PRIMARY_CLUSTER
          value: "gke_project_us-central1_primary-cluster"
        - name: DR_CLUSTER
          value: "gke_project_us-east1_dr-cluster"
        - name: DNS_ZONE
          value: "example.com"
        volumeMounts:
        - name: kubeconfig
          mountPath: /root/.kube
          readOnly: true
        - name: gcp-credentials
          mountPath: /root/.config/gcloud
          readOnly: true
      volumes:
      - name: kubeconfig
        secret:
          secretName: multi-cluster-kubeconfig
      - name: gcp-credentials
        secret:
          secretName: gcp-credentials
      restartPolicy: OnFailure
```

Implementation of `/failover.sh`:

```bash
#!/bin/bash
set -e

echo "Starting failover procedure at $(date)"

# 1. Verify DR cluster is healthy
kubectl --context=$DR_CLUSTER get nodes
if [ $? -ne 0 ]; then
  echo "ERROR: DR cluster not accessible"
  exit 1
fi

# 2. Check latest backup status
LATEST_BACKUP=$(velero --kubeconfig=/root/.kube/config backup get --output json | jq -r '.items | sort_by(.metadata.creationTimestamp) | last | .metadata.name')
BACKUP_STATUS=$(velero --kubeconfig=/root/.kube/config backup get $LATEST_BACKUP --output json | jq -r '.status.phase')

if [ "$BACKUP_STATUS" != "Completed" ]; then
  echo "WARNING: Latest backup status is $BACKUP_STATUS, not Completed"
fi

# 3. Scale up DR cluster if needed
kubectl --context=$DR_CLUSTER scale deployment --all --replicas=3 -n app

# 4. Restore latest backup to DR cluster
velero --kubeconfig=/root/.kube/config restore create --from-backup $LATEST_BACKUP \
  --namespace-mappings '*:*' \
  --include-cluster-resources=true

# 5. Update DNS records
gcloud dns record-sets transaction start --zone=$DNS_ZONE

# Get DR cluster ingress IP
DR_IP=$(kubectl --context=$DR_CLUSTER get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Update A record
gcloud dns record-sets transaction remove --zone=$DNS_ZONE \
  --name="*.example.com." --type=A --ttl=300 "$(dig +short *.example.com)"
  
gcloud dns record-sets transaction add --zone=$DNS_ZONE \
  --name="*.example.com." --type=A --ttl=300 "$DR_IP"

gcloud dns record-sets transaction execute --zone=$DNS_ZONE

# 6. Verify applications are running in DR cluster
for deployment in $(kubectl --context=$DR_CLUSTER get deployments -n app -o jsonpath='{.items[*].metadata.name}'); do
  READY=$(kubectl --context=$DR_CLUSTER get deployment $deployment -n app -o jsonpath='{.status.readyReplicas}')
  REPLICAS=$(kubectl --context=$DR_CLUSTER get deployment $deployment -n app -o jsonpath='{.status.replicas}')
  
  if [ "$READY" != "$REPLICAS" ]; then
    echo "WARNING: Deployment $deployment not fully ready ($READY/$REPLICAS)"
  fi
done

# 7. Run validation tests
/opt/validate-apps.sh

# 8. Send notification
echo "Failover completed at $(date). Cluster switched from $PRIMARY_CLUSTER to $DR_CLUSTER" | \
  mail -s "EMERGENCY: Automated Cluster Failover Executed" team@example.com

echo "Failover procedure completed at $(date)"
```

## DR for Managed Kubernetes Services

### AWS EKS Disaster Recovery

For EKS clusters:

```yaml
# Terraform configuration for multi-region EKS
provider "aws" {
  alias = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias = "dr"
  region = "us-west-2"
}

module "eks_primary" {
  source = "terraform-aws-modules/eks/aws"
  providers = {
    aws = aws.primary
  }
  cluster_name = "primary-eks"
  # ... other configuration
}

module "eks_dr" {
  source = "terraform-aws-modules/eks/aws"
  providers = {
    aws = aws.dr
  }
  cluster_name = "dr-eks"
  # ... other configuration
}

# S3 Cross-Region Replication
resource "aws_s3_bucket" "primary_bucket" {
  provider = aws.primary
  bucket = "app-data-primary"
}

resource "aws_s3_bucket" "dr_bucket" {
  provider = aws.dr
  bucket = "app-data-dr"
}

resource "aws_s3_bucket_replication_configuration" "replication" {
  provider = aws.primary
  bucket = aws_s3_bucket.primary_bucket.id
  
  rule {
    status = "Enabled"
    
    destination {
      bucket = aws_s3_bucket.dr_bucket.arn
      storage_class = "STANDARD"
    }
  }
}

# Route53 Failover Configuration
resource "aws_route53_health_check" "primary" {
  provider = aws.primary
  fqdn = module.eks_primary.cluster_endpoint
  port = 443
  type = "HTTPS"
  resource_path = "/healthz"
  request_interval = 30
  failure_threshold = 3
}

resource "aws_route53_record" "primary" {
  provider = aws.primary
  zone_id = aws_route53_zone.main.zone_id
  name = "api.example.com"
  type = "A"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  set_identifier = "primary"
  health_check_id = aws_route53_health_check.primary.id
  alias {
    name = module.eks_primary.cluster_endpoint
    zone_id = module.eks_primary.cluster_endpoint_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "dr" {
  provider = aws.dr
  zone_id = aws_route53_zone.main.zone_id
  name = "api.example.com"
  type = "A"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  set_identifier = "dr"
  alias {
    name = module.eks_dr.cluster_endpoint
    zone_id = module.eks_dr.cluster_endpoint_zone_id
    evaluate_target_health = true
  }
}
```

### GCP GKE Disaster Recovery

For GKE clusters:

```yaml
# Terraform configuration for GKE DR
provider "google" {
  alias = "primary"
  project = "my-project"
  region  = "us-central1"
}

provider "google" {
  alias = "dr"
  project = "my-project"
  region  = "us-east1"
}

# Primary GKE cluster
resource "google_container_cluster" "primary" {
  provider = google.primary
  name     = "primary-cluster"
  location = "us-central1"
  # ... other configuration
}

# DR GKE cluster
resource "google_container_cluster" "dr" {
  provider = google.dr
  name     = "dr-cluster"
  location = "us-east1"
  # ... other configuration
}

# Cloud SQL with cross-region replication
resource "google_sql_database_instance" "primary" {
  provider = google.primary
  name = "primary-db"
  database_version = "POSTGRES_14"
  region = "us-central1"
  
  settings {
    tier = "db-custom-4-15360"
    availability_type = "REGIONAL"
    
    backup_configuration {
      enabled = true
      point_in_time_recovery_enabled = true
      start_time = "02:00"
      transaction_log_retention_days = 7
    }
  }
}

resource "google_sql_database_instance" "replica" {
  provider = google.dr
  name = "dr-db"
  database_version = "POSTGRES_14"
  region = "us-east1"
  
  master_instance_name = google_sql_database_instance.primary.name
  
  replica_configuration {
    failover_target = true
  }
  
  settings {
    tier = "db-custom-4-15360"
    availability_type = "REGIONAL"
  }
}

# Global load balancer
resource "google_compute_global_address" "default" {
  name = "global-app-ip"
}

resource "google_compute_global_forwarding_rule" "default" {
  name       = "global-rule"
  target     = google_compute_target_http_proxy.default.id
  port_range = "80"
  ip_address = google_compute_global_address.default.address
}

resource "google_compute_url_map" "default" {
  name            = "url-map"
  default_service = google_compute_backend_service.default.id
}

resource "google_compute_backend_service" "default" {
  name        = "backend-service"
  port_name   = "http"
  protocol    = "HTTP"
  timeout_sec = 10
  
  backend {
    group = google_compute_region_instance_group_manager.us_central1.instance_group
  }
  
  backend {
    group = google_compute_region_instance_group_manager.us_east1.instance_group
  }
  
  health_checks = [google_compute_health_check.default.id]
  
  failover_policy {
    disable_connection_drain_on_failover = false
    drop_traffic_if_unhealthy = true
    failover_ratio = 1.0
  }
}
```

### Azure AKS Disaster Recovery

For AKS clusters:

```yaml
# Terraform configuration for AKS DR
provider "azurerm" {
  alias = "primary"
  features {}
}

provider "azurerm" {
  alias = "dr"
  features {}
}

# Primary resource group
resource "azurerm_resource_group" "primary" {
  provider = azurerm.primary
  name     = "primary-rg"
  location = "East US"
}

# DR resource group
resource "azurerm_resource_group" "dr" {
  provider = azurerm.dr
  name     = "dr-rg"
  location = "West US"
}

# Primary AKS cluster
resource "azurerm_kubernetes_cluster" "primary" {
  provider            = azurerm.primary
  name                = "primary-aks"
  location            = azurerm_resource_group.primary.location
  resource_group_name = azurerm_resource_group.primary.name
  dns_prefix          = "primary-aks"
  
  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D2_v2"
  }
  
  identity {
    type = "SystemAssigned"
  }
}

# DR AKS cluster
resource "azurerm_kubernetes_cluster" "dr" {
  provider            = azurerm.dr
  name                = "dr-aks"
  location            = azurerm_resource_group.dr.location
  resource_group_name = azurerm_resource_group.dr.name
  dns_prefix          = "dr-aks"
  
  default_node_pool {
    name       = "default"
    node_count = 2  # Scaled down
    vm_size    = "Standard_D2_v2"
  }
  
  identity {
    type = "SystemAssigned"
  }
}

# Azure Traffic Manager for failover
resource "azurerm_traffic_manager_profile" "app" {
  name                = "app-tm-profile"
  resource_group_name = azurerm_resource_group.primary.name
  
  traffic_routing_method = "Priority"
  
  dns_config {
    relative_name = "app"
    ttl           = 60
  }
  
  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 30
    timeout_in_seconds           = 10
    tolerated_number_of_failures = 3
  }
}

resource "azurerm_traffic_manager_endpoint" "primary" {
  name                = "primary-endpoint"
  resource_group_name = azurerm_resource_group.primary.name
  profile_name        = azurerm_traffic_manager_profile.app.name
  type                = "externalEndpoints"
  target              = azurerm_public_ip.primary_ingress.ip_address
  priority            = 1
}

resource "azurerm_traffic_manager_endpoint" "dr" {
  name                = "dr-endpoint"
  resource_group_name = azurerm_resource_group.primary.name
  profile_name        = azurerm_traffic_manager_profile.app.name
  type                = "externalEndpoints"
  target              = azurerm_public_ip.dr_ingress.ip_address
  priority            = 2
}
```

## Special DR Considerations

### Stateful Applications

Handle stateful applications with care in DR scenarios:

```yaml
# StatefulSet with persistence
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: app
spec:
  serviceName: "postgres"
  replicas: 3
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
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "ssd"
      resources:
        requests:
          storage: 100Gi
```

Implement a backup strategy for stateful data:

```yaml
# Postgres backup cronjob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: app
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:14
            command:
            - /bin/bash
            - -c
            - |
              BACKUP_FILE="postgres-$(date +%Y%m%d-%H%M%S).sql.gz"
              pg_dump -h postgres -U postgres | gzip > /backup/$BACKUP_FILE
              # Copy to long-term storage
              gsutil cp /backup/$BACKUP_FILE gs://app-backups/postgres/
              # Keep only last 10 backups locally
              ls -t /backup/postgres-*.sql.gz | tail -n +11 | xargs rm -f
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
```

For database restore procedures, create a documented script:

```bash
#!/bin/bash
# postgres-restore.sh

# Usage: ./postgres-restore.sh [backup-file]

set -e

if [ -z "$1" ]; then
  echo "Usage: ./postgres-restore.sh [backup-file]"
  echo "Backup files available:"
  gsutil ls gs://app-backups/postgres/
  exit 1
fi

BACKUP_FILE=$1

# Ensure we have a local copy
if [[ $BACKUP_FILE == gs://* ]]; then
  LOCAL_FILE=$(basename $BACKUP_FILE)
  gsutil cp $BACKUP_FILE /tmp/$LOCAL_FILE
  BACKUP_FILE="/tmp/$LOCAL_FILE"
fi

# Get postgres pod name
POD_NAME=$(kubectl get pods -n app -l app=postgres -o jsonpath='{.items[0].metadata.name}')

echo "Copying backup file to pod..."
kubectl cp $BACKUP_FILE app/$POD_NAME:/tmp/backup.sql.gz

echo "Restoring database..."
kubectl exec -it $POD_NAME -n app -- bash -c "gunzip -c /tmp/backup.sql.gz | psql -U postgres"

echo "Verifying database..."
kubectl exec -it $POD_NAME -n app -- psql -U postgres -c "\dt"

echo "Cleanup..."
kubectl exec -it $POD_NAME -n app -- rm /tmp/backup.sql.gz
if [[ $BACKUP_FILE == /tmp/* ]]; then
  rm $BACKUP_FILE
fi

echo "Restore completed successfully."
```

### Secret Management

Ensure secrets are properly managed in DR scenarios:

```yaml
# Using Sealed Secrets for secure GitOps
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-credentials
  namespace: app
spec:
  encryptedData:
    username: AgBy8hC...
    password: AgCtr85...
```

For Vault integration:

```yaml
# Vault integration
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-vault
  namespace: app
---
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
  name: vault
spec:
  size: 3
  image: vault:1.11.3
  bankVaultsImage: banzaicloud/bank-vaults:latest
  
  vaultConfigurerAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
  
  # Vault Pods , Services, and TLS Secret annotations
  vaultAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
  
  # Specify the ServiceAccount where Vault Pods and the configurer/unsealer should run
  serviceAccount: vault
  
  # Specify the PriorityClass for Vault Pods
  priorityClassName: "system-cluster-critical"
  
  # Support for injecting secrets from Vault
  vaultEnvsConfig:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: vault-root
          key: password
  
  # Request an Etcd instance for storage
  etcdSize: 3
  etcdVersion: "3.5.1"
  etcdAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "2379"
  
  serviceType: ClusterIP
  
  resources:
    vault:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 500m
```

For cross-cluster secret replication:

```yaml
# Kubernetes ExternalSecret for accessing AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: app
spec:
  refreshInterval: "15m"
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: database-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/app/database
      property: username
  - secretKey: password
    remoteRef:
      key: prod/app/database
      property: password
```

## Documentation and Procedures

### DR Runbook Template

Create comprehensive DR runbooks:

```markdown
# Disaster Recovery Runbook

## Overview
This runbook provides step-by-step procedures for recovering from various disaster scenarios affecting our Kubernetes infrastructure.

## Contacts and Escalation

| Role | Name | Contact | Escalation Order |
|------|------|---------|------------------|
| Primary On-Call | [Name] | [Phone/Email/Slack] | 1 |
| Secondary On-Call | [Name] | [Phone/Email/Slack] | 2 |
| Platform Lead | [Name] | [Phone/Email/Slack] | 3 |
| CTO | [Name] | [Phone/Email/Slack] | 4 |

## Procedure 1: Full Cluster Recovery

### Prerequisites
- Access to cloud provider console
- `kubectl` configured with multi-cluster contexts
- Access to backup storage
- DNS management access

### Assessment
1. Determine the nature and extent of the outage
2. Confirm primary cluster is unreachable:
   ```bash
   kubectl --context=primary-cluster get nodes
   ```
3. Check DR cluster health:
   ```bash
   kubectl --context=dr-cluster get nodes
   ```

### Recovery Steps
1. **Activate DR Cluster**
   ```bash
   # Scale up DR cluster if running in reduced capacity
   kubectl --context=dr-cluster scale deployment --all --replicas=3 -n app
   ```

2. **Restore from Latest Backup**
   ```bash
   # Find latest successful backup
   velero backup get
   
   # Restore to DR cluster
   velero --context=dr-cluster restore create --from-backup latest-backup \
     --namespace-mappings '*:*' \
     --include-cluster-resources=true
   ```

3. **Verify Critical Services**
   ```bash
   # Check pod status
   kubectl --context=dr-cluster get pods -n app
   
   # Check service endpoints
   kubectl --context=dr-cluster get endpoints -n app
   
   # Verify application health
   curl -k https://internal-service.dr-cluster.svc.cluster.local/health
   ```

4. **Update DNS Records**
   ```bash
   # Get DR cluster ingress IP
   DR_IP=$(kubectl --context=dr-cluster get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   
   # Update A record in Route53/Cloud DNS
   aws route53 change-resource-record-sets --hosted-zone-id Z123456789 \
     --change-batch '{
       "Changes": [
         {
           "Action": "UPSERT",
           "ResourceRecordSet": {
             "Name": "api.example.com",
             "Type": "A",
             "TTL": 300,
             "ResourceRecords": [
               {
                 "Value": "'$DR_IP'"
               }
             ]
           }
         }
       ]
     }'
   ```

5. **Verify External Access**
   ```bash
   # Wait for DNS propagation
   watch dig api.example.com
   
   # Test application functionality
   curl -v https://api.example.com/health
   ```

6. **Notify Stakeholders**
   - Send notification to stakeholders about the recovery
   - Provide status update and any limitations

### Rollback (When Primary Cluster is Restored)
1. Synchronize data from DR to primary cluster
2. Verify primary cluster functionality
3. Update DNS to point back to primary cluster
4. Scale down DR cluster (if operating in reduced-capacity model)

## Procedure 2: Database Recovery
[...]

## Procedure 3: Network Outage Recovery
[...]

## Validation Checklist
After recovery, validate:
- [ ] All critical services are operational
- [ ] Data is intact and consistent
- [ ] External access is functioning
- [ ] Monitoring and alerting is operational
- [ ] Backup systems are functioning

## Lessons Learned
After each DR event:
1. Document what happened
2. What worked well
3. What could be improved
4. Update this runbook with new information
```

### Training Plan for DR Procedures

Create a training plan:

```markdown
# Disaster Recovery Training Plan

## Training Objectives
1. Ensure team members understand DR procedures
2. Practice executing recovery processes
3. Test communication channels during an incident
4. Identify areas for improvement

## Training Schedule
| Training | Frequency | Participants | Duration |
|----------|-----------|--------------|----------|
| DR Orientation | New team members | DevOps team | 2 hours |
| Tabletop Exercise | Quarterly | DevOps & Dev teams | 3 hours |
| Game Day | Semi-annually | All technical staff | Full day |
| Full DR Test | Annually | All staff | 1-2 days |

## Training Scenarios
1. **Complete Cluster Failure**
   - Primary cluster becomes completely unavailable
   - Team must activate DR cluster and restore services

2. **Database Corruption**
   - Critical database experiences data corruption
   - Team must restore from backup and validate data integrity

3. **Networking Failure**
   - Network connectivity between services is lost
   - Team must diagnose and recover network functionality

4. **Security Incident**
   - Potential compromise of cluster requires isolation
   - Team must contain and recover from potential breach

## Training Resources
- DR Documentation
- Access to test environments
- Simulation tools
- Mock monitoring alerts

## Assessment Criteria
- Time to detect issues
- Time to recovery
- Data loss (if any)
- Communication effectiveness
- Documentation completeness

## Continuous Improvement
After each training session:
1. Debrief and gather feedback
2. Document lessons learned
3. Update DR procedures
4. Identify improvement areas
```

## Conclusion

A comprehensive disaster recovery strategy is essential for production Kubernetes deployments. By implementing appropriate backup solutions, creating recovery procedures, and regularly testing your DR plans, you can minimize downtime and data loss during critical incidents.

Key takeaways:

1. **Understand your RTO and RPO requirements** to select the right DR strategy
2. **Implement comprehensive backup solutions** for both Kubernetes resources and application data
3. **Consider multi-region and multi-cluster architectures** for critical workloads
4. **Automate detection and recovery where possible** to minimize downtime
5. **Document and test your DR procedures regularly** to ensure preparedness
6. **Train your team** on DR procedures and tools

With these practices in place, you can ensure business continuity even when facing significant infrastructure disruptions.

## Additional Resources

- [Kubernetes Disaster Recovery](https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/)
- [Velero Documentation](https://velero.io/docs/)
- [Multi-Cluster Management](https://kubernetes.io/docs/concepts/architecture/multi-cluster/)
- [CNCF Disaster Recovery Working Group](https://github.com/cncf/tag-app-delivery/blob/main/working-groups/disaster-recovery.md)
- [Google Cloud Disaster Recovery Planning Guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [AWS Disaster Recovery of Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Azure Kubernetes Service (AKS) Business Continuity and Disaster Recovery](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region)