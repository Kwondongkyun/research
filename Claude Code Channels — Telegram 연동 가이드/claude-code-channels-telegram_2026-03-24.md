# Claude Code Channels — Telegram 연동 가이드

> 최종 수정: 2026-03-24

## TL;DR

Claude Code Channels는 Telegram/Discord 같은 메시징 앱에서 실행 중인 Claude Code 세션에 메시지를 푸시하는 기능임. MCP 서버 기반 양방향 채팅 브릿지로, 폰에서 코드 작업 지시 → Claude가 로컬에서 실행 → 결과를 같은 채팅으로 응답하는 구조. 리서치 프리뷰 단계(2.1.80+), Bun 런타임 필수, claude.ai 로그인 필요(API 키 미지원). 페어링은 최초 1번만, `--channels` 플래그는 매 세션 시작 시 붙여야 함.

---

## Part 1: Channels가 뭔지

### 정의

Channel = **MCP 서버가 외부 메시지를 실행 중인 Claude Code 세션에 푸시**하는 기능.

기존에는 Claude Code를 쓰려면 터미널 앞에 앉아있어야 했음. Channels를 쓰면 **폰에서 Telegram으로 Claude Code에 직접 명령**을 보내고, Claude가 작업한 뒤 결과를 같은 채팅으로 돌려받을 수 있음.

### 작동 원리

```
폰 (Telegram)
    │
    ├── 메시지 전송: "git status 알려줘"
    │
    ▼
Telegram Bot (MCP 서버, Bun으로 실행)
    │
    ├── 메시지를 Claude Code 세션에 푸시
    │
    ▼
Claude Code (로컬 터미널)
    │
    ├── 작업 수행 (실제 파일, 실제 환경)
    ├── reply 도구로 응답
    │
    ▼
Telegram Bot → 폰으로 응답 전달
```

핵심: **클라우드가 아니라 로컬에서 실행**됨. 내 파일, 내 환경, 내 권한으로 작업함.

### 지원 플랫폼

| 플랫폼 | 상태 | 설명 |
|--------|------|------|
| **Telegram** | ✅ 공식 지원 | DM, 그룹, 사진 전달, 입력 표시기 |
| **Discord** | ✅ 공식 지원 | DM, 서버 채널 |
| **iMessage** | ✅ 공식 지원 | Apple 기기 |
| **Webhook** | ✅ 공식 지원 | CI/CD, 모니터링 등 커스텀 통합 |
| **Fakechat** | ✅ 데모 | localhost 테스트용 |

### 제약사항

| 제약 | 상세 |
|------|------|
| **세션 열려있어야 함** | 세션 닫으면 메시지 수신 안 됨. 상시 운용 시 tmux 권장 |
| **Bun 필수** | Node.js 불가. `bun --version`으로 확인 |
| **claude.ai 로그인** | API 키, Console 인증 미지원 |
| **리서치 프리뷰** | 문법/프로토콜 변경 가능 |
| **권한 승인은 터미널에서만** | 폰에서 직접 승인 불가 (permission relay 선언한 채널은 가능) |
| **1봇 = 1세션** | 동시에 2개 세션 연결 불가 |

### 다른 기능과의 비교

| 기능 | 뭘 하는지 | 적합한 경우 |
|------|----------|-----------|
| **Channels** | 외부 메시지를 열린 세션에 푸시 | 폰에서 제어, CI 알림 수신 |
| **Remote Control** | claude.ai에서 로컬 세션 조종 | 브라우저에서 세션 이어받기 |
| **Claude Code on Web** | GitHub 클론 후 클라우드에서 실행 | 독립적 비동기 작업 |
| **MCP 서버** | Claude가 필요 시 쿼리 | 도구/데이터 접근 |
| **/loop** | 타이머로 반복 실행 | 폴링, 빌드 감시 |

---

## Part 2: Telegram 연동 — 전체 세팅 과정

### 사전 준비

| 필요한 것 | 확인 방법 | 우리 상태 |
|----------|----------|----------|
| Claude Code 2.1.80+ | `claude --version` | ✅ 2.1.81 |
| Bun 런타임 | `bun --version` | ✅ 1.3.11 (curl로 설치) |
| claude.ai 로그인 (Pro/Max/Team) | 로그인 상태 | ✅ TeamPremium |
| Telegram 계정 | 앱 설치 | ✅ |

