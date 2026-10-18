# APM 서비스와 오픈소스 조합은 어떻게 다를까?

[목차](README.md) · [이전: OTel과 데이터 수집·관리](opentelemetry-and-collection.md)

## 다루는 범위

관측 데이터를 수집한 뒤에는 저장·조회·분석하는 시스템이 필요하다. 이미 만들어진 APM 제품을 이용하는 방식, 오픈소스 백엔드를 직접 운영하는 방식, 필요한 기능을 직접 구현하는 방식을 비교한다.

제품 설명은 2026-10-18에 확인한 공식 자료를 바탕으로 한 개념 비교다. 지원 범위·연동 방식·요금은 사용할 버전과 상품에서 다시 확인해야 한다. 이 문서에는 제품 설치나 운영 배포를 완료했다는 의미가 없다.

## 시작한 질문

로그·메트릭·트레이스를 직접 수집할 수 있다면 APM은 무엇을 더 해줄까? Agent를 붙이면 저장과 운영도 해결되는 것일까? 유료 서비스를 사용하지 않고 오픈소스를 조합한다면 어떤 부분을 우리가 맡게 될까?

## 내가 이해한 내용

### 1. APM은 애플리케이션 성능을 관측·분석하는 활동과 도구를 가리킨다

APM은 문맥에 따라 Application Performance Monitoring 또는 Management로 풀어 쓴다. 여기서는 요청 흐름, 처리 시간, 오류, 호출 관계 등을 관측하고 문제를 분석하도록 돕는 활동과 제품을 다룬다.

