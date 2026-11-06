---
sidebar_position: 5
description: LLM 추론 프레임워크가 하는 일, 프로덕션에서 모델을 그대로 실행하는 것만으로는 부족한 이유, 그리고 사용 사례에 맞는 추론 프레임워크를 고르는 방법을 알아봅니다.
keywords:
    - Inference frameworks, inference backends, inference runtimes, inference engines, inference platforms
    - Best inference frameworks, best LLM inference providers, LLM inference benchmark
    - vLLM, SGLang, LMDeploy, TensorRT LLM, Hugging Face TGI, llama.cpp, MLC-LLM, Ollama
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import LinkList from '@site/src/components/LinkList';

# 적합한 추론 프레임워크 선택하기

모델을 골랐다면 다음 단계는 모델을 어떻게 실행할지 정하는 것입니다. 어떤 추론
프레임워크를 선택하느냐에 따라 지연 시간, 처리량, 하드웨어 효율, 지원 기능이
직접 달라집니다. 모든 상황에 맞는 단일 해법은 없습니다. 적합한 선택은 배포
시나리오, 워크로드, 모델, 인프라에 따라 달라집니다.

## 추론 프레임워크란? {#what-are-inference-frameworks}

추론 프레임워크는 모델을 불러오고, 알맞은 하드웨어에서 실행하고, 애플리케이션에
출력을 제공하는 소프트웨어 레이어입니다. LLM에서는 보통 모델의 `forward()`
함수를 호출하는 것 이상을 뜻합니다. 프레임워크는 토큰 생성,
[KV 캐시](/inference-optimization/kv-cache-offloading/), 배칭(batching),
스트리밍 응답, 메모리 한도, 요청 처리도 관리합니다.

이런 도구는 추론 런타임, 추론 엔진, 추론 백엔드, 모델 서버라고 부르기도 합니다.
정확한 의미는 프로젝트마다 다르지만 핵심 역할은 같습니다. 모델 실행을 효율적으로
만들고, 학습용 노트북 밖에서도 쓸 수 있게 하는 것입니다.

## 추론 프레임워크가 왜 필요한가요? {#why-do-i-need-an-inference-framework}

PyTorch나 Hugging Face Transformers에서 모델을 그대로 사용해 직접 추론을 실행할
수도 있습니다. 실험이나 로컬 테스트, 한 번에 요청 하나를 처리하는 경우라면 이
방식으로 충분할 때가 많습니다. 하지만 프로덕션 추론에는 대개 충분하지 않습니다.

학습 프레임워크는 가중치를 학습하는 데 맞춰 만들어졌습니다. 역전파, 옵티마이저
단계, 그래디언트 누적, 큰 학습 배치를 지원합니다. 추론의 목표는 다릅니다. 낮은
지연 시간, 높은 처리량, 안정적인 메모리 사용, 스트리밍 출력, 동시 트래픽에서도
예측 가능한 동작이 필요합니다.

추론 프레임워크는 모델을 그대로 실행하는 방식으로는 잘 해결되지 않는 서빙 고유의
작업을 처리합니다. 예를 들면 다음과 같습니다.

- **배칭과 스케줄링**: 활성 요청을 묶어 GPU가 계속 일하도록 합니다.
- **KV 캐시 관리**: 긴 프롬프트와 멀티턴 대화를 위해 어텐션 상태를 효율적으로
  저장합니다.
- **스트리밍**: 전체 응답을 기다리지 않고 토큰이 생성되는 대로 반환합니다.
- **메모리 제어**: 제한된 GPU 메모리에 더 큰 모델과 더 많은 동시 요청을
  담습니다.
- **프로덕션 API**: [OpenAI 호환](/model-interaction/openai-compatible-api/)
  엔드포인트나 프레임워크 전용 엔드포인트를 제공합니다.
- **멀티 GPU 지원**: GPU 하나로 부족할 때 큰 모델을 여러 장치에 나눕니다.

이런 프레임워크는 모델 실행의 복잡한 부분 상당수를 조정 가능한 서빙 옵션 뒤로
감춥니다. 다음은 Gemma 4 31B를 서빙하는 예시입니다. 플래그 이름과 허용되는 값은
다르지만, 각 프레임워크는 모델 병렬화, 메모리 사용률, 최대 시퀀스 길이처럼
비슷한 범주의 조정 항목을 제공합니다. 이 값들은 모델 자체를 건드리지 않고 직접
조정합니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model google/gemma-4-31B-it \
  --devices gpu:0,1 \
  --max-batch-size 16 \
  --device-memory-utilization 0.95 \
  --max-length 262144
