---
sidebar_position: 4
description: 구조화된 출력은 JSON이나 XML처럼 정해진 형식을 따르는 모델 응답으로, AI가 생성한 데이터를 예측 가능하고 기계가 읽을 수 있게 만들어 애플리케이션과 워크플로에 쉽게 통합할 수 있게 해 줍니다.
keywords:
    - Structured outputs, structured generation, constrained decoding
---

import LinkList from '@site/src/components/LinkList';

# 구조화된 출력(structured outputs)

구조화된 출력(structured output)은 JSON, XML, 정규식으로 정의한 패턴처럼 기계가
읽을 수 있는 특정 형식을 따르는 LLM 응답입니다. 모델은 자유 형식의 글 대신,
다운스트림 시스템이 구문 분석해 바로 사용할 수 있는 데이터를 생성합니다.

예시는 다음과 같습니다.

```json
{
  "name": "LLM Inference Handbook",
  "author": "Modular",
  "website": "handbook.modular.com",
  "summary": "A practical handbook for engineers building, optimizing, scaling, and operating LLM inference systems in production."
}
```

## 구조화된 출력은 왜 중요할까? {#why-do-structured-outputs-matter}

LLM으로 작업하면 출력은 대개 자유 형식의 텍스트입니다. 사람은 이런 응답을 쉽게
읽고 해석할 수 있습니다.

하지만 LLM으로 더 큰 애플리케이션을 만든다면(예: 모델의 응답을 다른 서비스, API,
데이터베이스와 연결하는 애플리케이션) 예측 가능한 구조가 필요합니다. 구조가
없으면 프로그램이 무엇을 추출해야 하는지, 어떤 필드를 어디에 넣어야 하는지
어떻게 알 수 있을까요?

바로 이때 구조화된 출력이 필요합니다. 구조화된 출력은 모델이 따를 명확하고
기계가 읽을 수 있는 형식을 제공하므로, 자동화와 통합의 신뢰성을 높여 줍니다.

예를 들어 지원 티켓을 읽고 제품 팀을 위해 인사이트를 요약하는 분석 어시스턴트를
만든다고 해 봅시다. LLM이 다음을 반환하기를 원합니다.

- 사용자가 언급한 주요 이슈
- 각 이슈의 빈도
- 전반적인 감성 점수

모델이 다음처럼 일반 텍스트로 답한다고 해 봅시다.

> “대부분의 고객이 느린 로딩 시간과 결제 오류에 대해 불만을 제기합니다. 전반적인
> 어조는 약간 부정적입니다.”

사람이 읽기에는 괜찮지만, 자동화된 대시보드에는 거의 쓸모가 없습니다. 이런
인사이트를 직접 추출하거나 복잡한 구문 분석 코드를 작성해야 합니다.

이제 다음의 구조화된 출력과 비교해 보세요.

```json
{
  "issues": [
    {"topic": "Slow loading", "count": 42},
    {"topic": "Payment errors", "count": 31}
  ],
  "sentiment": "negative",
  "confidence": 0.87
}
```

시스템은 이 출력을 바로 구문 분석해 데이터베이스에 저장하고 대시보드에 시각화할
수 있습니다. 추측이나 후처리가 필요하지 않습니다.

구조화된 출력은 이제 다음을 비롯한 많은 실제 LLM 시스템에서 흔히 쓰입니다.

- **정보 추출**: 문서에서 개체, 숫자, 관계를 추출해 JSON이나 표로 만듭니다.
- **데이터 보강**: CRM이나 분석 파이프라인을 위해 레코드를 분류하고, 태그를
  달고, 요약합니다.
- **함수 호출과 API 체이닝**: LLM이 호출할 도구나 엔드포인트를 고르고 매개변수를
  구조화된 방식으로 전달하게 합니다.
- **에이전트 오케스트레이션**: 각 단계가 이전 단계의 출력을 입력으로 받는 다단계
  워크플로를 조율합니다.
