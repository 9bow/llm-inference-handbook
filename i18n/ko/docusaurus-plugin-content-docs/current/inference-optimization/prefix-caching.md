---
sidebar_position: 6
description: 프리픽스 캐싱은 공유 프롬프트의 KV 캐시를 여러 요청에서 재사용해 LLM 추론 속도를 높입니다.
keywords:
    - Prefix caching, prompt caching, context caching
    - KV cache, KV caching
    - Distributed inference, distributed LLM inference
    - Inference optimization
    - Dynamo, SGLang, vLLM, llm-d
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import LinkList from '@site/src/components/LinkList';

# 프리픽스 캐싱

프리픽스 캐싱은 프롬프트 캐싱(prompt caching) 또는 컨텍스트 캐싱(context
caching)이라고도 합니다. LLM 추론에서 지연 시간과 비용을 줄이는 가장 효과적인
기법 중 하나입니다. 채팅 시스템, AI 에이전트, RAG 파이프라인처럼 프롬프트 구조가
반복되는 프로덕션 워크로드에서 특히 유용합니다.

아이디어는 간단합니다. 기존 쿼리의 KV 캐시를 캐싱해 두면, 같은 프리픽스를
공유하는 새 쿼리는 프롬프트의 그 부분을 다시 계산하지 않아도 됩니다. 대신 캐시된
결과를 바로 재사용합니다.

프리픽스 캐싱은 단순한 시맨틱 캐싱(semantic caching)과 다릅니다. 시맨틱 캐싱은
입력과 출력 텍스트 전체를 데이터베이스에 저장하고, 정확히 일치하는 쿼리(또는
비슷한 쿼리)만 캐시에 적중해 즉시 결과를 반환합니다.

## 프리픽스 캐싱은 어떻게 동작하나요? {#how-does-prefix-caching-work}

프리픽스 캐싱은 모델이 이미 계산한 어텐션 상태를 재사용합니다.

1. 프리필(prefill) 단계에서 모델은 입력 토큰에 대해 순전파를 실행해 키-값(KV)
   캐시를 만듭니다.
2. 디코드(decode) 단계에서 모델은 프리필 단계에서 캐시된 상태를 사용해 출력
   토큰을 하나씩 생성합니다. 어텐션 메커니즘은 토큰 간 상호작용을 나타내는
   행렬을 계산합니다. 각 토큰에서 나온 KV 쌍은 GPU 메모리에 저장됩니다.
3. 새 요청이 도착하면 모델은 요청의 시작 부분부터 일치하는 가장 긴 캐시된 토큰
   시퀀스를 찾습니다. 그 KV 상태를 불러오고 나머지 토큰에 대해서만 프리필을
   실행합니다.

이 방식은 공백과 형식까지 포함해 프리픽스가 정확히 같을 때만 동작합니다. 문자
하나만 달라도 캐시를 쓸 수 없습니다. 다음 세 프롬프트를 살펴보세요:

```bash
Prompt 1: Summarize this incident report in one paragraph.
Prompt 2: Summarize this incident report in three bullet points.
Prompt 3: Write a one-paragraph summary of this incident report.
```

Prompt 2는 공유하는 시작 부분 `Summarize this incident report in`의 KV 상태를
재사용할 수 있습니다. 모델은 두 프롬프트가 갈라지는 서픽스(suffix)만 처리하면
됩니다. Prompt 3은 비슷한 결과를 요청하고 Prompt 1과 같은 단어를 많이
포함하지만, 같은 토큰 시퀀스로 시작하지 않습니다. 따라서 Prompt 1의 캐시를
처음부터 재사용할 수 없습니다.

이 점이 프리픽스 캐싱과 시맨틱 캐싱의 차이입니다. 의미가 같은 두 프롬프트라도
프리픽스 캐시에는 적중하지 못할 수 있습니다. 반대로 마지막 질문이 서로 다른 두
요청이 캐시된 연산의 대부분을 공유할 수도 있습니다.

다음은 프리픽스 캐싱의 대표적인 두 가지 사용 사례입니다:

### 고정된 시스템 프롬프트 재사용 {#reusing-a-static-system-prompt}

캐시 가능한 프리픽스의 흔한 예는 많은 요청이 공유하는 시스템 프롬프트입니다:

```bash
You are a helpful AI writer. Please write in a professional manner.
```

이 프롬프트와 직렬화 방식이 바뀌지 않는다면, 모델은 KV 상태를 한 번 계산해 여러
대화에서 재사용할 수 있습니다. 그러면 각 요청은 그 뒤에 오는 사용자별 내용에
대해서만 프리필을 실행합니다.

### 늘어나는 채팅 기록 재사용 {#reusing-a-growing-chat-history}

멀티턴 채팅에서는 효과가 더 두드러집니다. 첫 번째 턴이 다음과 같다고 해 봅시다:

```bash
User: Why did the checkout API slow down after deployment?
Assistant: Trace data shows that repeated inventory database lookups added most of the latency.
```

이어서 사용자가 이렇게 묻습니다:

