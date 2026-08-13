# Day 23 — GPU Scheduling, Resource Topology & AI/ML Workloads

This is the bridge between pure Kubernetes mastery and the AI/ML integration week. The market for engineers who understand both K8s and ML infrastructure is tiny and extremely well-paid. Today you build the foundation.

## 🧠 Part 1: Why AI/ML on Kubernetes

*Before 2020 ML teams ran on bare metal or VMs with manual setup. The problems:*

```
Resource waste       — GPU sits idle between training runs
Reproducibility      — "works on my machine" for training jobs
Dependency hell      — CUDA versions, driver conflicts per team
Scheduling           — who gets the A100 when 5 teams want it
Scaling              — manually provision more nodes for large jobs
Experiment tracking  — no lineage between code, data, model
```

Kubernetes solves all of these. A training job becomes a K8s Job. GPU allocation becomes a resource request. Scaling becomes HPA or KEDA. Experiment tracking integrates via init containers and sidecars.

## 🔧 Part 2: GPU Device Plugin — How K8s Knows About GPUs

Kubernetes doesn't natively understand GPUs. The Device Plugin framework lets hardware vendors extend the K8s resource model.

**How device plugins work**

```
1. Vendor writes a DaemonSet (Device Plugin)
2. Plugin runs on every GPU node
3. Plugin registers with kubelet via gRPC on /var/lib/kubelet/device-plugins/
4. Plugin reports available devices (GPU UUIDs) to kubelet
5. kubelet advertises nvidia.com/gpu: 4 as a node resource
6. Scheduler uses this extended resource for placement
7. When pod is assigned, plugin allocates specific GPUs
8. Plugin passes GPU device paths to container runtime
```

```
# Install NVIDIA GPU Operator — handles driver + device plugin + everything
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true \
  --set devicePlugin.enabled=true \
  --set dcgmExporter.enabled=true    # GPU metrics for Prometheus

# What GPU Operator deploys:
kubectl get pods -n gpu-operator
# gpu-operator                     ← operator controller
# nvidia-driver-daemonset          ← installs NVIDIA driver on nodes
# nvidia-container-toolkit         ← container runtime hook
# nvidia-device-plugin-daemonset   ← registers GPUs with kubelet
# nvidia-dcgm-exporter-daemonset   ← GPU metrics (temp, utilization, memory)

# Verify GPUs are registered with K8s
kubectl get nodes -o json \
  | jq '.items[] | {name: .metadata.name, gpus: .status.allocatable["nvidia.com/gpu"]}'
# {
#   "name": "gpu-node-1",
#   "gpus": "4"           ← 4 GPUs available
# }

# Describe a GPU node to see all extended resources
kubectl describe node gpu-node-1 | grep -A 20 "Allocatable:"
# Allocatable:
#   cpu:                   96
#   memory:                755Gi
#   nvidia.com/gpu:        8      ← 8 A100 GPUs
#   hugepages-1Gi:         0
#   hugepages-2Mi:         0
```

## 📊 Part 3: Requesting GPUs in Pods

```
# Basic GPU pod
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training
spec:
  containers:
  - name: trainer
    image: nvcr.io/nvidia/pytorch:23.10-py3
    resources:
      limits:
        nvidia.com/gpu: 1           # request 1 GPU
        cpu: "8"
        memory: 32Gi
      requests:
        nvidia.com/gpu: 1           # must equal limits for GPU
        cpu: "8"
        memory: 32Gi
    command: ["python", "train.py"]
```

**Critical:** 

GPU resources must have requests == limits. You cannot set a GPU request without a matching limit. GPUs are not overcommittable — if you request one, you get one dedicated GPU.

**Multi-GPU training job**

```
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training
spec:
  completions: 1
  parallelism: 1
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: trainer
        image: nvcr.io/nvidia/pytorch:23.10-py3
        resources:
          limits:
            nvidia.com/gpu: 4       # 4 GPUs on one node
            cpu: "32"
            memory: 128Gi
        env:
        - name: NVIDIA_VISIBLE_DEVICES
          value: all                # or specific GPU UUIDs
        - name: NCCL_DEBUG
          value: INFO               # NCCL communication debug
        command:
        - torchrun
        - --nproc_per_node=4        # 4 processes, one per GPU
        - train.py
        volumeMounts:
        - name: shm
          mountPath: /dev/shm       # shared memory for multi-process
      volumes:
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 16Gi           # PyTorch multi-GPU needs large /dev/shm
```

