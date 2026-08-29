# Gradle 병렬 테스트와 Worker/JVM 메모리 튜닝

## 핵심 요약

`./gradlew test --parallel`은 테스트 케이스 자체를 무조건 병렬 실행하는 옵션이 아니라, 멀티 프로젝트/멀티 모듈 환경에서 서로 의존하지 않는 Gradle task를 가능한 범위에서 병렬 실행하도록 하는 옵션이다.

병렬 실행 시 하나의 host에서 여러 Test JVM이 동시에 실행될 수 있으므로, 각 JVM에 큰 `-Xmx`가 설정되어 있으면 총 메모리 사용량이 크게 증가할 수 있다. 따라서 `--parallel`을 사용할 때는 Gradle worker 수와 테스트 JVM fork 수, 각 JVM의 heap 크기를 함께 조절해야 한다.

## 개념

### `./gradlew test --parallel`

- `./gradlew`: 프로젝트에 포함된 Gradle Wrapper 실행
- `test`: JVM 프로젝트의 테스트 task 실행
- `--parallel`: 서로 의존하지 않는 Gradle 프로젝트/task를 병렬 실행하도록 허용

예를 들어 멀티 모듈 프로젝트가 다음과 같다고 하자.

```text
root
├── api
├── domain
└── batch
```

각 모듈의 `test` task가 서로 독립적이라면 다음 명령에서 동시에 실행될 수 있다.

```bash
./gradlew test --parallel
```

개념적으로는 다음과 같다.

```text
:api:test       ───────▶
:domain:test    ───────▶
:batch:test     ───────▶
```

반대로 task 간 의존성이 있다면 해당 의존 관계는 지켜진다.

```text
A → B → C
```

이런 경우 A, B, C를 동시에 실행할 수는 없다.

## 동작 원리

### `--parallel`과 테스트 JVM

`--parallel`은 주로 서로 다른 Gradle project/task의 병렬 실행을 제어한다.

반면 하나의 `test` task 내부에서 여러 테스트 JVM을 띄우는 것은 `maxParallelForks`와 관련이 있다.

예:

```groovy
test {
    maxParallelForks = 4
    maxHeapSize = "2g"
}
```

이 설정에서는 하나의 `test` task가 최대 4개의 테스트 JVM을 동시에 사용할 수 있다.

멀티 모듈에서 `--parallel`까지 함께 사용하면 여러 모듈의 `test` task가 동시에 실행되고, 각 task가 다시 여러 JVM을 fork할 수 있다.

예를 들어 동시에 3개 모듈이 테스트되고 각 모듈이 최대 4개의 fork, 각 JVM이 `-Xmx2g`라면 이론적인 heap 상한은 다음과 같다.

```text
3 modules × 4 forks × 2 GB
= 최대 24 GB heap
```

실제로 모든 JVM이 항상 `Xmx`까지 사용하는 것은 아니지만, host 메모리 용량을 산정할 때는 이런 최악의 동시성도 고려해야 한다.

### 실제 host 메모리

전체 메모리는 테스트 JVM heap만으로 결정되지 않는다.

```text
전체 메모리 사용량
≈ Gradle Daemon JVM
 + (동시에 실행 중인 Test JVM 수 × Test JVM 메모리)
 + Metaspace
 + Thread Stack
 + Direct/Native Memory
 + OS 및 기타 프로세스 메모리
```

따라서 물리 메모리와 거의 같은 수준까지 JVM heap 총합을 잡으면 OOM Kill, swap, GC 증가 등으로 오히려 테스트 성능이 나빠질 수 있다.

## Worker 수 제한

`--parallel`을 유지하면서 Gradle이 사용하는 worker 수를 제한할 수 있다.

명령행에서 지정:

```bash
./gradlew test --parallel --max-workers=2
```

또는 `gradle.properties`에서 지정:

```properties
org.gradle.workers.max=2
```

이렇게 하면 한 host에서 과도한 병렬 작업이 발생하는 것을 제어할 수 있다.

## 헷갈리기 쉬운 점

### `--parallel` != 테스트 메서드 병렬 실행

`--parallel`은 Gradle task/project 수준의 병렬성에 가깝다.

테스트 클래스나 테스트 메서드 자체를 병렬로 실행하려면 Gradle의 Test 설정이나 JUnit parallel execution 설정을 별도로 확인해야 한다.

### `--max-workers`와 `maxParallelForks`는 다른 설정

- `--max-workers` / `org.gradle.workers.max`: Gradle 전체 worker 동시성 제한
- `maxParallelForks`: 하나의 `Test` task 내부에서 동시에 생성할 테스트 JVM 수
- `maxHeapSize` 또는 JVM `-Xmx`: 각 테스트 JVM의 최대 heap 크기

따라서 메모리 제한이 중요한 CI나 단일 host 환경에서는 이 세 요소를 같이 본다.

```text
Gradle worker 수
× Test JVM fork 수
× JVM당 메모리
```

정확히 단순 곱으로만 동작하는 것은 아니지만, 최대 메모리 압력을 판단하는 유용한 기준이다.

## 예시

메모리가 제한된 CI host라면 다음과 같이 시작할 수 있다.

```bash
./gradlew test --parallel --max-workers=2
```

그리고 테스트 설정도 필요에 따라 제한한다.

```groovy
test {
    maxParallelForks = 2
    maxHeapSize = "2g"
}
```

핵심은 병렬성을 무조건 크게 만드는 것이 아니라, host의 CPU와 RAM 범위 안에서 병렬성을 조절하는 것이다.

## 새롭게 알게 된 내용

- `--parallel` 사용 시 멀티 모듈의 독립적인 `test` task가 동시에 실행될 수 있다.
- 각각의 Test JVM이 별도의 heap을 사용하므로 병렬도가 커지면 host 메모리 사용량도 증가한다.
- 메모리 압박을 제어하려면 `--max-workers` 또는 `org.gradle.workers.max`로 Gradle worker 수를 제한할 수 있다.
- `maxParallelForks`가 별도로 설정되어 있다면 worker 수만 줄이는 것으로 충분하지 않을 수 있으므로 함께 확인해야 한다.
