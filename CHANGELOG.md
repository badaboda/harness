# Changelog

이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

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
