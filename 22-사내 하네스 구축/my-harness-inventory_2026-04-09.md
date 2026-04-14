# 내 하네스 현황 인벤토리 — 사내 하네스 구축 기초자료

> 최종 수정: 2026-04-09

## TL;DR

현재 Claude Code 하네스 구성요소 전수 조사. Agent 6개, Skill 43개, Command 7개, Hook 8개, Plugin 9개, MCP 2개. **전부 유저 레벨(`~/.claude/`)에 집중** — 프로젝트 레벨 설정은 CLAUDE.md 1개뿐. Rules, AGENTS.md 미사용.

---

## Part 1: 구성요소 전체 목록

### Agents (6개) — 전부 유저 레벨

| Agent | 모델 | 역할 | 위치 |
|-------|------|------|------|
| `pm` | Opus | 프로덕트 매니저 (PRD, RICE, 유저플로우, 에러시나리오) | `~/.claude/agents/` |
| `frontend` | Sonnet | Next.js + TypeScript 개발자 | `~/.claude/agents/` |
| `frontend-reviewer` | Sonnet | 코드 리뷰어 (버그/보안/성능/유지보수) | `~/.claude/agents/` |
| `frontend-test` | Opus | 테스트 작성 (TDD/E2E) | `~/.claude/agents/` |
| `iteration-evaluator` | Opus | Playwright 앱 검증 + 4축 점수 | `~/.claude/agents/` |
| `service-evaluator` | Opus | 서비스 종합 평가 (5명 심사위원) | `~/.claude/agents/` |

### Skills (43개) — 전부 유저 레벨

| 카테고리 | 수 | 목록 |
|---------|---|------|
| 프론트엔드 개발 | 11 | components, style, naming, structure, fundamentals, accessibility, design, form, axios, server-actions, tanstack-query |
| 프론트엔드 리뷰 | 4 | review-bugs, review-security, review-performance, review-maintainability |
| 프론트엔드 테스트 | 6 | unit-test, component-test, api-mock-test, e2e-test, visual-regression, tdd-workflow |
| 프론트엔드 에러 | 1 | error-handling |
| PM | 6 | requirements, user-flow, error-scenarios, information-architecture, ux-heuristics, dummy-dataset |
| 평가 | 6 | public, ux, tech, business, critic, playwright |
| 개인 도구 | 6 | context-sync, session-wrap, update-check, update-check-workspace, fetch-youtube, fetch-tweet |
| 학습/기타 | 3 | content-digest, ai-trend-collector, web-design-guidelines |

### Commands (7개) — 전부 유저 레벨

| 커맨드 | Phase | 용도 |
|--------|-------|------|
| `/commit` | 전체 | 커밋 메시지 자동 생성 |
| `/spec-review` | 2 | 스펙 심층 검토 (인터뷰 방식) |
| `/pre-mortem` | 2.5 | 개발 리스크 사전 분석 |
| `/test-scenarios` | 4.5 | 테스트 시나리오 생성 |
| `/sprint-contract` | 4.7 | Generator/Evaluator 합격 기준 합의 |
| `/pivot-check` | 7 | 점수 정체 감지 + 방향 전환 |
| `/learn-claude-code` | - | Claude Code 학습 가이드 |

### Hooks (8개) — 유저 레벨 (`settings.json`)

#### 훅 1: SessionStart → `init-context-files.sh`

**언제**: Claude Code 세션이 시작될 때 1번 실행.

**뭘 하나**: 작업 디렉토리에 `plan.md`, `progress.md`, `memory.md`, `findings.md` 4개 파일이 있는지 확인. 없으면 `~/.claude/templates/` 폴더의 빈 템플릿을 복사하고, 날짜(`{{DATE}}`)를 오늘 날짜로 바꿔서 생성.

**비유하면**: 출근하면 빈 노트를 책상 위에 올려놓는 것. 이미 노트가 있으면(이어 작업) 건드리지 않음.

**강제력**: 없음 (파일 생성만, 행동 제약 없음)

---

#### 훅 2: PreCompact → `pre-compact-marker.sh`

**언제**: Claude가 대화가 너무 길어져서 컨텍스트를 압축(compact)하기 직전.

