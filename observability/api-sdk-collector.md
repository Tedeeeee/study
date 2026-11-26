# API·SDK·Collector는 각각 무슨 일을 할까?

[목차](README.md) · [이전: APM이 있는데 OTel은 왜 나왔을까?](why-opentelemetry.md)

## 다루는 범위

OTel을 API와 SDK로 **왜 나눴는지**, 아무 설정이 없을 때 어떤 구현체가 쓰이는지, 앱과 백엔드 사이에 두는 **Collector**가 무엇이고 왜 쓰는지, OTel이 만들어 주는 것과 우리 몫을 정리한다.

부품 각각의 역할(계측 모듈, API, SDK, Exporter)은 [6장](why-opentelemetry.md)에 정리했다. Collector의 내부 구조(receiver → processor → exporter)와 Agent·Gateway 배포는 9장에서 다룬다.

## 시작한 질문

API와 SDK가 둘 다 OTel 것인데 왜 굳이 나눴을까? 앱이 백엔드로 바로 보내면 되는데 Collector는 왜 중간에 둘까?

## 내가 이해한 내용

### 1. API와 SDK를 나눈 이유: SLF4J와 Logback

HikariCP는 안에서 로그를 찍지만 Logback을 직접 쓰지 않고 **SLF4J만** 쓴다. Logback을 직접 끌고 오면 Log4j를 쓰려는 앱이 곤란해지기 때문이다. OTel도 똑같다.

| 로그 | OTel |
| --- | --- |
| SLF4J (창구) | **API** |
| Logback (실제 처리) | **SDK** |
| HikariCP는 SLF4J만 쓴다 | 라이브러리는 **API만** 쓴다 |
| 앱이 Logback을 고른다 | 앱이 **SDK(또는 Agent)**를 붙이고 설정한다 |

API와 SDK는 **둘 다 OTel 프로젝트 것**이다. 같은 프로젝트 안에서 조각을 나눠 둔 것이다.

라이브러리가 SDK까지 끌고 오면 생기는 문제는 두 가지다.

1. **설정을 앱이 자유롭게 못 한다.** 라이브러리가 정한 보낼 곳과 샘플링이 앱과 충돌한다.
2. **OTel을 안 쓰는 앱도 비용을 낸다.** 나눠 두면 SDK가 없을 때 API가 no-op이라 공짜다.

> 라이브러리는 API로 "여기서 재 줘"만 말하고, 켤지 말지와 설정은 앱이 정한다.

### 2. no-op: 호출은 되는데 아무 일도 안 한다

no-op(no operation)은 메서드는 있어서 에러 없이 호출되지만 안이 비어 있는 구현이다.

```java
// 설명용으로 단순화
class NoopSpan implements Span {
    public void setAttribute(String k, String v) { }
    public void end() { }
}
```

SLF4J도 구현체가 없으면 `SLF4J: Defaulting to no-operation (NOP) logger implementation` 경고를 띄우고 로그를 안 찍는다. OTel API도 SDK가 없으면 똑같이 조용히 지나간다. 그래서 라이브러리 개발자가 OTel API로 마음 놓고 계측을 넣을 수 있다.

### 3. 기본 구현체는 누가 정하나: 전역 자리 하나

랜덤이 아니고 규칙이 하나다. **앱이 시작할 때 누군가 등록했으면 그걸 쓰고, 아무도 안 했으면 no-op이다.**

```text
GlobalOpenTelemetry  ← 전역 자리 하나
   ├─ Agent를 붙였다     → Agent가 자기 SDK를 등록
   ├─ SDK를 직접 붙였다  → 앱 코드가 SDK를 만들어 등록 (law-erp의 telemetry.ts)
   └─ 아무도 안 했다     → no-op
```

"라이브러리 박고 대충 써도 된다"는 경험은 맞다. 다만 기본값을 깔아 주는 건 **라이브러리가 아니라 앱 쪽 묶음**이다. Spring Boot starter가 Logback을 기본으로 넣어 주는 것처럼, OTel은 Agent나 Boot starter가 SDK를 넣어 준다.

Agent를 설정 없이 붙였을 때의 기본값은 대략 이렇다.

