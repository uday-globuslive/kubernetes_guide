# GPU and Specialized Hardware in Kubernetes

## Introduction

Modern computing workloads, particularly in fields like artificial intelligence, machine learning, data analytics, and high-performance computing, often require specialized hardware accelerators beyond traditional CPUs. Kubernetes has evolved to efficiently manage and schedule these specialized hardware resources, enabling organizations to build scalable, container-based infrastructure for compute-intensive applications.

This guide covers the integration of GPUs (Graphics Processing Units), FPGAs (Field Programmable Gate Arrays), TPUs (Tensor Processing Units), and other specialized hardware within Kubernetes environments. We'll explore configuration approaches, scheduling strategies, monitoring techniques, and best practices for managing these resources effectively.

## Understanding Hardware Accelerators

### Types of Hardware Accelerators in Kubernetes

| Accelerator Type | Primary Use Cases | Key Characteristics |
|--------------------|-------------------|---------------------|
| **GPUs** | Deep learning, rendering, scientific computing | High parallelism, CUDA/ROCm compatibility, memory bandwidth |
| **TPUs** | TensorFlow workloads, ML training/inference | AI/ML optimization, high matrix multiplication performance |
| **FPGAs** | Custom algorithms, signal processing, encryption | Programmable hardware, low latency, energy efficiency |
| **SmartNICs** | Network function virtualization, security | Offloaded network processing, hardware-accelerated networking |
| **Custom ASICs** | Domain-specific workloads | Purpose-built for specific algorithms, maximum efficiency |
| **DPUs** | Infrastructure processing, storage acceleration | Offloaded data processing, storage operations |

### Hardware Accelerator Architecture

While each accelerator type has its unique architecture, they share a common integration pattern in Kubernetes:

```
┌────────────────────────────────────────┐
│               Node                     │
│                                        │
│  ┌─────────────┐     ┌─────────────┐   │
│  │ Pod         │     │ Pod         │   │
│  │ ┌─────────┐ │     │ ┌─────────┐ │   │
│  │ │Container│ │     │ │Container│ │   │
│  │ └─────────┘ │     │ └─────────┘ │   │
│  │             │     │             │   │
│  │ Device      │     │ Device      │   │
│  │ Access      │     │ Access      │   │
│  └──────┬──────┘     └──────┬──────┘   │
│         │                   │          │
│         ▼                   ▼          │
│  ┌─────────────────────────────────┐   │
│  │       Device Plugin API         │   │
│  └─────────────────────────────────┘   │
│         │                   │          │
│         ▼                   ▼          │
│  ┌─────────┐         ┌─────────────┐   │
│  │ GPU 0   │         │ GPU 1       │   │
│  └─────────┘         └─────────────┘   │
│                                        │
└────────────────────────────────────────┘
```

## GPU Support in Kubernetes

### Basic GPU Configuration

Kubernetes supports GPUs as a schedulable resource using the device plugin framework. The most common implementation is NVIDIA's device plugin for NVIDIA GPUs.

1. **Install GPU drivers on nodes**

   The node must have the appropriate GPU drivers installed. For NVIDIA GPUs:

   ```bash
   # Example for Ubuntu
   sudo apt-get update
   sudo apt-get install -y nvidia-driver-470
   
   # Verify installation
   nvidia-smi
   ```

2. **Deploy the device plugin**

   ```yaml
   # nvidia-device-plugin.yaml
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

   Apply the configuration:

   ```bash
   kubectl apply -f nvidia-device-plugin.yaml
   ```

3. **Request GPUs in pod specification**

   ```yaml
   # gpu-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-pod
   spec:
     containers:
     - name: gpu-container
       image: nvidia/cuda:11.6.2-base-ubuntu20.04
       command: ["bash", "-c", "nvidia-smi && sleep infinity"]
       resources:
         limits:
           nvidia.com/gpu: 1 # Requesting 1 GPU
     restartPolicy: Never
   ```

### Multi-GPU Workloads

For workloads requiring multiple GPUs:

```yaml
# multi-gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-gpu-pod
spec:
  containers:
  - name: multi-gpu-container
    image: nvidia/cuda:11.6.2-base-ubuntu20.04
    command: ["bash", "-c", "nvidia-smi && sleep infinity"]
    resources:
      limits:
        nvidia.com/gpu: 4 # Requesting 4 GPUs
  restartPolicy: Never
