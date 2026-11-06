---
sidebar_position: 3
description: 함수 호출이 무엇이며 어디에 쓰이는지 알아봅니다.
keywords:
    - Function calling, function calling APIs
    - LLM tool use, LLM tool integration
---

# 함수 호출(function calling)

함수 호출(function calling)을 사용하면 작업에 외부 데이터가 필요하거나 모델
바깥의 동작이 필요할 때 LLM이 특정 도구를 요청할 수 있습니다. 이런 도구가 필요한
일을 모델에게 요청하면, 모델은 그 도구를 호출하고 결과를 응답에 사용할 수
있습니다.

<figure>
  <Diagram name="function-calling-diagram" alt="" />
  <figcaption><b>그림 1.</b> LLM이 함수 호출을 내보내면 외부 API로 전달됩니다.</figcaption>
</figure>

구체적인 예를 들면 다음과 같습니다.

- **사용자 질문**: "지금 Apple 주가가 얼마야?"
- **LLM의 판단**: "현재 주가 데이터가 필요하니 주가 조회 함수를 써야겠다"
- **LLM의 호출**: `get_stock_price("AAPL")`
- **함수 반환값**: "$195.25"
- **LLM의 응답**: "현재 Apple 주가는 $195.25입니다"

기술적으로 보면 LLM은 다른 트랜스포머 기반 모델과 마찬가지로 여전히 다음 토큰을
예측할 뿐입니다. 프롬프트에 명확하게 정의된 함수 시그니처와 설명이 들어 있어서,
모델이 그 형식에 맞는 출력을 생성하도록 유도됩니다. 그래서 다음과 같은
워크플로를 만들 수 있습니다.

- 사용자 입력을 바탕으로 API 호출
- 동작 실행(예: 이메일 전송, 날씨 조회)
- 멀티턴 대화를 위해 출력을 다시 모델에 전달

## 자주 묻는 질문 {#faqs}

### 함수 호출과 구조화된 출력은 어떻게 다른가요? {#what-is-the-difference-between-function-calling-and-structured-outputs}

두 개념은 자주 혼동됩니다. 간단히 차이를 정리하면 다음과 같습니다.

- 구조화된 출력:
  [모델이 응답에 사용할 형식](/model-interaction/structured-outputs/)을 사용자가
  정합니다. 예를 들어 모델이 JSON, 목록, 특정 객체 형태로만 출력하도록 강제할 수
  있습니다.
- 함수 호출: 모델에게 언제 동작을 수행할지 알려 줍니다. 함수 호출용 도구는 보통
  구조화된 방식(예: JSON)으로 정의됩니다.

두 기능을 함께 사용할 수도 있습니다.

### 모든 LLM이 함수 호출을 지원하나요? {#do-all-llms-support-function-calling}

모두는 아니지만 최신 모델 중 다수가 지원합니다. Llama, Qwen, DeepSeek 같은
오픈소스 모델은 vLLM, SGLang, MAX로 서빙하면 대체로 잘 동작합니다.

### 함수 호출은 도구나 에이전트와 같은 것인가요? {#is-function-calling-the-same-as-tools-or-agents}

도구는 동작 그 자체입니다. 함수 호출은 모델이 그 동작을 요청하는 방식입니다.
에이전트는 추론과 함께 함수 호출을 반복적으로 사용할 수 있는 시스템입니다.
에이전트는 계획을 세우고, 여러 도구를 순서대로 호출하고, 결과를 평가하고, 알게
된 내용을 바탕으로 접근 방식을 조정할 수 있습니다.
