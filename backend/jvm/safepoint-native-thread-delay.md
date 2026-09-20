# Safepoint 지연과 Native Thread 분석

## 핵심 요약

Java 21 + ZGC 환경에서 p99 latency가 평소 수십 ms 수준에서 간헐적으로 1~2초까지 상승했지만, GC pause는 약 1ms, DB/Redis/외부 API latency는 정상, CPU 사용률은 약 60%, BLOCKED thread 증가도 거의 없었다.

JFR에서 latency spike와 같은 시점에 긴 Safepoint가 관찰되었고, 세부 시간은 다음과 같았다.

```text
Safepoint total      1.9s
Time to safepoint    1.896s
VM operation         16ms
```

핵심은 JVM이 Safepoint 상태에서 수행한 VM operation 자체가 오래 걸린 것이 아니라, 모든 thread가 Safepoint에 도달할 때까지 기다리는 시간이 거의 전체 지연을 차지했다는 점이다.

따라서 Safepoint 지연을 볼 때는 단순히 "Safepoint가 길다"고 판단하지 말고, 반드시 다음 두 구간을 분리해서 봐야 한다.

1. 모든 thread가 Safepoint에 도달하는 데 걸린 시간
2. Safepoint 상태에서 JVM이 실제 작업한 시간

이 사례에서는 특정 thread가 JNI/native 영역에서 오래 머물며 Safepoint 진입을 늦췄을 가능성을 추적하는 것이 핵심이었다.

## 개념

### Safepoint는 GC와 동일하지 않다

Safepoint는 JVM이 전역 작업을 안전하게 수행하기 위해 Java thread들을 정해진 안전 지점에 멈추게 하는 메커니즘이다.

GC가 Safepoint를 사용할 수는 있지만 다음과 같은 작업도 Safepoint를 사용할 수 있다.

- GC 관련 작업
- class metadata 처리
- VM operation
- deoptimization
- 일부 class loading/unloading 관련 작업
- 기타 JVM 전역 상태 변경 작업

따라서 다음 등식은 성립하지 않는다.

```text
Safepoint = GC
```

긴 Safepoint가 보였다고 해서 GC부터 원인으로 단정하면 안 된다.

### Safepoint 시간의 구성

Safepoint 지연은 개념적으로 두 구간으로 나누어 보는 것이 중요하다.

```text
Safepoint 요청
    ↓
모든 thread가 Safepoint에 도달할 때까지 대기
    ↓
Safepoint 진입 완료
    ↓
JVM이 VM operation 수행
    ↓
Safepoint 종료
```

즉 전체 시간은 대략 다음처럼 볼 수 있다.

```text
Safepoint total
≈ Time to safepoint
+ Safepoint 내 VM operation 수행 시간
```

`Time to safepoint`가 길다면 JVM 작업 자체보다 어떤 thread가 Safepoint에 늦게 도달하는지를 추적해야 한다.

## 동작 원리

### RUNNABLE은 CPU를 적극적으로 사용 중이라는 뜻이 아니다

문제가 된 thread는 다음과 같은 상태였다.

```text
RUNNABLE
CPU 사용량 거의 없음
Java lock 대기 아님
Socket / DB I/O 대기 아님
```

이 상태는 처음 보면 모순처럼 보일 수 있다. 하지만 Java의 `RUNNABLE` 상태는 "현재 CPU에서 Java bytecode를 열심히 실행 중"이라는 뜻이 아니다.

Java thread가 JNI 또는 native code 내부에 있더라도 Java 관점에서는 `RUNNABLE`로 보일 수 있다.

예를 들면 다음 경로가 가능하다.

```text
Java code
  ↓
JNI
  ↓
Native Library
  ↓
OS Syscall
  ↓
Kernel
```

thread가 native 영역이나 특정 syscall/kernel 경로에 오래 머무르면, JVM이 기대하는 Safepoint 확인 지점에 빠르게 도달하지 못할 수 있다.

다른 모든 thread가 이미 Safepoint에 도달했더라도 thread 하나가 늦으면 JVM 전체가 Safepoint 진입을 완료하지 못한다.

결과적으로 애플리케이션에서는 다음과 같은 현상이 나타날 수 있다.

```text
특정 native thread의 지연
    ↓
Time to safepoint 증가
    ↓
전체 Java thread의 진행 정지
    ↓
p99 latency 급증
```

이때 GC pause 자체는 짧을 수 있으므로 GC 지표만 보면 문제를 놓칠 수 있다.

## 분석 도구와 역할

### JFR: JVM 내부에서 무슨 일이 발생했는가

JFR(Java Flight Recorder)은 JVM 내부 이벤트를 시간축으로 분석하는 데 적합하다.

대표적으로 확인할 수 있는 항목은 다음과 같다.

- GC
- Safepoint
- Thread
- Lock
- Allocation
- Class Loading
- VM Operation

JFR은 다음 질문에 답하는 도구로 생각하면 좋다.

> JVM 안에서 언제, 어떤 이벤트가 발생했는가?

이 사례에서는 latency spike와 긴 Safepoint가 같은 시점에 발생했고, Safepoint 세부 시간에서 `Time to safepoint`가 대부분을 차지한다는 사실을 찾는 데 핵심적인 역할을 한다.

### async-profiler: Java와 Native 사이에서 어디에 머물렀는가

async-profiler는 Java stack뿐 아니라 native stack까지 함께 추적하는 데 유용하다.

예를 들면 다음과 같은 호출 경계를 볼 수 있다.

```text
Java Method
  ↓
JNI
  ↓
libssl.so
```

따라서 다음 질문에 적합하다.

> 어떤 Java/native 코드에서 시간을 보내고 있는가?

