---
sidebar_position: 4
description: 텐서 코어가 타일 단위 행렬 연산을 어떻게 실행하는지, 그리고 커널이 충족해야 하는 정밀도, 형태, 레이아웃 제약은 무엇인지 알아봅니다.
keywords:
    - Tensor Cores
    - matrix multiply accumulate
    - GPU mixed precision
    - Tensor Core tiles
    - LLM inference kernels
---

import LinkList from '@site/src/components/LinkList';

# 텐서 코어

**텐서 코어**(Tensor Core)는 NVIDIA 하드웨어에서 행렬 곱셈-누산(matrix
multiply-accumulate) 연산을 위한 전용 실행 유닛입니다. NVIDIA는 Volta
아키텍처에서 텐서 코어를 처음 도입했습니다. 워프 하나 또는 워프 그룹은 스칼라
곱셈과 덧셈 명령어를 길게 이어서 발행하는 대신, 작은 출력 타일을 갱신하는 행렬
명령어 하나를 사용할 수 있습니다.

트랜스포머는 어텐션 프로젝션과 순전파 신경망 레이어를 포함해 실행 시간의 상당
부분을 행렬 곱에 씁니다. 따라서 커널이 요구되는 정밀도, 타일, 레이아웃 제약을
충족하면 텐서 코어는 범용 산술 유닛보다 훨씬 높은 처리량을 낼 수 있습니다.

## 행렬 명령어와 타일 {#matrix-instructions-and-tiles}

개념적으로 텐서 코어 명령어는 다음 형태의 타일 단위 연산을 수행합니다.

$$
D = A \times B + C
$$

네 글자는 행렬 곱셈-누산 한 번의 피연산자입니다. 각각은 타일 크기의 작은
행렬입니다.

- `A`와 `B`는 서로 곱해지는 두 입력 행렬입니다. `A × B`는 행렬 곱입니다.
- `C`는 누산기 입력, 즉 이미 누적된 값입니다.
- `D`는 출력입니다.

애플리케이션이 다루는 행렬은 하드웨어 명령어 타일보다 훨씬 클 수 있습니다.
커널은 행렬을 타일로 나누고, 타일 조각을 스레드에 분배하고, 행렬 곱셈-누산
명령어를 발행한 뒤, 부분 결과를 합칩니다.

프로그래밍 인터페이스는 추상화 수준에 따라 달라집니다. 예를 들면 다음과
같습니다.

- cuBLAS와 cuBLASLt 같은 라이브러리는 내부적으로 텐서 코어 커널을 선택합니다
- Triton 같은 컴파일러는 블록 단위 텐서 프로그램에서 텐서 코어 명령어를 생성할
  수 있습니다

상위 수준 인터페이스는 구현 작업을 줄여 줍니다. 하위 수준 인터페이스는 데이터
이동과 스케줄링을 더 세밀하게 제어할 수 있게 해 주지만, 커널을 특정 아키텍처에
더 강하게 묶기도 합니다.

## 지원 정밀도 {#supported-precision}

텐서 코어가 모든 GPU 세대에서 모든 타입을 받아들이는 것은 아닙니다. 지원 범위는
시간이 지나며 넓어졌습니다.

- Volta는 FP16 행렬 입력과 더 높은 정밀도의 누산을 도입했습니다.
- Ampere는 BF16, TF32 같은 형식과 모드를 추가하고 정수 지원을 넓혔습니다.
- Hopper는 FP8 텐서 코어 경로와 새로운 행렬 명령어를 추가했습니다.
- Blackwell은 FP4와 FP6 형식, 그리고 2세대 Transformer Engine으로 텐서 코어를 더
  확장했습니다.

입력 정밀도와 누산기 정밀도는 다를 수 있다는 점에 유의하세요. 예를 들어 커널은
수치 오차를 줄이기 위해 FP16 또는 BF16 입력을 곱하면서 FP32로 누산할 수
있습니다(수치 오차는 주로 합산 단계에서 쌓입니다). 16비트 누적 합은 제한된
범위를 넘어 오버플로가 나거나, 훨씬 큰 합계에 작은 값을 더할 때 그 값을 잃을 수
있습니다.

입력 타입, 누산기 타입, 타일 형태, 처리량의 정확한 조합은 연산 능력(compute
capability)에 따라 달라집니다. 하나의 텐서 코어 모드가 어디서나 동작한다고
가정하지 말고, 대상 하드웨어에 맞춰 코드를 작성하세요.

## 타일과 레이아웃 요구 사항 {#tile-and-layout-requirements}

텐서 코어는 주변 커널이 데이터를 효율적으로 공급할 때만 높은 처리량을 냅니다.
중요한 요소는 다음과 같습니다.

- **형태**: 행렬 차원에 완전한 명령어 타일이 충분히 들어가야 합니다. 가장자리
  타일에는 패딩, 마스킹, 또는 대체 경로가 필요할 수 있습니다.
- **정렬**: 효율적인 벡터화 로드와 행렬 명령어를 위해 포인터와 선행 차원(leading
  dimension)에 적절한 정렬이 필요한 경우가 많습니다.
