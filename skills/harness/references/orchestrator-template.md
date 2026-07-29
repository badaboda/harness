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

# 오케스트레이터 스킬 템플릿

오케스트레이터는 팀 전체를 조율하는 상위 스킬이다. 실행 모드별로 4가지 템플릿을 제공한다:

- **템플릿 A: 에이전트 팀 모드 (기본)** — 2명 이상 협업 시 최우선 선택
- **템플릿 B: 서브 에이전트 모드 (대안)** — 팀 통신이 불필요한 경우
- **템플릿 C: 하이브리드 모드** — Phase마다 모드를 섞어 구성
- **템플릿 D: Workflow 모드** — 흐름이 사전 확정된 대량 반복을 스크립트로 고정 (사용자 opt-in 필요)

---

## 템플릿 A: 에이전트 팀 모드 (기본 · 최우선 선택)

2명 이상의 에이전트가 협업할 때 **가장 먼저 검토하는 기본 모드**. 팀원을 named `Agent`로 스폰하고(세션이 단일 암묵 팀), 공유 작업 목록과 `SendMessage`로 조율한다. (`TeamCreate` 폐지 → `references/team-mode-mechanism.md`)

```markdown
---
name: {domain}-orchestrator
description: "{도메인} 에이전트 팀을 조율하는 오케스트레이터. {초기 실행 키워드}. 후속 작업: {도메인} 결과 수정, 부분 재실행, 업데이트, 보완, 다시 실행, 이전 결과 개선 요청 시에도 반드시 이 스킬을 사용."
---

# {Domain} Orchestrator

{도메인}의 에이전트 팀을 조율하여 {최종 산출물}을 생성하는 통합 스킬.

## 실행 모드: 에이전트 팀

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 스킬 | 출력 |
|------|-------------|------|------|------|
| {teammate-1} | {커스텀 또는 빌트인} | {역할} | {skill} | {output-file} |
| {teammate-2} | {커스텀 또는 빌트인} | {역할} | {skill} | {output-file} |
| ... | | | | |

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)

기존 산출물 존재 여부를 확인하여 실행 모드를 결정한다:

1. `_workspace/` 디렉토리 존재 여부 확인
2. 실행 모드 결정:
   - **`_workspace/` 미존재** → 초기 실행. Phase 1로 진행
   - **`_workspace/` 존재 + 사용자가 부분 수정 요청** → 부분 재실행. 해당 에이전트만 재호출하고, 기존 산출물 중 수정 대상만 덮어쓴다
   - **`_workspace/` 존재 + 새 입력 제공** → 새 실행. 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동한 뒤 Phase 1 진행
3. 부분 재실행 시: 이전 산출물 경로를 에이전트 프롬프트에 포함하여, 에이전트가 기존 결과를 읽고 피드백을 반영하도록 지시

### Phase 1: 준비
1. 사용자 입력 분석 — {무엇을 파악하는지}
2. 작업 디렉토리에 `_workspace/` 생성
   - **초기 실행**: 새 `_workspace/` 생성
   - **새 실행**: 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동한 직후 새 `_workspace/` 재생성
3. 입력 데이터를 `_workspace/00_input/`에 저장

### Phase 2: 팀 구성

> 실행 메커니즘: `references/team-mode-mechanism.md`(TeamCreate 폐지 — named `Agent` 스폰). 아래 패턴을 따른다.

1. 공유 작업목록 등록 (+의존성):
   ```
   TaskCreate(subject:"{작업1}", description:"{상세}")            // → T1
   TaskCreate(subject:"{작업2}", description:"{상세}")            // → T2
   TaskUpdate(taskId:"2", addBlockedBy:["1"], owner:"{teammate-2}")  // 의존·소유 지정
   ```

2. 팀원 스폰 (named `Agent`, 세션이 단일 암묵 팀):
   ```
   // 파이프라인 게이트가 있으면 먼저 스폰(예: 계약 담당), 완료 후 팬아웃.
   // 독립 팀원은 한 메시지에 여러 Agent 호출로 동시 스폰:
   Agent(subagent_type:"{type}", name:"{teammate-1}", model:"opus", prompt:"{역할·작업·완료 시 main 통지 지시}")
   Agent(subagent_type:"{type}", name:"{teammate-2}", model:"opus", prompt:"{...}")
   ```
   팀원은 `TaskList`로 claim·`TaskUpdate`로 상태 갱신, `SendMessage({to:name, summary:"...", message:"..."})`로 상호 통신, `SendMessage({to:"main", ...})`로 리더 보고.

   > 팀원당 5~6개 작업이 적정. 의존성은 `TaskUpdate(taskId, addBlockedBy:[...])`로 지정한다(`depends_on` 같은 필드는 없다).

### Phase 3: {주요 작업 — 예: 조사/생성/분석}

**실행 방식:** 팀원들이 자체 조율

팀원들은 공유 작업 목록에서 작업을 요청(claim)하고 독립적으로 수행한다.
리더는 진행 상황을 모니터링하며 필요 시 개입한다.

**팀원 간 통신 규칙:**
- {teammate-1}은 {teammate-2}에게 {어떤 정보}를 SendMessage로 전달
- {teammate-2}는 작업 완료 시 결과를 파일로 저장하고 리더에게 알림
- 팀원이 다른 팀원의 결과가 필요하면 SendMessage로 요청

**산출물 저장:**

| 팀원 | 출력 경로 |
|------|----------|
| {teammate-1} | `_workspace/{phase}_{teammate-1}_{artifact}.md` |
| {teammate-2} | `_workspace/{phase}_{teammate-2}_{artifact}.md` |

**리더 모니터링:**
- 팀원이 유휴 상태가 되면 자동 알림 수신
- 특정 팀원이 막혔을 때 SendMessage로 지시 또는 작업 재할당
- 전체 진행률은 TaskGet으로 확인

### Phase 4: {후속 작업 — 예: 검증/통합}
1. 모든 팀원의 작업 완료 대기 (TaskGet으로 상태 확인)
2. 각 팀원의 산출물을 Read로 수집
3. {통합/검증 로직}
4. 최종 산출물 생성: `{output-path}/{filename}`

### Phase 5: 정리
1. 팀원들의 작업 완료 확인 (TaskList/TaskGet). 필요 시 SendMessage로 마무리 지시 (별도 해체 도구 없음 — 에이전트는 완료 시 종료)
2. `_workspace/` 디렉토리 보존 (중간 산출물은 삭제하지 않음 — 사후 검증·감사 추적용)
3. 사용자에게 결과 요약 보고

> **팀 재구성이 필요한 경우:** Phase별로 다른 전문가 조합이 필요하면, 이전 산출물을 `_workspace/`에 보존해 두고 다음 Phase에 필요한 에이전트를 새 `Agent`로 스폰한다(TeamDelete 불필요). 종료된 에이전트도 이름으로 SendMessage하면 재개된다.

## 데이터 흐름

```
[리더] → Agent 스폰 → [teammate-1] ←SendMessage→ [teammate-2]
                          │                           │
                          ↓                           ↓
                    artifact-1.md              artifact-2.md
                          │                           │
                          └───────── Read ────────────┘
                                     ↓
                              [리더: 통합]
                                     ↓
                              최종 산출물
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 팀원 1명 실패/중지 | 리더가 감지 → SendMessage로 상태 확인 → 재시작 또는 대체 팀원 생성 |
| 팀원 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 타임아웃 | 현재까지 수집된 부분 결과 사용, 미완료 팀원 종료 |
| 팀원 간 데이터 충돌 | 출처 명시 후 병기, 삭제하지 않음 |
| 작업 상태 지연 | 리더가 TaskGet으로 확인 후 수동으로 TaskUpdate |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 {입력}을 제공
2. Phase 1에서 {분석 결과} 도출
3. Phase 2에서 팀 구성 ({N}명 팀원 + {M}개 작업)
4. Phase 3에서 팀원들이 자체 조율하며 작업 수행
5. Phase 4에서 산출물 통합하여 최종 결과 생성
6. Phase 5에서 팀 정리
7. 예상 결과: `{output-path}/{filename}` 생성