**GPU sharing — MIG and Time-Slicing**

```
# Problem: A100 is expensive. Small models don't need a full GPU.
# Two solutions: MIG (hardware partitioning) and Time-Slicing (software)

# MIG (Multi-Instance GPU) — A100/H100 only
# Physically partitions GPU into isolated slices
# Each slice has dedicated compute + memory

# Node labels after MIG configuration:
# nvidia.com/mig-1g.5gb: 7    ← 7 instances of 1/7 GPU, 5GB each
# nvidia.com/mig-2g.10gb: 3   ← 3 instances of 2/7 GPU, 10GB each
# nvidia.com/mig-4g.20gb: 1   ← 1 instance of 4/7 GPU, 20GB

# Request a MIG slice
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1   # 1/7 of an A100

---
# Time-Slicing — all GPU types
# Multiple pods share one GPU via time multiplexing
# No memory isolation — pods can see each other's memory
# Good for: inference, dev workloads, not training

# ConfigMap for time-slicing config
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        replicas: 4              # make one GPU look like 4
        failRequestsGreaterThanOne: false

# After applying: nvidia.com/gpu: 4 per physical GPU
# Pods request nvidia.com/gpu: 1 and get time-shared access
```

## 🔍 Part 4: Resource Topology — NUMA Awareness

Modern servers have Non-Uniform Memory Access (NUMA) architecture. A GPU on NUMA node 0 accessing memory from NUMA node 1 crosses the QPI/UPI bus — this adds latency and kills ML training performance.

```
NUMA Node 0:              NUMA Node 1:
  CPU 0-47                  CPU 48-95
  Memory 0-512GB            Memory 512GB-1TB
  GPU 0-3                   GPU 4-7
  NIC (eth0)                NIC (eth1)

Cross-NUMA bandwidth: ~100 GB/s
Local NUMA bandwidth: ~400 GB/s
```

**Topology Manager — align GPU, CPU, and memory to same NUMA node**

```
# Enable Topology Manager on kubelet
# /var/lib/kubelet/config.yaml on each node:
# topologyManagerPolicy: best-effort  # try to align, don't fail if impossible
# topologyManagerPolicy: restricted   # fail if can't align
# topologyManagerPolicy: single-numa-node  # must fit on one NUMA node

# Check node topology
kubectl get node gpu-node-1 \
  -o jsonpath='{.status.capacity}' | jq .

# See NUMA topology (on the node)
docker exec k8s-mastery-worker \
  cat /sys/devices/system/node/node0/cpulist 2>/dev/null \
  || echo "NUMA info not available in kind"

# numactl on the node shows NUMA topology
docker exec k8s-mastery-worker \
  numactl --hardware 2>/dev/null || \
  echo "numactl not available"
```

**CPU Manager — dedicated CPUs for latency-sensitive workloads**

```
# CPU Manager policy: static — gives Guaranteed pods dedicated CPUs
# Prevents noisy neighbour CPU cache thrashing

# Enable on kubelet:
# cpuManagerPolicy: static

# For a pod to get dedicated CPUs:
# 1. QoS must be Guaranteed (requests == limits)
# 2. CPU request must be integer (not 100m — must be "1" or "2")

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dedicated-cpu-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "2"          # integer CPU = gets dedicated cores
        memory: 1Gi
      limits:
        cpu: "2"          # must equal requests
        memory: 1Gi
EOF

# Verify dedicated CPUs (on the node)
# cat /sys/fs/cgroup/cpuset/kubepods/guaranteed/<pod-uid>/cpuset.cpus
# Shows: 4,5  ← these CPUs are exclusively for this pod
```

## 🏋️ Part 5: Training Jobs — Kubernetes Patterns

**Simple batch training job**