JFR에서 특정 thread 또는 Safepoint 진입 지연이 의심된다면, async-profiler를 이용해 Java 코드가 어떤 JNI/native 라이브러리로 진입하는지 추적할 수 있다.

### perf: Native 아래에서 OS와 Kernel이 무엇을 하고 있는가

Linux `perf`는 분석 범위를 native와 kernel 영역까지 더 깊게 내릴 수 있다.

대표적으로 다음을 확인할 수 있다.

- CPU sampling
- Syscall
- Context Switch
- Scheduler
- Cache Miss
- Page Fault
- Kernel Stack

따라서 다음 질문에 적합하다.

> Native 영역 아래에서 OS나 Kernel이 실제로 무엇을 하고 있는가?

async-profiler에서 특정 native library나 JNI 경로가 의심된다면, `perf`를 이용해 syscall, scheduler, page fault, kernel stack 등 더 낮은 레이어의 원인을 추적한다.

## 실무 분석 흐름

성능 이슈를 처음부터 kernel에서 분석하면 탐색 범위가 너무 넓다. 따라서 보통은 위에서 아래로 범위를 좁히는 방식이 효율적이다.

```text
JFR
↓
JVM 레벨 이상 징후 확인

async-profiler
↓
Java → Native stack 추적

perf
↓
Native → Kernel / Syscall / Scheduler 분석
```

이 흐름을 질문 중심으로 바꾸면 다음과 같다.

```text
1. JFR
   "JVM에서 언제 이상 현상이 발생했는가?"

2. async-profiler
   "문제가 발생한 thread는 어떤 Java/native 경로에 있었는가?"

3. perf
   "그 native 경로 아래에서 OS/kernel은 무엇을 하고 있었는가?"
```

## 예시

### 관찰된 현상

```text
환경: Java 21 + ZGC

p99 latency:
수십 ms → 간헐적으로 1~2초

GC pause:
약 1ms

DB / Redis / External API:
정상

CPU:
약 60%

BLOCKED thread:
뚜렷한 증가 없음
```

전형적인 GC pause, 데이터 저장소 병목, Java monitor lock contention 패턴과 맞지 않는다.

### JFR에서 발견한 단서

```text
Safepoint total      1.9s
Time to safepoint    1.896s
VM operation         16ms
```

해석은 다음과 같다.

```text
VM operation이 느렸다        → 아님
Safepoint 진입 자체가 느렸다 → 핵심 단서
```

따라서 다음 분석 대상은 "어떤 thread가 늦게 Safepoint에 도달했는가"가 된다.

### 의심 가능한 실행 경로

```text
Application Java Code
        ↓
       JNI
        ↓
Native Library (예: libssl.so 등)
        ↓
      Syscall
        ↓
      Kernel
```

Java thread dump에서는 해당 thread가 단순히 `RUNNABLE`로 보일 수 있으므로 Java 상태만 보고 CPU-bound 작업이라고 단정해서는 안 된다.

## 헷갈리기 쉬운 점

### 1. 긴 Safepoint를 보면 GC 문제라고 생각하기

Safepoint는 GC보다 더 넓은 JVM 메커니즘이다. 먼저 어떤 VM operation 때문인지, 그리고 실제 시간이 VM operation에 쓰였는지 Safepoint 진입 대기에 쓰였는지 분리해야 한다.

### 2. RUNNABLE이면 CPU를 많이 쓰고 있다고 생각하기

Java thread state의 `RUNNABLE`은 native/JNI 영역에 들어간 thread도 포함할 수 있다. CPU 사용량과 native stack을 함께 봐야 한다.

### 3. CPU 사용률이 낮으니 thread가 문제일 수 없다고 생각하기

Safepoint 지연은 CPU를 많이 소비하는 thread만 만들 수 있는 문제가 아니다. native syscall, kernel scheduling, page fault 등 아래 레이어에서 지연되는 thread 하나가 전체 Safepoint 진입을 늦출 수 있다.

### 4. BLOCKED thread가 없으니 synchronization 문제는 아니라고 끝내기

Java monitor lock contention이 없다는 사실은 원인 범위를 줄여줄 뿐이다. JNI/native/kernel 영역의 대기는 Java의 `BLOCKED` 상태로 나타나지 않을 수 있다.

### 5. 관찰된 레이어를 원인 레이어로 착각하기

JFR에서 Safepoint 문제가 보였다고 해서 JVM 구현 자체가 반드시 원인은 아니다.

실제 시간 소비 지점은 다음처럼 더 아래에 있을 수 있다.

```text
JVM
  ↓
JNI
  ↓
Native Library
  ↓
OS
  ↓
Kernel
```

## 새롭게 알게 된 내용

성능 분석에서 가장 중요한 질문 중 하나는 다음이다.

> 어디에서 느려 보이는가가 아니라, 실제로 어느 레이어에서 시간이 소비되고 있는가?

관찰 지표와 근본 원인은 서로 다른 레이어에 존재할 수 있다.

이 사례의 핵심 사고 과정은 다음과 같다.

```text
Latency spike
↓
GC / DB / CPU / Java lock에서 설명되지 않음
↓
JFR에서 긴 Safepoint 발견
↓
Safepoint total을 세부 구간으로 분해
↓
Time to safepoint가 대부분임을 확인
↓
특정 thread의 Safepoint 도달 지연 의심
↓
RUNNABLE 상태만으로 판단하지 않음
↓
JNI / Native 영역 추적
↓
필요하면 OS / Kernel까지 분석 범위 확장
```

즉 JVM 성능 문제를 분석할 때는 Java 레이어에만 머물지 말고, 증거에 따라 JNI, native library, syscall, scheduler, kernel까지 단계적으로 내려가는 관점이 필요하다.