### 에러 흐름
1. Phase 3에서 {teammate-2}가 에러로 중지
2. 리더가 유휴 알림 수신
3. SendMessage로 상태 확인 → 재시작 시도
4. 재시작 실패 시 {teammate-2} 작업을 {teammate-1}에게 재할당
5. 나머지 결과로 Phase 4 진행
6. 최종 보고서에 "{teammate-2} 영역 일부 미수집" 명시
```

---

## 템플릿 B: 서브 에이전트 모드 (대안)

팀 통신 오버헤드가 불필요한 경우. `Agent` 도구로 직접 호출하고 반환값으로 결과를 수집한다.

```markdown
---
name: {domain}-orchestrator
description: "{도메인} 에이전트를 조율하는 오케스트레이터. {초기 실행 키워드}. 후속 작업 키워드 포함."
---

## 실행 모드: 서브 에이전트

## 에이전트 구성

| 에이전트 | subagent_type | 역할 | 스킬 | 출력 |
|---------|--------------|------|------|------|
| {agent-1} | {빌트인 또는 커스텀} | {역할} | {skill} | {output-file} |
| {agent-2} | ... | ... | ... | ... |

## 워크플로우

### Phase 0: 컨텍스트 확인
(Template A와 동일 — `_workspace/` 존재 여부 분기)

