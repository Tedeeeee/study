# 루프의 어디에 코드를 끼울 수 있을까?

[목차](README.md) · [이전: 무엇을 글로 안내하고, 무엇을 코드로 막을까?](guides-and-sensors.md)

## 다루는 범위

에이전트 루프의 정해진 순간에 우리 코드를 실행하는 장치, 훅을 정리한다. 훅을 이해하려면 먼저 도구·시점·훅·훅 스크립트를 구분해야 해서 거기서부터 시작한다.

- 도구, 시점, 훅, 훅 스크립트는 각각 다르다
- 훅을 건다는 것과 무조건 도는 자리
- 훅으로 할 수 있는 네 가지와 대표 시점, 루프 위의 위치
- 훅 설정을 읽는 법과 matcher
- 훅 스크립트의 입력과 출력, 표준 입출력과 종료 코드
- 훅을 적는 곳과 Codex 비교
- 작은 훅 직접 돌려 보기
- 에이전트와 워크플로

훅으로 끝낼 때 검사를 강제하는 완료 게이트는 [8번 주제](README.md#공부-계획)에서 다룬다.

## 시작한 질문

6장에서 센서를 만들었다. 이제 그 센서를 AI가 뭘 하든 반드시 도는 자리에 걸고 싶다.

1. 훅을 "건다"는 건 정확히 무슨 뜻일까?
2. 루프의 어느 순간에, 무엇을 걸 수 있을까?

## 내가 이해한 내용

### 1. 도구, 시점, 훅, 훅 스크립트

| | 정체 | 예 |
| --- | --- | --- |
| 도구(tool) | AI가 요청서로 불러서 쓰는 손 | Bash, Read, Edit, Write, Grep |
| 시점(이벤트) | 훅을 걸 수 있는 순간. Claude Code가 정해 둔 것 | PreToolUse, PostToolUse, Stop, SessionStart, UserPromptSubmit |
| 훅 | 시점 + 조건(matcher) + 실행할 것을 묶어 우리가 등록한 것 | "PreToolUse에, Bash일 때, block-rm.mjs 실행" |
| 훅 스크립트 | 훅이 실행하는 실제 파일 | `block-rm.mjs` |

도구와 시점을 구분하는 질문은 "AI가 요청서로 이걸 부를 수 있나?"다. 부를 수 있으면 도구, 부를 수 없고 그냥 그 순간이 오면 시점이다. PreToolUse는 이름에 Tool이 들어가지만 "도구(Tool)를 사용(Use)하기 전(Pre)"이라는 순간의 이름이다. Stop도 AI가 부르는 게 아니라 AI가 더 이상 도구를 요청하지 않고 끝내려는(`end_turn`) 순간이다. JPA의 `@PrePersist`가 저장 자체가 아니라 저장 직전이라는 순간인 것과 같다.

| | 도구 | 훅 |
| --- | --- | --- |
| 정체 | AI가 세상에 일을 하는 손 | 하네스가 정해진 순간에 우리 코드를 실행하는 알람 |
| AI에게 보이나 | 보인다(목록) | 안 보인다(막을 때 이유만 전달된다) |
| 언제 실행되나 | AI가 요청할 때만 | 정해진 시점이 오면 무조건 |
| 누가 정하나 | 쓸지 말지는 AI가 고른다 | 언제 돌지는 사람이 미리 정한다 |

도구는 사람이 준 것 중에서 AI가 골라 쓰는 손이고, 훅은 사람이 걸어 둔, AI가 피할 수 없는 알람이다. 둘은 함께 움직인다. 훅이 도구를 앞뒤로 감싼다.

```text
AI가 도구를 요청 → [PreToolUse 훅] → 도구 실행 → [PostToolUse 훅]
  (AI가 고름)       (무조건 실행)      (AI의 손)    (무조건 실행)
```

같은 검사 스크립트도 도구로 두면 AI가 골라야 돌고, 훅으로 걸면 AI가 깜빡해도 돈다. 안에 넣는 내용물은 자유롭다. 도구든 훅이든 스크립트, 외부 서버 호출, AI 판단을 담을 수 있다. 차이는 언제를 누가 정하느냐와 AI가 아느냐뿐이다.

사람도 도구를 만들 수 있다. Claude Code와 Codex에서는 MCP 서버를 만들어 붙이면 그 서버가 정의한 도구들이 AI의 도구 목록에 들어간다. MCP 서버는 도구를 묶어 제공하는 서버이고, 그 안의 기능 하나하나가 도구다(백엔드로 치면 API 서버와 엔드포인트). 이름은 `mcp__서버이름__도구이름` 모양이다. 도구를 만들어도 쓸지 말지는 여전히 AI가 정하고, 도구 설명이나 CLAUDE.md로 고를 가능성을 높일 수는 있어도 보장은 안 된다. 요즘 도구는 대부분 MCP로 붙인다. 표준이라 여러 하네스에서 쓸 수 있고, 외부 서비스를 붙이기 쉽고, 하네스와 분리돼 있기 때문이다. 대신 도구가 너무 많아지기 쉽고, 결과가 외부에서 온 글이라 프롬프트 주입 통로가 될 수 있고, Codex 문서 기준으로 샌드박스가 MCP 도구에는 걸리지 않는다.

도구는 AI에게 "이런 것도 할 수 있어"를 주는 것이고, AI가 하는 일을 막고 제한하고 확인하는 건 훅·권한·샌드박스다. 그중 훅이 원하는 판단을 가장 자유롭게 짤 수 있는 장치다. 도구도 실수하기 어렵게 만들어 둘 수는 있지만(Edit은 먼저 Read), 행동 자체를 막는 건 훅과 권한의 일이다.

### 2. 훅을 건다는 것과 무조건 도는 자리

훅(hook)은 갈고리라는 뜻이고, 훅을 건다는 건 정해진 순간에 내 코드가 실행되도록 미리 등록해 두는 것이다. 알람을 거는 것과 같다. JPA에서 `@PrePersist`를 붙인 메서드를 `save()` 때 JPA가 알아서 먼저 실행하듯, 훅도 그 순간이 오면 Claude Code가 알아서 실행한다.

| AI가 골라야 도는 자리 | 무조건 도는 자리 |
| --- | --- |
| 냉장고에 붙인 메모. 내가 보고 따라야 한다 | 알람. 내가 뭘 하든 그 시각이 되면 울린다 |
| CLAUDE.md에 "끝내기 전에 테스트를 돌려라" | 훅, CI |

Claude Code에는 훅을 걸 수 있는 시점이 문서 기준 33개 있다. JPA에 콜백을 걸 수 있는 순간이 7개(`@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostUpdate`, `@PreRemove`, `@PostRemove`, `@PostLoad`) 정해져 있는 것과 같다. 실제로 많이 쓰는 건 PreToolUse, PostToolUse, Stop 정도다. 이름을 다 외울 필요는 없고, 개념(직전·직후·끝내려 할 때)을 알고 설정할 때 문서를 보면 된다.

### 3. 훅으로 할 수 있는 네 가지와 대표 시점

| 할 수 있는 일 | 뜻 | 예 |
| --- | --- | --- |
| 막기 | 그 행동을 멈추고 이유를 AI에게 돌려준다 | `rm -rf` 막기, 검사 실패 시 못 끝내게 하기 |
| 고치기 | 도구에 들어갈 입력을 바꿔서 실행시킨다 | 명령에서 위험한 옵션을 빼고 실행 |
| 덧붙이기 | 도구 결과 옆에 안내를 붙인다 | 파일을 고친 뒤 "이 폴더는 외부 라이브러리 금지" |
| 기록하기 | 막지 않고 로그만 남긴다 | AI가 실행한 명령을 전부 파일에 남기기 |

| 시점 | 언제 | 막기 | 고치기 | 덧붙이기 | 기록 |
| --- | --- | --- | --- | --- | --- |
| SessionStart | 세션 시작 | ✗ | | | ✓ |
| UserPromptSubmit | 내가 입력하고 모델에게 가기 전 | ✓ | | | ✓ |
| PreToolUse | 도구 실행 직전 | ✓ | ✓(`updatedInput`) | | ✓ |
| PostToolUse | 도구 실행 직후 | ✗(이미 실행됨) | | ✓(`additionalContext`) | ✓ |
| Stop | AI가 끝내려 할 때 | ✓(계속 일하게 함) | | | ✓ |
| PreCompact | 압축 직전 | ✓ | | | ✓ |

빈칸은 그 기능을 문서에서 확인하지 않았다는 뜻이다. PostToolUse는 이미 실행된 뒤라 못 막지만, 결과 옆에 "문제 있으니 고쳐"를 덧붙여 AI가 고치게 할 수는 있다.

도구 주변 시점이 가장 중요한 이유는, 모델은 글만 쓰고 파일을 고치거나 명령을 실행하는 건 전부 도구라서 AI가 세상에 하는 일이 하나도 빠짐없이 도구를 거치기 때문이다. 루프는 두 겹이다.

```text
SessionStart ───────────────────────────── 세션에 한 번
┌→ UserPromptSubmit ────────────────────── 바깥 루프: 내가 말할 때마다
│   ┌→ 모델이 생각하고 요청서를 씀           ← 안쪽 루프(1장의 반복문)
│   │    PreToolUse → 도구 실행 → PostToolUse
│   └── 결과를 붙여 다시 모델에게(요청서가 있는 동안 반복)
│   모델이 요청서 없이 "다 했다"(end_turn)
│   Stop ──── 통과 → 바깥으로 / 막음 → 다시 안쪽 루프로
└── 다음 입력을 기다림
(창이 차면 어디서든 PreCompact → 압축)
```

Stop은 안쪽 루프의 출구에 서 있어서, 막으면 AI를 루프 안으로 돌려보낼 수 있다.

### 4. 훅 설정 읽기와 matcher

훅을 쓰려면 세 가지를 정한다.

| 정할 것 | 예 |
| --- | --- |
| 언제(시점) | 파일을 고친 직후(PostToolUse) |
| 어떤 경우(matcher) | Edit·Write일 때만 |
| 무엇을, 어떻게(실행할 것) | `lint.mjs`를 돌려서 어기면 이유를 돌려준다 |

설정은 이렇게 적는다(설명용).

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "node .claude/hooks/lint.mjs" }]
      }
    ]
  }
}
```

"도구 실행 직후(PostToolUse)에, 그 도구가 Edit이나 Write라면(matcher), `node .claude/hooks/lint.mjs`를 실행해라"로 읽는다. 시점 이름은 맨 바깥 칸에, 도구 이름은 matcher 칸에 들어간다.

**matcher**는 어떤 도구일 때 이 훅을 돌릴지 고르는 필터다. PostToolUse는 모든 도구 뒤에 오기 때문에, matcher 없이는 파일을 읽기만 해도 린트가 돈다. `|`는 "또는"이다. matcher는 도구 이름만 보고, 규칙을 어겼는지는 훅 스크립트가 안에서 판단한다. 공항에서 "국제선 탑승객만 이 검색대로" 안내판이 matcher이고, 검색대에서 위험물을 검사하는 게 스크립트다. 스프링 인터셉터의 `addPathPatterns(...)`와 같은 역할이다. 설정을 보면 도구 안에 훅이 있는 것처럼 보이지만, 훅은 도구 바깥에서 지켜보다가 그 도구가 불릴 때 끼어든다.

`Edit`, `Write` 같은 이름은 Claude Code가 정해 둔 도구 이름이고, 그중 무엇을 matcher에 적을지는 우리가 고른다. 없는 이름을 적으면 그 훅은 한 번도 돌지 않는다. 막고 싶으면 시점과 짝을 맞춘다. 실행 전에 막으려면 PreToolUse, 실행 뒤에 확인하려면 PostToolUse다. 경로 하나 막기처럼 간단한 제한은 권한 deny 규칙으로도 되고, 훅은 판단이 들어가는 제한에 쓴다. MCP 도구도 같은 방식으로 Pre·Post 훅을 걸 수 있다고 보지만, MCP 도구 이름으로 matcher를 거는 예시는 문서에서 직접 확인하지 않았다.

**만들기와 이용하기는 따로다.** 스크립트 파일을 만들고(`.claude/hooks/lint.mjs`), `command`에는 그 파일을 실행하는 명령만 적는다. 스프링에서 인터셉터 클래스를 짜고(`AuthInterceptor`), 설정에 등록하는(`registry.addInterceptor(...)`) 것과 같다. 하나 만들어 두면 PostToolUse 훅, Stop 훅, CI에 모두 걸 수 있다. 같은 스크립트도 직접 치면 그냥 스크립트, `package.json`에 넣으면 npm 스크립트, CI에 넣으면 CI 검사 단계, 훅에 등록하면 훅 스크립트다. `@Component`를 붙여 등록해야 빈이 되는 것과 같다.

팀 규칙이면 스크립트 파일을 프로젝트 안(`.claude/hooks/`나 `scripts/`)에 두고, 설정과 파일을 둘 다 저장소에 올린다. 설정만 올리면 팀원 PC에는 실행할 파일이 없다. 언어는 Node.js, Python, bash 무엇이든 되고, AI가 그 파일을 못 고치게 막아 둔다.

### 5. 훅 스크립트의 입력과 출력

훅 스크립트는 하네스가 부르는 함수 하나와 같다.

| 함수로 치면 | 훅에서는 | 통로 |
| --- | --- | --- |
| 매개변수 | 하네스가 넘겨주는 상황 정보 | 표준 입력(stdin)으로 JSON |
| 반환값 | 통과·막기 결론 | 종료 코드 |
| 예외 메시지 | 막은 이유 | 표준 오류(stderr) |

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf ./data" },
  "cwd": "/home/me/project"
}
```

