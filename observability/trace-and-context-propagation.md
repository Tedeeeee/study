# Trace와 Span은 어떻게 생겼고, 서비스 경계를 어떻게 넘을까?

[목차](README.md) · [이전: Metric·Log·Trace는 각각 어떤 질문에 답할까?](signals-and-cardinality.md) · [다음: Metric은 무엇으로 잴까?](metric-instruments.md)

## 다루는 범위

Span 하나에 무엇이 들어 있고, Span들이 어떻게 나무(Trace)가 되는지 본다. 이어서 서버를 넘을 때 `traceparent` 헤더로, 서버 안에서는 ThreadLocal로 그 나무가 이어지는 과정을 따라간다. 마지막으로 `@Async`처럼 스레드가 바뀔 때 끊기는 이유와 잇는 방법을 정리한다.

로그에 trace_id를 찍어 세 signal을 잇는 방법은 5번 주제에서 다룬다. Java Agent와 Spring Boot 내장 계측이 실제로 어떤 Span을 만드는지 비교하는 것은 10~11번 주제에서 다룬다.

## 시작한 질문

1장에서 "trace_id를 HTTP 헤더에 실어 보내면 서버를 넘어서도 이어진다"고 정리했다. 그 헤더는 실제로 어떻게 생겼을까? 그리고 받은 쪽은 그걸로 무엇을 만들고, 같은 서버 안에서는 어떻게 이어 갈까?

## 내가 이해한 내용

### 1. Span은 작업 하나, Trace는 같은 trace_id를 가진 Span 묶음

| 필드 | 뜻 |
| --- | --- |
| `trace_id` | 이 Span이 속한 요청(Trace)의 ID, 32자리 16진수 |
| `span_id` | 이 작업 자체의 ID, 16자리 16진수 |
| `parent_span_id` | 나를 부른 작업의 span_id. 없으면 root Span |
| `name` | 작업 이름 (`POST /orders`, `SELECT stock`) |
| 시작·끝 시각 | 걸린 시간 |
| `attributes` | 잘라 볼 속성 (`http.route`, 2장의 `user.id` 같은 고카디널리티 값도 여기) |
| `status` | `OK`, `ERROR`, `UNSET`(오류 아님) |
| kind | `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, `INTERNAL` |

Trace라는 데이터가 따로 있는 게 아니다. **trace_id가 같은 Span들을 parent_span_id로 이으면** 나무가 된다. 부모 시간에는 자식 시간이 포함되므로 Span 시간을 단순히 더하지 않는다.

### 2. Span은 계측이 심어진 곳에서 생긴다

메서드마다 자동으로 생기지 않는다.

| 누가 만드나 | 어디서 | kind |
| --- | --- | --- |
| 자동 계측 | 요청을 받을 때 | `SERVER` |
| 자동 계측 | 다른 서버, DB, Redis, 외부 API를 부를 때 | `CLIENT` |
| 자동 계측 | 메시지를 보낼 때 / 받을 때 | `PRODUCER` / `CONSUMER` |
| 내가 직접 (`@WithSpan`, `@Observed` 등) | 의미 있는 업무 단위 (할인 계산, 결제 승인) | `INTERNAL` |

직접 만들 때 기준은 "**장애가 났을 때 이 칸의 시간을 따로 보고 싶을까?**"다. Span도 요청마다 쌓이는 비용이고, 너무 잘게 쪼개면 범인을 못 찾는다.

모놀리식이면 나무가 대부분 `SERVER` 하나와 `CLIENT` 여러 개(DB, Redis, 외부 API)로 생긴다. 같은 모양의 DB Span이 수백 개 늘어서 있으면 N+1을 바로 알아볼 수 있다.

### 3. `traceparent`: 서버를 넘을 때 쓰는 표준 헤더

W3C Trace Context가 정한 형식이다.

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             버전 ──────────── trace-id ──────── ─── parent-id ── flags
```

| 칸 | 뜻 |
| --- | --- |
| version | 지금은 `00` |
| trace-id | 요청 전체 ID. 끝까지 그대로 간다 |
| parent-id | **헤더를 보내는 쪽의 현재 Span ID.** 받는 쪽의 `parent_span_id`가 된다 |
| flags | `01` 샘플링됨, `00` 안 됨. 2장의 "샘플링 결정은 첫 서비스가 하고 따라간다"의 실체 |

