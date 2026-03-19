# my-update-check

> `/my-update-check` 한 번이면 버전 확인부터 내 설정 영향 분석, 적용 제안까지.

Claude Code는 거의 매일 업데이트된다. 근데 매번 GitHub Releases 들어가서 확인하고, 내 스킬이나 에이전트에 영향 있는지 하나씩 대조하기는 번거롭다.

이 스킬은 **"지금 내가 얼마나 뒤처져 있고, 뭘 놓치고 있는지"**를 한 번에 알려준다.

---

## 뭘 가져오나

두 종류의 소스를 **병렬로** 수집한다.

### 코드 레벨 변경 — GitHub Releases API

버전별 "What's changed" 목록. 기능 추가, 버그 수정, 설정 변경 등. 매일 업데이트되며, API 실패 시 CHANGELOG.md로 자동 Fallback.

### 플랫폼/생태계 변경 — WebSearch

CHANGELOG에는 절대 안 잡히는 것들:

- Agent Skills 공식 블로그 — 새 생태계 발표
- skill-creator 개선 — 멀티에이전트 eval 지원
- 공식 Skills 문서 — /simplify, /batch 같은 번들 스킬
- SkillsMP 마켓플레이스 — 커뮤니티 스킬 생태계
- TechCrunch 같은 외부 보도

이 두 소스가 겹치는 부분은 거의 없어서 상호 보완적이다.

---

## 어떻게 동작하나

```
Step 1  버전 비교 (claude --version vs npm latest)
Step 2  변경사항 수집 (GitHub Releases + WebSearch 병렬)
Step 3  내 설정 영향 분석 (스킬/에이전트/CLAUDE.md 대조)
Step 4  마크다운 테이블 리포트 출력
```

---

## 왜 만들었나

- GitHub Releases를 매번 직접 확인 → **자동 수집 + 범위 필터링**
- 변경사항이 내 설정에 영향 있는지 모름 → **스킬/에이전트/CLAUDE.md 자동 대조**
- 블로그 발표, 새 번들 스킬 등 놓침 → **WebSearch로 플랫폼 변경까지 커버**
- 뭘 해야 하는지 모름 → **구체적 명령어 + 파일 경로로 적용 제안**

---

## 출력 예시

**2.1.63** → **2.1.74** (11개 릴리즈)

- agents `model:` 필드 full ID 지원 → agents/*.md (2.1.74)
- skill 디렉토리 대량 변경 시 데드락 수정 → skills/*/ (2.1.73)
- autoMemoryDirectory 설정 추가 (2.1.74)
- modelOverrides 설정 추가 (2.1.73)
- /plan 인자 지원 (2.1.72)

적용 제안:

- [ ] `npm update -g @anthropic-ai/claude-code`
- [ ] agents/*.md의 `model: opus` → full ID 전환 검토

---

## 스킬 코드

```
---
name: my-update-check
description: Claude Code 업데이트를 확인하고 내 설정 영향을 분석하는 스킬. "업데이트 확인", "update check", "버전 확인" 요청에 사용.
allowed-tools: Read, Glob, Grep, WebSearch, Bash(claude --version), Bash(npm view *), Bash(curl *)
---
```

### 데이터 소스

| 소스 | 용도 | 명령어 |
|------|------|--------|
| npm 레지스트리 | 최신 버전 번호 | `npm view @anthropic-ai/claude-code version` |
| GitHub Releases API | 코드 레벨 변경사항 (JSON) | `curl -s "https://api.github.com/repos/anthropics/claude-code/releases?per_page=20"` |
| WebSearch | 블로그/문서/플랫폼 레벨 변경 | `WebSearch("Claude Code update {year}-{month}")` |
| CHANGELOG.md | Fallback (API 실패 시) | `curl -s "https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md"` |

### Step 1: 버전 비교

두 명령어를 **병렬로** 실행:

```bash
claude --version
npm view @anthropic-ai/claude-code version
```

- **동일**: "Claude Code 최신 상태입니다" 출력 후 종료
- **다름**: Step 2로 진행

### Step 2: 변경사항 수집

**주 소스: GitHub Releases API**

```bash
curl -s "https://api.github.com/repos/anthropics/claude-code/releases?per_page=20"
```

JSON에서 현재 버전 초과 ~ 최신 버전 이하 범위만 필터링.

**Fallback: CHANGELOG.md** — API rate limit(403) 시 사용.

**보조 소스: WebSearch** — GitHub Releases와 병렬 실행:

```
WebSearch("Claude Code update {current_year}-{current_month} new features")
WebSearch("Claude Code skills agent update {current_year}")
```

### Step 3: 내 설정 영향 분석

| 대상 | 경로 | 방식 |
|------|------|------|
| 스킬 | `~/.claude/skills/*/SKILL.md` | frontmatter만 Grep으로 수집, 관련 스킬만 상세 확인 |
| 에이전트 | `~/.claude/agents/*.md` | 전체 읽기 |
| CLAUDE.md | `~/.claude/CLAUDE.md` | 전체 읽기 |

영향 3단계 분류:

| 수준 | 기준 | 예시 |
|------|------|------|
| **직접 영향** | 내가 쓰는 기능이 변경/제거됨 | agents의 `model:` 필드 동작 변경 |
| **간접 영향** | 새 기능이 내 설정을 개선할 수 있음 | 새 frontmatter 필드 추가 |
| **무관** | 내 설정과 관계없음 | 타 OS 버그 수정 |

### Step 4: 리포트 출력

마크다운 테이블 중심으로 화면에 직접 출력. 파일 저장 안 함.

### Error Handling

| 시나리오 | 대응 |
|---------|------|
| 네트워크 실패 | 에러 출력 후 종료 |
| GitHub API rate limit | CHANGELOG.md로 Fallback |
| npm 명령어 실패 | GitHub Releases의 최신 tag_name 사용 |
| claude --version 실패 | "Claude Code가 PATH에 없습니다" 출력 후 종료 |

### Limitations

- 인증 없는 GitHub API는 시간당 60회 요청 제한
- 비공개 릴리즈(pre-release)는 포함하지 않음
- 설정 영향 분석은 키워드 기반 추론이므로 100% 정확하지 않을 수 있음
- MCP 서버, 플러그인 업데이트는 확인하지 않음
- WebSearch 결과는 검색 시점의 인덱싱 상태에 따라 최신 블로그/문서가 누락될 수 있음
