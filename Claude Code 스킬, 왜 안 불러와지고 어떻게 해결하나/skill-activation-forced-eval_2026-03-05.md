# Claude Code 스킬, 왜 안 불러와지고 어떻게 해결하나

> 최종 수정: 2026-03-05

## TL;DR

Claude Code 스킬은 알아서 활성화되어야 하지만, 현실은 그렇지 않다. **강제 평가(Forced Eval) 훅**을 쓰면 활성화 성공률을 **20% → 84%**까지 끌어올릴 수 있다.

## 목차

1. 문제: 스킬이 있는데 안 쓴다
2. 테스트 환경
3. 네 가지 훅(Hook) 방식 비교
4. 결과 데이터
5. 강제 평가 훅이 통하는 이유
6. 적용 방법
7. 어떤 훅을 써야 할까
8. 핵심 정리

---

## 1. 문제: 스킬이 있는데 안 쓴다

Claude Code의 스킬(Skill)은 설명(description)을 기반으로 자율 활성화되도록 설계되어 있다.
→ 공식 문서에도 "사용자의 요청에 근거해 스킬 사용 여부를 스스로 판단한다"고 되어 있다.

하지만 실제로는 스킬을 무시하고 자기가 아는 선에서 답변을 때운다.

> **비유하면 이렇다.** 회사에 잘 정리된 사내 위키가 있는데, 신입 개발자한테 "필요하면 참고해"라고 말해놓으면 대부분 위키를 안 보고 자기가 아는 대로 처리한다. Claude Code도 마찬가지다.

---

## 2. 테스트 환경

원문 저자는 SvelteKit 도메인의 스킬 4개를 만들고, 5가지 프롬프트를 각각 10회씩 총 200회 이상 테스트했다.

### 테스트에 사용된 스킬 4개

| 스킬 | 역할 |
| --- | --- |
| svelte5-runes | Svelte 5 룬즈 시스템 ($state, $derived 등) |
| sveltekit-data-flow | 데이터 로딩, 폼 액션, 서버 로드 함수 |
| sveltekit-structure | 파일 기반 라우팅, 레이아웃, SSR |
| sveltekit-remote-functions | query()/command() 기반 원격 함수 |

### 테스트 프롬프트 예시

- **폼/라우트 생성** → 구조 + 데이터 흐름 + 룬즈, 3개 스킬이 동시에 필요한 복합 프롬프트
- **데이터 로딩** → 단일 스킬(data-flow)만 필요
- **Svelte5 룬즈** → 단일 스킬(runes)만 필요

복합 스킬이 필요한 프롬프트일수록 단순한 방법은 무너진다.

---

## 3. 네 가지 훅(Hook) 방식 비교

### 1. 훅 없음 (기본 상태)

아무 지침도 없이 Claude에게 맡기는 것. 스킬을 거의 활성화하지 않는다.

### 2. 단순 지시문 훅 — 성공률 약 20%

"프롬프트가 스킬 키워드와 일치하면 해당 스킬을 활성화하세요"라는 한 줄짜리 지침을 넣어두는 방식.

- **비유:** 주방 벽에 "레시피북 참고하면 좋겠죠?"라는 포스트잇을 붙여놓은 것. 주방장(Claude)이 보긴 보는데, 바쁘면 무시한다.

### 3. 강제 평가(Forced Eval) 훅 — 성공률 84%

작업 전에 반드시 모든 스킬을 하나씩 평가하고, YES로 판단한 스킬은 활성화한 뒤에야 구현으로 넘어가게 강제하는 방식.

- **비유:** 주문이 들어오면 주방장이 조리 시작 전에 체크리스트를 반드시 작성해야 하는 규칙.
    1. "이탈리안 레시피북 필요한가? → YES, 파스타 요리니까"
    2. "한식 레시피북 필요한가? → NO, 이탈리안이라서"
    3. YES 체크한 레시피북을 실제로 펼쳐서 읽는다
    4. **그 다음에야 조리 시작**

    → 체크리스트를 안 쓰면 조리를 시작할 수 없다는 강제 규칙이 핵심이다.

### 4. LLM 평가 훅 — 성공률 80%

Claude Code가 프롬프트를 보기 전에, 별도의 Claude API 호출로 "어떤 스킬이 필요한지" 미리 판단하는 방식.

- **비유:** 주문서를 받는 홀 매니저가 주방에 넘기기 전에 "이 주문이면 이탈리안 레시피북 필요할 거야"라고 메모를 붙여주는 것. 대부분 잘 맞추지만, 복잡한 퓨전 요리(복합 스킬) 주문에서는 필요한 레시피북을 빠뜨린다.

---

