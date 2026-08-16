# mmap

## 핵심 요약

`mmap`은 파일이나 디바이스, 또는 익명 메모리 영역을 프로세스의 **가상 주소 공간**에 매핑하는 운영체제 기능이다. Linux에서는 `mmap()` 시스템 콜로 제공된다.

핵심 아이디어는 파일을 `read()`로 복사해서 읽는 대신, 특정 파일 영역을 가상 메모리에 연결한 뒤 일반 메모리를 읽고 쓰듯 접근할 수 있게 하는 것이다.

## 개념

일반적인 파일 읽기는 다음과 같이 생각할 수 있다.

```text
파일
  ↓ read()
커널이 데이터 읽기
  ↓
사용자 메모리 버퍼
```

`mmap`을 사용하면 파일의 특정 범위를 프로세스의 가상 주소 공간에 연결한다.

```text
파일의 일부
   ↓ mmap
프로세스의 가상 주소 공간
   ↓
포인터로 일반 메모리처럼 접근
```

예를 들어 C에서는 다음과 같이 사용할 수 있다.

```c
int fd = open("data.txt", O_RDONLY);

char *p = mmap(
    NULL,
    4096,
    PROT_READ,
    MAP_PRIVATE,
    fd,
    0
);

printf("%c\n", p[0]);

munmap(p, 4096);
```

## 동작 원리

중요한 점은 `mmap()` 호출 직후 파일 전체가 물리 메모리(RAM)에 올라오는 것이 아니라는 것이다.

일반적으로 먼저 가상 주소 영역이 만들어지고, 실제로 해당 주소를 접근했을 때 필요한 페이지가 메모리에 없다면 **page fault**가 발생한다.

```text
mmap()
  ↓
가상 주소 영역(VMA) 생성
  ↓
프로세스가 해당 주소 접근
  ↓
페이지가 아직 RAM에 없음
  ↓
Page Fault
  ↓
OS가 필요한 파일 페이지를 메모리에 준비
  ↓
Page Table 엔트리 설정
  ↓
명령 재실행
  ↓
일반 메모리처럼 접근
```

따라서 `mmap`은 다음 OS 개념들과 밀접하게 연결된다.

- Virtual Memory
- Virtual Address Space
- Page Table
- Page Fault
- Demand Paging
- Page Cache
- File System
- Copy-on-Write

## 파일 매핑과 Page Cache

파일을 `mmap`하면 프로세스의 가상 주소가 파일의 특정 영역과 연결된다. 실제 파일 데이터 페이지는 일반적으로 커널의 **page cache**와 연계되어 관리된다.

개념적으로 보면 다음과 같다.

```text
Process Virtual Address
        ↓
     Page Table
        ↓
Physical Page / Page Cache
        ↓
       File
```

그래서 같은 파일을 여러 프로세스가 매핑하는 경우, 조건에 따라 같은 물리 페이지를 공유할 수 있다.

## MAP_SHARED와 MAP_PRIVATE

### MAP_SHARED

`MAP_SHARED`는 같은 파일 영역을 매핑한 프로세스끼리 변경 내용을 공유할 수 있는 매핑이다. 수정된 페이지는 파일에도 반영될 수 있다.

```text
           File
            ↓
       Shared Page
        ↙       ↘
 Process A     Process B
```

프로세스 간에 파일 기반 공유 메모리처럼 사용할 수도 있다.

### MAP_PRIVATE

`MAP_PRIVATE`는 해당 프로세스만의 private mapping을 만든다.

처음에는 원본 페이지를 공유할 수 있지만, 프로세스가 페이지를 수정하려 하면 **Copy-on-Write(COW)** 가 적용될 수 있다.

```text
수정 전

Process A ─┐
           ├── Shared Physical Page
Process B ─┘

Process A가 write

Process A ─── Copied Private Page
Process B ─── Original Page
```

즉 수정 내용은 원본 파일에 직접 반영되지 않는다.

## Anonymous mmap

`mmap`은 반드시 파일과 연결될 필요는 없다.

`MAP_ANONYMOUS`를 사용하면 파일 없이 가상 메모리 영역을 확보할 수 있다.

```c
void *p = mmap(
    NULL,
    4096,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
);
```

이 경우 파일 매핑이 아니라 새로운 메모리 영역을 프로세스의 주소 공간에 만든다.

큰 메모리 영역을 확보하는 allocator나 runtime에서 이런 anonymous mapping을 활용할 수 있다.

## mmap과 read의 차이

`read()`는 보통 명시적인 사용자 버퍼를 두고 파일 데이터를 그 버퍼로 읽는다.

```text
read()

File
 ↓
Kernel
 ↓
User Buffer
```

`mmap`은 파일을 가상 주소 공간에 연결한 뒤 load/store 명령으로 접근한다.

```text
mmap()

File
 ↓
Page Cache / Physical Page
 ↑
Page Table
 ↑
Virtual Address
```

따라서 애플리케이션 입장에서는 별도의 `read()` 호출 없이 메모리 접근으로 파일 내용을 사용할 수 있다.

다만 `mmap`이 항상 `read()`보다 빠른 것은 아니다. 접근 패턴, 파일 크기, page fault 비용, 순차/랜덤 접근 여부 등에 따라 적절한 방식이 달라진다.

## 헷갈리기 쉬운 점

### mmap을 호출하면 파일이 전부 RAM에 올라온다?

아니다. 일반적으로 필요한 페이지가 실제 접근될 때 page fault를 통해 준비되는 demand paging 방식으로 동작한다.

### mmap은 물리 메모리를 직접 매핑하는 기능이다?

일반적인 파일 `mmap`에서는 프로세스가 물리 주소를 직접 다루는 것이 아니다. 프로세스는 여전히 **가상 주소**를 사용하고, OS가 page table을 통해 실제 페이지와 연결한다.

### mmap은 파일 전용 기능이다?

아니다. `MAP_ANONYMOUS`를 사용하면 파일이 없는 메모리 영역도 만들 수 있다.

### mmap이면 복사가 무조건 없다?

단순히 "zero-copy"라고 외우면 부정확할 수 있다. `mmap`은 전통적인 `read()`처럼 애플리케이션 버퍼로 명시적으로 복사하는 단계를 피할 수 있지만, 실제 I/O 과정에서는 디스크에서 메모리로 데이터를 가져오는 작업과 page fault 처리는 여전히 필요하다.

## 관련 시스템 콜

`mmap`과 함께 자주 등장하는 시스템 콜은 다음과 같다.

- `munmap()` : 매핑 제거
- `mprotect()` : 메모리 영역의 읽기/쓰기/실행 권한 변경
- `msync()` : 파일-backed mapping의 변경 내용을 파일과 동기화

## 정리

`mmap`을 이해할 때는 단순히 "파일을 메모리처럼 읽는 기능"으로 끝내기보다 다음 흐름으로 이해하면 좋다.

```text
File
 ↓
mmap
 ↓
Virtual Address Space
 ↓
Memory Access
 ↓
Page Fault
 ↓
Page Cache / Physical Page
 ↓
Page Table Mapping
```

즉 `mmap`은 **파일 시스템과 가상 메모리를 연결하는 대표적인 OS 인터페이스**다.