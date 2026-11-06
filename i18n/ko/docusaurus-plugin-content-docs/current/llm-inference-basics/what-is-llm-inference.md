---
sidebar_position: 1
description: LLM 추론은 학습된 언어 모델을 사용해 프롬프트를 바탕으로 응답이나 예측을 생성하는 과정입니다.
keywords:
    - Large Language Models, LLM
    - LLM inference meaning, LLM inference concept
    - LLM inference, AI inference, inference layer
---

import RequestLifecycle from '@site/src/components/RequestLifecycle';

# LLM 추론이란?

LLM 추론은 GPT-5, GLM-5, DeepSeek-V3.2 같은 학습된 LLM을 사용해 사용자
입력으로부터 의미 있는 출력을 생성하는 것을 말합니다. 사용자 입력은 보통 자연어
[프롬프트](/model-interaction/prompt-engineering/)로 주어집니다. 추론 중에
모델은 방대한 매개변수를 거쳐 프롬프트를 처리하고 텍스트, 코드 조각, 요약, 번역
같은 응답을 생성합니다.

말하자면 추론은 LLM이 실제로 "작동하는" 순간입니다. 실제 사례는 다음과 같습니다.

- **고객 지원 챗봇**: 고객 문의에 맞춤화되고 맥락에 맞는 답변을 실시간으로
  생성합니다.
- **글쓰기 도우미**: 문장을 완성하거나, 문법을 교정하거나, 긴 문서를 요약합니다.
- **개발자 도구**: 자연어 설명을 실행 가능한 코드로 변환합니다.
- **AI 에이전트**: 복잡한 다단계 추론과 의사 결정 과정을 자율적으로 수행합니다.

아래 시각화 도구로 LLM 추론 중에 요청이 어떻게 흘러가는지 확인해 보세요. 자세한
내용은
[LLM 추론의 작동 방식](/llm-inference-basics/how-does-llm-inference-work/)을
참고하세요.

<RequestLifecycle />

## 추론 서버란? {#what-is-an-inference-server}

**추론 서버**는 LLM 추론이 실행되는 방식을 관리하는 구성 요소입니다. 모델을
불러오고, 필요한 하드웨어(GPU 등)에 연결하고, 애플리케이션 요청을 처리합니다.
프롬프트가 도착하면 서버는 자원을 할당하고, 모델을 실행하고, 출력을 반환합니다.

LLM 추론 서버는 단순한 요청-응답 처리보다 훨씬 많은 일을 합니다. LLM을 대규모로
운영하는 데 꼭 필요한 다음과 같은 기능을 제공합니다.

- **배칭(batching)**: 여러 요청을 묶어 GPU 효율을 높입니다
- **스트리밍**: 토큰이 생성되는 대로 전송해 지연 시간을 줄입니다
- **확장**: 수요에 따라 레플리카를 늘리거나 줄입니다
- **모니터링**: 성능과 디버깅을 위한 지표를 제공합니다

LLM 분야에서는 **추론 서버**와 **추론 프레임워크**를 어느 정도 같은 뜻으로 섞어
쓰곤 합니다.

- **추론 서버**는 보통 요청을 받고, 모델을 실행하고, 결과를 반환하는 런타임 구성
  요소를 강조합니다.
- **추론 프레임워크**는 모델을 효율적으로 서빙하기 위한 API, 최적화, 통합 기능을
  제공하는 더 넓은 범위의 툴킷이나 라이브러리를 가리킬 때가 많습니다.

많이 쓰이는
[추론 프레임워크](/getting-started/choosing-the-right-inference-framework/)로는
vLLM, SGLang, MAX, TensorRT LLM이 있습니다. 이 프레임워크들은 GPU 효율을
극대화하면서 LLM을 대규모로 더 쉽게 배포할 수 있도록 설계되었습니다.

## 서빙과 추론의 차이는? {#whats-the-difference-between-serving-and-inference}

추론은 연산입니다. 서빙은 그 연산을 사용자와 애플리케이션이 쓸 수 있게 만드는
프로덕션 과정입니다.

일상 대화에서는 최신 프레임워크가 둘 다 처리하기 때문에 두 용어를 흔히 섞어
씁니다. vLLM, SGLang, MAX, TensorRT LLM 같은 도구는 추론을 효율적으로 실행할 뿐
아니라 요청을 관리하고 API도 제공합니다. 그래서 "추론 프레임워크"와 "서빙
프레임워크"가 같은 시스템을 가리키곤 합니다.

하지만 엄밀히 말하면 둘은 같지 않습니다. 인프라를 설계하기 시작하는 순간 이
구분이 중요해집니다.

- 추론은 순전파입니다. 토큰이 들어가면 다음 토큰 분포가 나오고, 이 과정이
  프리필(prefill)과 디코드(decode) 루프를 거치며 반복됩니다. 추론은 연산
  작업입니다. 서버 없이 Jupyter 노트북이나 Python 스크립트에서도 추론을 실행할
  수 있습니다. 이 경우에도 추론이기는 하지만, 아무것도 서빙하고 있지는 않습니다.
