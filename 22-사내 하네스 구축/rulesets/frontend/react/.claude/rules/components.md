---
globs: ["src/**/*.tsx", "src/components/**/*", "src/features/**/components/**/*"]
---

# 컴포넌트 규칙

## 구조

- 디렉토리/index.tsx 패턴 필수 (단일 파일 컴포넌트 금지)
- Named export 필수, default export 금지
- 컴포넌트 200줄 초과 시 분리
- 함수 50줄 초과 시 분리
- useState 5개 초과 시 커스텀 훅 추출

## 선언

- "use client"는 훅/이벤트/브라우저 API 사용 시만
- 상태 업데이트 시 함수형 업데이터 필수 (`prev =>` 패턴)
- localStorage 접근은 useEffect 안에서만
- useState 초기값이 비용 큰 계산이면 lazy initializer 사용

## 패턴

- Compound Components: 서브컴포넌트를 네임스페이스로 묶기
- Props Drilling 3단계 이상 시 children 패턴 또는 Context API
- useTransition으로 비긴급 업데이트 분리

## Import 순서

1. React
2. 외부 라이브러리
3. UI 컴포넌트
4. 내부 컴포넌트
5. Hooks/Utils
6. 타입/상수

## 금지

- 조건부 렌더링에 삼항연산자 중첩
- HTML `<a>`, `<img>` 직접 사용 (Link, Image 컴포넌트 사용)
- HTML UI 요소 직접 사용 (shadcn/ui 우선)
