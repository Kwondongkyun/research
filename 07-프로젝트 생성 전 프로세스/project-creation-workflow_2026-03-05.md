# 프로젝트 생성 전 과정

> 최종 수정: 2026-03-05

## TL;DR

브레인스토밍부터 병렬 개발, 검증까지 Claude Code로 프로젝트를 만드는 전체 8단계 프로세스를 정리했다. 각 Phase별 사용 스킬, 에이전트, MCP를 매핑.

## 목차

1. 전체 흐름
2. Phase 0~8 상세
3. 사용하는 도구

---

## 1. 전체 흐름

```
브레인스토밍 → 기획 → 기획검증 → UI설계 → 구현계획 → Foundation → 병렬개발 → 검증(반복) → 기록
brainstorming   PM   spec-review  Pencil  writing-plans  Lead        Team      Review+Test   Log
```

---

### Phase 0: 브레인스토밍

```
"XX 만들고 싶어"
    ↓
/brainstorming 실행
    ↓
1. 한 번에 하나씩 질문 (목적, 제약, 성공 기준)
2. 2-3가지 접근 방식 제안 + 트레이드오프
3. 섹션별 설계 → 사용자 승인
4. 설계 문서 저장 → docs/plans/YYYY-MM-DD-<topic>-design.md
    ↓
방향 확정 → PM 기획으로
```

### Phase 1: 기획

```
확정된 방향 기반
    ↓
pm 에이전트 + pm skills
    ↓
docs/plans/
├── prd.md                      ← 기능 요구사항 정의
├── user-flow.md                ← 사용자 시나리오/플로우
├── information-architecture.md ← 사이트맵, 정보 구조
├── error-scenarios.md          ← 에러/엣지케이스 처리
└── design.md                   ← 기술 스택, 디자인 시스템
```

### Phase 2: 기획 검증

```
docs/plans/ 문서 완성
    ↓
/spec-review 실행
    ↓
인터뷰 방식으로 이슈 발견
  - PRD에 빠진 요구사항?
  - 유저 플로우에 빠진 엣지케이스?
  - IA 구조에 모순?
  - 에러 시나리오 누락?
    ↓
이슈 해결 → 문서 확정
```

### Phase 3: UI 설계

```
확정된 기획 문서 기반
    ↓
Pencil MCP로 페이지별 UI 목업
    ↓
시각적으로 확정된 화면 → 구현 계획으로
```

### Phase 4: 구현 계획

```
확정된 기획 + UI 목업 기반
    ↓
writing-plans 실행
    ↓
implementation.md ← 전체 Task 목록 (2-5분 단위, 파일 경로, 테스트 포함)
    ↓
에이전트 팀 구성

Lead (Opus) ─── Foundation + 마무리
  ├── dev-A (Sonnet) ─── 기능 그룹 A
  ├── dev-B (Sonnet) ─── 기능 그룹 B
  ├── dev-C (Sonnet) ─── 기능 그룹 C
  └── dev-D (Sonnet) ─── 기능 그룹 D
```

### Phase 5: Foundation (Lead 직접 수행)

```
프로젝트 초기 설정 (프레임워크, 패키지)
디자인 시스템 (색상, 타이포, 테마)
공통 레이아웃 (헤더, 푸터, 네비게이션)
DB/API 연동 (스키마, 클라이언트, 타입)
인증/스토리지 설정
    ↓
커밋 → 병렬 개발 시작 가능
```

### Phase 6: 병렬 개발

```
dev-A ──→ 기능 그룹 A
dev-B ──→ 기능 그룹 B    각자 worktree에서 동시 진행
dev-C ──→ 기능 그룹 C
dev-D ──→ 기능 그룹 D
    ↓
각자 커밋 → 머지
```

### Phase 7: 검증 (Closed-loop)

```
frontend-reviewer ──→ 코드 리뷰 (버그, 보안, 성능, 유지보수성)
frontend-test     ──→ 테스트 작성/실행 (단위, 컴포넌트, E2E)
Playwright MCP    ──→ 실제 브라우저로 탐색적 테스트 (개발 중 빠른 확인, 스크린샷)
    ↓
Critical > 0 → 피드백 → dev 수정 → 다시 검증 (반복)
Critical = 0 → 완료
```

### Phase 8: 기록

```
docs/conversations/
├── YYYY-MM-DD-<마일스톤1>.md   ← "여기까지 완료, 다음은 이것"
└── YYYY-MM-DD-<마일스톤2>.md   ← "전체 완료, 남은 이슈"
```

---

## 3. 사용하는 도구

### Skills

| Phase | 스킬 | 역할 |
|-------|------|------|
| 0 | brainstorming | 방향 탐색, 접근방식 제안 |
| 4 | writing-plans | 기획 문서 → 구현 Task 쪼개기 (2-5분 단위) |
| 1 | pm-requirements | PRD 작성 |
| 1 | pm-user-flow | 유저 플로우 정의 |
| 1 | pm-information-architecture | 사이트맵/IA 설계 |
| 1 | pm-error-scenarios | 에러 시나리오 정의 |
| 2 | spec-review | 기획 문서 검증 |
| 7 | frontend-review-* | 코드 리뷰 (버그, 보안, 성능, 유지보수성) |
| 7 | frontend-*-test | 테스트 (단위, 컴포넌트, API 모킹, E2E) |

### Agents

| Agent | 역할 |
|-------|------|
| pm | 기획 문서 작성 |
| frontend | 프론트엔드 개발 |
| frontend-reviewer | 코드 리뷰 |
| frontend-test | 테스트 작성/실행 |

### MCP

| MCP | 역할 |
|-----|------|
| Pencil | UI 목업 설계 |
| Playwright | 브라우저 직접 제어 (탐색적 테스트, 스크린샷) |

---

## 레퍼런스

- [Claude Code Agents](https://docs.anthropic.com/en/docs/claude-code/agents)
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Claude Code Hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