```

### GPU Memory Management

Unlike CPU and memory, Kubernetes does not natively support partial GPU allocation. However, NVIDIA's Multi-Instance GPU (MIG) feature allows for GPU partitioning on supported hardware:

```yaml
# mig-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mig-pod
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.6.2-base-ubuntu20.04
    command: ["bash", "-c", "nvidia-smi && sleep infinity"]
    resources:
      limits:
        nvidia.com/gpu-1g.5gb: 1 # Requesting 1 MIG GPU slice with 5GB memory
  restartPolicy: Never
```

### Time-Slicing GPUs

For GPUs that don't support MIG, you can use NVIDIA's GPU time-slicing feature:

```yaml
# Enabling GPU time-slicing in the device plugin
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  # ... other fields
  template:
    spec:
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
        # ... other fields
```

## NVIDIA GPU Operator

For a more comprehensive approach to managing NVIDIA GPUs in Kubernetes, NVIDIA offers the GPU Operator, which manages all the components needed to run GPU workloads:

```bash
# Add the NVIDIA Helm repository
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# Install the GPU Operator
helm install --wait --generate-name \
  -n gpu-operator --create-namespace \
  nvidia/gpu-operator
```

The GPU Operator installs and manages:
- NVIDIA drivers
- NVIDIA Container Runtime
- Device Plugin
- GPU Feature Discovery
- DCGM exporter for monitoring
- Node Feature Discovery

## Other Hardware Accelerators

### Google TPUs

For Google Kubernetes Engine (GKE) with TPUs:

```yaml
# tpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tpu-pod
spec:
  containers:
  - name: tpu-container
    image: tensorflow/tensorflow:2.9.0
    command: ["python", "-c", "import tensorflow as tf; print(tf.config.list_physical_devices('TPU'))"]
    resources:
      limits:
        cloud-tpus.google.com/v3: 8 # Request 8 TPU v3 cores
  restartPolicy: Never
```

### AMD GPUs

For AMD GPUs using ROCm:

```yaml
# amd-gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rocm-pod
spec:
  containers:
  - name: rocm-container
    image: rocm/tensorflow:latest
    command: ["bash", "-c", "rocm-smi && sleep infinity"]
    resources:
      limits:
        amd.com/gpu: 1 # Requesting 1 AMD GPU
    securityContext:
      privileged: true
  restartPolicy: Never
```

### FPGAs

For Intel FPGAs using their device plugin:

```yaml
# fpga-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fpga-pod
spec:
  containers:
  - name: fpga-container
    image: intel/fpga-sample:latest
    command: ["sh", "-c", "sleep infinity"]
    resources:
      limits:
        intel.com/fpga-arria10: 1 # Requesting 1 Arria10 FPGA
  restartPolicy: Never
```

## Advanced Scheduling for Specialized Hardware

### Node Labeling and Selection

Label nodes with hardware information:

```bash
# Label a node with GPU type
kubectl label nodes worker-node-1 gpu-type=nvidia-a100
kubectl label nodes worker-node-2 gpu-type=nvidia-v100
```

Then use node selectors to target specific hardware:

```yaml
# a100-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: a100-pod
spec:
  nodeSelector:
    gpu-type: nvidia-a100
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.6.2-base-ubuntu20.04
    resources:
      limits:
        nvidia.com/gpu: 1
```

### Node Affinity for Hardware Preferences

Use node affinity for more sophisticated scheduling rules:

```yaml
# gpu-pod-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu-type
            operator: In
            values:
            - nvidia-a100
            - nvidia-v100
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 10
        preference:
          matchExpressions:
          - key: gpu-type
            operator: In
            values:
            - nvidia-a100
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.6.2-base-ubuntu20.04
    resources:
      limits:
        nvidia.com/gpu: 1
