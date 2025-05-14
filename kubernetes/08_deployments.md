# Kubernetes Deployments

Deployments are one of the most common and powerful Kubernetes resources, providing declarative updates for Pods and ReplicaSets.

## What is a Deployment?

A Deployment provides declarative updates for Pods and ReplicaSets, allowing you to:

- Describe a desired state in a Deployment
- Change the actual state to the desired state at a controlled rate
- Define strategies for transitioning between states
- Roll back to previous Deployment versions

## Deployment Use Cases

Common scenarios where you'll use Deployments:

- Deploy a stateless application
- Update Pods' specifications (container image, resources, etc.)
- Rollback to a previous revision
- Scale the number of replicas up or down
- Pause and resume updates
- Implement advanced deployment strategies (blue/green, canary)

## Creating a Deployment

### Simple Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

Create the Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

## Deployment Concepts

### ReplicaSets

A Deployment manages a ReplicaSet, which in turn manages the Pods. When you update the Deployment, a new ReplicaSet is created, and pods are gradually moved to the new ReplicaSet.

### Pod Template

The pod template in a Deployment defines:
- Container images and versions
- Environment variables 
- Volume mounts
- Resource requests and limits

### Selectors

Selectors define which pods are managed by the Deployment. The pod template labels must match the selector.

## Managing Deployments

### Checking Deployment Status

```bash
# Get basic deployment information
kubectl get deployments

# Get detailed deployment information
kubectl describe deployment nginx-deployment

# Check the rollout status
kubectl rollout status deployment/nginx-deployment
```

### Updating a Deployment

Update the image:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

Or edit the Deployment:

```bash
kubectl edit deployment/nginx-deployment
```

Or update the YAML file and apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

### Scaling a Deployment

```bash
# Scale to 10 replicas
kubectl scale deployment/nginx-deployment --replicas=10

# Autoscale based on CPU
kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=80
```

### Rolling Back a Deployment

```bash
# View rollout history
kubectl rollout history deployment/nginx-deployment

# View details of a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### Pausing and Resuming

```bash
# Pause a deployment rollout
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes
kubectl set resources deployment/nginx-deployment -c nginx --limits=cpu=200m,memory=512Mi
kubectl set image deployment/nginx-deployment nginx=nginx:1.17.1

# Resume the rollout (all changes are applied together)
kubectl rollout resume deployment/nginx-deployment
```

## Deployment Strategies

The `.spec.strategy` field determines how an old Deployment is replaced with a new one.

### Recreate Strategy

All existing Pods are killed before new ones are created:

```yaml
spec:
  strategy:
    type: Recreate
```

Use when applications:
- Cannot handle multiple versions running simultaneously 
- Require a full restart
- Cannot support rolling updates

### Rolling Update Strategy (Default)

Gradually replaces old Pods with new ones:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
```

Parameters:
- `maxUnavailable`: Maximum number of Pods that can be unavailable during update (absolute or percentage)
- `maxSurge`: Maximum number of Pods that can be created over desired number of Pods (absolute or percentage)

### Blue/Green Deployment (Not Native)

Run two versions side by side and switch traffic all at once:

1. Create a new Deployment with a different name or label
2. Wait until it's ready
3. Switch the Service selector to the new Deployment
4. Delete the old Deployment when no longer needed

```yaml
# Original deployment (blue)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  selector:
    matchLabels:
      app: myapp
      version: blue

---
# New deployment (green)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  selector:
    matchLabels:
      app: myapp
      version: green

---
# Service points to blue initially
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
```

### Canary Deployment (Not Native)

Release to a subset of users, then expand:

1. Keep existing Deployment running
2. Create a new Deployment with the new version (fewer replicas)
3. Create/update Service that targets both Deployments
4. Gradually scale up new Deployment and scale down old one

