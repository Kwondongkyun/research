# .claude 커스터마이징 완전 가이드 — 디렉토리 구조부터 실전 패턴까지

> 최종 수정: 2026-04-10

## TL;DR

Claude Code의 `.claude` 디렉토리는 **7개 핵심 영역**(CLAUDE.md, skills, agents, rules, hooks, settings, memory)으로 구성된다. 글로벌(`~/.claude/`)과 프로젝트(`.claude/`)로 나뉘며, 조직 정책 > 프로젝트 > 개인 > 로컬 순으로 로드된다. rules는 "자동 적용되는 규칙", skills는 "호출해서 쓰는 워크플로우"로, 용도가 명확히 다르다. 실전에서는 이 둘을 조합해 팀 컨벤션(rules)과 반복 작업 자동화(skills)를 동시에 달성한다.

---

## Part 1: 전체 디렉토리 구조

### 1.1 두 개의 .claude

`.claude`는 **두 가지 수준**에서 존재한다.

| 위치 | 범위 | Git 공유 | 용도 |
|------|------|---------|------|
| `~/.claude/` | 전역 (모든 프로젝트) | X | 개인 설정, 글로벌 스킬, 자동 메모리 |
| `<project>/.claude/` | 프로젝트 단위 | O | 팀 공유 설정, 프로젝트 스킬, 훅 |

### 1.2 전체 트리

```
~/.claude/                              # 글로벌 (개인)
├── CLAUDE.md                           # 개인 글로벌 지침
├── settings.json                       # 글로벌 권한/훅
├── skills/                             # 개인 재사용 스킬
│   └── my-skill/SKILL.md
├── commands/                           # 글로벌 슬래시 커맨드 (레거시)
├── templates/                          # 템플릿 (비표준, 개인 관습)
└── projects/<hash>/                    # 자동 생성
    ├── memory/MEMORY.md                # 프로젝트별 자동 메모리
    └── sessions/*.jsonl                # 세션 기록

<project>/                              # 프로젝트 레벨 (팀 공유)
├── CLAUDE.md                           # 프로젝트 루트 지침
├── CLAUDE.local.md                     # 개인 로컬 (.gitignore)
└── .claude/
    ├── CLAUDE.md                       # 프로젝트 지침 (대안 위치)
    ├── settings.json                   # 프로젝트 권한/훅 (팀 공유)
    ├── settings.local.json             # 로컬 오버라이드 (.gitignore)
    ├── skills/<name>/SKILL.md          # 프로젝트 스킬
    ├── agents/<name>.md                # 서브에이전트 정의
    ├── rules/<name>.md                 # 경로 조건부 지침
    ├── hooks/hooks.json                # 이벤트 자동화
    ├── commands/<name>.md              # 슬래시 커맨드 (레거시 → skills로 이관 권장)
    ├── .mcp.json                       # MCP 서버 설정
    └── .lsp.json                       # LSP 서버 설정
```

### 1.3 로드 우선순위

```
조직 정책 (/etc/claude-code/CLAUDE.md)    ← 최고 (강제)
  ↓
프로젝트 (.claude/ + CLAUDE.md)            ← 팀 공유
  ↓
개인 (~/.claude/CLAUDE.md)                 ← 글로벌 선호
  ↓
로컬 (CLAUDE.local.md, settings.local.json) ← 개인 오버라이드
```

같은 이름의 스킬이 여러 수준에 존재하면 **높은 우선순위가 승리**한다.

---

## Part 2: 7개 핵심 구성 요소

### 2.1 CLAUDE.md — 지침 파일

모든 커스터마이징의 출발점. YAML frontmatter 없이 순수 마크다운으로 작성한다.

```markdown
# CLAUDE.md

## 빌드 명령어
- Build: npm run build
- Test: npm test

## 컨벤션
- 2-space 들여쓰기
- TypeScript strict mode

## 다른 파일 참조 (import)
@docs/guidelines.md
@~/.claude/shared-rules.md
```

**핵심 규칙:**
- 200줄 이하 권장 — 초과 시 `rules/`로 분산
- `@path` 문법으로 외부 파일 임포트 (최대 5 hop)
- 하위 디렉토리의 CLAUDE.md도 자동 발견 (해당 파일 작업 시 로드)

