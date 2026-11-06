---
sidebar_label: 소개
sidebar_position: 0
sidebar_class_name: hidden
description: 프로덕션 환경에서 LLM 추론 시스템을 구축, 최적화, 확장, 운영하는 엔지니어를 위한 실용 핸드북입니다.
keywords:
    - LLM inference guide, LLM inference handbook, LLM inference book, LLM inference best practices
    - Inference, LLM, LLM inference, AI inference, GenAI inference
    - Inference optimization, inference techniques, LLM fast inference
    - Inference platform, inference operations
    - Efficient generative LLM inference, distributed LLM inference
---

import Features from '@site/src/components/Features';
import StackQuiz from '@site/src/components/StackQuiz';

# LLM 추론 핸드북

*LLM 추론 핸드북*은 기술 용어집이자 가이드북이자 참고 자료를 하나로 모은
책입니다. 핵심 개념과 성능 지표(예:
[첫 토큰까지의 시간과 초당 토큰 수](/llm-inference-basics/llm-inference-metrics/))부터
최적화 기법(예:
[연속 배칭](/inference-optimization/static-dynamic-continuous-batching/),
[프리픽스 캐싱](/inference-optimization/prefix-caching/)),
[GPU 아키텍처](/kernel-optimization/gpu-architecture-fundamentals/), 그리고
[BYOC](/getting-started/bring-your-own-cloud/)나
[온프레미스](/getting-started/on-prem-llms/) 같은 배포 패턴까지, LLM 추론에 대해
알아야 할 모든 것을 다룹니다.

<Features>
  - 프로덕션 환경에서 LLM을 배포, 확장, 운영하기 위한 실용적인 가이드를
    제공합니다.
  - 인터랙티브 계산기, 시뮬레이터, 시각화 도구로 개념을 직접 탐색할 수 있습니다.
  - 사용 사례에 맞춘 최적화 기법으로 성능을 끌어올릴 수 있습니다.
  - 최신 모범 사례와 현장에서 검증된 인사이트를 꾸준히 반영합니다.
</Features>

## 만든 이유 {#motivation}

이 핸드북은 개발자들이 흔히 겪는 문제를 해결하려고 만들었습니다. LLM 추론 지식은
여기저기 흩어져 있는 경우가 많습니다. 학술 논문에 묻혀 있거나, 벤더 블로그
곳곳에 퍼져 있거나, GitHub 이슈에 숨어 있거나, Discord 스레드에서 오가는
식입니다. 더 큰 문제는 이런 자료 대부분이 독자가 스택의 절반쯤은 이미 이해하고
있다고 가정한다는 점입니다.