```
apiVersion: batch/v1
kind: Job
metadata:
  name: model-training-v1
  labels:
    experiment: bert-finetune
    version: v1
spec:
  completions: 1
  backoffLimit: 2
  activeDeadlineSeconds: 86400    # 24 hour max
  template:
    metadata:
      labels:
        app: trainer
        experiment: bert-finetune
    spec:
      restartPolicy: OnFailure

      # Schedule on GPU nodes
      nodeSelector:
        accelerator: nvidia-a100

      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule        # GPU nodes are often tainted

      initContainers:
      # Download training data before starting
      - name: data-downloader
        image: amazon/aws-cli:latest
        command:
        - aws
        - s3
        - sync
        - s3://ml-datasets/bert-data/
        - /data/
        volumeMounts:
        - name: training-data
          mountPath: /data
        env:
        - name: AWS_ROLE_ARN
          value: arn:aws:iam::123456789:role/ml-trainer

      containers:
      - name: trainer
        image: ghcr.io/youruser/bert-trainer:v1.2.3
        command:
        - python
        - train.py
        - --model=bert-base
        - --epochs=10
        - --batch-size=32
        - --output-dir=/models/bert-v1
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: "8"
            memory: 32Gi
          requests:
            nvidia.com/gpu: 1
            cpu: "8"
            memory: 32Gi
        env:
        - name: WANDB_API_KEY    # experiment tracking
          valueFrom:
            secretKeyRef:
              name: ml-secrets
              key: wandb-api-key
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow.mlops.svc:5000
        volumeMounts:
        - name: training-data
          mountPath: /data
        - name: model-output
          mountPath: /models
        - name: shm
          mountPath: /dev/shm

      volumes:
      - name: training-data
        persistentVolumeClaim:
          claimName: ml-dataset-pvc
      - name: model-output
        persistentVolumeClaim:
          claimName: model-storage-pvc
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 8Gi
```

**Distributed training with PyTorchJob (KubeFlow)**

```
# PyTorchJob CRD from KubeFlow Training Operator
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: bert-distributed-training
  namespace: ml-team
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: ghcr.io/youruser/bert-trainer:v1.2.3
            resources:
              limits:
                nvidia.com/gpu: 4
                cpu: "32"
                memory: 128Gi

    Worker:
      replicas: 3               # 3 worker nodes × 4 GPUs = 12 GPUs total
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: ghcr.io/youruser/bert-trainer:v1.2.3
            resources:
              limits:
                nvidia.com/gpu: 4
                cpu: "32"
                memory: 128Gi
            env:
            - name: NCCL_SOCKET_IFNAME
              value: eth0
            - name: NCCL_IB_DISABLE
              value: "1"        # disable InfiniBand if not available
```

**Gang scheduling — all pods or none**

Standard K8s scheduler places pods independently. For distributed training, if worker-0 is placed but worker-1 can't be placed (no GPU available), worker-0 sits idle. Gang scheduling places all pods simultaneously or holds all until resources are available.

```
# Volcano — gang scheduling for ML workloads
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: gang-training
spec:
  minAvailable: 4               # all 4 must be available before ANY starts
  schedulerName: volcano
  policies:
  - event: PodEvicted
    action: RestartJob          # if any pod evicted, restart entire job
  tasks:
  - replicas: 4
    name: trainer
    policies:
    - event: TaskCompleted
      action: CompleteJob
    template:
      spec:
        containers:
        - name: trainer
          image: ghcr.io/youruser/trainer:v1
          resources:
            limits:
              nvidia.com/gpu: 1
```

## 📈 Part 6: GPU Metrics and Monitoring

```
# DCGM Exporter exposes GPU metrics to Prometheus
# Install via GPU Operator (done in Part 2)

# Key GPU metrics to monitor:
```

```
# GPU utilization per GPU (should be >80% during training)
DCGM_FI_DEV_GPU_UTIL{namespace="ml-team"}

# GPU memory used vs total
DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_FREE

# GPU temperature (alert if > 85°C)
DCGM_FI_DEV_GPU_TEMP > 85

# GPU power usage (efficiency metric)
DCGM_FI_DEV_POWER_USAGE

# NVLink bandwidth (multi-GPU communication)
DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL

# PCIe bandwidth (GPU to CPU memory transfers)
DCGM_FI_DEV_PCIE_TX_THROUGHPUT

# GPU allocation efficiency (are GPUs sitting idle?)
# Count of pods using GPUs vs GPUs allocated
count(kube_pod_container_resource_limits{resource="nvidia_com_gpu"}) /
  count(DCGM_FI_DEV_GPU_UTIL > 0)
```

