---
sidebar_position: 5
description: 프리필과 디코드를 분리해 병렬 실행, 자원 할당, 확장성을 개선합니다.
keywords:
    - Prefill-decode disaggregation
    - Disaggregated inference
    - Disaggregating prefill and decode
    - Disaggregated serving
    - Prefill, decode
    - Cross-cluster prefill, cross-datacenter prefill
    - Distributed LLM inference
    - KV cache transfer
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

import LinkList from '@site/src/components/LinkList';

# 프리필-디코드 분리(prefill-decode disaggregation)

프리필-디코드(PD) 분리(prefill-decode disaggregation)는 분리형
추론(disaggregated inference)이라고도 부르며, LLM 추론의 두 주요 단계인
프리필(prefill)과 디코드(decode)를 서로 다른 하드웨어 자원에서 실행하는 서빙
아키텍처입니다. "분리형 프리필(disaggregated prefill)"과 "분리형
서빙(disaggregated serving)"이라는 용어도 대개 같은 핵심 개념을 가리킵니다. 즉,
연산 요구량과 메모리 요구량이 서로 다른 각 단계에 전용 자원을 할당하는 것입니다.

## 프리필과 디코드를 함께 배치할 때의 문제 {#the-problems-with-collocating-prefill-and-decode}

함께 배치하는 방식이 왜 문제를 일으키는지 알아보기 위해,
[LLM 추론이 동작하는 방식](/llm-inference-basics/how-does-llm-inference-work/)을
두 단계로 간단히 다시 살펴보겠습니다.

- **프리필**: 전체 시퀀스를 병렬로 처리하고, 어텐션 레이어의 키(key) 벡터와
  값(value) 벡터를 KV 캐시에 저장합니다. 큰 행렬 연산으로 모든 토큰을 한 번에
  처리하므로 프리필은 연산 바운드(compute-bound)이지만, GPU 메모리에는 큰 부담을
  주지 않습니다.
- **디코드**: 앞서 만든 KV 캐시를 재사용해 출력 토큰을 한 번에 하나씩
  생성합니다. 토큰을 하나 생성할 때마다 모델 가중치를 반복해서 불러오고 계속
  커지는 KV 캐시에 접근해야 합니다. 따라서 디코드에는 빠른 메모리 접근이
  필요하지만 연산량은 적습니다.

<figure>
  <Diagram name="llm-inference-flow" alt="" />
  <figcaption><b>그림 1.</b> 프롬프트에서 출력까지 이어지는 종단 간 LLM 추론 흐름.</figcaption>
</figure>

오랫동안 추론의 표준 방식은 이 두 단계를 함께 실행하는 것이었습니다. 겉보기에는
간단해 보일 수 있습니다.

하지만 실제로는 여러 요청이 동시에 들어오는 경우가 많습니다. 요청마다 각자의
프리필과 디코드 작업이 필요하지만, 한 번에 한 단계만 실행할 수 있습니다. GPU가
연산량이 많은 프리필 작업을 처리하는 동안에는 디코드 작업이 기다려야 하므로
ITL이 늘어나고, 그 반대도 마찬가지입니다.

프리필은 주로 TTFT를 결정하고 디코드는 ITL에 영향을 주므로, 두 단계를 함께
배치하면 두 지표를 동시에 최적화하기 어렵습니다.

<figure>
  <img src={require('@site/docs/inference-optimization/img/pd-disaggregation-results.png').default} alt="" />
  <figcaption><b>그림 2.</b> 프리필과 디코드를 함께 배치할 때 늘어나는 지연 시간. 출처: <cite><a href="https://arxiv.org/abs/2401.09670">DistServe:
  Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model
  Serving</a></cite>.</figcaption>
</figure>

## 분리가 합리적인 이유 {#why-disaggregation-makes-sense}

PD 분리의 아이디어는 간단합니다. 성격이 매우 다른 두 작업을 나누어 서로 방해하지
않게 하는 것입니다. 그림 3은 아키텍처 예시를 보여 줍니다.

<figure>
  <Diagram name="pd-disaggregation" alt="" />
  <figcaption><b>그림 3.</b> 프리필-디코드 분리 아키텍처 예시.</figcaption>
</figure>

주요 이점은 다음과 같습니다.

- **전용 자원 할당**: 프리필과 디코드를 서로 다른 하드웨어에서 독립적으로
  스케줄링하고 확장할 수 있습니다. 예를 들어 워크로드에 겹치는 프롬프트가
  많다면(멀티턴 대화나 에이전트 워크플로처럼) KV 캐시의 상당 부분을 재사용할 수
  있습니다. 그 결과 프리필의 연산 요구량이 줄어들고, 디코드에 더 많은 자원을
  배정할 수 있습니다.
