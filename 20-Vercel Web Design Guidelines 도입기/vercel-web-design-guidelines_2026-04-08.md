# Vercel Web Design Guidelines 도입기 — 기존 스킬 생태계와의 비교 분석

> 최종 수정: 2026-04-08

## TL;DR

Vercel의 `web-design-guidelines` 스킬은 UI 코드를 100+ 규칙으로 심사하는 "완성도 검증" 도구다. 기존 스킬들이 성능/보안/버그를 각각 잡는다면, 이 스킬은 "프로 수준으로 다듬어져 있는가"를 체크한다. nxtcloud-admin 프로젝트에 테스트한 결과, 4개 파일에서 29개 위반 사항을 발견했으며 접근성(role/aria 누락)이 가장 많이 걸렸다. Phase 7(검증 루프)에 배치하여 iteration마다 위반 목록을 다음 수정 사이클로 넘기는 구조로 운영한다.

---

## Part 1: 스킬 개요

### 무엇을 검증하나

"UI 코드가 웹 표준 모범 사례를 따르고 있는가"

| 검증 축 | 질문 |
|---------|------|
| 접근성 | 스크린 리더가 읽을 수 있나? 키보드로 조작 가능한가? |
| 폼 UX | autocomplete 있나? paste 막지 않았나? 에러가 인라인인가? |
| 애니메이션 | reduced-motion 존중하나? compositor 속성만 쓰나? |
| 타이포 | `...` 대신 `…` 쓰나? 숫자에 `tabular-nums` 있나? |
| 이미지 | width/height 있나? lazy 로딩 하나? |
| 성능 | 긴 리스트 가상화했나? render 중 layout 읽기 없나? |
| URL 상태 | 필터/탭이 URL에 반영되나? |
| 터치 | tap delay 제거했나? 모달에 overscroll 막았나? |
| 다크모드 | color-scheme 설정했나? |
| i18n | 날짜/숫자에 Intl.* 쓰나? |
| 하이드레이션 | SSR/CSR 불일치 없나? |
| 카피라이팅 | 버튼 라벨이 구체적인가? 에러에 해결책이 있나? |

### 기존 스킬과의 포지셔닝

| 스킬 | 질문 | 관점 |
|------|------|------|
| `frontend-review-performance` | "느린가?" | 성능 |
| `frontend-review-security` | "뚫리나?" | 보안 |
| `frontend-review-bugs` | "터지나?" | 안정성 |
| `frontend-review-maintainability` | "고치기 힘든가?" | 유지보수 |
| **`web-design-guidelines`** | **"잘 만들었나?"** | **품질/완성도** |

기존 스킬들이 "구현 가이드"(이렇게 만들어라)라면, Vercel은 "심사 가이드"(이걸 위반했는지 검사해라). **보완 관계**이며 충돌 없이 같이 쓸 수 있다.

---

## Part 2: 14개 카테고리 상세 규칙

### Accessibility
- Icon-only 버튼에 `aria-label` 필수
- Form controls에 `<label>` 또는 `aria-label`
- Interactive 요소에 keyboard handler (`onKeyDown`/`onKeyUp`)
- `<button>` for actions, `<a>`/`<Link>` for navigation (not `<div onClick>`)
- Images에 `alt` (decorative이면 `alt=""`)
- Decorative icons에 `aria-hidden="true"`
- Async updates에 `aria-live="polite"`
- Semantic HTML 우선, ARIA는 보조
- Headings 계층적 `<h1>`–`<h6>`, skip link 포함
- `scroll-margin-top` on heading anchors

### Focus States
- Interactive 요소에 visible focus: `focus-visible:ring-*`
- `outline-none` 단독 사용 금지 — focus replacement 필수
- `:focus-visible` > `:focus` (클릭 시 focus ring 방지)
- `:focus-within`으로 compound controls 그룹 포커스

### Forms
- `autocomplete`와 meaningful `name` 필수
- 올바른 `type` (`email`, `tel`, `url`, `number`) + `inputmode`
- paste 차단 금지 (`onPaste` + `preventDefault`)
- Labels 클릭 가능 (`htmlFor` or wrapping)
- `spellCheck={false}` on emails/codes/usernames
- Checkboxes/radios: label + control 단일 hit target
- Submit 버튼은 요청 시작까지 enabled, 이후 spinner
- Errors inline next to fields, focus first error
- Placeholders end with `…`
- `beforeunload` or router guard for unsaved changes

### Animation
- `prefers-reduced-motion` 존중
- `transform`/`opacity` only (compositor-friendly)
- `transition: all` 금지 — 속성 명시
- 올바른 `transform-origin` 설정
- Animations interruptible

