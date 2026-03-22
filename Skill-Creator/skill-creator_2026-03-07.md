# Skill-Creator — 스킬을 만드는 스킬

> 최종 수정: 2026-03-23

## TL;DR

Claude Code의 **스킬(Skill)**은 코드가 아닌 **프롬프트 시스템**임. SKILL.md에 "언제, 뭘, 어떻게"를 적으면 Claude가 알아서 발견하고 실행함. 핵심은 **Progressive Disclosure** — 메타데이터만 미리 로드하고, 나머지는 필요할 때만 읽는 구조.

**skill-creator**는 Anthropic 공식 메타 스킬로, 4가지 모드(Create, Eval, Improve, Benchmark)와 3개 서브에이전트(Grader, Comparator, Analyzer), 9개 스크립트, eval-viewer까지 갖춘 풀 스택 스킬 개발 도구임.

---

## Part 1: 스킬 해부학

---

### 1. 스킬이란 — 코드가 아닌 프롬프트 시스템

스킬은 **실행 코드가 아님.** Claude의 컨텍스트에 주입되는 **프롬프트 확장**임.

```
일반 도구 (Tool)           스킬 (Skill)
──────────────────         ──────────────────
동기식 직접 실행             프롬프트 확장
즉시 결과 반환              컨텍스트 변경
함수 호출과 비슷            매뉴얼 펼치기와 비슷
```

> Tool은 **자판기**(버튼 누르면 정해진 결과), Skill은 **바리스타에게 레시피를 건네는 것**(같은 사람이 레시피에 따라 다른 음료를 만듦)임.

스킬이 호출되면 Claude에게 **2개의 메시지**가 주입됨:

| 메시지 | 속성 | 용도 |
|--------|------|------|
| 메시지 1 | `isMeta: false` | 사용자 UI에 표시 (상태 안내) |
| 메시지 2 | `isMeta: true` | API로만 전송, UI에서 숨김 (Claude용 상세 지침) |

> 식당으로 치면 손님에게는 "파스타 나갑니다" 확인서를, 주방에는 "면 알덴테, 소스 진하게" 조리 지시서를 보내는 구조임.

스킬은 실행 중 임시로 환경을 변경할 수 있음:
- **도구 사전 승인** — 특정 도구를 사용자 프롬프트 없이 허용
- **모델 오버라이드** — 필요 시 다른 모델로 전환
- **effort 오버라이드** — 스킬별 thinking 깊이 조절
- 스킬 완료 후 **자동 복구**

> 핵심: 스킬 = "일반 목적 에이전트를 **특화된 에이전트로 임시 변환**하는 장치"

---

### 2. 스킬의 내부 동작 — 3단계 로딩

스킬은 한 번에 전부 로드되지 않음. **필요한 만큼만, 필요한 시점에** 로드됨.

```
[시스템 프롬프트 시작]
    │
    ├── 1단계: 메타데이터만 로드
    │   name: "pdf-processing"
    │   description: "PDF 파일에서 텍스트/테이블 추출..."
    │   (모든 스킬의 name + description이 시스템 프롬프트에 올라감)
    │   (컨텍스트 윈도우의 2% 또는 16,000자 예산)
    │
    ├── 2단계: SKILL.md 본문 로드
    │   사용자 요청이 스킬과 매칭 → Claude가 Skill 도구 호출
    │   → SKILL.md 전체 내용을 읽음
    │
    └── 3단계: 부가 파일 선택 로드
        SKILL.md에서 참조된 파일만 필요할 때 읽음
        예: reference/finance.md, scripts/validate.py
```

이 구조 덕분에:
- 스킬이 100개 있어도 **메타데이터만** 컨텍스트를 차지
- 큰 참고 자료를 번들링해도 **읽기 전까지 토큰 비용 0**
- 스크립트는 **실행만** 하면 되니 코드 자체를 컨텍스트에 올릴 필요 없음

---

### 3. SKILL.md 해부 — frontmatter부터 body까지

모든 스킬의 시작점은 **SKILL.md** 파일임. 두 부분으로 나뉨:

