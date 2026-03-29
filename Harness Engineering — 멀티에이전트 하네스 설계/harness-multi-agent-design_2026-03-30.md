# Harness Engineering — 멀티에이전트 하네스 설계

> 최종 수정: 2026-03-30

## TL;DR

AI 에이전트를 감싸는 운영 구조(Harness)를 Anthropic의 6축 프레임워크로 평가하고, 부족한 4가지 Gap(Playwright Evaluator, Context Reset, Score-based Pivot, Sprint Contract)을 구현하여 프로세스를 강화했다. 핵심 발견: 프롬프트 기술보다 **구조(하네스)**가 결과에 더 큰 영향을 미치며, 하네스의 모든 구성요소는 모델 한계에 대한 가정이므로 모델이 발전하면 재설계해야 한다.

---

## Part 1: 하네스 엔지니어링이란

### 정의

**하네스(Harness) = AI 모델을 감싸는 운영 구조.**

같은 AI 모델이라도 어떤 구조 안에서 돌리느냐에 따라 결과가 완전히 달라진다. 마치 같은 재능을 가진 사람이 혼자 일하는 것과 체계적인 팀 안에서 일하는 것의 차이와 같다.

**하네스 엔지니어링 = 그 구조를 설계하고, 실험하고, 모델 진화에 맞춰 재설계하는 엔지니어링.**

> "AI에게 '잘 해줘'라고 말하는 것보다, AI가 잘할 수밖에 없는 환경을 만들어주는 것이 더 중요하다."

### Claude Code에서 하네스를 구성하는 요소들

Claude Code 자체가 하나의 harness이다. Claude 모델을 감싸서 도구를 제공하고, 컨텍스트를 관리하고, 실행 루프를 제어한다. 그 위에 아래 요소들을 조합하면 커스텀 하네스가 된다:

| 구성 요소 | 역할 | 위치 |
|----------|------|------|
| **Agents** | 역할별 전문 에이전트 정의 (모델, 스킬, 시스템 프롬프트) | `~/.claude/agents/` |
| **Skills** | 에이전트가 따라야 할 규칙/체크리스트 | `~/.claude/skills/` |
| **Commands** | 사용자가 직접 호출하는 워크플로우 (`/spec-review` 등) | `~/.claude/commands/` |
| **Hooks** | 이벤트 기반 자동화 (SessionStart, Compact 등) | `settings.json` > hooks |
| **Templates** | 산출물의 표준 형식 (contract.md, handoff.md 등) | `~/.claude/templates/` |
| **Scripts** | 외부 명령 실행 (점수 기록, 정체 감지 등) | `~/.claude/scripts/` |
| **CLAUDE.md** | 전체 프로세스 정의 + 행동 규칙 | `~/.claude/CLAUDE.md` |
| **MCP** | 외부 도구 연동 (Playwright, Notion, Pencil 등) | `settings.json` > mcpServers |
| **Plugins** | 확장 기능 (Ralph Loop, code-review 등) | `settings.json` > enabledPlugins |

### Anthropic이 제시한 하네스의 6축

Anthropic 엔지니어링 블로그 "Harness design for long-running application development" (2026.03.24)에서 도출한 평가 프레임워크:

| # | 축 | 핵심 질문 |
|---|-----|----------|
| 1 | **역할 분리** | 만드는 AI와 평가하는 AI가 분리되어 있는가? |
| 2 | **컨텍스트 관리** | 긴 작업에서 컨텍스트 불안을 어떻게 해소하는가? |
| 3 | **품질 평가 체계** | "좋은가?"가 아니라 "기준에 맞는가?"로 평가하는가? |
| 4 | **반복 메커니즘** | 피드백 기반 반복이 있고, 정체 시 방향 전환이 가능한가? |
| 5 | **사전 합의** | 코딩 전 완료 기준을 합의하는가? (Sprint Contract) |
| 6 | **모델 적응** | 새 모델이 나오면 불필요한 구성요소를 제거하고 재설계하는가? |

### 하네스를 만들고 관리하는 방법론

Anthropic이 강조하는 3가지 원칙:

**원칙 1: 모델을 직접 관찰하라**
AI에게 실제 문제를 풀게 하고, 그 과정을 읽고, 원하는 결과가 나올 때까지 튜닝하라. 추측하지 말고 관찰하라.