시점 이름과 도구 이름이 서로 다른 칸(`hook_event_name`, `tool_name`)에 들어 있다. 판단하는 건 하네스가 아니라 스크립트다. 하네스는 그 순간이 오면 스크립트를 실행하고 스크립트가 남긴 숫자대로 따른다. 시점에 따라 스크립트가 볼 수 있는 게 다르다. PreToolUse는 결과가 나오기 전이라 요청서(도구 이름과 입력)만 보고, PostToolUse는 실행 결과까지, Stop은 마지막 말을 보거나 스크립트가 직접 검사를 돌려 볼 수 있다.

```java
int onPreToolUse(Situation s) {                       // 설명용
    if (s.toolInput().command().contains("rm -rf")) {
        System.err.println("rm -rf는 막혀 있어요. 지울 파일을 하나씩 지정하세요.");
        return 2;
    }
    return 0;
}
```

**종료 코드.** 원래 종료 코드는 0이면 성공, 0이 아니면 실패이고, 1이 가장 흔한 일반 실패다. 다른 숫자의 뜻은 프로그램마다 정한다. Claude Code 훅은 이렇게 읽는다.

| 종료 코드 | 뜻 | Claude Code가 하는 일 | HTTP로 치면 |
| --- | --- | --- | --- |
| 0 | 정상 | 그대로 진행 | 200 OK |
| 2 | "막아라"(일부러 막는 신호) | 행동을 막고 stderr를 이유로 AI에게 전달 | 403 Forbidden |
| 1 등 그 밖 | 훅 스크립트가 고장났다 | 오류로 표시하고 행동은 그대로 진행 | 500 Internal Server Error |

