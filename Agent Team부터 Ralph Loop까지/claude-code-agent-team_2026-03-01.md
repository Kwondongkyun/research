# Agent Team부터 Ralph Loop까지: AI 에이전트 팀 조합 실험기

> 최종 수정: 2026-03-01

## TL;DR

Agent Team과 Ralph Loop을 조합하는 4가지 방식(1회성/반복 × 같은 브랜치/Worktree)을 정리했다. 실험 결과 반복 루프가 있는 조합만 Critical 0을 달성했다. 기본값은 C. Ralph Loop + Team.

## 목차

1. Agent Team이란?
2. 개발도 여러 명이서 병렬로 가능할까?
3. 4가지 조합 소개
4. 조합 A: Agent Team — 1회성 병렬 협업
5. 조합 B: Agent Team + Worktree — 에이전트별 브랜치 분리
6. 조합 C: Ralph Loop + Team — 반복 + 팀 협업
7. 조합 D: Ralph Loop 병렬 — 서브에이전트 반복
8. 4가지 조합 비교표
9. PROMPT 설계가 핵심이다

---

## 1. Agent Team이란?

Claude Code에서 여러 서브에이전트를 **팀으로 묶어서** 각자 독립적인 작업을 **병렬로** 수행시키는 기능이다.

### 작동 방식

```
TeamCreate (팀 생성)
    ├── Agent (에이전트 A 생성) → 작업 1
    ├── Agent (에이전트 B 생성) → 작업 2
    └── Agent (에이전트 C 생성) → 작업 3
         │
    TaskList / TaskUpdate (작업 조율)
         │
    SendMessage (에이전트 간 통신)
```

### Agent Team 생성 및 운영 명령어

#### Step 1. 팀 생성

```
TeamCreate("todo-app-team")
```

- 팀 설정 파일 생성: `~/.claude/teams/{team-name}/config.json`
- 공유 태스크 디렉토리 생성: `~/.claude/tasks/{team-name}/`

#### Step 2. 태스크 생성 및 의존성 설정

```
TaskCreate("Todo 타입 + 훅 구현")        → Task #1
TaskCreate("Todo UI 컴포넌트 구현")       → Task #2
TaskCreate("Todo 메인 페이지 구현")       → Task #3

TaskUpdate(taskId="3", addBlockedBy=["1", "2"])  → #3은 #1, #2 완료 후 시작
```

#### Step 3. 에이전트 스폰 (병렬 실행)

```
Agent(subagent_type="frontend", name="frontend-a", team_name="todo-app-team")
Agent(subagent_type="frontend", name="frontend-b", team_name="todo-app-team")
```

- `subagent_type`: 사용할 에이전트 종류 (`.claude/agents/`에 정의된 이름)
- `name`: 팀 내에서 부를 이름
- `team_name`: 소속 팀
- `run_in_background: true`: 백그라운드 실행 (병렬 작업 시 필수)

#### Step 4. 에이전트 간 소통

```
SendMessage(type="message", recipient="frontend-a", content="공통 타입 먼저 만들어줘")
SendMessage(type="broadcast", content="전체 공지: API 스펙 변경됨")  // 전체 메시지 (비용 큼, 신중하게)
```

#### Step 5. 태스크 상태 관리

```
TaskList()                                    → 전체 태스크 현황 확인
TaskGet(taskId="1")                           → 특정 태스크 상세 조회
TaskUpdate(taskId="1", status="in_progress")  → 작업 시작
TaskUpdate(taskId="1", status="completed")    → 작업 완료
```

#### Step 6. 팀 종료 및 정리

```
SendMessage(type="shutdown_request", recipient="frontend-a")  → 에이전트 종료 요청
SendMessage(type="shutdown_request", recipient="frontend-b")  → 에이전트 종료 요청

TeamDelete()  → 팀 + 태스크 디렉토리 정리 (모든 에이전트 종료 후에만 가능)
```

### 전체 명령어 요약

