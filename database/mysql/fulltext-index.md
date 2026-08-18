# MySQL Full-Text Index

## 핵심 요약

MySQL의 Full-Text Index는 문자열 전체를 정렬해 찾는 일반 B-Tree 인덱스와 달리, 문서를 **token(단어) 단위로 분해한 뒤 역색인(inverted index)** 구조로 저장해 검색하는 인덱스다.

핵심 형태는 다음과 같다.

```text
단어 → 해당 단어가 등장하는 문서(row)
```

예를 들어 문서가 다음과 같다면:

```text
1: "mysql index performance"
2: "mysql full text search"
3: "elasticsearch full text search"
```

개념적인 역색인은 다음처럼 만들어진다.

```text
mysql         → [1, 2]
index         → [1]
performance   → [1]
full          → [2, 3]
text          → [2, 3]
search        → [2, 3]
elasticsearch → [3]
```

따라서 `MATCH ... AGAINST` 검색 시 모든 문자열을 처음부터 끝까지 훑는 대신, 검색어 token에 해당하는 역색인을 조회해 관련 문서를 찾는다.

## 개념

일반적인 문자열 검색과 Full-Text Search의 차이는 **검색 단위**에 있다.

### B-Tree 인덱스

문자열 값 자체를 정렬해서 관리한다.

```text
"mysql full text"
"mysql index"
"oracle database"
```

### Full-Text Index

문장을 token으로 분해한 뒤 단어에서 문서를 찾는 방향으로 관리한다.

```text
mysql → row 1, row 2
full  → row 1
text  → row 1
index → row 2
```

즉 Full-Text Index의 핵심은 **문서 → 단어**가 아니라 **단어 → 문서** 방향으로 정보를 저장하는 역색인이다.

## 동작 원리

대략적인 흐름은 다음과 같다.

```text
문서 입력
   ↓
Tokenizer
   ↓
단어 추출
   ↓
Stopword 및 token 조건 처리
   ↓
Inverted Index 생성
   ↓
검색어 token화
   ↓
Inverted Index 조회
   ↓
관련 문서 반환
```

예를 들어 다음 문장은:

```text
MySQL full text search
```

대략 다음 token들로 나뉠 수 있다.

```text
mysql
full
text
search
```

검색 시에도 같은 방식으로 검색어를 token화하고, 각 token이 등장한 문서를 역색인에서 찾아 결과를 만든다.

## 예시

```sql
CREATE TABLE articles (
    id BIGINT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    FULLTEXT INDEX ft_idx (title, content)
);
```

검색은 다음과 같이 할 수 있다.

```sql
SELECT *
FROM articles
WHERE MATCH(title, content)
      AGAINST('mysql search');
```

이 경우 MySQL은 `mysql`, `search`와 같은 token을 기준으로 Full-Text Index를 조회한다.

## 검색 모드

### Natural Language Mode

기본 검색 모드로, 단순 포함 여부뿐 아니라 관련도(relevance)를 계산해 결과를 정렬하는 데 사용할 수 있다.

```sql
SELECT id, title,
       MATCH(title, content)
       AGAINST('mysql index') AS score
FROM articles
WHERE MATCH(title, content)
      AGAINST('mysql index');
```

개념적으로는 문서 내 단어 빈도와 전체 문서에서의 희소성 같은 요소가 관련도에 영향을 준다.

### Boolean Mode

검색 조건을 직접 제어할 수 있다.

```sql
SELECT *
FROM articles
WHERE MATCH(title, content)
      AGAINST('+mysql -oracle' IN BOOLEAN MODE);
```

```text
+mysql   반드시 포함
-oracle  반드시 제외
```

prefix 검색도 가능하다.

```sql
AGAINST('mysql*' IN BOOLEAN MODE)
```

## 헷갈리기 쉬운 점

### `LIKE '%keyword%'`와 Full-Text Search는 같은 검색이 아니다

```sql
WHERE content LIKE '%mysql%'
```

은 substring 검색이다.

반면:

```sql
WHERE MATCH(content) AGAINST('mysql')
```

은 기본적으로 token 기반 검색이다.

예를 들어:

```text
content = "mysqlserver"
```

이라면 `LIKE '%mysql%'`은 매칭될 수 있지만, tokenizer가 `mysqlserver`를 하나의 token으로 처리했다면 Full-Text Search에서 `mysql` 검색으로는 매칭되지 않을 수 있다.

### 한글 검색

한국어처럼 단어 경계를 단순히 공백만으로 처리하기 어려운 언어에서는 tokenizer 설정이 중요하다. MySQL에서는 ngram parser를 활용할 수 있다.

예를 들어 2-gram이라면 `데이터베이스`를 개념적으로 다음처럼 나눌 수 있다.

```text
데이
이터
터베
베이
이스
```

이 때문에 한글 Full-Text Search에서는 ngram 관련 설정이 검색 품질에 큰 영향을 줄 수 있다.

## 비용과 트레이드오프

Full-Text Index는 검색 성능을 높이는 대신 인덱스 유지 비용이 추가된다.

```text
검색 속도 및 검색 기능 ↑

대신

INSERT/UPDATE/DELETE 비용 ↑
디스크 사용량 ↑
인덱스 관리 비용 ↑
```

## 한 문장 정리

**MySQL Full-Text Index는 문장을 token으로 분해한 뒤 `단어 → 해당 단어가 등장하는 row` 형태의 역색인을 만들고, `MATCH ... AGAINST` 실행 시 이 역색인을 조회하여 관련 문서를 찾는 인덱스다.**