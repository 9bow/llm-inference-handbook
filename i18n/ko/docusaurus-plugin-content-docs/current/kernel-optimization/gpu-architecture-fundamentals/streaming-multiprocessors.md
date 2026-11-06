---
sidebar_position: 2
description: 스트리밍 멀티프로세서의 내부 구조, 워프 스케줄링, 블록 상주, GPU 점유율을 이해합니다.
keywords:
    - streaming multiprocessor
    - GPU warp scheduler
    - GPU occupancy
    - CUDA SM
    - block residency
---

import LinkList from '@site/src/components/LinkList'; import
WarpSchedulerVisualizer from '@site/src/components/WarpSchedulerVisualizer';
import SMFloorplan from '@site/src/components/SMFloorplan';

# 스트리밍 멀티프로세서

**스트리밍 멀티프로세서**(streaming multiprocessor, SM)는 NVIDIA GPU의 주요 연산
유닛입니다. GPU에는 많은 SM이 있으며, 각 SM은
[스레드 블록](/kernel-optimization/gpu-architecture-fundamentals/threads-warps-blocks/#thread-blocks)을
받아 그 워프들을 스케줄링하고, 온칩 연산 자원과 메모리 자원으로 명령어를
실행합니다. AMD 하드웨어는 다른 이름을 쓰고 작업을 다르게 묶지만, 많은 병렬 연산
유닛이 워크로드를 나눠 맡는다는 큰 개념은 같습니다.

## SM의 구성 요소 {#what-an-sm-contains}

정확한 설계는 GPU 세대마다 바뀌지만, SM에는 일반적으로 다음이 들어 있습니다.

- 정수 및 부동소수점 산술을 위한 산술 실행 유닛 집합(NVIDIA GPU에서는 CUDA 코어,
  AMD GPU에서는 실행 유닛)
- 행렬 연산을 가속하는
  [텐서 코어](/kernel-optimization/gpu-architecture-fundamentals/tensor-cores/)(최신
  아키텍처의 경우)
- 준비된 워프를 골라 매 사이클 명령어를 발행하는 워프 스케줄러
- 레지스터 파일, 공유 메모리, L1 캐시. 많은 아키텍처에서 공유 메모리와 L1은 온칩
  자원을 공유하며 구성을 바꿀 수 있습니다. 자세한 내용은
  [GPU 메모리 계층 구조](/kernel-optimization/gpu-architecture-fundamentals/gpu-memory/)를
  참고하세요.

:::note
온칩(on-chip)은 GPU 실리콘 다이 위, 연산 유닛 바로 옆에 물리적으로 위치한다는
뜻입니다. 오프칩(off-chip)은 GPU 칩 바깥에 있어서 메모리 인터페이스(배선,
컨트롤러)를 거쳐야 한다는 뜻입니다.
:::

일부 자원은 네 개의 처리 블록으로 나뉘고, 나머지는 SM 전체가 공유합니다. H100
SM에서는 각 처리 블록에 전용 스케줄러, 레지스터 파일, 실행 유닛이 있습니다. 네
처리 블록은 모두 L1과 공유 메모리 풀을 공유합니다.

<SMFloorplan />

:::note
**처리 블록**([SM 하위 파티션](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#streaming-multiprocessor)이라고도
함)은
[스레드 블록](/kernel-optimization/gpu-architecture-fundamentals/threads-warps-blocks/#thread-blocks)과
같지 않습니다. 처리 블록은 SM을 고정적으로 나눈 하드웨어 단위입니다. 스레드
블록은 커널 실행 시점에 사용자가 정하는 소프트웨어상의 스레드 묶음입니다. GPU는
스레드 블록 하나를 SM 하나에 할당하고, 그 블록의 워프들은 SM의 처리 블록들에
분배됩니다.
:::

최신 데이터 센터 GPU에는 SM이 많습니다. 예를 들어 NVIDIA H100 SXM에는 132개,
A100에는 108개의 SM이 있습니다. GPU의 전체 처리량은 커널이 이 SM들을 유용한
작업으로 얼마나 잘 바쁘게 유지하는지에 달려 있습니다.

각 SM은 레지스터, 공유 메모리, 워프 슬롯이 충분하다면 여러 스레드 블록을 동시에
보유하고 실행할 수 있습니다.

## 워프 스케줄링이 지연 시간을 숨기는 방식 {#how-warp-scheduling-hides-latency}

SM은 많은 워프를 상주시킬 수 있습니다. 워프 스케줄러는 준비된 워프를 골라 필요한
실행 유닛에 명령어를 발행합니다. 한 워프가 메모리 로드나 의존성을 기다리면,
스케줄러는 준비된 다른 워프의 작업을 발행할 수 있습니다.

이런 빠른 전환에는 운영체제의 컨텍스트 스위치가 필요하지 않습니다. 상주 워프의
레지스터와 스케줄링 상태가 이미 SM에 올라와 있기 때문입니다. GPU는 이렇게 준비된
작업 풀을 이용해 지연 시간을 숨기고 처리량을 유지합니다.

상주 워프가 많아져도 유용한 대안이 될 때만 도움이 됩니다. 모든 워프가 같은
병목을 기다리거나 커널이 이미 연산 파이프라인을 포화시키고 있다면, 워프를 더
늘려도 이득이 거의 없을 수 있습니다.

아래에서 시나리오를 선택하고 각 열을 위에서 아래로 읽어 보세요. 워프 행은 각
워프가 명령어를 발행 중인지, 메모리를 기다리는 중인지, 준비 상태인지를 보여
줍니다. 마지막 행은 스케줄러가 어떤 워프를 선택하는지 보여 줍니다.

<WarpSchedulerVisualizer />

## 블록 상주 {#block-residency}

GPU는 각 블록을 SM 하나에 할당합니다. 일반적인 실행에서 블록은 완료될 때까지 그
SM에 머무르며, 블록 안의 모든 스레드는 그 SM의 자원을 사용합니다.

함께 상주할 수 있는 블록과 워프의 수는 여러 한도로 결정됩니다.

- SM당 스레드 수와 워프 수
- SM당 블록 수
- SM당 레지스터 수와 각 스레드가 사용하는 레지스터 수
- SM당 공유 메모리 용량과 각 블록이 사용하는 공유 메모리 용량
- 아키텍처별 스케줄링 한도

보통 한 가지 한도가 먼저 제약 조건이 됩니다. 예를 들어 큰 공유 메모리 타일을
할당하는 블록은 스레드 슬롯이 남아 있어도 다른 블록이 같은 SM에 상주하지 못하게
할 수 있습니다.

## 점유율 {#occupancy}

**점유율**(occupancy)은 SM의 활성 워프 수를 그 SM이 지원하는 최대 활성 워프 수로
나눈 비율입니다.

$$
\text{Occupancy} =
\frac{\text{Active Warps per SM}}{\text{Maximum Active Warps per SM}}
$$

점유율이 높으면 스케줄러가 고를 수 있는 워프 풀이 커지고, 지연 시간을 숨기는 데
도움이 될 수 있습니다. 점유율의 상한은 스레드당 레지스터 사용량과 블록당 공유
메모리 사용량이 정하는 경우가 많습니다.

점유율은 무작정 최대화할 점수가 아닙니다. 커널은 유용한 타일을 공유 메모리에
유지하거나, 중간 값을 레지스터에 보관하거나, 반복되는 HBM 트래픽을 줄이기 위해
더 낮은 점유율을 감수할 수 있습니다.
[FlashAttention](/kernel-optimization/flashattention/)이 이 패턴을 따릅니다.
온칩 저장 공간을 더 쓰면 점유율은 낮아질 수 있지만, 훨씬 비싼 오프칩 메모리
이동을 줄입니다.

점유율은 성능을 설명하는 데 쓰고, 측정을 대신하는 데 쓰지 마세요. 프로파일링을
하면 SM에 실행 가능한 워프가 부족한지, 메모리 때문에 멈추는지, 관련 실행
파이프라인이 이미 포화 상태인지 알 수 있습니다.

## 추론에서 SM 사용률이 중요한 이유 {#why-sm-utilization-matters-for-inference}

추론 워크로드가 이상적인 행렬 곱 하나로만 이루어지는 경우는 드뭅니다. 어텐션,
정규화, 샘플링, 데이터 이동, 그리고 수많은 작은 커널이 모두 SM 시간을 두고
경쟁합니다. 실행 구성(launch geometry)이 나쁘거나 병렬 작업이 부족하면 SM이 유휴
상태로 남을 수 있고, 그러면 지연 시간이 늘어납니다. 자원을 과도하게 사용하면
상주 블록 수가 줄어 지연 시간을 숨기기 어려워집니다.

배치 크기와 시퀀스 형태도 병렬 작업의 양에 영향을 줍니다.
[프리필](/llm-inference-basics/how-does-llm-inference-work/#prefill) 중에 GPU를
가득 채우는 커널이라도, 토큰을 하나씩 생성하는
[디코드](/llm-inference-basics/how-does-llm-inference-work/#decode) 중에는 같은
GPU를 충분히 활용하지 못할 수 있습니다. 그래서 사용률은 하나의 종합 백분율로
보지 말고 실제 추론 단계별로 해석해야 합니다.

## 자주 묻는 질문 {#faqs}

### SM이 더 많은 GPU가 항상 커널을 더 빠르게 실행하나요? {#does-a-gpu-with-more-sms-always-run-a-kernel-faster}

아니요. 추가된 SM을 활용하려면 실행 시 독립적인 블록이 충분해야 하고, 커널이 HBM
대역폭 같은 다른 제약에 걸리지 않아야 합니다. SM 설계와 클록 속도도 GPU 세대마다
다르므로, SM 개수만으로는 성능을 온전히 비교할 수 없습니다.

### 낮은 점유율은 항상 문제인가요? {#is-low-occupancy-always-a-problem}

아니요. 낮은 점유율은 SM에 준비된 워프가 부족해 지연 시간을 숨기지 못할 때
문제가 됩니다. 연산 바운드 커널이나 온칩 데이터 재사용이 많은 커널은 중간 수준의
점유율에서도 좋은 성능을 낼 수 있습니다.

<LinkList>

## 추가 자료 {#additional-resources}

- [CUDA C++ Best Practices Guide: Occupancy](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/#occupancy)
- [CUDA Occupancy Calculator](https://docs.nvidia.com/nsight-compute/NsightCompute/index.html#occupancy-calculator)
- [NVIDIA Hopper Architecture In-Depth](https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/)
</LinkList>
