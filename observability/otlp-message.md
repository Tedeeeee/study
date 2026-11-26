# OTLP 메시지는 어떻게 생겼을까?

[목차](README.md) · [이전: API·SDK·Collector는 각각 무슨 일을 할까?](api-sdk-collector.md) · [다음: Collector는 데이터를 어떻게 처리할까?](collector-pipeline.md)

## 다루는 범위

OTLP 메시지의 3단 구조(Resource → Scope → 데이터), Span 안에 붙는 Span Event와 LogRecord의 차이, 4317(gRPC)과 4318(HTTP) 두 가지 운반 방식, 그리고 속성 이름을 맞추는 semconv를 정리한다. 중간에 gRPC, protobuf, HTTP/2가 무엇인지도 OTel에 필요한 만큼만 정리한다.

## 시작한 질문

앱이 Collector로 보내는 OTLP 데이터는 실제로 어떤 모양일까? 포트는 왜 두 개이고, `http.request.method` 같은 이름은 누가 정했을까?

## 내가 이해한 내용

### 1. OTLP 메시지는 3단 상자다

```text
Resource      누가 보냈나            service.name=order-service
 └─ Scope     어느 계측 모듈이 만들었나  io.opentelemetry.tomcat-10.0
     └─ Span  실제 데이터             GET /orders, 120ms
 └─ Scope                            io.opentelemetry.jdbc
     └─ Span                         SELECT orders, 30ms
```

| 칸 | 담는 것 |
| --- | --- |
| Resource | 데이터를 만든 **앱** (`service.name`, version, host) |
| Scope | 데이터를 만든 **계측 모듈**의 이름과 버전 |
| 데이터 | Span, Metric, LogRecord |

Span이 1,000개여도 Resource와 Scope는 위에 한 번씩만 적으니 메시지가 작아진다. Scope는 이상한 Span이 보일 때 "누가 이걸 만들었지?"를 찾는 데 쓴다.

```json
{
  "resourceSpans": [{
    "resource": { "attributes": [{ "key": "service.name", "value": "order-service" }] },
    "scopeSpans": [{
      "scope": { "name": "io.opentelemetry.tomcat-10.0" },
      "spans": [{ "traceId": "4bf9...", "spanId": "00f0...", "name": "GET /orders" }]
    }]
  }]
}
```

signal이 셋이라 맨 위도 셋이고, 한 번 보낼 때는 한 종류만 담는다.

| signal | 맨 위 | HTTP 경로 |
| --- | --- | --- |
| Trace | `resourceSpans` | `/v1/traces` |
| Metric | `resourceMetrics` | `/v1/metrics` |
| Log | `resourceLogs` | `/v1/logs` |

law-erp의 Exporter 3개가 각각 이 경로로 보내던 이유다. `resourceSpans`가 배열인 건 Collector처럼 여러 서비스 데이터를 한 번에 보낼 때 서비스별로 칸이 여러 개 들어가기 때문이다.

### 2. Span Event와 LogRecord는 다른 길로 간다

**Span Event**는 Span 안에 "이 시점에 이런 일이 있었다"를 적는 기록이다.

```json
"events": [
  { "time": "14:03:07.500", "name": "재고 확인 완료" },
  { "time": "14:03:08.200", "name": "exception",
    "attributes": { "exception.type": "PaymentTimeoutException",
                    "exception.message": "PG 응답 3초 초과",
                    "exception.stacktrace": "..." } }
]
```

만드는 방법은 두 가지다.

| Span Event | 어떻게 |
| --- | --- |
| 예외 | **자동.** 예외가 밖으로 던져지면 Agent 계측 모듈이 `recordException(e)`을 부른다 |
| 이정표 | **직접.** `Span.current().addEvent("쿠폰 적용")` |
| 잡아서 처리한 예외 | 남기고 싶으면 직접 `span.recordException(e)` |

예외 메시지는 SDK가 지어내는 게 아니다. `throw new XxxException("PG 응답 3초 초과")`처럼 우리가 적은 문장을 SDK가 `e.getMessage()`로 꺼내 `exception.message`에 넣는다. 그래서 예외 메시지에 개인정보를 넣으면 그대로 Trace 저장소로 간다. law-erp의 `serverLog`가 예외 메시지를 일부러 안 보낸 이유와 같다.

