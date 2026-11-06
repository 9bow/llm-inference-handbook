---
sidebar_position: 4
description: LLM 서빙에 필요한 GPU 메모리를 계산하는 방법을 알아봅니다.
keywords:
    - GPU memory calculation, LLM inference hardware calculator
    - VRAM calculation
    - LLM memory requirements
---

import LinkList from '@site/src/components/LinkList';
import GPUMemoryCalculator from '@site/src/components/Calculator/GPUMemory';

# LLM 서빙을 위한 GPU 메모리 계산

LLM을 셀프 호스팅하려고 한다면 가장 먼저 추정해야 할 것 중 하나가 필요한 GPU
메모리(VRAM)의 양입니다.

LLM 추론 중에는 모델이 요청을 서빙하는 동안 모델 가중치가 GPU 메모리에 상주해야
합니다. 이 고정된 기본 사용량은 주로 모델 크기와 가중치에 사용하는 정밀도에 따라
달라집니다.

- **모델 크기(매개변수 수)**. 모델이 클수록 더 많은 메모리가 필요합니다.
  매개변수가 수백억 개에서 수천억 개에 이르는 모델은 보통 NVIDIA H100이나 H200
  같은 고사양 GPU가 필요합니다.
- **비트 정밀도**. 사용하는 정밀도(예: FP16, FP8, INT8)에 따라 메모리 사용량이
  달라집니다. 정밀도가 낮은 형식을 쓰면 메모리 사용량을 크게 줄일 수 있지만
  정확도에 영향을 줄 수 있습니다. Hugging Face에 있는 모델이라면 대개
  `config.json` 파일에서 가중치 데이터 타입을 확인할 수 있습니다. `torch_dtype`
  속성이 모델 가중치에 사용된 정밀도를 나타냅니다. 자세한 내용은
  [LLM 양자화](/model-preparation/llm-quantization/)를 참고하세요.

가중치 메모리가 서빙에 필요한 메모리의 전부는 아닙니다. 예를 들어 FP16 형식의 7B
모델은 가중치에만 약 14GB가 필요하므로 16GB GPU에 로드할 수는 있습니다. 그렇다고
해서 그 GPU에 프로덕션 추론을 위한 여유 공간이 충분하다는 뜻은 아닙니다. 짧은
요청 하나는 처리할 수 있겠지만, 프롬프트가 길어지거나 배치가 커지거나 동시
사용자가 늘어나면 남은 메모리가 금방 바닥납니다.

[KV 캐시](/inference-optimization/kv-cache-offloading/)는 보통 런타임에 가장 큰
추가 메모리를 차지합니다. KV 캐시는 어텐션 키와 값을 저장해 두어, 모델이 디코딩
중에 이전 토큰을 다시 계산하지 않고 재사용할 수 있게 합니다. 캐시는 시퀀스
길이와 활성 요청 수에 따라 커집니다. 서빙 엔진은 이 밖에도 임시 활성화 값, 작업
공간 버퍼, 프레임워크 할당, 때로는 CUDA 그래프를 위한 메모리가 필요합니다.
가중치 외에 GPU 메모리를 아주 조금만 남겨 두면 짧은 컨텍스트, 작은 배치, 낮은
동시성으로 제한됩니다.

서빙 메모리를 추정하는 대략적인 공식은 다음과 같습니다.

```bash
Memory (GB) = P * (Q / 8) * (1 + Overhead)
```

- **P**: 매개변수 수(10억 단위)
- **Q**: 비트 정밀도(예: 16, 32). 8로 나누면 비트가 바이트로 변환됩니다
- **Overhead (%)**: 가중치 외에 서빙에 추가로 필요한 메모리. KV 캐시, 활성화
  버퍼, 작업 공간 메모리, 프레임워크/런타임 예약 메모리 등이 포함됩니다

백분율 기반 오버헤드는 빠르게 규모를 가늠하기 위한 방법일 뿐, 정확한 용량 모델은
아닙니다. KV 캐시 메모리는 시퀀스 길이, 배치 크기, 동시성, 레이어 수, 은닉 차원
크기에 따라 달라집니다. 따라서 긴 컨텍스트 워크로드에는 단순히 10–30%로 잡은
추정치보다 훨씬 더 많은 여유 공간이 필요할 수 있습니다.

아래 계산기로 모델에 필요한 GPU 메모리를 추정해 보세요.

<GPUMemoryCalculator />

:::note
모든 GPU가 모든 정밀도 형식을 네이티브로 지원하지는 않습니다. A100을 비롯한
Ampere GPU는 INT8을 지원하지만 하드웨어 차원에서 FP8은 지원하지 않습니다.
네이티브 FP8을 쓰려면 Hopper, Ada 또는 그 이후 아키텍처가 필요합니다. 추론
스택이 FP8 커널에 의존한다면 GPU가 이를 지원하는지 확인하세요. 일부 4비트 모델은
INT4 양자화를 사용하며, 네이티브 FP4 지원은 더 새로운 아키텍처와 소프트웨어
스택에 의존합니다.
:::

<LinkList>

## 추가 자료 {#additional-resources}

- [GPU 아키텍처 기초](/kernel-optimization/gpu-architecture-fundamentals/)
</LinkList>