### Step 1: Bun 설치

**이 과정이 뭘 하는 건지:** Telegram 채널 플러그인은 Bun이라는 JavaScript 런타임 위에서 돌아감. Node.js로는 안 되고 반드시 Bun이어야 함. Bun이 봇과 Telegram 서버 사이의 통신(폴링)을 처리함.

```bash
curl -fsSL https://bun.sh/install | bash
source ~/.zshrc
bun --version  # 1.3.11 확인
```

### Step 2: Telegram 봇 생성

**이 과정이 뭘 하는 건지:** Telegram에서 "대리인" 역할을 할 봇을 만드는 거임. 이 봇이 내 메시지를 받아서 Claude Code에 전달하고, Claude의 응답을 나한테 돌려줌. BotFather는 Telegram 공식 봇 생성 도구임.

1. Telegram 앱에서 **@BotFather** 검색
2. `/newbot` 전송
3. 봇 이름 입력 (예: "Eren Claude Bot") — 채팅방에 보이는 표시 이름
4. 봇 username 입력 — 끝에 `bot` 필수 (예: `eren_claude_bot`) — 검색용 고유 ID
5. BotFather가 **토큰**을 줌 — `8271126390:AAE4lT...` 형태

> 토큰은 봇의 비밀번호와 같음. 이걸 가진 사람은 봇을 제어할 수 있으므로 노출 금지. 노출 시 BotFather에서 `/revoke`로 재발급.

### Step 3: 플러그인 설치

**이 과정이 뭘 하는 건지:** Claude Code에 "Telegram과 통신하는 방법"을 가르쳐주는 플러그인을 설치하는 거임. 이 플러그인이 MCP 서버로 동작하면서 Telegram Bot API와 Claude Code 세션 사이를 연결해줌.

Claude Code 프롬프트에서:

```
/plugin install telegram@claude-plugins-official
/reload-plugins
```

> 플러그인이 안 보이면: `/plugin marketplace update claude-plugins-official` 먼저 실행

### Step 4: 토큰 설정

**이 과정이 뭘 하는 건지:** Step 2에서 받은 봇 토큰을 Claude Code에 알려주는 거임. 이 토큰으로 플러그인이 Telegram Bot API에 접속해서 메시지를 주고받음.

방법 1 — 명령어:
```
/telegram:configure <토큰>
```

방법 2 — 수동 (명령어가 안 될 때):
```bash
mkdir -p ~/.claude/channels/telegram
echo 'TELEGRAM_BOT_TOKEN=<토큰>' > ~/.claude/channels/telegram/.env
```

우리 경우 `/telegram:configure`가 `Unknown skill`로 인식 안 돼서 방법 2로 수동 설정했음.

> 토큰은 `~/.claude/channels/telegram/.env`에 저장됨. 한번 설정하면 다시 안 해도 됨.

### Step 5: Channels 모드로 시작

**이 과정이 뭘 하는 건지:** Claude Code에게 "이번 세션에서 Telegram 메시지를 수신할 준비를 해"라고 알려주는 거임. 이 플래그 없이 시작하면 플러그인은 설치돼있지만 메시지를 안 받음. **매 세션마다 해야 함** (페어링과 다른 점).

현재 세션을 종료하고:

```bash
claude --channels plugin:telegram@claude-plugins-official
```

시작 시 `Listening for channel messages from: plugin:telegram@claude-plugins-official` 메시지가 뜨면 성공.

### Step 6: 페어링

**이 과정이 뭘 하는 건지:** "이 Telegram 계정이 내 거다"라고 Claude Code에 등록하는 거임. 6자리 코드를 교환해서 내 Telegram ID를 allowlist에 추가함. **최초 1번만 하면 됨** — 이후에는 `--channels`만 붙이면 자동 연결.

1. **Telegram에서 봇 검색** — `eren_claude_bot` 검색해서 채팅방 진입
2. **아무 메시지 전송** — "안녕" 같은 거
3. **봇이 6자리 코드 응답** — 예: `6b0480`
4. **Claude Code에서 승인**:
```
/telegram:access pair 6b0480
```
5. **보안 잠금** — 내 계정만 접근 가능하게:
```
/telegram:access policy allowlist
```