SDK 안에서는 Span 객체의 `events` 목록에 한 줄씩 추가되고, **Span이 끝날 때 통째로** Processor → Exporter로 넘어간다. 그래서 Span이 끝나기 전엔 안 보이고, Span이 샘플링에서 빠지면 Event도 사라지고, 개수 상한(기본 128개로 알고 있음)이 있다.

`log.*`는 Span Event가 되지 않는다. 항상 LogRecord 길로 간다.

```text
log.error("결제 실패", e) ──▶ Logback ─┬─▶ 콘솔 (우리가 보는 그 줄)
                                       └─▶ OTel appender → LogRecord → resourceLogs

throw e (밖으로) ──▶ Agent → recordException(e) → Span Event → resourceSpans
```

| | Span Event | LogRecord |
| --- | --- | --- |
| 만드는 코드 | `span.addEvent()`, `recordException()`, Agent 자동 | `log.info()`, `log.error()` |
| 담기는 곳 | Span 안 (`resourceSpans`) | 따로 (`resourceLogs`) |
| 보는 곳 | Trace 화면에서 Span을 눌러서 | 콘솔, 로그 저장소 |
| Span 없이 | 의미 없음 | 존재할 수 있음 |
| Trace가 샘플링에서 빠지면 | 같이 사라짐 | 남음 |

그래서 같은 예외가 두 군데 다 남을 수 있다. Trace 화면에서는 "이 Span에서 터졌다"를 보고, 로그 저장소에서는 "오늘 결제 실패가 몇 건인가"를 검색한다. 둘은 `trace_id`로 이어진다(5장).

### 3. 4317과 4318: 내용은 같고 나르는 방식만 다르다

| 포트 | 방식 | 본문 |
| --- | --- | --- |
| 4317 | gRPC | protobuf |
| 4318 | HTTP (`POST /v1/traces` 등) | protobuf 또는 JSON |

Collector가 두 대인 게 아니다. **Collector 하나가 문 두 개를 열어 둔 것**이고, 어느 문으로 들어오든 같은 OTLP 데이터로 처리한다.

```yaml
# 설명용
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }
```

| | gRPC (4317) | HTTP (4318) |
| --- | --- | --- |
| 장점 | 작고 빠르다 | 프록시, 로드밸런서, 방화벽을 잘 통과한다 |
| 꼭 이것만 | | 브라우저(JS는 gRPC를 직접 못 씀) |

주소에 `/v1/...` 경로가 붙어 있으면 HTTP 방식이다. law-erp는 `exporter-*-otlp-http`와 `${base}/v1/traces`를 쓰므로 HTTP 방식이다.

### 4. gRPC, protobuf, HTTP/2

| 이름 | 무엇인가 | 비유 |
| --- | --- | --- |
| protobuf (Protocol Buffers) | Google이 만든 **데이터 형식**. 결과물이 바이너리다 | JSON 같은 것 |
| HTTP/2 | HTTP의 두 번째 버전. **나르는 길** | 한 길에 여러 차선 |
| gRPC | Google이 만든 **통신 규칙**. protobuf로 적고 HTTP/2로 나른다 | REST 같은 것 |

> gRPC ≈ protobuf(데이터 모양) + HTTP/2(나르는 길) + 원격 메서드 호출 규칙

- **protobuf**는 설계도(`.proto`)에 필드마다 번호를 붙이고, 보낼 때 이름 대신 번호만 바이너리로 적어서 작다. 받는 쪽도 같은 설계도가 있어야 한다. OTel은 OTLP 설계도를 공개해 모두 같은 걸 쓴다.
- **HTTP/2**는 의미(GET, 헤더, 상태 코드)는 1.1과 같고 나르는 방식만 다르다. 연결 하나에서 요청마다 스트림 번호를 붙이고, 요청을 프레임(HEADERS, DATA)으로 쪼개 섞어 보낸 뒤 받는 쪽이 번호로 다시 맞춘다. 그래서 줄을 안 선다. 담는 내용은 상관없어서 REST + JSON도 HTTP/2로 많이 다닌다.
- Span 여러 개를 한 요청에 **묶는 건** gRPC 덕분이 아니라 SDK의 Batch Processor가 한다. HTTP로 보내도 묶어서 보낸다.

