---
sidebar_position: 9
description: 데이터 병렬화, 텐서 병렬화, 파이프라인 병렬화, 전문가 병렬화, 하이브리드 병렬화의 차이를 알아봅니다.
keywords:
    - LLM inference optimization, LLM inference optimization techniques
    - Data parallelism, tensor parallelism, pipeline parallelism, expert parallelism and hybrid parallelism
    - Speed up LLM inference
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import LinkList from '@site/src/components/LinkList';

# 데이터, 텐서, 파이프라인, 전문가, 하이브리드 병렬화(parallelism)

병렬화 전략은 추론 작업을 여러 장치에 나누어 처리량을 높이고, 더 큰 모델을
올리고, 하드웨어를 더 효율적으로 사용합니다. 모델이 더 커지고 복잡해지면서
새로운 병렬화 전략도 계속 등장하고 있습니다.

## 데이터 병렬화 {#data-parallelism}

데이터 병렬화(data parallelism, DP)는 개별 요청이나 마이크로배치(microbatch)를
같은 모델의 여러 레플리카에 나누어 전체 처리량을 높입니다. 각 레플리카는 모델
가중치의 사본을 각자 가지고 서로 다른 요청을 독립적으로 처리합니다. 레플리카
하나는 GPU 한 개에서 실행될 수도 있고, 내부적으로
[텐서 병렬화](#tensor-parallelism)나
[파이프라인 병렬화](#pipeline-parallelism)를 사용해 여러 GPU에 걸쳐 있을 수도
있습니다.

AI 팀은 흔히 라우터를 사용해 독립적인 요청을 여러 레플리카에 분배합니다. 배치
워크로드라면 큰 배치를 더 작은 마이크로배치로 나누고, 각 레플리카가 서로 다른
마이크로배치를 동시에 처리하게 할 수도 있습니다. 이렇게 하면 더 많은 요청을
동시에 처리할 수 있습니다.

<figure>
  <Diagram name="dp" alt="" />
  <figcaption><b>그림 1.</b> 데이터 병렬화: 각 GPU에 전체 모델 레플리카를 하나씩 둡니다.</figcaption>
</figure>

따라서 레플리카를 추가하면 전체 처리량과 동시성이 늘어납니다. 하지만 개별 요청의
연산 지연 시간이 직접 줄어들지는 않습니다. 그 요청은 여전히 레플리카 하나에서
처리되기 때문입니다.

데이터 병렬화는 장애 경계 역할도 합니다. 헬스 체크에서 레플리카 하나가 사용할 수
없는 상태로 표시되면, 라우터는 그 레플리카에 새 요청을 보내지 않고 나머지
레플리카를 사용할 수 있습니다. 이렇게 하면 가용성이 높아지지만, 정상 레플리카에
늘어난 트래픽을 받아낼 여유 용량이 있어야 합니다. 또한 장애가 난 레플리카에서
처리 중이던 요청과 로컬 KV 캐시는 사라질 수 있습니다.

여기서 핵심 고려 사항은 요청을 어떻게 라우팅하느냐입니다. LLM 추론에서는
요청마다 프롬프트 길이, 출력 길이, KV 캐시 사용량이 크게 다를 수 있습니다. GPU
용량은 비싸기도 해서, 부하를 잘못 분산하면 일부 레플리카는 과부하 상태가 되고
다른 레플리카는 놀게 됩니다. 그래서 프로덕션 라우터는 레플리카를 고를 때 여러
신호를 함께 고려하는 경우가 많습니다. 자세한 내용은
[추론 라우팅](/inference-optimization/inference-routing/)을 참고하세요.

데이터 병렬화의 주된 비용은 메모리 중복입니다. 모든 레플리카가 모델 가중치
사본을 가져야 하고 KV 캐시도 따로 유지합니다. 모델 사본 하나가 워커 하나에
들어가지 않는다면 데이터 병렬화만으로는 문제가 해결되지 않습니다. 이때는 텐서
병렬화나 파이프라인 병렬화와 함께 사용하세요.

## 텐서 병렬화 {#tensor-parallelism}

텐서 병렬화(tensor parallelism, TP)는 모델의 가중치를 여러 GPU에 나누어, GPU 한
개의 메모리에 들어가지 않는 큰 모델을 배포할 수 있게 합니다. 각 모델 레이어 안의
텐서를 분할하므로, 모든 장치는 레이어의 사본이 아니라 레이어의 샤드(shard)를
가집니다.

이 방식이 필요한 이유는 최신 LLM 중 다수가 양자화를 거친 뒤에도 GPU 한 개에
들어가지 않기 때문입니다. KV 캐시와 런타임 오버헤드까지 고려하면 더욱
그렇습니다. 예를 들어 FP8 가중치는 매개변수 10억 개당 약 1 GB가 필요하므로,
Llama 3.1 405B 같은 모델은 가중치에만 약 405 GB가 필요합니다. 이것만으로도 현재
어떤 단일 GPU의 메모리보다 크기 때문에, 장치 하나로는 모델을 올리지 못하거나
효율적으로 서빙하지 못하는 경우가 많습니다.

행렬 곱 같은 연산을 할 때 각 GPU는 가중치 텐서의 샤드를 가지고 결과의 일부를
계산합니다. 그런 다음 GPU들은 텐서를 분할한 방식에 따라 올리듀스나
올개더(all-gather) 같은 집합 통신 연산으로 중간 결과를 교환하거나 합칩니다.

<figure>
  <Diagram name="tp-inference" alt="" />
  <figcaption><b>그림 2.</b> 텐서 병렬화: 모든 레이어를 여러 GPU에 샤딩합니다.</figcaption>
</figure>

텐서 병렬화는 가중치, 캐시, 연산을 분산하여 GPU 한 개의 용량을 넘는 모델도
실제로 서빙할 수 있게 합니다. 병렬 연산으로 얻는 이득이 추가 통신 비용보다
크다면 개별 요청의 지연 시간도 줄일 수 있습니다.

이 통신이 핵심 트레이드오프입니다. 텐서 병렬화에서 GPU들은 요청이 트랜스포머
레이어를 통과하는 동안 중간 결과를 반복해서 교환해야 합니다. 그래서 인터커넥트
대역폭과 지연 시간이 성능에 큰 영향을 줍니다.

텐서 병렬화는 NVLink처럼 단일 노드 안에서 고대역폭 인터커넥트로 연결된 GPU들
사이에서 가장 잘 동작합니다. 서로 다른 노드에 있는 GPU에 텐서를 나누면 네트워크
통신이 주요 병목이 될 수 있습니다. 샤딩에 사용할 GPU 수에도 제약이 있습니다.
일반적으로 어텐션 헤드 수가 GPU 수로 나누어떨어져야 하므로, 임의의 GPU 수로
확장할 수는 없습니다.

텐서 병렬화를 위해 모델을 샤딩하기 전에, 가중치, KV 캐시, 기타 추론 메모리
요구량이 GPU 한 개에 여유 있게 들어가는지 먼저 확인하세요. 특히
[양자화](/model-preparation/llm-quantization/)를 적용한 뒤라면 더 그렇습니다.
단일 GPU 서빙은 장치 간 동기화가 전혀 필요 없고, 텐서 병렬화로 GPU를 늘려도
속도가 선형적으로 빨라지는 경우는 드뭅니다. 메모리 사용량이나 지연 시간 목표가
통신 오버헤드를 감수할 만할 때 텐서 병렬화를 사용하세요.

## 파이프라인 병렬화 {#pipeline-parallelism}

파이프라인 병렬화(pipeline parallelism, PP)는 모델의 레이어를
스테이지(stage)라는 순차적인 청크로 나누고, 각 스테이지를 서로 다른 장치에
할당합니다. 데이터는 조립 라인처럼 이 스테이지들을 차례로 통과하며, 한 장치의
출력이 다음 장치의 입력이 됩니다. 예를 들어 4-way 파이프라인에서는 각 장치가
모델 레이어의 4분의 1을 처리합니다.

<figure>
  <Diagram name="pp-diagram" alt="" />
  <figcaption><b>그림 3.</b> 파이프라인 병렬화: 연속된 레이어를 각 GPU에 할당합니다.</figcaption>
</figure>

텐서 병렬화와 달리 파이프라인 병렬화에서는 장치들이 레이어마다 부분 결과를 합칠
필요가 없습니다. 한 스테이지는 자신의 레이어를 모두 마친 뒤에만 다음 스테이지로
활성화 값을 보냅니다. 통신 빈도가 이렇게 낮기 때문에 파이프라인 병렬화는 노드 간
인터커넥트를 비롯한 저대역폭 연결에 더 잘 맞을 수 있습니다.

트레이드오프는 파이프라인 버블(pipeline bubble)입니다. 한 스테이지는 이전
스테이지에서 작업이 넘어오기를 기다리는 동안 놀 수 있습니다. 느리거나 메모리를
많이 쓰는 스테이지 하나가 뒤따르는 모든 스테이지를 지연시킬 수도 있습니다.
따라서 스테이지 경계는 레이어 수만이 아니라 실행 시간과 메모리 요구량의 균형을
맞추어 정해야 합니다.

<figure>
  <Diagram name="pp-batching" alt="" />
  <figcaption><b>그림 4.</b> 반복이 진행되면서 마이크로배치가 4단계 파이프라인을 채웁니다.</figcaption>
</figure>

유휴 구간을 줄이기 위해 서버는 여러 요청이나 마이크로배치를 동시에 처리 중인
상태로 유지할 수 있습니다. 뒤쪽 스테이지가 한 마이크로배치를 처리하는 동안 앞쪽
스테이지는 다음 마이크로배치 작업을 시작할 수 있습니다. 이 스케줄링 방식은
처리량을 높이지만, 파이프라인의 시작과 끝에서 생기는 유휴 시간을 완전히 없애지는
못합니다. 자세한 내용은 [GPipe 논문](https://arxiv.org/pdf/1811.06965)을
참고하세요.

파이프라인 병렬화가 개별 요청의 지연 시간을 반드시 줄이는 것은 아닙니다. 모든
요청은 여전히 모든 스테이지를 순서대로 통과해야 하고, 활성화 값 전송 비용도
치릅니다. 파이프라인 병렬화는 파이프라인을 가득 채울 만큼 동시 작업이 충분할 때,
또는 파이프라인 버블보다 낮은 통신 빈도가 더 중요할 때 가장 잘 동작합니다.

## 전문가 병렬화 {#expert-parallelism}

전문가 병렬화(expert parallelism)는 전문가 혼합(MoE) 모델에 쓰이는 특수한 병렬화
전략입니다. 이런 모델에서는 토큰마다 전문가 중 일부만 활성화됩니다. 전문가
병렬화는 모든 전문가를 모든 장치(예: GPU)에 복제하는 대신, 전문가 자체를 여러
장치에 나누어 배치합니다.

<figure>
  <Diagram name="ep-inference" alt="" />
  <figcaption><b>그림 5.</b> 전문가 병렬화: 전문가를 여러 GPU에 분산합니다.</figcaption>
</figure>

각 GPU는 전체 전문가가 아니라 일부 전문가의 전체 가중치만 가집니다. 따라서 각
GPU는 자신에게 저장된 전문가에 할당된 토큰만 처리합니다. 반면 MoE 모델에 텐서
병렬화를 적용하면, 모든 전문가의 가중치 행렬을 조각내어 그 조각들을 모든 장치에
분산할 뿐입니다.

전문가 병렬화를 사용하면 GPU를 더 효율적으로 활용할 수 있습니다. 모든 장치에
전체 모델을 복제하는 방식에 비해 메모리와 연산 오버헤드가 줄어듭니다.

## 하이브리드 병렬화 {#hybrid-parallelism}

모델에 따라서는 병렬화 전략 하나만으로는 부족한 경우가 많습니다. 하이브리드
병렬화(hybrid parallelism)는 두 가지 이상의 병렬화 기법을 결합해 더 나은 확장성,
효율, 하드웨어 사용률을 얻습니다.

그림 6은 데이터 병렬화와 텐서 병렬화를 결합한 전형적인 하이브리드 구성의 예를
보여줍니다.

<figure>
  <Diagram name="dptp" alt="" />
  <figcaption><b>그림 6.</b> TP=4, DP=2인 하이브리드 병렬화.</figcaption>
</figure>

GPU가 8개라면, 처음 GPU 4개에 텐서 병렬화를 적용하고(TP=4), 데이터 병렬화로 같은
구성을 나머지 GPU에 복제할 수 있습니다(DP=2).

이 구성은 가능한 여러 조합 중 하나일 뿐이며, 조합마다 장단점이 있습니다. 그림
6에서 텐서 병렬화는 특히 추론 중에 GPU 간 통신 오버헤드를 일으킵니다. 따라서 TP
차수를 높인다고 항상 성능이 좋아지지는 않습니다.

다른 구성으로는 텐서 병렬화를 줄이고 데이터 병렬화를 늘리는 방법이 있습니다.
예를 들어 그림 7처럼 TP=2, DP=4로 설정할 수 있습니다.

<figure>
  <Diagram name="dp4tp2" alt="" />
  <figcaption><b>그림 7.</b> TP=2, DP=4인 하이브리드 병렬화.</figcaption>
</figure>

이렇게 하면 GPU 간 통신이 줄어 추론 지연 시간을 낮추는 데 도움이 될 수 있습니다.
하지만 함정이 있습니다. 모델 가중치는 GPU 메모리의 상당 부분을 차지하며, 큰
모델일수록 더 그렇습니다. 텐서 병렬화를 낮추면 모델을 나누어 가지는 GPU 수가
줄어들어 KV 캐시를 위한 공간이 적어집니다. 그러면 프리픽스 캐싱 같은 추론
최적화의 효과가 떨어질 수 있습니다.

이런 트레이드오프는 텐서 병렬화와 데이터 병렬화에만 있는 것이 아닙니다.
하이브리드 병렬화 계획을 세울 때는 모델 크기, 하드웨어 구성, 추론 요구 사항에
맞추어 여러 구성을 반드시 벤치마크해야 합니다. 모든 상황에 맞는 단일 구성은
없습니다. 최적의 전략은 대개 튜닝과 실험을 거쳐 찾게 됩니다.

## 병렬화 구성 방법 {#how-to-configure-parallelism}

추론 프레임워크는 이런 전략을 서버 플래그로 제공합니다. 다음은 여러 GPU에서
모델을 서빙하는 몇 가지 예시입니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
# TP=4: one replica sharded across four GPUs
max serve --model meta-llama/Llama-3.3-70B-Instruct \
  --devices=gpu:0,1,2,3

# DP=8: eight single-GPU replicas
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --devices gpu:0,1,2,3,4,5,6,7 \
  --data-parallel-degree 8

# EP=8, DP=1: one replica, experts sharded across eight GPUs
max serve --model nvidia/DeepSeek-V3.1-NVFP4 \
  --devices gpu:0,1,2,3,4,5,6,7 \
  --data-parallel-degree 1 \
  --ep-size 8
```

MAX에서는 `--devices`로 GPU를 선택합니다. `--data-parallel-degree`는 `1`(선택한
모든 GPU에 걸친 TP 레플리카 하나)이거나 GPU 수와 같아야 합니다(GPU당 레플리카
하나). GPU 8개에서 TP=4 × DP=2처럼 부분적으로 나누는 구성은 현재 지원하지
않습니다. `--ep-size`는 `1`(전문가 병렬화 없음)이거나 모든 노드에 걸친 전체 GPU
수여야 합니다.

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
# TP=4: one replica sharded across four GPUs
vllm serve --model meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 4

# TP=4, DP=2: eight GPUs split into two replicas
vllm serve --model meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 4 \
  --data-parallel-size 2

# EP: MoE layers switch from tensor to expert parallelism
vllm serve --model deepseek-ai/DeepSeek-V3 \
  --tensor-parallel-size 8 \
  --enable-expert-parallel
```

vLLM에서 전문가 병렬화는 차수가 아니라 켜고 끄는 옵션입니다. 이 옵션은 TP와 DP가
이미 차지한 GPU들에 MoE 레이어를 분산합니다. `--enable-expert-parallel`이 없으면
MoE 레이어는 밀집(dense) 모델처럼 텐서 병렬화(크기가 TP × DP인 TP 그룹)를
사용합니다.

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
# TP=4: one replica sharded across four GPUs
sglang serve --model-path meta-llama/Llama-3.3-70B-Instruct \
  --tp-size 4

# TP=4, DP=2: eight GPUs split into two replicas
sglang serve --model-path meta-llama/Llama-3.3-70B-Instruct \
  --tp-size 4 \
  --dp-size 2

# EP=8: MoE experts sharded across all eight GPUs
sglang serve --model-path deepseek-ai/DeepSeek-V3 \
  --tp-size 8 \
  --ep-size 8
```

SGLang에서는 각 플래그에 더 긴 별칭도 쓸 수 있습니다. `--tensor-parallel-size`,
`--data-parallel-size`, `--expert-parallel-size`입니다.

</TabItem>
</Tabs>

어떤 프레임워크를 사용하든, 가장 높은 차수가 최선이라고 가정하지 말고 구성을
직접 벤치마크하세요.

<LinkList>

## 추가 자료 {#additional-resources}

- [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/pdf/1909.08053)
- [GPipe: Easy Scaling with Micro-Batch Pipeline Parallelism](https://arxiv.org/pdf/1811.06965)
- [DeepSpeed-MoE: Advancing Mixture-of-Experts Inference and Training to Power Next-Generation AI Scale](https://arxiv.org/abs/2201.05596)
</LinkList>
