# Skill-Creator — 스킬을 만드는 스킬

> 최종 수정: 2026-03-07

## TL;DR

Claude Code의 **스킬(Skill)**은 코드가 아니라 **프롬프트 시스템**이다. SKILL.md에 "언제, 뭘, 어떻게"를 적으면 Claude가 알아서 발견하고 실행한다. 핵심은 **Progressive Disclosure** — 메타데이터만 미리 로드하고, 나머지는 필요할 때만 읽는 구조다.

**skill-creator**는 Anthropic 공식 메타 스킬로, 이 과정을 체계화한다: 스킬 생성 → Eval 테스트 → 벤치마크 → Description 최적화까지 하나의 루프로 돌린다.

---

## Part 1: 스킬 해부학

---

### 1. 스킬이란 — 코드가 아닌 프롬프트 시스템

스킬은 **실행 코드가 아니다.** 스킬은 Claude의 컨텍스트에 주입되는 **프롬프트 확장**이다.

```
일반 도구 (Tool)           스킬 (Skill)
──────────────────         ──────────────────
동기식 직접 실행             프롬프트 확장
즉시 결과 반환              컨텍스트 변경
함수 호출과 비슷            매뉴얼 펼치기와 비슷
```

> ☕ Tool은 **자판기**(버튼 누르면 정해진 결과), Skill은 **바리스타에게 레시피를 건네는 것**(같은 사람이 레시피에 따라 다른 음료를 만듦)이다.

스킬이 호출되면 Claude에게 **2개의 메시지**가 주입된다:

| 메시지 | 속성 | 용도 |
|--------|------|------|
| 메시지 1 | `isMeta: false` | 사용자 UI에 표시 (상태 안내) |
| 메시지 2 | `isMeta: true` | API로만 전송, UI에서 숨김 (Claude용 상세 지침) |

이 이중 채널로 **사용자 투명성**과 **지침 명확성**을 동시에 확보한다.

> 🎭 식당으로 치면 손님에게는 "파스타 나갑니다" 확인서를, 주방에는 "면 알덴테, 소스 진하게" 조리 지시서를 보내는 구조다.

스킬은 실행 중 임시로 환경을 변경할 수 있다:
- **도구 사전 승인** — 특정 도구를 사용자 프롬프트 없이 허용
- **모델 오버라이드** — 필요 시 다른 모델로 전환
- 스킬 완료 후 **자동 복구**

> 🥼 수술실에 들어가면 수술복 입고 메스 권한을 받지만, 끝나면 다시 일상복으로 돌아가는 것과 같다.

> 🎬 같은 배우가 대본에 따라 다른 역할을 하듯, 같은 Claude가 Skill에 따라 다른 전문가로 변신한다.

> 핵심: 스킬 = "일반 목적 에이전트를 **특화된 에이전트로 임시 변환**하는 장치"

---

### 2. 스킬의 내부 동작 — 3단계 로딩

스킬은 한 번에 전부 로드되지 않는다. **필요한 만큼만, 필요한 시점에** 로드된다.

```
[시스템 프롬프트 시작]
    │
    ├── 1단계: 메타데이터만 로드
    │   name: "pdf-processing"
    │   description: "PDF 파일에서 텍스트/테이블 추출..."
    │   (모든 스킬의 name + description이 시스템 프롬프트에 올라감)
    │
    ├── 2단계: SKILL.md 본문 로드
    │   사용자 요청이 스킬과 매칭 → Claude가 Skill 도구 호출
    │   → SKILL.md 전체 내용을 읽음
    │
    └── 3단계: 부가 파일 선택 로드
        SKILL.md에서 참조된 파일만 필요할 때 읽음
        예: reference/finance.md, scripts/validate.py
```

**비유**: 매뉴얼의 **목차**만 펼쳐놓고, 필요한 **챕터**만 읽고, 필요하면 **부록**을 찾아보는 것과 같다.

이 구조 덕분에:
- 스킬이 100개 있어도 **메타데이터만** 컨텍스트를 차지
- 큰 참고 자료를 번들링해도 **읽기 전까지 토큰 비용 0**
- 스크립트는 **실행만** 하면 되니 코드 자체를 컨텍스트에 올릴 필요 없음

---

### 3. SKILL.md 해부 — frontmatter부터 body까지

