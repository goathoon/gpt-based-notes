# TCP 소켓의 송수신과 대규모 트래픽 처리

## 핵심 요약

- TCP 소켓은 보통 **하나의 연결에서 송신(send)과 수신(recv)을 모두 수행**한다.
- TCP는 **Full Duplex(전이중)** 통신이므로 송신과 수신을 동시에 진행할 수 있다.
- "송신용 소켓 하나 + 수신용 소켓 하나"가 반드시 필요한 것은 아니다.
- 하나의 TCP 연결도 큰 트래픽을 처리할 수 있지만, 실제 처리량은 네트워크 대역폭, RTT, TCP 윈도우, 커널 버퍼, CPU, 애플리케이션 처리 속도 등의 영향을 받는다.
- 대규모 사용자 서버에서는 모든 사용자가 하나의 TCP 연결을 공유하는 것이 아니라, 보통 **클라이언트 연결마다 connected socket이 하나씩 생성**된다.
- 수만~수십만 개의 소켓을 처리할 때는 보통 `소켓 1개 = 스레드 1개` 구조 대신 `epoll`, `kqueue`, Java NIO, Netty 같은 **이벤트 기반 I/O**를 사용한다.

## 개념

### TCP 소켓의 송신과 수신

하나의 TCP 연결은 양방향으로 데이터를 주고받을 수 있다.

```text
Client socket  <=================>  Server socket
                 TCP connection

Client: send()  ----------------->  Server: recv()
Client: recv()  <-----------------  Server: send()
```

즉 소켓 자체가 송신용과 수신용으로 분리되는 것이 아니라, **하나의 연결된 소켓에 송신 경로와 수신 경로가 모두 존재**한다.

### Full Duplex

TCP는 Full Duplex 방식이므로 다음 작업이 동시에 일어날 수 있다.

```text
Thread / Task A -> send()
Thread / Task B -> recv()
```

채팅, 게임, 실시간 메시징처럼 언제 데이터가 도착할지 모르는 프로그램에서는 송신과 수신 로직을 별도 스레드나 비동기 작업으로 분리하기도 한다.

이것은 논리적인 처리 분리이며, 반드시 TCP 소켓을 두 개 만든다는 뜻은 아니다.

## 동작 원리

### 서버의 listen socket과 connected socket

서버에는 먼저 연결 요청을 받기 위한 listen socket이 있다.

```text
Listen Socket
     |
   accept()
     |
Connected Socket
     ↕
 send / recv
```

`accept()`가 성공하면 실제 클라이언트와 데이터를 주고받는 새로운 connected socket이 만들어진다.

따라서 여러 클라이언트가 접속하면 일반적으로 다음과 같은 형태가 된다.

```text
Client A ---- Connection A ---- Socket A
Client B ---- Connection B ---- Socket B
Client C ---- Connection C ---- Socket C
                              |
                            Server
```

서버 하나가 여러 연결을 관리하지만, 각 TCP 연결은 독립적인 소켓 상태를 가진다.

### 하나의 TCP 연결이 처리할 수 있는 트래픽

하나의 TCP 연결도 상당히 큰 트래픽을 처리할 수 있다. 소켓 하나라는 이유만으로 처리량이 작은 것은 아니다.

실제 최대 처리량은 다음 요소들의 영향을 받는다.

- NIC 및 네트워크 링크 대역폭
- RTT(Round Trip Time)
- TCP congestion control
- TCP receive/send window
- 커널 socket buffer
- 패킷 처리 비용과 CPU 성능
- 애플리케이션의 직렬화/역직렬화 및 비즈니스 로직 처리 속도

즉 "소켓 하나니까 한 번에 데이터 하나만 처리한다"고 이해하면 안 된다.

## 대규모 연결 처리

### Thread-per-Connection의 문제

단순한 서버는 다음과 같이 구현할 수 있다.

```text
Socket 1 -> Thread 1
Socket 2 -> Thread 2
Socket 3 -> Thread 3
...
Socket 100000 -> Thread 100000
```

하지만 연결 수가 매우 많아지면 스레드 메모리, context switching, scheduler 비용이 커진다.

그래서 대규모 서버에서는 이벤트 기반 I/O를 많이 사용한다.

```text
              Event Loop
                  |
        +---------+---------+
        |         |         |
     Socket 1  Socket 2  Socket 3
        |         |         |
        +---- 수많은 연결 ----+
```

대표적인 기술은 다음과 같다.

- Linux: `epoll`
- BSD/macOS: `kqueue`
- Java: NIO
- JVM 네트워크 프레임워크: Netty

이 구조에서는 적은 수의 이벤트 루프 스레드가 많은 소켓의 읽기/쓰기 가능 이벤트를 감시한다.

## 헷갈리기 쉬운 점

### "송신과 수신이 두 개니까 소켓도 두 개 아닌가?"

아니다. TCP 연결 하나는 기본적으로 양방향 통신이 가능하다.

```text
1 TCP connection
    ├─ send 방향
    └─ recv 방향
```

송신 로직과 수신 로직을 코드에서 분리할 수는 있지만 TCP 연결 자체는 하나일 수 있다.

### "대용량 트래픽이면 TCP 소켓 하나로 불가능한가?"

반드시 그렇지는 않다. 단일 TCP 연결도 높은 throughput을 낼 수 있다.

다만 많은 사용자를 동시에 서비스하는 서버에서는 보통 **한 연결을 모두가 공유하는 것이 아니라 여러 TCP 연결을 동시에 관리**한다.

### "한 TCP 연결 안에서 여러 요청을 동시에 보내면 완전히 독립적인가?"

TCP는 한 방향에 대해 **순서가 보장되는 byte stream**이다. 따라서 애플리케이션이 여러 논리적 요청을 하나의 연결에 실을 경우, 그 위에서 요청을 구분하거나 multiplexing하는 프로토콜이 필요할 수 있다.

HTTP의 발전 과정도 이와 연관된다.

- HTTP/1.1: 여러 연결 또는 제한적인 pipelining 사용
- HTTP/2: 하나의 연결 위에서 여러 stream을 multiplexing
- HTTP/3: QUIC 위에서 stream 단위 독립성을 강화

## 새롭게 알게 된 내용

소켓의 "송신/수신"과 서버의 "연결 개수"는 서로 다른 개념으로 구분해야 한다.

```text
하나의 TCP 연결
  -> 하나의 connected socket
  -> send + recv 모두 가능

대규모 서버
  -> 많은 connected socket
  -> 적은 수의 event loop/thread로 관리 가능
```

따라서 대규모 네트워크 서버를 이해할 때는 단순히 소켓 개수보다 **연결 모델, I/O 모델, 이벤트 루프, 버퍼링, 네트워크 대역폭**을 함께 봐야 한다.