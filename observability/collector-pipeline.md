# Collector는 데이터를 어떻게 처리할까?

[목차](README.md) · [이전: OTLP 메시지는 어떻게 생겼을까?](otlp-message.md)

## 다루는 범위

Collector 설정의 뼈대(receiver → processor → exporter를 signal별 pipeline으로 잇기), 자주 쓰는 processor와 그 순서, 끝난 뒤에 고르는 tail sampling, Collector를 어디에 둘지(Agent 배포와 Gateway 배포)를 정리한다.

Collector가 무엇이고 왜 쓰는지는 [7장](api-sdk-collector.md)에 정리했다.

## 시작한 질문

7장에서 "받기 → 손보기 → 보내기"라고만 했던 Collector는 실제로 어떻게 설정하고, 어디에 두는 걸까?

## 내가 이해한 내용

### 1. 파이프라인: 부품 창고와 조립 라인

| 단계 | 이름 | 예 |
| --- | --- | --- |
| 받기 | receiver | `otlp`(4317·4318), `jaeger`, `prometheus`, `hostmetrics`, `filelog` |
| 손보기 | processor | `batch`, `filter`, `attributes` |
| 보내기 | exporter | `otlp/tempo`, `prometheusremotewrite` |

```yaml
# 설명용
receivers:  { otlp: { protocols: { grpc: {}, http: {} } } }
processors: { batch: {} }
exporters:
  otlp/tempo: { endpoint: tempo:4317 }
  prometheusremotewrite: { endpoint: http://prom:9090/api/v1/write }

service:
  pipelines:
    traces:  { receivers: [otlp], processors: [batch], exporters: [otlp/tempo] }
    metrics: { receivers: [otlp], processors: [batch], exporters: [prometheusremotewrite] }
```

| 부분 | 역할 |
| --- | --- |
| 위쪽 (`receivers`, `processors`, `exporters`) | 부품을 **정의만** 한다 (부품 창고) |
| `service.pipelines` | 부품을 골라 **signal별로 연결**한다 (조립 라인) |

정의만 하고 pipeline에 안 넣은 부품은 동작하지 않는다. 같은 `otlp` receiver로 들어와도 Trace는 traces 라인, Metric은 metrics 라인을 탄다.

receiver는 두 종류다. `otlp`처럼 **문을 열고 보내 주기를 기다리는** 것과, `hostmetrics`·`filelog`처럼 **직접 읽으러 가는** 것이다. 그래서 Collector는 서비스가 보낸 데이터뿐 아니라 **자기가 있는 컴퓨터의 상태**도 읽을 수 있다.

### 2. processor는 보내기 전에 거치는 필터 체인이다

Spring 필터 체인처럼 적은 순서대로 지나가고, 데이터를 버리거나 고칠 수 있다.

| processor | 하는 일 |
| --- | --- |
| `memory_limiter` | 메모리가 넘칠 것 같으면 받기를 잠시 거절 |
| `batch` | 모아서 묶어 보내기 |
| `filter` | 버리기 (`/health` Span 등) |
| `attributes` / `transform` | 속성 지우기·고치기·더하기 |
| `tail_sampling` | Trace가 끝난 뒤 남길지 정하기 |

```text
memory_limiter → filter → attributes → batch
 (맨 앞: 넘치면 바로 막기, 버릴 건 빨리 버리기)   (맨 뒤: 다 손본 뒤 묶기)
```

SDK 안에도 processor가 있다(law-erp의 `BatchSpanProcessor`). 앱 쪽은 묶어서 넘기는 정도이고, 버리기·고치기·tail sampling 같은 본격적인 손보기는 Collector에서 한다.

**지우기**: Collector에서 개인정보 속성을 지울 수 있지만, 서비스는 여전히 신경 써야 한다. Collector는 두 번째 안전망이다.

| 이유 | 설명 |
| --- | --- |
| 이미 앱 밖으로 나갔다 | 앱 → Collector 구간에는 실려 다닌다 |
| 콘솔·파일 로그는 Collector를 안 거친다 | 그대로 남는다 |
| 이름을 아는 속성만 지우기 쉽다 | `exception.message`나 로그 본문 같은 자유 문장 속은 잡기 어렵다 |
| 길이 바뀔 수 있다 | Collector를 거치지 않게 바뀌면 안전망이 사라진다 |