모든 스킬의 시작점은 **SKILL.md** 파일이다. 두 부분으로 나뉜다:

```yaml
---
# === YAML Frontmatter (1단계: 항상 로드) ===
name: pdf-processing                    # 필수, 64자 이하, 소문자+하이픈
description: >                          # 필수, 1024자 이하
  PDF 파일에서 텍스트와 테이블을 추출하고,
  폼을 채우고, 문서를 병합한다.
  PDF 파일 작업이나 문서 추출 요청 시 사용.
---

# PDF Processing                        ← 2단계: 스킬 트리거 시 로드

## Quick start
pdfplumber로 텍스트 추출:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features
**폼 채우기**: [FORMS.md](FORMS.md) 참고     ← 3단계: 필요 시에만 로드
**API 레퍼런스**: [REFERENCE.md](REFERENCE.md) 참고
```

#### Frontmatter 필드 정리

| 필드 | 필수 | 제약 | 설명 |
|------|------|------|------|
| `name` | ✅ | 64자, 소문자+숫자+하이픈만 | 스킬 식별자 |
| `description` | ✅ | 1024자 | 발견 + 트리거 기준 |
| `allowed-tools` | | | 스킬 실행 중 허용할 도구 목록 |
| `model` | | | 모델 오버라이드 (예: haiku) |
| `license` | | | 라이선스 정보 |

#### Body 작성 원칙

- **500줄 이하** 유지 — 넘으면 별도 파일로 분할
- **Claude가 이미 아는 것은 안 적는다** — PDF가 뭔지 설명하지 않아도 됨
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

> `{baseDir}` 변수로 경로를 지정하면 이식성이 보장된다.

---

### 4. description이 결정적인 이유 — LLM 기반 선택

스킬 선택에는 **임베딩도, 분류기도, 패턴 매칭도 사용하지 않는다.** Claude의 **순수 언어 이해**로 선택한다.

```
사용자: "이 PDF에서 텍스트 뽑아줘"
    │
    Claude: 100개 스킬의 description을 스캔
    │
    "PDF 파일에서 텍스트와 테이블을 추출..." ← 매칭!
    │
    → pdf-processing 스킬 트리거
```

따라서 description은 **검색 엔진의 메타 태그**와 같다.

#### 잘 쓰는 법

**3인칭으로 작성** — description은 시스템 프롬프트에 주입되므로 시점이 중요:

```yaml
# ✅ Good — 3인칭
description: PDF 파일에서 텍스트와 테이블을 추출한다. PDF 작업이나 문서 추출 요청 시 사용.

# ❌ Bad — 1인칭
description: 저는 PDF 파일을 처리할 수 있습니다.

# ❌ Bad — 2인칭
description: PDF 파일을 처리할 때 이 도구를 사용하세요.
```

**"뭘 하는지" + "언제 쓰는지" 둘 다 포함**:

```yaml
# ✅ Good — What + When
description: >
  Excel 스프레드시트를 분석하고, 피벗 테이블을 만들고, 차트를 생성한다.
  Excel 파일, 스프레드시트, 표 데이터, .xlsx 파일을 분석할 때 사용.

# ❌ Bad — What만
description: 문서를 처리한다.

# ❌ Bad — 너무 모호
description: 데이터 관련 작업을 한다.
```

**키워드를 포함** — Claude가 자연어로 매칭하므로, 사용자가 쓸 법한 단어를 넣는다:

```yaml
description: >
  git diff를 분석하여 서술적인 커밋 메시지를 생성한다.
  커밋 메시지 작성이나 스테이지된 변경사항 리뷰 요청 시 사용.
```

#### Naming 컨벤션

`name` 필드는 **소문자 + 숫자 + 하이픈**만 가능하다. **동명사(-ing)** 형태를 권장:

```
✅ processing-pdfs, analyzing-spreadsheets, testing-code
✅ pdf-processing, spreadsheet-analysis (명사구도 가능)
❌ helper, utils, tools (모호)
❌ anthropic-helper, claude-tools (예약어 포함)
```

---

### 5. Progressive Disclosure — 3가지 패턴

SKILL.md가 복잡해지면 **파일을 분할**한다. 핵심은 "목차에서 시작해 필요한 장만 읽는" 구조다.

#### 패턴 1: 고수준 가이드 + 참조 파일

가장 기본적인 패턴. SKILL.md는 개요와 퀵스타트만, 상세는 별도 파일로:

