---
sidebar_position: 1
description: LLM 성능 벤치마크는 특정 조건에서 LLM이 어떤 성능을 내는지 측정하는 표준화된 테스트입니다. 처리량, 지연 시간, 비용 효율, 자원 사용률 같은 실용적인 지표에 초점을 맞춥니다.
keywords:
    - LLM benchmarks, LLM performance benchmarks, LLM performance metrics, inference benchmarks
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# LLM 성능 벤치마크

LLM 리더보드를 본 적이 있을 것입니다. 깔끔하게 정리된 순위표로, 여러 벤치마크를
기준으로 상위 LLM을 보여 줍니다. 리더보드는 모델의 역량을 파악하는 데 유용한
참고 자료가 될 수 있습니다. 하지만 최적의 LLM을 고르는 유일한 근거로 삼으면
오해를 낳을 수도 있습니다.

LLM 벤치마킹은 복잡한 작업입니다. 공개 벤치마크에서 높은 점수를 받았다고 해서 그
모델이 여러분의 워크로드에서도 좋은 성능을 낸다는 보장은 없습니다. 여러분의
프로덕션 구성이 다른 사람의 LLM 순위에 쓰인 환경과 일치하는 경우는 드뭅니다.
종단 간 지연 시간을 300 ms 미만으로 유지하면서 출력 처리량 120 tokens/second를
내는 것처럼, 특정 시나리오에 맞춰 최적화하려 할 때는 더욱 그렇습니다.

사용 사례에 맞는 최적의 구성을 찾으려면 LLM 추론에 대한 맞춤형 성능 벤치마크를
직접 실행해야 합니다. 즉 테스트를 신중하게 설계해야 하고, 처리량, 지연 시간,
비용 사이에서 트레이드오프를 따져야 할 때가 많습니다.

## LLM 벤치마크란 {#what-are-llm-benchmarks}

사람들이 LLM 벤치마크라고 말할 때는 보통 서로 다른 두 가지를 가리킵니다.

- **품질 벤치마크**. 모델이 질문에 얼마나 잘 답하고, 추론하고, 지시를 따르는지
  측정합니다. MMLU, GSM8K, HumanEval, TruthfulQA 등이 있습니다. 온라인에서 볼 수
  있는, 최고의 LLM 순위를 매기는 LLM 리더보드는 대개 품질 벤치마크를 기반으로
  합니다.
- **성능 벤치마크**. 실제 환경에서 모델이 얼마나 빠르고 효율적으로 실행되는지
  측정합니다. 처리량(초당 토큰 수), 지연 시간(TTFT, 중앙값,
  P99), 비용 효율, GPU 사용률 같은 LLM 성능 지표에 초점을 맞춥니다.

두 유형의 벤치마크 모두 중요하지만 목적은 매우 다릅니다. 품질 리더보드에서 높은
순위에 오른 모델이라도 지연 시간이 너무 길거나 비용을 감당할 수 없다면
프로덕션에서 좋지 않은 결과를 낼 수 있습니다. 반대로 벤치마크 정확도는 낮더라도
추론 성능이 뛰어난 모델이라면, 속도가 더 중요한 애플리케이션에는 올바른 선택일
수 있습니다.

이 핸드북에서는 LLM 추론의 성능 벤치마크에 더 초점을 맞춥니다.

## 성능 벤치마크를 실행해야 할 때 {#when-you-should-run-performance-benchmarks}

맞춤형 벤치마킹이 의미 있는 상황은 다음과 같습니다.

- **여러 모델 비교**. 최고의 LLM 두 개 중 하나를 골라야 할 때, 벤치마크는
  여러분의 워크로드에서 두 모델의 처리량, 지연 시간, 비용이 어떻게 다른지 보여
  줍니다.
- **추론 프레임워크 평가**. vLLM, SGLang, MAX, TensorRT LLM, Hugging Face TGI
  같은 프레임워크는 각기 다른 추론 최적화를 제공합니다. 벤치마크로 여러분의
  구성에서 어느 프레임워크가 가장 좋은 트레이드오프를 내는지 확인할 수 있습니다.
- **인프라 변경 테스트**. A10G에서 H100 GPU로 옮기거나 온프레미스에서 클라우드로
  전환하면 성능이 달라집니다. 벤치마크가 그 영향을 확인해 줍니다.
- **최적화 효과 측정**.
  [추측 디코딩](/inference-optimization/speculative-decoding/),
  [프리픽스 캐싱](/inference-optimization/prefix-caching/),
  [분리형 서빙](/inference-optimization/prefill-decode-disaggregation/),
  [KV 캐시 오프로딩](/inference-optimization/kv-cache-offloading/) 같은 추론
  기법은 항상 반복 가능한 성능 테스트로 검증해야 합니다.
- **프로덕션 트래픽에 맞춘 확장**. 서비스를 시작하기 전에 현실적인 요청률과
  동시성 수준에서 벤치마크를 돌려 보면 시스템이 얼마나 버티는지 알 수 있습니다.