**뭘 하나**: `/tmp/.claude-compacted` 파일을 생성. 이 파일은 "compact가 발생했다"는 표식.

**비유하면**: 메모리가 날아가기 직전에 "기억 잃음" 포스트잇을 붙이는 것.

**강제력**: 없음 (마커 생성만)

---

#### 훅 3: PostCompact → `post-compact-reminder.sh`

**언제**: compact가 완료된 직후 1번.

**뭘 하나**: 위에서 만든 `/tmp/.claude-compacted` 마커가 있으면 삭제하고, Claude에게 "compact가 발생했으니 plan.md, progress.md, memory.md, findings.md를 다시 읽고 컨텍스트를 복구하라"는 메시지를 주입.

**비유하면**: 기억을 잃은 사람에게 "네 노트 다시 읽어"라고 알려주는 것.

**강제력**: 리마인더 수준 (Claude가 읽긴 하지만 강제 차단은 아님)

---

#### 훅 4: UserPromptSubmit → `post-compact-reminder.sh` (같은 스크립트)

**언제**: 사용자가 프롬프트를 입력할 때마다.

**뭘 하나**: 훅 3과 동일한 스크립트. compact 마커가 남아있으면 리마인더 주입. 이미 처리됐으면 아무것도 안 함.

**왜 필요한가**: PostCompact 훅만으로는 타이밍 문제로 리마인더가 누락될 수 있음. 매 프롬프트마다 한 번 더 체크해서 확실히 복구.

**강제력**: 리마인더 수준

---

#### 훅 5: PostToolUse(Edit|Write) → `lint-fix.sh`

**언제**: Claude가 Edit 또는 Write 도구로 파일을 수정/생성한 직후.

**뭘 하나**:
1. 수정된 파일이 `.ts/.tsx/.js/.jsx/.mjs`인지 확인 (아니면 무시)
2. 파일 위치에서 가장 가까운 `package.json`을 찾아 프로젝트 루트 결정
3. `npx eslint --fix`로 린트 자동 수정

**비유하면**: 개발자가 코드 저장할 때마다 자동으로 포맷팅하는 IDE 설정. Claude가 쓴 코드도 똑같이 린트를 돌림.

**강제력**: 실질적 강제 (코드가 자동으로 수정됨, Claude 의사와 무관)

---

#### 훅 6: PostToolUse(Skill|Agent) → `log-event.sh`

**언제**: Skill이나 Agent 도구가 호출된 직후 (async = 백그라운드 실행).

**뭘 하나**: 어떤 Skill/Agent가 언제 호출됐는지 로깅.

**현재 상태**: 스크립트 파일이 존재하지 않음 (설정만 있고 구현 안 됨).

**강제력**: 없음

---

#### 훅 7: StopFailure → `mark-crash.sh` + `notify.sh`

**언제**: Claude가 API 에러 등으로 비정상 중단됐을 때.

**뭘 하나**:
1. `mark-crash.sh`: 크래시 발생을 기록 (async)
2. `notify.sh`: macOS 알림으로 "Claude Code 중단 — API 에러로 작업이 멈췄습니다" 전송

**비유하면**: 서버 다운됐을 때 Slack 알림 보내는 것. 자리 비운 사이 Claude가 죽었으면 알림으로 알 수 있음.

**현재 상태**: 두 스크립트 모두 존재하지 않음 (설정만 있고 구현 안 됨).

**강제력**: 없음 (알림만)

---

#### 훅 8: ConfigChange → `validate-harness.sh`

**언제**: `settings.json` 등 설정 파일이 변경됐을 때.

**뭘 하나**: 하네스 구성요소(훅, 에이전트, 스킬 등)가 정상인지 건강 점검.

**현재 상태**: 스크립트 파일이 존재하지 않음 (설정만 있고 구현 안 됨).

**강제력**: 없음

---

### 훅 강제력 요약

