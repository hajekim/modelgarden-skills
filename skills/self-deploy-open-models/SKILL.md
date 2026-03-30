---
name: self-deploy-open-models
description: Use when deploying open models to Vertex AI or GKE in your own GCP project. Triggers on: deploy Llama/Gemma/Qwen/DeepSeek/Mistral to Vertex AI or GKE, Gemma 3/3n/2 deploy, fine-tune Gemma with Axolotl/PEFT/KerasNLP/LoRA/QLoRA, vLLM/SGLang/TGI/Hex-LLM containers, GPU/TPU v5e/v6e/v7x/CPU endpoint, GCS custom weights, multimodal vision/audio, batch inference. NOT for: Kimi/MiniMax/GLM/gpt-oss (MaaS-only, use maas-open-models skill instead).
---

# Vertex AI Model Garden: Self-Deployed Open Models

## Overview

Self-deployed models run **within your Google Cloud project and VPC network**, giving you full control over hardware, serving framework, and custom weights. Unlike MaaS, you provision and manage the infrastructure.

> **Models NOT available for self-deployment** — use the `maas-open-models` skill instead:
> Kimi K2 Thinking, MiniMax M2, GLM 5 / GLM 4.7, gpt-oss-120B / gpt-oss-20B
>
> **Models available via BOTH MaaS and self-deploy:** DeepSeek, Llama, Qwen
> → Use MaaS for quick start / variable traffic. Use self-deploy for sustained load, fine-tuned weights, or VPC isolation.
>
> **Models available ONLY via self-deploy:** Gemma, Gemma 3n, Mistral, Phi-4

## Deployment Options

```
Self-Deploy Options
├── 1. One-Click SDK Deploy     → OpenModel().deploy() — simplest path
├── 2. Prebuilt Containers      → vLLM, SGLang, TGI, TensorRT-LLM, Hex-LLM
├── 3. Custom vLLM Container    → Build your own container, max flexibility
└── 4. Custom Weights           → Your fine-tuned model from GCS
```

## When to Use Self-Deploy

- Fine-tuned or custom model weights
- Predictable, high-volume traffic (lower total cost vs. MaaS)
- Strict VPC isolation / data residency requirements
- Need specific GPU/TPU hardware (L4, H100, TPU v5)
- Custom preprocessing/postprocessing logic

## Prerequisites

```bash
# Enable required APIs
gcloud services enable aiplatform.googleapis.com
gcloud services enable artifactregistry.googleapis.com  # for custom containers

# Install Python SDK
pip install "google-cloud-aiplatform>=1.106.0" openai google-auth requests
```

---

## Option 1: One-Click SDK Deployment (Recommended Starting Point)

Deploy any Model Garden model with minimal configuration.

### List Available Models

```python
import vertexai
from vertexai import model_garden

vertexai.init(project="PROJECT_ID", location="us-central1")

# List all deployable models
models = model_garden.list_deployable_models()

# Filter by name (parameter is model_filter, NOT filter)
gemma_models = model_garden.list_deployable_models(model_filter="gemma")
for m in gemma_models:
    print(m)
```

### Check Deployment Options Before Deploying

```python
# Preview hardware requirements and available configs (concise=True for summary)
model = model_garden.OpenModel("google/gemma3@gemma-3-12b-it")
model.list_deploy_options(concise=True)
```

### Deploy a Model

```python
import vertexai
from vertexai import model_garden

vertexai.init(project="PROJECT_ID", location="us-central1")

# Model ID format: "provider/model-name@variant"
MODEL_ID = "google/gemma3@gemma-3-1b-it"

model = model_garden.OpenModel(MODEL_ID)
endpoint = model.deploy(
    accept_eula=True,                          # Required for EULA models
    hugging_face_access_token="HF_TOKEN",      # For gated HF models
    use_dedicated_endpoint=True,               # Recommended for production
)
```

### Deploy with Specific Hardware

```python
MODEL_ID = "meta/llama3-3@llama-3.3-70b-instruct-fp8"

model = model_garden.OpenModel(MODEL_ID)
endpoint = model.deploy(
    accept_eula=True,
    machine_type="a3-ultragpu-8g",
    accelerator_type="NVIDIA_H200_141GB",
    accelerator_count=8,
    serving_container_image_uri=(
        "us-docker.pkg.dev/deeplearning-platform-release/"
        "vertex-model-garden/tensorrt-llm.cu128.0-18.ubuntu2404.py312:latest"
    ),
    endpoint_display_name="llama-3-3-70b-endpoint",
    model_display_name="llama-3-3-70b-model",
    use_dedicated_endpoint=True,
)
```

### Run Inference

```python
import json, openai, google.auth, google.auth.transport.requests

# Raw prediction (works for all deployment types)
response = endpoint.predict(
    instances=[{
        "prompt": "Explain transformer architecture in simple terms.",
        "max_tokens": 512,
        "temperature": 0.7,
        "top_p": 0.9,
    }]
)
print(response.predictions[0])

# OpenAI-compatible chat completion (dedicated endpoint)
REGION = "us-central1"
# endpoint.resource_name format: projects/.../locations/.../endpoints/ENDPOINT_ID
BASE_URL = f"https://{REGION}-aiplatform.googleapis.com/v1beta1/{endpoint.resource_name}"

creds, _ = google.auth.default()
creds.refresh(google.auth.transport.requests.Request())

client = openai.OpenAI(base_url=BASE_URL, api_key=creds.token)
completion = client.chat.completions.create(
    model="",  # model is embedded in endpoint URL
    messages=[{"role": "user", "content": "Tell me a joke about AI."}],
    max_tokens=256,
)
print(completion.choices[0].message.content)
```

---

## Option 2: Prebuilt Containers

Use Google-optimized serving frameworks without building your own container.

### Available Containers

