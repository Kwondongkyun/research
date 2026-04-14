# Claude 모델 비교 — Opus, Sonnet, Haiku 각각 언제 쓸까

*최종 수정: 2026-03-19*

## TL;DR

"코드는 Sonnet"이라는 인식이 있지만, 벤치마크에서는 Opus가 모든 항목에서 1등이다. Sonnet이 코딩 모델로 유명한 진짜 이유는 성능이 더 좋아서가 아니라, **속도 × 비용 × 성능의 교차점**에서 가장 효율적이기 때문이다. Claude Code 자체도 이를 알고 Plan은 Opus, 실행은 Sonnet을 쓰는 하이브리드 전략을 채택하고 있다.

---

## Part 1: 최신 모델 라인업 (2026-03 기준)

### 현재 모델 (Current)

| 모델 | API ID | 가격 (Input/Output MTok) | 컨텍스트 | Max Output |
|------|--------|------------------------|----------|------------|
| **Opus 4.6** | `claude-opus-4-6` | $5 / $25 | 1M | 128k |
| **Sonnet 4.6** | `claude-sonnet-4-6` | $3 / $15 | 1M | 64k |
| **Haiku 4.5** | `claude-haiku-4-5` | $1 / $5 | 200k | 64k |

### 레거시 모델 (Legacy)

아직 API에서 사용 가능하지만, 더 이상 권장하지 않는 이전 세대 모델.

| 모델 | API ID | 가격 (Input/Output MTok) | 컨텍스트 |
|------|--------|------------------------|----------|
| Sonnet 4.5 | `claude-sonnet-4-5` | $3 / $15 | 200k (1M 베타) |
| Opus 4.5 | `claude-opus-4-5` | $5 / $25 | 200k |
| Opus 4.1 | `claude-opus-4-1` | $15 / $75 | 200k |
| Sonnet 4 | `claude-sonnet-4-0` | $3 / $15 | 200k (1M 베타) |
| Opus 4 | `claude-opus-4-0` | $15 / $75 | 200k |
| Haiku 3 | `claude-3-haiku` | $0.25 / $1.25 | 200k (deprecated, 2026-04-19 retired) |

참고: Opus 4.1 → 4.6으로 오면서 가격이 $15/$75 → $5/$25로 **3배 저렴**해졌다.

---

## Part 2: 벤치마크 — 사실 Opus가 모든 항목에서 1등

| 벤치마크 | Haiku 4.5 | Sonnet 4.6 | Opus 4.6 | 측정 대상 |
|---------|----------|-----------|---------|----------|
| SWE-bench Verified | 73.3% | ~79.6% | ~80.8% | 실제 GitHub 버그 수정 |
| HumanEval | - | ~92% | ~95% | 코드 생성 |
| MBPP | - | ~88% | ~91% | 코드 생성 |
| GPQA Diamond | - | ~65% | ~74% | 고급 추론 |
| ARC AGI 2 | - | ~58% | ~75% | 범용 지능 |

Opus가 모든 벤치마크에서 최고점이다. 특히 추론(GPQA, ARC AGI)에서 격차가 크다.

---

## Part 3: 그런데 왜 "코드는 Sonnet"인가?

### 이유 1: 코딩 성능 차이가 미미하다

SWE-bench 기준 Opus 80.8% vs Sonnet 79.6% → **1.2%p 차이**. 일반적인 코딩 작업(CRUD, 컴포넌트, 테스트 작성)에서는 체감 차이가 거의 없다.

### 이유 2: 속도가 코딩에서 더 중요하다

코딩은 반복 작업이다 — 쓰고 → 테스트 → 고치고 → 테스트. 이 루프에서 **왕복 속도(latency)**가 생산성에 직접 영향을 미친다. Sonnet이 체감상 훨씬 빠르다.

### 이유 3: 가격 × 지속 가능성

- Sonnet은 Opus 대비 약 40% 저렴 ($3/$15 vs $5/$25)
- 더 중요한 건 **Rate Limit** — Opus는 토큰 소모가 커서 한 세션에서 더 빨리 한도에 도달
- 실무 개발자 코멘트: *"최고 모델이 아니라, 내가 계속 쓸 수 있는 모델이 최고다"*

### 이유 4: 코딩의 90%는 "깊은 사고"가 필요 없다

대부분의 코딩 작업은 잘 정의된 패턴의 반복이다. API 엔드포인트, React 컴포넌트, 유틸 함수 — 이런 작업에 Opus의 "딥 씽킹"은 오버스펙.

---

## Part 4: 각 모델의 진짜 강점

