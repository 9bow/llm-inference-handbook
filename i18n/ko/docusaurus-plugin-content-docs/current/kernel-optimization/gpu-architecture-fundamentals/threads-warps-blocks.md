---
sidebar_position: 1
description: GPU 커널이 작업을 스레드, 워프, 스레드 블록, 그리드로 어떻게 구성하는지 알아봅니다.
keywords:
    - GPU threads
    - CUDA warps
    - thread blocks
    - CUDA grids
    - SIMT execution, warp divergence
---

import LinkList from '@site/src/components/LinkList'; import
WarpDivergenceVisualizer from '@site/src/components/WarpDivergenceVisualizer';

# GPU 스레드, 워프, 블록, 그리드

GPU 커널은 대량의 병렬 작업을 드러냅니다. 실행 모델은 이 작업을 스레드(thread),
워프(warp), 스레드 블록(thread block), 그리드(grid)의 계층 구조로 구성합니다. 각
단계는 서로 다른 질문에 답합니다. 논리적 작업자 하나가 무엇을 하는지, 어떤
작업자들이 함께 실행되는지, 어떤 작업자들이 서로 협력할 수 있는지, 그리고 전체
커널 실행이 입력을 어떻게 빠짐없이 처리하는지입니다.

## 스레드 {#threads}

**스레드**는 AMD GPU에서는 **작업 단위**(work unit)라고도 부르며, 커널 함수 안의
가장 작은 논리적 실행 단위입니다. 모든 스레드는 같은 커널 코드를 실행하지만,
내장 좌표를 사용해 스레드마다 서로 다른 데이터를 선택할 수 있습니다. 이러한
모델을 SIMT(Single Instruction, Multiple Threads)라고 합니다.