| 훅 | 강제 수준 | 실제 작동 |
|----|----------|----------|
| SessionStart → init-context-files | 파일 생성 | ✅ 작동 |
| PreCompact → pre-compact-marker | 마커 생성 | ✅ 작동 |
| PostCompact → post-compact-reminder | 리마인더 | ✅ 작동 |
| UserPromptSubmit → post-compact-reminder | 리마인더 | ✅ 작동 |
| PostToolUse(Edit\|Write) → lint-fix | **자동 수정** | ✅ 작동 |
| PostToolUse(Skill\|Agent) → log-event | 로깅 | ❌ 스크립트 없음 |
| StopFailure → mark-crash + notify | 알림 | ❌ 스크립트 없음 |
| ConfigChange → validate-harness | 건강 점검 | ❌ 스크립트 없음 |

**8개 훅 중 실제 작동하는 건 5개.** 나머지 3개는 settings.json에 등록만 돼있고 스크립트가 없음.

**코드로 "차단"하는 훅은 0개.** 전부 리마인더/자동수정/로깅 수준. PreToolUse 훅(차단 가능)은 아직 안 쓰고 있음.

---

### Scripts 상세 (보조 스크립트 포함)

| 스크립트 | 용도 | 훅 연결 |
|---------|------|---------|
| `init-context-files.sh` | 4개 컨텍스트 파일 없으면 템플릿에서 복사 | SessionStart |
| `pre-compact-marker.sh` | `/tmp/.claude-compacted` 마커 생성 | PreCompact |
| `post-compact-reminder.sh` | compact 감지 시 Claude에게 "파일 다시 읽어라" 메시지 주입 | PostCompact, UserPromptSubmit |
| `lint-fix.sh` | JS/TS 파일 수정 후 ESLint --fix 자동 실행 | PostToolUse(Edit\|Write) |
| `eval-score-log.sh` | evaluator 점수를 `score-history.jsonl`에 JSONL로 기록 (가중 평균 계산) | 직접 호출 (훅 아님) |
| `stall-detector.sh` | `score-history.jsonl` 최근 3회 점수 비교 → PROGRESSING/STALL/REGRESSING 판정 | 직접 호출 (훅 아님) |
| `context-bar.sh` | 상태바: 모델명, 디렉토리, git 브랜치, 컨텍스트 사용률 바, 마지막 사용자 메시지 표시 | statusLine (훅 아님) |

### StatusLine — `statusline.sh`

훅은 아니지만 하네스의 일부. Claude Code 하단에 항상 표시되는 상태바.

**표시 내용**:
- 모델명 + 작업 디렉토리 + git 브랜치
- 컨텍스트 사용률 (██░░░░░░░░ 형태 바 + 퍼센트)
- 누적 비용 ($X.XX)
- 세션 소요 시간
- Rate limit 사용률 (5시간/7일 윈도우)

### Templates (7개) — 유저 레벨

| 템플릿 | 용도 |
|--------|------|
| `plan.md` | 목표, 접근 방식, 작업 항목, 결정사항 |
| `progress.md` | 완료/진행중/남은작업/블로커 |
| `memory.md` | 프로젝트 컨텍스트, 주요 결정, 주의사항 |
| `findings.md` | 발견사항, 주의할 점, 참고 링크 |
| `handoff.md` | 세션 간 인수인계 (4개 파일 포인터 포함) |
| `contract.md` | Sprint Contract (합격 기준 합의) |
| `research.md` | 연구 문서 표준 형식 |

---

## Part 2: CLAUDE.md 활용 현황

### 유저 레벨 (글로벌) — `~/.claude/CLAUDE.md`

모든 프로젝트, 모든 세션에서 항상 로딩되는 글로벌 지시 파일.

#### 섹션 1: Planning

```
- 비자명한 작업(새 기능, 다중 파일 변경, 아키텍처 결정) 전에 간단한 계획을 먼저 제안하라.
- 자명한 작업(오타, 한 줄 수정)은 바로 실행.
- 합리적인 접근법이 여러 개면 트레이드오프와 함께 제시. 혼자 고르지 마라.
```

**효과**: Claude가 복잡한 작업을 시작하기 전에 반드시 사용자 확인을 거치게 함. "몰래 진행"하는 것을 방지.

#### 섹션 2: Communication

```
- 직설적이고 구체적으로. 기술적 추천에서 애매하게 말하지 마라.
- 불확실하면 그렇다고 말하고 질문하라 — 추측 금지.
- 추천할 때: 무엇을, 왜, 뭐가 잘못될 수 있는지 명시.
```