```

### Taints and Tolerations for Dedicated Hardware Nodes

Reserve GPU nodes for GPU workloads:

```bash
# Taint GPU nodes
kubectl taint nodes worker-node-1 dedicated=gpu:NoSchedule
```

Configure pods to tolerate the taint:

```yaml
# gpu-pod-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-toleration
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.6.2-base-ubuntu20.04
    resources:
      limits:
        nvidia.com/gpu: 1
```

### Extended Resources

Define custom hardware resources:

```bash
# Add 4 FPGA devices to a node
kubectl patch node worker-node-3 -p '{"status":{"capacity":{"example.com/fpga": "4"}}}'
```

Use the custom resource in pods:

```yaml
# custom-resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-resource-pod
spec:
  containers:
  - name: app
    image: custom/fpga-app:latest
    resources:
      limits:
        example.com/fpga: 1
```

## Monitoring and Managing Hardware Resources

### NVIDIA DCGM Exporter

Deploy NVIDIA's DCGM exporter for Prometheus monitoring:

```yaml
# dcgm-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dcgm-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  template:
    metadata:
      labels:
        app: dcgm-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9400"
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        nvidia.com/gpu: "true"
      containers:
      - name: dcgm-exporter
        image: nvcr.io/nvidia/k8s-device-plugin:v0.13.0-dcgm
        ports:
        - name: metrics
          containerPort: 9400
        securityContext:
          runAsNonRoot: false
          runAsUser: 0
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
```

### Prometheus Rules for GPU Monitoring

Create alerting rules for GPU health:

```yaml
# gpu-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alert-rules
  namespace: monitoring
spec:
  groups:
  - name: gpu.rules
    rules:
    - alert: GPUHighUtilization
      expr: DCGM_FI_DEV_GPU_UTIL > 90
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "GPU high utilization ({{ $labels.instance }})"
        description: "GPU utilization on {{ $labels.instance }} is above 90% for 15 minutes"
    
    - alert: GPUHighTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "GPU high temperature ({{ $labels.instance }})"
        description: "GPU temperature on {{ $labels.instance }} is above 85°C for 5 minutes"
    
    - alert: GPUMemoryErrorDetected
      expr: DCGM_FI_DEV_FB_MEMORY_USAGE > 0
      labels:
        severity: critical
      annotations:
        summary: "GPU memory error detected ({{ $labels.instance }})"
        description: "GPU memory error detected on {{ $labels.instance }}"
```

### Grafana Dashboard for GPU Metrics

Example Grafana dashboard configuration for GPU monitoring:

```json
{
  "annotations": {
    "list": []
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 19,
  "links": [],
  "panels": [
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 1,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "DCGM_FI_DEV_GPU_UTIL",
          "legendFormat": "GPU {{gpu}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "GPU Utilization",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": "100",
          "min": "0",
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 2,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "DCGM_FI_DEV_GPU_TEMP",
          "legendFormat": "GPU {{gpu}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "GPU Temperature",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "celsius",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "schemaVersion": 22,
  "style": "dark",
  "tags": [
    "gpu",
    "nvidia"
  ],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ]
  },
  "timezone": "",
  "title": "GPU Dashboard",
  "uid": "gpu",
  "version": 1
}
```

### AMD ROCm Monitoring

For AMD GPUs with ROCm, deploy the ROCm-SMI exporter:

```yaml
# rocm-smi-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rocm-smi-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: rocm-smi-exporter
  template:
    metadata:
      labels:
        app: rocm-smi-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9400"
    spec:
      tolerations:
      - key: amd.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        amd.com/gpu: "true"
      containers:
      - name: rocm-smi-exporter
        image: rocm/rocm-smi-exporter:latest
        ports:
        - name: metrics
          containerPort: 9400
        securityContext:
          privileged: true
        volumeMounts:
        - name: sys
          mountPath: /sys
          readOnly: true
      volumes:
      - name: sys
        hostPath:
          path: /sys
