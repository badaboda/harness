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

# 팀 모드 실행 메커니즘 (Claude Code 2026-07+ 업데이트)

Claude Code 업데이트로 에이전트 팀 실행 방식이 바뀌었다. **`TeamCreate`/`TeamDelete`는 더 이상 없다.** 이 문서가 현재 방식의 단일 진실 소스다. 오케스트레이터·에이전트 정의에서 팀을 구성할 때 이 방식을 따른다.

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

## 무엇이 바뀌었나

| 구분 | 이전 (deprecated) | 현재 (2026-07+) |
|------|-------------------|-----------------|
| 팀 생성 | `TeamCreate(team_name, members:[...])` 한 번에 팀 스폰 | **`Agent` 도구**로 팀원을 개별 스폰. `name`을 주면 팀원이 됨 |
| 팀 개념 | 명시적 팀 객체 | **단일 암묵 팀** — 한 세션이 곧 하나의 팀. spawn한 named 에이전트들이 팀원 |
| `team_name` | 필수 | **deprecated·무시됨** (`Agent`의 `team_name` 파라미터는 있어도 무시) |
| 팀 해체 | `TeamDelete` | 없음. 에이전트는 작업 완료 시 종료. Phase 재구성은 새 `Agent` 스폰으로 |
| 통신 | `SendMessage({to: name})` | **동일** — `SendMessage({to: name})` |
| 조율 | `TaskCreate`/`TaskUpdate` 공유 목록 | **동일** |
| 재개 | — | `SendMessage`를 그 이름으로 보내면 기존 에이전트가 컨텍스트 유지한 채 재개 |

## 현재 방식 4원칙

1. **팀원 = named `Agent` 호출.** `Agent(subagent_type, name, model:"opus", prompt)`. `name`을 주면 `SendMessage({to:name})`로 실행 중 통신 가능. 커스텀 타입은 `.claude/agents/{name}.md`가 정의.
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
- `-N`이 붙으면 = 동명 잔류 충돌 신호(teardown 누락) → 반환명 그대로 정본화(오탐 금지, 역할당 라이브 1개면 정상) + **원인 잔류를 규율 1로 정지**.

### 규율 3 — 세션 내 재사용/재시작
- 이어서 시킬 일 → `SendMessage(to:로스터명)`(재스폰 0, 기본값). fresh 필요 → `TaskStop(로스터명)` 후 재스폰. **살아있는 로스터명 위 동명 재스폰 금지**(증상 A).

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
- `Agent`의 `team_name` 파라미터가 보여도 쓰지 않는다(무시됨). `name`만 사용.
- 세션당 단일 팀이므로 "팀 활성 1개 제한"은 자연히 성립. 단 **Phase 전환·세션 종료 teardown은 필수**(규율 1) — 해체를 안 하면 유휴 팀원이 잔류해 다음 Phase 동명 스폰과 충돌(`-2`)한다.
- `TaskList`는 잔류를 못 보여 "고정 이름이 비었다"고 오판시킨다 → 잔류 유무는 `TaskList`가 아니라 **`TaskStop` 프로브(running-teammates 목록)**로 확인. 잔류는 정지 가능하다(관측상 세션 내 in_process_teammate로 잡힘).
- `fork` 타입은 자기 자신을 복제(부모 컨텍스트 상속)하며 항상 백그라운드. 전문가 팀원은 `fork`가 아니라 커스텀/빌트인 `subagent_type`으로 스폰한다.
