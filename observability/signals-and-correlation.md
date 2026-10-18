# 로그·메트릭·트레이스는 어떻게 연결될까?

[목차](README.md) · [다음: OTel과 데이터 수집·관리](opentelemetry-and-collection.md)

## 다루는 범위

로그, 메트릭, 트레이스를 각각 이해한 뒤 **같은 요청이 세 데이터에 어떻게 표현되고, 문제를 분석할 때 어떻게 연결되는지** 따라간다. Trace ID, Span ID, Exemplar도 이 흐름 안에서 설명한다.

이 문서의 요청, 식별자, 수치와 JSON은 모두 설명용이다. JSON은 필드의 관계를 보여주기 위한 구조이며, 실제 OTLP 메시지나 확정된 DB 스키마는 아니다.

## 시작한 질문

메트릭 대시보드에서 느린 요청이 많다는 것을 알았다고 하자. 그 숫자만으로 어떤 요청이 느렸고, 어느 작업에서 시간이 걸렸으며, 어떤 오류가 있었는지까지 알 수 있을까?

이를 이해하려면 세 데이터를 따로 정의하는 데서 한 걸음 더 나아가야 한다. 같은 요청을 각 데이터가 **어떤 관점에서 기록하는지** 비교해야 한다.

## 내가 이해한 내용

### 1. 세 데이터는 같은 시스템을 다른 관점에서 보여준다

| 데이터 | 기록하는 관점 | 확인할 수 있는 예 |
| --- | --- | --- |
| 메트릭(Metric) | 시간에 따른 수치와 집계 | 요청 수, 오류 수, 처리 시간 분포, 자원 사용량 |
| 트레이스(Trace) | 관련 작업들의 실행 흐름 | 요청 처리 중 검증, DB 호출, 외부 API 호출이 어떻게 이어졌는가 |
| 로그(Log) | 특정 시점의 사건과 설명 | 요청 처리 결과, 실패한 업무의 오류 코드, 필요한 진단 정보 |

예를 들어 요청 수와 오류 수를 측정하면 오류율을 계산할 수 있다. 대시보드의 모든 그래프가 원시 데이터를 그대로 그리는 것은 아니며, 저장된 메트릭에서 비율이나 증가량을 계산하기도 한다.

메트릭은 애플리케이션이나 시스템에서 직접 측정할 수 있다. 반드시 로그를 먼저 모아야만 만들어지는 것은 아니다. 반대로 로그나 Span을 집계해서 메트릭을 만드는 구성도 있다. **어떤 경로에서 측정한 값인지** 확인해야 한다. [OTel Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)

### 2. 하나의 느린 요청을 세 관점으로 따라가 보자

설명용 API `POST /api/messages`가 3초 뒤 HTTP 500으로 끝났다고 가정한다.

```text
요청 전체 처리: 3,000ms
  ├─ 입력 검증:       50ms
  └─ DB 호출:     2,700ms → 타임아웃

그 결과:
  메트릭 → 요청 수·오류 수에 반영, 처리 시간 3초를 분포에 반영
  트레이스 → 계측한 작업들의 시간·관계·상태를 Span으로 기록
  로그 → 요청 실패 사실과 오류 코드 등을 기록
```

이 세 데이터가 항상 하나의 저장 객체에 들어가는 것은 아니다. 각각의 수집·저장 경로를 가질 수 있고, 식별자와 시간·서비스 정보로 연결한다.

### 3. 메트릭은 요청별 목록 대신 집계된 상태를 보여준다

처리 시간을 Histogram으로 측정하면 관측값이 어느 구간에 속하는지 셀 수 있다. 100건을 평균 하나로만 줄이는 방식과는 다르다.

다음은 **계산을 위한 가상의 관측값**이다. 모두 같은 API에 대한 측정이라고 가정한다.

| 처리 시간 | 건수 |
| --- | ---: |
| 50ms | 70 |
| 300ms | 20 |
| 800ms | 8 |
| 1,100ms | 1 |
| 3,000ms | 1 |
| 합계 | 100 |

이를 Prometheus의 **classic Histogram** 형식으로 표현하면 다음과 같은 누적값이 될 수 있다. 메트릭 이름과 라벨 이름도 설명용으로 정했다.

