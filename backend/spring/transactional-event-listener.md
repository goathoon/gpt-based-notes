# Spring ApplicationEvent와 TransactionalEventListener

## 핵심 요약

Spring의 `ApplicationEvent`와 `@EventListener`를 사용하는 핵심 목적은 단순히 비동기 처리를 하기 위해서가 아니라, **이벤트를 발생시키는 주체와 이벤트에 반응하는 주체의 결합도를 낮추기 위해서**다.

특히 `@TransactionalEventListener`를 사용하면 이벤트 리스너의 실행 시점을 현재 트랜잭션의 라이프사이클에 연결할 수 있다.

핵심적으로 다음과 같이 이해할 수 있다.

- `@EventListener` → 이벤트가 발생하면 기본적으로 즉시 실행
- `@TransactionalEventListener` → 이벤트 발생 시 바로 실행하지 않고 현재 트랜잭션의 특정 시점까지 실행을 미룸
- `@Transactional`의 `propagation` → 이미 트랜잭션이 존재할 때 새로 호출되는 메서드가 기존 트랜잭션과 어떤 관계를 가질지 결정

---

## ApplicationEvent를 사용하는 이유

예를 들어 회원가입 시 다음 작업들이 필요하다고 가정한다.

```java
public void signup(User user) {
    userRepository.save(user);
    emailService.sendWelcomeEmail(user);
    couponService.issueCoupon(user);
    analyticsService.trackSignup(user);
}
```

이 구조에서는 회원가입 서비스가 다음 서비스들을 모두 알아야 한다.

```text
UserService
 ├─ EmailService
 ├─ CouponService
 └─ AnalyticsService
```

기능이 추가될수록 `UserService`의 의존성이 계속 증가한다.

이를 이벤트 기반으로 변경하면 다음과 같다.

```java
public void signup(User user) {
    userRepository.save(user);
    applicationEventPublisher.publishEvent(
        new UserSignedUpEvent(user.getId())
    );
}
```

각 기능은 이벤트를 독립적으로 구독한다.

```java
@EventListener
public void sendWelcomeEmail(UserSignedUpEvent event) {
    // 이메일 발송
}

@EventListener
public void issueCoupon(UserSignedUpEvent event) {
    // 쿠폰 발급
}
```

구조는 다음과 같이 바뀐다.

```text
UserService
     │
     ▼
UserSignedUpEvent
     │
 ┌───┼────────────┐
 ▼   ▼            ▼
Email Coupon Analytics
```

이 구조에서는 publisher가 subscriber를 알 필요가 없다.

새로운 기능이 추가되어도 기존 회원가입 로직을 수정하지 않고 listener를 추가할 수 있으므로 컴포넌트 간 결합도를 낮출 수 있다.

---

## Command와 Event의 차이

다음 코드는 다른 객체에게 특정 행동을 요구한다.

```java
emailService.sendEmail();
```

즉, 의미상 **Command**에 가깝다.

> 이메일을 보내라.

반면 다음 코드는 이미 발생한 사실을 알린다.

```java
applicationEventPublisher.publishEvent(
    new UserSignedUpEvent(userId)
);
```

즉, **Event**다.

> 회원가입이 발생했다.

Publisher는 이후 어떤 작업이 수행될지 몰라도 된다.

```text
Publisher
"회원가입이 완료됐다"

Listener A
→ 이메일 발송

Listener B
→ 쿠폰 발급

Listener C
→ 통계 기록
```

이 때문에 이벤트 이름은 보통 행동보다 **발생한 사실을 과거형으로 표현**한다.

예:

- `UserSignedUp`
- `OrderCompleted`
- `PaymentCompleted`
- `OrderCancelled`

---

## @EventListener의 기본 동작

Spring의 `ApplicationEvent`를 사용한다고 해서 Kafka나 RabbitMQ처럼 자동으로 비동기 메시징이 되는 것은 아니다.

기본적인 `@EventListener`는 **동기적으로 실행**된다.

```java
@EventListener
public void handle(UserSignedUpEvent event) {
}
```

동작 흐름은 대략 다음과 같다.