```

## Use Cases and Workload Types

### Machine Learning Training

Example of a TensorFlow training job with GPUs:

```yaml
# tensorflow-training.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: tensorflow-training
spec:
  backoffLimit: 4
  template:
    spec:
      nodeSelector:
        gpu-type: nvidia-a100
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:2.9.0-gpu
        command: ["python", "/code/train.py"]
        resources:
          limits:
            nvidia.com/gpu: 4
            cpu: "8"
            memory: "64Gi"
          requests:
            cpu: "4"
            memory: "32Gi"
        volumeMounts:
        - name: dataset
          mountPath: /data
        - name: code
          mountPath: /code
        - name: output
          mountPath: /output
      volumes:
      - name: dataset
        persistentVolumeClaim:
          claimName: ml-dataset
      - name: code
        configMap:
          name: training-code
      - name: output
        persistentVolumeClaim:
          claimName: ml-output
      restartPolicy: Never
```

### Distributed Training

Multi-node distributed training with Horovod:

```yaml
# distributed-training.yaml
apiVersion: v1
kind: Service
metadata:
  name: tensorflow-svc
spec:
  selector:
    app: tensorflow-training
  clusterIP: None
  ports:
  - port: 22
    name: ssh
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tensorflow-training
spec:
  serviceName: "tensorflow-svc"
  replicas: 4
  selector:
    matchLabels:
      app: tensorflow-training
  template:
    metadata:
      labels:
        app: tensorflow-training
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: tensorflow
        image: horovod/horovod:0.25.0-tf2.9.0-torch1.11.0-mxnet1.9.1-py3.9-cuda11.2
        command:
        - /bin/bash
        - -c
        - |
          if [[ $HOSTNAME == tensorflow-training-0 ]]; then
            # Master node
            ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
            cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
            # Share pubkey with worker nodes
            # Start training script
            horovodrun -np 16 -H tensorflow-training-0:4,tensorflow-training-1:4,tensorflow-training-2:4,tensorflow-training-3:4 python /code/train.py
          else
            # Worker node
            # Wait for master's pub key and add to authorized keys
            # Then wait for horovod command
            sleep infinity
          fi
        resources:
          limits:
            nvidia.com/gpu: 4
          requests:
            nvidia.com/gpu: 4
            cpu: "8"
            memory: "32Gi"
        volumeMounts:
        - name: ssh-key
          mountPath: /root/.ssh
        - name: dataset
          mountPath: /data
        - name: code
          mountPath: /code
      volumes:
      - name: dataset
        persistentVolumeClaim:
          claimName: ml-dataset
      - name: code
        configMap:
          name: training-code
  volumeClaimTemplates:
  - metadata:
      name: ssh-key
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### Inference Services

GPU-accelerated inference service:

```yaml
# inference-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-inference
  template:
    metadata:
      labels:
        app: model-inference
    spec:
      containers:
      - name: tensorrt-server
        image: nvcr.io/nvidia/tensorrtserver:21.08-py3
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8001
          name: grpc
        - containerPort: 8002
          name: metrics
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
            cpu: "4"
            memory: "8Gi"
        volumeMounts:
        - name: model-repository
          mountPath: /models
        readinessProbe:
          httpGet:
            path: /api/health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: model-repository
        persistentVolumeClaim:
          claimName: model-repository
---
apiVersion: v1
kind: Service
metadata:
  name: inference-service
spec:
  selector:
    app: model-inference
  ports:
  - port: 8000
    name: http
    targetPort: 8000
  - port: 8001
    name: grpc
    targetPort: 8001
```

### HPC Workloads

Scientific computing job with MPI:

```yaml
# mpi-job.yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: physics-simulation
spec:
  slotsPerWorker: 4
  runPolicy:
    cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
          - name: launcher
            image: mpioperator/mpi-sample:latest
            command:
            - mpirun
            - --allow-run-as-root
            - -np
            - "16"
            - python
            - /physics/simulation.py
    Worker:
      replicas: 4
      template:
        spec:
          nodeSelector:
            gpu-type: nvidia-a100
          containers:
          - name: worker
            image: mpioperator/mpi-sample:latest
            resources:
              limits:
                nvidia.com/gpu: 4
                cpu: "32"
                memory: "128Gi"
              requests:
                nvidia.com/gpu: 4
                cpu: "16"
                memory: "64Gi"
            volumeMounts:
            - name: physics-data
              mountPath: /physics/data
            - name: physics-output
              mountPath: /physics/output
          volumes:
          - name: physics-data
            persistentVolumeClaim:
              claimName: physics-data
          - name: physics-output
            persistentVolumeClaim:
              claimName: physics-output
```

## Best Practices for Specialized Hardware in Kubernetes

### Resource Efficiency

1. **Right-size hardware requests**

   Always specify both resource requests and limits:

   ```yaml
   resources:
     requests:
       nvidia.com/gpu: 1
       cpu: "4"
       memory: "16Gi"
     limits:
       nvidia.com/gpu: 1
       cpu: "8"
       memory: "32Gi"
   ```

2. **Use GPU fractioning when appropriate**

   Consider NVIDIA MIG for more granular GPU allocation:

   ```bash
   # Configure A100 GPU with 7 MIG instances
   nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C
   ```

3. **Implement auto-scaling based on hardware utilization**

   ```yaml
   # gpu-hpa.yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: inference-scaler
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: model-inference
     minReplicas: 1
     maxReplicas: 10
     metrics:
     - type: External
       external:
         metric:
           name: nvidia_gpu_duty_cycle
         target:
           type: AverageValue
           averageValue: 70
   ```

### Hardware Isolation

1. **Implement GPU isolation with taints**

   ```bash
   # Taint nodes with expensive GPUs
   kubectl taint nodes a100-node-1 gpu-type=a100:NoSchedule
   ```

2. **Use pod anti-affinity for workload separation**

   ```yaml
   # gpu-pod-anti-affinity.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-workload
   spec:
     affinity:
       podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
               - high-priority-job
           topologyKey: kubernetes.io/hostname
     containers:
     - name: cuda-container
       image: nvidia/cuda:11.6.2-base-ubuntu20.04
       resources:
         limits:
           nvidia.com/gpu: 1
   ```

3. **Reserve specialized hardware for specific tenants**

   ```yaml
   # Create a ResourceQuota for GPU usage
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: gpu-quota
     namespace: team-ml
   spec:
     hard:
       requests.nvidia.com/gpu: 8
       limits.nvidia.com/gpu: 8
   ```

### Performance Optimization

1. **Set appropriate NVIDIA CUDA settings**

   ```yaml
   # performance-optimized-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: optimized-gpu-pod
   spec:
     containers:
     - name: cuda-container
       image: nvidia/cuda:11.6.2-base-ubuntu20.04
       command: ["python", "/app/train.py"]
       env:
       - name: NVIDIA_DRIVER_CAPABILITIES
         value: "compute,utility"
       - name: CUDA_CACHE_PATH
         value: "/tmp/cuda-cache"
       - name: NVIDIA_VISIBLE_DEVICES
         value: "all"
       resources:
         limits:
           nvidia.com/gpu: 1
       volumeMounts:
       - name: cuda-cache
         mountPath: /tmp/cuda-cache
     volumes:
     - name: cuda-cache
       emptyDir: {}
   ```

2. **Enable GPU Direct for faster data transfer**

   For workloads that benefit from GPU Direct (RDMA), use host network:

   ```yaml
   # gpudirect-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpudirect-pod
   spec:
     hostNetwork: true
     containers:
     - name: gpudirect-container
       image: nvidia/cuda:11.6.2-base-ubuntu20.04
       securityContext:
         privileged: true
       resources:
         limits:
           nvidia.com/gpu: 1
   ```

3. **Implement topology awareness for multi-GPU workloads**

   Use topology manager to align CPU, GPU, and memory resources:

   ```yaml
   # kubelet configuration
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   topologyManagerPolicy: best-effort
   ```