| 도구 | 역할 | 언제 사용 |
|------|------|----------|
| `TeamCreate` | 팀 생성 + 공유 태스크 리스트 생성 | 맨 처음 |
| `TaskCreate` | 작업 생성 | 팀 생성 후 |
| `TaskUpdate` | 작업 상태/의존성/할당 변경 | 수시로 |
| `TaskList` | 전체 태스크 현황 확인 | 수시로 |
| `TaskGet` | 특정 태스크 상세 조회 | 필요시 |
| `Agent` | 서브에이전트 스폰 (팀에 합류) | 작업 시작 시 |
| `SendMessage` | 에이전트 간 메시지 / 종료 요청 | 소통 및 종료 시 |
| `TeamDelete` | 팀 + 태스크 정리 | 모든 작업 완료 후 |

### 커스텀 서브에이전트

`.claude/agents/` 폴더에 마크다운 파일로 서브에이전트를 정의할 수 있다. 예시:

| 에이전트 | 역할 |
|---------|------|
| **pm** | 프로덕트 매니저 — 기획, 유저 플로우, 요구사항 |
| **frontend** | 프론트엔드 개발자 — Next.js + TypeScript |
| **frontend-reviewer** | 코드 리뷰어 — 버그, 보안, 성능, 유지보수성 |
| **frontend-test** | 테스트 전문가 — TDD, 단위/E2E 테스트 |

이 에이전트들을 팀으로 구성할 때는 `Agent` 도구의 `subagent_type`에 에이전트 이름을 넣으면 된다:

```
TeamCreate("my-team")
    │
    ├── Agent(subagent_type="pm",                name="pm")
    ├── Agent(subagent_type="frontend",          name="frontend")
    ├── Agent(subagent_type="frontend-reviewer",  name="reviewer")
    └── Agent(subagent_type="frontend-test",      name="tester")
```

### 사전 설정

Agent Team은 **실험적 기능**으로 기본 비활성화다. `~/.claude/settings.json`에 플래그를 추가해야 한다:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

표시 모드도 설정할 수 있다:

| 모드 | 설정값 | 설명 |
|------|--------|------|
| **in-process** (기본) | `"in-process"` | 모든 팀원이 같은 터미널에서 실행. `Shift+Down`으로 팀원 간 전환 |
| **split-panes** | `"tmux"` | 팀원마다 별도 패널 (tmux 또는 iTerm2 필요) |

---

## 2. 개발도 여러 명이서 병렬로 가능할까?

**가능하다.** 핵심은 **파일 충돌이 안 나게 작업을 분리**하는 것이다.

### 도메인(기능)별 분리

```
frontend-a  ──→ features/auth/      (인증 도메인)
frontend-b  ──→ features/todo/      (투두 도메인)
frontend-c  ──→ features/dashboard/  (대시보드 도메인)
```

### 레이어별 분리

```
frontend-a  ──→ hooks + types       (로직 레이어)
frontend-b  ──→ components          (UI 레이어)
frontend-c  ──→ API routes          (백엔드 레이어)
```

같은 `frontend.md` 에이전트를 **이름만 다르게** 여러 개 스폰하면 된다.

### 주의할 점

- **같은 파일을 동시에 수정하면 충돌** → 담당 영역을 명확히 나눠야 함
- **공통 타입/유틸**은 한 에이전트가 먼저 만들고, 나머지가 import
- `git worktree`로 에이전트별 격리된 브랜치에서 작업 후 머지하는 방법도 있음

결국 인간 팀 개발이랑 같은 원리다 — **역할 분담 + 의존성 관리**가 핵심이다.

---

## 3. 4가지 조합 소개

Agent Team과 Ralph Loop을 어떻게 조합하느냐에 따라 4가지 방식이 나온다:

```
                  같은 브랜치           Worktree (격리)
              ┌──────────────────┬──────────────────────┐
  1회성       │ A. Agent Team    │ B. Agent Team+Worktree│
              │ (dev병렬→리뷰)   │ (dev격리→머지→리뷰)    │
              ├──────────────────┼──────────────────────┤
  반복 루프    │ C. Ralph Loop    │ D. Ralph Loop         │
              │    + Team        │    Parallel            │
              │ (dev→리뷰→수정)  │ (서브에이전트 반복)     │
              └──────────────────┴──────────────────────┘
```

> **1회성**: 개발 → 리뷰 → 끝. 수정 기회 없음.
> **반복**: 개발 → 리뷰 → 수정 → 재리뷰 → 버그 0까지 반복.

