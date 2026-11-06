---
sidebar_position: 4
description: FlashAttention은 트랜스포머를 위한 빠르고 메모리 효율적인 어텐션 알고리즘으로, LLM 학습과 추론을 가속하고 더 긴 컨텍스트 윈도우를 구현하는 데 도움이 됩니다.
keywords:
    - FlashAttention
    - FlashAttention-4
    - Blackwell attention kernels
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

import LinkList from '@site/src/components/LinkList';

# FlashAttention

FlashAttention은 트랜스포머 모델에서 어텐션 메커니즘을 계산하는 최적화된
알고리즘입니다. 표준 어텐션보다 빠르고, 메모리 효율이 높으며, 확장성도 더
뛰어납니다. 논문
[FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135)에서
처음 소개된 이후, 현대 LLM의 학습과 추론 모두에서 널리 채택된 어텐션 백엔드가
되었습니다.

:::note
FlashAttention은 커널 수준의 최적화입니다. 성능 향상은 어텐션 연산 자체가
GPU에서 실행되는 방식에서 나옵니다. 여기에는 HBM과 SRAM 사이에서 데이터가
이동하는 방식, 작업을 타일로 나누는 방식, 여러 연산을 단일 커널로 융합하는
방식이 포함됩니다. 그래서 이 페이지는 커널 최적화 장에 있습니다. 서빙 계층의
기법은 [추론 최적화](/inference-optimization/)를 참고하세요.
:::

## 어텐션은 애초에 왜 느릴까? {#why-attention-is-slow-in-the-first-place}

