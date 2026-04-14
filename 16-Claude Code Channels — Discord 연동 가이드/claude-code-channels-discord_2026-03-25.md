# Claude Code Channels — Discord 연동 가이드

> 최종 수정: 2026-03-25

## TL;DR

Claude Code Channels의 Discord 연동 가이드. Discord 봇을 만들어서 Claude Code 세션에 연결하면, Discord DM이나 서버 채널에서 코드 작업 지시 → Claude가 로컬에서 실행 → 결과를 같은 채팅으로 응답하는 구조임. Telegram과 거의 같은 흐름이지만, Discord는 **서버 초대 + Message Content Intent 활성화**가 추가로 필요하고, **fetch_messages/download_attachment** 도구가 추가로 있어서 첨부파일 관리가 더 풍부함.

---

## Part 1: Telegram과 뭐가 다른지

| | Telegram | Discord |
|---|---|---|
| **봇 생성** | @BotFather에서 `/newbot` | Developer Portal에서 New Application |
| **추가 설정** | 없음 | Message Content Intent 활성화 필요 |
| **서버 초대** | 불필요 (DM으로 바로) | OAuth2 URL로 서버에 초대해야 함 |
| **플러그인** | `telegram@claude-plugins-official` | `discord@claude-plugins-official` |
| **토큰 설정** | `/telegram:configure` | `/discord:configure` |
| **페어링** | `/telegram:access pair` | `/discord:access pair` |
| **도구** | reply, react, edit_message | reply, react, edit_message + **fetch_messages** + **download_attachment** |
| **첨부파일** | 사진 전달 | 최대 10개, 각 25MB. 수동 다운로드 |
| **DM** | 바로 가능 | 가능 |
| **그룹/채널** | 그룹 지원 | 서버 채널별 선택 활성화 |

### Discord만의 추가 도구

| 도구 | 기능 |
|------|------|
| **fetch_messages** | 채널의 최근 메시지 조회 (최대 100개). 첨부파일 있으면 `+Natt` 표시 |
| **download_attachment** | 특정 메시지의 모든 첨부파일을 `~/.claude/channels/discord/inbox/`로 다운로드 |

---

## Part 2: 전체 세팅 과정

### 사전 준비

| 필요한 것 | 확인 방법 | 우리 상태 |
|----------|----------|----------|
| Claude Code 2.1.80+ | `claude --version` | ✅ 2.1.81 |
| Bun 런타임 | `bun --version` | ✅ 1.3.11 |
| claude.ai 로그인 | 로그인 상태 | ✅ TeamPremium |
| Discord 계정 | 앱 설치 | ✅ |

### Step 1: Discord 봇 생성

**이 과정이 뭘 하는 건지:** Discord에서 "대리인" 역할을 할 봇 애플리케이션을 만드는 거임. Telegram의 BotFather와 비슷하지만, Discord는 Developer Portal 웹사이트에서 설정함.