## 4. 결과 데이터

| 훅 방식 | 전체 성공률 | 복합 프롬프트 | 특징 |
| --- | --- | --- | --- |
| 훅 없음 | 매우 낮음 | - | 기준점 |
| 단순 지시문 | ~20% | 0% | 동전 던지기 |
| **강제 평가** | **84%** | 처리 가능 | 가장 안정적 |
| LLM 평가 | 80% | 0% (폼/라우트) | 빠르지만 변동성 큼 |

### 비용/속도 비교 (Haiku 4.5 기준)

| 항목 | 강제 평가 | LLM 평가 |
| --- | --- | --- |
| 프롬프트당 비용 | $0.0673 | $0.0606 (10% 저렴) |
| 평균 지연 시간 | 5.4초 | 5.0초 (17% 빠름) |

---

## 5. 강제 평가 훅이 통하는 이유

단순 지시문은 "권유"에 불과하다. Claude는 이걸 보고도 무시하고 바로 구현으로 넘어간다.

강제 평가 훅은 **약속 이행 메커니즘**을 만든다.

1. **평가를 보여줘라** → 각 스킬의 적합성을 YES/NO로 명시
2. **약속을 이행해라** → YES로 답한 스킬은 반드시 활성화
3. **그 다음에야 구현해라** → 활성화 없이 구현으로 넘어갈 수 없음

**비유:** PR 머지 전에 코드 리뷰 승인이 필수인 것과 같다. "코드 리뷰 하면 좋겠다"라고 권유하면 안 하지만, 리뷰 승인 없이는 머지 자체가 안 되게 설정하면 할 수밖에 없다.

→ 프롬프트 안에 "MANDATORY(필수)", "CRITICAL(주의)", "WORTHLESS(무가치)" 같은 강한 단어를 넣어서 Claude가 무시하기 어렵게 만드는 것도 한몫한다.

---

## 6. 적용 방법

### 1단계: 훅 파일 생성

`.claude/hooks/skill-forced-eval-hook.sh` 파일을 프로젝트에 추가한다. (전역 적용: `~/.claude/hooks/`)

```bash
#!/bin/bash
cat <<'EOF'
INSTRUCTION: MANDATORY SKILL ACTIVATION SEQUENCE

Step 1 - EVALUATE:
For each skill in <available_skills>, state: [skill-name] - YES/NO - [reason]

Step 2 - ACTIVATE:
IF any skills are YES → Use Skill(skill-name) for EACH relevant skill NOW
IF no skills are YES → State "No skills needed" and proceed

Step 3 - IMPLEMENT:
Only after Step 2 is complete, proceed with implementation.

CRITICAL: You MUST call Skill() tool in Step 2. Do NOT skip to implementation.
The evaluation (Step 1) is WORTHLESS unless you ACTIVATE (Step 2) the skills.
EOF
```

### 2단계: 실행 권한 부여

```bash
chmod +x .claude/hooks/skill-forced-eval-hook.sh
```

### 3단계: settings.json에 등록

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/skill-forced-eval-hook.sh"
          }
        ]
      }
    ]
  }
}
```

이제 프롬프트를 던지면 Claude가 구현 전에 스킬 평가 과정을 거치는 걸 확인할 수 있다.

---

## 7. 어떤 훅을 써야 할까

| 상황 | 추천 |
| --- | --- |
| 일관된 성공률이 중요할 때 | 강제 평가 훅 |
| API 키 없이 순수 로컬에서 해결하고 싶을 때 | 강제 평가 훅 |
| 단순한 단일 스킬 프롬프트 위주일 때 | LLM 평가 훅 |
| 비용/속도 최적화가 우선일 때 | LLM 평가 훅 |
| 가끔 처참한 실패를 감수할 수 있을 때 | LLM 평가 훅 |

저자의 결론: **강제 평가 훅을 쓴다.** 응답이 길어지는 건 감수하되, 84%의 안정적인 활성화율과 외부 의존성이 없다는 점이 결정적이다.

---

## 8. 핵심 정리

- 스킬은 자율 활성화되어야 하지만 현실은 그렇지 않다
- 단순 지시문 = 권유 = 무시당함 (20%)
- **강제 평가 = 절차 강제 = 약속 이행 (84%)**
- 핵심 원리: "자율에 맡기지 말고, 절차를 강제하라"
- 200회 이상 테스트로 검증된 결과

---

## 레퍼런스

- [How to Make Claude Code Skills Activate Reliably](https://scottspence.com/posts/how-to-make-claude-code-skills-activate-reliably) — Scott Spence (원문)
- [번역/정리 — 조영제 미디엄](https://medium.com/@joyoungje)
