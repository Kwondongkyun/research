# AI 트렌드 자동화 파이프라인 — Cowork에서 Obsidian까지 완전 자동화

> 최종 수정: 2026-04-07

## TL;DR

Claude Cowork로 AI 트렌드 수집 에이전트를 만들고, Claude Code 스킬로 이식한 뒤, 리모트 에이전트 스케줄링 + launchd + Obsidian Git으로 "완전 자동화 → Obsidian 자동 반영" 파이프라인을 구축했다. 코딩 없이 자연어 명세서만으로 만든 자동화 시스템이다.

---

## Part 1: 전체 아키텍처

```
[Cowork 스케줄 / 리모트 에이전트]
  매주 월/금 오전 10시 KST
  → AI 트렌드 수집 (HN x2, Reddit, Product Hunt, 웹 검색)
  → 점수화 (언급빈도 40 + 관심도 30 + 최신성 30)
  → AI트렌드_YYYYMMDD.md 생성
  → GitHub {YOUR_GITHUB_USERNAME}/weekly-trends 커밋

[launchd - macOS]
  매시간 자동 실행
  → git pull (eren Obsidian/weekly-trends)

[Obsidian]
  → weekly-trends 폴더에서 바로 확인
```

**핵심:** 사람이 할 일은 0. 컴퓨터가 켜져 있고 인터넷 연결만 되어 있으면 됨.

---

## Part 2: 단계별 구축 과정

### Step 1: Cowork로 에이전트 프로토타입 만들기

Claude Desktop Cowork 탭에서 아래 명세서를 붙여넣어 에이전트를 빌드했다.

**핵심 명세서 구조:**
```
1. 수집 소스 및 가중치 정의
2. 점수화 로직 (언급빈도/관심도/최신성)
3. 결과물 형식 (파일명, 마크다운 포맷)
4. 스케줄 설정
```

**Cowork가 자동으로 한 것:**
- Python 코드 생성
- 웹 검색 커넥터 활성화
- 스케줄 등록

**피드백 반영:** 초기 결과가 GitHub 스타 중심으로 편향 → HN/Reddit/Product Hunt/웹 검색 4개 소스로 확장

### Step 2: Claude Code 스킬로 이식

Cowork에서 만든 에이전트를 Claude Code에서도 쓸 수 있게 `.skill` 파일로 export → 설치.

**설치 경로:** `~/.claude/skills/ai-trend-collector/SKILL.md`

**frontmatter 필수 설정:**
```yaml
---
name: ai-trend-collector
description: > (트리거 문구 다양하게)
allowed-tools: WebSearch, WebFetch, Write, Read
effort: medium
---
```

**저장 경로 명시:**
```
저장 경로: /Users/kwondong-kyun/eren Obsidian/weekly-trends/
파일명: AI트렌드_YYYYMMDD.md
```

**트리거 문구 예시:**
- "이번 주 AI 트렌드 정리해줘"
- "요즘 AI 쪽에서 뭐가 핫해?"
- "/ai-trend-collector"

### Step 3: GitHub 레포 연결

**새 레포 생성:** `{YOUR_GITHUB_USERNAME}/weekly-trends`

**로컬 초기화:**
```bash
cd "~/eren Obsidian/weekly-trends"
git init
git remote add origin https://github.com/{YOUR_GITHUB_USERNAME}/weekly-trends.git
git add .
git commit -m "init: AI트렌드 첫 리포트 추가"
git branch -M main
git push -u origin main
```

**주의:** GitHub에 README가 있으면 unrelated histories 충돌 발생 → `--allow-unrelated-histories --no-rebase`로 해결

### Step 4: Obsidian 연동

**구조:**
```
eren Obsidian/
  research/     ← 심볼릭 링크 (Desktop/Claude-Code/research)
  weekly-trends/ ← 독립 git 레포 (GitHub 연결)
  무제/
```

**심볼릭 링크 방식 (research):**
```bash
ln -s "/Users/kwondong-kyun/Desktop/Claude-Code/research" \
      "/Users/kwondong-kyun/eren Obsidian/research"
```

### Step 5: launchd 자동 pull 설정

