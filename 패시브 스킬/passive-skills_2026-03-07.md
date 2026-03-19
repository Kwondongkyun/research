# 패시브 스킬 — Hook으로 만드는 자동 품질 게이트

> 최종 수정: 2026-03-07

## TL;DR

Claude Code의 **Skill은 액티브 스킬**이고, **Hook은 패시브 스킬**이다. Skill은 "파이어볼"처럼 필요할 때 시전해야 하지만, Hook은 "방어력 +10"처럼 **한 번 설정하면 매 세션마다 알아서 발동**된다. `claude` 명령어로 세션을 시작하는 순간, 미리 설정된 Hook들이 자동으로 활성화되어 포맷팅, 보안, 테스트, 알림을 백그라운드에서 처리한다.

가설: **Hook을 패시브 스킬처럼 구성하면, 사용자가 의식하지 않아도 코드 품질과 보안이 세션 내내 자동으로 유지된다.**

---

## Part 1: 개념

---

### 1. 액티브 vs 패시브 — Skill과 Hook의 본질적 차이

#### 게임에서의 비유

RPG 게임에서 스킬은 두 종류다:

```
액티브 스킬                         패시브 스킬
───────────────────                ───────────────────
"파이어볼" — 직접 시전              "철벽 방어" — 항상 적용
MP 소모, 쿨다운 있음                자원 소모 없음
상황 판단 후 사용                   의식할 필요 없음
강력하지만 잊으면 못 씀             약하지만 항상 작동
```

Claude Code도 마찬가지다:

| | 액티브 (Skill) | 패시브 (Hook) |
|--|----------------|---------------|
| **발동 조건** | Claude가 매칭하거나 사용자가 호출 | **이벤트 발생 시 자동** |
| **의식 필요** | "이 스킬 써야지" 인지 필요 | **없음. 알아서 발동** |
| **동작 방식** | 프롬프트 확장 (컨텍스트 주입) | 셸 명령/LLM 판단/에이전트 실행 |
| **비용** | 토큰 소모 (컨텍스트 차지) | command 타입은 토큰 0 |
| **설정** | SKILL.md 작성 | settings.json에 등록 |
| **예시** | `/commit`, `/review`, `frontend-design` | 자동 포맷팅, 파일 보호, 알림 |

#### 핵심 차이

**Skill은 "뭘 할지"를 알려주고, Hook은 "뭘 하면 안 되는지/항상 해야 하는 것"을 강제한다.**

```
Skill: "커밋 메시지를 한국어로 쓰고, Co-Authored-By를 넣어라"
  → Claude가 기억하고 따를 수도 있고, 잊을 수도 있다

Hook: "Edit/Write 후 자동으로 Prettier 실행"
  → Claude가 뭘 하든 상관없이, 코드는 항상 포맷팅됨
```

Skill은 **LLM의 판단에 의존**한다. Hook은 **결정론적으로 강제**한다. 둘 다 필요하지만, **품질 게이트는 Hook이 더 적합**하다 — 잊을 수가 없으니까.

---

### 2. Hook의 내부 동작 — 18개 이벤트, 4가지 handler

#### 라이프사이클 이벤트

Hook은 Claude Code의 **라이프사이클 특정 지점에서 자동 실행**되는 사용자 정의 명령이다. 18개의 이벤트가 있다:

```
세션 시작 ─────────────────────────────────────── 세션 종료
  │                                                  │
  SessionStart                                   SessionEnd
  │                                                  │
  ├── UserPromptSubmit (사용자 입력 시)
  │
  ├── PreToolUse (도구 실행 전) ← 유일하게 차단 가능!
  │   ├── PermissionRequest (권한 요청 시)
  │   ├── PostToolUse (도구 실행 후)
  │   └── PostToolUseFailure (도구 실패 후)
  │
  ├── SubagentStart / SubagentStop (서브에이전트)
  ├── TeammateIdle / TaskCompleted (팀)
  │
  ├── Notification (알림 발생 시)
  ├── ConfigChange (설정 변경 시)
  ├── InstructionsLoaded (CLAUDE.md 로드 시)
  │
  ├── PreCompact (컴팩션 전)
  ├── WorktreeCreate / WorktreeRemove (워크트리)
  │
  └── Stop (Claude 응답 완료 시)
```

#### 4가지 handler 타입

| 타입 | 동작 | 용도 | 토큰 비용 |
|------|------|------|----------|
| **command** | 셸 명령 실행 | 포맷팅, 파일 보호, 로깅 | 0 |
| **prompt** | Claude 모델에 yes/no 판단 요청 | 의미적 판단 (코드 품질 검토) | 낮음 (Haiku) |
| **agent** | 서브에이전트 스폰 (파일 읽기, 검색 가능) | 복잡한 검증 (테스트 실행 등) | 중간 |
| **http** | HTTP POST로 외부 서비스 호출 | 감사 로깅, 팀 공유 서비스 | 0 |

