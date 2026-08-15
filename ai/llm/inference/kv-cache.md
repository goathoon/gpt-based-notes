# KV Cache

## 핵심 요약

KV cache는 autoregressive LLM inference에서 **과거 토큰의 Key(K)와 Value(V)를 저장해 재사용함으로써 중복 계산을 제거하는 기법**이다.

새 토큰을 생성할 때 과거 토큰 전체의 K/V를 다시 계산하지 않고, 새 토큰의 Q/K/V만 계산한 뒤 기존 K/V를 cache에서 읽어 attention에 사용한다.

즉, 핵심 trade-off는 다음과 같다.

> 연산량 감소 ↔ KV cache 메모리 사용 증가

## 개념

Self-Attention은 대략 다음과 같이 계산된다.

$$
Attention(Q,K,V)=softmax\left(\frac{QK^T}{\sqrt{d}}\right)V
$$

Autoregressive generation에서는 토큰을 한 번에 하나씩 생성한다.

예를 들어 다음과 같이 문장이 생성된다고 하자.

```text
입력: I love
생성: AI
다음: because
다음: ...
```

이미 생성된 과거 토큰은 다음 generation step에서도 그대로 존재한다. 따라서 과거 토큰에서 만들어지는 K와 V 역시 다시 계산할 필요가 없다.

## 동작 원리

### KV cache가 없는 경우

새 토큰이 생성될 때마다 이전 토큰까지 다시 처리하여 K/V를 계산하게 된다.

```text
I                 → K,V 계산
I love            → I, love의 K,V 다시 계산
I love AI         → I, love, AI의 K,V 다시 계산
I love AI because → 또 전부 계산
```

이 과정에는 이미 이전 step에서 수행했던 계산이 반복된다.

### KV cache가 있는 경우

각 Transformer layer에서 과거 토큰의 K/V를 GPU 메모리에 저장한다.

```text
Token       KV cache
-----------------------
I        →  K₁, V₁
love     →  K₂, V₂
AI       →  K₃, V₃
```

다음 토큰을 생성할 때는 새 토큰에 대한 Q/K/V만 새로 계산한다.

```text
새 토큰의 Q
    │
    ▼
Q_new × [K₁ K₂ K₃ K_new]ᵀ
                ▲
          KV cache에서 읽음
```

따라서 과거 토큰들을 다시 Transformer layer에 통과시켜 K/V를 만드는 중복 계산을 피할 수 있다.

## 계산 복잡도에서 주의할 점

KV cache를 사용한다고 해서 attention 계산 자체가 O(1)이 되는 것은 아니다.

새 토큰의 Query는 여전히 모든 과거 토큰의 Key와 attention을 계산해야 한다. Context length가 $n$이라면 한 generation step의 attention은 과거 K들을 읽고 비교해야 하므로 대략 $O(n)$에 해당하는 작업이 남는다.

KV cache가 제거하는 핵심 비용은 **과거 토큰들의 K/V를 만들기 위해 모델을 반복해서 forward하는 중복 계산**이다.

따라서 다음 두 개념을 구분해야 한다.

- 과거 K/V 생성 비용: cache를 통해 재계산 제거
- 새 Query와 과거 Key 사이의 attention 비용: 여전히 context length에 따라 증가

## 예시

현재 context가 다음과 같다고 하자.

```text
I love AI
```

각 토큰의 K/V가 이미 cache에 있다면 다음 토큰을 생성할 때 모델은 `I`, `love`, `AI`의 K/V를 다시 만들 필요가 없다.

새 위치에 대한 Q/K/V를 계산하고, 새 Query를 기존 cache의 Key들과 비교한 뒤 Value들을 이용해 attention output을 계산한다.

생성이 진행될수록 새로운 K/V가 cache 뒤에 추가된다.

## 헷갈리기 쉬운 점

### KV cache는 attention 계산을 없애는 기술이 아니다

Attention은 여전히 수행된다. KV cache는 과거 토큰에 대한 K/V **재계산**을 없애는 기술이다.

### 속도를 얻는 대신 메모리를 사용한다

KV cache는 각 layer와 각 token에 대한 K/V를 저장해야 하므로 context가 길어질수록 GPU 메모리 사용량이 증가한다.

특히 긴 context를 사용하거나 많은 요청을 동시에 처리하는 LLM serving 환경에서는 KV cache가 상당한 GPU 메모리를 차지할 수 있다.

이 때문에 실제 LLM serving에서는 다음과 같은 기술들이 중요해진다.

- KV cache memory management
- PagedAttention
- continuous batching

## 새롭게 알게 된 내용

KV cache의 핵심을 한 문장으로 정리하면 다음과 같다.

> KV cache는 과거 토큰의 Key/Value를 GPU 메모리에 저장해서 다음 토큰 생성 시 과거 토큰을 다시 forward하지 않도록 함으로써 inference를 빠르게 한다.

또한 KV cache를 이해할 때는 단순히 "attention이 빨라진다"고 기억하기보다, **무엇을 재사용하고 어떤 계산은 여전히 남아 있는지**를 구분하는 것이 중요하다.