LLM은 텍스트를 읽을 때 토큰 간의 관계를 파악하기 위해 모든 토큰을 살펴보고 각
토큰을 다른 모든 토큰과 비교해야 합니다. 이를
[어텐션](https://arxiv.org/abs/1706.03762)이라고 합니다.

표준 어텐션 메커니즘에는 근본적인 문제가 있습니다. 바로
**연산 바운드(compute-bound)가 아니라 메모리 바운드(memory-bound)라는 점**입니다.
이를 이해하려면 어텐션 연산 중에 어떤 일이 일어나는지 살펴봐야 합니다.

HBM, SRAM, 워프, 타일링 같은 개념의 바탕이 되는 저수준 GPU 배경 지식이
필요하다면
[GPU 아키텍처 기초](/kernel-optimization/gpu-architecture-fundamentals/)를
참고하세요.

표준 어텐션은 다음을 계산합니다.

$$
\text{Attention}(Q, K, V)
= \mathrm{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

단순한 구현은 다음 단계를 따릅니다.

1. **어텐션 점수 계산**: Q와 K^T를 곱해 N×N 행렬을 얻습니다(N은 시퀀스 길이).
2. **소프트맥스 적용**: 점수를 정규화합니다.
3. **값과 곱하기**: V와의 가중합을 계산합니다.

문제는 메모리 접근 패턴입니다. 현대 GPU에는 다음과 같은 메모리가 있습니다.

- **고대역폭 메모리(High Bandwidth Memory, HBM)**: 크지만 느립니다(대역폭 1-2
  TB/s).
- **SRAM(온칩 메모리)**: 작지만 빠릅니다(대역폭 10-20 TB/s).

표준 구현에서는 N×N 어텐션 행렬 전체를 HBM에 쓴 다음, 다음 연산을 위해 다시 읽어
와야 합니다. 시퀀스 길이가 4096 토큰이면 이 어텐션 행렬의 원소는 약 1,600만
개입니다. 읽기와 쓰기가 여러 번 반복되므로, 알고리즘은 실제 연산보다 메모리
전송을 기다리는 데 대부분의 시간을 씁니다.

시퀀스 길이가 늘어나면 다음과 같은 일이 일어납니다.

- 메모리 트래픽이 실행 시간의 대부분을 차지합니다.
- GPU 사용률이 떨어집니다.
- 긴 컨텍스트 윈도우를 현실적으로 사용하기 어려워집니다.

예를 들어, 16K 토큰에는 1K 토큰보다 **256× 더 많은 메모리**가 필요합니다.

## FlashAttention은 어떻게 동작할까? {#how-does-flashattention-work}

FlashAttention은 메모리 트래픽을 줄여 어텐션을 빠르게 만듭니다. 핵심 아이디어는
전체 어텐션 행렬을 HBM에 절대 실체화하지 않는 것입니다. 대신 두 가지 핵심 기법을
사용합니다.

- **타일링과 재계산**. FlashAttention은 연산을 빠른 SRAM에 들어가는
  블록(타일)으로 나눕니다.
  - Q, K, V의 타일을 HBM에서 빠른 SRAM으로 불러옵니다.
  - 해당 타일의 어텐션을 전부 SRAM 안에서 계산합니다.
  - 출력을 점진적으로 갱신하고 중간 결과는 버립니다.
- **커널 융합**. FlashAttention은 연산을 따로따로(matmul → softmax → matmul)
  수행하지 않고, 모든 연산을 단일 GPU 커널로 융합합니다. 그 결과는 다음과
  같습니다.
  - 중간 결과를 HBM에 쓰지 않습니다.
  - 별도의 커널 실행(오버헤드가 있음)이 없습니다.
  - 모든 연산이 빠른 SRAM에서 이루어집니다.

<figure>
  <img src={require('@site/docs/kernel-optimization/img/flashattention-diagram.png').default} alt="" />
  <figcaption><b>그림 1.</b> FlashAttention의 타일링은 어텐션 블록을 SRAM에 유지합니다. 출처: <cite><a href="https://arxiv.org/abs/2205.14135">FlashAttention:
  Fast and Memory-Efficient Exact Attention with IO-Awareness</a></cite>.</figcaption>
</figure>

간단히 말해, FlashAttention은 어텐션 연산을 더 효율적으로 만듭니다. 작업을
재구성해 GPU가 메모리를 기다리는 시간은 줄이고 실제 연산에 쓰는 시간은 늘립니다.

Triton, CUDA, 컴파일러 스택, 프로파일링 도구가 이런 작업에서 어떤 역할을 하는지
더 넓은 관점에서 보려면
[커널 최적화 도구](/kernel-optimization/kernel-optimization-tools/)를
참고하세요.

## FlashAttention의 이점 {#the-benefits-of-flashattention}

FlashAttention은 속도와 확장성 모두를 크게 개선합니다.

- 2–4× 빠른 어텐션
- N×N 어텐션 행렬을 저장하지 않으므로 훨씬 적은 메모리 사용량
- LLM이 더 긴 컨텍스트 윈도우(예: 128K 토큰)를 처리 가능
- 더 높은 처리량과 더 나은 GPU 사용률
- 채팅, 코딩, 추론 등에서 더 빠른 추론

현재 FlashAttention은 다음과 같은 곳에서 널리 쓰입니다.

- 학습 프레임워크(PyTorch, DeepSpeed)
- 추론 엔진(vLLM, SGLang, MAX, TensorRT LLM)
- 긴 컨텍스트를 지원하는 모델 아키텍처

## FlashAttention 버전 비교 {#flashattention-version-comparison}

주요 FlashAttention 계열에는 현재 4개의 메이저 버전이 있습니다. 아래 표는
버전별로 알고리즘이 어떻게 발전해 왔는지 나란히 비교합니다.

| 버전             | 연도 | 주요 개선 사항                                                                                                         | 성능                                                                                                              | 비고                                                                                                            |
|------------------|------|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| FlashAttention-1 | 2022 | IO 인식(IO-aware) 타일 기반 어텐션 알고리즘 도입. 소프트맥스 + matmul 커널 융합. 전체 어텐션 행렬의 실체화 회피        | 2–4× 빠른 어텐션, 최대 10× 적은 메모리 사용량                                                                     | 첫 번째 버전. 실용적인 긴 컨텍스트 지원. 정확한 어텐션(근사 없음)                                               |
| FlashAttention-2 | 2023 | 워프 간 병렬성과 작업 분할 개선. matmul이 아닌 FLOPs 감소                                                              | FA-1보다 2× 빠름, 특히 긴 시퀀스에서 효과가 큼                                                                    | 많은 긴 컨텍스트 LLM의 기반. 추론/학습 프레임워크에 널리 통합                                                   |
| FlashAttention-3 | 2024 | 텐서 코어 가속(FP8/BF16). Hopper GPU(예: H100)에 최적화                                                                | FA-2보다 최대 2× 빠르고 H100에서 740 TFLOPS(사용률 75%) 달성. FP8 수치 오차 2.6× 감소                             | Hopper의 비동기 실행과 워프 특화(warp specialization) 활용. 많은 프레임워크가 여전히 FA-2에서 먼저 업그레이드함 |
| FlashAttention-4 | 2026 | 완전 비동기 MMA, 더 큰 타일, 소프트웨어로 에뮬레이션한 지수 함수, 조건부 소프트맥스 리스케일링, 텐서 메모리, 2-CTA MMA | B200 BF16 벤치마크에서 cuDNN 9.13 대비 최대 1.3×, Triton 대비 최대 2.7× 빠름. 최대 1613 TFLOPS/s(사용률 71%) 달성 | CuTeDSL로 작성. 공식 구현은 `flash-attn-4`로 제공되며 H100, B200 같은 Hopper 및 Blackwell GPU 대상              |

FlashAttention-4는 특히 NVIDIA Blackwell 아키텍처에 맞게 튜닝되었습니다. 핵심
통찰은 하드웨어의 비대칭적인 확장입니다. 텐서 코어(QKᵀ와 PV 같은 큰 행렬 곱을
수행함)는 Blackwell에서 훨씬 빨라졌습니다. 하지만 다른 핵심 자원은 그만큼
확장되지 않았습니다.

- 공유 메모리 대역폭.
- 소프트맥스의 지수 연산에 쓰이는 특수 함수 유닛(special function unit, SFU).
- 레지스터 압박과 스케줄링 오버헤드.

그 결과 B200에서는 병목 지점이 바뀝니다. 자세한 내용은
[FlashAttention-4 논문](https://arxiv.org/abs/2603.05451)에서 확인하세요.

## FlashAttention 사용 방법 {#how-to-use-flashattention}

가장 쉽게 시작하는 방법은 공식 패키지를 사용하는 것입니다.

```bash
pip install flash-attn --no-build-isolation
```

최신 PyTorch 버전은 지원되는 경우 `scaled_dot_product_attention`을 통해 자동으로
FlashAttention으로 디스패치합니다. 자세한 내용은
[API 레퍼런스](https://docs.pytorch.org/docs/stable/backends.html#torch.backends.cuda.can_use_flash_attention)를
참고하세요.

많은 추론 프레임워크가 이미 FlashAttention을 통합했지만, 프레임워크별 릴리스
주기에 따라 사용하는 버전이 다를 수 있습니다.

FlashAttention-4의 경우, 공식 저장소에서 별도의 CuTeDSL 패키지를 안내합니다.

```bash
pip install flash-attn-4
```

CUDA 13에서는 저장소에서 다음 방법을 권장합니다.

```bash
pip install "flash-attn-4[cu13]"
```

FA4 API는 `flash_attn.cute`를 통해 제공됩니다.

```python
from flash_attn.cute import flash_attn_func

out = flash_attn_func(q, k, v, causal=True)
```

FA4 패키지와 프레임워크 통합이 빠르게 바뀌고 있으므로, 프로덕션에서 사용하기
전에 최신
[FlashAttention 저장소](https://github.com/Dao-AILab/flash-attention)를
확인하세요.

<LinkList>

## 추가 자료 {#additional-resources}

- [FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135)
- [FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning](https://arxiv.org/abs/2307.08691)
- [FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision](https://arxiv.org/abs/2407.08608)
- [FlashAttention-4: Algorithm and Kernel Pipelining Co-Design for Asymmetric Hardware Scaling](https://arxiv.org/abs/2603.05451)
- [공식 FlashAttention 저장소](https://github.com/Dao-AILab/flash-attention)
</LinkList>
