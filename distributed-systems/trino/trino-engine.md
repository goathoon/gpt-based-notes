# Trino 엔진

## 핵심 요약

Trino는 **여러 데이터 저장소 위에서 SQL을 실행하는 분산 SQL 쿼리 엔진**이다. 중요한 점은 **Trino 자체가 데이터를 저장하는 데이터베이스가 아니라는 것**이다.

데이터는 S3, Iceberg, Hive, PostgreSQL, MySQL, Kafka, Cassandra 같은 외부 시스템에 존재하고, Trino는 Connector를 통해 이 데이터를 읽어 SQL로 분석한다.

핵심 특징은 다음과 같다.

- 분산 SQL 실행
- MPP(Massively Parallel Processing)
- 여러 데이터 소스를 하나의 SQL에서 조회하는 Query Federation
- Coordinator + Worker 구조
- Connector 기반 데이터 소스 확장
- S3 + Iceberg 같은 Data Lakehouse 환경에서의 대화형 분석

---

## 개념

### Trino의 역할

Trino는 데이터베이스라기보다 **SQL 실행 계층**에 가깝다.

```text
                  SQL
                   |
                   v
              +---------+
              |  Trino  |
              |  Query  |
              | Engine  |
              +---------+
                 / | \
                /  |  \
               v   v   v
          Iceberg MySQL Kafka
             |
             v
             S3
```

예를 들어 다음과 같이 데이터가 여러 시스템에 나뉘어 있을 수 있다.

```text
사용자 정보     -> MySQL
이벤트 로그     -> S3
분석 데이터     -> Iceberg
운영 데이터     -> PostgreSQL
```

Trino를 사용하면 이들을 하나의 SQL 인터페이스로 조회할 수 있다.

### Query Federation

Trino는 서로 다른 데이터 소스의 테이블을 하나의 SQL에서 JOIN할 수 있다.

```sql
SELECT
    u.user_id,
    u.name,
    SUM(o.amount)
FROM mysql.users u
JOIN iceberg.analytics.orders o
    ON u.user_id = o.user_id
GROUP BY
    u.user_id,
    u.name;
```

개념적으로는 다음과 같다.

```text
MySQL users
     +
S3 Iceberg orders
     |
     v
   Trino
     |
     v
JOIN 결과
```

이처럼 여러 시스템의 데이터를 하나의 쿼리 엔진에서 다루는 것을 **Query Federation**이라고 한다.

---

## 동작 원리

### Coordinator와 Worker

Trino는 기본적으로 **Coordinator + Worker** 구조로 동작한다.

```text
                   SQL Query
                       |
                       v
              +------------------+
              |   Coordinator    |
              |                  |
              | Parser           |
              | Analyzer         |
              | Optimizer        |
              | Scheduler        |
              +------------------+
                /       |       \
               v        v        v
          +---------+ +---------+ +---------+
          | Worker1 | | Worker2 | | Worker3 |
          +---------+ +---------+ +---------+
               |           |           |
               v           v           v
              Data Sources
          S3 / Iceberg / DB ...
```

#### Coordinator

Coordinator는 전체 쿼리의 두뇌 역할을 한다.

대략 다음 과정을 수행한다.

```text
SQL
 ↓
Parsing
 ↓
Semantic Analysis
 ↓
Logical Plan
 ↓
Query Optimization
 ↓
Distributed Execution Plan
 ↓
Worker에 Task 배분
```

즉, Coordinator는 쿼리를 해석하고 최적화한 후 실제 작업을 Worker들에게 분산한다.

#### Worker

Worker는 실제 데이터를 읽고 계산을 수행한다.

예를 들어 1TB의 데이터가 있을 때 여러 Worker가 데이터 조각을 나눠 병렬 처리할 수 있다.

```text
1 TB dataset

    Coordinator
         |
  ----------------
  |      |       |
  v      v       v

Worker1 Worker2 Worker3
 300GB   350GB   350GB
```

이런 구조 때문에 Trino는 **MPP(Massively Parallel Processing)** 시스템으로 볼 수 있다.

---

## 실행 단위: Query, Stage, Task, Split

Trino 실행 구조를 이해할 때 중요한 개념은 다음과 같다.

```text
Query
  |
  +-- Stage
        |
        +-- Task
              |
              +-- Split
```

### Query

사용자가 실행한 전체 SQL 요청이다.

### Stage

