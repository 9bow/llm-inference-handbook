---
sidebar_position: 2
description: 정적 배칭, 동적 배칭, 연속 배칭으로 GPU 사용률을 높여 LLM 추론을 최적화합니다.
keywords:
    - Static batching, dynamic batching and continuous batching
    - Batch LLM inference, batch requests, batch processing, LLM inference batching, LLM batching
    - Batch size, batch window
    - Chunked prefill, decode-maximal batching
    - Padding tokens, pad tokens, ragged tensors
    - LLM inference optimization, LLM inference optimization techniques, LLM batch API
    - Speed up LLM inference
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import
LinkList from '@site/src/components/LinkList'; import BatchingSimulator
from '@site/src/components/BatchingSimulator'; import ChunkedPrefillVisualizer
from '@site/src/components/ChunkedPrefillVisualizer';

# 정적 배칭, 동적 배칭, 연속 배칭(static, dynamic, continuous batching)

GPU는 고도로 병렬적인 연산 워크로드를 위해 설계되었으며, 초당 수조에서 수천조
번의 부동소수점 연산(FLOPs)을 수행할 수 있습니다. 그런데도 LLM은 GPU를 충분히
활용하지 못하는 경우가 많습니다. 칩의 메모리 대역폭 상당 부분이 모델 매개변수를
불러오는 데 쓰이기 때문입니다.

배칭(batching)은 이 병목을 완화하는 데 도움이 됩니다. 프로덕션 환경에서는 여러
요청이 동시에 몰려들 수 있습니다. 각 요청을 따로 처리하는 대신 요청들을 묶어
배치로 처리하면, 한 번 불러온 모델 매개변수를 여러 요청에 함께 사용할 수 있어
처리량이 크게 향상됩니다.

아래 시뮬레이터로 여러 배칭 전략을 개략적으로 살펴보세요.

<BatchingSimulator />

## 정적 배칭 {#static-batching}

가장 단순한 형태의 배칭은 **정적 배칭**(static batching)입니다. 정적 배칭에서는
서버가 정해진 수의 요청이 도착할 때까지 기다린 뒤, 이 요청들을 하나의 배치로
묶어 함께 처리합니다.

<figure>
  <Diagram name="static-batching" alt="" />
  <figcaption><b>그림 1.</b> 정적 배칭: 고정 크기의 배치를 하나씩 차례로
  처리합니다.</figcaption>
</figure>

정적 배칭은 구현하기 쉽지만, 눈에 띄는 단점이 있습니다.

- 배치의 첫 번째 요청은 마지막 요청이 도착할 때까지 기다려야 하므로 불필요한
  지연이 생깁니다. 마지막 문서가 도착하기까지 얼마나 걸리든, 정해진 수의 문서가
  대기열에 쌓일 때까지 인쇄를 시작하지 않는 프린터를 떠올려 보세요.
- 배치 안의 요청이 모두 같지는 않습니다. LLM 추론에서는 어떤 요청은 아주 짧은
  응답을 생성하고, 어떤 요청은 길고 단계적인 추론을 거칠 수 있습니다. 배치의
  모든 요청은 가장 느린 요청이 끝날 때까지 기다려야 하므로, 연산 자원이 낭비되고
  지연 시간이 늘어날 수 있습니다.

## 동적 배칭 {#dynamic-batching}

정적 배칭의 문제를 해결하기 위해 많은 시스템이 **동적 배칭**(dynamic batching)을
사용합니다. 이 방식도 들어오는 요청을 배치로 모으지만, 배치 크기를 고정하지
않습니다. 대신 시간 창을 정해 두고 그 시간 안에 도착한 요청을 모두 처리합니다.
배치가 그보다 먼저 크기 한도에 도달하면 바로 실행합니다. 정해진 시간표에 따라,
또는 자리가 다 차면, 둘 중 먼저 일어나는 쪽에 맞춰 출발하는 버스와 비슷합니다.

<figure>
  <Diagram name="dynamic-batching" alt="" />
  <figcaption><b>그림 2.</b> 동적 배칭: 도착하는 요청에 따라 배치 크기가
  달라집니다.</figcaption>
</figure>

동적 배칭은 처리량과 지연 시간의 균형을 맞추는 데 도움이 됩니다. 먼저 도착한
요청이 나중 요청 때문에 한없이 지연되지 않도록 보장합니다. 하지만 실행 시점에
배치가 다 차지 않을 수도 있으므로 항상 GPU 효율을 최대로 끌어올리지는 못합니다.
또 다른 단점은 정적 배칭과 마찬가지로 배치에서 가장 긴 요청이 배치가 끝나는
시점을 결정한다는 점입니다. 짧은 요청은 불필요하게 기다려야 합니다.

## 연속 배칭 {#continuous-batching}

LLM 추론에서는 출력 시퀀스의 길이가 크게 다릅니다. 어떤 사용자는 간단한 질문을
하고, 어떤 사용자는 자세한 설명을 요청합니다. 정적 배칭과 동적 배칭에서는 짧은
요청이 가장 긴 요청을 기다려야 합니다. 그러면 GPU 자원을 다 쓰지 못한 채로 남게
됩니다.

