---
globs: ["src/**/*.tsx", "src/**/*.css"]
---

# 스타일링 규칙

## TailwindCSS

- cn() 유틸로 조건부 클래스 관리 (필수)
- 축약 클래스 사용 (flex-shrink-0 → shrink-0)
- 수직 간격: space-y 금지, flex flex-col + gap 사용

## 텍스트/타이포

- 시맨틱 HTML 사용 (h1~h6)
- 페이지당 h1은 1개만, 계층 순서 유지
- TailwindCSS로 스타일링 (별도 CSS 클래스 최소화)

## 금지

- 인라인 스타일
- bg 그라데이션
- 이모지
- CSS Modules (TailwindCSS 우선)