- **평가와 테스트**: 모델의 품질과 정확도를 벤치마킹하기 위해 일관된 응답을
  수집합니다.
- **콘텐츠 검토나 규정 준수 검사**:
  `{ "action": "flag", "reason": "PII detected" }` 같은 구조화된 판정을
  반환합니다.

## 구조화된 출력을 얻는 방법 {#how-to-obtain-structured-outputs}

구조화된 출력이 왜 중요한지 이해했으니, 다음 질문은 실제로 어떻게 얻느냐입니다.

물론 모델의 응답에서 필요한 데이터를 정리하고 추출하는 맞춤형 구문 분석 로직을
직접 작성할 수도 있습니다. 하지만 이 방식은 금세 지저분해집니다. 시간이 많이
들고, 오류가 나기 쉬우며, 규모가 커지면 쉽게 깨집니다. 새로운 형식이나 규칙이
생길 때마다 코드가 더 복잡해집니다.

더 나은 방법은 LLM이 직접 구조화된 데이터를 생성하게 하는 것입니다. 하지만 모든
LLM이 구조화된 출력을 기본으로 지원하지는 않습니다. 이런 모델은 대개 신중한
프롬프팅이나 특정 프레임워크의 스키마 정의로 유도해야 합니다. 형식을 명확하게
작성하면(예: 명시적인 예시나 정규식 사용) 모델이 기대한 구조를 따르는 출력을
안정적으로 생성할 수 있습니다.

현재 구조화된 출력을 얻는 주요 방법은 세 가지입니다.

### 서버리스 모델 API 공급자 {#serverless-model-api-providers}

구조화된 출력을 시작하는 가장 쉬운 방법은 이를 기본 지원하는 모델 API를 호출하는
것입니다.

OpenAI, Anthropic, Google 같은 공급자는 API 호출에서 스키마나 JSON 구조를 직접
지정할 수 있게 해 줍니다. 그러면 모델이 이 스키마를 따르는 응답을 자동으로
생성합니다.

