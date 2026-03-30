---
name: maas-open-models
description: Use when working with Vertex AI Model Garden's Managed Open Models (MaaS) - serverless access without infrastructure. Triggers on: Kimi K2 deploy/use, MiniMax M2 deploy/use, GLM deploy/use, gpt-oss deploy/use, DeepSeek/Llama/Qwen serverless, MaaS API calls, granting IAM access, embedding models MaaS, context caching. NOTE - Kimi/MiniMax/GLM/gpt-oss are MaaS-ONLY and cannot be self-deployed.
---

# Vertex AI Model Garden: MaaS (Managed Open Models)

## Overview

Model-as-a-Service (MaaS) provides **serverless access** to curated open and partner models via Vertex AI. No infrastructure provisioning required — pay per token consumed.

## MaaS-Only Models (No Deployment Required)

The following models are **only available via MaaS** and cannot be self-deployed to a Vertex AI endpoint:

| Provider | MaaS-Only Models |
|----------|------------------|
| Kimi | K2 Thinking |
| MiniMax | M2 |
| ZAI.org | GLM 5, GLM 4.7 |
| OpenAI OSS | gpt-oss-120B, gpt-oss-20B |

> **When a user asks to "deploy" Kimi / MiniMax / GLM / gpt-oss:** No infrastructure setup is needed. There is no endpoint to provision. Simply call the MaaS API directly using the Python SDK — see [Python SDK Usage](#python-sdk-usage) below.

---

## When to Use MaaS vs. Self-Deploy

| Criteria | MaaS | Self-Deploy |
|----------|------|-------------|
| Setup time | Minutes | Hours/days |
| Infrastructure ops | None (Google-managed) | Full responsibility |
| Cost model | Pay-per-token | Fixed GPU/TPU cost |
| Customization | Standard weights only | Custom/fine-tuned weights |
| Traffic pattern | Bursty / variable | Predictable / high-volume |
| Data residency | Multi-tenant | Full VPC isolation |

**Choose MaaS** for prototyping, variable traffic, or when no fine-tuning is needed.
**Choose Self-Deploy** for fine-tuned models, high throughput, or strict isolation.

> **DeepSeek / Llama / Qwen** are available via both MaaS and self-deploy. Use MaaS when traffic is low or bursty and you want to start immediately. Use self-deploy for sustained high-volume traffic or when you need custom weights.

> **Kimi / MiniMax / GLM / gpt-oss** → MaaS only. Use the `maas-open-models` skill (this skill).
> **Gemma / Gemma 3n** → Self-deploy only. Use the `self-deploy-open-models` skill.

## Supported Models

### LLMs
| Provider | Models |
|----------|--------|
| DeepSeek | V3.2, V3.1, R1-0528, OCR |
| Meta | Llama 3.3 (70B), Llama 4 Maverick (17B-128E-MoE), Llama 4 Scout (17B-16E-MoE) |
| Qwen | Qwen3-235B-A22B (instruct), Qwen3-Coder-480B-A35B, Qwen3-Next-80B-A3B (instruct/thinking) |
| Kimi | K2 Thinking |
| MiniMax | M2 |
| ZAI.org | GLM 5, GLM 4.7 |
| OpenAI OSS | gpt-oss-120b, gpt-oss-20b |

### Embedding Models
| Model | Dimensions | Max Tokens |
|-------|-----------|------------|
| multilingual-e5-small-maas | 384 | 512 |
| multilingual-e5-large-instruct-maas | 1024 | 512 |

## Prerequisites

### 1. Enable APIs

```bash
gcloud services enable aiplatform.googleapis.com
```

Also enable the specific model's API via its Model Garden page in the console.

### 2. Grant IAM Access

Two roles are required per user:

```bash
# Role 1: Enable model access in Model Garden
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=user:USER_EMAIL \
    --role=roles/consumerprocurement.entitlementManager

# Role 2: Allow prediction calls
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=user:USER_EMAIL \
    --role=roles/aiplatform.user
```

Supported principal formats: `user:email@example.com`, `group:name@example.com`, `serviceAccount:sa@project.iam.gserviceaccount.com`

> **Note:** Organization policy must allow `cloudcommerceconsumerprocurement.googleapis.com`. Check with your org admin if access is blocked.

## Python SDK Usage

### Install & Initialize

```python
pip install google-genai
```

```python
from google import genai
from google.genai import types

PROJECT_ID = "your-project-id"
LOCATION = "us-central1"       # Check model availability per region
MODEL = "meta/llama-3.3-70b-instruct-maas"  # Llama uses meta/ prefix
# MODEL = "deepseek-v3.2-maas"             # DeepSeek: no prefix
# MODEL = "kimi-k2-thinking-maas"          # Kimi: no prefix

client = genai.Client(
    vertexai=True,
    project=PROJECT_ID,
    location=LOCATION,
)
```

### Basic Inference

```python
response = client.models.generate_content(
    model=MODEL,
    contents="Explain the difference between RAG and fine-tuning.",
    config=types.GenerateContentConfig(
        temperature=0.7,
        max_output_tokens=1024,
    ),
)
print(response.text)
```

### Chat (Multi-turn)

```python
chat = client.chats.create(model=MODEL)

response = chat.send_message("What is quantum entanglement?")
print(response.text)

response = chat.send_message("Give me a real-world analogy.")
print(response.text)
```

### Structured Output / Function Calling

```python
from google.genai import types

tool = types.Tool(
    function_declarations=[
        types.FunctionDeclaration(
            name="get_weather",
            description="Get current weather for a city",
            parameters=types.Schema(
                type="object",
                properties={
                    "city": types.Schema(type="string"),
                    "unit": types.Schema(type="string", enum=["celsius", "fahrenheit"]),
                },
                required=["city"],
            ),
        )
    ]
)

response = client.models.generate_content(
    model=MODEL,
    contents="What's the weather like in Seoul?",
    config=types.GenerateContentConfig(tools=[tool]),
)
```

### Embedding Models

```python
EMBEDDING_MODEL = "multilingual-e5-large-instruct-maas"  # note: -instruct- in name

response = client.models.embed_content(
    model=EMBEDDING_MODEL,
    contents=["Hello world", "안녕하세요"],
)
for embedding in response.embeddings:
    print(embedding.values[:5])  # First 5 dimensions
```

### Streaming

```python
for chunk in client.models.generate_content_stream(
    model=MODEL,
    contents="Write a short story about a robot.",
):
    print(chunk.text, end="", flush=True)
```

## Context Caching (Preview)

Available on select models (Qwen3 Coder, Kimi K2, MiniMax M2, gpt-oss-20b, DeepSeek V3.1/V3.2):

- **90% discount** on cached input tokens
- Minimum **4,096 tokens** required to trigger caching
- **Implicit only** — no manual cache management
- Place repetitive content (system prompts, documents) at the **start** of the prompt to improve cache hit rates
- Not available for Provisioned Throughput or Batch modes

## Model Reference

Full model ID list, context windows, regions, and key features.

### LLMs

| Model ID | Context | Max Out | Region | Special |
|----------|---------|---------|--------|---------|
| `meta/llama-3.3-70b-instruct-maas` | 128K | 8K | us-central1 | Llama Guard |
| `meta/llama-4-maverick-17b-128e-instruct-maas` | 524K | 8K | us-east5 | Multimodal (text+image), no Llama Guard |
| `meta/llama-4-scout-17b-16e-instruct-maas` | 1.3M | 8K | us-east5 | Multimodal (text+image) |
| `deepseek-v3.2-maas` | 163K | 65K | global | Batch, function calling |
| `deepseek-v3.1-maas` | 163K | 32K | us-central1 | Hybrid thinking mode |
| `deepseek-r1-0528-maas` | 163K | 32K | us-central1 | Reasoning model |
| `deepseek-ocr-maas` | 8K | 8K | us-central1 | OCR / document parsing |
| `qwen3-235b-a22b-instruct-2507-maas` | 262K | 16K | global, us-south1 | Hybrid thinking |
| `qwen3-coder-480b-a35b-instruct-maas` | 262K | 65K | global, us-south1 | Code, context caching |
| `qwen3-next-80b-a3b-instruct-maas` | 262K | 262K | global | — |
| `qwen3-next-80b-a3b-thinking-maas` | 262K | 262K | global | Step-by-step reasoning |
| `kimi-k2-thinking-maas` | 262K | 262K | global | Thinking agent, 200-300 tool calls, context caching |
| `minimax-m2-maas` | 196K | 196K | global | Context caching |
| `gpt-oss-120b-maas` | 131K | 131K | global, us-central1 | Context caching |
| `gpt-oss-20b-maas` | 131K | 32K | us-central1 | Batch, context caching |
| `glm-5-maas` | 200K | 128K | global | ⚠️ Experimental (not GA) |
| `glm-4.7-maas` | 200K | 128K | global | — |

### Embedding Models

| Model ID | Dimensions | Max Tokens | Regions |
|----------|-----------|-----------|---------|
| `multilingual-e5-small-maas` | 384 | 512 | us-central1, europe-west4 |
| `multilingual-e5-large-instruct-maas` | 1024 | 512 | us-central1, europe-west4 |

### Feature Support Matrix

| Feature | DeepSeek | Llama | Qwen | Kimi | MiniMax | GLM | gpt-oss |
|---------|----------|-------|------|------|---------|-----|---------|
| Batch | ✅ (V3.x, R1, OCR) | ✅ | ✅ (235B, Coder) | ❌ | ❌ | ❌ | ✅ (20b only) |
| Function Calling | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Structured Output | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Thinking Mode | ✅ (V3.1) | ❌ | ✅ (Next-Thinking, 235B) | ✅ | ❌ | ❌ | ❌ |
| Multimodal (Image) | ❌ | ✅ (4 Maverick, Scout) | ❌ | ❌ | ❌ | ❌ | ❌ |
| Context Caching | ✅ (V3.1, V3.2) | ❌ | ✅ (Coder) | ✅ | ✅ | ❌ | ✅ |
| Provisioned Throughput | ✅ (V3.2) | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |

## Model ID Format

MaaS model IDs end with `-maas`. Llama models use `meta/` prefix in the Python genai SDK; others do not.

```python
# Llama — meta/ prefix (Python genai SDK)
"meta/llama-3.3-70b-instruct-maas"
"meta/llama-4-maverick-17b-128e-instruct-maas"
"meta/llama-4-scout-17b-16e-instruct-maas"

# All others — no prefix
"deepseek-v3.2-maas"
"kimi-k2-thinking-maas"
"qwen3-next-80b-a3b-instruct-maas"
"multilingual-e5-large-instruct-maas"  # note: -instruct- in name
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using wrong LOCATION | Check the model's regional availability on its Model Garden card |
| Missing IAM roles | Both `consumerprocurement.entitlementManager` AND `aiplatform.user` are required |
| Model API not enabled | Enable the model's specific API via the Model Garden console page before calling |
| Wrong model ID prefix | Llama uses `meta/` prefix; DeepSeek, Qwen, Kimi, MiniMax, GLM, gpt-oss have no prefix |
| Using `-maas` suffix for self-deployed | MaaS model IDs end with `-maas`; self-deployed use `@version` format |
| Expecting cached tokens for all models | Context caching only works on supported models and requires ≥4096 tokens |
| Trying to "deploy" Kimi/MiniMax/GLM/gpt-oss | These are MaaS-only — no deployment needed, call the API directly |
