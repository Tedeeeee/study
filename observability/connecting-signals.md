# 세 signal은 어떻게 이어질까?

[목차](README.md) · [이전: Metric은 무엇으로 잴까?](metric-instruments.md)

## 다루는 범위

Metric, Log, Trace를 서로 오가게 만드는 세 가지 연결 방법을 정리한다.

| 연결 | 키 | 묶는 단위 |
| --- | --- | --- |
| Log ↔ Trace | 로그에 찍힌 `trace_id`, `span_id` | 요청 하나 |
| Metric → Trace | Exemplar에 적힌 `trace_id` | 요청 하나(골라 둔 예시) |
| 세 signal 전체 | Resource의 `service.name` | 서비스 하나 |

기준 환경은 세미나와 같은 **Java + Logback + OTel Java Agent**다. Agent 없이 SDK를 직접 붙이는 경우와 Node 예시(law-erp)는 비교용으로만 적는다. Agent·Boot 내장·Starter의 구조 차이는 10장에서 다룬다.

## 시작한 질문

대시보드에서 p99가 튄 것을 봤다. 그 숫자에서 **실제로 느렸던 요청**의 Trace와, 그 요청이 남긴 로그까지 어떻게 찾아갈까?

## 내가 이해한 내용

### 1. signal은 관측 데이터의 종류다

OTel은 Metric, Log, Trace를 묶어 **signal**이라고 부른다. 시스템 상태를 바깥으로 알려 주는 단서라는 뜻이다. Profile이 네 번째 signal로 개발 중이지만, 지금 범위에서는 세 개만 본다.

세 signal은 따로 만들어지고 따로 저장된다. 그래서 "잇는다"는 건 **같은 키를 양쪽에 적어 두고, 그 키로 찾아간다**는 뜻이다.

### 2. Log ↔ Trace: 스레드 안의 두 저장소와 복사

요청을 처리하는 스레드에는 저장소가 두 개 있다. 둘 다 ThreadLocal이라는 같은 장치를 쓰지만, 주인과 칸이 다르다.

| 저장소 | 주인 | 들어 있는 것 | 누가 읽나 |
| --- | --- | --- | --- |
| OTel Context | OTel | 현재 Span(trace_id, span_id) | OTel 계측 코드 |
| MDC | Logback(SLF4J) | 로그에 같이 찍을 key-value | Logback 패턴의 `%X{...}` |

MDC는 스레드가 원래 가진 것이 아니라 **Logback이 ThreadLocal 위에 마련한 자기 저장소**다. Logback은 OTel Context를 모른다. 그래서 trace_id를 로그에 찍으려면 **OTel Context → MDC로 복사**해야 한다.

```text
요청 도착 → 계측 코드가 SERVER Span 생성 → OTel Context에 현재 Span으로 올림
          → (Agent) trace_id·span_id를 MDC에 넣음
          → log.info("주문 생성") → Logback이 %X{trace_id}로 MDC에서 꺼내 찍음
```

이 복사는 **OTel Java Agent**가 해 준다. Agent가 Logback 쪽에도 계측을 끼워 넣기 때문에 Logback 설정을 따로 고치지 않아도 trace_id가 MDC에 들어간다. 패턴에 `%X{trace_id}`만 있으면 콘솔에 찍힌다.

Agent 없이 SDK만 직접 붙이면 Trace는 잘 만들어져도 MDC가 비어 있다. 이때는 MDC 연동 라이브러리를 추가하는 식으로 복사할 주체를 직접 정해야 한다.

Agent는 이와 별개로 **OTel 로그 appender**도 붙인다. 로그 한 줄을 OTel LogRecord(trace_id·span_id 필드 포함)로 바꿔 OTLP로 보낸다. 로그를 만드는 건 여전히 Logback이고, appender는 그 출력을 OTel 쪽으로도 흘려보내는 출구다.

| 경로 | trace_id가 들어가는 곳 | 필요한 것 |
| --- | --- | --- |
| 콘솔·파일 로그 | 로그 문자열 안 | MDC 복사 + 패턴의 `%X{trace_id}` |
| OTLP로 보내는 로그 | LogRecord의 trace_id 필드 | OTel 로그 appender |

> 확인 수준: Agent가 기본으로 쓰는 MDC 키 이름과 로그 내보내기 기본값은 버전마다 다를 수 있어 직접 확인하지 않았다.