표준이 생기기 전에는 도구마다 헤더가 달랐다. Zipkin은 `X-B3-*`, Jaeger는 `uber-trace-id`, Datadog은 `x-datadog-*`를 썼다. 그래서 서로 다른 도구를 쓰는 서비스 사이에서는 헤더를 못 알아봐 Trace가 끊겼다. 선택 헤더로 벤더별 정보를 싣는 `tracestate`도 있다.

### 4. 서버를 한 번 넘을 때 Span은 2개 생긴다

부르는 쪽이 `CLIENT` Span을, 받는 쪽이 `SERVER` Span을 만든다. 헤더의 parent-id는 **부르는 쪽의 `CLIENT` Span**이다.

```text
[SERVER] order: POST /orders          span=O   parent=G     3000ms
 └─ [CLIENT] order → inventory        span=OC  parent=O     2850ms  → 헤더 00-abc-OC-01
      └─ [SERVER] inventory: GET /stock  span=I parent=OC   2800ms  ← 새 span_id
           └─ [CLIENT] SELECT stock   span=…   parent=I     2700ms
```

| 값 | 서버를 넘을 때 |
| --- | --- |
| trace_id | 그대로 |
| 헤더의 parent-id | 보내는 쪽의 현재 Span(`CLIENT`)으로 매번 바뀐다 |
| 받는 쪽 span_id | 새로 만든다 |

두 Span의 시간 차이(2850 − 2800 = 50ms)는 네트워크, 대기열, 커넥션 획득 같은 "사이" 시간이다. DB나 PG사처럼 상대가 계측하지 않는 곳은 `CLIENT` Span만 남는다.

나무는 Trace 전체에서 하나다. 서비스마다 자기 `SERVER` Span에서 시작하는 **서브트리**가 붙는다. root Span(parent 없음)은 맨 처음 요청을 받은 Span 하나뿐이다. 중간 서비스의 `SERVER` Span에 parent가 없다면 새 Trace가 시작된 것이다.

### 5. 헤더 읽기·싣기는 계측 코드가, 부모 찾기는 ThreadLocal이 한다

| 역할 | 하는 일 | 누가 |
| --- | --- | --- |
| extract | 헤더를 읽어 부모를 복원하고 `SERVER` Span을 만든다 | 서버 쪽 계측 |
| makeCurrent | 그 Span을 현재 스레드의 ThreadLocal에 넣는다 | 서버 쪽 계측 |
| 부모 찾기 | ThreadLocal의 현재 Span을 부모로 `CLIENT` Span을 만든다 | JDBC, Redis 등 각 계측 |
| inject | 현재 Span을 `traceparent`로 바꿔 싣는다 | HTTP 클라이언트 계측 |
| 형식 변환 | Span ↔ `traceparent` 문자열 | Propagator |

서버 쪽 계측이 끼는 위치는 두 가지다.

- **Spring Boot 내장(Micrometer):** Spring이 만든 관측용 서블릿 필터(`ServerHttpObservationFilter`)를 Boot가 Tomcat 필터 체인에 등록한다. Tomcat은 실행만 한다.
- **OTel Java Agent:** 필터를 등록하지 않는다. 앱이 뜰 때 Tomcat과 서블릿 클래스의 바이트코드를 고쳐 끼워 넣는다.

```java
// 설명용 의사 코드: 서버 쪽 계측이 하는 일
parent = propagator.extract(request.getHeader("traceparent"));
span = startSpan(SERVER, "GET /orders", parent);
try (scope = span.makeCurrent()) {   // ThreadLocal에 넣기
    chain.doFilter(request, response); // 내 컨트롤러, DB 호출이 이 안에서 실행
} finally {
    span.end();
}                                     // try를 나가면 ThreadLocal 원래대로
```

ThreadLocal에 들어가는 건 헤더 값이 아니라 **새로 만든 `SERVER` Span**이다. 그래야 DB Span이 그 Span을 부모로 삼는다. Tomcat 스레드는 풀에서 재사용되므로 요청이 끝나면 반드시 비운다.

> **서버 사이는 헤더로, 서버 안에서는 ThreadLocal로 이어진다.**

