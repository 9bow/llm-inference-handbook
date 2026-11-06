---
sidebar_position: 2
description: 셀프 호스팅 LLM 추론 비용은 GPU 비용에서 끝나지 않습니다. GPU 연산부터 엔지니어링, 유지보수, 인력까지 추론 비용을 결정하는 요소를 살펴봅니다.
keywords:
    - Inference cost
    - LLM inference cost, AI inference cost
    - Self-hosted LLM cost, LLM infrastructure cost
---

import LinkList from '@site/src/components/LinkList'; import
InferenceCostCalculator from '@site/src/components/InferenceCostCalculator';

# 추론 비용

추론 비용은 학습된 LLM을 실행하고 프로덕션에서 요청을 서빙하는 데 드는
비용입니다. OpenAI나 Anthropic 같은 서버리스 추론 서비스를 쓴다면 대부분 토큰당
가격입니다. 셀프 호스팅 추론이라면 인프라를 소유하거나 임대하고 모델을 직접
운영하는 데 드는 전체 비용입니다.

보통 가장 눈에 띄는 부분은 GPU 요금입니다. 예를 들어 2026년 10월 기준으로 AWS는
US East 리전에서 8×H100 `p5.48xlarge` 인스턴스를 Capacity Block으로 시간당 약
\$47.76에 제공합니다. 노드 하나를 24시간 내내 돌리면 한 달에 약 \$34,900입니다.
이 노드를 둘러싼 엔지니어링, 인력, 유지보수 비용은 잘 드러나지 않지만, 셀프
호스팅이 수지에 맞는지를 결정하는 경우가 많습니다.

## 추론 비용을 측정하는 방법 {#how-to-measure-inference-cost}

팀은 보통 다음 세 가지 단위 중 하나로 비용을 비교합니다.

- **100만 토큰당 비용**: API 가격의 표준 단위입니다. 셀프 호스팅 모델이라면
  시간당 GPU 비용을 배포가 시간당 처리하는 토큰 수(100만 단위)로 나눕니다.
- **요청당 비용**: 에이전트나
  [다중 모델 파이프라인](/infrastructure-and-operations/multi-model-inference-pipelines/)처럼
  사용자 요청 하나가 여러 번의 모델 호출을 일으킬 때 유용합니다.
- **GPU 시간당 비용**: 인프라 요금 그 자체입니다. GPU를 실제로 얼마나 잘
  쓰는지는 반영하지 않으므로 API 가격과 바로 비교할 수 없습니다.

간단한 출발점은 GPU 비용만 고려한 100만 토큰당 비용입니다.

$$
\text{Cost per million tokens}
= \frac{\text{GPU cost per hour} \times 1{,}000{,}000}
       {\text{Tokens per second} \times 3{,}600}
$$

위의 AWS 8×H100 노드가 실제 트래픽에서 초당 10,000 토큰을 꾸준히 처리한다고
가정해 보겠습니다. 시간당 3,600만 토큰을 처리하므로 비용은 100만 토큰당 약
\$1.33입니다. 이 속도로 요청을 서빙하는 시간이 전체의 30%에 그친다면(전체 시간
평균으로 초당 약 3,000 토큰), 같은 노드의 비용은 100만 토큰당 약 \$4.42가
됩니다.