| 항목 | 기본값 |
| --- | --- |
| 보낼 곳 | `localhost:4317`(gRPC) 또는 `4318`(HTTP)의 OTLP |
| 샘플링 | 부모 결정을 따르고, 루트는 전부 남김 |
| `service.name` | `unknown_service:java` (또는 Agent가 추정한 이름) |

그래서 실습 환경은 보통 로컬에 Collector를 띄워 둔다.

> 확인 수준: 기본 프로토콜(gRPC/HTTP)은 Agent 버전마다 바뀐 적이 있어 확인이 필요하다.

### 4. Collector: 앱과 백엔드 사이의 중간 정거장

```text
Collector 없이:   앱 ──OTLP──▶ 백엔드
Collector 있으면: 앱 ──OTLP──▶ Collector ──▶ 백엔드
                                받기 → 손보기 → 보내기
```

서비스가 30개일 때 차이가 보인다.

| 상황 | Collector 없이 | Collector 있으면 |
| --- | --- | --- |
| 백엔드 교체 (Jaeger → Tempo) | 30개 설정 변경·재배포·확인 | Collector 설정 하나 |
| 두 백엔드로 동시에 | 30개에 Exporter 추가 | Collector가 복사해서 보냄 |
| 헬스체크 Span 버리기, 민감 속성 지우기 | 30개에 각각 | Collector에서 한 번에 |
| 백엔드 일시 장애 | 앱마다 재시도 | Collector가 모아 두고 재시도 |
| API 키 | 30곳에 퍼짐 | Collector에만 |

> 앱은 "Collector로 보낸다"까지만 알고, 그 뒤는 Collector가 맡는다.

알아 둘 성질은 다음과 같다.

- **저장하지 않는다.** 어디로 보낼지를 정할 뿐이다. 저장, 검색, 화면은 백엔드 몫이다. 백엔드가 죽었을 때 잠시 들고 있는 정도만 한다.
- **필수가 아니다.** law-erp는 Collector 없이 앱이 monitoring 서버로 바로 OTLP를 보낸다. 서비스가 적고 백엔드가 하나면 충분하다.
- **나눠 보낼 수 있다.** signal별(Trace → Tempo, Metric → Prometheus, Log → Loki)로 나누는 게 흔하고, 5장의 `service.name`을 보고 서비스별로 나눌 수도 있다.
- **예전 형식도 받는다.** contrib 배포판은 Jaeger, Zipkin, Prometheus 형식을 받아 OTLP로 바꿀 수 있다. 한 번에 다 못 옮길 때 다리가 된다.
- 중앙에 하나 두는 방식을 **Gateway**, 서버마다 옆에 두는 방식을 **Agent** 배포라고 한다(9장).

### 5. Collector는 만드는 게 아니라 받아서 설정한다

OTel 프로젝트가 **Go로 만든 공식 프로그램**을 배포한다. 우리는 Docker 이미지 등으로 띄우고 YAML 설정만 쓴다. 그래서 Netty나 WebClient 같은 구현 기술을 고를 일이 없다. Go의 goroutine으로 동시 처리를 한다.

| 배포판 | 내용 |
| --- | --- |
| core (`otelcol`) | 기본 부품만 |
| contrib (`otelcol-contrib`) | 부품이 많다. 실습은 보통 이것 |
| Collector Builder | 필요한 부품만 골라 직접 빌드 |
| 회사 배포판 | Grafana, AWS 등이 OTel Collector를 바탕으로 낸 버전 |

```yaml
# 설명용: docker-compose.yml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib
    volumes:
      - ./otel-config.yaml:/etc/otelcol-contrib/config.yaml   # 설정 파일 마운트
    environment:
      - VENDOR_KEY=${VENDOR_KEY}                              # 비밀 값은 환경변수
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
```

```yaml
# 설명용: otel-config.yaml 중 exporters 부분
exporters:
  otlp/tempo: { endpoint: tempo:4317 }
  otlphttp/vendor:
    endpoint: https://otlp.vendor.example
    headers: { api-key: ${env:VENDOR_KEY} }
```

설정 파일에 비밀 값을 직접 쓰지 않고 `${env:...}`로 빼 두면 설정 파일은 Git에 올려도 된다. 쿠버네티스에서는 ConfigMap과 Secret으로 넣는다. 성능은 `batch`(묶어 보내기), `memory_limiter`(메모리 보호), 전송 큐와 재시도 설정으로 챙긴다.