### 3. Metric → Trace: Exemplar

Metric은 집계된 숫자라서 "어떤 요청이 3초 걸렸나"는 남아 있지 않다. 시간대만 보고 Trace 저장소를 뒤지는 건 추측이다. **Exemplar**는 이 빈틈을 메운다.

Exemplar는 Metric 데이터 포인트에 붙는 **예시 메모**다. 값을 `record()`할 때 현재 Span이 있으면 그 요청의 정보를 적어 둔다.

| 메모에 적는 것 | 예 |
| --- | --- |
| 값 | 3.2초 |
| 시각 | 14:03:08 |
| trace_id, span_id | `abc...`, `00f0...` |

```text
Metric 저장소:  http.server.duration 버킷 ... exemplar { 3.2초, trace_id: abc }
                                                             │ 주소만
Trace 저장소:   trace abc → [SERVER] → [CLIENT] SELECT …      ◀┘ 본체는 여기
```

알아 둘 성질은 다음과 같다.

- **주소만 적는다.** Trace 본체는 Trace 저장소에 그대로 있다.
- **일부만 적는다.** 그 Metric에 기여한 모든 요청이 아니라 SDK가 고른 몇 개뿐이다. Histogram은 기본으로 버킷마다 최근 것 하나 정도를 남긴다.
- **라벨이 아니다.** 데이터 포인트 옆의 메모라서 시계열 개수가 늘지 않는다. trace_id를 라벨로 붙이면 요청마다 시계열이 생겨 카디널리티가 폭발한다(2장).
- **샘플링과 다르다.** Trace 샘플링은 첫 서비스에서 "이 Trace를 남길까"를 정하는 일이다(3장). Exemplar는 그렇게 **남은 Trace 중에서** 고르는 것이라 순서상 샘플링이 먼저다.
- Counter에도 붙일 수 있지만, 느린 요청을 찾는 Histogram에서 가장 쓸모가 크다.

한계도 있다. 데이터 포인트마다 몇 개만 남고, 가리키는 Trace가 전송 실패나 보관 만료로 없을 수 있고, 저장소와 화면이 Exemplar를 지원해야 한다.

> 확인 수준: "버킷마다 최근 것 하나"는 OTel 명세의 기본 Exemplar 저장 방식으로 알고 있는 내용이다. SDK마다 세부는 다를 수 있다.

### 4. 세 signal 전체: Resource와 `service.name`

trace_id는 요청 하나를 묶는다. "order-service의 에러율, 로그, Trace를 다 보고 싶다"처럼 **서비스 단위**로 묶을 때는 Resource를 쓴다.

Resource는 **이 데이터를 누가 만들었나**를 적은 꼬리표다. 한 앱(프로세스) 안의 SDK가 내보내는 Metric, Log, Trace에 같은 Resource가 붙는다.

```text
Resource { service.name=order-service, service.version=1.2.0, deployment.environment=prod }   ← 고정
   ├─ Metric: http.server.duration  시각=14:03:10  값=...
   ├─ Log:    "결제 실패"  시각=14:03:08  trace_id=abc
   └─ Trace:  abc  [SERVER] 14:03:07.100 ~ 14:03:08.300
```

| 위치 | 담는 것 | 바뀌는 주기 |
| --- | --- | --- |
| Resource | 누가 (`service.name`, version, environment, `service.instance.id`, host) | 앱이 켜져 있는 동안 고정 |
| 각 데이터 | 언제, 무엇이 (시각, 값, 속성) | 데이터마다 다름 |

시간은 Resource에 들어가지 않는다. Resource는 편지 봉투의 보내는 사람이고, 시각은 편지 안 각 내용에 적힌 날짜다. 전송할 때도 Resource는 묶음 위에 한 번만 적는다(8장 OTLP 구조).

Java Agent에서는 보통 환경변수로 정한다.

```bash
OTEL_SERVICE_NAME=order-service
OTEL_RESOURCE_ATTRIBUTES=service.version=1.2.0,deployment.environment=prod
```

서비스 이름이 비거나 서비스마다 제각각이면 세 signal이 같은 서비스인지 알 수 없다. 그러면 대시보드에서 Metric → Log → Trace로 넘어가는 연결이 끊긴다. `service.instance.id`는 같은 서비스의 서버 여러 대를 구분한다.

