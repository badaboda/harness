# Changelog

이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

## [1.4.0] - 2026-07-29

Claude Code 2.1.x 도구 스키마 + **공식 문서**([agent-teams](https://code.claude.com/docs/en/agent-teams), [sub-agents](https://code.claude.com/docs/en/sub-agents), [skills](https://code.claude.com/docs/en/skills)) 전수 대조. 낡은 표기 정정 + 미반영 런타임 제약 반영.

### Fixed

- **`depends_on` → `TaskUpdate(addBlockedBy/addBlocks)`.** 존재하지 않는 필드였고, 같은 문서 내 다른 예시와도 모순이었다. (orchestrator-template.md)
- **`SendMessage`의 필수 `summary`(5~10단어) 누락 정정.** 전 예시가 `{to, message}`만 넘기고 있었다.
- **"병렬 실행 = `run_in_background: true`" 표기 정정.** v2.1.198부터 **서브에이전트는 기본이 백그라운드**이고 Claude가 결과를 바로 써야 할 때만 포그라운드로 돈다. 병렬은 **한 메시지 다중 `Agent` 호출**로 성립하므로 플래그 표기가 불필요하다. 항상 백그라운드로 고정할 역할은 정의 frontmatter `background: true`. (SKILL.md, orchestrator-template.md, agent-design-patterns.md, team-mode-mechanism.md)
- **`model` 기본값이 `inherit`임을 반영.** "모든 Agent 호출에 `model:"opus"` 명시"에서 **정의 frontmatter에 `model: opus`를 명시**하는 방식으로 전환. 명시하지 않으면 세션 모델을 상속하므로 opus가 보장되지 않는다. 허용 값은 `sonnet`·`opus`·`haiku`·`fable`·모델 ID·`inherit`.
- **`Skill` 도구 호출 표기** — `/skill-name`(슬래시) → 슬래시 없는 정확한 이름, 플러그인은 `plugin:skill`. (agent-design-patterns.md)

### Added

- **🔴 `references/team-mode-mechanism.md` §전제조건 — 에이전트 팀은 실험 기능이며 기본값이 꺼져 있다.** `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`이 없으면 **세션 시작 시 팀이 구성되지 않고 Claude가 팀원을 스폰·제안하지도 않는다.** 하네스가 팀 모드를 최우선 기본값으로 삼아 왔는데 이 전제를 한 번도 명시한 적이 없었다 — 가장 큰 공백.
  - 끈 채로 팀을 시도하면 **조용히 서브에이전트로 대체**되고, 문서 표현대로 *"에이전트 패널만으로는 팀이 형성됐는지 확인되지 않는다."* v1.3.0이 경고한 1계층 위반과 동일한 실패 모드.
  - 플래그 확인 → 꺼져 있으면 **사용자에게 알리고 결정받기**(하네스가 임의로 설정을 바꾸지 않는다) → 서브/Workflow 모드로 설계, 를 절차로 명문화. SKILL.md 2-1·6-2·체크리스트, agent-design-patterns 의사결정 트리, team-examples 경고에 모두 노출.
- **백그라운드 서브에이전트의 도구 축소 제약.** 백그라운드 서브에이전트는 빌트인 도구가 축소되어 **`TaskCreate`/`TaskUpdate`/`TaskList`가 목록에 들어오지 않는다** — 즉 공유 작업목록에 참여할 수 없다. 서브 모드 오케스트레이터가 서브에이전트에게 태스크 claim을 시키는 설계는 동작하지 않는다. (팀 *팀원*은 예외: 조율 도구가 `tools` 제한과 무관하게 항상 제공된다. `fork`는 두 필터를 모두 건너뛴다.)
  - **폐기가 아니라 실행 맥락별 필터임을 명시.** Task 계열은 오히려 현행 표준이다 — `TodoWrite`가 v2.1.142부터 기본 비활성이 되며 `TaskCreate`/`TaskGet`/`TaskList`/`TaskUpdate`가 그 자리를 대체했다. Task 계열 중 실제 deprecated는 **`TaskOutput`** 하나(출력 파일을 `Read`하는 방식으로 대체).
  - 모든 서브에이전트에서 제거되는 필터 1 목록도 기록: `AskUserQuestion`·`EnterPlanMode`·`ExitPlanMode`·`EndConversation`·`ScheduleWakeup`·`TaskOutput`·`WaitForMcpServers`·`Workflow`. **서브에이전트는 `Workflow`를 호출할 수 없다** — 워크플로우는 메인만 시작할 수 있다는 제약이 템플릿 D 설계의 전제가 된다.
- **스폰 한도** — 기본 **세션당 서브에이전트 200개**(+ 동시 실행 제한, 중첩 깊이 제한). 대규모 팬아웃 설계 시 계산에 넣도록 팀 크기 가이드라인에 반영.
- **팀원 재사용 시의 frontmatter 예외** — 서브에이전트 정의를 팀원 역할로 쓸 때 `tools`·`model`은 적용되고 본문은 시스템 프롬프트에 **덧붙지만(대체 아님)**, **`skills`·`mcpServers`는 적용되지 않는다.**
- **`SendMessage` 이름 충돌 검사(v2.1.199+)를 스폰 규율에 편입** — 더 새 에이전트가 이름을 가져가면 **오배달 대신 거부**하고 에러가 현재 소유자를 알려준다. 이때 **스폰 시 받은 `agentId`로 주소를 바꿔** 원래 에이전트에 닿는다. 로스터에 이름과 `agentId`를 함께 적도록 규율 2 보강. 완료·정지된 에이전트가 `SendMessage`로 자동 재개된다는 문서 근거도 규율 3에 추가.
- **에이전트 frontmatter 전체 필드표** (agent-design-patterns.md) — `tools`/`disallowedTools`·`model`·`effort`·`permissionMode`·`maxTurns`·`skills`·`background`·`isolation`. `name`에 `:`를 쓰면 파일이 로드되지 않는다는 제약 포함.
- **빌트인 타입 정정** — `Explore`는 세션 모델을 상속하되 **Opus 상한**이고 `Plan`과 함께 CLAUDE.md·git status를 건너뛴다. `claude`·`fork` 추가. 동명 정의로 오버라이드하거나 `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS=1`로 제거할 수 있음을 명시.
- **스킬 frontmatter 전체 필드표** (skill-writing-guide.md §0) — **모든 필드가 선택이고 `description`만 권장**(`name`조차 필수가 아니며 생략 시 디렉토리명). `when_to_use`, `allowed-tools`/`disallowed-tools`, `disable-model-invocation`, `user-invocable`, `model`/`effort`, `context: fork`+`agent`, `paths` 등. **`description`+`when_to_use`가 1,536자에서 잘린다**는 실무 제약과 호출 이름 결정 규칙표 포함.
  - "커맨드를 만들지 않는 이유"를 **"커스텀 커맨드가 스킬로 통합됐다"**는 공식 근거로 갱신.
- **Workflow 모드를 제4의 실행 모드로 신설.** 협업이 아니라 **동일 처리의 대량 반복**이고 흐름이 사전 확정일 때 `Workflow` 도구로 순서·병렬·검증을 스크립트로 고정한다. `pipeline()`이 기본이고 `parallel()`은 전역 집계 배리어에만, `schema`로 단계 반환을 구조화, `resumeFromRunId`로 재개.
  - **opt-in 게이팅 명문화:** 사용자의 명시적 opt-in 없이 호출할 수 없으므로, 하네스가 Workflow 오케스트레이터를 산출할 때는 **그 스킬 본문에 "이 스킬 호출이 곧 opt-in 근거"임을 명시**해야 실행 시점에 막히지 않는다.
  - `references/orchestrator-template.md`에 **템플릿 D: Workflow 모드** 추가 (스크립트 골격, 설계 규칙, 에러/재개 절차).
- **`isolation: "worktree"`** — 여러 에이전트가 같은 파일을 병렬 수정하는 팬아웃에서 서로의 편집을 덮어쓰는 문제. 각자 격리된 git worktree에서 작업 후 메인이 병합한다. 셋업 비용이 있으므로 실제 쓰기 충돌이 나는 경우로 한정. (SKILL.md 2-1, 팬아웃 패턴, orchestrator-template.md, team-mode-mechanism.md)
- **중첩 서브에이전트 깊이 3 (v2.1.219)** — 서브에이전트가 자신의 서브에이전트를 스폰할 수 있게 됐다(이전 기본값 1 = 중첩 불가). `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1`로 비활성 가능하며 중첩분도 세션 200개 한도에 포함. **에이전트 팀은 여전히 중첩 불가**이므로 "계층적 위임" 패턴 서술을 둘로 분리했다. 단, v2.1.203이 "에이전트가 작업 전체를 재위임하는 경향"을 억제하는 방향이므로 **2단계 이내 권장은 유지**.
- **Workflow 런타임 사실표** (orchestrator-template.md 템플릿 D) — opt-in 경로(사용자가 직접 타이핑한 프롬프트에서만 유효, `-p`·스케줄·웹훅 경유는 미적용), 실행 승인 흐름, **동시 16개/런당 1,000개** 한도, 기본 크기 가이드 `medium`(15개 미만), 세션 모델 사용 및 `CLAUDE_CODE_SUBAGENT_MODEL`의 우선 적용, 같은 세션 내 재개 및 **시작 순서 기반 replay 규칙**(중간에 멈추면 그 뒤 시작분은 완료됐어도 재실행 → 잘게 쪼갠 팬아웃이 진행을 더 보존), 저장 위치와 **플러그인 `workflows/` 배포**, 비활성화 방법.
  - ⚠️ **워크플로우 에이전트는 세션 권한 모드와 무관하게 항상 `acceptEdits`로 실행되고 파일 편집이 자동 승인된다.** 파괴적 변경을 포함하는 워크플로우 설계 시 경고를 남기도록 SKILL.md에도 노출.
- **CLAUDE.md 권장 크기(200줄 미만)와 `.claude/rules/` 경로 스코프 규칙** — Phase 5-4가 CLAUDE.md만 알고 있었다. 최소 포인터 원칙의 근거를 공식 권장으로 교체하고, 파일 유형별 규약은 `paths:` 글롭 규칙으로 보내도록 안내 추가.

### Changed

- **`team_name` 서술 유지** — 검토 중 "파라미터 자체가 삭제됨"으로 고쳤다가, 문서가 *"accepted but ignored"* 로 명시하고 있어 **"입력은 받지만 무시됨(deprecated)"** 로 되돌렸다. 훅 페이로드의 `team_name`도 세션 파생 이름을 실어 나를 뿐이다.
- **팀 해체 서술 정정** — "팀을 해체하고 새 팀을 생성"이라는 `TeamCreate` 시절 표현을 제거. 세션당 팀은 하나이고 **종료 시 자동 정리**되며, 공유 작업목록 디렉토리는 남아 재개 세션이 이어받는다. Phase 재구성은 유휴 팀원 `TaskStop` 후 새 스폰.
- **병렬 쓰기 충돌 대응 순서 정정** — 문서의 1차 권고가 **"파일 소유권을 팀원별로 분할"** 이므로 이를 앞에 두고, `isolation: worktree`를 그다음 수단으로. worktree는 **기본 브랜치에서 분기**하며(부모 `HEAD` 아님) 변경이 없으면 자동 정리된다는 점 명시.
- **팀 크기 가이드라인 보강** — 공식 권장 **3~5명 시작 / 팀원당 5~6개 작업**, 토큰 비용이 팀원 수에 선형 증가.
- 모드 선택 의사결정 트리에 실험 플래그 분기와 Workflow 분기 추가.
- Phase 6-2 실행 모드별 검증에 Workflow 항목 + 팀 플래그 확인 + 백그라운드 태스크 도구 확인 추가. 산출물 체크리스트도 동일하게 갱신.
- SKILL.md가 자체 500줄 기준을 넘겨 Workflow 상세를 템플릿 D로, frontmatter 필드표를 agent-design-patterns로 이관 (본문 기준 500줄 이내 복귀).

## [1.3.0] - 2026-07-29

### Added

- **`references/team-mode-mechanism.md` §최우선 안티패턴 — `*-team-lead` 서브에이전트에 오케스트레이션 위임 금지.** 팀은 **1계층**이며 이름 있는 팀메이트(`Agent(name:...)`)는 **메인 대화만** 생성할 수 있다는 런타임 제약을 정본화.
  - lead 를 서브에이전트로 스폰해 팀 소집을 위임하면 그 아래는 전부 **무명 서브에이전트로 폴백** → `SendMessage(to:name)` 재개 불가 · 공유 `TaskCreate` 목록 미참여 · 호출마다 풀 컨텍스트 재전달. 실행은 계속되므로 팀 모드가 **조용히 사라지는** 것을 알아채기 어렵다.
  - **올바른 형태:** 오케스트레이터는 **스킬**이고, 그 스킬을 읽는 **메인 대화가 리더**다. 메인이 `TaskCreate` 로 공유 목록을 만들고 전문가를 직접 `Agent(name:...)` 로 스폰한다.
  - 기존 프로젝트에 `*-team-lead` 정의가 쌓여 있어도 근거가 되지 않음을 명시(관행이 아니라 이 문서가 정본). 단, lead 를 *조언자*(설계 리뷰·계획 수립)로 쓰는 것은 예외로 허용 — 금지 대상은 **팀 소집 권한의 위임**.

### Changed

- SKILL.md 팀 패턴 섹션에 위 1계층 제약 경고 노출 + `references/team-mode-mechanism.md` 포인터.
- SKILL.md 산출물 체크리스트에 **"`*-team-lead` 오케스트레이터 에이전트를 만들지 않았음"** 항목 추가.

## [1.2.1] - 2026-07-22

### Changed

- **`references/team-mode-mechanism.md` §스폰 규율 — 실측 기반 근본원인 재규명 및 규율 재정렬.** `TaskStop` 조회로 정지된 중복 3개(`data-eng`·`backend-eng`·`architect-2`)가 전부 *같은 세션·이전 Phase 프롬프트*였음이 확인되어(2026-07-22 실측), 근본원인을 "세션 간 유령"에서 **"Phase 전환 시 이전 Phase 팀원 미-teardown → 동명 충돌"**로 정정.
  - **규율 우선순위 재정렬:** 규율 1 = **Phase/세션 teardown(근본 해법·필수)**, 규율 2 = 세션-스코프 이름(충돌 방벽·보조로 강등), 규율 3 = 세션 내 재사용/재시작, 규율 4 = main 브로커·File-first.
  - **`TaskStop` 프로브 기법 추가:** 존재하지 않는 id로 `TaskStop` 호출 시 반환되는 running-teammates 목록으로 실 로스터·잔류를 확인·정지. `TaskList`는 잔류를 못 보여 오판을 유발하므로 잔류 판정은 `TaskStop` 프로브로.

## [1.2.0] - 2026-07-22

### Changed

- **`references/team-mode-mechanism.md` §스폰 규율 확장 — 세션 간 "유령(ghost) 에이전트" 방지 추가.** 기존 세션 내 중복(`-2`) 방지에 더해, 이름 레지스트리가 세션을 넘어 잔류하는 반면 `TaskList`/`TaskStop`은 세션 스코프라 어긋나는 문제(증상 B)를 다룬다. 4개 규율 체계로 재구성:
  - **규율 1 (세션-스코프 이름):** 고정 역할이름 재사용 금지 → `Agent(name:"data-eng-p3")`처럼 세션/Phase 유니크 토큰을 붙여 잔류 충돌·유령을 원천 분리, 반환명을 정본으로 라이브 로스터에 기록.
  - **규율 2 (세션-시작 위생):** "고정 이름이 비어있다" 가정 금지, 로스터는 세션 시작 시 리셋.
  - **규율 3 (main 브로커 + File-first):** 팀원→팀원 P2P `SendMessage`는 best-effort로 강등, main 릴레이가 유일한 신뢰 경로.
  - **규율 4 (Phase/세션 종료 teardown):** 라이브 로스터 전원 `TaskStop`으로 잔류 누적 차단.

## [1.1.0] - 2026-07-22

### Changed

- **팀 실행 방식 갱신 (Claude Code 2026-07+)** — `TeamCreate`/`TeamDelete` 폐지 반영. 팀원은 named `Agent` 스폰(세션이 단일 암묵 팀) + `SendMessage` + 공유 `TaskCreate`로 조율하는 현행 방식으로 전면 전환 (SKILL.md, orchestrator-template.md, agent-design-patterns.md, team-examples.md).
- `team-examples.md` 다이어그램의 폐지 API(`TeamCreate(...)`) 예시를 전부 현행 `Agent(name:...)` 스폰 표기로 치환.

### Added

- `references/team-mode-mechanism.md` — 현행 팀 실행 메커니즘의 단일 진실 소스(SSOT). 이전↔현재 마이그레이션 표, 4원칙, 코드 패턴, **§스폰 규율(중복 에이전트 `-2` 누적 방지: 재사용=`SendMessage` / fresh=`TaskStop`→재스폰)** 포함.
- SKILL.md 팀 패턴 섹션에 **중복 스폰 금지** 안내 + §스폰 규율 포인터 노출.
- 전 소스파일(.md)에 Apache License 2.0 헤더 주석 + 루트 `NOTICE` 파일 (SKILL.md는 스킬 로더 frontmatter 보존 위해 frontmatter 직후 삽입).

## [1.0.1] - 2026-03-28

### Changed

- SKILL.md ↔ references 간 중복 내용 제거 (330줄 → 285줄)
  - Phase 2-1: 실행 모드 비교표/불릿 → 핵심 원칙 + agent-design-patterns.md 포인터
  - Phase 2-3: 에이전트 분리 기준 불릿 → 4축 요약 + agent-design-patterns.md 포인터
  - Phase 3: 에이전트 정의 템플릿 코드블록 → 필수 섹션 나열 + references 포인터
  - Phase 5-2: 에러 핸들링 5행 테이블 → 핵심 원칙 + orchestrator-template.md 포인터

## [1.0.0] - 2026-03-27

### Added

- 6 Phase 워크플로우 기반 하네스 구성 메타 스킬
- 6가지 에이전트 아키텍처 패턴 (파이프라인, 팬아웃/팬인, 전문가 풀, 생성-검증, 감독자, 계층적 위임)
- 에이전트 팀 / 서브 에이전트 실행 모드 지원
- Progressive Disclosure 기반 스킬 생성 가이드
- 오케스트레이터 템플릿 (에이전트 팀 모드 + 서브 에이전트 모드)
- QA 에이전트 통합 가이드 (실제 프로젝트 7개 버그 사례 기반)
- 스킬 테스트/평가 방법론 (With-skill vs Without-skill 비교)
- 실전 팀 구성 예시 5종 (리서치, 소설, 웹툰, 코드리뷰, 마이그레이션)
