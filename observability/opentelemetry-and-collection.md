# 관측 데이터를 어떤 형식으로 수집하고 관리할까?

[목차](README.md) · [이전: 로그·메트릭·트레이스의 관계](signals-and-correlation.md) · [다음: APM과 저장 방식](apm-and-backends.md)

## 다루는 범위

세 데이터의 역할을 이해했다면, 이제 서로 연결할 수 있는 형식으로 만들고 전달해야 한다. 공통 식별자와 속성, OpenTelemetry의 역할, 수집 방식, 샘플링·보관·전송 실패 정책을 하나의 흐름으로 정리한다.

실행 설정이나 완성된 저장 시스템을 제공하는 문서는 아니다. 예시와 적용 검토 사항은 실제 계측·저장 검증 결과와 구분한다.

## 시작한 질문

각 서비스가 데이터를 남기더라도 이름과 구조가 제각각이면 어떻게 함께 분석할까? 데이터를 모두 수집하면 정보는 풍부해지는데, 비용과 저장 실패는 어떻게 관리해야 할까?

## 내가 이해한 내용

### 1. 먼저 세 데이터의 형식과 연결 기준을 정한다

[앞 문서](signals-and-correlation.md)에서 본 관계를 유지하려면 무엇을 공통으로 넣을지 정해야 한다.

| 데이터 | 주요 내용 | 연결할 때 사용하는 정보 |
| --- | --- | --- |
| 메트릭 | 이름, 단위, 속성, 시각·구간, 수치 또는 집계값 | 서비스·환경·API·시간 조건, 가능한 경우 Exemplar |
| Span | Trace ID, Span ID, 부모 관계, 시작·종료, 작업명, 상태·속성 | 같은 Trace의 다른 Span과 해당 작업의 로그 |
| 로그 | 발생 시각, 심각도, 본문·속성, 가능한 경우 Trace ID·Span ID | 같은 요청 또는 작업의 Span |

모든 데이터에 똑같은 필드가 필수인 것은 아니다. 요청과 무관한 시스템 지표에는 Trace ID가 없을 수 있고, 요청 밖의 로그에는 Span ID가 없을 수 있다. 연결할 수 있는 경우에 올바른 Context를 담는다.

### 2. Resource는 데이터를 만든 대상을 설명한다