훅 스크립트에 버그가 있거나 실행 자체가 안 될 때 전부 "막아라"로 읽으면 훅 하나가 고장 나 AI가 아무것도 못 하게 되므로, 일부러 막는 것과 고장을 구분한다. 문서상 `onFailure: "block"`을 붙이면 고장일 때도 막게 바꿀 수 있다.

숫자 대신 종료 코드 0과 stdout의 JSON으로 답하면 더 세밀하게 할 수 있다.

| JSON에 쓰는 것 | 효과 |
| --- | --- |
| `permissionDecision`: allow / deny / ask | 허용, 막기, 사람에게 물어보기(PreToolUse) |
| `updatedInput` | 도구 입력을 바꿔 실행(고치기) |
| `additionalContext` | 결과 옆에 안내를 덧붙이기(최대 1만 자) |
| `decision: "block"` + `reason` | 막고 이유를 남기기(PostToolUse, Stop 등) |

**표준 입출력.** 모든 프로그램은 실행될 때 통로 세 개를 받는다. Java의 `System.in`, `System.out`, `System.err`가 그것이다.

| 이름 | 뜻 | Java | fd |
| --- | --- | --- | --- |
| stdin(표준 입력) | 프로그램이 받는 통로 | `new Scanner(System.in)` | 0 |
| stdout(표준 출력) | 결과를 내보내는 통로 | `System.out.println` | 1 |
| stderr(표준 오류) | 오류·경고를 내보내는 통로 | `System.err.println` | 2 |

