---
sidebar_position: 1
description: GPU와 서빙 런타임부터 스케일링, 라우팅, 관측 가능성, 그리고 직접 구축할지 구매할지의 선택까지 LLM 추론 인프라 스택을 살펴봅니다.
keywords:
    - LLM infrastructure
    - LLM inference infrastructure
    - LLM inference stack
    - LLM infrastructure solutions
    - Inference platforms
    - Self-hosted LLM inference
---

import LinkList from '@site/src/components/LinkList';

# LLM 추론 인프라란?

LLM 인프라는 LLM을 학습, 미세조정, 평가, 서빙하는 데 쓰이는 하드웨어,
소프트웨어, 운영 관행의 집합입니다.

이 페이지는 그중 학습된 모델을 프로덕션에서 실행하는 부분인
**LLM 추론 인프라**를 다룹니다. LLM 추론 인프라에는 모델을 실행하는 GPU, 토큰을
생성하는 런타임, 그리고 배포, 스케일링, 라우팅, 모니터링을 위한 시스템이
포함됩니다. 이 구성 요소들이 함께 작동하여 트래픽이 변해도 모델을 계속 사용할 수
있게 하고, 지연 시간, 처리량, 비용 목표를 달성하도록 돕습니다.

## 범용 클라우드 스택이 부족한 이유 {#why-general-purpose-cloud-stacks-fall-short}

표준 클라우드 네이티브 인프라는 상태가 없는(stateless) CPU 서비스를 위해
설계되었습니다. LLM 서빙은 여러 면에서 이 전제를 깨뜨립니다.

- **GPU는 CPU처럼 나눠 쓸 수 없습니다.** 컨테이너가 순간적인 부하에 대응하려고
  "GPU 절반"을 가져갈 수는 없습니다. 가중치가 수 기가바이트에 이르는 모델은
  장치에 들어가거나 들어가지 않거나 둘 중 하나입니다. 유휴 GPU는 비싸기 때문에,
  콜드 스타트를 피하려고 자원을 과잉 프로비저닝하면 비용이 늘어납니다.