- 서빙은 그 능력을 신뢰할 수 있는 프로덕션 서비스로 바꾸는 데 필요한 모든
  것입니다. API 인터페이스(HTTP/gRPC), 요청 큐잉과 스케줄링, 동시 요청 간 배칭,
  레플리카 간 부하 분산, 오토스케일링, 모델 로딩과 버전 관리, 헬스 체크, 관측
  가능성, 자원 할당이 여기에 포함됩니다.

따라서 누군가 두 용어를 섞어 쓴다면, 대개 자신의 추론 엔진이 서버 역할도 함께
한다는 사실을 느슨하게 표현하는 것입니다. 이런 경우는 아주 흔합니다.

## 추론 최적화란? {#what-is-inference-optimization}

**추론 최적화**는 LLM 추론을 더 빠르고, 더 저렴하고, 더 효율적으로 만드는
기법들입니다. 모델 품질을 해치지 않으면서 지연 시간을 줄이고, 처리량을 높이고,
하드웨어 비용을 낮추는 것이 목표입니다.

흔히 쓰는 전략은 다음과 같습니다.

- [연속 배칭](/inference-optimization/static-dynamic-continuous-batching/):
  요청을 동적으로 묶어 GPU 사용률을 높입니다
- [KV 캐시 관리](/inference-optimization/kv-cache-offloading/): 어텐션 캐시를
  재사용하거나 오프로드해 긴 프롬프트를 효율적으로 처리합니다
- [추측 디코딩](/inference-optimization/speculative-decoding/): 더 작은 드래프트
  모델을 사용해 토큰 생성 속도를 높입니다
- [양자화](/model-preparation/llm-quantization/): 더 낮은 정밀도(예: INT8,
  FP8)로 모델을 실행해 메모리와 연산을 절약합니다
- [프리픽스 캐싱](/inference-optimization/prefix-caching/): 공통 프롬프트 구간을
  캐싱해 중복 연산을 줄입니다
- [멀티 GPU 분산/병렬화](/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/):
  LLM을 여러 GPU에 나누어 더 큰 컨텍스트 윈도우를 처리합니다

실제로 추론 최적화는 느리고 비싸게 느껴지는 애플리케이션과 빠르고 비용 효율적인
사용자 경험을 제공하는 애플리케이션을 가르는 차이가 될 수 있습니다.

자세한 내용은 [추론 최적화](/inference-optimization/) 장에서 알아보세요.

## LLM 추론에 왜 관심을 가져야 하나요? {#why-should-i-care-about-llm-inference}

이렇게 생각할 수도 있습니다.
_OpenAI API를 쓰고 있을 뿐인데, 정말 추론까지 이해해야 할까?_

OpenAI, Anthropic 등의 서버리스 API는 추론을 단순해 보이게 만듭니다. 프롬프트를
보내고, 응답을 받고, 토큰 단위로 비용을 냅니다. 인프라, 모델 최적화, 확장은 모두
보이지 않는 곳에 숨겨져 있습니다.

하지만 핵심은 이것입니다. **서비스가 커질수록 추론은 더 중요해집니다.**

애플리케이션이 성장하면 결국 서버리스 API로는 완전히 해결할 수 없는 한계(예:
비용, 지연 시간, 커스터마이징, 규정 준수)에 부딪히게 됩니다. 이때부터 팀은
하이브리드나 셀프 호스팅 방식을 검토하기 시작합니다.

LLM 추론(예:
[배칭](/inference-optimization/static-dynamic-continuous-batching/),
[캐싱](/inference-optimization/prefix-caching/),
[양자화](/model-preparation/llm-quantization/),
[라우팅](/inference-optimization/inference-routing/))을 일찍 이해해 두면 분명한
우위를 얻을 수 있습니다. 여러 모델 서비스 공급자와
[추론 프레임워크](/getting-started/choosing-the-right-inference-framework/)의
기능을 평가할 수 있으므로, 더 현명하게 선택하고, 예상치 못한 문제를 피하고, 더
확장성 있는 시스템을 구축할 수 있습니다.

- **개발자나 엔지니어라면**: 최신 AI 애플리케이션 개발에서 추론은 데이터베이스나
  API만큼 기본적인 요소가 되어 가고 있습니다. 추론의 작동 방식을 알면 더 빠르고,
  더 저렴하고, 더 안정적인 시스템을 설계할 수 있습니다. 추론을 잘못 구현하면
  응답 시간이 느려지고, 연산 비용이 높아지고, 사용자 경험이 나빠질 수 있습니다.
- **기술 리더라면**: 추론 효율은 수익에 직접 영향을 줍니다. 제대로 최적화되지
  않은 구성은 GPU 시간 비용이 10× 더 들면서도 성능은 더 나쁠 수 있습니다. 추론을
  이해하면 벤더를 평가하고, 직접 구축할지 구매할지 결정하고, 팀에 현실적인 성능
  목표를 세울 수 있습니다.
- **AI에 그저 호기심이 있다면**: 추론은 AI가 실제로 동작하는 바로 그 지점입니다.
  추론의 작동 방식을 알면 AI에 대한 과장과 현실을 구분할 수 있고, AI를 더 잘
  알고 쓰는 사용자가 되어 AI 논의에도 더 잘 기여할 수 있습니다.

자세한 내용은
[서버리스 vs. 셀프 호스팅 LLM 추론](/getting-started/serverless-vs-self-hosted-llm-inference/)을
참고하세요.