요컨대 어떤 추론 구성이 요구 사항을 충족할지에 대해 근거 있는 답이 필요할 때마다
LLM 성능 벤치마크를 실행해야 합니다.

## 벤치마크가 도움이 되지 않을 때 {#when-benchmarks-dont-help}

벤치마크가 유용한 통찰을 주지 못하는 상황은 다음과 같습니다.

- **워크로드와의 불일치**. 일부 결과는 범위가 좁은 테스트 세트에서 나옵니다.
  애플리케이션의 프롬프트가 더 길거나, 동시성이 더 높거나, 지연 시간 예산이
  엄격하다면 그 수치가 그대로 적용되지 않을 수 있습니다.
- **관련 없는 데이터셋**. 어떤 벤치마크는 코딩을, 어떤 벤치마크는 수학을, 또
  어떤 벤치마크는 추론 능력을 테스트합니다. 고객 서비스용 챗봇을 만든다면
  GSM8K나 HumanEval의 높은 점수는 전혀 중요하지 않을 수 있습니다.
- **다른 인프라 구성**. 여러분의 프로덕션 하드웨어, 프레임워크, 캐싱 전략은
  공개된 LLM 순위에 쓰인 환경과 정확히 일치하는 일은 없습니다.
- **단일 지표에만 집중**. tokens/second나 지연 시간만 보면 트레이드오프를
  놓칩니다.
- **비용과 확장 제약 간과**. 모델이 서류상으로는 빨라 보여도, 대규모로 실행하면
  비용이 너무 크거나 운영이 지나치게 복잡할 수 있습니다.

벤치마크는 최종 결정권자가 아니라 참고 자료로 활용해야 한다는 점을 기억하세요.

## 적합한 벤치마크 도구 선택 {#choosing-the-right-benchmark-tools}

LLM 성능을 테스트하고 측정하는 방법은 많지만, 모든 도구가 같은 목적을 위한 것은
아닙니다.

### 범용 부하 테스트 도구 {#general-load-testing-tools}

