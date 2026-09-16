# SimpleJpaRepository와 Spring Data JPA Repository 런타임 호출 원리

## 핵심 요약

Spring Data JPA에서 우리가 작성하는 `UserRepository extends JpaRepository<User, Long>` 같은 타입은 **인터페이스**일 뿐이다.

그런데도 `save()`, `findById()`, `delete()` 같은 메서드를 바로 호출할 수 있는 이유는 Spring Data JPA가 런타임에 Repository 프록시를 만들고, 기본 CRUD 메서드의 실제 구현체로 `SimpleJpaRepository`를 사용하기 때문이다.

전체 흐름을 단순화하면 다음과 같다.

```text
UserRepository interface
        ↓
Repository Proxy
        ↓
SimpleJpaRepository
        ↓
EntityManager
        ↓
Hibernate 등 JPA Provider
        ↓
SQL
        ↓
Database
```

즉, `JpaRepository`는 우리가 사용하는 추상화된 인터페이스이고, `SimpleJpaRepository`는 그 기본 CRUD 구현체이며, 실제 JPA 작업은 내부의 `EntityManager`를 통해 수행된다.

---

## 개념

### JpaRepository

애플리케이션 코드에서 직접 사용하는 Repository 인터페이스다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

개발자는 이 인터페이스만 선언하고 별도의 구현 클래스를 작성하지 않아도 된다.

`JpaRepository` 계층은 CRUD뿐 아니라 페이징, 정렬 등 데이터 접근에 자주 필요한 고수준 API를 제공한다.

---

### SimpleJpaRepository

Spring Data JPA가 제공하는 기본 Repository 구현체다.

개념적으로 다음과 비슷하다.

```java
public class SimpleJpaRepository<T, ID> {

    private final EntityManager entityManager;

    public <S extends T> S save(S entity) {
        if (isNew(entity)) {
            entityManager.persist(entity);
            return entity;
        }

        return entityManager.merge(entity);
    }

    public Optional<T> findById(ID id) {
        return Optional.ofNullable(
            entityManager.find(domainClass, id)
        );
    }
}
```

실제 구현은 더 복잡하지만 핵심은 `SimpleJpaRepository`가 `EntityManager`를 내부적으로 사용한다는 점이다.

따라서 다음 두 코드는 추상화 수준만 다르다.

```java
entityManager.persist(user);
```

```java
userRepository.save(user);
```

Spring Data JPA를 사용하면 개발자가 `EntityManager`를 직접 다루는 코드를 크게 줄일 수 있다.

---

## 왜 EntityManager보다 더 높은 수준의 인터페이스인가

`EntityManager`는 JPA 자체의 핵심 API다.

직접 사용한다면 보통 다음과 같은 코드를 작성한다.

```java
@Repository
public class UserRepository {

    @PersistenceContext
    private EntityManager em;

    public User save(User user) {
        em.persist(user);
        return user;
    }

    public User findById(Long id) {
        return em.find(User.class, id);
    }
}
```

Spring Data JPA를 사용하면 다음 정도만 작성해도 된다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

즉 Spring Data Repository는 `EntityManager`보다 더 강력한 ORM 엔진이라는 의미가 아니라, 애플리케이션 레벨에서 반복적으로 필요한 데이터 접근 로직을 **더 높은 수준으로 추상화한 API**라고 보는 것이 정확하다.

예를 들면 다음과 같은 기능을 제공한다.

- CRUD
- Pagination
- Sorting
- Query Method
- `@Query`
- Specification
- Query by Example

---

## 동작 원리

### 1. Repository 인터페이스 탐색

애플리케이션 시작 시 Spring Data JPA가 Repository 인터페이스를 탐색한다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

이때 Spring은 단순히 `UserRepository` 인터페이스 자체를 Bean으로 등록하는 것이 아니다.

Repository 생성을 담당하는 인프라가 개입한다.

개념적인 흐름은 다음과 같다.

```text
UserRepository
      ↓
JpaRepositoryFactoryBean
      ↓
JpaRepositoryFactory
      ↓
Repository Proxy 생성
```

---

### 2. JpaRepositoryFactory가 기본 구현체를 준비한다

`JpaRepositoryFactory`는 Repository의 메타데이터를 분석한다.

예를 들어 다음 정보를 파악한다.

```text
Domain Type = User
ID Type     = Long
```

그리고 Entity 관련 정보를 담은 `JpaEntityInformation`과 `EntityManager`를 이용하여 기본 구현체를 준비한다.

개념적으로는 다음과 비슷하다.

```java
new SimpleJpaRepository<User, Long>(
    entityInformation,
    entityManager
);
```

따라서 런타임에 대략 다음 구조가 만들어진다.

```text
ApplicationContext

UserRepository Bean
        ↓
Repository Proxy
        ↓
SimpleJpaRepository<User, Long>
        ↓
EntityManager
```

중요한 점은 애플리케이션에서 주입받는 객체가 일반적으로 `SimpleJpaRepository` 그 자체가 아니라 **Repository Proxy**라는 것이다.

---

### 3. Repository Proxy가 호출을 가로챈다