### Typography
- `…` not `...`
- Curly quotes `"` `"` not straight `"`
- Non-breaking spaces: `10&nbsp;MB`, `⌘&nbsp;K`
- Loading states end with `…`
- `font-variant-numeric: tabular-nums` for number columns
- `text-wrap: balance` or `text-pretty` on headings

### Content Handling
- Text containers: `truncate`, `line-clamp-*`, or `break-words`
- Flex children에 `min-w-0`
- Empty states 처리
- User-generated content: short/average/very long 대응

### Images
- `<img>`에 explicit `width`/`height` (CLS 방지)
- Below-fold: `loading="lazy"`
- Above-fold critical: `priority` or `fetchpriority="high"`

### Performance
- 50+ items: virtualize
- Render 중 layout reads 금지
- DOM reads/writes batch
- Uncontrolled inputs 선호
- `<link rel="preconnect">` for CDN
- Critical fonts: `<link rel="preload" as="font">`

### Navigation & State
- URL에 상태 반영 (filters, tabs, pagination)
- Links는 `<a>`/`<Link>` (Cmd/Ctrl+click 지원)
- 파괴적 행동은 confirmation modal or undo

### Touch & Interaction
- `touch-action: manipulation`
- `-webkit-tap-highlight-color` 설정
- `overscroll-behavior: contain` in modals/drawers
- Drag 중 text selection 비활성화
- `autoFocus` 신중하게

### Safe Areas & Layout
- Full-bleed layouts에 `env(safe-area-inset-*)`
- `overflow-x-hidden`으로 unwanted scrollbars 방지
- Flex/grid > JS measurement

### Dark Mode & Theming
- `color-scheme: dark` on `<html>`
- `<meta name="theme-color">` matches background
- Native `<select>`: explicit `background-color`/`color`

### Locale & i18n
- `Intl.DateTimeFormat` / `Intl.NumberFormat` 사용
- `Accept-Language` / `navigator.languages`로 언어 감지 (IP 금지)
- Brand names: `translate="no"`

### Hydration Safety
- `value` + `onChange` 쌍 (or `defaultValue`)
- Date/time hydration mismatch 방지
- `suppressHydrationWarning` 필요한 곳만

### Anti-patterns (즉시 플래그)
- `user-scalable=no` / `maximum-scale=1`
- `onPaste` + `preventDefault`
- `transition: all`
- `outline-none` without focus-visible replacement
- `<div onClick>` navigation
- Images without dimensions
- Large arrays `.map()` without virtualization
- Form inputs without labels
- Hardcoded date/number formats

### 출력 형식

```text
## src/Button.tsx
src/Button.tsx:42 - icon button missing aria-label
src/Button.tsx:55 - animation missing prefers-reduced-motion

## src/Card.tsx
✓ pass
```

---

## Part 3: 기존 스킬과의 커버리지 비교

### 커버리지 매트릭스

| Vercel 카테고리 | 기존 스킬에서 커버? | 커버하는 스킬 | 차이점 |
|---|---|---|---|
| Accessibility | 대부분 | `frontend-accessibility` | Vercel이 더 넓음: `aria-live`, skip link, `scroll-margin-top` |
| Focus States | 부분 | `frontend-accessibility` | Vercel이 `:focus-visible` vs `:focus`, `:focus-within` 명시 |
| Forms | 부분 | `frontend-accessibility`, `frontend-form` | Vercel에만: `autocomplete`, paste 차단 금지, `spellCheck={false}`, `beforeunload` |
| Animation | 부분 | `frontend-design` | Vercel이 더 구체적: `transition: all` 금지, compositor 속성만, `transform-origin` |
| Typography | 다른 관점 | `frontend-design`, `frontend-style` | 기존은 폰트/크기, Vercel은 문자 교정 (`…`, 곱슬 따옴표, `tabular-nums`) |
| Content Handling | **없음** | — | `min-w-0`, truncate, empty state 등 완전히 새로운 영역 |
| Images | **없음** | — | CLS 방지 `width`/`height`, lazy/priority 로딩 |
| Performance | 대부분 | `frontend-review-performance` | 기존이 더 상세 (N+1, 메모리 누수, 번들 크기 등) |
| Navigation & State | **없음** | — | URL 상태 반영, 파괴적 행동 확인/undo |
| Touch & Interaction | **없음** | — | `touch-action`, `overscroll-behavior`, 드래그 UX |
| Dark Mode | **없음** | — | `color-scheme: dark`, `theme-color` meta |
| Locale & i18n | **없음** | — | `Intl.*` 사용, `translate="no"` |
| Hydration Safety | **없음** | — | `value`+`onChange` 쌍, 날짜 mismatch |
| Content & Copy | **없음** | — | 능동태, Title Case, 버튼 라벨 구체화 |
| Anti-patterns | 일부 | `frontend-review-security` | Vercel은 UI 안티패턴, 기존은 보안 안티패턴 — 겹침 거의 없음 |