이 기능은 초기의
[JSON mode](https://platform.openai.com/docs/guides/structured-outputs#json-mode)에서
발전했습니다. JSON mode는 모델에게 JSON 형식으로 답하라고 요청하는 단순한
방식이었습니다. 동작은 했지만 일관성이 없었습니다. 모델이 형식이 잘못되었거나
불완전한 JSON을 자주 생성했습니다. 이 문제를 해결하려고 OpenAI는
[Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)를
도입했습니다. Structured Outputs는 스키마를 강제해 응답이 항상 정의된 구조와
일치하게 만드는 더 엄격한 방식입니다.

다음은 OpenAI의 구조화된 출력 API를 사용한 예시입니다.

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()

# Define the expected fields with Pydantic
class SupportSummary(BaseModel):
    issues: list[str]
    sentiment: str
    confidence: float

completion = client.chat.completions.parse(
    model="gpt-5",
    messages=[
        {"role": "system", "content": "Summarize support ticket feedback."},
        {"role": "user", "content": "The app is terrible! It crashes every time it opens."},
    ],
    response_format=SupportSummary,
)

event = completion.choices[0].message.parsed
```

서드파티 모델 API로 구조화된 출력을 얻는 방법은 간단합니다. 유지보수할 맞춤형
구문 분석 로직이 없고, 모델이 스키마 검증을 대신 처리합니다.

하지만 몇 가지 트레이드오프가 있습니다.

- **벤더 종속**: 특정 공급자의 API에 묶입니다.
- **출력 제한**: 페이로드가 크면 잘려서 JSON이 불완전해질 수 있습니다.
- **일관되지 않은 강제**: 모든 공급자가 스키마 검증을 똑같이 잘 처리하지는
  않습니다.

### 재프롬프팅 {#re-prompting}

재프롬프팅(re-prompting)은 [Instructor](https://github.com/567-labs/instructor)
같은 라이브러리를 사용해 구조화된 출력을 얻는 단순하고 효과적인 방법입니다.

동작 방식은 다음과 같습니다.

1. 원하는 형식(예: JSON 스키마)을 설명하는
   [프롬프트](/model-interaction/prompt-engineering/)를 모델에 보냅니다.
2. 라이브러리가 응답이 유효한지 확인합니다.
3. 유효하지 않으면 무엇이 잘못되었는지에 대한 상세 정보와 함께 모델에 자동으로
   다시 프롬프트를 보냅니다.
4. 출력이 검증을 통과하거나 재시도 한도에 도달할 때까지 이 과정을 반복합니다.

이 루프 덕분에 재시도 로직을 직접 작성하지 않아도 결국 유효한 구조화된 출력을
얻을 수 있습니다.

또한 매우 유연합니다. 맞춤형 정규식 규칙을 정의하고 날짜나 숫자 형식을 강제할 수
있습니다. 사실상 어떤 모델이나 API 공급자와도 함께 쓸 수 있습니다.

트레이드오프는 지연 시간과 비용입니다. 재시도할 때마다 모델을 한 번 더
호출하므로 시간과 토큰이 늘어납니다. 스키마가 복잡하거나 모델이 지시를 잘 따르지
못하면 여러 번 재시도해야 할 수 있고, 모든 시도를 소진한 뒤에도 실패할 수
있습니다.

### 제약 디코딩 {#constrained-decoding}

오픈소스 LLM을 셀프 호스팅한다면, 구조화된 생성(structured generation)이라고도
하는 제약 디코딩(constrained decoding)이 구조화된 출력을 생성하는 가장 신뢰할
만한 방법 가운데 하나입니다.

이 방법은 출력이 생성된 뒤에 검증하는 대신, 토큰을 생성하는 동안 구조를
강제합니다. 모델이 사용자가 정의한 형식이나 스키마에 맞는 토큰만 샘플링할 수
있도록 보장합니다.

내부에서 일어나는 일은 다음과 같습니다.

LLM은 텍스트를 생성할 때 가능한 다음 토큰 각각의 확률을 예측합니다(이 확률값을
*로짓*이라고 합니다). 제약 디코딩에서는 이 로짓을 실시간으로 수정해, 정의된
구조를 위반하는 토큰을 모두 제거합니다. 그러면 모델은 유효하게 이어지는 토큰만
생성할 수 있으므로, 최종 출력이 항상 스키마를 따르도록 보장됩니다.

이 방식은 재시도나 후처리에 의존하지 않으므로 빠릅니다. 오픈소스 모델과 잘
맞으며, [Outlines](https://github.com/outlines-dev/outlines),
[Microsoft Guidance](https://github.com/guidance-ai/llguidance),
[XGrammar](https://github.com/mlc-ai/xgrammar) 같은 라이브러리가 지원합니다.
[vLLM](https://docs.vllm.ai/en/latest/features/structured_outputs.html),
[SGLang](https://docs.sglang.ai/advanced_features/structured_outputs.html),
[MAX](https://max.modular.com/serve/structured-output/) 같은 추론 프레임워크는
이미 이런 도구를 직접 통합했습니다.

주요 장점은 속도, 정밀성, 신뢰성입니다. Outlines 팀은
[구조화된 출력이 LLM 성능을 높일 수 있다는 것](https://blog.dottxt.ai/performance-gsm8k.html)을
보여 주기도 했습니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [Efficient Guided Generation for Large Language Models](https://arxiv.org/pdf/2307.09702)
- [Fast JSON Decoding for Local LLMs with Compressed Finite State Machine](https://lmsys.org/blog/2024-02-05-compressed-fsm/)
- [XGrammar: Flexible and Efficient Structured Generation Engine for Large Language Models](https://arxiv.org/abs/2411.15100)
</LinkList>
