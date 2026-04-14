# Pretext — 실전 가이드

> 최종 수정: 2026-04-06

## TL;DR

Tailwind로 스타일 다 잡아도, JS 코드가 텍스트 높이를 알아야 하는 순간이 있다. 기존엔 브라우저한테 물어봤는데 그때마다 페이지 전체가 다시 계산됐다. Pretext는 그 질문 자체를 없앤다. 텍스트를 미리 분석해두고 이후론 덧셈만 해서 계산한다.

---

## Part 1: Pretext가 뭔가

CSS 없이 텍스트 레이아웃을 계산하는 라이브러리다. 브라우저한테 "이 텍스트 높이 얼마야?" 묻는 대신, 스스로 계산한다. 기존 방식보다 약 500배 빠르고, CSS로 불가능했던 레이아웃을 가능하게 한다.

```sh
npm install @chenglou/pretext
```

---

## Part 2: 기존 방식의 문제

Tailwind로 스타일을 다 잡고 나서, JS 코드에서 텍스트 높이를 알아야 하는 순간이 있다.

```jsx
// 텍스트 높이를 JS에서 알아야 할 때 쓰던 방식
const ref = useRef()
useEffect(() => {
  const height = ref.current.getBoundingClientRect().height
}, [])
```

이 `getBoundingClientRect()` 를 호출하는 순간 브라우저가 **페이지 전체 레이아웃을 다시 계산**한다. 이걸 DOM reflow라고 한다.

```
"이 텍스트 높이 얼마야?" 
→ 브라우저: 잠깐, 전체 페이지 다시 계산
→ 그 다음 높이 알려줌
```

아이템 하나면 괜찮다. 근데 수천 개면 수천 번 전체 재계산이 일어난다.

---

## Part 3: Pretext가 해결하는 방법

```ts
import { prepare, layout } from '@chenglou/pretext'

// 1. 텍스트 분석 (한 번만)
const prepared = prepare("안녕하세요", "16px Inter")

// 2. 높이 계산 (브라우저한테 안 물어봄, 그냥 덧셈)
const { height } = layout(prepared, 300, 24) // maxWidth=300px, lineHeight=24px
```

내부 동작:
1. 텍스트를 단어/글자 단위로 쪼갬
2. Canvas로 각 조각 너비를 측정해서 저장 (Canvas는 reflow 없음)
3. 이후 "300px 너비에 몇 줄이야?" → 저장된 숫자 더하기만

React에서 쓸 때 핵심 패턴:

```tsx
// 텍스트 바뀔 때만 prepare 다시 실행
const prepared = useMemo(() => prepare(text, "16px Inter"), [text])

// 컨테이너 크기 바뀔 때는 layout만 (빠름)
const { height } = layout(prepared, containerWidth, 24)
```

---

## Part 4: 어떤 상황에서 쓰나

### 가상화 리스트

댓글, 채팅, 피드처럼 아이템이 수천~수만 개면 전부 렌더링할 수 없다. 브라우저가 다 만들어놓으면 메모리가 터진다.

가상화는 화면에 보이는 것만 렌더링하는 방식이다.

```
스크롤 위치 기준으로 보이는 20개만 렌더링
나머지 9만 9980개는 빈 공간으로 처리
```

근데 빈 공간 높이를 맞추려면 "각 아이템이 몇 px인지" 미리 알아야 한다. 텍스트 길이가 메시지마다 달라서 높이가 제각각이면 더 문제다.

기존:
```jsx
// 아이템마다 getBoundingClientRect() 호출 → 수천 번 reflow
useEffect(() => {
  const height = ref.current.getBoundingClientRect().height
  setHeight(height)
}, [])
```

Pretext:
```jsx
// 렌더링 전에 높이를 미리 앎 → reflow 없음
const prepared = useMemo(() => prepare(text, "16px Inter"), [text])
const { height } = layout(prepared, containerWidth, 24)
```

---

### 채팅 말풍선 shrink-wrap

Tailwind `w-fit` 또는 CSS `fit-content`는 마지막 줄이 짧으면 말풍선이 어색하게 좁아진다.
![[스크린샷 2026-04-06 12.51.48.png]]

---

### Canvas / SVG 텍스트 배치

Canvas나 SVG에 텍스트를 그릴 때 멀티라인 처리가 문제다. DOM이 없어서 CSS도 안 되고, 줄바꿈을 직접 계산해야 한다.

```tsx
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments("긴 텍스트...", "18px Inter")
const { lines } = layoutWithLines(prepared, 320, 26)

// Canvas에 줄 단위로 그리기
for (let i = 0; i < lines.length; i++) {
  ctx.fillText(lines[i].text, 0, i * 26)
}
```

---

### 아코디언 / 높이 애니메이션

아코디언 열고 닫을 때 높이 애니메이션을 주려면 목표 높이를 미리 알아야 한다. 기존엔 렌더링 후 DOM에서 재고 애니메이션을 주는 방식이라 깜빡임이 생겼다.

Pretext는 렌더링 전에 높이를 알기 때문에 처음부터 올바른 높이로 애니메이션 가능.

---

## Part 5: 언제 쓰고 언제 안 써도 되나

### 쓸 때
- 가변 높이 아이템 수천 개 이상 가상화
- 채팅 말풍선 shrink-wrap
- Canvas/SVG에 멀티라인 텍스트
- 높이 애니메이션에서 깜빡임 없애고 싶을 때

### 안 써도 될 때
- 일반 웹페이지, 대시보드
- 텍스트 아이템 수십 개 수준
- CSS로 이미 잘 되는 레이아웃

### 주의사항
- `system-ui` 폰트는 macOS에서 정확도 떨어짐 → `Inter`, `Helvetica Neue` 등 명시적 폰트명 사용
- `prepare()`를 같은 텍스트에 반복 호출하면 오히려 느림 → `useMemo`로 캐시 필수

---

## 핵심 인사이트

1. **Tailwind랑 겹치지 않는다** — Tailwind는 스타일링, Pretext는 JS에서 텍스트 높이를 알아야 할 때. 같이 쓴다.

2. **reflow가 문제인 상황에서만 필요하다** — 아이템 수십 개면 체감 못 한다. 수천 개가 넘어갈 때부터 의미 있다.

3. **CSS로 불가능했던 것들이 열린다** — 채팅 말풍선 shrink-wrap, Canvas 멀티라인, 렌더 전 높이 계산. 이것들이 Pretext의 진짜 가치다.