이렇게 하면 내 Telegram 계정만 봇에 접근 가능. 다른 사람이 메시지 보내도 무시됨.

> 페어링 정보는 `~/.claude/channels/telegram/access.json`에 영구 저장됨.

### Step 7: 테스트

**이 과정이 뭘 하는 건지:** 실제로 연동이 작동하는지 확인하는 거임.

Telegram에서 봇한테 메시지 보내기:
```
git status 알려줘
```

Claude Code 터미널에 메시지가 `<channel source="telegram">` 이벤트로 도착하고, Claude가 작업 수행 후 Telegram으로 응답이 옴.

### 전체 세팅 요약

```
Step 1: Bun 설치        → 플러그인 런타임 준비         (1번만)
Step 2: 봇 생성         → Telegram 대리인 생성         (1번만)
Step 3: 플러그인 설치    → Claude Code에 통신 방법 추가  (1번만)
Step 4: 토큰 설정       → 봇 인증 정보 등록            (1번만)
Step 5: --channels 시작 → 메시지 수신 활성화           (매 세션)
Step 6: 페어링          → 내 Telegram 계정 등록        (1번만)
Step 7: 테스트          → 동작 확인                   (1번만)
```

**1번만 하는 것 (Step 1~4, 6):** 환경 준비 + 인증 등록. 한번 하면 영구 저장.
**매번 하는 것 (Step 5):** `ct` (alias) 한번이면 끝.

---

## Part 3: 페어링 vs --channels — 뭐가 다른지

| | 페어링 | `--channels` 플래그 |
|---|---|---|
| **뭘 하는 건지** | "이 Telegram 계정을 신뢰함" 등록 | "이 세션에서 Telegram 수신 활성화" |
| **언제 하는지** | **최초 1번만** | **매 세션 시작 시** |
| **저장 위치** | `~/.claude/channels/telegram/access.json` (영구) | 세션 메모리 (휘발) |
| **안 하면** | 봇이 메시지를 무시함 | 봇이 메시지를 수신 못 함 |

```
최초 설정 (1번만)
├── 봇 생성 (BotFather) ✅
├── 토큰 설정 (.env) ✅
└── 페어링 (access pair) ✅

매 세션 시작 시
└── claude --channels plugin:telegram@... ← 이것만 하면 됨
```

---

## Part 4: 실전 사용 팁

### alias 등록 — 매번 긴 명령어 안 치려면

```bash
# ~/.zshrc에 추가
alias ct='claude --channels plugin:telegram@claude-plugins-official'
```

그러면:
```bash
ct              # Telegram 연결된 Claude Code 시작
ct --resume     # 이전 세션 이어서 + Telegram
```

### 다른 디렉토리에서 사용

**디렉토리와 무관**함. 토큰/페어링은 `~/.claude/channels/telegram/`에 전역 저장이라 어디서든 `ct`만 치면 연결됨.

```bash
cd ~/project-a && ct    # A 프로젝트에서 Telegram 연결
# ... 작업 ...
# 세션 종료 후
cd ~/project-b && ct    # B 프로젝트에서도 바로 연결 (재페어링 불필요)
```

### 동시에 2개 세션은?

**1봇 = 1세션.** 봇은 항상 **마지막에 `--channels`로 시작한 세션**에만 연결됨.

```
상황 1: A 닫고 B 열기
A (ct) → 종료 → B (ct)
→ 정상. B에서 수신.

상황 2: A 열린 채로 B 열기
A (ct) 실행 중... → B (ct) 실행
→ B가 봇 폴링을 가져감. A는 더 이상 메시지 안 옴.

상황 3: B 닫으면?
→ A가 자동으로 복구되지 않음. 아무 세션도 수신 안 함.
→ A를 다시 ct로 시작해야 함.
```

**결론:** 동시에 2개는 안 됨. 마지막에 연 세션이 봇을 독점함.

정말 동시에 2개 필요하면:
1. BotFather에서 봇 2개 생성 (`bot_a`, `bot_b`)
2. 각각 다른 토큰 설정
3. 각 세션에서 다른 봇으로 연결

현실적으로는 **한번에 하나, 프로젝트 바꿀 때 세션 재시작**이 일반적.