전체 Query를 분산 실행 가능한 큰 단계로 나눈 단위다. 일반적으로 데이터 교환이나 aggregation, join 같은 실행 경계를 중심으로 Stage가 나뉜다.

### Task

각 Stage를 실제 Worker에서 실행하기 위한 단위다. 동일한 Stage의 여러 Task가 서로 다른 Worker에서 병렬 수행될 수 있다.

### Split

Split은 Worker가 처리할 수 있는 **데이터 조각**이다.

예를 들어 S3에 다음과 같은 Parquet 파일이 있을 수 있다.

```text
sales/
 ├── part-001.parquet
 ├── part-002.parquet
 ├── part-003.parquet
 └── part-004.parquet
```

Trino는 이를 여러 Split으로 나눠 Worker에 배분할 수 있다.

```text
Worker1 -> split 1
Worker2 -> split 2
Worker3 -> split 3
Worker1 -> split 4
```

Split 수와 데이터 분포는 병렬성과 성능에 직접적인 영향을 준다.

---

## Connector와 Catalog

### Connector

Connector는 Trino와 특정 데이터 시스템 사이의 어댑터다.

```text
             Trino

        Connector Interface

       /        |         \
      v         v          v
   Hive      PostgreSQL    MySQL
 Connector    Connector   Connector
    |            |           |
    v            v           v
   S3        PostgreSQL     MySQL
```

Connector는 다음과 같은 역할을 담당한다.

- 테이블과 컬럼 메타데이터 조회
- Split 생성
- 데이터 읽기/쓰기
- Predicate Pushdown
- Projection Pushdown
- Connector가 지원하는 경우 aggregation/join 등의 추가 Pushdown

### Catalog

Catalog는 특정 Connector에 대한 **설정 인스턴스**라고 이해하면 된다.

예를 들어 다음처럼 설정 파일을 만들 수 있다.

```text
etc/catalog/

analytics.properties
mysql.properties
postgres.properties
```

```properties
# analytics.properties
connector.name=iceberg
...
```

Trino에서 테이블은 일반적으로 다음 형태로 접근한다.

```text
catalog.schema.table
```

예:

```sql
SELECT *
FROM analytics.sales.orders;
```

여기서 각각의 의미는 다음과 같다.

```text
analytics -> catalog
sales     -> schema
orders    -> table
```

---

## Trino와 Data Lakehouse

Trino는 Data Lakehouse 환경에서 자주 사용된다.

대표적인 조합은 다음과 같다.

```text
            Trino
              |
              v
           Iceberg
              |
              v
             S3
```

각 구성 요소의 역할은 서로 다르다.

```text
S3
↓
실제 데이터 저장

Iceberg
↓
테이블 포맷 / 메타데이터 / Snapshot

Trino
↓
SQL 실행 엔진
```

즉 Trino가 Iceberg 데이터를 조회한다고 해서 Trino가 Iceberg나 S3를 대체하는 것은 아니다.

- **S3**: 데이터 파일 저장
- **Iceberg**: 테이블 구조와 Snapshot/Metadata 관리
- **Trino**: SQL 분석 및 분산 실행

이 역할 분리를 이해하는 것이 중요하다.

---

## Trino와 Spark 비교

Trino와 Spark는 모두 분산 데이터 처리가 가능하지만 주요 목적이 다르다.

| 항목 | Trino | Spark |
|---|---|---|
| 주 목적 | SQL Analytics | 범용 데이터 처리 |
| Ad-hoc SQL | 매우 강함 | 가능 |
| Query latency | 낮은 편 | 상대적으로 높음 |
| ETL | 가능 | 매우 강함 |
| Python/Scala 데이터 처리 | 제한적 | 매우 강함 |
| ML | 주 목적 아님 | 지원 |
| BI 연결 | 매우 적합 | 가능 |

단순화하면 다음과 같다.

```text
Trino
-> SQL 분석 엔진

Spark
-> 분산 데이터 처리 플랫폼
```

실제 데이터 플랫폼에서는 둘을 함께 사용하는 경우가 많다.

```text
             Data Lake
                |
             S3/Iceberg
             /       \
            v         v

         Spark       Trino
           |           |
           v           v
       ETL/Batch    BI / Ad-hoc
```

---

## Trino가 빠른 이유

### 1. 분산 병렬 처리

대량의 데이터를 여러 Worker가 동시에 처리한다.

