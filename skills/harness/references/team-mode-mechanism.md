<!--
Copyright 2026 marcus-kkb

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# 팀 모드 실행 메커니즘 (Claude Code 2.1.x 기준)

Claude Code 업데이트로 에이전트 팀 실행 방식이 바뀌었다. **`TeamCreate`/`TeamDelete`는 더 이상 없다**(v2.1.178에서 제거). 이 문서가 현재 방식의 단일 진실 소스다. 오케스트레이터·에이전트 정의에서 팀을 구성할 때 이 방식을 따른다.

> 근거: 공식 문서 [agent-teams](https://code.claude.com/docs/en/agent-teams) · [sub-agents](https://code.claude.com/docs/en/sub-agents). 런타임 제약을 바꾸기 전 원문을 재확인하라.

## 🚫 최우선 안티패턴 — `*-team-lead` 서브에이전트에 오케스트레이션 위임 금지

**팀은 1계층이다. 이름 있는 팀메이트(`Agent(name:...)`)는 메인 대화만 만들 수 있다.**

`foo-team-lead` 같은 오케스트레이터 에이전트를 정의해 서브에이전트로 스폰하고 "네가 팀을 소집해라"라고 위임하면:

- 그 lead 는 **이미 팀메이트**라서 그 아래로 이름 있는 팀메이트를 만들지 못한다
- 하네스가 **무명 서브에이전트로 폴백**한다 → `SendMessage(to:name)` 재개 불가 · 공유 `TaskCreate` 목록 미참여 · 호출마다 풀 컨텍스트 재전달
- 즉 팀 모드의 핵심(팀원 간 직접 통신·상충 토론·누락 보완)이 **조용히 사라진다**. 실행은 계속되므로 알아채기 어렵다

**올바른 형태**: 오케스트레이터는 **스킬**이고, 그 스킬을 읽는 **메인 대화가 리더**다. 메인이 `TaskCreate` 로 공유 목록을 만들고 전문가들을 직접 `Agent(name:...)` 로 스폰한다.

```
❌  Agent(name:"foo-lead", subagent_type:"foo-team-lead", prompt:"팀 8명 소집해서 큐 소진해라")
        └─ foo-lead 가 스폰하는 것들은 전부 무명 → 팀 아님

✅  (메인이 스킬을 읽고 직접)
    TaskCreate(...) × N
    Agent(name:"oracle", ...)  Agent(name:"eng", ...)  Agent(name:"gate", ...)
```

기존 프로젝트에 `*-team-lead` 에이전트 정의가 쌓여 있어도 그것이 근거가 되지 않는다 — 관행이 아니라 이 문서가 정본이다. 새 하네스에는 `*-team-lead` 에이전트를 **만들지 마라**. 오케스트레이션 지침은 오케스트레이터 스킬 본문에 둔다.

> **예외**: lead 를 *조언자*(설계 리뷰·계획 수립)로 쓰는 것은 무방하다. 금지되는 것은 **팀 소집 권한의 위임**이다.

## ⚠️ 전제조건 — 에이전트 팀은 실험 기능이고 **기본값이 꺼져 있다**

**공식 문서 정본:** 에이전트 팀은 experimental이며 **기본 비활성**이다. `settings.json` 또는 환경변수에 **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`** 을 설정해야 켜진다.

```json
// settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

**끄여 있으면:** 세션 시작 시 팀이 구성되지 않고, 팀 디렉토리도 만들어지지 않으며, **Claude가 팀원을 스폰하지도 제안하지도 않는다.** 즉 이 문서의 팀 방식 전체가 성립하지 않는다.

**그래서 팀 모드로 하네스를 설계하기 전에 반드시:**

1. **플래그가 켜져 있는지 확인한다.** 꺼져 있으면 **사용자에게 알리고**, 켤지(설정 변경) 아니면 다른 모드로 갈지 결정받는다. 하네스가 임의로 사용자 설정을 바꾸지 않는다.
2. **끈 상태로 "팀"을 시도하면 조용히 서브에이전트가 된다.** 문서도 이를 경고한다 — *"Claude may sometimes use subagents instead of creating a team. Subagents appear in the same agent panel as teammates, so the panel alone doesn't confirm a team formed."* 패널에 에이전트가 보인다고 팀이 만들어진 게 아니다. 위 안티패턴과 **완전히 같은 실패 모드**다.
3. **플래그가 꺼진 환경에서의 선택지:** 팀 통신이 설계의 핵심이 아니면 서브 에이전트 모드로, 흐름이 사전 확정이면 Workflow 모드로 설계한다. **팀인 척하는 무명 스폰이 가장 나쁜 선택이다.**

### 켜져 있을 때의 팀 골격 (문서 확인 사항)

- **팀은 첫 팀원이 스폰되는 순간 형성**되고, 메인 세션이 리드다. 별도 생성 단계 없음.
- **이름은 리드가 부여한다** — "The lead assigns every teammate a name when it spawns them." 나중에 참조할 이름을 고정하려면 스폰 지시에 이름을 명시한다.
- **팀 이름은 세션 파생**(`session-` + 세션 ID 앞 8자). 팀 설정은 `~/.claude/teams/{team}/config.json`, 공유 작업목록은 `~/.claude/tasks/{team}/`.
- **정리는 세션 종료 시 자동**이다. 팀 설정 디렉토리는 제거되고, **작업목록 디렉토리는 남아** 재개 세션이 태스크를 유지한다.
- **세션당 팀 1개, 중첩 팀 불가, 리드 고정**(양도 불가) — 위 안티패턴의 근거.
- **팀원은 리드의 `/model` 선택을 기본 상속하지 않는다.** `/config`의 "Default teammate model"로 정하거나 스폰 프롬프트에 지정한다. **effort는 리드에서 상속**된다.
- **서브에이전트 정의를 팀원 역할로 재사용**할 수 있다. 이때 정의의 `tools`·`model`은 적용되고 본문은 시스템 프롬프트에 **덧붙는다**(대체 아님). 단 **`skills`·`mcpServers` frontmatter는 팀원으로 실행될 때 적용되지 않는다** — 팀원은 프로젝트/사용자 설정에서 스킬·MCP를 로드한다.
- **`SendMessage`와 태스크 관리 도구는 `tools`로 제한해도 팀원에게 항상 제공된다.**

## 무엇이 바뀌었나

| 구분 | 이전 (deprecated) | 현재 (2026-07+) |
|------|-------------------|-----------------|
| 팀 생성 | `TeamCreate(team_name, members:[...])` 한 번에 팀 스폰 | **`Agent` 도구**로 팀원을 개별 스폰. `name`을 주면 팀원이 됨 |
| 팀 개념 | 명시적 팀 객체 | **단일 암묵 팀** — 한 세션이 곧 하나의 팀. spawn한 named 에이전트들이 팀원 |
| `team_name` | 필수 | **입력은 받지만 무시된다(deprecated).** 훅 페이로드의 `team_name`도 세션 파생 이름을 실어 나를 뿐이다. 쓰지 않는다 |
| 팀 해체 | `TeamDelete` | 없음. 세션 종료 시 자동 정리. Phase 재구성은 새 `Agent` 스폰으로 |
| 백그라운드 | `Agent(run_in_background:true)` 로 명시 | **v2.1.198부터 서브에이전트는 기본이 백그라운드다.** Claude가 결과를 바로 써야 할 때만 포그라운드로 돌린다. 항상 백그라운드로 고정하려면 정의 frontmatter `background: true` |
| 통신 | `SendMessage({to: name})` | **동일** — 단 `summary`(5~10단어)가 함께 필요: `SendMessage({to, summary, message})` |
| 조율 | `TaskCreate`/`TaskUpdate` 공유 목록 | **동일** |
| 재개 | — | `SendMessage`를 그 이름으로 보내면 기존 에이전트가 컨텍스트 유지한 채 재개 |

## 현재 방식 4원칙

1. **팀원 = named `Agent` 호출.** `Agent(subagent_type, name, model:"opus", prompt)`. `name`을 주면 `SendMessage({to:name})`로 실행 중 통신 가능. 커스텀 타입은 `.claude/agents/{name}.md`가 정의(frontmatter에 `model: opus`를 넣으면 호출마다 `model`을 반복하지 않아도 된다).
2. **동시 실행 = 한 메시지에 여러 `Agent` 호출.** 독립 작업은 단일 메시지에 여러 `Agent` 도구를 넣으면 병렬 실행된다. 의존 작업은 선행 에이전트 완료 후 다음 메시지에서 스폰(파이프라인).
3. **조율 = 공유 작업목록 + 메시지.** `TaskCreate`로 작업 등록(`addBlockedBy`로 의존성), 에이전트는 `TaskList`로 claim·`TaskUpdate`로 상태 갱신, `SendMessage`로 상호 통신. 백그라운드 에이전트는 `SendMessage({to:"main"})`로 리더(메인 대화)에 보고.
4. **Phase 재구성 = 새 스폰.** `TeamDelete` 없음. 다른 전문가 조합이 필요하면 이전 산출물을 `_workspace/`에 저장해 두고(에이전트 종료 후에도 보존) 다음 Phase에서 필요한 에이전트를 새로 `Agent` 스폰한다. 종료된 에이전트도 이름으로 `SendMessage`하면 재개된다.

## 코드 패턴

**이전 (쓰지 말 것):**
```
TeamCreate(team_name:"research-team", members:[
  {name:"researcher", agent_type:"general-purpose", prompt:"..."},
  {name:"analyst", agent_type:"general-purpose", prompt:"..."},
])
```

**현재 — 팬아웃(동시): 한 메시지에 여러 Agent 호출**
```
// 단일 메시지 안에서:
Agent(subagent_type:"researcher", name:"researcher", model:"opus", prompt:"...")
Agent(subagent_type:"analyst",    name:"analyst",    model:"opus", prompt:"...")
// → 병렬 실행. 이후 SendMessage({to:"researcher"})로 통신, TaskUpdate로 조율.
```

**현재 — 파이프라인(의존): 선행 완료 후 다음 스폰**
```
// 메시지 1: 계약 담당 먼저
Agent(subagent_type:"architect", name:"architect", model:"opus", prompt:"계약 산출 후 main에 통지")
// (architect 완료·통지 수신 후)
// 메시지 2: 계약에 의존하는 구현진 동시 스폰
Agent(subagent_type:"data-eng",    name:"data-eng",    model:"opus", prompt:"...")
Agent(subagent_type:"backend-eng", name:"backend-eng", model:"opus", prompt:"...")
```

**공유 작업목록 + 의존성 (스폰 전/후):**
```
TaskCreate(subject:"계약 정의", description:"...")            // T1
TaskCreate(subject:"구현", description:"...")                 // T2
TaskUpdate(taskId:"2", addBlockedBy:["1"], owner:"data-eng") // T2는 T1 의존
```

## 오케스트레이터 작성 시 표기

오케스트레이터 스킬의 "팀 구성" 섹션은 `TeamCreate(...)` 블록 대신 위 **named `Agent` 스폰 패턴**으로 쓴다. 리더(오케스트레이터를 실행하는 메인)가:
1. `TaskCreate`로 공유 작업목록 + 의존성 구성
2. 파이프라인 게이트(예: 계약 담당)를 먼저 `Agent` 스폰
3. 게이트 완료 후 독립 구현진을 **한 메시지에 여러 `Agent`**로 팬아웃
4. `SendMessage`로 경계면 계약·gap 전달, `TaskUpdate`로 진행 추적
5. 백그라운드 에이전트의 `to:"main"` 보고를 수신해 종합

## 스폰 규율 (중복·유령 에이전트 방지 — 필수 절차)

**증상 A(같은 Phase 내):** `architect→architect-2`처럼 살아있는 팀원 위에 동명 재스폰 → 이중 소유(latest-wins).
**증상 B(Phase 전환 잔류):** 다음 Phase에서 고정 역할이름(`data-eng`)으로 스폰했는데 `-2`가 붙거나, 이전 Phase 팀원이 아직 살아 지시 안 한 일을 하거나 P2P가 엉뚱하게 닿음.

**실측 근본원인(2026-07-22 확정):** `TaskStop`으로 조회하니 정지된 중복 3개(`data-eng`·`backend-eng`·`architect-2`)가 **전부 같은 세션이고 프롬프트가 이전 Phase(Phase 2)**였다. 즉 세션-간 유령이 아니라 — **이전 Phase 팀원을 Phase 전환 때 teardown하지 않아 잔류**하고, 그 잔류가 새 Phase의 동명 스폰과 충돌해 `-2`가 붙은 것. **∴ 근본 해법은 teardown(규율 1). 세션-스코프 이름(규율 2)은 잔류가 있어도 충돌을 막는 보조 방벽.**

### 규율 1 — Phase/세션 teardown (근본 해법, 필수)
잔류를 **애초에 남기지 않는다.** 유휴 팀원 방치가 `-2`의 근본 공급원이다.
- **Phase 전환:** 다음 Phase를 스폰하기 **전에** 직전 Phase 팀원을 반드시 정리 — 이어서 쓸 팀원은 `SendMessage`로 재개(재스폰 0), 안 쓸 팀원은 **`TaskStop`으로 종료**. standby 방치 금지.
- **세션 마무리:** main이 **라이브 로스터 전원에 `TaskStop`**. 못 닿으면(이미 죽음) 무해.
- **로스터 확인:** `TaskStop`에 존재하지 않는 id를 주면 에러에 **현재 running-teammates 전체 목록**이 나온다. 이걸로 실 로스터를 확인하고, 활성 로스터에 없는 이름(잔류)을 `TaskStop`. 이전-Phase 프롬프트를 든 잔류도 이렇게 잡아 정지한다.

### 규율 2 — 세션-스코프 이름 (충돌 방벽, 보조)
teardown을 놓쳐 잔류가 남더라도 **이름이 안 겹치면 `-2`가 안 생긴다.** main은 스폰 시 역할이름에 **Phase/세션 토큰**을 붙인다: `Agent(name:"data-eng-p3")`, `Agent(name:"feng-p3")`.
- **반환명 캡처:** 스폰이 반환한 **실제 이름을 정본으로 채택**해 `STATE.md`의 "라이브 로스터(이번 세션/Phase)"에 기록. 이후 모든 라우팅(`SendMessage`)은 **로스터에 적힌 이름으로만**.
- **문서가 뒷받침하는 안전장치(v2.1.199+):** `SendMessage`는 그 이름이 **아까 닿았던 그 에이전트를 여전히 가리키는지 검사**한다. 더 새 에이전트가 이름을 가져갔으면 **잘못 배달하지 않고 거부**하고, 에러가 "지금 그 이름이 누구를 가리키는지"를 알려준다. 이때 **먼저 스폰될 때 받은 `agentId`로 주소를 바꿔** 원래 에이전트에 닿는다. 검사 범위는 현재 대화이며 `/clear`로 초기화된다. → **로스터에 이름과 `agentId`를 함께 적어두면 `-N` 충돌 시 즉시 복구된다.**
- `-N`이 붙으면 = 동명 잔류 충돌 신호(teardown 누락) → 반환명 그대로 정본화(오탐 금지, 역할당 라이브 1개면 정상) + **원인 잔류를 규율 1로 정지**.

### 규율 3 — 세션 내 재사용/재시작
- 이어서 시킬 일 → `SendMessage(to:로스터명)`(재스폰 0, 기본값). fresh 필요 → `TaskStop(로스터명)` 후 재스폰. **살아있는 로스터명 위 동명 재스폰 금지**(증상 A).
- 문서 확인: **완료된 서브에이전트가 `SendMessage`를 받으면 `Agent` 재호출 없이 백그라운드에서 자동 재개**된다. `TaskStop`으로 멈춘 것도 마찬가지다. → 재개가 기본이고 재스폰은 예외라는 규율의 근거.

### 규율 4 — main 브로커 + File-first (팀원 조율)
- 팀원이 다른 팀원의 산출이 필요하면 **파일을 읽고 main에 보고 → main이 릴레이**. 팀원→팀원 `SendMessage`는 best-effort(의존 금지). 팀원은 다른 팀원을 스폰·가정하지 않는다 — 협업(예: ux↔domain 논의)은 main이 양쪽 스폰 후 파일+릴레이로 수렴.

| 의도 | 도구 | 결과 |
|------|------|------|
| Phase 전환·세션 종료 | 유휴/전원 `TaskStop` **(먼저)** | 잔류 0 → 다음 스폰 충돌 0 |
| 신규 스폰 | `Agent(name:"역할-<Phase토큰>")` → 반환명 로스터 기록 | 이름 충돌 방벽 |
| 세션 내 후속 | `SendMessage(to:"로스터명")` | transcript 재개, 중복 0 |
| 같은 이름 fresh | `TaskStop("로스터명")` → 재스폰 | 이름 회수 후 재스폰 |

**이미 중복/잔류가 관측되면:** `TaskStop` 프로브로 running-teammates를 확인 → 활성 로스터에 없는 이름은 잔류(대개 이전-Phase 프롬프트) → `TaskStop`으로 정지. 활성 빌드 팀원은 중간에 죽이지 말 것(작업 손실) — teardown은 Phase/세션 경계에서.

## 주의
- `team_name`은 **입력을 받지만 무시된다**(deprecated). 오케스트레이터에서 지정하지 않는다.
- **백그라운드는 이제 기본값이다**(v2.1.198+). "병렬 실행하려면 `run_in_background:true`" 같은 옛 표기는 불필요 — 병렬은 **한 메시지 다중 `Agent` 호출**로 성립한다. 항상 백그라운드로 고정할 역할은 정의 frontmatter `background: true`.
  - ⚠️ **백그라운드 서브에이전트는 빌트인 도구가 축소된다.** MCP 도구는 전부 유지되지만 빌트인은 `Read`·`Grep`·`Glob`·`Bash`·`PowerShell`·`Edit`·`Write`·`NotebookEdit`·`WebFetch`·`WebSearch`·`Skill`·`ToolSearch`·`EnterWorktree`·`ExitWorktree`·`Monitor`·`TaskStop`·`SendMessage`·`Artifact` 정도로 제한되고, **이 목록에 `TaskCreate`/`TaskUpdate`/`TaskList`가 없다.**
    - **이 도구들이 폐기된 것이 아니다.** 오히려 **현행 표준**이다 — `TodoWrite`가 v2.1.142부터 **기본 비활성**이 되면서 그 자리를 `TaskCreate`/`TaskGet`/`TaskList`/`TaskUpdate`가 대체했다. 메인 대화·포그라운드 서브에이전트·팀원·fork에서는 정상 동작하고, **백그라운드 서브에이전트라는 실행 맥락에서만 도구 목록에 들어오지 않는다.** 같은 정의 파일이라도 포그라운드/백그라운드에 따라 해석되는 도구가 달라지며, 이 제거는 **에러 없이 조용히** 일어난다(`tools` 목록이 통째로 비는 경우만 에러).
    - Task 계열 중 실제로 deprecated인 것은 **`TaskOutput` 하나**다(출력은 태스크 출력 파일 경로를 `Read`하는 방식으로 대체). 백그라운드 허용 목록에 `TodoWrite`가 보여도 기본 비활성이므로 대안으로 삼지 않는다.
    - ∴ 백그라운드 *서브에이전트*에게 공유 작업목록 claim을 시키는 설계는 동작하지 않는다. 조율이 필요하면 팀 모드를 쓰거나 파일 기반으로 간다. (팀 *팀원*은 예외 — `SendMessage`와 태스크 도구가 `tools` 제한과 무관하게 항상 제공된다.)
    - 모든 서브에이전트에서 제거되는 도구(필터 1)도 별도로 있다: `AskUserQuestion`·`EnterPlanMode`·`ExitPlanMode`·`EndConversation`·`ScheduleWakeup`·`TaskOutput`·`WaitForMcpServers`·`Workflow`, 그리고 깊이 한도에 닿은 경우의 `Agent`. **서브에이전트는 `Workflow`를 호출할 수 없다** — 워크플로우 오케스트레이션은 메인만 시작할 수 있다. `fork`는 두 필터를 모두 건너뛴다.
  - 포그라운드로 강제하려면 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`. fork 모드(`CLAUDE_CODE_FORK_SUBAGENT=1`)에서는 모든 서브에이전트가 백그라운드가 되고 `Agent`의 `run_in_background` 파라미터 자체가 사라진다.
- `SendMessage`는 `to`·`message` 외에 **`summary`(5~10단어)** 를 함께 넘겨야 한다. 오케스트레이터 예시에 빠뜨리지 말 것.
- **스폰 총량 제한:** 기본 **세션당 서브에이전트 200개**(`CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION`으로 상향 가능, 끌 수는 없음). 동시 실행 제한·중첩 깊이 제한도 별도로 있다. 한도에 닿으면 `Subagent spawn limit reached`로 실패하므로, 대규모 팬아웃을 설계할 때 반복 스폰 횟수를 계산에 넣는다.
- ⚠️ **팀원은 worktree로 격리되지 않는다.** 공식 문서 명시: *"Agent teams don't isolate teammates in worktrees, so partition the work so each teammate owns a different set of files."* 즉 팀 모드에서 파일 충돌의 해법은 **파일 소유권 분할뿐**이다 — 팀 구성 단계에서 팀원별 담당 파일 집합을 겹치지 않게 설계하라. (worktree 격리는 *서브에이전트*와 사용자가 직접 띄우는 세션에서 쓰는 수단이다.)
- 세션당 단일 팀이므로 "팀 활성 1개 제한"은 자연히 성립. 단 **Phase 전환·세션 종료 teardown은 필수**(규율 1) — 해체를 안 하면 유휴 팀원이 잔류해 다음 Phase 동명 스폰과 충돌(`-2`)한다.
- `TaskList`는 잔류를 못 보여 "고정 이름이 비었다"고 오판시킨다 → 잔류 유무는 `TaskList`가 아니라 **`TaskStop` 프로브(running-teammates 목록)**로 확인. 잔류는 정지 가능하다(관측상 세션 내 in_process_teammate로 잡힘).
- `fork` 타입은 자기 자신을 복제(부모 컨텍스트 상속)하며 항상 백그라운드. 전문가 팀원은 `fork`가 아니라 커스텀/빌트인 `subagent_type`으로 스폰한다.