```

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model google/gemma-4-31B-it \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.90 \
  --enable-auto-tool-choice \
  --reasoning-parser gemma4 \
  --tool-call-parser gemma4 \
  --chat-template examples/tool_chat_template_gemma4.jinja \
  --limit-mm-per-prompt '{"image": 4, "audio": 1}' \
  --async-scheduling
```

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path google/gemma-4-31B-it \
  --reasoning-parser gemma4 \
  --tool-call-parser gemma4 \
  --mem-fraction-static 0.9 \
  --host 0.0.0.0 --port 30000
```

</TabItem>
</Tabs>

추론 프레임워크가 저수준 인프라 작업을 추상화해 주므로, 모델 아키텍처마다 추론
로직을 다시 구현하는 대신 애플리케이션 개발에 집중할 수 있습니다.

이점은 편의성에 그치지 않습니다. 좋은 추론 프레임워크는 같은 하드웨어에서 같은
모델의 지연 시간, 처리량, 비용 특성을 개선할 수 있습니다. 예를 들어
[2023년 벤치마크](https://vllm.ai/blog/2023-06-20-vllm)에서 vLLM 팀은 모델
아키텍처 자체는 전혀 바꾸지 않고도 Hugging Face Transformers보다 최대 24× 높은
처리량을 얻었다고 보고했습니다.

이런 차이는 Transformers를 그대로 쓸 때와의 비교에서만이 아니라 추론 프레임워크
사이에서도 나타납니다.
[2026년 벤치마크](https://www.modular.com/blog/three-trends-from-mlsys-2026)에서
Modular는 NVIDIA B200에서 MAX로 `google/gemma-4-31B-it`를 서빙했을 때 vLLM 대비
P99 [TTFT](/llm-inference-basics/llm-inference-metrics/)가 2.5× 빠르고 처리량은
1.5×였다고 보고했습니다. 이런 수치는 워크로드에 따라 달라집니다. 따라서 어디서나
통하는 순위로 받아들이기보다는 자신의 트래픽으로 직접 벤치마크해 볼 이유로
삼으세요.

## 추론 프레임워크와 도구 {#inference-frameworks-and-tools}

높은 처리량과 낮은 지연 시간을 갖춘 LLM 애플리케이션을 구축할 때 많이 쓰는 추론
프레임워크는 다음과 같습니다.

- [vLLM](https://github.com/vllm-project/vllm). LLM 서빙에 최적화된 고성능 추론
  엔진입니다. GPU 자원을 효율적으로 사용하고 디코딩이 빠른 것으로 알려져
  있습니다.
- [SGLang](https://github.com/sgl-project/sglang). LLM과 비전-언어 모델을 위한
  빠른 서빙 프레임워크입니다. 백엔드 런타임과 프론트엔드 언어를 함께 설계해
  모델과의 상호작용을 더 빠르고 더 세밀하게 제어할 수 있게 합니다.
- [MAX](https://github.com/modular/modular). Modular가 만든 고성능 AI 서빙
  프레임워크입니다. CPU와 GPU에서 AI 연산 워크로드를 처리하는 통합 도구 모음을
  제공하며, 모델 수준과
  [커널 수준](/kernel-optimization/kernel-optimization-tools/) 모두에서
  커스터마이징을 지원합니다.
- [LMDeploy](https://github.com/InternLM/lmdeploy). 빠른 디코딩 속도와 효율적인
  동시 요청 처리에 초점을 둔 추론 백엔드입니다. 다양한 양자화 기법을 지원하므로
  메모리 요구량을 줄여 큰 모델을 배포하는 데 적합합니다.
- [TensorRT LLM](https://github.com/NVIDIA/TensorRT-LLM). 고성능 심층 학습 추론
  라이브러리인 NVIDIA TensorRT를 활용하는 추론 백엔드입니다. NVIDIA GPU에서 큰
  모델을 실행하도록 최적화되어 있으며, 빠른 추론과 양자화 같은 고급 최적화를
  지원합니다.
- [TokenSpeed](https://github.com/lightseekorg/tokenspeed). LightSeek
  Foundation이 에이전트 워크로드를 겨냥해 만든 추론 엔진입니다. 플러그인 방식의
  계층형 커널 시스템, 안전한 KV 캐시 관리를 갖춘 정교한 스케줄러, 컴파일러 기반
  병렬화, OpenAI 호환 서빙 API를 갖추고 있습니다.
- [Hugging Face TGI](https://github.com/huggingface/text-generation-inference).
  LLM을 배포하고 서빙하기 위한 툴킷입니다. Hugging Face에서 Hugging Chat,
  Inference API, Inference Endpoint를 구동하는 데 프로덕션으로 쓰이고 있습니다.
  다만 Hugging Face TGI는 현재 **유지 관리 모드**(maintenance mode)입니다.
  지금도 지원되고 사용할 수 있지만, 더 이상 주요 기능 개발이나 새로운 성능
  최적화는 없다는 뜻입니다. 프로덕션에서 TGI를 운영하고 있다면 성능이나 확장
  요구가 커질 때를 대비해 전환 경로를 계획해 두는 것이 좋습니다.

하드웨어가 제한적이거나 데스크톱/엣지 기기를 대상으로 한다면 다음 도구들이
저사양 환경에 최적화되어 있습니다.

- [llama.cpp](https://github.com/ggml-org/llama.cpp). 외부 의존성 없이 순수
  C/C++로 구현한 LLM용 경량 추론 런타임입니다. 주된 목표는 LLM 추론을 빠르고
  이식성 있게 만들고, 다양한 하드웨어에서 쉽게 실행할 수 있게 하는 것입니다.
  이름과 달리 llama.cpp는 Llama 모델 외에도 훨씬 많은 모델을 지원합니다. Qwen,
  DeepSeek, Mistral 같은 인기 아키텍처도 지원합니다. 저지연 추론에 이상적이며
  소비자용 GPU에서도 좋은 성능을 냅니다.
- [MLC-LLM](https://github.com/mlc-ai/mlc-llm). LLM을 위한 ML 컴파일러이자
  고성능 배포 엔진입니다. Apache TVM 위에 구축되었으며, 모델을 서빙하기 전에
  컴파일과 가중치 변환이 필요합니다. MLC-LLM은 Linux, Windows, macOS, iOS,
  Android, 웹 브라우저에서 AMD, NVIDIA, Apple, Intel GPU를 지원하므로 폭넓은
  하드웨어 플랫폼에서 사용할 수 있습니다.
- [Ollama](https://ollama.com/). llama.cpp 위에 구축된 사용하기 쉬운 로컬 추론
  도구입니다. 단순함과 사용 편의성을 중시해 설계되었으며, 최소한의 설정으로
  노트북에서 모델을 실행하기에 이상적입니다. 다만 Ollama는 주로 단일 요청 사용
  사례에 쓰입니다. vLLM, SGLang, MAX 같은 런타임과 달리 동시 요청을 지원하지
  않습니다. 이 차이는 중요합니다. 페이지드 어텐션(paged attention), 프리픽스
  캐싱, 동적 배칭 같은 많은 추론 최적화는 여러 요청을 병렬로 처리할 때만 효과가
  있기 때문입니다.

이 프레임워크 중 일부는 텍스트 생성을 넘어 디퓨전 모델 서빙까지 지원하도록
발전했습니다.

- [SGLang Diffusion](https://docs.sglang.io/docs/sglang-diffusion)은 FLUX, Wan,
  Qwen-Image 같은 이미지·비디오 모델을 지원합니다.
- [vLLM-Omni](https://docs.vllm.ai/projects/vllm-omni/en/latest/)는 vLLM을
  확장해 텍스트, 이미지, 비디오, 오디오 전반에서 Diffusion Transformers(DiT)와
  기타 병렬·비자기회귀 생성 모델을 지원합니다.
- [MAX](https://www.modular.com/solutions/image-generation?utm_source=llm_handbook)는
  FLUX 같은 디퓨전 모델을 네이티브 PyTorch보다 최대 4x 빠르게 서빙합니다.

## 라이브러리 모드와 서버 모드 {#library-mode-vs-server-mode}

vLLM과 SGLang 같은 많은 모델 추론 프레임워크는 두 가지 배포 패턴을 지원합니다.
프레임워크를 애플리케이션 안에 라이브러리로 내장해 실행을 더 세밀하게 제어할
수도 있고, 독립 서버로 실행해 외부 클라이언트가 HTTP API로 호출하게 할 수도
있습니다.

### 프레임워크를 라이브러리로 내장하기 {#embed-the-framework-as-a-library}

라이브러리 모드는 추론 엔진을 애플리케이션과 같은 프로세스에 불러옵니다.
오프라인 배치 작업, 평가 파이프라인, 추가 네트워크 홉 없이 엔진 출력에 직접
접근해야 하는 커스텀 서비스에 잘 맞습니다.

예를 들어 vLLM과 SGLang 모두 이를 위한 인프로세스 엔진을 제공합니다.

<Tabs groupId="inference-framework">
<TabItem value="vllm" label="vLLM">

[vLLM 오프라인 추론 API](https://docs.vllm.ai/en/latest/serving/offline_inference.html)는
`LLM` 클래스를 제공합니다.

```python
from vllm import LLM, SamplingParams

