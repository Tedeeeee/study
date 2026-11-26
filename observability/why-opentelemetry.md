# APM이 있는데 OTel은 왜 나왔을까?

[목차](README.md) · [이전: 세 signal은 어떻게 이어질까?](connecting-signals.md) · [다음: API·SDK·Collector는 각각 무슨 일을 할까?](api-sdk-collector.md)

## 다루는 범위

OTel이 나오기 전의 문제(APM 벤더 종속, 서로 다른 헤더·전송 양식), 먼저 나온 표준 시도인 OpenTracing과 OpenCensus, 둘을 합친 OTel이 표준으로 정한 것과 **정하지 않은 것**을 정리한다.

그 전에 OTel을 이루는 부품(계측 모듈, API, SDK, Exporter)을 먼저 정리한다. 6장의 문제들이 결국 "이 부품들이 누구 것이냐"의 문제라서다. 부품별 자세한 역할은 7장에서 다룬다.

## 시작한 질문

Datadog, Pinpoint 같은 APM이 이미 잘 돌아가는데, 왜 OpenTelemetry라는 표준이 또 필요했을까?

## 내가 이해한 내용

### 1. 먼저 부품: 계측 모듈 → API → SDK → Exporter

| 부품 | 하는 일 | JPA 쪽 비유 |
| --- | --- | --- |
| 계측 모듈 | Tomcat, JDBC, HTTP 클라이언트처럼 **남이 만든 라이브러리 안에서** 언제·무엇을 잴지 정하고 API를 호출한다 | `repository.save()`를 부르는 서비스 코드 |
| API | 호출 창구(인터페이스) | JPA 인터페이스 |
| SDK | 호출을 받아 실제 signal 데이터로 만든다. ID 생성, 샘플링, Metric 집계, Exemplar, Resource | Hibernate |
| Exporter | 밖으로 보낸다(OTLP 등) | JDBC 드라이버 |

- API만 넣고 SDK가 없으면 OTel API는 **no-op**으로 동작한다. 에러 없이 아무것도 안 나온다. 라이브러리가 OTel API로 계측해 둬도, OTel을 안 쓰는 앱에서는 비용 없이 넘어가게 하려는 설계다.
- 계측 모듈이 없으면 내가 직접 `spanBuilder`를 쓴 곳에만 Span이 생긴다. Tomcat이나 JDBC 안에 내가 코드를 넣을 수는 없기 때문이다.
- **Java Agent = API + SDK + Exporter + 많은 계측 모듈 + 자동 조립**을 jar 하나로 묶은 것이다. 계측 모듈은 클래스를 불러올 때 바이트코드를 고쳐서 끼워 넣는다.
- **law-erp(Node)는 부품을 골라서 직접 조립**했다. SDK, 계측 모듈 4개(Http, Express, Pg, RuntimeNode), signal별 OTLP HTTP Exporter 3개를 설치하고 `telemetry.ts`에서 조립했다.

### 2. OTel 이전: APM은 부품이 한 회사 것으로 묶인 세트였다

| 부품 | APM 시절 |
| --- | --- |
| 계측 모듈 | A사 Agent 안 |
| API·SDK | A사 전용 |
| 데이터 형식·전송 | A사 형식으로 A사 서버에만 |
| 저장·화면 | A사 서비스 |

여기서 생긴 문제는 세 가지다.

1. **벤더 종속.** 업무 구간을 A사 전용 API(`import com.vendora...`)로 계측해 두면, 회사를 바꿀 때 그 코드를 전부 고쳐야 한다. Agent 자동 계측만 썼다면 코드는 그대로지만 Agent, 설정, 대시보드는 다시 해야 한다. log4j를 직접 import하던 SLF4J 이전 시절과 같다.
2. **서비스마다 도구가 다르면 Trace가 끊긴다.** 헤더 양식이 달라서 받는 쪽이 못 읽고 새 Trace를 시작한다.
3. **라이브러리 개발자가 계측을 못 넣는다.** 어느 회사 API로 넣을지 정할 수 없다. 그래서 APM 회사마다 남의 라이브러리용 계측 모듈을 따로 만들어 유지했다.

### 3. "보내는 곳"은 두 군데이고, 둘 다 규격이 없었다