### 상시 운용 — tmux

세션이 닫히면 봇이 응답 안 함. 상시 운용하려면:

```bash
# tmux 세션 생성
tmux new -s claude

# 안에서 Claude Code 시작
ct

# 터미널에서 빠져나오기 (세션은 유지)
# Ctrl+B → D

# 나중에 다시 붙기
tmux attach -t claude
```

### Context rot 주의

Telegram으로 메시지를 계속 보내면, 그 내용이 **세션의 컨텍스트 윈도우**에 계속 쌓임.

```
세션 시작 (0%)
  → "git status 알려줘" (5%)
    → "이 파일 수정해줘" (15%)
      → "테스트 돌려줘" (30%)
        → ... 계속 쌓임 ...
          → 80%+ : 초반 대화를 "잊기" 시작 (lost-in-the-middle)
            → 100% : 자동 compaction (오래된 내용 압축)
```

**lost-in-the-middle**: 컨텍스트가 길어지면 Claude가 처음과 끝은 잘 기억하는데 **중간 내용을 놓치는 현상**. LLM의 알려진 한계임.

**Telegram에서 특히 문제인 이유**: 터미널에서는 세션을 자연스럽게 끊지만, Telegram은 카톡처럼 편해서 **한 세션이 지나치게 길어지기 쉬움**.

**해결법:**
- `/compact` — 수동으로 컨텍스트 압축 (오래된 내용 요약)
- 작업이 바뀔 때 **세션 새로 시작**하는 게 가장 깔끔

### 권한 문제

Claude가 작업 중 권한 승인이 필요하면 세션이 **일시 정지**됨. 터미널에서 직접 승인해야 함.

> permission relay 기능을 선언한 채널은 폰에서 승인 가능하지만, allowlist에 등록된 사람만 승인 권한이 있으므로 신뢰할 수 있는 사람만 등록할 것.

---

## Part 5: 보안

### 발신자 허용 목록 (Allowlist)

모든 채널 플러그인은 **sender allowlist**를 유지함. 등록 안 된 발신자의 메시지는 **자동 무시**.

```json
// ~/.claude/channels/telegram/access.json
{
  "dmPolicy": "allowlist",     // allowlist = 등록된 사람만
  "allowFrom": ["8678018625"], // 내 Telegram ID
  "groups": {},
  "pending": {}
}
```

### 정책 옵션

| 정책 | 설명 |
|------|------|
| `allowlist` | 등록된 ID만 메시지 수신 (권장) |
| `pairing` | 누구나 페어링 코드를 받을 수 있음 (초기 설정용) |

페어링 완료 후 반드시 `allowlist`로 전환:
```
/telegram:access policy allowlist
```

### Enterprise 제어

| 플랜 | 기본 상태 |
|------|----------|
| Pro/Max (조직 미소속) | 채널 사용 가능, 세션별 opt-in |
| Team/Enterprise | **기본 비활성**. 관리자가 `channelsEnabled: true` 설정 필요 |

---

## Part 6: 실제 사용 시나리오

| 시나리오 | 폰에서 보내는 메시지 |
|---------|-------------------|
| 파일 확인 | "package.json 보여줘" |
| 코드 작업 | "src/app.tsx에서 버그 찾아줘" |
| 테스트 실행 | "npm test 돌려보고 결과 알려줘" |
| git 상태 | "git status 알려줘" |
| 빌드 모니터링 | "빌드 끝나면 알려줘" |
| 코드 리뷰 | "PR #42 리뷰해줘" |
| 원격 트러블슈팅 | "서버 로그에서 에러 찾아줘" |

---

## Part 7: 커뮤니티 반응 (GeekNews)

GeekNews에서의 핵심 논의 포인트:

### 왜 Telegram이 먼저?
- Telegram MAU **10억 명** (Slack 5천만, Teams 3억보다 훨씬 큼)
- Telegram Bot API가 역대급으로 간편. "챗봇 만들려면 사실상 유일한 선택지"
- iMessage는 폐쇄적, WhatsApp은 유료, Discord/Slack은 무거움

### 한계와 우려
- **Context rot**: 긴 채팅 시 lost-in-the-middle 문제. `/compact` 필요
- **터미널 의존성**: 세션 닫히면 끝. tmux로 해결 가능하지만 근본적 한계
- **보안 우려**: 원격 기능 = 잠재적 백도어. 기업은 정상/악성 인스턴스 구분 필요

