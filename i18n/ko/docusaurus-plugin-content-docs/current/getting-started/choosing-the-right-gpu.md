---
sidebar_position: 3
description: "LLM 추론에 맞는 NVIDIA 또는 AMD GPU(예: L4, A100, H100, B200, MI250X, MI300X, MI350X)를 고르는 방법을 알아봅니다."
keywords:
    - NVIDIA GPUs, AMD GPUs
    - GPU inference
---

import GPUTable from '@site/src/components/GPUTable';

# 적합한 GPU 고르기

LLM을 셀프 호스팅하는 AI 팀에게 적합한 GPU를 고르는 일은 초기에 내려야 하는 가장
중요한 결정 중 하나입니다. 이 선택은 처리량, 지연 시간, 메모리 한계, 전체 비용에
직접 영향을 줍니다. GPU 벤치마크나 GPU 비교 차트에 기대어 결정하기 쉽지만, 이런
수치만으로는 특정 사용 사례에서 LLM 워크로드가 어떻게 동작하는지 제대로 알기
어렵습니다.

## GPU, 그래픽 카드, 가속기의 차이 {#gpus-vs-graphics-cards-vs-accelerators}

먼저 흔히 같은 뜻으로 쓰이지만 실제로는 의미가 다른 몇 가지 용어를
정리하겠습니다.

### GPU(Graphics Processing Unit) {#gpu-graphics-processing-unit}

GPU는 프로세서 칩 자체를 가리킵니다. 원래 그래픽 렌더링용으로 설계되었으며, 수천
개의 계산을 동시에 실행할 수 있습니다. 현대 AI 워크로드에서 GPU는 무거운 연산을
담당하는 "두뇌"입니다.

### 그래픽 카드 {#graphics-card}

그래픽 카드는 GPU를 담고 있는 하드웨어 패키지 전체를 말합니다. 칩, 메모리(VRAM),
냉각 장치, 전원 커넥터, 출력 포트가 모두 포함됩니다. "비디오 카드"나 "그래픽
어댑터"라는 용어를 보게 될 수도 있습니다. GPU는 카드의 일부일 뿐이지만 핵심
부품입니다.

### 가속기 {#accelerator}

가속기는 특정 유형의 연산을 빠르게 처리하도록 만든 특수 하드웨어를 아우르는 더
넓은 범주입니다. GPU도 가속기의 한 종류이며, 그 밖에도 다음과 같은 가속기가
있습니다.

- AI/ML 가속기(예: Google TPU, Intel NPU)
- 암호화 가속기
- 물리 연산 장치(Physics Processing Unit, PPU)
- 특정 작업용으로 구성한 FPGA(Field-Programmable Gate Array)

핵심 차이는 이렇습니다. 현대 그래픽 카드에는 모두 GPU가 들어 있고 모든 GPU는
가속기이지만, 모든 가속기가 GPU이거나 그래픽 중심인 것은 아닙니다. 오늘날 많은
GPU는 이미지 렌더링보다 ML/AI 학습과 추론 같은 그래픽 외 작업에 주로 쓰입니다.

클라우드 벤더에서 컴퓨팅 자원을 빌리면 보통 **GPU** 항목 아래에 비용이
표시됩니다. 하지만 벤더의 마케팅 자료나 블로그 글을 읽을 때는 정확히 무엇을
설명하는지 알아 두는 것이 중요합니다.

## 추론에서 GPU 선택이 중요한 이유 {#why-gpu-choice-matters-for-inference}

현대 AI 애플리케이션은 점점 더 LLM 같은 생성형 AI(GenAI) 모델로 구동됩니다. 기존
ML 모델과 달리 이런 모델은 매개변수가 수천억 개에 이를 수 있습니다. 예를 들어
DeepSeek-V3.1은 매개변수가 671B개입니다. 이런 규모의 모델을 실행하려면 NVIDIA
H200이나 AMD MI300X처럼 매우 강력한 GPU가 필요합니다. 그래야 최신 최적화
기법으로 모델의 추론 성능을 온전히 끌어낼 수 있습니다.

하지만 모든 워크로드에 그만한 성능이 필요하지는 않습니다. Llama-3.1-8B 같은 작은
오픈소스 LLM은 NVIDIA L4나 AMD MI250 같은 중급 GPU에서도 효율적으로 실행됩니다.
경량 모델은 보급형 카드나 일반 클라우드 인스턴스에서도 실행할 수 있습니다.