### Phase 1: 준비
1. 입력 분석
2. `_workspace/` 생성 (초기 실행 시, 또는 새 실행에서 기존 `_workspace/`를 보관 디렉토리로 이동한 직후)

### Phase 2: 병렬 실행
단일 메시지에서 N개 Agent 도구를 동시 호출한다 (이것만으로 병렬 실행된다. 서브에이전트는 기본이 백그라운드이므로 별도 플래그가 필요 없다):

| 에이전트 | 입력 | 출력 | model | isolation |
|---------|------|------|-------|-----------|
| {agent-1} | {소스} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | - |
| {agent-2} | {소스} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | - |

> 여러 에이전트가 **같은 소스 파일을 동시에 수정**하면 `isolation:"worktree"`를 지정해 각자 격리된 git worktree에서 작업시킨다. 읽기 전용이거나 서로 다른 파일만 쓰면 불필요하다.

### Phase 3: 통합
1. 각 에이전트의 반환값 수집
2. 파일 기반 산출물은 Read로 수집
3. 통합 로직 적용 → 최종 산출물

### Phase 4: 정리
1. `_workspace/` 보존
2. 결과 요약 보고

## 에러 핸들링
- 에이전트 1개 실패: 1회 재시도. 재실패 시 누락 명시하고 진행
- 과반 실패: 사용자에게 알리고 진행 여부 확인
- 타임아웃: 현재까지 수집된 부분 결과 사용
```

---

## 템플릿 C: 하이브리드 모드

Phase마다 다른 실행 모드를 사용한다. 각 Phase 상단에 `**실행 모드:** {팀 | 서브}`를 명시한다.

```markdown
---
name: {domain}-orchestrator
description: "{도메인} 오케스트레이터 (하이브리드). {키워드}. 후속 작업 키워드 포함."
---

## 실행 모드: 하이브리드

| Phase | 모드 | 이유 |
|-------|------|------|
| Phase 2 (병렬 수집) | 서브 에이전트 | 독립 자료 수집, 팀 통신 불필요 |
| Phase 3 (합의 통합) | 에이전트 팀 | 상충 데이터 토론·합의 필요 |
| Phase 4 (독립 검증) | 서브 에이전트 | QA 에이전트 1명이 객관 검증 |

## 워크플로우

### Phase 2: 병렬 자료 수집
**실행 모드:** 서브 에이전트

단일 메시지에서 Agent 도구로 N개 에이전트 병렬 호출.
각 결과는 `_workspace/02_{agent}_raw.md`에 저장.

### Phase 3: 합의 기반 통합
**실행 모드:** 에이전트 팀

1. `TaskCreate`로 작업 분배 후 팀원을 named `Agent`로 스폰 (editor + fact-checker + synthesizer) — 모두 Phase 2의 `_workspace/02_*` 파일을 Read
2. 팀원들이 `SendMessage`로 상충 데이터를 논의, 파일 기반으로 합의안 도출
3. 최종 통합본 `_workspace/03_integrated.md` 생성
4. (해체 불필요 — 완료 시 종료)

### Phase 4: 독립 검증
**실행 모드:** 서브 에이전트