model = "meta-llama/Llama-3.1-8B-Instruct"
llm = LLM(model=model)

outputs = llm.generate(
    ["Explain continuous batching in two sentences."],
    SamplingParams(temperature=0.2, max_tokens=64),
)
```

</TabItem>
<TabItem value="sglang" label="SGLang">

[SGLang의 오프라인 엔진](https://docs.sglang.io/docs/basic_usage/offline_engine_api)도
비슷한 인터페이스를 제공합니다.

```python
import sglang as sgl

model = "meta-llama/Llama-3.1-8B-Instruct"
llm = sgl.Engine(model_path=model)

outputs = llm.generate(
    ["Explain continuous batching in two sentences."],
    {"temperature": 0.2, "max_new_tokens": 64},
)
```

</TabItem>
</Tabs>

이 경우 엔진의 수명 주기는 애플리케이션이 책임집니다. 크래시, 의존성 충돌, GPU
메모리 부족 오류가 프로세스 전체에 영향을 줄 수 있으므로, 이 패턴에서는 자원과
동시성을 신중하게 관리해야 합니다.

### 프레임워크를 독립 서버로 실행하기 {#run-the-framework-as-a-standalone-server}

서버 모드는 추론 엔진을 별도 프로세스에서 실행하고 REST API나 스트리밍 API를
제공합니다. 여러 애플리케이션이 한 모델을 공유하거나, 클라이언트가 서로 다른
프로그래밍 언어를 쓰거나, 서빙 레이어를 독립적으로 확장하고 배포해야 할 때는
대개 이쪽이 더 나은 경계가 됩니다.

OpenAI 호환 서버를 시작합니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct
```

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct
```

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path meta-llama/Llama-3.1-8B-Instruct
```