### 2.2 skills/ — 호출형 워크플로우

"절차가 있는 작업"을 정의한다. `/name`으로 수동 호출하거나, Claude가 description을 보고 자동 판단한다.

```yaml
# .claude/skills/commit/SKILL.md
---
name: commit
description: Stage and commit with conventional format
argument-hint: "[message]"
user-invocable: true
disable-model-invocation: false
allowed-tools: "Bash(git *) Read"
model: sonnet
---

1. git diff로 변경사항 확인
2. conventional commit 형식으로 메시지 작성
3. 커밋 실행

현재 diff: !`git diff --staged`
사용자 메시지: $ARGUMENTS
```

**frontmatter 주요 필드:**

| 필드 | 설명 | 기본값 |
|------|------|--------|
| `name` | 슬래시 커맨드명 | 디렉토리명 |
| `description` | 자동 호출 판단 기준 (250자 이내) | — |
| `argument-hint` | 자동완성 힌트 | — |
| `user-invocable` | 사용자가 `/name`으로 호출 가능 | true |
| `disable-model-invocation` | true면 수동 호출만 | false |
| `allowed-tools` | 권한 사전 승인 | — |
| `model` | 스킬 전용 모델 (sonnet, opus, haiku) | 부모 상속 |
| `effort` | low\|medium\|high\|max | — |
| `context: fork` | 서브에이전트에서 실행 | — |
| `agent` | 어떤 agent 타입 사용 | — |
| `paths` | 조건부 매칭 (rules처럼) | — |

**특수 기능:**
- `$ARGUMENTS`, `$0`, `$1` — 사용자 인자 접근
- `` !`shell command` `` — 전처리로 셸 명령 실행 결과 삽입
- `${CLAUDE_SKILL_DIR}` — 스킬 폴더 경로 동적 참조

**디렉토리 구조:**
```
skills/my-skill/
├── SKILL.md              # 필수
├── reference.md          # 선택: 상세 문서
├── examples/sample.md    # 선택: 예제
└── scripts/helper.sh     # 선택: 실행 스크립트
```

### 2.3 agents/ — 커스텀 서브에이전트

역할을 가진 전문 에이전트를 정의한다. skills에서 `agent` 필드로 연결하거나, Agent 도구에서 `subagent_type`으로 호출한다.

```yaml
# .claude/agents/architect.md
---
name: architect
description: Strategic Architecture Advisor (READ-ONLY)
model: claude-opus-4-6
tools: Read, Grep, Glob
disallowedTools: Write, Edit
color: green
---

# 아키텍처 분석 전문 에이전트
코드베이스를 분석하고 설계 제안을 합니다.
절대 코드를 직접 수정하지 않습니다.
```

**frontmatter 주요 필드:**

| 필드 | 설명 |
|------|------|
| `name` | 에이전트 식별자 |
| `description` | 언제 사용할지 설명 |
| `model` | 전용 모델 |
| `tools` | 허용 도구 화이트리스트 |
| `disallowedTools` | 금지 도구 (Write, Edit 등) |
| `permissionMode` | bypassPermissions, default 등 |

### 2.4 rules/ — 자동 적용 지침

**CLAUDE.md를 주제별로 쪼갠 것.** 조건부 컨텍스트 주입으로 불필요한 로드를 줄인다.

```yaml
# .claude/rules/frontend.md
---
paths:
  - "src/**/*.tsx"
  - "src/**/*.css"
---

# Frontend Rules
- Tailwind CSS만 사용. inline style 금지.
- 컴포넌트명 PascalCase
- Props는 TypeScript interface로 정의
```

**동작 원리:**
- `paths` **있음** → 해당 파일 작업 시에만 로드 (컨텍스트 절약)
- `paths` **없음** → 항상 로드 (글로벌 규칙)
- frontmatter에 `paths`만 존재 — skills처럼 복잡한 설정 없음

### 2.5 hooks/ — 이벤트 자동화