| Framework | Best For | Hardware | Port | Predict Route |
|-----------|----------|----------|------|---------------|
| **vLLM** | General GPU/CPU serving, OpenAI-compatible API | GPU, CPU | 8080 | `/v1/completions` |
| **SGLang** | DeepSeek, Qwen3, structured/high-throughput | GPU | 8080 | `/v1/completions` |
| **TensorRT-LLM** | NVIDIA-optimized high throughput | H100, H200 | 8080 | `/v1/completions` |
| **Hex-LLM** | TPU-based LLM serving (XLA-optimized) | TPU v5 | 7080 | `/generate` |
| **HF TGI** | HuggingFace generative models | GPU | 8080 | `/generate` |
| **HF TEI** | Embedding models | GPU/CPU | 8080 | (SDK predict) |
| **HF PyTorch Inference** | Classification, NLP tasks | GPU | 7080 | `/pred` |

### Deploy with vLLM Prebuilt Container (GPU)

```python
from google.cloud import aiplatform

aiplatform.init(project="PROJECT_ID", location="us-central1")

MODEL_ID = "google/gemma-3-1b-it"
ACCELERATOR_COUNT = 1

vllm_args = [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--host=0.0.0.0",
    "--port=8080",
    f"--model={MODEL_ID}",
    "--max-model-len=8192",
    "--gpu-memory-utilization=0.9",
    "--enable-prefix-caching",
    f"--tensor-parallel-size={ACCELERATOR_COUNT}",
]

# Upload model specification
model = aiplatform.Model.upload(
    display_name="gemma-3-vllm",
    serving_container_image_uri=(
        "us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/"
        "pytorch-vllm-serve:20251205_0916_RC01"
    ),
    serving_container_args=vllm_args,
    serving_container_environment_variables={
        "HF_TOKEN": "YOUR_HF_TOKEN",
        "LD_LIBRARY_PATH": "$LD_LIBRARY_PATH:/usr/local/nvidia/lib64",
    },
    serving_container_ports=[8080],
    serving_container_predict_route="/v1/completions",
    serving_container_health_route="/health",
    serving_container_shared_memory_size_mb=(16 * 1024),  # 16 GB
    serving_container_deployment_timeout=1800,
)

# Create endpoint and deploy
endpoint = aiplatform.Endpoint.create(display_name="gemma-3-endpoint")
model.deploy(
    endpoint=endpoint,
    deployed_model_display_name="gemma-3-vllm",
    machine_type="g2-standard-12",
    accelerator_type="NVIDIA_L4",
    accelerator_count=ACCELERATOR_COUNT,
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=2,
    autoscaling_target_accelerator_duty_cycle=60,
)
```

### Deploy with HF TGI (Text Generation Inference)

```python
from google.cloud import aiplatform

aiplatform.init(project="PROJECT_ID", location="us-central1")

MODEL_ID = "google/gemma-2-2b-it"

model = aiplatform.Model.upload(
    display_name="gemma-2-tgi",
    serving_container_image_uri=(
        "us-docker.pkg.dev/deeplearning-platform-release/gcr.io/"
        "huggingface-text-generation-inference-cu124.2-4.ubuntu2204.py311"
    ),
    serving_container_environment_variables={
        "MODEL_ID": MODEL_ID,
        "NUM_SHARD": "1",            # Number of GPUs for sharding
        "MAX_INPUT_LENGTH": "2047",
        "MAX_TOTAL_TOKENS": "2048",
        "MAX_BATCH_PREFILL_TOKENS": "2048",
        "HF_TOKEN": "YOUR_HF_TOKEN", # Required for gated models
    },
    serving_container_ports=[8080],
    serving_container_predict_route="/generate",
    serving_container_health_route="/health",
    serving_container_shared_memory_size_mb=(16 * 1024),
    serving_container_deployment_timeout=1800,
)

endpoint = aiplatform.Endpoint.create(display_name="gemma-2-tgi-endpoint")
model.deploy(
    endpoint=endpoint,
    machine_type="g2-standard-12",
    accelerator_type="NVIDIA_L4",
    accelerator_count=1,
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=2,
)

# Inference
response = endpoint.predict(instances=[{
    "inputs": "What is the capital of France?",
    "parameters": {
        "max_new_tokens": 256,
        "temperature": 1.0,
        "top_p": 0.9,
        "top_k": 1,
    },
}])
print(response.predictions[0])
```

### Deploy with HF TEI (Text Embeddings Inference)

```python
MODEL_ID = "Qwen/Qwen3-Embedding-8B"  # or any HF embedding model

model = aiplatform.Model.upload(
    display_name="qwen3-embedding-tei",
    serving_container_image_uri=(
        "us-docker.pkg.dev/deeplearning-platform-release/vertex-model-garden/"
        "hf-tei.cu125.0-1.ubuntu2204.py310:model-garden.hf-tei-0-1-release_20251205.00_p0"
    ),
    serving_container_environment_variables={
        "MODEL_ID": MODEL_ID,
        "JSON_OUTPUT": "true",
        "HF_API_TOKEN": "YOUR_HF_TOKEN",  # Optional, avoids rate limiting
    },
    serving_container_ports=[8080],
    serving_container_shared_memory_size_mb=(4 * 1024),  # 4 GB for embedding models
    serving_container_deployment_timeout=1800,
)

endpoint = aiplatform.Endpoint.create(display_name="embedding-tei-endpoint")
model.deploy(
    endpoint=endpoint,
    machine_type="g2-standard-8",
    accelerator_type="NVIDIA_L4",
    accelerator_count=1,
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=2,
)

# Inference
response = endpoint.predict(instances=[{"inputs": "This is a sentence."}])
print(response.predictions)
```

### Deploy with HF PyTorch Inference

> Note: Uses port **7080** and route `/pred` — different from vLLM/TGI.