**효과**: Claude의 기본 성향(과잉 공손, 애매한 표현)을 억제. "~할 수도 있겠네요" 대신 "A를 하세요. 이유: B. 리스크: C" 형태로 응답하도록 유도.

#### 섹션 3: 프로젝트 생성 프로세스 (Phase 0~8)

```
Phase 0:   브레인스토밍 → brainstorming 스킬
Phase 1:   기획 → pm 에이전트 (PRD + RICE 우선순위, 유저플로우, 에러시나리오)
Phase 2:   기획 검증 → /spec-review 커맨드
Phase 2.5: 리스크 분석 → /pre-mortem 커맨드
Phase 3:   UI 설계 → Pencil MCP
Phase 4:   구현 계획 → writing-plans 스킬
Phase 4.5: 테스트 계획 → /test-scenarios 커맨드
Phase 4.7: Sprint Contract → /sprint-contract 커맨드
Phase 5:   Foundation → Lead 직접 (프레임워크, DB, 인증, 더미데이터)
Phase 6:   병렬 개발 → 팀 에이전트 (worktree, 규모 무관 항상 분기)
Phase 7:   검증 루프 → evaluator + handoff + pivot + reviewer + test
Phase 8:   기록 → /wrap + handoff + score-history
```

**효과**: 프로젝트를 처음부터 끝까지 어떤 도구로 어떤 순서로 진행할지 **매핑**. 각 Phase가 어떤 agent/skill/command를 호출해야 하는지 1:1 대응.

**제약 규칙**:
- 각 Phase는 독립적. Phase 전환 시 반드시 사용자에게 확인.
- Phase 간 자동 호출 금지 (brainstorming이 끝났다고 자동으로 writing-plans로 넘어가지 않음).

**현재 한계**: Phase 매핑은 문서에 적혀있을 뿐, **코드로 강제되지 않음**. Phase 2를 건너뛰고 Phase 5를 해도 아무 에러가 나지 않는다.

#### 섹션 4: 컨텍스트 관리

```
세션 시작 시:
  handoff.md 읽기 (최우선) → 안의 링크를 따라 plan/progress/memory/findings.md 읽기

파일별 업데이트 트리거:
  progress.md — 코드 수정 시작 전 "진행 중" 기록, 완료 시 이동, 블로커 즉시 추가
  plan.md    — 접근 방식 변경 시, 작업 완료 시 체크, 새 결정사항 추가
  findings.md — 예상과 다른 동작 발견 시 즉시, 외부 자료 링크 추가
  memory.md  — 프로젝트 방향이 바뀌는 중요 결정 시만 (자주 안 씀)

compact 복구 리마인더가 오면 위 파일들 반드시 다시 읽기.
```

**효과**: Claude가 언제 어떤 파일을 업데이트해야 하는지 구체적 트리거 제공. "계획 변경 시 업데이트" 같은 모호한 지시 대신 "Edit/Write 도구 사용 전에 progress.md 기록"처럼 행동 시점을 특정.

**현재 한계**: CLAUDE.md 지시일 뿐, 강제가 아님. Claude가 바쁘면 업데이트를 건너뛸 수 있음.

#### 섹션 5: Defaults

```
- 새 파일 생성보다 기존 파일 편집 우선.
- 문서 파일(README 등) 자동 생성 금지.
- 3회 실패 시 접근 방식 재평가.
```

**효과**: Claude의 기본 성향(파일 과다 생성, README 자동 작성)을 억제.

---

### 프로젝트 레벨 — `~/CLAUDE.md` (research 폴더 전용)

research 레포에서 작업할 때만 추가 로딩되는 프로젝트별 지시 파일. 유저 글로벌 CLAUDE.md 위에 덧씌워진다.

#### 섹션 1: 연구 문서 작성 규칙

```
- 반드시 ~/.claude/templates/research.md 템플릿 형식 사용
- 구조: # 제목 — 부제 → > 최종 수정: YYYY-MM-DD → ## TL;DR → ## Part N: → ## 핵심 인사이트
- 폴더명 = 주제명 (한글 그대로)
- 파일명 컨벤션: slug_YYYY-MM-DD.md
```

