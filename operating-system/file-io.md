# 프로그램이 파일을 읽을 때 운영체제와 어떻게 협력할까?

노션에 정리했던 파일 I/O 학습 내용을 Markdown으로 옮기고, 용어와 코드 표기를 다듬은 문서다.

> **설명 범위**: Linux의 일반 파일을 동기식으로 읽는 기본적인 buffered I/O 경로를 다룬다. `O_DIRECT`, `mmap`, 비동기 I/O와 파일 시스템별 세부 동작은 별도로 공부한다.

> **학습 목표**
> 서버 성능 이슈가 왜 외부 I/O와 밀접한지 이해하고, 그중 파일 I/O가 OS를 거쳐 처리되는 흐름을 따라간다.

## 전체 흐름

외부 I/O는 파일이나 다른 프로그램과 데이터를 주고받는 작업으로, 결과를 기다리는 시간이 생길 수 있다. 파일 I/O도 그중 하나다.
파일을 읽을 때 OS는 먼저 RAM의 **Page Cache**를 확인한다.

- **Cache hit**: 필요한 데이터가 RAM에 있어 빠르게 반환할 수 있다.
- **Cache miss**: 저장장치의 읽기가 필요하다. 동기 방식이라면 요청 스레드는 그동안 블로킹 대기한다.

```text
프로그램 → open() → fd 획득 → read()
                                ↓
                         Page Cache 확인
                     ┌──────────┴──────────┐
                 Cache hit              Cache miss
                     ↓                      ↓
            프로그램에 결과 반환     저장장치 I/O 요청
                                           ↓
                                  요청 스레드 블로킹
                                           ↓
                                  완료 Interrupt
                                           ↓
                           스레드 재실행 → 결과 반환
```

> 핵심은 “디스크는 무조건 느리다”가 아니다. **외부 결과를 기다리는 동안 요청 처리 자원이 점유될 수 있고, 이 대기가 많이 쌓이면 서버 성능 문제가 된다**는 것이다.

## 1. 서버가 만나는 외부 I/O

여기서는 파일·네트워크·다른 프로세스와 데이터를 주고받는 일을 외부 I/O라고 부른다. 같은 컴퓨터의 다른 프로세스와 통신하거나 RAM의 Page Cache에서 파일 데이터를 받는 경우도 있으므로, ‘RAM 바깥의 작업’이라는 뜻은 아니다. 속도와 성공 여부를 프로그램 혼자 결정할 수 없기 때문에, 지연과 실패를 고려해야 한다.

| 분류 | 서버에서 만나는 예시 | 프로그램이 기다리는 대상 |
| --- | --- | --- |
| 네트워크 I/O | 외부 HTTP API, gRPC, JDBC DB 조회, Redis, 메시지 브로커, DNS | 네트워크와 다른 서버의 응답 |
| 파일·저장장치 I/O | 로컬 파일 읽기·쓰기, 업로드 파일, 로그 flush | Page Cache 또는 저장장치 |
| 원격 저장소 I/O | S3 같은 Object Storage | 원격 저장소의 응답 |
| 로컬 프로세스 I/O | Pipe, Unix Domain Socket, 외부 명령 실행 | 같은 컴퓨터의 다른 프로세스 |

예를 들어 Spring 서버가 JDBC로 DB를 조회하면, Spring 프로세스 입장에서는 DB 서버 응답을 기다리는 **네트워크 I/O**다. DB 서버가 자기 데이터 파일을 읽는 일은 DB 프로세스 입장의 **파일·저장장치 I/O**다.

```text
외부 대상에 요청
  → 결과가 늦거나 실패할 수 있음
  → 동기 방식이라면 요청 처리 스레드가 대기할 수 있음
  → 스레드·커넥션·파일 핸들 같은 자원이 점유됨
  → 응답 시간과 처리량에 영향
```

비동기·논블로킹 방식을 사용하면 기다리는 동안 특정 스레드를 묶지 않을 수 있다. 하지만 외부 결과가 늦거나 실패할 수 있다는 사실 자체는 사라지지 않는다.

## 2. “우리 서버는 파일 I/O를 별로 안 하는데요?”

Controller 코드에서 `Files.readString()` 같은 메서드를 거의 보지 못할 수 있다. 그래서 “우리 서버는 DB와 외부 API만 호출하니 파일 I/O는 별로 없지 않을까?”라고 생각하기 쉽다.
그 말은 **부분적으로 맞다.** 예를 들어 DB만 조회하는 어떤 API 요청은 애플리케이션이 로컬 파일을 직접 읽지 않을 수 있다. 이때 Spring 프로세스가 DB를 기다리는 일은 네트워크 I/O이고, DB가 자기 데이터 파일을 읽는 일은 DB 프로세스의 파일 I/O다.
하지만 **내 코드에 `Files`가 없다고 파일 I/O가 없다는 뜻은 아니다.** Spring, 라이브러리, HTTP 서버가 기능을 처리하는 과정에서 파일을 열고 읽거나 쓸 수 있다.

