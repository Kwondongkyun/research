# Claude Code 컨텍스트 관리

*최종 수정: 2026-03-17*

## TL;DR

Claude Code는 `/compact`나 세션 전환 시 컨텍스트를 잃는다. 공식 방법(`--resume`, CLAUDE.md, Auto Memory)이 있지만 세션 시스템에 의존적이다. **파일 기반 접근**(plan.md, progress.md, memory.md, findings.md)을 SessionStart 훅으로 자동화하면, 도구에 비의존적이면서 git으로 버전 관리까지 되는 견고한 컨텍스트 관리 시스템을 만들 수 있다.

---

## Part 1: 문제 — 왜 컨텍스트가 날아가는가

### 1. compact의 동작 원리

`/compact`는 대화 내용을 요약하여 새 컨텍스트의 기초로 로드한다. 자동 압축(auto-compact)은 컨텍스트 사용률이 ~95%에 도달하면 실행된다.

```
작업 중 (토큰 95% 도달)
→ 벌크한 도구 출력 제거
→ 구조화된 "작업 상태"로 요약
→ 최근 파일, TODO, 계속 지시만 재수화
→ 세부 컨텍스트 손실
```

### 2. 세션 전환의 한계

새 세션을 시작하면 이전 대화 컨텍스트가 완전히 리셋된다. `--resume`이나 `--continue`로 복원할 수 있지만, 세션 데이터가 손상되면 복구 불가.

### 3. 핵심 문제

대화로만 준 지시사항은 compact 후 **손실**된다. CLAUDE.md에 적은 것만 **생존**한다.

---

## Part 2: 공식 방법 — Claude Code가 제공하는 것

### 1. 세션 관리

| 기능 | 설명 |
|------|------|
| `claude --continue` | 가장 최근 세션 이어가기 |
| `claude --resume` | 세션 선택 UI로 특정 세션 재개 |
| `/compact focus on X` | 압축 시 특정 주제만 보존 지시 |

### 2. 영구 저장소

| 기능 | 설명 |
|------|------|
| CLAUDE.md | 매 세션 자동 로드, compact 후에도 생존 |
| Auto Memory (MEMORY.md) | 200줄 자동 로드 |
| `.claude/rules/` | 경로별 조건부 규칙 |

### 3. 훅

| 기능 | 설명 |
|------|------|
| PreCompact 훅 | 압축 직전 커스텀 명령 실행 |
| PostCompact 훅 | 압축 후 커스텀 명령 실행 |

---

## Part 3: 실전 방법 — 커뮤니티는 어떻게 하는가

### 1. Dev Docs 메서드

`dev/` 디렉토리에 plan.md, context.md, tasks.md를 관리. CLAUDE.md에 "continue라고 하면 dev/ 파일들을 읽어라" 지시를 넣어둔다. 30분 이상 걸리는 작업에 권장.

### 2. PostCompact 리마인더 훅

compact 후 Claude가 규칙을 잊는 "post-compaction amnesia" 방지. PreCompact에서 마커 생성 → UserPromptSubmit에서 감지 → "CLAUDE.md 다시 읽어" 리마인더 주입.

### 3. Document & Clear 패턴

Claude에게 현재 상태를 .md에 덤프 → `/clear` → 새 세션에서 "이 파일 읽고 이어가". 가장 단순하지만 효과적.

### 4. PreCompact 훅으로 git diff 백업

compact 직전 `git diff`를 파일로 저장하는 안전장치.

### 5. 서브에이전트 위임

무거운 작업을 서브에이전트에 분리 → 메인 컨텍스트 윈도우 절약. 최대 10개 동시 실행 가능.

### 6. BMAD Method

Claude Code를 역할별 에이전트 오케스트라로 변환. 파일 자체에 전체 컨텍스트를 내장하여 세션 의존성 제거.

---

## Part 4: 내가 세팅한 구조 — 파일 기반 자동 컨텍스트 관리

### 1. 4개 파일 역할

| 파일 | 역할 | 내용 |
|------|------|------|
| **plan.md** | 뭘 할 건지 (계획) | 목표, 접근 방식, 작업 항목. 전체 청사진으로 바뀌지 않는 한 고정 |
| **progress.md** | 어디까지 했는지 (진행) | plan.md의 어디를 지나고 있는지. 계속 변함 |
| **memory.md** | 뭘 기억해야 하는지 (결정) | 주요 결정사항, 주의사항, 프로젝트 컨텍스트 |
| **findings.md** | 뭘 발견했는지 (조사) | 작업 중 발견한 것, 참고 링크 |

### 2. 파일 구조