그래서 1차는 서비스(애초에 안 넣기), 2차는 Collector(새어 나온 것 지우기)다. law-erp가 앱에서 예외 메시지를 미리 뺀 것이 1차 방어다.

**더하기**: 서비스가 모르거나 30개에 일일이 넣기 번거로운 공통 정보를 붙인다. `deployment.environment=prod`, 쿠버네티스 Pod 이름과 Namespace, 클라우드 리전, 담당 팀 같은 것이다. 서비스가 직접 보내도 되지만 공통 정보는 한 곳에서 붙이는 게 관리하기 편하다.

### 3. tail sampling: 끝난 뒤에 고르기

| | 앞에서 정하기 (2·3장) | tail sampling |
| --- | --- | --- |
| 누가 | 서비스 | Collector |
| 언제 | 요청이 시작될 때 | Trace가 끝난 뒤 |
| 결과를 보고 정하나 | 못 한다 | 에러, 지연을 보고 정한다 |
| 단점 | 에러 난 Trace가 버려질 수 있다 | 끝날 때까지 메모리에 들고 있어야 한다 |

서비스가 끝나고 고르지 못하는 이유는 두 가지다.

1. **결정을 바로 넘겨야 한다.** 다음 서비스를 호출하는 순간 `traceparent`의 flags에 "남긴다/안 남긴다"를 실어 보내야 Trace가 반쪽이 안 된다.
2. **자기 조각만 본다.** order는 payment에서 에러가 났는지 모른다. 전체를 보려면 모든 서비스의 Span이 한곳에 모여야 한다.

그래서 tail sampling을 쓰면 서비스는 **일단 100% 보낸다.** 미리 10%만 보내면 버린 90% 안의 에러를 Collector가 볼 기회조차 없다. 대신 네트워크와 Collector 메모리 비용이 든다. 트래픽이 아주 많으면 서비스에서 먼저 줄이고 Collector에서 다시 고르는 식으로 섞기도 한다.

Trace를 거르는 시점을 모으면 세 가지다.

```text
① 계측 모듈: 애초에 Span을 안 만든다 (law-erp 미들웨어 Span 무시)
② 서비스 샘플러: 시작할 때 확률로 정한다 (Spring 0.1, 세미나 1.0)
③ Collector: 받은 뒤 버리거나(filter) 끝난 뒤 골라 남긴다(tail_sampling)
```

### 4. Agent 배포와 Gateway 배포

둘 다 **같은 Collector 프로그램, 같은 이미지**다. 다른 건 **어디에 띄우나**와 **설정에 뭘 적나**뿐이다. 그래서 Agent와 Gateway는 종류 이름이 아니라 역할 이름이다. 같은 Spring 앱을 `application-dev.yml`과 `application-prod.yml`로 다르게 띄우는 것과 비슷하다.

| | Agent | Gateway |
| --- | --- | --- |
| 비유 | 집 앞 우체통 | 동네 우체국 |
| 위치 | 각 서버 옆 | 중앙 |
| 보는 범위 | **자기 서버 하나** | **모든 서버** |
| 맡는 일 | 받기, 서버 상태 읽기, 서버 정보 붙이기, 묶기, 잠시 들고 있기, Gateway로 넘기기 | 어디로 보낼지, API 키, 개인정보 제거, tail sampling, signal별 백엔드 분배 |

> Agent = 현장에서 모으고 다듬어 넘기는 곳, Gateway = 무엇을 남길지 정하는 곳.

Agent는 주로 거르지 않는다. 버리는 결정은 규칙을 한 곳에 두려고 Gateway에서 한다. 다만 `/health`처럼 버릴 게 확실하고 양이 많으면 네트워크를 아끼려고 Agent에서 미리 버리기도 한다.

Agent가 있어서 생기는 장점은 세 가지다.

| 장점 | 한 줄 |
| --- | --- |
| 서버 정보 수집 | 그 기계 옆에 있어야 기계 전체 상태와 로그 파일을 읽을 수 있다 |
| 앱 부담 감소 | 앱은 `localhost`에 던지고 끝, 중앙이 끊겨도 Agent가 잠시 들고 있다 |
| 피해 범위 축소 | 한 서버 폭주는 그 서버 Agent에서 먼저 막는다 |