인플라이트 배칭(in-flight batching)이라고도 하는 연속 배칭(continuous
batching)은 이런 비효율을 해결합니다. 연속 배칭은 결과를 반환하기 전에 배치
전체가 끝나기를 기다리게 하지 않습니다. 대신 배치 안의 각 시퀀스가 독립적으로
끝나도록 하고, 끝난 시퀀스 자리를 즉시 새 시퀀스로 채웁니다. 조립 라인에서 한
품목이 완성되는 즉시(얼마나 오래 걸렸든) 새 품목을 투입하여 라인을 최대 용량으로
계속 돌리는 것과 비슷합니다.

<figure>
  <img src={require('@site/docs/inference-optimization/img/continuous-batching-diagram.png').default} alt="" />
  <figcaption><b>그림 3.</b> 연속 배칭으로 시퀀스 일곱 개를 생성하는
  모습. 출처: <cite><a
  href="https://www.anyscale.com/blog/continuous-batching-llm-inference">Anyscale</a></cite>.</figcaption>
</figure>

이 기법은 반복 수준 스케줄링(iteration-level scheduling)을 사용합니다. 즉,
디코딩 반복마다 배치 구성이 동적으로 바뀝니다. 배치 안의 한 시퀀스가 토큰 생성을
마치면 서버는 곧바로 그 자리에 새 요청을 넣습니다. 이렇게 하면 배치에서 가장
느린 시퀀스가 끝나기를 기다리며 낭비될 유휴 시간을 없애, GPU 점유율을 최대로
높이고 연산 자원을 계속 바쁘게 유지합니다.

vLLM, SGLang, MAX, TensorRT LLM(인플라이트 배칭), LMDeploy(지속 배칭, persistent
batching) 같은 주요
[추론 프레임워크](/getting-started/choosing-the-right-inference-framework/)는
모두 연속 배칭이나 그와 비슷한 메커니즘을 지원합니다. 길거나 길이가 섞인 배치의
메모리 관리에 대해서는
[PagedAttention](/inference-optimization/pagedattention/)을 참고하세요.

## 청크 단위 프리필 {#chunked-prefill}

연속 배칭에서는 다른 요청들이 디코딩하는 도중에 새 요청이 도착하면 스케줄링
충돌이 생깁니다. 새 요청의 프롬프트 전체를 한 번의 프리필(prefill) 반복으로
처리하면 첫 토큰까지의 시간(TTFT)은 최소화되지만, 긴 프리필은 디코드 중인 모든
요청의 다음 토큰을 지연시킬 수 있습니다. 스트리밍 애플리케이션에서는 다른
사용자의 프롬프트가 처리되는 동안 출력이 멈춘 것처럼 보일 수 있습니다.

**청크 단위 프리필**(chunked prefill)은 프롬프트를 더 작은 토큰 범위로 나누어
여러 반복에 걸쳐 스케줄링합니다. 각 청크는 그 요청의 KV 캐시를 확장하고, 나중
청크는 앞서 처리된 프롬프트 토큰에 어텐션을 적용합니다. 어텐션 연산 자체는
달라지지 않으므로(모든 토큰은 여전히 KV 캐시를 통해 앞선 모든 토큰에 어텐션을
적용합니다), 청크 단위 프리필은 프롬프트를 한 번에 처리하는 것과 수학적으로
동등합니다. 첫 토큰은 마지막 청크를 처리한 뒤에야 출력됩니다.

<ChunkedPrefillVisualizer />

