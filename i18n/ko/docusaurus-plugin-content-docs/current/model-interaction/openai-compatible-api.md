---
sidebar_position: 1
description: OpenAI 호환 API는 OpenAI 공식 API와 같은 요청·응답 형식을 구현하므로, 개발자는 기존 코드를 바꾸지 않고도 여러 모델 사이를 전환할 수 있습니다.
keywords:
    - OpenAI-compatible API, OpenAI-compatible endpoint, OpenAI-compatible server
    - OpenAI API, OpenAI compatibility, ChatGPT
    - LLM inference API
---

import LinkList from '@site/src/components/LinkList';

# OpenAI 호환 API

LLM을 실행했다면 이제 모델과 상호작용할 표준 방법이 필요합니다. 여기서 OpenAI
호환 API가 등장합니다.

## OpenAI 호환 API란? {#what-is-an-openai-compatible-api}

OpenAI 호환 API는 OpenAI의 일반적인 인터페이스, 요청/응답 스키마, 인증 규칙을
그대로 따르는 모든 API를 말합니다. OpenAI가 이를 업계 표준으로 공식 정의한 적은
없지만, OpenAI의 API는 사실상 LLM의 표준 인터페이스가 되었습니다.

2022년 말 ChatGPT의 부상은 이 방식이 얼마나 강력하고 사용하기 쉬운지 보여
주었습니다.

- 깔끔하고 문서화가 잘된 API 덕분에 개발자는 LLM으로 애플리케이션을 쉽게 만들 수
  있습니다.
- `gpt-4o` 같은 모델을 단순하고 일관된 엔드포인트로 사용할 수 있습니다.

그 결과 다양한 산업에서 빠르게 채택되었고 생태계도 성장했습니다.

## 호환성이 중요한 이유 {#why-does-compatibility-matter}

OpenAI의 API는 AI 애플리케이션 개발의 시작을 이끌었지만, 널리 채택되면서 생태계
종속도 만들었습니다. 이제 많은 개발 도구, 프레임워크, SDK가 OpenAI 스키마를
기준으로 만들어집니다. 다음과 같은 일을 하려면 이 점이 문제가 됩니다.

- 다른 모델로 전환
- 셀프 호스팅 배포로 이전
- 새로운 추론 공급자 시험

이런 경우 새 API에 맞춰 애플리케이션 로직을 다시 작성하는 일은 번거롭고 오류가
생기기 쉽습니다.

OpenAI 호환 API는 다음을 제공해 이런 문제를 해결합니다.

- **간편한 대체(drop-in replacement)**: OpenAI의 호스팅 API를 셀프 호스팅
  모델이나 오픈소스 모델로 바꿀 수 있으며, 대개 애플리케이션 코드를 바꿀 필요가
  없습니다.
- **원활한 마이그레이션**: 공급자나 셀프 호스팅 배포 사이를 최소한의 중단으로
  옮길 수 있습니다.
- **일관된 통합**: OpenAI API 스키마(예: `chat/completions`, `embeddings`
  엔드포인트)에 의존하는 도구와 프레임워크와의 호환성을 유지합니다.

많은 [추론 백엔드](/getting-started/choosing-the-right-inference-framework/)(예:
vLLM, SGLang, MAX)가 OpenAI 호환 엔드포인트를 기본으로 제공합니다. 그래서
클라이언트 코드를 바꾸지 않고도 여러 모델 사이를 쉽게 전환할 수 있습니다.

## OpenAI 호환 API 호출 방법 {#how-to-call-an-openai-compatible-api}

기존 SDK와 프레임워크가 폭넓게 지원하기 때문에 많은 호환 서버가 Chat Completions
API를 대상으로 합니다. OpenAI 공식 문서는 이제 OpenAI 호스팅 환경의 새
애플리케이션에는 새로운 Responses API를 권장하지만, Responses API 호환 범위는
서빙 프레임워크마다 다릅니다.

다음과 같이 기존 OpenAI 클라이언트가 셀프 호스팅 또는 대체 공급자의 Chat
Completions 엔드포인트를 가리키도록 설정합니다.

```python
from openai import OpenAI

# Use your custom endpoint URL and API key
client = OpenAI(
    base_url="https://your-custom-endpoint.com/v1",
    api_key="your-api-key"
)

response = client.chat.completions.create(
    model="your-model-name",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "How can I integrate OpenAI-compatible APIs?"}
    ]
)

print(response.choices[0].message)
```

OpenAI API는 `api_key` 필드를 필수로 요구합니다. 대부분의 추론 프레임워크는 이
값을 검증하지 않으므로 `api_key="EMPTY"`처럼 아무 값이나 넣어도 됩니다.

간단한 HTTP 요청으로 API를 직접 호출할 수도 있습니다. 다음은 `curl`을 사용한
예시입니다.

```bash
curl https://your-custom-endpoint.com/v1/chat/completions \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-model-name",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "How can I integrate OpenAI-compatible APIs?"}
    ]
  }'
```

이미 OpenAI SDK나 REST 인터페이스를 사용하고 있다면 대개 자체 API 엔드포인트로
방향만 바꾸면 됩니다. 이렇게 하면 LLM 배포를 직접 통제하고 벤더 종속을 줄일 수
있습니다.

### 스트리밍 응답 {#streaming-responses}

`stream=True`로 설정하면 토큰이 생성되는 대로 하나씩 받을 수 있습니다. 채팅 UI나
지연 시간에 민감한 애플리케이션에 유용합니다.

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://your-custom-endpoint.com/v1",
    api_key="your-api-key"
)

