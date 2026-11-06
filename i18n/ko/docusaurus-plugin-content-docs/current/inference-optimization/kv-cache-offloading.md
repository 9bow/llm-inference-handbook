---
sidebar_position: 8
description: KV 캐시 오프로딩이 GPU 메모리 사용량을 줄이고 지연 시간을 낮추며 연산 비용을 절감해 LLM 추론을 개선하는 방법을 알아봅니다.
keywords:
    - KV cache offloading, KV cache, KV caching
    - LMCache, Mooncake, KVBM, HiCache, KV connector
    - vLLM, SGLang, MAX
    - Distributed inference, distributed LLM inference
    - Inference optimization
    - LLM inference optimization, LLM inference optimization techniques
    - Speed up LLM inference
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import LinkList from '@site/src/components/LinkList';
import KVCacheCalculator from '@site/src/components/Calculator/KVCache';

# KV 캐시 오프로딩

KV 캐시 오프로딩(KV cache offloading)은 어텐션의 키/값 데이터를 GPU 메모리에서
CPU 메모리나 디스크 같은 저렴한 저장소로 옮기는 과정입니다. 이렇게 하면 GPU
자원을 확보하면서도 재계산 없이 추론을 이어갈 수 있습니다. 성능과 메모리 사용량
사이의 균형을 맞춰 LLM 워크로드를 효율적으로 확장하는 데 도움이 됩니다.

## LLM 추론에서 KV 캐시가 병목이 되는 이유는 무엇인가요? {#why-does-kv-cache-become-a-bottleneck-in-llm-inference}

LLM은 추론 속도를 높이기 위해 KV 캐시에 크게 의존합니다. KV 캐시는 입력 시퀀스의
모든 토큰에 대한 어텐션 키와 값을 저장하므로, 모델은 이후 단계에서 이 값들을
다시 계산하지 않고 재사용할 수 있습니다. 이 방식은 연산 자원을 크게 절약하고
추론을 빠르게 만들지만, 그 대가로 메모리 비용이 매우 큽니다.

컨텍스트 윈도우가 커질수록
**KV 캐시 크기는 시퀀스 길이에 비례해 선형적으로 증가합니다**. 특히 긴
컨텍스트를 다루는 상황에서는 사용 가능한 GPU 메모리가 금방 바닥날 수 있습니다.
GPU 메모리는 한정되어 있으므로, 긴 컨텍스트가 필요한 애플리케이션을 실행할 때 KV
캐시가 병목이 되곤 합니다.

사실 **모든 KV 캐시 데이터가 항상 GPU 메모리에 남아 있을 필요는 없습니다**. 실제
애플리케이션에서는 사용자가 LLM과 계속 상호작용하지 않는 경우가 많습니다. 예를
들어 사용자가 입력 도중 잠시 멈추거나, 자리를 떴다가 몇 시간 뒤에 돌아올 수도
있습니다. 이런 경우 그 사용자의 KV 캐시는 실제로 쓰이지 않는데도 GPU 메모리에
그대로 남아 있습니다. 마찬가지로 여러 사용자나 에이전트가 같은 대화, 문서,
세션에 서로 다른 시점에 접근하면, 같은 KV 캐시가 상호작용 사이사이에 GPU에서
놀고 있을 수 있습니다(같은 내용을 다시 계산하느라 GPU 자원을 낭비하고 싶지는
않을 것입니다).

그 결과 메모리가 비효율적으로 사용됩니다. 귀한 GPU 메모리가 새 요청을 처리하는
데 쓰이지 못하고 비활성 세션에 묶여 있기 때문입니다. 시간이 지나면 시스템이
지원할 수 있는 동시 사용자 수가 제한되고 전체 처리량도 줄어듭니다.

KV 캐시 오프로딩은 이런 문제를 해결하기 위해 비활성 상태이거나 접근 빈도가 낮은
캐시 데이터를 GPU 메모리에서 CPU RAM, 로컬 SSD, 원격 객체 스토리지처럼 더
저렴하고 용량이 큰 저장소로 옮깁니다. 사용자가 상호작용을 재개하거나 다른
사용자가 같은 내용에 접근하면, 필요할 때 캐시를 GPU 메모리로 다시 불러올 수
있습니다. 이렇게 하면 비용이 큰 재계산을 피하면서 활성 워크로드를 위한 GPU
자원을 확보할 수 있습니다.