서버마다 Collector가 생겨 관리할 게 늘지만, 모든 Agent가 "Gateway로 넘긴다"는 **같은 설정**을 쓰고 거의 안 바뀐다. 쿠버네티스가 일괄 배포한다. 자주 바뀌는 정책은 Gateway 한 곳에 있다.

**Gateway만 있으면 못 보는 것**은 서비스 상황이 아니라 **기계 전체**다.

| 무엇 | Gateway만 있어도 보이나 | 누가 보내나 |
| --- | --- | --- |
| 서비스별 Trace, 에러율, 로그 | ○ | 앱 (Resource의 `service.name`이 붙어 Gateway를 그대로 통과) |
| 앱 프로세스의 CPU, JVM 힙, GC | ○ | 앱 안의 Java Agent (런타임 메트릭) |
| 기계 전체 CPU, 메모리, 디스크 | ✕ | 서버 옆 Agent Collector |
| 앱이 파일에만 쓰는 로그 | ✕ | 서버 옆 Agent Collector |

기계 상태는 그 기계 안에서만 읽을 수 있다. 작업 관리자가 내 컴퓨터 CPU만 보여 주는 것과 같다. Gateway는 다른 기계에 있어서 못 읽는다.

**tail sampling과 Gateway 여러 대**: 한 Trace의 Span이 여러 Gateway에 흩어지면 둘 다 반쪽만 본다. 그래서 앞단에서 같은 trace_id는 항상 같은 Gateway로 가도록 나눈다(`loadbalancing` exporter).

규모별로는 이렇게 고른다.

| 규모 | 구성 |
| --- | --- |
| 서비스 몇 개 | Collector 하나, 또는 없이 바로 전송(law-erp) |
| 서비스 많음 | Gateway |
| 기계 상태·로그 파일까지 필요 | Agent + Gateway |

여기서 Agent는 Collector 배포 방식 이름이고, Java Agent(`-javaagent`)와는 다르다.

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| processor가 뭐지? | 보내기 전에 거치는 필터 체인이다. Spring 필터 체인과 같다. |
| processor는 Collector의 역할이다 | 9-2에서 본 것들은 Collector 부품이다. SDK 안에도 `BatchSpanProcessor` 같은 processor가 있다. |
| 그게 이미지로 띄우는 그 Collector인가? | 그렇다. 이미지 안의 부품을 YAML에서 이름으로 골라 쓴다. |
| Collector는 로그를 보낼 때 필터링한다 | 세 signal 모두이고, 버리기뿐 아니라 묶기·고치기·골라 남기기도 한다. |
| Collector에서 개인정보를 지우면 서비스는 신경 안 써도 되나? | 아니다. 콘솔 로그, 앱 → Collector 구간, 자유 문장 속 정보 때문에 서비스가 1차, Collector가 2차다. |
| 더하기는 뭐야? 서비스가 안 보내면 그만 아닌가? | 서비스가 모르는 인프라 정보나 공통 정보를 한 곳에서 붙이는 것이다. |
| 끝나고 남길지 정하는 게 서비스가 아니라 Collector인가? | 그렇다. 서비스는 자기 조각만 보고, 결정을 헤더로 바로 넘겨야 한다. |
| 그럼 서비스는 일단 전부 보내는 건가? | tail sampling을 쓰면 그렇다. |
| Trace를 거르는 작업이 분명 있었는데? | 2장의 시작 시점 샘플링(Spring 기본 10%)이다. 처음에 세 가지 거르기 방식을 한꺼번에 설명해서 이해가 안 됐다. |
| 샘플링도 끝나고 선택할 수 있는 것 아니었나? | 서비스의 샘플링은 시작할 때만 정한다. 다음 서비스에 헤더로 결정을 바로 알려야 하기 때문이다. |
| Agent를 두면 결국 서버마다 Collector를 또 다뤄야 하는 것 아닌가? | 늘긴 하지만 모두 같은 설정을 쓰고 거의 안 바뀐다. 자주 바뀌는 정책은 Gateway에 있다. |
| 중앙이 전부 control하기는 무리인가? | 못 하는 게 아니라 일을 나눈다. 본사(Gateway)는 규칙, 지점(Agent)은 현장 일이다. |
| 서비스마다 Collector가 붙으면 각각 뭘 맡나? | 받기, 서버 상태 읽기, 서버 정보 붙이기, 지키기, Gateway로 넘기기다. |
| Agent가 한 번 거른다? | 거르기보다 모으고 다듬어 넘긴다. 버리는 결정은 주로 Gateway다. |
| Gateway 하나로 모으면 각 서비스 상황은 못 보나? | 본다. `service.name`이 붙어 그대로 통과한다. |
| 그럼 서비스의 CPU·메모리는 못 보나? | 앱 프로세스 자신의 CPU·메모리는 Java Agent가 보내서 보인다. 못 보는 건 **기계 전체**다. 내가 처음에 "서버의 CPU·메모리"라고 뭉뚱그려 설명해서 헷갈렸다. |
| Agent와 Gateway의 차이를 모르겠다, 둘 다 Collector인데 다르게 할 수 있나? | 같은 프로그램이다. 위치와 설정이 다를 뿐이고, 차이는 보는 범위(자기 서버 하나 vs 모든 서버)다. |
| 왜 Agent를 달아야 기계 상태를 보나? | 기계 상태는 그 기계 안에서만 읽힌다. Collector는 보내 준 것뿐 아니라 자기가 있는 컴퓨터의 상태도 직접 읽을 수 있다. |