Spring Boot 내장 계측은 Spring이 아는 객체만 감싼다. 그래서 `RestClient.Builder`나 `RestTemplateBuilder` 빈 대신 `new RestTemplate()`으로 만들면 헤더가 안 실린다.

### 6. 스레드가 바뀌면 끊긴다: `@Async`

`@Async` 메서드는 Executor의 다른 스레드에서 실행된다. 그 스레드의 ThreadLocal은 비어 있다. 그래서 거기서 부른 RestClient 계측은 현재 Span을 못 찾고 **부모 없는 root Span을 새 trace_id로** 만든다.

Executor에 넘어가는 건 `Runnable` 하나뿐이다. 풀 스레드는 누가 작업을 넘겼는지 모른다. `InheritableThreadLocal`은 스레드를 **새로 만들 때만** 복사해서 풀에는 소용이 없다. 그래서 **작업을 넘기는 순간에 Context를 같이 포장하는 로직**이 필요하다.

```java
// 설명용: 원리가 보이도록 직접 구현한 TaskDecorator
public class TraceContextTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable task) {
        Context captured = Context.current();          // 스레드 A, 지금 실행
        return () -> {                                 // 상자만 만들어 반환
            try (Scope scope = captured.makeCurrent()) { // 스레드 B, 나중에 실행
                task.run();
            }                                          // 끝나면 비우기
        };
    }
}
```

- `TaskDecorator`는 Spring의 인터페이스다. 이 클래스는 그걸 구현한 설명용 예시다.
- 람다 바깥 코드는 **스레드 A에서 지금** 실행된다. 람다 안쪽은 **스레드 B에서 나중에** 실행된다. `captured` 변수를 람다가 붙잡아 B까지 들고 간다.
- 원래 작업을 같은 타입(`Runnable`)으로 감싸 앞뒤에 동작을 덧붙인다. 이게 데코레이터 패턴이다.

| | 서버를 넘을 때 | 스레드를 넘을 때 (같은 JVM) |
| --- | --- | --- |
| 왜 끊기나 | 메모리를 공유하지 않는다 | ThreadLocal은 스레드 전용이다 |
| 무엇을 넘기나 | `traceparent` **문자열** | Context **객체** |
| 언제 담나 | HTTP 요청을 보낼 때 | 작업을 Executor에 넘길 때 |

실무에서는 직접 만들지 않는다.

| 계측 | 방법 |
| --- | --- |
| Spring Boot 내장 | `ContextPropagatingTaskDecorator`를 `TaskDecorator` 빈으로 등록 (Spring 6.1+) |
| OTel API | `Context.taskWrapping(executorService)` |
| OTel Java Agent | 흔한 Executor, `@Async`, `CompletableFuture`를 자동으로 감싼다 |

비동기 자식 Span은 부모보다 늦게 끝날 수 있다. 화면에서 막대가 부모 밖으로 삐져나와도 정상이다.

## 헷갈렸던 점

- **`traceparent`가 trace_id만 넘기는 게 아니라 "형식 약속"이라는 게 새로웠다.**
  - trace_id만 넘기면 받는 쪽은 같은 요청인 건 알아도 누가 불렀는지(parent)와 기록할지(flags)를 모른다.
  - 도구마다 헤더가 달라 끊기던 문제를 W3C가 통일했다.
- **"받으면 그 서비스 안에서 span id를 다시 만드냐"고 물었다.**
  - 맞다. 받은 parent-id는 내 부모로 쓰고, 내 span_id는 새로 만든다. trace_id만 끝까지 그대로다.
- **"Span을 만드는 기준이 뭐냐"고 물었다.**
  - 계측이 심어진 곳이다. 자동 계측은 들어오고 나가는 경계에, 직접 계측은 의미 있는 업무 단위에 만든다.
- **"모놀리식이면 Span은 SERVER와 DB 호출 두 종류뿐이겠다"고 생각했다.**
  - 거의 맞다. 다만 PG사, 소셜 로그인, Redis, S3 같은 외부 호출도 `CLIENT` Span이다.
  - DB Span이 여러 개라는 점이 오히려 N+1을 찾는 단서가 된다.
