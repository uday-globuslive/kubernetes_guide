# Jobs and CronJobs

Kubernetes Jobs and CronJobs are designed to run tasks that complete over time, rather than running continuously like typical long-running services. Jobs run a task to completion, while CronJobs run Jobs on a time-based schedule.

## Kubernetes Jobs

A Job creates one or more Pods to perform a task and tracks the successful completion of those Pods. When the specified number of successful completions is reached, the Job is complete.

### Use Cases for Jobs

- Batch processing
- Data migrations
- Database backups
- Data analysis tasks
- File processing
- Report generation
- Email sending
- CI/CD build tasks
- Scheduled cleanup jobs

### Basic Job Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

Create the Job:

```bash
kubectl apply -f job.yaml
```

### Job Specifications

#### Required Fields

- `spec.template`: The Pod template defining the container(s) to run
- `spec.template.spec.restartPolicy`: Must be `Never` or `OnFailure` (default is `Always` but not valid for Jobs)

#### Optional Fields

- `spec.completions`: Number of Pods that should successfully complete (default: 1)
- `spec.parallelism`: Number of Pods that can run in parallel (default: 1)
- `spec.backoffLimit`: Number of retries before marking the Job as failed (default: 6)
- `spec.activeDeadlineSeconds`: Maximum duration the Job can run before being terminated
- `spec.ttlSecondsAfterFinished`: Time to keep the Job after it finishes
- `spec.podFailurePolicy`: Defines how to handle Pod failures (beta feature)

### Parallel Jobs

There are three main types of parallel Jobs:

1. **Non-parallel Jobs**: Only one Pod is started (default)
   ```yaml
   spec:
     completions: 1
     parallelism: 1
   ```

2. **Fixed Completion Count**: Multiple Pods run with a fixed completion count
   ```yaml
   spec:
     completions: 5  # Run 5 Pods to completion
     parallelism: 2  # Run up to 2 Pods at a time
   ```

3. **Work Queue**: Multiple pods process items from a work queue
   ```yaml
   spec:
     completions: 1     # Only one job completion is needed
     parallelism: 5     # Run up to 5 worker Pods
   ```
   For work queue jobs, your application needs to manage the work queue.

### Handling Job Failures

Control pod failure handling with backoff limits and deadlines:

```yaml
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 600
```

For more granular control over failure handling:

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: main-container
        operator: In
        values: [1, 2, 3]
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget
```

### Managing Jobs

Check Job status:

```bash
kubectl get jobs
kubectl describe job pi-calculator
```

View pods created by a Job:

```bash
kubectl get pods --selector=job-name=pi-calculator
```

Get logs from a Job:

```bash
kubectl logs -l job-name=pi-calculator
```

Delete a Job (and its Pods):

```bash
kubectl delete job pi-calculator
```

### Automatically Cleaning Up Finished Jobs

Use `ttlSecondsAfterFinished` to automatically clean up Jobs after completion:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-example
spec:
  ttlSecondsAfterFinished: 100  # Delete 100 seconds after completion
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Job completed successfully"]
      restartPolicy: Never
```

## CronJobs

A CronJob creates Jobs on a time-based schedule, using cron syntax.

### Use Cases for CronJobs

- Scheduled backups
- Report generation
- Periodic data processing
- Email newsletters
- Cleanup tasks
- Log rotation
- Metrics collection
- Health checks
- Data synchronization

### Basic CronJob Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"  # Run every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "Hello from the Kubernetes cluster"]
          restartPolicy: OnFailure
