# AI/ML on Kubernetes

## Introduction

Artificial Intelligence (AI) and Machine Learning (ML) workloads present unique challenges for infrastructure management due to their specialized hardware requirements, complex dependencies, and non-traditional development workflows. Kubernetes has emerged as a leading platform for orchestrating AI/ML workloads by providing scalability, resource management, and workflow automation.

This guide explores how to effectively deploy, manage, and optimize AI/ML workloads on Kubernetes, including infrastructure considerations, tooling, frameworks, and best practices.

## AI/ML Workload Characteristics

AI/ML workloads have unique attributes that differentiate them from traditional applications:

| Characteristic | Description | Kubernetes Consideration |
|----------------|-------------|--------------------------|
| GPU Dependencies | Many ML workloads require accelerated compute (GPUs, TPUs) | Specialized hardware scheduling and resource management |
| Batch Processing | Training jobs often run as one-time or periodic batch jobs | Job and CronJob resources |
| Distributed Training | Complex workloads may be distributed across multiple nodes | Custom orchestration and networking |
| Large Datasets | Training requires access to substantial datasets | Persistent storage management |
| Experiment Tracking | ML development involves iterative experimentation | Metadata management and versioning |
| Model Serving | Deploying models as APIs for inference | Service deployment and scaling |
| Resource Intensity | Training can consume significant compute resources | Resource quotas and prioritization |

## Kubernetes Components for AI/ML

### GPU Support in Kubernetes

Kubernetes supports GPU scheduling through device plugins:

```yaml
# Example Pod with GPU request
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: gpu-container
    image: nvidia/cuda:11.6.0-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1 # requesting 1 GPU
```

To enable GPU support:

1. Install GPU drivers on nodes
2. Deploy the NVIDIA device plugin (for NVIDIA GPUs):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.13.0
        name: nvidia-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
```

### Multi-Instance GPU (MIG)

NVIDIA's MIG allows partitioning a GPU into multiple instances:

```yaml
# Pod using MIG GPU partition
apiVersion: v1
kind: Pod
metadata:
  name: mig-pod
spec:
  containers:
  - name: mig-container
    image: nvidia/cuda:11.6.0-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/mig-1g.5gb: 1 # requesting 1 MIG partition of 1g.5gb size
```

### Fractional GPU Allocation with Time-Slicing

For older GPUs without MIG support, time-slicing allows sharing:

```yaml
# DaemonSet configuration for time-slicing
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  # ... other fields omitted for brevity
  template:
    spec:
      # ... other fields omitted for brevity
      containers:
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.13.0
        name: nvidia-device-plugin-ctr
        env:
        - name: NVIDIA_VISIBLE_DEVICES
          value: all
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: all
        - name: NVIDIA_MPS_ENABLE_TIMESLICING
          value: "1"
```

### TPU Support

For Google Cloud TPUs:

```yaml
# Pod with TPU request
apiVersion: v1
kind: Pod
metadata:
  name: tpu-pod
spec:
  containers:
  - name: tpu-container
    image: tensorflow/tensorflow:2.12.0
    command: ["python", "/app/train.py"]
    resources:
      limits:
        cloud-tpus.google.com/v3: 8 # requesting 8 TPU v3 cores
```

## ML Training Workflows

### Basic Training Job

```yaml
# Simple ML training job
apiVersion: batch/v1
kind: Job
metadata:
  name: tf-mnist
spec:
  template:
    spec:
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:2.12.0-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: data
          mountPath: /data
        - name: models
          mountPath: /models
        command:
        - "python"
        - "/app/train.py"
        - "--data_dir=/data"
        - "--model_dir=/models"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mnist-data-pvc
      - name: models
        persistentVolumeClaim:
          claimName: model-storage-pvc
      restartPolicy: Never
  backoffLimit: 4
```

### Distributed Training with TensorFlow

```yaml
# TensorFlow distributed training job
apiVersion: v1
kind: ConfigMap
metadata:
  name: tf-distributed-config