핵심은 작업에 맞는 GPU를 찾아 가격 대비 성능을 최대로 끌어올리는 것입니다. 잘못
고르면 병목이 생겨 처리량이 제한되고, 지연 시간이 늘어나며, 비용이 올라갈 수
있습니다.

## GPU 유형의 이해 {#understanding-gpu-types}

모든 GPU가 같은 목적으로 만들어지지는 않습니다. GPU 벤치마크를 보면 데이터
센터용 카드, 소비자용 그래픽 카드, 심지어 모바일 칩까지 섞여 있는 경우가
많습니다. 추론 워크로드에 맞는 GPU를 고르기 전에 주요 범주를 이해해 두는 것이
중요합니다.

### 소비자용 GPU {#consumer-gpus}

이 GPU들은 원래 게임용으로 만들어졌지만, 작은 오픈소스 LLM이나 실험 용도로도
널리 쓰입니다. 보통 VRAM은 적지만 비용 효율이 좋습니다. NVIDIA RTX 4090이나 AMD
Radeon RX 7900 XTX가 대표적입니다.

### 워크스테이션 GPU {#workstation-gpus}

워크스테이션 카드는 소비자용 하드웨어와 데이터 센터용 하드웨어 사이에
위치합니다. 3D 디자인, 시각화, 모델 프로토타이핑 등을 위해 단일 머신에서 강력한
연산 성능이 필요한 전문가에게 잘 맞습니다. NVIDIA RTX A6000이나 AMD Radeon Pro
W6800 같은 카드가 이 범주에 속합니다.

### 데이터 센터 GPU {#data-center-gpus}

기업은 대규모 AI 추론과 고성능 컴퓨팅(high-performance computing, HPC)
워크로드에 데이터 센터 GPU를 사용합니다. 데이터 센터 GPU는 큰 VRAM(40–192GB),
높은 메모리 대역폭, 그리고 클러스터 단위 확장을 위한 MIG(Multi-Instance GPU)나
NVLink 같은 기능을 제공합니다. NVIDIA A100, H100, B200과 AMD MI300X, MI350X가
대표적입니다.

클라우드 컴퓨팅 자원을 빌리거나
[LLM을 온프레미스에 배포하는](/getting-started/on-prem-llms/) 팀에게는 보통
데이터 센터 GPU가 가장 현실적인 선택입니다.

## GPU를 고를 때 고려할 사항 {#key-considerations-for-choosing-gpus}

GPU를 고를 때는 벤치마크 수치만으로 모든 것을 알 수 없다는 점을 기억해야 합니다.
최선의 선택은 하드웨어 사양, 워크로드 규모, 생태계 지원의 조합에 따라
달라집니다.

### GPU 메모리(VRAM) {#gpu-memory-vram}

추론 중에 GPU가 다루는 모든 데이터, 즉 모델 가중치, 활성화 값, KV 캐시는
메모리에 올라가 있어야 합니다. 그래서 VRAM이 모델 크기와 컨텍스트 길이의 상한을
결정합니다. 가중치는 기본 메모리 사용량을 결정하므로, 모델을 서빙하려면 먼저
모델이 메모리에 들어가야 합니다. 예를 들어 매개변수가 671B개인 DeepSeek V3와
R1을 실행하려면 NVIDIA H200 GPU(각 141 GB) 8개가 필요합니다. 반면 Phi-3 같은
작은 모델은 양자화하면 16–24GB 안에 들어갈 수 있습니다. 남은 VRAM은 KV 캐시가
사용하며, 이 공간이 지원할 수 있는 컨텍스트 길이를 제한합니다.

프로덕션 환경에서는 KV 캐시가 주된 난관이 되곤 합니다. KV 캐시 크기는 시퀀스
길이에 비례해 선형으로 늘어나므로, 긴 컨텍스트를 다루는 워크로드는 메모리를 금방
소진할 수 있습니다. 병목을 피하려면
[프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)나
[KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/) 같은
[분산 추론](/infrastructure-and-operations/distributed-inference/) 기법이
필요합니다.

자세한 내용은
[GPU 메모리 계층 구조](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/)를
참고하세요.

### 메모리 대역폭 {#memory-bandwidth}