```

Create the CronJob:

```bash
kubectl apply -f cronjob.yaml
```

### CronJob Specifications

#### Required Fields

- `spec.schedule`: Cron schedule syntax (see below)
- `spec.jobTemplate`: Job template to run on the schedule

#### Optional Fields

- `spec.timeZone`: IANA time zone (e.g., "Europe/Paris", "America/New_York")
- `spec.startingDeadlineSeconds`: Deadline to start a Job if missed
- `spec.concurrencyPolicy`: How to handle concurrent executions
- `spec.suspend`: Boolean to suspend subsequent executions
- `spec.successfulJobsHistoryLimit`: How many successful Jobs to keep (default: 3)
- `spec.failedJobsHistoryLimit`: How many failed Jobs to keep (default: 1)

### Cron Schedule Syntax

The schedule field uses standard cron syntax:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

Common examples:
- `0 * * * *`: Every hour at minute 0
- `*/15 * * * *`: Every 15 minutes
- `0 0 * * *`: Daily at midnight
- `0 0 * * 0`: Weekly on Sunday at midnight
- `0 0 1 * *`: Monthly on the 1st at midnight
- `0 0 1 1 *`: Yearly on January 1st at midnight
- `30 20 * * 1-5`: Weekdays at 8:30 PM

### Concurrency Policies

Control how concurrent executions are handled:

```yaml
spec:
  concurrencyPolicy: Forbid  # Allow, Forbid, or Replace
```

Options:
- `Allow` (default): Allow concurrent Jobs to run
- `Forbid`: Skip the new Job if the previous one is still running
- `Replace`: Cancel the currently running Job and start a new one

### Time Zones

Specify a time zone for the schedule:

```yaml
spec:
  schedule: "0 6 * * *"  # Daily at 6 AM
  timeZone: "America/New_York"  # In the Eastern Time zone
```

### Managing CronJobs

List CronJobs:

```bash
kubectl get cronjobs
```

Describe a CronJob:

```bash
kubectl describe cronjob hello
```

View the Jobs created by a CronJob:

```bash
kubectl get jobs --selector=job-name=hello-27867840
```

Manually trigger a CronJob:

```bash
kubectl create job --from=cronjob/hello manual-hello
```

Suspend a CronJob:

```bash
kubectl patch cronjob hello -p '{"spec": {"suspend": true}}'
```

Delete a CronJob:

```bash
kubectl delete cronjob hello
```

## Real-World Examples

### Database Backup Job

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  timeZone: "UTC"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  concurrencyPolicy: Forbid
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
              set -e
              BACKUP_FILE="/backup/db-$(date +%Y%m%d-%H%M%S).sql"
              pg_dump -h ${POSTGRES_HOST} -U ${POSTGRES_USER} -d ${POSTGRES_DB} > ${BACKUP_FILE}
              gzip ${BACKUP_FILE}
              echo "Backup completed: ${BACKUP_FILE}.gz"
            env:
            - name: POSTGRES_HOST
              value: postgres-service
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
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
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### Large Data Processing Job with Parallelism

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 50        # Process 50 batches
  parallelism: 10        # Run 10 pods in parallel
  backoffLimit: 5
  activeDeadlineSeconds: 3600  # Timeout after 1 hour
  template:
    spec:
      containers:
      - name: data-processor
        image: data-processing-app:latest
        command: ["python", "process.py", "--batch-id=$(JOB_COMPLETION_INDEX)"]
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        volumeMounts:
        - name: data-volume
          mountPath: /data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-processing-pvc
      restartPolicy: OnFailure
```

### Log Cleanup CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 1 * * *"  # Run at 1 AM daily
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: log-cleaner
            image: busybox:1.28
            command:
            - /bin/sh
            - -c
            - |
              echo "Starting log cleanup job"
              find /var/log -type f -name "*.log" -mtime +7 -delete
              find /var/log -type f -name "*.gz" -mtime +30 -delete
              echo "Log cleanup completed"
            volumeMounts:
            - name: log-volume
              mountPath: /var/log
          volumes:
          - name: log-volume
            hostPath:
              path: /var/log
          restartPolicy: OnFailure
```

### Email Report Generation

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
spec:
  schedule: "0 9 * * 1"  # 9 AM every Monday
  timeZone: "Europe/London"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: reporting-tool:v1.2
            command: ["python", "/app/generate_report.py", "--weekly", "--send-email"]
            env:
            - name: REPORT_RECIPIENTS
              value: "team@example.com,managers@example.com"
            - name: DB_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: reporting-db-credentials
                  key: connection-string
            - name: SMTP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: email-credentials
                  key: password
          restartPolicy: OnFailure
```

