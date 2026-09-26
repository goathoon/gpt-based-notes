# Read-only 트랜잭션과 여러 SELECT를 묶는 이유

## 핵심 요약

`@Transactional(readOnly = true)`는 단순히 `SELECT` 성능을 높이는 마법의 옵션이 아니다.

핵심 목적은 다음과 같다.

- 이 작업이 **조회 전용**이라는 의도를 명확하게 표현한다.
- Spring/Hibernate/JDBC/DB가 상황에 따라 조회 전용 힌트를 활용할 수 있다.
- ORM이 변경 감지 등 일부 불필요한 작업을 줄일 수 있다.
- 여러 `SELECT`를 하나의 논리적인 작업으로 묶어 **트랜잭션 문맥과 일관성 규칙**을 공유할 수 있다.

특히 여러 `SELECT`를 하나의 트랜잭션으로 묶는 가장 중요한 이유는, 여러 조회 결과가 합쳐져 하나의 의미 있는 결과를 만들 때 **서로 일관된 데이터로 다루기 위해서**다.

다만 트랜잭션으로 묶었다고 해서 모든 `SELECT`가 항상 완전히 같은 시점의 데이터를 보는 것은 아니다. 실제 보장 수준은 DB의 **Transaction Isolation Level**에 따라 달라진다.

## 개념

### readOnly 트랜잭션

Spring에서는 조회 전용 서비스 메서드를 다음과 같이 표현할 수 있다.

```java
@Transactional(readOnly = true)
public Member findMember(Long id) {
    return memberRepository.findById(id)
        .orElseThrow();
}
```

`readOnly = true`는 "이 트랜잭션에서는 데이터를 변경하지 않을 것이다"라는 의도를 프레임워크와 DB 계층에 알려주는 역할을 한다.

반대로 수정이 필요한 로직은 일반 트랜잭션을 사용한다.

```java
@Transactional
public void changeName(Long id, String name) {
    Member member = memberRepository.findById(id)
        .orElseThrow();

    member.changeName(name);
}
```

## 동작 원리

### Hibernate의 변경 감지와 readOnly

일반적인 JPA 트랜잭션에서는 Hibernate가 영속 상태 엔티티의 변경 여부를 확인한다.

```text
DB에서 Member 조회
        ↓
영속성 컨텍스트에 엔티티 저장
        ↓
기존 상태를 추적
        ↓
트랜잭션 종료 / flush
        ↓
현재 상태와 비교
        ↓
변경되었다면 UPDATE
```

이 과정을 **dirty checking(변경 감지)** 이라고 한다.

조회 전용 트랜잭션에서는 "이 작업은 수정하지 않는다"는 힌트를 줄 수 있기 때문에 Hibernate가 변경 감지와 관련된 일부 불필요한 작업을 줄일 수 있다.

따라서 `readOnly = true`의 의미는 단순한 SQL 최적화보다 다음에 가깝다.

> 조회 전용이라는 의도를 명시하고, ORM과 데이터 접근 계층이 불필요한 작업을 피할 수 있도록 힌트를 제공한다.

### 여러 SELECT를 하나의 트랜잭션으로 묶는 이유

예를 들어 주문 화면을 만들기 위해 여러 데이터를 조회한다고 하자.

```java
@Transactional(readOnly = true)
public OrderSummary getOrderSummary(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    List<OrderItem> items = orderItemRepository.findAllByOrderId(orderId);
    Payment payment = paymentRepository.findByOrderId(orderId);

    return new OrderSummary(order, items, payment);
}
```

실제로는 다음과 같은 여러 조회가 발생한다.

```text
1. 주문 조회
2. 주문 상품 조회
3. 결제 정보 조회
```

이 조회들이 하나의 결과를 만드는 과정에서 다른 트랜잭션이 데이터를 변경할 수 있다.

```text
내 트랜잭션                        다른 트랜잭션

SELECT 주문
  상태 = PAID

                              UPDATE 주문 상태 = CANCELLED
                              DELETE 결제 정보
                              COMMIT

SELECT 주문상품
SELECT 결제정보
```

격리 수준에 따라 이후 `SELECT`가 변경된 값을 볼 수도 있다.

그 결과 애플리케이션 입장에서는 다음처럼 서로 다른 시점의 데이터를 조합할 가능성이 생긴다.

```text
주문 상태: PAID
결제 정보: 없음
```

