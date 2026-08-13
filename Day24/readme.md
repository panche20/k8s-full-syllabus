# Day 24 — KubeFlow, LLM Inference & AI/K8s Integration

The final frontier. Today you deploy the full MLOps stack and serve Large Language Models on Kubernetes. This is the most in-demand skill combination in the market right now — K8s engineers who understand LLM infrastructure are extremely rare and extremely well-paid.

## 🧠 Part 1: The MLOps Stack on Kubernetes

```
Data Layer          → object storage (S3/GCS), feature stores
Experiment Layer    → MLflow, Weights & Biases, Neptune
Training Layer      → KubeFlow Pipelines, Argo Workflows, Ray
Model Registry      → MLflow Model Registry, BentoML
Serving Layer       → KServe, Triton, vLLM, BentoML
Monitoring Layer    → Prometheus + Grafana + Evidently (drift)
Orchestration       → all of the above running on Kubernetes
```

*Today covers the Training Layer (KubeFlow) and Serving Layer (KServe + vLLM) in depth.*

## 🏭 Part 2: KubeFlow — The ML Platform

KubeFlow is a collection of K8s-native ML tools. Not a monolith — pick the components you need.

```
KubeFlow Pipelines      — DAG-based ML workflow orchestration
Training Operator       — distributed training (PyTorchJob, TFJob, MXJob)
KServe                  — model serving with autoscaling
Katib                   — hyperparameter tuning (AutoML)
Notebooks               — JupyterHub on K8s
```

**Install KubeFlow Pipelines**

```
# KubeFlow full install is complex — let's do Pipelines standalone
# This is what most teams actually deploy

export PIPELINE_VERSION=2.0.5

kubectl apply -k \
  "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"

kubectl wait --for=condition=established \
  --timeout=60s \
  crd/applications.app.k8s.io

kubectl apply -k \
  "github.com/kubeflow/pipelines/manifests/kustomize/env/dev?ref=$PIPELINE_VERSION"

# Watch everything come up (takes 3-5 minutes)
kubectl get pods -n kubeflow -w

# Access the UI
kubectl port-forward -n kubeflow \
  svc/ml-pipeline-ui 8080:80
# http://localhost:8080

# Components deployed:
kubectl get pods -n kubeflow
# ml-pipeline              ← pipeline API server
# ml-pipeline-ui           ← web UI
# ml-pipeline-scheduledworkflow  ← recurring runs
# ml-pipeline-viewer-crd   ← visualization CRDs
# minio                    ← artifact storage
# mysql                    ← metadata storage
# cache-server             ← step caching
```

## 📋 Part 3: KubeFlow Pipelines — Build a Real Pipeline

A KFP Pipeline is a DAG of containerized steps. Each step is a K8s Pod. Artifacts flow between steps via object storage.

**Pipeline with Python SDK v2**