```python
model = aiplatform.Model.upload(
    display_name="distilbert-sentiment",
    serving_container_image_uri=(
        "us-docker.pkg.dev/deeplearning-platform-release/vertex-model-garden/"
        "hf-inference-toolkit.cu125.0-1.ubuntu2204.py311:"
        "model-garden.hf-inference-toolkit-0-1-release_20251206.00_p0"
    ),
    serving_container_environment_variables={
        "HF_TASK": "text-classification",       # Task type
        "HF_MODEL_ID": "distilbert-base-uncased-finetuned-sst-2-english",
        "HF_TOKEN": "YOUR_HF_TOKEN",
    },
    serving_container_ports=[7080],
    serving_container_predict_route="/pred",
    serving_container_health_route="/ping",
    serving_container_deployment_timeout=1800,
)

# Inference
response = endpoint.predict(
    instances=["I love this product!", "This is terrible."],
)
print(response.predictions)
```

### Deploy with Hex-LLM (TPU)

> Note: Uses port **7080**, route `/generate`, and `serving_container_command` (not `args`).

```python
from google.cloud import aiplatform

TPU_COUNT = 4    # 1 for 1B, 4 for 8B, 16 for 70B
MODEL_ID = "meta-llama/Llama-3.1-8B-Instruct"

hexllm_args = [
    "--host=0.0.0.0",
    "--port=7080",
    f"--model={MODEL_ID}",
    f"--tensor_parallel_size={TPU_COUNT}",
    "--hbm_utilization_factor=0.9",  # TPU HBM usage (0.0–1.0)
    "--max_running_seqs=256",
    "--max_model_len=4096",
]

model = aiplatform.Model.upload(
    display_name="llama-hexllm",
    serving_container_image_uri=(
        "us-docker.pkg.dev/vertex-ai-restricted/vertex-vision-model-garden-dockers/"
        "hex-llm-serve:20241210_2323_RC00"
    ),
    serving_container_command=["python", "-m", "hex_llm.server.api_server"],
    serving_container_args=hexllm_args,
    serving_container_environment_variables={
        "MODEL_ID": MODEL_ID,
        "HF_TOKEN": "YOUR_HF_TOKEN",
        "HEX_LLM_LOG_LEVEL": "info",
    },
    serving_container_ports=[7080],
    serving_container_predict_route="/generate",
    serving_container_health_route="/ping",
    serving_container_shared_memory_size_mb=(16 * 1024),
)

endpoint = aiplatform.Endpoint.create(display_name="llama-hexllm-endpoint")
model.deploy(
    endpoint=endpoint,
    machine_type="ct5lp-hightpu-4t",  # 4 TPU v5e cores
    tpu_topology="2x2",               # Required for TPU deployment
    service_account="SERVICE_ACCOUNT_EMAIL",
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=1,
)

# Inference (raw predict)
response = endpoint.predict(instances=[{
    "prompt": "What is quantum entanglement?",
    "max_tokens": 100,
    "temperature": 0.8,
    "top_p": 1.0,
    "top_k": 1,
}])
print(response.predictions)

# OpenAI-compatible chat completion
import openai, google.auth, google.auth.transport.requests

creds, _ = google.auth.default()
creds.refresh(google.auth.transport.requests.Request())
BASE_URL = f"https://us-central1-aiplatform.googleapis.com/v1beta1/{endpoint.resource_name}"

client = openai.OpenAI(base_url=BASE_URL, api_key=creds.token)
response = client.chat.completions.create(
    model="hexllm",
    messages=[{"role": "user", "content": "What is a plane?"}],
    temperature=1.0,
    max_tokens=50,
)
print(response.choices[0].message.content)
```

---

## Option 3: Custom vLLM Container

For maximum flexibility — custom preprocessing, postprocessing, or non-standard configs.

### Step 1: Create Dockerfile

```dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
    apt-get install -y apt-utils git apt-transport-https gnupg ca-certificates curl \
    && echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | \
       tee -a /etc/apt/sources.list.d/google-cloud-sdk.list \
    && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
       gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg \
    && apt-get update -y && apt-get install google-cloud-cli -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace/vllm
COPY ./entrypoint.sh /workspace/vllm/vertexai/entrypoint.sh
RUN chmod +x /workspace/vllm/vertexai/entrypoint.sh
ENTRYPOINT ["/workspace/vllm/vertexai/entrypoint.sh"]
```

### Step 2: Build with Cloud Build

```bash
# Create Artifact Registry repo
gcloud artifacts repositories create model-serving \
    --repository-format=docker \
    --location=us-central1 \
    --description="Vertex AI Docker repository"

# Configure Docker auth
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

# Build and push (GPU)
DOCKER_REPO="us-central1-docker.pkg.dev/PROJECT_ID/model-serving"
gcloud builds submit --config=cloudbuild.yaml \
    --region=us-central1 \
    --timeout="2h" \
    --machine-type=e2-highcpu-32 \
    --substitutions=_REPOSITORY=${DOCKER_REPO},_DEVICE_TYPE=gpu,_BASE_IMAGE=vllm/vllm-openai
```

### Step 3: Deploy Custom Container

```python
MODEL_ID = "meta-llama/Llama-3.2-3B-Instruct"
ACCELERATOR_COUNT = 1

vllm_args = [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--host=0.0.0.0", "--port=8080",
    f"--model={MODEL_ID}",
    "--max-model-len=4096",
    "--gpu-memory-utilization=0.9",
    f"--tensor-parallel-size={ACCELERATOR_COUNT}",
]

model = aiplatform.Model.upload(
    display_name="custom-vllm-model",
    serving_container_image_uri=(
        "us-central1-docker.pkg.dev/PROJECT_ID/model-serving/vllm-gpu:latest"
    ),
    serving_container_args=vllm_args,
    serving_container_environment_variables={
        "HF_TOKEN": "YOUR_HF_TOKEN",
        "LD_LIBRARY_PATH": "$LD_LIBRARY_PATH:/usr/local/nvidia/lib64",
    },
    serving_container_ports=[8080],
    serving_container_predict_route="/v1/completions",
    serving_container_health_route="/health",
    serving_container_shared_memory_size_mb=(16 * 1024),
    serving_container_deployment_timeout=1800,
)
```