```text
1 worker
100 GB 처리

vs

50 workers
각 2 GB 처리
```

물론 실제 성능은 데이터 분포, 네트워크, Split 수, Connector, skew 등 다양한 요인에 따라 달라진다.

### 2. Pipeline Execution

가능하면 각 Operator의 결과를 모두 디스크에 쓴 다음 다음 연산을 시작하는 대신, 데이터가 생성되는 대로 다음 Operator로 전달한다.

즉 다음과 같은 형태의 파이프라인 처리를 지향한다.

```text
Table Scan
   |
   v
Filter
   |
   v
Project
   |
   v
Aggregation
```

### 3. In-memory Processing

중간 데이터와 연산 상태를 메모리에서 적극적으로 처리해 대화형 SQL에 적합한 성능을 낸다.

다만 메모리는 Trino 성능과 안정성에서 매우 중요한 자원이기 때문에 대형 JOIN, aggregation, skew가 있는 쿼리에서는 메모리 관리가 핵심 이슈가 된다.

### 4. Predicate Pushdown

예를 들어 다음 SQL이 있다고 하자.

```sql
SELECT user_id
FROM orders
WHERE date = '2026-08-23';
```

가능하다면 Trino는 `date` 조건을 Connector 또는 Storage 계층까지 전달하여 불필요한 데이터를 읽지 않도록 한다.

### 5. Projection Pushdown

전체 컬럼을 읽는 대신 쿼리에서 필요한 컬럼만 읽도록 한다.

```text
전체 데이터
  ↓
필요한 partition
+
필요한 column
  ↓
Trino
```

Columnar format인 Parquet/ORC와 함께 사용할 때 특히 효과적이다.

---

## 예시

### 서로 다른 Catalog JOIN

```sql
SELECT
    c.customer_id,
    c.name,
    SUM(o.amount) AS total_amount
FROM mysql.crm.customers c
JOIN iceberg.analytics.orders o
    ON c.customer_id = o.customer_id
GROUP BY
    c.customer_id,
    c.name;
```

이 SQL에서는 MySQL과 Iceberg라는 서로 다른 시스템의 데이터를 Trino가 하나의 쿼리에서 처리한다.

### Iceberg 테이블 조회

```sql
SELECT
    region,
    SUM(amount)
FROM iceberg.analytics.sales
WHERE order_date >= DATE '2026-08-01'
GROUP BY region;
```

적절한 partitioning과 predicate pushdown이 적용되면 전체 데이터를 읽지 않고 필요한 범위만 스캔할 수 있다.

---

## 헷갈리기 쉬운 점

### Trino는 데이터베이스가 아니다

Trino는 데이터를 영구 저장하는 시스템이 아니라 외부 데이터에 SQL을 실행하는 Query Engine이다.

### Trino와 Iceberg는 역할이 다르다

Iceberg는 Table Format이고, Trino는 Query Engine이다.

```text
Trino = SQL 실행
Iceberg = Table Metadata / Snapshot 관리
S3 = 데이터 저장
```

### Trino와 Hive는 동일한 것이 아니다

Trino가 Hive 또는 Iceberg Connector를 사용해 S3의 데이터를 읽을 수 있지만, Hive 실행 엔진을 호출해서 Trino SQL을 처리하는 것은 아니다. Trino Worker가 Connector를 통해 데이터를 직접 읽고 연산한다.

### Split은 단순히 파일 하나와 항상 동일하지 않다

Split은 Connector가 정의하는 논리적인 데이터 처리 단위다. 파일 기반 저장소에서는 파일 또는 파일 일부와 연관될 수 있지만, 데이터 소스에 따라 Split의 의미와 생성 방식이 달라질 수 있다.

---

## 새롭게 알게 된 내용

Trino를 깊게 이해하려면 다음 실행 계층을 이어서 공부하는 것이 좋다.

```text
Query
  ↓
Stage
  ↓
Task
  ↓
Split
  ↓
Driver
  ↓
Operator
```

이 구조를 이해하면 다음 주제로 자연스럽게 확장된다.

- Exchange와 Shuffle
- Broadcast Join / Partitioned Join
- Dynamic Filtering
- Data Skew
- Memory Pool과 Spill
- `EXPLAIN` / `EXPLAIN ANALYZE`
- Predicate / Projection Pushdown
- Query Scheduler
- Worker 병렬성
- Trino 성능 튜닝