평소엔 키보드가 stdin에, 터미널 화면이 stdout·stderr에 연결되지만 다른 프로그램을 연결할 수도 있다. `npm run test:fast | grep "FAIL"`의 파이프는 npm의 stdout을 grep의 stdin에 연결한다. 훅에서는 Claude Code가 이 통로를 붙잡아 stdin으로 상황 JSON을 넣고, stdout의 JSON과 stderr의 이유와 종료 코드를 읽는다. stdout과 stderr를 나눈 덕분에 정상 결과와 오류를 따로 받는다.

이건 AI 전용 용어가 아니라 프로그램끼리의 일반적인 통로다. 훅 스크립트는 Claude Code와 표준 입출력으로 주고받고, Claude Code는 모델과 API 메시지(요청서와 결과)로 주고받는다. 훅 스크립트가 stderr로 쓴 이유를 Claude Code가 받아 API 메시지에 담아 모델에게 전달한다.

```text
훅 스크립트 ◀── stdin·stdout·stderr ──▶ Claude Code ◀── API 메시지 ──▶ 모델
```

### 6. 훅을 적는 곳과 Codex 비교

| 파일 | 적용 범위 | 언제 쓰나 |
| --- | --- | --- |
| `~/.claude/settings.json` | 내 PC의 모든 프로젝트 | 개인 습관 |
| 프로젝트의 `.claude/settings.json` | 이 프로젝트, git으로 팀 공유 | 팀 규칙 |
| `.claude/settings.local.json` | 이 프로젝트, 나만(git에 안 올림) | 실험용 |
| 스킬이나 하위 에이전트 설정 안의 `hooks` | 그 스킬·에이전트가 돌 때만 | 특정 작업 전용 검사 |