```
# pipeline.py — URL shortener model training pipeline
from kfp import dsl
from kfp import compiler
from kfp.dsl import (
    Dataset, Input, Output, Model,
    Metrics, ClassificationMetrics,
    component, pipeline
)

# ── Components — each becomes a K8s Pod ──────────────────────

@component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "boto3"]
)
def load_data(
    dataset_uri: str,
    output_dataset: Output[Dataset]
):
    """Load and validate training data."""
    import pandas as pd

    print(f"Loading data from {dataset_uri}")
    # In production: load from S3/GCS
    df = pd.DataFrame({
        'url_length': [23, 45, 67, 12, 89],
        'domain_age': [5, 2, 10, 1, 7],
        'is_spam': [0, 1, 0, 1, 0]
    })

    df.to_csv(output_dataset.path, index=False)
    print(f"Loaded {len(df)} samples")

@component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn"]
)
def train_model(
    input_dataset: Input[Dataset],
    learning_rate: float,
    n_estimators: int,
    output_model: Output[Model],
    metrics: Output[Metrics]
):
    """Train a spam URL classifier."""
    import pandas as pd
    import pickle
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import accuracy_score, f1_score

    df = pd.read_csv(input_dataset.path)
    X = df[['url_length', 'domain_age']]
    y = df['is_spam']

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    model = RandomForestClassifier(
        n_estimators=n_estimators,
        random_state=42
    )
    model.fit(X_train, y_train)

    y_pred = model.predict(X_test)
    acc = accuracy_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred, average='weighted')

    # Log metrics to KFP
    metrics.log_metric("accuracy", acc)
    metrics.log_metric("f1_score", f1)
    metrics.log_metric("n_estimators", n_estimators)
    metrics.log_metric("learning_rate", learning_rate)

    print(f"Accuracy: {acc:.4f}, F1: {f1:.4f}")

    # Save model
    with open(output_model.path, 'wb') as f:
        pickle.dump(model, f)

@component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "boto3"]
)
def evaluate_and_register(
    model: Input[Model],
    metrics: Input[Metrics],
    accuracy_threshold: float,
    model_name: str
) -> str:
    """Evaluate model and register if it meets threshold."""
    import pickle
    import json

    # Read metrics
    with open(metrics.path) as f:
        m = json.load(f)

    accuracy = m.get("accuracy", 0)
    print(f"Model accuracy: {accuracy}")

    if accuracy >= accuracy_threshold:
        print(f"✅ Model passes threshold {accuracy_threshold}")
        # In production: push to MLflow or KServe model registry
        status = "registered"
    else:
        print(f"❌ Model fails threshold {accuracy_threshold}")
        status = "rejected"

    return status

# ── Pipeline — connects components into a DAG ──────────────────

@pipeline(
    name="url-shortener-spam-classifier",
    description="Train and deploy a URL spam classifier"
)
def url_classifier_pipeline(
    dataset_uri: str = "s3://ml-datasets/urls/latest/",
    learning_rate: float = 0.01,
    n_estimators: int = 100,
    accuracy_threshold: float = 0.85,
    model_name: str = "url-spam-classifier"
):
    # Step 1: Load data
    load_task = load_data(dataset_uri=dataset_uri)
    load_task.set_cpu_request("500m")
    load_task.set_memory_request("1Gi")

    # Step 2: Train model — depends on load_task
    train_task = train_model(
        input_dataset=load_task.outputs["output_dataset"],
        learning_rate=learning_rate,
        n_estimators=n_estimators
    )
    train_task.set_cpu_request("2")
    train_task.set_memory_request("4Gi")
    # GPU request — uncomment for GPU training
    # train_task.set_gpu_limit("1")

    # Step 3: Evaluate and register
    eval_task = evaluate_and_register(
        model=train_task.outputs["output_model"],
        metrics=train_task.outputs["metrics"],
        accuracy_threshold=accuracy_threshold,
        model_name=model_name
    )

    # Conditional: only deploy if registered
    with dsl.If(eval_task.output == "registered"):
        # deploy_task = deploy_model(...)
        print("Model would be deployed here")

# Compile the pipeline
compiler.Compiler().compile(
    pipeline_func=url_classifier_pipeline,
    package_path="url_classifier_pipeline.yaml"
)

print("Pipeline compiled to url_classifier_pipeline.yaml")
```

```
# Install KFP SDK
pip install kfp==2.5.0

# Compile pipeline
python pipeline.py

# Submit to KFP via CLI
pip install kfp-kubernetes

# Or upload via UI: localhost:8080
# Pipelines → Upload Pipeline → url_classifier_pipeline.yaml

# Create a run with parameters
# In UI: Run → Create Run → set parameters → Start
```

## 🤖 Part 4: LLM Inference on Kubernetes

Serving LLMs (Llama, Mistral, GPT-style models) has unique requirements:

- Large model weights (7B = ~14GB, 70B = ~140GB)
- GPU memory management (KV cache)
- Continuous batching (not request-by-request)
- Streaming responses (SSE/WebSocket)

**vLLM — the production LLM inference engine**

vLLM uses PagedAttention — manages GPU memory like virtual memory in an OS. This allows batching many requests efficiently even with variable sequence lengths.

