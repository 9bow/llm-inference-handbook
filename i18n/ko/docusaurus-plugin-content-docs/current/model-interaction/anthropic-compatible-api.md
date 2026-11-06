---
sidebar_position: 2
description: Anthropic 호환 API는 Anthropic의 Messages API를 그대로 따르므로, Claude 기반 클라이언트, SDK, 에이전트 도구가 코드를 거의 바꾸지 않고 다른 모델이나 공급자를 사용할 수 있습니다.
keywords:
    - Anthropic-compatible API, Anthropic-compatible endpoint
    - Anthropic Messages API, Claude API compatibility
    - Claude Code, Claude Agent SDK, LLM inference API
---

import LinkList from '@site/src/components/LinkList';

# Anthropic 호환 API

Anthropic API는 특히 Claude 기반 애플리케이션과 에이전트 워크플로에서 LLM을
다루는 또 하나의 주요 인터페이스가 되었습니다.

## Anthropic 호환 API란? {#what-is-an-anthropic-compatible-api}

Anthropic 호환 API는 Anthropic API의 인터페이스, 요청/응답 스키마, 인증 방식을
그대로 재현한 API를 말합니다.
[Claude Code](https://www.anthropic.com/claude-code)와
[Claude Agent SDK](https://docs.claude.com/en/api/agent-sdk/overview) 같은
에이전트 도구를 중심으로 Claude 모델이 널리 쓰이면서, 많은 애플리케이션과
프레임워크가 Anthropic Messages API 형식을 채택했습니다.

Anthropic 호환 엔드포인트를 제공하면 기존의 Anthropic 기반 클라이언트, SDK,
에이전트 루프를 거의 바꾸지 않고도 오픈소스 모델(예: Llama, Qwen, DeepSeek)이나
다른 공급자를 서빙할 수 있습니다.

## Anthropic 호환 API 호출 방법 {#how-to-call-an-anthropic-compatible-api}

공식 Anthropic SDK를 사용하고 `base_url`을 자신의 엔드포인트로 지정하세요.

```python
from anthropic import Anthropic

client = Anthropic(
    base_url="https://your-custom-endpoint.com",
    api_key="your-api-key"
)

response = client.messages.create(
    model="your-model-name",
    max_tokens=1024,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "How can I integrate Anthropic-compatible APIs?"}
    ]
)

print(response.content[0].text)
```

`curl`로 엔드포인트를 직접 호출할 수도 있습니다.

```bash
curl https://your-custom-endpoint.com/v1/messages \
  -H "x-api-key: your-api-key" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-model-name",
    "max_tokens": 1024,
    "system": "You are a helpful assistant.",
    "messages": [
      {"role": "user", "content": "How can I integrate Anthropic-compatible APIs?"}
    ]
  }'
```

### 스트리밍 응답 {#streaming-responses}

Anthropic SDK는 모델이 응답을 생성하는 동안 타입이 지정된 이벤트를 차례로
내보내는 `messages.stream()` 헬퍼를 제공합니다.

```python
from anthropic import Anthropic

client = Anthropic(
    base_url="https://your-custom-endpoint.com",
    api_key="your-api-key"
)

with client.messages.stream(
    model="your-model-name",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Write a short poem about streaming."}
    ],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

정확한 이벤트 스키마는 프레임워크마다 다를 수 있습니다. 항상 해당 프레임워크의
공식 문서를 확인하세요.

### 사용 가능한 모델 목록 조회 {#listing-available-models}

Anthropic은 `/v1/models` 엔드포인트를 제공하며, 많은 호환 서버도 이를
구현합니다. 이 엔드포인트로 백엔드가 받아들이는 `model` 이름을 확인할 수
있습니다.

```python
from anthropic import Anthropic

client = Anthropic(
    base_url="https://your-custom-endpoint.com",
    api_key="your-api-key"
)

for model in client.models.list().data:
    print(model.id)
```

`curl`로는 다음과 같이 호출합니다.

```bash
curl https://your-custom-endpoint.com/v1/models \
  -H "x-api-key: your-api-key" \
  -H "anthropic-version: 2023-06-01"
```

반환된 `id` 중 아무것이나 `messages.create()` 호출의 `model` 필드에 사용하면
됩니다.

## 유의할 점 {#things-to-keep-in-mind}

호환 엔드포인트는 Anthropic 스키마를 따르지만 공식 Anthropic API는 아닙니다.
실무에서 유의할 점은 다음과 같습니다.

- **API 키를 받기만 하고 검증하지 않을 수 있습니다**. 많은 셀프 호스팅 추론
  프레임워크는 키 값을 확인하지 않으므로 아무 문자열(예: `"EMPTY"`)이나 넘겨도
  되는 경우가 많습니다. 엔드포인트나 게이트웨이가 실제로 키를 확인한다면 진짜
  비밀 값으로 다루세요.
- **설정은 주로 환경 변수로 합니다**. 많은 프레임워크 문서는 API 키와 기본 URL을
  클라이언트 코드에 하드코딩하지 말고 환경 변수로 설정하라고 권장합니다(그러면
  Anthropic SDK가 자동으로 읽어 들입니다). 기본 방식은 어느 프레임워크나 같지만,
  구체적인 변수 이름은 다를 수 있습니다.
- **모든 API 필드가 지원되지는 않습니다**. `model`, `messages`, `max_tokens`
  같은 공통 필드는 보통 문제없지만, 그 밖의 필드는 지원 범위가 좁아집니다. 예를
  들면 다음과 같습니다.
  - **모달리티**. 공식 Anthropic API는 `"image"`, `"document"` 같은 타입을
    받습니다. 많은 오픈소스 LLM에서는 이런 타입을 아예 지원하지 않습니다. 어떤
    콘텐츠 타입이 통할 것이라고 가정하기 전에 항상 호환성 문서를 확인하세요.
  - **고급 기능**.
    [프롬프트 캐싱](/inference-optimization/prefix-caching/)(프리픽스를 캐싱하는
    `cache_control`), 확장 사고(extended thinking), 일부
    [도구 사용](/model-interaction/function-calling/) 옵션 같은 기능은
    무시되거나 거부될 수 있습니다. 이런 기능에 의존한다면 Anthropic 기반
    애플리케이션을 옮기기 전에 처음부터 끝까지 제대로 동작하는지 확인하세요.

## 언제 사용해야 할까? {#when-to-use-it}

다음과 같은 경우 Anthropic 호환 엔드포인트를 선택하세요.

- 애플리케이션이나 에이전트 스택이 이미 Anthropic API 위에 구축되어 있는
  경우(예: Claude Code, Claude Agent SDK, Anthropic 방식으로 도구를 사용하는
  커스텀 에이전트 루프)
- 하위 도구(SDK, 프록시, 평가 도구)가 Anthropic 스키마를 기대하고 있어서, 이를
  OpenAI 호환 방식으로 다시 작성하는 것보다 호환 엔드포인트를 운영하는 편이
  수고가 덜한 경우

기존 통합이 없는 새 애플리케이션이라면
[OpenAI 호환 API](/model-interaction/openai-compatible-api/)가 여전히 더 널리
지원되는 기본 선택지입니다. 기계가 읽을 수 있는 예측 가능한 응답이 주된
관심사라면, 선택한 백엔드의
[구조화된 출력](/model-interaction/structured-outputs/) 지원 여부와 API 범위도
함께 비교해 보세요.

## 자주 묻는 질문 {#faqs}

### OpenAI 호환 API와 Anthropic 호환 API 중 무엇을 골라야 하나요? {#should-i-pick-an-openai-compatible-or-anthropic-compatible-api}

모델만 보지 말고 기존 스택을 기준으로 고르세요. 클라이언트, 에이전트 프레임워크,
SDK가 이미 OpenAI 스키마를 사용한다면 OpenAI 호환 엔드포인트가 가장 쉬운
길입니다. Anthropic 스키마를 사용한다면 Anthropic 호환 엔드포인트를 쓰면 기존
통합을 다시 작성하지 않아도 됩니다. 두 엔드포인트 뒤의 모델은 같을 수 있으며,
달라지는 것은 API 형태뿐입니다.

### OpenAI API와 Anthropic API의 차이는 무엇인가요? {#what-is-the-difference-between-the-openai-api-and-the-anthropic-api}

두 API 모두 애플리케이션이 프롬프트를 보내고, 모델 응답을 받고, 출력을
스트리밍하고, 도구를 사용할 수 있게 해 주지만, 요청과 응답 스키마가 서로
다릅니다. 호환 엔드포인트는 클라이언트가 기대하는 스키마와 맞아야 합니다.

| 영역                 | OpenAI API                                                                                        | Anthropic API                                                                     |
|----------------------|---------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| 주요 채팅 엔드포인트 | 보통 `/v1/chat/completions` 또는 더 새로운 Responses API 엔드포인트                               | `/v1/messages`                                                                    |
| 클라이언트 형태      | 채팅 완성(chat completions), 응답(responses), 도구, 선택지(choices)를 중심으로 한 OpenAI SDK 규칙 | 메시지, 콘텐츠 블록, 타입이 지정된 스트림 이벤트를 중심으로 한 Anthropic SDK 규칙 |
| 시스템 프롬프트      | 보통 `system` 또는 `developer` 메시지나 이에 준하는 지시 필드로 표현                              | Messages API의 최상위 `system` 필드로 전달                                        |
| 인증 헤더            | 보통 `Authorization: Bearer ...`                                                                  | 보통 `x-api-key`와 `anthropic-version` 헤더                                       |
| 도구 사용            | OpenAI 방식의 도구 정의와 도구 호출 필드                                                          | Anthropic 방식의 도구 정의와 도구 사용 콘텐츠 블록                                |

<LinkList>

## 추가 자료 {#additional-resources}

- [Anthropic Messages API documentation](https://docs.claude.com/en/api/messages)
- [Claude Agent SDK overview](https://docs.claude.com/en/api/agent-sdk/overview)
</LinkList>