**원칙 2: 복잡한 작업은 분해하고 전문화하라**
하나의 AI에게 모든 걸 시키는 대신, 기획/개발/검수처럼 역할별 전문 에이전트를 두면 성능이 올라간다.

**원칙 3: 새 모델이 나오면 하네스를 다시 점검하라**
하네스의 모든 구성요소는 "모델이 혼자 못 하는 것"에 대한 가정이다. 그 가정은 모델이 발전하면 깨질 수 있다. 불필요해진 구성요소는 빼고, 새로 가능해진 것을 위한 구성요소를 추가하라.

> "흥미로운 하네스 조합의 공간은 모델이 발전해도 줄어들지 않는다. 이동할 뿐이다."

---

## Part 2: Anthropic의 하네스 설계 사례

> 출처: Anthropic Engineering Blog, "Harness design for long-running application development" (2026.03.24)

### AI를 그냥 시키면 왜 실패하는가

**문제 1: 컨텍스트 불안 (Context Anxiety)**

대화가 길어지면 AI가 앞에서 뭘 했는지 헷갈리고, 아직 할 일이 남았는데도 서둘러 끝내버린다. 100페이지 보고서를 쓰는 신입사원이 60페이지쯤 되면 나머지를 대충 마무리하는 것과 같다.

해결: **Context Reset** — AI 한 마리가 전부 하는 대신, AI 1번이 30페이지를 쓰고 인수인계 메모를 남기면 AI 2번이 그 메모를 읽고 다음 30페이지를 쓴다. 각 AI는 항상 머리가 맑은 상태에서 시작한다.

Compaction(같은 AI가 앞부분을 요약해 줄이는 것)과의 차이: Compaction은 "방이 점점 좁아지는" 느낌을 해소 못 한다. Sonnet 4.5에서는 Compaction만으로 부족했다.

**문제 2: 자기 평가의 한계**

"방금 네가 만든 디자인 평가해봐"라고 하면, 품질이 별로여도 "아주 훌륭합니다!"라고 답한다. 자기 작업을 비판하도록 만드는 것보다, 별도의 채점자를 엄격하게 세팅하는 것이 훨씬 쉽다.

### 디자인 실험: Generator/Evaluator 분리

GAN(적대적 생성 신경망)에서 영감을 받아, Generator(만드는 AI)와 Evaluator(채점하는 AI)를 분리했다.

**4가지 채점 기준표 (비중 배분이 핵심):**

| 기준 | 비중 | 이유 |
|------|------|------|
| Design Quality (디자인 품질) | **높음** | AI가 약한 영역 |
| Originality (독창성) | **높음** | AI가 약한 영역 |
| Craft (기술적 완성도) | 낮음 | AI가 이미 잘하는 영역 |
| Functionality (기능성) | 낮음 | AI가 이미 잘하는 영역 |

AI가 잘하는 영역에 낮은 비중, 못하는 영역에 높은 비중을 주면 "안전한 뻔한 디자인" 대신 "미적 모험"을 하도록 유도된다.

**네덜란드 미술관 사례 — 10번째 반복의 창의적 도약:**

9회차까지는 점점 다듬어지면서 깔끔한 다크 테마 랜딩페이지. 잘 만들었지만 예상 범위 안이었다. 그런데 10회차에서 AI가 완전히 다른 것을 시도 — CSS로 원근감 있는 3D 방을 만들어서 바닥에 체크무늬가 깔려 있고, 벽에 그림이 걸려 있고, 문을 열면 다음 전시실로 이동하는 "공간 탐험형" 미술관을 구현했다. 한 번에 나올 수 있는 결과가 아니다. 9번의 "이 정도로는 부족하다" 신호를 받다가 완전히 새로운 발상으로 넘어간 것.

흥미로운 발견: 채점 기준에 "최고의 디자인은 미술관 수준이다"라는 문장을 넣었더니, 평가 피드백이 오기 전인 **1회차부터** 이미 결과가 달라졌다. 기준표의 문구 자체가 AI의 마인드셋을 바꾼 것.

### 풀스택 앱: 3인 에이전트 체제

디자인 실험에서 검증된 패턴에 Planner를 추가:

| 역할 | 하는 일 |
|------|--------|
| **Planner (기획팀장)** | 한 줄 프롬프트를 16개 기능, 10개 스프린트짜리 상세 설계서로 확장. 단, 세부 기술 구현은 정하지 않음 |
| **Generator (개발자)** | 설계서를 보고 한 기능씩 스프린트 방식으로 코딩. React + FastAPI + DB. Git 버전 관리 |
| **Evaluator (QA)** | Playwright로 실제 앱을 브라우저에서 열어 버튼 클릭, API 호출, DB 확인. 기준 미달이면 구체적 피드백과 함께 반려 |

**Sprint Contract:** 코딩 시작 전에 Generator와 Evaluator가 "이번 스프린트에서 뭘 만들고, 뭘 기준으로 합격/불합격을 판단할지" 합의.

**Solo vs 3-Agent 비교 (레트로 게임 메이커):**

| | Solo Agent | 3-Agent 하네스 |
|--|-----------|---------------|
| 시간 | 20분 | 6시간 |
| 비용 | $9 | $200 |
| 결과 | 겉보기 그럴듯, 실제로 게임 플레이 안 됨 | 16개 기능 작동, 프로덕션급 |

비용은 20배 차이나지만, Solo의 $9은 버린 돈이고 하네스의 $200은 실제로 쓸 수 있는 결과물.

### 모델 진화에 따른 하네스 재설계

Opus 4.5 → Opus 4.6 업그레이드 시 하네스에서 3가지를 뺄 수 있었다:

| 제거한 것 | 이유 |
|----------|------|
| 스프린트 분할 | 4.6은 2시간 넘게 연속으로 일관되게 코딩 가능 |
| Context Reset | 컨텍스트 불안이 크게 줄어듦 |
| 매 스프린트 QA → 최종 QA만 | 중간 품질이 충분히 높아짐 |

업데이트된 하네스로 "브라우저 DAW(음악 제작 프로그램)" 제작: 4시간, $125에 핵심 기능 작동.

---

## Part 3: 내 하네스 현황 + 평가 (6축)

### 현황: 구성 요소 전체 목록

**에이전트 6개:**

| 에이전트 | 모델 | 역할 | 스킬 수 |
|---------|------|------|--------|
| `pm` | Opus | 프로덕트 매니저 | 4개 |
| `frontend` | Sonnet | Next.js + TypeScript 개발자 | 7개 |
| `frontend-reviewer` | Sonnet | 코드 리뷰어 (버그/보안/성능/유지보수) | 6개 |
| `frontend-test` | Opus | 테스트 작성 (TDD/E2E) | 6개 |
| `evaluator` | Opus | Playwright 앱 검증 + 4축 점수 | 3개 |
| `eval-all` | Opus | 서비스 종합 평가 (5명 심사위원) | 5개 |

**스킬 44개:**

| 카테고리 | 스킬 수 | 목록 |
|---------|--------|------|
| 프론트엔드 개발 | 7개 | components, style, naming, structure, fundamentals, accessibility, design |
| 프론트엔드 리뷰 | 4개 | review-bugs, review-security, review-performance, review-maintainability |
| 프론트엔드 테스트 | 6개 | unit-test, component-test, api-mock-test, e2e-test, visual-regression, tdd-workflow |
| 프론트엔드 기타 | 2개 | form, axios |
| PM | 5개 | requirements, user-flow, error-scenarios, information-architecture, ux-heuristics |
| PM 기타 | 1개 | dummy-dataset |
| 평가 | 6개 | public, ux, tech, business, critic, playwright |
| 개인 도구 | 6개 | context-sync, session-wrap, update-check, fetch-youtube, fetch-tweet, content-digest |
| 학습 | 6개 | day1~6 온보딩/실습 |
| 기타 | 1개 | update-check-workspace |

**커맨드 7개:**

| 커맨드 | 용도 | Phase |
|--------|------|-------|
| `/spec-review` | 스펙 심층 검토 (인터뷰 방식) | 2 |
| `/pre-mortem` | 개발 리스크 사전 분석 | 2.5 |
| `/test-scenarios` | 테스트 시나리오 생성 | 4.5 |
| `/sprint-contract` | Generator/Evaluator 합격 기준 합의 | 4.7 |
| `/pivot-check` | 점수 정체 감지 + 방향 전환 | 7 |
| `/commit` | 커밋 메시지 자동 생성 | 전체 |
| `/learn-claude-code` | Claude Code 학습 | - |