여러 조회를 트랜잭션으로 묶으면 모든 조회가 같은 트랜잭션 문맥 안에서 실행되고, DB의 isolation level에 따른 일관성 규칙을 적용받는다.

```text
@Transactional(readOnly = true)
            │
            ├─ 여러 SELECT를 하나의 논리적 작업으로 묶음
            ├─ 같은 트랜잭션 문맥 사용
            ├─ Isolation Level에 따른 일관성 제공
            └─ JPA에서는 같은 Persistence Context 사용
```

## 예시

### 조회 중심 서비스의 일반적인 패턴

클래스 전체를 조회 전용으로 선언하고, 수정이 필요한 메서드만 별도의 일반 트랜잭션으로 덮어쓸 수 있다.

```java
@Service
@Transactional(readOnly = true)
public class MemberService {

    public Member findMember(Long id) {
        return memberRepository.findById(id).orElseThrow();
    }

    @Transactional
    public void updateMember(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.changeName(name);
    }
}
```

이 방식은 서비스의 기본 성격이 조회 전용임을 명확하게 표현한다.

### JPA 영속성 컨텍스트 공유

같은 트랜잭션에서는 같은 영속성 컨텍스트를 사용하기 때문에 동일한 엔티티를 다시 조회할 때 1차 캐시를 활용할 수 있다.

```java
Member member1 = memberRepository.findById(1L).orElseThrow();
Member member2 = memberRepository.findById(1L).orElseThrow();

System.out.println(member1 == member2);
```

일반적인 JPA 환경에서는 같은 영속성 컨텍스트 안에서 동일 식별자의 엔티티는 동일한 객체로 관리된다.

## 헷갈리기 쉬운 점

### readOnly=true면 UPDATE가 무조건 차단되는가?

그렇다고 단정하면 안 된다.

`readOnly = true`의 실제 동작은 다음 요소에 따라 달라질 수 있다.

- Spring
- JPA Provider / Hibernate
- JDBC Driver
- DBMS

따라서 `readOnly = true`를 보안 장치처럼 믿기보다 **의도 표현 + 최적화 힌트**로 이해하는 것이 안전하다.

### readOnly=true면 자동으로 Read Replica로 가는가?

아니다.

`readOnly = true`만 붙였다고 조회가 자동으로 replica DB로 라우팅되지는 않는다. 이를 위해서는 별도의 DataSource routing이나 DB 인프라 구성이 필요하다.

### 트랜잭션으로 묶으면 모든 SELECT가 같은 스냅샷을 보는가?

항상 그렇지는 않다.

실제 동작은 **Transaction Isolation Level**에 따라 달라진다.

예를 들어 `READ COMMITTED` 환경에서는 첫 번째 `SELECT` 이후 다른 트랜잭션이 커밋한 변경사항을 다음 `SELECT`에서 볼 수 있는 경우가 있다.

반면 MVCC와 `REPEATABLE READ`를 사용하는 환경에서는 같은 트랜잭션에서 더 일관된 스냅샷을 제공할 수 있다.

따라서 다음 둘을 구분해야 한다.

```text
트랜잭션으로 묶는다
    ≠
무조건 모든 SELECT가 완전히 동일한 시점의 데이터를 본다
```

정확한 보장은 isolation level과 DBMS 구현까지 봐야 한다.

### 여러 SELECT는 무조건 하나의 트랜잭션으로 묶어야 하는가?

아니다.

서로 완전히 독립적인 조회라면 하나의 큰 트랜잭션으로 묶을 필요가 적다.

예를 들어 다음 결과들이 서로 관계가 없다면 하나의 트랜잭션으로 만들 실익이 크지 않을 수 있다.

```java
getRecentPosts();
getRecommendedProducts();
getWeather();
```

반대로 여러 `SELECT`가 합쳐져 하나의 도메인 결과를 만든다면 트랜잭션으로 묶을 가치가 커진다.

## 새롭게 알게 된 내용

여러 `SELECT`를 묶는 이유를 단순히 "트랜잭션이니까"라고 이해하기보다 다음 기준으로 판단하면 좋다.

> 여러 조회 결과가 합쳐져 **하나의 의미 있는 결과**를 만드는가?

그렇다면 하나의 read-only 트랜잭션 안에서 처리하는 것이 트랜잭션 문맥, 영속성 컨텍스트, isolation level에 따른 데이터 일관성을 이해하고 관리하기에 유리하다.

반대로 서로 관련 없는 조회라면 억지로 하나의 트랜잭션으로 묶을 필요는 없다.