```text
study_http_request_duration_seconds_bucket{method="POST",route="/api/messages",le="0.1"} 70
study_http_request_duration_seconds_bucket{method="POST",route="/api/messages",le="0.5"} 90
study_http_request_duration_seconds_bucket{method="POST",route="/api/messages",le="1"} 98
study_http_request_duration_seconds_bucket{method="POST",route="/api/messages",le="+Inf"} 100
study_http_request_duration_seconds_count{method="POST",route="/api/messages"} 100
study_http_request_duration_seconds_sum{method="POST",route="/api/messages"} 20.0
```

`le`는 해당 값 이하라는 뜻이다. 이 표현에서 bucket 값은 서로 겹치는 **누적 건수**다.

- 0.1초 이하: 70건
- 0.1초 초과, 0.5초 이하: `90 - 70 = 20건`
- 0.5초 초과, 1초 이하: `98 - 90 = 8건`
- 1초 초과: `100 - 98 = 2건`
- 전체 관측값 합계: 20초, 평균: `20 / 100 = 0.2초`

저장된 bucket만 봐서는 1초 초과 두 건이 각각 1.1초와 3초였다는 사실까지 복원할 수 없다. 위의 원본 표는 예제를 만들기 위해 알고 있는 값이고, Histogram은 그보다 요약된 정보다.