```markdown
# PDF Processing

## Quick start
pdfplumber로 텍스트 추출:
...

## Advanced features
**폼 채우기**: [FORMS.md](FORMS.md) 참고
**API 레퍼런스**: [REFERENCE.md](REFERENCE.md) 참고
**예제 모음**: [EXAMPLES.md](EXAMPLES.md) 참고
```

Claude는 폼 채우기 요청이 올 때만 FORMS.md를 읽는다.

#### 패턴 2: 도메인별 분리

여러 도메인을 다루는 스킬에서 유용:

```
bigquery-skill/
├── SKILL.md (개요 + 네비게이션)
└── reference/
    ├── finance.md (매출, 빌링)
    ├── sales.md (파이프라인, 기회)
    ├── product.md (API 사용량)
    └── marketing.md (캠페인, 어트리뷰션)
```

```markdown
# BigQuery Data Analysis

## Available datasets
**Finance**: 매출, ARR, 빌링 → [reference/finance.md](reference/finance.md) 참고
**Sales**: 기회, 파이프라인 → [reference/sales.md](reference/sales.md) 참고
...
```

사용자가 "매출 데이터 분석해줘"라고 하면 `finance.md`만 로드. 나머지는 토큰 0.

#### 패턴 3: 조건부 상세

기본 내용은 인라인으로, 고급 내용만 분리:

```markdown
# DOCX Processing

## Creating documents
docx-js로 새 문서 생성. [DOCX-JS.md](DOCX-JS.md) 참고.

## Editing documents
간단한 수정은 XML 직접 편집.

**변경 추적 필요 시**: [REDLINING.md](REDLINING.md) 참고
**OOXML 상세**: [OOXML.md](OOXML.md) 참고
```

#### 주의사항

- **참조는 1단계 깊이만** — `SKILL.md → a.md` (O), `SKILL.md → a.md → b.md` (X)
- 깊은 참조는 Claude가 `head -100`으로 미리보기만 할 수 있음 → 정보 누락
- 긴 참조 파일(100줄 이상)은 **상단에 목차** 포함

---

### 6. Anti-patterns

| Anti-pattern | 문제 | 해결 |
|-------------|------|------|
| Claude가 아는 것을 설명 | 토큰 낭비 | "PDF란..." 같은 설명 삭제 |
| 옵션을 너무 많이 제시 | 결정 마비 | 기본값 제시 + 탈출구 하나 |
| Windows 경로 사용 | 크로스 플랫폼 에러 | 항상 `/` (forward slash) |
| 시간에 의존하는 정보 | 금방 구식됨 | "Old patterns" 접기 사용 |
| 용어 불일치 | 혼란 | 하나의 용어 일관 사용 |
| 깊은 참조 (2단계+) | 정보 누락 | 1단계 깊이만 |
| Magic number | 이유 불명 | 상수에 이유를 주석으로 |
| 도구 설치 가정 | 실행 실패 | `pip install ...` 명시 |

---

## Part 2: skill-creator 사용법

---

### 7. 설치 및 설정

skill-creator는 Anthropic 공식 GitHub 저장소에 있다:

```
Repository: anthropics/skills
경로: skills/skill-creator/
```

Claude Code에서 플러그인으로 설치:

```bash
# Claude Code 내에서
/install-plugin https://github.com/anthropics/skills
```

설치 후 사용 가능한 명령:

```
"새 스킬 만들어줘"          → 스킬 생성 모드
"이 스킬 개선해줘"          → 스킬 분석 + 최적화
"스킬 eval 돌려줘"          → 테스트 실행
"스킬 벤치마크 해줘"        → 성능 측정
"description 최적화해줘"    → 트리거 정확도 개선
```

---

### 8. 새 스킬 생성 — init_skill.py

skill-creator의 가장 기본적인 기능. **템플릿 기반으로 스킬 디렉토리를 자동 생성**한다.

#### 프로세스

```
"새 스킬 만들어줘"
    │
    ├── 1. 의도 파악 — 무슨 스킬을 만들 건지 질문
    ├── 2. init_skill.py 실행 — 디렉토리 + SKILL.md 템플릿 생성
    ├── 3. SKILL.md 작성 — frontmatter + body 채우기
    ├── 4. 테스트 케이스 생성 — 2-3개의 현실적 프롬프트
    └── 5. Eval 실행 — 스킬 있을 때 vs 없을 때 비교
```