```
# Alert: GPU sitting idle (allocated but not used)
- alert: GPUUnderutilized
  expr: |
    DCGM_FI_DEV_GPU_UTIL < 10
    and
    kube_pod_container_resource_limits{resource="nvidia_com_gpu"} > 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "GPU allocated but underutilized"
    description: "GPU {{ $labels.gpu }} on {{ $labels.node }} at {{ $value }}% utilization"

# Alert: GPU memory pressure
- alert: GPUMemoryHigh
  expr: |
    DCGM_FI_DEV_FB_USED /
    (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) > 0.95
  for: 5m
  labels:
    severity: warning

# Alert: GPU temperature critical
- alert: GPUTemperatureCritical
  expr: DCGM_FI_DEV_GPU_TEMP > 90
  for: 2m
  labels:
    severity: critical
```

## 🚀 Part 7: Model Serving on Kubernetes

Training produces a model. Serving makes it accessible via API. Two patterns: custom serving (just a Deployment) and KServe (production ML serving platform).

**Simple model serving deployment**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bert-inference
  namespace: ml-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bert-inference
  template:
    metadata:
      labels:
        app: bert-inference
    spec:
      containers:
      - name: inference-server
        image: ghcr.io/youruser/bert-server:v1
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: metrics
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: "4"
            memory: 16Gi
          requests:
            nvidia.com/gpu: 1
            cpu: "4"
            memory: 16Gi
        env:
        - name: MODEL_PATH
          value: /models/bert-v1
        - name: MAX_BATCH_SIZE
          value: "32"
        - name: MAX_SEQUENCE_LENGTH
          value: "512"
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 30    # model loading takes time
          periodSeconds: 10
          failureThreshold: 12       # 2 minutes to load
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 30
          failureThreshold: 3
        volumeMounts:
        - name: model-store
          mountPath: /models
          readOnly: true             # inference only reads models
      volumes:
      - name: model-store
        persistentVolumeClaim:
          claimName: model-storage-pvc
```

**KServe — production ML serving**

```
# KServe provides: auto-scaling, canary, A/B testing, explainability
# Install KServe
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml

# InferenceService CRD
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: bert-classifier
  namespace: ml-serving
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch           # pytorch | tensorflow | sklearn | xgboost | onnx
      storageUri: s3://ml-models/bert-v1/
      resources:
        requests:
          nvidia.com/gpu: "1"
          cpu: "4"
          memory: 8Gi
        limits:
          nvidia.com/gpu: "1"
          cpu: "4"
          memory: 16Gi
      runtimeVersion: "2.0.0"

  # Canary rollout — 20% to new model
  canaryTrafficPercent: 20

  # Transformer — pre/post process requests
  transformer:
    containers:
    - name: transformer
      image: ghcr.io/youruser/bert-preprocessor:v1
      resources:
        requests: {cpu: "1", memory: 2Gi}

  # Explainer — model explanations (SHAP, LIME)
  explainer:
    alibi:
      type: AnchorText
      resources:
        requests: {cpu: "1", memory: 2Gi}
```

**KEDA — event-driven autoscaling for inference**

HPA scales on CPU/memory. Inference workloads scale on queue depth or request rate. KEDA (Kubernetes Event-Driven Autoscaling) handles this.

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace
```

```
# ScaledObject — scale inference deployment based on request queue
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: bert-inference-scaler
  namespace: ml-serving
spec:
  scaleTargetRef:
    name: bert-inference
  pollingInterval: 15             # check every 15 seconds
  cooldownPeriod: 300             # wait 5 min before scaling down
  minReplicaCount: 1              # keep at least 1 (cold start)
  maxReplicaCount: 10
  triggers:
  # Scale based on Prometheus metric — requests per second per replica
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: inference_requests_per_second
      query: |
        sum(rate(
          http_requests_total{
            service="bert-inference",
            endpoint="/predict"
          }[1m]
        )) / count(kube_pod_info{pod=~"bert-inference.*"})
      threshold: "10"             # scale up when >10 RPS per replica

  # Scale to zero when SQS queue is empty (batch inference)
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.eu-west-1.amazonaws.com/123456/inference-queue
      queueLength: "5"            # scale up when >5 messages waiting
      awsRegion: eu-west-1
```