```
# vLLM deployment for Llama-2-7B
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-2-7b
  namespace: llm-serving
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llama-2-7b
  template:
    metadata:
      labels:
        app: llama-2-7b
    spec:
      nodeSelector:
        accelerator: nvidia-a100

      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule

      initContainers:
      # Download model weights before starting vLLM
      - name: model-downloader
        image: python:3.11-slim
        command:
        - sh
        - -c
        - |
          pip install -q huggingface_hub
          python -c "
          from huggingface_hub import snapshot_download
          snapshot_download(
              repo_id='meta-llama/Llama-2-7b-chat-hf',
              local_dir='/models/llama-2-7b',
              token='$HF_TOKEN'
          )
          print('Model downloaded successfully')
          "
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-credentials
              key: token
        volumeMounts:
        - name: model-storage
          mountPath: /models

      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        command:
        - python
        - -m
        - vllm.entrypoints.openai.api_server
        args:
        - --model=/models/llama-2-7b
        - --host=0.0.0.0
        - --port=8000
        - --tensor-parallel-size=1     # GPUs to use (1 A100 for 7B)
        - --max-model-len=4096         # max context length
        - --gpu-memory-utilization=0.90  # use 90% of GPU memory for KV cache
        - --max-num-batched-tokens=4096
        - --served-model-name=llama-2-7b-chat
        ports:
        - containerPort: 8000
          name: http
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: 32Gi
          requests:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: 32Gi
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120    # model loading takes time
          periodSeconds: 10
          failureThreshold: 30        # 5 minutes to load
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          periodSeconds: 30
          failureThreshold: 3
        volumeMounts:
        - name: model-storage
          mountPath: /models
        - name: shm
          mountPath: /dev/shm

      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: llm-model-pvc
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 16Gi
---
apiVersion: v1
kind: Service
metadata:
  name: llama-2-7b
  namespace: llm-serving
spec:
  selector:
    app: llama-2-7b
  ports:
  - port: 80
    targetPort: 8000
    name: http
```

**Test the LLM endpoint**

```
# Once deployed — call the OpenAI-compatible API
kubectl port-forward -n llm-serving svc/llama-2-7b 8000:80

# List available models
curl http://localhost:8000/v1/models | jq .

# Chat completion (same API as OpenAI)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b-chat",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is Kubernetes?"}
    ],
    "max_tokens": 200,
    "temperature": 0.7,
    "stream": false
  }' | jq .choices[0].message.content

# Streaming response
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b-chat",
    "messages": [{"role": "user", "content": "Explain Kubernetes in 3 sentences"}],
    "stream": true
  }'
# Returns Server-Sent Events
```

## 🚀 Part 5: KServe with LLM — Production Pattern

KServe handles the serving infrastructure — scaling, canary, batching. vLLM handles the inference engine. Together they form the production pattern.

```
# KServe InferenceService wrapping vLLM
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-2-7b-kserve
  namespace: llm-serving
  annotations:
    serving.kserve.io/deploymentMode: RawDeployment  # direct K8s, no Knative
spec:
  predictor:
    containers:
    - name: kserve-container
      image: vllm/vllm-openai:latest
      command:
      - python
      - -m
      - vllm.entrypoints.openai.api_server
      args:
      - --model=/mnt/models
      - --host=0.0.0.0
      - --port=8080
      - --tensor-parallel-size=1
      - --served-model-name=llama-2-7b
      ports:
      - containerPort: 8080
        protocol: TCP
      resources:
        requests:
          nvidia.com/gpu: "1"
          cpu: "8"
          memory: 32Gi
        limits:
          nvidia.com/gpu: "1"
          cpu: "8"
          memory: 32Gi
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 180
        periodSeconds: 10
      volumeMounts:
      - mountPath: /dev/shm
        name: shm
    volumes:
    - name: shm
      emptyDir:
        medium: Memory
        sizeLimit: 16Gi

    # Model storage via PVC
    storageSpec:
      storageUri: pvc://llm-model-pvc/llama-2-7b
```

## ⚡ Part 6: Multi-Model Serving — Triton Inference Server

Triton serves multiple models simultaneously. Critical when you have dozens of models and can't afford one GPU per model.

```
# Triton serves multiple models from a model repository
# Model repository structure:
# /model-repo/
#   bert-classifier/
#     1/model.pt          ← version 1
#     config.pbtxt        ← model config
#   sentiment-model/
#     1/model.onnx
#     config.pbtxt
#   embedding-model/
#     1/model.pt
#     config.pbtxt

apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-server
  namespace: ml-serving
spec:
  replicas: 1
  selector:
    matchLabels:
      app: triton
  template:
    metadata:
      labels:
        app: triton
    spec:
      containers:
      - name: triton
        image: nvcr.io/nvidia/tritonserver:23.10-py3
        args:
        - tritonserver
        - --model-repository=/model-repo
        - --strict-model-config=false
        - --log-verbose=1
        - --model-control-mode=poll      # hot-reload new models
        - --repository-poll-secs=30      # check every 30s
        ports:
        - containerPort: 8000            # HTTP
          name: http
        - containerPort: 8001            # gRPC
          name: grpc
        - containerPort: 8002            # metrics
          name: metrics
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: 32Gi
          requests:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: 32Gi
        readinessProbe:
          httpGet:
            path: /v2/health/ready
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 10
        volumeMounts:
        - name: model-repo
          mountPath: /model-repo
      volumes:
      - name: model-repo
        persistentVolumeClaim:
          claimName: triton-model-pvc
```