</TabItem>
</Tabs>

이 서버들은 [OpenAI 호환 API](/model-interaction/openai-compatible-api/)를
제공하므로, 애플리케이션 코드는 다음과 같이 같은 클라이언트 인터페이스를 사용할
수 있습니다.

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[
        {"role": "user", "content": "Explain continuous batching in two sentences."}
    ],
    temperature=0.2,
    max_tokens=64,
)

print(response.choices[0].message.content)
```

서버 경계를 두면 네트워크와 직렬화 오버헤드가 생기지만, 추론 런타임을
애플리케이션 코드와 격리해 줍니다. 덕분에 라우팅, 인증, 관측 가능성, 독립적인
확장을 더하기가 쉬워집니다. 어느 모드든 모델 실행, 배칭, KV 캐시 관리는 여전히
프레임워크가 처리합니다. 구체적인 선택은 사용 사례와 애플리케이션이 엔진과
통합되는 방식에 따라 달라집니다.

## 여러 추론 런타임이 필요한 이유 {#why-do-you-need-multiple-inference-runtimes}

실제 배포 환경에서 모든 시나리오에 완벽한 런타임은 없습니다. AI 팀이 결국 여러
런타임을 함께 쓰게 되는 이유는 다음과 같습니다.

### 사용 사례마다 요구 사항이 다릅니다 {#different-use-cases-have-different-needs}

모델, 하드웨어, 워크로드는 제각각입니다. 최고의 성능은 대개 각 사용 사례를 그
환경에 맞춘 런타임과 짝지을 때 나옵니다.

- **높은 처리량, 배칭**: vLLM, SGLang, MAX, LMDeploy, TensorRT LLM (더 나은
  성능을 위해 튜닝 필요)
- **엣지/모바일 배포**: MLC-LLM, llama.cpp
- **로컬 실험 또는 단일 사용자 시나리오**: Ollama, llama.cpp
- **디퓨전 모델 서빙(이미지/비디오, 멀티모달)**: SGLang Diffusion, vLLM-Omni,
  MAX

### 툴체인과 프레임워크는 빠르게 발전합니다 {#toolchains-and-frameworks-evolve-fast}

추론 런타임은 끊임없이 업데이트됩니다. 오늘 최고의 도구가 다음 달에는 필요한
기능을 갖추지 못할 수도 있습니다. 또한 일부 모델은 출시 시점에 특정 런타임에서만
최적화(또는 지원)됩니다.

유연성을 유지하려면 인프라가 특정 런타임에 구애받지 않아야 합니다. 그래야 단일
스택에 묶이지 않고 각 도구의 장점을 조합할 수 있습니다.

## 로컬 LLM에서 분산 추론으로 확장하기 {#scaling-from-local-llms-to-distributed-inference}

많은 팀이 LLM 추론을 확장할 때 대체로 비슷한 경로를 밟습니다.

처음에는 Ollama 같은 도구로 노트북이나 소형 워크스테이션에서 모델을 로컬로
실행하는 경우가 많습니다. 빠른 데모와 초기 프로토타이핑에는 이 방식이 잘
맞습니다. 단순하고 프라이빗하지만, 실질적인 동시성이나 배칭이 없는 단일 사용자
워크로드로 제한됩니다.

그다음 팀은 고성능 서버 런타임으로 넘어갑니다. 이런 런타임은 데이터 센터 GPU에서
연속 배칭(continuous batching), KV 캐시 최적화, 향상된 GPU 사용률을 제공합니다.
하지만 대부분은 멀티 리전 라우팅, 자동 장애 조치(failover), 진정한 수평 확장을
기본으로 갖추고 있지 않습니다. GPU 프로비저닝, 성능 튜닝, 내결함성도 여전히
구현하기 복잡하고 시간이 많이 듭니다.

여러 GPU 클러스터, 리전, 클라우드에 걸쳐 추론을 실행하고 확장해야 할 때가
옵니다. 이때 팀은 보통
[분산 추론](/infrastructure-and-operations/distributed-inference/) 플랫폼을
도입해 프로덕션 규모의 오토스케일링, 라우팅, 관측 가능성, 규정 준수 요구 사항을
처리합니다. 이런 플랫폼은 고급 기능을 기본으로 제공하므로, 엔지니어링 팀은
인프라를 구축하고 유지하는 대신 제품 혁신에 집중할 수 있습니다.

## 자주 묻는 질문 {#faqs}

### 모든 추론 프레임워크가 모든 LLM과 호환되나요? {#are-all-inference-frameworks-compatible-with-every-llm}

항상 그렇지는 않습니다. 일부 프레임워크는 특정 아키텍처부터 먼저 지원합니다.
멀티 GPU 지원, 추측 디코딩(speculative decoding), 커스텀 어텐션 백엔드 같은 고급
기능을 추가하는 데 시간이 걸리는 프레임워크도 있습니다. 런타임을 선택하기 전에
항상 모델별 호환성을 확인하세요.

### LLM 분산 추론을 지원하는 추론 프레임워크는 무엇인가요? {#which-inference-frameworks-support-distributed-inference-for-llms}

일부 모델은 너무 커서 GPU 하나에 들어가지 않으므로 분산 추론이 필요합니다. vLLM,
SGLang, MAX 같은 프레임워크는 프리필-디코드 분리(prefill-decode
disaggregation)나 여러 워커에 걸친 KV 인식 라우팅(KV-aware routing) 같은 고급
최적화를 제공합니다. 덕분에 메모리 한도에 걸리지 않고 더 큰 모델을 실행하고, 더
긴 컨텍스트 윈도우를 처리하고, 더 많은 동시 트래픽을 서빙할 수 있습니다.

### 추론 프레임워크를 실험해 보기에 가장 좋은 방법은 무엇인가요? {#whats-the-best-way-to-start-experimenting-with-inference-frameworks}

작게 시작해서 차근차근 수준을 높여 가는 것이 좋습니다. 많은 사람이 Ollama로
시작합니다. 설정이 거의 필요 없이 노트북에서 실행되기 때문입니다. 빠른 테스트,
[프롬프트 다듬기](/model-interaction/prompt-engineering/), 여러 모델이 어떻게
동작하는지 감을 잡는 데 안성맞춤입니다. 기본을 이해하고 프로덕션을 위한 실제
성능을 평가하고 싶다면 vLLM, SGLang, MAX로 넘어가세요. 이 프레임워크들은
프로덕션 수준의 워크로드를 위해 만들어졌으므로, 현실적인 환경에서 지연 시간,
처리량, 배칭 동작, GPU 효율을 측정할 수 있습니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [LLM 성능 벤치마크](/inference-optimization/llm-performance-benchmarks/)
</LinkList>