26개 이벤트에 대해 자동 실행할 액션을 정의한다.

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "npm run lint:fix",
        "statusMessage": "린트 자동 수정..."
      }]
    }],
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/security/block-dangerous-bash.sh"
      }]
    }]
  }
}
```

**4가지 훅 타입:**
- `command` — 셸 스크립트 실행
- `http` — 웹훅 POST 요청
- `prompt` — LLM에게 yes/no 판단 요청
- `agent` — 서브에이전트로 검증

**주요 이벤트 (26개 중 핵심):**
- `SessionStart` / `SessionEnd` — 세션 생명주기
- `PreToolUse` / `PostToolUse` — 도구 실행 전후 (차단 가능)
- `Stop` — Claude 응답 완료
- `SubagentStart` / `SubagentStop` — 서브에이전트 생명주기
- `PreCompact` / `PostCompact` — 컨텍스트 압축 전후

### 2.6 settings.json — 권한과 설정

```json
{
  "permissions": {
    "allow": ["Bash(npm *)", "Bash(git status)", "Read", "Write"],
    "deny": ["Bash(rm -rf *)", "Bash(git push --force*)"],
    "ask": ["Bash(curl *)"]
  },
  "hooks": { ... },
  "mcpServers": { ... },
  "autoMemoryEnabled": true
}
```

| 파일 | Git 공유 | 용도 |
|------|---------|------|
| `settings.json` | O | 팀 공유 설정 |
| `settings.local.json` | X (.gitignore) | 개인 오버라이드 |

### 2.7 memory/ — 자동 메모리

```
~/.claude/projects/<hash>/memory/
├── MEMORY.md              # 인덱스 (200줄/25KB 제한)
├── user_role.md           # 사용자 정보
├── feedback_testing.md    # 피드백 기록
└── project_auth.md        # 프로젝트 컨텍스트
```

세션 시작마다 `MEMORY.md`가 자동 로드된다. Claude가 학습한 내용을 자동 기록하고, `/memory` 커맨드로 수동 관리 가능.

---

## Part 3: rules vs skills 심층 비교

### 3.1 핵심 차이

| | **rules** | **skills** |
|---|---|---|
| **비유** | 교통법규 (알아서 적용) | 레시피 (꺼내서 따라함) |
| **파일** | `.claude/rules/name.md` | `.claude/skills/name/SKILL.md` |
| **로드** | 자동 (paths 매칭) | 호출 시 (`/name` 또는 Claude 판단) |
| **frontmatter** | `paths`만 | 10개+ 필드 |
| **슬래시 커맨드** | X | O (`/skill-name`) |
| **인자 전달** | X | O (`$ARGUMENTS`, `$0`) |
| **셸 명령 삽입** | X | O (`` !`cmd` ``) |
| **모델 지정** | X | O (`model: sonnet`) |
| **서브에이전트 연결** | X | O (`context: fork`, `agent:`) |
| **용도** | 컨벤션, 규칙, 제약 | 워크플로우, 절차, 자동화 |

### 3.2 판단 기준

**rules로 만들 것** — "~할 때는 항상 ~해라" (선언적)
- API 파일 수정 시 OpenAPI 스펙도 업데이트해라
- Python 코드는 type hint 필수
- 테스트에서 mock 대신 fixture 사용
- DB 마이그레이션 파일 직접 수정 금지

**skills로 만들 것** — "~를 해줘" (절차적)
- PR 생성 워크플로우
- 코드 리뷰 체크리스트
- 디버깅 프로세스
- 커밋 자동화
- 브레인스토밍 절차

### 3.3 조합 패턴

실전에서는 둘을 **같이 쓴다**:

```
.claude/
├── rules/
│   ├── api-conventions.md       # "API는 항상 이 패턴을 따라라"
│   └── testing-rules.md         # "테스트는 항상 이렇게 작성해라"
└── skills/
    ├── api-create/SKILL.md      # "/api-create → 새 API 엔드포인트 생성 절차"
    └── test-write/SKILL.md      # "/test-write → 테스트 작성 워크플로우"