## 🔄 Part 7: Ray on Kubernetes — Distributed ML

Ray is the standard for distributed Python ML — training, hyperparameter tuning, inference, and data processing.

```
# Install KubeRay Operator
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update

helm install kuberay-operator kuberay/kuberay-operator \
  --namespace ray-system \
  --create-namespace

kubectl get pods -n ray-system -w
```

```
# RayCluster — managed by KubeRay Operator
apiVersion: ray.io/v1alpha1
kind: RayCluster
metadata:
  name: ml-cluster
  namespace: ray-system
spec:
  rayVersion: "2.8.0"

  # Head node — coordinator
  headGroupSpec:
    rayStartParams:
      dashboard-host: "0.0.0.0"
      num-cpus: "0"              # head doesn't do compute — just coordination
    template:
      spec:
        containers:
        - name: ray-head
          image: rayproject/ray-ml:2.8.0-gpu
          resources:
            requests: {cpu: "2", memory: 8Gi}
            limits:   {cpu: "4", memory: 8Gi}
          ports:
          - containerPort: 6379  # Ray GCS
          - containerPort: 8265  # Ray dashboard
          - containerPort: 10001 # Ray client

  # Worker nodes — do the actual work
  workerGroupSpecs:
  - groupName: gpu-workers
    replicas: 2
    minReplicas: 1
    maxReplicas: 4              # auto-scale workers based on load
    rayStartParams: {}
    template:
      spec:
        nodeSelector:
          accelerator: nvidia-a100
        tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
        containers:
        - name: ray-worker
          image: rayproject/ray-ml:2.8.0-gpu
          resources:
            requests:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: 32Gi
            limits:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: 32Gi
```

```
# Ray job — distributed hyperparameter tuning
# submit as: kubectl apply -f rayjob.yaml

import ray
from ray import tune
from ray.tune.search.optuna import OptunaSearch

@ray.remote(num_gpus=1)        # each trial gets 1 GPU
def train_with_config(config):
    import torch

    model = build_model(
        hidden_size=config["hidden_size"],
        dropout=config["dropout"]
    )
    optimizer = torch.optim.Adam(
        model.parameters(),
        lr=config["lr"]
    )

    for epoch in range(10):
        loss = train_epoch(model, optimizer)
        tune.report(loss=loss, epoch=epoch)

# Launch distributed hyperparameter search
ray.init("ray://ray-head.ray-system.svc:10001")

analysis = tune.run(
    train_with_config,
    config={
        "lr": tune.loguniform(1e-4, 1e-1),
        "hidden_size": tune.choice([256, 512, 1024]),
        "dropout": tune.uniform(0.1, 0.5)
    },
    num_samples=20,             # 20 trials
    resources_per_trial={"gpu": 1, "cpu": 4},
    search_alg=OptunaSearch(),  # Bayesian optimization
    scheduler=tune.schedulers.ASHAScheduler(
        metric="loss",
        mode="min",
        max_t=10,
        grace_period=3
    )
)

print("Best config:", analysis.best_config)
print("Best loss:", analysis.best_result["loss"])
```

## 📡 Part 8: LLM Gateway — Production API Layer

In production you don't expose vLLM directly. You need: authentication, rate limiting, routing to multiple models, cost tracking, prompt logging.