---

## 4. 조합 A: Agent Team — 1회성 병렬 협업

### 개념

여러 에이전트를 팀으로 묶어 **각자 1번 실행하고 끝**나는 방식이다. 팀 리드(메인 Claude 세션)가 작업을 조율하고, 팀원들은 각자 독립된 컨텍스트에서 작업하며 서로 직접 메시지를 주고받는다.

> **Subagent와 다른 점**: Subagent는 결과만 메인에 반환하고 서로 소통이 안 된다. Agent Team은 팀원끼리 **직접 메시지를 주고받고**, **공유 태스크 리스트**로 자기 조율이 가능하다.

### 동작 방식

```
팀 리드 (메인 세션)
  │
  ├── dev-1 (타입+훅)      ─┐
  ├── dev-2 (UI 컴포넌트)   ─┘── 병렬
  │     ↓ 둘 다 완료
  ├── dev-3 (메인 페이지 조립)
  │     ↓ 완료
  ├── reviewer → 코드 리뷰  ─┐
  └── tester → 테스트 작성   ─┘── 병렬
```

### 실행 방법

자연어로 Claude에게 팀 구성을 요청한다:

```
Todo 앱을 만들 에이전트 팀을 구성해줘.
스펙은 docs/specs/todo/spec.md에 있어.

팀원 구성:
- frontend 2명: 타입+훅 개발, UI 컴포넌트 개발 (병렬)
- frontend 1명: 메인 페이지 조립 (위 2명 완료 후)
- frontend-reviewer 1명: 코드 리뷰
- frontend-test 1명: 테스트 작성

각 팀원의 담당 파일:
- dev-1: src/features/todo/ (타입 + 훅)
- dev-2: src/components/todo/ (UI 컴포넌트)
- dev-3: src/app/page.tsx (메인 페이지)
```

Claude가 내부적으로 `TeamCreate → TaskCreate → Agent(스폰) → SendMessage(종료) → TeamDelete` 순서로 처리한다.

### 적합한 상황

- 기획 → 개발 → 리뷰 같은 **순차 파이프라인**이 필요할 때
- 팀원 간 **실시간 소통**이 필요할 때
- 작업 범위가 명확히 분리되어 **파일 충돌 걱정이 없을 때**

### 한계

- 리뷰 피드백을 반영하려면 수동으로 다시 돌려야 한다
- **세션 복원 불가**: 팀원은 `/resume`이나 `/rewind`로 복원 불가
- **세션당 1개 팀만** 가능

---

## 5. 조합 B: Agent Team + Worktree — 에이전트별 브랜치 분리

### 개념

조합 A와 동일하지만, 각 에이전트가 **독립된 git worktree(브랜치)**에서 작업한다. 파일 충돌이 **물리적으로 원천 차단**된다.

> Git Worktree가 생소하다면 → [Git Worktree — 하나의 저장소, 여러 브랜치 동시 작업](../Git%20Worktree/git-worktree_2026-03-01.md)

```
main 브랜치
├── .claude/worktrees/feat-hooks/       ← dev-1 전용
├── .claude/worktrees/feat-components/  ← dev-2 전용
└── main에서 작업                       ← dev-3, reviewer, tester
```

### 동작 방식

```
팀 리드
  │
  ├── dev-1 (worktree) → feat/hooks 브랜치    ─┐
  ├── dev-2 (worktree) → feat/components 브랜치 ─┘── 병렬
  │     ↓ 둘 다 완료 → main에 머지
  ├── dev-3 → 메인 페이지 조립
  │     ↓
  ├── reviewer → 코드 리뷰  ─┐
  └── tester → 테스트 작성   ─┘── 병렬
```

### 조합 A vs B 비교

| | A: Agent Team만 | B: Agent Team + Worktree |
|--|-----------------|--------------------------|
| 파일 충돌 | 같은 파일 수정 시 충돌 가능 | 물리적으로 분리, **작업 중 충돌 없음** |
| 작업 분리 | PROMPT로 담당 영역 지정 | 물리적으로 분리됨 |
| 머지 | 불필요 (같은 브랜치) | 작업 후 **main에 머지 필요** |
| 설정 복잡도 | 낮음 | 높음 (worktree 관리 필요) |