### 6. 정리: OTel이 만드는 것과 우리 몫

```text
[앱]                                          [중간]           [백엔드]
계측 모듈 → API → SDK → Exporter ──OTLP──▶  Collector ──▶  저장·검색·화면·알림
└──────── Agent가 한 묶음으로 ────────┘     (선택)
◀───────────── OTel이 만들어 줌 ─────────────────────▶     ◀── OTel이 안 만듦 ──▶
```

| 부품 | OTel이 만드나 | 우리가 하는 일 |
| --- | --- | --- |
| API | ○ | 업무 구간 계측 코드 작성 |
| SDK | ○ | 붙이고 설정 (샘플링, Resource) |
| 계측 모듈 | ○ | 대부분 Agent가 자동으로 |
| Exporter | ○ | 보낼 주소 설정 |
| Collector | ○ | 이미지 띄우고 YAML 설정 |
| 규칙 (OTLP, `traceparent`, semconv) | ○ | 따르기 |
| 백엔드 | ✕ | 골라서 운영하거나 상용 서비스 사용 |

부품이 아니라 **결정**이라서 OTel이 못 해 주는 것도 있다. 무엇을 잴지(업무 구간), 얼마나 남길지(샘플링, 보관 기간), 무엇을 뺄지(개인정보)다.

> OTel은 데이터를 만들고 나르는 표준 도구와 규칙이고, 저장·화면과 "무엇을 얼마나 남길지"는 우리 몫이다.

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| API와 SDK를 나눈 설명이 와닿지 않았다 (라이브러리 의존성, 버전 충돌 설명) | 설명이 길어서 이해가 안 됐다. HikariCP가 Logback이 아니라 SLF4J만 쓰는 이유로 다시 이해했다. |
| SDK를 OTel로 안 쓰고 싶은데 강제되는 게 문제인가? | API도 SDK도 둘 다 OTel 것이다. 문제는 SDK 종류가 아니라 **켤지 말지와 설정**을 라이브러리가 정해 버리는 것이다. |
| no-op이 뭐지? | 호출은 되는데 아무 일도 안 하는 빈 구현이다. SLF4J의 NOP 로거와 같다. |
| SDK를 강제하면 설정을 자유롭게 못 하는 거다 | 맞다. 더해서 OTel을 안 쓰는 앱도 비용을 낸다. |
| 라이브러리를 박고 대충 쓰는데, 기본 구현은 라이브러리가 넣어 두는 것 아닌가? | 기본값을 깔아 주는 건 라이브러리가 아니라 Spring Boot starter(Logback)나 Agent(SDK) 같은 앱 쪽 묶음이다. |
| 아무 설정 없을 때 구현체는 그때그때 다른가? | 전역 자리(`GlobalOpenTelemetry`)에 누가 등록했느냐로 정해진다. 아무도 안 했으면 no-op이다. |
| Collector는 모니터링의 gateway 같은 것이다 | 맞다. 실제로 중앙에 하나 두는 방식을 Gateway 배포라고 부른다. |
| Collector가 어디에 저장할지 정한다 | **어디로 보낼지**를 정한다. 저장은 하지 않는다. |
| Collector는 Netty, WebClient 같은 걸로 빠르게 처리하게 만들면 좋겠다 | 직접 만드는 게 아니다. OTel이 Go로 만든 공식 프로그램을 받아서 YAML로 설정한다. |
| Collector도 OTel이 가지고 있나? | 그렇다. core와 contrib 배포판이 있고 Docker 이미지로 띄운다. |
| 백엔드마다 서버 정보(주소, 인증)가 다르지 않나? | 다르지만 Collector 설정에 Exporter별로 적어 둔다. API 키가 Collector 한 곳에만 있게 된다. |
| 이미지로 띄울 때 설정을 다 적어 줘야 하나? | 설정 파일(YAML)을 마운트하고, 비밀 값은 환경변수로 넣는다. |

### 확인 문제

**Q1.** 사내 HTTP 클라이언트 라이브러리 `fast-http`(가상)에 OTel SDK와 OTLP Exporter를 넣고 보낼 곳을 우리 팀 Jaeger로 박아 두자는 제안이 있다. ① 어떤 문제가 생기나 ② `fast-http`는 무엇에만 의존해야 하고, OTel을 쓰는 팀과 안 쓰는 팀에서 각각 어떻게 동작하나