**파일 경로:** `~/Library/LaunchAgents/com.kwondongkyun.weekly-trends-pull.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.kwondongkyun.weekly-trends-pull</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/git</string>
        <string>-C</string>
        <string>/Users/kwondong-kyun/eren Obsidian/weekly-trends</string>
        <string>pull</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

**등록/해제:**
```bash
# 등록
launchctl load ~/Library/LaunchAgents/com.kwondongkyun.weekly-trends-pull.plist

# 해제
launchctl unload ~/Library/LaunchAgents/com.kwondongkyun.weekly-trends-pull.plist
```

### Step 6: 리모트 에이전트 스케줄 등록

**Claude GitHub App 설치 필요:** https://github.com/apps/claude
- `{YOUR_GITHUB_USERNAME}/weekly-trends` 레포에 권한 부여

**스케줄 설정:**
- 실행 주기: 매주 월/금 오전 10시 KST
- cron (UTC): `0 1 * * 1,5`
- 모델: claude-sonnet-4-6
- 레포: `{YOUR_GITHUB_USERNAME}/weekly-trends`

**관리 페이지:** https://claude.ai/code/scheduled

**프롬프트 작성 원칙:**
- 리모트 에이전트는 매번 새 세션으로 시작 (이전 기억 없음)
- 프롬프트가 길수록 좋음 — 모든 지시사항을 자체 포함해야 함
- 파일 저장 + git commit + push까지 프롬프트에 명시

---

## Part 3: 점수화 로직 상세

### 수집 소스 및 가중치

| 소스 | 가중치 | 특징 |
|------|--------|------|
| Hacker News | x2 | 개발자/연구자 커뮤니티, 기술적 깊이 |
| Reddit r/ML, r/artificial | x1 | 학계 논문, 오픈소스 릴리스 |
| Product Hunt AI | x1 | 실제 제품 출시 동향 |
| 일반 웹 검색 | x1 | 규제, M&A, 시장 동향 |

### 점수 산출 기준

| 기준 | 만점 | 세부 |
|------|------|------|
| 언급 빈도 | 40점 | 4곳=35~40 / 3곳=25~34 / 2곳=15~24 / 1곳=5~14 |
| 관심도 | 30점 | 높음=25~30 / 중간=15~24 / 낮음=5~14 |
| 최신성 | 30점 | 24h=26~30 / 3일=18~25 / 7일=10~17 / 7일+=5~9 |

**편향 방지 규칙:**
- 단일 소스 토픽은 15위 이하 배치
- 특정 지표(GitHub 스타 등) 단독 의존 금지

---

## Part 4: 스킬 파일 전문 (복붙 설치용)

다른 사람이 이 스킬을 사용하려면 아래 내용을 그대로 복사해서 저장하면 된다.

**설치 방법:**
1. `~/.claude/skills/ai-trend-collector/` 폴더 생성
2. 아래 내용을 `SKILL.md`로 저장
3. **저장 경로를 본인 경로로 수정** (86번째 줄)
4. Claude Code 재시작

```markdown
---
name: ai-trend-collector
description: >
  AI 테크 분야 인기 주제 Top 20을 수집하고 점수화하여 마크다운 리포트를 생성하는 스킬.
  Hacker News, Reddit, Product Hunt, 웹 검색 4개 소스에서 교차 수집한 뒤,
  언급 빈도/관심도/최신성 3가지 기준으로 종합 점수를 산출한다.
  "AI 트렌드", "AI 뉴스 요약", "이번 주 AI 핫토픽", "AI 키워드 리포트",
  "요즘 AI 뭐가 뜨는지", "AI 동향 정리" 등의 요청에 반드시 이 스킬을 사용한다.
  트렌드 분석, 기술 동향, 주간 리포트 생성이 필요할 때도 적극 사용한다.
allowed-tools: WebSearch, WebFetch, Write, Read
effort: medium
---

# AI 트렌드 수집기

최근 7일간 AI/ML 분야에서 가장 주목받는 키워드 20개를 수집하고, 점수화하여 마크다운 파일로 출력한다.

## 왜 이 스킬이 필요한가

