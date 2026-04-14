---
globs: ["src/**/*.tsx"]
---

# 접근성 규칙

## 필수

- html lang 속성을 콘텐츠 언어와 일치
- 모든 인터랙티브 요소는 button 또는 적절한 role 필요
- 탭/필터 UI: role="tablist" + role="tab" + aria-selected
- 탭 화살표 키: ArrowLeft/Right로 포커스 이동 + tabIndex 관리

## 키보드

- 키보드 접근 불가능한 div 클릭: role + tabIndex + onKeyDown (Enter/Space) 추가
- 모달: 포커스 트랩 + Escape로 닫기

## 금지

- label + aria-label 중복 (하나만 사용)
- 장식용 이미지에 의미 있는 alt (빈 alt="" 사용)