## KV 캐시 크기를 계산하는 방법 {#how-to-calculate-the-kv-cache-size}

KV 캐시를 오프로드할 때는 KV 캐시가 실제로 메모리를 얼마나 차지하는지 이해해
두면 좋습니다.

트랜스포머 기반 LLM에서는 각 어텐션 레이어가 입력 시퀀스의 모든 토큰마다 벡터 두
개(키 하나와 값 하나)를 저장해야 합니다. 각 레이어에는 여러 개의 어텐션 헤드가
있으며, 보통 모든 헤드의 차원은 같습니다.

KV 캐시가 메모리를 얼마나 사용하는지 추정하려면 다음 계산기를 사용하세요.

<KVCacheCalculator />

:::info
LLM의 아키텍처 세부 정보는 보통 그 모델의 Hugging Face 저장소에 있는
`config.json` 파일에서 찾을 수 있습니다. 이 파일에는 모델 아키텍처(예:
트랜스포머 디코더), 레이어 수, 은닉 차원 크기, 어텐션 헤드 수, 어휘 크기, 그
밖의 아키텍처 하이퍼파라미터가 들어 있습니다. 모델의 차원을 이미 알고 있다면
`H × D`를 그 값으로 바꿔 공식을 단순화할 수 있습니다(위의 Simplified
Calculation).
:::

## LLM에서 KV 캐시를 언제 오프로드해야 하나요? {#when-should-you-offload-the-kv-cache-for-llms}

KV 캐시 오프로딩은 특히 다음과 같은 경우에 유용합니다.

- 컨텍스트 윈도우가 긴 LLM을 배포하고 있어서 KV 캐시가 GPU 메모리를 금방 초과할
  수 있는 경우
- 여러 사용자나 에이전트가 세션을 넘나들며 같은 내용이나 컨텍스트와 상호작용해야
  하는 경우. 예를 들어 LLM이 통합된 IDE에서 작업하는 개발자는 같은 코드 조각과
  반복해서 상호작용하곤 합니다.
- 배포 환경의 메모리가 제한되어 있거나 인프라 비용을 최적화해야 하는 경우
- GPU 자원이 제한된 상태에서 여러 분산 워커에 걸쳐 추론을 확장하는 경우
- 워크로드에 간헐적이거나 유휴 상태인 사용자 세션이 포함되어 있어서, KV 캐시를
  GPU 메모리에 계속 두면 낭비가 되는 경우

## KV 캐시 오프로딩의 이점 {#benefits-of-kv-cache-offloading}

KV 캐시를 오프로드하면 LLM 추론을 확장하고 최적화하는 데 다음과 같은 중요한
이점이 있습니다.

- **자원 사용률 향상.** 비활성 상태이거나 공유되는 KV 데이터를 GPU 메모리 밖으로
  옮기면 새 요청을 위한 공간을 확보할 수 있습니다. 따라서 같은 GPU로 메모리
  한계에 부딪히지 않고 더 많은 동시 사용자나 더 긴 입력 시퀀스를 처리할 수
  있습니다.
- **연산 비용 절감.** GPU 메모리는 비싸고 한정되어 있습니다. 오프로딩을 사용하면
  워크로드가 더 저렴한 저장소(예: CPU RAM이나 디스크)를 활용할 수 있으므로,
  캐시를 관리하려고 고성능 GPU를 과도하게 프로비저닝할 필요가 줄어듭니다.