**훅 5개:**

| 훅 | 트리거 | 역할 |
|----|--------|------|
| SessionStart | 세션 시작 | plan/progress/memory/findings.md 자동 생성 |
| PreCompact | Compact 전 | `/tmp/.claude-compacted` 마커 생성 |
| PostCompact | Compact 후 | 컨텍스트 복구 리마인더 |
| UserPromptSubmit | 사용자 입력 | Compact 감지 시 복구 리마인더 |
| StopFailure | API 에러 | 에러 알림 전송 |

**프로세스 (Phase 0~8):**

```
Phase 0:   브레인스토밍 → brainstorming 스킬
Phase 1:   기획 → pm 에이전트
Phase 2:   기획 검증 → /spec-review
Phase 2.5: 리스크 분석 → /pre-mortem
Phase 3:   UI 설계 → Pencil MCP
Phase 4:   구현 계획 → writing-plans
Phase 4.5: 테스트 계획 → /test-scenarios
Phase 4.7: Sprint Contract → /sprint-contract
Phase 5:   Foundation → Lead 직접
Phase 6:   병렬 개발 → 팀 에이전트 (worktree)
Phase 7:   검증 루프 → evaluator + handoff + pivot + reviewer + test
Phase 8:   기록 → /wrap + handoff + score-history
```

### 평가: 6축 채점

#### 축 1. 역할 분리 — ⭐⭐⭐⭐⭐ (탁월)

Anthropic의 3인 체제(Planner/Generator/Evaluator)보다 더 세분화:

```
Anthropic:  Planner(1) + Generator(1) + Evaluator(1) = 3명
나:          PM(1) + Frontend(1) + Reviewer(1) + Tester(1) + Evaluator(1) + Eval(5명) = 10명
```

특히 Evaluator를 코드 리뷰어(frontend-reviewer)와 앱 검증자(evaluator)로 분리한 것, eval-all의 5명 페르소나 분리는 독창적.

#### 축 2. 컨텍스트 관리 — ⭐⭐⭐⭐½

- Compact 감지 + 자동 복구 리마인더: 실용적
- 4개 컨텍스트 파일 (plan/progress/memory/findings): 구조적
- handoff.md 템플릿: 구조화된 인수인계 (신규 추가)
- 부족: 완전한 Context Reset(새 에이전트로 교체)은 아님. 같은 에이전트가 handoff를 읽는 방식

#### 축 3. 품질 평가 체계 — ⭐⭐⭐⭐

- evaluator: Playwright MCP로 실제 앱 검증 + 4축 가중 채점 (신규 추가)
- frontend-reviewer: Confidence Score 0-100 (80+ 보고, Critical 90+)
- eval-all: 5관점 가중 평균 /100
- 부족: 실전 미검증, Sprint Contract 없으면 일반 체크리스트에 의존

#### 축 4. 반복 메커니즘 — ⭐⭐⭐⭐⭐

- Ralph Loop (Closed-loop + 피드백 주입): 8회 실험으로 검증 (반복 조합 100% Critical 0)
- stall-detector + /pivot-check: 3회 정체 시 방향 전환 (신규 추가)
- score-history.jsonl: 점수 추이 추적

#### 축 5. Sprint Contract — ⭐⭐⭐⭐

- `/sprint-contract` 커맨드: spec.md 기반 4축 합격 기준 생성 (신규 추가)
- evaluator가 contract.md 기준으로 Pass/Fail 판정
- 부족: 실전 미검증

#### 축 6. 모델 적응 — ⭐ (미구현)

- 하네스가 고정 구조
- 어떤 컴포넌트가 "모델 한계 보정용"이고 어떤 것이 "필수"인지 구분 없음
- 새 모델 나와도 재설계 기준 없음

---

## Part 4: Gap 개선 — 무엇을 만들었나

Anthropic 6축 기준으로 평가한 후, 부족한 4개 Gap을 구현했다.

### Gap 2: evaluator 에이전트 (축 3 강화)

**문제:** frontend-reviewer가 `git diff` 기반 정적 코드 리뷰만 수행. 실제 앱을 열어보지 않음.

**구현:**

