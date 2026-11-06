---
sidebar_position: 6
description: LLM 추론을 위한 프롬프트 엔지니어링을 이해합니다. 시스템 프롬프트와 사용자 프롬프트, 제로샷과 퓨샷 프롬프팅, KV 캐시에 미치는 영향, 토큰 비용, 프로덕션 모범 사례를 알아봅니다.
keywords:
    - Prompt engineering
    - LLM prompting
    - Prompt design
---

import LinkList from '@site/src/components/LinkList';

# 프롬프트 엔지니어링

프롬프트 엔지니어링(prompt engineering)은 LLM을 다룰 때 가장 실용적인 기법 중
하나입니다. 모델 자체를 수정하지 않고도 모델의 동작을 유도하고, 출력 품질을
높이고, 신뢰할 수 있는 애플리케이션을 만들 수 있습니다.

## 프롬프트 엔지니어링이란? {#what-is-prompt-engineering}

프롬프트는 AI 모델에 보내는 입력일 뿐입니다. 모델은 추론하고 텍스트를 생성할 수
있지만, 다음 내용은 프롬프트에 의존해 파악합니다.

- 어떤 작업을 수행해야 하는지
- 답변이 어떤 형식을 따라야 하는지
- 어떤 제약 조건을 지켜야 하는지

프롬프트 엔지니어링은 지시문, 예시, 컨텍스트를 구조화해 모델이 원하는 결과를
내도록 유도하는 작업입니다. 많은 추론 워크로드에서 더 나은 결과를 얻는 가장
간단한 방법은 더 나은 프롬프트입니다.

예를 들어 다음 두 프롬프트는 크게 다른 결과를 냅니다.

```bash
# Basic prompt
Summarize this article.

# Improved prompt
Summarize the article in three bullet points.
Focus on key technical insights.
Don't include opinions or marketing language.
```

두 번째 프롬프트는 모델에 기대하는 바를 명확히 전달하므로 더 의미 있는 출력으로
이어집니다.

## 추론에서 프롬프트 엔지니어링이 중요한 이유 {#why-prompt-engineering-matters-for-inference}

LLM 프롬프트 엔지니어링은 추론의 작동 방식과 밀접하게 연결되어 있습니다.

추론 중에 모델은 가중치를 바꾸지 않습니다. 받은 입력을 바탕으로 토큰을 생성할
뿐입니다. 따라서 프롬프트는 애플리케이션과 모델을 잇는 주된 인터페이스가 됩니다.
프롬프트를 어떻게 설계하느냐가 런타임에 모델이 동작하는 방식에 직접 영향을
줍니다.

### 토큰 사용량과 비용 {#token-usage-and-cost}

모든 요청에는 입력 토큰(프롬프트)과 출력 토큰(생성된 응답)이 포함됩니다.
프롬프트가 길어지면 모델이 처리해야 하는 토큰 수가 늘어납니다. 이는 다음에
영향을 줍니다.

- **지연 시간**: 토큰이 많을수록 프리필 단계에서 연산이 더 많이 필요합니다
- **GPU 사용률**: 토큰이 많을수록 연산과 메모리 대역폭이 더 많이 필요합니다
- **추론 비용**: 토큰 기반 요금이나 인프라 비용은 토큰 사용량에 비례해
  늘어납니다

처리량이 높은 추론 시스템에서는 프롬프트 길이가 중요한 최적화 요소가 됩니다.

### KV 캐시 사용량 {#kv-cache-usage}