```text
           ① 서비스 → 서비스 (헤더)
 order-service ───────────────▶ payment-service
      │ ② 서비스 → 저장소 (데이터)        │
      ▼                                  ▼
   Jaeger 서버                       Zipkin 서버
```

**① 헤더.** 담는 정보(trace_id, span_id)는 같아도 **헤더 이름과 값 양식이 달랐다.** 주소를 한국식과 미국식으로 쓰는 것과 비슷하다.

```text
Jaeger:  uber-trace-id: abc:123:0:1
W3C:     traceparent:   00-abc-123-01
B3:      X-B3-TraceId: abc / X-B3-SpanId: 123 / ...  (헤더 여러 개)
```

그래서 회사 안에서 한 양식으로 통일하거나, 여러 양식을 동시에 보내거나, 받는 쪽이 여러 양식을 읽게 하는 식으로 **서로 맞추는 수고**가 필요했다. B3가 사실상의 표준처럼 쓰였지만 공식 표준은 아니었다. trace_id 길이도 64비트와 128비트가 섞여 있었다.

**② 전송.** 저장소마다 받는 양식이 달라서, 목적지에 맞는 라이브러리나 Exporter(번역기)를 따로 써야 했다. 저장소를 바꾸면 다시 맞춰야 했다.

### 4. 먼저 나온 표준 시도: OpenTracing과 OpenCensus

| 항목 | OpenTracing | OpenCensus |
| --- | --- | --- |
| 시작 | 2016년쯤, CNCF | 2018년쯤, Google |
| signal | Trace만 | Trace + Metric |
| 제공 | **API만** | **API + SDK + Exporter** |
| 장점 | 코드가 회사에서 떨어진다 (구현체를 바꿔도 코드 그대로) | 넣으면 바로 쓸 수 있다 |
| 한계 | 구현체마다 헤더 양식·전송 형식이 다르다. inject/extract **동작**만 정했고 헤더 **양식**은 안 정했다 | OpenTracing 생태계와 따로 논다 |

두 API는 **서로 호환되지 않았다.** 라이브러리 A는 OpenTracing, B는 OpenCensus로 계측돼 있으면 한 Trace로 깔끔하게 안 이어졌다. 표준을 만들려다 선택지가 하나 더 늘어난 셈이다.

OpenCensus처럼 SDK가 있으면 그 안에는 ID 규칙과 헤더 양식이 있다. 다만 **그 라이브러리 안에서만 통하는 규칙**이었고 업계 공통 규칙은 아니었다.

### 5. OTel: 둘을 합치고 공통 규칙을 더했다

2019년쯤 두 프로젝트가 합쳐 OpenTelemetry가 됐다(CNCF). OpenTracing의 API 분리와 OpenCensus의 구현체 제공을 둘 다 가져왔고, 나중에 Log도 추가했다.

| OTel이 표준으로 정한 것 | 예 | 자세히 |
| --- | --- | --- |
| API, SDK | `tracer.spanBuilder(...)` | 7장 |
| 헤더 양식 (경로 ①) | W3C `traceparent`가 기본 | 3장 |
| 전송 양식 (경로 ②) | OTLP | 8장 |
| 이름 규칙 (semconv) | `http.request.method`, `service.name` | 8장 |
| 계측 모듈, Collector | Agent, receiver/exporter | 9~10장 |

예전 시스템과 섞여 있으면 `OTEL_PROPAGATORS=tracecontext,b3`처럼 여러 헤더 양식을 같이 쓰게 할 수도 있다.

"하나의 통로"라는 건 **길이 하나**가 아니라 **양식이 하나**라는 뜻이다. 같은 OTLP 데이터를 여러 백엔드로 보낼 수 있고, 어디로 가든 받는 쪽이 알아듣는다.

### 6. OTel이 안 하는 것: 저장과 화면

```text
[OTel]                                   [백엔드]
계측 → API·SDK → Exporter ──OTLP──▶  저장 · 검색 · 화면 · 알림
                                       Jaeger, Tempo, Prometheus, Grafana,
                                       Datadog, law-erp의 monitoring 서버 ...
```

OTel은 데이터를 만들고 보내는 데까지만 한다. 그래서 APM을 없앤 게 아니라, APM 회사들은 **OTLP를 받아 주는 백엔드**가 됐다. 계측은 OTel로 한 번 하고, 백엔드는 골라서 바꾼다.

