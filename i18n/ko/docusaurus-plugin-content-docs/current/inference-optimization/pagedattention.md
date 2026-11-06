---
sidebar_position: 3
description: PagedAttention의 블록 기반 KV 캐시 저장 방식으로 LLM의 메모리 사용을 개선합니다.
keywords:
    - vLLM, Hugging Face TGI, TensorRT LLM
    - PagedAttention
    - KV cache, KV cache optimization, KV caching
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import LinkList from '@site/src/components/LinkList';

# PagedAttention

PagedAttention은 LLM 추론에서 KV 캐시를 메모리 효율적으로 관리하는 방식입니다.
서빙 측면의 주된 이점은 더 빠른 어텐션 커널에서 나오지 않습니다. 서빙 엔진이 KV
캐시 메모리를 할당하고 관리하는 방식에서 나옵니다. 어텐션 커널이 메커니즘의
일부를 구현하기는 하지만(룩업 테이블을 통해 KV 블록을 읽음), 이점은 서빙
레이어에서 생깁니다. 이 페이지가 추론 최적화 장에 있는 이유도 그래서입니다. 커널
수준의 어텐션 효율에 대해서는
[FlashAttention](/kernel-optimization/flashattention/)을 참고하세요.

## 어텐션과 KV 캐시 {#attention-and-the-kv-cache}

어텐션은 트랜스포머가 토큰끼리 얼마나 강하게 연관되어 있는지 평가하게 해 주는
메커니즘입니다. 모델은 각 토큰에 대해 세 가지 벡터를 계산합니다.

- **쿼리(Query, Q)**: 현재 토큰이 찾고 있는 것
- **키(Key, K)**: 각 토큰이 매칭을 위해 내놓는 것
- **값(Value, V)**: 각 토큰이 기여하는 내용

모델은 쿼리와 키를 비교해 어텐션 점수를 만들고, 소프트맥스로 정규화한 뒤, 그
결과로 얻은 가중치로 값들의 가중합을 구합니다.

자기회귀 생성 중에는 새 토큰마다 이전 모든 토큰의 키와 값이 필요합니다. 서빙
엔진은 매 단계마다 키와 값을 다시 계산하는 대신
[KV 캐시에 저장합니다](/llm-inference-basics/how-does-llm-inference-work/#what-is-kv-cache).
이 캐시는 시퀀스 길이에 따라 커지며, 여러 동시 요청에 걸쳐 상당한 양의 GPU
메모리를 차지할 수 있습니다. 엔진이 이 메모리를 어떻게 할당하느냐가 바로
PagedAttention이 해결하는 문제입니다.

자세한 내용은
[어텐션 메커니즘](/llm-inference-basics/how-does-llm-inference-work/#the-attention-mechanism)을
참고하세요.

## 연속된 KV 캐시 할당이 메모리를 낭비하는 이유 {#why-contiguous-kv-cache-allocation-wastes-memory}

일반적으로 KV 캐시는 하나의 거대한 연속 블록으로 저장되기 때문에 메모리를 많이
차지합니다. 다 채우지 않더라도 큰 블록을 미리 확보해야 하므로 메모리
단편화(fragmentation)나 공간 낭비로 이어질 수 있습니다.

구체적으로, 초기 서빙 엔진은 KV 캐시를 최악의 경우에 맞춘 크기의 연속 텐서로
할당하는 경우가 많았습니다.
[단순화한 형태](/inference-optimization/kv-cache-offloading/#how-to-calculate-the-kv-cache-size)는
다음과 같습니다.

```bash
2 × num_layers × num_heads × head_dim × max_seq_len
```

이 할당은 배치 크기와 요소당 바이트 수를 고려하기 전, 활성 요청마다
이루어집니다. 방식은 단순하지만, 모든 요청이 최대 시퀀스 길이를 사용한다고
가정합니다. 실제 트래픽은 들쭉날쭉합니다. 어떤 요청은 짧은 답변을 생성하고, 어떤
요청은 긴 대화를 이어 가며, 또 어떤 요청은 일찍 끝날 수 있습니다. 각 요청이
`max_seq_len`만큼 메모리를 확보한다면, 확보된 GPU 메모리의 상당 부분이 쓰이지
않은 채 남을 수 있습니다.

그 결과 실질 배치 크기가 줄고, 메모리 단편화가 심해지며, 처리할 수 있는 동시
요청 수가 줄어듭니다.

자세한 내용은
[PagedAttention 블로그 글](https://blog.vllm.ai/2023/06/20/vllm.html)을
참고하세요.

## PagedAttention은 어떻게 동작할까? {#how-does-pagedattention-work}

PagedAttention은 이 큰 덩어리를 책의 페이지처럼 더 작은 블록으로 나눕니다. 즉,
KV 캐시를 연속되지 않은 블록에 저장합니다. 그리고 룩업 테이블로 이 블록들을
추적합니다. LLM은 모든 것을 한꺼번에 로드하는 대신 필요한 블록만 로드합니다.

이렇게 하면 메모리가 절약되고 전체 과정이 더 효율적이 됩니다. 필요하면 서로 다른
출력 간에 같은 블록을 공유할 수도 있습니다.

PagedAttention 원 논문에 따르면, PagedAttention이 없을 때는 할당된 KV 캐시
메모리 중 20.4%-38.2%만 실제 토큰 상태를 저장하는 데 쓰이고 나머지는 단편화로
낭비됩니다. 반면 PagedAttention은 KV 캐시 메모리 낭비를 거의 0에 가깝게
줄입니다.

PagedAttention이 단일 어텐션 커널 이상의 의미를 갖는 이유가 여기에 있습니다.
PagedAttention은 서빙 엔진에 더 나은 KV 캐시 메모리 할당기를 제공하고, 그 덕분에
[연속 배칭](/inference-optimization/static-dynamic-continuous-batching/),
[프리픽스 캐싱](/inference-optimization/prefix-caching/),
[KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/) 같은 기법을
결합하기가 쉬워집니다.

PagedAttention은 vLLM에서 처음 구현되었습니다. 이후 MAX 같은 다른 프로젝트도
PagedAttention을 도입해 구현했습니다. 이 프로젝트들은 블록 크기를 서빙 플래그로
노출하지만, 플래그 이름은 다를 수 있습니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --kv-cache-page-size 256
```

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct \
  --block-size 16
```

</TabItem>
</Tabs>

<LinkList>

## 추가 자료 {#additional-resources}

- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
</LinkList>