### Step 4: Get Predictions

```python
import json

response = endpoint.raw_predict(
    body=json.dumps({
        "prompt": "Distance of moon from earth is ",
        "temperature": 0.0,
    }),
    headers={"Content-Type": "application/json"},
)
print(json.loads(response.text))
```

### CPU Deployment Variant

No GPU required. Use CPU-optimized machine types and omit `accelerator_*` parameters.

```bash
# Build CPU container
gcloud builds submit --config=cloudbuild.yaml \
    --region=us-central1 \
    --timeout="2h" \
    --machine-type=e2-highcpu-32 \
    --substitutions=_REPOSITORY=${DOCKER_REPO},_DEVICE_TYPE=cpu,_BASE_IMAGE=vllm/vllm-openai
```

```python
vllm_args = [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--host=0.0.0.0", "--port=8080",
    "--model=meta-llama/Llama-3.2-3B",
    "--max-model-len=2048",   # Lower limit recommended for CPU
]

model = aiplatform.Model.upload(
    display_name="llama-vllm-cpu",
    serving_container_image_uri=(
        "us-central1-docker.pkg.dev/PROJECT_ID/model-serving/vllm-cpu:latest"
    ),
    serving_container_args=vllm_args,
    serving_container_environment_variables={"HF_TOKEN": "YOUR_HF_TOKEN"},
    serving_container_ports=[8080],
    serving_container_predict_route="/v1/completions",
    serving_container_health_route="/health",
    serving_container_shared_memory_size_mb=(16 * 1024),
    serving_container_deployment_timeout=1800,
)

endpoint = aiplatform.Endpoint.create(display_name="llama-cpu-endpoint")
model.deploy(
    endpoint=endpoint,
    machine_type="c2-standard-16",   # CPU-optimized, no accelerator
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=4,
    autoscaling_target_cpu_utilization=60,
)
```

### TPU Deployment Variant

```bash
# Build TPU container (uses vllm-tpu:nightly base)
gcloud builds submit --config=cloudbuild.yaml \
    --region=us-central1 \
    --timeout="2h" \
    --machine-type=e2-highcpu-32 \
    --substitutions=_REPOSITORY=${DOCKER_REPO},_DEVICE_TYPE=tpu,_BASE_IMAGE=vllm/vllm-tpu:nightly
```

```python
TPU_COUNT = 1   # Number of TPU chips
MODEL_ID = "meta-llama/Llama-3.2-3B"

vllm_args = [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--host=0.0.0.0", "--port=8080",
    f"--model={MODEL_ID}",
    "--max-model-len=2048",
    "--enable-prefix-caching",
    f"--tensor-parallel-size={TPU_COUNT}",
]

model = aiplatform.Model.upload(
    display_name="llama-vllm-tpu",
    serving_container_image_uri=(
        "us-central1-docker.pkg.dev/PROJECT_ID/model-serving/vllm-tpu:latest"
    ),
    serving_container_args=vllm_args,
    serving_container_environment_variables={"HF_TOKEN": "YOUR_HF_TOKEN"},
    serving_container_ports=[8080],
    serving_container_predict_route="/v1/completions",
    serving_container_health_route="/health",
    serving_container_shared_memory_size_mb=(16 * 1024),
    serving_container_deployment_timeout=1800,
)

endpoint = aiplatform.Endpoint.create(display_name="llama-tpu-endpoint")
model.deploy(
    endpoint=endpoint,
    machine_type="ct5lp-hightpu-1t",  # TPU v5e, 1 chip
    service_account="SERVICE_ACCOUNT_EMAIL",  # Needs storage.objectViewer for GCS models
    traffic_percentage=100,
    deploy_request_timeout=1800,
    min_replica_count=1,
    max_replica_count=1,
)
```

> **TPU + GCS model:** Store model in GCS and reference the GCS path as `--model=gs://bucket/model-path`. Grant the service account `roles/storage.objectViewer` on the bucket. Omit `HF_TOKEN`.

---

## Option 4: Custom Weights Deployment (Preview)

Deploy your fine-tuned model weights stored in Cloud Storage.

### Supported Base Models

Llama (2, 3.1, 3.2, 4), Gemma (2, 3, MedGemma), Qwen (2, 2.5, 3), DeepSeek (R1, V3, V3.1), Mistral/Mixtral, Phi-4

### Required Files in GCS

```
gs://your-bucket/model-path/
├── config.json               # Model configuration
├── model.safetensors         # Weights (safetensors preferred over .bin)
├── model.safetensors.index.json
├── tokenizer.json
├── tokenizer_config.json
└── tokenizer.model
```

### Upload Model from Hugging Face to GCS

```python
from huggingface_hub import snapshot_download
import subprocess

# Download from HF
local_dir = snapshot_download(
    repo_id="your-org/your-fine-tuned-model",
    token="HF_TOKEN",
)

# Upload to GCS
GCS_URI = "gs://your-bucket/your-model"
subprocess.run([
    "gsutil", "-m", "cp", "-r", f"{local_dir}/*", GCS_URI
], check=True)
```

### Python SDK Deployment

```python
import vertexai
from vertexai.preview import model_garden

vertexai.init(project="PROJECT_ID", location="us-central1")

GCS_URI = "gs://your-bucket/your-fine-tuned-model"

custom_model = model_garden.CustomModel(gcs_uri=GCS_URI)

# Check available deployment options
options = custom_model.list_deploy_options()
for opt in options:
    print(opt)

# Deploy
endpoint = custom_model.deploy(
    machine_type="g2-standard-12",
    accelerator_type="NVIDIA_L4",
    accelerator_count=1,
)

# Predict
response = endpoint.predict(
    instances=[{"prompt": "What is the capital of France?"}],
    use_dedicated_endpoint=True,
)
print(response.predictions[0])
```