- **"그럼 다른 서버를 부를 때 호출하는 쪽이 새 Span을 만든다는 거 아니냐"고 짚었다.**
  - 맞았다. 앞 설명이 단순화되어 있었다. 서버를 넘을 때마다 부르는 쪽 `CLIENT`와 받는 쪽 `SERVER`로 Span이 2개 생기고, 헤더의 parent-id는 `CLIENT` Span이다.
- **"트리 안의 트리냐"고 물었다.**
  - Trace 나무는 하나다. 서비스마다 `SERVER` Span에서 시작하는 서브트리가 붙고, root는 하나뿐이다.
- **"서블릿 필터 같은 계측 코드"가 무슨 말인지 몰랐다. 그 뒤엔 "톰캣이 가진 필터를 이용하는 거냐"고 물었다.**
  - Tomcat이 가진 필터가 아니다. **Spring이 만든 관측 필터를 Boot가 Tomcat 필터 체인에 끼워 넣고**, Tomcat은 실행만 한다.
  - Java Agent는 필터 대신 바이트코드에 끼운다.
- **"필터에서 넘어온 헤더 데이터를 ThreadLocal에 넣는 거냐"고 정리했다.**
  - 흐름은 맞다. 다만 들어가는 건 헤더 값이 아니라 그걸 부모로 **새로 만든 `SERVER` Span**이다.
  - 풀 스레드라 요청이 끝나면 비운다.
- **`@Async` 문제의 첫 답에서 스레드 사이에도 `traceparent` 헤더를 넣으면 된다고 생각했다.**
  - 헤더는 메모리가 다른 서버 사이의 도구다. 같은 JVM에서는 Context 객체를 넘긴다.
  - 넣는 시점도 메서드 안이 아니라 Executor에 넘기는 순간이다. 메서드가 B에서 시작될 땐 이미 꺼낼 게 없다.
- **"스레드가 달라서 context를 넣어주는 로직이 필요해지는 거냐"고 물었다.**
  - 맞다. Executor에는 `Runnable`만 넘어가고, Java는 ThreadLocal을 다른 스레드로 복사해 주지 않는다.
  - `@Transactional`이 `@Async`로 안 넘어가는 것과 같은 이유다.
- **TaskDecorator 코드가 어려웠다. "Executor로 한 요청에서 스레드 2개를 다루는 방법이냐"고 물었다.**
  - 스레드를 2개로 만드는 건 `@Async`와 Executor다. TaskDecorator는 작업 상자를 한 번 더 감싸는 포장지일 뿐이다.
  - 헷갈린 진짜 이유는 한 메서드 안에서 람다 바깥(A, 지금)과 안쪽(B, 나중)이 다른 스레드, 다른 시점에 실행된다는 점이었다.
  - `TaskDecorator`는 Spring의 인터페이스이고, 예시 구현체는 설명용으로 직접 만든 것이다. 실무에서는 `ContextPropagatingTaskDecorator`를 쓴다.
  - 이름은 데코레이터 패턴에서 왔다. 같은 타입으로 감싸 원래 코드와 사용하는 코드를 안 고치고 앞뒤에 동작을 덧붙인다. 다형성과 OCP 얘기로 잠깐 샜는데, 모니터링과는 별개라 여기서는 이 정도만 남긴다.

### 확인 문제

**문제 1.** inventory가 `traceparent: 00-abc-OC-01` 헤더를 받았다. (1) inventory의 첫 Span의 `trace_id`, `span_id`, `parent_span_id`와 kind는? (2) 이어서 DB를 조회할 때, 인자로 Span을 넘기지 않았는데 DB Span은 부모를 어떻게 알아내나?

- 내 답:
  1. order가 호출할 때 헤더에 trace id와 span id를 실어 보내고, 그걸 기반으로 inventory가 또 span id를 만든다. 요청이 들어온 것이라 `SERVER`로 생성된다.
  2. 하나의 요청은 하나의 스레드로 진행되고, 스레드의 ThreadLocal에 span이 들어 있어서 부모의 span id를 알 수 있다.
- 채점: 정답.
  - 1번은 값을 콕 집어야 했다. `trace_id = abc`는 그대로 쓰고, `span_id`는 새로 만들고, `parent_span_id = OC`는 order의 `CLIENT` Span이다.
  - 2번에서 ThreadLocal에 있는 건 부모의 span id(OC)가 아니라 inventory가 만든 `SERVER` Span(I)이다. DB Span은 I를 부모로 삼는다.