- **지연 시간 감소**: 오프로딩을 사용하면 모델이 추론 중에 중복되는 KV 계산을
  건너뛸 수 있습니다. 특히 멀티턴 상호작용에서 겹치는 컨텍스트에 효과가 큽니다.
  그 결과 TTFT와 전체 지연 시간이 크게 줄어듭니다. NVIDIA에 따르면 KV 캐시
  오프로딩은 입력 시퀀스가 큰 경우 KV 캐시를 처음부터 다시 계산하는 방식에 비해
  [TTFT를 최대 14× 단축](https://developer.nvidia.com/blog/nvidia-gh200-superchip-accelerates-inference-by-2x-in-multiturn-interactions-with-llama-models/)할
  수 있습니다.

## KV 캐시 오프로딩의 트레이드오프 {#trade-offs-in-kv-cache-offloading}

KV 캐시 오프로딩은 메모리 효율과 처리량을 크게 개선할 수 있지만, 오프로딩 대상
저장소의 속도가 관건입니다. 저장 계층(예: CPU RAM이나 디스크)이 너무 느리면 KV
데이터를 GPU로 다시 전송하는 오버헤드가 이점을 상쇄할 수 있습니다. 지연 시간에
민감한 애플리케이션에서는 특히 그렇습니다.

데이터 전송 비용이 캐시를 처음부터 다시 계산하는 비용보다 낮은지 확인하세요. 긴
멀티턴 대화에서는 대체로 전송 쪽이 더 저렴합니다. 이전 컨텍스트를 재사용하는
것이 중요하고 재계산 비용이 크기 때문입니다.

시스템이 선택적 KV 오프로딩을 사용할 때는 품질 측면의 트레이드오프도 있습니다.
디코딩 중에 런타임은 어떤 키와 값을 GPU로 되돌릴지 결정해야 할 수 있습니다. 이때
중요한 컨텍스트 토큰을 놓치면 모델의 답변 품질이 떨어질 수 있습니다. 다중 문서
QA, 법률 검토, 코드베이스 추론처럼 프롬프트의 여러 세부 정보가 중요할 수 있는
컨텍스트 집약적 워크로드에서는 이런 위험이 큽니다.

[이 논문](https://arxiv.org/abs/2604.08426)은 이 문제를 잘 보여 줍니다. 일부 KV
오프로딩 방법은 일반적인 긴 컨텍스트 벤치마크에서는 좋은 성능을 내지만,
프롬프트에서 많은 사실을 찾아와야 하는 작업에서는 성능이 떨어집니다. 실용적인
교훈은 컨텍스트 길이가 긴 것과 컨텍스트 집약도가 높은 것은 서로 다르다는
점입니다. 프로덕션에서 선택적 KV 오프로딩을 켜기 전에, 워크로드와 비슷한
작업에서 전체 어텐션(full attention) 기준선과 비교해 보세요. TTFT, TPOT, 처리량,
GPU 메모리 사용량, 호스트-디바이스 간 전송량과 함께 답변 품질도 추적하세요.

## KV 캐시를 오프로드하는 방법 {#how-to-offload-the-kv-cache}

구현을 직접 만들 필요는 거의 없습니다. 이제 주요 추론 프레임워크는 모두 KV 캐시
오프로딩을 기본 기능으로 지원합니다. 또한 프레임워크만으로 해결할 수 없는 경우를
위해 전용 KV 캐시 시스템이라는 두 번째 계층도 있습니다.

### 추론 프레임워크에 내장된 오프로딩 {#built-in-offloading-in-your-inference-framework}

vLLM, SGLang, MAX는 모두 별도 인프라 없이 호스트 메모리와 디스크로의 오프로딩을
지원합니다. 작동 방식은 비슷합니다. 기존
[프리픽스 캐시](/inference-optimization/prefix-caching/)를 확장해서, 완성된
블록을 더 크고 느린 계층으로 내려보내고, 이후 요청이 해당 블록에 적중하면 다시
GPU로 올립니다. 전송은 DMA(Direct Memory Access) 복사 엔진에서 비동기로 실행될
수 있으므로, 모델 실행을 멈추지 않고 그와 겹쳐서 진행됩니다.

다음은 몇 가지 예시입니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --kv-connector-config '{"type": "rust_tiered"}'
```

MAX는 이 계층을
[KV 커넥터](https://max.modular.com/api/python/generated/max.pipelines.lib.KVConnectorConfig/)라고
부릅니다. `rust_tiered` 커넥터는 축출된 블록을 호스트 메모리와 로컬 디스크에
계층적으로 나누어 저장합니다. 원하면 다음과 같이 용량 한도를 직접 지정할 수
있습니다.

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --kv-connector-config '{
    "type": "rust_tiered",
    "host_offload_max_gb": 128,
    "disk_offload_dir": "/mnt/kv_cache",
    "disk_offload_max_gb": 512
  }'
```

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct \
  --kv-offloading-backend native \
  --kv-offloading-size 100 # the buffer size in GiB
```

최신 vLLM 버전은 이 두 개의 최상위 플래그로 오프로딩을 제공합니다. 내부
메커니즘은
[OffloadingConnector](https://docs.vllm.ai/en/stable/features/kv_offloading_usage/)이며,
다중 계층 구성이 필요하면 이를 직접 설정할 수 있습니다.

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct \
  --kv-transfer-config '{
    "kv_connector": "OffloadingConnector",
    "kv_role": "kv_both",
    "kv_connector_extra_config": {
      "cpu_bytes_to_use": 100000000000,
      "spec_name": "TieringOffloadingSpec",
      "secondary_tiers": [{"type": "fs", "root_dir": "/mnt/kv_cache"}]
    }
  }'
```

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path meta-llama/Llama-3.1-8B-Instruct \
  --enable-hierarchical-cache \
  --hicache-ratio 2 \                     # Host memory ratio (2x GPU memory)
  --hicache-size 100 \                    # Host memory size in GBs, will override the above ratio
  --hicache-write-policy write_through \  # Cache write policy from GPU to CPU
  --hicache-storage-backend               # Optional storage backend
```

SGLang의 오프로딩 계층은
[HiCache](https://docs.sglang.io/docs/advanced_features/hicache)라고 하며,
RadixAttention에 3계층 구조의 계층형 KV 캐싱 시스템을 더해 확장한 것입니다.

</TabItem>
</Tabs>

:::note
KV 캐시 오프로딩은 세 프레임워크 모두에서 활발히 개발되고 있으므로, 이 플래그
이름은 최신이 아닐 수 있습니다. 사용 중인 버전의 문서를 확인하세요.
:::

### 전용 KV 캐시 시스템 {#dedicated-kv-cache-systems}

프레임워크 자체의 오프로딩 구현 외에도, 확장 기능이 필요하면 전용 KV 캐시 계층을
추가할 수 있습니다.

- [LMCache](https://github.com/LMCache/LMCache)는 특정 벤더에 종속되지 않는 KV
  캐시 관리 계층입니다. TTFT를 줄이고 처리량을 높여 LLM 추론을 최적화하도록
  설계되었으며, 특히 긴 컨텍스트 워크로드에 효과적입니다. CPU RAM, 로컬 디스크,
  Redis, Valkey, Mooncake, InfiniStore 등 다양한 스토리지 백엔드로의 영속적이고
  계층적인 KV 캐시 오프로딩을 지원합니다.
- [Mooncake Store](https://kvcache-ai.github.io/Mooncake/design/store/mooncake-store.html)는
  LLM 추론 전용으로 설계된 분산 KV 캐시 스토리지 엔진입니다. 추론 엔진이 GPU,
  노드, 인스턴스 사이에서 KV 캐시를 저장, 조회, 전송할 수 있게 해 주며,
  프리필-디코드 분리를 지원하고 캐시 재사용률을 높입니다. vLLM, SGLang, LMCache
  같은 시스템과 폭넓게 통합되어 있습니다.
- [NVIDIA Dynamo KVBM](https://docs.nvidia.com/dynamo/knowledge-base/modular-components/kvbm/overview)(KV
  Block Manager)은 이기종 분산 환경에서 추론 작업에 쓰이는 KV 블록의 메모리
  할당, 관리, 원격 공유를 처리하는 통합 메모리 계층입니다.

---

어떤 도구를 선택하든 실제 워크로드로 벤치마크하고, TTFT와 처리량과 함께 계층별
캐시 적중률을 측정하세요. 올바르게 구성된 호스트 계층은 적중률이 높고 TTFT가
개선되어야 합니다. 반면 적중이 거의 없는 디스크 계층은 대부분 아무 이득 없이
전송 비용만 치르고 있는 것입니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [NVIDIA GH200 Superchip Accelerates Inference by 2x in Multiturn Interactions with Llama Models](https://developer.nvidia.com/blog/nvidia-gh200-superchip-accelerates-inference-by-2x-in-multiturn-interactions-with-llama-models/)
- [5x Faster Time to First Token with NVIDIA TensorRT LLM KV Cache Early Reuse](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)
- [KV Cache Offloading for Context-Intensive Tasks](https://arxiv.org/abs/2604.08426)
</LinkList>
