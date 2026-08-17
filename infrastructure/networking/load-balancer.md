# Load Balancer 구조와 동작 원리

## 핵심 요약

Load Balancer는 클라이언트의 요청이나 연결을 받아 여러 Backend Server 중 하나로 전달하는 앞단 컴포넌트다.

주요 목적은 두 가지다.

- **High Availability**: 특정 서버가 장애가 나더라도 다른 서버로 요청을 전달한다.
- **Scale-out / Capacity 확장**: 여러 서버에 트래픽을 분산해 전체 처리 용량을 늘린다.

서버를 여러 대 둔다고 해서 항상 주목적이 처리량 증가인 것은 아니다. 서버 한 대로도 충분한 트래픽을 처리할 수 있지만 장애 대응을 위해 두 대 이상 배치한다면, 구조적으로는 Scale-out이지만 실질적인 목적은 High Availability다.

또한 Application Server만 여러 대 둔다고 시스템 전체가 고가용성이 되는 것은 아니다. DNS, Load Balancer, Application, Cache, Message Broker, Database 등 **전체 Request Path에서 SPOF와 Bottleneck을 확인해야 한다.**

---

## 개념

### Load Balancer란?

개념적으로 요청 흐름은 다음과 같다.

```text
Client
   │
   ▼
Load Balancer
   │
   ├── App A
   ├── App B
   └── App C
```

Load Balancer는 단순히 요청을 무작위로 분배하는 장비가 아니다. 일반적으로 다음 기능을 담당한다.

```text
Connection / Request 수신
        ↓
Backend Health 확인
        ↓
Backend 선택
        ↓
Connection / Request 전달
        ↓
Connection 및 Routing 상태 관리
```

### 기본 구성 요소

```text
┌──────────────────────────────┐
│         Load Balancer        │
│                              │
│  Listener (:80, :443)        │
│          │                   │
│          ▼                   │
│  Routing / Scheduling        │
│  - Round Robin               │
│  - Least Connections         │
│  - Hash                      │
│          │                   │
│          ▼                   │
│  Backend Pool                │
│  App1 / App2 / App3          │
│                              │
│  Health Checker              │
└──────────────────────────────┘
```

#### Listener

클라이언트의 Connection이나 Request를 특정 Port에서 수신한다.

예:

```text
HTTP  :80
HTTPS :443
```

#### Routing / Scheduling

어느 Backend Server로 요청을 전달할지 결정한다.

대표적인 방식은 다음과 같다.

- Round Robin
- Least Connections
- IP Hash
- Consistent Hashing

#### Backend Pool

실제 요청을 처리하는 서버 집합이다.

```text
App1
App2
App3
```

#### Health Check

Load Balancer는 Backend Server의 상태를 지속적으로 확인하고 비정상 서버를 후보군에서 제외할 수 있다.

```text
App1 ✅ Healthy
App2 ❌ Unhealthy
App3 ✅ Healthy
```

App2가 장애 상태라면 App1과 App3에만 요청을 전달한다.

---

## 동작 원리

## L4 Load Balancer

L4 Load Balancer는 Transport Layer, 즉 TCP/UDP 수준에서 동작한다.

주로 다음 정보를 이용한다.

- Source IP
- Destination IP
- Source Port
- Destination Port
- Protocol

예:

```text
Client
   │ TCP :443
   ▼
L4 Load Balancer
   │
   ├── App1
   ├── App2
   └── App3
```

HTTP의 `/payment`, `/user` 같은 Path나 Header까지 이해할 필요는 없다.

대표적인 예로 AWS NLB(Network Load Balancer)가 있다.

특징:

- TCP/UDP 레벨에서 동작
- 처리 과정이 상대적으로 단순함
- 높은 Throughput에 적합
- HTTP 내용 기반의 복잡한 Routing에는 적합하지 않음

## L7 Load Balancer

L7 Load Balancer는 Application Layer, 대표적으로 HTTP를 이해한다.

따라서 다음 정보를 기준으로 Routing할 수 있다.

- Host
- Path
- Header
- Cookie
- HTTP Method

Path 기반 Routing 예:

```text
/payment/** → Payment Server
/user/**    → User Server
```

Host 기반 Routing 예:

```text
payment.example.com → Payment Backend
admin.example.com   → Admin Backend
```

대표적인 예로 AWS ALB(Application Load Balancer)가 있다.

---

## L7 Load Balancer와 TCP Connection

L7 Load Balancer를 단순히 클라이언트의 패킷을 Backend Server로 그대로 전달하는 구조라고 이해하면 안 된다.

일반적으로 다음과 같이 연결이 분리될 수 있다.

```text
Client
   │
   │ TCP Connection #1
   ▼
Load Balancer
   │
   │ TCP Connection #2
   ▼
Backend Server
```

즉,

```text
Client ↔ Load Balancer
```

와

```text
Load Balancer ↔ Backend
```