**문제 2.** 알림 발송을 `@Async`로 바꿨더니 알림 서버 호출이 별도의 새 Trace로 보인다. (1) 왜 새 Trace가 됐나? (2) 원리적으로 어떻게 이어야 하나?

- 내 답:
  1. `@Async`는 다른 스레드를 할당받아 작업하는데, trace id는 ThreadLocal에 들어가서 스레드에 고유하다. 서로 공유하지 않아 기존 흐름을 알아낼 수 없다.
  2. 해당 메서드 안에 trace id를 넣어줄 수 있는 traceHeader 같은 걸 넣어줘야 한다.
- 채점: 1번 정답, 2번 절반 정답.
  - 1번에는 "RestClient 계측이 현재 Span을 못 찾아 root Span을 새로 만들었다"를 덧붙이면 왜 **새** Trace인지까지 설명된다.
  - 2번은 넘겨야 한다는 방향은 맞았다. 하지만 헤더는 서버 사이의 도구이고, 넣는 시점도 메서드 안이 아니라 Executor에 넘기는 순간이다.

**문제 2 다시.** 같은 상황에서 (1) 새 Trace가 된 이유를 "현재 Span을 찾는 과정"으로, (2) 언제·무엇을·어떻게 옮기는지를 `traceparent`와 비교해 설명하라.

- 내 답:
  1. `@Async`는 다른 스레드에 일을 던지는데, trace id와 span id는 한 스레드의 ThreadLocal에 묶인다. 스레드가 바뀌어 공유되지 않으니 새 trace id가 발급되어 새로 시작한다.
  2. Executor에서 context를 넘겨받을 수 있게 해야 한다. `TaskDecorator`를 상속받아 기존 스레드의 context를 꺼내 넣는 작업을 하고, config에서 executor에 넣어 특정 executor를 만든다. 그리고 `@Async`에 파라미터로 그 executor를 지정한다.
- 채점: 정답.
  - trace id는 스레드가 발급받는 게 아니다. 계측이 ThreadLocal에서 부모를 못 찾아 root Span을 만들면서 생긴다.
  - `TaskDecorator`는 인터페이스라 "상속"보다 "구현"이 맞다.
  - 꺼내고 넣는 것에 더해 **끝나면 비우기**까지 해야 한다.
  - `traceparent`와의 차이가 빠졌다. 서버 사이는 문자열 헤더, 같은 JVM의 스레드 사이는 Context 객체를 그대로 넘긴다.

## 예제와 확인 결과

이번 주제는 개념 정리라서 직접 실행한 예제는 없다. trace_id `abc`, span id `G`/`O`/`OC`/`I`, 시간 수치, TaskDecorator와 AsyncConfig 코드는 모두 설명용이다.

`ServerHttpObservationFilter`, `ContextPropagatingTaskDecorator`(Spring 6.1+), Boot 3.2+에서 `TaskDecorator` 빈을 자동 구성 Executor에 붙여 준다는 점은 알고 있던 내용으로 설명했다. 세미나 코드나 실행으로 확인하지는 않았다.

## 남은 질문

- 세미나 실습 앱(Boot 내장 계측)에서 필터 체인에 관측 필터가 실제로 보일까? Java Agent로 띄우면 안 보일까? → 10번 주제
- Spring Boot 버전별로 `@Async`가 설정 없이 전파되는지, 어떤 설정이 필요한지 → 10번 주제 실습
- Kafka처럼 HTTP가 아닌 메시지는 `traceparent`를 어디에 실을까? 메시지 헤더일까?
- 로그에 trace_id를 찍는 MDC도 ThreadLocal인데, `@Async`에서 같이 옮겨질까? → 5번 주제
- 1장의 남은 질문 "`@Async`에서 컨텍스트를 어떻게 옮겨 담나"는 원리(TaskDecorator로 포장)까지 답했다.

## 참고 자료

- OpenTelemetry 세미나 1회차 "OpenTelemetry 아키텍처 & OTLP 데이터 모델" — 분산 Trace, Span 구조 ([VSFe/otel-seminar](https://github.com/VSFe/otel-seminar))
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry — Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry — Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Spring Framework — Observability Support](https://docs.spring.io/spring-framework/reference/integration/observability.html)