### gcloud CLI Deployment

```bash
# Minimal (auto-selects hardware)
gcloud ai model-garden models deploy \
    --model=gs://your-bucket/your-fine-tuned-model \
    --region=us-central1

# With explicit hardware specification
gcloud ai model-garden models deploy \
    --model=gs://your-bucket/your-fine-tuned-model \
    --machine-type=g2-standard-12 \
    --accelerator-type=NVIDIA_L4 \
    --accelerator-count=1 \
    --region=us-central1
```

---

## Fine-Tuning on Vertex AI

Fine-tune Gemma models using Vertex AI Training Jobs and deploy the results.

### Framework Comparison

| Framework | Models | Method | GPUs (train) | LoRA Rank | Output |
|-----------|--------|--------|--------------|-----------|--------|
| **Axolotl** | Gemma 3 (1B–27B) | LoRA / QLoRA | 8x A100 or 8x H100 | config-driven | GCS (HF merged) |
| **HF PEFT** | Gemma 1 (2B/7B), Gemma 2 (2B/9B/27B) | LoRA 4-bit | 8x A100 or 8x H100 | 16 | GCS (adapter + merged) |
| **KerasNLP** | Gemma 1 (2B/7B) | LoRA | 1x L4 | 4 | GCS (HDF5 → HF) |
| **Ray + TRL** | Gemma 1 2B | QLoRA | 1x A100 | 32 | GCS (HF merged) |

### Axolotl (Gemma 3 — Recommended)

Config-file driven LoRA/QLoRA training with Dynamic Workload Scheduler.

```python
from google.cloud import aiplatform

aiplatform.init(project="PROJECT_ID", location="us-central1")

# Training container
TRAIN_IMAGE = (
    "us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/"
    "axolotl-train-dws:20250812-1800-rc1"
)

job = aiplatform.CustomContainerTrainingJob(
    display_name="gemma3-axolotl-finetune",
    container_uri=TRAIN_IMAGE,
)

job.run(
    args=[
        "--base_model=google/gemma-3-4b-it",
        "--output_dir=gs://your-bucket/axolotl/output",
        "--max_steps=1000",
        "--val_set_size=0.1",
    ],
    replica_count=1,
    machine_type="a3-highgpu-8g",
    accelerator_type="NVIDIA_H100_80GB",
    accelerator_count=8,
    boot_disk_size_gb=2000,   # Large disk for H100 jobs
)
```

vLLM serving of merged output:
```python
# Inference container
"us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/pytorch-vllm-serve:20250312_0916_RC01"
# 1B → g2-standard-12 + 1x L4; 4B/12B → a3-highgpu-2g + 2x H100; 27B → a3-highgpu-4g + 4x H100
# max_model_len: 32768 (1B), 131072 (larger); gpu_memory_utilization: 0.95
```

### HF PEFT (Gemma 1 / Gemma 2)

Standard LoRA with 4-bit quantization via HuggingFace PEFT library.

```python
# Training container (Gemma 2)
TRAIN_IMAGE = (
    "us-docker.pkg.dev/vertex-ai-restricted/vertex-vision-model-garden-dockers/"
    "pytorch-peft-train:stable_20250705"
)

# Key hyperparameters (passed as training args)
# --lora_rank=16 --lora_alpha=32 --lora_dropout=0.05
# --learning_rate=5e-5 --lr_scheduler_type=cosine
# --per_device_train_batch_size=1 --gradient_accumulation_steps=4
# --max_seq_length=4096 --num_train_epochs=1
# --quantization=4bit --train_precision=bfloat16
# --optimizer=paged_adamw_32bit --warmup_ratio=0.01

# Dataset format (JSONL)
# {"text": "### Human: <prompt>### Assistant: <response>"}

# Output paths
# Adapter:      gs://bucket/job_name/adapter/
# Merged model: gs://bucket/job_name/merged-model/
```

Machine types: `a2-ultragpu-8g` (8x A100 80GB) or `a3-highgpu-8g` (8x H100 80GB, via DWS)

> **Container registry difference:** A100 training uses `vertex-ai-restricted` registry; H100 training uses `vertex-ai` (public) registry. Both use tag `stable_20250705`.

> **`--attn_implementation`** is a **training** flag (not vLLM serving), passed to the PEFT training container to set attention implementation (e.g., `eager`).

### KerasNLP (Gemma 1 — Lightweight)

Single L4 GPU fine-tuning using Keras 3 with PyTorch backend. Best for quick experiments.

```python
# Preset IDs for KerasNLP
"gemma_2b_en"           # Gemma 1 2B base
"gemma_instruct_2b_en"  # Gemma 1 2B instruct
"gemma_7b_en"           # Gemma 1 7B base
"gemma_instruct_7b_en"  # Gemma 1 7B instruct

# LoRA rank=4, lr=5e-5, batch_size=1, seq_len=128, bfloat16
# Output: HDF5 weights → convert to HF format via KerasNLP export utility
# Machine: g2-standard-8 (1x L4) for 2B; g2-standard-12 for 7B
```

### Ray Distributed (Gemma 1 — Multi-node)

Ray on Vertex for distributed training with ROUGE evaluation and batch inference.

```python
# Head node: n1-standard-16 (CPU); Worker: a2-highgpu-1g (1x A100)
# QLoRA: rank=32, alpha=32, dropout=0.05, target_modules="all-linear"
# lr=2e-4, batch_size=1, grad_accum=4, max_steps=100, max_seq_len=512
# Dataset: formatted with Gemma chat template (user/assistant pairs)
# Output: GCS via save_pretrained(), evaluated with ROUGE metrics
```

---

## Hardware Reference

### GPU Machine Types