## Best Practices

### For Jobs

1. **Set appropriate backoff limits**
   ```yaml
   backoffLimit: 6  # Default is 6
   ```

2. **Set job timeouts to prevent infinite hangs**
   ```yaml
   activeDeadlineSeconds: 600  # 10 minutes
   ```

3. **Use automatic cleanup**
   ```yaml
   ttlSecondsAfterFinished: 86400  # 1 day
   ```

4. **Set resource requests and limits**
   ```yaml
   resources:
     requests:
       memory: "256Mi"
       cpu: "100m"
     limits:
       memory: "512Mi"
       cpu: "500m"
   ```

5. **Add labels for organization**
   ```yaml
   metadata:
     labels:
       app: data-processor
       type: batch
       tier: backend
   ```

6. **Use podFailurePolicy for deterministic failures**
   ```yaml
   podFailurePolicy:
     rules:
     - action: FailJob
       onExitCodes:
         containerName: main
         operator: In
         values: [1, 2, 3]
   ```

7. **Add monitoring and logging**
   - Ensure stdout/stderr is captured
   - Add appropriate tags for log aggregation

### For CronJobs

1. **Set history limits**
   ```yaml
   successfulJobsHistoryLimit: 3
   failedJobsHistoryLimit: 1
   ```

2. **Use concurrency policies consciously**
   ```yaml
   concurrencyPolicy: Forbid  # Prevent overlapping runs
   ```

3. **Set time zones explicitly**
   ```yaml
   timeZone: "UTC"  # Be explicit about time zone
   ```

4. **Add a startingDeadlineSeconds for critical jobs**
   ```yaml
   startingDeadlineSeconds: 300  # Must start within 5 minutes of scheduled time
   ```

5. **Set appropriate Pod settings**
   ```yaml
   jobTemplate:
     spec:
       template:
         spec:
           restartPolicy: OnFailure
           terminationGracePeriodSeconds: 60
   ```

6. **Include alert mechanisms**
   - Emit events on failure
   - Add alerting annotations

## Troubleshooting

### Common Job Issues

1. **Job never completes**
   - Check pod status: `kubectl get pods --selector=job-name=<job-name>`
   - Check logs: `kubectl logs --selector=job-name=<job-name>`
   - Verify resources are adequate
   - Check for hanging processes

2. **Job keeps failing**
   - Check exit codes: `kubectl get pods --selector=job-name=<job-name> -o jsonpath='{.items[*].status.containerStatuses[0].state.terminated.exitCode}'`
   - Check events: `kubectl get events --field-selector involvedObject.name=<pod-name>`
   - Review container logs

3. **Job doesn't clean up**
   - Check TTL setting
   - Review job controller logs
   - Check job status: `kubectl get job <job-name> -o yaml`

### Common CronJob Issues

1. **CronJob not running on schedule**
   - Check cluster controller manager
   - Verify schedule syntax
   - Check if suspended: `kubectl get cronjob <name> -o jsonpath='{.spec.suspend}'`
   - Check startingDeadlineSeconds

2. **CronJob running at unexpected times**
   - Verify time zone setting
   - Check cluster time sync
   - Review cron syntax

3. **Too many or too few Jobs kept**
   - Check history limits
   - Review controller behavior

### Useful Debug Commands

```bash
# Verify cron format
kubectl create cronjob test --image=busybox --schedule="*/1 * * * *" --dry-run=client -o yaml

# Check CronJob last schedule time
kubectl get cronjob <name> -o jsonpath='{.status.lastScheduleTime}'

# Check Job completion status
kubectl get jobs --selector=job-name=<job-name> -o jsonpath='{.items[0].status.conditions[?(@.type=="Complete")].status}'

# Get Job duration
kubectl get job <job-name> -o jsonpath='{.status.completionTime}{"\n"}{.status.startTime}'
```

Jobs and CronJobs are powerful Kubernetes resources for running batch processes, scheduled tasks, and other non-continuous workloads. By leveraging their features and following best practices, you can effectively automate routine tasks and batch processing in your Kubernetes environment.