```
# LiteLLM — LLM gateway for Kubernetes
# Routes requests to OpenAI, vLLM, Anthropic, Bedrock from one API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-gateway
  namespace: llm-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llm-gateway
  template:
    metadata:
      labels:
        app: llm-gateway
    spec:
      containers:
      - name: litellm
        image: ghcr.io/berriai/litellm:main-latest
        command:
        - litellm
        - --config=/config/litellm-config.yaml
        - --port=4000
        - --num_workers=4
        ports:
        - containerPort: 4000
        resources:
          requests: {cpu: "1", memory: 2Gi}
          limits:   {cpu: "2", memory: 4Gi}
        volumeMounts:
        - name: config
          mountPath: /config
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: llm-secrets
              key: openai-api-key
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: llm-secrets
              key: anthropic-api-key
        - name: DATABASE_URL
          value: postgresql://litellm:password@postgres.llm-serving.svc:5432/litellm
      volumes:
      - name: config
        configMap:
          name: litellm-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: litellm-config
  namespace: llm-serving
data:
  litellm-config.yaml: |
    model_list:
    # Self-hosted models
    - model_name: llama-2-7b
      litellm_params:
        model: openai/llama-2-7b-chat    # OpenAI-compatible
        api_base: http://llama-2-7b.llm-serving.svc/v1
        api_key: none

    - model_name: mistral-7b
      litellm_params:
        model: openai/mistral-7b-instruct
        api_base: http://mistral-7b.llm-serving.svc/v1
        api_key: none

    # Cloud models
    - model_name: gpt-4
      litellm_params:
        model: gpt-4
        api_key: os.environ/OPENAI_API_KEY

    - model_name: claude-3-sonnet
      litellm_params:
        model: anthropic/claude-3-sonnet-20240229
        api_key: os.environ/ANTHROPIC_API_KEY

    router_settings:
      routing_strategy: least-busy      # route to least loaded model
      fallbacks:
      - llama-2-7b: [gpt-4]            # fallback to OpenAI if local is down

    general_settings:
      master_key: sk-your-gateway-key   # API key for all clients
      database_url: os.environ/DATABASE_URL
      store_model_in_db: true
      
    litellm_settings:
      success_callback: ["langfuse"]    # log successful calls
      failure_callback: ["langfuse"]    # log failures
      callbacks: ["prometheus"]         # expose metrics
```

```
# Test the gateway — single API for all models
kubectl port-forward svc/llm-gateway 4000:4000 -n llm-serving

# Use local Llama
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-your-gateway-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b",
    "messages": [{"role": "user", "content": "Hello"}]
  }' | jq .

# Use GPT-4 via same API
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-your-gateway-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}]
  }' | jq .

# View usage dashboard
# kubectl port-forward svc/llm-gateway 4000:4000
# http://localhost:4000/ui
```

## 📊 Part 9: LLM Observability

```
# Prometheus metrics from LiteLLM gateway
# Add to your PrometheusRule:

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: llm-gateway
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: llm-gateway
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

```
# LLM-specific metrics to track:

# Request rate per model
rate(litellm_requests_total[5m]) by (model)

# Token usage per model (cost driver)
rate(litellm_prompt_tokens_total[1h]) by (model)
rate(litellm_completion_tokens_total[1h]) by (model)

# Cost per hour per model
(rate(litellm_prompt_tokens_total[1h]) * 0.00001 +
 rate(litellm_completion_tokens_total[1h]) * 0.00003) by (model)

# P99 latency per model (TTFT — Time to First Token)
histogram_quantile(0.99,
  rate(litellm_request_duration_seconds_bucket[5m])
) by (model)

# Error rate per model
rate(litellm_requests_total{status="error"}[5m]) /
rate(litellm_requests_total[5m])

# GPU utilization during inference
DCGM_FI_DEV_GPU_UTIL{pod=~"llama.*|mistral.*"}

# KV cache hit rate (vLLM efficiency metric)
rate(vllm_cache_events_total{type="hit"}[5m]) /
rate(vllm_cache_events_total[5m])
```

## 🖥️ Part 10: Hands-On Exercises

**Exercise 1: Deploy a simulated ML pipeline**

```
# Simulate the full pipeline without real GPUs

# Step 1: Create namespaces
kubectl create namespace ml-training
kubectl create namespace ml-serving

# Step 2: Simulate training job
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: url-classifier-training
  namespace: ml-training
  labels:
    experiment: url-spam-v1
    pipeline: url-classifier
spec:
  completions: 1
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: data-prep
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import csv, os, random
          os.makedirs('/data', exist_ok=True)
          with open('/data/train.csv', 'w') as f:
              w = csv.writer(f)
              w.writerow(['url', 'length', 'dots', 'is_spam'])
              for i in range(1000):
                  length = random.randint(10, 200)
                  dots = random.randint(1, 10)
                  is_spam = 1 if (length > 100 and dots > 5) else 0
                  w.writerow([f'http://example{i}.com', length, dots, is_spam])
          print(f'Created training data: {len(open("/data/train.csv").readlines())} rows')
        volumeMounts:
        - name: data
          mountPath: /data

      containers:
      - name: trainer
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import csv, json, random, os, time

          # Read data
          rows = list(csv.DictReader(open('/data/train.csv')))
          print(f'Loaded {len(rows)} training samples')

          # Simulate training epochs
          os.makedirs('/models/url-classifier', exist_ok=True)
          best_loss = 1.0
          for epoch in range(10):
              loss = best_loss * 0.85 + random.uniform(-0.02, 0.02)
              acc = 1 - loss + random.uniform(0, 0.05)
              best_loss = min(best_loss, loss)
              print(f'Epoch {epoch+1}/10 loss={loss:.4f} acc={acc:.4f}')
              time.sleep(1)

          # Save model metadata
          model_meta = {
              'model_name': 'url-spam-classifier',
              'version': 'v1',
              'accuracy': float(acc),
              'f1_score': float(acc * 0.95),
              'framework': 'sklearn',
              'training_samples': len(rows)
          }
          with open('/models/url-classifier/metadata.json', 'w') as f:
              json.dump(model_meta, f, indent=2)

          print('Model saved:')
          print(json.dumps(model_meta, indent=2))
        resources:
          requests: {cpu: "500m", memory: "512Mi"}
          limits:   {cpu: "1", memory: "1Gi"}
        volumeMounts:
        - name: data
          mountPath: /data
        - name: models
          mountPath: /models

      volumes:
      - name: data
        emptyDir: {}
      - name: models
        emptyDir: {}
