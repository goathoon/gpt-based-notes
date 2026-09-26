# Read-only 트랜잭션, 여러 SELECT, OSIV 이해하기

## 핵심 요약

`@Transactional(readOnly = true)`는 단순히 `SELECT` 성능을 높이는 옵션이 아니다.

핵심 목적은 다음과 같다.

- 이 작업이 **조회 전용**이라는 의도를 명확하게 표현한다.
- Spring/Hibernate/JDBC/DB가 상황에 따라 조회 전용 힌트를 활용할 수 있다.
- ORM이 변경 감지 등 일부 불필요한 작업을 줄일 수 있다.
- 여러 `SELECT`를 하나의 논리적인 작업으로 묶어 **트랜잭션 문맥과 일관성 규칙**을 공유할 수 있다.

그리고 Spring JPA 환경에서 함께 자주 등장하는 개념이 **OSIV(Open Session In View)** 다.

OSIV는 웹 요청이 끝날 때까지 `EntityManager`/Hibernate `Session`, 즉 영속성 컨텍스트를 열어두는 방식이다.

중요한 점은 다음이다.

> **OSIV가 켜져 있다고 해서 트랜잭션이 요청 끝까지 유지되는 것은 아니다.**

트랜잭션은 서비스 계층에서 끝날 수 있지만, 영속성 컨텍스트는 HTTP 요청이 끝날 때까지 살아 있을 수 있다.

---

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

### OSIV란?

OSIV는 **Open Session In View**의 줄임말이다.

Spring + JPA 관점에서는 다음처럼 이해하면 된다.

> 웹 요청이 시작될 때 영속성 컨텍스트를 열고, HTTP 응답이 끝날 때 닫는다.

즉, 서비스 계층의 `@Transactional` 메서드가 종료된 뒤에도 영속성 컨텍스트가 계속 살아 있을 수 있다.

```text
HTTP 요청 시작
    ↓
EntityManager / Session 생성
    ↓
Controller
    ↓
Service
    ↓
@Transactional 시작
    ↓
DB 조회
    ↓
@Transactional 종료
    ↓
영속성 컨텍스트는 아직 살아 있음
    ↓
Controller / View / 직렬화 과정에서 Lazy Loading 가능
    ↓
HTTP 응답 종료
    ↓
EntityManager / Session 종료
```

---

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

그 결과 애플리케이션 입장에서는 서로 다른 시점의 데이터를 조합할 가능성이 생긴다.

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

### OSIV와 Lazy Loading

예를 들어 `Order`가 `orderItems`를 지연 로딩으로 가지고 있다고 하자.

```java
@Entity
public class Order {

    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> orderItems;
}
```

서비스에서는 `Order`만 조회한다.

```java
@Transactional(readOnly = true)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

그리고 컨트롤러나 응답 직렬화 과정에서 다음 코드가 실행될 수 있다.

```java
order.getOrderItems().size();
```

#### OSIV가 OFF라면

서비스의 트랜잭션과 함께 영속성 컨텍스트 사용 범위도 사실상 끝났다면, 아직 초기화되지 않은 LAZY 연관관계에 접근할 때 다음 문제가 발생할 수 있다.

```text
LazyInitializationException
```

#### OSIV가 ON이라면

트랜잭션은 이미 끝났더라도 영속성 컨텍스트가 요청 끝까지 살아 있기 때문에 LAZY 로딩이 가능할 수 있다.

```text
Service의 트랜잭션 종료
        ↓
Controller에서 order.getOrderItems()
        ↓
영속성 컨텍스트가 살아 있음
        ↓
필요하면 SQL 실행
```

이것이 OSIV의 가장 직관적인 장점이다.

---

## OSIV와 DB Connection

OSIV를 이해할 때 가장 헷갈리는 부분은 다음 둘을 구분하는 것이다.

```text
영속성 컨텍스트(EntityManager / Session)
DB Connection
```

둘은 같은 것이 아니다.

### 영속성 컨텍스트

JPA 엔티티를 관리한다.

- 엔티티 식별
- 1차 캐시
- dirty checking
- Lazy Loading을 위한 프록시 관리

### DB Connection

실제 DB와 통신할 때 사용하는 JDBC Connection이다.

Connection Pool을 사용한다면 보통 HikariCP 같은 풀에서 빌려온다.

```text
애플리케이션
    ↓
Connection Pool
    ↓
JDBC Connection
    ↓
