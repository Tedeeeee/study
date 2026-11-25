# Metric은 무엇으로 잴까?

[목차](README.md) · [이전: Trace와 Span은 어떻게 생겼고, 서비스 경계를 어떻게 넘을까?](trace-and-context-propagation.md)

## 다루는 범위

OpenTelemetry Metric API의 Instrument 네 종류(Counter, UpDownCounter, Histogram, Gauge)가 각각 어떤 값을 재는지 비교한다. 같은 종류 안에서 동기와 비동기 중 무엇을 고를지, 그 기준도 정리한다.

2장에서 본 "Metric은 앱 메모리의 숫자를 주기적으로 내보낸다"는 내용을 전제로 한다. Metric과 Trace를 Exemplar로 잇는 방법은 5번 주제에서 다룬다.

## 시작한 질문

2장에서 Metric은 라벨 조합마다 숫자 하나를 들고 있다고 했다. 그런데 요청 수, 응답 시간, CPU 사용률은 성격이 전혀 다른데 다 같은 "숫자 하나"로 잴 수 있을까? p99는 그 숫자로 어떻게 구할까?

## 내가 이해한 내용

### 1. Instrument는 숫자를 어떤 규칙으로 바꿀지 정한 도구다

| Instrument | 값의 움직임 | 앱 메모리에 들고 있는 것 | 예시 |
| --- | --- | --- | --- |
| **Counter** | 올라가기만 한다 | 누적 합계 하나 | 요청 수, 에러 수, 결제 실패 수 |
| **UpDownCounter** | 오르내린다 (더할 수 있는 개수) | 합계 하나 | 처리 중 요청 수, WebSocket 연결 수, 큐 대기 작업 수 |
| **Histogram** | 측정값의 분포 | 칸(bucket)별 개수 + count + sum | 응답 시간, 업로드 파일 크기 |
| **Gauge** | 그 순간의 현재 값 (더하면 이상한 값) | 마지막 값 하나 | CPU 사용률, 디스크 사용률 |

### 2. Counter: 횟수를 세고, 속도는 저장소가 계산한다

```java
// 설명용
requests.add(1, Attributes.of(ROUTE, "/orders", STATUS, 200));
```

앱은 +1만 한다. RPS, 에러율 같은 속도와 비율은 저장소에서 `rate()`로 두 시점의 차이를 계산한다. 2장의 누적값(Cumulative) 얘기와 이어진다.

```promql
rate(http_requests_total[1m])                                          # RPS
rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])  # 에러율
```

Counter를 그대로 그리면 계속 올라가는 선이라 의미가 없다. UpDownCounter는 값 자체가 "지금 몇 개"라서 바로 그린다. 둘을 바꿔 쓰면 안 된다. 에러 수를 UpDownCounter로 잡으면 저장소가 앱 재시작으로 0이 된 걸 리셋으로 보정하지 못하고, 처리 중 요청을 Counter로 잡으면 −1을 못 넣는다.

Spring Boot의 `http.server.requests`는 Counter가 아니라 Histogram 계열(Micrometer Timer)이다. Histogram이 같이 세는 `_count`에 `rate()`를 씌워 RPS를 구하기도 한다.

주문 수 같은 비즈니스 값도 Counter로 많이 잰다. 목적은 "평소보다 확 줄었다"를 잡는 **장애 감지**다. 정산이나 보고에 쓰는 정확한 숫자는 DB에서 센다. Metric은 아직 안 보낸 값이 앱 종료로 유실될 수 있고, `increase()`는 외삽해서 소수가 나오기도 하고, 보관 기간도 짧다.

### 3. Histogram: 분포를 남겨야 p99를 구한다

Counter로는 건수와 합계까지만 알 수 있다. 평균은 구해도 느린 1%가 묻힌다. Histogram은 값이 속한 칸의 개수를 센다.

```text
http.server.duration{route="/orders"}
  ≤ 0.1초 : 70
  ≤ 0.5초 : 20
  ≤ 1초   : 8
  ≤ +Inf  : 2
  count   : 100
  sum     : 19.9
```

- 원래 값(0.12초)은 저장하지 않는다.
- p99는 저장소가 `histogram_quantile()`로 **칸 안을 보간해 추정**한다. 그래서 칸 경계를 내 API 응답 시간대에 촘촘하게 잡아야 정확하다.
- 칸 하나하나가 시계열이다. 칸이 10개면 라벨 조합 하나당 약 12개가 된다. 고카디널리티 라벨이 Counter보다 더 치명적이다.

### 4. Gauge: 지금 이 순간의 상태

| | UpDownCounter | Gauge |
| --- | --- | --- |
| 기록 방법 | 변화량을 더한다 (+1, −1) | 현재 값 자체를 기록한다 (72%) |
| 서버 여러 대를 더하면 | 의미 있다 (연결 100 + 150 = 250) | 대개 의미 없다 (CPU 70% + 80%) |