단일 QA 서브 에이전트가 `_workspace/03_integrated.md`를 입력으로 받아 검증 보고서 생성.
```

**하이브리드 전환 규칙:** (TeamCreate/TeamDelete 폐지 — `references/team-mode-mechanism.md`)
- 팀 ↔ 서브 구분은 본질적으로 같은 `Agent` 도구다. "팀"은 named 에이전트를 공유 task list로 조율, "서브"는 이름 없이 결과만 수집.
- 서브 → 팀: 서브의 파일 산출물을 새로 스폰하는 named 팀원들에게 Read 경로로 전달
- Phase 전환: 이전 산출물을 `_workspace/`에 보존하고 다음 Phase 에이전트를 새로 스폰 (별도 해체 불필요)

---

## 템플릿 D: Workflow 모드 (결정적 오케스트레이션)

에이전트 간 **협업**이 아니라 동일 처리의 **대량 반복**이고, 단계 순서·검증이 사전에 확정된 경우. 모델 판단이 아니라 스크립트가 흐름을 고정한다.

> 🚫 **`Workflow` 도구는 사용자 opt-in 없이 호출할 수 없다.** 이 템플릿으로 오케스트레이터를 만들 때는 **본문에 "이 스킬 호출이 곧 Workflow 사용 근거"임을 명시**해야 실행 시점에 막히지 않는다.

**설계 전에 알아야 할 런타임 사실 (공식 문서):**

| 항목 | 사실 |
|------|------|
| opt-in 경로 | `ultracode` 키워드 또는 "워크플로우로 해줘" 같은 자연어 요청. **사용자가 직접 타이핑한 프롬프트에서만** 유효하다 — `-p` 전달, 스케줄 작업, 웹훅/PR 코멘트 경유 입력에서는 트리거되지 않는다 |
| 실행 승인 | 매 실행마다 phase 목록을 보여주는 승인 프롬프트(권한 모드에 따라 다름). bypass/`-p`/SDK에서는 즉시 시작 |
| **에이전트 권한** | ⚠️ 워크플로우가 스폰하는 서브에이전트는 **세션 모드와 무관하게 항상 `acceptEdits`로 실행되고 파일 편집이 자동 승인**된다. 파괴적 편집을 포함하는 워크플로우는 이 점을 오케스트레이터에 경고로 남긴다 |
| 한도 | **동시 16개**(CPU에 따라 더 적음), **런당 총 1,000개**. 25개 초과 또는 예상 150만 토큰 초과 시 `Large workflow` 경고 |
| 크기 가이드 | 기본 `medium`(15개 미만). `/config`의 Dynamic workflow size 또는 `workflowSizeGuideline` 설정으로 조정 — **권고이지 강제 상한이 아니다** |
| 모델 | 각 에이전트는 **세션 모델**을 쓴다(스크립트가 단계별로 지정하지 않는 한). `CLAUDE_CODE_SUBAGENT_MODEL`이 설정돼 있으면 **그것이 둘 다 덮어쓴다** — opus를 전제한 하네스라면 큰 실행 전에 `/model`과 이 변수를 확인 |
| 재개 | **같은 세션 안에서만** 가능. 캐시는 **에이전트 시작 순서**를 따라 replay되므로, 중간에 멈추면 그 뒤에 시작된 에이전트는 완료됐어도 재실행된다 → **긴 단일 에이전트보다 잘게 쪼갠 팬아웃이 진행을 더 보존한다** |
| 저장·배포 | `/workflows`에서 `s`로 저장 → `.claude/workflows/`(프로젝트) 또는 `~/.claude/workflows/`(개인). 이후 `/<name>`으로 실행. **플러그인 배포는 플러그인 루트의 `workflows/` 디렉토리**, `/plugin:name`으로 네임스페이스됨 |
| 비활성화 | `disableWorkflows` 설정 또는 `CLAUDE_CODE_DISABLE_WORKFLOWS=1`. 꺼져 있으면 `ultracode` 키워드도 동작하지 않는다 |

````markdown
---
name: {domain}-orchestrator
description: "{도메인} 오케스트레이터 (Workflow 기반). {키워드}. 후속 작업 키워드 포함."
---

## 실행 모드: Workflow

이 스킬은 `Workflow` 도구 사용을 전제한다 — 이 스킬이 호출된 것 자체가 멀티에이전트 오케스트레이션에 대한 사용자 opt-in이다.

## 워크플로우

### Phase 0-1: 컨텍스트 확인 · 준비
(템플릿 A와 동일 — `_workspace/` 분기, 작업 목록 산출)

### Phase 2: 스크립트 구성

메인이 먼저 **인라인으로 정찰**하여 처리 대상 목록을 확정한 뒤, 그 목록을 `args`로 넘긴다.
스크립트는 `export const meta = {...}` 리터럴로 시작하고, `meta.phases`의 제목과 `phase()` 호출 제목을 일치시킨다.

```js
export const meta = {
  name: '{domain}-sweep',
  description: '{한 줄 설명}',
  phases: [{ title: 'Process' }, { title: 'Verify' }],
}
const results = await pipeline(
  args,                                                    // 정찰로 확정한 대상 목록
  item => agent(`{처리 지시}: ${item}`, {phase:'Process', schema: RESULT_SCHEMA}),
  (res, item) => agent(`{검증 지시}: ${item}`, {phase:'Verify', schema: VERDICT_SCHEMA})
                   .then(v => ({...res, verdict: v}))
)
return results.filter(Boolean).filter(r => r.verdict?.ok)
```

**설계 규칙:**
- **기본은 `pipeline()`** — 항목이 단계를 독립 통과해 배리어 대기가 없다. 전역 dedup·0건 조기종료처럼 *모든* 이전 결과가 동시에 필요한 자리에서만 `parallel()`을 쓴다
- **`schema`로 반환을 구조화** — 파싱 불필요, 형식 일탈 시 모델이 재시도한다
- 실패한 항목은 `null`이 되므로 **`.filter(Boolean)` 후 사용**
- 파일을 병렬 수정하면 `{isolation:'worktree'}`
- 스크립트에서 `Date.now()`/`Math.random()`은 쓸 수 없다 — 타임스탬프는 `args`로 주입하고, 결과 스탬프는 워크플로우 반환 후에 찍는다
- 커버리지를 잘라냈으면(top-N, 샘플링) `log()`로 무엇을 뺐는지 남긴다 — 조용한 절단은 "전부 처리함"으로 읽힌다

### Phase 3: 결과 종합
1. 워크플로우 반환값을 메인이 수집
2. 산출물은 `_workspace/`에 저장, 최종본만 사용자 경로로 출력
3. 다단계 작업이면 **결과를 읽고 다음 워크플로우를 결정** — 한 번에 다 넣지 말고 Phase 단위로 끊는다

## 에러 핸들링
- 단계가 throw하면 해당 항목만 `null`로 떨어지고 나머지는 계속 진행 — 최종 보고서에 누락 항목 명시
- 스크립트 수정 후에는 `{scriptPath, resumeFromRunId}`로 재개 (변경되지 않은 앞부분은 캐시에서 즉시 반환)
- 결과가 비었으면 추측하지 말고 transcript 디렉토리의 `journal.jsonl`에서 각 에이전트의 실제 반환값을 확인
````

---

## 작성 원칙

1. **실행 모드를 먼저 명시** — 오케스트레이터 상단에 "에이전트 팀" / "서브 에이전트" / "Workflow" / "하이브리드" 중 하나 명시. 하이브리드면 Phase별 모드 표 필수
2. **팀 모드는 named `Agent` 스폰 + SendMessage + TaskCreate 사용법을 구체적으로** — 작업목록·의존성(`addBlockedBy`), 팀원 스폰, 통신 규칙. 스폰 전 `name` 지원 여부 확인까지 포함 (`references/team-mode-mechanism.md`)
3. **서브 모드는 Agent 도구 파라미터를 실재하는 것만 명시** — `subagent_type`, `prompt`, `model`, (필요 시) `isolation`. 무시되는 `team_name`은 쓰지 않고, 병렬을 `run_in_background`로 표기하지 않는다(백그라운드가 기본값). **백그라운드 서브에이전트에는 `TaskCreate`/`TaskList`가 없으므로** 공유 작업목록 참여를 요구하지 않는다
4. **Workflow 모드는 opt-in 근거를 본문에 명시** — 없으면 실행 시점에 도구를 쓸 수 없다
4. **파일 경로는 절대적으로** — 상대 경로 금지, `_workspace/` 기준 명확한 경로
5. **Phase 간 의존성 명시** — 어떤 Phase가 어떤 Phase의 결과에 의존하는지. 하이브리드는 모드 전환 지점을 특히 강조
6. **에러 핸들링은 현실적으로** — "모든 것이 성공한다"고 가정하지 않음
7. **테스트 시나리오 필수** — 정상 1 + 에러 1 이상

## description 작성 시 후속 작업 키워드

오케스트레이터 description은 초기 실행 키워드만으로는 부족하다. 다음 후속 작업 표현을 반드시 포함하라:

- 재실행/다시 실행/업데이트/수정/보완
- "{도메인}의 {부분}만 다시"
- "이전 결과 기반으로", "결과 개선"
- 도메인 관련 일상적 요청 (예: 런치 전략 하네스라면 "런치", "홍보", "트렌딩" 등)

후속 키워드가 없으면 첫 실행 후 하네스가 사실상 죽은 코드가 된다.

## 실제 오케스트레이터 참고

팬아웃/팬인 패턴의 오케스트레이터 기본 구조:
준비 → Phase 0(컨텍스트 확인) → TaskCreate(작업목록) + named `Agent` 스폰 → N개 팀원 병렬 실행 → Read + 통합 → 정리.
`references/team-examples.md`의 리서치 팀 예시를 참조.