- **병렬 실행**: 프리필 단계와 디코드 단계가 더 이상 서로 간섭하지 않습니다. 두
  단계를 더 효율적으로 병렬 실행할 수 있으므로 동시성과 처리량이 좋아집니다.
  분리는 [꼬리 지연 시간](/llm-inference-basics/llm-inference-metrics)도 개선할
  수 있습니다. 함께 배치하면 긴 프리필 하나가 그 뒤에서 진행 중인 모든 디코드
  요청을 멈추게 해 P95와 P99 지연 시간이 올라갈 수 있습니다.
- **독립적 튜닝**: 프리필과 디코드에 서로 다른 최적화 기법(텐서 병렬화나
  파이프라인 병렬화 등)을 적용해 TTFT와 ITL 목표를 더 잘 맞출 수 있습니다.

여러 오픈소스 프레임워크와 프로젝트가 이미 PD 분리를 지원하고 있습니다.
대표적으로
[vLLM](https://docs.vllm.ai/en/latest/features/disagg_prefill.html),
[SGLang](https://docs.sglang.io/docs/advanced_features/pd_disaggregation),
[MAX](https://max.modular.com/cli/serve/#--pipeline-role-pipeline_role),
[Dynamo](https://docs.nvidia.com/dynamo/dev/user-guides/disaggregated-serving),
[llm-d](https://llm-d.ai/docs/architecture/advanced/disaggregation)가 있습니다.

## 분리가 항상 만능 해결책은 아닙니다 {#disaggregation-isnt-always-a-silver-bullet}

PD 분리가 유망해 보이더라도 모든 상황에 맞는 해결책은 아닙니다.

- **임곗값이 중요합니다**: 워크로드가 너무 작거나 GPU 구성이 이 방식에 맞게
  조정되지 않았다면 성능이 떨어질 수 있습니다(저희 테스트에서는 20-30% 하락).
- **로컬 프리필이 더 빠를 수 있습니다**: 프롬프트가 짧거나 디코드 엔진의
  프리픽스 캐시 적중률이 높을 때는 디코드 워커에서 프리필을 로컬로 실행하는 편이
  더 빠르고 간단한 경우가 많습니다.
- **데이터 전송 비용**: 분리하려면 프리필 워커와 디코드 워커 사이에서 KV 캐시를
  빠르고 안정적으로 옮겨야 합니다. 따라서 솔루션이 하드웨어와 네트워크에
  구애받지 않는 빠르고 지연 시간이 낮은 통신 프로토콜을 지원해야 합니다. 분리로
  얻는 성능 이득이 데이터 전송 비용보다 크지 않으면 전체 성능이 오히려 나빠질 수
  있습니다. 참고할 만한 기존 데이터 전송 방법으로는
  [NVIDIA Inference Xfer Library (NIXL)](https://github.com/ai-dynamo/nixl),
  CXL, NVMe-oF가 있습니다.

  프로덕션에서는 다음과 같은 설계 질문을 고려하세요.

  - 디코드 워커가 프리필 워커에서 KV 블록을 직접 가져와야 할까요, 아니면 양쪽이
    공유 캐시 계층을 사용해야 할까요?
  - KV 블록을 프리필 직후에 즉시 옮겨야 할까요, 아니면 디코드가 실제로 필요로 할
    때 지연해서 옮겨야 할까요?
  - 라우터는 기존 캐시를 재사용할지, 로컬 프리필을 실행할지, 요청을 별도의
    프리필 풀로 보낼지를 어떻게 결정할까요?

  이 질문들은 PD 분리를
  [KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/) 및
  [프리픽스 캐싱](/inference-optimization/prefix-caching/)과 연결합니다. 이들을
  서로 무관한 개별 조정 항목이 아니라, 같은 서빙 아키텍처의 구성 요소로 다루세요.

- **캐시 호환성이 중요합니다**: 프리필 워커와 디코드 워커는 KV 레이아웃, 페이지
  크기, dtype, 어텐션 변형, 그 밖의 추가 캐시 메타데이터에 대해 서로 일치해야
  합니다. 이종 KV 유형(예: 양자화된 KV 캐시, VLM 인코더 상태, 추측 디코딩
  캐시)이 있으면 표준 전체 어텐션 KV 텐서 하나를 옮기는 것보다 이 전달 과정이 더
  복잡해질 수 있습니다.

## 클러스터 간 프리필-디코드 분리 {#cross-cluster-prefill-decode-disaggregation}

많은 분리형 시스템에서 프리필 머신과 디코드 머신은 같은 클러스터 안에 나란히
있으며, 고속 저지연 인터커넥트로 연결되어 있습니다.

클러스터 간(또는 데이터센터 간) PD 분리는 워크플로를 나누어 프리필 단계와 디코드
단계를 완전히 별개의 클러스터에서 실행하는 방식입니다. 때로는 두 클러스터가 서로
다른 데이터센터나 리전에 있기도 합니다.

### 왜 프리필과 디코드를 클러스터 간에 나눌까요? {#why-split-prefill-and-decode-across-clusters}

인프라를 이런 분산 모델로 이끄는 주요 동인은 두 가지입니다.

- **작업에 맞는 칩 사용**. 프리필은 연산 부담이 크고, 디코드는 메모리 대역폭
  부담이 큽니다. 실리콘 설계도 이처럼 서로 다른 요구에 맞춰 갈라지고 있습니다.
  - NVIDIA Rubin CPX는 프리필 처리량을 극대화하도록 만들어졌습니다.
  - Groq LPU는 빠른 디코드 대역폭을 위해 설계되었습니다.

    문제는 이런 칩들이 항상 같은 곳에 있지는 않다는 점입니다. 보통 하드웨어
    유형별로 묶여 별도의 클러스터에 배포됩니다. 프리필과 디코드를 한곳에 묶어
    두면 각 단계에 가장 적합한 하드웨어를 충분히 활용할 수 없습니다. 두 단계를
    나누면 긴 프리필은 연산에 최적화된 머신에서 실행하고, 디코드는 대역폭에
    최적화된 머신에 둘 수 있습니다.

- **유연성**. 프로덕션에서 프리필과 디코드는 균등하게 확장되지 않습니다.
  트래픽과 프롬프트 길이가 바뀌고, 프리픽스 캐시 적중률도 바뀝니다. 어떤 때는
  프리필이 병목이 되고, 어떤 때는 디코드가 병목이 됩니다.

  두 단계가 한 클러스터에 묶여 있으면 고정된 자원 비율에 얽매이게 됩니다. 그러면
  한쪽에는 자원이 과잉 할당되고 다른 쪽에는 병목이 생길 수 있습니다. 단계를
  클러스터별로 나누면 각 쪽을 독립적으로 확장할 수 있습니다.

### 핵심 문제: KV 캐시 이동 {#the-core-problem-kv-cache-movement}

프리필이 끝나면 KV 캐시를 디코드 클러스터로 보내야 합니다. 한 클러스터 안에서는
이 비용이 적지만, 클러스터 간에는 클 수 있습니다. 두 가지 이유로 이 문제가
까다로워집니다.

- **KV 캐시는 클 수 있습니다**. 네트워크로 KV 캐시를 보내다 보면 프리필을 빠르게
  해서 얻은 이득이 모두 사라질 수 있습니다.
- **모든 요청이 이득을 보지는 않습니다**. 짧은 프롬프트나 캐시된 프롬프트는 원격
  프리필로 얻는 이득이 크지 않지만, 네트워크 비용은 그대로 치러야 합니다.

따라서 모든 요청을 원격 프리필 클러스터로 무작정 보내면 성능이 오히려 나빠질 수
있습니다.

### Prefill-as-a-Service {#prefill-as-a-service}

[이 논문](https://arxiv.org/abs/2604.15039)은 이 구성을 실제로 작동하게 하는
방법으로 Prefill-as-a-Service(PrfaaS)를 제안합니다. 아이디어는 간단합니다. 모든
요청을 클러스터 간에 보내지 말고, 선별해서 보내는 것입니다.

- 짧거나 캐시된 요청은 로컬에 둡니다
- 길고 캐시되지 않은 프리필만 원격 연산 클러스터로 보냅니다

이렇게 하면 클러스터 간 PD는 라우팅 문제가 됩니다. 스케줄러는 프롬프트 길이(특히
캐시되지 않은 토큰), 프리픽스 캐시의 지역성, 프리필 대기열 압박, 디코드 용량,
사용 가능한 네트워크 대역폭을 바탕으로 각 요청을 어디로 보낼지 결정합니다.

논문은 이 선별적 방식이 표준 PD 기준선보다 큰 성능 향상을 보인다고 보고합니다.

- 처리량 54% 향상
- P90 TTFT 64% 감소

이런 향상은 단순히 더 빠른 하드웨어가 아니라 더 나은 자원 사용률에서 나옵니다.

### 클러스터 간 PD 분리는 언제 사용해야 할까요? {#when-should-you-use-cross-cluster-pd-disaggregation}

이 배포 방식은 다음과 같은 경우에 가장 효과적입니다.

- 워크로드에 길고 캐시되지 않은 프롬프트가 많은 경우
- 캐시 지역성, 대기열, 대역폭을 고려하는 스케줄러가 있는 경우

요청 대부분이 짧거나 캐시 적중이 많다면, 보통은 모든 것을 한 클러스터에 두는
편이 더 간단하고 빠릅니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving](https://arxiv.org/abs/2401.09670)
- [SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills](https://arxiv.org/pdf/2308.16369)
- [The Five Eras of KVCache](https://www.modular.com/blog/the-five-eras-of-kvcache?utm_source=llm_handbook)
- [Prefill-as-a-Service: KVCache of Next-Generation Models Could Go Cross-Datacenter](https://arxiv.org/abs/2604.15039)
</LinkList>