```yaml
---
# === YAML Frontmatter (1단계: 항상 로드) ===
name: pdf-processing                    # 선택, 64자 이하 (생략 시 디렉토리명)
description: >                          # 권장, 1024자 이하 (생략 시 첫 문단)
  PDF 파일에서 텍스트와 테이블을 추출하고,
  폼을 채우고, 문서를 병합한다.
  PDF 파일 작업이나 문서 추출 요청 시 사용.
---

# PDF Processing                        ← 2단계: 스킬 트리거 시 로드

## Quick start
pdfplumber로 텍스트 추출:
...

## Advanced features
**폼 채우기**: [FORMS.md](FORMS.md) 참고     ← 3단계: 필요 시에만 로드
```

#### Frontmatter 필드 전체 정리

| 필드 | 필수 | 제약 | 설명 |
|------|------|------|------|
| `name` | 선택 | 64자, 소문자+숫자+하이픈만 | 스킬 식별자. 생략 시 디렉토리명 사용 |
| `description` | 권장 | 1024자 | 발견 + 트리거 기준. 생략 시 본문 첫 문단 사용 |
| `allowed-tools` | 선택 | | 스킬 실행 중 허용할 도구 목록 |
| `model` | 선택 | | 모델 오버라이드 (예: haiku) |
| `effort` | 선택 | low/medium/high/max | thinking 깊이. max는 Opus 4.6 전용 |
| `context` | 선택 | `fork` | 서브에이전트에서 격리 실행 |
| `agent` | 선택 | | `context: fork` 시 에이전트 타입 (Explore, Plan 등) |
| `disable-model-invocation` | 선택 | true/false | true면 Claude 자동 호출 차단 (수동 전용) |
| `user-invocable` | 선택 | true/false | false면 `/` 메뉴에서 숨김 (Claude만 호출) |
| `argument-hint` | 선택 | | 자동완성 힌트 (예: `[issue-number]`) |
| `hooks` | 선택 | | 스킬 라이프사이클 훅 |

#### 호출 제어 조합

| 설정 | 사용자 호출 | Claude 호출 | 컨텍스트 로드 |
|------|-----------|------------|-------------|
| (기본) | O | O | description 항상 로드 |
| `disable-model-invocation: true` | O | X | description 로드 안 됨 |
| `user-invocable: false` | X | O | description 항상 로드 |

#### 변수 치환

| 변수 | 설명 |
|------|------|
| `$ARGUMENTS` | 스킬 호출 시 전달된 전체 인자 |
| `$ARGUMENTS[N]` / `$N` | N번째 인자 (0-based) |
| `${CLAUDE_SKILL_DIR}` | 스킬의 SKILL.md가 있는 디렉토리 경로 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |

#### 동적 컨텍스트 주입

`` !`command` `` 문법으로 쉘 명령 출력을 스킬에 주입할 수 있음:

```yaml
---
name: pr-summary
context: fork
agent: Explore
---

## PR 컨텍스트
- PR diff: !`gh pr diff`
- 변경 파일: !`gh pr diff --name-only`

이 PR을 요약해줘.
```

`!`command``는 **전처리**임 — Claude가 보기 전에 실행되고, 출력이 스킬 내용에 삽입됨.

#### Body 작성 원칙

- **500줄 이하** 유지 — 넘으면 별도 파일로 분할
- **Claude가 이미 아는 것은 안 적음** — PDF가 뭔지 설명할 필요 없음
- 참조 파일은 **1단계 깊이**만 — SKILL.md → reference.md (O), SKILL.md → a.md → b.md (X)
- 100줄 이상의 참고 파일은 **상단에 목차(TOC)** 포함

#### 디렉토리 구조

```
skill-name/
├── SKILL.md              # 핵심 프롬프트 (항상 여기서 시작)
├── references/           # 상세 문서 (필요 시 로드)
│   ├── finance.md
│   └── sales.md
├── scripts/              # 실행 코드 (실행만, 로드 아님)
│   ├── validate.py
│   └── analyze.py
└── assets/               # 템플릿/바이너리
    └── template.docx
```

---

### 4. 번들 스킬 — Claude Code에 기본 탑재된 스킬