AI 분야는 매주 새로운 모델, 논문, 제품, 정책이 쏟아진다. 하나의 소스만 보면 편향되기 쉽고, 여러 소스를 일일이 확인하는 건 시간이 많이 든다. 이 스킬은 4개 소스를 교차 대조해서 "진짜 뜨고 있는 것"과 "한 곳에서만 언급된 것"을 구분해준다. 신문 4개를 동시에 펼쳐놓고 겹치는 헤드라인을 골라내는 작업을 자동화한다고 생각하면 된다.

## 수집 프로세스

### 1단계: 4개 소스에서 병렬 수집

WebSearch를 사용하여 아래 4개 소스에서 **동시에** 검색한다. 병렬 실행이 핵심이다 -- 순차적으로 하면 시간이 4배 걸린다.

1. **Hacker News** (가중치 x2)
   - `"Hacker News top AI artificial intelligence stories this week {month} {year}"`

2. **Reddit r/MachineLearning, r/artificial**
   - `"Reddit r/MachineLearning r/artificial top posts this week AI {month} {year}"`

3. **Product Hunt AI**
   - `"Product Hunt AI launches trending {month} {year}"`

4. **일반 웹 검색**
   - `"AI technology trends news this week {month} {year}"`

### 2단계: 심층 검색

1단계에서 발견된 주요 토픽에 대해 추가 WebSearch를 2~4회 수행한다.

### 3단계: 점수화 (100점 만점)

#### 언급 빈도 (40점)
- 4곳 모두: 35~40점 / 3곳: 25~34점 / 2곳: 15~24점 / 1곳: 5~14점
- HN 출처는 2배 카운트

#### 관심도 (30점)
- 높음: 25~30점 / 중간: 15~24점 / 낮음: 5~14점

#### 최신성 (30점)
- 24시간: 26~30점 / 3일: 18~25점 / 7일: 10~17점 / 7일 초과: 5~9점

#### 편향 방지
- 단일 소스 토픽은 15위 이하 배치
- 특정 소스 과도 의존 금지

### 4단계: 마크다운 리포트 생성

**저장 경로**: `/본인경로/weekly-trends/`  ← 본인 경로로 수정
**파일명**: `AI트렌드_YYYYMMDD.md`

출력 포맷:
# AI 트렌드 키워드 Top 20 (YYYY-MM-DD)

> 수집 소스: Hacker News (x2 가중치), Reddit r/MachineLearning & r/artificial, Product Hunt AI, 웹 검색 (최근 7일)
> 점수 산출 기준: 언급 빈도 (40점) + 관심도 (30점) + 최신성 (30점) = 종합 100점

## 1. [한국어 키워드명] / [영어 키워드명] (점수: XX/100)
- 출처: [소스 목록]
- 최신성: [몇 시간/일 전]
- 한줄 요약: [핵심 내용 1~2문장]

(이하 동일 형식으로 20개)

*생성 일시: YYYY-MM-DD | 다음 업데이트: YYYY-MM-DD (요일)*

## 출력 규칙
- 영어 키워드는 한국어 번역 병기
- 이모지 사용 금지
- 점수는 1위→20위 단조 감소

## 사용 예시

Input: "이번 주 AI 트렌드 정리해줘"
Output: AI트렌드_YYYYMMDD.md 파일 생성

Input: "요즘 AI 쪽에서 뭐가 핫해?"
Output: AI트렌드_YYYYMMDD.md 파일 생성 + 상위 5개 키워드 대화로 요약
```

---

## 핵심 인사이트

1. **Cowork = 비개발자용 에이전트 프로토타이핑 도구**: 명세서(자연어)만으로 Python 코드 생성 + 웹 검색 + 스케줄링까지 완성. Claude Code 스킬로 이식하면 터미널에서도 동일하게 사용 가능.
2. **리모트 에이전트 프롬프트는 길수록 좋다**: 매번 새 세션이라 맥락이 없음. 수집 로직, 점수화 기준, 저장 경로, git 커밋까지 모두 프롬프트에 명시해야 의도대로 실행됨.
3. **launchd + GitHub + Obsidian 조합으로 완전 자동화**: 컴퓨터만 켜져 있으면 매시간 자동 pull → 별도 동기화 작업 없이 Obsidian에서 바로 확인. 클라우드 의존 없이 로컬에서 완결되는 구조.