| Machine Type | GPUs | VRAM | Recommended Models |
|-------------|------|------|--------------------|
| `g2-standard-8` | 1x L4 | 24 GB | 3B–7B models |
| `g2-standard-12` | 1x L4 | 24 GB | 7B models |
| `g2-standard-48` | 4x L4 | 96 GB | 13B–34B models |
| `a2-highgpu-1g` | 1x A100 40GB | 40 GB | 13B models |
| `a2-ultragpu-1g` | 1x A100 80GB | 80 GB | 34B models, Llama 3.2 11B Vision |
| `a3-highgpu-8g` | 8x H100 80GB | 640 GB | 70B+ models, Llama 3.2 90B Vision |
| `a3-ultragpu-8g` | 8x H200 | 1 TB | 70B+ FP8, DeepSeek-V3/R1 single host |
| `g4-standard-48` | 1x RTX Pro 6000 | 96 GB | Gemma 3 (vLLM) |

### CPU Machine Types

| Machine Type | vCPUs | Use Case |
|-------------|-------|----------|
| `c2-standard-16` | 16 | CPU vLLM (low-traffic, no GPU quota) |
| `c2-standard-60` | 60 | CPU vLLM (higher throughput) |

### TPU Machine Types

| Machine Type | TPU | Cores | Recommended Models | Container |
|-------------|-----|-------|--------------------|----|
| `ct5lp-hightpu-1t` | v5e | 1 | 1B–3B (vLLM/Hex-LLM) | `pytorch-vllm-serve:*tpu_experimental*` |
| `ct5lp-hightpu-4t` | v5e | 4 | 8B (Hex-LLM, vLLM) | `hex-llm-serve:*` |
| `ct5lp-hightpu-8t` | v5e | 8 | 27B Gemma 2 (Hex-LLM) | `hex-llm-serve:*` |
| `ct6e-standard-1t` | v6e | 8 | Llama 3.1 8B, Qwen3 4B/8B | `vllm-inference-tpu.*` |
| `ct6e-standard-4t` | v6e | 32 | Qwen3 32B | `pytorch-vllm-serve:*tpu_experimental*` |
| `tpu7x-standard-4t` | v7x | — | Llama 3.3 70B | `vllm-serve:tpu7x` |

**TPU container images:**
```python
# v5e (vLLM experimental) — Llama 3.1/Qwen2.5
"us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/pytorch-vllm-serve:20241107_0917_tpu_experimental_RC01"

# v6e (vLLM + Qwen3) — Llama 3.1 / Qwen3
"us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/pytorch-vllm-serve:20250529_0917_tpu_experimental_RC00"
# or (Model Garden SDK path)
"us-docker.pkg.dev/deeplearning-platform-release/vertex-model-garden/vllm-inference-tpu.0-11.ubuntu2204.py312:model-garden.vllm-tpu-release_20251015.00_p0"

# v7x — Llama 3.3 70B
"us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/vllm-serve:tpu7x"
# port 7080, predict /generate, health /ping
# --max-model-len=65536, --max-running-seqs=128, VLLM_USE_V1=1
```

> **Qwen3 "thinking" disable on TPU:** append `<think></think>` to the prompt to skip reasoning mode.

### Accelerator Type Constants

```python
# GPU accelerator_type values
"NVIDIA_L4"                  # g2-standard-* (24 GB/chip)
"NVIDIA_TESLA_A100"          # a2-highgpu-1g (40 GB)
"NVIDIA_A100_80GB"           # a2-ultragpu-1g (80 GB) — Gemma 3n
"NVIDIA_H100_80GB"           # a3-highgpu-8g
"NVIDIA_H200_141GB"          # a3-ultragpu-8g
"NVIDIA_RTX_PRO_6000"        # g4-standard-48 (96 GB) — Gemma 3
# TPU: set machine_type to ct5lp-* or ct6e-* and use tpu_topology parameter
```

---

## Model-Specific Notes

### DeepSeek-V3 / R1 (671B MoE)

Three serving options, ranked by availability:
1. **SGLang** — recommended, supports low-latency and high-throughput profiles
   - Container: `us-docker.pkg.dev/deeplearning-platform-release/vertex-model-garden/sglang-serve.cu124.0-4.ubuntu2204.py310:20250427-1800-rc0`
2. **vLLM** — single host (8x H200) or multi-host (2x 8x H100)
3. **TensorRT-LLM** — H200 only, limited regions (us-south1, asia-south2, us-east4)

### Qwen3 Thinking Mode

Thinking-enabled variants generate `<|think|>...</|think|>` blocks. Use these sampling parameters:

```python
parameters = {
    "sampling_params": {
        "max_new_tokens": 8192,  # Must be large to fit thinking content
        "temperature": 0.6,      # Recommended for thinking models
        "top_p": 0.95,
        "top_k": 20,
        "min_p": 0,
    }
}
response = endpoint.predict(instances=[{"text": prompt}], parameters=parameters)
```

### Multimodal Models (Llama 3.2 Vision, Gemma 3, Gemma 3n)

**Gemma 3 / Llama 3.2** — text + image via OpenAI-compatible chat:

```python
response = client.chat.completions.create(
    model="",
    messages=[{
        "role": "user",
        "content": [
            {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
            {"type": "text", "text": "Describe this image."},
        ],
    }],
    temperature=1.0,
    max_tokens=256,
)
```

**Gemma 3n** — text + image + **audio** (unique, uses SGLang):

```python
# Audio input uses audio_url (direct URL, not base64)
AUDIO_URL = "https://example.com/audio.wav"  # publicly accessible URL

response = client.chat.completions.create(
    model="",
    messages=[{
        "role": "user",
        "content": [
            {"type": "audio_url", "audio_url": {"url": AUDIO_URL}},
            {"type": "text", "text": "Transcribe and summarize this audio."},
        ],
    }],
)
```

Hardware mapping for multimodal models:
- Llama 3.2 1B/3B → L4 (`g2-standard-8`)
- Llama 3.2 11B Vision → A100 80GB (`a2-ultragpu-1g`)
- Llama 3.2 90B Vision → 8x H100 (`a3-highgpu-8g`)
- Gemma 3 4B–27B → RTX Pro 6000 (`g4-standard-48`) via vLLM
- Gemma 3n e2b/e4b → A100 80GB (`a2-ultragpu-1g`) via SGLang