DB
```

### OSIV ON에서 Connection은 무조건 요청 내내 잡고 있는가?

이 부분은 구현과 Connection 획득 방식에 따라 세부 동작이 달라질 수 있으므로, 단순히 "OSIV가 켜져 있으면 요청 시작부터 끝까지 Connection 하나를 무조건 점유한다"고 외우면 부정확하다.

보다 중요한 관점은 다음이다.

> OSIV가 켜져 있으면 서비스 트랜잭션 종료 이후에도 Lazy Loading을 통해 추가 SQL이 발생할 수 있는 범위가 HTTP 요청 끝까지 늘어난다.

즉, 컨트롤러, View 렌더링, JSON 직렬화 등 서비스 계층 바깥에서 DB 접근이 일어날 수 있다.

```text
HTTP 요청 시작
    ↓
영속성 컨텍스트 Open
    ↓
Service
    ↓
Transaction 시작
    ↓
Connection 사용 + SQL
    ↓
Transaction 종료
    ↓
Controller
    ↓
Lazy Loading 발생
    ↓
추가 DB 접근 / Connection 사용 가능
    ↓
응답 종료
    ↓
영속성 컨텍스트 Close
```

따라서 외부 API 호출이나 복잡한 응답 처리처럼 요청 시간이 길어질 때, 예상하지 못한 시점의 DB 접근이 성능 문제로 이어질 수 있다.

---

## OSIV ON의 장점

### 1. 서비스 밖에서도 Lazy Loading이 가능하다

개발이 편하다.

```java
Order order = orderService.getOrder(id);

// 서비스 트랜잭션이 끝난 뒤에도 가능할 수 있음
order.getOrderItems().size();
```

서비스에서 필요한 모든 연관관계를 미리 초기화하지 않아도 된다.

### 2. View 계층에서 필요한 데이터를 쉽게 탐색할 수 있다

서버 사이드 렌더링 환경에서는 View가 엔티티 연관관계를 접근해야 하는 경우가 있는데, OSIV가 이를 편리하게 해준다.

---

## OSIV ON의 단점

### 1. SQL이 어디서 실행되는지 예측하기 어려워진다

아래 코드는 단순한 getter처럼 보인다.

```java
order.getOrderItems().size();
```

하지만 LAZY 관계라면 실제로는 SQL이 실행될 수 있다.

즉, DB 접근이 Service에만 존재하는 것이 아니라 Controller나 직렬화 과정까지 퍼질 수 있다.

### 2. N+1 문제를 숨기기 쉽다

예를 들어 주문 목록을 반환한다고 하자.

```java
List<Order> orders = orderService.findOrders();
```

응답 생성 과정에서 각 주문의 회원을 접근하면:

```java
for (Order order : orders) {
    order.getMember().getName();
}
```

OSIV 덕분에 코드 자체는 정상 동작할 수 있지만 실제 SQL은 다음처럼 될 수 있다.

```text
SELECT * FROM orders;

SELECT * FROM member WHERE id = 1;
SELECT * FROM member WHERE id = 2;
SELECT * FROM member WHERE id = 3;
...
```

즉, 문제를 즉시 드러내기보다 숨겨버릴 수 있다.

### 3. 서비스 계층 밖에서 DB I/O가 발생한다

일반적으로 다음 구조가 더 명확하다.

```text
Controller
    ↓
Service  ← DB 접근과 트랜잭션 경계
    ↓
Repository
```

그런데 OSIV를 사용하면:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Controller에서 다시 Lazy Loading → DB 접근
```

DB 접근 경계가 불명확해질 수 있다.

### 4. 긴 요청과 결합될 때 주의가 필요하다

예를 들어 다음과 같은 흐름이 있다고 하자.

```text
DB 조회
  ↓
외부 API 호출 (느림)
  ↓
응답 객체 생성
  ↓
Lazy Loading으로 추가 DB 조회
```

OSIV가 켜져 있으면 요청의 후반부에서도 DB 접근이 발생할 수 있기 때문에, 부하 상황에서 Connection Pool 사용 패턴이나 응답 지연을 분석하기 어려워질 수 있다.

---

## OSIV OFF

OSIV를 끄면 보통 다음과 같은 설계를 유도한다.

```text
Controller
    ↓
Service
    ↓
@Transactional
    ↓
필요한 데이터 전부 조회
    ↓
DTO 생성
    ↓
Transaction 종료
    ↓
Controller에 DTO 반환
```

