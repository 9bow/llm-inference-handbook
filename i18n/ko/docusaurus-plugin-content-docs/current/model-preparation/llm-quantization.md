---
sidebar_position: 2
description: LLM 양자화의 개념과 다양한 양자화 형식 및 방법을 알아봅니다.
keywords:
    - LLM quantization, how does quantization work, LLM quantization accuracy
    - Quantization formats, quantization types, quantization techniques
    - Quantization vs pruning
    - AWQ, SmoothQuant, GPTQ
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import
LinkList from '@site/src/components/LinkList'; import QuantizationVisualizer
from '@site/src/components/Calculator/Quantization';

# LLM 양자화(quantization)

양자화(quantization)는 모델의 가중치와 활성화 값(activations)을 고정밀도
형식(FP32 등)에서 INT8, INT4, 심지어 INT2 같은 저정밀도 형식으로 변환하여 모델의
메모리 및 연산 요구량을 줄이는 기법입니다.

비트 수가 적을수록 모델의 메모리 사용량도 줄어듭니다. 예를 들면 다음과 같습니다.

- FP32 형식의 7B 모델은 정밀도가 매우 높지만, 가중치만으로도 28 GB의 메모리가
  필요합니다.
- 같은 모델을 FP16으로 저장하면 메모리 사용량이 절반으로 줄어듭니다.
- INT8이나 INT4 같은 저정밀도 형식은 모델을 더욱 압축하여 크기를 크게 줄입니다.

이 수치는 모델 가중치만 고려한 것입니다. 어텐션 캐시, 활성화 값, 프레임워크
오버헤드 같은 런타임 요소에는 추가 메모리가 필요합니다.

## 양자화를 사용하는 이유 {#why-use-quantization}

양자화는 크게 세 가지 방식으로 LLM 추론에 도움이 됩니다.

- **모델 크기 축소**. 매개변수당 비트 수는 모델 가중치에 필요한 메모리 양에 직접
  영향을 줍니다. 예를 들어 7B 모델은 FP16 가중치로는 약 14 GB가 필요하지만, INT8
  가중치로는 약 7 GB면 충분합니다. 이 차이로 모델을 GPU 한 대에 올릴 수 있는지,
  아니면 여러 GPU나 노드에 나눠야 하는지가 갈릴 수 있습니다.
- **데이터 이동 감소**. LLM 디코딩은 토큰을 생성하는 동안 런타임이 모델 가중치를
  반복해서 읽기 때문에 GPU 메모리 대역폭이 병목이 되는 경우가 많습니다. 가중치의
  정밀도가 낮으면
  [GPU 메모리에서 연산 유닛으로](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/)
  옮겨야 하는 바이트 수가 줄어들어 토큰당 지연 시간을 낮출 수 있습니다.