이 모든 내용을 한데 모은 자료는 많지 않습니다. 예를 들어
[추론이 학습과 어떻게 다른지](/llm-inference-basics/training-inference-differences/),
SLO를 맞추는 데
[왜 단순 처리량보다 굿풋(goodput)이 더 중요한지](/llm-inference-basics/llm-inference-metrics/#goodput),
[프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)가
실제로 어떻게 동작하는지 같은 내용입니다.

그래서 이 모든 내용을 한데 모으기 시작했습니다.

## 이 핸드북의 대상 독자 {#who-this-is-for}

이 핸드북은 프로덕션 환경에서 LLM을 배포, 확장, 운영하는 엔지니어를 위한
책입니다. 작은 오픈 모델을 미세조정하든, 자체 스택에서 대규모 배포를 운영하든
상관없습니다.

LLM 추론을 더 빠르고, 더 저렴하고, 더 안정적으로 만들고 싶다면 이 핸드북이
도움이 될 것입니다.

## 활용 방법 {#how-to-use-this}

처음부터 끝까지 읽어도 되고, 필요할 때 찾아보는 참고표처럼 써도 됩니다. 정해진
읽는 방법은 없습니다. LLM 추론 분야는 빠르게 변하고 있어서 오늘 잘 통하는 방법이
내일은 최선이 아닐 수 있습니다. 그래서 분야의 변화에 맞춰 핸드북을 계속
업데이트할 예정입니다.

<StackQuiz />

## 인터랙티브 도구 {#interactive-tools}

이 핸드북은 개념을 직접 다뤄 보며 배울 수 있도록 다양한 인터랙티브 도구를
제공합니다.

- [LLM 추론 시각화 도구](/llm-inference-basics/what-is-llm-inference/): 요청의
  생애 주기를 단계별로 따라가며 토큰이 프리필과 디코드를 거쳐 어떻게 흐르는지
  확인합니다.
- [LLM 생애 주기 시각화 도구](/llm-inference-basics/training-inference-differences/):
  모델 생애 주기에서 학습과 추론이 어디에 위치하는지, 그리고 추론이 요청마다
  어떻게 실행되는지 확인합니다.
- [토큰 단위 디코드 루프](/llm-inference-basics/how-does-llm-inference-work/#decode):
  자기회귀 디코딩을 한 단계씩 진행하며 새 토큰이 시퀀스와 KV 캐시를 어떻게 늘려
  가는지 관찰합니다.
- [지연 시간 타임라인 시각화 도구](/llm-inference-basics/how-does-llm-inference-work/#decode):
  디코드 단계마다 역토큰화가 뒤따르는 과정과 TTFT, ITL, E2EL이 각각 어느 단계에
  걸쳐 있는지 확인합니다.
- [컨텍스트 윈도우 시뮬레이터](/llm-inference-basics/how-does-llm-inference-work/#what-is-a-context-window-and-how-does-it-work-in-llm-inference):
  대화 전체가 매 턴마다 다시 전송되어 컨텍스트 윈도우를 채우는 과정을
  확인합니다.
- [지연 시간 지표 플레이그라운드](/llm-inference-basics/llm-inference-metrics/#latency):
  TTFT, E2EL, TPOT, 그리고 SLO 기반 굿풋을 살펴봅니다.
- [Top-p와 Top-k 필터 비교](/model-interaction/inference-parameters/#top-p-and-top-k-sampling):
  뾰족한 분포, 혼합 분포, 평탄한 분포를 각 필터가 어떻게 처리하는지 비교합니다.
- [모델 탐색기](/getting-started/choosing-the-right-model/): 널리 쓰이는
  오픈소스 LLM을 둘러보고 아키텍처, 규모, 컨텍스트, 일반적인 GPU 배포 구성을
  비교합니다.
- [GPU 비교표](/getting-started/choosing-the-right-gpu/#matching-gpus-to-open-source-llms):
  널리 쓰이는 오픈소스 LLM에 알맞은 NVIDIA와 AMD GPU를 짝지어 봅니다.
- [GPU 메모리 계산기](/getting-started/calculating-gpu-memory-for-llms/#): LLM
  서빙에 필요한 VRAM을 추정합니다.
- [양자화 메모리 영향 시각화 도구](/model-preparation/llm-quantization/#quantization-formats):
  양자화 형식별 가중치 메모리를 비교합니다.
- [배칭 전략 시뮬레이터](/inference-optimization/static-dynamic-continuous-batching/):
  정적 배칭, 동적 배칭, 연속 배칭의 동작을 비교합니다.
- [청크 단위 프리필 스케줄러](/inference-optimization/static-dynamic-continuous-batching/#chunked-prefill):
  프리필 전체를 한 번에 처리하면 진행 중인 디코드가 어떻게 멈추는지, 그리고
  청크로 나누면 디코드가 어떻게 계속 진행되는지 확인합니다.
- [KV 캐시 메모리 계산기](/inference-optimization/kv-cache-offloading/#how-to-calculate-the-kv-cache-size):
  KV 캐시가 메모리를 얼마나 차지하는지 추정합니다.
- [GPU 실행 및 메모리 맵](/kernel-optimization/gpu-architecture-fundamentals/):
  스레드, 워프, SM, GPU 메모리 계층이 어떻게 맞물리는지 시각화합니다.
- [워프 분기 시각화 도구](/kernel-optimization/gpu-architecture-fundamentals/threads-warps-blocks/#warps):
  분기 하나가 워프를 직렬화된 여러 패스로 나누고 레인을 놀게 만드는 과정을
  확인합니다.
- [GPU 메모리 탐색기](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/):
  메모리 계층별 차이를 이해하고 데이터가 이동할 때 재사용 범위가 어떻게 바뀌는지
  확인합니다.
- [메모리 병합 시각화 도구](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/#memory-coalescing):
  연속, 정렬되지 않은, 스트라이드, 분산 접근을 비교하고 각 방식에서 GPU가 섹터를
  몇 개나 가져와야 하는지 확인합니다.
- [스트리밍 멀티프로세서 탐색기](/kernel-optimization/gpu-architecture-fundamentals/streaming-multiprocessors/#what-an-sm-contains):
  SM 내부의 유닛을 살펴보고 H100에 유닛이 종류별로 몇 개씩 있는지 확인합니다.
- [워프 스케줄러 시각화 도구](/kernel-optimization/gpu-architecture-fundamentals/streaming-multiprocessors/#how-warp-scheduling-hides-latency):
  상주 워프가 너무 적을 때, 충분할 때, 여유 있을 때를 스케줄러 타임라인에서
  비교합니다.
- [커널 융합 시각화 도구](/kernel-optimization/kernel-optimization-for-llm-inference/#what-is-kernel-fusion):
  커널이 분리되어 있으면 중간 결과가 매번 HBM을 왕복하는 과정과 커널을 융합하면
  무엇을 절약할 수 있는지 확인합니다.
- [추론 비용 계산기](/infrastructure-and-operations/build-and-maintenance-cost/#hidden-costs-of-self-hosted-inference):
  평균 처리량과 월별 추가 비용에 따라 실질 100만 토큰당 비용이 어떻게 달라지는지
  확인합니다.

## 기여하기 {#contributing}

기여를 환영합니다! 오류를 발견했거나, 개선 제안이 있거나, 새로운 주제를 추가하고
싶다면 [GitHub 저장소](https://github.com/modular/llm-inference-handbook)에
이슈를 열거나 풀 리퀘스트를 보내 주세요.
