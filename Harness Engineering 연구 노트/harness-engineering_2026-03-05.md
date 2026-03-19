# Harness Engineering 연구 노트

> 최종 수정: 2026-03-05

## TL;DR

AI 에이전트가 장시간 안정적으로 작업할 수 있게 감싸는 구조(Harness)를 연구했다. Ralph Loop을 Open-loop에서 Closed-loop(피드백 주입)으로 개선하고, 8회 실험으로 반복 루프의 효과를 검증. 실전(nxtcloud-homepage)에도 적용했다.

## 목차

1. Harness란?
2. 비유: 매일 기억이 리셋되는 천재 신입사원
3. Anthropic 공식 가이드 — 핵심 구성요소
4. Ralph Loop — 반복의 힘
5. 내 실험 — Ralph Loop을 변형한 Harness 엔지니어링
6. 실험 결과 — 8회 실측 데이터
7. 실전 적용 — nxtcloud-homepage
8. 평가 — Harness 엔지니어링을 잘 했는가?
9. 핵심 결론

---

## 1. Harness란?

**AI 에이전트가 안정적으로, 일관되게, 장시간 작업할 수 있도록 감싸는 구조/프레임워크**

프롬프트 튜닝에 의존하는 대신, **파일 기반 상태 관리 + 작업 분할 + 자동 검증**으로 에이전트의 품질과 안정성을 높이는 접근법이다.

### 왜 필요한가?

AI 에이전트에게 "이거 만들어줘"라고 하면 흔히 발생하는 문제:

- 컨텍스트가 길어지면 **앞에서 뭘 했는지 까먹음**
- 여러 세션에 걸쳐 작업하면 **진행 상황이 유실됨**
- 한 번에 다 하려다가 **품질이 떨어지거나 방향을 잃음**

---

## 2. 비유: 매일 기억이 리셋되는 천재 신입사원

AI 에이전트 = **매일 아침 기억이 완전히 리셋되는 천재 신입사원**

### Harness 없이 일하면?

```
월요일: "로그인 페이지 만들어줘" → 잘 만듦
화요일: 기억 리셋 → "...제가 뭘 하고 있었죠?"
수요일: 또 리셋 → 로그인 페이지를 처음부터 다시 만들기 시작
```

능력은 뛰어난데, 기억이 없으니 매번 헤매고, 중복 작업하고, 방향을 잃는다.

### Harness를 도입하면?

이 신입에게 **업무 시스템**을 세팅해주는 것:

| 비유 | Harness 구성요소 | 하는 일 |
|------|----------------|--------|
| 프로젝트 위키 | `CLAUDE.md` | "우리 팀은 이런 프로젝트야" |
| 칸반 보드 | 기능 목록 파일 | "전체 할 일은 이거야" |
| 업무 일지 | 진행 상황 로그 | "어제까지 여기까지 했어" |
| Git 커밋 | 작업 저장 | "네가 한 일은 여기 저장돼 있어" |
| QA 체크리스트 | 자동 테스트 | "다 만들면 이걸로 검증해" |

화요일 아침에 기억이 리셋돼도:

1. 업무 일지를 읽고 → "아, 로그인 페이지까지 완성했구나"
2. 칸반 보드를 보고 → "다음은 회원가입이네"
3. Git 히스토리를 보고 → "코드는 여기 있구나"
4. 바로 이어서 작업 시작!

> **Harness = 기억이 리셋되는 천재가 매일 아침 즉시 업무에 복귀할 수 있게 해주는 업무 시스템**

---

## 3. Anthropic 공식 가이드 — 핵심 구성요소