는 서로 다른 TCP Connection일 수 있다.

Load Balancer가 클라이언트 연결을 직접 받아 종료하고 Backend와 별도의 연결을 생성하는 구조로, 흔히 Proxy 방식으로 이해할 수 있다.

---

## TLS Termination

L7 Load Balancer가 클라이언트 Connection을 직접 처리하는 경우 HTTPS의 TLS 처리도 Load Balancer에서 수행할 수 있다.

```text
Client
   │
   │ HTTPS
   ▼
Load Balancer
   │
   │ HTTP
   ▼
Application
```

Load Balancer가 TLS Handshake와 암복호화를 담당하고 Backend에는 HTTP로 전달하는 구성을 **TLS Termination**이라고 한다.

보안 요구사항에 따라 Backend 구간도 다시 HTTPS로 구성할 수 있다.

```text
Client
   │ HTTPS
   ▼
Load Balancer
   │ HTTPS
   ▼
Application
```

---

## Load Balancer 자체의 자원과 한계

Load Balancer 역시 무한한 성능을 가진 장비가 아니라 하나의 고성능 Network Server로 볼 수 있다.

대량의 트래픽을 처리할 때 다음과 같은 상태와 자원을 관리해야 한다.

- Socket
- TCP Connection State
- Network Buffer
- Timeout
- TLS State
- Backend Connection
- Routing State

예를 들어 Client가 100만 개의 TCP Connection을 유지하고 있다면 Load Balancer도 해당 Connection들을 처리해야 한다.

따라서 Load Balancer 성능은 다음 자원의 영향을 받는다.

- CPU
- Memory
- Network Bandwidth
- File Descriptor
- Connection Limit
- TLS 암복호화 비용

---

## High Availability와 SPOF

Load Balancer가 한 대뿐이라면 자체적으로 SPOF(Single Point of Failure)가 될 수 있다.

```text
Client
   │
   ▼
Load Balancer
   │
   ├── App1
   ├── App2
   └── App3
```

Application Server를 여러 대 두더라도 Load Balancer가 장애 나면 모든 요청이 Backend까지 도달하지 못한다.

그래서 실제 운영 환경에서는 Load Balancer도 다중화한다.

```text
             ┌── LB A ──┐
Client ──────┤          ├── App1
             └── LB B ──┤── App2
                        └── App3
```

AWS ALB/NLB와 같은 Managed Load Balancer를 사용하는 경우 Load Balancer 계층의 고가용성을 클라우드 제공자가 내부적으로 관리한다.

---

## 전체 Request Path 관점

High Availability는 특정 계층 하나가 아니라 전체 요청 경로를 기준으로 판단해야 한다.

```text
Client
   │
   ▼
DNS
   │
   ▼
Load Balancer
   │
   ▼
Application
   │
   ├── Redis
   ├── Kafka
   └── Database
```

각 계층은 새로운 SPOF나 Bottleneck이 될 수 있다.

예를 들어:

```text
Application Server 100대
          │
          ▼
      Database 1대
```

Application Layer는 충분히 Scale-out되어 있어도 모든 요청이 하나의 Database에 집중된다면 Database가 새로운 병목이 된다.

특정 계층을 Scale-out하면 병목이 완전히 사라진다기보다 **다른 계층으로 이동할 수 있다.**

---

## 헷갈리기 쉬운 점

### 서버 수가 늘면 항상 처리량 증가가 목적이다?

아니다. 서버를 여러 대 두는 목적은 처리량 증가일 수도 있고 장애 대응을 위한 High Availability일 수도 있다.

### Load Balancer는 요청을 그냥 랜덤하게 뿌린다?

아니다. Listener, Health Check, Scheduling, Backend Pool, Connection 관리 등 여러 기능을 수행한다.

### L7 Load Balancer는 패킷을 그대로 Backend로 넘긴다?

항상 그렇지 않다. 많은 L7 Load Balancer는 클라이언트의 TCP Connection을 직접 종료하고 Backend와 별도의 TCP Connection을 맺는 Proxy 구조로 동작한다.

### Application Server만 다중화하면 시스템 전체가 HA가 된다?

아니다. DNS, Load Balancer, Cache, Broker, Database 등 전체 Request Path의 SPOF와 Bottleneck을 확인해야 한다.

---

## 새롭게 알게 된 내용

Scale-out 시스템을 볼 때 중요한 질문은 단순히 다음이 아니다.

> 서버를 몇 대 늘렸는가?

보다 중요한 질문은 다음이다.

> 현재 전체 Request Path에서 병목과 SPOF가 어디에 존재하는가?

Load Balancer 자체도 Connection, Buffer, TLS State, Backend Connection 등 많은 상태와 자원을 관리하는 서버이며, Application 계층을 Scale-out했다고 해서 시스템 전체의 확장성과 가용성이 자동으로 보장되지는 않는다.
