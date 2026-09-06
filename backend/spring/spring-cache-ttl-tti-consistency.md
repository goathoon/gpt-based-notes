# Spring Cache의 TTL, TTI와 캐시 일관성

## 핵심 요약

- `@Cacheable` 자체가 TTL이나 TTI를 관리하는 것은 아니다. 실제 만료 정책은 Redis, Caffeine 같은 캐시 구현체와 `CacheManager` 설정이 담당한다.
- 일반적인 TTL(Time To Live)은 **캐시에 저장된 시점부터 일정 시간이 지나면 만료**된다.
- 일반 TTL에서는 캐시를 조회해서 Hit가 나더라도 만료 시간이 보통 갱신되지 않는다.
- TTI(Time To Idle)는 **마지막 접근 시점부터 만료 시간을 다시 계산**하는 방식이다.
- Spring Data Redis에서는 버전에 따라 `enableTimeToIdle()`을 사용해 캐시 조회 시 만료 시간을 연장하는 형태의 TTI 동작을 구성할 수 있다.
- DB 데이터가 변경되었는데 캐시가 갱신되지 않으면 오래된 값이 반환되는 stale cache 문제가 생긴다.
- 이를 완화하기 위해 보통 `@CacheEvict`, `@CachePut`, TTL 등을 함께 사용한다.
- TTI는 오래된 캐시가 계속 조회될 경우 만료 시간이 계속 연장될 수 있으므로 데이터 일관성 관점에서 더 주의해야 한다.

## TTL과 TTI

### TTL(Time To Live)

TTL은 캐시 엔트리가 저장된 시점을 기준으로 일정 시간이 지나면 만료시키는 정책이다.

예를 들어 TTL을 10분으로 설정한 경우:

```text
00분: 캐시 저장 → TTL 10분 시작
05분: 캐시 조회 → Cache Hit, 남은 TTL 약 5분
10분: 캐시 만료
11분: 다시 조회 → Cache Miss → DB 조회 → 캐시에 다시 저장 → TTL 10분 재시작
```

일반적인 Redis TTL 설정에서는 단순 조회만으로 TTL이 다시 10분으로 늘어나지 않는다.

즉 다음처럼 생각할 수 있다.

```text
저장 -------- 10분 -------- 만료
       ↑
     조회해도 만료 시각은 그대로
```

캐시에 값을 다시 `put`하거나, 만료 후 재조회되어 새 값이 저장되는 경우에는 TTL이 새로 시작된다.

### TTI(Time To Idle)

TTI는 마지막 접근 이후 일정 시간 동안 접근이 없으면 캐시를 만료시키는 방식이다.

예를 들어 TTI가 10분이라면:

```text
00분: 캐시 저장
05분: 조회 → TTL이 다시 10분으로 연장
12분: 조회 → 다시 10분으로 연장
18분: 조회 → 다시 10분으로 연장
```

따라서 계속 접근되는 데이터는 오랫동안 캐시에 남을 수 있다.

```text
저장 ---- 조회 ---- 조회 ---- 조회
         ↓       ↓       ↓
       10분    10분    10분 재시작
```

반대로:

```text
00분: 저장
05분: 조회 → 만료 시각 연장
15분까지 추가 조회 없음
→ 만료
```

### Spring Data Redis에서의 예시

Spring Data Redis의 지원 버전에서는 다음과 같이 TTI 동작을 설정할 수 있다.

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration config =
        RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .enableTimeToIdle();

    return RedisCacheManager.builder(connectionFactory)
        .cacheDefaults(config)
        .build();
}
```

서비스에서는 일반적인 `@Cacheable`을 사용할 수 있다.

```java
@Cacheable(cacheNames = "users", key = "#id")
public User getUser(Long id) {
    System.out.println("DB 조회");
    return userRepository.findById(id).orElseThrow();
}
```

주의할 점은 TTI 갱신이 Spring Cache가 사용하는 접근 경로를 전제로 동작한다는 점이다. 다른 코드가 Redis에 직접 `GET`을 수행하면 기대한 방식으로 TTI가 갱신되지 않을 수 있다.

또한 `enableTimeToIdle()` 지원 여부와 내부 동작은 Spring Data Redis 버전에 따라 달라질 수 있으므로 실제 프로젝트 버전을 확인해야 한다.

## 캐시 데이터가 변경될 때 생기는 문제

캐시를 사용하면 DB와 캐시가 서로 다른 값을 가지는 **캐시 일관성(cache consistency)** 문제가 발생할 수 있다.

예를 들어 최초 상태가 다음과 같다고 하자.

```text
DB
user:1 → name = "Kim"

Cache
users::1 → "Kim"
```

이후 DB만 변경되면:

```text
DB
user:1 → name = "Lee"

