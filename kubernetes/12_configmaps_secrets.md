# ConfigMaps and Secrets

ConfigMaps and Secrets are Kubernetes resources that allow you to decouple configuration from container images. This separation enables you to create portable and flexible applications that can be easily deployed across different environments.

## ConfigMaps

ConfigMaps store configuration data as key-value pairs. This data can be consumed by pods in various ways, including environment variables, command-line arguments, and configuration files.

### Creating ConfigMaps

#### From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

#### From Files

```bash
# Create a properties file
echo "app.env=production" > app.properties
echo "app.debug=false" >> app.properties
echo "app.port=8080" >> app.properties

# Create ConfigMap from the file
kubectl create configmap app-config --from-file=app.properties
```

#### From Multiple Files

```bash
# Create ConfigMap from multiple files
kubectl create configmap app-config \
  --from-file=app.properties \
  --from-file=database.properties \
  --from-file=cache.properties
```

#### Using YAML Definition

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_DEBUG: "false"
  APP_PORT: "8080"
  app.properties: |
    app.name=MyApp
    app.version=1.0.0
    app.description=My Application Description
  nginx.conf: |
    server {
      listen 80;
      server_name myapp.example.com;
      
      location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
```

Create the ConfigMap:

```bash
kubectl apply -f configmap.yaml
```

### Using ConfigMaps in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: APP_DEBUG
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_DEBUG
    - name: APP_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_PORT
```

#### All ConfigMap Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config
```

#### As Command-Line Arguments

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    command: ["/bin/sh", "-c"]
    args: ["echo $(APP_ENV) && java -jar /app.jar"]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
```

#### As Configuration Files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

With this configuration, each key in the ConfigMap becomes a file in the `/etc/config` directory. For example, `/etc/config/APP_ENV` would contain "production".

#### Mounting Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: nginx.conf
        path: default.conf
```

This mounts only the `nginx.conf` key from the ConfigMap as `/etc/nginx/conf.d/default.conf`.

### Managing ConfigMaps

```bash
# List ConfigMaps
kubectl get configmaps

# View ConfigMap details
kubectl describe configmap app-config

# Edit a ConfigMap
kubectl edit configmap app-config

# Delete a ConfigMap
kubectl delete configmap app-config
```

## Secrets

Secrets are similar to ConfigMaps but are specifically designed to hold sensitive information such as passwords, OAuth tokens, and SSH keys. Kubernetes helps protect this data by:

- Not writing Secret data to disk (stored in memory)
- Encoding data in base64 (but not encrypted by default)
- Restricting access via RBAC
- Encrypting etcd storage (if configured)

### Secret Types

- `Opaque`: Arbitrary user-defined data (default)
- `kubernetes.io/service-account-token`: Service account token
- `kubernetes.io/dockerconfigjson`: Docker registry credentials
- `kubernetes.io/tls`: TLS data
- `kubernetes.io/ssh-auth`: SSH authentication
- `kubernetes.io/basic-auth`: Basic authentication

### Creating Secrets

#### From Literal Values

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

#### From Files

```bash
# Create files with sensitive data
echo -n "admin" > ./username.txt
echo -n "supersecret" > ./password.txt

# Create Secret from files
kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

#### Using YAML Definition

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded "admin"
  password: c3VwZXJzZWNyZXQ=  # base64 encoded "supersecret"
```

For the `data` field, values must be base64-encoded:

```bash
echo -n "admin" | base64        # Outputs: YWRtaW4=
echo -n "supersecret" | base64  # Outputs: c3VwZXJzZWNyZXQ=
```

Alternatively, you can use the `stringData` field to provide unencoded strings:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: supersecret
```

Create the Secret:

```bash
kubectl apply -f secret.yaml
```

#### Creating TLS Secrets

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=example.com"

# Create Secret of type TLS
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

#### Creating Docker Registry Secrets

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=username \
  --docker-password=password \
  --docker-email=email@example.com
```

### Using Secrets in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

#### All Secret Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    envFrom:
    - secretRef:
        name: db-credentials
```

#### As Mounted Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

With this configuration, each key in the Secret becomes a file in the `/etc/secrets` directory. For example, `/etc/secrets/username` would contain "admin".

#### Using Docker Registry Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: private-container
    image: private-registry.example.com/myapp:1.0
  imagePullSecrets:
  - name: regcred
```

#### Using TLS Secrets in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Managing Secrets

```bash
# List Secrets
kubectl get secrets

# View Secret details (without showing values)
kubectl describe secret db-credentials

# View Secret values
kubectl get secret db-credentials -o jsonpath="{.data.username}" | base64 --decode
kubectl get secret db-credentials -o jsonpath="{.data.password}" | base64 --decode

# Edit a Secret
kubectl edit secret db-credentials

# Delete a Secret
kubectl delete secret db-credentials
```

## Best Practices

### For ConfigMaps

1. **Group related configuration**
   - Group related configuration items into a single ConfigMap
   - Separate unrelated configurations into different ConfigMaps

2. **Use meaningful names**
   - Name ConfigMaps according to their purpose: `app-config`, `nginx-config`, etc.

3. **Version your ConfigMaps**
   - Add version annotations to track changes
   - Consider using immutable ConfigMaps for production

4. **Keep ConfigMaps small**
   - Avoid very large ConfigMaps (>1MB)
   - Split large configurations into multiple ConfigMaps

5. **Use subPaths for critical files**
   ```yaml
   volumeMounts:
   - name: config-volume
     mountPath: /etc/nginx/nginx.conf
     subPath: nginx.conf
   ```

6. **Prefer environment variables for simple values**
   - Use environment variables for simple key-value pairs
   - Use volume mounts for complex or multi-line configurations

### For Secrets

1. **Limit access with RBAC**
   - Create specific roles and service accounts for Secret access
   - Follow the principle of least privilege

2. **Enable etcd encryption**
   - Configure Kubernetes to encrypt Secret data at rest
   ```yaml
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

3. **Use external secret management**
   - Consider using Vault, AWS Secrets Manager, GCP Secret Manager, etc.
   - Use Kubernetes External Secrets operator for integration

4. **Don't commit Secrets to source control**
   - Generate Secrets at deployment time
   - Use a CI/CD pipeline with secure storage for Secrets

5. **Regularly rotate credentials**
   - Implement a process for rotating Secret data
   - Use short-lived credentials where possible

6. **Minimize Secret usage in Pod specs**
   - Mount only the Secrets a Pod needs
   - Use specific keys instead of entire Secrets

7. **Use Secret Immutability**
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: db-credentials
   immutable: true
   ```

8. **Set restrictive file permissions**
   ```yaml
   volumes:
   - name: secret-volume
     secret:
       secretName: db-credentials
       defaultMode: 0400  # Read-only by owner
   ```

## Advanced Patterns

### Dynamic Updates

ConfigMap and Secret updates are eventually propagated to Pods:

- **Volume mounts**: Updated automatically (may take up to several minutes)
- **Environment variables**: Not updated without Pod restart

To handle dynamic updates:

1. **Use volume mounts for configurations that need to be updated**

2. **Implement file watchers in your application**
   ```java
   // Java example with WatchService
   WatchService watchService = FileSystems.getDefault().newWatchService();
   Path path = Paths.get("/etc/config");
   path.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);
   ```

3. **Add tools like Reloader** to automatically restart deployments when ConfigMaps or Secrets change

### Immutable ConfigMaps and Secrets

Kubernetes 1.19+ supports immutable ConfigMaps and Secrets:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
immutable: true
data:
  APP_ENV: production
```

