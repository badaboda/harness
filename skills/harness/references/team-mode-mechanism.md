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

## 스폰 규율 (중복·유령 에이전트 방지 — 필수 절차)

**증상 A(세션 내):** `architect→architect-2`처럼 살아있는 팀원 위에 동명 재스폰 → 이중 소유(latest-wins).
**증상 B(세션 간):** 새 세션에서 고정 역할이름(`data-eng`)으로 스폰했는데 `-2`가 붙거나, 지난 세션 팀원(유령)이 아직 reachable해 지시 안 한 일을 함. **원인:** 이름 레지스트리는 **세션을 넘어 잔류**하는데 `TaskList`/`TaskStop`은 **세션 스코프**라 둘이 어긋난다 — 잔류 이름은 TaskList에 안 보이고(→ "없음"으로 오판 후 스폰 → `-2`) TaskStop으로 못 죽인다(→ 깨끗한 이름 회수 불가). 팀원→팀원(P2P) 이름 해석도 잔류와 뒤섞여 엉뚱한 유령에 닿는다.

### 규율 1 — 세션-스코프 이름 (원천 차단, 핵심)
**고정 역할이름을 재사용하지 않는다.** main은 스폰 시 역할이름에 **세션/Phase 유니크 토큰**을 붙인다: `Agent(name:"data-eng-p3")`, `Agent(name:"feng-p3")`. 잔류(`data-eng`)와 절대 겹치지 않아 `-N`이 원천 소멸하고 유령과 이름이 분리된다.
- **반환명 캡처:** 스폰이 반환한 **실제 이름을 정본으로 채택**해 `STATE.md`의 "라이브 로스터(이번 세션)"에 기록. 이후 모든 라우팅(`SendMessage`)은 **로스터에 적힌 이름으로만**.
- `-N`이 그래도 붙으면 = 세션-간 잔류 충돌 신호일 뿐 **결함이 아니다** → 반환명 그대로 정본화(오탐 금지). 세션 내 라이브가 역할당 1개면 정상.

### 규율 2 — 세션-시작 위생 (오판 차단)
- **"고정 이름이 비어있다"고 가정하지 말 것.** `TaskList`는 세션-간 잔류를 못 본다 → "없음"은 "충돌 없음"을 보장하지 않는다. 신규 스폰은 항상 세션-스코프 이름(규율 1)으로.
- `STATE.md`의 라이브 로스터는 **세션 시작 시 리셋**한다(지난 세션 로스터를 현 세션 걸로 오인 금지). 지난 세션이 남긴 이름은 "잔류(정리 대상)"로만 취급.

### 규율 3 — main 브로커 + File-first가 유일한 신뢰 경로 (P2P 강등)
- 팀원이 다른 팀원의 산출이 필요하면 **파일을 읽고 main에 보고 → main이 릴레이**한다. 팀원→팀원 `SendMessage`는 **best-effort**(잔류로 실패·오도달 가능) — **의존 금지**.
- 팀원은 다른 팀원을 **스폰·가정하지 않는다**. 협업(예: ux↔domain 논의)이 필요하면 main이 양쪽을 스폰하고 파일+릴레이로 수렴시킨다.

### 규율 4 — Phase/세션 종료 teardown (잔류 누적 차단)
- **세션 내:** 재사용은 `SendMessage`, fresh 필요 시 `TaskStop(로스터명)` 후 재스폰. 살아있는 로스터명 위 동명 재스폰 금지(증상 A).
- **Phase 전환:** 다음 Phase 스폰 전 직전 Phase 유휴 팀원 정리(이어서=SendMessage / 폐기=TaskStop).
- **세션 마무리:** main이 **라이브 로스터 전원에 `TaskStop`**. 못 닿으면(이미 죽음) 무해. 핵심 = **유휴 방치 금지** → 다음 세션 레지스트리 청결(증상 B의 잔류 공급 차단).

| 의도 | 도구 | 결과 |
|------|------|------|
| 신규 스폰 | `Agent(name:"역할-<세션/Phase토큰>")` → 반환명 로스터 기록 | 잔류 충돌 0, 유령 분리 |
| 기존(이번 세션) 후속 | `SendMessage(to:"로스터명")` | transcript 재개, 중복 0 |
| 같은 이름 fresh | `TaskStop("로스터명")` → 재스폰 | 이름 회수 후 재스폰 |
| Phase/세션 종료 | 라이브 로스터 전원 `TaskStop` | 잔류 누적 차단 |

**이미 이중/유령이 관측되면:** 이번 세션 로스터에 없는 이름이 활동하면 유령(잔류) — 지시하지 말고, 필요한 역할은 세션-스코프 이름으로 새로 스폰해 로스터로만 라우팅. 유령엔 `TaskStop` 시도(못 닿아도 무해).

## 주의
- `Agent`의 `team_name` 파라미터가 보여도 쓰지 않는다(무시됨). `name`만 사용.
- 세션당 단일 팀이므로 "팀 활성 1개 제한"은 자연히 성립. 단 **Phase/세션 종료 teardown은 필수**(규율 4) — 해체를 안 하면 유휴 팀원이 다음 세션 잔류(유령)로 남는다.
- 이름 레지스트리는 **세션을 넘어 잔류**한다. 고정 역할이름 재사용 금지, 항상 세션-스코프 이름(규율 1). `TaskList`/`TaskStop`은 세션 스코프라 잔류를 못 보고/못 죽인다 — 잔류 판정을 이 도구에 의존하지 말 것.
- `fork` 타입은 자기 자신을 복제(부모 컨텍스트 상속)하며 항상 백그라운드. 전문가 팀원은 `fork`가 아니라 커스텀/빌트인 `subagent_type`으로 스폰한다.