각 스레드가 할 일을 직접 하나씩 지정하지는 않습니다. 대신 보통 각 스레드는 블록
안에서의 자기 위치와 블록의 위치를 조합해 입력의 어느 부분을 처리할지
계산합니다. 자세한 내용은 아래의 [그리드와 인덱싱](#grids-and-indexing)을
참고하세요.

같은 커널 본문으로 원소가 수천 개 또는 수백만 개인 벡터를 처리할 수 있습니다.
스레드는 2차원이나 3차원 좌표도 사용할 수 있으며, 이러한 좌표는 행렬, 이미지,
타일 단위 연산에 자연스럽게 대응됩니다.

## 워프 {#warps}

**워프**는 AMD GPU에서는 **웨이브프런트**(wavefront)라고도 부르며, 스레드 블록에
속한 스레드 중 함께 실행되는 일부 스레드의 묶음입니다. NVIDIA GPU에서 워프
하나는 32개의 스레드로 이루어집니다. AMD의 웨이브프런트는 전통적으로 64개이며,
RDNA에서는 32개 또는 64개입니다.

워프는 실제 스케줄링 단위입니다. 워프 스케줄러에 전달되는 단위가 바로 워프이며,
워프에는 한 번에 명령어 하나만 발행될 수 있습니다. 워프가 명령어를 받으면 워프
안의 활성 스레드들이 각자 자신의 레지스터와 데이터에 대해 그 명령어를
실행합니다. 서로 다른 워프는 독립적이므로 같은 시점에 서로 다른 명령어를 실행할
수 있습니다.

워프 안의 스레드들이 같은 결과를 낼 필요는 없지만, 제어 흐름은 모든 스레드가
같은 경로를 따를 때 가장 효율적입니다. 분기 때문에 일부 스레드는 한쪽 경로로,
나머지 스레드는 다른 경로로 가게 되면 워프는 각 경로를 따로 실행합니다. 경로마다
활성 스레드 마스크가 달라서 현재 경로에 있는 스레드만 활성화됩니다. 이를
**워프 분기**(warp divergence)라고 합니다. 갈라진 경로에 작업량이 많으면 현재
경로에 속하지 않은 스레드가 유휴 상태로 기다려야 하므로 효율이 떨어집니다.

<WarpDivergenceVisualizer />

## 스레드 블록 {#thread-blocks}

**스레드 블록**(또는 **블록**)은 AMD GPU에서는 **워크그룹**(workgroup)이라고도
부르며, 그리드에 속한 스레드의 부분 집합입니다. 그리드는 커널 함수를 실행하는
스레드들을 구성하는 최상위 구조입니다. 스레드 블록은 워크로드 분배의 기본 구성
요소로서 다음과 같은 중요한 역할을 합니다.

1. 커널 함수의 전체 워크로드(그리드가 관리함)를 독립적으로 처리할 수 있는 더
   작고 다루기 쉬운 부분으로 나눕니다. 이렇게 나누면 GPU에 있는 여러
   [스트리밍 멀티프로세서(SM)](/kernel-optimization/gpu-architecture-fundamentals/streaming-multiprocessors/)에
   걸쳐 자원을 더 잘 활용하고 스케줄링을 더 유연하게 할 수 있습니다.
2. 스레드 블록은 스레드들이
   [공유 메모리](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/#shared-memory-smem-and-l1-cache)와
   동기화 프리미티브를 통해 협력할 수 있는 범위를 제공합니다. 덕분에 효율적인
   병렬 알고리즘과 데이터 공유 패턴을 구현할 수 있습니다.
3. 스레드 블록은 확장성을 높입니다. 하드웨어가 사용 가능한 자원에 따라 블록을
   자동으로 분배할 수 있으므로, 같은 프로그램이 여러 GPU 아키텍처에서 효율적으로
   실행됩니다.

그리드 안의 스레드 블록 개수와, 블록을 1차원, 2차원, 3차원 중 어떻게 배치할지
지정할 수 있습니다. 그리드 안의 각 블록에는 그리드 내 위치를 나타내는 고유한
블록 인덱스가 부여됩니다. 마찬가지로 스레드 블록당 스레드 개수와, 스레드를
1차원, 2차원, 3차원 중 어떻게 배치할지도 지정합니다. 현재 NVIDIA와 AMD GPU는
보통 블록당 최대 1024개의 스레드를 허용하며, 기기별 차원, 레지스터, 공유 메모리
한도의 제약을 받습니다.

GPU는 그리드 안의 각 스레드 블록을 스트리밍 멀티프로세서(SM)에 할당합니다.
블록은 일반적으로 완료될 때까지 해당 SM에 상주하며, SM 사이에서 이동하지
않습니다. 레지스터, 공유 메모리, 스케줄러 슬롯에 여유가 있으면 SM 하나에 여러
블록이 동시에 상주할 수 있습니다.

블록 안의 스레드들은 공유 메모리로 데이터를 공유하고 내장 메커니즘으로 동기화할
수 있습니다. 하지만 다른 블록의 스레드와는 직접 통신할 수 없습니다.

## 그리드와 인덱싱 {#grids-and-indexing}

**그리드**는 커널 실행을 정의하는 최상위 구성 구조입니다. 그리드는 모든 스레드
블록을 담고, 스레드 블록은 실행되는 모든 스레드를 담습니다.

GPU 커널을 작성하려면 그리드를 만들어 수행할 작업을 지정해야 합니다. NVIDIA
GPU의 CUDA 프로그래밍 모델에서는 다음 값으로 커널의 실행 차원과 좌표를
정의합니다.

- `gridDim`: 그리드 차원, 즉 각 차원의 블록 개수입니다.
- `blockDim`: 블록 차원, 즉 블록의 각 차원에 있는 스레드 개수입니다.
- `blockIdx`: 그리드 안에서 현재 블록의 위치입니다.
- `threadIdx`: 블록 안에서 현재 스레드의 위치입니다.

커널은 보통 이 값들을 조합해 전역 스레드 ID를 계산하고, 이를 바탕으로 각
스레드가 처리할 원소나 데이터 부분을 결정합니다.

## 커널 함수 작성하기 {#writing-a-kernel-function}

이제 GPU 작업이 스레드, 워프, 스레드 블록, 그리드로 어떻게 구성되는지 알았으니
커널 코드를 더 잘 이해할 수 있습니다.

다음은 벡터 덧셈을 수행하는 간단한 CUDA 커널입니다. 각 스레드가 원소 하나를
처리합니다.

```cpp
__global__ void vecAdd(float* A, float* B, float* C, int vectorLength)
{
    int workIndex = threadIdx.x + blockDim.x * blockIdx.x;
    if (workIndex < vectorLength)
    {
        C[workIndex] = A[workIndex] + B[workIndex];
    }
}
```

그리드가 입력을 딱 맞게 나누는 경우는 드물기 때문에 여기서 경계 검사가
중요합니다. 커널 실행 자체에는 세 겹의 꺾쇠괄호 안에 **실행 구성**(execution
configuration)을 지정합니다. 첫 번째 값은 그리드의 블록 개수를, 두 번째 값은
블록당 스레드 개수를 정합니다.

```cpp
int threads = 256;
int blocks = (vectorLength + threads - 1) / threads;

vecAdd<<<blocks, threads>>>(devA, devB, devC, vectorLength);
```

그리드에는 보통 GPU가 한 번에 실행할 수 있는 것보다 많은 블록이 들어 있습니다.
블록이 끝나면 GPU는 비어 있는 SM 용량에 새 블록을 할당합니다. 이러한 스케줄링
모델 덕분에 커널을 바꾸지 않고도 한 번의 실행이 SM 개수가 서로 다른 GPU에 맞게
확장됩니다.

하지만 위의 CUDA 코드는 NVIDIA GPU에서만 동작합니다. AMD GPU를 프로그래밍하려면
AMD의 네이티브 ROCm 소프트웨어를 대신 사용할 수 있습니다. 두 프레임워크 모두
C/C++를 핵심 프로그래밍 언어로 사용하며, 몇 가지 키워드와 문법 확장이 추가되어
있습니다.

또 다른 방법으로, [MAX](https://www.modular.com/open-source/max)는
[Mojo 프로그래밍 언어](https://mojolang.org/)를 사용해 GPU와 기타 가속기를 위한
하드웨어에 구애받지 않는 프로그래밍 모델을 제공합니다. 이 프레임워크는 위에서
설명한 것과 같은 그리드-블록 프로그래밍 모델을 제공하지만, 코드가 NVIDIA GPU,
AMD GPU, Apple silicon 등을 대상으로 할 수 있다는 점이 다릅니다. 예를 들어,
다음은 같은 벡터 덧셈 함수를 Mojo로 작성한 것입니다.

```mojo
from std.gpu import block_dim, block_idx, thread_idx
from max.gpu.host import DeviceContext

def kernel():
    var i = block_idx.x * block_dim.x + thread_idx.x
    # Process element i.

def main() raises:
    var ctx = DeviceContext()
    # Same geometry as the CUDA launch above.
    ctx.enqueue_function[kernel](grid_dim=4096, block_dim=256)
    ctx.synchronize()
```

`grid_dim`과 `block_dim`은 CUDA 실행 시 지정하는 두 값에 대응하며, 인덱스 계산
방식도 같습니다. Mojo는 `block_idx.x * block_dim.x + thread_idx.x`를 대신 계산해
주는 축약형인 `global_idx`도 제공하므로, 같은 줄을 `var i = global_idx.x`로 쓸
수도 있습니다.

:::note
프로그래밍 모델의 선택은 추론 프레임워크 안에서 커널을 커스터마이즈할 때
드러납니다. vLLM과 SGLang은 모두 벤더 툴체인 위에 구축되어 있습니다. 두
프레임워크의 커널은 NVIDIA용은 CUDA C++, AMD용은 ROCm/HIP로 작성되어 있으며,
일부 연산에는
[Triton](/kernel-optimization/kernel-optimization-tools/#triton-and-python-kernel-dsls)과
기타 커널 라이브러리를 함께 사용합니다. 한 벤더용으로 작성한 커널은 보통 다른
벤더용으로 별도의 변형과 별도의 튜닝이 필요합니다. MAX는 다른 접근 방식을
취합니다. 같은 Mojo 커널 구현으로 NVIDIA와 AMD GPU를 모두 대상으로 할 수
있습니다.
:::

NVIDIA GPU 프로그래밍에 대해 알아보려면
[CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/intro-to-cuda-cpp.html)를
참고하세요.

어떤 GPU든 프로그래밍하는 방법을 알아보려면
[Mojo GPU programming guide](https://max.modular.com/gpu/intro-tutorial/)를
참고하세요.

## 계층 구조가 커널 성능에 미치는 영향 {#how-the-hierarchy-affects-kernel-performance}

실행 구성(launch geometry)은 커널이 하드웨어를 사용하는 방식을 바꿉니다.

- 블록에는 보통 여러 워프를 구성할 만큼 충분한 스레드가 필요합니다. 하지만
  블록이 지나치게 크면 레지스터나 공유 메모리를 너무 많이 차지할 수 있습니다.
- 가능하면 인접한 스레드가 인접한 전역 메모리 주소에 접근해야 합니다. 그래야
  GPU가 메모리 요청을 효율적인 트랜잭션으로 합칠 수 있습니다.
- 알고리즘이 허용하는 한, 분기가 있더라도 워프 안의 스레드들이 같은 경로를
  따르도록 해야 합니다.
- 그리드는 모든 SM을 바쁘게 유지할 만큼 충분한 독립 블록을 제공해야 합니다.

가장 좋은 실행 구성은 스레드당 작업량과 각 블록에 필요한 자원에 따라 달라집니다.
자원 사용량은
[점유율](/kernel-optimization/gpu-architecture-fundamentals/streaming-multiprocessors/#occupancy)에도
영향을 줍니다. "항상 256개의 스레드를 사용하라" 같은 고정 규칙은 출발점일 뿐,
모든 경우에 맞는 정답은 아닙니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/index.html)
- [AMD ROCm HIP programming model](https://rocm.docs.amd.com/projects/HIP/en/latest/understand/programming_model.html)
- [Mojo GPU programming fundamentals](https://max.modular.com/gpu/fundamentals/)
</LinkList>