EOF

# Watch training
kubectl logs -f -l job-name=url-classifier-training \
  -n ml-training --all-containers

# Wait for completion
kubectl wait --for=condition=complete \
  job/url-classifier-training \
  -n ml-training \
  --timeout=120s

kubectl get job url-classifier-training -n ml-training
# COMPLETIONS: 1/1
```

**Exercise 2: Deploy a simulated LLM inference server**

```
# Since we don't have GPUs — simulate vLLM with FastAPI
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-simulator
  namespace: ml-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llm-simulator
  template:
    metadata:
      labels:
        app: llm-simulator
    spec:
      containers:
      - name: server
        image: python:3.11-slim
        command:
        - sh
        - -c
        - |
          pip install -q fastapi uvicorn
          cat > /app.py << 'PYEOF'
          from fastapi import FastAPI
          from pydantic import BaseModel
          from typing import List, Optional
          import time, random, os

          app = FastAPI(title="LLM Simulator")
          MODEL_NAME = os.environ.get("MODEL_NAME", "llama-2-7b-chat-sim")

          class Message(BaseModel):
              role: str
              content: str

          class ChatRequest(BaseModel):
              model: str = MODEL_NAME
              messages: List[Message]
              max_tokens: Optional[int] = 100
              temperature: Optional[float] = 0.7

          @app.get("/health")
          async def health():
              return {"status": "healthy", "model": MODEL_NAME}

          @app.get("/v1/models")
          async def list_models():
              return {"data": [{"id": MODEL_NAME, "object": "model"}]}

          @app.post("/v1/chat/completions")
          async def chat(req: ChatRequest):
              # Simulate model latency
              latency = random.uniform(0.1, 0.5)
              time.sleep(latency)
              user_msg = req.messages[-1].content if req.messages else ""
              response = f"[{MODEL_NAME}] Responding to: '{user_msg[:50]}...' (simulated inference)"
              tokens_used = len(response.split())
              return {
                  "id": f"sim-{int(time.time())}",
                  "object": "chat.completion",
                  "model": req.model,
                  "choices": [{
                      "index": 0,
                      "message": {"role": "assistant", "content": response},
                      "finish_reason": "stop"
                  }],
                  "usage": {
                      "prompt_tokens": len(user_msg.split()),
                      "completion_tokens": tokens_used,
                      "total_tokens": len(user_msg.split()) + tokens_used
                  }
              }
          PYEOF
          python -m uvicorn app:app --host 0.0.0.0 --port 8000 --workers 2
        ports:
        - containerPort: 8000
        env:
        - name: MODEL_NAME
          value: llama-2-7b-chat-sim
        resources:
          requests: {cpu: "200m", memory: "256Mi"}
          limits:   {cpu: "500m", memory: "512Mi"}
        readinessProbe:
          httpGet: {path: /health, port: 8000}
          initialDelaySeconds: 15
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: llm-simulator
  namespace: ml-serving
spec:
  selector:
    app: llm-simulator
  ports:
  - port: 80
    targetPort: 8000
EOF

kubectl get pods -n ml-serving -w &
kubectl wait --for=condition=ready \
  pod -l app=llm-simulator \
  -n ml-serving \
  --timeout=120s

# Test the simulated LLM
kubectl run test-llm --image=curlimages/curl:latest \
  --rm -i --restart=Never -q \
  -- curl -s -X POST \
     http://llm-simulator.ml-serving.svc/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{
       "model": "llama-2-7b-chat-sim",
       "messages": [
         {"role": "user", "content": "What is Kubernetes?"}
       ]
     }' | python3 -m json.tool