서비스에서 다음 코드를 실행한다고 하자.

```java
userRepository.save(user);
```

`userRepository`는 Repository Proxy이므로 호출이 바로 `SimpleJpaRepository`로 직행하는 것이 아니라 먼저 프록시가 메서드 호출을 받는다.

```text
UserService
    ↓
userRepository.save(user)
    ↓
Repository Proxy
```

프록시는 해당 메서드를 누가 처리할 수 있는지 판단하고 적절한 구현으로 위임한다.

`save()`는 기본 CRUD 메서드이므로 `SimpleJpaRepository`가 처리한다.

```text
Repository Proxy
      ↓
SimpleJpaRepository.save()
      ↓
EntityManager.persist() 또는 merge()
      ↓
JPA Provider
      ↓
SQL
```

---

## save() 호출 흐름

전체 흐름을 조금 더 구체적으로 보면 다음과 같다.

```java
userRepository.save(user);
```

```text
UserService
    |
    | save(user)
    ↓
Repository Proxy
    |
    | CRUD 기본 메서드인가?
    ↓
Repository composition / interceptor
    |
    ↓
SimpleJpaRepository.save(user)
    |
    ├─ 새 엔티티이면 EntityManager.persist()
    │
    └─ 기존 엔티티이면 EntityManager.merge()
            ↓
        Hibernate
            ↓
           SQL
```

즉 `save()`의 실제 구현은 Repository 인터페이스에 있는 것이 아니라 `SimpleJpaRepository` 쪽에 있다.

---

## Query Method는 SimpleJpaRepository로 가지 않는다

다음 Repository를 생각해보자.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    User findByName(String name);
}
```

`SimpleJpaRepository`에는 당연히 `findByName()`이라는 메서드가 없다.

이런 메서드는 Spring Data의 Query Method 처리 인프라가 처리한다.

개념적으로는 다음 흐름이다.

```text
userRepository.findByName("Kim")
              ↓
       Repository Proxy
              ↓
 QueryExecutorMethodInterceptor
              ↓
 QueryLookupStrategy
              ↓
 PartTreeJpaQuery 등 RepositoryQuery
              ↓
 EntityManager
              ↓
 JPQL / SQL
```

따라서 모든 Repository 메서드가 `SimpleJpaRepository`로 가는 것은 아니다.

다음처럼 나눠서 이해하는 것이 좋다.

```text
save()
findById()
delete()
findAll()
    ↓
SimpleJpaRepository
```

```text
findByName()
findByEmailAndStatus()
@Query(...)
    ↓
Spring Data Query Infrastructure
```

---

## Repository Proxy가 필요한 이유

단순히 Spring이 다음과 같은 구현 클래스를 생성하면 되지 않을까 생각할 수 있다.

```java
class UserRepositoryImpl
        extends SimpleJpaRepository<User, Long>
        implements UserRepository {
}
```

하지만 실제 Repository에는 여러 종류의 동작이 섞여 있을 수 있다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    User findByEmail(String email);

    @Query("...")
    List<User> search(...);

    // Custom Repository fragment method도 가능
}
```

각 메서드의 실제 처리 주체가 서로 다르다.

```text
                    Repository Proxy
                          |
          ┌───────────────┼────────────────┐
          ↓               ↓                ↓
SimpleJpaRepository   Query Method    Custom Fragment
          |               |                |
        CRUD         findByEmail()    customMethod()
```

Repository Proxy는 이 여러 구현 조각을 하나의 Repository 인터페이스 뒤에서 조합하는 facade 역할을 한다.

Spring Data에서는 이런 구조를 Repository Composition / Repository Fragment 관점으로 이해할 수 있다.

---

## @Transactional과의 관계

Repository 호출은 Spring의 여러 interceptor와 함께 동작할 수 있다.

개념적으로는 다음과 같은 구조로 볼 수 있다.

```text
UserService
    ↓
Repository Proxy
    ↓
TransactionInterceptor
    ↓
Repository Method Interceptor
    ↓
SimpleJpaRepository.save()
    ↓
EntityManager
```

따라서 Spring의 Proxy 개념을 이해하면 다음 기능도 함께 이해하기 쉬워진다.

```text
@Transactional
@Async
@Cacheable
Spring Data Repository
Spring AOP
```

세부 구현 방식은 기능마다 다를 수 있지만, 공통적으로 런타임 프록시와 interceptor를 활용하는 구조가 자주 등장한다.

---

## 전체 초기화 흐름

애플리케이션 시작부터 Repository 사용까지 전체 흐름을 정리하면 다음과 같다.

```text
애플리케이션 시작
       │
       ▼
Spring Data가 Repository interface 탐색

UserRepository
extends JpaRepository<User, Long>
       │
       ▼
JpaRepositoryFactoryBean
       │
       ▼
JpaRepositoryFactory
       │
       ├─ Domain type 분석: User
       ├─ ID type 분석: Long
       ├─ JpaEntityInformation 생성
       ├─ EntityManager 확보
       │
       ▼
SimpleJpaRepository<User, Long> 준비
       │
       ▼
Repository Proxy 생성
       │
       ▼
Spring Bean으로 등록
```