```

rules가 **"무엇을 지켜야 하는지"** 정의하고, skills가 **"어떻게 수행하는지"** 정의한다. skills 실행 중에도 rules는 자동 적용되므로, 워크플로우 안에서 컨벤션이 자동으로 지켜진다.

---

## Part 4: 실전 레퍼런스

### 4.1 주요 오픈소스 사례

| 레포 | 특징 | 구조 핵심 |
|------|------|----------|
| **anthropics/claude-code** | 공식 레포 | plugins/ 시스템, 표준 agent frontmatter |
| **affaan-m/everything-claude-code** | 30+ 스킬 컬렉션 | skills/\*/SKILL.md + agents/openai.yaml |
| **Yeachan-Heo/oh-my-claudecode** | 16개 역할별 에이전트 | agents/\*.md, `disallowedTools`로 권한 제한 |
| **robertdanco/writ** | Spec-Driven 개발 | agents/ + commands/ 분리, `permissionMode` 활용 |
| **AliErcanOzgokce/claude-code-starter-kit** | 훅/보안 포함 종합 킷 | hooks/security/\*.sh, hooks/quality/\*.sh |
| **hesreallyhim/awesome-claude-code** | 큐레이션 리스트 | 카테고리별 리소스 정리 |

### 4.2 공식 에이전트 frontmatter 패턴

```yaml
# anthropics/claude-code의 code-architect.md
---
name: code-architect
description: Designs feature architectures by analyzing existing codebase patterns...
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch
model: sonnet
color: green
---
```

### 4.3 커뮤니티 에이전트 패턴 (oh-my-claudecode)

```yaml
# READ-ONLY 에이전트 (아키텍트)
---
name: architect
description: Strategic Architecture & Debugging Advisor (Opus, READ-ONLY)
model: claude-opus-4-6
level: 3
disallowedTools: Write, Edit
---
```

`disallowedTools`로 쓰기를 금지하여 **분석 전용** 에이전트를 만드는 패턴.

### 4.4 보안 훅 패턴 (claude-code-starter-kit)

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/security/block-dangerous-bash.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/security/scan-secrets-on-commit.sh"
      }]
    }]
  }
}
```

---

## Part 5: 설계 원칙과 팁

### 5.1 컨텍스트 예산 관리

Claude의 컨텍스트 윈도우는 유한하다. 모든 CLAUDE.md, rules, 메모리가 로드되면 작업 공간이 줄어든다.

**전략:**
- CLAUDE.md는 200줄 이하 → 초과 시 `rules/`로 분산
- rules에 `paths` 지정 → 필요할 때만 로드
- skills는 500줄 이하 → 초과 시 `reference.md`로 분리
- 메모리 인덱스(MEMORY.md)는 200줄 이하

### 5.2 팀 공유 vs 개인 설정 분리

```
# Git에 커밋 (팀 공유)
.claude/settings.json
.claude/skills/
.claude/agents/
.claude/rules/
CLAUDE.md

# .gitignore에 추가 (개인)
.claude/settings.local.json
CLAUDE.local.md
```

### 5.3 commands → skills 마이그레이션

`commands/`는 레거시. 같은 이름이면 skill이 우선하므로, 점진적으로 이관하면 된다.

```
# Before
.claude/commands/deploy.md

# After
.claude/skills/deploy/SKILL.md
```

### 5.4 플러그인 구조

외부 배포용 스킬/에이전트 패키지:

```
my-plugin/
├── .claude-plugin/plugin.json    # 메타데이터
├── skills/review/SKILL.md
├── agents/reviewer.md
├── hooks/hooks.json
└── settings.json
```

호출 시 네임스페이싱: `/my-plugin:review`

---

## 핵심 인사이트

1. **rules는 선언적, skills는 절차적** — "무엇을 지켜야 하는지"는 rules, "어떻게 수행하는지"는 skills. 둘을 조합하면 컨벤션이 자동으로 지켜지는 워크플로우를 만들 수 있다.
2. **컨텍스트 예산이 설계를 결정한다** — CLAUDE.md 200줄, rules의 `paths` 조건부 로드, skills의 `reference.md` 분리 등 모든 구조적 결정은 "유한한 컨텍스트를 어떻게 효율적으로 쓸 것인가"에서 나온다.
3. **공식 레포와 커뮤니티 패턴이 수렴하고 있다** — `disallowedTools`로 READ-ONLY 에이전트 만들기, `PreToolUse` 훅으로 보안 가드레일 세우기, `context: fork`로 서브에이전트 격리하기 등 실전 패턴이 표준화되고 있다.
