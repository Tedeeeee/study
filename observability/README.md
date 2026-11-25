# 모니터링과 옵저버빌리티

[전체 목차](../README.md)

시스템을 이해하기 위해 로그·메트릭·트레이스를 어떻게 남기고 잇는지, OpenTelemetry가 그 생성과 전송을 어떻게 표준화하는지, 코드를 고치지 않고 계측하는 방법은 무엇인지 공부한다. 참고 자료와 실습 환경은 [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar) 0~2회차를 쓴다.

## 읽는 순서

1. [모니터링과 관측성은 뭐가 다를까?](monitoring-vs-observability.md)
   - 미리 정한 질문과 정하지 못한 질문, p99, MSA에서 요청이 끊기는 이유와 컨텍스트 전파를 정리한다.
2. [Metric·Log·Trace는 각각 어떤 질문에 답할까?](signals-and-cardinality.md)
   - 세 signal의 질문·비용·약점, Trace 샘플링, Metric이 앱 메모리에서 쌓였다 나가는 방식, 라벨 카디널리티를 정리한다.

### 기존 정리

공부 계획을 세우기 전에 쓴 노트다. 아래 계획의 해당 주제를 통과하면 새 노트로 바꾼다.

- [로그·메트릭·트레이스는 어떻게 연결될까?](signals-and-correlation.md)
  - 같은 요청의 메트릭, Span, 로그를 비교하고 Trace ID·Span ID·Exemplar로 이어서 분석한다.
- [관측 데이터를 어떤 형식으로 수집하고 관리할까?](opentelemetry-and-collection.md)
  - 공통 속성과 OTel을 이해하고, 시계열·샘플링·배치·재시도·보관 정책을 살펴본다.
- [APM 서비스와 오픈소스 조합은 어떻게 다를까?](apm-and-backends.md)
  - 대표 제품과 OSS 조합의 수집·저장·조회 흐름, 직접 구현할 때의 책임을 비교한다.

## 공부 계획

주제를 통과하면 체크하고, 노트를 위 "읽는 순서"에 올린다.

**Part 1. 관측 기본**

- [x] 1. 모니터링과 관측성은 뭐가 다를까? — 미리 정한 질문과 정하지 못한 질문, 모니터링이 답하는 범위
- [x] 2. Metric·Log·Trace는 각각 어떤 질문에 답할까? — 비용과 약점, 카디널리티
- [ ] 3. Trace와 Span은 어떻게 생겼고, 서비스 경계를 어떻게 넘을까? — 부모-자식 관계, `traceparent`, 컨텍스트 전파
- [ ] 4. Metric은 무엇으로 잴까? — Counter, UpDownCounter, Histogram, Gauge, 동기와 비동기 Instrument
- [ ] 5. 세 signal은 어떻게 이어질까? — 로그 속 trace_id(appender와 MDC), Exemplar, `service.name`

**Part 2. OpenTelemetry 구조**

- [ ] 6. APM이 있는데 OTel은 왜 나왔을까? — 도구마다 제각각인 문제, 벤더 종속, OpenTracing + OpenCensus
- [ ] 7. API·SDK·Collector는 각각 무슨 일을 할까? — OTel이 표준화하는 것과 안 하는 것
- [ ] 8. OTLP 메시지는 어떻게 생겼을까? — Resource → Scope → 데이터, 4317/4318 포트, semconv
- [ ] 9. Collector는 데이터를 어떻게 처리할까? — receiver → processor → exporter, Agent와 Gateway 배포

**Part 3. 코드 수정 없이 계측하기**

- [ ] 10. Java Agent·Boot 내장·OTel Starter는 구조가 어떻게 다를까?
- [ ] 11. 같은 요청인데 왜 이름·단위·Resource가 다를까? — 실습 비교와 흔한 함정

기존 노트와 겹치는 주제: `signals-and-correlation.md` → 5번(2번 내용은 새 노트로 옮김), `opentelemetry-and-collection.md` → 7~9번, `apm-and-backends.md` → 6번.

## 기록 기준

- 각 문서는 시작한 질문, 이해한 내용, 헷갈렸던 점, 예제와 확인 결과, 남은 질문, 참고 자료를 연결한다.
- 헷갈렸던 점에는 공부하며 실제로 헷갈린 것과 확인 문제·내 답을 남긴다.
- 설명용 예제와 실제로 실행한 결과를 구분한다. 실행했다면 실행 환경과 관찰한 결과를 적는다.
- 일반적인 개념과 세미나 실습 환경에서 확인한 내용을 구분한다.