#### 게임 패시브 스킬과의 매핑

```
게임                              Claude Code
─────────────────                ─────────────────
장착 슬롯                         settings.json
캐릭터 생성 시 적용               세션 시작 시 적용
특정 조건에서 자동 발동            이벤트 + matcher로 자동 발동
중첩 가능 (여러 패시브)            병렬 실행 (여러 hook)
```

---

### 3. 패시브 스킬로서의 Hook — 구성 원리

#### 설정 위치

```bash
# 전역 패시브 (모든 프로젝트에 적용)
~/.claude/settings.json

# 프로젝트 패시브 (해당 프로젝트만)
.claude/settings.json       ← git에 커밋 가능 (팀 공유)
.claude/settings.local.json ← gitignore (개인 설정)
```

#### 기본 구조

```json
{
  "hooks": {
    "이벤트명": [
      {
        "matcher": "필터 패턴 (regex)",
        "hooks": [
          {
            "type": "command|prompt|agent|http",
            "command": "실행할 명령"
          }
        ]
      }
    ]
  }
}
```

#### 패시브 스킬의 3계층

```
계층 1: 결정론적 강제 (command)
  → "Prettier 실행", ".env 파일 수정 차단"
  → 판단 불필요, 항상 같은 결과
  → 토큰 0, 가장 빠름

계층 2: 의미적 판단 (prompt)
  → "이 코드 변경이 보안 규칙을 위반하는가?"
  → Haiku 모델이 yes/no 판단
  → 토큰 소량, 빠름

계층 3: 심층 검증 (agent)
  → "테스트 스위트를 실행하고 결과를 검증하라"
  → 서브에이전트가 파일 읽기, 명령 실행 가능
  → 토큰 중간, 느림 (최대 60초)
```

> **원칙: 가장 낮은 계층으로 해결할 수 있으면 그걸 쓴다.** command로 되는 걸 agent로 하면 낭비다.

---

## Part 2: 실전 패시브 스킬 구성

---

### 4. 포맷팅 패시브 — 코드 스타일 자동 유지

**이벤트**: `PostToolUse` (파일 수정 후)
**계층**: command (결정론적)

Claude가 파일을 수정할 때마다 **자동으로 포맷터를 실행**한다. Claude가 포맷팅 규칙을 모르더라도, 결과물은 항상 프로젝트 컨벤션에 맞는다.

#### Prettier (JavaScript/TypeScript)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### Black (Python)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path'); [[ \"$FILE\" == *.py ]] && black \"$FILE\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### 게임 비유

```
🛡️ [패시브] 코드 정화
   "모든 코드 수정 시 자동으로 포맷팅 적용"
   발동 조건: Edit/Write 도구 사용 후
   효과: 코드 스타일 일관성 +100%
```

---

### 5. 보안 패시브 — 위험한 파일 수정 자동 차단

**이벤트**: `PreToolUse` (파일 수정 전)
**계층**: command (결정론적)

`.env`, `package-lock.json`, `.git/` 등 **민감한 파일을 Claude가 건드리지 못하게 차단**한다. `PreToolUse`는 **유일하게 동작을 차단할 수 있는 이벤트**다.

#### 보호 스크립트 (.claude/hooks/protect-files.sh)

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED_PATTERNS=(".env" "package-lock.json" "yarn.lock" ".git/" "credentials" "secrets")

for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2  # exit 2 = 차단. Claude에게 이유가 피드백됨
  fi
done

exit 0  # exit 0 = 허용
```

#### Hook 등록

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ]
  }
}
```

#### exit code 규칙

| exit code | 의미 | Claude에게 |
|-----------|------|-----------|
| **0** | 허용 — 진행 | stdout이 컨텍스트에 추가 |
| **2** | 차단 — 중지 | stderr가 피드백으로 전달 |
| **기타** | 허용 — 진행 | stderr는 로그에만 기록 |

#### 게임 비유

```
🛡️ [패시브] 성역 수호
   "보호된 파일(.env, lock 등) 수정 시도를 자동 차단"
   발동 조건: Edit/Write 도구 사용 전
   효과: 민감 파일 보호율 100%
```

---

### 6. 품질 패시브 — 테스트/타입체크 자동 실행

**이벤트**: `PostToolUse` (파일 수정 후) 또는 `Stop` (응답 완료 시)
**계층**: command 또는 agent

#### 타입 체크 (PostToolUse + command)