stream = client.chat.completions.create(
    model="your-model-name",
    messages=[
        {"role": "user", "content": "Write a short poem about streaming."}
    ],
    stream=True,
)

for chunk in stream:
    delta = chunk.choices[0].delta
    if delta.content:
        print(delta.content, end="", flush=True)
```

정확한 스트리밍 스키마는 사용하는 프레임워크에 따라 다를 수 있습니다. 항상 해당
프레임워크의 공식 문서를 확인하세요.

### 사용 가능한 모델 목록 조회 {#listing-available-models}

대부분의 OpenAI 호환 서버는 `/v1/models` 엔드포인트도 구현합니다. 이
엔드포인트로 백엔드가 받아들이는 `model` 이름을 확인할 수 있습니다.

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://your-custom-endpoint.com/v1",
    api_key="your-api-key"
)

for model in client.models.list().data:
    print(model.id)
```

`curl`로도 조회할 수 있습니다.

```bash
curl https://your-custom-endpoint.com/v1/models \
  -H "Authorization: Bearer your-api-key"
```

반환된 `id` 중 하나를 Chat Completions 요청의 `model` 필드에 사용하면 됩니다.
단, 모든 프레임워크가 이 엔드포인트를 제공하지는 않습니다.

대부분의 호환 엔드포인트는 `temperature`, `top_p`, `max_tokens` 같은 일반적인
[LLM 추론 매개변수](/model-interaction/inference-parameters/)도 받습니다. 지원
범위는 공급자와 셀프 호스팅 백엔드마다 다르므로, 프로덕션에서 사용하기 전에
서버가 받아들이는 필드를 정확히 확인하세요.

## 자주 묻는 질문 {#faqs}

### OpenAI 호환 API는 OpenAI 공식 API와 같은가? {#is-an-openai-compatible-api-the-same-as-openais-official-api}

아닙니다. OpenAI 호환 API는 인터페이스만 따라 할 뿐, 그 아래의 모델이나
인프라까지 같지는 않습니다. 같은 "언어"로 말하지만 상대는 다른 시스템이라고
생각하면 됩니다. 공급자에 따라 백엔드는 다음 중 하나일 수 있습니다.

- Llama나 DeepSeek 같은 셀프 호스팅 LLM
- Together AI나 Fireworks 같은 호스팅 공급자
- VPC 안에 구축한 맞춤형 엔터프라이즈 배포

API 형태가 같아 보여도 백엔드마다 속도와 비용이 다릅니다.

### OpenAI 호환 API 뒤에서 어떤 모델을 실행할 수 있는가? {#what-models-can-i-run-behind-an-openai-compatible-api}

Llama, Qwen, Mistral, DeepSeek, Kimi, 도메인 특화 미세조정된 모델 등 최신
오픈소스 LLM이라면 무엇이든 OpenAI 호환 API 뒤에서 서빙할 수 있습니다.

vLLM, SGLang, MAX 같은 프레임워크를 사용한다면 이런 모델을 OpenAI 호환
엔드포인트로 자동으로 노출할 수 있습니다.

### LLM을 셀프 호스팅하려면 OpenAI 호환 API가 꼭 필요한가? {#is-an-openai-compatible-api-required-to-self-host-an-llm}

반드시 필요하지는 않지만, 현실적으로는 대개 이 방식을 택합니다. OpenAI 호환
API가 없으면 에이전트 통합, SDK 통합, 프레임워크 호환성 등을 직접 다시 구축해야
할 수 있습니다. OpenAI 스키마를 사용하면 스택을 더 단순하고 이식성 있게 유지할
수 있습니다.

### OpenAI 호환 API를 사용하면 비용이 절감되는가? {#does-using-an-openai-compatible-api-save-cost}

그 자체만으로는 아닙니다. API 형식은 인터페이스일 뿐이며 추론을 더 저렴하게
만들지 않습니다.

비용 절감은 API가 어디서 실행되느냐에서 나옵니다. 경우별로 정리하면 다음과
같습니다.

- **vLLM, SGLang, MAX 같은 도구로 LLM을 셀프 호스팅한다면** 토큰당 요금 대신
  주로 GPU 비용을 냅니다.
  [KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/)이나
  [프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)
  같은 추론 최적화를 적용해 사용률을 높이고 서빙 비용을 줄일 수도 있습니다.
  배포의 사용률이 높게 유지된다면, 꾸준하거나 대량인 워크로드에서 훨씬 저렴할 수
  있습니다.
- **호스팅 공급자(예: Together AI, Fireworks)를 사용한다면** API가 "OpenAI
  호환"이더라도 여전히 토큰당 또는 요청당 비용을 냅니다.
- **OpenAI를 계속 사용한다면** OpenAI 요금으로 토큰당 비용을 냅니다.

일부 AI 팀이 비용을 아끼는 이유는 OpenAI 호환 API 자체가 아닙니다. 기존
애플리케이션 코드를 깨뜨리지 않고 어떤 모델이든 셀프 호스팅할 수 있기
때문입니다.
[서버리스와 셀프 호스팅 LLM 추론 비교](/getting-started/serverless-vs-self-hosted-llm-inference/)에서
더 알아보세요.

<LinkList>

## 추가 자료 {#additional-resources}

- [OpenAI 문서](https://platform.openai.com/docs/quickstart?api-mode=chat)
- [MAX: OpenAI 호환 엔드포인트로 LLM 서빙하기](https://github.com/modular/modular)
</LinkList>