### Security Considerations

1. **Limit access to specialized hardware**

   Use RBAC to control who can schedule hardware resources:

   ```yaml
   # gpu-resource-access.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: gpu-user
     namespace: ml-team
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: gpu-users
     namespace: ml-team
   subjects:
   - kind: Group
     name: ml-engineers
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: Role
     name: gpu-user
     apiGroup: rbac.authorization.k8s.io
   ```

2. **Implement pod security policies for GPU workloads**

   ```yaml
   # gpu-psp.yaml
   apiVersion: policy/v1beta1
   kind: PodSecurityPolicy
   metadata:
     name: gpu-psp
   spec:
     privileged: false
     allowPrivilegeEscalation: false
     requiredDropCapabilities:
     - ALL
     volumes:
     - 'configMap'
     - 'emptyDir'
     - 'persistentVolumeClaim'
     hostNetwork: false
     hostIPC: false
     hostPID: false
     runAsUser:
       rule: 'MustRunAsNonRoot'
     seLinux:
       rule: 'RunAsAny'
     supplementalGroups:
       rule: 'RunAsAny'
     fsGroup:
       rule: 'RunAsAny'
   ```

3. **Secure container images for GPU workloads**

   ```bash
   # Regularly scan GPU container images
   trivy image nvidia/cuda:11.6.2-base-ubuntu20.04
   ```

## Troubleshooting GPU Issues in Kubernetes

### Common Issues and Solutions

1. **GPU not visible to container**

   Symptoms:
   - `nvidia-smi` returns "No devices were found"
   - Container doesn't see the GPU

   Troubleshooting steps:
   ```bash
   # Check if GPU is visible on the host
   kubectl exec -it <gpu-pod> -- nvidia-smi
   
   # Check device plugin logs
   kubectl logs -n kube-system -l k8s-app=nvidia-device-plugin
   
   # Verify GPU resource allocation
   kubectl describe node <node-name> | grep nvidia.com/gpu
   ```

   Solutions:
   - Ensure the NVIDIA device plugin is running
   - Check that the container has proper GPU resource requests
   - Verify the NVIDIA driver is installed on the host

2. **"Failed to initialize NVML" error**

   Symptoms:
   - Container shows "Failed to initialize NVML: Driver/library version mismatch"

   Troubleshooting steps:
   ```bash
   # Check NVIDIA driver version on host
   kubectl exec -it -n kube-system <device-plugin-pod> -- nvidia-smi
   
   # Check NVIDIA driver version in container
   kubectl exec -it <gpu-pod> -- nvidia-smi
   ```

   Solutions:
   - Ensure container CUDA version is compatible with host driver
   - Use NVIDIA GPU Operator to manage driver compatibility

3. **GPU memory leaks**

   Symptoms:
   - GPU memory doesn't release after pods terminate
   - `nvidia-smi` shows memory in use with no active processes

   Troubleshooting steps:
   ```bash
   # Check GPU memory usage
   kubectl exec -it -n kube-system <device-plugin-pod> -- nvidia-smi
   
   # Check for zombie processes
   kubectl exec -it -n kube-system <device-plugin-pod> -- ps aux | grep defunct
   ```

   Solutions:
   - Restart the kubelet
   - Apply proper CUDA memory management in applications
   - Consider implementing pod preStop hooks to clean up GPU resources

### Debugging Tools

1. **NVIDIA System Management Interface (SMI)**

   ```bash
   # Basic GPU information
   kubectl exec -it <gpu-pod> -- nvidia-smi
   
   # GPU utilization and memory usage over time
   kubectl exec -it <gpu-pod> -- nvidia-smi dmon
   
   # Process-level GPU usage
   kubectl exec -it <gpu-pod> -- nvidia-smi pmon
   ```