| | Claude Code | Codex |
| --- | --- | --- |
| 적는 곳 | `settings.json`(사용자, 프로젝트, 로컬) | `hooks.json`(`~/.codex/`, 프로젝트의 `.codex/`) 또는 `config.toml` |
| 시점 수 | 33개 | 14개 |
| 막는 법 | exit 2 | exit 2 |
| 끝내려 할 때 막기 | Stop 훅 | Stop 훅. 막으면 그 이유를 다음 지시로 삼아 계속 일한다 |
| 입력 | stdin으로 JSON | stdin으로 JSON |

Codex는 훅을 `/hooks`에서 하나하나 검토하고 신뢰해야 돌고, 프로젝트의 `.codex/hooks.json`도 신뢰한 프로젝트에서만 읽는다. 남이 만든 저장소의 훅이 내 PC에서 아무 코드나 실행할 수 있기 때문이다. 훅은 내 PC에서 코드를 실행하는 장치라 그 자체가 보안 위험이 될 수 있다. Codex에는 모델에게 묻기 직전·직후 시점(`PreModelRequest`, `PostModelRequest`)도 있다.

### 7. 작은 훅 직접 돌려 보기

Claude Code에 실제로 등록하는 대신, Claude Code가 훅 스크립트에 하는 일(stdin으로 JSON 넣기, 종료 코드와 stderr 읽기)을 파이프로 흉내 냈다.