| 스킬 | 기능 |
|------|------|
| `/batch <instruction>` | 대규모 코드 변경을 병렬 에이전트로 분산 처리. 각 에이전트가 worktree에서 독립 작업 후 PR |
| `/loop [interval] <prompt>` | 반복 실행. 빌드 폴링, PR 감시 등. 기본 10분 간격 |
| `/simplify [focus]` | 최근 변경 코드를 3개 리뷰 에이전트가 병렬 검토 후 수정 |
| `/debug [description]` | 세션 디버그 로그를 읽고 문제 분석 |
| `/claude-api` | Claude API/SDK 참고 자료 로드. `anthropic` import 감지 시 자동 활성화 |

---

### 5. description이 결정적인 이유 — LLM 기반 선택

스킬 선택에는 **임베딩도, 분류기도, 패턴 매칭도 사용하지 않음.** Claude의 **순수 언어 이해**로 선택함.

```
사용자: "이 PDF에서 텍스트 뽑아줘"
    │
    Claude: 100개 스킬의 description을 스캔
    │
    "PDF 파일에서 텍스트와 테이블을 추출..." ← 매칭!
    │
    → pdf-processing 스킬 트리거
```

따라서 description은 **검색 엔진의 메타 태그**와 같음.

#### 잘 쓰는 법

**3인칭으로 작성** — description은 시스템 프롬프트에 주입되므로 시점이 중요:

```yaml
# ✅ Good — 3인칭
description: PDF 파일에서 텍스트와 테이블을 추출한다. PDF 작업이나 문서 추출 요청 시 사용.

# ❌ Bad — 1인칭
description: 저는 PDF 파일을 처리할 수 있습니다.
```

**"뭘 하는지" + "언제 쓰는지" 둘 다 포함**:

```yaml
# ✅ Good — What + When
description: >
  Excel 스프레드시트를 분석하고, 피벗 테이블을 만들고, 차트를 생성한다.
  Excel 파일, 스프레드시트, 표 데이터, .xlsx 파일을 분석할 때 사용.

# ❌ Bad — What만
description: 문서를 처리한다.
```

#### Naming 컨벤션

`name` 필드는 **소문자 + 숫자 + 하이픈**만 가능. **동명사(-ing)** 형태를 권장:

```
✅ processing-pdfs, analyzing-spreadsheets, testing-code
✅ pdf-processing, spreadsheet-analysis (명사구도 가능)
❌ helper, utils, tools (모호)
❌ anthropic-helper, claude-tools (예약어 포함)
```

---

### 6. Progressive Disclosure — 3가지 패턴

SKILL.md가 복잡해지면 **파일을 분할**함. 핵심은 "목차에서 시작해 필요한 장만 읽는" 구조.

#### 패턴 1: 고수준 가이드 + 참조 파일
SKILL.md는 개요와 퀵스타트만, 상세는 별도 파일로. Claude는 폼 채우기 요청이 올 때만 FORMS.md를 읽음.

#### 패턴 2: 도메인별 분리
여러 도메인을 다루는 스킬에서 유용. "매출 데이터 분석해줘"라고 하면 `finance.md`만 로드. 나머지는 토큰 0.

#### 패턴 3: 조건부 상세
기본 내용은 인라인으로, 고급 내용만 분리.

#### 주의사항
- **참조는 1단계 깊이만** — `SKILL.md → a.md` (O), `SKILL.md → a.md → b.md` (X)
- 긴 참조 파일(100줄 이상)은 **상단에 목차** 포함

---

### 7. Anti-patterns

| Anti-pattern | 문제 | 해결 |
|-------------|------|------|
| Claude가 아는 것을 설명 | 토큰 낭비 | "PDF란..." 같은 설명 삭제 |
| 옵션을 너무 많이 제시 | 결정 마비 | 기본값 제시 + 탈출구 하나 |
| Windows 경로 사용 | 크로스 플랫폼 에러 | 항상 `/` (forward slash) |
| 용어 불일치 | 혼란 | 하나의 용어 일관 사용 |
| 깊은 참조 (2단계+) | 정보 누락 | 1단계 깊이만 |
| 도구 설치 가정 | 실행 실패 | `pip install ...` 명시 |
| Magic number | 이유 불명 | 상수에 이유를 주석으로 |

---

## Part 2: skill-creator 사용법

---

### 8. 설치 및 설정