LLM 추론은 생성 속도를 높이기 위해
[KV 캐시(Key-Value cache)](/llm-inference-basics/how-does-llm-inference-work/#prefill)에
크게 의존합니다.

프리필(prefill) 단계에서 모델은 프롬프트 전체를 처리하고 중간 어텐션 상태를 KV
캐시에 저장합니다. 디코딩(decoding) 단계에서는 이전 토큰의 어텐션을 다시
계산하는 대신 이 캐시를 재사용합니다.

그런데 KV 캐시의 크기는 프롬프트의 토큰 수에 따라 커집니다. 따라서 프롬프트가
길어지면 다음과 같은 결과가 생깁니다.

- KV 캐시 메모리 사용량 증가
- GPU VRAM 소비 증가
- GPU당 동시 요청 수 감소

대형 모델이나 긴 컨텍스트 워크로드에서는 프롬프트 크기가 처리량과 시스템
확장성에 직접 영향을 줄 수 있습니다.

### 모델 신뢰성 {#model-reliability}

프로덕션 추론 시스템에서는 수천에서 수백만 건의 요청에 걸쳐 모델이 일관되게
동작해야 합니다.

구조가 부실한 프롬프트는 다음과 같은 문제를 일으킬 수 있습니다.

- 일관성 없는 출력
- 잘못된 형식
- 예상치 못한 추론 경로

신중하게 설계한 프롬프트는 이런 위험을 줄입니다. 작업, 출력 형식, 제약 조건을
명확히 정의하면 추론 중 모델 동작이 더 예측 가능해집니다.

이는 LLM을 자동화된 워크플로, API, 에이전트 시스템 안에서 사용할 때 특히
중요합니다.

### 런타임 제어 레이어 {#runtime-control-layer}

많은 AI 배포 환경에서 프롬프트 엔지니어링은 모델을 제어하는 첫 번째 레이어가
됩니다. 개발자는 모델 자체를 수정하는 대신 프롬프트를 조정해 다음과 같은 일을
합니다.

- 출력 구조 강제(JSON, 마크다운 등)
- 어조나 스타일 제어
- 동작 제한
- 추론 유도

프롬프트는 요청 페이로드의 일부이므로 모델을 다시 학습하거나 서빙 인프라를
바꾸지 않고도 빠르게 수정할 수 있습니다. 그래서 프롬프트는 프로덕션에서 모델
동작을 조정하는 유연한 수단이 됩니다.

프롬프트 엔지니어링은 흔히
[추론 매개변수](/model-interaction/inference-parameters/)와 함께 조정합니다.
프롬프트는 작업과 제약 조건을 정의하고, `temperature`, `top_p`, `max_tokens`
같은 매개변수는 모델이 샘플링하는 방식과 응답의 범위를 제어합니다.

## 프롬프트 역할 {#prompt-roles}

최신 LLM API 대부분은 프롬프트를 여러 역할로 나눕니다. 가장 흔한 유형은 다음과
같습니다.

- 시스템 프롬프트
- 사용자 프롬프트
- 어시스턴트 메시지

### 시스템 프롬프트 {#system-prompt}

시스템 프롬프트는 모델의 전반적인 동작을 정의합니다. 모델이 어떻게 응답해야
하는지 안내하는 운영 지침 역할을 합니다.

예시:

```bash
You are a technical assistant that explains LLM inference concepts clearly.
Use short paragraphs and simple language.
Avoid unnecessary jargon.
```

개발자는 보통 다음 목적으로 시스템 프롬프트를 사용합니다.

- 어시스턴트의 역할이나 페르소나 정의
- 응답 형식(예: JSON) 강제
- 주제나 동작 제한
- 작업별 지시 제공

채팅 인터페이스를 쓰는 개인 사용자에게 시스템 프롬프트는 대개 보이지 않는 곳에
있습니다. 애플리케이션 개발자가 설정하며, 모든 사용자에게 일관된 동작을 보장하는
데 도움을 줍니다.

### 사용자 프롬프트 {#user-prompt}

사용자 프롬프트에는 사용자나 애플리케이션이 보내는 실제 요청이 담깁니다.

예시:

```bash
Explain what a transformer model is in simple terms.
```

개인 사용자에게 사용자 프롬프트는 채팅 인터페이스에 입력하는 질문이나 지시일
뿐입니다.

개발자 입장에서 사용자 프롬프트는 애플리케이션이 동적으로 생성하는 경우가
많습니다. 여기에는 다음이 포함될 수 있습니다.

- 사용자 입력
- 데이터베이스에서 검색한 문서
- 대화 컨텍스트
- 구조화된 작업 지시

예를 들어 문서 질의응답 시스템에서는 사용자 프롬프트가 다음과 같을 수 있습니다.

```bash
Answer the question using the context below.

Context:
{retrieved_documents}

Question:
{user_question}
```

프로덕션 추론 시스템에서 사용자 프롬프트는 대개 모델에 보내기 전에 프로그래밍
방식으로 조립합니다.

### 프롬프트 템플릿 {#prompt-templates}

프로덕션 시스템에서는 매번 프롬프트를 직접 작성하는 경우가 드뭅니다. 대신
개발자는 동적 데이터를 끼워 넣는 프롬프트 템플릿을 사용합니다.

템플릿 예시:

```bash
You are a helpful assistant.

User question:
{question}

Provide a concise answer in three bullet points.
```

프롬프트 템플릿을 쓰면 다음 작업이 쉬워집니다.

- 애플리케이션 전반에서 프롬프트 표준화
- 일관성 유지
- 한 곳에서 동작 수정

LangChain, LlamaIndex 같은 프레임워크와 많은 추론 플랫폼이 템플릿 프롬프트를
지원합니다.

### 어시스턴트 메시지 {#assistant-messages}

어시스턴트 메시지는 대화의 이전 턴에서 모델이 생성한 응답을 나타냅니다. 주로 두
가지 용도로 쓰입니다.

- 이전 모델 출력
- 개발자가 이후 응답을 유도하거나 제약하는 수단

#### API가 반환하는 메시지 {#returned-by-the-api}

대부분의 채팅 API에서 모델은 `assistant` 역할의 메시지를 반환합니다. 멀티턴
대화를 구성할 때 애플리케이션은 보통 이전 메시지를 다음 요청에 포함합니다.

예시:

```bash
messages = [
  {"role": "system", "content": "You are a helpful assistant."},
  {"role": "user", "content": "Who won the world series in 2020?"},
  {"role": "assistant", "content": "The Los Angeles Dodgers won the World Series in 2020."},
  {"role": "user", "content": "Where was it played?"}
]
```

이전 메시지를 포함하면 모델이 대화의 흐름을 일관되게 유지할 수 있습니다.

하지만 모델 입장에서 이 메시지들은 특별히 취급되지 않습니다. 추론 중에는 전체
대화 기록이 하나의 토큰 시퀀스로 변환되어 입력 프롬프트의 일부로 처리될
뿐입니다.

대규모 추론 시스템에서는 대화가 길어지면 처리량이 줄고 인프라 비용이 늘어날 수
있습니다. 이를 관리하기 위해 AI 팀은 흔히 다음과 같은 전략을 적용합니다.

- 가장 최근 턴만 남기는 메시지 잘라내기
- 오래된 컨텍스트를 압축하는 대화 요약
- 공유 프롬프트 구간을 재사용하는
  [프리픽스 캐싱](/inference-optimization/prefix-caching/)
- 필요할 때 캐시 데이터를 저렴한 저장소로 옮기는
  [KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/)

#### 개발자가 삽입하는 메시지 {#injected-by-developers}

어시스턴트 메시지가 꼭 모델에서 나와야 하는 것은 아닙니다. 개발자가 합성
어시스턴트 메시지를 직접 삽입해 모델의 동작을 유도할 수도 있습니다.

이 기법을 **어시스턴트 프리필링**(assistant prefilling) 또는
**응답 프리픽스 지정**(response prefixing)이라고 부르기도 합니다.

LLM은 자기회귀(autoregressive) 모델이므로(이전의 모든 토큰을 바탕으로 다음
토큰을 예측함) "Assistant" 역할에 들어 있는 단어를 모델이 직접 생성했는지,
사람이 직접 입력했는지 구분하지 않습니다. 모델은 그 단어들을 이미 정해진 시작
부분으로 보고 거기서부터 이어서 생성합니다.

이를 이용하면 개발자는 응답을 유도하거나 구조를 강제할 수 있습니다. 예를 들어
모델이 올바른 JSON을 출력하도록 유도할 수 있습니다.

```bash
messages = [
  {"role": "user", "content": "Show me a list of popular open-source llms in JSON."},
  {"role": "assistant", "content": "Here is the JSON list you need:\n{"}, # The model continues from here
]
```

:::note
최신 채팅형 LLM API 대부분은 어시스턴트 프리필링을 지원합니다. 다만 정확한 API
구조와 이 동작의 신뢰성은 공급자와 모델에 따라 다를 수 있습니다.
:::

## 더 나은 프롬프트를 위한 실용적인 팁 {#practical-tips-for-better-prompts}

몇 가지 간단한 습관만으로도 결과를 크게 개선할 수 있습니다.

### 명시적으로 작성하기 {#be-explicit}

원하는 것을 모델에 정확히 알려 주세요.

나쁜 예:

```text
Explain this.
```

더 나은 예:

```text
Explain this concept to a beginner using simple language and one example.
```

### 출력 형식 지정하기 {#specify-output-format}

애플리케이션이 구조화된 출력을 기대한다면 그렇다고 명시하세요.

예시:

```bash
Return the answer in JSON format with fields:
- summary
- key_points
- confidence
```

### 프롬프트의 초점 유지하기 {#keep-prompts-focused}

관련 없는 지시가 섞인 긴 프롬프트는 모델을 혼란스럽게 할 수 있습니다. 지시는
명확하고 작업에 맞게 작성하세요.

### 여러 변형 시험하기 {#test-variations}

표현을 조금만 바꿔도 결과가 크게 달라질 때가 있습니다. 프롬프트 엔지니어링에는
흔히 실험이 필요합니다.

---

실제 시스템을 구축하다 보면 LLM 프롬프트 엔지니어링을
[구조화된 출력](/model-interaction/structured-outputs/),
[도구 호출](/model-interaction/function-calling/),
[미세조정](/model-preparation/llm-fine-tuning/) 같은 다른 기법과 함께 사용하게
될 가능성이 높습니다. 프롬프트가 모델 동작에 어떤 영향을 주는지 이해하는 것이
견고한 LLM 애플리케이션을 만드는 첫걸음입니다.

## 자주 묻는 질문 {#faqs}

### 제로샷, 원샷, 퓨샷 프롬프팅이란? {#what-are-zero-shot-one-shot-and-few-shot-prompting}

이 용어들은 모델을 안내하기 위해 프롬프트에 예시를 몇 개 포함하는지를
나타냅니다.

제로샷 프롬프팅(zero-shot prompting)은 프롬프트에 예시 없이 지시만 담는
방식입니다.

```bash
Classify the sentiment of the following sentence as Positive or Negative.

Sentence: I really enjoyed this movie.
```

모델은 지시만 보고 작업을 수행해야 합니다.

원샷 프롬프팅(one-shot prompting)은 기대하는 동작을 보여 주는 예시를 하나
사용합니다.

```bash
Classify the sentiment of the following sentences.

Sentence: This restaurant is amazing.
Sentiment: Positive

Sentence: The service was terrible.
Sentiment:
```

예시는 모델이 따라야 할 패턴을 이해하는 데 도움을 줍니다.

퓨샷 프롬프팅(few-shot prompting)은 실제 입력 앞에 여러 예시를 제공합니다.
예시가 여러 개 있으면 모델이 패턴을 더 잘 파악하고 더 신뢰할 수 있는 결과를 낼
수 있습니다.

```bash
Classify the sentiment of the following sentences.

Sentence: This restaurant is amazing.
Sentiment: Positive

Sentence: The service was terrible.
Sentiment: Negative

Sentence: I really enjoyed the meal.
Sentiment:
```

다음은 세 방식을 나란히 비교한 표입니다.

| 유형                | 의미                                                      | 사용 예                                  | 장단점                                                                                   |
|---------------------|-----------------------------------------------------------|------------------------------------------|------------------------------------------------------------------------------------------|
| **제로샷 프롬프팅** | 프롬프트에 예시 없이 지시만 담습니다.                     | 요약이나 번역 같은 일반적인 작업.        | 프롬프트가 짧고 토큰 비용이 낮지만, 모델이 형식이나 기대 사항을 잘못 이해할 수 있습니다. |
| **원샷 프롬프팅**   | 원하는 입력-출력 패턴을 보여 주는 예시를 하나 포함합니다. | 모델에 특정 형식이나 스타일을 가르칠 때. | 프롬프트가 조금 길어지지만 작업을 명확히 하는 데 도움이 됩니다.                          |
| **퓨샷 프롬프팅**   | 실제 입력 앞에 여러 예시를 포함합니다.                    | 분류, 구조화된 출력, 도메인 특화 형식.   | 흔히 신뢰성이 높아지지만 토큰 사용량과 추론 비용이 늘어납니다.                           |

### 프롬프트 엔지니어링과 미세조정의 차이는? {#what-are-the-differences-between-prompt-engineering-and-fine-tuning}

프롬프트 엔지니어링과 미세조정은 둘 다 모델 출력을 개선하려는 방법이지만 작동
방식은 매우 다릅니다.

| 접근 방식           | 무엇을 바꾸는가                        | 언제 사용하는가                 |
|---------------------|----------------------------------------|---------------------------------|
| 프롬프트 엔지니어링 | 입력 프롬프트의 지시를 조정합니다      | 빠른 반복, 형식 제어, 작업 안내 |
| 미세조정            | 학습 데이터로 모델 가중치를 갱신합니다 | 도메인 지식, 특화된 작업        |

보통 프롬프트 엔지니어링이 첫 단계입니다. 빠르고 유연하며 학습 인프라가 필요하지
않습니다. 미세조정은 다음과 같은 경우에 유용해집니다.

- 프롬프트만으로는 신뢰할 수 있는 출력을 얻을 수 없을 때
- 모델에 없는 도메인 지식이 작업에 필요할 때
- 프롬프트가 너무 길거나 복잡해질 때
- 일관성 요구 사항이 매우 엄격할 때

많은 프로덕션 시스템은 두 기법을 함께 사용합니다.

1. 프롬프트 엔지니어링으로 시작합니다.
2. 필요하면 퓨샷 예시를 추가합니다.
3. 프롬프트 개선이 한계에 이르면 모델을 미세조정합니다.

### 사고 과정(CoT) 프롬프팅이란? {#what-is-chain-of-thought-cot-prompting}

사고 과정(Chain-of-Thought, CoT) 프롬프팅은 LLM이 최종 답을 내기 전에 중간 추론
단계를 생성하도록 유도하는 프롬프트 엔지니어링 기법입니다.

이 기법은 [Jason Wei et al. (2022)](https://arxiv.org/abs/2201.11903)의 논문
_Chain-of-Thought Prompting Elicits Reasoning in Large Language Models_ 에서
처음 소개되었습니다. 저자들은 모델이 중간 추론 단계를 생성하도록 프롬프팅하면
수학 문제, 추론, 논리적 추론 같은 작업에서 성능이 크게 향상된다는 것을
보였습니다.

<figure>
  <img src={require('@site/docs/model-interaction/img/chain-of-thought-cot-prompting-example.png').default} alt="" />
  <figcaption><b>그림 1.</b> 표준 프롬프팅과 사고 과정 프롬프팅 비교. 출처: <cite><a
  href="https://arxiv.org/abs/2201.11903">Chain-of-Thought Prompting Elicits
  Reasoning in Large Language Models</a></cite>.</figcaption>
</figure>

하지만 추론 시스템 입장에서는 몇 가지 트레이드오프가 있습니다.

- 추론 단계 때문에 출력 토큰 길이가 늘어납니다
- 출력이 길어지면 지연 시간과 추론 비용이 늘어납니다
- 최종 답만 필요하다면 추론 텍스트를 걸러 내야 할 수 있습니다

그래서 일부 프로덕션 시스템은 추론을 내부에서 생성하고 최종 결과만 사용자에게
반환합니다.

### 프롬프트 엔지니어링은 언제 한계에 부딪히는가? {#when-does-prompt-engineering-fail}

프롬프트 엔지니어링은 강력하지만 한계가 있습니다. 모델이 갖고 있지 않은 도메인
지식이 작업에 필요하거나, 수백만 건의 요청에 걸쳐 엄격한 일관성이 필요하다면
프롬프트 엔지니어링만으로는 충분하지 않을 수 있습니다. 이런 경우에는
[미세조정](/model-preparation/llm-fine-tuning/), RAG,
[제약 디코딩](/model-interaction/structured-outputs/#constrained-decoding) 같은
기법이 필요할 수 있습니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Automatic Chain of Thought Prompting in Large Language Models](https://arxiv.org/abs/2210.03493)
</LinkList>
