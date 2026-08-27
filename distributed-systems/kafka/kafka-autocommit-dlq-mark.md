# Kafka Auto Commit, DLQ, Mark

## 핵심 요약

Kafka에서 **auto commit**, **DLQ**, **mark**는 모두 결국 **메시지를 어디까지 처리했다고 기록할 것인지**와 관련된 개념이다.

- **Auto Commit**: Consumer가 주기적으로 offset을 자동 commit한다.
- **Mark / Ack**: 애플리케이션 또는 프레임워크가 특정 레코드를 처리 완료로 표시한다.
- **DLQ (Dead Letter Queue)**: 반복적으로 처리에 실패하는 메시지를 별도 topic으로 격리해 정상 흐름이 계속 진행되도록 한다.

중요한 메시지를 다루는 경우에는 보통 `enable.auto.commit=false`로 두고, 처리 성공 여부에 맞춰 직접 offset을 관리하는 편이 안전하다.

## 개념

### Auto Commit

Kafka Consumer에서 다음과 같이 설정하면 consumer가 주기적으로 현재 offset을 자동 commit한다.

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

문제는 **비즈니스 로직이 실제로 성공했는지와 무관하게 offset이 commit될 수 있다는 점**이다.

예를 들어:

```text
offset 10 poll
   ↓
비즈니스 로직 처리 중
   ↓
auto commit 발생 → offset 11 commit
   ↓
처리 실패 / consumer 종료
```

재시작 시 Kafka는 이미 `11`부터 읽게 되므로, `10`번 메시지는 실제 처리는 실패했지만 다시 읽히지 않을 수 있다.

따라서 중요한 처리에서는 보통:

```properties
enable.auto.commit=false
```

로 두고 직접 offset을 관리한다.

### Mark / Ack

`mark`는 Kafka 표준 Consumer API의 공식 용어라기보다는, 사용하는 프레임워크나 라이브러리에서 **이 레코드를 처리 완료로 표시한다**는 의미로 자주 사용된다.

개념적으로는 다음과 같다.

```text
poll
 ↓
message 처리
 ↓
성공
 ↓
mark / acknowledge
 ↓
offset commit
```

예를 들어 다음 레코드를 모두 처리했다면:

```text
offset 10 성공
offset 11 성공
offset 12 성공
```

committed offset은 일반적으로 다음에 읽을 offset인 `13`이 된다.

```text
처리 완료: 10, 11, 12
committed offset = 13
```

즉 Kafka의 committed offset은 보통 **마지막으로 처리한 offset이 아니라 다음에 읽을 offset**을 의미한다.

### DLQ

DLQ(Dead Letter Queue)는 처리를 계속 실패하는 메시지를 별도 Kafka topic으로 보내는 패턴이다.

예:

```text
orders topic

offset 100 → 성공
offset 101 → 성공
offset 102 → 계속 실패
offset 103 → 성공
```

`102`를 계속 retry만 하면 consumer가 해당 메시지에서 계속 막힐 수 있다.

그래서 일정 횟수 재시도한 후:

```text
orders
  ↓
offset 102 처리 실패
  ↓
retry 3회
  ↓
DLQ topic으로 전송
  ↓
offset 102 처리 완료(mark)
  ↓
다음 메시지 진행
```

처럼 처리할 수 있다.

## 동작 원리

일반적인 흐름은 다음과 같다.

```text
Kafka Topic
    │
    ▼
Consumer
    │
    ├── 성공 ─────────→ mark/commit
    │
    └── 실패
          │
          ▼
        Retry
          │
          ├── 성공 → mark/commit
          │
          └── 최종 실패
                  │
                  ▼
                 DLQ
                  │
                  ▼
              mark/commit
```

중요한 점은 **DLQ로 보냈다고 바로 mark하면 안 될 수 있다는 것**이다.

안전한 순서는 보통 다음과 같다.

```text
1. 원본 메시지 처리 실패
2. DLQ publish
3. DLQ publish 성공 확인
4. 원본 메시지 mark
5. offset commit
```

반대로 다음 순서는 메시지 유실 위험이 있다.

```text
mark
 ↓
commit
 ↓
DLQ publish
 ↓
consumer 종료
```

이 경우 원본 offset은 이미 앞으로 진행됐지만 DLQ publish가 실패해, 원본에도 없고 DLQ에도 없는 상태가 될 수 있다.

## 예시

안정성을 중요하게 보는 경우 흔히 다음 패턴을 사용한다.

```text
enable.auto.commit = false

poll
 ↓
processing
 ↓
success ───────────────→ mark
 ↓
commit

failure
 ↓
retry
 ↓
still failure
 ↓
DLQ publish 성공
 ↓
mark
 ↓
commit
```

## 헷갈리기 쉬운 점

- `auto commit`은 비즈니스 처리 성공과 commit 시점이 정확히 일치하지 않을 수 있다.
- `mark`와 실제 broker에 반영되는 `commit`은 프레임워크에 따라 분리되어 있을 수 있다.
- DLQ에 publish하기 전에 offset을 commit하면 메시지 유실이 생길 수 있다.
- committed offset은 보통 마지막 처리 offset이 아니라 **다음에 읽을 offset**이다.

## 새롭게 알게 된 내용

- DLQ 처리에서 핵심 순서는 **DLQ publish 성공 → mark/ack → commit**이다.
- Auto Commit을 사용하는 경우 처리 실패와 offset commit 사이에 불일치가 생길 수 있으므로 DLQ/retry가 중요한 시스템에서는 수동 commit이 더 제어하기 쉽다.
- Auto Commit, Mark, DLQ는 별도 개념처럼 보이지만 실제로는 모두 **offset 관리와 메시지 처리 보장**이라는 하나의 문제로 연결된다.