```js
// block-rm.mjs
import { readFileSync } from 'node:fs';

const input = JSON.parse(readFileSync(0, 'utf8'));   // stdin(fd 0)으로 상황 받기
const cmd = input.tool_input?.command ?? '';

if (/\brm\s+-rf\b/.test(cmd)) {
  console.error('rm -rf는 막혀 있어요. 지울 파일을 하나씩 지정하거나 사용자에게 먼저 물어보세요.');
  process.exit(2);
}
process.exit(0);
```

결과는 "예제와 확인 결과"에 적었다. Claude Code에 걸려면 이 파일을 `.claude/hooks/`에 두고 `.claude/settings.json`에 PreToolUse + `"Bash"` + `node .claude/hooks/block-rm.mjs`로 등록한다.

### 8. 에이전트와 워크플로

에이전트를 만드는 건 도구의 개수가 아니라 누가 다음 행동을 정하느냐다. Anthropic의 "Building effective agents"는 미리 정해 둔 코드 경로로 LLM과 도구를 부르는 것을 워크플로, 모델이 스스로 과정과 도구 사용을 정하는 것을 에이전트로 나눈다. 도구가 bash 하나뿐인 mini-swe-agent도 에이전트이고, 도구가 많아도 코드가 순서를 다 정하면 워크플로다. 도구가 많아지면 할 수 있는 일의 범위가 넓어질 뿐이다.

| 메일 답장 기능 | 흐름 | 모델이 판단하는 것 | 분류 |
| --- | --- | --- | --- |
| A. 순서 고정 | 도착 → (항상) 답장 작성 → (항상) 발송 | 글의 내용만 | 워크플로 |
| B. 갈림길만 판단 | 도착 → 모델이 분류 → 분류별로 미리 짠 처리 | 어느 길로 갈지만 | 워크플로(routing 패턴) |
| C. 다음 행동을 스스로 | 도착 → "캘린더부터 보자" → 조회 → "자료도 붙이자" → 검색 → 작성 → 발송, 또는 "답장 불필요" | 어떤 도구를, 어떤 순서로, 언제 끝낼지 | 에이전트 |

AI가 중간에 여러 번 쓰여도 어디서 무슨 판단을 할지를 코드가 정해 두면 워크플로다. AI가 칸을 채우면 워크플로, AI가 다음 칸을 정하면 에이전트다. Anthropic은 가장 단순한 것부터 시작하라고 한다. 늘 비슷한 일이면 워크플로가 싸고 예측 가능하고, 순서를 미리 정할 수 없을 때 에이전트를 쓴다.

"리뷰 에이전트", "탐색 에이전트"처럼 맡은 일로 부르는 건 역할 이름이고, 구조로는 에이전트다. 스프링에서 빈이 구조이고 `OrderService` 빈이 역할인 것과 같다. Claude Code의 Explore·Plan 에이전트, Anthropic 리서치 시스템의 LeadResearcher·CitationAgent, 긴 작업 하네스의 planner·generator·evaluator가 그 예다. 일상적으로는 정해진 순서로만 도는 프로그램도 에이전트라고 부르지만, 개념을 물으면 "모델이 스스로 다음 행동을 정하느냐"로 구분한다.