- **레플리카는 상태를 가집니다.** 각 레플리카는 활성 요청을 위해 메모리를 많이
  차지하는
  [KV 캐시](/llm-inference-basics/how-does-llm-inference-work/#what-is-kv-cache)를
  보유합니다. 프리픽스 캐싱을 사용하면 레플리카가 캐시를 유지해 여러 요청에서
  재사용할 수 있으므로, 어느 레플리카가 요청을 처리하는지가 지연 시간에 영향을
  줍니다. 라운드 로빈 방식의 부하 분산은 다른 레플리카에 있는 재사용 가능한
  프리픽스를 놓칠 수 있습니다.
- **요청이 균일하지 않습니다.** 작업량은 프롬프트와 출력 길이에 따라 달라지며,
  어떤 요청은 20개 토큰을 생성하고 다른 요청은 4,000개를 생성할 수 있습니다.
  초당 요청 수만 세어서는 실제 GPU 부하를 거의 알 수 없습니다.
- **시작이 느립니다.** LLM 컨테이너는 크기가 큽니다. 이미지를 내려받고 가중치를
  로드하는 데 시간이 걸리기 때문에, 사후 대응형 오토스케일링은 트래픽 급증이
  이미 타임아웃을 일으킨 뒤에야 용량을 추가합니다. 사용자는 실시간으로 응답을
  기다리고 있으므로 이 지연을 바로 체감합니다.

이러한 한계 때문에 LLM 추론에는 전용으로 설계된 스택이 필요합니다.

## LLM 추론 스택 {#the-llm-inference-stack}

위의 요구 사항이 추론 스택의 각 레이어가 함께 작동하는 방식을 결정합니다. 아래
섹션에서는 스택의 각 부분을 개략적으로 설명하고, 관련된 핸드북 페이지 링크를
함께 제공합니다.

### 컴퓨팅 하드웨어 {#compute-hardware}

하드웨어 레이어는 주로 GPU와 기타 가속기, 메모리, 인터커넥트, 스토리지로
구성됩니다. LLM 추론, 특히 디코드 단계는 순수 FLOPS보다 메모리 용량과 대역폭에
더 크게 제약을 받습니다. 핵심 하드웨어 질문은 장치마다 고대역폭 메모리(HBM)가
얼마나 있는지, 그리고 장치끼리 얼마나 빠르게 통신할 수 있는지입니다.

메모리가 제약 요인인 이유를 두 가지 예로 살펴보겠습니다.

- BF16 형식의 Llama 3.1 70B는 가중치만으로 약 **140 GB**가 필요합니다. NVIDIA
  H100 한 장의 HBM은 **80 GB**이므로, 요청을 하나라도 처리하려면 먼저 모델을
  최소 두 장의 H100 GPU에 나눠 올려야 합니다.
- 같은 70B 모델의 KV 캐시는 **토큰당 약 320 KB**를 사용합니다. 128K 토큰
  컨텍스트에서는 요청 하나가 가중치와 별도로 약 **40 GB**의 캐시를 차지할 수
  있습니다.

모델을 여러 GPU에 나눠 올리기 때문에 인터커넥트 대역폭도 서빙의 병목이 됩니다.

모델 크기와 지연 시간 목표에 맞춰
[GPU를 선택](/getting-started/choosing-the-right-gpu/)할 때는 다음을 고려하세요.

- [온프레미스](/getting-started/on-prem-llms/)에서 실행할지,
  [BYOC](/getting-started/bring-your-own-cloud/)로 자체 클라우드 계정에서
  실행할지, 관리형 플랫폼에서 실행할지
- 특히 트래픽이 정점일 때 단일 리전으로 수요를 감당할 수 없다면
  [여러 리전과 공급자](/infrastructure-and-operations/multi-cloud-and-cross-region-inference/)에
  걸쳐 GPU를 어떻게 확보할지

### 서빙 런타임 {#serving-runtime}

서빙 런타임은 추론 엔진 또는 추론 프레임워크라고도 하며, 모델 가중치를 GPU에
로드하고 요청을 받아 토큰을 생성하는 소프트웨어입니다. vLLM, SGLang, MAX 등이 그
예입니다.

런타임은 각 GPU가 요청을 얼마나 효율적으로 처리하는지를 결정합니다. 런타임마다
지원하는 [최적화 기법](/inference-optimization/)과 그 구현 수준이 다릅니다.
이러한 기법으로는
[연속 배칭](/inference-optimization/static-dynamic-continuous-batching/),
[PagedAttention](/inference-optimization/pagedattention/),
[프리픽스 캐싱](/inference-optimization/prefix-caching/),
[양자화](/model-preparation/llm-quantization/),
[추측 디코딩](/inference-optimization/speculative-decoding/),
[병렬화](/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/)
등이 있습니다.

자세한 내용은
[적합한 추론 프레임워크 선택하기](/getting-started/choosing-the-right-inference-framework/)를
참고하세요.

### 모델 패키징과 배포 {#model-packaging-and-deployment}

이 레이어는 모델 가중치를 배포 가능한 산출물로 만듭니다. 컨테이너 이미지, 모델
파일, [미세조정된](/model-preparation/llm-fine-tuning/) 어댑터, 런타임 구성,
그리고 모델을 API로 감싸는 서빙 코드가 여기에 포함됩니다.

패키징 방식은 콜드 스타트 시간에 영향을 줍니다. LLM 컨테이너 이미지는 크기가
크고, 모델 가중치는 수십 기가바이트에서 조 단위 매개변수 모델의 경우
1테라바이트를 넘기도 합니다. 그래서 새 노드에서 이미지를 내려받고 가중치를
로드하는 데 오랜 시간이 걸린 뒤에야 첫 요청을 처리할 수 있습니다. 이 단계에서
내리는 결정(예: 가중치를 이미지에 포함할지, 오브젝트 스토리지에서
스트리밍할지)에 따라 오케스트레이션 레이어가 용량을 얼마나 빨리 추가할 수
있는지가 정해집니다.

### 오케스트레이션과 스케일링 {#orchestration-and-scaling}

오케스트레이션 레이어는 레플리카를 몇 개, 어디에서 실행할지, 그리고 언제
추가하거나 제거할지를 결정합니다. 대부분의 프로덕션 배포에서는 Kubernetes와
GPU를 인식하는 오토스케일러를 함께 사용합니다.

LLM 워크로드는 표준 오토스케일링이 전제하는 가정에 맞지 않습니다. 콜드 스타트가
좋은 예입니다. GPU 노드를 프로비저닝하고, 이미지를 내려받고, 가중치를 로드하는
데 오랜 시간이 걸릴 수 있습니다. 트래픽 급증에 대응해 추가한 용량은 급증이 이미
타임아웃을 일으킨 뒤에 도착하는 경우가 많습니다. 일반적인 스케일링 신호도 도움이
되지 않습니다. CPU 사용률은 GPU 바운드 워크로드에 대해 알려 주는 것이 거의 없고,
GPU 사용률은 더 큰 배치를 처리할 여유가 있는 장치도 바쁘다고 보고합니다. 두 지표
모두 오토스케일러에게 언제 용량을 추가해야 하는지 확실하게 알려 주지 못합니다.

배치(placement) 문제도 또 다른 예입니다. 레플리카 하나가 여러 GPU나 노드에 걸쳐
있을 수 있으며,
[프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)를
적용하면 한 요청의 프리필 단계와 디코드 단계가 각각 독립적으로 스케일링되는
별도의 GPU 풀에서 실행됩니다. 오케스트레이터는 이 풀들 사이에 적절한
인터커넥트가 있도록 배치하고, KV 캐시 전송을 빠르게 유지하며, 각 풀을 알맞은
신호에 따라 스케일링해야 합니다.

자세한 내용은 다음 섹션을 참고하세요.

- [빠른 스케일링](/infrastructure-and-operations/fast-scaling/)은 콜드 스타트
  문제와 스케일링 기준으로 삼을 지표를 설명합니다.
- [분산 추론](/infrastructure-and-operations/distributed-inference/)은 하나의
  모델을 여러 GPU, 노드, 리전에 걸쳐 실행하는 방법을 다룹니다.
- [멀티 클라우드 및 리전 간 추론](/infrastructure-and-operations/multi-cloud-and-cross-region-inference/)은
  가용성, 규정 준수, 비용을 위해 여러 공급자에 용량을 배치하는 방법을 다룹니다.

### 게이트웨이와 라우팅 {#gateway-and-routing}

게이트웨이는 애플리케이션 트래픽의 진입점입니다. 게이트웨이는 API(보통
[OpenAI 호환](/model-interaction/openai-compatible-api/) 또는
[Anthropic 호환](/model-interaction/anthropic-compatible-api/) 엔드포인트)를
노출하고, 인증과 요청 속도 제한을 적용하며, 각 요청을 어느 레플리카가 처리할지
결정합니다.

라우팅 결정은 모델이 처리해야 하는 작업량에 영향을 줄 수 있습니다. 각 레플리카는
KV 캐시 상태를 보유하므로, 일치하는 프롬프트 프리픽스가 캐시에 있는 레플리카로
요청을 보내면 수천 개의 프롬프트 토큰을 다시 계산하지 않아도 됩니다.
[추론 라우팅](/inference-optimization/inference-routing/)에서는 프리픽스 인식,
캐시 인식, 프리필/디코드 인식 라우팅 전략을 다룹니다. 애플리케이션이 여러 모델을
연결해 사용하는 경우, 게이트웨이가
[멀티 모델 추론 파이프라인](/infrastructure-and-operations/multi-model-inference-pipelines/)을
조율하기도 합니다.

### 관측 가능성 {#observability}

관측 가능성(observability)은 스택 전반에서 지표, 로그, 트레이스, 이벤트를
수집하여 시스템이 무엇을 하고 있으며 왜 그런지 파악할 수 있게 합니다. 요청률이나
오류율 같은 일반 애플리케이션 지표도 여전히 중요하지만, LLM 추론에는 표준
대시보드가 추적하지 않는 지표가 필요합니다.

- TTFT(Time to First Token)와 ITL(Inter-Token Latency)
- 레플리카별, 요청별 TPS(Tokens per Second)
- 대기열 대기 시간과 처리 중인 요청 수
- KV 캐시 사용률과 프리픽스 캐시 적중률
- GPU 메모리 사용량과 메모리 부족(out-of-memory) 이벤트

이러한 측정값이 없으면 낮은 GPU 사용률이나 증가하는 TTFT의 원인을 진단하는 일은
추측에 그치게 됩니다. 주요 지연 시간 및 처리량 지표에 대한 자세한 내용은
[LLM 추론 지표](/llm-inference-basics/llm-inference-metrics/)를 참고하세요.

관측 가능성 레이어는 **서빙 비용 관리**도 지원해야 합니다. 많은 팀이 GPU
청구서를 받기 전까지 이 부분을 간과합니다. 서빙 비용 관리란 워크로드별 비용을
추적하고, 비용 목표에 맞춰 모델 선택, 하드웨어, 배칭, 라우팅, 스케일링 정책을
조정하는 것을 뜻합니다. 추론 비용은 지속적으로 발생하며 토큰 사용량이나
프로비저닝된 GPU 용량에 따라 늘어나기 때문에 이 관리가 중요합니다. 출시 시점에는
부담이 적던 모델도 트래픽이 늘면 가장 큰 비용 항목이 될 수 있습니다. 워크로드별
비용 데이터가 없으면 어떤 모델, 테넌트, 스케일링 정책이 원인인지 알 수 없습니다.

자세한 내용은
[LLM 관측 가능성](/infrastructure-and-operations/comprehensive-observability/)을
참고하세요.

### 추론 운영 {#inference-operations}

추론 운영은 모델, 트래픽, 팀이 바뀌어도 스택이 계속 돌아가게 합니다. 여기에는
모델 릴리스를 위한 CI/CD, 카나리 및 블루-그린 롤아웃, 자동 롤백, 접근 제어, 비용
귀속, 유휴 GPU 정리 같은 관행이 포함됩니다.

이러한 워크플로는 모델 업데이트가 반복되어도 배포가 안정적으로 유지되도록
돕습니다.
[InferenceOps와 관리](/infrastructure-and-operations/inferenceops-and-management/)에서
워크플로를 설명하고,
[추론 비용](/infrastructure-and-operations/build-and-maintenance-cost/)에서는
이를 운영하고 유지하는 데 필요한 인력과 노력을 다룹니다.

### 보안과 규정 준수 {#security-and-compliance}

보안과 규정 준수는 스택 전체에 적용됩니다. 표준 애플리케이션 통제와 더불어, 각
레이어에는 LLM에 특화된 책임이 있습니다.

- **게이트웨이**: 인증, 테넌트별 요청 속도 제한, 그리고 프롬프트 인젝션,
  탈옥(jailbreak), 비밀 정보 유출에 대비한 입력 및 출력 필터링
- **오케스트레이션**: 데이터 상주(data residency) 요건과 GDPR 같은 국가 간
  데이터 이전 규정을 충족하는 리전에 레플리카를 배치하고, 클러스터 또는 노드
  수준에서 테넌트를 격리
- **패키징과 런타임**: 모델 가중치, 어댑터, 미세조정 데이터에 누가 접근할 수
  있는지 통제하고, 배포 전에 산출물의 출처를 검증
- **관측 가능성**: 디버깅을 위해 프롬프트와 출력을 기록하되, 민감한 데이터를
  정책이 허용하는 기간보다 오래 보관하지 않음
- **운영**: 프로덕션 모델에 대한 역할 기반 접근 제어, 모든 배포에 대한 감사
  로그, 모델 변경에 대한 승인 워크플로

많은 팀이 프롬프트와 데이터를 자체 환경 안에 두기 위해 셀프 호스팅을 선택합니다.
모든 레이어에는 그 경계를 지키는 통제 장치가 필요합니다.

## 추론 인프라 솔루션: 구축, 구매, 또는 둘 다 {#inference-infrastructure-solutions-build-buy-or-both}

추론 인프라를 마련하는 일반적인 방법은 세 가지입니다.

| 방식                          | 직접 관리하는 부분                             | 적합한 경우                                                        | 트레이드오프                                                            |
|-------------------------------|------------------------------------------------|--------------------------------------------------------------------|-------------------------------------------------------------------------|
| **서버리스 LLM API**          | API 호출 아래로는 없음                         | 프로토타입, 사용량이 적거나 급변하는 경우, 독점 프런티어 모델      | 규모가 커지면 토큰당 비용 부담, 제한된 통제권, 데이터가 VPC 밖으로 나감 |
| **관리형 추론 플랫폼**        | 모델과 애플리케이션 코드(스택은 플랫폼이 운영) | 오픈 모델을 셀프 호스팅하되 모든 레이어에 인력을 두고 싶지 않은 팀 | 플랫폼 적합성과 가격, 일부 레이어는 플랫폼 방식이 정해져 있음           |
| **Kubernetes 기반 자체 구축** | 모든 레이어                                    | 인프라 엔지니어링 역량이 뛰어나고 맞춤화 요구가 엄격한 팀          | 엔지니어링 및 유지 관리 비용이 가장 높고, 프로덕션까지 가장 오래 걸림   |

각 방식을 추론 스택에 대응시켜 보면 차이를 다른 각도에서 볼 수 있습니다. 직접
관리하는 레이어가 많을수록 통제권은 커지지만 떠안는 엔지니어링 작업도
늘어납니다.

<figure>
  <img className="theme-logo-light" src={require('@site/docs/infrastructure-and-operations/img/inference-solutions-stack.png').default} alt="" />
  <img className="theme-logo-dark" src={require('@site/docs/infrastructure-and-operations/img/inference-solutions-stack-dark.png').default} alt="" />
  <figcaption>
    <b>그림 1.</b> 각 방식에서 직접 관리하는 추론 스택 레이어
  </figcaption>
</figure>

결정은 보통 사용량, 통제권, 인력 구성으로 귀결됩니다. 아래 섹션에서는 각 방식이
실제로 어떤 모습인지 설명합니다.

### 서버리스 LLM API {#serverless-llm-apis}

호스팅된 엔드포인트로 요청을 보내고 토큰 단위로 비용을 지불합니다. 공급자가
하드웨어를 고르고, 서빙 런타임을 실행하고, 용량을 스케일링하므로 사용자는 스택을
전혀 볼 일이 없습니다. 이 범주에는 OpenAI와 Anthropic 같은 독점 모델 API뿐
아니라, Together AI와 Fireworks AI 같은 공급자가 제공하는 오픈 모델용 서버리스
엔드포인트도 포함됩니다. 가장 빠르게 시작할 수 있는 방법인 경우가 많지만,
사용량이 많고 꾸준하다면 사용률이 높은 전용 배포보다 비용이 더 들 수 있습니다.

전체 비교는
[서버리스 vs. 셀프 호스팅 LLM 추론](/getting-started/serverless-vs-self-hosted-llm-inference/)을
참고하세요.

### 관리형 추론 플랫폼 {#managed-inference-platforms}

사용자는 자체 모델 가중치와 애플리케이션 코드를 가져오고, 플랫폼이 서빙 런타임,
라우팅, 스케일링, 관측 가능성을 대신 운영합니다. 플랫폼은 vLLM이나 MAX 같은 서빙
런타임과 같지 않다는 점에 유의하세요. 플랫폼은 사용자를 대신해 런타임과 그 주변
레이어를 함께 운영합니다.

관리형 추론 플랫폼은 보통 두 가지 형태로 제공됩니다.

- 호스팅 플랫폼에서는 공급자가 GPU를 제공하고, 요청과 데이터는 공급자에게
  전달됩니다. Fireworks AI나 Together AI의 전용 배포가 여기에 해당합니다.
- [BYOC](/getting-started/bring-your-own-cloud/) 플랫폼에서는 공급자가 사용자의
  클라우드 계정 안에서 GPU를 관리합니다. 따라서 클라우드 인프라는 사용자에게
  남아 있고, 스택은 여전히 플랫폼이 처리합니다.

<ContactCallout>

자체 배포를 위해 이러한 선택지를 검토하고 있다면 기꺼이 함께 이야기를
나누겠습니다. 저희는 위에서 설명한 런타임, 라우팅, 스케일링, 관측 가능성
레이어를 모두 아우르는 추론 플랫폼을 구축했습니다. 이 플랫폼을 사용하면 모든
레이어에 직접 인력을 두지 않고도 자체 GPU에서 오픈 모델을 실행할 수 있습니다.

</ContactCallout>

### Kubernetes 기반 자체 구축 {#self-built-on-kubernetes}

GPU 노드, vLLM, SGLang, MAX 같은 오픈 서빙 런타임, 오토스케일러, 라우터, 관측
가능성, 릴리스 도구까지 모든 레이어를 직접 조립합니다. 이 방식은 통제권을 최대로
확보할 수 있으며, 어떤 플랫폼도 충족하지 못하는 맞춤화 요구가 있는 팀이 주로
선택합니다. 동시에 엔지니어링 작업과 지속적인 유지 관리 부담도 가장 큽니다.
[추론 비용](/infrastructure-and-operations/build-and-maintenance-cost/)에서 그
작업에 무엇이 필요한지 다룹니다.

## 자주 묻는 질문 {#faqs}

### LLM 인프라를 직접 갖춰야 하나요, 아니면 API만 사용해도 되나요? {#do-i-need-my-own-llm-infrastructure-or-can-i-just-use-an-api}

사용량이 적고 데이터가 자체 환경 밖으로 나가도 괜찮다면 API로 시작하세요. 규모가
커져 토큰 비용이 오픈 모델 운영 비용을 넘어설 때, 규정 준수 요건이 적용될 때,
또는 지연 시간과 맞춤화를 직접 통제해야 할 때 자체 인프라로 옮기세요. 많은 팀이
일부 워크로드에는 API를, 다른 워크로드에는 셀프 호스팅 모델을 쓰는 식으로 둘 다
운영합니다.

### 기존 Kubernetes 클러스터에서 LLM 추론을 실행할 수 있나요? {#can-i-run-llm-inference-on-my-existing-kubernetes-cluster}

네, 그리고 대부분의 셀프 호스팅 배포가 그렇게 합니다. Kubernetes는 GPU 노드에
Pod를 스케줄링하고, 새 버전을 롤아웃하고, 실패한 레플리카를 재시작하는 등
범용적인 부분을 잘 처리합니다. 하지만 LLM을 인식하는 기능은 제공하지 않습니다.
CPU가 아니라 요청 동시성을 기준으로 스케일링하는 오토스케일러, 단순 라운드 로빈
분산 대신 KV 캐시 지역성을 이해하는 라우터, 그리고 새 노드에 모델 가중치를
빠르게 로드하는 경로를 직접 추가해야 합니다. 엔지니어링 노력의 대부분이 바로 이
구성 요소들에 들어갑니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [Deploying Disaggregated LLM Inference Workloads on Kubernetes](https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/)
- [Why LLM Inference Needs a New Kind of Router - Part 1](https://www.modular.com/blog/why-llm-inference-needs-a-new-kind-of-router-part-1)
</LinkList>