인프라(CPU, 메모리, 디스크)뿐 아니라 앱에도 있다. 커넥션 풀 사용률, 캐시 항목 수, consumer lag, 마지막 배치 성공 시각 같은 것이다. Gauge는 내보내는 순간의 값만 남아서 그 사이에 잠깐 튄 값은 놓칠 수 있다.

**이벤트마다 생기는 측정값은 Histogram, 지금 이 순간의 상태는 Gauge다.** 업로드 크기를 Gauge로 잡으면 마지막 값 하나만 남고 나머지는 덮어써진다.

### 5. 동기와 비동기: 값을 누가, 언제 넣나

스레드나 `@Async`와는 관계없다.

| | 동기 | 비동기 (Observable*) |
| --- | --- | --- |
| 방법 | 이벤트가 일어나는 코드에서 `add()`, `record()` | 읽는 방법(callback)만 등록 → SDK가 **내보낼 때마다** 불러서 읽어 감 |
| 비유 | 문 앞에서 드나들 때마다 딸깍 센다 | 5분마다 CCTV로 매장 안을 센다 |

```java
// 비동기 예시 (설명용)
meter.gaugeBuilder("jvm.memory.used")
     .buildWithCallback(m -> m.record(runtime.totalMemory() - runtime.freeMemory()));
```

고르는 기준은 "볼 수 있나"가 아니라, **어떤 방식으로 볼 수 있나**다.

| | ① 변화마다 불러 주는 자리(이벤트)가 있다 | ② 현재 값을 읽을 수만 있다 |
| --- | --- | --- |
| 예 | `catch` 블록, 요청 끝, WebSocket `afterConnectionEstablished` | OS의 CPU, JVM 힙, `HikariPool.getActiveConnections()`, `getQueueSize()` |
| 가능한 방식 | 동기 (②도 있으면 비동기도 가능) | 비동기만 |

- 둘 다 가능하면 값을 이미 들고 있는 곳에서 읽는 쪽이 보통 간단하다. 예를 들어 세션 목록을 이미 들고 있으면 `size()`를 비동기로 읽는다. 목록이 없으면 이벤트 때 +1/−1 한다.
- CPU 사용률처럼 연속적으로 변하는 값은 "+1 되는 순간" 자체가 없다.

| | 동기 | 비동기 |
| --- | --- | --- |
| 올라가기만 | Counter | ObservableCounter |
| 더할 수 있는 개수 | UpDownCounter | ObservableUpDownCounter |
| 더하면 이상한 현재 값 | Gauge | ObservableGauge |
| 분포 | Histogram | (없음) |

Spring의 Micrometer에는 UpDownCounter 개념이 없어서 HikariCP 활성 커넥션 수를 Gauge로 등록한다. OTel 기준으로는 ObservableUpDownCounter다.

## 헷갈렸던 점

- **"Counter는 TPS, RPS 잴 때 많이 쓰냐"고 물었다.**
  - 맞다. Counter에 `rate()`를 씌운다. 다만 Spring Boot에서는 Histogram의 `_count`로 구하는 경우가 많다.
- **"오늘 총 주문 수 같은 것도 메트릭으로 재냐", 그리고 "실무에서는 안 그러지?"라고 물었다.**
  - 실무에서도 주문 Counter는 많이 둔다. 안 하는 건 그 숫자를 정확한 공식 숫자로 쓰는 것이다.
  - 목적은 장애 감지다. 기준은 "이 숫자가 갑자기 변하면 장애일 수 있나?"다.
- **가지고 있던 모니터링 화면을 Instrument에 대 보았다.**
  - RPS 패널은 Counter, 응답 시간 P50/P95/P99는 Histogram, 현재 처리 중 요청은 UpDownCounter로 대응된다.
  - 실제 구현 방식은 코드를 봐야 알고, 여기서는 차이를 이해하는 예시로만 썼다.
- **"Gauge는 대부분 현재 사용량 조회 같은 거냐"고 물었다.**
  - 맞다. 서버나 OS뿐 아니라 앱 상태 값도 있다.
- **문제 1 첫 시도에서 WebSocket 연결 수와 Hikari 사용 중 커넥션을 둘 다 "Gauge · 비동기"로 골랐다.**
  - "현재 상태"라는 느낌으로 골랐다.
  - 둘 다 더하면 의미 있는 개수라 UpDownCounter 계열이다. WebSocket은 이벤트 자리가 있어 동기, Hikari는 읽기만 돼서 비동기다.
