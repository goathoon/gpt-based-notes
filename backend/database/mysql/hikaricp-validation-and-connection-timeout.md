# MySQL과 HikariCP의 validationTimeout 및 connectionTimeout 관계

## 핵심 요약

- `validationTimeout`과 `connectionTimeout`은 MySQL 서버 자체의 설정이 아니라 주로 **HikariCP 커넥션 풀 설정**이다.
- `connectionTimeout`은 애플리케이션이 풀에서 사용 가능한 커넥션을 획득할 때 기다리는 최대 시간이다(기본 30,000ms).
- `validationTimeout`은 HikariCP가 커넥션의 유효성을 검증하는 데 허용하는 시간이다(기본 5,000ms).
- HikariCP 설정에서는 **`validationTimeout < connectionTimeout`** 조건이 필요하다.
- 유효성 검사 실패가 즉시 애플리케이션의 커넥션 획득 실패를 의미하지는 않는다. 죽은 커넥션을 폐기하고 다른 커넥션을 시도할 수 있다.
- JDBC 호출 및 네트워크 지연으로 인해 실제 경과 시간이 설정된 `connectionTimeout`과 정확히 일치하지 않을 수 있다.

## 개념

| 설정 | 계층 | 의미 |
| --- | --- | --- |
| HikariCP `connectionTimeout` | 커넥션 풀 | `getConnection()`에서 커넥션을 얻기까지 기다리는 시간 한도 |
| HikariCP `validationTimeout` | 커넥션 풀 | 커넥션이 살아 있는지 확인할 때의 타임아웃 |
| MySQL Connector/J `connectTimeout` | JDBC 드라이버/네트워크 | MySQL 서버와 연결을 수립할 때의 타임아웃 |
| MySQL Connector/J `socketTimeout` | JDBC 드라이버/네트워크 | 네트워크 소켓 읽기 작업 등의 타임아웃 |

HikariCP는 일반적으로 JDBC `Connection.isValid()`나 설정된 `connectionTestQuery`를 통해 커넥션을 검증한다. 이는 애플리케이션의 일반적인 SQL 쿼리 실행 시간 제한과는 별개이다.

## 동작 원리

1. 애플리케이션이 `DataSource.getConnection()`을 호출한다.
2. HikariCP는 풀에서 커넥션을 획득하려고 한다. 이때 `connectionTimeout`이 전체 획득 대기 시간의 기준이 된다.
3. 필요한 경우 획득한 커넥션의 유효성을 검사한다. 검증에는 `validationTimeout`이 적용된다.
4. 유효한 커넥션이면 애플리케이션에 반환한다.
5. 유효하지 않으면 해당 커넥션을 폐기하고 남은 시간 동안 다른 커넥션을 시도한다.
6. 유효한 커넥션을 획득하지 못하면 대체로 `SQLTransientConnectionException` 등 커넥션 획득 실패 예외가 발생한다.

### 숫자로 보는 관계

`connectionTimeout=10000ms`, `validationTimeout=3000ms`라면, 검증이 필요한 커넥션 하나의 유효성 검사에 최대 약 3초를 할당한다. 검사에 실패하면 다른 커넥션을 다시 시도할 수 있지만, 이미 경과한 시간만큼 전체 획득 예산이 감소한다.

`validationTimeout`이 8초처럼 너무 길면 10초의 전체 획득 예산 중 대부분을 한 번의 검사에서 소비할 수 있다. 반대로 너무 짧으면 일시적인 데이터베이스/네트워크 지연을 커넥션 장애로 오인할 수 있다.

**주의:** `connectionTimeout`은 실행 중인 모든 JDBC/네트워크 작업을 정확히 그 시점에 강제 중단시키는 절대적 마감시간은 아니다. 드라이버, 네트워크, 풀 구현의 상태에 따라 실제 경과 시간이 더 길어질 수 있다. 또한 HikariCP가 커넥션을 대여할 때마다 반드시 검증을 실행하는 것은 아니며, 최근 사용된 커넥션에 대한 검증 생략 최적화가 있을 수 있다.

## 예시

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?connectTimeout=3000&socketTimeout=30000
    hikari:
      maximum-pool-size: 20
      connection-timeout: 10000
      validation-timeout: 3000
      max-lifetime: 1800000
```

- `connection-timeout: 10000` — 풀에서 커넥션 획득 시 대기 한도 10초
- `validation-timeout: 3000` — 유효성 검사 한도 3초
- JDBC `connectTimeout=3000` — TCP 연결 수립 관련 타임아웃 3초
- JDBC `socketTimeout=30000` — 소켓 응답 대기 관련 타임아웃 30초

위 값들은 예시이며 서비스의 지연 허용치, MySQL 장애 시 복구 전략, 네트워크 환경에 맞게 정해야 한다.

## 헷갈리기 쉬운 점

- **`connectionTimeout`과 JDBC `connectTimeout`은 다르다.** 전자는 풀에서 커넥션을 얻는 과정, 후자는 드라이버의 연결 수립과 관련된다.
- **`validationTimeout` 발생 = 곧바로 애플리케이션 타임아웃은 아니다.** 해당 커넥션을 폐기한 뒤 재시도할 수 있다.
- **`validationTimeout`과 `connectionTimeout`을 더해서 전체 소요 시간을 계산하지 않는다.** 유효성 검사는 커넥션 획득 과정 안에서 일어날 수 있다.
- **일반 SQL 실행 시간 제한과 다르다.** SQL 실행/소켓 I/O 타임아웃은 별도 설정을 확인해야 한다.
- **실제 오류 발생 시각이 `connectionTimeout` 값과 정확히 일치한다고 보장할 수 없다.** 드라이버 호출과 커넥션 풀 내부 처리의 영향을 받는다.

## 참고 자료

- HikariCP GitHub README: https://github.com/brettwooldridge/HikariCP
- MySQL Connector/J Configuration Properties: https://dev.mysql.com/doc/connector-j/en/connector-j-reference-configuration-properties.html