런타임에는 다음처럼 동작한다.

```text
userRepository.save(user)
       │
       ▼
Repository Proxy
       │
       ▼
SimpleJpaRepository.save()
       │
       ▼
EntityManager.persist() / merge()
       │
       ▼
Hibernate 등 JPA Provider
       │
       ▼
SQL
```

반면 Query Method는 다음 경로를 탄다.

```text
userRepository.findByEmail(email)
       │
       ▼
Repository Proxy
       │
       ▼
QueryExecutorMethodInterceptor
       │
       ▼
RepositoryQuery
       │
       ▼
EntityManager
```

---

## 헷갈리기 쉬운 점

### 1. 주입받는 객체가 SimpleJpaRepository 그 자체인가?

보통은 아니다.

애플리케이션 코드에서 주입받는 것은 Repository 인터페이스를 구현하는 런타임 프록시다.

```java
@Autowired
private UserRepository userRepository;
```

개념적으로 다음 구조다.

```text
UserRepository 타입
        ↑
Repository Proxy
        ↓
SimpleJpaRepository + Query Infrastructure + Custom Fragment
```

---

### 2. 모든 Repository 메서드가 SimpleJpaRepository로 가는가?

아니다.

기본 CRUD 구현은 주로 `SimpleJpaRepository`가 담당하고, Query Method나 `@Query` 기반 메서드는 Spring Data의 Query 실행 인프라가 처리한다.

```text
save()        → SimpleJpaRepository
findById()    → SimpleJpaRepository
findAll()     → SimpleJpaRepository

findByName()  → Query Method Infrastructure
@Query(...)   → Query Infrastructure
```

---

### 3. JpaRepository가 구현체인가?

아니다.

`JpaRepository`는 인터페이스이고 `SimpleJpaRepository`가 기본 구현체 역할을 한다.

정리하면 다음과 같다.

```text
JpaRepository
= 개발자가 사용하는 Repository API / interface

SimpleJpaRepository
= 기본 CRUD implementation

JpaRepositoryFactory
= Repository 구현 요소들을 조립하는 factory

Repository Proxy
= 애플리케이션이 실제 호출하는 runtime 객체

EntityManager
= 실제 JPA 연산을 수행하는 API
```

---

## 예시

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);
}
```

서비스 코드:

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public User create(User user) {
        return userRepository.save(user);
    }

    public Optional<User> findByEmail(String email) {
        return userRepository.findByEmail(email);
    }
}
```

호출 흐름은 서로 다르다.

```text
create()
  ↓
userRepository.save()
  ↓
Repository Proxy
  ↓
SimpleJpaRepository.save()
  ↓
EntityManager
```

```text
findByEmail()
  ↓
userRepository.findByEmail()
  ↓
Repository Proxy
  ↓
QueryExecutorMethodInterceptor
  ↓
RepositoryQuery
  ↓
EntityManager
```

---

## 소스 코드 읽는 순서

Spring Data JPA 내부 동작을 소스 레벨에서 확인하고 싶다면 다음 순서로 따라가면 이해하기 좋다.

```text
JpaRepositoryFactoryBean
        ↓
JpaRepositoryFactory
        ↓
RepositoryFactorySupport#getRepository()
        ↓
SimpleJpaRepository
        ↓
QueryExecutorMethodInterceptor
```

여기서 확인할 핵심 질문은 다음과 같다.

1. Repository interface는 어디서 발견되는가?
2. Repository Proxy는 어디서 만들어지는가?
3. 기본 Repository 구현체는 어디서 생성되는가?
4. `save()`는 왜 `SimpleJpaRepository`로 가는가?
5. `findByEmail()` 같은 Query Method는 왜 다른 경로로 가는가?

이 순서로 소스를 따라가면 Spring Data JPA의 Repository 추상화가 어떻게 런타임 객체로 완성되는지 연결해서 이해할 수 있다.

---

## 새롭게 알게 된 내용

Spring Data JPA Repository는 단순히 `SimpleJpaRepository` 하나를 자동 생성해서 사용하는 구조가 아니다.

핵심은 **Repository Proxy가 여러 종류의 구현을 조합하고 메서드 호출을 적절한 처리 주체에 전달한다는 점**이다.

```text
JpaRepository interface
       ↓
Repository Proxy
       ├─ CRUD → SimpleJpaRepository
       ├─ Query Method → Query Infrastructure
       ├─ @Query → Query Infrastructure
       └─ Custom Method → Repository Fragment
```

따라서 Spring Data JPA를 이해할 때는 다음 세 가지를 분리해서 보는 것이 중요하다.

```text
1. API 추상화
   JpaRepository

2. 기본 구현
   SimpleJpaRepository

3. 런타임 연결/디스패치
   Repository Proxy + RepositoryFactory + Interceptor
```

이 구조를 이해하면 "인터페이스만 선언했는데 왜 구현 없이 동작하는가?"라는 질문에 단순히 "Spring이 알아서 해준다"가 아니라, Repository Factory가 Proxy와 기본 구현체 및 Query 처리 인프라를 조립하기 때문이라고 설명할 수 있다.
