# Kotlin 확장 함수와 Receiver 람다: `runCatching`을 Java와 비교해서 이해하기

## 핵심 요약

Kotlin 표준 라이브러리의 다음 코드는 확장 함수, 제네릭, receiver가 있는 함수 타입, `inline`, `Result`, expression 기반 `try-catch`를 한 번에 보여준다.

```kotlin
public inline fun <T, R> T.runCatching(
    block: T.() -> R
): Result<R> {
    return try {
        Result.success(block())
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

Java 개발자 관점에서 핵심은 두 가지다.

- `T.runCatching(...)`은 **확장 함수(extension function)** 다. 실제로 `T` 클래스에 메서드를 추가하는 것이 아니라, 마치 멤버 함수처럼 호출할 수 있게 해주는 문법이다.
- `block: T.() -> R`은 **receiver가 있는 함수 타입(function type with receiver)** 이다. 람다 내부에서 `T` 객체를 `this`처럼 사용할 수 있다.

개념적으로 Java로 옮기면 다음과 비슷하다.

```java
static <T, R> Result<R> runCatching(
    T receiver,
    Function<T, R> block
) {
    try {
        return Result.success(block.apply(receiver));
    } catch (Throwable e) {
        return Result.failure(e);
    }
}
```

Kotlin에서는 이를 다음처럼 호출할 수 있다.

```kotlin
"123".runCatching {
    toInt()
}
```

Java식 사고로 보면 대략 다음과 같다.

```java
runCatching("123", str -> Integer.parseInt(str));
```

## 개념

### 1. `<T, R>`: 제네릭 함수

Kotlin:

```kotlin
fun <T, R> ...
```

Java:

```java
<T, R> ...
```

둘 다 타입 파라미터를 선언한다.

`runCatching`에서는:

- `T`: 함수를 호출하는 receiver 객체의 타입
- `R`: 람다가 반환하는 결과 타입

예를 들어:

```kotlin
"123".runCatching {
    toInt()
}
```

이라면 대략:

```text
T = String
R = Int
```

이다.

### 2. `T.runCatching`: 확장 함수

```kotlin
fun <T, R> T.runCatching(...)
```

여기서 `T.`는 이 함수의 **extension receiver**를 의미한다.

즉 다음처럼 호출할 수 있다.

```kotlin
val text = "123"

text.runCatching {
    toInt()
}
```

겉보기에는 `String` 클래스에 `runCatching()`이라는 멤버 함수가 있는 것처럼 보이지만, 실제로 `String` 클래스가 수정된 것은 아니다.

Java식으로 생각하면 receiver가 명시적인 첫 번째 인자로 들어간 static 메서드에 가깝다.

```java
static <T, R> Result<R> runCatching(
    T receiver,
    Function<T, R> block
)
```

따라서:

```kotlin
text.runCatching { ... }
```

은 개념적으로:

```java
runCatching(text, ...);
```

과 유사하다.

### 3. `(T) -> R`과 `T.() -> R`의 차이

일반적인 함수 타입은 다음과 같다.

```kotlin
(T) -> R
```

예:

```kotlin
val length: (String) -> Int = {
    it.length
}
```

Java의 `Function<T, R>`와 비슷하게 생각할 수 있다.

```java
Function<String, Integer> length = str -> str.length();
```

반면 receiver가 있는 함수 타입은:

```kotlin
T.() -> R
```

이다.

예:

```kotlin
val length: String.() -> Int = {
    length
}
```

이 람다 내부에서 `this`는 `String` 객체다.

```kotlin
val length: String.() -> Int = {
    this.length
}
```

따라서 다음 두 형태를 비교하면 차이가 명확하다.

```kotlin
// 일반 람다
val a: (String) -> Int = {
    it.length
}
```

```kotlin
// receiver 람다
val b: String.() -> Int = {
    length
}
```

receiver 람다는 인자를 `it`으로 받는 대신, 그 객체를 `this`로 사용한다.

## 동작 원리

### `"123".runCatching { toInt() }` 해석

다음 코드를 보자.

```kotlin
val result = "123".runCatching {
    toInt()
}
```

`T`는 `String`이므로 함수 시그니처는 개념적으로 다음처럼 구체화된다.

```kotlin
String.runCatching(
    block: String.() -> Int
): Result<Int>
```

람다 내부에서는 receiver인 `"123"`이 `this`가 된다.

그래서:

```kotlin
toInt()
```

는 사실상:

```kotlin
this.toInt()
```

이다.

### 왜 `block()`만 호출해도 receiver가 전달되는가

구현에는 다음 코드가 있다.

```kotlin
Result.success(block())
```

`block`의 타입은:

```kotlin
T.() -> R
```

인데 `block(this)`가 아니라 `block()`이라고 호출한다.

그 이유는 현재 함수 자체가 이미 `T`의 확장 함수이기 때문이다.

```kotlin
fun <T, R> T.runCatching(...)
```

따라서 함수 내부의 `this`가 이미 `T` 타입 receiver다.

```kotlin
"123".runCatching { ... }
```

에서는 함수 내부의:

```kotlin
this
```

가 `"123"`이다.

receiver 람다인 `block()`을 현재 receiver 컨텍스트에서 호출하면, 이 `this`를 receiver로 사용한다.

개념적으로는 다음처럼 생각할 수 있다.

```kotlin
block.invoke(this)
```

## `Result<R>`와 예외 처리

`runCatching`은 예외를 밖으로 던지는 대신 `Result` 값으로 감싼다.

```kotlin
return try {
    Result.success(block())
} catch (e: Throwable) {
    Result.failure(e)
}
```

성공하면:

```kotlin
Result.success(value)
```

실패하면:

```kotlin
Result.failure(exception)
```

을 반환한다.

예:

```kotlin
val result = runCatching {
    "abc".toInt()
}
```

`"abc".toInt()`는 예외를 발생시키지만, `runCatching`이 이를 잡아 실패한 `Result`로 만든다.

```kotlin
println(result.isFailure) // true
```

이후 다음과 같이 처리할 수 있다.

```kotlin
val value = runCatching {
    "abc".toInt()
}.getOrElse {
    0
}
```

## Kotlin의 `try-catch`는 expression이다

Kotlin에서는 `try-catch` 자체가 값을 만든다.

```kotlin
val value = try {
    10
} catch (e: Exception) {
    20
}
```

따라서 다음처럼 `return try`가 가능하다.

```kotlin
return try {
    Result.success(block())
} catch (e: Throwable) {
    Result.failure(e)
}
```

Java의 전통적인 `try-catch`는 statement 중심이라 이런 형태로 직접 값을 반환하는 표현은 사용할 수 없다.

## `catch (e: Throwable)`과 Java 비교

Kotlin:

```kotlin
catch (e: Throwable)
```

Java:

```java
catch (Throwable e)
```

차이는 Kotlin의 변수 선언 순서가 `이름: 타입`이라는 점이다.

```kotlin
val name: String
```

Java에서는:

```java
String name;
```

이다.

또한 `Throwable`은 `Exception`보다 더 넓다.

```text
Throwable
├── Exception
│   ├── RuntimeException
│   └── ...
└── Error
    ├── OutOfMemoryError
    ├── StackOverflowError
    └── ...