> `gemma-3-1b-it` does **not** support multimodal. Use 4B or larger.
> Gemma 3n is the **only** model supporting audio input.

### Gemma Model IDs

#### Gemma 3

```python
# Instruction-tuned (recommended for chat/instruction following)
"google/gemma3@gemma-3-270m-it"
"google/gemma3@gemma-3-1b-it"
"google/gemma3@gemma-3-4b-it"    # default, multimodal (text+image)
"google/gemma3@gemma-3-12b-it"   # multimodal
"google/gemma3@gemma-3-27b-it"   # multimodal

# Pretrained (base) variants
"google/gemma3@gemma-3-270m"
"google/gemma3@gemma-3-1b-pt"
"google/gemma3@gemma-3-4b-pt"
"google/gemma3@gemma-3-12b-pt"
"google/gemma3@gemma-3-27b-pt"
```

> **Note:** `gemma-3-1b-it` does NOT support multimodal (image) input. Use 4B or larger.
> **Serving:** vLLM (`g4-standard-48` + `NVIDIA_RTX_PRO_6000`)
> **Container:** `us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/pytorch-vllm-serve:20251205_0916_RC01`

#### Gemma 3n

```python
# Instruction-tuned
"google/gemma3n@gemma-3n-e2b-it"   # multimodal: text + image + audio
"google/gemma3n@gemma-3n-e4b-it"   # multimodal: text + image + audio

# Pretrained
"google/gemma3n@gemma-3n-e2b"
"google/gemma3n@gemma-3n-e4b"
```

> **Unique:** Gemma 3n supports **audio** input in addition to text and image.
> **Serving:** SGLang (not vLLM) on A100 80GB (`a2-ultragpu-1g`)
> **Container:** `us-docker.pkg.dev/deeplearning-platform-release/vertex-model-garden/sglang-serve.cu124.0-4.ubuntu2204.py310:model-garden.sglang-0-4-release_20250817.00_p0`

#### Gemma 2

```python
# Text-only models
"google/gemma-2-2b"
"google/gemma-2-2b-it"    # 2B instruction-tuned
"google/gemma-2-9b"
"google/gemma-2-9b-it"    # 9B instruction-tuned
"google/gemma-2-27b"
"google/gemma-2-27b-it"   # 27B instruction-tuned
```

Two deployment paths for Gemma 2:

**GPU (TGI) — general availability:**
```python
# TGI container
"us-docker.pkg.dev/deeplearning-platform-release/gcr.io/huggingface-text-generation-inference-cu121.2-1.ubuntu2204.py310"
# Machine: g2-standard-12 (2B), g2-standard-24 (9B), g2-standard-48 (27B)
# Accelerator: NVIDIA_L4
# NUM_SHARD: match GPU count; MAX_TOTAL_TOKENS: 2048
```

**TPU (Hex-LLM) — us-west1 only:**
```python
# Container: hex-llm-serve:20241210_2323_RC00 (vertex-ai-restricted registry)
# Machine: ct5lp-hightpu-1t (2B), ct5lp-hightpu-4t (9B, tpu_topology="1x1"), ct5lp-hightpu-8t (27B)
# hbm_utilization_factor: 0.6; deployment_timeout: 7200s (vs 1800s for GPU)
# Requires HF_TOKEN and service account with roles/storage.objectViewer
```

---

## Option 5: GKE Deployment

Deploy models on Google Kubernetes Engine for full infrastructure control, multi-tenant serving, or when existing GKE clusters are already in use.

> **When to choose GKE over Vertex AI endpoint:**
> - Existing GKE infrastructure / multi-tenant clusters
> - Need fine-grained pod-level resource control
> - Custom ingress, sidecar containers, or service mesh requirements
> - Cost optimization via shared node pools

### Cluster Setup

```bash
# Autopilot (recommended — GKE manages nodes automatically)
gcloud container clusters create-auto CLUSTER_NAME \
    --location=us-central1 \
    --project=PROJECT_ID

# Standard (manual node pool control; --workload-pool needed for Workload Identity)
gcloud container clusters create CLUSTER_NAME \
    --project=PROJECT_ID \
    --region=us-central1 \
    --release-channel=rapid \
    --num-nodes=4 \
    --workload-pool=PROJECT_ID.svc.id.goog  # optional: only if using Workload Identity

# Add GPU node pool (Standard clusters only)
gcloud container node-pools create gpupool \
    --accelerator=type=nvidia-l4,count=2,gpu-driver-version=latest \
    --cluster=CLUSTER_NAME \
    --machine-type=g2-standard-24 \
    --num-nodes=1
```

### Store HuggingFace Token as Kubernetes Secret

```bash
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=YOUR_HF_TOKEN
```

### Deploy with vLLM (Llama 3.2)

```yaml
# llama32-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  containers:
  - name: vllm-server
    image: us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/pytorch-vllm-serve:20241007_2233_RC00
    env:
    - name: MODEL_ID
      value: "meta-llama/Llama-3.2-3B-Instruct"
    - name: DEPLOY_SOURCE
      value: "notebook"
    - name: HUGGING_FACE_HUB_TOKEN
      valueFrom:
        secretKeyRef:
          name: hf-secret
          key: hf_api_token
    args:
    - --host=0.0.0.0
    - --port=7080
    - --swap-space=16
    - --gpu-memory-utilization=0.95
    - --tensor-parallel-size=1
    resources:
      limits:
        cpu: "10"
        memory: "39Gi"
        ephemeral-storage: "100Gi"
        nvidia.com/gpu: "1"
    volumeMounts:
    - name: shm
      mountPath: /dev/shm
  volumes:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 512Mi
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-l4
```