즉, **트랜잭션 안에서 필요한 데이터를 모두 준비해서 밖으로 내보내는 방식**이다.

### Fetch Join 사용

```java
@Query("""
    select o
    from Order o
    join fetch o.orderItems
    where o.id = :id
""")
Optional<Order> findWithItemsById(Long id);
```

### DTO로 변환

```java
@Transactional(readOnly = true)
public OrderResponse getOrder(Long id) {
    Order order = orderRepository.findWithItemsById(id)
        .orElseThrow();

    return OrderResponse.from(order);
}
```

이렇게 하면 Controller는 더 이상 JPA 엔티티를 탐색하면서 추가 SQL을 발생시키지 않는다.

```text
Controller
    ↓
OrderResponse
    ↓
DB 접근 없음
```

---

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

### JPA 영속성 컨텍스트 공유

같은 트랜잭션에서는 같은 영속성 컨텍스트를 사용하기 때문에 동일한 엔티티를 다시 조회할 때 1차 캐시를 활용할 수 있다.

```java
Member member1 = memberRepository.findById(1L).orElseThrow();
Member member2 = memberRepository.findById(1L).orElseThrow();

System.out.println(member1 == member2);
```

일반적인 JPA 환경에서는 같은 영속성 컨텍스트 안에서 동일 식별자의 엔티티는 동일한 객체로 관리된다.

---

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

```text
트랜잭션으로 묶는다
    ≠
무조건 모든 SELECT가 완전히 동일한 시점의 데이터를 본다
```

정확한 보장은 isolation level과 DBMS 구현까지 봐야 한다.

### 여러 SELECT는 무조건 하나의 트랜잭션으로 묶어야 하는가?

아니다.

서로 완전히 독립적인 조회라면 하나의 큰 트랜잭션으로 묶을 필요가 적다.

반대로 여러 `SELECT`가 합쳐져 하나의 도메인 결과를 만든다면 트랜잭션으로 묶을 가치가 커진다.

### OSIV = 트랜잭션을 요청 끝까지 유지하는 것인가?

아니다.

가장 중요하게 구분해야 한다.

```text
OSIV
→ 영속성 컨텍스트 생명주기를 HTTP 요청까지 확장

@Transactional
→ DB 트랜잭션 경계를 정의
```

따라서 다음 상태가 가능하다.

```text
트랜잭션: 종료됨
영속성 컨텍스트: 살아 있음
```

### 영속성 컨텍스트 = DB Connection인가?

아니다.

```text
EntityManager / Session
    ≠
JDBC Connection
```

영속성 컨텍스트는 엔티티를 관리하는 JPA/Hibernate 개념이고, Connection은 실제 DB 통신 자원이다.

---

## 새롭게 알게 된 내용

여러 `SELECT`를 묶는 이유는 단순히 "트랜잭션이니까"가 아니다.

> 여러 조회 결과가 합쳐져 **하나의 의미 있는 결과**를 만든다면 하나의 트랜잭션 경계 안에서 다루는 것이 자연스럽다.

그리고 OSIV는 이 트랜잭션 경계와는 별개의 개념이다.

가장 중요한 그림은 다음과 같다.

```text
OSIV ON

HTTP Request
│
├── EntityManager OPEN ------------------------------┐
│                                                   │
│   Controller                                      │
│      ↓                                            │
│   Service                                         │
│      ↓                                            │
│   Transaction BEGIN                               │
│      ↓                                            │
│   SQL                                             │
│      ↓                                            │
│   Transaction END                                 │
│                                                   │
│   Controller / JSON Serialization                 │
│      ↓                                            │
│   Lazy Loading → 추가 SQL이 발생할 수도 있음       │
│                                                   │
└── HTTP Response ---------------- EntityManager CLOSE
```

반면 OSIV OFF에서는 보통 필요한 데이터 접근을 서비스 트랜잭션 안에서 끝내도록 설계하게 된다.

```text
OSIV OFF

Controller
   ↓
Service
   ↓
@Transactional
   ↓
필요한 데이터 조회
   ↓
DTO 변환
   ↓
트랜잭션 종료
   ↓
Controller
   ↓
추가 DB 접근 없음
```

따라서 OSIV를 이해할 때는 다음 세 개의 생명주기를 따로 생각하는 것이 중요하다.

1. **HTTP Request 생명주기**
2. **EntityManager / Persistence Context 생명주기**
3. **DB Transaction / Connection 사용 범위**

이 셋은 항상 동일하지 않다.