## 🗂️ Part 8: MLflow on Kubernetes — Experiment Tracking

```
# MLflow tracking server — deployed as a K8s service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow
  namespace: mlops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow
  template:
    metadata:
      labels:
        app: mlflow
    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.9.0
        command:
        - mlflow
        - server
        - --backend-store-uri=postgresql://mlflow:password@postgres.mlops.svc:5432/mlflow
        - --default-artifact-root=s3://ml-artifacts/
        - --host=0.0.0.0
        - --port=5000
        ports:
        - containerPort: 5000
        resources:
          requests: {cpu: "1", memory: 2Gi}
          limits: {cpu: "2", memory: 4Gi}
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: ml-secrets
              key: aws-access-key
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow
  namespace: mlops
spec:
  selector:
    app: mlflow
  ports:
  - port: 5000
    name: http
```

```
# In your training job — log to MLflow running in cluster
import mlflow
import os

mlflow.set_tracking_uri(
    os.environ.get("MLFLOW_TRACKING_URI",
    "http://mlflow.mlops.svc:5000")
)

with mlflow.start_run(run_name=f"bert-{os.environ.get('JOB_NAME')}"):
    # Log hyperparameters
    mlflow.log_params({
        "model": "bert-base",
        "epochs": 10,
        "batch_size": 32,
        "learning_rate": 3e-5,
        "pod": os.environ.get("POD_NAME"),
        "node": os.environ.get("NODE_NAME")
    })

    for epoch in range(10):
        loss, accuracy = train_epoch()

        # Log metrics per epoch
        mlflow.log_metrics({
            "train_loss": loss,
            "train_accuracy": accuracy,
            "gpu_memory_used": get_gpu_memory()
        }, step=epoch)

    # Log the model
    mlflow.pytorch.log_model(
        model,
        "bert-classifier",
        registered_model_name="bert-classifier"
    )
    mlflow.log_artifact("/data/config.yaml")
```

## 🖥️ Part 9: Hands-On Exercises

**Exercise 1: Simulate GPU scheduling without real GPUs**

```
# kind doesn't have GPUs — fake the resource for scheduling practice

# Add fake GPU resource to a node
kubectl proxy --port=8001 &
sleep 2

# Patch node to advertise fake GPUs
curl -s -X PATCH \
  -H "Content-Type: application/json-patch+json" \
  http://localhost:8001/api/v1/nodes/k8s-mastery-worker/status \
  -d '[
    {
      "op": "add",
      "path": "/status/capacity/example.com~1gpu",
      "value": "4"
    },
    {
      "op": "add",
      "path": "/status/allocatable/example.com~1gpu",
      "value": "4"
    }
  ]' | jq .status.capacity

# Schedule a pod requesting the fake GPU
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fake-gpu-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      limits:
        example.com/gpu: "1"
      requests:
        example.com/gpu: "1"
EOF

kubectl get pod fake-gpu-pod -o wide
# Should be Running on k8s-mastery-worker

# Try to schedule 5 pods requesting 1 GPU each
# 4th and 5th should be Pending (only 4 fake GPUs)
for i in $(seq 1 5); do
  kubectl run gpu-pod-$i \
    --image=nginx:1.25 \
    --overrides="{\"spec\":{\"containers\":[{\"name\":\"gpu-pod-$i\",\"image\":\"nginx:1.25\",\"resources\":{\"limits\":{\"example.com/gpu\":\"1\"},\"requests\":{\"example.com/gpu\":\"1\"}}}]}}"
done

kubectl get pods | grep gpu-pod
# gpu-pod-1   Running  ✓
# gpu-pod-2   Running  ✓
# gpu-pod-3   Running  ✓
# gpu-pod-4   Running  ✓
# gpu-pod-5   Pending  ← no GPUs left

kubectl describe pod gpu-pod-5 | grep -A 5 Events
# Insufficient example.com/gpu

# Cleanup
for i in $(seq 1 5); do kubectl delete pod gpu-pod-$i --force 2>/dev/null; done
kubectl delete pod fake-gpu-pod --force 2>/dev/null
```