### 수치 요약

- **겹침: ~30%** — 접근성, 애니메이션, 성능 일부
- **새로운 영역: ~50%** — Content Handling, Touch, i18n, Dark Mode, Hydration, Navigation State, Copy
- **기존이 더 나은 영역: ~20%** — 성능 상세(N+1, 번들, 메모리 누수), 보안

---

## Part 4: 실전 테스트 — nxtcloud-admin

### 테스트 대상

nxtcloud-admin 프로젝트의 주요 UI 컴포넌트 4개:
- `LoginForm.tsx` — 로그인 폼
- `AdminSidebar.tsx` — 사이드바 네비게이션
- `ContentTable.tsx` — 콘텐츠 목록 테이블 (검색, 필터, 페이지네이션)
- `InsightForm.tsx` — 인사이트 에디터 (다국어, 미리보기, 메타 사이드바)

### 심사 결과

#### LoginForm.tsx

```text
src/components/auth/LoginForm.tsx:50 - email input missing autocomplete="email"
src/components/auth/LoginForm.tsx:65 - password input missing autocomplete="current-password"
src/components/auth/LoginForm.tsx:53 - placeholder "admin@nxtcloud.kr" should end with …
src/components/auth/LoginForm.tsx:83 - "로그인 중..." → "로그인 중…" (ellipsis)
src/components/auth/LoginForm.tsx:39 - space-y-6 → flex flex-col gap-6
src/components/auth/LoginForm.tsx:76 - error message missing next step
```

#### AdminSidebar.tsx

```text
src/components/layout/AdminSidebar.tsx:57 - space-y-1 → flex flex-col gap-1
src/components/layout/AdminSidebar.tsx:74 - decorative icon missing aria-hidden="true"
src/components/layout/AdminSidebar.tsx:156 - Sheet(drawer) missing overscroll-behavior: contain
src/components/layout/AdminSidebar.tsx:68 - transition-colors → list properties explicitly
```

#### ContentTable.tsx

```text
src/components/ContentTable.tsx:139 - search input missing aria-label or <label>
src/components/ContentTable.tsx:148 - clear button missing aria-label
src/components/ContentTable.tsx:158 - filter tabs missing role="tablist" / role="tab" / aria-selected
src/components/ContentTable.tsx:322 - pagination prev button missing aria-label
src/components/ContentTable.tsx:340 - pagination next button missing aria-label
src/components/ContentTable.tsx:326 - page buttons missing aria-current="page" or aria-label
src/components/ContentTable.tsx:143 - placeholder "검색..." → "검색…"
src/components/ContentTable.tsx:370 - "삭제 중..." → "삭제 중…"
src/components/ContentTable.tsx:257 - date format: consider Intl.DateTimeFormat
```

#### InsightForm.tsx

```text
src/components/admin/InsightForm.tsx:208 - label for "본문" missing htmlFor
src/components/admin/InsightForm.tsx:281 - "저장 중..." → "저장 중…"
src/components/admin/InsightForm.tsx:234 - Image missing priority/loading attribute
src/components/admin/InsightForm.tsx:389 - second Image also missing priority/loading
src/components/admin/InsightForm.tsx:382 - file input missing aria-label
src/components/admin/InsightForm.tsx:385 - "업로드 중..." → "업로드 중…"
src/components/admin/InsightForm.tsx:141-149 - language tabs missing role="tablist"/role="tab"/aria-selected
src/components/admin/InsightForm.tsx:300-310 - category buttons missing role="radiogroup"/role="radio"
src/components/admin/InsightForm.tsx:364-370 - thumbnail mode tabs missing role="tablist"
src/components/admin/InsightForm.tsx:257 - hardcoded toLocaleDateString → Intl.DateTimeFormat
```

### 결과 요약

| 파일 | 위반 수 | 주요 이슈 |
|------|---------|----------|
| LoginForm | 6 | autocomplete 누락, ellipsis |
| AdminSidebar | 4 | overscroll-behavior, aria-hidden |
| ContentTable | 9 | 접근성 (label, tablist, aria-label) |
| InsightForm | 10 | 접근성 (tablist, radiogroup), ellipsis, Image 로딩 |
| **합계** | **29** | |

### 위반 패턴 분포

| 패턴 | 건수 | 비율 |
|------|------|------|
| 접근성 (role/aria 누락) | 14 | 48% |
| ellipsis (`...` → `…`) | 6 | 21% |
| autocomplete/name 누락 | 3 | 10% |
| Image 로딩 속성 누락 | 2 | 7% |
| overscroll/transition | 2 | 7% |
| 기타 (에러 메시지, space-y) | 2 | 7% |