> **직접 파일 I/O**: 애플리케이션 코드가 `Files`, `FileInputStream` 등을 직접 호출한다.
>
> **간접 파일 I/O**: 프레임워크·라이브러리·서버가 우리가 사용한 기능을 처리하기 위해 내부에서 파일을 다룬다.

### 언제 파일 I/O가 숨어 있을까?

| 시점 | 우리가 쓰는 기능 | 내부에서 일어날 수 있는 파일 I/O |
| --- | --- | --- |
| 서버 시작 | Spring Boot 실행 | `application.yml`, JAR 내부 클래스·리소스, 자동 설정 메타데이터 읽기 |
| 서버 시작 | HTTPS 적용 | 인증서와 개인키 파일 읽기 |
| 서버 시작·실행 중 | 로그 | 설정 파일을 읽고, 실행 중에는 로그 파일에 쓰기 |
| 요청 처리 | `@Value("classpath:...")`, `ResourceLoader` | 클래스패스의 설정·템플릿·파일 읽기 |
| 요청 처리 | 정적 이미지·CSS·JS 제공 | 리소스 파일을 HTTP 응답으로 전송 |
| 요청 처리 | PDF·Excel·이미지 다운로드 | 파일 내용을 HTTP 응답으로 전송 |
| 요청 처리 | 파일 업로드 | 컨테이너가 업로드 내용을 메모리 또는 임시 파일에 보관 |

> 모든 API 요청이 파일 I/O를 하는 것은 아니다. 이 표의 목적은 “파일 I/O는 애플리케이션 코드에 직접 보이지 않아도, 서버의 시작 과정과 특정 기능의 내부에서 발생할 수 있다”는 점을 보여 주는 것이다.
예를 들어 파일 다운로드에서 우리는 `Resource`만 반환할 수 있다.

```java
@GetMapping("/manual")
public ResponseEntity<Resource> download() {
    Resource pdf = new FileSystemResource("/data/manual.pdf");
    return ResponseEntity.ok().body(pdf);
}
```

하지만 응답을 만들기 위해 프레임워크와 HTTP 서버는 결국 파일 내용을 HTTP 응답으로 전송한다.

```text
Controller가 Resource 반환
  → 프레임워크와 HTTP 서버가 파일 응답 처리
  → 파일 내용이 HTTP 응답으로 전송
  → 클라이언트가 다운로드
```

이 한 기능 안에는 파일 I/O와 네트워크 I/O가 이어질 수 있다.

```text
파일 읽기: SSD/HDD 또는 Page Cache → 서버
네트워크 쓰기: 서버 → Socket → 클라이언트
```

> 따라서 성능을 볼 때는 “내 Controller가 `Files`를 호출했는가?”보다, **사용한 기능의 호출 경로 어디에서 외부 I/O가 발생하는가?**를 보는 편이 더 중요하다.

## 3. 프로그램은 파일 읽기를 어떻게 시작할까?

Java에서는 다음처럼 파일 읽기를 시작할 수 있다.

```java
String notice = Files.readString(Path.of("/data/notice.txt"));
```

개발자는 Java 파일 API를 호출하고, JDK가 그 요청을 OS의 파일 읽기 요청으로 이어 준다.

```text
애플리케이션 코드 또는 프레임워크 기능
  → Java 파일 API
  → JDK의 파일 처리 코드
  → OS에 파일 접근 요청
  → 읽은 바이트가 Java로 돌아옴
```