```text
Thread 1
signup()
   ↓
publishEvent()
   ↓
Listener A 실행
   ↓
Listener B 실행
   ↓
publishEvent() 종료
   ↓
signup() 계속 실행
```

따라서 Spring Event의 핵심 목적을 단순히 "비동기 처리"라고 이해하면 안 된다.

핵심은 **컴포넌트 간 결합도를 낮추는 것**이다.

---

## 트랜잭션과 일반 @EventListener의 문제

다음 코드가 있다고 가정한다.

```java
@Transactional
public void signup(User user) {
    userRepository.save(user);
    applicationEventPublisher.publishEvent(
        new UserSignedUpEvent(user.getId())
    );
}
```

일반 `@EventListener`를 사용하면 listener가 transaction commit 이전에 실행될 수 있다.

```text
회원 저장
   ↓
이벤트 발생
   ↓
이메일 발송
   ↓
DB 오류
   ↓
ROLLBACK
```

그 결과 다음과 같은 데이터 정합성 문제가 생길 수 있다.

```text
DB
→ 회원 없음

Email
→ "회원가입을 축하합니다"
```

---

## @TransactionalEventListener

이런 문제를 해결하기 위해 `@TransactionalEventListener`를 사용할 수 있다.

기본 phase는 `AFTER_COMMIT`이다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

즉, 트랜잭션 commit이 성공한 이후 listener를 실행한다.

```text
회원 저장
   ↓
이벤트 발행
   ↓
DB COMMIT
   ↓
Listener 실행
```

중요한 점은 **이벤트 자체를 commit 이후에 발행하는 것이 아니라는 것**이다.

```java
applicationEventPublisher.publishEvent(event);
```

이 코드는 실제 코드 위치에서 바로 실행된다.

대신 listener 호출만 트랜잭션 이후로 미뤄진다.

---

## 동작 원리

`@TransactionalEventListener`는 개념적으로 다음과 같이 동작한다고 이해할 수 있다.

```text
publishEvent()
     ↓
TransactionalEventListener 발견
     ↓
현재 Transaction 존재 여부 확인
     ↓
TransactionSynchronization 등록
     ↓
Listener는 아직 실행하지 않음
```

대략적인 의사 코드는 다음과 같다.

```java
public void onApplicationEvent(Event event) {
    if (현재 transaction이 존재함) {
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    listener(event);
                }
            }
        );
    }
}
```

실제 Spring 구현은 더 복잡하지만 개념적으로는 transaction lifecycle에 callback을 등록하는 방식으로 이해할 수 있다.

Spring은 transaction lifecycle에 callback을 등록할 수 있도록 `TransactionSynchronizationManager`를 사용한다.

Transaction 종료 과정은 개념적으로 다음과 같은 callback 흐름을 가진다.

```text
beforeCommit()
     ↓
beforeCompletion()
     ↓
실제 DB COMMIT
     ↓
afterCommit()
     ↓
afterCompletion()
```

`@TransactionalEventListener`는 지정된 `TransactionPhase`에 따라 적절한 시점에 listener를 실행한다.

---

## TransactionPhase

### BEFORE_COMMIT

```java
@TransactionalEventListener(
    phase = TransactionPhase.BEFORE_COMMIT
)
```

```text
비즈니스 로직
   ↓
이벤트 발행
   ↓
BEFORE_COMMIT listener
   ↓
DB COMMIT
```

### AFTER_COMMIT

가장 일반적으로 사용되는 phase이며 `@TransactionalEventListener`의 기본값이다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

```text
비즈니스 로직
   ↓
이벤트 발행
   ↓
DB COMMIT 성공
   ↓
listener 실행
```

### AFTER_ROLLBACK

Transaction이 rollback된 경우에만 실행된다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_ROLLBACK
)
```

```text
비즈니스 로직
   ↓
이벤트 발행
   ↓
Exception
   ↓
ROLLBACK
   ↓
listener 실행
```

### AFTER_COMPLETION

Commit 또는 rollback 여부에 관계없이 transaction이 끝나면 실행된다.

```text
          ┌─ COMMIT
Transaction
          └─ ROLLBACK
               ↓
        AFTER_COMPLETION
```

---

## @EventListener와 @TransactionalEventListener 차이

### @EventListener

```text
publishEvent()
    ↓
