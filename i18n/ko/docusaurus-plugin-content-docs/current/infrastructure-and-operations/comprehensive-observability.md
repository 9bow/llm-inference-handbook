---
sidebar_position: 4
description: LLM 관측 가능성을 설명합니다. LLM 추론에서 모니터링해야 할 지표, 로그, 트레이스와 문제를 진단하는 방법, 그리고 사용할 만한 도구를 알아봅니다.
keywords:
    - LLM observability, LLM observability tools
    - LLM monitoring, LLM monitoring and observability
    - LLM inference observability, LLM inference monitoring
    - LLM metrics, inference metrics, LLM-specific metrics
    - KV cache usage, prefix cache hit rate, TTFT, ITL, goodput
    - GPU monitoring, DCGM, GPU utilization
    - Prometheus, Grafana, OpenTelemetry, LLM tracing
    - Self-hosted LLM challenges
---

import LinkList from '@site/src/components/LinkList';

# LLM 관측 가능성(observability)

LLM 관측 가능성(observability)이란 LLM 시스템의 모든 레이어에서 지표, 로그,
트레이스, 이벤트를 수집하고 서로 연결해 프로덕션 환경에서 시스템이 어떻게
동작하는지 파악하는 활동입니다. GPU, 추론 엔진, 요청 경로, 그리고 그 위의
애플리케이션까지 모두 대상으로 합니다. 목표는 문제를 일찍 발견하고, 문제가 생긴
이유를 설명하며, 응답을 빠르고 안정적이며 저렴하게 유지하는 것입니다.

관측 가능성이 없으면 지연 시간 급증, 스케일링 문제, 낭비되는 GPU 용량을 진단하는
일이 추측에 의존하게 됩니다. 더 나쁘게는, 캐시 스래싱(cache thrashing)이나 요청
선점(preemption) 같은 조용한 문제가 누군가 알아차리기 전까지 몇 시간 동안 서비스
품질을 떨어뜨릴 수도 있습니다.