### 관련 프로젝트
- **agent-http** ([GitHub](https://github.com/mberg/agent-http/)): Channels를 이용해 Claude Code를 HTTP API로 감싼 프로젝트
- **OpenClaw**: 이미 비슷한 기능을 1인 오픈소스로 구현했었음. Anthropic이 빠르게 따라잡은 형태
- **nanoclaw**: Docker 컨테이너에서 PID 1로 실행하는 포크

### 전문가 의견
- "Claude는 로컬 세션 중심 구조로 가는 중. 사용자의 개인 인증 정보를 활용해 통합 구축 가능"
- "엔터프라이즈 보안에 부합. 내부 네트워크가 이미 잠겨있으니 추가 외부 API 보안 불필요"
- "Anthropic이 매주 새로운 제품을 실험적으로 출시하는 느낌. 대부분 1년 내에 사라질 수도"

---

## Part 8: 우리의 세팅 기록

### 환경

| 항목 | 값 |
|------|-----|
| Claude Code | 2.1.81 |
| Bun | 1.3.11 |
| 플랜 | TeamPremium |
| 봇 이름 | Eren Claude Bot |
| 봇 username | @eren_claude_bot |
| 봇 토큰 | `8271126390:AAE4lT...` (보안상 일부 마스킹) |
| Telegram ID | 8678018625 |

### 세팅 중 겪은 이슈

| 이슈 | 원인 | 해결 |
|------|------|------|
| Bun 미설치 | macOS에 기본 설치 안 됨 | `curl -fsSL https://bun.sh/install \| bash` |
| `/telegram:configure` 인식 안 됨 | `Unknown skill` 에러 | 수동으로 `~/.claude/channels/telegram/.env`에 토큰 작성 |
| 봇 메시지 미응답 | `--channels` 없이 시작 | `claude --channels plugin:telegram@...`로 재시작 |

### 파일 구조

```
~/.claude/channels/telegram/
├── .env                # TELEGRAM_BOT_TOKEN=...
├── access.json         # allowlist + 승인된 sender ID
└── approved/           # 페어링 승인 기록
```

### 등록한 alias

```bash
# ~/.zshrc
alias ct='claude --channels plugin:telegram@claude-plugins-official'
```

---

## 핵심 인사이트

1. **Channels = 폰에서 Claude Code 제어.** 터미널 앞에 안 앉아있어도 Telegram으로 코드 작업 지시 → 결과 수신 가능. 로컬 실행이라 내 파일/환경/권한 그대로 사용.

2. **세팅은 한번, 사용은 매번 `ct`.** 봇 생성 + 토큰 + 페어링은 최초 1번만. 이후 매 세션마다 `ct` (alias)만 치면 연결. 디렉토리 무관.

3. **1봇 = 1세션, 상시 운용은 tmux.** 동시 2세션 불가. 세션 닫히면 봇 무응답. tmux로 세션 유지하면 상시 사용 가능. Context rot 방지를 위해 주기적 `/compact` 권장.

---

## Sources

- [Push events into a running session with channels — Claude Code Docs](https://code.claude.com/docs/en/channels)
- [Telegram plugin source — GitHub](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram)
- [Claude Code Channels 공개 — GeekNews](https://news.hada.io/topic?id=22988)
- [Claude Code Channels: Telegram & Discord Setup Guide — claudefast](https://claudefa.st/blog/guide/development/claude-code-channels)
- [What Is Claude Code Channels? — lowcode.agency](https://www.lowcode.agency/blog/claude-code-channels)
- [I Set Up Claude Code Channels on Telegram — Medium](https://medium.com/@markchen69/i-set-up-claude-code-channels-on-telegram-now-my-phone-controls-my-terminal-5cdd9dece513)
- [Claude Code Telegram Plugin Setup Guide — DEV Community](https://dev.to/czmilo/claude-code-telegram-plugin-complete-setup-guide-2026-3j0p)
- [First Look: Telegram and Discord Integrations — MacStories](https://www.macstories.net/stories/first-look-hands-on-with-claude-codes-new-telegram-and-discord-integrations/)