```bash
kubectl apply -f llama32-deployment.yaml
kubectl apply -f llama32-service.yaml  # ClusterIP, port 8000 → container 7080
```

### Deploy with TGI (Gemma / Llama / Mistral)

```yaml
# Key differences from vLLM:
# - Container: huggingface-text-generation-inference-cu121.*
# - Inference endpoint: POST /generate (port 8080)
# - Env: MODEL_ID, NUM_SHARD, MAX_INPUT_TOKENS, MAX_TOTAL_TOKENS, CUDA_MEMORY_FRACTION
```

### Resource Sizing

| Model | GPU Count | CPUs | Memory | Ephemeral |
|-------|-----------|------|--------|-----------|
| 2B (TGI) | 1x L4 | 2 | 7Gi | 20Gi |
| 7B–8B (TGI) | 1–2x L4 | 3–10 | 10–25Gi | 40–80Gi |
| 11B Vision (vLLM) | 2x L4 | 15 | 58Gi | 120Gi |

### Inference API

```bash
# vLLM (port 7080)
curl -X POST http://localhost:7080/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is quantum computing?", "max_tokens": 100, "temperature": 0.9}'

# TGI (port 8080)
curl -X POST http://localhost:8080/generate \
  -H "Content-Type: application/json" \
  -d '{"inputs": "What is AI?", "temperature": 0.4, "top_p": 0.1, "max_tokens": 250}'
```

---

## Batch Inference

### Vertex AI Batch Prediction (Managed)

For offline/bulk inference without a running endpoint:

```python
from google.cloud import aiplatform

aiplatform.init(project="PROJECT_ID", location="us-central1")

model = aiplatform.Model(model_name="MODEL_RESOURCE_NAME")

batch_job = model.batch_predict(
    job_display_name="batch-inference-job",
    gcs_source="gs://your-bucket/input/*.jsonl",
    gcs_destination_prefix="gs://your-bucket/output/",
    instances_format="jsonl",
    machine_type="g2-standard-8",
    accelerator_type="NVIDIA_L4",
    accelerator_count=1,
    max_replica_count=4,
    sync=False,  # async execution
)
```

Input JSONL format:
```jsonl
{"prompt": "Summarize: ...", "max_tokens": 100}
{"inputs": "Classify: ...", "temperature": 0.1}
```

### Ray on Vertex AI (Distributed Batch)

For distributed batch inference with Ray. Requires custom Docker image.

```python
from google.cloud import aiplatform
from ray.job_submission import JobSubmissionClient

# Ray cluster config: head=n1-standard-16 (CPU), workers=a2-highgpu-1g (1x A100)
# Submit batch inference job via Ray Jobs API
client = JobSubmissionClient(address=RAY_DASHBOARD_ADDRESS)
client.submit_job(entrypoint="python batch_inference.py", ...)
```

---

## Resource Cleanup

**Always delete endpoints and models** when done to avoid ongoing charges:

```python
# Delete endpoint (also undeploys all models)
endpoint.delete(force=True)

# Delete model resource
model.delete()
```

```bash
# gcloud cleanup
gcloud ai endpoints delete ENDPOINT_ID --region=us-central1
gcloud ai models delete MODEL_ID --region=us-central1
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `list_deployable_models(filter=...)` | Wrong param — use `model_filter=` (e.g., `model_filter="gemma"`) |
| Forgetting `accept_eula=True` | Required for Llama, Gemma, and other EULA models |
| Wrong region for model | Not all models available in all regions — check Model Garden card |
| Insufficient GPU memory | Use `model.list_deploy_options(concise=True)` to check minimum requirements |
| Not deleting endpoints | Endpoints charge even when idle — always clean up |
| Wrong safetensors format | Custom weights must be in Hugging Face format with `config.json` |
| Timeout on large models | Set `serving_container_deployment_timeout=1800` (30 min) for 70B+ |
| Missing HF_TOKEN for gated models | Set as env var for Llama, some Gemma variants |
| Wrong port/route for Hex-LLM | Uses port 7080, `/generate` (predict), `/ping` (health) — not 8080/`/v1/completions` |
| Wrong port/route for HF PyTorch | Uses port 7080, `/pred` (predict), `/ping` (health) |
| Missing `tpu_topology` for TPU | Required for TPU deployment in `model.deploy()` |
| Qwen3 thinking: short `max_new_tokens` | Thinking content is long — use 8192+ tokens |
| Gemma 3 1B multimodal | 1B variant does NOT support image inputs — use 4B+ |
| Gemma fine-tune: wrong LoRA rank | Axolotl: config-driven; PEFT: rank=16; KerasNLP: rank=4; Ray: rank=32 |
| Gemma fine-tune: disk too small | H100 Axolotl jobs need `boot_disk_size_gb=2000`; A100: 500GB |
| KerasNLP output not HF-compatible | Must convert HDF5 weights via KerasNLP export utility before vLLM |
| Gemma 3n audio: using base64 | Audio input uses `audio_url` with a public URL, not base64-encoded `input_audio` |
| Gemma 3n with vLLM | Gemma 3n uses SGLang, not vLLM — use the SGLang container |
| Gemma 2 TPU outside us-west1 | Gemma 2 TPU (Hex-LLM) is only available in `us-west1` |
| Gemma 2 Hex-LLM timeout too short | Use `deployment_timeout=7200` (2h) for Hex-LLM, not 1800 |
| TPU v7x quota | Needs `CustomModelServing7XTPUPerProjectPerRegion` quota approved |
| GKE: vLLM port vs TGI port | vLLM uses 7080; TGI uses 8080 — use correct port in Service manifest |
| GKE: missing `/dev/shm` volume | Mount emptyDir with `medium: Memory` at `/dev/shm` for shared memory |
| Qwen3 thinking on TPU | Append `<think></think>` to prompt to disable thinking mode |
| Missing service account for GCS models | TPU deployment loading from GCS needs `roles/storage.objectViewer` |