기존 `frontend-accessibility` 스킬이 커버하는 영역(접근성)이 가장 많이 걸렸다는 점이 흥미롭다. 이는 기존 스킬이 "구현 시점"에만 적용되고, "심사 시점"에는 적용되지 않기 때문이다. web-design-guidelines가 Phase 7에서 사후 검증함으로써 이 갭을 메운다.

---

## Part 5: 프로세스 배치

### Phase 7 검증 루프 내 위치

```
Phase 7: 검증 루프
  1. iteration-evaluator (Playwright 동적 테스트 + 4축 점수)
  2. reviewer (코드 품질)
  3. test (테스트 통과)
  4. web-design-guidelines (UI 완성도)  ← 신규
  5. handoff.md 업데이트
```

### 결과 흐름

web-design-guidelines의 위반 목록은 reviewer 코멘트와 같은 방식으로 다음 iteration에서 수정된다.

| 결과물 | 흘러가는 곳 | 역할 |
|--------|------------|------|
| iteration-evaluator 점수 | score-history.jsonl, /pivot-check | 정량 추적 |
| reviewer 코멘트 | 다음 iteration에서 수정 | 정성 피드백 |
| test 결과 | pass 안 되면 루프 계속 | 게이트 |
| **web-design-guidelines** | **다음 iteration에서 수정** | **UI 완성도 피드백** |

### 선택하지 않은 대안

| 대안 | 불채택 이유 |
|------|------------|
| iteration-evaluator에 통합 (점수화) | 100+ 규칙을 점수로 환산하는 기준을 새로 만들어야 함. 과도한 엔지니어링 |
| 게이트로 사용 (위반 0개 필수) | i18n이나 카피라이팅 규칙은 프로젝트에 따라 해당 안 되는 것도 많음. 루프가 안 끝남 |
| Phase 6 셀프 체크 | 코드가 계속 바뀌는 중이라 헛검사가 많고, 개발 흐름 끊김 |

---

## Part 6: Vercel 나머지 스킬 평가

설치 시 함께 나왔던 6개 스킬 중 `web-design-guidelines` 외 나머지의 평가.

| 스킬 | 설명 | 기존 겹침 | 도입 가치 |
|------|------|----------|----------|
| `react-best-practices` | 69개 성능 규칙 | `frontend-review-performance`와 많이 겹침 | 서버 성능, 리렌더 15규칙 등 새로운 30%는 기존 스킬에 녹여넣기 추천 |
| `composition-patterns` | 8개 컴포넌트 구조 규칙 | `frontend-components` + `frontend-fundamentals`와 부분 겹침 | generic context interface 등 기존 스킬에 녹여넣기 추천 |
| `react-view-transitions` | View Transition API | 기존에 없음 | 니치한 영역. 필요할 때 참고 수준 |
| `deploy-to-vercel` | Vercel 배포 | 해당 없음 | 불필요 |
| `vercel-cli-with-tokens` | CLI 토큰 인증 | 해당 없음 | 불필요 |
| `react-native-guidelines` | React Native | 모바일 안 함 | 불필요 |

### 기존 스킬 보강 후보 (별도 설치 대신 녹여넣기)

| 기존 스킬 | 추가할 내용 | 출처 |
|-----------|------------|------|
| `frontend-review-performance` | 서버 성능 (auth-actions, parallel-fetching, LRU 캐시, after()), inline component 금지, Suspense 스트리밍, resource hints | react-best-practices |
| `frontend-components` | generic context interface (state/actions/meta), 상태 분리 원칙 심화, React 19 API 변경 | composition-patterns |
| `frontend-review-security` | Server Action 인증 필수 (CRITICAL) | react-best-practices |

---

## 핵심 인사이트

1. **기존 스킬은 "만드는 시점"에 강하고, Vercel은 "검증하는 시점"에 강하다.** 같은 접근성 규칙이라도 구현 가이드와 사후 심사는 다른 역할이다. Phase 7에 심사 도구를 추가함으로써 "만들 때 놓친 것"을 잡는 안전망이 생겼다.

2. **Vercel 스킬을 통째로 설치하기보다, 기존 스킬에 없는 규칙만 선별해서 녹여넣는 것이 효과적이다.** 중복 70%를 가져가면 심사 시 노이즈만 늘어난다. `web-design-guidelines`만 독립 설치한 이유도 이것 — 기존 스킬과 완전히 다른 관점이기 때문이다.

3. **실전 테스트에서 가장 많이 걸린 건 접근성(48%)이다.** `frontend-accessibility` 스킬이 있음에도 걸렸다는 건, 구현 시점의 가이드만으로는 누락이 발생한다는 뜻이다. 심사 시점의 체크리스트가 이 갭을 메운다.