| 2절의 문제 | OTel 이후 |
| --- | --- |
| 회사를 바꾸면 코드를 다시 고친다 | OTel API로 한 번 옮겨 두면, 이후엔 Exporter 설정만 바꾼다 |
| 서비스마다 도구가 달라 Trace가 끊긴다 | 모두 같은 헤더 양식(`traceparent`)을 쓴다 |
| 라이브러리 개발자가 계측을 못 넣는다 | OTel API로 넣는다 (SDK가 없으면 no-op) |

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| SDK는 signal을 측정하는 구현체다 | "측정"을 둘로 나눠야 한다. **언제·무엇을 잴지**는 계측 모듈이 정하고, SDK는 그 호출을 받아 **signal 데이터로 만들고 관리**한다. 보내기는 Exporter다. |
| law-erp는 SDK만 설치했다 | SDK, 계측 모듈 4개, Exporter 3개를 골라 설치하고 직접 조립했다. Agent는 "다 켜 놓은 완제품"이고 law-erp는 "직접 조립한 것"이다. |
| API, SDK, Exporter만 있으면 되지 않나? | 계측 모듈이 없으면 남의 라이브러리(Tomcat, JDBC) 안에서 Span을 만들어 줄 쪽이 없다. Agent의 진짜 편리함은 계측 모듈을 알아서 붙여 주는 것이다. |
| APM이 바뀌면 계측 도구에 맞게 바꾼다 | 바꿀 건 도구 하나가 아니라 세트 전체다. **직접 쓴 계측 코드까지** 고쳐야 한다. |
| 구현체가 다르면 뭐가 문제인지 와닿지 않는다 | SLF4J 비유로 이해했다. APM 시절 = log4j 직접 import, OpenTracing = SLF4J처럼 코드는 같지만 구현체마다 헤더·전송 양식이 달라 서로 못 알아본다. |
| SDK와 Exporter가 있으면 trace_id 규격이 이미 있는 거 아닌가? | 있었다. 다만 그 라이브러리 안에서만 통하는 규칙이었다. ID 규칙, 헤더 양식, 전송 양식 세 층 모두 업계 공통 규칙이 없었다. |
| 규격은 같고 key(헤더 이름)만 다르다 | key뿐 아니라 **value 양식**(구분자, 칸 순서·개수, ID 길이)도 달랐다. 처음 설명을 너무 자세히 해서 이해가 안 됐고, 주소 양식 비유로 다시 이해했다. |
| 그럼 보내는 서버가 받는 쪽 양식에 맞춰 보내야 했나? | 맞다. 통일, 여러 양식 동시 전송, 여러 양식 읽기 같은 방법으로 서로 맞추는 수고를 했다. |
| 계측 도구에 보낼 때 규격이 없어서 도구마다 맞췄다 | 맞다. 다만 "보내는 곳"은 **서비스 → 서비스(헤더)**와 **서비스 → 저장소(데이터)** 두 군데다. OTel은 각각 `traceparent`와 OTLP로 표준을 정했다. |

### 확인 문제

**SDK 확인 문제.** 동료가 OTel **API** 의존성만 넣고 `tracer.spanBuilder("주문 처리").startSpan()`을 썼는데 Jaeger에 아무것도 안 나온다. ① 왜 그런가 ② 해결 방법 두 가지

내 답:

> 1. Otel 의 API 만 추가한건 JPA를 넣기만 하고 실제 hibernate 를 사용하지 않는것과 같아...? sdk 를 넣어야 실제 api 의 구현체를 사용할 수 있는거지
> 2. SDK까지 같이 넣어서 실제로 동작하게 하거나 Agent 를 설치해서 쉽게 사용한다?

채점: 통과. 보충하면 JPA와 달리 OTel API는 구현체가 없으면 에러 대신 no-op으로 조용히 넘어간다. SDK 방법은 넣기만 하면 부족하고, OTLP Exporter까지 넣고 앱 시작 때 SDK를 초기화해 전역으로 등록해야 한다(law-erp의 `telemetry.ts`가 이 작업이다).