| 파일 | 역할 |
|------|------|
| `~/.claude/agents/evaluator.md` | Playwright로 실제 앱 검증 + 4축 가중 점수 |
| `~/.claude/skills/eval-playwright/SKILL.md` | Playwright MCP 동적 테스트 규칙 |
| `~/.claude/scripts/eval-score-log.sh` | 점수를 score-history.jsonl에 기록 |

**evaluator 3단계 프로세스:**

```
1단계: 정적 분석 (tsc --noEmit, lint, 코드 리뷰)
2단계: 동적 테스트 (Playwright로 앱 열기, 클릭, 스크린샷, 반응형)
3단계: 점수 산출 (4축 가중 평균)
```

**4축 채점 기준:**

| 축 | 가중치 | 이유 |
|----|--------|------|
| 기능성 | 40% | 핵심. 버튼이 작동해야 함 |
| 디자인 품질 | 25% | AI가 약한 영역 → 높은 비중 |
| 코드 품질 | 20% | AI가 잘하는 영역 → 낮은 비중 |
| 완성도 | 15% | Empty/Loading/Error 상태 등 |

### Gap 1: handoff.md (축 2 강화)

**문제:** Ralph Loop 매 iteration에서 에이전트가 파일 시스템을 스스로 추측. 구조화된 인수인계 없음.

**구현:**

| 파일 | 역할 |
|------|------|
| `~/.claude/templates/handoff.md` | 인수인계 문서 템플릿 |
| `~/CLAUDE.md` (수정) | handoff 규칙 추가 |
| `~/.claude/CLAUDE.md` (수정) | 컨텍스트 관리에 handoff 최우선 읽기 추가 |

**handoff.md 구조:**

```
현재 상태 (Phase, 마지막 점수)
완료된 작업 (체크리스트)
진행 중인 작업 + 의도
미해결 이슈
Evaluator 피드백 (마지막)
주의사항 (시도했으나 실패한 접근)
다음 할 일
```

**구현 방식:** Ralph Loop의 stop-hook을 수정하지 않고, 프롬프트 자체에 "handoff.md 읽기 → 작업 → handoff.md 업데이트" 사이클을 포함.

### Gap 3: Score-based Pivot (축 4 보완)

**문제:** 반복해도 점수가 안 오르면 같은 방향으로만 수정. 창의적 점프 없음.

**구현:**

| 파일 | 역할 |
|------|------|
| `~/.claude/scripts/stall-detector.sh` | score-history.jsonl에서 정체 감지 |
| `~/.claude/commands/pivot-check.md` | 수동 pivot 커맨드 |

**판정 기준 (최근 3회 점수 변화):**

| 판정 | 조건 |
|------|------|
| PROGRESSING | delta > +2점 1회 이상 |
| STALL | 3회 연속 ±2점 이내 |
| REGRESSING | 3회 연속 < -2점 |

**검증 결과:** 3가지 시나리오 모두 정확 판정 (48→63→73 = PROGRESSING, 75→75.5→74.8 = STALL, 80→72→65 = REGRESSING)

### Gap 5: Sprint Contract (축 5 신설)

**문제:** Generator(frontend)가 평가 기준을 모르고 코딩, Evaluator가 자기 기준으로 채점. 사전 합의 없음.

**구현:**

| 파일 | 역할 |
|------|------|
| `~/.claude/commands/sprint-contract.md` | spec.md → contract.md 생성 |
| `~/.claude/templates/contract.md` | 합격 기준 문서 템플릿 |

**워크플로우:** spec.md 분석 → 4축별 테스트 가능한 합격 기준 작성 → 자동화 가능 여부 태깅 → 사용자 확인 → contract.md 저장

---

## Part 5: 업데이트된 프로세스 (Phase 0~8)

### 전체 흐름

```
Phase 0:   브레인스토밍 → brainstorming 스킬
Phase 1:   기획 → pm 에이전트
Phase 2:   기획 검증 → /spec-review
Phase 2.5: 리스크 분석 → /pre-mortem
Phase 3:   UI 설계 → Pencil MCP
Phase 4:   구현 계획 → writing-plans
Phase 4.5: 테스트 계획 → /test-scenarios
Phase 4.7: Sprint Contract → /sprint-contract
Phase 5:   Foundation → Lead 직접
Phase 6:   병렬 개발 → 팀 에이전트 (worktree)
Phase 7:   검증 루프 → evaluator + handoff + pivot + reviewer + test
Phase 8:   기록 → /wrap + handoff + score-history
```

