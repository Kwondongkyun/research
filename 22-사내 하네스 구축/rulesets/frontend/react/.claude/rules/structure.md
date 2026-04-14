---
globs: ["src/**/*"]
---

# React 프로젝트 구조

```
src/
  components/       ← 공통 UI 컴포넌트
    Button/
      index.tsx
  features/         ← 도메인별 모듈
    auth/
      api.ts        ← API 함수
      types.ts      ← 타입 정의
      schemas.ts    ← Zod 스키마
      hooks.ts      ← 커스텀 훅
      keys.ts       ← TanStack Query 키
      AuthContext.tsx
  hooks/            ← 전역 공통 훅만
  lib/              ← 유틸 (axios.ts, utils.ts)
  pages/            ← 페이지 컴포넌트 (라우터 엔트리)
  App.tsx
  main.tsx
```

## 규칙

- features에 컴포넌트 금지, 도메인 로직만
- 컴포넌트는 components/ 또는 페이지 근처에 배치
- 함께 수정되는 파일은 같은 디렉토리에

## 라우팅

- React Router: 중앙 라우트 정의 (App.tsx 또는 routes.tsx)
- 라우트 보호: PrivateRoute 컴포넌트로 인증 래핑
- 404: catch-all 라우트(`*`)에 NotFound 컴포넌트

## 상태 관리

- 로컬 상태: useState/useReducer
- 서버 상태: TanStack Query
- 전역 상태: Context API (소규모) 또는 Zustand (대규모)
- Redux 금지 (레거시 제외)

## 환경 변수

- 접두사: 빌드 도구에 따라 다름 (CRA: REACT_APP_)
- 공개되어도 괜찮은 것만 클라이언트에 노출
