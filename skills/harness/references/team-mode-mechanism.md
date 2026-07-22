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

## 스폰 규율 (중복 에이전트 방지 — 필수 절차)

**증상:** `architect→architect-2`, `ux-expert→ux-expert-2`처럼 같은 역할이 `-2` 접미사로 계속 늘어남. **원인:** 살아있는(실행 중이거나 유휴로 안 멈춘) 이름으로 `Agent`를 재호출 → 하네스가 `-2`를 붙여 별개 에이전트를 생성(latest-wins). 둘이 같은 산출 파일·태스크를 놓고 이중 소유.

**규칙: "재사용 아니면 중지 — blind-spawn 금지".** `Agent(name:X)`를 호출하기 **전에 반드시** 아래 절차를 밟는다:

```
1) X가 이미 존재하나? — TaskList(러닝 팀원) + 스폰 결과의 running-teammates 확인.
2) 분기:
   - 기존 X에게 이어서 시킬 일  → SendMessage(to:X)  [재개 — 새 에이전트 0]  ← 기본값
   - X를 fresh로 다시 만들어야  → TaskStop(X) 먼저 → 그 다음 Agent(name:X)
   - X가 없음(최초)            → Agent(name:X)
3) 절대 금지: 살아있는 X 위에 Agent(name:X) 재호출(= -2 생성).
```

| 의도 | 도구 | 결과 |
|------|------|------|
| 기존 에이전트 후속 작업 | `SendMessage(to:"X")` | transcript 재개, 중복 0 |
| 같은 이름 fresh 재시작 | `TaskStop("X")` → `Agent(name:"X")` | 이름 회수 후 재스폰, 중복 0 |
| 스폰 전 생사 확인 | `TaskList` / running-teammates | 재사용/중지 판단 |

**Phase 전환 위생:** 한 Phase가 끝나면 다음 Phase 스폰 전에 **직전 Phase의 유휴 팀원을 정리**한다 — 이어서 쓸 에이전트는 `SendMessage`로 재개, 안 쓸 에이전트는 `TaskStop`으로 종료. standby 원본을 방치하면 다음 Phase 동명 스폰이 충돌한다(이 규율 없이 방치 → `-2` 누적의 근본 원인).

**이미 이중 스폰됐으면:** 한쪽에 스탠드다운 `SendMessage` 후 `TaskStop`으로 종료해 단독 소유를 확정. `-2`가 최신이면 그것을 정본으로, 원본을 종료.

## 주의
- `Agent`의 `team_name` 파라미터가 보여도 쓰지 않는다(무시됨). `name`만 사용.
- 세션당 단일 팀이므로 "팀 활성 1개 제한"은 자연히 성립. Phase 전환에 별도 해체 불필요.
- `fork` 타입은 자기 자신을 복제(부모 컨텍스트 상속)하며 항상 백그라운드. 전문가 팀원은 `fork`가 아니라 커스텀/빌트인 `subagent_type`으로 스폰한다.