### 적합한 상황

- 각 에이전트의 **작업 범위가 겹칠 수 있을 때**
- 대규모 기능을 **완전히 격리해서 개발**하고 싶을 때

### 한계

- 머지 과정에서 충돌 해결이 필요할 수 있다
- 1회성이라는 한계는 조합 A와 동일

---

## 6. 조합 C: Ralph Loop + Team — 반복 + 팀 협업

### 개념

**반복 메커니즘** 안에서 매 iteration마다 **에이전트 팀이 협업**한다. 리뷰 피드백이 파일로 남고, 다음 iteration에서 이를 읽고 반영한다.

```
iteration 1: dev 개발 → reviewer 리뷰 → REVIEW.md "버그 3개"
iteration 2: dev가 REVIEW.md 읽고 수정 → reviewer 재리뷰 → "버그 0개" ✅
```

### 동작 방식

```
Ralph Loop (반복)
  │
  ├── iteration 1:
  │   ├── dev → 전체 구현
  │   ├── reviewer + tester (병렬)
  │   └── REVIEW.md → "Critical 1개"
  │
  ├── iteration 2:
  │   ├── dev → REVIEW.md 읽고 수정
  │   ├── reviewer → 재리뷰
  │   └── REVIEW.md → "Critical 0개" ✅
  │
  └── 종료
```

### 피드백 반영의 핵심

Claude는 **이전 대화를 기억하지 못한다.** 파일만 본다.

따라서:
- 리뷰 결과를 **REVIEW.md에 파일로 남기고**
- 프롬프트에 **"REVIEW.md를 봐라"고 명시적으로 적어야** 한다

### 핵심 파일들

| 파일 | 역할 | 누가 쓰나 |
|------|------|----------|
| `REVIEW.md` | 리뷰 피드백 기록 | reviewer가 매 iteration마다 갱신 |
| `progress.md` | 진행 상황 기록 | 매 iteration 끝에 갱신 |
| `PROMPT.md` | 반복될 프롬프트 | 최초 1회 작성 (고정) |

### 적합한 상황

- **리뷰 → 수정 → 재리뷰** 사이클이 필요할 때
- **품질 기준이 높아서** 반복적인 검증이 필요할 때
- 한 번에 완벽하게 만들기 어려운 **복잡한 기능**을 점진적으로 개선할 때

### 한계

- 매 iteration마다 팀 전체가 동작하므로 토큰 비용이 높다
- PROMPT 설계가 복잡하다 — 파일 참조 규칙, 종료 조건을 모두 명시해야 한다

---

## 7. 조합 D: Ralph Loop 병렬 — 서브에이전트 반복

### 개념

조합 C와 동일한 반복 루프지만, **TeamCreate 없이 일반 Agent 서브에이전트로 병렬 실행**한다.

```
Agent(dev) → Agent(reviewer) + Agent(tester) 병렬
           → Critical > 0 → Agent(dev-fix) → Agent(reviewer) 재검증
           → Critical 0 → 종료
```

### 조합 C vs D 비교

| | C: Ralph Loop + Team | D: Ralph Loop Parallel |
|--|---------------------|----------------------|
| 팀 관리 | TeamCreate 사용 | **TeamCreate 미사용** |
| 통신 방식 | SendMessage | 서브에이전트 반환값 |
| 설정 복잡도 | 팀 생성/삭제 필요 | 단순 (Agent 호출만) |
| 비용 | 높음 | 높음 |

### 적합한 상황

- 팀 관리 오버헤드 없이 **간단하게 반복 루프**를 돌리고 싶을 때
- 품질에 타협 없이 **모든 이슈를 잡아야** 할 때

---

## 8. 4가지 조합 비교표

| 조합 | 실행 | 소통 | 반복 개선 | 파일 충돌 | 비용 |
|------|------|------|----------|----------|------|
| **A. Agent Team** | 1회성 | SendMessage | 없음 | 가능 | 낮음 |
| **B. Team + Worktree** | 1회성 | SendMessage | 없음 | 없음 | 낮음 |
| **C. Ralph Loop + Team** | 반복 | 파일 기반 | 있음 | 가능 | 높음 |
| **D. Ralph Loop Parallel** | 반복 | 서브에이전트 | 있음 | 없음 | 높음 |