skill-creator는 Anthropic 공식 GitHub 저장소에 있음:

```bash
# Claude Code 내에서
/plugin install https://github.com/anthropics/skills
```

4가지 모드로 호출:

```
"새 스킬 만들어줘"          → Create 모드
"이 스킬 eval 돌려줘"       → Eval 모드
"이 스킬 개선해줘"          → Improve 모드
"스킬 벤치마크 해줘"        → Benchmark 모드
```

---

### 9. skill-creator 내부 구조

```
skill-creator/
├── SKILL.md                          # 메인 지침 (4모드 + 전체 프로세스)
├── LICENSE.txt
│
├── agents/                           # 3개 서브에이전트
│   ├── grader.md                     # 결과 채점
│   ├── analyzer.md                   # 패턴 분석 + 개선 제안
│   └── comparator.md                 # 블라인드 A/B 비교
│
├── scripts/                          # 9개 유틸리티 스크립트
│   ├── __init__.py
│   ├── init_skill.py                 # 스킬 디렉토리 초기화
│   ├── run_eval.py                   # Eval 실행 자동화
│   ├── run_loop.py                   # Description 최적화 반복
│   ├── aggregate_benchmark.py        # 벤치마크 결과 통계 집계
│   ├── generate_report.py            # 보고서 생성
│   ├── improve_description.py        # description 자동 개선
│   ├── package_skill.py              # 스킬 패키징 (배포용)
│   ├── quick_validate.py             # 빠른 유효성 검사
│   └── utils.py                      # 공용 유틸리티
│
├── references/
│   └── schemas.md                    # 8개 JSON 스키마 정의
│
├── eval-viewer/                      # 결과 비교 뷰어
│   ├── generate_review.py            # HTML 리뷰 생성
│   └── viewer.html                   # 브라우저 뷰어 템플릿
│
└── assets/
    └── eval_review.html              # 트리거 Eval 리뷰 템플릿
```

#### 서브에이전트 3개

| 에이전트 | 역할 | 핵심 프로세스 |
|---------|------|-------------|
| **Grader** | 결과 채점 | 7단계: 트랜스크립트 읽기 → 출력 검사 → assertion별 PASS/FAIL 판정(근거 포함) → 주장 검증 → 사용자 노트 확인 → Eval 자체 비평 → grading.json 작성 |
| **Comparator** | 블라인드 A/B 비교 | 어떤 스킬이 만든 건지 **모르는 상태**에서 품질만으로 판단. Content(정확성·완성도) + Structure(조직·가독성) 이중 루브릭 1-5점. TIE는 진짜 동등할 때만 |
| **Analyzer** | 패턴 분석 + 인사이트 | (1) 비교 후 왜 이겼는지/졌는지 추출 + priority별 개선 제안, (2) 벤치마크 전체에서 패턴·이상치 탐지. 추측 없이 데이터 기반만 |

#### 8개 JSON 스키마 (`references/schemas.md`)

| 스키마 | 용도 |
|--------|------|
| `evals.json` | 테스트 케이스 정의 (프롬프트 + 기대 결과) |
| `grading.json` | 채점 결과 (PASS/FAIL + 근거 + 통계) |
| `metrics.json` | 도구 사용량, 스텝 수, 에러, 글자 수 |
| `timing.json` | 실행 시간 측정 (시작/끝/분할) |
| `benchmark.json` | 벤치마크 집계 (N회 반복 통계 + 비교) |
| `comparison.json` | 블라인드 비교 결과 (승자 + 루브릭 점수) |
| `analysis.json` | 인사이트 + 개선 제안 (priority: high/medium/low) |
| `history.json` | 버전별 개선 이력 (pass rate 추적) |

---

### 10. Create 모드 — 새 스킬 생성

#### 프로세스

```
Phase 1: 의도 파악
    → 뭘 하는 스킬인지, 언제 트리거되는지, 출력 형태는
Phase 2: 인터뷰 + 리서치
    → 엣지 케이스, 의존성, 성공 기준 질문
Phase 3: SKILL.md 작성
    → frontmatter + body + 참조 파일
Phase 4: 테스트 케이스 생성
    → evals/evals.json에 2-3개의 현실적 프롬프트
Phase 5: Eval 실행
    → with-skill vs baseline 비교
```

