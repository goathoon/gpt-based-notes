# JVM과 mmap

## 핵심 요약

`mmap`은 파일이나 메모리 영역을 프로세스의 가상 주소 공간에 매핑하는 운영체제 기능이다. Linux에서는 `mmap`, `munmap`, `mprotect` 같은 시스템 콜로 제공된다.

JVM과 JDK도 필요할 때 이런 가상 메모리 기능을 사용한다. 대표적으로 `FileChannel.map()`을 통한 메모리 매핑 파일, JVM의 heap/native memory/code cache 같은 큰 메모리 영역 확보가 있다.

중요한 점은 Java에서 `new Object()`를 호출할 때마다 `mmap()`이 호출되는 것은 아니라는 것이다. JVM은 보통 OS로부터 큰 가상 메모리 영역을 확보한 뒤 그 내부에서 객체를 자체적으로 할당한다.

## 개념

일반적인 파일 읽기는 다음과 같이 생각할 수 있다.

```text
파일 → read() → 커널이 데이터를 읽음 → 사용자 메모리에서 사용
```

`mmap`을 사용하면 파일의 특정 영역을 프로세스의 가상 주소 공간에 연결할 수 있다.

```text
파일
  ↓ mmap
프로세스의 가상 주소 공간
  ↓
일반 메모리처럼 접근
```

예를 들어 Linux C에서는 다음처럼 사용할 수 있다.

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

`mmap()`을 호출했다고 파일 전체가 즉시 RAM에 올라오는 것은 아니다. 먼저 가상 주소 영역이 설정되고, 실제로 해당 주소를 접근할 때 필요한 페이지가 메모리에 없으면 page fault가 발생할 수 있다.

```text
mmap()
  ↓
가상 주소 영역 생성
  ↓
해당 주소 접근
  ↓
Page Fault
  ↓
OS가 필요한 파일 페이지를 메모리에 준비
  ↓
Page Table 연결
  ↓
이후 일반 메모리처럼 접근
```

따라서 `mmap`은 다음 개념들이 만나는 지점이다.

- Virtual Memory
- Page Table
- Page Fault
- Page Cache
- File System
- Copy-on-Write

### MAP_SHARED와 MAP_PRIVATE

`MAP_SHARED`는 변경 내용을 같은 매핑을 사용하는 다른 프로세스와 공유할 수 있고, 파일에도 반영될 수 있다.

`MAP_PRIVATE`는 수정이 발생하면 일반적으로 Copy-on-Write 방식으로 프로세스 전용 페이지가 만들어진다.

```text
파일
 ├── Process A mmap(MAP_SHARED)
 └── Process B mmap(MAP_SHARED)
```

### Anonymous mmap

파일 없이 메모리 영역 자체를 확보할 수도 있다.

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

이런 anonymous mapping은 네이티브 메모리 allocator가 큰 메모리 영역을 확보할 때 활용될 수 있다.

## JVM에서의 mmap

Java 애플리케이션이 직접 Linux의 `mmap()` 시스템 콜을 호출하는 경우보다는 JDK/JVM 내부의 네이티브 코드가 OS 기능을 사용하는 경우가 일반적이다.

```text
Java application
    ↓
JDK / JVM
    ↓
native code
    ↓
mmap / munmap / mprotect
    ↓
Linux virtual memory
```

### FileChannel.map()

Java의 `FileChannel.map()`은 파일을 메모리에 매핑해서 `MappedByteBuffer`로 접근할 수 있게 한다.

```java
try (FileChannel ch = FileChannel.open(Path.of("data.bin"))) {
    MappedByteBuffer buf =
        ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());

    byte b = buf.get(0);
}
```

Linux 환경에서는 이런 기능이 결국 OS의 memory mapping 기능과 연결된다.

### JVM 메모리 영역

HotSpot 같은 JVM은 heap, native memory, code cache 등 큰 메모리 영역을 관리하면서 OS의 가상 메모리 기능을 사용한다.

하지만 객체 하나를 만들 때마다 시스템 콜을 호출하는 구조는 아니다.

```text
OS
 ↓ mmap 등으로 큰 가상 메모리 영역 확보
JVM Heap
 ↓
JVM 내부 allocator
 ↓
new Object()
new Object()
new Object()
```

즉 `mmap`은 JVM이 OS로부터 큰 가상 메모리 영역을 확보하거나 파일을 매핑하는 데 사용하는 저수준 메커니즘이고, Java의 객체 할당은 그 위에서 JVM이 자체적으로 처리하는 고수준 동작이다.

## 헷갈리기 쉬운 점

### `new`를 할 때마다 mmap이 호출된다?

아니다. 일반적인 객체 할당은 JVM이 이미 확보한 heap 내부에서 처리된다. 특히 빠른 경로에서는 시스템 콜 없이 포인터 이동에 가까운 방식으로 객체 공간을 할당할 수 있다.

### mmap은 파일에만 사용된다?

아니다. `MAP_ANONYMOUS`처럼 파일과 연결되지 않은 가상 메모리 영역을 만들 수도 있다.

### mmap을 호출하면 데이터가 즉시 모두 RAM에 올라온다?

아니다. 일반적으로 실제 페이지 접근 시점에 page fault를 통해 필요한 페이지가 준비되는 demand paging 방식과 함께 동작한다.

## 새롭게 알게 된 내용

JVM을 이해할 때 Java heap만 독립적으로 보는 것보다 아래처럼 계층을 연결해서 보는 것이 중요하다.

```text
Java 객체
   ↓
JVM allocator / GC
   ↓
JVM의 가상 메모리 영역
   ↓
mmap / mprotect / munmap
   ↓
OS Virtual Memory
   ↓
Page Table / Page Fault / Physical Memory
```

이 관점을 가지면 JVM의 heap reservation/commit, GC 메모리 관리, `MappedByteBuffer`, native memory 같은 개념을 운영체제의 가상 메모리와 연결해서 이해할 수 있다.