실제로는 둘을 섞는다. 워크플로 안에 에이전트 단계를 두기도 하고, 에이전트 안에 코드로 정한 관문을 두기도 한다. Claude Code는 에이전트이고, 우리가 거는 훅은 그 위에 얹는 "코드로 정해 둔 관문"이다. "정할 수 있는 건 코드로 강제하고, 애매한 판단만 AI에게 맡긴다"는 생각이 바로 이 섞기다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| PreToolUse 같은 시점이 도구(tool)다 | 시점은 훅을 거는 순간의 이름이고, 도구는 AI가 요청서로 부르는 손(Bash, Read, Edit)이다. "AI가 요청서로 부를 수 있나?"로 구분한다. 그동안 훅 이야기가 엉켰던 가장 큰 이유였다. |
| Stop은 도구다 | AI가 끝내려는 순간의 이름(시점)이다. |
| 훅은 센서의 영역이다 | 훅은 "언제"를 정하는 장치(알람)이고, 센서는 "무엇을 확인할지"(검사)다. 훅에는 센서뿐 아니라 가이드(안내 덧붙이기)나 기록도 건다. |
| 센서는 AI 판단과 상관없이 꼭 하게 하고 싶은 것이다 | 센서는 확인하는 검사일 뿐이고, 꼭 돌게 할지는 자리(훅·CI)가 정한다. |
| 센서는 무조건 코드다 | AI가 확인하는 추론형 센서도 있고, `prompt`·`agent` 훅으로 훅에 걸 수도 있다. 다만 무조건 막는 데는 주로 코드 센서를 쓴다. |
| 하네스가 판단한다 | 판단은 훅 스크립트가 하고, 하네스는 그 순간에 실행하고 숫자대로 따른다. |
| 훅 스크립트와 훅은 같다 | 훅은 등록(설정)이고 훅 스크립트는 그 등록이 실행하는 파일이다(인터셉터 등록과 인터셉터 클래스). |
| 코드로 작성한 건 모두 훅 스크립트다 | 훅에 등록됐을 때만 그렇게 부른다. 같은 파일도 자리에 따라 이름이 바뀐다. "기계어"가 아니라 "코드"가 맞는 말이다. |
| 설정을 보면 도구 안에 훅이 있다 | matcher의 도구 이름은 조건일 뿐이고, 훅은 도구 바깥에서 지켜보다 끼어든다. |
| matcher는 "규칙을 어겼을 때" 같은 조건이다 | matcher는 도구 이름으로 훅을 돌릴 때를 좁히는 필터이고, 규칙 위반은 스크립트가 판단한다. |
| 지켰을 때는 축하 메시지를 준다 | 성공은 조용히 exit 0으로 끝낸다. 매번 붙는 글은 컨텍스트만 차지한다. |
| stdin·stdout은 AI와 주고받을 때 쓰는 용어다 | 모든 프로그램의 일반적인 통로다. 훅 스크립트는 Claude Code와 주고받고, 모델에게는 Claude Code가 API 메시지로 전달한다. |
| exit 2는 그냥 에러이고 1은 뭔지 모르겠다 | 2는 "막아라"라는 약속된 신호이고, 1 등은 훅이 고장 났다는 뜻이라 행동이 그대로 진행된다. |
| 도구가 많아지면 에이전트가 된다 | 에이전트는 모델이 스스로 다음 행동을 정하는 루프다. 도구가 하나여도 에이전트일 수 있다. |
| 에이전트는 특정 작업을 하는 것이다 | 그건 역할 이름이고, 구조로는 모델이 다음 행동을 정하는 시스템이다. 둘은 겹쳐 쓴다. |

### 공부하며 던진 질문

- **시점 이름을 다 외워야 하나?** 개념만 알면 되고, 자주 쓰는 PreToolUse·PostToolUse·Stop 정도만 바로 나오면 좋다.
- **사람도 도구를 만들 수 있나?** MCP 서버로 만든다. 그래도 쓸지는 AI가 고른다.
- **인프런 MCP도 도구인가?** MCP 서버는 도구를 묶어 주는 서버이고, 강의 검색 같은 기능 하나하나가 도구다.
- **MCP 도구 실행 전후에도 훅을 걸 수 있나?** 같은 도구라 Pre·Post 시점을 지나간다. matcher에 그 이름을 적으면 된다(문서 예시는 미확인).
- **훅 스크립트 파일은 어디에 두나?** 팀 공유면 프로젝트의 `.claude/hooks/`, 개인이면 `~/.claude/hooks/`. 설정과 파일을 같이 올린다.
- **메일이 오면 스스로 판단해 답장을 보내는 건 에이전트인가?** 판단이 글 내용까지면 워크플로, 다음에 무엇을 할지까지면 에이전트다.