**Q1.** order-service는 A사 APM(업무 구간을 A사 전용 API로 계측), payment-service는 Jaeger(OpenTracing)를 쓴다. order → payment Trace가 끊기고, A사를 그만 쓰려니 코드를 다 고쳐야 한다. ① 각 문제는 왜 생겼나 ② OTel로 옮기면 어떻게 풀리나 ③ 저장과 화면도 OTel이 해 주나

내 답:

> 1. 둘이 계측 도구가 달라서 Trace id의 값을 담는 이름이 다르다 그래서 받는 서비스가 자기가 아는 trace id 이름이 안오니까 새로운 요청이라고 생각해서 트레이스가 끊겨서 하나의 요청으로 이어지지 않았던거지
> 2. 하나의 규격 그리고 각 계측 도구의 규격이 같아지면서 계측 방식을 따로 고민하지 않아도 괜찮다
> 3. OTel 은 각 서비스 혹은 다른 계측 도구마다 번거롭게 코드와 계측 도구 설정 방식을 귀찮게 하는걸 방지해준거지 뭘 보여주는것이 아니라서 기존의 계측 모니터링 도구는 그대로 사용하면 된다

채점: 3번은 맞았다. 1번은 Trace가 끊긴 이유는 맞았지만(이름뿐 아니라 값 양식도 다르다) **코드를 고쳐야 하는 이유**가 빠졌다. 2번은 방향은 맞지만 어떤 규격이 어느 문제를 푸는지가 없었다. `traceparent`라는 이름까지는 필요 없다고 정리했고, 코드 수정 부분만 다시 물었다.

**Q1 재시도.** A사 APM을 그만 쓰려는데 왜 코드를 다 고쳐야 하고, OTel로 옮기면 다음번엔 왜 안 고쳐도 될까?

내 답:

> 아 APM 을 그만 사용하고 싶으면 현재 서비스에서 해당 APM의 규격에 맞는 호출을 변경하려는 APM 혹은 계측 도구에 맞게 전부 변경해야하는데 OTel 은 다른 계측 도구의 공통 규격이라서 이후 뭐가 변경되어도 괜찮다는거지

채점: 통과. "뭐가 변경되어도"는 정확히는 **백엔드가 바뀌어도** 코드는 그대로이고 Exporter 설정만 바꾼다는 뜻이다. A사 API에서 OTel API로 옮기는 처음 한 번은 코드를 고쳐야 한다.

## 예제와 확인 결과

- 이 문서의 A사·B사 코드와 헤더 값은 구조를 보여 주는 설명용이다. 헤더 이름(`uber-trace-id`, `X-B3-TraceId`, `traceparent`)과 `io.opentracing.Tracer`는 실제 이름이다.
- law-erp 서버 코드를 읽어 확인했다(실행하지 않음).
  - `package.json:83~85`: `@opentelemetry/exporter-{logs,metrics,trace}-otlp-http`
  - `telemetry.ts:82, 92, 107`: Metric, Trace, Log Exporter가 각각 `${base}/v1/metrics`, `/v1/traces`, `/v1/logs`로 보낸다.
  - Metric은 `PeriodicExportingMetricReader`, Trace와 Log는 `BatchSpanProcessor`, `BatchLogRecordProcessor`가 모아서 보낸다. 세 Provider 모두 같은 `resource`를 받는다.
  - `metric-exporter.ts`: `OTLPMetricExporter`를 상속해 ObservableGauge만 DELTA로 보낸다. 누적으로 두면 끝난 요청의 마지막 값이 계속 보내져 처리 중으로 남기 때문이다. 직접 조립하면 이런 세부를 바꿀 수 있다는 예다.

## 남은 질문

- OTel API와 SDK를 왜 굳이 나눴고, no-op은 정확히 어떻게 동작하나? (7장)
- Agent jar 안의 구성(API, SDK, 계측 모듈, 클래스로더 분리)은 실제로 어떻게 되어 있나? (10장)
- OpenTracing, OpenCensus 시작 연도와 OTel 합병 시점, W3C Trace Context 권고 시점을 원문으로 확인하기
- law-erp `telemetry.ts`의 `kind`(API, TIMER 등)에 따라 Tracer와 Meter를 나누는 이유

## 참고 자료

- [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar)
- [OpenTelemetry — What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)
- [OpenTelemetry — History](https://opentelemetry.io/docs/what-is-opentelemetry/#history)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry — Propagators 설정](https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_propagators)
