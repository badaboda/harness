# Changelog

이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

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