[Locust](https://locust.io/)와 [K6](https://k6.io/)는 실제 트래픽을
시뮬레이션하는 데 널리 쓰여 온 오래된 도구입니다. 이 도구들은 부하 테스트에
초점을 맞춥니다. 대량의 동시 요청을 생성해 LLM 배포가 어떤 성능을 내고 어떻게
확장되는지 확인합니다. 서버 용량, 오토스케일링 정책, 네트워크 지연 시간, 자원
사용률의 병목을 찾는 데 도움이 됩니다.

### LLM 전용 벤치마킹 도구 {#specialized-llm-benchmarking-tools}

[NVIDIA GenAI-Perf](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/genai-perf/README.html)와
[LLMPerf](https://github.com/ray-project/llmperf) 같은 도구는 LLM 성능
벤치마킹을 위해 만들어졌습니다. 이 도구들은 처리량과 지연 시간 같은 추론 수준의
지표에 초점을 맞춥니다. 모델 성능에 대한 상세한 통찰을 제공하지만, 도구마다
지표를 정의하거나 계산하는 방식이 다를 수 있습니다.

### 프레임워크 내장 벤치마킹 도구 {#built-in-framework-benchmarking-tools}

[vLLM](https://docs.vllm.ai/en/latest/benchmarking/),
[SGLang](https://docs.sglang.ai/developer_guide/benchmark_and_profiling.html#),
[MAX](https://max.modular.com/serve/benchmark/)처럼 많은 추론 프레임워크가 자체
벤치마킹 스크립트, 명령어, 사용 가이드를 함께 제공합니다. 예를 들어 각
프레임워크에서 다음과 같은 명령어로 벤치마크를 시작할 수 있습니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max benchmark \
  --backend modular \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --endpoint /v1/chat/completions \
  --dataset-name sonnet \
  --num-prompts 500
```

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm bench serve \
  --backend vllm \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --endpoint /v1/completions \
  --dataset-name sharegpt \
  --dataset-path <data_path>/ShareGPT_Data.json \
  --num-prompts 500
```

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
python3 -m sglang.bench_serving \
  --backend sglang \
  --model-path meta-llama/Llama-3.1-8B-Instruct \
  --dataset-name random \
  --random-input-len 256 \
  --random-output-len 32 \
  --max-concurrency 16 \
  --num-prompts 500
```

</TabItem>
</Tabs>

:::note
위 명령어를 실행하기 전에 먼저 서버를 시작하세요.
:::

이런 도구는 빠른 실험이나, 해당 프레임워크의 최적화가 적용된 상태의 성능을
파악하는 데 유용합니다. 하지만 기본 매개변수와 설정에는 항상 주의를 기울여야
합니다. 결과가 좋아 보여도 여러분의 프로덕션 구성을 반영하지 않을 수 있습니다.

## 어떤 지표를 벤치마크해야 할까 {#what-metrics-should-you-benchmark}

LLM 추론의 성능 벤치마크를 실행할 때 숫자 하나만 보고하는 것으로는 충분하지
않습니다. 실제 성능은 여러 LLM 성능 지표에 따라 달라지며, 지표마다
트레이드오프가 다릅니다. 가장 중요한 지표는 다음과 같습니다.

- **처리량**: 모델이 초당 처리하거나 생성할 수 있는 토큰 수 또는 요청 수입니다.
  확장성과 순수한 효율을 나타냅니다.
- **지연 시간**: 모델이 요청에 얼마나 빨리 응답하는지를 나타냅니다. 주요 지연
  시간 지표로는 TTFT, ITL, 중앙값 지연 시간, 꼬리 지연 시간(P95, P99)이
  있습니다. 이 지표들이 사용자가 체감하는 LLM의 응답성을 결정합니다.

  자세한 내용은 [LLM 추론 지표](/llm-inference-basics/llm-inference-metrics/)를
  참고하세요.

- **비용**: 보통 1,000 토큰당 비용, 요청당 비용, 단위 시간당 비용으로
  측정합니다. 셀프 호스팅 추론에서는 대개 GPU가 가장 큰 비용 요인입니다. 많은
  최상위 LLM은 효율적으로 실행하려면 크고 비싼 GPU가 필요합니다.
- **자원 사용률**: GPU/CPU 사용률, 메모리 할당, 캐시 적중률입니다. 사용률이
  높으면 대개 하드웨어 효율이 좋다는 뜻입니다. 사용률은 1달러당 얻는 성능을 직접
  결정하므로 프로덕션에서 특히 중요합니다.

벤치마크 결과는 실행 방식에 크게 좌우됩니다. 통제해야 할 주요 매개변수는 다음과
같습니다.

- **서버 매개변수**: 병렬화(예: 데이터, 텐서, 전문가 병렬화), 캐싱 전략, 메모리
  할당
- **클라이언트 매개변수**: 요청률, 동시성 한도, 배치 크기, 타임아웃
- **프레임워크 차이**: vLLM, SGLang, MAX, TensorRT LLM, TGI 등의 최적화
- **워크로드 변화**: 모델 선택, 입력 시퀀스 길이, 출력 길이, 요청 분포

경우에 따라서는 이런 변수의 작은 차이만으로도 결과가 크게 바뀔 수 있습니다.

## 좋은 LLM 성능 벤치마크를 만드는 방법 {#how-to-create-good-llm-performance-benchmarks}

LLM 성능 벤치마크를 실행하는 일은 보기보다 어렵습니다. "200 tokens/second" 같은
수치는 단순해 보이지만, 맥락 없이는 거의 의미가 없습니다. 제대로 된 LLM
벤치마킹은 테스트 환경, 워크로드, 제약 조건을 함께 기록해 결과를 재현할 수 있고
실제와 관련성이 있도록 해야 합니다.

다음은 더미 데이터를 넣은 벤치마크 템플릿 예시입니다.

| 분류                | 매개변수          | 값                                 |
|---------------------|-------------------|------------------------------------|
| **모델**            | 이름              | `meta-llama/Llama-3.1-8B-Instruct` |
|                     | 프레임워크        | `vLLM 0.4.2`                       |
|                     | 양자화            | `FP16`                             |
| **하드웨어**        | GPU               | `4 x NVIDIA A10G`                  |
|                     | GPU 메모리        | `24 GB`                            |
| **서버 설정**       | 텐서 병렬화       | `1`                                |
|                     | 데이터 병렬화     | `4`                                |
|                     | 최대 배치 크기    | `16`                               |
| **클라이언트 설정** | 요청률            | `50 req/s`                         |
|                     | 동시성            | `16`                               |
|                     | 입력 길이         | `256 tokens`                       |
|                     | 출력 길이         | `512 tokens`                       |
| **결과**            | 처리량            | `98 tokens/sec`                    |
|                     | 중앙값 TTFT       | `210 ms`                           |
|                     | P99 지연 시간     | `1.4 s`                            |
|                     | GPU 사용률        | `89%`                              |
|                     | 100만 토큰당 비용 | `$0.52`                            |

이 템플릿을 사용할 때의 팁은 다음과 같습니다.

- 프레임워크 버전과 하드웨어 유형을 항상 기록하세요. 이 부분의 작은 차이만으로도
  결과가 크게 달라질 수 있습니다.
- 입력/출력 길이를 포함하세요. 입력/출력 길이는 처리량과 지연 시간에 큰 영향을
  줍니다.
- 스트리밍 성능을 측정할 때는 모델의 순수 출력 속도보다 체감 tokens/sec(사용자가
  실제로 보는 속도)가 더 나은 지표일 수 있습니다.
- 비용 지표는 요청당, 100만/1,000 토큰당처럼 일관된 기준을 유지하세요.