# Expected Output:

{
    "id": "sim-1786625089",
    "object": "chat.completion",
    "model": "llama-2-7b-chat-sim",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "[llama-2-7b-chat-sim] Responding to: 'What is Kubernetes?...' (simulated inference)"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 3,
        "completion_tokens": 8,
        "total_tokens": 11
    }
}
```

**Exercise 3: Load test the LLM endpoint**

```
# Simulate 100 concurrent requests
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-load-test
  namespace: ml-serving
spec:
  parallelism: 5              # 5 parallel workers
  completions: 20             # 20 total requests
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: load-tester
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import urllib.request, json, time, os
          url = "http://llm-simulator.ml-serving.svc/v1/chat/completions"
          worker_id = os.environ.get("JOB_COMPLETION_INDEX", "?")
          payload = json.dumps({
              "model": "llama-2-7b-chat-sim",
              "messages": [{"role": "user", "content": f"Test request from worker {worker_id}"}],
              "max_tokens": 50
          }).encode()
          start = time.time()
          req = urllib.request.Request(url,
              data=payload,
              headers={"Content-Type": "application/json"},
              method="POST")
          with urllib.request.urlopen(req) as resp:
              result = json.loads(resp.read())
          latency = time.time() - start
          content = result["choices"][0]["message"]["content"]
          tokens = result["usage"]["total_tokens"]
          print(f"Worker {worker_id}: {latency:.3f}s | {tokens} tokens | {content[:50]}")
        resources:
          requests: {cpu: "50m", memory: "64Mi"}
EOF

# Watch the load test results
kubectl logs -f -l job-name=llm-load-test -n ml-serving --max-log-requests=20

#Tail Logs Without Following (-f)
kubectl logs -l job-name=llm-load-test -n ml-serving --tail=-1

# Step 1: Create the HPA for llm-simulator
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-simulator-hpa
  namespace: ml-serving
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-simulator
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# Step 2: Verify the HPA
kubectl get hpa -n ml-serving

# Step 3: Trigger Scaling with Load Test
# 1. Start continuous watch on HPA and Deployment replicas
kubectl get hpa,deployment -n ml-serving -w &

# 2. Re-trigger the load test job
kubectl delete job llm-load-test -n ml-serving --ignore-not-found
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-load-test
  namespace: ml-serving
spec:
  parallelism: 10
  completions: 50
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: load-tester
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import urllib.request, json, time
          url = "http://llm-simulator.ml-serving.svc/v1/chat/completions"
          payload = json.dumps({
              "model": "llama-2-7b-chat-sim",
              "messages": [{"role": "user", "content": "Stress test request"}],
              "max_tokens": 50
          }).encode()
          req = urllib.request.Request(url, data=payload, headers={"Content-Type": "application/json"}, method="POST")
          with urllib.request.urlopen(req) as resp:
              print("Response status:", resp.status)
        resources:
          requests: {cpu: "50m", memory: "64Mi"}
EOF

# Check HPA kicks in (if configured)
kubectl get hpa -n ml-serving

```

**Exercise 4: Pipeline observability**

```
# Create Namespace
kubectl create ns monitoring

# Create a Prometheus metrics simulation for the LLM
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: llm-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  llm-dashboard.json: |
    {
      "title": "LLM Serving Dashboard",
      "panels": [
        {"title": "Request Rate", "type": "graph"},
        {"title": "Token Usage", "type": "stat"},
        {"title": "P99 Latency", "type": "gauge"},
        {"title": "Error Rate", "type": "graph"}
      ]
    }
EOF

# Add metrics endpoint to LLM simulator
kubectl exec -n ml-serving \
  $(kubectl get pod -n ml-serving -l app=llm-simulator -o jsonpath='{.items[0].metadata.name}') \
  -- python3 -c 'import urllib.request; print(urllib.request.urlopen("http://localhost:8000/v1/models").read().decode())' | jq .