이런 위험 때문에 조직들은 투자를 늘리고 있습니다. Gartner는 2028년까지
[LLM 관측 가능성 투자가 생성형 AI 배포의 50%를 차지할 것](https://www.gartner.com/en/newsroom/press-releases/2026-03-30-gartner-predicts-by-2028-explainable-ai-will-drive-llm-observability-investments-to-50-percent-for-secure-genai-deployment)으로
예측합니다. 2026년의 15%에서 늘어난 수치입니다.

이 페이지는 인프라 스택을 직접 소유하는 **셀프 호스팅 LLM 추론**의 관측 가능성을
중심으로 다룹니다.

## LLM 추론에 전용 관측 가능성이 필요한 이유 {#why-llm-inference-needs-dedicated-observability}

LLM 관측 가능성은 일반적인 클라우드 모니터링을 바탕으로 합니다. 평소에 의존하던
지표만으로는 전체 상황을 알 수 없기 때문입니다. 일반 지표가 부족한 이유는 다음과
같습니다.

- **요청이 균일하지 않습니다.** 어떤 요청은 토큰을 20개 생성하고, 다른 요청은
  4,000개를 생성합니다. 초당 요청 수(RPS)로는 GPU의 실제 부하를 거의 알 수
  없습니다. 토큰 수와 동시성이 훨씬 많은 것을 알려 줍니다.
- **지연 시간에는 두 단계가 있습니다.** 프리필은 첫 토큰이 얼마나 빨리
  도착하는지를 결정합니다. 디코드는 나머지 토큰이 얼마나 빨리 스트리밍되는지에
  영향을 줍니다. "요청 지연 시간"이라는 숫자 하나로는 어느 단계가 느린지 알 수
  없습니다.
  [LLM 추론의 동작 방식](/llm-inference-basics/how-does-llm-inference-work/#the-two-phases-of-llm-inference)을
  참고하세요.
- **레플리카에는 상태가 있습니다.** 각 레플리카는 활성 요청을 위한
  [KV 캐시](/llm-inference-basics/how-does-llm-inference-work/#what-is-kv-cache)를
  보유하며, [프리픽스 캐싱](/inference-optimization/prefix-caching/)을 사용하면
  여러 요청에 걸쳐 재사용할 캐시도 보유합니다. 캐시 압박과 캐시 적중률이 지연
  시간을 좌우하지만, 일반적인 웹 대시보드는 둘 다 보여 주지 않습니다.
- **GPU 사용률은 오해를 부를 수 있습니다.** 표준 지표(`nvidia-smi`에 표시되는
  `utilization.gpu` 등)는
  [각 샘플 기간 중 커널이 하나 이상 실행된 시간의 비율](https://nvidia.custhelp.com/app/answers/detail/a_id/3751/~/useful-nvidia-smi-queries)을
  보고합니다. GPU의 SM이 얼마나 점유되었는지, 메모리 대역폭을 얼마나
  사용했는지는 측정하지 않습니다. 따라서 커널이 GPU 가용 자원의 일부만 쓰면서
  계속 실행되어도 GPU는 사용률 100%로 보고될 수 있습니다.
- **장애는 대개 조용히 일어납니다.** 레플리카가 요청을 선점하거나, 캐시 스래싱을
  일으키거나, 더 나쁜 출력을 내는 동안에도 "정상" 상태로 남아 있을 수 있습니다.

## LLM 관측 가능성의 네 가지 신호 {#the-four-signals-of-llm-observability}

LLM 관측 가능성은 크게 네 가지 유형의 텔레메트리에 의존합니다. 유형마다 답하는
질문이 다릅니다.

- **지표**는 TTFT, KV 캐시 사용량, GPU 메모리 같은 수치 시계열입니다. 지금 무슨
  일이 일어나고 있는지 알려 주며, 저장하고 알림을 걸기에 비용이 적게 듭니다.
- **로그**는 컨테이너와 서비스가 남기는 타임스탬프 기록입니다. 메모리
  부족(out-of-memory, OOM) 오류나 잘못된 형식의 입력처럼 특정 요청이나
  프로세스에서 무슨 일이 있었는지 알려 줍니다. 레플리카와 시간대를 가로질러
  검색할 수 있도록 로그를 한곳에 모으세요.
- **트레이스**는 게이트웨이, 라우터, 추론 엔진, 도구 호출 등 여러 서비스를
  거치는 요청 하나를 따라갑니다. 시간이 어디에서 소요되었는지 알려 줍니다.
  사용자 요청 하나가 LLM 호출을 여러 번 일으킬 수 있는
  [멀티 모델 파이프라인](/infrastructure-and-operations/multi-model-inference-pipelines/)과
  에이전트 워크플로에서는 트레이스가 필수입니다.
- **이벤트**는 Pod 재시작, 스케일링 작업, 스케줄링 지연, 모델 롤아웃 같은
  클러스터의 변화를 기록합니다. 시스템이 언제, 그리고 대개 왜 바뀌었는지 설명해
  줍니다.

진짜 가치는 이 신호들을 서로 연관 지을 때 나옵니다. 예를 들어 TTFT 급증(지표)이
스케일 업(이벤트), 느린 가중치 로딩(로그)과 시점이 겹친다면 곧바로 콜드 스타트
문제를 가리킵니다.

## 모니터링 대상: 레이어별 LLM 지표 {#what-to-monitor-llm-metrics-by-layer}

프로덕션 LLM 관측 가능성 스택은 크게 다섯 개의 레이어를 다룹니다. 하위 레이어의
문제는 상위 레이어에서 증상으로 나타나므로 모든 레이어를 볼 수 있어야 합니다.

<figure>
  <Diagram name="llm-observability-layers" alt="" />
  <figcaption><b>그림 1.</b> 클러스터 인프라부터 애플리케이션까지, LLM 관측 가능성의 다섯 레이어.</figcaption>
</figure>

다음 절에서는 가장 아래의 클러스터 수준부터 위로 올라가며 각 레이어를
설명합니다.

### 클러스터와 컨테이너 {#cluster-and-containers}

이 지표들은 서빙 인프라 자체가 정상인지 알려 줍니다.

| **지표**                           | **알 수 있는 것**                                                                               |
|------------------------------------|-------------------------------------------------------------------------------------------------|
| Pod 상태와 재시작                  | 가용성에 영향을 줄 수 있는 실패, 정체, 크래시 루프 상태의 Pod를 탐지합니다.                     |
| 레플리카 수                        | 오토스케일링이 동작하는지 확인하고 스케일링 지연을 디버깅하는 데 도움이 됩니다.                 |
| 스케일링 이벤트와 콜드 스타트 시간 | 이미지 풀과 가중치 로딩을 포함해 새 레플리카가 트래픽을 처리하기까지 걸리는 시간을 보여 줍니다. |
| 리소스 요청과 제한                 | 스케줄링을 조정하고 과다 또는 과소 프로비저닝을 피하는 데 도움이 됩니다.                        |

콜드 스타트가 느리면 트래픽이 몰릴 때 지연 시간이 급증할 수 있습니다. 콜드
스타트를 줄이는 방법은
[빠른 스케일링](/infrastructure-and-operations/fast-scaling/)을 참고하세요.

### GPU {#gpus}

GPU는 스택에서 가장 비싼 부분이므로, GPU 지표가 성능 튜닝과 비용 관리를 모두
이끕니다. 추적할 만한 일반적인 GPU 지표는 다음과 같습니다.

| **지표**                  | **알 수 있는 것**                                                                                                                                                                                           |
|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GPU 사용률                | 샘플 기간 중 커널이 하나 이상 실행된 시간의 비율입니다. 유휴 GPU를 찾는 데는 유용하지만 GPU가 얼마나 바쁜지 측정하는 데는 적합하지 않습니다.                                                                |
| SM 활동                   | 스트리밍 멀티프로세서(SM)마다 하나 이상의 워프가 활성 상태인 시간의 비율을 전체 SM에 대해 평균한 값입니다. GPU가 얼마나 넓게 점유되어 있는지 보여 주지만, 활성 워프가 메모리를 기다리고 있을 수도 있습니다. |
| 텐서 코어 활동            | 행렬 연산 하드웨어를 얼마나 사용하고 있는지 보여 줍니다.                                                                                                                                                    |
| 메모리 대역폭 활동        | GPU 메모리가 얼마나 바쁜지 보여 줍니다. 배치 크기가 작을 때 디코드는 대개 메모리 대역폭 바운드입니다.                                                                                                       |
| 사용 중인 GPU 메모리      | 가중치와 KV 캐시를 위한 용량 여유분입니다.                                                                                                                                                                  |
| 전력, 온도, 스로틀링 지표 | 열이나 전력 제한이 GPU 성능에 미치는 영향입니다.                                                                                                                                                            |
| NVLink 대역폭             | [분산 추론](/infrastructure-and-operations/distributed-inference/)에서 NVLink로 연결된 GPU 간 인터커넥트 부하입니다.                                                                                        |
| XID 오류                  | 하드웨어, 드라이버, 애플리케이션 결함을 가리킬 수 있는 GPU 오류 보고입니다.                                                                                                                                 |

이 지표들을 직접 수집할 필요는 없습니다. NVIDIA GPU용
[DCGM exporter](https://github.com/NVIDIA/dcgm-exporter) 같은 도구가 지원되는
상태 및 프로파일링 지표를 Prometheus에 대신 노출해 줄 수 있습니다.

### 추론 엔진 {#inference-engine}

추론 엔진 지표는 일반적인 모니터링이 가장 자주 놓치는 레이어입니다. 모델 서버가
요청을 어떻게 스케줄링하고 메모리를 어떻게 관리하는지 보여 줍니다.

| **지표**                  | **알 수 있는 것**                                                                                                                           |
|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| 실행 중인 요청            | 현재 배치에 포함된 요청 수입니다.                                                                                                           |
| 대기 중인(큐에 쌓인) 요청 | 엔진이 아직 스케줄링하지 못한 요청입니다. 큐가 계속 늘어나면 포화 상태라는 신호일 수 있습니다.                                              |
| KV 캐시 사용량            | KV 캐시가 얼마나 찼는지 보여 줍니다. 100%에 가까워지면 새 요청이 대기하거나, 캐시된 블록이 축출되거나, 실행 중인 요청이 선점될 수 있습니다. |
| 프리픽스 캐시 적중률      | 프롬프트 연산을 얼마나 재사용하는지 보여 줍니다. 프롬프트에 재사용 가능한 프리픽스가 있는데 적중률이 낮다면 GPU 시간이 낭비됩니다.          |
| 선점                      | KV 캐시 같은 자원이 부족할 때 일시 중지되거나 다시 스케줄링되는 요청입니다. 선점이 지속적으로 발생한다면 경고 신호입니다.                   |
| 프롬프트 토큰과 생성 토큰 | 실제로 처리한 작업량을 토큰 단위로 보여 줍니다. 용량 계획과 비용 산정에는 RPS보다 낫습니다.                                                 |

실행 중인 요청과 대기 중인 요청을 합친 수(동시성)는
[LLM 워크로드 오토스케일링](/infrastructure-and-operations/fast-scaling/#scaling-metrics)에
가장 좋은 신호 중 하나이기도 합니다. KV 캐시 사용량은
[KV 캐시 인식 라우팅](/inference-optimization/inference-routing/)의 핵심
입력값입니다.

대부분의 오픈소스 추론 엔진은 `/metrics` 엔드포인트로 Prometheus 지표를
노출하므로, 별도의 계측 없이도 시작할 수 있습니다.

### 요청과 지연 시간 {#requests-and-latency}

이 지표들은 사용자가 실제로 체감하는 것을 설명합니다. 각 지표의 자세한 정의는
핸드북의 [LLM 추론 지표](/llm-inference-basics/llm-inference-metrics/) 페이지를
참고하세요.

| **지표**                                        | **알 수 있는 것**                                                                                         |
|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 첫 토큰까지의 시간(TTFT)                        | 응답이 시작되기 전까지 사용자가 기다리는 시간입니다. 큐 대기와 프리필이 좌우합니다.                       |
| 토큰 간 지연 시간(ITL) / 출력 토큰당 시간(TPOT) | 토큰이 얼마나 매끄럽게 스트리밍되는지 보여 줍니다. 디코드가 좌우합니다.                                   |
| 종단 간 지연 시간                               | 전체 응답에 걸린 총시간입니다.                                                                            |
| 큐 대기 시간                                    | 엔진이 요청 처리를 시작하기 전까지 요청이 기다리는 시간입니다.                                            |
| 초당 토큰 수                                    | 레플리카별 처리량과 전체 플릿의 처리량입니다.                                                             |
| 오류율                                          | 실패하거나, 시간 초과되거나, 거부된 요청입니다.                                                           |
| 굿풋(goodput)                                   | 지연 시간 SLO를 충족하는 초당 요청 수입니다. 정해진 서비스 목표에서 쓸모 있는 용량을 나타내는 척도입니다. |

평균과 함께 지연 시간 백분위수(P50, P90, P99)도 추적하세요. 중앙값이 양호해
보여도 그 뒤에 10배 더 오래 기다리는 사용자들의 꼬리가 숨어 있을 수 있습니다.

### 애플리케이션과 품질 {#application-and-quality}

최상위 레이어는 응답이 올바른지, 기능마다 비용이 얼마나 드는지처럼 인프라
지표로는 답할 수 없는 질문에 답합니다. 이 수준에서 성능을 평가할 때는 다음
신호를 사용하세요.

- 파이프라인이나 에이전트 루프에 있는 각 LLM 호출, 검색 단계, 도구 호출의
  **트레이스와 스팬**.
- 모델, 테넌트, 사용자, 기능별 **토큰 사용량과 비용**. 셀프 호스팅 모델이라면
  GPU 시간(GPU-hours)으로 100만 토큰당 연산 비용을 추정하고 API 가격과
  비교하세요.
  [추론 비용](/infrastructure-and-operations/build-and-maintenance-cost/)을
  참고하세요.
- 평가, 사용자 피드백, LLM-as-a-judge 점수로 측정하는 **출력 품질**. 모델이나
  프롬프트를 바꾼 뒤 생기는 품질 저하를 잡아냅니다.
- 잘못된 [구조화된 출력](/model-interaction/structured-outputs/), 실패한
  [함수 호출](/model-interaction/function-calling/), 가드레일 발동 같은
  **안전성 및 형식 검사**.

## 흔한 LLM 추론 문제를 진단하는 방법 {#how-to-diagnose-common-llm-inference-problems}

LLM 추론 문제 중 상당수는 몇 가지 지표만으로 원인을 좁힐 수 있습니다. 아래 표는
흔한 증상별로 확인할 지표와 일반적인 해결 방법을 정리한 것입니다.

| **증상**                                            | **확인할 지표**                              | **가능성 높은 원인**                                              | **조치 방법**                                                                                                                                       |
|-----------------------------------------------------|----------------------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| 부하가 걸리면 TTFT가 상승함                         | 대기 중인 요청, 큐 대기 시간, KV 캐시 사용량 | 레플리카가 포화되어 요청이 큐에 쌓임                              | 스케일 아웃합니다. KV 캐시 압박이 크다면 레플리카당 동시성을 제한하거나 [KV 캐시를 오프로드](/inference-optimization/kv-cache-offloading/)합니다    |
| 일부 레플리카에서만 TTFT가 높음                     | 레플리카별 큐 길이와 캐시 적중률             | 부하 불균형 또는 캐시를 고려하지 않는 라우팅                      | [부하와 캐시를 인식하는 라우팅](/inference-optimization/inference-routing/)을 사용합니다                                                            |
| TTFT는 괜찮은데 ITL이 급증함                        | ITL 백분위수, 배치 크기, 프롬프트 길이       | 긴 프리필이 디코드를 방해하거나 배치가 너무 큼                    | 청크 단위 프리필이나 배치 제한을 조정하거나 [프리필-디코드 분리](/inference-optimization/prefill-decode-disaggregation/)를 사용합니다               |
| 선점이 계속 증가함                                  | 선점, KV 캐시 사용량                         | KV 캐시 공간이 부족함                                             | 최대 동시 시퀀스 수를 낮추거나, KV 캐시에 메모리를 더 할당하거나, 레플리카를 추가합니다                                                             |
| 프리픽스가 반복되는데도 프리픽스 캐시 적중률이 낮음 | 프리픽스 캐시 적중 수 대비 쿼리 수           | 프롬프트가 동적 콘텐츠로 시작하거나 요청이 여러 레플리카로 흩어짐 | [프롬프트 구조를 바꾸고](/inference-optimization/prefix-caching/#how-to-structure-prompts-for-maximum-cache-hits) 프리픽스 인식 라우팅을 사용합니다 |
| GPU 사용률은 높은데 처리량은 낮음                   | SM 활동, 텐서 활동, 배치 크기, 초당 토큰 수  | 작은 배치 또는 연산 외의 병목                                     | 배치 크기나 동시성을 늘리고, 토큰화 같은 CPU 측 병목이 있는지 확인합니다                                                                            |
| 트래픽이 몰릴 때 지연 시간이 급증함                 | 레플리카 수, 스케일링 이벤트, 모델 로드 시간 | 새 레플리카의 콜드 스타트                                         | 레플리카를 미리 워밍업하고 [이미지 및 가중치 로딩](/infrastructure-and-operations/fast-scaling/) 속도를 높입니다                                    |
| Pod가 오류와 함께 재시작됨                          | Pod 이벤트, 로그, GPU 메모리, XID 오류       | OOM 오류 또는 GPU, 드라이버, 애플리케이션 결함                    | 메모리 설정을 확인하고, 노드를 드레인하기 전에 XID 코드를 조사합니다                                                                                |

## LLM 관측 가능성 도구 {#llm-observability-tools}

모든 레이어를 다루는 단일 오픈소스 LLM 관측 가능성 도구는 없습니다. 대부분의
팀은 서빙 측을 위한 인프라 스택과 애플리케이션 측을 위한 LLM 트레이싱 도구를
함께 사용합니다.

### 인프라와 서빙 {#infrastructure-and-serving}

- [Prometheus](https://prometheus.io/): 추론 엔진, DCGM, Kubernetes에서 지표를
  수집하고 저장합니다. 서빙 지표의 사실상 표준입니다.
- [Grafana](https://grafana.com/): Prometheus, 로그, 트레이스를 바탕으로
  대시보드와 알림을 제공합니다.
- [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter): GPU 상태 및
  프로파일링 지표를 Prometheus에 노출합니다.
- [OpenTelemetry](https://opentelemetry.io/): 지표, 로그, 트레이스 수집을 위한
  벤더 중립적인 표준입니다. OpenTelemetry의
  [GenAI 시맨틱 규약](https://github.com/open-telemetry/semantic-conventions-genai)은
  LLM 텔레메트리의 표준 이름을 정의합니다. 아직 개발 단계이므로 변경될 수
  있습니다.
- 컨테이너 전반을 한곳에서 검색할 수 있게 해 주는
  [Loki](https://github.com/grafana/loki)나
  [Elasticsearch](https://github.com/elastic/elasticsearch) 같은 로그 수집 도구.

### 애플리케이션 트레이싱과 평가 {#application-tracing-and-evaluation}

- [Langfuse](https://github.com/langfuse/langfuse): LLM 트레이싱, 평가, 프롬프트
  관리를 제공하는 오픈소스 도구입니다.
- [Arize Phoenix](https://github.com/Arize-ai/phoenix): OpenTelemetry 기반으로
  트레이싱과 평가를 제공하며, 소스 코드가 공개되어 있습니다.
- [OpenLLMetry](https://github.com/traceloop/openllmetry): LLM 공급자, 벡터
  데이터베이스, 프레임워크를 위한 오픈소스 OpenTelemetry 계측 도구입니다.
- [MLflow Tracing](https://mlflow.org/docs/latest/genai/tracing/)과
  [W&B Weave](https://github.com/wandb/weave): 널리 쓰이는 ML 실험 플랫폼에
  내장된 트레이싱 및 평가 기능입니다.

애플리케이션 트레이싱 도구는 각 요청을 외부에서 바라봅니다. 일반적으로 이
도구들만으로는 KV 캐시 압박, 선점, GPU 상태를 볼 수 없습니다. 셀프 호스팅
추론이라면 엔진 지표와 GPU 지표도 함께 필요합니다.

## 자주 묻는 질문 {#faqs}

### LLM 추론에서는 어떤 지표를 모니터링해야 하나요? {#what-metrics-should-i-monitor-for-llm-inference}

사용자 경험을 위해 TTFT, ITL, 종단 간 지연 시간, 오류율, 초당 토큰 수부터
시작하세요. 그다음 추론 엔진의 실행 중인 요청과 대기 중인 요청, KV 캐시 사용량,
프리픽스 캐시 적중률, 선점을 추가합니다. 마지막으로 하드웨어의 GPU 메모리, SM
활동, XID 오류를 추적합니다.

### LLM 애플리케이션에 적합한 TTFT는 얼마인가요? {#what-is-a-good-ttft-for-an-llm-application}

사용 사례에 따라 다릅니다. 대화형 채팅은 보통 TTFT를 약 500밀리초 미만으로 잡고,
코드 자동 완성은 100밀리초 미만이 필요할 수 있습니다. 배치 작업은 몇 초 이상도
감수할 수 있습니다. 목표값을 SLO로 정하고 P90이나 P99에서 측정하세요.

### 관측 가능성은 LLM 추론 비용을 줄이는 데 어떻게 도움이 되나요? {#how-does-observability-help-reduce-llm-inference-costs}

관측 가능성은 유휴 레플리카, 작은 배치, 낮은 캐시 적중률, 과다 프로비저닝된
클러스터처럼 GPU 용량이 낭비되는 지점을 보여 줍니다. 모델이나 테넌트별로 토큰과
GPU 시간을 추적하면 어떤 워크로드가 지출을 늘리는지도 알 수 있습니다. 두 정보
모두 스케일링과 최적화 결정의 근거가 됩니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter)
- [OpenTelemetry GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai)
</LinkList>