메모리 대역폭은 GPU가
[HBM](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/#hbm-high-bandwidth-memory)과
연산 코어 사이에서 데이터를 얼마나 빠르게 옮길 수 있는지를 나타내며, GB/s나 TB/s
단위로 측정합니다. LLM 추론에서 메모리 대역폭은 가장 중요한 사양 중 하나입니다.
**디코드 단계가 메모리 바운드**(memory-bound)이기 때문입니다. 새 토큰을 하나
생성할 때마다 GPU는 모델 가중치의 대부분 또는 전부(그리고 점점 커지는 KV 캐시)를
HBM에서 반복해서 읽어 와야 합니다. 그런데 읽어 온 데이터 1바이트당 수행하는
연산은 상대적으로 적습니다.

그 결과 단일 스트림 디코드 속도의 이론적 상한은 대략 다음과 같습니다.

```bash
maximum decode tokens/sec ≈ memory bandwidth / bytes read per token
```

예를 들어 FP16으로 저장한 70B 매개변수 모델의 가중치는 약 140 GB입니다. HBM
대역폭이 약 3.35 TB/s인 H100 SXM에서는 다른 오버헤드(예: KV 캐시)를 고려하기
전에도 시퀀스당 이론적 상한이 약 24 tokens/s입니다.

이 때문에 프롬프트를 병렬로 처리하는 프리필 단계는 디코드와 매우 다르게
동작합니다.
[배칭](/inference-optimization/static-dynamic-continuous-batching/)이 처리량을
효과적으로 높이는 이유도 여기에 있습니다. 같은 가중치 읽기를 여러 시퀀스에서
재사용할 수 있으므로, GPU는 메모리 대역폭 부담을 추가 연산 작업과 맞바꿀 수
있습니다.

### 연산 처리량 {#compute-throughput}

연산 처리량은 GPU가 초당 수행할 수 있는 수학 연산의 양으로, FLOPS(floating-point
operations per second, 초당 부동소수점 연산 횟수) 단위로 측정합니다. 연산
처리량은 **연산 바운드(compute-bound)인 프리필 단계**를 제한합니다. 긴 프롬프트
인코딩, 큰 배치 처리, 그리고 산술 강도가 높은 모든 워크로드가 여기에 해당합니다.
현대 데이터 센터 GPU는 전용 행렬 연산 장치(NVIDIA Tensor Core, AMD Matrix
Core)로 이런 수치를 달성하며, 정밀도를 낮추면 처리 속도가 대략 두 배씩
늘어납니다(예: FP16 → FP8 → FP4).

사양표를 읽을 때는 몇 가지 주의할 점이 있습니다.

- **별표를 확인하세요**. 벤더는 흔히 희소성을 적용했거나 지원하는 가장 낮은
  정밀도에서 측정한 최대 FLOPS를 내세웁니다. 카드를 비교할 때는 같은 정밀도,
  같은 밀집/희소 가정에서 비교해야 합니다.
- **FLOPS는 사용자가 체감하는 지표가 아닙니다**. 결국 중요한 것은
  [초당 토큰 수와 지연 시간](/llm-inference-basics/llm-inference-metrics/)(TTFT와
  ITL)입니다. FLOPS가 아무리 높아도 디코드 중에는
  [메모리 대역폭](#memory-bandwidth)이 병목이 될 수 있으므로 두 수치를 함께 봐야
  합니다.
- **정밀도 지원 여부는 절대 조건입니다**. 네이티브 FP8을 쓰려면 NVIDIA H 시리즈
  이상이 필요합니다. 이전 세대 카드는 더 높은 정밀도로 대체해 실행하므로 처리량
  이점을 잃습니다.

실효 처리량을 더 높이려면
[추측 디코딩](/inference-optimization/speculative-decoding/),
[프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/),
[연속 배칭](/inference-optimization/static-dynamic-continuous-batching/) 같은
기법을 적용하세요.

### GPU 인터커넥트 {#gpu-interconnect}

GPU 인터커넥트(interconnect)는 워크로드가 여러 GPU에 걸쳐 있을 때 단일 노드
안에서든 여러 노드 사이에서든 데이터(예: KV 캐시)를 얼마나 빠르게 주고받을 수
있는지를 결정합니다.
[텐서 병렬화, 파이프라인 병렬화](/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/)나
그 밖의
[분산 추론 기법](/infrastructure-and-operations/distributed-inference/)을
사용하는 대형 모델에서 특히 중요합니다. 인터커넥트가 느리면 GPU를 추가해 메모리
용량은 늘릴 수 있어도 기대한 만큼 처리량이나 지연 시간이 개선되지 않을 수
있습니다.

- **노드 내 인터커넥트**. 같은 서버 안에 있는 GPU 간 통신을 말합니다. 긴밀하게
  결합된 멀티 GPU 추론에서는 NVIDIA NVLink/NVSwitch나 AMD Infinity Fabric 같은
  고대역폭 패브릭이 PCIe만 사용하는 구성보다 훨씬 빠른 경우가 많습니다. 예를
  들어 H100에서 NVLink는 양방향 900 GB/s의 GPU 간 대역폭을 제공하는데, 이는
  양방향 128 GB/s인 PCIe 5.0 단일 링크보다
  [약 7배 빠릅니다](https://www.nvidia.com/en-us/data-center/h100/).
- **노드 간 인터커넥트**. 서로 다른 GPU 서버 간 네트워킹을 말합니다. 더 크거나
  특수한 시스템도 있지만, 단일 노드에는 보통 GPU가 4–8개 들어 있습니다. 전력,
  냉각, 폼 팩터 같은 한계로 모델이나 워크로드가 단일 노드의 수용 범위를 넘어서면
  네트워크를 통해 통신해야 합니다. 현실적인 해법은 InfiniBand(GPUDirect RDMA
  포함)나 세밀하게 튜닝한 RoCEv2 이더넷입니다.

경험적으로는 통신량이 가장 많은 병렬화를 가능한 한 단일 노드 안에 두는 것이
좋습니다. 여러 노드로 확장해야 한다면 GPU 종류만 보지 말고 클러스터 토폴로지
전체를 평가하세요.

### 비용과 가용성 {#cost-and-availability}

소비자용 GPU와 워크스테이션 GPU는 구하기 쉽고 저렴하지만 VRAM이 제한적인
편입니다. 데이터 센터 GPU는 기업용 AI 배포에 필요한 규모와 신뢰성을 제공하지만
**가격이 높습니다**. NVIDIA H100이나 H200 같은 고성능 GPU는 특히 그렇습니다.

기업 AI 팀에게 더 큰 과제는 **GPU CAP 정리**(GPU CAP Theorem)입니다. GPU
인프라는 **통제권**(Control), 온디맨드 **가용성**(Availability),
**가격**(Price)을 동시에 보장할 수 없습니다.

|                     | 하이퍼스케일러 | NeoCloud(서버리스) | NeoCloud(장기 약정) | 온프레미스 |
|---------------------|----------------|--------------------|---------------------|------------|
| **통제권**          | 높음           | 낮음               | 중간                | 높음       |
| 온디맨드 **가용성** | 중간           | 높음               | 낮음                | 낮음       |
| **가격**            | 높음           | 중간               | 낮음                | 중간       |

### 생태계와 프레임워크 지원 {#ecosystem-and-framework-support}

GPU의 효과는 이를 지원하는 소프트웨어에 달려 있습니다. NVIDIA는 성숙한 CUDA
Toolkit과 TensorRT LLM 생태계의 이점을 누립니다. AMD의 ROCm 스택도 꾸준히
개선되고 있으며, PyTorch, vLLM, SGLang, MAX 전반에서 지원이 늘고 있습니다.

## 오픈소스 LLM별 적합한 GPU {#matching-gpus-to-open-source-llms}

모델마다 가장 좋은 성능을 내는 GPU 유형이 다릅니다. 아래 표는 널리 쓰이는
NVIDIA와 AMD GPU를 그에 적합한 오픈소스 LLM과 짝지은 것입니다. 일부 모델은
**VRAM 요구량을 충족하려면 여러 GPU나 노드가 필요**하며,
[양자화](/model-preparation/llm-quantization/) 같은 최적화 기법이 필요할 수도
있습니다.

<GPUTable />

:::note
이 표는 참고용으로만 사용하세요. 프로덕션 배포에서는 반드시 대상 하드웨어에서
직접 모델을 벤치마크해야 합니다.
:::

유의할 점은 다음과 같습니다.

- **FP8 지원**. NVIDIA GPU를 고른다면, 네이티브 FP8 가중치를 사용하는 LLM은
  NVIDIA H 시리즈(또는 그 이후) GPU에서만 실행된다는 점에 유의하세요. A 시리즈
  카드에는 FP8 하드웨어 지원이 없기 때문입니다.
- **단일 GPU와 멀티 GPU**. 카드 한 장으로 실행할 수 있는 모델도 있지만, 보통은
  GPU를 여러 개 쓰면 성능이 향상됩니다(예: 동시성이 높은 시나리오).
- **하드웨어 유연성**. 대부분의 모델은 여러 하드웨어에서 실행할 수 있습니다.
  예를 들어 gpt-oss-20b와 gpt-oss-120b는 NVIDIA A100, H100, H200, B200 GPU나 AMD
  MI300X, MI325X, MI355X GPU에서 실행할 수 있습니다. 제약이 되는 요인은 보통
  아키텍처가 아니라 VRAM과 클러스터 규모입니다.
  [LLM 서빙에 필요한 GPU 메모리를 계산하는 방법](/getting-started/calculating-gpu-memory-for-llms/)을
  알아보세요.

<ContactCallout>

LLM 셀프 호스팅을 위해 GPU 옵션을 검토하고 있나요? 저희는 단일 코드베이스로
NVIDIA, AMD, Apple Silicon, CPU 등에서 오픈 모델과 커스텀 모델을 모두 실행할 수
있도록 지원합니다. 필요에 따라 모델을 로컬에서 실행하거나, 자체 클라우드(BYOC)에
배포하거나, 공유 엔드포인트와 전용 엔드포인트를 사용할 수 있습니다.

</ContactCallout>

## 자주 묻는 질문 {#faqs}

### 가중치는 언제, 어떻게 GPU 메모리에 로드되나요? {#when-and-how-are-weights-loaded-into-gpu-memory}

가중치 로드는 서비스가 시작될 때 이루어집니다. 구체적인 파이프라인은 다음과
같습니다.

1. 디스크(SSD/네트워크 스토리지)에서 모델 파일을 읽습니다
   - 모델 체크포인트(예: `.safetensors`, `.bin`)
   - CPU를 통해 읽음
   - GPU 메모리에 비해 매우 느림
   - 대역폭: SSD 기준 약 1–10 GB/s
2. CPU RAM에 스테이징합니다
   - 가중치를 시스템 메모리에 임시로 올려 둠
   - 역직렬화하거나 메모리 매핑하는 경우가 많음
   - CPU RAM 대역폭: 일반적인 구성에서 약 50–200 GB/s
3. [GPU HBM](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/#hbm-high-bandwidth-memory)으로
   전송합니다
4. 서빙 프로세스가 살아 있는 동안 HBM에 캐시됩니다

복사가 끝나면 가중치는 HBM에 저장되어 매번 순전파할 때마다 반복해서 읽을 수
있습니다. 디코딩할 때 가중치를 디스크나 CPU에서 다시 로드하지 않으며, HBM에 있는
가중치를 그대로 재사용합니다.

[텐서 병렬화](/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/)를
사용한다면 각 GPU는 모든 레이어의 가중치 행렬을 일부분씩 나눠 가집니다.

### AI 워크로드에 가장 좋은 GPU 비교 도구는 무엇인가요? {#what-is-the-best-gpu-comparison-tool-for-ai-workloads}

일반적인 GPU 비교 도구는 대부분 게임이나 그래픽 성능에 초점을 맞추므로 실제 AI
추론 워크로드를 반영하지 못합니다. LLM에는
[TTFT, ITL 같은 처리량 및 지연 시간 지표](/llm-inference-basics/llm-inference-metrics/)를
측정하는 도구가 필요합니다.

먼저 vLLM, SGLang, MAX, TensorRT LLM 같은 프레임워크의 벤치마크 결과를 확인해
보세요. 이들 프레임워크는 여러 GPU의 추론 성능을 비교하는 데 도움이 되는 바로 쓸
수 있는 스크립트를 제공합니다.

다만 이런 프레임워크는 보통 수동 설정과 튜닝이 필요해 시간이 많이 들 수
있습니다.

### GPU 서버는 어디서 구매하거나 대여할 수 있나요? {#where-can-i-buy-or-rent-gpu-servers}

규모, 통제 요구 사항, 예산에 따라 온프레미스 GPU 서버를 구매하거나 클라우드
GPU를 대여할 수 있습니다.

AWS, Google Cloud, Azure 같은 클라우드 공급자에서는 H100, H200, MI300X GPU를
온디맨드로 대여할 수 있습니다.

CoreWeave나 Nebius 같은 NeoCloud는 더 저렴하게 이용할 수 있고 과금 방식도
유연합니다. 하지만 일반적으로 통제권이 적고, 규제 대상 환경이나 기업 환경에
필요한 컴플라이언스 보장도 더 적습니다.

완전한 소유를 원한다면 NVIDIA, AMD와 직접 협력하는 Dell, GIGABYTE, HPE 같은
OE(Original Equipment) 파트너에게서 GPU 서버를 직접 구매할 수 있습니다. 이
방식은 통제권을 최대로 확보할 수 있지만, 초기 비용이 높고 조달 기간도 깁니다.

### 내 GPU가 무엇인지 어떻게 확인하나요? {#how-can-i-check-what-gpu-i-have}

대부분의 시스템에서는 명령줄 도구로 GPU 종류를 빠르게 확인할 수 있습니다.

- **Linux**: `nvidia-smi`(NVIDIA) 또는 `amd-smi`(AMD).
- **macOS**: `system_profiler SPDisplaysDataType`.
- **Windows**: **장치 관리자(Device Manager)** →
  **디스플레이 어댑터(Display Adapters)** 를 엽니다.

### GPU를 고를 때 CUDA와 드라이버 버전은 얼마나 중요한가요? {#how-important-are-cuda-and-driver-versions-when-choosing-a-gpu}

매우 중요합니다. GPU 성능은 하드웨어만으로 결정되지 않습니다. NVIDIA 드라이버,
CUDA 버전, 프레임워크 빌드(예: PyTorch, vLLM, SGLang, MAX, TensorRT LLM)가 모두
서로 맞아야 합니다. 맞지 않으면 오류가 나거나, 속도가 느려지거나, FP8이나
[FlashAttention](/kernel-optimization/flashattention/) 같은 기능을 쓸 수 없게
됩니다. 이런 불일치가 왜 문제가 되는지 더 낮은 수준에서 이해하고 싶다면
[GPU 아키텍처 기초](/kernel-optimization/gpu-architecture-fundamentals/)를
참고하세요.

NVIDIA GPU의 경우 다음과 같습니다.

- **드라이버**는
  [CUDA Driver API](https://docs.nvidia.com/cuda/cuda-driver-api/index.html)를
  포함하며 GPU와 직접 통신합니다
- **CUDA 툴킷**은 개발 도구, 컴파일러, 라이브러리를 제공합니다
- **cuDNN, cuBLAS, NCCL**은 PyTorch와 대부분의 추론 엔진 내부 연산을 담당합니다
- **프레임워크 빌드**는 특정 CUDA 툴킷 버전에 맞춰 컴파일됩니다

스택 중 어느 하나라도 오래되었다면 다음과 같은 문제가 생길 수 있습니다.

- "CUDA driver version is insufficient"
- 커널 실패
- 낮은 처리량
- FP8, FlashAttention 또는 장치 수준 최적화 누락

간단한 경험칙은 다음과 같습니다.

- **드라이버의 CUDA 버전**은 프레임워크를 빌드할 때 사용한 **CUDA 툴킷 버전**
  이상이어야 합니다.
- 최신 드라이버는 보통 이전 CUDA 툴킷과 하위 호환됩니다.
- 오래된 드라이버로는 최신 CUDA 런타임을 실행할 수 없습니다.

드라이버와 GPU는 다음 명령으로 확인할 수 있습니다.

```bash
nvidia-smi

# Example output:
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.129.03             Driver Version: 535.129.03   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
```

이 출력은 드라이버가 CUDA 12.2 런타임까지 지원한다는 뜻입니다. 프레임워크는 CUDA
12.2, 12.1, 11.8 등으로 빌드된 것은 쓸 수 있지만, 12.3 이상으로 빌드된 것은 쓸
수 없습니다.

업그레이드하려면 공식 [CUDA 툴킷](https://developer.nvidia.com/cuda-downloads)과
[드라이버](https://www.nvidia.com/en-us/drivers/) 패키지를 내려받으세요.