또한 위 숫자가 자동으로 “최근 1분의 100건”을 뜻하는 것은 아니다. 누적 Histogram이라면 두 시점의 차이로 구간별 증가량을 구한다. 실제 시스템에서는 재시작에 따른 counter reset도 고려해야 한다. OTel에는 구간 증가량을 내보내는 delta 집계도 있으므로, 수집 형식을 함께 확인한다. 다른 Histogram 표현도 있으며, 위 예시는 classic 형식에 한정된다. [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/), [OTel Metrics Data Model](https://opentelemetry.io/docs/specs/otel/metrics/data-model/)

### 4. 처리 시간과 RPS·TPS는 다른 질문에 답한다

| 개념 | 묻는 내용 | 단위 예 |
| --- | --- | --- |
| 처리 시간 | 한 번의 작업에 얼마나 걸렸는가? | ms, s |
| RPS | 초당 HTTP 요청이 몇 건인가? | requests/s |
| TPS | 정의한 트랜잭션·업무가 초당 몇 건 처리됐는가? | transactions/s |

RPS도 **어디서 세는지**가 중요하다. 유입 시점에서 세면 들어오는 요청량이고, HTTP 처리가 끝날 때 세면 완료된 요청량이다. 애플리케이션의 처리 시간 Histogram에 붙은 `count`는 보통 그 시간을 기록한 완료 건수다. 이를 시간당 증가량으로 계산하면 해당 측정 지점의 RPS를 얻을 수 있다.

TPS의 트랜잭션은 업무 한 건을 뜻할 수도 있고, DB 트랜잭션을 뜻할 수도 있다. 성공만 세는지 실패한 시도도 포함하는지 정해야 한다. **실제 관측한 TPS와 부하 테스트에서 확인한 최대 처리 가능 TPS도 구분한다.**

예를 들어 API 요청 하나로 메시지 10건을 전송한다면, 요청은 1건이지만 메시지 전송을 업무 단위로 정한 처리 건수는 10건일 수 있다. 한 요청이 한 업무를 처리하는 서비스에서는 RPS와 업무 TPS가 비슷하게 나올 수 있다.

> 처리 시간 Histogram으로 분포를 보고, 그 관측 건수의 증가량으로 처리된 요청량을 계산할 수도 있다. 서로 다른 질문이라고 해서 반드시 완전히 다른 원천 데이터를 수집해야 하는 것은 아니다. [Prometheus 설명](https://prometheus.io/docs/practices/histograms/#count-and-sum-of-observations)

### 5. 트레이스는 관련 Span들의 실행 흐름이다

Span은 시작과 끝이 있는 하나의 작업을 표현한다. 작업 이름, 시간, 부모 관계, 속성, 상태 등을 담을 수 있다. 하나의 Trace에 속한 Span들은 같은 Trace ID를 공유하고, 각 Span은 자신의 Span ID를 가진다. [OTel Traces](https://opentelemetry.io/docs/concepts/signals/traces/)

아래에서는 Trace ID를 `11111111111111111111111111111111`로 정했다.

| 작업 | Span ID | Parent Span ID | 시간 | 상태 |
| --- | --- | --- | ---: | --- |
| HTTP 요청 전체 | `aaaaaaaaaaaaaaaa` | 없음 | 3,000ms | ERROR |
| 입력 검증 | `bbbbbbbbbbbbbbbb` | `aaaaaaaaaaaaaaaa` | 50ms | UNSET |
| DB 호출 | `cccccccccccccccc` | `aaaaaaaaaaaaaaaa` | 2,700ms | ERROR |

여기서는 애플리케이션에 들어온 요청에서 새 Trace를 시작했다고 가정했다. 실제로는 앞단에서 전달받은 Trace에 참여할 수도 있다. 비동기 작업이나 여러 서비스로 이어지는 Trace도 있으므로, “Trace 하나는 언제나 HTTP 요청 하나”로 정의하지는 않는다.

Span의 `UNSET`은 오류를 의미하지 않는다. 정상 작업에서 오류 상태를 지정하지 않은 경우에도 사용한다. 부모 Span의 시간에는 자식 작업 시간이 포함될 수 있고, 자식 작업들이 병렬로 실행될 수도 있으므로 모든 Span의 시간을 단순히 더하지 않는다.

트레이스를 저장한다는 것은 **계측해서 생성한 Span들을 수집·저장하는 것**이다. 모든 함수와 모든 내부 작업이 자동으로 기록된다는 뜻은 아니다. 예제에서 설명하지 않은 나머지 시간도 존재한다.

### 6. 로그는 Span과 별도로 발생하는 사건 기록이다

같은 요청의 완료 시점에 다음과 같은 **요약 로그 하나를 남기는 정책**을 정할 수 있다.

```json
{
  "occurred_at": "2026-10-18T06:00:03Z",
  "severity_text": "ERROR",
  "trace_id": "11111111111111111111111111111111",
  "span_id": "aaaaaaaaaaaaaaaa",
  "body": "메시지 전송 요청 처리 실패",
  "attributes": {
    "http.request.method": "POST",
    "http.route": "/api/messages",
    "http.response.status_code": 500,
    "app.duration_ms": 3000,
    "app.error_code": "DB_TIMEOUT"
  }
}
```

`app.*`는 이 예제에서 정한 사용자 정의 속성이다. 요청 본문이나 인증 정보는 넣지 않았다.

이 로그의 `span_id`는 요청 전체 Span을 가리킨다. DB 작업 중 별도의 오류 로그를 남긴다면 같은 `trace_id`와 DB Span의 `span_id`를 넣을 수 있다. **로그를 모든 Span마다 하나씩 남길 필요는 없다.**

이 예제의 로그가 하나인 것은 선택한 기록 정책 때문이다. Trace 하나에 연결된 로그 수가 언제나 하나로 고정되는 것은 아니다. 요청과 무관한 시작·종료·백그라운드 로그에는 Trace ID가 없을 수도 있다. [OTel Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)

Span과 로그를 “간단한 정보와 상세한 정보”로 나누기도 어렵다. Span도 속성과 이벤트를 풍부하게 담을 수 있고, 로그는 짧은 문장일 수 있다. 핵심 차이는 **작업의 구간·관계와 특정 사건을 기록하는 관점**이다.

### 7. Trace ID와 Span ID는 저장된 데이터를 연결한다

```text
Trace ID = 11111111111111111111111111111111
  ├─ Span 저장소에서 조회
  │    ├─ 요청 전체 Span
  │    ├─ 검증 Span
  │    └─ DB Span
  └─ 로그 저장소에서 조회
       └─ 이 요청에 연결된 로그들
```

이것은 논리적인 연결이다. 로그가 반드시 트레이스 테이블의 하위 행으로 저장되어야 하는 것은 아니다. 트레이스 저장소와 로그 저장소가 서로 다른 제품일 수도 있다.

분산 환경에서는 관련 서비스가 요청의 Trace Context를 전달하고 이어받아야 같은 흐름으로 연결된다. 각 서비스에서 새 ID를 만들기만 하면 이 관계가 끊긴다. [OTel Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)

### 8. Exemplar는 메트릭의 관측값과 Trace를 연결하는 단서다

Histogram으로 1초 초과 요청이 두 건이라는 것은 알았다. 하지만 bucket 안에는 그 두 요청의 전체 목록이 없다. 여기서 Exemplar를 이용할 수 있다.

Exemplar는 **집계에 참여한 관측값 중 선택된 예시**로, 측정값과 시각, 선택적으로 Trace ID·Span ID 등을 담는다. 요청의 Trace Context에서 기록된 Exemplar라면 그 요청의 Trace를 찾는 연결점이 될 수 있다. [OTel Exemplar 정의](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars)

다음은 설명을 위해 bucket 정보 등을 생략한 구조다.

```json
{
  "metric_name": "study.http.request.duration",
  "unit": "s",
  "aggregation": { "count": 100, "sum": 20.0 },
  "exemplars": [
    {
      "value": 3.0,
      "trace_id": "11111111111111111111111111111111",
      "span_id": "aaaaaaaaaaaaaaaa"
    }
  ]
}
```

화면과 저장소의 연동이 지원된다면 다음처럼 분석할 수 있다.

```text
메트릭에서 느린 요청 구간 발견
  → Exemplar가 가리키는 예시 Trace 선택
  → Span들을 보고 오래 걸린 작업 확인
  → 같은 Trace ID 또는 Span ID의 로그 확인
```

여기서 기억할 관계는 다음과 같다.

- Exemplar 하나가 여러 Trace ID를 목록으로 묶는 것은 아니다. Trace Context가 있다면 한 쌍의 Trace ID·Span ID를 담을 수 있다.
- 데이터 포인트에는 여러 Exemplar가 붙을 수 있다. 그렇다고 모든 요청의 식별자가 보관되는 것은 아니다.
- 같은 요청에서 HTTP 시간과 DB 시간을 각각 측정하면, 서로 다른 메트릭의 Exemplar에 **같은 Trace ID와 서로 다른 Span ID**가 나타날 수 있다.
- Trace Context가 없는 메트릭에도 값은 존재할 수 있다. Exemplar가 모든 메트릭에 반드시 붙는 것은 아니다.

Exemplar는 상세 데이터 자체를 대신하지 않는다. 계측·Exporter·저장소·조회 화면이 연결을 지원해야 하고, 가리키는 Trace가 샘플링·전송 실패·보관 만료 등으로 조회되지 않을 가능성도 고려한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| Histogram은 평균 처리 시간만 저장한다 | 구간별 건수와 합계·전체 건수 등을 저장할 수 있다. 평균과 분포는 다른 정보다. |
| RPS는 항상 유입량, TPS는 요청 한 건의 처리 속도다 | 측정 지점과 업무 단위를 정의해야 한다. 요청 한 건의 소요 시간은 latency다. |
| 로그는 Trace의 하위에 자동으로 하나씩 생긴다 | 독립된 사건 기록이다. 같은 Trace ID로 여러 로그를 연결할 수 있고, 개수는 기록 정책에 달려 있다. |
| Span마다 결과 로그를 하나씩 남겨야 한다 | Span 수와 로그 수는 일치할 필요가 없다. |
| Span은 간단한 정보, 로그는 상세 정보다 | 정보량보다는 시간 구간·관계와 사건 기록이라는 관점의 차이다. |
| 메트릭에는 모든 요청의 Trace ID가 들어 있다 | 집계값과 모든 요청의 목록은 다르다. Exemplar는 선택된 관측값의 단서다. |
| 세 데이터에 Trace ID만 넣으면 언제나 연결된다 | 메트릭 연결에는 Exemplar나 공통 검색 조건 등이 필요하고, 대상 데이터도 실제로 남아 있어야 한다. |

## 예제와 확인 결과

- 가상의 100건을 이용해 Histogram의 누적 bucket과 평균 계산을 설명했다.
- 동일한 느린 요청을 Span과 요약 로그, Exemplar의 관점으로 표현했다.
- 실제 서버에서 이 형태로 수집·저장·조회한 결과는 아니다. 실행 환경을 구성한 실습 기록은 별도로 추가한다.

## 남은 질문

- 어떤 HTTP·업무·DB 구간을 Span으로 계측할 것인가?
- 요청 요약 로그와 별도로 남길 진단 로그의 기준은 무엇인가?
- 사용할 저장소와 화면이 Exemplar 및 Trace↔Log 이동을 어떻게 지원하는가?
- 어떤 bucket 경계가 우리 API의 응답 시간 목표에 적절한가?

## 참고 자료

- [OpenTelemetry — Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/)
- [OpenTelemetry — Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry — Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [OpenTelemetry — Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [OpenTelemetry — Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [OpenTelemetry — Metrics Data Model / Exemplars](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars)
- [Prometheus — Histograms and summaries](https://prometheus.io/docs/practices/histograms/)

## 정리 이력

- 2026-10-18: 학습 대화에서 헷갈렸던 관계를 중심으로 정리하고 공식 문서와 대조했다. 예시는 설명용으로 작성했다.