### 확인 문제

**Q1.** 서비스 20개. 서버(기계) CPU·메모리를 보고 싶다, 로그의 개인정보 속성을 지우고 싶다, 에러 Trace는 빠짐없이·나머지는 10%만 남기고 싶다, Trace는 Tempo·Metric은 Prometheus로 보내고 싶다. ① Collector를 어디에 어떻게 두고 요구사항을 어디서 처리하나 ② 에러 Trace를 빠짐없이 남기려면 서비스 샘플링은 어떻게 하고 왜인가

내 답:

> 1. 각 서비스 별로 collector 를 옆에 두어서 해당 정보를 추리고 gateway 로 보내주면 됩니다.
> 2. 서비스에서 샘플링은 100%로 하고 gateway 혹은 agent 쪽에서 실패한 에러 트레이스만 남기는 로직을 적으면 된다

채점: 통과. 1번은 Agent + Gateway 구성이 맞다. 요구사항별로는 서버 상태는 Agent, 개인정보 제거·tail sampling·백엔드 분배는 Gateway다. 2번은 100%가 맞다. 다만 tail sampling은 **Gateway만** 된다. Agent는 자기 서버의 Span만 봐서 다른 서비스에서 난 에러를 모른다. 100%인 이유(시작할 때는 에러를 모르고, 미리 버리면 다시 볼 수 없다)도 빠졌다. 채점 뒤에 Agent와 Gateway의 차이를 다시 물어보며 이해했다.

## 예제와 확인 결과

- 이 문서의 YAML은 모두 설명용이다. 실행해 확인하지 않았다.
- `filter`, `transform`, `tail_sampling`, `k8sattributes`, `resourcedetection`, `hostmetrics`, `filelog`, `loadbalancing`은 contrib 배포판에 있다고 알고 있다. 정확한 설정 문법은 확인하지 않았다.

## 남은 질문

- 세미나 실습 환경의 Collector 설정 파일은 어떻게 생겼나? pipeline을 직접 읽어 보기
- Java Agent가 기본으로 보내는 JVM 런타임 메트릭의 이름과 활성화 여부 (10장)
- 서비스별로 나눠 보내는 routing 부품의 현재 이름과 설정
- tail sampling 정책(에러, 지연, 확률)을 실제로 어떻게 조합해 쓰나

## 참고 자료

- [OpenTelemetry 세미나(VSFe/otel-seminar)](https://github.com/VSFe/otel-seminar)
- [OpenTelemetry — Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
- [OpenTelemetry — Collector 배포 패턴 (Agent, Gateway)](https://opentelemetry.io/docs/collector/deployment/)
- [OpenTelemetry — Sampling (head, tail)](https://opentelemetry.io/docs/concepts/sampling/)
- [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