- **레이아웃**: 스레드는 아키텍처별 레지스터 레이아웃으로 배치된 행렬 조각을
  기대합니다. 공유 메모리 레이아웃은
  [뱅크 충돌](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/#shared-memory-smem-and-l1-cache)을
  피하기 위해 스위즐링(swizzling)이나 패딩이 필요할 수 있습니다.
- **정밀도**: 입력은 지원되는 타입과 누산 모드를 사용해야 합니다.
- **데이터 공급**: HBM과 공유 메모리 전송이 텐서 코어 파이프라인의 속도를
  따라가야 합니다.

커널이 텐서 코어 명령어를 발행하더라도 성능이 나쁠 수 있습니다. 타일 로딩이 실행
시간의 대부분을 차지하거나 행렬 차원이 너무 작으면, 이론상의 텐서 처리량이
애플리케이션 처리량으로 이어지지 않습니다.

## LLM 추론에서 텐서 코어가 중요한 이유 {#why-tensor-cores-are-important-for-llm-inference}

LLM 추론은 대부분 큰 행렬 곱(matmul)의 연속입니다. QKV 프로젝션과 출력 프로젝션,
순전파 신경망(MLP) 레이어, 그리고 어텐션 내부의 행렬 곱이 여기에 해당합니다. 이
연산들은 텐서 코어가 가속하도록 설계된 바로 그 형태입니다. 그래서 텐서 코어
처리량(TFLOPS 또는 TOPS로 측정)이 추론에서 연산 비중이 큰 부분의 실행 속도를
대체로 결정합니다.

앞서 언급했듯이 텐서 코어는 FP16, BF16, TF32, INT8, FP8, FP4 같은 낮은 정밀도의
입력에서도 동작합니다. 이 점이 텐서 코어를
[양자화](/model-preparation/llm-quantization/)와 직접 연결합니다. 가중치와
활성화 값을 더 낮은 정밀도 형식으로 바꾸면 메모리 사용량이 줄어들고, 더 빠른
텐서 코어 경로도 쓸 수 있습니다. 따라서 두 최적화는 서로를 강화합니다.

LLM 추론의 두 단계를 구체적으로 보면 다음과 같습니다.

- [프리필](/llm-inference-basics/how-does-llm-inference-work/#prefill)
  단계에서는 많은 토큰을 한 번에 처리하므로 행렬 곱이 크고 산술 강도가 높습니다.
  텐서 코어에 데이터가 충분히 공급되고 이 단계는 보통 연산
  바운드(compute-bound)이므로, 텐서 코어 처리량이 핵심입니다.
- [디코드](/llm-inference-basics/how-does-llm-inference-work/#decode) 단계에서는
  병목이 메모리 대역폭(가중치와 KV 캐시 스트리밍)으로 옮겨 갑니다. 그래서 텐서
  코어의 순수 FLOPS는 더 이상 제약 요인이 아니며, 코어는 일부 유휴 상태로
  남습니다.
  [여러 시퀀스를 배칭하면](/inference-optimization/static-dynamic-continuous-batching/)
  행렬 차원이 다시 커지고 텐서 코어 사용률이 회복됩니다. 추론 서버가 공격적으로
  배칭하는 주요 이유가 여기에 있습니다.

따라서 텐서 코어는 추론 성능에 필수적이지만, 그 영향이 균일하지는 않습니다. 연산
비중이 큰 프리필과 큰 배치의 디코드에서는 텐서 코어가 성능 상한을 정하는 경우가
많습니다. 작은 배치의 디코드는 흔히 대역폭 바운드로 남으므로, 텐서 코어 처리량만
늘려서는 도움이 되지 않을 수 있습니다.
[점유율](/kernel-optimization/gpu-architecture-fundamentals/streaming-multiprocessors/#occupancy)과
마찬가지로, 텐서 코어는 연산 상한이 어디인지 알려 줍니다.

## 자주 묻는 질문 {#faqs}

### CUDA 코어와 텐서 코어의 차이는 무엇인가요? {#what-is-the-difference-between-cuda-cores-and-tensor-cores}

CUDA 코어는 일반적인 부동소수점 및 정수 연산을 위한 범용 실행 유닛입니다. 텐서
코어는 작은 타일(예: 아키텍처와 데이터 타입에 따라 4×4 또는 16×16)에 대한 행렬
곱셈-누산을 위해 설계된 전용 유닛입니다. 트랜스포머 레이어처럼 행렬 연산이 많은
워크로드에서는 텐서 코어가 CUDA 코어보다 훨씬 높은 처리량을 냅니다.

### FP16이나 FP8을 사용하면 텐서 코어가 자동으로 활성화되나요? {#does-using-fp16-or-fp8-automatically-activate-tensor-cores}

아니요. 연산, 차원, 레이아웃, 정렬, 생성된 명령어가 모두 지원되는 텐서 코어
경로와 일치해야 합니다. 커널이 실제로 행렬 명령어를 발행하는지, 그리고 해당
파이프라인이 얼마나 잘 활용되는지 항상 코드를 테스트해 확인하세요.

<LinkList>

## 추가 자료 {#additional-resources}

- [CUDA C++ Programming Guide: WMMA](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#wmma)
- [NVIDIA Tensor Cores for HPC](https://docs.nvidia.com/deeplearning/performance/dl-performance-matrix-multiplication/index.html)
- [PTX ISA: Matrix Multiply and Accumulate Instructions](https://docs.nvidia.com/cuda/parallel-thread-execution/#warp-level-matrix-instructions)
</LinkList>