### 어떤 조합을 선택할까?

```
"프로토타입 빨리 필요해"
  → A. Agent Team — ~8분, 가장 빠름

"병렬 개발인데 파일 충돌이 걱정돼"
  → B. Team + Worktree — ~8분, 물리적 격리

"프로덕션 코드야, 버그 없어야 해"
  → C. Ralph Loop + Team — ~10분, Critical 0 보장 ⭐

"품질에 타협 없어, Warning도 다 잡아"
  → D. Ralph Loop Parallel — ~14분, 테스트 최다
```

---

> **실험 설계, 실험 결과, 핵심 발견**은 별도 글에서 자세히 다룬다:
> → [AI 코딩 에이전트, 어떻게 팀을 짜야 버그 없는 코드가 나올까?](../AI%20코딩%20에이전트,%20어떻게%20팀을%20짜야%20버그%20없는%20코드가%20나올까?/ai-coding-agent-team-experiment_2026-03-04.md)

---

## 9. PROMPT 설계가 핵심이다

Ralph Loop에서 가장 중요한 것은 **PROMPT 설계**다.

### Ralph Loop의 원리

매 iteration마다 Claude가 보는 것은 딱 두 가지다:

1. **같은 PROMPT** (고정)
2. **현재 파일 상태** (변경됨)

Claude는 **이전 대화를 기억하지 못한다.** 파일만 본다.

### 피드백 반영의 핵심: 파일로 남겨야 한다

```
/ralph-loop "
## 규칙
1. 작업 전에 REVIEW.md가 있으면 반드시 읽고 피드백 반영
2. 개발 완료 후 리뷰 결과를 REVIEW.md에 기록
3. REVIEW.md에 이슈가 0개이면 <promise>DONE</promise>
"
```

### PROMPT에서 결정하는 것들

| 요소 | PROMPT에서 결정 |
|------|----------------|
| **뭘 할지** | 체크리스트, 작업 범위 |
| **어떻게 할지** | 에이전트 역할, 실행 순서 |
| **뭘 볼지** | REVIEW.md, progress.md 등 참조 파일 |
| **언제 끝낼지** | completion promise 조건 |
| **어떻게 소통할지** | 파일 기반 피드백 규칙 |

**PROMPT에 안 적으면 안 한다. 적으면 한다. 그게 전부다.**

---

## 마무리

### 결론

| 상황 | 추천 조합 | 이유 |
|------|----------|------|
| 빠른 프로토타입 | A. Agent Team | ~8분, 최소 시간 |
| 병렬 독립 기능 | B. Team + Worktree | 파일 충돌 방지 |
| **프로덕션 코드** | **C. Ralph Loop + Team** ⭐ | **~10분, Critical 0, 간단** |
| 최고 품질 필요 | D. Ralph Loop Parallel | ~14분, 테스트 최다 |

**기본값: C. Ralph Loop + Team.**

에이전트가 좋으면 1회 만에 끝나고, 나쁘면 자동으로 여러 번 돈다.
어느 쪽이든 손해가 없는 구조다.

AI 코딩 에이전트를 실무에 쓰려면 —
"한 번에 완벽하게"보다 **"빠르게 만들고, 검증하고, 고치는 루프"**가 답이다.

### 실험 환경

- Claude Code v2.1.63 + Claude Opus 4.6 / Sonnet 4.6
- macOS, Next.js 16.1.6, TypeScript 5
- 에이전트: `.claude/agents/` (frontend, frontend-reviewer, frontend-test)
- 스킬: `.claude/skills/` (14개 규칙)
- 스펙: `docs/specs/todo/spec.md` (PM 에이전트 + /spec-review)

---

## 레퍼런스

- [Claude Code Agent Teams](https://docs.anthropic.com/en/docs/claude-code/agent-teams)
- [Ralph Wiggum Technique — Geoffrey Huntley](https://ghuntley.com/ralph/)
- [Claude Code Git Worktrees](https://docs.anthropic.com/en/docs/claude-code/git-worktrees)
