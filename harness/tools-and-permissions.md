# 도구와 권한은 어떻게 설계할까?

[목차](README.md) · [이전: 컨텍스트가 차면 무슨 일이 생길까?](when-context-fills.md)

## 다루는 범위

모델이 쓰는 도구를 어떻게 만들고, 무엇을 허용하고 막을지 정리한다. 중간에 나온 기초 개념(스크립트, 셸, bash)도 함께 적는다.

- 도구는 모델을 위한 API다
- 결과와 오류 메시지는 모델이 읽을 글로 쓴다
- 스크립트, 셸, bash
- bash 하나와 전용 도구
- 권한은 되돌릴 수 있는지로 나누고, AI가 자기 울타리를 못 고치게 한다

권한과 샌드박스의 기본 동작은 [3장](instructions-and-memory.md#7-위험한-명령을-막는-층과-샌드박스)에서 다뤘다. 훅을 직접 거는 법은 [7번 주제](README.md#공부-계획)에서 자세히 다룬다.

## 시작한 질문

백엔드 개발자는 API를 설계할 때 "쓰는 사람이 편한가"를 고민한다. 그런데 도구를 쓰는 쪽은 사람이 아니라 모델이다.

1. 같은 원칙으로 만들면 될까?
2. 모델이 쓰는 도구 중 무엇을 허용하고 무엇을 막아야 할까?

## 내가 이해한 내용

### 1. 도구는 모델을 위한 API다

Anthropic은 도구 설계를 ACI(agent-computer interface)라고 부르고, 도구를 "결정적인 시스템과 비결정적인 에이전트 사이의 계약"이라고 설명한다.

| | 사람용 API | 모델용 도구 |
| --- | --- | --- |
| 문서를 읽는 때 | 한 번 읽고 익힌다 | 매번 설명만 보고 판단한다. 설명도 컨텍스트를 차지한다 |
| 실수했을 때 | 디버거로 원인을 찾는다 | 오류 메시지만 보고 다시 시도한다 |
| 호출 한 번의 비용 | 코드에 박아 두면 끝 | 호출 하나가 루프 한 바퀴. 시간도 토큰도 든다 |
| 결과를 읽을 때 | 필요한 필드만 골라 쓴다 | 결과 전체가 컨텍스트에 올라간다 |

**원칙 ①: 적게, 그리고 일 단위로.** 연락처 전체를 돌려주는 `list_contacts`보다 찾는 사람만 돌려주는 `search_contacts`가 낫다. 모델은 받은 걸 전부 컨텍스트에 올리기 때문이다. 기능이 겹치는 도구가 여러 개면 모델이 고르기 어려워지고, 이름이 겹치면 `asana_search`, `jira_search`처럼 접두사로 묶는다.

**원칙 ②: 실수하기 어렵게(poka-yoke).** 부품을 거꾸로 끼울 수 없게 모양을 만드는 것처럼, 도구도 실수할 여지를 없앤다. Anthropic은 모델이 상대 경로를 자꾸 틀리자 도구가 절대 경로만 받게 바꿨고, 그 뒤 모델이 흠잡을 데 없이 썼다고 한다. SWE-bench를 준비할 때는 전체 프롬프트보다 도구를 다듬는 데 시간을 더 썼다고 한다.

Claude Code의 도구 설명에도 그 흔적이 있다.

| 도구 | 설명에 적힌 것 | 막는 실수 |
| --- | --- | --- |
| Read | `file_path`는 절대 경로여야 한다 | 상대 경로 혼동 |
| Edit | 편집 전에 그 파일을 이 대화에서 Read 해야 한다. 아니면 실패한다 | 안 읽은 파일을 짐작으로 고치기 |
| Edit | `old_string`은 파일과 정확히 같고 유일해야 한다. 아니면 실패한다 | 엉뚱한 곳 고치기 |

아래 두 개는 설명으로만 있는 게 아니라 실제로 실패하게 만들어져 있다. 글이 아니라 코드로 강제하는 생각이 도구 설계 안에도 들어 있다.

### 2. 결과와 오류 메시지는 모델이 읽을 글로 쓴다

모델은 도구 결과만 보고 다음 행동을 정한다. 그래서 결과가 곧 모델에게 주는 프롬프트다.

**신호만, 짧게.** Anthropic 글의 예에서는 같은 결과를 자세히 주면 약 206토큰, 간단히 주면 약 72토큰이었다. 페이지네이션, 필터, 길이 제한을 쓰고, 하네스가 상한도 건다(Claude Code는 도구 응답을 기본 25,000토큰으로, mini-swe-agent는 1만 자가 넘는 출력을 줄인다).

**성공은 조용히, 실패는 자세히(back-pressure).**

```text
너무 짧음:  OK
딱 좋음:    Tests 1800 passed (12s)
            × 쿠폰 중복 적용 테스트: 기대 1000원, 실제 2000원 (coupon.test.ts:42)
너무 김:    ✓ 주문 생성 성공 ✓ 주문 취소 성공 … (1,800줄)
```

성공은 짧게 하되, 무엇이 확인됐는지와 다음에 필요한 정보(만든 파일 경로, 레코드 ID 등)는 남긴다. Anthropic은 C 컴파일러를 Claude 팀에게 만들게 하면서 테스트 하네스를 사람이 아니라 Claude가 읽을 용도로 다시 짰다. 출력은 짧게 하고, 긴 로그는 파일로 빼고, 일부만 빠르게 돌리는 옵션을 뒀다.

**오류는 어디서·왜·어떻게.**

```text
나쁜 오류:  Error: layer violation
좋은 오류:  src/application/order.service.ts:12
            application 계층이 presentation의 OrderDto를 import했습니다.
            DTO는 application/dto/로 옮기거나 매퍼를 거치세요.
            규칙: docs/architecture.md#의존-방향
```

"자세히"는 "길게"가 아니라 "정확하게"다. 500줄짜리 스택 트레이스는 오히려 핵심을 묻는다. 긴 설명은 문서에 두고 링크만 건다. OpenAI는 커스텀 린터의 오류 메시지에 고치는 방법까지 넣었고, Böckeler는 모델이 읽도록 쓴 센서 메시지를 "a positive kind of prompt injection"이라고 부른다. mini-swe-agent는 도구 호출이 없으면 "No tool calls found in the response. Every response MUST include at least one tool call."을 돌려주고, 오류 템플릿이 진짜 형식 실수와 `max_tokens`로 잘린 경우를 구분할 수 있게 만들어져 있다.

**메시지는 누가 쓰나.** 하네스는 결과를 전달할 뿐이고, 메시지 내용은 검사를 만든 사람이 쓴다. 사용자 하네스에서는 그게 우리다.

| 요소 | 누가 채우나 |
| --- | --- |
| 어디서(파일, 줄) | 검사 도구가 자동으로 |
| 왜(어떤 규칙) | 규칙에 이름·ID를 붙여 두면 자동으로 |
| 어떻게 고치나 | 우리 팀만 아는 지식이라 규칙마다 미리 써 둔다 |

설명 글은 규칙마다 한 번만 쓰고, 위치만 코드가 채운다. 백엔드의 ErrorCode enum과 같은 방식이다(설명용).

```js
const RULES = {
  "app-imports-presentation": {
    why: "application 계층이 presentation을 import했습니다.",
    fix: "DTO는 application/dto/로 옮기거나 매퍼를 거치세요.",
    doc: "docs/architecture.md#의존-방향",
  },
};

function report(file, line, ruleId) {
  const r = RULES[ruleId];
  console.error(`${file}:${line}`);
  console.error(r.why);
  console.error(r.fix);
  console.error(`규칙: ${r.doc}`);
}
```

Claude Code 훅은 막을 때 stderr에 쓴 메시지를 이유로 모델에게 전달하고, PostToolUse 훅은 `additionalContext`로 도구 결과 옆에 안내를 덧붙일 수 있다. 그래서 막는 것만큼 막을 때 뭐라고 말하느냐가 중요하다.

### 3. 스크립트, 셸, bash

**스크립트**는 실행하면 일 하나 하고 끝나는 작은 프로그램이다.

| | 애플리케이션(예: Spring 서버) | 스크립트 |
| --- | --- | --- |
| 실행하면 | 계속 떠 있으며 요청을 기다린다 | 일 하나 하고 바로 끝난다 |
| 실행 방법 | 빌드하고 배포한다 | `node 파일.mjs`, `python 파일.py`, `bash 파일.sh`처럼 바로 실행한다 |
| 용도 | 서비스 제공 | 손으로 하던 일을 자동화 |

스크립트는 언어가 아니라 쓰는 방식이다. 셸, PowerShell, Node.js, Python 무엇으로 짜도 되고, Java도 11부터 파일 하나짜리 소스를 컴파일 없이 바로 실행할 수 있다. `package.json`의 `"scripts"`, DB 마이그레이션 파일, 배포용 `.sh`, 계층 검사용 `.mjs`가 모두 스크립트다.

검사 스크립트는 코드를 읽고 규칙 위반을 찾으면 우리가 써 둔 문장을 출력하고, 성공·실패를 **종료 코드**로 남긴다. 0이면 성공, 0이 아니면 실패다. CI와 훅은 이 숫자를 본다. Claude Code 훅은 2로 끝나야 막으므로, 검사 스크립트를 훅에 걸려면 실패 시 `exit 2`로 끝내야 한다. Bean Validation에서 빈 값인지는 프레임워크가 검사하지만 `message = "이름은 필수입니다"`는 우리가 쓰는 것과 같다.

**셸**은 터미널에서 친 명령을 알아듣고 실행하는 프로그램이고, **bash**는 그중 가장 유명한 셸이다. Windows의 cmd·PowerShell, 맥의 zsh도 셸이다. Windows에는 bash가 기본으로 없어서 Git for Windows가 bash와 리눅스식 명령들을 묶어 설치해 주는데, 그게 Git Bash다.

셸이 실행하는 명령의 대부분은 셸이 아니라 따로 있는 프로그램이다. 셸은 맞는 프로그램을 찾아 실행해 주는 접수 창구다. DispatcherServlet이 요청을 받아 맞는 컨트롤러를 찾아 실행하고 일부만 직접 처리하는 것과 같다.

| 종류 | 예 | 정체 |
| --- | --- | --- |
| 셸 내장 명령 | `cd`, `echo`, `export` | bash 안에 들어 있는 기능 |
| 외부 프로그램 | `git`, `ls`, `grep`, `npm`, `node` | 디스크에 있는 별도 실행 파일 |

그래서 Claude Code의 Bash 도구는 모델에게 터미널을 통째로 빌려준 것과 같다. 파일 읽기(`cat`), 고치기(`sed`), 검색(`grep`), 테스트, git, 인터넷 요청(`curl`)까지 거의 뭐든 할 수 있다. 훅의 `"type": "command"`도 실행할 수 있는 명령이면 언어를 가리지 않고, 하네스는 출력과 종료 코드만 본다.

### 4. bash 하나와 전용 도구

| | mini-swe-agent | Claude Code |
| --- | --- | --- |
| 도구 | bash 하나 | bash + Read, Edit, Write, Grep, Glob, 질문하기 같은 전용 도구 |
| 장점 | 단순하고 어떤 모델에도 붙는다. README 기준 SWE-bench Verified 74%를 넘는다 | 하네스가 행동마다 따로 다룰 수 있다 |
| 하네스가 보는 것 | 모든 행동이 `bash: "명령 한 줄"`로 같은 모양 | `Edit { file_path, old_string, new_string }`처럼 행동마다 모양이 다르다 |

Anthropic의 Claude API 에이전트 설계 가이드는 bash가 모델에게 넓은 힘을 주지만 하네스에게는 "opaque command string"(속을 알 수 없는 명령 문자열)만 준다고 설명한다. 전용 도구로 빼면 하네스가 타입이 있는 인자를 받아 가로채고, 막고, 보여 주고, 기록할 수 있다.

| 전용 도구로 빼는 이유 | 가이드 설명 | Claude Code에서 |
| --- | --- | --- |
| 막아야 할 때 | 되돌리기 어려운 행동은 확인을 받아야 한다. `send_email`은 막기 쉽지만 `bash -c "curl -X POST …"`는 어렵다 | 권한 규칙과 훅을 도구별로 건다 |
| 상태를 확인해야 할 때 | 전용 `edit`은 마지막으로 읽은 뒤 파일이 바뀌었으면 쓰기를 거절할 수 있다 | Edit 전에 Read 해야 한다 |
| 화면에 보여 줘야 할 때 | 행동에 맞는 UI가 필요하면 도구로 뺀다 | 질문하기 도구가 선택지 창을 띄우고 답이 올 때까지 루프를 멈춘다 |
| 동시에 돌려야 할 때 | 읽기 전용 `grep`, `glob`은 동시에 돌려도 안전하다고 표시할 수 있다. bash로 하면 안전한 `grep`과 위험한 `git push`를 구분 못 해 하나씩 돌린다 | 읽기 도구 여러 개를 한꺼번에 실행한다 |

> 넓게는 bash로 시작하고, 막거나·보여 주거나·기록하거나·동시에 돌려야 할 때 전용 도구로 뺀다.

백엔드로 비유하면 bash 하나는 `POST /execute { "sql": "아무 SQL" }` 하나만 있는 API이고, 전용 도구는 `POST /orders`, `DELETE /orders/{id}`처럼 기능별 API다. `/execute`만 있으면 "삭제만 승인 받기"를 하려면 SQL 글자를 뜯어봐야 하고, 돌려 쓰면 놓친다. 3장에서 본 "deny 규칙은 다른 형태의 명령에 뚫릴 수 있다"도 bash가 속을 알 수 없는 글자 한 줄이라서 생긴다.

그래서 Claude Code 샌드박스는 셸 명령(Bash)과 그 명령이 띄운 프로그램에만 걸린다. Read·Edit·Write는 인자가 정해져 있어 권한 규칙으로 경로를 정확히 막을 수 있고, 속을 알 수 없는 Bash는 실행 중 동작을 샌드박스로 감시한다.

Codex는 셸에 더해 파일 수정용 `apply_patch`와 계획 표시용 `update_plan`을 따로 두고, Hermes는 기본 도구가 60~70개 넘게 있다.

### 5. 권한은 되돌릴 수 있는지로 나눈다

Anthropic 가이드는 되돌릴 수 있는지(reversibility)를 기준으로 쓰라고 한다.

| 칸 | 기준 | 예 |
| --- | --- | --- |
| 묻지 않고 허용(allow) | 읽기만 하거나 git으로 되돌릴 수 있는 것 | `git status`, `git diff`, `npm test`, 파일 읽기 |
| 물어보기(ask) | 밖으로 나가서 되돌리기 어려운 것 | `git push`, 배포, 메시지 보내기, 외부 API 호출 |
| 막기(deny) | 절대 하면 안 되는 것 | 비밀 파일 읽기, 위험한 삭제 |
| 울타리로 받치기(sandbox) | 속을 알 수 없는 코드를 실행하는 것 | 테스트 실행, 패키지 설치 |

**허용 목록의 빈틈.** `npm test`를 허용 목록에 넣으면 권한 확인은 `npm test`라는 이름만 보고 통과시킨다. 그런데 그 명령은 `package.json`의 test 명령과 테스트 파일 안의 코드를 전부 실행하고, 그 파일들은 AI가 고칠 수 있다.

```text
AI가 직접:      "Bash로 rm -rf ./data"      → 허용 목록에 없음 → 사람에게 물어봄
테스트에 넣고:  (테스트에 ./data 지우는 코드) → "npm test" → 허용 목록에 있음 → 그냥 통과
```

AI가 일부러 그런다는 게 아니라, 실수(잘못된 경로 정리, 목 대신 실제 DB·외부 API 호출)나 속임수(읽은 문서에 숨은 지시를 따르는 프롬프트 주입)로 생긴다. 결과적으로 "테스트니까 괜찮겠지" 하고 열어 둔 통로로 AI가 쓴 코드가 검사 없이 실행돼 실제 환경(파일, 데이터, 외부 서비스, 비밀 정보)에 피해가 생길 수 있다. 그렇다고 `npm test` 허용이 틀린 건 아니다. 매번 물으면 너무 불편하므로 **허용하되 샌드박스로 받친다.**

**너무 많이 물으면 위험하다.** 묻는 게 많아지면 사람이 안 읽고 승인만 누르게 된다. 그래서 Claude Code는 auto 모드에서 분류기가 위험한 것만 막고, Codex는 승인 요청을 리뷰어 에이전트에게 보내 리뷰가 실패하면 막는 설정(`approvals_reviewer = "auto_review"`)이 있다.

**AI가 자기 울타리를 못 고치게.** Claude Code 샌드박스는 `.claude` 설정, `.mcp.json`, `.git/hooks` 같은 곳의 쓰기를 막는다. AI가 훅이나 권한 설정을 고칠 수 있으면 막는 장치를 스스로 풀 수 있기 때문이다. 완료 게이트 같은 검사를 만들 때도 검사 스크립트와 훅 설정은 AI가 건드리지 못하게 해야 한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 좋은 오류는 왜 오류가 났는지 자세히 말하는 것이고, 성공은 "좋다"면 충분하다 | 오류는 어디서·왜·어떻게 고치나를 정확하게. "자세히"는 "길게"가 아니다. 성공도 무엇이 확인됐는지는 보이게 한다. |
| 하네스가 그런 자세한 오류를 알려 줄 수 있나 | 하네스는 전달만 하고, 메시지 내용은 검사를 만든 사람이 쓴다. 고치는 방법은 우리 팀만 아는 지식이라 규칙마다 미리 써 둔다. |
| "검사 스크립트가 우리가 쓴 메시지를 출력한다"는 게 무슨 뜻인지 | 스크립트 안의 `console.error("…")`에 우리가 쓴 문장이 그대로 출력된다. Bean Validation의 `message`와 같다. |
| 스크립트는 셸로 짜는 것이다 | 스크립트는 언어가 아니라 "바로 실행해서 일 하나 하고 끝나는" 쓰는 방식이다. |
| bash에서 쓰는 명령어는 모두 셸이다 | 셸은 접수 창구이고, `git`, `npm`, `ls` 같은 대부분의 명령은 따로 있는 프로그램이다. `cd`, `echo`만 셸 내장이다. |
| `npm test`는 테스트만 돌리니까 허용 목록에 넣어도 안전하다 | 그 안에서 실행되는 코드는 AI가 고칠 수 있어 권한 확인을 우회하는 뒷문이 된다. 허용하되 샌드박스로 받친다. |
| 긴 실패 사유는 코드에 넣을 수 없다 | 규칙마다 설명을 한 번만 써 두고 위치만 코드가 채운다(ErrorCode enum 방식). 긴 설명은 문서에 두고 링크한다. |

### 공부하며 던진 질문

- **샌드박스는 Bash일 때만 걸리나?** Claude Code에서는 그렇다. 전용 도구는 인자로 정확히 막을 수 있고, 속을 알 수 없는 Bash는 실행 중 동작을 감시해야 하기 때문이다.
- **Git Bash의 bash가 이 bash인가?** 그렇다. Windows에서 bash와 리눅스식 명령을 쓰게 Git for Windows가 묶어 준 것이다.
- **AI가 테스트 코드를 고칠 수 있는 게 왜 문제인가?** 그 자체가 아니라, AI가 쓴 코드가 묻지 않고 실행되는 통로(`npm test` 허용)를 타고 바로 실행되는 조합이 문제다.

## 예제와 확인 결과

- **셸 내장 명령과 외부 프로그램:** Windows 11의 Git Bash에서 `type cd echo git ls grep npm node`를 실행한 결과다.

```text
cd is a shell builtin
echo is a shell builtin
git is /mingw64/bin/git
ls is /usr/bin/ls
grep is /usr/bin/grep
npm is /c/Program Files/nodejs/npm
node is /c/Program Files/nodejs/node
```

- **Claude Code 도구 설명:** Read의 절대 경로 조건, Edit의 Read 선행 조건과 `old_string` 유일성 조건은 Claude Code 2.1.330의 도구 설명에서 확인했다. Edit이 조건을 어기면 실제로 실패한다.
- **mini-swe-agent:** v2.4.6(커밋 `04d809c`)의 `models/utils/actions_toolcall.py`에서 오류 문장과 `max_tokens` 구분을 확인했다.
- **설명용:** `RULES` 예시 코드, 오류·테스트 출력 예시, SQL API 비유, 권한 분류 예시는 설명용이다.

## 남은 질문

- Codex가 파일 수정을 셸이 아니라 `apply_patch`라는 전용 도구로 둔 이유는 무엇일까? 모델 학습과도 관련이 있을까?
- MCP로 붙인 도구는 샌드박스와 권한에서 어떻게 다뤄질까?
- Claude Code auto 모드의 분류기는 무엇을 기준으로 위험하다고 판단할까?
- 프롬프트 주입을 하네스 쪽에서 어떻게 막을 수 있을까?
- 도구 설명을 고친 뒤 정말 나아졌는지는 어떻게 잴까? (11장)

## 참고 자료

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic — Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Anthropic — Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)
- Anthropic — Claude API 에이전트 설계 가이드("Bash vs. dedicated tools"), Claude Code에 포함된 claude-api 스킬 문서
- [OpenAI — Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [Böckeler — Harness engineering for coding agent users (martinfowler.com)](https://martinfowler.com/articles/harness-engineering.html)
- [HumanLayer — Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- [Claude Code — Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Code — Permission modes](https://code.claude.com/docs/en/permission-modes)
- [Claude Code — Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Claude Code — Hooks](https://code.claude.com/docs/en/hooks)
- [Codex — Approvals and security](https://learn.chatgpt.com/docs/agent-approvals-security)
- [SWE-agent/mini-swe-agent (GitHub)](https://github.com/SWE-agent/mini-swe-agent)

## 정리 이력

- 2026-11-27: Claude와의 학습 대화와 복습을 바탕으로 정리했다. 셸 명령 확인 결과는 직접 실행한 것이다.