> 확인 수준: 이름을 안 정했을 때의 기본값은 명세상 `unknown_service:java` 형태로 알고 있다. 최신 Agent가 `spring.application.name`이나 jar 이름으로 추정하는지는 직접 확인하지 않았다.

### 5. 세 연결을 합친 분석 흐름

```text
Metric: p99가 3초로 튐 (service.name=order-service)
  → Exemplar의 trace_id로 실제 느린 요청의 Trace 열기
  → Span에서 느린 SQL 확인
  → 같은 trace_id로 로그 검색 → 그 요청의 에러 로그 확인
  → 필요하면 service.name으로 같은 서비스의 다른 Metric·로그로 넓혀 보기
```

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| MDC는 ThreadLocal에 있는 거 아닌가? | 맞다. 다만 스레드가 원래 가진 게 아니라 Logback이 ThreadLocal 위에 마련한 **자기 저장소**다. OTel Context도 같은 장치를 쓰는 **별도의 칸**이다. |
| 복사 방향이 MDC → OTel인가? | Logback은 OTel을 모르니까 **OTel Context → MDC**로 복사해야 로그에 찍힌다. |
| 예전에 Logback 설정 없이 Agent만 붙였는데 잘 됐던 것 같다 | 기분 탓이 아니다. Agent가 MDC 복사를 해 줘서 그랬다. 다만 패턴에 `%X{trace_id}`가 있어야 콘솔에 보인다. |
| law-erp 서버도 Agent로 붙인 줄 알았다 | 코드를 같이 보니 law-erp는 **Node(NestJS)에 OTel SDK를 `telemetry.ts`에서 직접 조립**한 구조였다. Agent가 아니다. 그래서 Agent 기준 설명과 안 맞아서 헷갈렸다. |
| SDK와 Agent는 완전히 다른 건가? | 안에서 도는 건 같은 SDK다. 차이는 **누가 조립하느냐**다. Agent는 실행할 때 자동으로 끼워 넣고, 직접 붙이면 코드로 조립한다. law-erp가 직접 조립한 이유도 "모니터링 서버로 보내서"는 아니다. 보내는 곳(OTLP 엔드포인트)과 조립 방식은 별개다. |
| Exemplar는 Trace를 저장하는 건가? | trace_id라는 **주소만** 적는다. 본체는 Trace 저장소에 있다. |
| Exemplar는 그 Metric에 쓰인 Trace 전부인가? | **일부만** 고른다. Histogram은 기본으로 버킷마다 최근 것 하나 정도다. |
| signal이 뭐지? | Metric, Log, Trace 같은 관측 데이터의 **종류**를 부르는 이름이다. |
| Resource에 시간도 들어가나? | 안 들어간다. Resource는 앱이 켜져 있는 동안 고정된 "누가"이고, 시각은 데이터 하나하나에 붙는다. |

### 확인 문제

**Q1.** Spring Boot + Logback + Java Agent 환경에서 패턴 `%d %-5level [%X{trace_id}] %msg%n`으로 trace_id가 찍혔다. ① trace_id는 어디서 와서 어떻게 찍혔나(저장소 이름과 복사 방향) ② 누가 해 줬나

내 답:

> 1. 기존에 threadLocal 에 MDC와 OTel 로 저장소가 만들어지고 OTel 은 SDK로 인해 측정이 되면 그걸 MDC로 복사하는 과정을 거치게 됩니다. 그렇게 해서 OTel 에서 사용하고 있는 trace id 를 같이 볼 수 있습니다
> 2. 누가...?? OTel 이?

채점: 1번은 저장소 두 개와 OTel → MDC 방향을 맞혔다. 다만 trace_id가 OTel Context에 올라가는 시점은 "측정될 때"가 아니라 **SERVER Span이 현재 Span이 될 때**이고, Logback이 `%X{trace_id}`로 MDC에서 꺼내 찍는 부분이 빠졌다. 2번은 범위가 너무 넓었다. 답은 **OTel Java Agent**다. SDK만 쓰면 복사를 직접 챙겨야 하므로 "OTel이 알아서"는 틀린 답이 될 수 있다.

**Q1 재시도.** Agent를 빼고 SDK만 직접 붙였더니 Trace는 잘 보이는데 로그에는 `[]`로 찍힌다. ① 왜 비었나 ② Agent를 쓸 때는 왜 괜찮았나