```bash
User: Which lookup should we optimize first?
```

모델은 최신 질문만 받는 것이 아닙니다. 애플리케이션이 이전 메시지를
[컨텍스트 윈도우](/llm-inference-basics/how-does-llm-inference-work/#what-is-a-context-window-and-how-does-it-work-in-llm-inference)의
일부로 다시 보내므로, 모델은 사용자가 말하는 조회가 무엇인지 알 수 있습니다.
따라서 직렬화된 두 번째 요청은 대략 다음과 같습니다:

```bash
User: Why did the checkout API slow down after deployment?
Assistant: Trace data shows that repeated inventory database lookups added most of the latency.
User: Which lookup should we optimize first?
```

처음 두 메시지는 모델이 이미 처리한, 정확히 일치하는 프리픽스입니다. 이
메시지들의 KV 상태가 아직 남아 있다면 모델은 이를 재사용하고 새 사용자 메시지만
프리필할 수 있습니다. 프리픽스 캐싱이 없으면 대화 전체를 다시 처리해야 합니다.

대화가 길어질수록 재사용할 수 있는 프리픽스도 함께 늘어납니다. 반복되는 프리필을
피하면 GPU 연산이 줄고, 턴이 거듭될수록 TTFT가 가파르게 늘어나는 것을
억제합니다. 다만 캐시 적중 여부는 서빙 엔진이 캐시 항목을 유지하는지,
[캐시에 접근할 수 있는 워커로 요청을 라우팅하는지](/inference-optimization/inference-routing/),
이전 메시지를 똑같이 직렬화하는지에 여전히 좌우됩니다.

## KV 캐싱과 프리픽스 캐싱의 차이는 무엇인가요? {#what-is-the-difference-between-kv-caching-and-prefix-caching}

KV 캐싱은 각 토큰의 중간 어텐션 상태를 GPU 메모리에 저장하는 데 쓰입니다. 원래는
**단일 추론 요청** 안에서의 캐싱을 가리키는 말이었고, 특히 디코딩 단계의 속도를
높이는 데 필수적입니다.

LLM은 디코드 중에 자기회귀 방식으로 동작합니다. 이전에 생성한 토큰을 바탕으로
다음 새 토큰을 출력합니다(즉, 이전 토큰의 KV 캐시를 재사용합니다). KV 캐시가
없으면 모델은 디코드 단계마다 이전 토큰 전체를 다시 계산해야 하고, 컨텍스트는
단계마다 늘어납니다. 이는 엄청난 자원 낭비입니다.

이 캐싱 개념을 **여러 요청**에 걸쳐 확장할 때는 프리픽스 캐싱이라고 부르는 편이
더 정확합니다. KV 캐시 계산은 앞선 모든 토큰에만 의존합니다. 따라서 프리픽스가
같은 서로 다른 요청은 프리픽스 토큰의 같은 캐시를 재사용해 다시 계산하지 않아도
됩니다.

## 캐시 적중을 최대화하도록 프롬프트를 구성하는 방법 {#how-to-structure-prompts-for-maximum-cache-hits}

프리픽스 캐싱은 프롬프트가 일관될 때만 도움이 됩니다. 캐시 적중률을 최대화하는
모범 사례는 다음과 같습니다:

- **고정된 내용을 앞에 배치**: 변하지 않거나 거의 바뀌지 않는 정보를 프롬프트
  앞부분에 두세요. 여러 쿼리에서 똑같이 유지되는 시스템 메시지, 컨텍스트, 지시
  사항 등이 여기에 해당합니다. 동적인 내용이나 사용자별 내용은 프롬프트 끝으로
  옮기세요.
- **비슷한 요청을 묶어 처리**: 같은 프리픽스를 공유하는 쿼리를 함께
  묶으세요(특히 여러 사용자나 에이전트를 서빙할 때). 그래야 캐시된 결과를
  효율적으로 재사용할 수 있습니다.
- **프리픽스에 동적 요소 넣지 않기**: 타임스탬프, 요청 ID 등 요청마다 달라지는
  변수를 프롬프트 앞부분에 넣지 마세요. 이런 값은 캐시 적중률을 떨어뜨립니다.
- **결정적 직렬화 사용**: 컨텍스트나 메모리의 직렬화(예: JSON)가 키 순서와 구조
  면에서 안정적인지 확인하세요. 직렬화가 비결정적이면 내용이 논리적으로 같아도
  캐시 미스가 발생합니다.
- **캐시 적중률 모니터링과 분석**: 캐시 성능을 정기적으로 검토해 최적화할 기회를
  찾으세요.

## 도입 현황과 성능 향상 {#adoption-and-performance-gains}

일부 사용 사례에서는 프리픽스 캐싱으로 연산량과 지연 시간을 10배 수준으로 줄일
수 있습니다.

- Anthropic Claude Sonnet은 긴 프롬프트에 대해 최대 90%의 비용 절감과 85%의 지연
  시간 감소를 제공하는
  [프롬프트 캐싱](https://www.anthropic.com/news/prompt-caching)을 지원합니다.
- Google Gemini는
  [캐시된 토큰을 할인](https://ai.google.dev/gemini-api/docs/caching?lang=python)하고,
  저장 비용은 별도로 청구합니다.

vLLM, SGLang, MAX 같은 프레임워크는 여러 오픈소스 LLM에 대해 프리픽스 캐싱을
지원합니다. 이런 엔진에서 프리픽스 캐싱은 KV 캐시 할당 방식과 연결되어 있으며,
대개 기본으로 켜져 있습니다. 실제로 제어하는 것은 캐싱을 끌지 여부와, 캐시된
토큰을 어떤 단위로 매칭할지입니다:

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model google/gemma-3-27b-it \
  --enable-prefix-caching \
  --kv-cache-page-size 256
```

MAX에서는 프리픽스 캐싱이 기본으로 켜져 있습니다.
`--no-enable-prefix-caching`으로 끌 수 있습니다. 페이지 크기는 128의 배수여야
합니다.

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model google/gemma-3-27b-it \
  --enable-prefix-caching \
  --block-size 16
```

vLLM에서는 `--no-enable-prefix-caching`으로 끌 수 있습니다.

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path google/gemma-3-27b-it
```

SGLang은 프리픽스 캐싱을 KV 캐시 위의 radix tree인
[RadixAttention](https://arxiv.org/pdf/2312.07104)으로 구현하며, 기본으로 켜
둡니다. 고정 크기 블록 단위가 아니라 토큰 단위로 매칭합니다(`--page-size`
기본값은 `1`). `--disable-radix-cache`로 끌 수 있습니다.

</TabItem>
</Tabs>

:::note
프리픽스 캐싱을 끄는 것은 대개 성능 최적화가 아닙니다. 주로 캐시를 쓰지 않는
기준선을 잡을 때 유용합니다.
:::

에이전트 워크플로에서는 효과가 더욱 두드러집니다. 일부 사용 사례는 입력 대 출력
토큰 비율이 100:1에 달합니다. 그래서 큰 프롬프트를 다시 처리하는 비용이 지나치게
커집니다.

## 한계 {#limitations}

길고 반복적인 프롬프트를 쓰는 애플리케이션에서는 프리픽스 캐싱이 지연 시간과
비용을 모두 크게 줄일 수 있습니다. 하지만 시간이 지나면 KV 캐시 크기가 상당히
커질 수 있습니다. GPU 메모리는 한정되어 있고, 많은 사용자의 긴 프리픽스를
저장하면 공간이 빠르게 줄어듭니다. 캐시 축출(eviction) 전략이나 메모리
계층화(memory tiering)가 필요합니다.

오픈소스 커뮤니티는 분산 서빙 전략을 활발히 개발하고 있습니다. 자세한 내용은
[추론 라우팅](/inference-optimization/inference-routing/)을 참고하세요.

또 다른 실질적인 한계는 기능 조합입니다. 모델에 표준 전체 어텐션(full attention)
KV 캐시가 하나만 있을 때는 프리픽스 캐싱을 이해하기 쉽습니다. 최신 서빙 스택은
캐시와 비슷한 여러 상태를 한꺼번에 관리해야 할 수 있습니다.
[추측 디코딩](/inference-optimization/speculative-decoding/)을 위한 드래프트
모델과 타깃 모델의 캐시, VLM의 이미지 인코더 상태, 양자화된 KV 캐시의 스케일링
메타데이터, 하이브리드 어텐션 레이어용 별도 캐시 등이 그 예입니다.

이런 모델에서는 텍스트 프리픽스가 같다고 해서 캐시된 모든 상태를 같은 방식으로
재사용할 수 있는 것은 아닙니다. 예를 들어 슬라이딩 윈도우 어텐션(sliding-window
attention)은 최근의 제한된 윈도우만 유지하므로, 캐시 관리자는 어떤 토큰이 아직
유효한지 알아야 합니다. 프로덕션에서는 프리픽스 캐시 적중률을 하나의 전역 수치가
아니라 워크로드별 지표로 다루세요. 또한 사용하는 추론 프레임워크가 프리픽스
캐싱을 함께 켠 다른 최적화와 조합할 수 있는지 확인하세요.

<ContactCallout>

LLM 프리픽스 캐싱을 최적화하려면 LLM 서빙과 인프라 스택을 유연하게
커스터마이징할 수 있어야 합니다. 빠른 오토스케일링과 스케일 투
제로(scale-to-zero) 기능을 갖춘 효율적인 LLM 배포를 도와드릴 수 있습니다.

</ContactCallout>

<LinkList>

## 추가 자료 {#additional-resources}

- [Prompt Cache: Modular Attention Reuse for Low-Latency Inference](https://arxiv.org/abs/2311.04934)
- [Prompt Caching in Claude](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Design Around the KV-Cache](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [The Five Eras of KVCache](https://www.modular.com/blog/the-five-eras-of-kvcache?utm_source=llm_handbook)
</LinkList>
