# cgroup 메모리 제한 감소로 인한 mmap 검색 인덱스 페이지 캐시 회수와 p99 지연

## 핵심 요약

JVM Heap이 정상이어도 컨테이너의 **cgroup 메모리 제한**에 가까워지면 Linux가 파일 기반 페이지 캐시를 회수할 수 있다. `mmap()`으로 접근하는 검색 인덱스의 페이지가 회수된 후 다시 필요해지면, 디스크 읽기를 수반하는 **major page fault**가 발생할 수 있고 API의 p99 지연이 증가한다. 이는 Heap/GC 지표만으로는 드러나지 않는 OS 메모리 압박 문제다.

> **암기 문장:** Heap은 정상인데 memory limit을 줄였더니 mmap 인덱스의 페이지 캐시가 reclaim되고, 재접근 시 major fault와 disk read가 증가해 p99가 튀었다.

## 상황

| 항목 | 관측값 |
| --- | --- |
| JVM Heap | 약 900 MB, 정상 범위 |
| Container Memory | 3.92 GiB / 4 GiB |
| `memory.stat` anon | 약 1.1 GiB |
| `memory.stat` file | 약 2.7 GiB |
| 검색 인덱스 | 약 3 GB, `mmap()`으로 읽음 |
| 변경 사항 | 컨테이너 memory limit 6 GiB → 4 GiB |
| 증상 | 변경 후 API p99 급증 |

`anon`은 주로 익명 메모리, `file`은 파일 기반 캐시/매핑 등에 해당한다. `file` 전체가 특정 검색 인덱스만을 의미하지는 않는다. 또한 `mmap()`으로 매핑한 파일의 **가상 주소 공간 크기**와 실제 RAM에 상주하며 cgroup에 계상되는 메모리 크기는 다르다.

## 원인과 동작 원리

```text
Container memory limit 6 GiB → 4 GiB
                 ↓
cgroup 메모리 압박 증가
                 ↓
Linux 메모리 reclaim 활성화
                 ↓
회수 가능한 파일 기반 페이지 캐시 축소
                 ↓
mmap 검색 인덱스의 상주 페이지 일부가 제거됨
                 ↓
검색 요청에서 해당 페이지 재접근
                 ↓
페이지가 RAM에 없고 저장장치에서 읽어야 하는 경우
                 ↓
Major Page Fault → Disk Read I/O
                 ↓
요청 처리 지연 증가 → API p99 급증
```

파일-backed 페이지는 깨끗한(clean) 상태라면 필요할 때 다시 읽을 수 있어 메모리 압박 상황에서 회수 대상이 되기 쉽다. 재접근 시 항상 major fault가 나는 것은 아니다. 페이지가 다른 캐시 계층에 있거나 RAM 내에서 해결되면 minor fault 등으로 처리될 수 있다. **디스크 I/O가 필요한 fault가 major fault**다.

p99는 일부 요청만 디스크 읽기를 기다려도 크게 영향을 받을 수 있다. 따라서 Heap 사용량이 안정적이더라도 파일 캐시의 working set을 담기에 제한이 부족하면 tail latency가 악화될 수 있다.

## 진단 지표 3가지

| 지표 | 해석 |
| --- | --- |
| `pgscan` ↑ | 커널이 메모리 회수를 위해 페이지를 스캔함 |
| `pgmajfault` ↑ | 디스크 I/O가 필요한 주요 페이지 폴트 증가 |
| Disk read I/O ↑ | 저장장치에서 데이터를 다시 읽는 작업 증가 |

세 지표가 **memory limit 감소 이후 p99 증가와 시간상 함께 움직이는지** 확인한다. `pgscan`만으로 특정 인덱스 페이지가 회수되었다고 단정할 수 없고, `pgmajfault`와 디스크 읽기 증가도 다른 워크로드 때문에 발생할 수 있다. 가능하면 `memory.stat`의 reclaim 관련 지표, `memory.events`의 `high`/`max`/`oom` 이벤트, major fault 및 블록 I/O 지표를 함께 비교한다. 지표 이름과 제공 범위는 cgroup v1/v2 및 커널 버전에 따라 달라질 수 있다.

## 헷갈리기 쉬운 점

- **JVM Heap ≠ Container Memory**: 컨테이너 메모리에는 Heap 외에도 Metaspace, thread stack, native/direct memory, `mmap` 파일 페이지, page cache 등이 관여한다. 일부 계상·공유 방식은 cgroup과 커널 설정에 좌우된다.
- **Heap 정상 ≠ 메모리 압박 없음**: Heap이 약 900 MB여도 cgroup 전체가 4 GiB 제한에 가까우면 파일 페이지 reclaim이 발생할 수 있다.
- **`mmap` 자체 ≠ 모든 페이지가 항상 RAM에 상주**: 파일 페이지는 접근에 따라 적재·회수된다.
- **Page fault ≠ 반드시 디스크 읽기**: minor fault는 디스크 I/O 없이 해결될 수 있고, major fault는 I/O가 필요한 경우다.
- **GC 문제와 구분**: 이 시나리오의 유력 원인은 OS/cgroup memory pressure다. GC 로그에 상응하는 pause 증가가 없다면 GC 튜닝만으로 근본 해결하기 어렵다.

## 확인 및 대응 방향

1. 제한 변경 전후의 container memory, `anon`/`file`, reclaim, major fault, disk reads, API p99를 같은 시간축에서 비교한다.
2. 가능하면 제한을 복구하거나 인덱스의 실제 hot working set을 수용할 만큼 메모리 여유를 확보했을 때 p99가 회복되는지 검증한다.
3. 인덱스 접근 패턴과 캐시 효율을 분석하고 필요한 경우 검색 인덱스 크기, 접근 지역성, 동시성, 메모리 요청·제한을 조정한다.

## 결론

**관측된 상관관계:** memory limit을 6 GiB에서 4 GiB로 줄인 뒤 컨테이너 메모리가 제한에 근접하고 API p99가 급증했다.

**유력한 원인 가설:** 제한 축소로 `mmap` 인덱스의 파일 페이지 캐시가 압박을 받아 회수되고, 재접근 시 major page fault와 디스크 읽기가 늘어 tail latency가 악화되었다.

**확증에 필요한 증거:** 동일 시간대의 reclaim(`pgscan`), major fault(`pgmajfault`), disk read 증가와, 메모리 여유를 복구했을 때의 개선 여부. 단순히 Heap이 정상이라는 사실만으로 GC 또는 다른 병목을 완전히 배제하지는 않는다.