### Phase별 에이전트/스킬/커맨드 매핑

| Phase | 에이전트 | 스킬/커맨드 |
|-------|---------|------------|
| 0. 브레인스토밍 | - | brainstorming (superpowers) |
| 1. 기획 | `pm` (opus) | pm-requirements, pm-user-flow, pm-error-scenarios, pm-ux-heuristics |
| 2. 기획 검증 | - | `/spec-review` |
| 2.5 리스크 분석 | - | `/pre-mortem` |
| 3. UI 설계 | - | Pencil MCP |
| 4. 구현 계획 | - | writing-plans (superpowers) |
| 4.5 테스트 계획 | - | `/test-scenarios` |
| 4.7 Sprint Contract | - | `/sprint-contract` |
| 5. Foundation | Lead (opus) | frontend-* 스킬 + pm-dummy-dataset |
| 6. 병렬 개발 | `frontend` (sonnet) × N | frontend-components, style, naming, structure, fundamentals, accessibility, design |
| 7. 검증 루프 | `evaluator` (opus) | eval-playwright, frontend-review-bugs, frontend-review-performance |
| | `frontend-reviewer` (sonnet) | frontend-review-bugs/security/performance/maintainability, fundamentals, accessibility |
| | `frontend-test` (opus) | frontend-unit/component/api-mock/e2e/visual-regression-test, tdd-workflow |
| | - | `/pivot-check`, stall-detector.sh, eval-score-log.sh |
| 8. 기록 | - | `/wrap` (my-session-wrap) |

### 이전 프로세스 대비 변경점

| Phase | 이전 | 이후 |
|-------|------|------|
| **4.7** | (없음) | `/sprint-contract` 추가 |
| **7** | reviewer + test + Playwright | evaluator + handoff + pivot + reviewer + test |
| **8** | docs/conversations/ (수동 기록) | /wrap + handoff + score-history (자동화) |
| **컨텍스트 관리** | plan/progress/memory/findings | **handoff.md 최우선** + plan/progress/memory/findings |

---

## 핵심 인사이트

1. **구조가 프롬프트보다 중요하다.** 같은 프롬프트, 같은 스킬인데 조합(구조)만 바꿔서 결과가 달라진다. 8회 실험에서 반복 조합만 100% Critical 0을 달성. 이번에 추가한 evaluator, handoff, pivot, sprint-contract도 모두 "구조"의 개선이다.

2. **만드는 AI와 평가하는 AI를 분리하라.** 그리고 평가 기준에서 "AI가 약한 영역"에 높은 비중을 두면 AI가 안전한 선택 대신 도전적인 시도를 한다. Anthropic의 네덜란드 미술관 사례가 이를 증명.

3. **하네스는 고정되면 안 된다.** Anthropic: "하네스의 모든 구성요소는 모델 한계에 대한 가정이다. 그 가정은 모델이 발전하면 깨진다." 현재 남은 Gap 6(모델 적응)이 이 원칙을 구현하는 것. 흥미로운 하네스 조합의 공간은 줄어들지 않는다 — 이동할 뿐이다.

---

## 레퍼런스

- [Anthropic - Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) (2026.03.24)
- [Anthropic - Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Harness Engineering 연구 노트 (기존 문서)](../Harness%20Engineering%20연구%20노트/harness-engineering_2026-03-05.md) — Ralph Loop + Open/Closed-loop 실험
- [LinkedIn 해설 — Seungpil Lee](https://www.linkedin.com/pulse/%ED%95%98%EB%84%A4%EC%8A%A4-%EC%97%94%EC%A7%80%EB%8B%88%EC%96%B4%EB%A7%81%EC%9D%B4-%EB%AD%94%EB%8D%B0-anthropic%EC%9D%B4-%EC%A7%81%EC%A0%91-%EA%B3%B5%EA%B0%9C%ED%95%9C-ai-%EC%84%B1%EB%8A%A5%EC%9D%98-%EC%A7%84%EC%A7%9C-%EB%B9%84%EB%B0%80-seungpil-lee-xnnmc)