#### 핵심 원칙: Eval 먼저

```
1. 스킬 없이 Claude에게 작업을 시켜봄
2. 어디서 실패하는지 관찰
3. 실패 지점을 해결하는 최소한의 스킬을 만듦
4. Eval로 비교
```

> "상상한 요구사항"이 아니라 **"실제 실패"**를 해결하는 스킬을 만들라는 철학.

---

### 11. Eval 모드 — with-skill vs baseline 비교

#### Eval 파일 구조

```json
{
  "skill_name": "pdf-processing",
  "evals": [
    {
      "id": 0,
      "prompt": "이 PDF에서 텍스트를 추출해서 output.txt에 저장해줘",
      "expected_output": "모든 페이지의 텍스트가 output.txt에 저장됨",
      "files": ["test-files/document.pdf"],
      "expectations": [
        "적절한 PDF 라이브러리를 사용하여 파일을 읽음",
        "모든 페이지에서 텍스트를 누락 없이 추출",
        "추출된 텍스트를 output.txt에 저장"
      ]
    }
  ]
}
```

#### 실행 과정

```
1. With-skill 실행 → iteration-N/eval-0/with_skill/outputs/
2. Baseline 실행 → iteration-N/eval-0/without_skill/outputs/
   (개선 시에는 old_skill/ 사용)
3. Grader 에이전트가 각 실행을 채점 → grading.json
4. aggregate_benchmark.py로 통계 집계 → benchmark.json
5. Analyzer 에이전트가 패턴 분석
6. eval-viewer로 브라우저에서 결과 비교
7. 사용자 피드백 → feedback.json → 다음 이터레이션에 반영
```

---

### 12. eval-viewer — 브라우저에서 결과 비교

```bash
# 기본 (로컬 서버)
python <skill-creator-path>/eval-viewer/generate_review.py \
  <workspace>/iteration-N \
  --skill-name "my-skill" \
  --benchmark <workspace>/iteration-N/benchmark.json

# 이전 이터레이션과 비교
--previous-workspace <workspace>/iteration-<N-1>

# headless 환경 (정적 HTML)
--static <output_path>
```

사용자가 리뷰를 끝내면 `feedback.json`이 생성됨:

```json
{
  "reviews": [
    {
      "run_id": "eval-0-with_skill",
      "feedback": "차트에 축 레이블이 없음",
      "timestamp": "..."
    }
  ],
  "status": "complete"
}
```

빈 피드백 = 사용자가 괜찮다고 판단한 것.

---

### 13. Improve 모드 — 기존 스킬 개선

#### 개선 시 사고 방식

1. **피드백에서 일반화** — 테스트 케이스에 과적합하지 말고 넓게 적용되게
2. **프롬프트를 날씬하게** — 효과 없는 부분 제거. 출력만 보지 말고 트랜스크립트를 읽기
3. **이유를 설명** — Claude한테 "왜 이렇게 해야 하는지" 이론적 맥락 제공
4. **반복 작업 번들링** — 서브에이전트들이 독립적으로 같은 헬퍼 스크립트를 만들면 scripts/에 포함

#### 이터레이션 루프

```
개선 적용 → 전체 Eval 재실행 (iteration-N+1/) → eval-viewer 실행
→ 사용자 리뷰 → feedback.json 읽기 → 다시 개선 → 반복
```

**종료 조건:**
- 사용자가 만족한다고 말함
- 피드백이 전부 빈칸
- 의미 있는 진전이 없음

#### Claude A / Claude B 패턴

| 역할 | 하는 일 |
|------|---------|
| **Claude A** (작성자) | 스킬을 분석하고 개선안 작성 |
| **Claude B** (테스터) | 개선된 스킬로 실제 작업 수행 |
| **사용자** | Claude B의 행동을 관찰하고 피드백 |

이 패턴은 **Ralph Loop**과 본질적으로 같음 — 개발 → 리뷰 → 수정 → 재리뷰의 반복.

---

### 14. Benchmark 모드 — 분산 분석

LLM은 **비결정적**임. 같은 프롬프트로 10번 돌리면 결과가 다를 수 있음.