예를 들어 [Datadog APM](https://docs.datadoghq.com/tracing/)은 분산 추적, 대시보드, 다른 관측 데이터와의 연결을 제공한다. [Pinpoint](https://github.com/pinpoint-apm/pinpoint)도 분산 애플리케이션의 호출 관계와 트랜잭션 흐름을 분석하는 APM 도구다.

로그를 많이 쌓는 것만으로 같은 기능이 자동으로 생기지는 않는다. **데이터를 연결하고, 검색 범위를 좁히고, 어느 작업에 시간이 걸렸는지 빠르게 찾게 하는 기능**이 필요하다.

대시보드의 요청량·오류율·처리 시간 그래프에는 메트릭이 많이 사용된다. 제품에 따라 로그나 Span에서 집계한 통계를 보여주기도 하므로, 화면의 숫자가 어떤 원천과 샘플링 정책에서 계산됐는지 확인한다. [Datadog APM Metrics](https://docs.datadoghq.com/tracing/metrics/)

### 2. 제품이 달라도 기본 구조는 비슷하다

```text
업무 애플리케이션
  → 계측: 로그·메트릭·Span 생성
  → 수집·전송·처리
  → 저장소
  → 검색·집계·대시보드·요청 흐름 분석·알림
```

각 제품은 이 단계들을 묶어 제공하거나, 일부 기능만 담당한다. 그래서 이름만 나열하기보다 **어느 단계를 제공하고 누가 운영하는가**를 비교해야 한다.

### 3. 계측 방식과 배포 모델을 나눠서 비교한다

| 비교 축 | 확인할 질문 |
| --- | --- |
| 계측 방식 | Agent·SDK·수동 코드 중 무엇으로 데이터를 만드는가? 어떤 프레임워크·작업을 지원하는가? |
| 전송 규격 | OTLP 또는 제품 전용 규격 중 무엇을 사용하는가? 지원 신호와 변환 범위는 무엇인가? |
| 배포 모델 | 백엔드를 사업자가 운영하는가, 우리가 운영하는가, 일부만 나눠 운영하는가? |
| 종속 지점 | 도구 교체 시 계측 코드, 속성, Exporter, 저장 구조, 쿼리·대시보드 중 무엇을 바꿔야 하는가? |

**Agent 사용과 SaaS 사용은 다른 선택이다.** Agent는 데이터를 만드는 쪽의 구성이고, SaaS·직접 운영은 수집 이후 시스템을 누가 맡는지에 대한 선택이다.

```text
Agent로 계측 → 사업자의 수집·저장·분석 서비스
Agent로 계측 → 우리가 운영하는 수집·저장·분석 시스템
```

두 구성 모두 가능하다. 자동 계측으로 지원되는 호출을 수집해도 Agent의 설치·연결 설정은 필요하고, 업무상 필요한 추가 계측이나 로그 정책도 남는다.

OTLP를 사용하면 전송 지점의 변경 부담을 줄일 수 있지만, 제품별 검색 언어·대시보드·알림·보관 정책까지 자동으로 호환되는 것은 아니다. 표준 전송 규격과 종속 지점이 겹치는 부분은 있어도 같은 비교 축은 아니다.

### 4. 대표 제품은 어떤 흐름으로 사용하는가?

| 제품·구성 | 대표적인 사용 흐름 | 우리가 확인·운영할 부분 |
| --- | --- | --- |
| WhaTap(와탭) | 애플리케이션에 Agent 적용 → 와탭 수집 서버 → 서비스 화면에서 분석 | 프로젝트 연결, Agent 설정, 지원 환경, 업무 계측·수집 범위 |
| Datadog | 지원되는 계측 방식·SDK와 Agent 등을 구성 → Datadog 백엔드 → APM·로그·메트릭 화면 | 계측·전송 설정, 데이터 연결, 수집량·보관 정책·사용 상품 |
| Pinpoint | Agent → Collector → 저장소 → Pinpoint Web | 직접 운영하는 구성에서는 수집·저장·조회 구성 요소의 설치·업데이트·장애 대응 |
| Elastic APM | 대표적인 직접 운영 구성: 계측 → APM Server → Elasticsearch → Kibana | 직접 운영 또는 Elastic Cloud 선택, 데이터 수집 경로·저장·인덱스·조회 운영 |
| AWS CloudWatch·X-Ray | Agent·SDK·OTel 계측 등으로 데이터 전달 → AWS의 로그·메트릭·Trace 기능에서 분석 | 권한, 수집·연동 설정, 로그 그룹·보관·추적 정책, 사용 기능별 비용 |

표는 각 제품의 모든 배포 방식을 제한하는 정의가 아니라, 학습한 구조를 비교하는 예다.

#### WhaTap·Datadog: 서비스 백엔드를 이용하는 구성

와탭 Java Agent는 지원되는 작업에서 데이터를 추출해 설정한 수집 서버로 전달한다. 설치 안내에 따라 프로젝트 연결 정보와 JVM 설정 등을 적용한다. **백엔드를 서비스로 이용하더라도 우리 애플리케이션 쪽 설정은 필요하다.** [WhaTap 설치 문서](https://docs.whatap.io/java/install-agent)

Datadog도 지원되는 Agent·SDK·자동 계측 경로를 구성한다. 공식 문서는 자동 생성 Span에 더해 필요한 사용자 정의 Span을 추가할 수 있다고 설명한다. OTel을 이용하는 연동 경로도 있으므로, 제품명만 보고 한 가지 계측 방식으로 고정해서 생각하지 않는다. [Datadog Application Instrumentation](https://docs.datadoghq.com/tracing/trace_collection/)

#### Pinpoint: 오픈소스 APM 제품을 직접 운영하는 구성

Pinpoint 자체도 오픈소스 APM이다. 따라서 “Pinpoint는 서비스, 오픈소스 조합은 다른 종류”로 완전히 나누기는 어렵다.

Agent가 만든 정보를 Collector가 받아 저장하고, Web 화면에서 분석하는 구성 요소가 제공된다. 직접 운영할 때는 이 구성 요소와 필요한 저장소를 우리가 관리한다. Web 기능을 처음부터 직접 만드는 것과는 다르다. 기능·버전에 따라 저장·보조 구성 요소가 달라질 수 있으므로 공식 배포 구성을 확인해야 한다. [Pinpoint 공식 저장소](https://github.com/pinpoint-apm/pinpoint), [공식 배포 구성](https://github.com/pinpoint-apm/pinpoint-docker)

#### Elastic: Elasticsearch에 저장하고 Kibana에서 분석

전형적인 직접 운영 경로에서 APM Server는 Agent의 성능 데이터를 처리해 Elasticsearch 문서로 저장한다. Kibana는 그 데이터를 검색·분석하는 화면을 제공한다. **Elasticsearch는 로그 저장에도 사용할 수 있지만, 로그 때문에만 쓰는 저장소는 아니다.**

`ES`는 이 문맥에서 Elasticsearch의 약자다. Elasticsearch를 사용하면 저장뿐 아니라 인덱스, 조회 성능, 보관·삭제 정책도 함께 운영해야 한다. Elastic Cloud를 이용하는 선택도 있다.

위 표의 경로가 모든 Elastic 구성의 필수 경로는 아니다. 현재 공식 문서는 신규 사용자에게 Elastic Agent 또는 Managed OTLP를 통한 OTel 경로도 안내한다. 사용할 배포 모델의 최신 안내를 확인한다. [Elastic APM Server](https://www.elastic.co/docs/solutions/observability/apm/apm-server)

#### CloudWatch·X-Ray: 로그·메트릭·추적 기능의 역할을 구분

CloudWatch는 여러 모니터링 기능을 포함한다. CloudWatch Logs는 로그를 받아 보관·검색하는 기능이고, X-Ray는 요청 흐름과 관련된 추적 기능을 제공한다. 둘을 연결해 분석할 수 있다. [AWS 연동 설명](https://docs.aws.amazon.com/xray/latest/devguide/xray-services-cloudwatch.html)

호스트의 로그 파일을 전송하는 설정만으로 애플리케이션 내부 Span이 모두 만들어지지는 않는다. Trace를 만들려면 지원되는 SDK·OTel 계측 등의 경로도 구성해야 한다. AWS는 OTel을 통해 로그·메트릭·트레이스를 보내는 안내도 제공한다. [AWS OTel 안내](https://docs.aws.amazon.com/en_en/AmazonCloudWatch/latest/monitoring/GettingSetup.html)

서비스를 여러 개 별도로 설치하는 그림보다는, **필요한 AWS 기능을 활성화하고 권한·수집 경로를 연결하는 그림**으로 이해한다. 사용 기능과 보관·조회에 따른 비용은 따로 확인해야 한다.

### 5. 오픈소스 조합에서는 역할별 백엔드를 연결한다

학습에서 이야기한 한 가지 조합은 다음과 같다.

| 역할 | 도구 예 | 담당 내용 |
| --- | --- | --- |
| 공통 계측·전달 | OpenTelemetry | 로그·메트릭·트레이스의 생성·처리·전달 |
| 트레이스 백엔드 | Jaeger 또는 Zipkin | Span 수집·저장·조회, 요청 흐름 분석 |
| 메트릭 백엔드 | Prometheus | 시계열 수집·저장·쿼리 |
| 로그 백엔드 | Loki 또는 Elasticsearch | 로그 수집·저장·검색 |
| 통합 조회 화면 | Grafana | 설정한 데이터 소스들을 조회하고 시각화 |

이 도구들은 이미 구현된 기능을 제공한다. Jaeger나 Prometheus를 애플리케이션의 단순한 의존성으로 추가한다고 수집·저장 시스템 전체가 실행되는 것은 아니다. **애플리케이션의 계측 라이브러리와 데이터를 받는 백엔드 프로그램을 구분한다.**

아래는 Collector 없이 백엔드에 연결하는 개념 예시다. 실제 전송 경로는 사용할 SDK·Exporter·백엔드의 지원에 맞춰 구성해야 한다.

```text
계측된 애플리케이션
  ├─ Span Exporter ───────────→ Jaeger ─→ 트레이스 저장소
  ├─ 메트릭 노출 지점 ← 수집 ── Prometheus ─→ 내장 시계열 저장소
  └─ Log Exporter ────────────→ Loki ─→ 로그 저장소

Grafana
  ├─ Prometheus에 메트릭 질의
  ├─ Jaeger에 트레이스 질의
  └─ Loki에 로그 질의
```

이 예제의 메트릭 경로에서는 Prometheus가 노출된 endpoint를 주기적으로 가져온다. 다른 계측·전송 경로도 가능하므로 모든 메트릭이 반드시 같은 방식으로 이동하는 것은 아니다. 중간에 Collector를 두어 처리·분기하는 구성도 선택할 수 있다.

Jaeger는 자체 Trace 조회 화면도 제공한다. Grafana는 여러 데이터 소스를 함께 탐색하기 위한 선택이며, 모든 백엔드가 Grafana 없이는 조회할 수 없다는 의미가 아니다.

### 6. 모은 데이터는 실제로 어디에 저장될까?

| 도구 | 저장에 관한 핵심 |
| --- | --- |
| Prometheus | 내장 로컬 TSDB에 시계열을 저장한다. 보관 기간·용량과 필요 시 외부 연동을 설정한다. |
| Jaeger | 저장 백엔드를 선택한다. 기본 메모리 저장만으로 시작한 예제는 영구 보관 구성이 아니다. Badger 같은 내장 저장소나 외부 저장소를 구성할 수 있다. |
| Loki | 로그와 인덱스에 필요한 저장 구성을 정한다. 로컬 실습의 파일 저장과 운영용 Object Storage 구성을 구분한다. |
| Grafana | 대시보드·사용자·데이터 소스 등의 자체 설정을 저장하고, 관측 데이터는 연결된 백엔드에 질의한다. |

Jaeger의 Badger는 별도 DB 서버 없이 사용할 수 있는 로컬 영속 저장 방식이며 단일 노드에 적합하다. [Jaeger Badger](https://www.jaegertracing.io/docs/2.22/storage/badger/)

Loki의 파일 저장은 작은 실습·검증에 사용할 수 있지만, 공식 문서는 운영 환경의 지원 대상 저장 방식으로 안내하지 않는다. 운영에서는 Object Storage 등 해당 환경에 맞는 구성을 선택한다. [Loki Filesystem Storage](https://grafana.com/docs/loki/latest/operations/storage/filesystem/)

Grafana의 데이터 소스 연결은 데이터의 조회 대상 설정이다. 연결했다고 모든 로그·메트릭·트레이스 원본이 Grafana의 설정 DB로 복사되는 것은 아니다. [Grafana Data Sources](https://grafana.com/docs/grafana/latest/datasources/)

### 7. 직접 운영할 때 무엇을 관리해야 할까?

전용 백엔드가 저장 구조·조회 기능을 제공하므로 이를 모두 처음부터 만들 필요는 없다. 대신 수신 경로와 공통 속성, 인증·접근 권한, 보관·삭제, 업데이트, 장애 대응, 데이터 연결을 구성하고 운영한다.

예를 들어 Trace를 Jaeger에 보내고 로그를 Loki에 보내기만 해서 조회 화면의 연결까지 항상 자동 완성되는 것은 아니다. Trace ID가 기록되고 유지되어야 하며, 화면의 데이터 소스와 링크 설정도 맞아야 한다.

오픈소스 사용은 제품 이용료와 별개로 인프라·운영 작업이 필요하다는 뜻이다. SaaS는 이 운영의 일부를 사업자에게 맡기는 대신 상품의 수집·보관·조회 정책과 비용을 검토한다. 어느 방식이 무조건 유리하다는 결론보다 **제공되는 기능과 우리가 맡는 책임**을 비교한다.

### 8. PostgreSQL에 직접 저장하는 방식은 어디가 다를까?

전용 관측 백엔드 대신 일반 DB에 저장하는 구성을 만들 수도 있다.

```text
애플리케이션의 OTel 계측
  → SDK의 배치·주기적 Export
  → 사용자 정의 DB Exporter
  → PostgreSQL의 신호별 저장 구조
  → 직접 만든 조회 API·화면
```

OTel SDK의 Exporter 확장 지점을 이용할 수 있지만, PostgreSQL 행으로 변환하고 SQL을 실행하는 부분은 직접 구현해야 한다. PostgreSQL 메트릭을 관측하는 `postgres_exporter` 같은 도구와, 관측 데이터를 PostgreSQL에 저장하기 위해 만드는 Exporter는 서로 다른 역할이다. [OTel Java SDK](https://opentelemetry.io/docs/languages/java/sdk/)

이 구성에서는 로그 검색, Trace ID로 Span 조회, 메트릭 집계, 대시보드, 보관·삭제와 실패 처리를 직접 설계한다. 동일한 DB 인스턴스의 업무 데이터와 자원을 공유한다면 쓰기·조회·정리 작업의 영향을 검증해야 한다. 별도 테이블이나 스키마로 구분하는 것만으로 자원이 격리되는 것은 아니다.

따라서 다음 세 선택은 구분된다.

| 선택 | 주로 제공받는 것 | 직접 맡을 일 |
| --- | --- | --- |
| 관리형 APM 서비스 | 서비스의 수집·저장·검색·분석 기능 | 애플리케이션 연결·계측, 정책·권한·비용 관리 |
| 전용 OSS 백엔드 조합 | 각 백엔드의 저장·쿼리·분석 기능 | 구성 요소의 운영과 상호 연결 |
| 일반 DB 기반 직접 구현 | DB의 저장·쿼리 기능과 OTel의 계측 기반 | 관측 데이터 모델·저장·검색·집계·화면·운영 정책 구현 |

PostgreSQL과 OTel도 오픈소스이므로, 마지막 방식은 “오픈소스를 전혀 사용하지 않는다”보다는 **전용 관측 백엔드 없이 직접 구현한다**고 표현하는 편이 정확하다.

학습 대화에서 OTel과 기존 PostgreSQL, 최근 7일 보관을 활용하는 초기안을 검토했다. 실제 자원 소모가 더 작거나 운영에 충분한지는 아직 검증한 결과가 없다. 이 문서는 특정 제품의 도입 결정서나 배포 완료 기록이 아니다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| Agent를 쓰면 저장소와 조회 서버도 알아서 운영된다 | 계측 방식과 배포·운영 주체는 별개의 축이다. |
| Pinpoint·Elastic은 서비스이고, 오픈소스는 별도 종류다 | 제품마다 제공·운영 방식이 다르다. Pinpoint 자체도 OSS APM이며 Elastic에도 관리형·직접 운영 선택이 있다. |
| 오픈소스를 쓰면 화면을 모두 직접 만들어야 한다 | Jaeger·Grafana 등 이미 제공되는 조회 기능을 구성해 사용할 수 있다. |
| Grafana가 세 종류의 원본 데이터를 모두 저장한다 | 데이터 소스에 질의하고, 자체 설정은 별도로 저장한다. |
| Prometheus가 로그·트레이스도 모두 저장한다 | 이 조합에서는 메트릭을 담당한다. 로그·트레이스에는 각각 맞는 백엔드를 연결한다. |
| DB에 직접 저장하면 전용 백엔드의 기능도 그대로 생긴다 | 연결·조회·집계·화면·관리 기능을 별도로 만들어야 한다. |

## 예제와 확인 결과

- 제품별 흐름과 OSS 조합은 공식 문서를 바탕으로 정리한 개념도다.
- 특정 제품을 설치하거나 계측 데이터를 전달해 비교한 성능 결과는 없다.
- 실제 실습에서는 동일 요청의 메트릭 반영, Trace 조회, 로그 연결, 보관 만료와 수집 실패를 확인해야 한다.

## 남은 질문

- 우리가 꼭 필요한 검색·분석 기능은 어디까지인가?
- 같은 요청을 기준으로 메트릭→Trace→Log 이동을 어느 화면에서 구현할 것인가?
- 자동 계측 범위 밖의 업무 Span과 로그는 무엇인가?
- 전용 백엔드 조합과 직접 구현의 운영·개발 부담을 어떤 기준으로 비교할 것인가?

## 참고 자료

- [WhaTap — Java 에이전트 설치](https://docs.whatap.io/java/install-agent)
- [Datadog — APM](https://docs.datadoghq.com/tracing/)
- [Datadog — Application Instrumentation](https://docs.datadoghq.com/tracing/trace_collection/)
- [Datadog — APM Metrics](https://docs.datadoghq.com/tracing/metrics/)
- [Pinpoint — 공식 저장소](https://github.com/pinpoint-apm/pinpoint)
- [Pinpoint — 공식 배포 구성](https://github.com/pinpoint-apm/pinpoint-docker)
- [Elastic — APM Server](https://www.elastic.co/docs/solutions/observability/apm/apm-server)
- [AWS — CloudWatch integration with X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/xray-services-cloudwatch.html)
- [AWS — OTel로 CloudWatch에 데이터 보내기](https://docs.aws.amazon.com/en_en/AmazonCloudWatch/latest/monitoring/GettingSetup.html)
- [Jaeger — Getting Started](https://www.jaegertracing.io/docs/2.22/getting-started/)
- [Jaeger — Badger Storage](https://www.jaegertracing.io/docs/2.22/storage/badger/)
- [Zipkin — Architecture](https://zipkin.io/pages/architecture.html)
- [Prometheus — Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Loki — Filesystem Storage](https://grafana.com/docs/loki/latest/operations/storage/filesystem/)
- [Loki — OpenTelemetry 수집](https://grafana.com/docs/loki/latest/send-data/otel/)
- [Grafana — Data Sources](https://grafana.com/docs/grafana/latest/datasources/)
- [OpenTelemetry — Java SDK](https://opentelemetry.io/docs/languages/java/sdk/)

## 정리 이력

- 2026-10-18: 계측·전송·배포·종속 지점으로 비교하고, 관리형 서비스·전용 OSS 백엔드·직접 구현의 책임 범위를 정리했다.