1. **[Discord Developer Portal](https://discord.com/developers/applications)** 접속
2. **New Application** 클릭 → 이름 입력
3. 왼쪽 메뉴 **Bot** 클릭

> 이름에 "Claude"가 들어가면 유효하지 않다고 뜰 수 있음. 다른 이름 사용 권장.

### Step 2: Message Content Intent 활성화

**이 과정이 뭘 하는 건지:** Discord 봇이 메시지 **내용**을 읽으려면 이 권한이 필요함. 안 켜면 봇이 메시지가 왔다는 건 알지만 내용을 못 읽음.

1. Bot 설정 페이지에서 **Privileged Gateway Intents** 섹션 찾기
2. **Message Content Intent** ✅ 활성화
3. 저장

### Step 3: 봇 토큰 생성

**이 과정이 뭘 하는 건지:** 봇의 비밀번호를 만드는 거임. 이 토큰으로 Claude Code가 봇에 접속해서 메시지를 주고받음.

1. Bot 페이지의 **Token** 섹션에서 **Reset Token** 클릭
2. 토큰 복사 — **1회만 표시됨!** 놓치면 다시 Reset 해야 함

> 토큰은 봇의 완전한 제어 권한을 줌. 절대 공개하지 말 것. 노출 시 즉시 Reset Token.

### Step 4: 서버에 봇 초대

**이 과정이 뭘 하는 건지:** Telegram은 DM으로 바로 쓸 수 있지만, Discord는 봇을 **서버에 초대**해야 함. OAuth2 URL을 생성해서 봇에게 필요한 권한을 부여하고 서버에 추가하는 과정.

1. 왼쪽 메뉴 **OAuth2** → **URL Generator**
2. Scopes: **`bot`** 체크
3. Bot Permissions 체크:
   - ✅ 채널 보기 (View Channels)
   - ✅ 메시지 보내기 (Send Messages)
   - ✅ 스레드에서 메시지 보내기 (Send Messages in Threads)
   - ✅ 메시지 기록 보기 (Read Message History)
   - ✅ 파일 첨부 (Attach Files)
   - ✅ 반응 추가 (Add Reactions)
4. 하단에 생성된 **URL 복사** → 브라우저에서 열기 → 봇 넣을 서버 선택

### Step 5: 플러그인 설치

**이 과정이 뭘 하는 건지:** Claude Code에 "Discord와 통신하는 방법"을 가르쳐주는 플러그인을 설치하는 거임. Telegram 플러그인과 동일한 구조의 MCP 서버.

Claude Code 프롬프트에서:

```
/plugin install discord@claude-plugins-official
/reload-plugins
```

### Step 6: 토큰 설정

**이 과정이 뭘 하는 건지:** Step 3에서 받은 봇 토큰을 Claude Code에 알려주는 거임.

방법 1 — 명령어:
```
/discord:configure <토큰>
```

방법 2 — 수동:
```bash
mkdir -p ~/.claude/channels/discord
echo 'DISCORD_BOT_TOKEN=<토큰>' > ~/.claude/channels/discord/.env
```

> 토큰은 `~/.claude/channels/discord/.env`에 저장됨. 한번 설정하면 다시 안 해도 됨.

### Step 7: Channels 모드로 시작

**이 과정이 뭘 하는 건지:** Claude Code에게 "이번 세션에서 Discord 메시지를 수신할 준비를 해"라고 알려주는 거임. **매 세션마다 필요.**

```bash
claude --channels plugin:discord@claude-plugins-official
```

`Listening for channel messages from: plugin:discord@claude-plugins-official` 메시지가 뜨면 성공.

### Step 8: 페어링

**이 과정이 뭘 하는 건지:** "이 Discord 계정이 내 거다"라고 Claude Code에 등록. **최초 1번만.**

1. Discord에서 **Eren_Claude_Bot한테 DM** 보내기 — "안녕" 같은 거
2. 봇이 **6자리 페어링 코드** 응답
3. Claude Code에서:
```
/discord:access pair <코드>
```
4. 보안 잠금:
```
/discord:access policy allowlist
```

### Step 9: 테스트

Discord DM에서 봇한테:
```
git status 알려줘
```

Claude Code 터미널에 메시지 도착 → 작업 수행 → Discord로 응답.

### 전체 세팅 요약

```
Step 1: 봇 생성            → Developer Portal에서 앱 생성    (1번만)
Step 2: Intent 활성화      → Message Content Intent 켜기     (1번만)
Step 3: 토큰 생성          → Reset Token → 복사              (1번만)
Step 4: 서버 초대          → OAuth2 URL로 봇 추가            (1번만)
Step 5: 플러그인 설치      → /plugin install                 (1번만)
Step 6: 토큰 설정          → .env에 저장                     (1번만)
Step 7: --channels 시작    → 메시지 수신 활성화               (매 세션)
Step 8: 페어링             → DM → 코드 → pair               (1번만)
Step 9: 테스트             → 동작 확인                       (1번만)
```

**1번만 하는 것 (Step 1~6, 8):** 환경 준비 + 인증 등록.
**매번 하는 것 (Step 7):** `claude --channels plugin:discord@...` 한번이면 끝.

---

## Part 3: Telegram + Discord 동시 사용

### 동시 연결

```bash
claude --channels plugin:telegram@claude-plugins-official plugin:discord@claude-plugins-official
```

공백으로 구분해서 여러 플러그인 동시 전달 가능. 양쪽 모두에서 메시지 수신됨.

### alias 업데이트

```bash
# ~/.zshrc
alias ct='claude --channels plugin:telegram@claude-plugins-official'
alias cd_='claude --channels plugin:discord@claude-plugins-official'
alias ctd='claude --channels plugin:telegram@claude-plugins-official plugin:discord@claude-plugins-official'
```

- `ct` — Telegram만
- `cd_` — Discord만
- `ctd` — 둘 다

---

## Part 4: Discord 특수 기능

### 서버 채널에서 사용

DM뿐 아니라 **서버 채널**에서도 봇을 사용할 수 있음. 채널별로 선택 활성화:

```
/discord:access group add <채널ID>
```

봇을 멘션해야 반응하게 하려면 (기본값):
```
/discord:access group add <채널ID>  # requireMention: true (기본)
```

멘션 없이도 모든 메시지에 반응하게 하려면:
```
/discord:access group add <채널ID> --no-mention
```

> 채널 ID 확인: Discord 설정 → 고급 → 개발자 모드 활성화 → 채널 우클릭 → ID 복사

### 첨부파일 관리

Discord에서 파일을 보내면:
1. `fetch_messages`로 메시지 조회 시 `+Natt` 표시로 첨부파일 확인
2. `download_attachment(chat_id, message_id)`로 다운로드
3. 다운로드 경로: `~/.claude/channels/discord/inbox/`

> 첨부파일은 **자동 다운로드 안 됨**. 필요할 때만 수동 다운로드.

### 메시지 편집

봇이 보낸 메시지를 나중에 수정 가능:
- "작업 중..." → 완료 후 결과로 교체
- `edit_message` 도구 사용

---

## Part 5: 보안

### 접근 제어 (Telegram과 동일 구조)

```json
// ~/.claude/channels/discord/access.json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["<Discord User ID>"],
  "groups": {},
  "pending": {}
}
```

| 정책 | 설명 |
|------|------|
| `allowlist` | 등록된 ID만 메시지 수신 (권장) |
| `pairing` | 누구나 페어링 코드 받을 수 있음 (초기 설정용) |
| `disabled` | DM 비활성화 |

> Discord User ID 확인: 개발자 모드 → 사용자 우클릭 → ID 복사

---

## Part 6: 우리의 세팅 기록

### 환경

| 항목 | 값 |
|------|-----|
| Claude Code | 2.1.81 |
| Bun | 1.3.11 |
| 앱 이름 | Eren_Claude_Bot |
| 앱 ID | 1486149332326416405 |
| 서버 이름 | Eren Claude Code Bot |

### 세팅 중 겪은 이슈

| 이슈 | 원인 | 해결 |
|------|------|------|
| 앱 이름 "Eren Claude Bot" 거부 | "애플리케이션 이름이 유효하지 않습니다" — 이름 규칙 위반 | `Eren_Claude_Bot` (언더스코어)으로 변경 |
| `~/.claude` 디렉토리에서 시작 시 에러 | `plugin not installed` + `not on approved channels allowlist` | `~/Desktop/Claude-Code/research`에서 시작하니 해결 |
| `/discord:configure` 안 될 수 있음 | Telegram처럼 Unknown skill 가능성 | 수동으로 `~/.claude/channels/discord/.env`에 토큰 작성 |

### 파일 구조

```
~/.claude/channels/discord/
├── .env                # DISCORD_BOT_TOKEN=...
├── access.json         # allowlist + 승인된 sender ID
├── approved/           # 페어링 승인 기록
└── inbox/              # 다운로드된 첨부파일
```

---

## Part 7: Telegram vs Discord — 어떤 걸 쓸까

| 기준 | Telegram 추천 | Discord 추천 |
|------|-------------|-------------|
| **개인 사용** | ✅ 더 간편 | |
| **팀 협업** | | ✅ 서버 채널별 제어 |
| **파일 주고받기** | | ✅ fetch_messages + download_attachment |
| **모바일 사용** | ✅ 가볍고 빠름 | |
| **봇 API 간편함** | ✅ BotFather만으로 끝 | Developer Portal + Intent + OAuth2 |
| **이미 쓰는 플랫폼** | Telegram 유저 | Discord 유저 |

**결론:** 개인 사용은 Telegram이 더 간편하고, 팀 협업이나 파일 관리가 필요하면 Discord.

---

## 핵심 인사이트

1. **Discord 연동은 Telegram보다 초기 설정이 많지만, 기능은 더 풍부함.** Message Content Intent, OAuth2 서버 초대 등 추가 단계가 있지만, fetch_messages/download_attachment로 첨부파일 관리가 가능하고, 서버 채널별 선택 활성화로 팀 협업에 유리함.

2. **Telegram과 동시 사용 가능.** `--channels` 플래그에 공백으로 여러 플러그인을 넘기면 양쪽 모두에서 메시지 수신. alias(`ctd`)로 편하게.

3. **페어링은 1번, --channels는 매번.** Telegram과 같은 패턴. 토큰/페어링은 `~/.claude/channels/discord/`에 전역 저장되니까 어느 디렉토리에서든 사용 가능.

---

## Sources

- [Discord plugin source — GitHub](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/discord)
- [Push events into a running session with channels — Claude Code Docs](https://code.claude.com/docs/en/channels)
- [Discord Developer Portal](https://discord.com/developers/applications)
- [Claude Code Channels: Telegram & Discord Setup Guide — claudefast](https://claudefa.st/blog/guide/development/claude-code-channels)