2. **NVIDIA Debug Tools**

   ```bash
   # Install nvidia-debugdump in container
   kubectl exec -it <gpu-pod> -- apt-get update
   kubectl exec -it <gpu-pod> -- apt-get install -y nvidia-cuda-toolkit
   
   # Get detailed GPU info
   kubectl exec -it <gpu-pod> -- nvidia-debugdump --list
   ```

3. **GPU Feature Discovery**

   ```bash
   # Check discovered GPU features
   kubectl get node <node-name> -o json | jq '.metadata.labels | with_entries(select(.key | startswith("nvidia.com")))'
   ```

## GPU Cost Optimization in Kubernetes

### Strategies for Reducing GPU Costs

1. **Use GPU time-slicing for low-intensity workloads**

   Enable GPU time-slicing in the device plugin to share GPUs across pods:

   ```yaml
   # time-slicing-plugin.yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: nvidia-device-plugin-daemonset
     namespace: kube-system
   spec:
     # ... other fields
     template:
       spec:
         containers:
         - name: nvidia-device-plugin-ctr
           env:
           - name: NVIDIA_MPS_ENABLE_TIME_SLICING
             value: "4"  # Share each GPU among 4 containers
   ```

2. **Implement spot instances for fault-tolerant workloads**

   For interruptible GPU workloads like ML training:

   ```yaml
   # spot-gpu-training.yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: spot-training
   spec:
     backoffLimit: 5
     template:
       spec:
         nodeSelector:
           cloud.google.com/gke-spot: "true"
         tolerations:
         - key: cloud.google.com/gke-spot
           operator: Equal
           value: "true"
           effect: NoSchedule
         containers:
         - name: training
           image: tensorflow/tensorflow:2.9.0-gpu
           command: ["python", "/app/train.py", "--checkpoint-path=/checkpoints"]
           resources:
             limits:
               nvidia.com/gpu: 1
           volumeMounts:
           - name: checkpoints
             mountPath: /checkpoints
         volumes:
         - name: checkpoints
           persistentVolumeClaim:
             claimName: training-checkpoints
         restartPolicy: OnFailure
   ```

3. **Autoscale GPU nodes based on workload**

   ```yaml
   # gpu-cluster-autoscaler.yaml
   apiVersion: autoscaling.openshift.io/v1
   kind: ClusterAutoscaler
   metadata:
     name: gpu-cluster-autoscaler
   spec:
     resourceLimits:
       maxNodesTotal: 20
     scaleDown:
       enabled: true
       delayAfterAdd: 10m
       delayAfterDelete: 5m
       delayAfterFailure: 3m
       unneededTime: 10m
   ---
   apiVersion: autoscaling.openshift.io/v1
   kind: MachineAutoscaler
   metadata:
     name: gpu-worker-us-east-1a
     namespace: openshift-machine-api
   spec:
     minReplicas: 0
     maxReplicas: 5
     scaleTargetRef:
       apiVersion: machine.openshift.io/v1beta1
       kind: MachineSet
       name: gpu-worker-us-east-1a
   ```

4. **Schedule batch workloads during off-peak hours**

   ```yaml
   # overnight-training.yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: overnight-training
   spec:
     schedule: "0 20 * * 1-5"  # 8PM on weekdays
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: training
               image: tensorflow/tensorflow:2.9.0-gpu
               command: ["python", "/app/train.py"]
               resources:
                 limits:
                   nvidia.com/gpu: 4
             restartPolicy: OnFailure
   ```

## Cloud-Specific GPU Configurations

### AWS EKS with GPU Nodes

1. **Launch GPU node group**

   ```bash
   # Create GPU node group using eksctl
   eksctl create nodegroup \
     --cluster=my-cluster \
     --name=gpu-nodes \
     --node-type=p3.2xlarge \
     --nodes=2 \
     --nodes-min=0 \
     --nodes-max=4 \
     --managed
   ```

2. **Deploy NVIDIA device plugin**

   ```bash
   # Apply the device plugin
   kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml
   ```

### GCP GKE with GPU Nodes