```yaml
# Original deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable

---
# Canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1  # 10% of traffic
  selector:
    matchLabels:
      app: myapp
      version: canary

---
# Service targeting both deployments
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp  # matches both versions
  ports:
  - port: 80
```

## Advanced Deployment Features

### Pod Disruption Budgets

Define minimum available or maximum unavailable Pods during voluntary disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2  # or maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
```

### Deployment Hooks

Use init containers or lifecycle hooks for pre/post actions:

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: init-myservice
        image: busybox:1.28
        command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
      containers:
      - name: nginx
        image: nginx:1.14.2
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo Deployment started > /usr/share/message"]
          preStop:
            exec:
              command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

### Specific Node Deployments

Control which nodes your Pods get deployed to:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
      # Alternative: more flexible node affinity
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/e2e-az-name
                operator: In
                values:
                - e2e-az1
                - e2e-az2
```

## Deployment Best Practices

1. **Set resource requests and limits**:
   ```yaml
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"
       cpu: "500m"
   ```

2. **Add readiness and liveness probes**:
   ```yaml
   readinessProbe:
     httpGet:
       path: /ready
       port: 8080
     initialDelaySeconds: 10
     periodSeconds: 5
   livenessProbe:
     httpGet:
       path: /health
       port: 8080
     initialDelaySeconds: 15
     periodSeconds: 20
   ```

3. **Define update strategies carefully**:
   - Conservative: `maxSurge: 1, maxUnavailable: 0`
   - Balanced: `maxSurge: 25%, maxUnavailable: 25%`
   - Fast: `maxSurge: 50%, maxUnavailable: 0`

4. **Add labels and annotations**:
   ```yaml
   metadata:
     labels:
       app: nginx
       tier: frontend
       environment: production
     annotations:
       kubernetes.io/change-cause: "Update to version 1.16.1"
   ```

5. **Set a revision history limit**:
   ```yaml
   spec:
     revisionHistoryLimit: 10  # Default is 10
   ```

6. **Configure deployment progress deadlines**:
   ```yaml
   spec:
     progressDeadlineSeconds: 600  # Default is 600 (10 minutes)
   ```

7. **Add Pod disruption budgets** for production deployments

8. **Use init containers** for setup operations

9. **Set appropriate termination grace periods**:
   ```yaml
   spec:
     template:
       spec:
         terminationGracePeriodSeconds: 60  # Default is 30
   ```

10. **Consider stability during rollouts**:
    - Ensure new pods are fully ready before reducing old pods
    - Add appropriate readiness probe delays

## Troubleshooting Deployments

### Common Issues

1. **Deployment not progressing**:
   ```bash
   kubectl rollout status deployment/nginx-deployment
   kubectl describe deployment nginx-deployment
   ```
   Look for:
   - Image pull errors
   - Resource constraints (not enough CPU/memory)
   - Node selector not matching any nodes
   - Readiness probe failures

2. **Pods crashing or in CrashLoopBackOff**:
   ```bash
   kubectl get pods -l app=nginx
   kubectl logs <pod-name>
   kubectl describe pod <pod-name>
   ```

3. **Rollback not working**:
   ```bash
   kubectl rollout history deployment/nginx-deployment
   ```
   Check for:
   - Revision history limit reached
   - Invalid revision number

### Deployment Debugging

```bash
# Get detailed deployment status
kubectl get deployment nginx-deployment -o yaml

# Get deployment conditions
kubectl get deployment nginx-deployment -o jsonpath='{.status.conditions}'

# Check events related to deployment
kubectl get events --field-selector involvedObject.name=nginx-deployment

# See ReplicaSets managed by the deployment
kubectl get rs -l app=nginx
```

## Real-World Examples

### Web Application Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
    tier: frontend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
        tier: frontend
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: api_url
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: kubernetes.io/hostname
```

By understanding Kubernetes Deployments, you can efficiently manage the lifecycle of your applications, implement sophisticated deployment strategies, and ensure high availability while minimizing disruptions during updates.