TypeScript 파일 수정 후 자동으로 타입 체크:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path'); [[ \"$FILE\" == *.ts || \"$FILE\" == *.tsx ]] && npx tsc --noEmit 2>&1 | head -20 || true"
          }
        ]
      }
    ]
  }
}
```

#### 작업 완료 시 테스트 검증 (Stop + agent)

Claude가 "끝났다"고 할 때, 에이전트가 **정말 끝난 건지 검증**:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "테스트 스위트를 실행하고 결과를 확인하라. 실패한 테스트가 있으면 {\"ok\": false, \"reason\": \"실패한 테스트 목록\"} 을 반환하라.",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

> **주의**: `Stop` hook은 `stop_hook_active` 필드를 체크해야 무한 루프를 방지할 수 있다.

```bash
#!/bin/bash
INPUT=$(cat)
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0  # 이미 Stop hook에 의해 재실행된 경우 → 그냥 종료
fi
# ... 검증 로직
```

#### 게임 비유

```
🛡️ [패시브] 품질 감시
   "코드 수정 시 타입 에러 자동 감지"
   발동 조건: TypeScript 파일 수정 후
   효과: 타입 안전성 유지

⚔️ [패시브] 최종 점검
   "작업 완료 선언 시 테스트 자동 실행"
   발동 조건: Claude 응답 완료 시
   효과: 테스트 통과 없이 끝날 수 없음
```

---

### 7. 알림 패시브 — 놓치지 않는 알림 시스템

**이벤트**: `Notification`
**계층**: command (결정론적)

Claude가 권한을 요청하거나 입력을 기다릴 때, **데스크톱 알림으로 즉시 알려준다.** 다른 작업을 하면서도 Claude를 놓치지 않는다.

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude가 권한을 요청합니다\" with title \"Claude Code\" sound name \"Ping\"'"
          }
        ]
      },
      {
        "matcher": "idle_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude가 입력을 기다립니다\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

#### matcher 옵션

| matcher | 발동 시점 |
|---------|----------|
| `permission_prompt` | Claude가 도구 권한을 요청할 때 |
| `idle_prompt` | Claude가 60초 이상 대기 중일 때 |
| `auth_success` | 인증 성공 시 |
| `elicitation_dialog` | MCP 도구가 입력을 필요로 할 때 |

#### 게임 비유

```
🔔 [패시브] 전투 알림
   "파티원(Claude)이 도움을 요청하면 알림"
   발동 조건: 권한 요청 또는 대기 상태
   효과: 반응 시간 -90%
```

---

### 8. 컨텍스트 패시브 — 컴팩션 후 기억 복구

**이벤트**: `SessionStart` (matcher: `compact`)
**계층**: command (결정론적)

Claude의 컨텍스트 윈도우가 가득 차면 **컴팩션**이 일어나 대화가 요약된다. 이때 중요한 컨텍스트를 잃을 수 있다. `SessionStart` hook의 `compact` matcher를 사용하면, **컴팩션 후 자동으로 핵심 정보를 재주입**한다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Reminder: bun 사용 (npm 아님). 커밋 전 bun test 필수. 현재 스프린트: 인증 리팩토링.'"
          }
        ]
      }
    ]
  }
}
```

stdout으로 출력한 텍스트가 **Claude의 컨텍스트에 자동 추가**된다. 동적 정보도 넣을 수 있다:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== 최근 커밋 ===' && git log --oneline -5 && echo '=== 현재 브랜치 ===' && git branch --show-current"
          }
        ]
      }
    ]
  }
}
```

#### 게임 비유

```
🛡️ [패시브] 기억 복원
   "기억이 초기화되면 핵심 정보를 자동 복구"
   발동 조건: 컴팩션 발생 시
   효과: 컨텍스트 손실 방지
```

---

## Part 3: 확장

---

### 9. 개인 → 팀 → 조직으로 확장하는 전략

패시브 스킬의 힘은 **한 번 검증하면 팀 전체에 자동 적용**할 수 있다는 점이다.

#### 3단계 확장 모델

```
개인 (Personal)
  │  ~/.claude/settings.json
  │  나만 적용, 자유롭게 실험
  │
  ├── 검증 완료 → 팀에 공유
  │
팀 (Team)
  │  .claude/settings.json (git commit)
  │  프로젝트 참여자 전원에게 적용
  │
  ├── 검증 완료 → 조직 정책으로
  │
조직 (Organization)
     Managed policy settings (관리자 제어)
     조직 전체 강제 적용
```

#### 각 단계별 Hook 예시

**개인**: 내 취향대로 실험

```json
// ~/.claude/settings.json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "idle_prompt",
        "hooks": [
          { "type": "command", "command": "afplay /System/Library/Sounds/Ping.aiff" }
        ]
      }
    ]
  }
}
```

**팀**: 프로젝트 컨벤션 강제

```json
// .claude/settings.json (git에 커밋)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null || true" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh" }
        ]
      }
    ]
  }
}
```

**조직**: 보안 정책 강제 (관리자만 변경 가능)

```
Managed policy settings:
  - .env 파일 수정 차단 (전사)
  - 보안 검사 Hook 필수 (모든 프로젝트)
  - 감사 로깅 (HTTP hook → 중앙 로그 서버)