[`Files.readString()`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/file/Files.html#readString(java.nio.file.Path))은 Java API이고, OS의 `read()`는 시스템 레벨의 파일 읽기 인터페이스다. 둘이 함수 하나씩 정확히 1:1로 연결된다고 외울 필요는 없다. JDK 버전, OS, 버퍼링 방식에 따라 세부 경로는 달라질 수 있다.
OS 입장에서는 보통 다음 두 단계로 시작한다.

```c
// 흐름 설명용 코드 조각: 헤더, 오류 처리, 반복 읽기는 생략했다.
int fd = open("/data/notice.txt", O_RDONLY);
char buffer[1024];
ssize_t count = read(fd, buffer, sizeof(buffer));
close(fd);
```

- `open()`: OS가 파일과 권한을 확인하고, 이후 이 파일을 가리킬 **File Descriptor(FD)** 를 반환한다.
- `read()`: 프로그램이 FD, 데이터를 받을 자신의 버퍼, 읽고 싶은 크기를 OS에 전달한다.

FD는 파일 데이터가 들어 있는 공간이 아니라, 프로세스가 OS에 “아까 열었던 그 파일”을 가리킬 때 쓰는 식별자다. OS는 열린 파일 상태에 어떤 파일인지와 현재 읽기 위치(offset)를 관리한다. 새로 독립적으로 연 일반 파일은 처음 위치(offset 0)에서 시작하며, 읽기에 성공하면 읽은 만큼 위치가 이동한다. 이 기본 동작은 [Linux `open(2)`](https://man7.org/linux/man-pages/man2/open.2.html)과 [`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html)에서 확인할 수 있다.

### 한 번에 보는 파일 읽기 순서

```text
1. 애플리케이션 또는 프레임워크가 파일 읽기 기능을 호출
   ↓
2. JDK가 OS에 파일 접근 요청
   ↓
3. open() → OS가 파일을 확인하고 fd 반환
   ↓
4. read(fd, buffer, size) → OS에 데이터 읽기 요청
   ↓
5. OS가 Page Cache 확인
   ├─ Cache hit
   │    → RAM에 있는 데이터를 프로그램 buffer로 전달
   │    → read()가 읽은 바이트 수와 함께 반환
   │
   └─ Cache miss
        → OS가 저장장치에 읽기 작업을 요청
        → 요청 스레드는 블로킹 대기
        → CPU는 다른 실행 가능한 작업 수행
        → 저장장치 완료 → Interrupt로 OS에 통지
        → OS가 대기 스레드를 runnable 상태로 변경
        → 스레드가 다시 실행되면 read()가 결과 반환
```

> 아래 4번과 5번은 이 순서에서 **Page Cache 확인 이후**에 일어나는 일을 단계별로 풀어 설명한다.

## 4. `read()`를 호출하면 OS는 무엇을 할까?

```c
ssize_t count = read(fd, buffer, 1024);
```

OS는 이 FD가 어떤 파일을 가리키는지와 현재 읽기 위치를 확인한 뒤, 필요한 바이트가 Page Cache에 있는지 먼저 확인한다.

### Cache hit: RAM에 데이터가 이미 있다

필요한 파일 데이터가 Page Cache에 있다면, OS는 저장장치 작업을 새로 시작하지 않아도 된다. 데이터를 프로그램이 요청한 버퍼로 전달하고 실제 읽은 바이트 수를 반환한다.

```text
read()
  → Page Cache hit
  → 프로그램 buffer에 데이터 전달
  → 읽은 바이트 수 반환
```

### Cache miss: 저장장치에 데이터를 요청해야 한다

필요한 데이터가 Page Cache에 없다면, OS는 파일 시스템을 통해 필요한 데이터를 찾아 저장장치에 읽기 작업을 맡긴다. 이때 장치 드라이버는 OS의 읽기 요청을 저장장치와 연결해 주는 커널 구성 요소다.
여기서는 장치별 명령이나 DMA 구현까지 알 필요는 없다. 중요한 것은 **파일 데이터가 아직 RAM에 없으므로, OS가 저장장치 작업이 끝나기를 기다려야 한다**는 점이다.

## 5. 저장장치가 읽는 동안 무슨 일이 일어날까?

일반적인 동기 `read()`에서 데이터가 아직 준비되지 않았다면, 요청한 스레드는 CPU를 계속 쓰며 확인하지 않는다. OS는 그 스레드를 블로킹(대기) 상태로 두고, 스케줄러는 다른 실행 가능한 스레드나 프로세스에 CPU를 할당한다.

```text
요청 스레드: 읽기 결과가 필요함
  → OS: 저장장치 작업 진행 중
  → 요청 스레드: 블로킹 대기
  → CPU: 다른 실행 가능한 작업 수행
```

> **블로킹은 CPU가 멈춘다는 뜻이 아니다.** 기다리는 스레드 하나가 실행을 멈출 뿐, CPU는 다른 요청과 다른 프로세스를 실행할 수 있다.
일반적인 인터럽트 기반 저장장치에서는 작업이 끝나면 **Interrupt**로 OS에 완료 사실을 알린다. OS는 완료된 I/O를 처리하고, 기다리던 스레드를 다시 실행 가능한 상태(runnable)로 바꾼다.

```text
저장장치 읽기 완료
  → Interrupt로 OS에 완료 통지
  → OS가 I/O 완료 처리
  → 대기 스레드를 runnable 상태로 변경
  → 스케줄러가 해당 스레드를 다시 실행
  → read()가 바이트 수와 함께 반환
```

runnable 상태가 되었다고 바로 CPU를 받는 것은 아니다. 스케줄러가 다시 선택했을 때 `read()`가 끝나고 프로그램이 다음 코드를 실행한다.

## 6. 왜 파일 I/O가 성능 문제로 이어질까?

파일 I/O가 항상 느린 것은 아니다. Page Cache hit이면 저장장치를 기다리지 않아 빠르게 끝날 수 있다. 문제는 Cache miss가 발생하거나 파일 작업이 오래 걸릴 때다.

```text
Cache miss 또는 느린 파일 I/O
  → 동기 read()를 호출한 요청 스레드가 대기
  → 동시에 많은 요청이 같은 대기를 반복
  → 제한된 요청 처리 스레드가 점유됨
  → 새 요청은 처리 시작을 기다림
  → 응답 시간 증가, 처리량 감소
```

따라서 서버 개발에서 외부 I/O를 항상 의식해야 한다.

- I/O가 **어디에서** 발생하는가?
- 동기 방식이라면 **어떤 스레드가 얼마나 오래** 대기하는가?
- 동시에 I/O 대기가 많아졌을 때 **스레드·커넥션·메모리 같은 자원**이 버틸 수 있는가?
- 자주 읽는 데이터는 캐시하거나, 요청 경로에서 파일 I/O를 줄일 수 있는가?

파일 I/O는 이 원리를 가장 선명하게 보여 주는 사례다. 같은 관점은 DB 호출, 외부 API, Redis, 메시지 브로커, S3에도 적용된다.

## 7. 파일과 Socket은 같은 `read()`를 써도 다르다

파일과 Socket은 모두 FD로 가리킬 수 있고 `read()`, `write()`, `close()` 같은 비슷한 인터페이스를 쓴다. 하지만 데이터가 준비되는 방식은 다르다.

| 자원 | 데이터를 얻는 방식 | 대기할 수 있는 대상 |
| --- | --- | --- |
| 일반 파일 | 이미 저장된 바이트를 특정 위치에서 조회 | Page Cache 또는 저장장치 |
| Socket | 상대 프로그램이 보낸 바이트를 수신 | 네트워크를 거쳐 도착할 데이터 |
| Pipe | 같은 컴퓨터의 다른 프로세스가 쓴 바이트를 수신 | Pipe 버퍼에 들어올 데이터 |

파일의 현재 읽기 위치가 끝에 도달하면 일반적으로 `read()`는 0(EOF)을 반환한다. 미래에 누군가 파일 끝에 내용을 추가할 때까지 Socket처럼 자동으로 기다리는 통신 자원은 아니다.

## 8. 복습

1. 서버 성능 문제는 외부 I/O의 지연과 자원 점유에 매우 밀접하다.
2. 파일 I/O는 외부 I/O의 한 종류이며, 프레임워크의 익숙한 기능 뒤에서도 발생할 수 있다.
3. 프로그램은 `open()`으로 FD를 얻고, `read()`로 OS에 파일 바이트를 요청한다.
4. OS는 먼저 Page Cache를 확인한다.
5. Cache hit이면 빠르게 결과를 반환할 수 있지만, Cache miss이면 저장장치 작업이 필요하다.
6. 동기 읽기에서는 요청 스레드가 블로킹 대기하고, CPU는 다른 실행 가능한 작업을 수행한다.
7. 저장장치 완료는 Interrupt로 OS에 알려지며, OS는 대기 스레드를 다시 실행 가능하게 만든다.
8. 파일 I/O를 이해하면 DB·외부 API·S3 같은 다른 외부 I/O의 성능 문제도 같은 관점으로 볼 수 있다.

## 다시 설명해볼 질문

- `open()`이 반환한 FD에는 파일 내용이 들어 있을까?
- Cache miss로 스레드가 기다릴 때 CPU도 멈출까?
- 애플리케이션이 JDBC로 DB를 조회하는 일은 어느 프로세스 기준에서 어떤 I/O일까?

## 참고 자료

- [Linux open(2)](https://man7.org/linux/man-pages/man2/open.2.html)
- [Linux read(2)](https://man7.org/linux/man-pages/man2/read.2.html)
- [Linux Kernel — Page Cache](https://www.kernel.org/doc/html/latest/mm/page_cache.html)
- [Java Files API — readString](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/file/Files.html#readString(java.nio.file.Path))
- [OpenJDK Files.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/nio/file/Files.java)
- [OpenJDK FileInputStream.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/io/FileInputStream.java)

## 정리 이력

- 2026-10-18: [노션 원문](https://app.notion.com/p/3d3cea2af9f4818c8364ebe34856542d)을 Markdown으로 옮겼다. 표를 변환하고, 외부 I/O의 설명 범위와 Linux 파일 읽기의 가정을 명시했다.
- 이 문서의 코드 조각은 흐름 설명용이며, 실행을 검증한 실습 코드는 아직 포함하지 않았다.