내 답:

> 1. 기존에 Agent 가 threadLocal 에 있는 OTel 의 트레이스 정보를 찍어서 MDC로 넘겨주는 상황이였는데 그걸 진행해줄 애가 없어서 로그를 찍는 logback이사용하는 MDC에 데이터가 없음면서 콘솔에는 찍지 못한다
> 2. Agent 가 span 을 찍을때 MDC 영역에 복사를 하는데 그 작업을 하는 애를 빼버려서

채점: 통과. 보충하면, Agent는 Span을 만들 때보다 **Logback이 로그 이벤트를 만들 때 끼어들어** MDC에 넣는 방식으로 알고 있다(코드로 확인하지는 않음).

**Q2.** p99가 3초로 튄 점을 눌렀더니 Trace로 바로 넘어갔다. ① 그걸 가능하게 한 건 무엇이고 무엇을 얼마나 붙였나 ② trace_id를 라벨로 붙이면 왜 안 되나 ③ 세 signal을 서비스 단위로 묶는 키는 무엇이고 어디에 있나

내 답:

> 1. sampling 이다. 각 메트릭을 측정할때 통계적인면만 보여줄수 있기 때문에 그때 발생한 요청을 알려면 개발자는 시간을 추측해서 직접 가서 하나씩 찾아볼 수 밖에 없어서 불편하기 때문에 메트릭을 저장할때 트레이스 id 를 샘플링 해두면 그때쯤 발생한 트레이스들을 보고 유추를 좀 더 빨리 할 수 있다
> 2. trace id를 그냥 붙이면 이전에 말햇던 카디널리티가 높아지면서 시계열 데이터가 급상승하게 되고 또 성능에 문제가 발생한다
> 3. resource 를 보면 된다. 모든 signal은 resource 를 기반으로 모두 묶이기 때문에 그걸 알 고 있으면 분석하기 편해진다. 그 안에 service.name 이라는 것으로 묶어준다

채점: 2, 3번은 맞았다. 1번은 필요한 이유와 "trace_id를 일부만 붙인다"는 내용은 맞았지만 이름이 **Exemplar**다. "샘플링"은 Trace를 남길지 정하는 다른 개념이라 섞으면 안 된다. 또 Exemplar는 "그때쯤 발생한 Trace들"이 아니라 **그 값을 실제로 기록한 요청 하나**를 가리킨다. 이름만 틀린 것이라 재시도 없이 넘어갔다.

## 예제와 확인 결과

- 이 문서의 로그 패턴, 환경변수, 다이어그램은 설명용이다. 세미나 실습 환경에서 직접 실행해 확인하지는 않았다.
- 비교용으로 law-erp 서버 코드를 읽기만 했다(실행하지 않음). 확인한 사실은 다음과 같다.
  - Node(NestJS)에서 `telemetry.ts`가 OTel SDK를 직접 조립한다. Agent가 아니다.
  - `AsyncLocalStorageContextManager`를 쓴다. Java의 ThreadLocal 역할을 Node에서 하는 장치다.
  - `serverLog`가 콘솔 출력과 별도로 `logs.getLogger('server-log').emit(...)`으로 OTel LogRecord를 직접 내보낸다. Java의 MDC 복사 + appender 역할을 코드로 하는 셈이다.
  - Resource는 `resourceFromAttributes({'service.name': serviceName, 'service.instance.id': randomUUID()})`로 정한다. 실행할 때마다 instance id가 새로 생긴다.

## 남은 질문

- Java Agent가 MDC에 넣는 키 이름과 로그 내보내기 기본값은 버전별로 어떻게 다른가?
- `OTEL_SERVICE_NAME`을 안 주면 최신 Agent는 이름을 어떻게 정하나? (10장 실습에서 확인)
- 세미나 환경의 저장소와 화면(Grafana 등)에서 Exemplar → Trace, Trace → Log 이동이 실제로 되는가?

## 참고 자료

- [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar)
- [OpenTelemetry — Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [OpenTelemetry — Metrics Data Model / Exemplars](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars)
- [OpenTelemetry — Resource SDK](https://opentelemetry.io/docs/specs/otel/resource/sdk/)
- [OpenTelemetry — Java Agent 설정](https://opentelemetry.io/docs/zero-code/java/agent/configuration/)