data:
  tf-config.json: |
    {
      "cluster": {
        "worker": ["tf-worker-0.tf-worker:2222", "tf-worker-1.tf-worker:2222"],
        "ps": ["tf-ps-0.tf-ps:2222"]
      },
      "task": {
        "type": "${POD_TYPE}",
        "index": ${POD_INDEX}
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: tf-worker
spec:
  clusterIP: None
  selector:
    job: tf-distributed
    role: worker
  ports:
  - port: 2222
    name: tfjob-port
---
apiVersion: v1
kind: Service
metadata:
  name: tf-ps
spec:
  clusterIP: None
  selector:
    job: tf-distributed
    role: ps
  ports:
  - port: 2222
    name: tfjob-port
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tf-worker
spec:
  serviceName: tf-worker
  replicas: 2
  selector:
    matchLabels:
      job: tf-distributed
      role: worker
  template:
    metadata:
      labels:
        job: tf-distributed
        role: worker
    spec:
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:2.12.0-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
        env:
        - name: POD_TYPE
          value: "worker"
        - name: POD_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: TF_CONFIG
          valueFrom:
            configMapKeyRef:
              name: tf-distributed-config
              key: tf-config.json
        command:
        - "python"
        - "/app/distributed_train.py"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tf-ps
spec:
  serviceName: tf-ps
  replicas: 1
  selector:
    matchLabels:
      job: tf-distributed
      role: ps
  template:
    metadata:
      labels:
        job: tf-distributed
        role: ps
    spec:
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:2.12.0
        env:
        - name: POD_TYPE
          value: "ps"
        - name: POD_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: TF_CONFIG
          valueFrom:
            configMapKeyRef:
              name: tf-distributed-config
              key: tf-config.json
        command:
        - "python"
        - "/app/distributed_train.py"
```

### PyTorch Distributed Training

```yaml
# PyTorch distributed training job
apiVersion: v1
kind: Service
metadata:
  name: pytorch-master
spec:
  selector:
    role: master
  ports:
  - port: 29500
    name: master-port
---
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-master
  labels:
    role: master
spec:
  containers:
  - name: pytorch
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    args:
    - "python"
    - "/app/distributed_train.py"
    - "--master=pytorch-master:29500"
    - "--rank=0"
    - "--world-size=3"
    ports:
    - containerPort: 29500
    resources:
      limits:
        nvidia.com/gpu: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-worker-1
spec:
  containers:
  - name: pytorch
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    args:
    - "python"
    - "/app/distributed_train.py"
    - "--master=pytorch-master:29500"
    - "--rank=1"
    - "--world-size=3"
    resources:
      limits:
        nvidia.com/gpu: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-worker-2
spec:
  containers:
  - name: pytorch
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    args:
    - "python"
    - "/app/distributed_train.py"
    - "--master=pytorch-master:29500"
    - "--rank=2"
    - "--world-size=3"
    resources:
      limits:
        nvidia.com/gpu: 1
```

## ML Operators and Frameworks

### Kubeflow

Kubeflow is an end-to-end ML platform for Kubernetes.

#### Installing Kubeflow

```bash
# Install Kubeflow using kustomize
kustomize build https://github.com/kubeflow/manifests/examples/kustomize/deployment > kubeflow_installation.yaml
kubectl apply -f kubeflow_installation.yaml
```

#### TFJob Example

```yaml
# TensorFlow training job using Kubeflow TFJob
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  name: tensorflow-mnist
spec:
  tfReplicaSpecs:
    Worker:
      replicas: 2
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.12.0-gpu
            resources:
              limits:
                nvidia.com/gpu: 1
            command:
            - "python"
            - "/app/train.py"
    PS:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.12.0
            command:
            - "python"
            - "/app/train.py"
```

#### PyTorchJob Example

```yaml
# PyTorch training job using Kubeflow PyTorchJob
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-mnist
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
            resources:
              limits:
                nvidia.com/gpu: 1
            command:
            - "python"
            - "/app/distributed_train.py"
    Worker:
      replicas: 2
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
            resources:
              limits:
                nvidia.com/gpu: 1
            command:
            - "python"
            - "/app/distributed_train.py"
```

### KServe (formerly KFServing)

KServe provides serverless inferencing on Kubernetes:

```yaml
# KServe model deployment
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
spec:
  predictor:
    sklearn:
      storageUri: "gs://kserve-examples/models/sklearn/iris"
      resources:
        requests:
          cpu: 100m
          memory: 512Mi
        limits:
          cpu: 1
          memory: 1Gi
```

For GPU-accelerated inferencing:

```yaml
# GPU-accelerated inference service
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: bert-transformer
spec:
  predictor:
    tensorflow:
      storageUri: "gs://kserve-examples/models/tensorflow/bert"
      resources:
        requests:
          cpu: 1
          memory: 2Gi
        limits:
          cpu: 2
          memory: 4Gi
          nvidia.com/gpu: 1
```

### MLflow on Kubernetes

```yaml
# MLflow tracking server deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-tracking
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow-tracking
  template:
    metadata:
      labels:
        app: mlflow-tracking
    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.4.1
        ports:
        - containerPort: 5000
        args:
        - "mlflow"
        - "server"
        - "--backend-store-uri=postgresql://mlflow:password@postgres:5432/mlflow"
        - "--default-artifact-root=s3://mlflow/artifacts"
        - "--host=0.0.0.0"
        - "--port=5000"
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: mlflow-s3-credentials
              key: AWS_ACCESS_KEY_ID
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: mlflow-s3-credentials
              key: AWS_SECRET_ACCESS_KEY
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-tracking
spec:
  selector:
    app: mlflow-tracking
  ports:
  - port: 5000
    targetPort: 5000
  type: ClusterIP
```

## Model Serving Patterns

### Basic Model Serving with TensorFlow Serving

```yaml
# TensorFlow Serving deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tensorflow-serving
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tensorflow-serving
  template:
    metadata:
      labels:
        app: tensorflow-serving
    spec:
      containers:
      - name: tensorflow-serving
        image: tensorflow/serving:2.10.0
        args:
        - "--model_name=resnet"
        - "--model_base_path=/models/resnet"
        ports:
        - containerPort: 8501
        resources:
          limits:
            cpu: 2
            memory: 4Gi
            nvidia.com/gpu: 1
        volumeMounts:
        - name: model-volume
          mountPath: /models
      volumes:
      - name: model-volume
        persistentVolumeClaim:
          claimName: model-storage-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: tensorflow-serving
spec:
  selector:
    app: tensorflow-serving
  ports:
  - port: 8501
    targetPort: 8501
  type: ClusterIP
```

### Triton Inference Server

```yaml
# NVIDIA Triton Inference Server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-inference
spec:
  replicas: 2
  selector:
    matchLabels:
      app: triton-inference
  template:
    metadata:
      labels:
        app: triton-inference
    spec:
      containers:
      - name: triton
        image: nvcr.io/nvidia/tritonserver:22.12-py3
        args:
        - "tritonserver"
        - "--model-repository=/models"
        - "--strict-model-config=false"
        - "--log-verbose=1"
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8001
          name: grpc
        - containerPort: 8002
          name: metrics
        resources:
          limits:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
        volumeMounts:
        - name: model-repository
          mountPath: /models
      volumes:
      - name: model-repository
        persistentVolumeClaim:
          claimName: triton-models-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: triton-inference
spec:
  selector:
    app: triton-inference
  ports:
  - port: 8000
    targetPort: 8000
    name: http
  - port: 8001
    targetPort: 8001
    name: grpc
  - port: 8002
    targetPort: 8002
    name: metrics
  type: ClusterIP
```

### Serving with Auto-scaling

```yaml
# HPA for model serving based on request rate
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: model-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: model-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: inference_requests_per_second
      target:
        type: AverageValue
        averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
```

### A/B Testing with Model Variants

```yaml
# Istio VirtualService for model A/B testing
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: model-ab-test
spec:
  hosts:
  - "model-service.example.com"
  http:
  - route:
    - destination:
        host: model-service-v1
        port:
          number: 8501
      weight: 80
    - destination:
        host: model-service-v2
        port:
          number: 8501
      weight: 20
```

## Data Processing Pipelines

### Apache Spark on Kubernetes

```yaml
# Spark operator config
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-data-processing
  namespace: spark-jobs
spec:
  type: Python
  mode: cluster
  image: "apache/spark-py:v3.3.1"
  imagePullPolicy: Always
  mainApplicationFile: "local:///opt/spark/examples/src/main/python/ml/als_example.py"
  sparkVersion: "3.3.1"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.3.1
    serviceAccount: spark
  executor:
    cores: 1
    instances: 2
    memory: "512m"
    labels:
      version: 3.3.1
```

### Apache Airflow for ML Pipelines

```yaml
# Airflow DAG for ML workflow
apiVersion: v1
kind: ConfigMap
metadata:
  name: ml-pipeline-dag
data:
  ml_pipeline.py: |
    from airflow import DAG
    from airflow.operators.bash_operator import BashOperator
    from airflow.operators.python_operator import PythonOperator
    from airflow.kubernetes.secret import Secret
    from airflow.kubernetes.pod import Resources
    from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
    from datetime import datetime, timedelta
    
    default_args = {
        'owner': 'data-science',
        'depends_on_past': False,
        'start_date': datetime(2023, 1, 1),
        'email_on_failure': True,
        'email_on_retry': False,
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
    }
    
    dag = DAG(
        'ml_training_pipeline',
        default_args=default_args,
        schedule_interval='@weekly',
        catchup=False
    )
    
    data_preprocessing = KubernetesPodOperator(
        namespace='ml-pipelines',
        image='data-science/preprocessing:latest',
        cmds=["python", "-c"],
        arguments=["from preprocessing import main; main()"],
        labels={"pipeline": "ml-training"},
        name="data-preprocessing",
        task_id="data-preprocessing",
        get_logs=True,
        resources=Resources(request_memory="2Gi", request_cpu="1"),
        dag=dag
    )
    
    model_training = KubernetesPodOperator(
        namespace='ml-pipelines',
        image='data-science/training:latest',
        cmds=["python", "-c"],
        arguments=["from training import main; main()"],
        labels={"pipeline": "ml-training"},
        name="model-training",
        task_id="model-training",
        get_logs=True,
        resources=Resources(
            request_memory="4Gi", 
            request_cpu="2",
            limit_memory="8Gi",
            limit_cpu="4",
            request_gpu="1",
            limit_gpu="1"
        ),
        dag=dag
    )
    
    model_evaluation = KubernetesPodOperator(
        namespace='ml-pipelines',
        image='data-science/evaluation:latest',
        cmds=["python", "-c"],
        arguments=["from evaluation import main; main()"],
        labels={"pipeline": "ml-training"},
        name="model-evaluation",
        task_id="model-evaluation",
        get_logs=True,
        resources=Resources(request_memory="2Gi", request_cpu="1"),
        dag=dag
    )
    
    model_deployment = KubernetesPodOperator(
        namespace='ml-pipelines',
        image='data-science/deployment:latest',
        cmds=["python", "-c"],
        arguments=["from deployment import main; main()"],
        labels={"pipeline": "ml-training"},
        name="model-deployment",
        task_id="model-deployment",
        get_logs=True,
        resources=Resources(request_memory="1Gi", request_cpu="500m"),
        dag=dag
    )
    
    data_preprocessing >> model_training >> model_evaluation >> model_deployment
```

### Argo Workflows for ML Pipelines

```yaml
# Argo Workflow for ML pipeline
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ml-pipeline-
spec:
  entrypoint: ml-pipeline
  serviceAccountName: argo-workflow
  volumes:
  - name: workdir
    persistentVolumeClaim:
      claimName: ml-pipeline-data
  
  templates:
  - name: ml-pipeline
    dag:
      tasks:
      - name: data-preparation
        template: data-prep
      - name: feature-engineering
        dependencies: [data-preparation]
        template: feature-eng
      - name: model-training
        dependencies: [feature-engineering]
        template: train-model
      - name: model-evaluation
        dependencies: [model-training]
        template: evaluate-model
      - name: model-deployment
        dependencies: [model-evaluation]
        template: deploy-model
        when: "{{tasks.model-evaluation.outputs.parameters.model-accuracy}} >= 0.85"

  - name: data-prep
    container:
      image: data-science/data-prep:latest
      command: [python, /scripts/data_prep.py]
      volumeMounts:
      - name: workdir
        mountPath: /data
    outputs:
      artifacts:
      - name: processed-data
        path: /data/processed_data.csv
  
  - name: feature-eng
    container:
      image: data-science/feature-eng:latest
      command: [python, /scripts/feature_eng.py]
      volumeMounts:
      - name: workdir
        mountPath: /data
    inputs:
      artifacts:
      - name: processed-data
        path: /data/processed_data.csv
    outputs:
      artifacts:
      - name: feature-data
        path: /data/features.csv
  
  - name: train-model
    container:
      image: tensorflow/tensorflow:2.12.0-gpu
      command: [python, /scripts/train.py]
      volumeMounts:
      - name: workdir
        mountPath: /data
      resources:
        limits:
          nvidia.com/gpu: 1
    inputs:
      artifacts:
      - name: feature-data
        path: /data/features.csv
    outputs:
      artifacts:
      - name: model
        path: /data/model
  
  - name: evaluate-model
    container:
      image: tensorflow/tensorflow:2.12.0
      command: [python, /scripts/evaluate.py]
      volumeMounts:
      - name: workdir
        mountPath: /data
    inputs:
      artifacts:
      - name: model
        path: /data/model
      - name: feature-data
        path: /data/features.csv
    outputs:
      parameters:
      - name: model-accuracy
        valueFrom:
          path: /data/metrics.json
          jsonPath: '$.accuracy'
  
  - name: deploy-model
    container:
      image: data-science/deployer:latest
      command: [python, /scripts/deploy.py]
      volumeMounts:
      - name: workdir
        mountPath: /data
    inputs:
      artifacts:
      - name: model
        path: /data/model
```

## Data Management for ML

### Data Versioning with DVC

```yaml
# Using DVC in a Kubernetes job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-versioning
spec:
  template:
    spec:
      containers:
      - name: dvc
        image: iterativeai/dvc:2.10.2
        command:
        - "bash"
        - "-c"
        - |
          git clone https://github.com/company/ml-project.git
          cd ml-project
          dvc pull
          python process_data.py
          dvc add processed_data
          dvc push
          git add .
          git commit -m "Updated processed data"
          git push
        volumeMounts:
        - name: git-config
          mountPath: /root/.gitconfig
        - name: git-credentials
          mountPath: /root/.git-credentials
        - name: dvc-config
          mountPath: /root/.dvc
      volumes:
      - name: git-config
        configMap:
          name: git-config
      - name: git-credentials
        secret:
          secretName: git-credentials
      - name: dvc-config
        secret:
          secretName: dvc-config
      restartPolicy: Never
  backoffLimit: 2
```

### S3-Compatible Storage for ML Data

```yaml
# PVC with S3 storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ml-dataset
spec:
  storageClassName: s3-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```

## Resource Optimization

### GPU Memory Optimization

```yaml
# NVIDIA MPS Server configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-mps-config
data:
  start-mps.sh: |
    #!/bin/bash
    nvidia-smi -c EXCLUSIVE_PROCESS
    nvidia-cuda-mps-control -d
```

### Dynamic Resource Allocation

```yaml
# Dynamic allocation for training jobs
apiVersion: batch/v1
kind: Job
metadata:
  name: elastic-training
spec:
  template:
    spec:
      containers:
      - name: elastic-training
        image: company/elastic-training:latest
        resources:
          limits:
            cpu: 8
            memory: 16Gi
            nvidia.com/gpu: 1
          requests:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
        env:
        - name: TF_FORCE_GPU_ALLOW_GROWTH
          value: "true"
        command:
        - "python"
        - "/app/elastic_train.py"
        - "--initial_workers=1"
        - "--max_workers=4"
      restartPolicy: Never
```

### Spot Instances for Training

```yaml
# Using spot instances for non-critical training
apiVersion: batch/v1
kind: Job
metadata:
  name: spot-training
spec:
  template:
    metadata:
      labels:
        job-type: training
    spec:
      nodeSelector:
        cloud.google.com/gke-spot: "true" # GKE example
      # Or for AWS:
      # nodeSelector:
      #   eks.amazonaws.com/capacityType: SPOT
      containers:
      - name: training
        image: tensorflow/tensorflow:2.12.0-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
        command:
        - "python"
        - "/app/train_with_checkpoints.py"
        - "--checkpoint_frequency=10"
      tolerations:
      - key: "preemptible"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      restartPolicy: OnFailure
```

## MLOps Best Practices

### Environment Reproducibility

```yaml
# Containerized ML environment
apiVersion: v1
kind: ConfigMap
metadata:
  name: ml-environment-config
data:
  requirements.txt: |
    tensorflow==2.12.0
    scikit-learn==1.2.2
    pandas==2.0.1
    numpy==1.23.5
    matplotlib==3.7.1
    mlflow==2.4.1
  
  Dockerfile: |
    FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04
    RUN apt-get update && apt-get install -y python3-pip
    COPY requirements.txt /tmp/
    RUN pip install -r /tmp/requirements.txt
```

### CI/CD for ML Models

```yaml
# GitHub Actions workflow for ML CI/CD
name: ML Model CI/CD

on:
  pull_request:
    branches: [ main ]
    paths:
      - 'models/**'
      - 'data/**'
      - 'training/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run tests
      run: |
        pytest tests/

  train-model:
    needs: test
    runs-on: self-hosted
    steps:
    - uses: actions/checkout@v3
    - name: Train model
      run: |
        python training/train.py --data data/train.csv --output models/model.pkl
    - name: Evaluate model
      run: |
        python training/evaluate.py --model models/model.pkl --data data/test.csv
    - name: Upload model
      uses: actions/upload-artifact@v3
      with:
        name: trained-model
        path: models/model.pkl

  deploy-model:
    needs: train-model
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Download model
      uses: actions/download-artifact@v3
      with:
        name: trained-model
        path: models/
    - name: Set up Kubernetes
      uses: Azure/k8s-set-context@v1
      with:
        kubeconfig: ${{ secrets.KUBE_CONFIG }}
    - name: Build and push container
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: company/model-server:${{ github.sha }}
    - name: Deploy model
      run: |
        kubectl set image deployment/model-server model-server=company/model-server:${{ github.sha }}
```

### Model Monitoring

```yaml
# Prometheus monitoring for model performance
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: model-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: model-server
  endpoints:
  - port: metrics
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: model-alerts
  namespace: monitoring
spec:
  groups:
  - name: model.rules
    rules:
    - alert: ModelAccuracyDrop
      expr: increase(model_accuracy_drop_count[1h]) > 10
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Model accuracy has dropped"
        description: "The model accuracy has dropped significantly in the last hour"
    - alert: ModelLatencyHigh
      expr: avg_over_time(model_inference_latency_seconds[5m]) > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Model inference latency is high"
        description: "The model inference latency is higher than 500ms for the last 10 minutes"
```

### Feature Store Integration

```yaml
# Feast feature store deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feast-online-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: feast-online-serving
  template:
    metadata:
      labels:
        app: feast-online-serving
    spec:
      containers:
      - name: feast-server
        image: feastdev/feature-server:0.30.1
        ports:
        - containerPort: 6566
        env:
        - name: FEAST_REDIS_HOST
          value: "feast-redis"
        - name: FEAST_REDIS_PORT
          value: "6379"
        - name: FEAST_FEATURE_STORE_CONFIG_PATH
          value: "/etc/feast/feature_store.yaml"
        volumeMounts:
        - name: feast-config
          mountPath: /etc/feast
      volumes:
      - name: feast-config
        configMap:
          name: feast-config
---
apiVersion: v1
kind: Service
metadata:
  name: feast-online-serving
spec:
  selector:
    app: feast-online-serving
  ports:
  - port: 6566
    targetPort: 6566
  type: ClusterIP
```

## Security and Compliance for AI/ML

### Secure Model Serving

```yaml
# Secure Model Serving with mTLS
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: model-server-mtls
spec:
  host: model-server.ml.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: model-server-rbac
spec:
  selector:
    matchLabels:
      app: model-server
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend-client"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/v1/models/*/infer"]
```

### Model Validation Pipeline

```yaml
# Model validation pipeline
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: model-validation-
spec:
  entrypoint: validate-model
  templates:
  - name: validate-model
    steps:
    - - name: validate-inputs
        template: check-inputs
    - - name: validate-robustness
        template: robustness-test
    - - name: validate-fairness
        template: fairness-test
    - - name: validate-explainability
        template: explainability-test
    - - name: validate-security
        template: security-test
    - - name: approval
        template: manual-approval
        when: "{{steps.validate-fairness.outputs.result}} == 'passed' && {{steps.validate-security.outputs.result}} == 'passed'"
    - - name: deploy
        template: deploy-model
        when: "{{steps.approval.outputs.result}} == 'approved'"

  - name: check-inputs
    container:
      image: model-validators/input-validator:latest
      command: [python, /scripts/validate_inputs.py]
  
  - name: robustness-test
    container:
      image: model-validators/robustness:latest
      command: [python, /scripts/test_robustness.py]
  
  - name: fairness-test
    container:
      image: model-validators/fairness:latest
      command: [python, /scripts/test_fairness.py]
  
  - name: explainability-test
    container:
      image: model-validators/explainability:latest
      command: [python, /scripts/test_explainability.py]
  
  - name: security-test
    container:
      image: model-validators/security:latest
      command: [python, /scripts/test_security.py]
  
  - name: manual-approval
    suspend: {}
  
  - name: deploy-model
    container:
      image: model-deployers/deployer:latest
      command: [python, /scripts/deploy.py]
```

## Case Studies and Examples

### Real-time Recommendation System

```yaml
# Real-time recommendation system architecture
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feature-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: feature-processor
  template:
    metadata:
      labels:
        app: feature-processor
    spec:
      containers:
      - name: processor
        image: company/feature-processor:latest
        resources:
          limits:
            cpu: 2
            memory: 4Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-engine
spec:
  replicas: 5
  selector:
    matchLabels:
      app: recommendation-engine
  template:
    metadata:
      labels:
        app: recommendation-engine
    spec:
      containers:
      - name: engine
        image: company/recommendation-engine:latest
        resources:
          limits:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
        env:
        - name: FEATURE_STORE_URL
          value: "feast-online-serving:6566"
        - name: REDIS_CACHE_URL
          value: "redis:6379"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-api
spec:
  replicas: 10
  selector:
    matchLabels:
      app: recommendation-api
  template:
    metadata:
      labels:
        app: recommendation-api
    spec:
      containers:
      - name: api
        image: company/recommendation-api:latest
        resources:
          requests:
            cpu: 1
            memory: 2Gi
          limits:
            cpu: 2
            memory: 4Gi
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 30
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: recommendation-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: recommendation-api
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
```

### Computer Vision Processing Pipeline

```yaml
# Computer vision processing pipeline
apiVersion: apps/v1
kind: Deployment
metadata:
  name: video-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: video-processor
  template:
    metadata:
      labels:
        app: video-processor
    spec:
      containers:
      - name: processor
        image: company/video-processor:latest
        resources:
          limits:
            cpu: 2
            memory: 4Gi
            nvidia.com/gpu: 1
        volumeMounts:
        - name: video-store
          mountPath: /data
      volumes:
      - name: video-store
        persistentVolumeClaim:
          claimName: video-data-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: object-detector
spec:
  replicas: 5
  selector:
    matchLabels:
      app: object-detector
  template:
    metadata:
      labels:
        app: object-detector
    spec:
      containers:
      - name: detector
        image: company/object-detector:latest
        resources:
          limits:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
        volumeMounts:
        - name: model-store
          mountPath: /models
      volumes:
      - name: model-store
        persistentVolumeClaim:
          claimName: model-data-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feature-extractor
spec:
  replicas: 4
  selector:
    matchLabels:
      app: feature-extractor
  template:
    metadata:
      labels:
        app: feature-extractor
    spec:
      containers:
      - name: extractor
        image: company/feature-extractor:latest
        resources:
          limits:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
```

### Natural Language Processing Service

```yaml
# NLP service architecture
apiVersion: v1
kind: ConfigMap
metadata:
  name: nlp-config
data:
  model-config.json: |
    {
      "transformer_model": "bert-base-uncased",
      "max_sequence_length": 128,
      "batch_size": 32,
      "use_gpu": true
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nlp-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nlp-service
  template:
    metadata:
      labels:
        app: nlp-service
    spec:
      containers:
      - name: nlp
        image: company/nlp-service:latest
        resources:
          limits:
            cpu: 4
            memory: 8Gi
            nvidia.com/gpu: 1
        volumeMounts:
        - name: config
          mountPath: /app/config
        - name: models
          mountPath: /app/models
        env:
        - name: MODEL_CONFIG_PATH
          value: "/app/config/model-config.json"
        - name: CUDA_VISIBLE_DEVICES
          value: "0"
        - name: TF_FORCE_GPU_ALLOW_GROWTH
          value: "true"
        ports:
        - containerPort: 8000
      volumes:
      - name: config
        configMap:
          name: nlp-config
      - name: models
        persistentVolumeClaim:
          claimName: nlp-models-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nlp-service
spec:
  selector:
    app: nlp-service
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
```

## Conclusion

Kubernetes provides a powerful and flexible platform for AI/ML workloads, offering solutions for the unique challenges of training, serving, and managing ML models at scale. By leveraging Kubernetes' native capabilities along with specialized tools like Kubeflow, KServe, and MLflow, organizations can build robust, scalable, and efficient ML platforms.

Key takeaways from this guide:

1. **Specialized Resource Management**: Kubernetes can efficiently schedule and manage specialized hardware like GPUs for ML workloads
2. **Workflow Orchestration**: Tools like Kubeflow, Argo Workflows, and Airflow help orchestrate complex ML pipelines
3. **Model Deployment**: Frameworks like KServe provide scalable, serverless model serving capabilities
4. **Distributed Training**: Kubernetes supports distributed training across multiple nodes for large-scale ML workloads
5. **MLOps Practices**: The platform enables CI/CD, monitoring, and reproducibility for ML systems
6. **Resource Optimization**: Features like auto-scaling ensure efficient resource utilization for variable ML workloads

As AI/ML continues to evolve, Kubernetes remains a foundational technology for building scalable, reliable, and efficient ML platforms.

## Additional Resources

- [Kubeflow Documentation](https://www.kubeflow.org/docs/)
- [KServe Documentation](https://kserve.github.io/website/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [Argo Workflows](https://argoproj.github.io/workflows/)
- [Feast Feature Store](https://docs.feast.dev/)
- [TensorFlow on Kubernetes](https://www.tensorflow.org/guide/distributed_training)
- [PyTorch on Kubernetes](https://pytorch.org/tutorials/intermediate/dist_tuto.html)
- [Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html)