listener 즉시 실행
    ↓
publishEvent() 반환
    ↓
나중에 Transaction commit
```

### @TransactionalEventListener

```text
publishEvent()
    ↓
listener 실행 예약
    ↓
publishEvent() 반환
    ↓
Transaction commit
    ↓
listener 실행
```

따라서 `@TransactionalEventListener`는 **이벤트 listener의 실행 시점을 현재 transaction lifecycle과 결합시키는 기능**이라고 볼 수 있다.

---

## Transaction Propagation

`propagation`은 `@Transactional`의 옵션으로, 다음을 결정한다.

> 이미 Transaction이 존재할 때 새로 호출되는 메서드가 기존 Transaction과 어떻게 관계를 맺을 것인가?

대표적으로 `REQUIRED`와 `REQUIRES_NEW`가 중요하다.

### Propagation.REQUIRED

Spring Transaction의 기본값이다.

```java
@Transactional
```

는 사실상 다음과 같다.

```java
@Transactional(
    propagation = Propagation.REQUIRED
)
```

동작 방식:

```text
기존 Transaction 있음
→ 기존 Transaction에 참여

기존 Transaction 없음
→ 새로운 Transaction 생성
```

예:

```java
@Transactional
public void order() {
    paymentService.pay();
}

@Transactional
public void pay() {
}
```

```text
Transaction A 시작
   ↓
order()
   ↓
pay()
   ↓
Transaction A 그대로 사용
   ↓
COMMIT
```

둘은 같은 transaction에 포함된다.

따라서 하나에서 예외가 발생하면 전체가 rollback될 수 있다.

```text
Transaction A
 ├─ 주문 저장
 ├─ 결제 저장
 └─ Exception
       ↓
    전체 ROLLBACK
```

### Propagation.REQUIRES_NEW

`REQUIRES_NEW`는 항상 새로운 transaction을 만든다.

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
public void saveLog() {
}
```

기존 transaction이 있다면 다음처럼 동작한다.

```text
Transaction A
     ↓
suspend
Transaction B 시작
     ↓
작업 수행
     ↓
COMMIT
Transaction A
     ↓
resume
```

따라서 새로운 transaction은 기존 transaction과 독립적으로 commit 또는 rollback될 수 있다.

예:

```java
@Transactional
public void order() {
    orderRepository.save(order);
    logService.saveLog();
    throw new RuntimeException();
}
```

`saveLog()`가 `REQUIRED`라면:

```text
Transaction A
 ├─ 주문 저장
 ├─ 로그 저장
 └─ Exception
       ↓
전체 ROLLBACK
```

하지만 다음과 같이 `REQUIRES_NEW`를 사용하면:

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
public void saveLog() {
    logRepository.save(...);
}
```

```text
Transaction A
 ├─ 주문 저장
 │
 ├─ Transaction B
 │    └─ 로그 저장
 │         ↓
 │       COMMIT
 │
 └─ Exception
      ↓
Transaction A ROLLBACK
```

결과:

```text
주문
→ ROLLBACK

로그
→ COMMIT
```

---

## AFTER_COMMIT과 REQUIRES_NEW

`@TransactionalEventListener(AFTER_COMMIT)`에서 새로운 DB 작업을 하는 경우 `REQUIRES_NEW`가 자주 등장한다.

```java
@TransactionalEventListener
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
public void handle(UserSignedUpEvent event) {
    repository.save(...);
}
```

동작 구조는 다음과 같다.

```text
Transaction A
────────────────────
회원 저장
이벤트 발행
COMMIT
────────────────────
       ↓
AFTER_COMMIT listener
       ↓