> 출처: [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

| 구성요소 | 역할 |
|---------|------|
| **Initializer Agent** | 첫 세션에서 환경 세팅, 기능 목록 작성, git 초기 커밋 |
| **Coding Agent** | 이후 세션에서 한 번에 하나의 기능만 작업, 커밋으로 기록 |
| **진행 상황 파일** | 기능 목록(JSON), 진행 로그를 파일로 관리 |
| **테스트 도구** | 브라우저 자동화로 실제 end-to-end 검증 |

인간 엔지니어의 작업 방식에서 영감을 받은 구조다.

### Claude Code에서의 Harness

Claude Code 자체가 하나의 harness이다. Claude 모델을 감싸서:

- 도구(파일 읽기/쓰기, 터미널 등)를 제공
- 컨텍스트를 관리
- 실행 루프를 제어

그 위에 **CLAUDE.md, Skills, Hooks, MCP, 서브에이전트**를 조합하면 더 강력한 커스텀 harness를 만들 수 있다.

---

## 4. Ralph Loop — 반복의 힘

### Ralph Loop이란?

Geoffrey Huntley가 만든 반복적 AI 개발 방법론.

```bash
while :; do
  cat PROMPT.md | claude-code --continue
done
```

**같은 프롬프트를 계속 반복해서 먹인다.** 단순하지만 강력하다.

### 작동 방식

```
1. Claude가 같은 프롬프트를 받음
2. 작업 수행 (파일 수정, 코드 작성)
3. 종료하려고 함
4. Stop hook이 가로채서 같은 프롬프트를 다시 줌
5. Claude가 자기가 이전에 한 작업을 파일/git에서 봄
6. 이전 작업 위에 점진적으로 개선
7. 완료될 때까지 반복
```

### 프로그래밍 개념과의 관계

Ralph Loop은 프로그래밍의 기본 구조와 정확히 대응된다.

#### 반복문(while)으로 보기

```typescript
// Ralph Loop의 본질은 조건부 반복문이다
while (critical > 0) {
  const code = dev(prompt);        // 개발
  const review = reviewer(code);   // 리뷰
  critical = review.criticalCount; // 조건 체크
}
// critical === 0 이면 탈출
```

단순 `while (true)` 와 조건부 `while (condition)` 의 차이를 생각해보자:

```typescript
// 단순 반복 — 종료 조건이 불명확
while (true) {
  claude.run(SAME_PROMPT);
  // "끝났나?" 판단을 에이전트에게 맡김
  // 종료 시점이 불분명 → 무한 루프 위험
}

// 조건부 반복 — 측정값으로 종료 판단
while (review.critical > 0) {               // ← 측정값이 종료 조건
  const code = dev.run(PROMPT);
  review = reviewer.run(code);               // ← 결과를 측정
}
// review.critical === 0 이면 탈출
```

프로그래밍에서 `while (true) { if (maybe) break; }` 보다 `while (condition)` 이 더 안전한 것과 같은 원리다. Ralph Loop의 `--completion-promise`와 `--max-iterations`가 바로 이 종료 조건 역할을 한다.

#### 재귀함수로 보기

```typescript
function ralphLoop(prompt: string, iteration: number = 1): Code {
  const code = dev(prompt);
  const review = reviewer(code);

  // Base Case (종료 조건) = completion promise
  if (review.critical === 0) return code;

  // Stack Overflow 방지 = max-iterations
  if (iteration >= MAX_ITERATIONS) return code;

  // 재귀 호출 — 같은 prompt, 하지만 파일은 변경된 상태
  return ralphLoop(prompt, iteration + 1);
}
```

재귀함수에서 핵심적인 세 가지 요소가 Ralph Loop에 그대로 존재한다:

| 재귀함수 요소 | 역할 | Ralph Loop 대응 |
|-------------|------|----------------|
| **Base Case** | 재귀 종료 조건 | `--completion-promise` ("DONE" 출력하면 멈춤) |
| **Recursive Case** | 자기 자신 호출 | stop hook이 같은 prompt를 다시 먹임 |
| **Stack Overflow 방지** | 최대 깊이 제한 | `--max-iterations 10` |

그리고 Ralph Loop의 상태 관리 방식은 재귀의 두 가지 패턴과 대응된다:

```typescript
// 인자 없는 재귀 — 전역 상태(파일)에 의존
function ralph() {
  const state = readFiles();        // 전역 상태에서 추측
  improve(state);
  if (done()) return;               // 종료 조건도 스스로 판단
  ralph();                          // 인자 없이 같은 호출
}
```

이건 원본 Ralph Loop의 구조다. 파일 시스템이라는 전역 상태에 의존하고, 뭐가 바뀌었는지 스스로 파악해야 한다.

#### 1:1 매핑 요약

| 프로그래밍 개념 | Ralph Loop |
|---------------|------------|
| `while (조건)` | `while (critical > 0)` |
| **Base Case** (재귀 종료 조건) | `--completion-promise` ("DONE" 출력하면 멈춤) |
| **Recursive Case** (자기 호출) | stop hook이 같은 prompt를 다시 먹임 |
| **Stack Overflow 방지** | `--max-iterations 10` |
| **상태 변이** (변수 업데이트) | 파일 시스템 + git 히스토리 변경 |
| **무한 루프** | promise도 없고 max-iterations도 없으면 영원히 돔 |
| **수렴 조건** | Critical 0 도달 → 탈출 |

### Prompt 불변 원칙과 상태 관리

Ralph Loop에서 **Prompt는 불변(immutable)**이고, **상태는 파일 시스템(mutable)**에 저장된다.

```
┌─────────────────────────────────────────────┐
│  PROMPT (불변)                               │
│  "Todo 앱 만들어. Critical 0이면 끝."         │
└──────────────┬──────────────────────────────┘
               │ 매 iteration마다 동일하게 전달
               ▼
┌──────────────────────────────────────────────┐
│  Claude (매 iteration)                        │
│                                              │
│  1. 같은 prompt를 받음                        │
│  2. 파일 시스템을 읽음 ← 여기서 "지금 상태" 파악 │
│  3. git log를 봄 ← "뭘 했는지" 파악            │
│  4. 이어서 작업                               │
└──────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  파일 시스템 (가변 = 상태 저장소)               │
│                                              │
│  - 소스 코드 (수정됨)                          │
│  - git history (커밋 쌓임)                     │
│  - .claude/.ralph-loop.local.md (iteration 추적)│
└──────────────────────────────────────────────┘
```

#### 프로그래밍 비유

```typescript
const PROMPT = "Todo 앱 만들어" as const;  // ← 불변 (const)
const fileSystem = new MutableState();     // ← 가변 (파일 시스템)

while (true) {
  claude.run(PROMPT, fileSystem);  // 매번 같은 prompt + 변경된 파일 상태
  if (fileSystem.read("review.md").critical === 0) break;
}
```

#### 순수함수와의 비교

```
순수함수:  f(x) → 항상 같은 결과 (부수효과 없음)
Ralph:    f(같은 prompt) → 매번 다른 결과 (파일이라는 부수효과 존재)
```

같은 prompt인데 결과가 다른 이유 = **파일 시스템이라는 외부 상태**를 읽기 때문.

이건 클로저(closure)에서 외부 변수를 캡처하는 것과 비슷:

```typescript
let files = {};  // 외부 상태

const ralph = () => {
  // 항상 같은 함수지만, files가 바뀌니까 동작이 달라짐
  const currentState = readFiles(files);
  files = improve(currentState);
};
```

#### 상태 관리 요약

| 요소 | 성질 | 역할 |
|------|------|------|
| **Prompt** | **불변** (immutable) | "뭘 해야 하는지" 지시 |
| **파일 시스템** | **가변** (mutable) | 작업 상태 저장 |
| **git history** | **append-only** | 뭘 했는지 기록 |
| **`.ralph-loop.local.md`** | **가변** | iteration 카운터 추적 |

> Prompt는 "목표"이고, 파일은 "현재 위치"다.
> 목표는 안 변하지만, 매 iteration마다 현재 위치가 목표에 가까워지는 구조.

---

## 5. 내 실험 — Ralph Loop을 변형한 Harness 엔지니어링

### 원본 vs 내 버전

| | 원본 Ralph Loop | 내 버전 |
|--|----------------|--------|
| 에이전트 | 1명 (혼자 다 함) | 3명 (dev / reviewer / tester 역할 분리) |
| 피드백 | 없음 (스스로 파일에서 추측) | 있음 (리뷰/테스트 결과를 prompt에 주입) |
| 품질 기준 | 없음 | Critical(90+) / Important(80-89) 분류 체계 |
| 검증 | 없음 | 14개 스킬 기반 자동 검증 |
| 병렬 실행 | 없음 | reviewer + tester 동시 수행 |
| 스펙 문서 | 없음 | pm 에이전트로 만든 상세 spec.md |

### 핵심 변형: Open-loop → Closed-loop (피드백 주입)

#### Open-loop vs Closed-loop이란?

제어 공학에서 나온 개념으로, **시스템이 결과를 보고 스스로 조정하느냐**의 차이다.

**Open-loop (개루프 제어)** — 결과를 확인하지 않고 입력만 반복하는 구조

```
일상 비유: 토스터기
→ "3분 구워" 버튼을 누르면, 빵이 탔든 덜 익었든 무조건 3분만 돌림
→ 결과(빵 상태)를 확인하지 않음
→ 운이 좋으면 잘 되고, 나쁘면 실패

프로그래밍 비유: 하드코딩된 반복
for (let i = 0; i < 3; i++) {
  improve(code);  // 결과가 어떻든 그냥 3번 돌림
}
```

**Closed-loop (폐루프 제어)** — 결과를 측정해서 다음 입력에 반영하는 구조

```
일상 비유: 에어컨 온도 조절
→ "25도로 맞춰" → 현재 28도 → 냉방 세게 → 현재 26도 → 냉방 약하게 → 25도 도달 → 멈춤
→ 결과(현재 온도)를 계속 측정하고 그에 맞게 조정
→ 목표에 수렴

프로그래밍 비유: 조건부 반복
while (review.critical > 0) {
  fix(review.feedback);           // 피드백 기반으로 수정
  review = reviewer(code);        // 결과 다시 측정
}
// critical === 0이면 탈출 → 목표 수렴
```

**핵심 차이:**

```
Open-loop:   입력 → 시스템 → 출력  (끝. 출력을 안 봄)
Closed-loop: 입력 → 시스템 → 출력 → 측정 → 입력에 반영 → 시스템 → ... (수렴까지 반복)

              ┌──────── 피드백 ────────┐
              │                       │
         입력 ──→ 시스템 ──→ 출력 ──→ 측정
              ↑                       │
              └───────────────────────┘
```

| | Open-loop | Closed-loop |
|--|-----------|-------------|
| **결과 확인** | 안 함 | 매번 측정 |
| **자기 수정** | 불가 | 가능 |
| **수렴 보장** | 운에 의존 | 구조적으로 수렴 |
| **일상 예시** | 토스터기, 세탁기 | 에어컨, 크루즈 컨트롤 |
| **Ralph Loop** | 원본 (같은 prompt 반복) | 내 버전 (피드백 주입) |

#### 내 실험에서의 적용

원본 Ralph Loop은 prompt가 완전 불변이다. 내 버전은 **베이스 prompt는 불변이지만, 리뷰/테스트 결과를 매 iteration마다 주입**한다.

```
원본 Ralph Loop:
  같은 Prompt → Claude → 파일 변경 → 같은 Prompt → Claude → ...
  Claude가 파일을 읽고 "뭐가 문제인지" 스스로 파악해야 함

내 버전:
  Base Prompt + 리뷰 결과 → dev → 코드 수정
                                     ↓
                            reviewer + tester
                                     ↓
                            결과를 prompt에 주입
                                     ↓
  Base Prompt + 새로운 결과 → dev → 코드 수정 → ...
```

#### 피드백 주입이 프로그래밍 구조를 바꾼다

피드백 주입은 섹션 4에서 설명한 반복문/재귀 구조를 한 단계 발전시킨다.

**재귀함수: 인자 없음 → 인자 있음으로 변화**

```typescript
// 원본 — 인자 없는 재귀 (섹션 4의 기본 구조)
function ralph() {
  const state = readFiles();        // 전역 상태에서 추측
  improve(state);
  if (done()) return;
  ralph();                          // 같은 호출
}

// 내 버전 — 인자 있는 재귀, 피드백이 명시적으로 전달됨
function ralph(feedback?: ReviewResult) {
  if (feedback) fix(feedback);      // 뭘 고쳐야 하는지 정확히 앎
  else build();

  const review = reviewer(code);
  const test = tester(code);

  if (review.critical === 0) return;
  ralph({ ...review, ...test });    // 결과를 다음 호출에 전달
}
```

**반복문: 단일 스레드 → 병렬 비동기로 변화**

```typescript
// 내 버전은 reviewer + tester를 병렬로 실행한다
async function ralphLoop() {
  let code = await dev.build(BASE_PROMPT);

  while (true) {
    // reviewer와 tester를 동시에 실행 (Promise.all)
    const [review, test] = await Promise.all([
      reviewer.review(code),   // 비동기 측정 1
      tester.test(code),       // 비동기 측정 2
    ]);

    if (review.critical === 0) break;  // 수렴 → 탈출
    code = await dev.fix(BASE_PROMPT, { review, test }); // 피드백 주입
  }
}
```

**종합: 원본 vs 피드백 주입이 바꾼 프로그래밍 구조**

| 프로그래밍 개념 | 원본 Ralph Loop | 내 버전 (피드백 주입) |
|---------------|----------------|---------------------|
| `while (true)` + 에이전트 판단 | ✅ | |
| `while (조건)` + 측정값 기반 | | ✅ |
| 인자 없는 재귀 `f()` | ✅ | |
| 인자 있는 재귀 `f(feedback)` | | ✅ |
| 전역 상태 의존 (파일 읽기) | ✅ | 베이스만 |
| 명시적 데이터 흐름 | | ✅ |
| 단일 스레드 | ✅ | |
| 병렬 실행 (`Promise.all`) | | ✅ |

#### 왜 이게 더 좋은가

| | 원본 Ralph Loop | 내 버전 |
|--|----------------|--------|
| **문제 발견** | Claude가 파일 읽고 스스로 추측 | 리뷰어/테스터가 **명시적으로 알려줌** |
| **수정 방향** | 불확실 (뭐가 문제인지 모를 수도) | **정확함** ("Critical: stale closure 발견") |
| **수렴 속도** | 느림 (헤맬 수 있음) | 빠름 (뭘 고칠지 바로 앎) |
| **비유** | 시험 보고 점수만 받음 | 시험 보고 **오답노트**까지 받음 |

#### 제어 공학 관점

```
원본:    Open-loop 제어  — 출력을 확인 안 하고 입력만 반복
내 버전: Closed-loop 제어 — 출력(리뷰 결과)을 입력에 피드백

         ┌─────────────────────────────────┐
         │          Feedback Loop          │
         │                                 │
Prompt ──┴──→ dev ──→ code ──→ reviewer ───┤
                                 tester ───┘
                                   │
                          결과를 prompt에 주입
```

이건 **PID 제어기**나 **경사하강법(Gradient Descent)**과 같은 원리:

```
경사하강법: loss(오차)를 계산 → 다음 step에 반영 → 수렴
내 루프:    review(피드백)를 계산 → 다음 iteration에 반영 → Critical 0 수렴
```

> 원본 Ralph Loop은 **"다시 해봐"**이고,
> 내 버전은 **"이거 고쳐봐"**이다.

---

## 6. 실험 결과 — 8회 실측 데이터

### 실험 조건 (모든 조합 동일)

| 항목 | 내용 |
|------|------|
| 과제 | Todo 앱 (CRUD + localStorage + 에러 처리 + 접근성) |
| 스펙 | `docs/specs/todo/spec.md` (pm 에이전트 + /spec-review 확정) |
| 에이전트 | frontend (7 skills), frontend-reviewer (6 skills), frontend-test (6 skills) |
| 스킬 | 14개 규칙 |
| 프롬프트 | 동일한 함정 명시 (무한 루프, alert 금지, useEffect 저장 금지 등) |
| 모델 | Sonnet 4.6 (dev, reviewer) |

**바뀐 것은 "조합(구조)" 하나뿐이다.**

### 2차 실험

| | 조합 5 | 조합 6 | 조합 7 | 조합 8 |
|--|--------|--------|--------|--------|
| **방식** | Agent Team | Team+Worktree | Ralph Loop+Team | Ralph Loop Parallel |
| **시간** | ~9분 14초 | ~7분 45초 | ~10분 10초 | ~15분 15초 |
| **Iterations** | 1 | 1 | 1 | 2 |
| **테스트** | 82개 | 65개 | 78개 | 109개 |
| **커버리지** | 86.72% | 86.94% | **100%** | **100%** |
| **Critical** | **1** | **2** | **0** | **0** |

### 3차 실험 (동일 환경 재실험)

| | 조합 9 | 조합 10 | 조합 11 | 조합 12 |
|--|--------|---------|---------|---------|
| **방식** | Agent Team | Team+Worktree | Ralph Loop+Team | Ralph Loop Parallel |
| **시간** | ~6분 57초 | ~7분 41초 | ~9분 53초 | ~12분 39초 |
| **Iterations** | 1 | 1 | 2 | 2 |
| **테스트** | 106개 | 99개 | 88개 | 115개 |
| **커버리지** | 99.33% | 96.64% | 97.59% | **100%** |
| **Critical** | **1** | **1** | **0** | **0** |

### 종합 결과

| | 1회성 (4회) | 반복 (4회) |
|--|------------|-----------|
| Critical 0 달성 | **0/4 (0%)** | **4/4 (100%)** |
| 평균 Critical | 1.25개 | 0개 |

**8회 실험에서 단 한 번도 예외 없이:** 1회성은 Critical이 남고, 반복은 Critical 0에 수렴.

### 핵심 발견 5가지

1. **반복 루프가 결정적** — 1회성은 리뷰어가 버그를 찾아도 수정할 기회가 없다. 구조적 차이.
2. **구조가 프롬프트보다 중요** — 같은 프롬프트, 같은 스킬인데 조합만 바꿔서 결과가 달라짐.
3. **시간 vs 품질은 정비례** — Agent Team ~8분(C:1) ↔ Ralph Parallel ~14분(C:0).
4. **LLM 비결정성을 반복이 흡수** — 편차가 있어도 반복 루프가 자동으로 수렴시킴.
5. **새로운 버그 유형 출현** — 프롬프트에 명시한 함정은 사라지지만, 명시하지 않은 함정이 새로 등장. 반복 루프가 안전망.

### 상황별 선택 가이드

```
"프로토타입 빨리 필요해"          → Agent Team (~8분, Critical 있어도 OK)
"병렬 개발인데 파일 충돌 걱정"    → Team + Worktree (~8분, 물리적 격리)
"프로덕션 코드야, 버그 없어야 해" → Ralph Loop + Team (~10분, Critical 0) ⭐
"품질에 타협 없어"               → Ralph Loop Parallel (~14분, 모든 이슈 수정)
```

**기본값: Ralph Loop + Team.** 에이전트가 좋으면 1 iteration으로 끝나고(사실상 1회성 속도), 나쁘면 자동으로 여러 번 돌아 품질을 맞춘다. 어느 쪽이든 손해가 없다.

---

## 7. 실전 적용 — nxtcloud-homepage

ralph-loop에서 검증한 harness 패턴을 실제 프로덕션 프로젝트에 적용한 사례.

### 파일 기반 상태 관리

```
docs/
├── plans/                              ← 기억이 리셋돼도 읽으면 되는 문서들
│   ├── prd.md                          ← 968줄 PRD
│   ├── user-flow.md                    ← 1,051줄 유저 플로우
│   ├── information-architecture.md     ← 715줄 정보 구조
│   ├── error-scenarios.md              ← 391줄 에러 시나리오
│   ├── homepage-renewal-design.md      ← 340줄 설계서
│   └── homepage-renewal-implementation.md ← 1,479줄, 24개 Task
│
└── conversations/                      ← 세션 간 진행 상황 기록
    ├── ui-mockup-and-team-setup.md     ← "여기까지 했고, 팀은 이렇게 구성"
    └── phase1-2-implementation.md      ← "Task 1-21 완료, 22-24 남음"
```

### 비유 → 실제 매핑

| 신입사원 비유 | nxtcloud-homepage 실제 적용 |
|-------------|----------------------------|
| 프로젝트 위키 | `prd.md` + `design.md` + `information-architecture.md` |
| 칸반 보드 | `implementation.md`의 24개 Task 목록 |
| 업무 일지 | `conversations/` 폴더의 대화 기록 |
| 역할 분담표 | 5인 에이전트 팀 (Lead + 4 dev) |
| QA 체크리스트 | `error-scenarios.md` + `user-flow.md` 엣지케이스 |

### 에이전트 팀 구조

```
Lead (Opus) ─── 전체 조율, Phase 1 직접 수행
  ├── homepage-dev (Sonnet) ─── Task 6-10 (홈페이지)
  ├── pages-dev (Sonnet)    ─── Task 11-15 (서브페이지)
  ├── admin-dev (Sonnet)    ─── Task 16-21 (백오피스)
  └── test-dev (Sonnet)     ─── 테스트 작성

  Phase 1 (Foundation)  → 리드 직접 수행
  Phase 2 (병렬 개발)   → 4개 에이전트 동시 spawn
  Phase 3 (리뷰/통합)   → 머지 + 검증
```

결과: 20개 라우트, 101개 테스트 통과

---

## 8. 평가 — Harness 엔지니어링을 잘 했는가?

### 잘한 점 — 데이터가 증명함

#### 1. 실험으로 검증했다

대부분은 "이게 좋을 것 같다"로 끝난다. 하지만 이 실험은:

```
같은 조건 × 4가지 조합 × 2회 = 8회 실험
→ 변인 통제 (프롬프트, 스킬, 에이전트 동일)
→ 바꾼 건 "구조" 하나뿐
→ 결과: 반복 조합만 Critical 0 (4/4, 100%)
```

"느낌"이 아니라 **실측 데이터**로 구조의 효과를 증명.

#### 2. 원본을 의미있게 개선했다

Open-loop → Closed-loop 전환 (피드백 주입)은 구조적으로 의미 있는 개선이다.

#### 3. 실전에 적용했다

실험에서 끝나지 않고 nxtcloud-homepage에서 실제로 적용:
- 4,000줄+ 기획 문서
- 5인 에이전트 팀
- 24개 Task → 20개 라우트, 101개 테스트 통과

#### Harness 엔지니어링 3단계 달성

```
1단계: 구조를 설계한다          ✅ (4가지 조합 설계)
2단계: 실험으로 검증한다        ✅ (8회 실험, 데이터 수집)
3단계: 실전에 적용한다          ✅ (nxtcloud-homepage)
```

> 대부분은 1단계(설계)에서 멈추거나, 검증 없이 바로 3단계(적용)로 간다.
> **설계 → 검증 → 적용** 사이클을 돈 건 엔지니어링으로서 제대로 한 것.

### 더 파고들 수 있는 영역

| 질문 | 아직 검증 안 된 것 |
|------|-------------------|
| **규모 확장** | Todo 앱(단일 기능)에서 검증됨 → 대규모 멀티 서비스에서도 통할까? |
| **비용 효율** | Ralph Parallel은 ~14분, 비용 최고 → 비용 대비 최적점은 어디? |
| **피드백 주입의 정량적 효과** | 피드백 주입 vs 미주입을 동일 조건에서 비교하면? |
| **에이전트/스킬 품질의 영향** | 구조가 같아도 스킬 품질이 낮으면 수렴 속도가 얼마나 느려지나? |

---

## 9. 핵심 결론

```
ralph-loop        = Harness를 "연구"한 것 (어떤 구조가 좋은가?)
nxtcloud-homepage = Harness를 "실전 적용"한 것 (실제 프로젝트에 써먹기)
```

> **AI에게 "잘 해줘"라고 말하는 것보다,
> AI가 잘할 수밖에 없는 환경을 만들어주는 것이 더 중요하다.**

스펙 문서, 진행 기록, 역할 분리, 반복 루프 — 이 **구조**가 프롬프트 기술보다 결과에 더 큰 영향을 미친다.

---

## 레퍼런스

- [Anthropic - Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Harness Engineering 101 - muraco.ai](https://muraco.ai/en/articles/harness-engineering-claude-code-codex/)
- [Agent Harness: Understanding Claude Code's Superpower Engine - Medium](https://medium.com/@fruitful2007/agent-harness-understanding-claude-codes-superpower-engine-85e35a7ec764)
- [claude-code-harness (GitHub)](https://github.com/Chachamaru127/claude-code-harness)
- [Open Harness](https://www.maxgfeller.com/blog/open-harness/)
- [everything-claude-code (GitHub)](https://github.com/affaan-m/everything-claude-code)
- [Ralph Wiggum Technique - Geoffrey Huntley](https://ghuntley.com/ralph/)
- [Ralph Orchestrator (GitHub)](https://github.com/mikeyobrien/ralph-orchestrator)