### Haiku 4.5 — 워크호스 (일꾼)

- **강점**: 속도, 가성비 (Opus 대비 5배 저렴)
- **코딩 적합**: 코드 리뷰, 린트, 테스트 생성, 문서화
- **언제 쓸까**: 잘 정의된 단순 작업, 고빈도 반복 작업
- **한계**: 복잡한 멀티스텝 추론에서 약함

### Sonnet 4.6 — 데일리 드라이버

- **강점**: 코딩 + 속도의 최적 밸런스
- **코딩 적합**: CRUD, 컴포넌트 구현, 디버깅, 리팩토링, 멀티파일 변경
- **언제 쓸까**: 일상 개발의 90% → Sonnet이면 충분
- **한계**: 대규모 아키텍처 설계에서 Opus보다 약간 얕은 판단

### Opus 4.6 — 딥 씽커

- **강점**: 복잡한 추론, 아키텍처 설계, Agent Teams
- **코딩 적합**: 아키텍처 설계, 레거시 분석, 대규모 리팩토링, 애매한 요구사항 해석
- **언제 쓸까**: 틀리면 비용이 큰 복잡한 문제, 10% 깊은 사고가 필요한 작업
- **한계**: 느리고 비쌈, Rate Limit 빨리 소진

---

## Part 5: Claude Code의 하이브리드 전략

Claude Code 자체도 단일 모델을 쓰지 않는다.

### opusplan 모드

- **Plan 모드** → Opus (아키텍처, 설계 판단)
- **실행 모드** → Sonnet (코드 생성, 구현)

이게 Claude Code가 내린 결론이다: **설계는 Opus, 구현은 Sonnet**.

### 서브에이전트 모델 선택

Claude Code에서 Agent 도구로 서브에이전트를 띄울 때 `model` 파라미터로 선택 가능:
- `opus` — 복잡한 리서치, 아키텍처 결정
- `sonnet` — 코드 구현, 일반 작업
- `haiku` — 빠르고 단순한 작업 (비용 최소화)

---

## Part 6: 실전 가이드 — 5초 룰

> 어떤 모델 쓸지 5초 이상 고민되면:
> - **복잡한 설계/리서치** → Opus
> - **그 외 전부** → Sonnet

| 작업 | 추천 모델 | 이유 |
|------|----------|------|
| API 엔드포인트 구현 | Sonnet | 패턴화된 작업, 속도 중요 |
| React 컴포넌트 개발 | Sonnet | 반복적, 빠른 피드백 루프 |
| 테스트 코드 작성 | Haiku/Sonnet | 잘 정의된 패턴 |
| 디버깅 | Sonnet | 속도 + 충분한 추론력 |
| 아키텍처 설계 | Opus | 깊은 트레이드오프 분석 필요 |
| 레거시 코드 분석 | Opus | 컨텍스트 이해력 중요 |
| 대규모 리팩토링 계획 | Opus | 전체 구조 파악 필요 |
| 코드 리뷰/린트 | Haiku | 단순 반복, 비용 최소화 |
| 문서화 | Haiku/Sonnet | 패턴화, 속도 중요 |

---

## 핵심 인사이트

1. **벤치마크 1등은 Opus** — 모든 항목에서 최고 성능
2. **"코드는 Sonnet"의 진짜 의미** — 성능이 아니라 속도 × 비용 × 성능의 최적점
3. **코딩의 90%는 깊은 사고가 필요 없다** — 패턴 반복에 Opus는 오버스펙
4. **Rate Limit이 실무에서 가장 중요** — 계속 쓸 수 있는 모델이 진짜 최고
5. **하이브리드가 정답** — Claude Code도 Plan=Opus, 실행=Sonnet 전략 사용
6. **5초 룰** — 고민되면 Sonnet, 복잡한 설계만 Opus

---

## Sources

- [Claude Sonnet 4.6 vs Opus 4.6 - Which One is Better for Coding?](https://blog.getbind.co/claude-sonnet-4-6-vs-opus-4-6-which-one-is-better-for-coding/)
- [Which Claude Model Is Best for Coding: Opus vs Sonnet vs Haiku](https://www.dataannotation.tech/developers/which-claude-model-is-best-for-coding)
- [Models overview - Claude API Docs](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Sonnet 4.6 vs Opus 4.6: Quick Decision Guide](https://www.nxcode.io/resources/news/claude-sonnet-4-6-vs-opus-4-6-which-model-to-choose-2026)
- [Claude Sonnet vs Opus (2026): Which Is Actually Worth It?](https://emergent.sh/learn/claude-sonnet-vs-opus)