1. **Create GPU node pool**

   ```bash
   # Create GKE GPU node pool
   gcloud container node-pools create gpu-pool \
     --cluster=my-cluster \
     --accelerator=type=nvidia-tesla-t4,count=1 \
     --machine-type=n1-standard-4 \
     --num-nodes=2 \
     --min-nodes=0 \
     --max-nodes=4 \
     --enable-autoscaling
   ```

2. **Install NVIDIA drivers (automatic on GKE)**

   GKE automatically installs NVIDIA drivers on GPU nodes

### Azure AKS with GPU Nodes

1. **Create GPU node pool**

   ```bash
   # Create AKS GPU node pool
   az aks nodepool add \
     --resource-group myResourceGroup \
     --cluster-name myAKSCluster \
     --name gpunodepool \
     --node-count 2 \
     --node-vm-size Standard_NC6s_v3 \
     --min-count 0 \
     --max-count 4 \
     --enable-cluster-autoscaler
   ```

2. **Apply NVIDIA device plugin**

   ```bash
   # AKS can automatically apply the device plugin with this command
   az aks enable-addons \
     --resource-group myResourceGroup \
     --name myAKSCluster \
     --addons nvidia-device-plugin
   ```

## Future Trends in Specialized Hardware

### GPU-Powered Kubernetes at Scale

As organizations scale their GPU deployments, advanced multi-cluster management becomes essential:

```yaml
# Example of multi-cluster scheduler for GPU workloads
apiVersion: scheduling.karmada.io/v1alpha1
kind: ResourceBinding
metadata:
  name: gpu-job-binding
spec:
  resource:
    apiVersion: batch/v1
    kind: Job
    namespace: default
    name: gpu-training-job
  clusters:
  - name: cluster1
    replicas: 1
  - name: cluster2
    replicas: 1
  replicaRequirements:
    nodeClaim:
      nodeSelector:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: nvidia.com/gpu
              operator: Exists
            - key: gpu-type
              operator: In
              values:
              - nvidia-a100
```

### Specialized AI Hardware Integration

As new AI accelerators emerge, Kubernetes will adapt to support them:

```yaml
# Example of hypothetical future AI accelerator
apiVersion: v1
kind: Pod
metadata:
  name: ai-accelerator-pod
spec:
  containers:
  - name: ai-container
    image: ai-framework:latest
    resources:
      limits:
        custom.accelerator/neuromorphic-unit: 1
```

### Serverless GPU Workloads

Kubernetes platforms will increasingly support serverless GPU compute:

```yaml
# Example of GPU-enabled Knative service
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: gpu-inference
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "10"
    spec:
      containers:
      - image: my-repo/gpu-inference:latest
        resources:
          limits:
            nvidia.com/gpu: 1
```

## Conclusion

Specialized hardware accelerators significantly enhance Kubernetes capabilities for compute-intensive workloads. Whether you're running deep learning training, inference services, or scientific simulations, Kubernetes provides the infrastructure to efficiently manage these valuable resources.

Key takeaways from this document:

1. **Device Plugins** enable Kubernetes to manage specialized hardware resources
2. **Advanced Scheduling** techniques help optimize hardware allocation
3. **Monitoring Tools** provide visibility into hardware utilization and health
4. **Cost Optimization** strategies help manage expensive hardware resources
5. **Security Considerations** are essential when managing valuable hardware

As specialized hardware continues to evolve, Kubernetes will remain at the forefront of orchestrating these resources in containerized environments, enabling organizations to build scalable, efficient platforms for their most demanding computational workloads.

## Additional Resources

- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
- [Kubernetes Device Plugins Documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA DCGM for Kubernetes](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/kubernetes.html)
- [TensorFlow on Kubernetes Guide](https://www.tensorflow.org/guide/distributed_training)
- [Kubeflow Documentation](https://www.kubeflow.org/docs/)
- [KubeVirt for GPU Virtualization](https://kubevirt.io/user-guide/virtual_machines/dedicated_cpu_resources/)
- [AMD ROCm Kubernetes Documentation](https://github.com/RadeonOpenCompute/k8s-device-plugin)