- **동기/비동기 기준을 여러 번 고쳤다.**
  - "코드를 타야 알 수 있는 것과 SDK가 상태를 확인하는 것은 비동기"라고 했다. 앞쪽은 반대로 동기였다.
  - "현재 상태를 계속 보고 싶으면 비동기, 변화를 알고 싶으면 동기"라고 했다. 하지만 동기 UpDownCounter도 대시보드에는 현재 값으로 보인다. "보고 싶은 것"으로는 구분이 안 된다.
  - "WebSocket은 코드가 볼 수 있나?"라고 물었고, 연결과 끊김 때 프레임워크가 내 핸들러를 불러 준다는 답을 들었다.
  - 그러자 "그렇게 따지면 OS, JVM, 라이브러리도 코드에서 다 볼 수 있지 않냐, WebSocket도 그런 것 아니냐"고 짚었다. 맞는 지적이었다.
  - 그래서 기준을 **① 변화마다 불러 주는 이벤트 자리가 있나 / ② 현재 값을 읽을 수만 있나**로 고쳤다. WebSocket은 둘 다 가능해서 상황에 따라 고르는 것이다.

### 확인 문제

**문제 1.** 결제 서비스에 Metric을 추가한다. Instrument와 동기/비동기, 이유는? (1) 결제 실패 횟수 (2) PG사 API 응답 시간, p99를 보고 싶다 (3) WebSocket 연결 수, 연결과 끊김 때 내 코드가 실행된다 (4) HikariCP 사용 중 커넥션 수, 풀 객체가 숫자를 들고 있다

- 내 답:
  1. Counter · 동기. 올라가기만 하니까.
  2. Histogram · 동기. 총 요청을 비교해 각 걸린 시간을 시계열로 저장하니까.
  3. Gauge · 비동기. 현재 서버 상황을 보고 말하는 것뿐이라 SDK 실행 때마다 말하면 될 것 같다.
  4. Gauge · 비동기. 위와 같은 이유.
- 채점: 1, 2번 정답, 3번 오답, 4번 절반 정답.
  - 2번의 이유가 틀렸다. 각 값을 시계열로 저장하는 게 아니라 칸별 개수로 분포를 남긴다.
  - 3번은 UpDownCounter · 동기다. 이벤트 자리가 있고 더하면 의미 있는 개수다.
  - 4번의 비동기는 맞지만 Instrument는 ObservableUpDownCounter다.

**문제 1 다시.** (1) 쿠폰 적용 실패 횟수, `catch`가 실행된다 (2) 이미지 업로드 파일 크기, 내 메서드에서 크기를 알고 "대부분 몇 MB, 큰 파일은 얼마나 큰지" 보고 싶다 (3) `ThreadPoolTaskExecutor` 큐 대기 작업 수, `getQueueSize()`만 있다 (4) 서버 디스크 사용률, OS에 조회한다

- 내 답:
  1. Counter · 동기. 실패하면 올라가고 내려갈 일이 없고, 로직을 타야 알 수 있으니 동기.
  2. Gauge? · 동기. 로직을 타야 알 수 있어 동기. 평균이라면 Histogram이 맞나?
  3. UpDownCounter · 비동기. 로직을 안 타도 SDK가 실행될 때마다 판단돼야 한다.
  4. Gauge · 비동기. 로직을 안 타고 현재 사용량을 계속 체크한다.
- 채점: 정답. 동기/비동기는 넷 다 맞았다.
  - 2번은 스스로 의심한 Histogram이 정답이다. 분포와 꼬리를 보려면 Histogram이고, Gauge는 마지막 값 하나만 남는다.
  - 3, 4번은 정확히는 ObservableUpDownCounter, ObservableGauge다. 기준 ②(서버끼리 더하면 의미 있나)로 둘을 가르는 이유도 붙이면 완벽하다.

## 예제와 확인 결과

이번 주제는 개념 정리라서 직접 실행한 예제는 없다. 코드와 PromQL, bucket 수치는 설명용이다.

## 남은 질문

- Spring Boot의 `http.server.requests`는 기본 설정에서 bucket을 내보내지 않아 p99를 보려면 설정이 필요하다고 알고 있다. 세미나 실습에서 확인한다. → 10~11번 주제
- 칸 경계를 자동으로 잡는 Exponential Histogram은 어떻게 동작할까?
- 동기 Gauge는 OTel 명세에 비교적 최근 추가됐다. Java SDK에서는 언제부터 쓸 수 있을까?
- Micrometer와 OTel이 같은 값을 다른 Instrument와 이름으로 부르는 문제 → 11번 주제

## 참고 자료

- OpenTelemetry 세미나 1회차 "OpenTelemetry 아키텍처 & OTLP 데이터 모델" — Metric Instrument ([VSFe/otel-seminar](https://github.com/VSFe/otel-seminar))
- [OpenTelemetry — Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [OpenTelemetry — Metrics API (Instruments)](https://opentelemetry.io/docs/specs/otel/metrics/api/)
- [OpenTelemetry — Supplementary Guidelines: Instrument selection](https://opentelemetry.io/docs/specs/otel/metrics/supplementary-guidelines/)
- [Prometheus — Histograms and summaries](https://prometheus.io/docs/practices/histograms/)
- [Prometheus — Query functions (`rate`, `increase`, `histogram_quantile`)](https://prometheus.io/docs/prometheus/latest/querying/functions/)