```
같은 Eval을 N번 반복 실행
    │
    ├── aggregate_benchmark.py로 결과 통합
    ├── 성공률, 평균 소요 시간, 토큰 사용량 집계
    └── 분산이 크면 → 스킬 지침이 모호하다는 신호
```

| 증상 | 원인 | 해결 |
|------|------|------|
| 성공/실패가 반반 | description이 모호 | 키워드 + 트리거 조건 강화 |
| 매번 다른 방법 사용 | 선택지가 너무 많음 | 기본값 하나 명시 |
| Haiku에서만 실패 | 지침이 고수준 모델에 의존 | 더 구체적인 가이드 추가 |

---

### 15. Description 최적화 루프

#### Step 1: 트리거 Eval 쿼리 생성

should-trigger(8-10개) + should-not-trigger(8-10개), 총 20개:

```json
[
  {"query": "보스가 보낸 Q4 sales final FINAL v2.xlsx 분석해줘", "should_trigger": true},
  {"query": "이 CSV 파일 열어줘", "should_trigger": false}
]
```

**품질 기준:**
- 현실적이고 구체적 (파일명, 회사명, 오타 포함)
- 길이, 대소문자, 구어체 혼합
- should-trigger: 다양한 표현, 비주류 사용 사례, 경쟁 스킬과 겹치는 경우
- should-not-trigger: 비슷하지만 다른 도메인, 진짜 헷갈리는 경우

#### Step 2: 사용자 리뷰

`eval_review.html` 템플릿으로 브라우저에서 편집 → `eval_set.json` 내보내기

#### Step 3: 자동 최적화

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id> \
  --max-iterations 5 \
  --verbose
```

#### Step 4: 결과 적용

최적화된 description을 SKILL.md frontmatter에 반영.

---

## 마무리

### 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| **스킬** | 코드가 아닌 프롬프트 확장. Claude를 임시로 특화 에이전트로 변환 |
| **Progressive Disclosure** | 메타데이터 → SKILL.md → 참조 파일, 3단계로 필요한 만큼만 로드 |
| **description** | 스킬 발견의 열쇠. LLM이 자연어로 매칭하므로 키워드와 컨텍스트가 핵심 |
| **skill-creator** | Create → Eval → Improve → Benchmark의 4모드 루프 |
| **서브에이전트** | Grader(채점) + Comparator(블라인드 비교) + Analyzer(패턴 분석) |
| **eval-viewer** | 브라우저에서 결과 비교 + 피드백 수집 |
| **Eval 우선** | 상상한 요구사항이 아닌 실제 실패 지점을 해결하는 스킬을 만듦 |

### 스킬 생성 전체 흐름

```
의도 파악
    → init_skill.py (템플릿 생성)
        → SKILL.md 작성 (frontmatter + body)
            → Eval 생성 (evals.json)
                → Baseline vs With-skill 비교
                    → Grader 채점 → eval-viewer 리뷰
                        → 피드백 기반 개선 (Improve)
                            → Benchmark (N회 반복, 분산 분석)
                                → Description 최적화 (run_loop)
                                    → 팀 피드백 → 반복
```

### Ralph Loop과의 유사성

| Ralph Loop | skill-creator |
|-----------|---------------|
| REVIEW.md에 피드백 기록 | Eval 결과 + feedback.json |
| dev가 REVIEW.md 읽고 수정 | description/body 수정 |
| Critical 0까지 반복 | 트리거 정확도 + Eval 통과까지 반복 |
| `<promise>DONE</promise>` | 모든 Eval 통과 + 사용자 만족 |

**핵심은 같음**: 한 번에 완벽하게가 아니라, **빠르게 만들고, 측정하고, 고치는 루프**.

---

## 레퍼런스

- [Skill authoring best practices — Anthropic Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Agent Skills Overview — Anthropic Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Extend Claude with Skills — Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Equipping Agents for the Real World with Agent Skills — Anthropic Engineering](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Agent Skills: A First Principles Deep Dive — Lee Han Chung](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [skill-creator — GitHub (anthropics/skills)](https://github.com/anthropics/skills/tree/main/skills/skill-creator)
- [How to create custom Skills — Claude Help Center](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
- [Skill Creator Plugin — Anthropic](https://claude.com/plugins/skill-creator)