Cache
users::1 → "Kim"
```

`@Cacheable` 조회에서는 캐시가 Hit되기 때문에 실제 DB를 조회하지 않고 계속 `"Kim"`을 반환할 수 있다.

이처럼 최신 데이터가 아닌 오래된 캐시 값을 **stale cache**라고 한다.

## 캐시 일관성을 유지하는 방법

### 1. 변경 시 캐시 삭제: `@CacheEvict`

가장 흔한 방식은 DB 데이터가 변경될 때 관련 캐시를 삭제하는 것이다.

```java
@CacheEvict(cacheNames = "users", key = "#id")
public void updateUser(Long id, String name) {
    userRepository.update(id, name);
}
```

그다음 조회에서는:

```text
Cache Miss
→ DB에서 최신 데이터 조회
→ 캐시에 최신 값 저장
→ 반환
```

흐름이 된다.

이 방식은 캐시에 최신 값을 직접 계산해서 넣을 필요가 없다는 장점이 있다.

### 2. 변경 결과로 캐시 갱신: `@CachePut`

수정 결과 자체를 캐시에 즉시 반영할 수도 있다.

```java
@CachePut(cacheNames = "users", key = "#result.id")
public User updateUser(Long id, String name) {
    return userRepository.update(id, name);
}
```

`@CachePut`은 캐시 Hit 여부와 상관없이 메서드를 항상 실행한 후 반환값으로 캐시를 갱신한다.

### 3. TTL을 이용한 최종적인 정리

애플리케이션에서 캐시 무효화가 누락되거나 일시적인 장애가 발생하더라도 TTL을 설정해 두면 오래된 데이터가 영구적으로 남는 것을 막을 수 있다.

따라서 TTL은 캐시 일관성을 완전히 해결하는 방법이라기보다 **stale cache의 최대 생존 시간을 제한하는 안전장치**로 보는 것이 좋다.

## 캐시 갱신에서 발생할 수 있는 Race Condition

단순히 `@CacheEvict`를 사용한다고 해서 모든 문제가 해결되는 것은 아니다.

예를 들어 다음 순서를 생각해 볼 수 있다.

```text
1. DB 업데이트 성공
2. Redis 캐시 삭제 시도
3. Redis 장애 발생
```

이 경우 DB에는 새 값이 있지만 캐시에는 이전 값이 남을 수 있다.

반대로 캐시를 먼저 삭제하면:

```text
1. 캐시 삭제
2. DB 업데이트 진행
```

그 사이 다른 요청이 들어와 아직 수정되지 않은 DB 값을 조회하고 다시 캐시에 저장할 수도 있다.

```text
Request A: Cache 삭제

Request B: Cache Miss
           ↓
           기존 DB 값 조회
           ↓
           기존 값을 다시 Cache에 저장

Request A: DB 업데이트 완료
```

결과적으로 DB와 캐시가 다시 불일치할 수 있다.

즉 캐시를 도입하면 다음을 함께 고려해야 한다.

- DB 변경과 캐시 무효화의 순서
- 트랜잭션과 캐시 갱신 시점
- 동시 요청에 의한 Race Condition
- Redis 장애 시 처리 방식
- TTL 설정
- 데이터가 일시적으로 불일치해도 허용 가능한지

## TTI와 캐시 일관성의 관계

TTI는 자주 사용되는 데이터를 캐시에 오래 유지한다는 장점이 있지만 stale cache 문제에서는 주의가 필요하다.

예를 들어 캐시에 잘못된 값이 들어간 상태에서 계속 조회가 발생하면:

```text
stale cache 저장
    ↓
조회 → TTI 연장
    ↓
조회 → TTI 연장
    ↓
조회 → TTI 연장
```

처럼 오래된 데이터의 수명이 계속 연장될 수 있다.

따라서 TTI를 사용할 때는 단순히 만료 정책에만 의존하지 말고 데이터 변경 시 명시적인 캐시 무효화 전략을 함께 사용하는 것이 중요하다.

## 어떤 데이터가 캐시에 적합한가

일반적으로 다음과 같은 데이터는 캐싱하기 좋다.

- 조회 빈도가 높다.
- 변경 빈도가 낮다.
- 약간의 데이터 지연을 허용할 수 있다.
- 조회 비용이 크다.

예:

- 상품 카테고리
- 코드 테이블
- 설정 정보
- 반복 조회되는 상품 상세 정보

반대로 다음과 같이 항상 최신 값이 중요한 데이터는 더 신중하게 캐싱해야 한다.

- 계좌 잔액
- 재고
- 좌석 수
- 결제 상태
- 실시간 권한 정보

이런 데이터에서는 캐시를 사용할 수 없는 것은 아니지만 일관성 전략과 무효화 전략을 명확하게 설계해야 한다.

## 헷갈리기 쉬운 점

### `@Cacheable`이 TTL을 관리하는가?

아니다.

```text
@Cacheable
   ↓
Spring Cache Abstraction
   ↓
CacheManager
   ↓
Redis / Caffeine / ...
```

TTL, TTI 같은 실제 만료 정책은 보통 하위 캐시 구현체가 담당한다.

### Cache Hit가 나면 TTL이 자동 갱신되는가?

일반 TTL에서는 아니다.

```text
Cache Hit → 값 반환
           TTL은 그대로
```

TTI가 활성화된 경우에는 접근 시 만료 시간을 연장하도록 구성할 수 있다.

### TTL만 짧게 두면 일관성 문제가 해결되는가?

완전히 해결되지는 않는다.

TTL이 10분이면 잘못된 캐시 값이 최대 10분 동안 노출될 가능성이 있다. 따라서 데이터 변경 이벤트를 알고 있는 경우 `@CacheEvict`나 `@CachePut`으로 적극적으로 캐시를 갱신하는 것이 일반적이다.

## 새롭게 알게 된 내용

Spring Cache를 설계할 때 중요한 것은 단순히 `@Cacheable`을 붙이는 것이 아니라 다음 세 가지를 함께 보는 것이다.

```text
캐시 조회 정책
     +
만료 정책(TTL / TTI)
     +
데이터 변경 시 무효화 정책
```

특히 TTI는 자주 접근되는 데이터의 캐시 효율을 높일 수 있지만, stale cache 역시 계속 살아남을 수 있기 때문에 데이터 정합성이 중요한 시스템에서는 명시적인 캐시 무효화 전략이 필수적이다.