**Exercise 2: ML training Job with init container pattern**

```
# Simulate a training job with data preparation
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training-sim
spec:
  completions: 1
  backoffLimit: 2
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
      - name: data-prep
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Downloading training data..."
          mkdir -p /data/train /data/val
          for i in $(seq 1 10); do
            echo "sample_$i,label_$i" >> /data/train/data.csv
          done
          echo "Data prep complete. Files: $(ls /data/train/)"
        volumeMounts:
        - name: training-data
          mountPath: /data

      containers:
      - name: trainer
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import time, os, random
          print("Starting training...")
          data_files = os.listdir('/data/train')
          print(f"Found {len(data_files)} data files")
          for epoch in range(5):
            loss = 1.0 / (epoch + 1) + random.uniform(0, 0.1)
            acc = 1 - loss + random.uniform(0, 0.05)
            print(f"Epoch {epoch+1}/5 - loss: {loss:.4f} - accuracy: {acc:.4f}")
            time.sleep(2)
          print("Training complete! Saving model...")
          with open('/models/model.txt', 'w') as f:
            f.write('trained_model_v1')
          print(f"Model saved: {os.listdir('/models')}")
        resources:
          requests: {cpu: "500m", memory: "256Mi"}
          limits: {cpu: "1", memory: "512Mi"}
        volumeMounts:
        - name: training-data
          mountPath: /data
        - name: model-output
          mountPath: /models

      - name: metrics-sidecar
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "$(date): trainer running, memory=$(cat /sys/fs/cgroup/memory/memory.usage_in_bytes 2>/dev/null || echo 'N/A')"
            sleep 5
          done
        resources:
          requests: {cpu: "50m", memory: "32Mi"}

      volumes:
      - name: training-data
        emptyDir: {}
      - name: model-output
        emptyDir: {}
EOF

# Watch the job
kubectl get job ml-training-sim -w &
kubectl logs -f -l job-name=ml-training-sim -c trainer
```

**Exercise 3: KEDA installation and scaling**

```
# Install KEDA
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace

kubectl get pods -n keda -w

# Create a deployment to auto-scale
kubectl create deployment inference-server \
  --image=nginx:1.25 \
  --replicas=1

# Create ScaledObject based on Prometheus metrics
cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inference-scaler
spec:
  scaleTargetRef:
    name: inference-server
  minReplicaCount: 0            # scale to zero when idle
  maxReplicaCount: 5
  cooldownPeriod: 60
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: http_requests_total
      query: sum(rate(nginx_http_requests_total[1m]))
      threshold: "10"
EOF

kubectl get scaledobject
kubectl describe scaledobject inference-scaler

# Verify HPA was created by KEDA
kubectl get hpa
# keda-hpa-inference-scaler   Deployment/inference-server   0/10 (avg)   0   5   1
```

**Exercise 4: Node affinity for GPU workloads**

```
# Label a node as GPU-capable (simulated)
kubectl label node k8s-mastery-worker \
  accelerator=gpu \
  gpu-type=a100 \
  gpu-count=8

# Create a pod that requires GPU nodes
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: [gpu]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: gpu-type
            operator: In
            values: [a100, h100]    # prefer A100 or H100
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests: {cpu: "100m", memory: "128Mi"}
EOF

kubectl get pod gpu-affinity-pod -o wide
# Should land on k8s-mastery-worker (the labeled node)
kubectl describe pod gpu-affinity-pod | grep "Node:"
```

## 🎯 Part 10: Interview Questions — Day 26

**Q1: How does Kubernetes know a node has GPUs and how does it schedule GPU workloads?**

