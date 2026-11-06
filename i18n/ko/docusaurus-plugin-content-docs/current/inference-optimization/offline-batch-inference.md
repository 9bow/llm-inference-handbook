---
sidebar_position: 10
description: 오프라인 배치 추론으로 실시간이 아닌 방식으로 대규모 예측을 효율적으로 실행합니다.
keywords:
    - Offline batch inference, batch inference, batch LLM inference, batch requests, batch processing, LLM inference batching
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

# 오프라인 배치 추론

오프라인 배치 추론(offline batch inference)은 대규모의 정적인 데이터셋에 모델을
실행해, 실시간으로 하나씩 예측을 생성하는(온라인 추론) 대신 배치 단위로 예측을
생성하는 과정입니다. 대화형으로 이루어지지 않고 대량 처리 작업으로 실행되기
때문에 "오프라인"이라고 부릅니다.

반면 온라인 추론은 모델이 필요할 때마다(예: 클라이언트가 요청을 보낼 때) 예측을
생성하는 방식입니다.

오프라인 배치 추론의 주요 이점은 다음과 같습니다.

- 예측을 미리 계산해 두므로 실시간 시스템의 부하가 줄어듭니다
- 실시간 추론에는 너무 느린 복잡한 모델도 더 유연하게 사용할 수 있습니다.
- 예측을 프로덕션에서 사용하기 전에 후처리와 검증을 할 수 있습니다.

다음과 같은 경우에는 오프라인 배치 추론을 고려할 수 있습니다.

- 데이터가 자주 바뀌지 않아서 실시간 예측이 필요하지 않은 경우.
- 처리할 데이터셋이 크고, 예측을 저장해 두었다가 나중에 재사용할 수 있는 경우.
- 모델이 실시간 예측에는 너무 크거나 느리지만, 미리 실행해 두면 문제없이
  동작하는 경우.
- 사용자에게 예측을 제공하기 전에 검증하거나 검토하고 싶은 경우(예: 품질 또는
  규정 준수 확인).