추론 비용을 가장 정확하게 계산하려면 최고 벤치마크 수치가 아니라 프로덕션
트래픽에서 꾸준히 유지되는
[처리량](/llm-inference-basics/llm-inference-metrics/#throughput)을 기준으로
초당 토큰 수를 측정해야 합니다. 처리량은 모델, 하드웨어, 배칭 전략, 지연 시간
목표에 따라 달라지므로 자신의 워크로드로 직접 측정하세요. 측정 방법은
[LLM 성능 벤치마크](/inference-optimization/llm-performance-benchmarks/)를
참고하세요. 입력 토큰과 출력 토큰은 비교 대상인 API 가격과 같은 방식으로 세어야
합니다.

## 셀프 호스팅 추론의 숨은 비용 {#hidden-costs-of-self-hosted-inference}

위 공식은 GPU 연산 비용, 즉 GPU에 대해 시간 단위로 지불하는 가격만 반영합니다.
아래 표에 셀프 호스팅 LLM 추론의 주요 비용 항목을 정리했습니다. GPU 연산 외의
항목은 모두 놓치기 쉽습니다. 서버리스 추론 서비스를 쓰면 이 항목 대부분이 하나의
토큰당 가격으로 합쳐집니다.

| 비용 항목               | 포함 내용                                                                 | 결정 요인                                              |
|-------------------------|---------------------------------------------------------------------------|--------------------------------------------------------|
| **GPU 연산**            | 온디맨드, 예약, 스팟 GPU 인스턴스                                         | GPU 종류, GPU 수, 요금 모델, 리전                      |
| **유휴 용량**           | 요청을 기다리며 보낸 유료 GPU 시간(GPU 연산 비용에 이미 포함됨)           | 트래픽 변동성, 스케일링 속도, 스케일 투 제로 지원 여부 |
| **스토리지와 네트워크** | 모델 가중치 저장, 컨테이너 레지스트리, 리전 간 데이터 전송                | 모델 크기, 레플리카와 리전 수                          |
| **엔지니어링 구축**     | 서빙 런타임, 오토스케일링, 라우팅, 관측 가능성 구성                       | 스택을 얼마나 직접 구축하는지                          |
| **유지보수**            | 프레임워크 업그레이드, 새 모델 지원, CUDA 및 드라이버 업데이트, 보안 패치 | 모델과 프레임워크의 출시 속도                          |
| **운영**                | 온콜, 장애 대응, 용량 계획                                                | 가동 시간 목표, 배포 수                                |
| **인력**                | 전문 엔지니어 채용, 교육, 유지                                            | 시장 임금 수준, 팀 규모                                |

추론 비용을 더 정확하게 추정하려면 배포를 운영하는 총비용에서 시작하세요.

$$
\text{Total cost}
= \text{GPU cost} + \text{Infra cost} + \text{Ops cost}
+ \text{Eng and talent cost} + \cdots
$$

그런 다음 같은 기간(예: 한 달) 동안 서빙한 토큰 수로 나눕니다.

$$
\text{Cost per million tokens}
= \frac{\text{Total cost} \times 1{,}000{,}000}{\text{Tokens served}}
$$

아래 계산기는 시간당 \$2인 GPU 1개와 최대 부하에서 초당 2,500 토큰의 지속
처리량을 초깃값으로 사용합니다. 자신의 배포에 맞게 입력값을 바꿔 보세요.

<InferenceCostCalculator />

같은 GPU라도 평균 처리량에 따라 경제성이 크게 달라집니다. 처리량이 낮으면 유휴
용량에 돈을 내게 됩니다. 처리량이 높으면 GPU 요금과 다른 고정 비용이 훨씬 많은
토큰에 나뉘어 분산됩니다.

다음 섹션에서는 추론 비용을 구성하는 요소를 자세히 살펴봅니다.

## GPU 연산 비용 {#gpu-compute-cost}

GPU 연산은 대개 셀프 호스팅 추론 비용에서 가장 큰 항목입니다. 시간당 가격은 GPU
종류, 클라우드 공급자, 리전, 약정 수준(온디맨드, 예약, 스팟)에 따라 달라집니다.

다음은 US East 리전의 AWS Capacity Block 가격으로, GPU 세대에 따라 요금이 얼마나
달라지는지 보여 줍니다.

| 인스턴스           | GPU    | 인스턴스 시간당 가격 | GPU 시간당 가격 |
|--------------------|--------|----------------------|-----------------|
| `p5.48xlarge`      | 8×H100 | \$47.76              | \$5.97          |
| `p5en.48xlarge`    | 8×H200 | \$63.16              | \$7.90          |
| `p6-b200.48xlarge` | 8×B200 | \$113.67             | \$14.21         |

*출처: [AWS EC2 Capacity Blocks
pricing](https://aws.amazon.com/ec2/capacityblocks/pricing/), 2026년 10월.
가격은 바뀔 수 있으므로 예산을 세우기 전에 공급자에게 확인하세요.*

새 GPU는 시간당 가격이 더 비싸지만, 처리량이 충분히 더 높다면 토큰당 비용은 더
낮을 수 있습니다. 모델에 맞는 하드웨어 규모를 정하려면
[적합한 GPU 선택하기](/getting-started/choosing-the-right-gpu/)와
[LLM에 필요한 GPU 메모리 계산하기](/getting-started/calculating-gpu-memory-for-llms/)를
참고하세요.

### GPU 용량 사용이 중요한 이유 {#why-gpu-capacity-use-matters}

GPU가 요청을 서빙하든 유휴 상태로 있든 비용은 똑같이 나갑니다. 이 간단한 예에서
용량 사용이란 비용을 낸 시간 중 가정한 최대 부하 처리량으로 서빙한 시간의 비율을
뜻합니다. 위의 H100 요금 \$5.97을 기준으로 보면 다음과 같습니다.

| 비용을 낸 시간 중 서빙 비율 | 실제 서빙한 GPU 시간당 실효 비용 |
|-----------------------------|----------------------------------|
| 100%                        | \$5.97                           |
| 70%                         | \$8.53                           |
| 30%                         | \$19.90                          |
| 5%                          | \$119.40                         |

대부분의 팀은 최대 트래픽에 맞춰 용량을 확보하므로, 트래픽이 적은 시간대에는
GPU가 놀게 됩니다.
[빠른 스케일링](/infrastructure-and-operations/fast-scaling/)과 스케일 투
제로(scale-to-zero)는 이런 낭비를 줄입니다. 또한
[연속 배칭](/inference-optimization/static-dynamic-continuous-batching/) 같은
일부 추론 최적화 기법은 GPU가 실행되는 동안 GPU마다 더 많은 일을 처리하게
합니다.

## 스토리지와 네트워크 비용 {#storage-and-network-cost}

스토리지와 네트워크 비용은 보통 GPU 요금보다 작지만, 모델 크기와 서빙 위치의
수가 늘면 함께 늘어납니다.

이 비용은 주로 다음에서 발생합니다.

- **모델 스토리지**: 가중치, 미세조정된 변형 모델, 이전 버전을 오브젝트
  스토리지나 모델 레지스트리에 보관하는 비용입니다.
- **컨테이너 레지스트리**: CUDA 라이브러리와 서빙 런타임을 포함한 대용량
  이미지를 저장하고 가져오는 비용입니다.
- **데이터 전송**: 새 레플리카마다 가중치를 복사하고, 리전이나 클라우드 사이에
  데이터를 옮기는 비용입니다. 클라우드 공급자는 보통 리전 간 트래픽과 인터넷
  송신(egress) 트래픽에 요금을 부과합니다.
- **고속 로컬 스토리지**: 새 레플리카가 가중치를 빠르게 불러올 수 있도록
  처리량이 높은 디스크나 캐시를 마련하는 비용입니다.

스토리지와 네트워크 성능은 GPU 비용에도 영향을 줍니다. 새 레플리카가 가중치를
내려받는 데 몇 분이 걸리면
[콜드 스타트](/infrastructure-and-operations/fast-scaling/#the-cold-start-problem)를
기다리거나, 여유분으로 GPU를 추가로 계속 실행해 두어야 합니다.
[멀티 클라우드 또는 리전 간 구성](/infrastructure-and-operations/multi-cloud-and-cross-region-inference/#what-data-transfer-costs-come-with-multi-cloud-inference)에서는
리전을 하나 추가할 때마다 가중치 사본이 하나 더 필요하고 데이터 전송도
늘어납니다. [양자화](/model-preparation/llm-quantization/)는 가중치 크기를 줄여
두 가지를 모두 줄입니다.

## 엔지니어링 비용 {#engineering-cost}

셀프 호스팅 LLM 추론 인프라를 구축하려면 첫 프로덕션 요청을 받기 전부터 상당한
엔지니어링 작업이 필요합니다. 그리고 그 작업 대부분은 모델 자체와 관계가
없습니다. 프로덕션 구성에는 보통 다음 작업이 포함됩니다.

- 흔히 공급이 부족하고 일부 리전에서만 쓸 수 있는 고성능 GPU 확보
- CUDA 버전 호환성과 드라이버 의존성 관리
- [오토스케일링](/infrastructure-and-operations/fast-scaling/), 동시성 제어,
  스케일 투 제로 동작 구성
- [프리픽스 캐싱](/inference-optimization/prefix-caching/)과
  [프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)
  같은 추론 최적화 기법 적용
- 전처리, 후처리, 커스텀 API 등 복잡한 AI 시스템에 LLM 통합
- GPU 모니터링, 요청 추적, 장애 감지를 위한
  [관측 가능성](/infrastructure-and-operations/comprehensive-observability/)
  구성
- 스트리밍, 캐싱, [라우팅](/inference-optimization/inference-routing/) 같은
  모델별 동작 처리
- 모델이 GPU 하나에 들어가지 않을 때
  [분산 추론](/infrastructure-and-operations/distributed-inference/) 실행

[범용 클라우드 네이티브 스택](/infrastructure-and-operations/what-is-llm-inference-infrastructure/#why-general-purpose-cloud-stacks-fall-short)은
이런 요구를 염두에 두고 설계되지 않았습니다. LLM 워크로드를 범용 스택에 억지로
맞추면 성능이 떨어지고 준비 기간이 길어지기 쉽습니다.

팀이 이 일을 해내더라도 기회비용이 생깁니다. 인프라 구축에 쓴 한 주는 모델을
개선하거나 제품 기능을 출시하는 데 쓰지 못한 한 주입니다. AI 팀에게 이 비용은
GPU 요금만큼이나 실질적입니다.

## 유지보수 비용 {#maintenance-cost}

유지보수 비용은 모델과 프레임워크의 출시 속도에 비례해 늘어나는데, LLM 추론
분야에서는 그 속도가 매우 빠릅니다. 새로운 모델 아키텍처와 런타임 버전은 몇
달마다, 새로운 GPU 세대는 대략 해마다 나옵니다.

많은 AI 스택은 PyTorch, vLLM, Transformers 같은 모델 런타임을 특정 버전에
고정합니다. 이렇게 하면 배포를 재현하기 쉽고, 런타임이 다른 인프라 구성 요소와
호환된 상태를 유지합니다. 배포는 단순해지지만 지속적인 비용이 생깁니다.

- **느린 모델 도입**: 지원 목록에 없는 새 모델이나 프레임워크 버전은 배포하기
  전에 테스트, 이미지 재빌드, 호환성 확인을 거쳐야 합니다.
- **늘어나는 기술 부채**: 업그레이드를 건너뛸 때마다 스택과 커뮤니티 또는 벤더
  업데이트 사이의 격차가 벌어지고, 다음 업그레이드는 더 어려워집니다.
- **경쟁력 저하**: LLM 배포가 느리면 최신 모델을 바로 시험해 볼 수 있는 팀에
  뒤처집니다.

프레임워크 업그레이드 외에도 팀은 CUDA와 드라이버 업데이트, 보안 패치, 새 GPU
종류로의 마이그레이션까지 처리해야 합니다. 일관된
[InferenceOps 관행](/infrastructure-and-operations/inferenceops-and-management/)이
이 작업을 줄여 주지만, 여전히 누군가는 책임지고 맡아야 합니다.

## 인력 비용 {#talent-cost}

전문 인력은 추론 비용에서 가장 과소평가되기 쉬운 부분입니다. LLM 인프라를
운영하려면 GPU, Kubernetes, ML 프레임워크, 분산 시스템을 이해하는 엔지니어가
필요합니다. 이런 역량을 한 사람이 모두 갖춘 경우는 드물고, 수요도 많습니다.

채용은 시작일 뿐입니다. 24시간 연중무휴 가동 시간 목표가 있는 프로덕션
서비스에는 온콜 순번이 필요하고, 이는 둘 이상의 엔지니어가 전체 스택을 이해해야
한다는 뜻입니다. 이들을 교육하고 스택이 바뀔 때마다 지식을 최신 상태로 유지하는
일은 지속적인 투자입니다.

## 추론 비용을 줄이는 방법 {#how-to-reduce-inference-cost}

추론 비용 절감은 대부분 GPU를 더 효율적으로 쓰고 필요한 용량에만 비용을 내는
데서 나옵니다. 흔히 쓰는 방법은 다음과 같습니다.

- **하드웨어 규모 맞추기**: GPU 종류와 수를 모델과 트래픽에 맞춥니다.
  [적합한 GPU 선택하기](/getting-started/choosing-the-right-gpu/)를 참고하세요.
- **모델 양자화**: [LLM 양자화](/model-preparation/llm-quantization/)는 메모리
  사용량을 줄여, 더 적거나 더 작은 GPU로 모델을 서빙할 수 있게 해 줍니다.
- **요청 배칭**:
  [연속 배칭](/inference-optimization/static-dynamic-continuous-batching/)은
  GPU당 처리량을 높일 수 있습니다.
- **연산 재사용**: [프리픽스 캐싱](/inference-optimization/prefix-caching/)은
  공유 컨텍스트를 다시 계산하지 않게 해 줍니다.
  [KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/)은 재사용할 수
  있는 캐시 블록을 GPU 메모리 밖에 보관할 수 있습니다.
- **디코딩 가속**:
  [추측 디코딩](/inference-optimization/speculative-decoding/)은 여러 드래프트
  토큰을 한 번에 검증해 타깃 모델의 디코딩 단계를 줄일 수 있습니다.
- **수요에 맞춘 스케일링**:
  [빠른 스케일링](/infrastructure-and-operations/fast-scaling/)과 스케일 투
  제로는 GPU 유휴 시간을 줄입니다.
- **더 저렴한 용량 활용**:
  [멀티 클라우드 및 리전 간 추론](/infrastructure-and-operations/multi-cloud-and-cross-region-inference/)을
  쓰면 GPU를 구할 수 있고 비용 효율적인 곳 어디서든 워크로드를 실행할 수
  있습니다.
- **급하지 않은 작업은 오프라인으로**:
  [오프라인 배치 추론](/inference-optimization/offline-batch-inference/)은
  대규모 작업을 더 저렴한 용량이나 유휴 용량에서 실행합니다.
- **기능별 비용 추적**:
  [관측 가능성](/infrastructure-and-operations/comprehensive-observability/)을
  갖추면 어떤 모델, 테넌트, 기능이 지출을 일으키는지 알 수 있습니다.

<ContactCallout>

모든 레이어를 직접 구축하고 인력을 배치하고 싶지 않다면, 배포 방안을 기꺼이 함께
논의하겠습니다. 저희는 스케일링, 최적화, 관측 가능성을 처리하는 추론 플랫폼을
구축했습니다. 이 플랫폼으로 자체 GPU에서 오픈 모델을 실행하면서 추론 비용을
통제할 수 있습니다.

</ContactCallout>

## 자주 묻는 질문 {#faqs}

### LLM을 셀프 호스팅하는 것이 API를 쓰는 것보다 저렴한가요? {#is-self-hosting-an-llm-cheaper-than-using-an-api}

사용량이 많고 꾸준하다면 고정 GPU 비용이 많은 토큰에 분산되므로 LLM 셀프
호스팅이 API보다 저렴할 수 있습니다. 사용량이 적거나 들쭉날쭉하다면 유휴 GPU와
엔지니어링 부담 때문에 보통 API가 더 저렴합니다. 현실적인 사용률에서 100만
토큰당 비용을 계산하고, 엔지니어링 비용과 인력 비용을 더한 다음, 그 총액을 API
가격과 비교하세요. 전체 비교는
[서버리스 vs. 셀프 호스팅 LLM 추론](/getting-started/serverless-vs-self-hosted-llm-inference/)을
참고하세요. 비교할 때는 모델 품질과 지연 시간을 비슷하게 맞추고, 입력 토큰과
출력 토큰의 비율도 같게 두세요.

### LLM 추론 비용은 떨어지고 있나요? 그렇다면 이유는 무엇인가요? {#are-llm-inference-costs-falling-and-why}

네. 2026년 9월
[Epoch AI](https://epoch.ai/publications/the-plunging-price-of-thought)는
자신들이 조사한 벤치마크에서 일정 수준의 AI 성능에 도달하는 비용이 2023년 이후
해마다 약 13x씩 떨어졌다고 추정했습니다. 참고로 모델 전반의 100만 토큰당 지출을
사용량 가중으로 추적하는
[Silicon Data LLM Token Expenditure Index](https://www.silicondata.com/products/silicon-index/llm-token-expenditure-index)는
2026년 10월 20일 기준 \$0.96이었습니다. 이 지수는 공급자 가격 변화와 모델 구성
변화 모두에 따라 움직일 수 있습니다.

주된 요인은 더 저렴하고 효율적인 하드웨어, 더 나은 서빙 런타임, 저비용 오픈
모델과의 경쟁, 그리고 이전의 더 큰 모델과 같은 품질을 내는 더 작은 모델입니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [Epoch AI: The plunging price of thought](https://epoch.ai/publications/the-plunging-price-of-thought)
- [Silicon Data: LLM Token Expenditure Index](https://www.silicondata.com/products/silicon-index/llm-token-expenditure-index)
</LinkList>