### 5. semconv: 같은 뜻이면 같은 이름

양식이 같아도 속성 이름이 제각각(`method`, `httpMethod`, `http_verb`)이면 같은 쿼리로 못 찾는다. **semconv(Semantic Conventions)**는 OTel이 정한 이름 약속이다. 예를 들면 `service.name`, `http.request.method`, `http.response.status_code`, `http.route`, `exception.type`, `http.server.request.duration`(단위: 초)이 있다. 외울 목록이 아니라, 필요할 때 문서에서 찾고 Agent가 대부분 알아서 붙인다.

경계는 이렇다.

| 층 | 정하는 쪽 | 성격 |
| --- | --- | --- |
| 상자 구조와 고정 칸 (`resourceSpans`, `traceId`, `name`, `attributes`, `events`) | OTLP | **반드시** 지킨다. 안 지키면 받는 쪽이 못 읽는다 |
| `attributes` 안의 key 이름 | semconv | **권장**이다. 다른 이름도 전송은 되지만 대시보드와 쿼리가 못 알아본다 |
| `traceparent` 헤더 | W3C | |
| `coupon.id`, `lawerp.http.server.active_requests` | 우리 | semconv에 없으면 직접 짓는다. 겹치지 않게 접두어를 붙이는 게 관례다 |

> OTLP는 서류 양식, semconv는 양식의 자유 칸에 적는 용어 사전이다.

semconv도 버전이 바뀌어 왔다(`http.method` → `http.request.method`). 그래서 같은 요청인데 계측에 따라 이름이 다르게 나올 수 있다(11장).

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| `resourceSpans`, `resourceMetrics`, `resourceLogs` 세 가지가 대표적인가? | signal이 셋이라 셋이다. 한 번에 한 종류만 담아 `/v1/...` 경로로 따로 보낸다. |
| Span을 다룰 때 log가 들어갈 수 있다고 들었는데 그게 뭐지? | Span Event다. Span 안의 시점 기록이라 로그와 비슷해 보이지만 LogRecord와는 다른 길로 간다. |
| 로그는 대부분 비즈니스에서 직접 찍기보다 `@RestControllerAdvice`에서 찍는다 | 에러 로그는 대부분 그렇다. 업무 흐름이나 경고 로그는 비즈니스 로직에서도 찍는다. Advice도 같은 요청 스레드라 MDC의 trace_id가 그대로 찍힌다. 다만 Advice가 예외를 잡아 버리면 Agent가 exception 이벤트를 자동으로 안 남길 수 있다(미확인). |
| `log.*`로 찍은 것과 Span Event는 다르게 찍히나? | 완전히 다른 길이다. `log.*`는 LogRecord, `span.*`은 Span Event다. 서로 알아서 바뀌지 않는다. |
| Span Event는 예외 기록이니 메시지는 없겠다 | 있다. 예외 객체의 `getMessage()`를 SDK가 꺼내 `exception.message`에 넣는다. 예외를 던질 때 우리가 적은 문장이다. |
| Span Event는 알아서 쓰는 건가? | 예외는 Agent가 알아서 남긴다. 그 밖의 이정표는 `addEvent()`로 직접 넣는다. |
| 포트를 다르게 하면 Collector 서버가 2개인가? | Collector 하나가 문 두 개를 연 것이다. |
| gRPC 안에 protobuf가 있나? HTTP 같은 프로토콜인가? | gRPC는 HTTP/2 위의 통신 규칙(프로토콜)이고, protobuf는 데이터 형식이라 프로토콜이 아니다. |
| protobuf가 바이너리라는 뜻인가? | protobuf는 이름(형식)이고, 바이너리는 그 결과물의 성질이다. |
| "바이너리 조각"이 어렵다 | HTTP/2는 요청을 번호 붙은 상자(프레임)로 쪼개 한 길에 섞어 보내고 받는 쪽이 번호로 맞춘다. OTel 공부에는 "4317은 빠른 길, 4318은 평범한 길, 내용은 같다"면 충분하다. |
| gRPC는 바이너리라서 한 번에 많이 보낸다? | 장점이 둘이다. protobuf라 작고, HTTP/2라 동시에 여럿 보낸다. 여러 Span을 한 요청에 묶는 건 SDK Batch Processor다. |
| gRPC 없이 HTTP/2만 있으면 JSON을 그대로 쓰나? | 그렇다. HTTP/2는 길일 뿐이라 JSON도 HTML도 실린다. |
| semconv 표가 갑자기 나와서 당황했다 | 외울 목록이 아니라 "이름을 정해 둔 약속이 있다"는 예시였다. |
| 지금까지 나온 key 이름은 전부 semconv인가? | 속성과 메트릭 이름은 semconv다. 상자 구조는 OTLP, `traceparent`는 W3C, `coupon.id`는 우리가 지은 것이다. |
| `resourceSpans` 같은 타입은 OTLP이고 그 안의 필드명은 semconv다? | 한 칸 더 안쪽이다. `traceId`, `name`, `attributes` 같은 고정 칸도 OTLP다. semconv는 `attributes` 안의 key 이름만 정한다. OTLP가 크게 양식을 정하고, 그 안의 상세 이름을 semconv가 잡는다. |