#### 생성되는 구조

```
my-skill/
├── SKILL.md              # 템플릿 (name, description 빈칸)
├── scripts/              # 유틸리티 스크립트 디렉토리
└── evals/
    └── evals.json        # 테스트 케이스 파일
```

#### 핵심 원칙: Eval 먼저

skill-creator의 접근법은 **"Eval 우선 개발"**이다:

```
1. 스킬 없이 Claude에게 작업을 시켜본다
2. 어디서 실패하는지 관찰한다
3. 실패 지점을 해결하는 최소한의 스킬을 만든다
4. Eval로 비교한다
```

> "상상한 요구사항"이 아니라 **"실제 실패"**를 해결하는 스킬을 만들라는 철학이다.

---

### 9. 기존 스킬 개선

이미 만든 스킬을 분석하고 최적화하는 기능.

#### 개선 루프

```
기존 SKILL.md 분석
    │
    ├── description 검토 — 트리거 정확도 확인
    ├── body 검토 — 토큰 효율, 구조 최적화
    ├── 참조 파일 검토 — Progressive Disclosure 적용 여부
    └── 개선안 제시 → 적용 → Eval로 검증
```

#### Claude A / Claude B 패턴

Anthropic이 권장하는 **이중 인스턴스 개선법**:

| 역할 | 하는 일 |
|------|---------|
| **Claude A** (작성자) | 스킬을 분석하고 개선안 작성 |
| **Claude B** (테스터) | 개선된 스킬로 실제 작업 수행 |
| **사용자** | Claude B의 행동을 관찰하고 피드백 |

```
Claude A: "description에 키워드를 추가하자"
    → 스킬 수정
        → Claude B: 실제 작업 수행
            → 사용자: "이 경우에 스킬이 안 불린다"
                → Claude A: "트리거 조건을 보강하자"
                    → 반복
```

이 패턴은 **Ralph Loop**과 본질적으로 같다 — 개발 → 리뷰 → 수정 → 재리뷰의 반복.

---

### 10. Eval 실행 — with-skill vs baseline

스킬의 효과를 **정량적으로 측정**하는 기능.

#### Eval 파일 구조

```json
{
  "skills": ["pdf-processing"],
  "query": "이 PDF 파일에서 텍스트를 추출해서 output.txt에 저장해줘",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "적절한 PDF 처리 라이브러리를 사용하여 파일을 읽는다",
    "모든 페이지에서 텍스트를 누락 없이 추출한다",
    "추출된 텍스트를 output.txt에 명확하게 저장한다"
  ]
}
```

#### 실행 방식

```
1. Baseline 실행 — 스킬 없이 같은 query 실행
2. With-skill 실행 — 스킬 있이 같은 query 실행
3. 결과 비교 — expected_behavior 기준으로 채점
4. generate_review.py — 비교 리포트 생성
```

#### 평가 기준 설계 팁

- **구체적인 행동**을 기술 — "잘 처리한다" (X), "pdfplumber를 사용하여 텍스트를 추출한다" (O)
- **최소 3개의 시나리오** — 기본, 엣지 케이스, 에러 상황
- **모든 대상 모델에서 테스트** — Haiku에서 되면 Opus에서도 됨은 보장 못 함

---

### 11. 벤치마크 — 분산 분석

단순히 "되냐 안 되냐"가 아니라, **얼마나 안정적으로 되는지**를 측정한다.

#### 왜 분산 분석인가

LLM은 **비결정적**이다. 같은 프롬프트로 10번 돌리면 결과가 다를 수 있다.

```
실행 1: ✅ 성공
실행 2: ✅ 성공
실행 3: ❌ 실패 (다른 라이브러리 사용)
실행 4: ✅ 성공
실행 5: ✅ 성공
```

한 번 성공했다고 끝이 아니다. **성공률과 분산**을 봐야 한다.

#### 프로세스

```
같은 Eval을 N번 반복 실행
    │
    ├── aggregate_benchmark 스크립트로 결과 통합
    ├── 성공률, 평균 소요 시간, 토큰 사용량 집계
    └── 분산이 크면 → 스킬 지침이 모호하다는 신호
```

#### 분산이 큰 경우 대응

