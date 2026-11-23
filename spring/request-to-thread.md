# 요청은 어떻게 스레드 하나에게 갈까?

[목차](README.md) · [다음: 필터는 무엇을 걸러내고, Spring은 어디서부터일까?](filter.md)

## 다루는 범위

브라우저가 보낸 요청이 Spring 코드에 닿기 직전까지, 서버 입구에서 일어나는 일을 따라간다. Spring 요청 흐름 [0~10단계](README.md#전체-흐름) 중 0단계(연결)와 1단계(톰캣)다.

- nginx가 TLS를 풀고 톰캣에 넘기는 과정
- 톰캣 안의 세 일꾼: Acceptor, Poller, 작업 스레드
- "요청 하나에 스레드 하나"의 정확한 뜻 (연결 하나가 아니다)
- 스레드 풀과 기본 최대 200개
- Nest(Node)는 같은 자리에서 어떻게 다른가

스레드가 200개뿐이라서 생기는 문제는 [4장](transaction-and-db.md#5-연결을-들고-있는-시간과-외부-api)에서 다룬다. 이 문서의 코드와 숫자는 모두 설명용이다.

## 시작한 질문

`GET /api/health`를 처리하는 컨트롤러가 있다.

```java
@RestController
@RequestMapping("/api/health")
public class HealthController {
    @GetMapping
    public Map<String, String> health() {
        return Map.of("status", "ok");
    }
}
```

1. `health()`를 호출하는 코드가 어디에도 없는데, 요청이 오면 실행된다. 요청은 서버에 어떻게 도착하고, 누가 처리를 맡을까?
2. "요청 하나, 스레드 하나"라는데, 연결 하나마다 스레드가 하나 붙는다는 뜻일까?

## 내가 이해한 내용

### 1. nginx는 TLS를 풀고, 톰캣에는 새로 HTTP로 연결한다 (0단계)

사용자의 HTTPS 요청은 보통 톰캣으로 바로 가지 않고 nginx 같은 앞단을 먼저 지난다.

```
클라이언트 ──HTTPS──▶ nginx ──HTTP──▶ 톰캣(8080)
              (TLS를 여기서 푼다)   (새 연결, 평문)
```

- nginx가 인증서로 TLS 핸드셰이크를 하고 암호를 푼다(TLS 종료).
- 그다음 톰캣에는 평범한 HTTP로 **새 연결을 맺어** 요청을 넘긴다(리버스 프록시). 그래서 톰캣은 보통 TLS를 모른다.
- nginx는 이 밖에도 정적 파일 응답, 여러 서버로 나눠 보내기, 응답 대기 한도(`proxy_read_timeout`, 기본 60초) 같은 일을 맡는다.

### 2. Spring Boot 앱은 톰캣을 품은 자바 프로그램이다 (1단계)

`spring-boot-starter-web`을 쓰면 톰캣이 라이브러리로 들어 있다. `main`을 실행하면 내장 톰캣이 같이 켜지고 8080 포트에서 기다린다. 톰캣이 우리 앱을 실행하는 게 아니라, 우리 앱 안에서 톰캣이 돈다.

### 3. 톰캣 안의 세 일꾼 (NIO 커넥터)

| 일꾼 | 하는 일 |
| --- | --- |
| Acceptor | `accept()`로 새 연결을 받아서 Poller에게 등록한다 |
| Poller | 받아 둔 여러 연결을 Java NIO `Selector`로 한꺼번에 지켜보다가, 요청 데이터가 도착한 연결만 골라낸다. 리눅스에서 `Selector`는 OS의 `epoll`을 쓴다 |
| 작업 스레드 | Poller가 넘긴 연결에서 요청을 읽고, 응답을 보낼 때까지 끝까지 처리한다. 이름이 `http-nio-8080-exec-N`이다 |

요청을 맡은 작업 스레드는 그 뒤로 필터, DispatcherServlet, 내 컨트롤러, 서비스, DB까지 전부 혼자 지나간다. 이 노트 묶음에서는 그 스레드를 `exec-3`이라고 부르고 주인공처럼 따라간다.

### 4. 연결 하나가 아니라 요청 하나에 스레드 하나

HTTP/1.1은 keep-alive로 연결 하나를 여러 요청에 다시 쓴다. 그래서 연결은 오래 살아 있을 수 있다.

- 연결이 살아 있어도 **요청이 없으면 스레드를 붙잡지 않는다.** 그동안은 Poller가 지켜본다.
- 요청이 도착하면 그때 작업 스레드 하나가 맡아서 1~10단계를 혼자 처리하고, 응답을 보낸 뒤 스레드 풀로 돌아간다.
- 같은 연결로 다음 요청이 와도 다른 스레드가 맡을 수 있다. 그래서 스레드 이름을 찍어 보면 호출할 때마다 `exec-1`, `exec-2`, `exec-3`처럼 바뀐다.

옛날 BIO 방식은 연결마다 스레드가 붙어서 요청이 없어도 기다렸다. 지금의 NIO 방식은 기다리는 일을 Poller 하나에 맡기고, 작업 스레드는 실제 요청에만 쓴다.

### 5. 스레드 풀: 기본 최대 200개

작업 스레드는 요청마다 새로 만들지 않고 풀에 담아 두고 빌려 쓴다. Spring Boot의 톰캣 기본값은 이렇다.

| 설정 | 기본값 | 뜻 |
| --- | --- | --- |
| `server.tomcat.threads.max` | 200 | 작업 스레드 최대 개수 |
| `server.tomcat.threads.min-spare` | 10 | 늘 준비해 두는 스레드 |
| `server.tomcat.max-connections` | 8192 | 한 번에 붙잡아 둘 수 있는 연결 수 |
| `server.tomcat.accept-count` | 100 | 연결이 꽉 찼을 때 OS가 대신 줄 세워 두는 수 |

스레드 200개가 모두 일하는 중이면 새 요청은 빈 스레드가 날 때까지 기다린다. 요청 하나를 스레드 하나가 끝까지 붙잡는 구조라서, 한 요청이 오래 기다리면 그만큼 스레드가 묶인다. 이게 [4장](transaction-and-db.md#5-연결을-들고-있는-시간과-외부-api)의 외부 API 문제로 이어진다.

### 6. Nest(Node)와 비교

| | Spring MVC (톰캣) | Nest (Node) |
| --- | --- | --- |
| 요청을 처리하는 스레드 | 요청마다 작업 스레드 하나가 끝까지 맡는다 | 이벤트 루프 스레드 하나가 모든 요청을 나눠 처리한다 |
| I/O를 기다릴 때 | 그 스레드는 멈춰서 기다린다(블로킹) | `await` 동안 이벤트 루프는 다른 요청을 처리한다 |
| 약한 곳 | 오래 기다리는 요청이 쌓이면 스레드가 바닥난다 | CPU를 오래 쓰는 동기 코드 하나가 서버 전체를 멈춘다 |
| 연결 감시 | Poller가 `Selector`(epoll) | libuv가 epoll |

Nest는 Express(기본) 위에서 동작하는 프레임워크다. Express는 Node의 `http` 모듈 위에 있다. Spring Boot에서 톰캣이 하는 자리를 Node에서는 런타임과 `http` 모듈이 맡는다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| nginx와 톰캣이 둘 다 "요청을 받는 곳"이라 역할이 겹친다 | nginx는 바깥 문이다. TLS를 풀고 톰캣에 새 HTTP 연결로 넘긴다. 톰캣은 그 요청을 스레드에 맡기고 Spring을 부른다 |
| Poller는 OS의 `epoll`과 비슷한 것 같다 | 맞다. Poller는 Java NIO `Selector`로 여러 연결을 지켜보고, 리눅스에서 `Selector`가 `epoll`을 쓴다 |
| 요청 하나에 스레드 하나 = 연결 하나에 스레드 하나 | 아니다. 연결은 keep-alive로 살아 있어도, 스레드는 요청이 왔을 때만 쓴다. 기다리는 동안은 Poller가 본다 |
| 톰캣이 내 앱을 실행한다 | Spring Boot에서는 반대다. 내 앱의 `main` 안에서 내장 톰캣이 켜진다 |
| Nest가 Express를 대신한다 | Nest는 Express 위에서 동작한다. 기본 HTTP 어댑터가 Express다 |

### 공부하며 던진 질문

이 폴더는 문제 풀이 없이 발표 준비로 공부해서, 확인 문제 대신 실제로 물어본 것을 남긴다.

- **TLS는 누가 풀어?** 보통 nginx다. 톰캣은 nginx가 새로 맺은 평문 HTTP 연결로 요청을 받는다.
- **스레드 이름이 왜 호출할 때마다 바뀌어?** 풀에 있는 아무 빈 스레드나 요청을 맡기 때문이다. 같은 연결이어도 요청마다 다를 수 있다.
- **Nest라면 이 자리는?** 이벤트 루프 하나가 모든 요청을 나눠 처리한다. "요청 하나 = 스레드 하나"가 아니라 "스레드 하나 = 요청 전부"다.

## 예제와 확인 결과

설명용이다. 직접 실행하지 않았다.

```java
@GetMapping("/api/health")
public Map<String, String> health() {
    log.info(Thread.currentThread().getName());
    return Map.of("status", "ok");
}
```

```
http-nio-8080-exec-1
http-nio-8080-exec-2
http-nio-8080-exec-3
```

```yaml
server:
  tomcat:
    threads:
      max: 200        # 작업 스레드 최대
      min-spare: 10
    max-connections: 8192
    accept-count: 100
```

## 남은 질문

- Java 21 가상 스레드(`spring.threads.virtual.enabled=true`)를 켜면 "요청 하나 = 스레드 하나"와 200개 제한은 어떻게 달라질까?
- HTTP/2는 연결 하나에 여러 요청이 동시에 흐른다. 톰캣은 이걸 작업 스레드에 어떻게 나눌까?
- Spring WebFlux(Netty)는 Node의 이벤트 루프와 어디가 같고 어디가 다를까?

## 참고 자료

이번 공부는 발표("요청 하나, 스레드 하나", 2026-11-24)를 준비하며 Claude와 대화로 진행했다. 더 읽어볼 공식 문서:

- [Apache Tomcat - HTTP Connector 설정](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)
- [Spring Boot - 내장 웹 서버](https://docs.spring.io/spring-boot/how-to/webserver.html)

## 정리 이력

- 2026-11-23: 발표 준비로 공부한 내용을 노트로 정리했다.
