# Jenkins with Kubernetes

This guide explores how to effectively use Jenkins CI/CD in a Kubernetes environment, from basic setup to advanced configurations and best practices.

## Table of Contents

1. [Introduction to Jenkins in Kubernetes](#introduction-to-jenkins-in-kubernetes)
2. [Deployment Architecture](#deployment-architecture)
3. [Setting Up Jenkins on Kubernetes](#setting-up-jenkins-on-kubernetes)
4. [Jenkins Agents in Kubernetes](#jenkins-agents-in-kubernetes)
5. [Pipeline as Code](#pipeline-as-code)
6. [Jenkins Configuration as Code](#jenkins-configuration-as-code)
7. [Security Considerations](#security-considerations)
8. [Performance Optimization](#performance-optimization)
9. [Integration with Kubernetes](#integration-with-kubernetes)
10. [Best Practices](#best-practices)
11. [Real-World Examples](#real-world-examples)
12. [Troubleshooting](#troubleshooting)

## Introduction to Jenkins in Kubernetes

Jenkins is one of the most popular open-source automation servers, and running it on Kubernetes offers several advantages, including scalability, high availability, and efficient resource utilization. This combination provides a powerful platform for implementing CI/CD pipelines.

### Key Benefits

- **Dynamic Agent Provisioning**: Spin up Jenkins agents on-demand in Kubernetes
- **Resource Efficiency**: Only consume resources when builds are running
- **Isolation**: Each build runs in its own container
- **Scalability**: Easily scale up during peak demand
- **High Availability**: Leverage Kubernetes for resilient Jenkins deployments
- **Infrastructure as Code**: Define your entire CI/CD infrastructure as code

## Deployment Architecture

### Standard Deployment Model

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────┐       ┌──────────────────────────────┐ │
│  │                 │       │                              │ │
│  │  Jenkins Master │       │  Dynamic Jenkins Agents      │ │
│  │  ┌───────────┐  │       │                              │ │
│  │  │ Jenkins   │  │       │  ┌──────────┐  ┌──────────┐  │ │
│  │  │ Controller│  │       │  │ Agent    │  │ Agent    │  │ │
│  │  └───────────┘  │       │  │ Pod      │  │ Pod      │  │ │
│  │                 │       │  └──────────┘  └──────────┘  │ │
│  │  ┌───────────┐  │       │                              │ │
│  │  │Persistence│  │       │  ┌──────────┐  ┌──────────┐  │ │
│  │  │ Volume    │  │       │  │ Agent    │  │ Agent    │  │ │
│  │  └───────────┘  │       │  │ Pod      │  │ Pod      │  │ │
│  │                 │       │  └──────────┘  └──────────┘  │ │
│  └─────────────────┘       └──────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### High Availability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────┐       ┌──────────────────────────────┐ │
│  │                 │       │                              │ │
│  │  Jenkins HA     │       │  Dynamic Jenkins Agents      │ │
│  │                 │       │                              │ │
│  │  ┌───────────┐  │       │  ┌──────────┐  ┌──────────┐  │ │
│  │  │ Primary   │  │       │  │ Agent    │  │ Agent    │  │ │
│  │  │ Controller│  │       │  │ Pod      │  │ Pod      │  │ │
│  │  └───────────┘  │       │  └──────────┘  └──────────┘  │ │
│  │                 │       │                              │ │
│  │  ┌───────────┐  │       │  ┌──────────┐  ┌──────────┐  │ │
│  │  │ Standby   │  │       │  │ Agent    │  │ Agent    │  │ │
│  │  │ Controller│  │       │  │ Pod      │  │ Pod      │  │ │
│  │  └───────────┘  │       │  └──────────┘  └──────────┘  │ │
│  │                 │       │                              │ │
│  │  ┌───────────┐  │       └──────────────────────────────┘ │
│  │  │Shared     │  │                                        │
│  │  │Storage    │  │       ┌──────────────────────────────┐ │
│  │  └───────────┘  │       │                              │ │
│  │                 │       │  Shared Services             │ │
│  └─────────────────┘       │  (Artifact Repository,       │ │
│                            │   Docker Registry, etc.)     │ │
│                            │                              │ │
│                            └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Setting Up Jenkins on Kubernetes

### Helm Chart Installation

The easiest way to deploy Jenkins on Kubernetes is using the official Helm chart.

```bash
# Add the Jenkins Helm repository
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Create a namespace for Jenkins
kubectl create namespace jenkins

# Install Jenkins using Helm
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.serviceType=ClusterIP \
  --set persistence.storageClass=managed-premium \
  --set persistence.size=10Gi
```

### Custom Values Configuration

```yaml
# jenkins-values.yaml
controller:
  # Jenkins controller configurations
  installPlugins:
    - kubernetes:1.31.3
    - workflow-aggregator:2.6
    - git:4.10.0
    - configuration-as-code:1.55
  
  # Configure the Jenkins job that creates agent pods
  JCasC:
    configScripts:
      k8s-agent: |
        jenkins:
          clouds:
            - kubernetes:
                name: "kubernetes"
                serverUrl: "https://kubernetes.default.svc.cluster.local"
                namespace: "jenkins"
                jenkinsUrl: "http://jenkins-controller.jenkins.svc.cluster.local:8080"
                connectTimeout: 5
                readTimeout: 15
                containerCapStr: "10"
                maxRequestsPerHostStr: "32"
                retentionTimeout: 5
                templates:
                  - name: "jenkins-agent"
                    namespace: "jenkins"
                    label: "jenkins-agent"
                    nodeUsageMode: NORMAL
                    containers:
                      - name: "jnlp"
                        image: "jenkins/inbound-agent:4.11-1-jdk11"
                        alwaysPullImage: true
                        workingDir: "/home/jenkins/agent"
                        ttyEnabled: true
                        resourceRequestCpu: "500m"
                        resourceRequestMemory: "512Mi"
                        resourceLimitCpu: "1000m"
                        resourceLimitMemory: "1Gi"

persistence:
  # Storage configuration for Jenkins
  storageClass: "managed-premium"
  size: "10Gi"
  
serviceAccount:
  # Service account configuration
  create: true
  name: "jenkins"
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/jenkins-controller-role"
```

### Installing with Custom Values

```bash
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  -f jenkins-values.yaml
```

### RBAC Configuration

```yaml
# jenkins-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-agent
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-agent
  namespace: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-agent
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-agent
subjects:
- kind: ServiceAccount
  name: jenkins-agent
  namespace: jenkins
```

## Jenkins Agents in Kubernetes

Jenkins agents in Kubernetes are dynamically provisioned pods that execute build jobs and then terminate when the job is complete.

### Pod Template Configurations

```yaml
podTemplate(
  containers: [
    containerTemplate(
      name: 'maven',
      image: 'maven:3.8.4-openjdk-11',
      command: 'sleep',
      args: '99d',
      resourceRequestCpu: '500m',
      resourceRequestMemory: '512Mi',
      resourceLimitCpu: '1000m',
      resourceLimitMemory: '1Gi'
    ),
    containerTemplate(
      name: 'golang',
      image: 'golang:1.17',
      command: 'sleep',
      args: '99d',
      resourceRequestCpu: '500m',
      resourceRequestMemory: '512Mi',
      resourceLimitCpu: '1000m',
      resourceLimitMemory: '1Gi'
    )
  ],
  volumes: [
    persistentVolumeClaim(
      mountPath: '/root/.m2',
      claimName: 'maven-cache-pvc',
      readOnly: false
    )
  ]
) {
  node(POD_LABEL) {
    stage('Build with Maven') {
      container('maven') {
        sh 'mvn clean package'
      }
    }
    
    stage('Build with Go') {
      container('golang') {
        sh 'go build ./...'
      }
    }
  }
}
```

### Specialized Build Environments

```yaml
podTemplate(
  yaml: '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.8.0-debug
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    secret:
      secretName: docker-credentials
      items:
      - key: .dockerconfigjson
        path: config.json
'''
) {
  node(POD_LABEL) {
    stage('Build and Push with Kaniko') {
      container('kaniko') {
        sh '''
        /kaniko/executor --context=dir:///workspace \
                         --destination=myregistry.example.com/myapp:$BUILD_NUMBER \
                         --cache=true
        '''
      }
    }
  }
}
```

## Pipeline as Code

Jenkins Pipeline as Code allows you to define your CI/CD pipelines as code in Jenkinsfile, which can be stored alongside your project source code.

### Basic Jenkinsfile for a Kubernetes Application

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.4-openjdk-11
    command:
    - sleep
    args:
    - 99d
  - name: kubectl
    image: bitnami/kubectl:1.23
    command:
    - sleep
    args:
    - 99d
  - name: docker
    image: docker:20.10.12-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
"""
    }
  }
  
  stages {
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn clean package'
        }
      }
    }
    
    stage('Build Docker Image') {
      steps {
        container('docker') {
          sh 'docker build -t myregistry.example.com/myapp:$BUILD_NUMBER .'
          withCredentials([string(credentialsId: 'docker-registry-credentials', variable: 'DOCKER_AUTH')]) {
            sh 'echo $DOCKER_AUTH | docker login myregistry.example.com -u _json_key --password-stdin'
            sh 'docker push myregistry.example.com/myapp:$BUILD_NUMBER'
          }
        }
      }
    }
    
    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
              envsubst < kubernetes/deployment.yaml | kubectl apply -f -
              kubectl rollout status deployment/myapp -n application
            '''
          }
        }
      }
    }
  }
  
  post {
    success {
      echo 'Successfully built and deployed!'
    }
    failure {
      echo 'Build or deployment failed!'
    }
  }
}
```

### Shared Libraries

Shared libraries allow you to define reusable pipeline components that can be shared across multiple projects.

```groovy
// In your Jenkinsfile
@Library('my-shared-library') _

pipeline {
  agent {
    kubernetes {
      yaml libraryResource('podTemplates/java-build-pod.yaml')
    }
  }
  
  stages {
    stage('Build and Test') {
      steps {
        javaBuildAndTest()
      }
    }
    
    stage('Build and Push Image') {
      steps {
        buildDockerImage(
          imageName: 'myapp',
          tag: "${BUILD_NUMBER}",
          registry: 'myregistry.example.com'
        )
      }
    }
    
    stage('Deploy') {
      steps {
        deployToKubernetes(
          namespace: 'application',
          deployment: 'myapp',
          imageTag: "${BUILD_NUMBER}"
        )
      }
    }
  }
}
```

## Jenkins Configuration as Code

Jenkins Configuration as Code (JCasC) allows you to configure Jenkins and its plugins from simple YAML files.

### JCasC Example

```yaml
jenkins:
  systemMessage: "Jenkins configured using JCasC"
  numExecutors: 0
  scmCheckoutRetryCount: 3
  mode: NORMAL
  
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false
  
  securityRealm:
    ldap:
      configurations:
        - server: "ldap.example.com"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=users"
          userSearch: "uid={0}"
          groupSearchBase: "ou=groups"
          groupSearchFilter: "(memberUid={0})"
          managerDN: "cn=admin,dc=example,dc=com"
          managerPasswordSecret: "${LDAP_PASSWORD}"
  
  clouds:
    - kubernetes:
        name: "kubernetes"
        serverUrl: "https://kubernetes.default.svc.cluster.local"
        namespace: "jenkins"
        jenkinsUrl: "http://jenkins-controller.jenkins.svc.cluster.local:8080"
        connectTimeout: 5
        readTimeout: 15
        containerCapStr: "10"
        podRetention: "never"
        templates:
          - name: "default"
            namespace: "jenkins"
            label: "jenkins-agent"
            nodeUsageMode: NORMAL
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:4.11-1-jdk11"
                alwaysPullImage: true
  
jobs:
  - script: >
      folder('applications')
  
  - script: >
      multibranchPipelineJob('applications/api-service') {
        branchSources {
          github {
            id('api-service-repo')
            repository('api-service')
            repositoryUrl('https://github.com/myorg/api-service')
            repoOwner('myorg')
            credentialsId('github-credentials')
          }
        }
        orphanedItemStrategy {
          discardOldItems {
            numToKeep(10)
          }
        }
      }
```

## Security Considerations

### Secrets Management

There are several ways to manage secrets in Jenkins on Kubernetes:

1. **Kubernetes Secrets**: Store sensitive information as Kubernetes secrets and mount them in Jenkins pods.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-credentials
  namespace: jenkins
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded 'admin'
  password: cGFzc3dvcmQxMjM=  # base64 encoded 'password123'
```

2. **Jenkins Credentials Plugin**: Use Jenkins built-in credentials store.

```groovy
pipeline {
  agent {
    kubernetes {
      // Pod template
    }
  }
  
  stages {
    stage('Deploy') {
      steps {
        container('kubectl') {
          withCredentials([
            usernamePassword(credentialsId: 'registry-credentials', 
                            usernameVariable: 'REGISTRY_USER', 
                            passwordVariable: 'REGISTRY_PASSWORD')
          ]) {
            sh 'docker login -u $REGISTRY_USER -p $REGISTRY_PASSWORD'
          }
        }
      }
    }
  }
}
```

3. **External Secret Management**: Integrate with external secret management tools like HashiCorp Vault, AWS Secrets Manager, etc.

```groovy
pipeline {
  agent {
    kubernetes {
      // Pod template with vault agent
    }
  }
  
  stages {
    stage('Get Secrets') {
      steps {
        container('vault') {
          script {
            def secrets = sh(script: 'vault kv get -format=json secret/myapp/credentials', returnStdout: true)
            def parsed = readJSON text: secrets
            env.API_KEY = parsed.data.data.api_key
          }
        }
      }
    }
  }
}
```

### RBAC Configuration

Ensure Jenkins has the minimum required permissions in your Kubernetes cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-deployer-binding
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
roleRef:
  kind: ClusterRole
  name: jenkins-deployer
  apiGroup: rbac.authorization.k8s.io
```

## Performance Optimization

### Resource Management

1. **Optimize Agent Pod Resources**: Configure resource requests and limits for agent pods.

```yaml
podTemplate(
  containers: [
    containerTemplate(
      name: 'maven',
      image: 'maven:3.8.4-openjdk-11',
      resourceRequestCpu: '500m',
      resourceRequestMemory: '1Gi',
      resourceLimitCpu: '2',
      resourceLimitMemory: '2Gi'
    )
  ]
)
```

2. **Caching**: Use persistent volume claims for caching dependencies.

```yaml
podTemplate(
  volumes: [
    persistentVolumeClaim(
      mountPath: '/root/.m2',
      claimName: 'maven-cache-pvc',
      readOnly: false
    )
  ]
)
```

3. **Parallel Execution**: Run build steps in parallel where possible.

```groovy
pipeline {
  agent {
    kubernetes {
      // Pod template
    }
  }
  
  stages {
    stage('Parallel Tests') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('Integration Tests') {
          steps {
            container('maven') {
              sh 'mvn verify -DskipUnitTests'
            }
          }
        }
      }
    }
  }
}
```

## Integration with Kubernetes

### Deploying to Kubernetes

1. **Using kubectl**: Deploy directly using kubectl in pipeline.

```groovy
stage('Deploy') {
  steps {
    container('kubectl') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh 'kubectl apply -f kubernetes/manifests/'
        sh 'kubectl rollout status deployment/myapp -n application'
      }
    }
  }
}
```

2. **Using Helm**: Deploy applications using Helm.

```groovy
stage('Deploy with Helm') {
  steps {
    container('helm') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh '''
          helm upgrade --install myapp ./helm/myapp \
            --namespace application \
            --set image.tag=${BUILD_NUMBER} \
            --set environment=production
        '''
      }
    }
  }
}
```

### Blue/Green Deployments

```groovy
stage('Blue/Green Deployment') {
  steps {
    container('kubectl') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        script {
          // Deploy the new version (green)
          sh '''
            cat kubernetes/deployment.yaml | \
            sed "s/APP_VERSION/${BUILD_NUMBER}/g" | \
            sed "s/DEPLOYMENT_NAME/myapp-green/g" | \
            kubectl apply -f -
          '''
          
          // Wait for green deployment to be ready
          sh 'kubectl rollout status deployment/myapp-green -n application'
          
          // Switch traffic to green deployment
          sh '''
            cat kubernetes/service.yaml | \
            sed "s/SELECTOR_COLOR/green/g" | \
            kubectl apply -f -
          '''
          
          // Wait for verification or approval
          input 'Do you want to delete the old blue deployment?'
          
          // Delete the old blue deployment
          sh 'kubectl delete deployment myapp-blue -n application'
        }
      }
    }
  }
}
```

## Best Practices

### Jenkins on Kubernetes Best Practices

1. **Immutable Build Environments**: Use container images with all tools pre-installed.
2. **Ephemeral Agents**: Treat build agents as disposable, don't persist state.
3. **Pipeline as Code**: Store your pipeline configurations in version control.
4. **Configuration as Code**: Use JCasC for Jenkins configuration.
5. **Resource Management**: Set appropriate resource requests and limits.
6. **Least Privilege**: Grant only necessary permissions to Jenkins.
7. **Artifact Management**: Store build artifacts in external repositories.
8. **Monitoring and Observability**: Monitor Jenkins and pipeline performance.
9. **High Availability**: Configure Jenkins for high availability in production.
10. **Backup**: Regularly backup Jenkins configuration and job data.

### Security Best Practices

1. **Keep Jenkins Updated**: Regularly update Jenkins and plugins.
2. **Secure Jenkins UI**: Use HTTPS and proper authentication.
3. **Audit Logs**: Enable and monitor audit logs.
4. **Scan Images**: Include security scanning in your pipelines.
5. **Review Plugin Permissions**: Only install necessary plugins.

## Real-World Examples

### Microservices CI/CD Pipeline

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.4-openjdk-11
    command: ['sleep']
    args: ['infinity']
    volumeMounts:
    - name: m2-cache
      mountPath: /root/.m2
  - name: docker
    image: docker:20.10.12-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
  - name: kubectl
    image: bitnami/kubectl:1.23
    command: ['sleep']
    args: ['infinity']
  volumes:
  - name: m2-cache
    persistentVolumeClaim:
      claimName: m2-cache
  - name: docker-socket
    emptyDir: {}
"""
    }
  }
  
  environment {
    SERVICE_NAME = 'user-service'
    REGISTRY = 'myregistry.example.com'
    VERSION = "${env.BUILD_NUMBER}"
  }
  
  stages {
    stage('Tests') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
          post {
            always {
              junit 'target/surefire-reports/**/*.xml'
            }
          }
        }
        
        stage('Code Analysis') {
          steps {
            container('maven') {
              sh 'mvn sonar:sonar -Dsonar.host.url=http://sonarqube-service:9000'
            }
          }
        }
      }
    }
    
    stage('Build and Package') {
      steps {
        container('maven') {
          sh 'mvn package -DskipTests'
        }
      }
    }
    
    stage('Build Docker Image') {
      steps {
        container('docker') {
          sh "docker build -t ${REGISTRY}/${SERVICE_NAME}:${VERSION} ."
          withCredentials([string(credentialsId: 'registry-token', variable: 'REGISTRY_AUTH')]) {
            sh 'echo $REGISTRY_AUTH | docker login ${REGISTRY} -u _token_key --password-stdin'
            sh "docker push ${REGISTRY}/${SERVICE_NAME}:${VERSION}"
          }
        }
      }
    }
    
    stage('Deploy to Development') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG')]) {
            sh '''
              envsubst < kubernetes/deployment.yaml | kubectl apply -f -
              kubectl rollout status deployment/${SERVICE_NAME} -n development
            '''
          }
        }
      }
    }
    
    stage('Integration Tests') {
      steps {
        container('maven') {
          sh 'mvn verify -DskipUnitTests'
        }
      }
      post {
        always {
          junit 'target/failsafe-reports/**/*.xml'
        }
      }
    }
    
    stage('Deploy to Production') {
      when {
        branch 'master'
      }
      steps {
        timeout(time: 1, unit: 'DAYS') {
          input 'Deploy to production?'
        }
        container('kubectl') {
          withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
            sh '''
              envsubst < kubernetes/deployment.yaml | sed 's/namespace: development/namespace: production/' | kubectl apply -f -
              kubectl rollout status deployment/${SERVICE_NAME} -n production
            '''
          }
        }
      }
    }
  }
  
  post {
    success {
      slackSend channel: '#ci-cd', color: 'good', message: "Build successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    failure {
      slackSend channel: '#ci-cd', color: 'danger', message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    always {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
  }
}
```

## Troubleshooting

### Common Issues and Solutions

1. **Jenkins Agents Not Starting**
   - Check RBAC permissions
   - Verify network connectivity
   - Check resource constraints

2. **Pipeline Failures**
   - Enable pipeline debug logging:
     ```groovy
     pipeline {
       agent {
         kubernetes {
           // Pod template
         }
       }
       options {
         timestamps()
         ansiColor('xterm')
       }
       stages {
         stage('Debug Info') {
           steps {
             sh 'env | sort'
             sh 'kubectl version'
           }
         }
       }
     }
     ```

3. **Performance Issues**
   - Check resource usage in Jenkins and agents
   - Optimize parallel execution
   - Review volume mounting and caching

### Debugging Jenkins in Kubernetes

```bash
# Get Jenkins pod name
kubectl get pods -n jenkins

# Check Jenkins logs
kubectl logs jenkins-0 -n jenkins

# Describe Jenkins pod for events and status
kubectl describe pod jenkins-0 -n jenkins

# Check agent pod logs
kubectl logs jenkins-agent-xyz -n jenkins

# Exec into Jenkins pod for troubleshooting
kubectl exec -it jenkins-0 -n jenkins -- bash
```

## Summary

Jenkins on Kubernetes provides a powerful and flexible CI/CD platform that can scale to meet the needs of modern application development. By leveraging Kubernetes for dynamic agent provisioning, resource management, and high availability, organizations can build efficient and reliable CI/CD pipelines.

Key takeaways:
1. Use Helm for easy deployment of Jenkins on Kubernetes
2. Leverage dynamic agent provisioning for scalability
3. Implement Pipeline as Code for maintainable CI/CD workflows
4. Use Configuration as Code for consistent Jenkins setup
5. Follow security best practices for both Jenkins and Kubernetes
6. Optimize performance through proper resource management
7. Implement backup and disaster recovery for production Jenkins installations

By following the patterns and best practices outlined in this guide, you can build a robust CI/CD platform that enables your teams to deliver software efficiently and reliably.