내 답:

> 1. 이렇게 되면 다른 사람들이 fast-http를 이용하면서 sdk 를 강제로 사용하게 되면서 자유도가 낮아진다? 그리고 Jeager 로 박아두면 다른 사람들은 해당 라이브러리를 사용하면 원하는 모니터링 도구쪽으로 보낼 수 가 없다
> 2. fast-http 는 OTel 의 API만 의존하게 해서 어떤 방식으로 사용할 예정으로만 하고 어떤 데이터를 모으고 어떤 방식으로 수집할지 그리고 어떤 방식으로 보낼지 선택을 할 수 있게 해줘야 한다

채점: 통과. 1번은 맞았다(보태면 안 쓰는 팀의 비용, SDK 버전 충돌). 2번은 API에만 의존한다는 핵심은 맞았다. 다만 "어떤 데이터를 모을지"(어디서 Span을 시작할지)는 라이브러리가 정하고, 앱은 샘플링·Resource·보낼 곳을 정한다. 쓰는 팀은 진짜 Span이 자기 백엔드로 나가고, 안 쓰는 팀은 no-op이라는 부분이 빠졌다.

**Q2.** 서비스 30개가 Agent로 Jaeger에 바로 보낸다. Tempo로 교체, 결제 서비스만 상용 APM에도 전송(API 키 필요), `/health` Span 버리기 요청이 들어왔다. ① 지금 구조에서 무엇이 힘든가 ② Collector를 두면 어떻게 바뀌고 세 요청은 어디서 처리되나 ③ Collector에 데이터가 저장되나

내 답:

> 1. 각 서비스마다 설정을 모두 변경해줘야 하고 제대로 변경했는지도 또 확인을 해야한다
> 2. collector 를 두면 기존의 서비스들은 모두 그대로 둬도 되고 collector 에서 보내는 설정만 변경하면 된다
> 3. collector 는 gateway 로써 길잡이의 역할을 하는것이지 데이터를 저장하는 행위는 하지 않는다

채점: 통과. 1, 3번은 맞았다. 2번은 핵심은 맞았지만, 서비스의 보낼 곳을 Collector로 돌리는 **처음 한 번**은 바꿔야 한다. 세 요청은 모두 Collector 안에서 처리된다. Tempo는 Exporter 주소 변경, 결제 서비스는 `service.name`으로 나눠 Exporter 추가(키는 Collector에만), `/health`는 processor에서 버린다.

## 예제와 확인 결과

- 이 문서의 코드, YAML, docker-compose는 모두 설명용이다. 세미나 실습 환경에서 실행해 확인하지는 않았다.
- law-erp 서버는 Collector 없이 `telemetry.ts`에서 `${base}/v1/metrics`, `/v1/traces`, `/v1/logs`로 바로 보낸다는 것을 코드에서 확인했다. `base` 값(.env)은 열지 않았다.

## 남은 질문

- Agent의 기본 OTLP 프로토콜은 gRPC인가 HTTP인가? 설정 없이 붙였을 때 `service.name`은 어떻게 정해지나? (10장 실습)
- 세미나 실습 환경의 Collector는 어떤 이미지와 설정을 쓰나?
- 서비스별로 나눠 보내는 routing 부품의 현재 이름과 설정 방법 (9장)
- Spring이 OTel API 대신 Micrometer Observation으로 계측하고 OTel로 bridge하는 구조 (10장)
- 실제로 OTel API를 직접 내장한 Java 라이브러리는 얼마나 되나?

## 참고 자료

- [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar)
- [OpenTelemetry — Components](https://opentelemetry.io/docs/concepts/components/)
- [OpenTelemetry — Specification Overview (API와 SDK)](https://opentelemetry.io/docs/specs/otel/overview/)
- [OpenTelemetry — Collector](https://opentelemetry.io/docs/collector/)
- [OpenTelemetry — Collector 배포 방식](https://opentelemetry.io/docs/collector/deployment/)
- [OpenTelemetry — Java Agent 설정](https://opentelemetry.io/docs/zero-code/java/agent/configuration/)