### 확인 문제

**Q1.** 동료가 묻는다. ① Span이 1,000개인데 `service.name`이 왜 맨 위에 한 번만 있고, Span마다 다른 `scope`는 뭔가 ② 4317과 4318로 보내는 건 Collector가 두 대라서인가, 내용이 다른가 ③ `http.request.method`와 `coupon.id`는 누가 정한 이름인가

내 답:

> 1. span 은 한 번 저장될때 한개씩 저장하면 효율이 떨어져서 name 을 기반으로 해당 서비스에서 발생한 span 을 모아주는 역할을 해 그리고 scope는 해당 span이 어떤 이유 server 혹은 db 이런곳에서 시작하면 span 이 새로 시작하기 때문에 scope가 있어야 해
> 2. 아 2대는 아니고 우리는 17로 gRPC 기반으로 데이터를 전송하는거야 18은 기존의 우리가 아는 HTTP 방식으로 전달하는거고 보내는 내용은 똑같은데 보내는 데이터의 방식이 조금 달라
> 3. 이건 semconv 라고 약속을 정한거야

채점: 통과. 1번의 Resource를 한 번만 적는 이유는 맞았다. Scope는 "server냐 db냐"와 거의 겹치지만 기준은 **어느 계측 모듈이 만들었나**다(Tomcat 계측이 server Span을, JDBC 계측이 db Span을 만들어서 겹쳐 보인다). 2번은 맞았다. 3번은 `http.request.method`는 semconv가 맞지만 `coupon.id`는 우리가 직접 지은 이름이다.

## 예제와 확인 결과

- 이 문서의 JSON, Java 코드, YAML은 모두 설명용이다. 실행해 확인하지 않았다.
- law-erp 코드에서 확인한 것: 패키지 `exporter-*-otlp-http`와 `${base}/v1/traces` 등 경로 사용으로 HTTP 방식임을 확인했다. 실제 포트(4318 여부)는 `base` 값(.env)을 열지 않아 확인하지 않았다.

## 남은 질문

- `@RestControllerAdvice`가 잡아서 처리한 예외가 SERVER Span에 exception 이벤트로 남는가? Agent와 Spring 버전별로 다른가? (10~11장 실습)
- Span Event 개수 상한의 기본값은 정말 128인가?
- 새 이벤트를 Span Event 대신 Log 쪽으로 기록하자는 OTel의 방향 논의는 현재 어떻게 됐나?
- Agent의 기본 OTLP 프로토콜은 gRPC인가 HTTP인가? (10장)
- HTTP/2의 더 깊은 동작과 HTTP/3는 `network/` 주제로 따로 다룰 만하다.

## 참고 자료

- [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar)
- [OpenTelemetry — OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)
- [opentelemetry-proto (OTLP `.proto` 정의)](https://github.com/open-telemetry/opentelemetry-proto)
- [OpenTelemetry — Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OpenTelemetry — Exceptions semconv](https://opentelemetry.io/docs/specs/semconv/exceptions/)
- [OpenTelemetry — Traces (Span Events)](https://opentelemetry.io/docs/concepts/signals/traces/#span-events)
- [gRPC](https://grpc.io/), [Protocol Buffers](https://protobuf.dev/)
