# "하네스"는 정확히 어디까지일까?

[목차](README.md) · [이전: 모델은 글만 만드는데, 어떻게 파일을 고칠까?](agent-loop.md)

## 다루는 범위

하네스라는 말이 가리키는 범위를 정리하고, 하네스 전체가 어떤 부품으로 이뤄지는지 지도를 그린다.

- 글마다 하네스의 정의가 갈리는 이유
- 도구에 이미 들어 있는 하네스와 사용자가 얹는 하네스
- 가장 작은 하네스(mini-swe-agent)의 부품과 끝내는 방식
- 다 갖춘 하네스(Hermes Agent)에서 더 붙는 부품
- 하네스만 바꿔도 결과가 달라지는 사례, 그리고 많이 붙일수록 좋지는 않은 이유
- 하네스 엔지니어링과 컨텍스트 엔지니어링의 관계

각 부품의 자세한 동작은 [공부 계획](README.md#공부-계획)의 해당 장에서 다룬다.

## 시작한 질문

면접관이 "하네스가 뭐예요?"라고 묻는다. 그런데 어떤 글은 "모델 빼고 전부"라고 하고, 어떤 글은 "사용자가 얹는 것"이라고 하고, 어떤 글은 "루프"라고 한다.

1. 누구 말이 맞을까? 내가 만든 하네스는 그중 어디에 해당할까?
2. 하네스 전체는 어떤 부품으로 이뤄져 있을까?
3. 하네스만 바꿔도 결과가 정말 달라질까?

## 내가 이해한 내용

### 1. 정의가 갈리는 건 서 있는 자리가 달라서다

| 누가 | 원문 정의 | 범위 | 왜 그렇게 보나 |
| --- | --- | --- | --- |
| LangChain (Trivedy) | "Agent = Model + Harness". 하네스는 "every piece of code, configuration, and execution logic that isn't the model itself" | 가장 넓게 | 에이전트를 직접 만드는 입장. 모델 말고 자기가 만드는 건 전부 하네스다 |
| Anthropic (Managed Agents 글) | "the loop that calls Claude and routes Claude's tool calls to the relevant infrastructure" | 가장 좁게 | 인프라를 운영하는 입장. 루프(뇌)와 실행 환경(손)을 떼어 내서 따로 다룬다 |
| Böckeler (martinfowler.com) | 넓은 정의를 인용하면서, "part of the harness is already built in"이라고 하고 사용자가 만드는 outer harness / user harness를 중심으로 다룬다 | 사용자 몫 | 도구를 쓰는 개발자 입장. 내가 손댈 수 있는 층에 관심이 있다 |

```text
에이전트
├─ 모델
└─ 하네스  ◀─────────────────── LangChain: 여기 전부
   ├─ 도구에 이미 들어 있는 것
   │   ├─ 루프, 도구 호출 전달하기  ◀── Anthropic(Managed Agents): 좁은 의미의 하네스
   │   └─ 도구 실행, 실행 환경         ("손(hands)"으로 따로 부른다)
   └─ 사용자가 그 위에 얹는 것  ◀──── Böckeler: user harness (outer harness)
```

셋 다 맞다. 같은 그림의 어디를 하네스라고 부르는지가 다를 뿐이다. 그래서 정의 하나를 외우기보다, 경계를 먼저 말하고 목적을 나중에 말한다.

> 하네스는 넓게 보면 에이전트에서 모델을 뺀 나머지 전부입니다. 그중 루프나 도구 실행은 Claude Code 같은 도구가 이미 갖고 있고, 제가 설계한 건 그 위에 팀이 얹는 사용자 하네스입니다. 사용자 하네스의 목적은 모델이 우리 팀이 원하는 방식으로 일하게 만드는 것입니다.

### 2. 내장 하네스는 자리를 만들고, 사용자 하네스는 그 자리를 채운다

| 부품 | Claude Code가 이미 만든 것 (자리) | 사용자가 채우는 것 (내용) |
| --- | --- | --- |
| 루프 | 상황 파악 → 행동 → 확인을 반복하는 반복문 | 없음. 그대로 쓴다 |
| 지시 | CLAUDE.md를 찾아 읽고 대화 앞쪽에 넣어 주는 기능 | CLAUDE.md에 무엇을 쓸지 |
| 스킬 | 설명만 먼저 보여 주고 필요할 때 본문을 불러오는 기능 | `SKILL.md` 내용 |
| 도구 | 기본 도구(Read, Edit, Bash 등), MCP를 붙이는 기능 | 어떤 MCP 서버를 붙일지 |
| 권한 | 실행 전에 규칙을 확인하는 기능(deny → ask → allow 순서) | `settings.json`의 허용·금지 규칙 |
| 훅 | 정해진 시점(문서 기준 33개)에 코드를 끼울 자리 | 그 시점에 돌릴 스크립트 |
| 하위 에이전트 | 새 컨텍스트에서 따로 일하고 요약만 돌려주는 기능 | `.claude/agents/`에 정의하는 역할 |
| 확인 수단 | 명령을 실행하고 결과를 돌려주는 기능 | 테스트, 린터, 타입 검사, 구조 검사 같은 실제 검사 코드 |

테스트나 린터는 Claude Code 설정이 아니라 코드베이스 안에 있다. 그래도 AI가 그 결과를 보고 스스로 고치니까, Böckeler는 이것도 사용자 하네스의 센서로 본다.

Spring에 대어 보면 구조가 같다.

| Spring | Claude Code |
| --- | --- |
| DispatcherServlet이 요청 흐름을 쥐고 있다 | 루프가 작업 흐름을 쥐고 있다 |
| 필터·인터셉터를 끼울 자리가 정해져 있다 | 훅을 끼울 시점이 정해져 있다 |
| 우리는 컨트롤러와 필터를 짠다 | 우리는 지시 파일, 스킬, 훅 스크립트를 쓴다 |
| `application.yml`로 설정한다 | `settings.json`으로 권한을 정한다 |

Codex도 같은 구조다. 루프, 기본 도구(셸, `apply_patch`), AGENTS.md를 루트부터 작업 폴더까지 합쳐 읽는 기능, 샌드박스 모드, 훅 같은 자리를 갖고 있고, 사용자는 AGENTS.md, 스킬, `config.toml`의 승인 정책, `hooks.json`을 채운다.

### 3. 가장 작은 하네스의 다섯 부품

mini-swe-agent는 SWE-agent 팀이 만든 최소 하네스다. 핵심 파일이 `agents/default.py`(190줄)와 `environments/local.py`(92줄)뿐이다.

| 부품 | mini-swe-agent에서는 | 파일 |
| --- | --- | --- |
| ① 시작 메시지 | 시스템 메시지에 응답 형식 규칙, 첫 사용자 메시지에 과제·권장 작업 순서·끝내는 법 | `config/default.yaml` |
| ② 반복문 | 모델에게 묻고 → 행동을 실행하고 → 결과를 붙이고를 반복 | `agents/default.py`의 `run()` |
| ③ 행동 꺼내기 | 답에서 `bash` 도구 호출을 꺼낸다. 없거나 형식이 틀리면 오류 메시지를 모델에게 돌려주고 다시 돈다 | `models/utils/actions_toolcall.py` |
| ④ 실행 환경 | 꺼낸 명령을 bash로 실행한다(제한 시간 30초, 명령마다 새 셸) | `environments/local.py` |
| ⑤ 끝내기와 기록 | 끝내는 조건을 확인하고, 매 단계 대화 전체를 파일로 저장한다 | `agents/default.py`의 `run()`, `save()` |

```python
messages = [system 메시지, 과제 메시지]
while True:
    message = model.query(messages)              # ② 모델에게 묻기
    actions = message의 bash 호출들               # ③ 없으면 오류를 돌려주고 다시
    outputs = [env.execute(a) for a in actions]  # ④ 실행
    messages += 관찰 결과(returncode + 출력)
    save(messages)                               # ⑤ 기록
    if 마지막 메시지가 "exit": break              # ⑤ 끝내기
```

끝내는 방식은 Claude Code와 정반대다.

| | Claude Code | mini-swe-agent |
| --- | --- | --- |
| 도구 호출이 없으면 | 끝 (`end_turn`) | 오류. "도구 호출이 없다"고 돌려주고 다시 돈다 |
| 정상적으로 끝내는 법 | 모델이 도구를 그만 부르면 | 모델이 `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT`을 실행하고, 하네스가 출력 첫 줄에서 그 글자를 확인하면 |
| 강제로 끝내는 법 | 사용자가 Esc를 누르거나 오류가 계속되면 | 단계 수·비용·시간 한도, 또는 형식 오류 3번 연속 |

mini-swe-agent는 말없이 멈추는 것을 허용하지 않고, 모델이 명시적으로 제출 신호를 보내야만 끝나게 했다. **끝내는 조건도 하네스가 정하는 설계 선택이다.**

### 4. 다 갖춘 하네스는 루프 바깥이 커진다

mini-swe-agent의 핵심은 300줄이 안 되는데, Hermes Agent는 테스트·웹·문서를 빼고도 Python 파일이 약 3,400개다. 루프는 거의 그대로고, 차이는 전부 루프 바깥이다.

| 질문 | 더 붙는 부품 | Claude Code (쓰는 쪽) | Hermes 소스 (만드는 쪽) | 계획 |
| --- | --- | --- | --- | --- |
| 무엇을 보여 줄까 | 시스템 프롬프트 조립 | CLAUDE.md를 찾아 넣어 주기 | `agent/prompt_builder.py`: 정체성, 스킬 목록, 지시 파일(SOUL.md, AGENTS.md, .cursorrules)을 모아 조립 | 3장 |
| | 기억 | auto memory | `agent/memory_manager.py` | 3장 |
| | 압축 | 자동 압축 | `agent/context_compressor.py`: 싼 보조 모델이 중간 대화를 요약하고 앞뒤는 지킨다 | 4장 |
| 무엇을 할 수 있게 할까 | 도구와 실행 환경 | 기본 도구, MCP, 샌드박스 | `tools/`, `tools/environments/`(local, docker, modal, daytona 등) | 5장 |
| 어떻게 바로잡을까 | 승인과 훅 | 권한 규칙, 훅 | 승인 관련 파일, 플러그인 훅 | 6~8장 |
| 오래, 크게 일하려면 | 하위 에이전트 | 하위 에이전트 | `tools/delegate_tool.py` | 10장 |
| | 세션 저장, 예약, 메신저 | 세션 기록(JSONL) | SQLite 세션 저장, `cron/`, `gateway/` | 9·11장 |
| | 스스로 고치기 | 기본 기능은 아님 | 턴이 끝난 뒤 스킬을 만들고 고치는 리뷰 | 12장 |

하네스를 잘 만든다는 건 루프를 화려하게 만드는 게 아니라 루프 바깥을 채우는 일이다. 모델에게 무엇을 보여 주고, 어디서 무엇을 하게 하고, 무엇을 막고, 시간이 지나도 어떻게 이어 가게 할지를 정한다.

### 5. 하네스만 바꿔도 결과가 달라진다. 그런데 많이 붙인다고 좋은 건 아니다

**같은 모델, 다른 하네스, 다른 결과**

| 사례 | 내용 | 출처 |
| --- | --- | --- |
| LangChain 코딩 에이전트 | Terminal Bench 2.0에서 하네스만 바꿔 Top 30 → Top 5 | LangChain, "The Anatomy of an Agent Harness" |
| Opus 4.6 | 같은 모델이 Claude Code에서는 33위, 다른 하네스에서는 5위라고 주장 | HumanLayer, "Skill Issue" |
| 도구 설명 다듬기 | 도구 설명을 다듬어 Sonnet 3.5가 SWE-bench Verified 최고 기록 | Anthropic, "Writing tools for agents" |

그래서 AI가 일을 못 하면 모델 탓을 하기 전에 하네스 설정부터 의심한다("skill issue").

**많이 붙인다고 좋지 않다.** 도구가 bash 하나뿐인 mini-swe-agent도 README 기준 SWE-bench Verified 74%를 넘는다. Anthropic은 가장 단순한 것부터 시작하고, 결과가 실제로 나아질 때만 복잡도를 더하라고 한다.

**하네스의 부품에는 유통기한이 있다.**

> "every component in a harness encodes an assumption about what the model can't do on its own" — Anthropic

Anthropic은 예전 모델이 긴 작업에서 흔들려서 "컨텍스트 리셋"을 넣었지만, 다음 모델부터는 짐(dead weight)이 돼서 걷어냈다. 모델이 좋아지면 사용자 하네스가 오히려 방해가 된다는 이야기도 같은 맥락이다.

| 방해가 되기 쉬운 것 | 여전히 도움이 되는 것 |
| --- | --- |
| 길고 중복된 지시 문서 | 짧고 우리 팀만 아는 정보(용어, 금지 사항, 구조의 지도) |
| 예전 모델의 약점을 메우려던 요령 | 테스트, 린터, 구조 검사 같은 센서 |
| AI가 대충 써 준 규칙 문서 | 실제 실수를 보고 사람이 한 줄씩 다듬은 규칙 |

근거는 이렇다. Claude Code best practices에는 이제 그만해도 되는 예전 요령 표("What you can stop doing")가 있고, CLAUDE.md는 짧게 두라고 한다. HumanLayer가 인용한 ETH 연구에서는 LLM이 써 준 지시 파일이 성능을 떨어뜨리고 비용을 20% 넘게 늘렸다. OpenAI는 AGENTS.md를 약 100줄짜리 목차(지도)로 두라고 한다. 흐름은 **글(가이드)은 줄이고 코드(센서)는 남기는** 쪽이다.

코드는 두 가지 일을 한다. 하나는 **막기**(훅, deny 규칙)이고, 다른 하나는 **알려 주기**(린터·테스트가 "여기가 틀렸고 이렇게 고쳐"라고 돌려주는 것)다. 실제로는 알려 주기가 더 많다. Böckeler는 모델이 읽도록 쓴 오류 메시지를 "a positive kind of prompt injection"이라고 부른다.

코드가 모델이 놓친 걸 "무조건" 잡으려면 조건이 두 개 있다.

1. 기계가 판정할 수 있는 규칙이어야 한다.
2. 모델의 선택과 상관없이 실행되는 자리(훅, CI)에 걸려 있어야 한다.

### 6. 하네스 엔지니어링과 컨텍스트 엔지니어링

**책상 비유.** 모델은 아주 똑똑하지만 기억이 없는 사람이고, 매번 책상 위에 놓인 자료만 보고 일한다. 책상 크기(컨텍스트 창)는 정해져 있다. 일하다 보면 파일 읽은 결과, 테스트 출력이 쌓여서 처음에 둔 "내 의도" 메모가 묻힌다(context rot). 컨텍스트 엔지니어링은 AI에게 더 집중하라고 하는 게 아니라, 책상을 대신 정리해 주는 일이다.

| 책상 정리 | 실제로는 |
| --- | --- |
| 지금 필요한 자료만 올려 둔다 | 필요한 파일만 읽게 한다 |
| 두꺼운 매뉴얼은 목차만 두고 필요할 때 그 장만 꺼내 준다 | 스킬은 설명만 먼저 넣고 본문은 필요할 때 불러온다 |
| 다 본 자료는 치우거나 요약본으로 바꾼다 | 대화가 길어지면 압축한다 |
| 너무 긴 출력물은 앞뒤만 잘라 올린다 | mini-swe-agent는 1만 자가 넘는 출력을 줄여 넣는다 |
| 중요한 메모는 늘 맨 위에 둔다 | 압축 뒤에도 CLAUDE.md는 다시 넣어 준다 |

> 컨텍스트 엔지니어링은 AI가 지금 일에 필요한 것(내 의도, 관련 코드, 규칙, 직전 결과)은 잘 보이게 하고, 필요 없는 것은 빼서 창을 깔끔하게 유지하는 일이다.

Anthropic은 목표를 "smallest set of high-signal tokens"라고 한다. 작게(빼기)와 신호가 강하게(필요한 건 남기기)를 같이 잡아야 한다. 너무 많이 넣으면 중요한 게 묻히고, 너무 많이 자르면 필요한 정보가 사라진다. 그래서 자르기보다 **고르기**에 가깝다. 방법은 세 가지다.

1. **빼기:** 다 본 출력을 줄이고, 오래된 대화를 압축한다.
2. **늦게 넣기:** 필요할 때 불러온다(스킬, 파일 검색).
3. **짧게 돌려받기:** 도구 결과를 애초에 핵심만 나오게 만든다(성공은 조용히, 실패는 자세히).

**프롬프트 엔지니어링과의 차이**

| | 프롬프트 엔지니어링 | 컨텍스트 엔지니어링 |
| --- | --- | --- |
| 무엇을 | 지시문 한 덩어리를 잘 쓰기 | 모델이 보는 창 전체를 관리하기 |
| 언제 | 보낼 때 한 번 | 루프가 도는 내내, 요청할 때마다 |
| 대상 | 사람이 쓴 글 | 사람이 쓴 글 + 도구 결과 + 대화 기록 + 기억 + 불러온 파일 |

에이전트는 일 하나에 모델에게 수십 번 다시 묻고, 그때마다 대화 전체를 보낸다. 일이 길어질수록 사람이 쓴 프롬프트는 창의 작은 일부가 되고 나머지는 도구 결과가 채운다. 그래서 처음 프롬프트를 완벽하게 쓰는 것보다 쌓이는 걸 관리하는 게 더 중요해졌다.

**하네스 엔지니어링과의 관계**

```text
하네스 엔지니어링
├─ 무엇을 보여 줄까 ◀── 컨텍스트 엔지니어링
├─ 무엇을 할 수 있게 할까  (도구, 실행 환경, 샌드박스)
├─ 무엇을 막을까          (권한 규칙, 훅)
└─ 어떻게 확인할까        (테스트, 린터, AI 리뷰)
```

Böckeler와 HumanLayer는 반대로 "사용자 하네스를 만드는 건 컨텍스트 엔지니어링의 한 형태"라고 본다. 아래 세 칸의 결과도 결국 글(막은 이유, 오류 메시지)로 모델에게 돌아가기 때문이다. 그래서 둘 다 맞게 이해하면 이렇다. 모델 입장에서는 전부 컨텍스트로 보이지만, 하네스는 그 글을 만들어 내는 바깥 세상(실행 환경, 강제, 검사)까지 설계한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 넓은 의미의 하네스는 "모델이 원하는 방식으로 운영되도록 하는 모든 것"이다 | 넓은 정의는 목적이 아니라 경계로 말한다: 모델을 뺀 나머지 전부. "원하는 방식으로"는 사용자 하네스의 목적을 말할 때 맞는 표현이다. |
| 내가 만든 건 "사용자전용 하네스" | 원문은 user harness, 즉 사용자 하네스다. "전용"이 붙으면 다른 사람은 못 쓰는 것처럼 들린다. |
| Hermes는 Claude Code 소스가 유출된 뒤 거기서 나온 비서형 하네스다 | Hermes는 유출(2026-03-31)보다 먼저(2026년 2월) 나왔다. 한 제품이 바뀐 게 아니라, 같은 하네스 부품을 코딩 말고 개인 비서에 쓰는 흐름이 생긴 것이다. 유출에서는 Claude Code의 미출시 상주 모드(KAIROS)가 발견됐다. |
| 하네스를 공부하려면 Hermes 하나를 기준으로 삼으면 된다 | Hermes는 비서형이라 코딩 하네스의 핵심(센서, 완료 게이트)이 약하고, 너무 크고, 한 프로젝트의 선택일 뿐이다. 그래서 역할별로 나눴다: 최소 루프는 mini-swe-agent, 코딩 하네스는 Codex, 다 갖춘 하네스는 Hermes. |
| 모델이 좋아져서 사용자가 설정한 하네스가 오히려 방해가 된다 | 맞는 말이지만 방해가 되는 종류가 따로 있다. 긴 문서, 낡은 요령, AI가 써 준 규칙은 방해가 되고, 짧은 고유 정보와 센서, 사람이 다듬은 규칙은 여전히 돕는다. |
| 모델이 놓친 걸 코드로 짜 두면 무조건 잡는다 | 기계가 판정할 수 있는 규칙이어야 하고, 모델의 선택과 상관없이 실행되는 자리에 걸려 있어야 한다. |
| 컨텍스트 엔지니어링은 한 번에 좋은 결과를 뽑도록 프롬프트를 잘 쓰는 것이다 | 그건 프롬프트 엔지니어링이다. 컨텍스트 엔지니어링은 루프가 도는 내내 쌓이는 창 전체를 관리하는 일이고, 프롬프트 잘 쓰기는 그 안에 들어간다. |
| 컨텍스트 엔지니어링은 AI가 내 의도를 잘 보게 만드는 것이다 | 의도만이 아니라 지금 일에 필요한 정보 전부를 잘 보이게 하고, 필요 없는 것은 빼는 일이다. |
| 방해되는 요소를 잘 자르는 게 기술이다 | 너무 자르면 필요한 정보가 사라진다. 자르기보다 고르기이고, 빼기·늦게 넣기·짧게 돌려받기 세 가지다. |

### 공부하며 던진 질문

이 장은 개념 정리 장이라 문제 풀이 없이 진행했다. 확인 문제 대신 실제로 물어본 것을 남긴다.

- **헤르메스가 뭐야?** Nous Research의 오픈소스(MIT) 에이전트다. 공식 사이트가 스스로를 "A harness with persistent memory, a skills system and real tool use"라고 부른다. 서버에 상주하면서 메신저와 cron으로 일하고, 턴이 끝난 뒤 스스로 스킬을 만들고 고친다.
- **Hermes를 만든다고 생각하면, 내가 만든 건 극히 일부 아닌가?** 맞다. 회사에서 만든 건 사용자 하네스이고, Hermes는 루프부터 다 갖춘 하네스다. 그래서 각 장을 쓰는 쪽(Claude Code)과 만드는 쪽(오픈소스 하네스 소스) 양쪽에서 보기로 했다.
- **세션을 다시 열면 전역 CLAUDE.md를 읽나?** 이번에는 세션을 다시 열 때 지시 파일을 다시 읽어서 새 내용이 들어왔다. 세션 도중에 고친 내용이 바로 반영되는지는 확인하지 않았다(3장에서 다룬다).

## 예제와 확인 결과

**직접 읽은 소스**
- mini-swe-agent v2.4.6(커밋 `04d809c`): `src/minisweagent/agents/default.py`(190줄), `environments/local.py`(92줄), `models/utils/actions_toolcall.py`, `config/default.yaml`
  - 끝내는 조건은 `local.py`의 `_check_finished()`에 있다. 출력 첫 줄이 `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT`이고 returncode가 0이면 `Submitted`로 끝낸다.
  - 도구 호출이 없으면 `actions_toolcall.py`가 `FormatError`를 내고, "No tool calls found in the response. Every response MUST include at least one tool call."을 모델에게 돌려준다.
- Hermes Agent(커밋 `f85f33bf`, 2026-11-25): 파일 목록과 각 파일 첫머리 설명만 확인했다. 저장소 전체는 27,672개 파일이다. `tests`, `web`, `website`, `evals`, `apps`, `ui-tui`, `locales`를 빼면 Python 파일이 3,432개다. 각 파일의 내부 동작은 아직 읽지 않았다.

**설명용:** Spring 비교표, 책상 비유, 하네스 엔지니어링 그림은 설명용이다.

## 남은 질문

- Hermes의 `run_agent.py` 루프는 mini-swe-agent의 루프와 어디가 다를까? 재시도, 다른 모델로 넘기기, 압축이 루프의 어디에 붙어 있을까?
- 끝내는 조건을 "명시적 제출"로 만들면 생기는 문제는 없을까? 모델이 제출 신호를 너무 일찍 보내면?
- Codex는 AGENTS.md를 실제 코드에서 어떻게 합칠까?
- 순위 사례들의 실제 점수와 조건은 무엇일까?
- 사용자 하네스가 도움이 되는지 방해가 되는지는 어떻게 잴까? (11장)

## 참고 자료

- [Böckeler — Harness engineering for coding agent users (martinfowler.com)](https://martinfowler.com/articles/harness-engineering.html)
- [LangChain — The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [HumanLayer — Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- [Addy Osmani — Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/)
- [OpenAI — Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic — Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Anthropic — Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Anthropic — Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- [Claude Code — Best practices](https://code.claude.com/docs/en/best-practices)
- [Codex — AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- [SWE-agent/mini-swe-agent (GitHub)](https://github.com/SWE-agent/mini-swe-agent)
- [Hermes Agent 문서](https://hermes-agent.nousresearch.com/docs)
- [awesome-harness-engineering (GitHub)](https://github.com/walkinglabs/awesome-harness-engineering)

## 정리 이력

- 2026-11-25: 1차 자료 조사와 Claude와의 학습 대화를 바탕으로 정리했다. 문제 풀이 없이 진행한 장이다.