Resource 속성은 **이 데이터가 어느 서비스의 어느 실행 주체에서 나왔는가**를 설명한다. 같은 실행 주체가 만든 세 신호에 일관된 Resource를 사용하면 검색 범위를 맞추기 쉽다. [OTel Resources](https://opentelemetry.io/docs/concepts/resources/)

```json
{
  "service.name": "study-chat",
  "service.instance.id": "study-chat-local-1",
  "deployment.environment.name": "development"
}
```

| 속성 | 이 예제의 의미 |
| --- | --- |
| `service.name` | 논리적인 서비스 이름 |
| `service.instance.id` | 그 서비스의 개별 실행 인스턴스를 구분하는 식별자 |
| `deployment.environment.name` | development, staging, production 같은 실행 환경 |

위 값들은 설명용이다. 실제 인스턴스 식별자의 생성·재시작 정책은 별도로 정해야 한다.

“셋 모두 같게”는 **같은 생산자의 로그·메트릭·트레이스를 일관되게 식별하라**는 뜻이다. 다른 서비스들까지 같은 `service.name`이나 `service.instance.id`를 쓰라는 뜻은 아니다. 같은 분산 Trace에 참여하더라도 서비스마다 Resource는 달라질 수 있다.

환경 속성이 있다고 로컬 데이터가 자동으로 운영 수집기에 전달되는 것도 아니다. 어느 환경에서 계측과 전송을 켤지, 어디로 보낼지는 별도의 설정이다.

### 3. Semantic Conventions는 이름과 의미에 대한 공통 약속이다

어떤 서비스는 `method`, 다른 서비스는 `requestMethod`라고 쓰면 같은 의미를 조회할 때마다 매핑해야 한다. 단위가 ms인지 s인지 다르면 숫자 자체도 잘못 비교할 수 있다.

OTel의 Semantic Conventions는 작업과 데이터의 **속성명·의미·단위 등에 공통 기준**을 제공한다. HTTP의 `http.request.method`, `http.route`, `http.response.status_code` 등이 그 예다. [공식 설명](https://opentelemetry.io/docs/concepts/semantic-conventions/)

다만 OTel을 설치했다고 임의로 만든 모든 로그 필드가 자동으로 통일되는 것은 아니다. 표준을 따르는 계측을 사용하고, 사용자 정의 속성도 팀에서 일관되게 정해야 한다. 사용하는 계측 라이브러리와 Semantic Conventions의 버전도 확인한다.

이름의 일관성은 MSA에서 특히 유용하지만 단일 서비스에서도 재배포·환경 구분·계측 변경을 거치며 도움이 된다.

### 4. OTel은 수집과 전달을 위한 공통 기반이다

OpenTelemetry는 관측 데이터의 생성·처리·전달을 위한 API, SDK, 자동 계측, 공통 규약 등을 제공한다. 저장소와 분석 화면은 다른 도구가 담당한다. [What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)

```text
애플리케이션의 계측
  → SDK의 처리·집계
  → Exporter
  → 저장·분석 백엔드

필요한 구성에서는:
Exporter → Collector → 저장·분석 백엔드
```

각 구성 요소는 다음 역할을 맡는다.

| 구성 요소 | 역할 |
| --- | --- |
| 계측(Instrumentation) | HTTP 요청, DB 호출, 업무 작업 등에서 관측값·Span·로그를 생성 |
| API | 코드가 관측 데이터를 기록하는 인터페이스 |
| SDK | 기록된 데이터를 처리·집계하고 Exporter와 연결 |
| Exporter | 데이터를 수신 시스템이 받을 수 있게 변환·전달 |
| Collector | 애플리케이션 밖에서 수신·처리·전달을 담당할 수 있는 선택적 중간 구성 요소 |

백엔드가 직접 수신할 수 있는 형식을 지원하면 Collector를 반드시 거칠 필요는 없다. 반대로 여러 목적지 분기, 공통 변환, 중앙 정책이 필요하면 Collector를 둘 수 있다.

### 5. SDK와 Agent는 어떤 차이가 있을까?

SDK는 애플리케이션에 포함되는 라이브러리다. 직접 작성한 업무 코드에서도 계측 API를 호출할 수 있다.

Java Agent는 JVM에 붙어 지원되는 프레임워크·라이브러리 호출을 자동 계측할 수 있다. 자동 계측 구성도 내부적으로 SDK를 사용할 수 있으므로 둘은 배타적인 제품 분류가 아니다. 제품에 따라 호스트에서 별도 실행하는 수집 프로그램도 Agent라고 부른다. [OTel Java 계측 방식](https://opentelemetry.io/docs/languages/java/instrumentation/)

Agent를 쓴다는 말은 단순히 객체를 외부 API에 POST한다는 뜻이 아니다. 무엇을 관찰해서 데이터를 만들고, 어느 경로로 전달하는지까지 살펴봐야 한다.

자동 계측은 지원되는 HTTP·DB 호출 등을 기록하는 부담을 줄여준다. 하지만 업무상 중요한 실패 이유나 도메인별 처리 단위까지 모두 알아서 판단해주는 것은 아니다. 필요한 업무 Span과 로그는 추가할 수 있다.

### 6. 전송 규격은 객체의 모양과 전달 방법에 대한 약속이다

전송 규격에는 데이터의 필드·인코딩뿐 아니라 전송 방식, 응답과 오류 처리 등이 포함된다. 송신자와 수신자가 이 약속을 함께 지원해야 한다.

OTLP(OpenTelemetry Protocol)는 관측 데이터를 교환하는 규격이다. HTTP나 gRPC를 통한 전달과 Protocol Buffers 기반 메시지 등을 정의한다. **자바 객체를 아무 형태로 보내도 저장소가 해석해주는 것은 아니다.** [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)

DB에 직접 저장하는 사용자 정의 Exporter라면 SDK 데이터를 읽어 DB 행으로 매핑하는 코드를 작성할 수 있다. 이 경로가 반드시 OTLP 전송을 거쳐야 하는 것은 아니다. 반대로 OTLP 수신기를 만들겠다면 그 규격에 맞는 수신·변환 구현이 필요하다.

### 7. 라벨이 늘면 왜 시계열이 폭발할까?

Prometheus에서는 메트릭 이름과 라벨 값 조합이 시계열을 구분한다. 같은 시계열에 시각별 측정값이 계속 붙는다. **다른 시각에 측정했다고 매번 새로운 시계열이 되는 것은 아니다.** [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/)

```text
requests_total{route="/api/messages",method="POST"}
  → 15:00의 값
  → 15:01의 값
  → 15:02의 값
```

아래 숫자는 가능한 조합을 보여주기 위한 가정이다.

```text
API 경로 20종 × 메서드 5종 × 상태 코드 5종 × 인스턴스 10개
  = 최대 5,000개 조합

여기에 사용자 ID 500종을 추가
  = 최대 2,500,000개 조합
```

모든 조합이 반드시 실제로 생기는 것은 아니다. 계측·저장된 조합만 존재하지만, 요청마다 바뀌는 값을 라벨로 쓰면 시계열 수가 빠르게 증가할 수 있다. Classic Histogram에서는 bucket 등에 따른 추가 시계열도 고려한다.

따라서 API 경로는 `/users/123` 같은 원본 값 대신 `/users/{id}`처럼 정규화된 route를 사용한다. 사용자 ID, 요청 ID, Trace ID를 일반 메트릭 라벨로 넣어 개별 요청을 추적하려 하지 않는다. 필요한 요청 연결은 Exemplar나 로그·트레이스 조회로 해결한다.

Resource 속성도 백엔드에서 메트릭 라벨로 변환될 수 있으므로, 공통 속성을 통일하는 일과 메트릭 차원의 개수를 제한하는 일을 함께 고려한다.

### 8. Trace 샘플링과 로그·메트릭 수집은 구분한다

여기서 샘플링은 **생성 가능한 Trace 중 어떤 것을 수집·보관 대상으로 선택할지** 결정하는 이야기다. Head와 Tail은 서로 다른 판단 시점이며, 반드시 둘을 함께 써야 하는 것은 아니다. [OTel Sampling](https://opentelemetry.io/docs/concepts/sampling/)

| 방식 | 판단 시점 | 예 | 고려할 점 |
| --- | --- | --- | --- |
| Head | Trace 시작 시점 | Trace ID 기반으로 약 10% 선택 | 최종 오류나 전체 지연을 아직 모름 |
| Tail | 들어온 Span을 일정 시간 모아 결과를 평가한 뒤 | 오류·느린 Trace를 우선 보관 | 판단 전 수집·버퍼 비용, 늦게 온 Span·불완전한 Trace 처리 |

Head 10%는 선택 대상 10%, 미선택 대상 90%라는 뜻이다. 각 API나 매초마다 정확히 10건 중 1건을 보장하는 할당량은 아니다. 부모의 결정을 이어받는 등 계측 구성도 맞춰야 관련 Span의 선택을 일관되게 유지할 수 있다.

Head에서 미선택되어 전달되지 않은 Span은 Tail이 복원할 수 없다. 따라서 모든 오류 Trace를 Tail에서 고르고 싶다면 그 판단 지점까지 필요한 Span이 전달되어야 한다. Head와 Tail을 혼합하면 이 제약을 받아들이고 정책을 설계해야 한다.

다음은 **메트릭을 별도로 측정하고, 요청 요약 로그는 모든 대상 요청에 남기는 구성**의 설명용 비교다.

| 요청의 Trace 선택 결과 | 메트릭 반영 | 요청 요약 로그 | 계측한 Span의 전송 |
| --- | --- | --- | --- |
| 선택됨 | 반영 | 남김 | 전송 대상 |
| 미선택 | 반영 | 남김 | 전송하지 않음 |

이렇게 구성하면 메트릭상 오류 건수와 해당 요청의 로그가 있어도, 저장된 상세 Trace는 없을 수 있다. 로그에 Trace ID가 있어도 그 ID의 Span이 저장되어 있다는 보장은 없다.

다만 **Span에서 메트릭을 만드는 파이프라인**이라면 앞단의 Span 샘플링이 메트릭에도 영향을 줄 수 있다. 모든 수집 방식에서 위 표가 자동으로 보장되는 것은 아니다. 로그에도 별도의 필터·샘플링·전송 실패 정책이 있을 수 있다.

학습 중 검토한 초기안은 수집량을 먼저 관찰하고 Trace 샘플링을 나중에 고려하는 것이다. 이는 **정한 계측 범위에서 100% 수집을 목표로 시작하자**는 적용안이며, 실제 누락이 전혀 없다는 의미는 아니다.

### 9. 배치 지원과 DB 재시도 지원은 별개다

OTel Java SDK는 로그와 Span을 모아 Exporter로 전달하는 배치 처리를 제공한다. 메트릭은 집계한 데이터를 주기적으로 읽어 내보내는 구성이 가능하다. [OTel Java SDK](https://opentelemetry.io/docs/languages/java/sdk/)

```text
요청 처리 중 기록
  → 크기가 제한된 대기열·집계 구조
  → 배치 또는 주기적 Export
  → 저장소
```

배치 Exporter에 여러 레코드가 전달된다는 것과 DB INSERT가 실제로 일괄 실행된다는 것은 구분한다. 사용자 정의 DB Exporter에서는 SQL 배치 처리와 트랜잭션 경계를 직접 구현해야 한다.

SDK의 대기열은 무제한 임시 저장소가 아니고, 기본 인메모리 버퍼가 재시작 뒤에도 남는 것도 아니다. SDK의 배치 기능만으로 DB 재시도와 영구 보관까지 보장되지는 않는다. OTLP Exporter나 Collector의 재시도도 해당 구현과 설정을 확인해야 한다. Collector에는 저장 대기열을 영속화하는 구성도 있다. [Collector Resiliency](https://opentelemetry.io/docs/collector/resiliency/)

### 10. 수집 경로의 어느 지점이 실패했는지 구분한다

| 문제가 난 위치 | 확인하고 정할 내용 |
| --- | --- |
| 계측·로그 생성 | 무엇을 누락했는지, 애플리케이션 오류와 수집 오류가 구분되는지 |
| SDK 대기열 | 상한, 가득 찼을 때의 폐기 방식, 누락 건수 확인 방법 |
| Exporter·통신 | 연결·요청 제한 시간, 재시도 가능한 오류, 재시도 횟수·총 대기 시간 |
| Collector | 수신 제한, 대기열, 디스크 사용, 뒤쪽 저장소로의 전달 실패 |
| 최종 저장소 | INSERT 실패, 저장 처리 지연, 인덱스·공간·쓰기 부하 |
| 보관 데이터 정리 | 정리 작업 실패, 누적되는 오래된 데이터, 조회 범위 |

예를 들어 수신기까지의 통신 실패와, 수신기는 받았지만 DB 저장에 실패한 상황은 다르다. 어느 지점의 성공 응답이 무엇을 보장하는지 확인한다. 전송 재시도 시 중복 저장 가능성도 함께 고려한다.

수집이 밀릴 때 요청 처리까지 무한히 기다리게 하지 않도록 버퍼와 대기 시간을 제한한다. 그렇다고 모든 실패를 조용히 버리는 정책으로 끝내지도 않는다. **업무 처리 영향, 누락 가능성, 장애를 알아챌 방법**을 같이 설계한다.

### 11. 보관 기간과 비용도 수집 설계의 일부다

보관 비용은 기간뿐 아니라 초당 요청량, 요청당 Span 수, 로그 크기, 메트릭 시계열 수, 인덱스 등에 따라 달라진다. 같은 7일도 수집량에 따라 크기가 다르다.

Hot·Cold는 조회 빈도·속도·비용에 따라 저장 단계를 나누는 정책이다. 별도 저장 계층으로 옮기려면 이동, 검증, 복원·조회 경로가 필요하다. 이름만 Cold라고 붙인다고 데이터가 자동으로 이동하지 않는다.

**2026-10-18 학습 대화의 최신 적용 검토안**은 로그·메트릭·트레이스를 최근 7일치로 제한하고, 이전 데이터는 주기적으로 삭제하는 것이다. 이전에 이야기한 장기 Cold 보관은 이 안에 포함하지 않는다. 이는 보편적인 권장 기간이나 적용 완료 사실이 아니다.

삭제 기준 시각과 실행 주기를 정해야 하며, 정리 작업 사이에는 만료된 데이터가 잠시 남을 수 있다. 조회 범위도 제한한다. DB를 사용한다면 대량 삭제와 조회가 업무 데이터에 주는 영향을 측정하고, 규모에 따라 날짜 파티션 등도 검토할 수 있다. [PostgreSQL Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

### 12. 필요한 데이터와 민감한 데이터의 경계도 정한다

오류 원인을 분석할 수 있는 정보는 필요하지만, 요청 본문·인증 토큰·개인정보를 무작정 수집하면 안 된다. 허용할 필드와 크기를 정하고, 경로는 정규화하며, 비밀값이 섞일 수 있는 예외 메시지도 수집 정책을 적용한다.

메트릭의 오류 수, Span의 오류 상태, 로그의 오류 설명은 서로 보완한다. 같은 정보를 모든 단계에서 로그로 중복 생성해야 하는 것은 아니다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| OTel을 쓰면 저장소와 화면까지 제공된다 | 데이터 생성·처리·전달의 공통 기반이다. 저장·분석 백엔드는 별도다. |
| Resource를 통일하면 모든 서비스의 값이 같아야 한다 | 같은 생산자를 일관되게 식별하고, 다른 생산자는 구분해야 한다. |
| 환경 속성이 있으면 로컬 데이터도 운영에 자동 수집된다 | 실제 계측·전송 활성화와 목적지는 별도 설정이다. |
| Head 10%면 로그와 메트릭도 무조건 10%만 남는다 | 신호별 생성·집계 경로와 정책에 달려 있다. 독립 측정 구성에서는 다르게 운영할 수 있다. |
| Tail이 Head에서 버린 오류 요청도 다시 골라준다 | 전달되지 않은 Span은 복원할 수 없다. |
| SDK가 배치를 지원하니 DB 재시도도 끝났다 | SQL 배치, 실패 분류, 재시도·중복·누락 정책은 별도로 확인해야 한다. |
| 7일치만 보관하면 용량 문제는 해결된다 | 기간은 한 요소다. 유입량과 정리 성공 여부, 인덱스·조회 비용도 영향을 준다. |

## 예제와 확인 결과

- 공통 Resource, 시계열 조합, 샘플링 표는 구성 원리를 설명하기 위한 예제다.
- OTel SDK와 Exporter를 실제 애플리케이션에 연결해 검증한 결과는 아니다.
- 실제 실습에서는 정상 저장뿐 아니라 수신기 중단, 대기열 포화, DB 실패, 보관 만료도 확인해야 한다.

## 남은 질문

- 세 신호의 계측·Export 경로와 계측 범위를 어떻게 정할 것인가?
- 사용자 정의 DB Exporter의 재시도·중복 방지·누락 집계 정책은 무엇인가?
- 보관 기준 시각, 정리 주기, 파티션 또는 배치 삭제 방식은 무엇이 적절한가?
- 저장·조회·삭제를 함께 실행했을 때 실제 업무 응답에 어떤 영향이 있는가?

## 참고 자료

- [OpenTelemetry — What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)
- [OpenTelemetry — Resources](https://opentelemetry.io/docs/concepts/resources/)
- [OpenTelemetry — Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [OpenTelemetry — Java Instrumentation](https://opentelemetry.io/docs/languages/java/instrumentation/)
- [OpenTelemetry — OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)
- [OpenTelemetry — Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [OpenTelemetry — Java SDK](https://opentelemetry.io/docs/languages/java/sdk/)
- [OpenTelemetry — Collector Resiliency](https://opentelemetry.io/docs/collector/resiliency/)
- [Prometheus — Data model](https://prometheus.io/docs/concepts/data_model/)
- [PostgreSQL — Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

## 정리 이력

- 2026-10-18: 수집 형식과 관리 정책을 연결해 정리했다. 보관 기간 등 학습 중 검토한 선택은 일반 개념과 구분했다.