| 증상 | 원인 | 해결 |
|------|------|------|
| 성공/실패가 반반 | description이 모호 | 키워드 + 트리거 조건 강화 |
| 매번 다른 방법 사용 | 선택지가 너무 많음 | 기본값 하나 명시 |
| Haiku에서만 실패 | 지침이 고수준 모델에 의존 | 더 구체적인 가이드 추가 |

---

### 12. Description 최적화 루프

스킬의 **발견율(트리거 정확도)**을 반복적으로 개선하는 기능.

#### 문제

description이 부실하면:

```
사용자: "이 PDF에서 표 뽑아줘"
Claude: (100개 스킬의 description 스캔)
Claude: "매칭되는 스킬 없음" → 스킬 없이 직접 처리
```

스킬이 있는데도 **안 불리는** 것이 가장 큰 문제다.

#### 최적화 루프 (scripts.run_loop)

```
현재 description
    │
    ├── 1. 다양한 query로 트리거 테스트
    │   "PDF 텍스트 추출" → ✅ 트리거됨
    │   "이 문서에서 표 뽑아줘" → ❌ 트리거 안 됨
    │   "PDF 양식 채워줘" → ❌ 트리거 안 됨
    │
    ├── 2. 실패한 query 분석 → 누락된 키워드 식별
    │   "표", "양식" 키워드가 description에 없음
    │
    ├── 3. description 수정
    │   + "표 추출, 양식 채우기" 추가
    │
    ├── 4. 재테스트
    │   "이 문서에서 표 뽑아줘" → ✅ 트리거됨
    │   "PDF 양식 채워줘" → ✅ 트리거됨
    │
    └── 5. False positive 확인
        "엑셀 파일 열어줘" → ❌ 트리거 안 됨 (정상)
```

#### 좋은 description의 체크리스트

- [ ] 3인칭으로 작성
- [ ] "뭘 하는지" + "언제 쓰는지" 둘 다 포함
- [ ] 사용자가 쓸 법한 키워드 포함
- [ ] 너무 넓지 않음 (false positive 방지)
- [ ] 1024자 이내
- [ ] XML 태그 미포함

---

## 마무리

### 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| **스킬** | 코드가 아닌 프롬프트 확장. Claude를 임시로 특화 에이전트로 변환 |
| **Progressive Disclosure** | 메타데이터 → SKILL.md → 참조 파일, 3단계로 필요한 만큼만 로드 |
| **description** | 스킬 발견의 열쇠. LLM이 자연어로 매칭하므로 키워드와 컨텍스트가 핵심 |
| **skill-creator** | 생성 → Eval → 벤치마크 → 최적화의 체계적 루프 |
| **Eval 우선** | 상상한 요구사항이 아닌 실제 실패 지점을 해결하는 스킬을 만든다 |

### 스킬 생성 전체 흐름

```
의도 파악
    → init_skill.py (템플릿 생성)
        → SKILL.md 작성 (frontmatter + body)
            → Eval 생성 (2-3개 시나리오)
                → Baseline vs With-skill 비교
                    → 벤치마크 (N회 반복, 분산 분석)
                        → Description 최적화 루프
                            → 팀 피드백 → 반복
```

### Ralph Loop과의 유사성

skill-creator의 Eval → 수정 → 재검증 루프는 [Ralph Loop의 반복 개선 구조](../Agent%20Team부터%20Ralph%20Loop까지/claude-code-agent-team_2026-03-01.md)와 본질적으로 같다:

| Ralph Loop | skill-creator |
|-----------|---------------|
| REVIEW.md에 피드백 기록 | Eval 결과로 피드백 |
| dev가 REVIEW.md 읽고 수정 | description/body 수정 |
| Critical 0까지 반복 | 트리거 정확도 100%까지 반복 |
| `<promise>DONE</promise>` | 모든 Eval 통과 |

**핵심은 같다**: 한 번에 완벽하게가 아니라, **빠르게 만들고, 측정하고, 고치는 루프**.

---

## 레퍼런스

- [Skill authoring best practices — Anthropic Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Equipping Agents for the Real World with Agent Skills — Anthropic Engineering](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Agent Skills: A First Principles Deep Dive — Lee Han Chung](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [Extend Claude with Skills — Claude Code Docs](https://code.claude.com/docs/en/skills)
- [skill-creator — GitHub (anthropics/skills)](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [How to create custom Skills — Claude Help Center](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
