# Model Garden Skills for Gemini CLI

Vertex AI Model Garden의 오픈 모델을 Gemini CLI에서 바로 활용할 수 있도록 만든 Skills 모음입니다.
**MaaS** (서버리스 API)와 **Self-Deployed** (Vertex AI Endpoint·GKE) 두 가지 서빙 방식을 모두 지원합니다.

---

## 목차

1. [사전 요구사항](#사전-요구사항)
2. [설치](#설치)
3. [스킬 목록](#스킬-목록)
4. [사용 방법](#사용-방법)
5. [스킬 상세](#스킬-상세)
6. [서빙 옵션 선택 가이드](#서빙-옵션-선택-가이드)
7. [참고 문서](#참고-문서)

---

## 사전 요구사항

### Gemini CLI

[Gemini CLI](https://github.com/google-gemini/gemini-cli)가 설치되어 있어야 합니다.

### Python 패키지

```bash
# MaaS 사용 시
pip install google-genai

# Self-Deployed 사용 시
pip install "google-cloud-aiplatform>=1.106.0" openai google-auth requests
```

### GCP 인증 및 API 활성화

```bash
# 애플리케이션 기본 인증
gcloud auth application-default login

# 필수 API 활성화
gcloud services enable aiplatform.googleapis.com
gcloud services enable artifactregistry.googleapis.com  # 커스텀 컨테이너 사용 시
```

---

## 설치

### 1단계: 리포지토리 클론

```bash
git clone https://github.com/hajekim/modelgarden-skills.git ~/sandbox/modelgarden-skills
cd ~/sandbox/modelgarden-skills
```

### 2단계: 스킬 디렉토리 생성

Gemini CLI는 `~/.agents/skills/` 디렉토리에서 스킬을 자동으로 불러옵니다.

```bash
mkdir -p ~/.agents/skills
```

### 3단계: 스킬 설치

**심볼릭 링크 방식 (권장 — `git pull` 시 자동 업데이트)**

```bash
ln -s "$(pwd)/skills/maas-open-models" ~/.agents/skills/maas-open-models
ln -s "$(pwd)/skills/self-deploy-open-models" ~/.agents/skills/self-deploy-open-models
```

**복사 방식 (독립적인 설치)**

```bash
cp -r skills/maas-open-models ~/.agents/skills/
cp -r skills/self-deploy-open-models ~/.agents/skills/
```

### 4단계: 설치 확인

```bash
ls ~/.agents/skills/
# maas-open-models/
# self-deploy-open-models/
```

### 업데이트

```bash
cd ~/sandbox/modelgarden-skills && git pull
# 심볼릭 링크 방식을 사용 중이라면 자동으로 반영됩니다.
# 복사 방식을 사용 중이라면 3단계를 다시 실행하십시오.
```

---

## 스킬 목록

| 스킬 | 서빙 방식 | 자동 활성화 키워드 |
|------|-----------|-------------------|
| [`maas-open-models`](#maas-open-models) | 서버리스 API | Kimi K2, MiniMax M2, GLM, gpt-oss, DeepSeek/Llama/Qwen MaaS, 임베딩 MaaS, IAM 접근 권한, 컨텍스트 캐싱 |
| [`self-deploy-open-models`](#self-deploy-open-models) | Vertex AI Endpoint / GKE | Gemma 3/3n/2 배포, vLLM/SGLang/TGI, GPU/TPU, 파인튜닝, GKE 배포, 배치 추론 |

> **MaaS 전용 모델** (Self-Deployed 불가): Kimi K2 Thinking, MiniMax M2, GLM 5/4.7, gpt-oss-120b/20b
>
> **Self-Deployed 전용 모델**: Gemma 3, Gemma 3n, Gemma 2, Mistral, Phi-4
>
> **MaaS·Self-Deployed 모두 가능**: DeepSeek, Llama, Qwen

---

## 사용 방법

관련 키워드가 포함된 질문을 하면 Gemini CLI가 해당 스킬을 자동으로 로드합니다.

### maas-open-models 트리거 예시

```
"Kimi K2 모델을 배포하는 코드를 짜줘"             ← MaaS 전용임을 안내
"DeepSeek R1을 서버리스로 호출하고 싶어"
"MaaS 오픈 모델 접근 권한을 IAM으로 설정해 줘"
"Qwen3에 함수 호출 기능을 사용하는 코드 작성해 줘"
"MaaS 임베딩 모델로 벡터를 생성하는 코드 써줘"
```

### self-deploy-open-models 트리거 예시

```
"Gemma 3 12B를 Vertex AI Endpoint에 배포하고 싶어"
"Gemma 3n으로 오디오 파일을 처리하는 코드 써줘"
"Gemma 2 9B를 TPU v5e로 서빙하는 방법은?"
"vLLM 컨테이너로 DeepSeek V3를 서빙하고 싶어"
"GKE Autopilot 클러스터에 Llama 3.2를 배포하고 싶어"
"Gemma 3를 Axolotl로 파인튜닝하는 방법은?"
"Vertex AI 배치 예측으로 대량 추론하는 방법은?"
```

### 명시적 호출

```
"maas-open-models 스킬을 사용해서 Kimi K2 Thinking 모델 호출 코드를 작성해 줘"
"self-deploy-open-models 스킬 참고해서 Gemma 3 27B를 RTX Pro 6000으로 배포하는 코드 써줘"
```

---

## 스킬 상세

### maas-open-models

**스킬 파일**: `skills/maas-open-models/SKILL.md`

#### 지원 모델

| 제공사 | 모델 ID | 컨텍스트 | 특이사항 |
|--------|---------|----------|----------|
| Meta | `meta/llama-3.3-70b-instruct-maas` | 128K | Llama Guard, us-central1 |
| Meta | `meta/llama-4-maverick-17b-128e-instruct-maas` | 524K | 멀티모달, us-east5 |
| Meta | `meta/llama-4-scout-17b-16e-instruct-maas` | 1.3M | 멀티모달, us-east5 |
| DeepSeek | `deepseek-v3.2-maas` | 163K | Batch, global |
| DeepSeek | `deepseek-v3.1-maas` / `deepseek-r1-0528-maas` | 163K | Thinking |
| DeepSeek | `deepseek-ocr-maas` | 8K | OCR 특화 |
| Qwen | `qwen3-235b-a22b-instruct-2507-maas` | 262K | Hybrid thinking |
| Qwen | `qwen3-coder-480b-a35b-instruct-maas` | 262K | 코딩, 캐싱 |
| Qwen | `qwen3-next-80b-a3b-instruct-maas` / `-thinking-maas` | 262K | — |
| Kimi | `kimi-k2-thinking-maas` | 262K | MaaS 전용, 캐싱 |
| MiniMax | `minimax-m2-maas` | 196K | MaaS 전용, 캐싱 |
| ZAI.org | `glm-5-maas` ⚠️ / `glm-4.7-maas` | 200K | MaaS 전용 |
| OpenAI OSS | `gpt-oss-120b-maas` / `gpt-oss-20b-maas` | 131K | MaaS 전용 |
| 임베딩 | `multilingual-e5-small-maas` | 384d, 512 tokens | — |
| 임베딩 | `multilingual-e5-large-instruct-maas` | 1024d, 512 tokens | `-instruct-` 필수 |

> ⚠️ `glm-5-maas`는 Experimental 단계입니다.

#### IAM 접근 권한 설정

두 개의 역할이 모두 필요합니다.

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=user:USER_EMAIL \
    --role=roles/consumerprocurement.entitlementManager

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=user:USER_EMAIL \
    --role=roles/aiplatform.user
```

#### Python SDK 빠른 시작

```python
from google import genai

client = genai.Client(vertexai=True, project="PROJECT_ID", location="us-central1")

# 단일 추론
response = client.models.generate_content(
    model="meta/llama-4-scout-17b-16e-instruct-maas",
    contents="설명해 줘.",
)
print(response.text)

# 스트리밍
for chunk in client.models.generate_content_stream(
    model="deepseek-v3.2-maas",
    contents="짧은 이야기를 써줘.",
):
    print(chunk.text, end="", flush=True)

# 임베딩 (모델 ID에 -instruct- 포함 여부 주의)
response = client.models.embed_content(
    model="multilingual-e5-large-instruct-maas",
    contents=["Hello world", "안녕하세요"],
)
for embedding in response.embeddings:
    print(embedding.values[:5])
```

#### 컨텍스트 캐싱

- **90% 할인**, 최소 4,096 토큰 필요, 묵시적(Implicit) 방식
- 반복 콘텐츠 (시스템 프롬프트, 문서)를 프롬프트 **앞쪽**에 배치하면 캐시 적중률이 높아집니다.
- 지원 모델: Qwen3 Coder, Kimi K2, MiniMax M2, gpt-oss-20b, DeepSeek V3.1/V3.2

---

### self-deploy-open-models

**스킬 파일**: `skills/self-deploy-open-models/SKILL.md`

#### 배포 옵션 선택 가이드

| 상황 | 권장 옵션 | 특징 |
|------|-----------|------|
| 빠른 시작, 하드웨어 모름 | Option 1: 원클릭 SDK | Python 5줄로 배포 완료, 하드웨어 자동 구성 |
| 서빙 프레임워크 직접 지정 | Option 2: 사전 빌드 컨테이너 | vLLM/TGI/TEI/Hex-LLM 중 선택 가능 |
| 커스텀 로직·비표준 설정 | Option 3: 커스텀 vLLM 컨테이너 | Dockerfile로 완전한 제어, GPU/CPU/TPU |
| 파인튜닝된 자체 모델 보유 | Option 4: 커스텀 웨이트 | GCS 모델을 Vertex AI Endpoint로 직접 배포 |
| 기존 GKE 인프라 활용 | Option 5: GKE 배포 | kubectl/YAML, Autopilot 또는 Standard 선택 |

#### Python SDK 빠른 시작 (Option 1)

```python
import vertexai
from vertexai import model_garden

vertexai.init(project="PROJECT_ID", location="us-central1")

# 배포 가능한 모델 검색 (model_filter= 사용, filter= 아님)
model_garden.list_deployable_models(model_filter="gemma")

# 배포 전 하드웨어 요구사항 확인
model = model_garden.OpenModel("google/gemma3@gemma-3-12b-it")
model.list_deploy_options(concise=True)

# 배포
endpoint = model.deploy(
    accept_eula=True,
    machine_type="g4-standard-48",
    accelerator_type="NVIDIA_RTX_PRO_6000",
    accelerator_count=1,
    use_dedicated_endpoint=True,
)

# 추론 — raw predict
response = endpoint.predict(instances=[{"prompt": "안녕하세요!", "max_tokens": 256}])
print(response.predictions[0])

# 추론 — OpenAI 호환 클라이언트 (dedicated endpoint 전용)
import openai, google.auth, google.auth.transport.requests

creds, _ = google.auth.default()
creds.refresh(google.auth.transport.requests.Request())

client = openai.OpenAI(
    base_url=f"https://us-central1-aiplatform.googleapis.com/v1beta1/{endpoint.resource_name}",
    api_key=creds.token,
)
response = client.chat.completions.create(
    model="",
    messages=[{"role": "user", "content": "안녕하세요!"}],
    max_tokens=256,
)
print(response.choices[0].message.content)

# 리소스 정리 (필수 — idle 상태에서도 과금됨)
endpoint.delete(force=True)
```

#### 서빙 프레임워크 포트·라우트

| 프레임워크 | 포트 | 예측 경로 | 헬스 경로 |
|-----------|------|-----------|-----------|
| vLLM / SGLang / TensorRT-LLM | 8080 | `/v1/completions` | `/health` |
| HF TGI | 8080 | `/generate` | `/health` |
| Hex-LLM | 7080 | `/generate` | `/ping` |
| HF PyTorch Inference | 7080 | `/pred` | `/ping` |
| HF TEI | 8080 | (SDK predict) | — |

#### 하드웨어 레퍼런스

| 머신 타입 | 사양 | 권장 모델 규모 |
|-----------|------|----------------|
| `g2-standard-8/12` | 1x L4 24GB | 3B–7B |
| `g2-standard-48` | 4x L4 96GB | 13B–34B |
| `a2-ultragpu-1g` | 1x A100 80GB | 34B, Gemma 3n |
| `a3-highgpu-8g` | 8x H100 640GB | 70B+ |
| `a3-ultragpu-8g` | 8x H200 1TB | 70B+ FP8 |
| `g4-standard-48` | 1x RTX Pro 6000 96GB | Gemma 3 (vLLM) |
| `c2-standard-16` | 16 vCPU | CPU vLLM |
| `ct5lp-hightpu-4t` | v5e 4코어 | 8B Hex-LLM (tpu_topology="1x1") |
| `ct6e-standard-1t` | v6e 8코어 | Llama 3.1 8B, Qwen3 4B/8B |
| `tpu7x-standard-4t` | v7x | Llama 3.3 70B (tensor_parallel=8) |

#### Gemma 모델 ID

```python
# Gemma 3 (vLLM, g4-standard-48 + NVIDIA_RTX_PRO_6000)
"google/gemma3@gemma-3-270m-it"   # 멀티모달 불가
"google/gemma3@gemma-3-1b-it"     # 멀티모달 불가
"google/gemma3@gemma-3-4b-it"     # 멀티모달 (text+image)
"google/gemma3@gemma-3-12b-it"    # 멀티모달
"google/gemma3@gemma-3-27b-it"    # 멀티모달

# Gemma 3n (SGLang 전용, A100 80GB, text+image+audio)
"google/gemma3n@gemma-3n-e2b-it"
"google/gemma3n@gemma-3n-e4b-it"
# 오디오 입력: audio_url 형식 사용 (base64 아님)

# Gemma 2 (text-only — GPU TGI 또는 TPU Hex-LLM)
"google/gemma-2-2b-it"    # GPU: g2-standard-12 / TPU: ct5lp-hightpu-1t
"google/gemma-2-9b-it"    # GPU: g2-standard-24 / TPU: ct5lp-hightpu-4t
"google/gemma-2-27b-it"   # GPU: g2-standard-48 / TPU: ct5lp-hightpu-8t
# TPU Hex-LLM은 us-west1 리전 전용, deployment_timeout=7200s
```

#### 파인튜닝

| 프레임워크 | 대상 모델 | 학습 방법 | GPU | LoRA Rank |
|-----------|----------|----------|-----|-----------|
| Axolotl | Gemma 3 (1B–27B) | LoRA/QLoRA | 8x H100 | config |
| HF PEFT | Gemma 1/2 | LoRA 4-bit | 8x A100/H100 | 16 |
| KerasNLP | Gemma 1 (2B/7B) | LoRA | 1x L4 | 4 |
| Ray + TRL | Gemma 1 2B | QLoRA | 1x A100 | 32 |

---

## 서빙 옵션 선택 가이드

### MaaS vs Self-Deployed

| 기준 | MaaS | Self-Deployed |
|------|------|-----------|
| **설정 시간** | 분 단위 | 수 시간 ~ 수 일 |
| **인프라 운영** | 없음 (Google 관리) | 직접 관리 |
| **비용 구조** | 토큰당 과금 | 고정 GPU/TPU 비용 |
| **커스터마이징** | 표준 웨이트만 가능 | 파인튜닝/커스텀 웨이트 가능 |
| **트래픽 패턴** | 불규칙/간헐적 | 예측 가능/대용량 |
| **데이터 격리** | 멀티테넌트 | 완전한 VPC 격리 |

**MaaS를 선택하세요** → 빠른 프로토타이핑, 트래픽이 불규칙하거나, 파인튜닝이 필요 없는 경우

**Self-Deployed를 선택하세요** → 파인튜닝 모델 사용, 대용량 안정적 트래픽, VPC 완전 격리가 필요한 경우

### Vertex AI Endpoint vs GKE

| 기준 | Vertex AI Endpoint | GKE |
|------|---------------------|-----|
| **설정 복잡도** | 낮음 (Python SDK) | 높음 (kubectl/YAML) |
| **자동 스케일링** | 내장 | HPA 직접 설정 |
| **멀티 모델** | 엔드포인트당 여러 모델 | Pod 단위 관리 |
| **기존 인프라** | GCP 신규 프로젝트에 적합 | 기존 GKE 클러스터 재활용 |
| **모니터링** | Vertex AI 대시보드 | Cloud Monitoring + 커스텀 |

---

## 참고 문서

### 공식 문서

- [Vertex AI MaaS 개요](https://cloud.google.com/vertex-ai/generative-ai/docs/maas/use-open-models)
- [오픈 모델 접근 권한 부여](https://cloud.google.com/vertex-ai/generative-ai/docs/maas/grant-access-open-models)
- [서빙 옵션 선택 가이드](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/choose-serving-option)
- [Self-Deployed 모델 개요](https://cloud.google.com/vertex-ai/generative-ai/docs/model-garden/self-deployed-models)
- [사전 빌드 컨테이너 사용](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/use-prebuilt-containers)
- [커스텀 웨이트로 모델 배포](https://cloud.google.com/vertex-ai/generative-ai/docs/model-garden/deploy-models-with-custom-weights)

### 배포 튜토리얼 노트북

- [Model Garden 배포 튜토리얼](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_deployment_tutorial.ipynb)
- [Gemma 3 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma3_deployment_on_vertex.ipynb)
- [Gemma 3n 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma3n_deployment_on_vertex.ipynb)
- [Gemma 2 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma2_deployment_on_vertex.ipynb)
- [DeepSeek 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_deepseek_deployment.ipynb)
- [Qwen3 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_qwen3_deployment.ipynb)
- [vLLM GPU 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_gpu_llama3_2_3B.ipynb)
- [vLLM CPU 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_cpu_llama3_2_3B.ipynb)
- [vLLM TPU 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_tpu_llama3_2_3B.ipynb)
- [HF TGI 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_huggingface_tgi_deployment.ipynb)
- [HF TEI 임베딩 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_huggingface_tei_deployment.ipynb)
- [Hex-LLM TPU 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_hexllm_deep_dive_tutorial.ipynb)
- [커스텀 웨이트 임포트](https://github.com/GoogleCloudPlatform/generative-ai/blob/main/open-models/get_started_with_model_garden_sdk_custom_import.ipynb)
- [TPU v5e 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_1_qwen2_5_deployment_tpu.ipynb)
- [TPU v6e 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_1_qwen3_deployment_tpu.ipynb)
- [TPU v7x 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_3_tpu7x_deployment.ipynb)

### GKE 배포 노트북

- [Gemma on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_deployment_on_gke.ipynb)
- [Llama 3.2 on GKE (vLLM)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_llama3_2_deployment_on_gke.ipynb)
- [Llama 3.1 on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_tgi_llama3_1_deployment_on_gke.ipynb)
- [Mistral on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_tgi_mistral_deployment_on_gke.ipynb)

### 파인튜닝 노트북

- [Gemma 3 파인튜닝 (Axolotl)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_axolotl_gemma3_finetuning.ipynb)
- [Gemma 1 파인튜닝 (HF PEFT)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_finetuning_on_vertex.ipynb)
- [Gemma 2 파인튜닝 (HF PEFT)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma2_finetuning_on_vertex.ipynb)
- [Gemma 1 파인튜닝 (KerasNLP)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_kerasnlp_to_vertexai.ipynb)
- [Gemma 파인튜닝 + 배치 추론 (Ray)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_fine_tuning_batch_deployment_on_rov.ipynb)
