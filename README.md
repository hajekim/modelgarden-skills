# Model Garden Skills for Gemini CLI

Vertex AI Model Garden의 오픈 모델 서빙 두 가지 방식—**MaaS(관리형 오픈 모델)**와 **자체 배포(Self-Deploy)**—을 Gemini CLI에서 바로 활용할 수 있도록 만든 Skills 모음입니다.

---

## 목차

1. [이 스킬의 특징](#이-스킬의-특징)
2. [설치 방법](#설치-방법)
3. [사용 방법](#사용-방법)
4. [스킬 상세 설명](#스킬-상세-설명)
5. [서빙 옵션 선택 가이드](#서빙-옵션-선택-가이드)
6. [참고 문서](#참고-문서)

---

## 이 스킬의 특징

### 1. 두 가지 서빙 방식을 모두 커버

| 스킬 | 대상 | 핵심 내용 |
|------|------|-----------|
| `maas-open-models` | 서버리스 API 호출 | IAM 설정, Python SDK 5가지 패턴, 17개 모델 전체 ID·컨텍스트·리전, 컨텍스트 캐싱 |
| `self-deploy-open-models` | 전용 엔드포인트 + GKE 배포 | 5가지 배포 옵션, Gemma 전 패밀리, GPU/CPU/TPU v5e~v7x, 7가지 서빙 프레임워크, 파인튜닝, 배치 추론 |

### 2. Python SDK + gcloud CLI 이중 지원

모든 주요 작업에 대해 **Python SDK 코드**와 **gcloud 명령어**를 함께 제공하여, Jupyter Notebook 환경과 터미널 환경 모두에서 즉시 활용하실 수 있습니다.

### 3. 공식 튜토리얼 노트북 기반 검증된 코드

25개 이상의 공식 튜토리얼 노트북과 Vertex AI Python SDK 소스를 직접 대조 검증한 코드만 포함합니다. 컨테이너 이미지 URI, 환경변수, 포트, 라우트, SDK 파라미터명 등 세부 설정값이 모두 검증된 값입니다.

### 4. 프레임워크별 포트·라우트 차이 명시

서빙 프레임워크마다 포트와 예측 경로가 다릅니다. 이 차이를 명확히 정리하여 흔한 설정 오류를 방지합니다.

| 프레임워크 | 포트 | 예측 경로 | 헬스 경로 |
|-----------|------|-----------|-----------|
| vLLM / SGLang / TensorRT-LLM | 8080 | `/v1/completions` | `/health` |
| HF TGI | 8080 | `/generate` | `/health` |
| Hex-LLM | 7080 | `/generate` | `/ping` |
| HF PyTorch Inference | 7080 | `/pred` | `/ping` |
| HF TEI | 8080 | (SDK predict) | — |

### 5. GPU / CPU / TPU 하드웨어 레퍼런스 내장

L4부터 H200까지의 GPU, CPU 최적화 머신, TPU v5e / v6e / v7x 머신 타입과 모델 크기별 권장 스펙을 한눈에 확인하실 수 있습니다.

### 6. Gemma 전 패밀리 커버

Gemma 3(270m~27b), Gemma 3n(오디오 입력), Gemma 2(GPU TGI / TPU Hex-LLM 이중 경로)의 정확한 모델 ID, 서빙 프레임워크, 하드웨어 요구사항을 모두 포함합니다.

### 7. Gemma 파인튜닝 4가지 프레임워크

Axolotl(Gemma 3), HF PEFT(Gemma 1/2), KerasNLP(Gemma 1), Ray on Vertex(분산) 각각의 컨테이너 이미지, LoRA 하이퍼파라미터, 데이터셋 형식, GCS 출력 경로를 포함합니다.

### 8. GKE 배포 + 배치 추론

Vertex AI 엔드포인트 외에 GKE Autopilot/Standard 클러스터 배포와 Vertex AI 관리형 배치 예측을 지원합니다.

### 9. 흔한 실수 사전 방지

각 스킬 말미에 **Common Mistakes** 표를 포함하여, IAM 역할 누락, 잘못된 OpenAI URL, `model_filter` 파라미터명 혼동, Gemma 3n 오디오 형식 오류, TPU `tpu_topology` 누락 등 실제로 자주 겪는 문제를 미리 안내합니다.

---

## 설치 방법

### 사전 요건

- [Gemini CLI](https://github.com/google-gemini/gemini-cli)가 설치되어 있어야 합니다.
- Git이 설치되어 있어야 합니다.

### 1단계: 리포지토리 클론

```bash
git clone https://github.com/YOUR_USERNAME/modelgarden-skills.git
cd modelgarden-skills
```

### 2단계: Gemini CLI 스킬 디렉토리 확인

Gemini CLI는 `~/.agents/skills/` 디렉토리에서 스킬을 자동으로 불러옵니다. 디렉토리가 없다면 생성합니다.

```bash
mkdir -p ~/.agents/skills
```

### 3단계: 스킬 설치

**방법 A — 복사 (권장, 독립적인 설치)**

```bash
cp -r skills/maas-open-models ~/.agents/skills/
cp -r skills/self-deploy-open-models ~/.agents/skills/
```

**방법 B — 심볼릭 링크 (리포지토리와 동기화 유지)**

```bash
ln -s $(pwd)/skills/maas-open-models ~/.agents/skills/maas-open-models
ln -s $(pwd)/skills/self-deploy-open-models ~/.agents/skills/self-deploy-open-models
```

심볼릭 링크 방식을 사용하시면 `git pull`로 스킬을 업데이트했을 때 자동으로 반영됩니다.

### 4단계: 설치 확인

```bash
ls ~/.agents/skills/
# 출력 예시:
# maas-open-models/
# self-deploy-open-models/
```

### 업데이트

```bash
cd modelgarden-skills
git pull
# 심볼릭 링크를 사용 중이라면 자동 반영됩니다.
# 복사 방식을 사용 중이라면 3단계를 다시 실행하십시오.
```

---

## 사용 방법

스킬이 설치되면 Gemini CLI와 대화할 때 관련 주제가 나올 경우 자동으로 스킬이 로드됩니다. 명시적으로 호출하실 수도 있습니다.

### 자동 트리거 (권장)

아래와 같이 자연어로 질문하시면 Gemini가 스킬을 자동으로 인식하고 활용합니다.

```
# maas-open-models 스킬이 자동 로드되는 질문 예시
"Vertex AI MaaS로 Llama 4 Maverick을 사용하는 방법을 알려줘"
"Kimi K2 모델을 배포하는 코드를 짜줘"          ← MaaS 전용임을 안내
"DeepSeek R1을 서버리스로 호출하고 싶어"
"MaaS 오픈 모델 접근 권한을 IAM으로 설정해 줘"
"Qwen3에 함수 호출 기능을 사용하는 코드 작성해 줘"
"MaaS 임베딩 모델로 벡터를 생성하는 코드 써줘"
"gpt-oss-120b 모델 사용법 알려줘"

# self-deploy-open-models 스킬이 자동 로드되는 질문 예시
"Gemma 3 12B를 Vertex AI 엔드포인트에 배포하고 싶어"
"Gemma 3n으로 오디오 파일을 처리하는 코드 써줘"
"Gemma 2 9B를 TPU v5e로 서빙하는 방법은?"
"파인튜닝한 Llama 모델을 GCS에서 가져와서 배포하는 방법은?"
"vLLM 컨테이너로 DeepSeek V3를 서빙하고 싶어"
"H100 GPU 8장으로 Llama 3.3 70B를 배포하는 코드 써줘"
"TPU v7x로 Llama 3.3 70B 서빙하는 방법 알려줘"
"Gemma 3를 Axolotl로 파인튜닝하는 방법은?"
"HF PEFT로 Gemma 2를 LoRA 파인튜닝하고 싶어"
"GKE Autopilot 클러스터에 Llama 3.2를 vLLM으로 배포하고 싶어"
"Vertex AI 배치 예측으로 대량 추론하는 방법은?"
"GPU 없이 CPU로 vLLM 배포하는 방법이 있어?"
```

### 명시적 호출

스킬을 직접 지정하여 호출하실 수도 있습니다.

```
"maas-open-models 스킬을 사용해서 Kimi K2 Thinking 모델 호출 코드를 작성해 줘"
"self-deploy-open-models 스킬 참고해서 Gemma 3 27B를 RTX Pro 6000으로 배포하는 코드 써줘"
```

### 실제 대화 예시

**예시 1: MaaS로 Llama 4 Scout 스트리밍 호출**

```
사용자: Llama 4 Scout 모델로 스트리밍 응답을 받는 Python 코드 짜줘.

Gemini: [maas-open-models 스킬 로드]

from google import genai

client = genai.Client(
    vertexai=True,
    project="your-project-id",
    location="us-east5",   # Llama 4 Scout는 us-east5 리전
)

for chunk in client.models.generate_content_stream(
    model="meta/llama-4-scout-17b-16e-instruct-maas",
    contents="한국의 주요 관광지를 소개해 줘.",
):
    print(chunk.text, end="", flush=True)
```

**예시 2: Gemma 3 원클릭 배포 후 추론**

```
사용자: Gemma 3 12B 모델을 배포하고 추론하는 전체 코드를 작성해 줘.

Gemini: [self-deploy-open-models 스킬 로드]

import vertexai
from vertexai import model_garden

vertexai.init(project="your-project-id", location="us-central1")

model = model_garden.OpenModel("google/gemma3@gemma-3-12b-it")
model.list_deploy_options(concise=True)  # 배포 전 요구사항 확인

endpoint = model.deploy(
    accept_eula=True,
    machine_type="g4-standard-48",
    accelerator_type="NVIDIA_RTX_PRO_6000",
    accelerator_count=1,
    use_dedicated_endpoint=True,
)

response = endpoint.predict(
    instances=[{"prompt": "서울의 역사를 설명해 줘.", "max_tokens": 512}]
)
print(response.predictions[0])

endpoint.delete(force=True)  # 사용 후 반드시 삭제 (과금 방지)
```

**예시 3: Gemma 3n 오디오 입력 처리**

```
사용자: Gemma 3n으로 오디오 파일의 내용을 텍스트로 변환하는 코드 써줘.

Gemini: [self-deploy-open-models 스킬 로드]

# Gemma 3n은 SGLang으로 배포 (vLLM 불가), A100 80GB 필요
model = model_garden.OpenModel("google/gemma3n@gemma-3n-e4b-it")
endpoint = model.deploy(accept_eula=True, use_dedicated_endpoint=True)

# 오디오 입력: audio_url 형식 사용 (base64 아님)
import openai, google.auth, google.auth.transport.requests

creds, _ = google.auth.default()
creds.refresh(google.auth.transport.requests.Request())
BASE_URL = f"https://us-central1-aiplatform.googleapis.com/v1beta1/{endpoint.resource_name}"

client = openai.OpenAI(base_url=BASE_URL, api_key=creds.token)
response = client.chat.completions.create(
    model="",
    messages=[{
        "role": "user",
        "content": [
            {"type": "audio_url", "audio_url": {"url": "https://example.com/speech.wav"}},
            {"type": "text", "text": "이 오디오를 한국어로 요약해 줘."},
        ],
    }],
)
print(response.choices[0].message.content)
```

**예시 4: Gemma 3를 Axolotl로 파인튜닝**

```
사용자: Gemma 3 4B 모델을 QLoRA로 파인튜닝하는 Vertex AI 학습 잡 코드 써줘.

Gemini: [self-deploy-open-models 스킬 로드]

from google.cloud import aiplatform

aiplatform.init(project="your-project-id", location="us-central1")

TRAIN_IMAGE = (
    "us-docker.pkg.dev/vertex-ai/vertex-vision-model-garden-dockers/"
    "axolotl-train-dws:20250812-1800-rc1"
)

job = aiplatform.CustomContainerTrainingJob(
    display_name="gemma3-4b-qlora-finetune",
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
    boot_disk_size_gb=2000,
)
```

**예시 5: GKE Autopilot에 Llama 3.2 배포**

```
사용자: GKE Autopilot 클러스터에 Llama 3.2 3B를 vLLM으로 배포하는 방법을 알려줘.

Gemini: [self-deploy-open-models 스킬 로드]

# 1. Autopilot 클러스터 생성
gcloud container clusters create-auto llama-cluster \
    --location=us-central1 \
    --project=your-project-id

# 2. HF 토큰 시크릿 생성
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=YOUR_HF_TOKEN

# 3. Kubernetes YAML 배포 (vLLM, port 7080 → Service port 8000)
kubectl apply -f llama32-deployment.yaml
kubectl apply -f llama32-service.yaml

# 추론 (포트 포워딩 후)
curl -X POST http://localhost:7080/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is AI?", "max_tokens": 100, "temperature": 0.9}'
```

**예시 6: IAM 권한 설정**

```
사용자: 팀원이 MaaS 오픈 모델을 사용할 수 있도록 권한을 부여하는 gcloud 명령어를 알려줘.

Gemini: [maas-open-models 스킬 로드]

두 개의 IAM 역할이 모두 필요합니다.

# Model Garden에서 모델 활성화 권한
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member=user:team-member@example.com \
    --role=roles/consumerprocurement.entitlementManager

# Vertex AI 예측 API 호출 권한
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member=user:team-member@example.com \
    --role=roles/aiplatform.user
```

---

## 스킬 상세 설명

### `maas-open-models` — MaaS (관리형 오픈 모델)

**스킬 파일**: `skills/maas-open-models/SKILL.md`

**자동 활성화 조건**: 서버리스 LLM 추론, Kimi/MiniMax/GLM/gpt-oss 배포 문의, pay-per-use 오픈 모델, MaaS API 호출, IAM 접근 권한 부여, 임베딩 모델 MaaS, 컨텍스트 캐싱

#### 포함 내용

**① MaaS 전용 모델 안내**

아래 모델은 MaaS로만 사용할 수 있으며, 별도 인프라 배포가 필요 없습니다.

| 제공사 | MaaS 전용 모델 |
|--------|---------------|
| Kimi | K2 Thinking |
| MiniMax | M2 |
| ZAI.org | GLM 5 ⚠️, GLM 4.7 |
| OpenAI OSS | gpt-oss-120b, gpt-oss-20b |

> ⚠️ `glm-5-maas`는 Experimental 단계입니다.

**② 지원 모델 전체 목록 (검증된 모델 ID)**

| 제공사 | 모델 ID | 컨텍스트 | 특이사항 |
|--------|---------|---------|----------|
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
| ZAI.org | `glm-5-maas` / `glm-4.7-maas` | 200K | MaaS 전용 |
| OpenAI OSS | `gpt-oss-120b-maas` / `gpt-oss-20b-maas` | 131K | MaaS 전용 |
| 임베딩 | `multilingual-e5-small-maas` | 384d, 512 tokens | — |
| 임베딩 | `multilingual-e5-large-instruct-maas` | 1024d, 512 tokens | `-instruct-` 주의 |

**③ IAM 접근 권한 설정** — gcloud 명령어 포함
- `consumerprocurement.entitlementManager` 역할
- `aiplatform.user` 역할
- 조직 정책 주의사항

**④ Python SDK 사용 패턴** (`google-genai` 라이브러리)
- 단일 추론 (`generate_content`)
- 멀티턴 채팅 (`chats.create` + `send_message`)
- 함수 호출 / 구조화 출력 (`Tool` + `FunctionDeclaration`)
- 임베딩 (`embed_content`, `response.embeddings[i].values`)
- 스트리밍 (`generate_content_stream`)

**⑤ 컨텍스트 캐싱 (Preview)**
- 캐시된 토큰 **90% 할인**, 최소 4,096 토큰 필요, 묵시적(Implicit) 방식
- 지원 모델: Qwen3 Coder, Kimi K2, MiniMax M2, gpt-oss-20b, DeepSeek V3.1/V3.2

**⑥ 모델 참조 표** — 전체 컨텍스트 윈도우, 최대 출력, 리전, Feature 지원 매트릭스 (Batch / Function Calling / Thinking / Multimodal / Context Caching / Provisioned Throughput)

---

### `self-deploy-open-models` — 자체 배포 오픈 모델

**스킬 파일**: `skills/self-deploy-open-models/SKILL.md`

**자동 활성화 조건**: Llama/Gemma/Qwen/DeepSeek를 Vertex AI 또는 GKE에 배포, Gemma 3/3n/2 배포, 파인튜닝 모델 서빙, vLLM/SGLang/TGI/Hex-LLM 컨테이너, GPU/TPU/CPU 엔드포인트, GCS 커스텀 웨이트, 멀티모달 비전·오디오 모델, 파인튜닝(Axolotl/PEFT/LoRA/QLoRA), GKE 배포, 배치 추론

#### 포함 내용

**① 5가지 배포 옵션**

```
Option 1: OpenModel().deploy()         ← 가장 빠른 시작, list_deploy_options(concise=True) 포함
Option 2: 사전 빌드 컨테이너           ← vLLM, TGI, TEI, HF PyTorch, Hex-LLM 예제 포함
Option 3: 커스텀 vLLM 컨테이너        ← GPU / CPU / TPU 3가지 변형 전체 포함
Option 4: 커스텀 웨이트               ← GCS 모델 → CustomModel().deploy() + gcloud CLI
Option 5: GKE 배포                    ← Autopilot/Standard 클러스터, kubectl/YAML, vLLM/TGI
```

**② Option 1 — 원클릭 SDK 배포**
- `list_deployable_models(model_filter="gemma")` 모델 검색 (`filter=` 아님 — TypeError 발생)
- `list_deploy_options(concise=True)` 배포 전 하드웨어 요구사항 확인
- `OpenModel(MODEL_ID).deploy(accept_eula=True)` 기본/하드웨어 지정 배포
- `endpoint.resource_name` 기반 올바른 OpenAI 호환 URL 형식

**③ Option 2 — 사전 빌드 컨테이너 (7가지 프레임워크)**

| 프레임워크 | 적합한 사용 사례 | 하드웨어 | 예제 포함 |
|-----------|-----------------|----------|----------|
| vLLM | 범용 GPU 서빙, OpenAI 호환 | GPU | ✓ |
| SGLang | DeepSeek, Qwen3, Gemma 3n, 고처리량 | GPU | — (모델별 노트 참조) |
| TensorRT-LLM | NVIDIA 최적화 | H100/H200 | — |
| Hex-LLM | TPU XLA 최적화 서빙 | TPU v5e | ✓ |
| HF TGI | HuggingFace 생성 모델 | GPU | ✓ |
| HF TEI | 임베딩 모델 | GPU/CPU | ✓ |
| HF PyTorch Inference | 분류 등 NLP 태스크 | GPU | ✓ |

**④ Option 3 — 커스텀 vLLM 컨테이너 (3가지 변형)**
- **GPU**: `vllm/vllm-openai` 베이스, `NVIDIA_L4` ~ `NVIDIA_H200`, Dockerfile + Cloud Build 전체 흐름
- **CPU**: `c2-standard-16`, 가속기 없음, `autoscaling_target_cpu_utilization`
- **TPU**: `vllm/vllm-tpu:nightly` 베이스, `ct5lp-hightpu-1t`, GCS 모델 경로 지원

**⑤ Option 4 — 커스텀 웨이트 배포 (Preview)**
- 지원 베이스 모델: Llama, Gemma, Qwen, DeepSeek, Mistral, Phi-4
- GCS 필수 파일 목록 (config.json, safetensors, tokenizer 파일)
- `CustomModel(gcs_uri=...).deploy()` Python SDK + gcloud CLI

**⑥ Option 5 — GKE 배포**
- Autopilot (`create-auto`) / Standard (`create` + GPU node pool) 클러스터 생성
- HF 토큰 Kubernetes Secret 설정
- vLLM YAML 구조 (Llama 3.2 기준, port 7080 → Service 8000)
- TGI YAML 핵심 설정 (port 8080, `/generate`)
- 모델 크기별 리소스 할당표 (CPU/Memory/Ephemeral/GPU)
- `/dev/shm` Memory 볼륨 마운트 필수

**⑦ 하드웨어 레퍼런스**

| 유형 | 머신 타입 | 사양 | 권장 용도 |
|------|-----------|------|-----------|
| GPU | `g2-standard-8/12` | 1x L4 24GB | 3B–7B 모델 |
| GPU | `g2-standard-48` | 4x L4 96GB | 13B–34B 모델 |
| GPU | `a2-ultragpu-1g` | 1x A100 80GB | 34B, Llama 3.2 11B Vision, Gemma 3n |
| GPU | `a3-highgpu-8g` | 8x H100 640GB | 70B+, Llama 3.2 90B Vision |
| GPU | `a3-ultragpu-8g` | 8x H200 1TB | 70B+ FP8, DeepSeek 단일 호스트 |
| GPU | `g4-standard-48` | 1x RTX Pro 6000 96GB | Gemma 3 (vLLM) |
| CPU | `c2-standard-16` | 16 vCPU | CPU vLLM |
| TPU | `ct5lp-hightpu-1t` | v5e 1코어 | 1B–3B (vLLM/Hex-LLM) |
| TPU | `ct5lp-hightpu-4t` | v5e 4코어 | 8B (Hex-LLM, tpu_topology="1x1") |
| TPU | `ct5lp-hightpu-8t` | v5e 8코어 | 27B Gemma 2 (Hex-LLM) |
| TPU | `ct6e-standard-1t` | v6e 8코어 | Llama 3.1 8B, Qwen3 4B/8B |
| TPU | `ct6e-standard-4t` | v6e 32코어 | Qwen3 32B |
| TPU | `tpu7x-standard-4t` | v7x | Llama 3.3 70B (tensor_parallel=8) |

**⑧ 모델별 특이사항 섹션**
- **DeepSeek-V3/R1**: SGLang(권장) / vLLM / TensorRT-LLM 3가지 옵션 비교
- **Qwen3 Thinking Mode**: 권장 파라미터 (temperature 0.6, top_p 0.95, max_new_tokens 8192+)
- **멀티모달**: 이미지(Llama 3.2 Vision, Gemma 3) + 오디오(Gemma 3n, `audio_url` 형식)
- **Gemma 3**: 전체 모델 ID (270m~27b, it/pt 변형), vLLM + RTX Pro 6000, 1B는 멀티모달 불가
- **Gemma 3n**: SGLang 전용(vLLM 불가), 오디오 지원, A100 80GB
- **Gemma 2**: GPU TGI(L4) / TPU Hex-LLM(us-west1 전용, timeout=7200s) 이중 경로

**⑨ Gemma 파인튜닝 (4가지 프레임워크)**

| 프레임워크 | 대상 모델 | 학습 방법 | 학습 GPU | LoRA Rank |
|-----------|----------|----------|---------|-----------|
| Axolotl | Gemma 3 (1B–27B) | LoRA / QLoRA | 8x A100/H100 | config |
| HF PEFT | Gemma 1/2 | LoRA 4-bit | 8x A100/H100 | 16 |
| KerasNLP | Gemma 1 (2B/7B) | LoRA | 1x L4 | 4 |
| Ray + TRL | Gemma 1 2B | QLoRA | 1x A100 | 32 |

컨테이너 이미지 URI(검증 완료), 하이퍼파라미터, JSONL 데이터셋 형식, GCS 출력 경로 포함

**⑩ 배치 추론**
- Vertex AI 관리형 배치 예측 (`model.batch_predict()`) — JSONL 입력, GCS 출력
- Ray on Vertex AI — 분산 배치 처리, ROUGE 평가 메트릭 포함

---

## 서빙 옵션 선택 가이드

### MaaS vs 자체 배포

| 기준 | MaaS | 자체 배포 |
|------|------|-----------|
| **설정 시간** | 분 단위 | 수 시간 ~ 수 일 |
| **인프라 운영** | 없음 (Google 관리) | 직접 관리 |
| **비용 구조** | 토큰당 과금 | 고정 GPU/TPU 비용 |
| **커스터마이징** | 표준 웨이트만 가능 | 파인튜닝/커스텀 웨이트 가능 |
| **트래픽 패턴** | 불규칙/간헐적 | 예측 가능/대용량 |
| **데이터 격리** | 멀티테넌트 | 완전한 VPC 격리 |

**MaaS를 선택하세요** → 빠른 프로토타이핑, 트래픽이 불규칙하거나, 파인튜닝이 필요 없는 경우

**자체 배포를 선택하세요** → 파인튜닝 모델 사용, 대용량 안정적 트래픽, VPC 완전 격리가 필요한 경우

### Vertex AI 엔드포인트 vs GKE

| 기준 | Vertex AI 엔드포인트 | GKE |
|------|---------------------|-----|
| **설정 복잡도** | 낮음 (Python SDK) | 높음 (kubectl/YAML) |
| **자동 스케일링** | 내장 | HPA 직접 설정 |
| **멀티 모델** | 엔드포인트당 여러 모델 | Pod 단위 관리 |
| **기존 인프라** | GCP 신규 프로젝트에 적합 | 기존 GKE 클러스터 재활용 |
| **모니터링** | Vertex AI 대시보드 | Cloud Monitoring + 커스텀 |

---

## 참고 문서

### MaaS — 공식 문서

- [Vertex AI MaaS 개요](https://cloud.google.com/vertex-ai/generative-ai/docs/maas/use-open-models)
- [MaaS API 호출 방법](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/use-maas)
- [오픈 모델 접근 권한 부여](https://cloud.google.com/vertex-ai/generative-ai/docs/maas/grant-access-open-models)
- [서빙 옵션 선택 가이드](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/choose-serving-option)

### 자체 배포 — 공식 문서

- [자체 배포 모델 개요](https://cloud.google.com/vertex-ai/generative-ai/docs/model-garden/self-deployed-models)
- [Model Garden에서 모델 배포](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/deploy-model-garden)
- [사전 빌드 컨테이너 사용](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/use-prebuilt-containers)
- [커스텀 vLLM 컨테이너 배포](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/deploy-custom-vllm)
- [커스텀 웨이트로 모델 배포](https://cloud.google.com/vertex-ai/generative-ai/docs/model-garden/deploy-models-with-custom-weights)

### 자체 배포 — 배포 튜토리얼 노트북

- [Model Garden 배포 튜토리얼](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_deployment_tutorial.ipynb)
- [Gemma 3 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma3_deployment_on_vertex.ipynb)
- [Gemma 3n 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma3n_deployment_on_vertex.ipynb)
- [Gemma 2 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma2_deployment_on_vertex.ipynb)
- [DeepSeek 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_deepseek_deployment.ipynb)
- [Qwen3 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_qwen3_deployment.ipynb)
- [vLLM GPU 배포 (Llama 3.2)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_gpu_llama3_2_3B.ipynb)
- [vLLM CPU 배포 (Llama 3.2)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_cpu_llama3_2_3B.ipynb)
- [vLLM TPU 배포 (Llama 3.2)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_tpu_llama3_2_3B.ipynb)
- [vLLM TPU + GCS 모델](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/official/prediction/vertexai_serving_vllm/vertexai_serving_vllm_tpu_gcs_llama3_2_3B.ipynb)
- [vLLM 멀티모달](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_vllm_multimodal_tutorial.ipynb)
- [HF TGI 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_huggingface_tgi_deployment.ipynb)
- [HF TEI 임베딩 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_huggingface_tei_deployment.ipynb)
- [HF PyTorch Inference 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_huggingface_pytorch_inference_deployment.ipynb)
- [Hex-LLM TPU 배포](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_hexllm_deep_dive_tutorial.ipynb)
- [커스텀 웨이트 임포트](https://github.com/GoogleCloudPlatform/generative-ai/blob/main/open-models/get_started_with_model_garden_sdk_custom_import.ipynb)
- [TPU v5e 배포 (Llama 3.1 / Qwen2.5)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_1_qwen2_5_deployment_tpu.ipynb)
- [TPU v6e 배포 (Llama 3.1 / Qwen3)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_1_qwen3_deployment_tpu.ipynb)
- [TPU v7x 배포 (Llama 3.3 70B)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_pytorch_llama3_3_tpu7x_deployment.ipynb)

### GKE 배포 — 튜토리얼 노트북

- [Gemma on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_deployment_on_gke.ipynb)
- [Llama 3.2 on GKE (vLLM)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_llama3_2_deployment_on_gke.ipynb)
- [Llama 3.1 on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_tgi_llama3_1_deployment_on_gke.ipynb)
- [Mistral on GKE (TGI)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_tgi_mistral_deployment_on_gke.ipynb)

### Gemma 파인튜닝 — 튜토리얼 노트북

- [Gemma 3 파인튜닝 (Axolotl)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_axolotl_gemma3_finetuning.ipynb)
- [Gemma 1 파인튜닝 (HF PEFT)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_finetuning_on_vertex.ipynb)
- [Gemma 2 파인튜닝 (HF PEFT)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma2_finetuning_on_vertex.ipynb)
- [Gemma 1 파인튜닝 (KerasNLP)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_kerasnlp_to_vertexai.ipynb)
- [Gemma 파인튜닝 + 배치 추론 (Ray)](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_fine_tuning_batch_deployment_on_rov.ipynb)