```

따라서 `runCatching`은 `Exception`뿐 아니라 `Throwable` 계열 전체를 잡는다는 점을 기억해야 한다.

Coroutine 코드에서는 `CancellationException` 같은 취소 신호를 일반 실패처럼 처리하지 않도록 특히 주의할 필요가 있다.

## `inline`의 의미

```kotlin
public inline fun ...
```

Kotlin의 `inline` 함수는 람다를 사용하는 고차 함수에서 호출 비용과 람다 객체 생성 비용을 줄이기 위해 컴파일러가 함수 본문을 호출 위치에 펼칠 수 있게 한다.

예:

```kotlin
runCatching {
    foo()
}
```

개념적으로 다음과 비슷하게 펼쳐질 수 있다.

```kotlin
try {
    Result.success(foo())
} catch (e: Throwable) {
    Result.failure(e)
}
```

Java에는 이에 직접 대응하는 언어 키워드가 없다. Java/JVM의 JIT compiler도 런타임에 method inlining을 수행할 수 있지만, Kotlin의 `inline`은 언어/컴파일러 차원의 기능이다.

## 표준 라이브러리의 annotation

### `@InlineOnly`

```kotlin
@InlineOnly
```

Kotlin 표준 라이브러리 내부에서 inline을 전제로 사용하는 API임을 나타내는 annotation이다. 일반 애플리케이션 코드에서 직접 사용할 일은 거의 없다.

### `@SinceKotlin("1.3")`

```kotlin
@SinceKotlin("1.3")
```

Kotlin 1.3부터 제공된 API임을 나타낸다.

## Kotlin의 기본 visibility

Kotlin에서는 아무 visibility modifier를 쓰지 않으면 기본적으로 `public`이다.

```kotlin
fun hello()
```

은 사실상:

```kotlin
public fun hello()
```

와 같다.

Java에서는 modifier가 없으면 package-private이므로 차이가 있다.

## 일반 `runCatching`과 확장 `runCatching`의 차이

Kotlin 표준 라이브러리에는 두 형태가 있다.

### 일반 함수

```kotlin
public inline fun <R> runCatching(
    block: () -> R
): Result<R>
```

사용:

```kotlin
runCatching {
    text.toInt()
}
```

### 확장 함수

```kotlin
public inline fun <T, R> T.runCatching(
    block: T.() -> R
): Result<R>
```

사용:

```kotlin
text.runCatching {
    toInt()
}
```

차이는 두 번째 형태가 receiver를 가지고 있다는 점이다.

## 다른 Kotlin API와의 연결

receiver 람다를 이해하면 `apply`, `run`, `with` 같은 scope function도 쉽게 이해할 수 있다.

예:

```kotlin
val user = User().apply {
    name = "Kim"
    age = 30
}
```

`apply`의 람다는 개념적으로 다음 형태다.

```kotlin
T.() -> Unit
```

그래서 람다 내부에서:

```kotlin
name = "Kim"
```

처럼 receiver의 필드나 메서드에 바로 접근할 수 있다.

Java라면 보통 다음처럼 작성한다.

```java
User user = new User();
user.setName("Kim");
user.setAge(30);
```

이 receiver 람다 문법은 Kotlin DSL, Gradle Kotlin DSL, Jetpack Compose와 같은 API 설계에서도 매우 자주 활용된다.

## Java와 비교

| Kotlin | 의미 | Java 관점 |
|---|---|---|
| `public` | 공개 함수 | `public` |
| `inline` | 함수/람다 inline 가능 | 직접 대응 없음 |
| `fun` | 함수 선언 | 메서드 선언 |
| `<T, R>` | 제네릭 타입 | `<T, R>` |
| `T.` | extension receiver | 명시적인 `T receiver` 인자와 유사 |
| `runCatching` | 함수 이름 | 메서드 이름 |
| `block: T.() -> R` | T를 receiver로 쓰는 람다 | `Function<T, R>`와 유사 |
| `Result<R>` | 성공/실패를 값으로 표현 | 별도 Result 타입 필요 |
| `try { ... }` | 값을 반환하는 expression | 전통 Java에서는 statement 중심 |
| `catch (e: Throwable)` | 예외 처리 | `catch (Throwable e)` |

## 예시

### 일반 람다

```kotlin
val convert: (String) -> Int = {
    it.toInt()
}
```

Java:

```java
Function<String, Integer> convert = str -> Integer.parseInt(str);
```

### receiver 람다

```kotlin
val convert: String.() -> Int = {
    toInt()
}
```

호출 대상 객체를 람다 내부의 `this`처럼 사용할 수 있다.

### 확장 함수 + receiver 람다

```kotlin
val result = "123".runCatching {
    toInt()
}
```

Java식으로 생각하면:

```java
Result<Integer> result = runCatching(
    "123",
    str -> Integer.parseInt(str)
);
```

## 헷갈리기 쉬운 점

### 확장 함수는 실제 클래스 멤버를 추가하는 것이 아니다

```kotlin
fun String.foo() { ... }
```

라고 선언했다고 해서 JVM의 `String` 클래스 자체에 `foo()`가 추가되는 것은 아니다. Kotlin 컴파일러가 확장 함수를 정적으로 해석한다.

### `(T) -> R`과 `T.() -> R`은 다르다

```kotlin
(T) -> R
```

은 `T`를 인자로 받는다.

```kotlin
T.() -> R
```

은 `T`를 receiver로 받는다.

따라서 전자는 보통 `it.foo()`를 사용하고, 후자는 `foo()` 또는 `this.foo()`처럼 사용할 수 있다.

### `runCatching`은 모든 `Throwable`을 잡는다

편리하지만, 무조건적인 예외 삼키기 방식으로 사용하면 안 된다. 특히 coroutine cancellation이나 시스템 수준 오류를 다룰 때는 어떤 예외를 다시 던져야 하는지 검토해야 한다.

## 새롭게 알게 된 내용

`runCatching`의 짧은 선언 한 줄에는 Kotlin의 중요한 언어 기능이 여러 개 결합되어 있다.

```kotlin
public inline fun <T, R> T.runCatching(
    block: T.() -> R
): Result<R>
```

이를 읽을 때 다음 순서로 해석하면 쉽다.

```text
1. <T, R>       → 제네릭 타입 T와 R을 사용한다.
2. T.runCatching → T 타입에 대해 호출할 수 있는 확장 함수다.
3. block:        → block이라는 파라미터를 받는다.
4. T.() -> R     → block 내부의 this는 T이며 R을 반환한다.
5. Result<R>     → 성공 값 또는 예외를 Result로 반환한다.
6. inline        → 고차 함수 호출 비용을 줄이기 위한 컴파일러 기능이다.
```

Java 개발자가 Kotlin을 읽을 때는 확장 함수와 receiver 람다를 각각 다음처럼 변환해서 생각하면 이해가 빠르다.

```text
extension receiver  → 명시적인 첫 번째 receiver 인자
receiver lambda     → Function<T, R>에서 T 인자를 this처럼 숨긴 형태
```

즉:

```kotlin
someObject.runCatching {
    doSomething()
}
```

을 보면 머릿속에서:

```java
runCatching(
    someObject,
    obj -> obj.doSomething()
);
```

으로 번역해보면 된다.