Benefits:
- Prevents accidental updates
- Improves performance (no need to watch for changes)
- Enforces versioning and proper change management

### Environment-Specific Configurations

Use Kustomize to manage environment-specific configurations:

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
configMapGenerator:
- name: app-config
  literals:
  - APP_ENV=base
```

```yaml
# overlays/production/kustomization.yaml
bases:
- ../../base
configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - APP_ENV=production
```

### External Secret Management

Use External Secrets Operator to integrate with external secret providers:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: username
    remoteRef:
      key: database/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: database/credentials
      property: password
```

### ConfigMap and Secret Projections

Project multiple ConfigMaps and Secrets into a single directory:

```yaml
volumes:
- name: config
  projected:
    sources:
    - configMap:
        name: app-config
    - secret:
        name: app-secrets
    - downwardAPI:
        items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
```

## Real-World Examples

### Web Application with Frontend and Backend

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  config.js: |
    window.config = {
      apiUrl: '/api',
      features: {
        darkMode: true,
        analytics: true
      },
      version: '1.2.3'
    };
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  application.properties: |
    server.port=8080
    spring.application.name=backend-service
    spring.profiles.active=production
    management.endpoints.web.exposure.include=health,info,metrics
    logging.level.root=INFO
    logging.level.com.example=DEBUG
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
type: Opaque
stringData:
  db-url: jdbc:postgresql://db-service:5432/myapp
  db-username: app_user
  db-password: super-secret-password
  jwt-secret: another-super-secret-key
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html/config.js
          subPath: config.js
      volumes:
      - name: config-volume
        configMap:
          name: frontend-config
          items:
          - key: config.js
            path: config.js
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: spring-boot
        image: myapp/backend:1.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /app/config/application.properties
          subPath: application.properties
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: db-url
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: db-username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: jwt-secret
      volumes:
      - name: config-volume
        configMap:
          name: backend-config
```

### Database with ConfigMap and Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgres.conf: |
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 768MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 7864kB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 2621kB
    min_wal_size = 1GB
    max_wal_size = 4GB
  
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    all             all             0.0.0.0/0               md5
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
type: Opaque
stringData:
  POSTGRES_USER: dbadmin
  POSTGRES_PASSWORD: complex-password-here
  POSTGRES_DB: application_db
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
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
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secrets
        volumeMounts:
        - name: postgres-config-volume
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgres.conf
        - name: postgres-config-volume
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        command:
        - "postgres"
        - "-c"
        - "config_file=/etc/postgresql/postgresql.conf"
        - "-c"
        - "hba_file=/etc/postgresql/pg_hba.conf"
      volumes:
      - name: postgres-config-volume
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Understanding ConfigMaps and Secrets is essential for properly implementing the configuration aspects of your applications in Kubernetes. By following best practices and leveraging the patterns outlined in this guide, you can create secure, maintainable, and flexible applications across various environments.