Transaction B 시작
────────────────────
추가 DB 작업
COMMIT
────────────────────
```

원래 transaction은 이미 commit이 끝났기 때문에 listener에서 수행하는 DB 작업을 명확하게 별도의 transaction으로 처리하고 싶다면 `REQUIRES_NEW`를 사용할 수 있다.

다만 이메일 발송처럼 DB transaction이 필요 없는 작업에서는 새로운 transaction이 필요하지 않다.

---

## Propagation 종류

| Propagation | 의미 |
|---|---|
| `REQUIRED` | 기존 Transaction이 있으면 참여하고 없으면 생성 |
| `REQUIRES_NEW` | 항상 새로운 Transaction 생성 |
| `SUPPORTS` | Transaction이 있으면 참여하고 없으면 없이 실행 |
| `MANDATORY` | 기존 Transaction이 반드시 있어야 함 |
| `NOT_SUPPORTED` | Transaction 없이 실행 |
| `NEVER` | Transaction이 존재하면 예외 |
| `NESTED` | Savepoint를 이용한 중첩 Transaction |

실무에서는 우선 다음 두 개를 확실하게 이해하는 것이 중요하다.

- `REQUIRED` → 기존 Transaction과 같이 성공하고 같이 실패한다.
- `REQUIRES_NEW` → 별도의 Transaction으로 독립적으로 동작한다.

---

## 헷갈리기 쉬운 점

### ApplicationEvent는 기본적으로 비동기가 아니다

`ApplicationEvent = Kafka`가 아니다.

일반 `@EventListener`는 기본적으로 같은 thread에서 동기적으로 실행된다.

비동기로 실행하려면 별도의 설정과 `@Async` 등을 사용할 수 있지만, 이것 역시 Kafka나 RabbitMQ 같은 외부 message broker와 동일한 신뢰성을 제공하는 것은 아니다.

### @TransactionalEventListener는 이벤트 발행을 늦추는 것이 아니다

잘못된 이해:

```text
Transaction commit
   ↓
publishEvent()
```

실제 동작:

```text
publishEvent()
   ↓
listener callback 등록
   ↓
Transaction commit
   ↓
listener 실행
```

### AFTER_COMMIT에서 DB 작업 시 주의

`AFTER_COMMIT`은 기존 transaction이 이미 commit된 이후다.

따라서 listener에서 추가적인 DB 변경을 독립적으로 보장하고 싶다면 새로운 transaction을 사용하는 것을 고려해야 한다.

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
```

### REQUIRES_NEW를 남용하면 안 된다

`REQUIRES_NEW`를 사용하면 기존 transaction을 suspend하고 새로운 transaction을 시작한다.

구현 환경에 따라 별도의 DB connection이 필요할 수 있기 때문에 과도하게 사용하면 connection pool 사용량 증가 등의 문제가 발생할 수 있다.

따라서 단순히 "안전해 보인다"는 이유로 사용하기보다 transaction을 분리해야 하는 명확한 이유가 있을 때 사용하는 것이 좋다.

---

## 전체 흐름

Spring Event와 Transaction을 연결해서 보면 다음 흐름으로 이해할 수 있다.

```text
@Transactional
signup()
   │
   ├─ 회원 DB 저장
   │
   ├─ publishEvent()
   │      │
   │      └─ TransactionSynchronization 등록
   │
   └─ 메서드 종료
          │
          ▼
     Transaction COMMIT
          │
          ▼
@TransactionalEventListener(AFTER_COMMIT)
          │
          ├─ 이메일 발송
          │
          └─ 필요하면
              REQUIRES_NEW Transaction 생성
```

---

## 기억할 핵심

- `ApplicationEvent` → 컴포넌트 간 결합도를 낮추기 위한 이벤트 전달 방식
- `@EventListener` → 이벤트 발생 시 기본적으로 즉시 실행
- `@TransactionalEventListener` → listener 실행 시점을 Transaction lifecycle에 연결
- `AFTER_COMMIT` → Transaction commit 성공 이후 listener 실행
- `Propagation.REQUIRED` → 기존 Transaction이 있으면 같이 사용
- `Propagation.REQUIRES_NEW` → 기존 Transaction과 분리된 새로운 Transaction 생성

특히 가장 중요한 개념은 다음 두 문장으로 요약할 수 있다.

> `@TransactionalEventListener`는 이벤트를 늦게 발행하는 것이 아니라, 이벤트가 발생했을 때 listener 실행을 transaction lifecycle에 등록해 두었다가 지정된 시점에 실행한다.

> Propagation은 Transaction이 이미 존재하는 상황에서 새 메서드가 그 Transaction에 참여할지, 새로운 Transaction을 만들지 등을 결정하는 정책이다.