- **더 빠른 연산**. GPU와 기타 가속기는 지원하는 저정밀도 형식을 FP32나 FP16보다
  높은 처리량으로 처리할 수 있습니다. 예를 들어
  [H100 SXM에서는](https://www.nvidia.com/en-us/data-center/h100/) BF16/FP16
  텐서 코어가 1,979 TFLOPS에 이르는 반면, FP8과 INT8은 그 두 배인 3,958
  TFLOPS/TOPS에 이릅니다. 비트 폭을 절반으로 줄여 정확히 2x를 얻는 셈입니다.
  실제 속도 향상은 하드웨어와 추론 런타임이 선택한 형식에 맞는 최적화된 커널을
  제공하는지에 따라 달라집니다.

가중치가 차지하는 메모리가 줄어들면
[KV 캐시](/llm-inference-basics/how-does-llm-inference-work/#the-two-phases-of-llm-inference),
더 큰 배치, 더 많은 동시 요청에 쓸 수 있는 GPU 메모리도 늘어납니다. 가중치
양자화만으로는 토큰당 KV 캐시 크기가 줄어들지 않습니다. KV 캐시는 별도로
양자화해야 합니다.

정밀도와 크기 사이의 이러한 트레이드오프에는 **어느 정도의 정확도 하락**이
따릅니다. 많은 애플리케이션에서 위의 이점은 생성된 출력이 프로덕션에서 쓸 수
있을 만큼 신뢰할 만할 때에만 의미가 있습니다. 예를 들어 더 빠르지만 응답 품질이
눈에 띄게 나쁜 모델은 좀처럼 감수할 만한 선택이 아닙니다.

다행히 최신 양자화 방법 덕분에 이 트레이드오프는 훨씬 덜 심각해졌습니다. GPTQ
W4A16, AWQ, 그리고 가중치와 활성화 값을 모두 양자화하는 FP8 양자화 같은 기법은
추론 효율을 의미 있게 개선하면서도 원본 모델과
[거의 같은 정확도를 유지하는](https://developers.redhat.com/articles/2024/10/17/we-ran-over-half-million-evaluations-quantized-llms)
경우가 많습니다. 그 결과, 많은 프로덕션 배포에서 모델 품질에 거의 또는 전혀 눈에
띄는 영향 없이 양자화를 도입할 수 있습니다.

## 양자화 형식 {#quantization-formats}

양자화 형식마다 크기 절감과 정확도 사이의 균형이 다릅니다. 간단히 정리하면
다음과 같습니다.

| 형식     | FP32 대비 크기 | 정확도 하락 | 사용 사례    | 메모리    | 비고                      |
|----------|----------------|-------------|--------------|-----------|---------------------------|
| **FP32** | 100%           | 없음        | 학습         | 높음      | 완전 정밀도이지만 느림    |
| **FP16** | 50%            | 최소        | 학습 및 추론 | 중간      | 대부분의 LLM에서 표준     |
| **FP8**  | 25%            | 낮음        | 학습 및 추론 | 낮음      | 아직 확산 중              |
| **INT8** | 25%            | 낮음        | 추론         | 낮음      | 전반적으로 균형이 좋음    |
| **INT4** | 12.5%          | 보통        | 추론         | 매우 낮음 | GPTQ/AWQ 같은 방법이 필요 |
| **INT2** | 6.25%          | 높음        | 드묾/실험적  | 극소      | 정확도가 낮은 경우가 많음 |

아래 시각화 도구로 모델 크기에 따라 이러한 트레이드오프가 어떻게 나타나는지
확인해 보세요. MoE 모델의 경우 이 계산기는 토큰당 활성화되는 매개변수가 아니라
저장된 전체 매개변수를 기준으로 계산합니다.

<QuantizationVisualizer />

이 계산기는 **가중치 메모리만** 추정합니다. 실제 사용량은 KV 캐시, 활성화 값,
프레임워크 오버헤드 때문에 훨씬 클 수 있습니다. 전체 요구량을 추정하려면
[GPU 메모리 계산기](/getting-started/calculating-gpu-memory-for-llms/)를
사용하세요.

## 무엇을 양자화할 것인가 {#what-to-quantize}

일반적으로는 성능을 크게 해치지 않으면서 메모리를 가장 많이 차지하는 부분에
집중하는 것이 좋습니다.

- 모델 가중치는 가장 흔하게 양자화하는 구성 요소입니다. 가중치는 값이 안정적이고
  메모리 사용량에서 큰 비중을 차지합니다.
- 활성화 값도 양자화할 수 있지만, 더 까다롭고 정확도 손실이 더 클 수 있습니다.
- KV 캐시는 긴 컨텍스트를 서빙할 때 메모리 부담을 줄이기 위해 런타임에 양자화할
  수 있습니다. KV 캐시 양자화는 가중치 양자화와 다릅니다. 캐시는 추론 중에
  생성되고 디코딩 동안 반복해서 읽히기 때문입니다. 핵심 과제는 어텐션 품질을
  유지하는 것입니다. 키(key)와 값(value) 벡터를 더 적은 비트로 저장한 뒤에도
  모델은 여전히 정확한 키/쿼리 유사도 점수를 얻어야 합니다.

## 양자화와 가지치기 비교 {#quantization-vs-pruning}

모델 크기를 줄이는 방법이 양자화만 있는 것은 아닙니다. 관련된 또 다른 기법으로
모델 가지치기(pruning)가 있습니다.

가지치기는 모델 출력에 거의 기여하지 않는 매개변수를 제거합니다. 제거 대상은
개별 가중치, 뉴런, 어텐션 헤드, 심지어 레이어 전체일 수도 있습니다. 가지치기는
불필요한 구성 요소를 없애 더 작고 희소한 모델을 만들며, 그 결과 연산 요구량이
줄고 추론 속도가 빨라질 수 있습니다.

가지치기와 양자화는 배포 파이프라인에서 함께 쓰이는 경우가 많습니다.

1. 모델 학습
2. 덜 중요한 가중치 가지치기
3. 모델 미세조정
4. 가중치 양자화
5. 추론용으로 배포

간단히 말하면 다음과 같습니다.

- 양자화는 각 가중치를 표현하는 데 쓰는 비트 수를 줄입니다.
- 가지치기는 모델의 가중치 개수를 줄입니다.

두 기법 모두 추론 중 메모리 사용량과 연산 비용을 줄이는 것이 목표입니다. 다만
최신 하드웨어가 저정밀도 연산을 강력하게 지원하므로, 일반적으로 프로덕션
시스템에는 양자화를 적용하기가 더 쉽습니다.

## 양자화를 사용해야 할 때 {#when-to-use-quantization}

다음과 같은 경우 양자화가 좋은 선택입니다.

- GPU 메모리가 제한된 하드웨어(예: 24GB 이하)에 배포하는 경우
- 추론 지연 시간을 줄이고 싶은 경우
- 서빙 비용을 줄여야 하는 경우
- 더 높은 동시성을 지원하고 싶은 경우. 양자화는 토큰당 KV 캐시 크기를 줄여 같은
  GPU 메모리에 더 많은 토큰(따라서 더 많은 병렬 요청)을 담을 수 있게 합니다.
- 약간의 정확도 손실을 감수할 수 있는 경우

다음과 같은 경우에는 양자화가 좋은 선택이 아닐 수 있습니다.

- 가능한 한 가장 높은 정확도가 필요한 경우(예: 민감하거나 안전이 중요한 작업)
- 모델이 이미 작은 경우(이때 양자화의 이점은 제한적입니다)
- 배포 하드웨어가 양자화된 형식을 지원하지 않는 경우

:::tip
많은 인기 파운데이션 모델의 양자화된 변형이 이미 Hugging Face에 공개되어
있으므로 모델을 직접 양자화할 필요가 없습니다. 이러한 변형은 보통 모델
트리(model tree) 섹션에서 찾을 수 있습니다.
:::

## 양자화 방법 {#quantization-methods}

성능을 크게 떨어뜨리지 않으면서 LLM의 효율을 높이기 위해 여러 고급 양자화 기법이
개발되었습니다.

널리 쓰이는 양자화 방식은 다음과 같습니다.

### AWQ {#awq}

[Activation-aware Weight Quantization (AWQ)](https://arxiv.org/pdf/2306.00978)는
엣지 기기나 자원이 제한된 기기에서 LLM을 실행하기 위해 특별히 설계된 방법입니다.
핵심 통찰은 모든 가중치가 성능에 똑같이 기여하지는 않는다는 것입니다. AWQ
개발자들은 가중치의 약 1%만이 "중요(salient)"하며 양자화 과정에서 특별히 다뤄야
한다고 봅니다. 따라서 AWQ는 가중치 자체만이 아니라 활성화 값의 분포를 기준으로
가장 영향력이 큰 가중치를 선택적으로 보호합니다.

큰 틀에서 보면, AWQ는 오프라인으로 수집한 활성화 값 통계를 바탕으로 중요한
가중치 채널의 스케일을 조정하는 등가 변환을 적용합니다.

AWQ는 엣지 환경이나 지연 시간에 민감한 환경에 배포하는 모델의 저비트 양자화에
적합합니다.

### SmoothQuant {#smoothquant}

[SmoothQuant](https://arxiv.org/abs/2211.10438)는 범용적이고 학습이 필요 없는
학습 후 양자화(post-training quantization, PTQ) 방법으로, 가중치와 활성화 값을
모두 효율적으로 8비트 양자화(W8A8)할 수 있게 해 줍니다.

가중치 양자화는 비교적 간단하지만, 활성화 값은 정확도를 크게 떨어뜨릴 수 있는
이상치(outlier) 때문에 훨씬 어렵습니다. SmoothQuant는 활성화 값의 이상치를
"평활화(smoothing)"하여 이 문제를 해결합니다. 등가 변환으로 양자화의 어려움을
활성화 값에서 가중치로 수학적으로 옮기는 방식입니다. 그 결과 LLM에서 최대 2×
메모리 절감과 최대 1.56× 속도 향상을 달성합니다.

SmoothQuant는 다음이 필요할 때 훌륭한 선택입니다.

- 완전한 INT8 양자화(가중치와 활성화 값 모두)
- 재학습 없이 높은 하드웨어 효율
- 최소한의 정확도 하락
- 대부분의 트랜스포머 모델과 바로 쓸 수 있는 호환성

SmoothQuant는 정확도, 성능, 사용 편의성의 균형을 맞춘 즉시 사용 가능한
솔루션입니다. 대규모 환경에서 효율이 요구되는 프로덕션 시나리오에 적합합니다.

### GPTQ {#gptq}

[GPTQ](https://arxiv.org/abs/2210.17323)는 대규모 트랜스포머 모델을 최소한의
정확도 손실로 가중치당 3–4비트로 압축하는 빠른 학습 후 양자화 방법입니다. 수천억
개의 매개변수를 가진 모델까지 재학습 없이 확장할 수 있도록 특별히
설계되었습니다.

주요 특징은 다음과 같습니다.

- 대규모에서도 효율적: OPT-175B나 BLOOM-176B 같은 모델을 약 4 GPU 시간 만에
  양자화할 수 있습니다.
- 최소한의 정확도 손실: 공격적으로 압축해도 낮은 펄플렉서티를 유지합니다.
- 극단적 양자화: 2비트 및 3진(ternary) 양자화를 지원하면서도 쓸 만한 성능을
  냅니다.
- 단일 GPU에서 초대형 모델 실행: A100 한 대나 A6000 두 대로 175B 모델 추론이
  가능합니다.
- 성능 향상: 커스텀 GPU 커널로 FP16 대비 약 3.25× 속도 향상을 얻습니다.

GPTQ는 오픈소스 모델 서빙 파이프라인, 특히 AutoGPTQ와 함께 널리 쓰입니다. 대형
모델을 빠르고 적은 메모리로 추론하고자 할 때 가장 먼저 고려되는 선택지입니다.

---

많은
[최신 추론 프레임워크](/getting-started/choosing-the-right-inference-framework/)는
양자화된 모델을 효율적으로 서빙할 뿐만 아니라, 모델을 양자화하기 위한 API나
도구도 기본으로 제공합니다. 예를 들어 온라인 양자화는 보통 서빙 플래그 하나만
지정하면 됩니다.

<Tabs groupId="inference-framework">
<TabItem value="max" label="MAX">

```bash
max serve --model meta-llama/Llama-3.1-8B-Instruct \
  --quantization-encoding float8_e4m3fn
```

`--quantization-encoding`은 고정밀도 인코딩(`float32`, `float16`, `bfloat16`),
저정밀도 부동소수점 형식(`float8_e4m3fn`, `float4_e2m1fnx2`), GGUF 정수
형식(`q4_0`, `q4_k`, `q6_k`), 그리고 GPTQ 체크포인트용 `gptq`를 지원합니다.
자세한 내용은
[MAX 문서의 Quantization](https://max.modular.com/develop/quantize/)을
참고하세요.

</TabItem>
<TabItem value="vllm" label="vLLM">

```bash
vllm serve --model meta-llama/Llama-3.1-8B-Instruct \
  --quantization mxfp8
```

`--quantization`은 `fp8_per_tensor`, `fp8_per_block`, `mxfp8`, `bitsandbytes`
같은 방법을 지원합니다. 이 값을 지정하지 않으면 vLLM은 모델 설정에서
`quantization_config`를 읽고, 없으면 양자화되지 않은 가중치를 사용합니다. 자세한
내용은
[vLLM 문서의 Quantization](https://docs.vllm.ai/en/latest/features/quantization/)을
참고하세요.

</TabItem>
<TabItem value="sglang" label="SGLang">

```bash
sglang serve --model-path meta-llama/Llama-3.1-8B-Instruct \
  --quantization fp8
```

`--quantization`은 `awq`, `gptq`, `fp8`, `bitsandbytes` 같은 방법을 지원합니다.
자세한 내용은
[SGLang 문서의 Quantization](https://docs.sglang.io/docs/advanced_features/quantization)을
참고하세요.

</TabItem>
</Tabs>

그 밖의 경우에는 전용 도구로 모델을 오프라인에서 양자화한 뒤, 서빙 프레임워크가
이를 바로 불러옵니다. 그 결과 대부분의 사용자는 더 이상 양자화 알고리즘을 직접
구현할 필요가 없습니다.

보통은
[Hugging Face](/getting-started/choosing-the-right-model/#hugging-face)에서 이미
양자화된 모델로 시작할 수 있습니다. Hugging Face에는 8비트, 4비트 모델처럼 미리
양자화된 변형이 많이 공개되어 있습니다. 이 변형은 바로 추론에 쓸 수 있고, 메모리
사용량을 줄이고 배포를 빠르게 하도록 최적화되어 있습니다. 동시에 직접 양자화
전략을 적용하고 싶은 경우를 위해 완전 정밀도 베이스 모델도 제공합니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference](https://arxiv.org/abs/1712.05877)
- [AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://arxiv.org/abs/2306.00978)
- [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323)
- [SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://arxiv.org/abs/2211.10438)
</LinkList>