### 확인 문제와 내 답

| 문제 | 내 답의 요지 | 결과 |
| --- | --- | --- |
| 1. `Bash`, `PreToolUse`, `Edit`, `Stop`, `lint.mjs`, "PostToolUse에 Edit일 때 lint 실행"을 도구·시점·훅·훅 스크립트로 나누기 | 도구, 시점, 도구, 도구, 훅 스크립트, 훅 | Stop만 틀렸다(시점). "AI가 요청서로 부를 수 있나?"로 `UserPromptSubmit`(시점)과 `Grep`(도구)을 다시 판단해 통과 |
| 2. 파일을 고칠 때마다 린트를 돌려 바로 고치게 하는 훅 설계(언제, matcher, 스크립트의 답, 어디에) | 코드를 고친 시점 / 타입·레이아웃 규칙을 어겼을 때 / 어기면 어떻게 어겼고 어떻게 고치나, 지키면 축하 / settings.json | 부분 통과. 시점은 맞음(PostToolUse). matcher는 규칙이 아니라 도구 이름(`Edit\|Write`), 성공은 조용히, 팀 공유는 프로젝트의 `.claude/settings.json` |
| 3. "PostToolUse + `exit 1`"로 `prod.yml` 수정을 막으려 했는데 안 막힌다 | 시점이 잘못됐다(이미 수정된 뒤) / exit 1이 아니라 2 | 통과. PreToolUse로 바꾸고 stderr + exit 2. 보충: deny 규칙으로도 가능, Bash `sed` 우회, 스크립트 보호 |

## 예제와 확인 결과

Windows 11의 Git Bash와 Node.js로 `block-rm.mjs`에 상황 JSON을 파이프로 넣어 실행했다.

```text
경우 1: {"tool_name":"Bash","tool_input":{"command":"rm -rf ./data"}}
  출력: rm -rf는 막혀 있어요. 지울 파일을 하나씩 지정하거나 사용자에게 먼저 물어보세요.
  종료 코드: 2

경우 2: {"tool_name":"Bash","tool_input":{"command":"npm test"}}
  출력 없음
  종료 코드: 0

stderr를 버리면(2>/dev/null): 문장이 사라지고 종료 코드 2
stderr만 파일로 받으면(2>reason.txt): 파일에 그 문장이 들어 있음
```

실제 Claude Code에 등록해서 AI가 막히는 것까지는 확인하지 않았다. 그 밖의 설정 예시와 코드는 설명용이다.

## 남은 질문

- MCP 도구 이름이나 `mcp__inflearn__.*` 같은 패턴으로 matcher를 거는 게 실제로 될까?
- 훅 명령에서 프로젝트 위치를 가리키는 환경 변수(`$CLAUDE_PROJECT_DIR`)는 어떻게 쓰나?
- `prompt`·`agent` 훅은 내부에서 어떻게 동작하고, 비용은 얼마나 들까?
- Claude Code도 남이 만든 저장소의 훅을 실행하기 전에 확인 절차가 있을까?
- 설치된 Codex(0.165)에는 훅 시점이 어디까지 들어 있을까?
- Stop 훅이 계속 막으면 무한 루프가 되지 않을까? (8장)

## 참고 자료

- [Claude Code — Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code — Hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code — Best practices](https://code.claude.com/docs/en/best-practices)
- [Codex — Hooks](https://learn.chatgpt.com/docs/hooks)
- [Codex — MCP](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)
- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Addy Osmani — Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/)

## 정리 이력

- 2026-11-27: Claude와의 학습 대화, 실습, 확인 문제를 바탕으로 정리했다.