**효과**: 연구 문서의 형식을 통일. 어떤 세션에서 쓰든 동일한 구조가 나오도록 강제.

#### 섹션 2: Context Reset (Handoff)

```
- Ralph Loop 사용 시, 프롬프트에 반드시 handoff.md 읽기/쓰기 포함
- 세션 시작 시 handoff.md가 있으면 plan.md/progress.md보다 우선 읽기
- handoff.md는 매 iteration 종료 시 업데이트
```

**효과**: 반복 루프(Ralph Loop)에서 iteration 간 컨텍스트가 유실되지 않도록 handoff 규칙 강제. 글로벌 CLAUDE.md의 컨텍스트 관리를 research 프로젝트에 맞게 보강.

#### 섹션 3: Obsidian 동기화

```
- 연구 문서는 로컬 /Users/kwondong-kyun/Desktop/Claude-Code/research에 저장
- 해당 폴더는 Obsidian vault로 심볼릭 링크됨
- 로컬 md가 원본. 수정하면 Obsidian에 자동 반영. 별도 동기화 불필요.
```

**효과**: Claude가 연구 문서를 엉뚱한 경로에 저장하지 않도록 경로 지정. Obsidian 연동 구조를 명시하여 "파일을 어디에 넣을까요?" 질문을 제거.

---

### 미사용 항목

| 항목 | 상태 | 사내 하네스에서의 가능성 |
|------|------|----------------------|
| `AGENTS.md` | 미생성 | 프로젝트 루트에 두면 팀 전체가 공유하는 agent 정의 가능 |
| `.claude/rules/` | 빈 디렉토리 | agent별 행동 규칙을 파일 단위로 강제 가능 (예: `no-skip-phase.md`) |

---

## Part 3: Plugins & MCP

### Plugins (활성 9개)

| Plugin | 용도 |
|--------|------|
| `superpowers` | 계획 수립, 브레인스토밍, 병렬 에이전트, TDD, 코드리뷰 |
| `ralph-loop` | Closed-loop 반복 실행 |
| `code-review` | PR 코드 리뷰 |
| `context7` | 라이브러리 최신 문서 조회 |
| `github` | GitHub 연동 |
| `telegram` | Telegram 채널 연동 |
| `discord` | Discord 채널 연동 |
| `skill-creator` | 스킬 생성/수정 |
| `security-guidance` | 보안 가이드 |

### MCP 서버 (2개)

| 서버 | 용도 |
|------|------|
| `playwright` | E2E 브라우저 테스트 (Evaluator가 사용) |
| `pencil` | UI 와이어프레임 설계 (Phase 3) |

---

## Part 4: 레벨 분포 요약

| 구성요소 | 유저 레벨 (`~/.claude/`) | 프로젝트 레벨 (레포 내) |
|---------|------------------------|----------------------|
| Agents | 6개 | 0 |
| Skills | 43개 | 0 |
| Commands | 7개 | 0 |
| Hooks | 8개 | 0 |
| Scripts | 7개 | 0 |
| Templates | 7개 | 0 |
| CLAUDE.md | 1개 (글로벌) | 1개 (research) |
| AGENTS.md | 0 | 0 |
| Rules | 0 | 0 |
| Plugins | 9개 | 0 |
| MCP | 2개 | 0 |

**전부 유저 레벨에 집중.** 프로젝트 레벨 커스터마이징은 `CLAUDE.md` 하나뿐.

---

## Part 5: 사내 하네스 구축 시 시사점

1. **프로젝트 레벨 설정이 비어있다** — 현재 구조는 "내 머신에서만 작동". 팀원이 같은 레포를 클론해도 하네스가 따라오지 않음.
2. **Rules 미사용** — `.claude/rules/`는 agent별 행동 규칙을 강제할 수 있는 채널인데 활용 안 됨.
3. **AGENTS.md 미사용** — 프로젝트 루트에 agent 정의를 공유할 수 있는 메커니즘이 빠져있음.
4. **코드 강제 부재** — Phase gate, artifact 검증, 역할 매핑이 모두 텍스트 권고 수준.