```

## 🎯 Part 11: Interview Questions — Day 27

**Q1: What is PagedAttention in vLLM and why does it matter for serving LLMs on Kubernetes?**

Standard LLM serving allocates a fixed KV (key-value) cache chunk for each request's maximum possible length — even if the actual sequence is much shorter. This wastes GPU memory and limits how many requests can be batched simultaneously. PagedAttention borrows the OS virtual memory concept: it divides the KV cache into fixed-size pages and allocates them on demand as the sequence grows. Pages are stored non-contiguously like virtual memory — the attention mechanism uses a block table to find them. This allows: higher GPU memory utilization (no wasted pre-allocated space), more concurrent requests per GPU, and efficient KV cache sharing between requests with common prefixes (system prompts). On Kubernetes this directly affects how many pods you need — better utilization means fewer GPU nodes.

**Q2: How do you handle LLM cold start latency in a Kubernetes deployment?**

Cold start is severe for LLMs — loading a 7B model takes 2-5 minutes. Three strategies: First, keep minimum replicas above zero — minReplicas: 1 in HPA/KEDA, never scale to zero for latency-sensitive models. Trade: GPU cost when idle. Second, model caching on the node — use a DaemonSet to pre-pull model weights to NVMe SSD on every GPU node. When a pod starts, it reads from local disk (seconds) not object storage (minutes). Use a hostPath or local PV for this cache. Third, quantized models — INT8 or INT4 quantization reduces model size 4-8x, so loading is proportionally faster. GPTQ or AWQ quantization for LLMs trades a small accuracy loss for dramatic speed improvement. On Kubernetes: the readinessProbe with initialDelaySeconds: 180 and high failureThreshold buys the pod time to load before receiving traffic.

**Q3: Why is KubeFlow Pipelines better than a simple CI/CD pipeline for ML workflows?**

CI/CD pipelines are designed for software deployment — linear steps, no native concept of artifacts, no step caching, no experiment tracking. KFP is built for ML workflows: each step is a typed component that produces/consumes typed Artifacts (Dataset, Model, Metrics) stored in object storage. If step 2 fails, you re-run from step 2 — not from data ingestion (step 1). Built-in caching: if the input data hasn't changed, step 1 is skipped entirely using cached output. Native ML concepts: hyperparameter recording, metric visualization, model lineage. Conditional execution: only deploy if accuracy exceeds threshold. Pipeline versioning: every pipeline run is recorded with exact code, parameters, and output artifacts — full reproducibility. The DAG visualization makes debugging non-linear ML workflows tractable where CI/CD logs make it impossible.

**Q4: How do you implement A/B testing for LLM models on Kubernetes?**

Two approaches. Traffic splitting via Istio VirtualService: deploy model-A and model-B as separate Deployments, use a VirtualService with 50/50 weight split, record which model served each request via request headers, analyze downstream metrics (user engagement, task completion) per model. KServe canary: set canaryTrafficPercent: 20 in the InferenceService — 20% of requests go to the new model, 80% to stable. KServe handles the traffic split and metrics. For LLMs specifically: track per-model metrics in LiteLLM gateway — latency, token usage, user rating (if you have a feedback loop). Key difference from standard A/B: LLM quality is hard to measure automatically. Use human preference data (thumbs up/down) logged via the gateway, analyzed per model version. Roll out the winning model via GitOps — update the weight in the VirtualService YAML.

**Q5: What are the resource challenges unique to serving multiple LLMs in the same cluster?**

GPU memory fragmentation: a 7B model needs ~14GB, a 13B needs ~26GB. An A100 (80GB) could fit one 70B model or five 7B models but not one 70B and one 7B simultaneously. Plan model placement carefully. Model loading time: hot-swapping models wastes minutes — Triton's model-control-mode=poll allows loading new models without restart but doesn't solve GPU memory limits. KV cache competition: multiple models sharing a GPU via MIG have separate KV caches — requests to model A don't evict model B's cache. Time-slicing shares KV cache implicitly — one model's large batch can starve another's cache. Networking: large models served across multiple GPUs (tensor parallelism) need high-bandwidth node-local NVLink — K8s scheduling must place all shards on the same node or nodes connected by InfiniBand. Use pod affinity + topology spread constraints for this.

**Q6: How does Ray on Kubernetes differ from a standard K8s Job for distributed training?**

K8s Jobs are static — you specify N pods and they run independently. For distributed training, pods must discover each other, coordinate gradient all-reduce, and handle failures coherently. If one pod dies, K8s creates a replacement but the distributed training group is broken — the remaining pods hang. Ray provides: dynamic task scheduling (workers pull tasks from a queue, no static assignment), automatic failure recovery (if a worker dies, its tasks are re-queued and picked up by another worker), actor model (stateful distributed objects that persist across tasks), and the Ray Train library handles distributed deep learning setup (process groups, NCCL communication) automatically. KubeRay Operator manages the RayCluster lifecycle — scales workers based on queue depth, handles head/worker coordination. The key difference: Ray is a distributed execution framework that happens to run on Kubernetes, not Kubernetes Job primitives that happen to do ML.