스케줄러는 프리필 청크 하나와 디코드 중인 요청들의 디코드 토큰을 같은 배치에
함께 넣을 수 있습니다. [SARATHI](https://arxiv.org/abs/2308.16369)는 이를
**디코드 최대화 배칭**(decode-maximal batching)이라고 부릅니다. 프리필 청크가
GPU의 연산 용량을 포화시킬 만큼 충분한 병렬 작업을 제공하고, 디코드 토큰은 같은
모델 실행에 거의 추가 비용 없이 편승합니다. 이렇게 하면 긴 프롬프트가 한 반복을
독점하는 것을 막을 수 있고,
[파이프라인 병렬화](/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/#pipeline-parallelism)에서
생기는 파이프라인 버블도 줄일 수 있습니다.

청크 단위 프리필의 장점은 요청의 프롬프트가 길더라도 프리필 연산이 더 작은
청크로 나뉜다는 점입니다. 디코드 중인 요청은 긴 프리필 반복 하나가 끝나기를
기다리지 않고 프리필 청크 사이사이에 토큰을 계속 생성할 수 있습니다. 그 결과
토큰 간 지연 시간(ITL)이 줄고 스트리밍 응답이 더 매끄러워집니다.

대가도 있습니다. 프롬프트를 큰 단계 하나가 아니라 더 작은 프리필 단계 여러 개로
처리하므로 스케줄링과 어텐션 오버헤드가 추가로 생깁니다. 그 결과 TTFT가 늘어날
수 있으며, 특히 청크 크기가 작을 때 그렇습니다.

대부분의 추론 프레임워크에서는 청크 크기를 조정할 수 있습니다. 다만 청크 개수가
아니라 배치당 토큰 예산으로 설정합니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --enable-chunked-prefill \
  --max-batch-input-tokens 8192 \
  --chunked-prefill-min-chunk-size 512
```

`--max-batch-input-tokens`는 배치당 인코딩되지 않은 토큰의 목표 개수를 설정하고,
`--chunked-prefill-min-chunk-size`는 스케줄러가 만들 수 있는 청크 크기의 하한을
정합니다.

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct \
  --enable-chunked-prefill \
  --max-num-batched-tokens 8192
```

`--max-num-batched-tokens`는 스케줄러가 한 반복에서 처리할 수 있는 토큰 수의
상한을 정합니다. 그래서 더 긴 프롬프트는 여러 반복에 나뉘어 처리됩니다.

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path meta-llama/Llama-3.1-8B-Instruct \
  --chunked-prefill-size 8192
```

`--chunked-prefill-size`는 프리필 청크 하나에 들어가는 최대 토큰 수입니다.
SGLang에서 청크 단위 프리필을 끄려면 `-1`로 설정하세요.

</TabItem>
</Tabs>

값을 고를 때는 다음 사항에 유의하세요.

- **작은 청크**는 스케줄러가 디코드를 실행할 기회를 더 많이 주므로, 디코드 중인
  요청의 ITL 급증을 줄여 줍니다.
- **큰 청크**는 새 프롬프트를 더 효율적으로 처리하고 대개 TTFT를 개선하지만,
  디코드 중인 요청은 토큰 사이에 더 오래 기다릴 수 있습니다.
- **너무 작은 청크**는 GPU 사용률을 떨어뜨리고 어텐션 오버헤드를 늘릴 수
  있습니다. 나중 청크가 앞선 청크가 만든 KV 캐시 항목을 다시 읽어야 하기
  때문입니다.

모든 경우에 맞는 최적값은 없습니다. 적절한 크기는 모델, GPU, 프롬프트 길이 분포,
동시성, 지연 시간 목표에 따라 달라집니다. 청크 크기는 보편적인 상수가 아니라
워크로드별 스케줄링 매개변수로 다뤄야 합니다.

## 자주 묻는 질문 {#faqs}

### LLM 배칭에서 패딩 토큰이란? {#what-are-padding-tokens-in-llm-batching}

텍스트 시퀀스는 토큰화하고 나면 자연히 길이가 제각각입니다. 예를 들면 다음과
같습니다.

```bash
Sentence A: "Hello world"        # [15496, 995] - length 2
Sentence B: "How are you today?" # [2437, 389, 345, 1909] - length 4
```

이 시퀀스들은 길이가 다르므로 하나의 밀집 직사각형 텐서로 바로 쌓을 수 없습니다.
패딩(padding)은 짧은 시퀀스에 자리 표시용 토큰을 덧붙여 배치 안의 모든 요청이
같은 텐서 길이를 갖게 함으로써 이 문제를 해결합니다. 흔히 쓰이는 패딩 토큰
형태로는 `PAD`, `<pad>`, 토큰 ID `0` 등이 있지만, 정확한 ID는 토크나이저에 따라
다릅니다.

어텐션 마스크는 어느 위치가 패딩인지 모델에 알려 주지만, 그래도 패딩은 연산과
메모리를 낭비할 수 있습니다. 배치에 다음과 같은 길이의 시퀀스가 들어 있다고 해
봅시다.

```bash
[1024, 1000, 50, 20]
```

짧은 시퀀스들은 길이 `1024`까지 패딩될 수 있습니다. 그러면 GPU 커널이 다루기
좋게 텐서 모양은 규칙적으로 유지되지만, `50`토큰짜리 요청과 `20`토큰짜리 요청은
사용하지 않는 위치를 잔뜩 떠안게 됩니다.

### LLM 추론에서 래그드 텐서란? {#what-are-ragged-tensors-in-llm-inference}

래그드 텐서(ragged tensor)는 가변 길이 시퀀스를 모두 하나의 고정 길이로 패딩하지
않고 표현합니다. 대신 런타임은 실제 토큰과 함께 시퀀스 길이, 오프셋, KV 캐시
블록 위치 같은 메타데이터를 저장합니다. 이 방식은 추론 엔진이 길이가 섞인
프롬프트와 연속 배칭을 더 효율적으로 처리하도록 도와주며, 특히
[PagedAttention](/inference-optimization/pagedattention/) 같은 페이지 단위 KV
캐시 레이아웃과 함께 쓸 때 그렇습니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [How continuous batching enables 23x throughput in LLM inference while reducing p50 latency](https://www.anyscale.com/blog/continuous-batching-llm-inference)
- [SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills](https://arxiv.org/abs/2308.16369)
- [Mastering LLM Techniques: Inference Optimization](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
</LinkList>
