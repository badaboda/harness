# Changelog

이 프로젝트는 [Semantic Versioning](https://semver.org/)을 따릅니다.

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