```
~/.claude/
├── settings.json
├── CLAUDE.md
├── notify.sh                  # macOS 네이티브 알림
├── scripts/
│   ├── init-context-files.sh      # SessionStart: 4개 파일 자동 생성
│   ├── pre-compact-marker.sh      # PreCompact: 마커 생성
│   └── post-compact-reminder.sh   # UserPromptSubmit: 리마인더 주입
└── templates/
    ├── plan.md
    ├── progress.md
    ├── memory.md
    └── findings.md
```

### 3. 훅 구성

| 훅 | 이벤트 | 동작 |
|------|------|------|
| SessionStart | claude 실행 시 | 4개 파일 없으면 템플릿에서 생성 (날짜 주입) |
| PreCompact | compact 직전 | 마커 생성 |
| UserPromptSubmit | 메시지 입력 시 | 마커 있으면 "4개 파일 읽어" 리마인더 주입 |
| Stop | Claude 멈출 때 | macOS 알림 |
| Notification | 알림 발생 시 | macOS 알림 |
| PreToolUse | AskUserQuestion 시 | macOS 알림 |

### 4. CLAUDE.md에 추가한 컨텍스트 관리 규칙

```
- 세션 시작 시 plan.md, progress.md, memory.md, findings.md가 있으면 읽고 현재 상황을 파악한다.
- 작업 중 계획 변경 시 plan.md, 진행 상태 변경 시 progress.md, 중요 결정 시 memory.md, 조사/발견 시 findings.md를 업데이트한다.
- compact 복구 리마인더가 오면 위 파일들을 반드시 읽는다.
```

### 5. MEMORY.md vs memory.md

| | MEMORY.md | memory.md |
|------|------|------|
| 위치 | `~/.claude/projects/*/memory/` | 프로젝트 루트 |
| 관리 | Claude Auto Memory (자동) | 사용자/Claude (수동) |
| 로드 | 매 세션 200줄 자동 로드 | CLAUDE.md 지시로 읽기 |
| 용도 | 세션 간 영구 기억 (장기) | 현재 작업 컨텍스트 (단기~중기) |

### 6. planning-with-files 플러그인과의 관계

planning-with-files는 task_plan.md, findings.md, progress.md를 사용하는 유사한 접근. 차이는 스킬 수동 호출 필요 vs 우리 세팅은 SessionStart 훅으로 자동. 역할 중복이므로 비활성화하고 우리 세팅으로 대체.

---

## Part 5: 전체 흐름

```
claude 실행
│
├── 1. CLAUDE.md 자동 로드
├── 2. Auto Memory (MEMORY.md) 자동 로드
├── 3. SessionStart 훅 → 4개 파일 자동 생성 (없을 때만)
├── 4. CLAUDE.md 지시로 4개 파일 자동 읽기 → 상황 파악
│
├── 5. 작업 시작
│   ├── [기존 프로젝트] → 파일 읽고 이어서 작업
│   └── [새 프로젝트] → 프로젝트 생성 프로세스 (Phase 0~8)
│
├── 6. 작업 중
│   ├── 4개 파일 수시 업데이트 + 커밋
│   └── Claude 멈춤 → macOS 알림
│
├── 7. compact 발생 시
│   ├── PreCompact → 마커 생성
│   ├── CLAUDE.md 자동 재로드
│   └── 다음 입력 시 → 리마인더 → 4개 파일 읽기 → 복구
│
└── 8. 새 세션 → 1번으로 돌아감
```

---

## Part 6: 공식 vs 파일 기반 — 비교

| 기준 | 공식 | 파일 기반 |
|------|------|----------|
| 컨텍스트 복구 | `--resume`으로 세션 통째로 복원 | 파일 읽기로 복구 |
| 환경 의존성 | Claude Code 세션 시스템에 의존 | 파일 시스템만 있으면 됨 |
| 버전 관리 | 세션 기록은 git에 안 들어감 | 커밋으로 히스토리 남음 |
| 복구 신뢰도 | 세션 데이터 손상 시 복구 불가 | 커밋만 있으면 항상 복구 |
| 자동화 | `--continue`만 자동 | 훅으로 전체 자동화 |
| 팀 공유 | CLAUDE.md git 공유 가능 | 4개 파일도 git 공유 가능 |

---

## 핵심 인사이트

1. **공식 기능은 도구에 의존적** — 세션 시스템이 망가지면 복구 불가
2. **파일 기반은 도구에 비의존적** — 파일 + git만 있으면 어디서든 복구
3. **CLAUDE.md는 compact를 생존** — 여기에 규칙을 넣으면 영구적
4. **훅으로 자동화** — 수동 개입 없이 전체 흐름이 동작
5. **기존 도구와 역할 겹침 주의** — planning-with-files 비활성화하고 하나로 통일
6. **stale 컨텍스트는 없는 것보다 나쁨** — 파일 업데이트를 까먹으면 오히려 잘못된 판단 유발