```

#### HTTP Hook으로 팀 감사 시스템

모든 도구 사용 내역을 중앙 서비스에 자동 기록:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://audit.internal.company.com/hooks/tool-use",
            "headers": {
              "Authorization": "Bearer $AUDIT_TOKEN"
            },
            "allowedEnvVars": ["AUDIT_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

---

### 10. Best Practices + Anti-patterns

#### Best Practices

| 원칙 | 설명 |
|------|------|
| **최소 계층 원칙** | command로 되는 걸 prompt/agent로 하지 않는다 |
| **실패 안전** | Hook 스크립트가 실패해도 세션이 죽지 않게 `\|\| true` 또는 에러 핸들링 |
| **개인에서 시작** | `~/.claude/settings.json`에서 충분히 테스트 후 팀에 공유 |
| **matcher 활용** | 필요한 이벤트에만 정확히 걸기. 빈 matcher는 모든 이벤트에 발동 |
| **verbose 디버깅** | `Ctrl+O`로 hook 출력 확인. `claude --debug`로 상세 로그 |

#### Anti-patterns

| 패턴 | 문제 | 해결 |
|------|------|------|
| **Stop hook 무한 루프** | Stop hook이 작업을 계속 시키면 영원히 안 끝남 | `stop_hook_active` 체크 |
| **무거운 PostToolUse** | 매 편집마다 전체 테스트 실행 | PostToolUse는 가볍게, 무거운 건 Stop에 |
| **셸 프로필 오염** | `.zshrc`의 echo가 JSON 파싱 깨뜨림 | 인터랙티브 셸 체크: `[[ $- == *i* ]]` |
| **PostToolUse로 되돌리기** | 이미 실행된 도구는 취소 불가 | 차단은 반드시 PreToolUse에서 |
| **headless 모드 함정** | PermissionRequest hook은 `-p` 모드에서 안 됨 | PreToolUse 사용 |

#### 패시브 스킬 슬롯 추천 구성

"처음 시작한다면 이 조합부터":

```
슬롯 1: 🛡️ 포맷팅    — PostToolUse + Prettier/Black
슬롯 2: 🛡️ 파일 보호  — PreToolUse + protect-files.sh
슬롯 3: 🔔 알림       — Notification + 데스크톱 알림
슬롯 4: 🛡️ 기억 복원  — SessionStart(compact) + 컨텍스트 재주입
```

이 4개면 **포맷팅 자동화, 민감 파일 보호, 알림, 컨텍스트 보존**이 커버된다. 나머지는 필요에 따라 추가.

---

## 마무리

### 가설 검증 결과

**"Hook을 패시브 스킬처럼 구성하면, 사용자가 의식하지 않아도 코드 품질과 보안이 세션 내내 자동으로 유지된다."**

✅ **검증됨.**

- `settings.json`에 한 번 등록하면, `claude` 실행 시 **자동 활성화**
- 18개 라이프사이클 이벤트로 **거의 모든 시점을 커버** 가능
- `PreToolUse`로 **차단**, `PostToolUse`로 **보정**, `Stop`으로 **최종 검증**
- `command` 타입은 **토큰 0**으로 비용 부담 없음
- 개인 → 팀 → 조직으로 **점진적 확장** 가능

### Skill vs Hook 최종 비교

```
[액티브 스킬 — Skill]
  "코드 리뷰 해줘" → Claude가 skill-creator 판단 → 실행
  ✅ 복잡한 판단, 맥락에 따른 동작
  ❌ Claude가 안 부르면 안 됨

[패시브 스킬 — Hook]
  Claude가 파일 수정 → 자동으로 Prettier 실행
  ✅ 결정론적, 항상 실행, 잊을 수 없음
  ❌ 고정된 규칙만 (유연한 판단은 prompt/agent 필요)
```

**최적의 조합**: 액티브 스킬(Skill)로 **무엇을 만들지** 가이드하고, 패시브 스킬(Hook)로 **품질을 자동 유지**한다. RPG에서 액티브 스킬로 공격하고 패시브 스킬로 방어하듯이.

---

## 레퍼런스

- [Automate workflows with hooks — Claude Code Docs](https://code.claude.com/docs/en/hooks-guide)
- [Hooks reference — Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks: All 12 Lifecycle Events Explained — Pixelmojo](https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns)
- [Claude Code Hooks: A Practical Guide — DataCamp](https://www.datacamp.com/tutorial/claude-code-hooks)
- [Automate Your AI Workflows with Claude Code Hooks — GitButler](https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks)
- [claude-code-hooks-mastery — GitHub](https://github.com/disler/claude-code-hooks-mastery)