The NVIDIA Device Plugin runs as a DaemonSet on GPU nodes. It connects to kubelet via a gRPC socket at /var/lib/kubelet/device-plugins/, registers itself, and reports available GPU device IDs. Kubelet advertises these as extended resources on the node — nvidia.com/gpu: 8. When a pod requests nvidia.com/gpu: 1, the scheduler filters to only nodes with sufficient nvidia.com/gpu in their allocatable capacity and scores/selects from those. When kubelet assigns the pod to a GPU node, the device plugin allocates a specific GPU UUID and passes the device path (/dev/nvidia0) to the container runtime, which mounts it into the container. GPU resources must have requests == limits — GPUs are not overcommittable.

**Q2: What is gang scheduling and why does standard K8s scheduling fail for distributed ML training?**

Standard K8s schedules pods independently and greedily. If you have a 4-worker distributed training job and only 3 GPU slots available, the scheduler places 3 workers, which then wait for the 4th. The 3 running workers sit idle consuming GPU hours, never making progress. This is a distributed deadlock. Gang scheduling (Volcano, YuniKorn) holds all pods of a job in a pending queue until ALL required resources are available simultaneously, then places them all at once. The minAvailable field defines the minimum cohort size. If it can't be satisfied, no pods start — zero wasted GPU hours. Essential for distributed training where all workers must communicate.

**Q3: What is MIG and when would you use it over time-slicing?**

MIG (Multi-Instance GPU) is a hardware-level partitioning feature on A100/H100 GPUs. It physically partitions the GPU's compute engines and memory into isolated slices — each slice has dedicated SM clusters and HBM memory with no cross-contamination. Time-slicing is software-level — multiple processes share the GPU via time multiplexing but share the memory address space. Use MIG when: workloads need memory isolation (different teams, security concerns), predictable latency is required (no time-sharing jitter), you have small inference models that don't need a full GPU. Use time-slicing when: all workloads are trusted, latency variation is acceptable, you need maximum utilization across many small workloads or dev environments.

**Q4: How does KEDA differ from HPA and when do you need it for ML workloads?**

HPA scales based on CPU, memory, and custom metrics — but requires those metrics to be served via the metrics API, and it can't scale to zero. KEDA is an event-driven autoscaler that extends HPA — it creates and manages HPAs behind the scenes but adds: scale-to-zero capability (remove all replicas when idle, start from zero on demand), 50+ built-in scalers (SQS queue depth, Kafka lag, Prometheus query, Redis list length), and external event sources. For ML inference: scale to zero when no requests are queued (save GPU costs), scale up instantly when SQS has pending inference requests, scale based on actual model serving latency — none of which HPA can do natively.

**Q5: Explain the NUMA awareness problem for GPU training workloads.**

Modern servers have multiple NUMA nodes — each has local CPUs and local memory. GPUs are physically connected to specific NUMA nodes via PCIe. When a training process runs on CPU cores in NUMA node 0 but its GPU is connected to NUMA node 1, all CPU-GPU data transfers cross the QPI/UPI interconnect — typically 4x slower than local NUMA bandwidth. For GPU training, this manifests as: slow data loading to GPU (CPU reads dataset from memory, transfers to GPU), slow gradient all-reduce (GPUs communicate via NVLink but CPU operations cross NUMA). The Topology Manager in kubelet can align CPU cores, memory, and GPU to the same NUMA node for a pod. Combined with CPU Manager (dedicated cores) this ensures zero cross-NUMA traffic for latency-sensitive training jobs.

**Q6: How would you design a multi-tenant GPU cluster for a research team with 5 projects competing for 20 GPUs?**

Namespace per project with ResourceQuota limiting nvidia.com/gpu — prevents any one project from monopolizing all 20. PriorityClass hierarchy: interactive (Jupyter notebooks) > batch-high (deadline jobs) > batch-low (hyperparameter sweeps). KEDA to scale inference services to zero when idle — frees GPUs for training. Fair queuing via Volcano's queue system — each project gets a guaranteed quota and can burst into shared capacity when others are idle. GPU time-slicing on 2-4 GPUs dedicated to dev/interactive workloads — researchers get fast iteration without waiting for full GPUs. Prometheus + Grafana dashboard showing GPU utilization per project, idle GPU hours wasted, and queue depth per priority. Weekly GPU usage reports per namespace via cost allocation labels. Preemption: batch-low pods are evicted if batch-high needs GPUs — using PriorityClass with preemptionPolicy: PreemptLowerPriority.












