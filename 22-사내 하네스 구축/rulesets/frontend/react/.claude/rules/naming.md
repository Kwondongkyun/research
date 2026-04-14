---
globs: ["src/**/*.{ts,tsx}"]
---

# 네이밍 규칙

## 파일/폴더

- 파일/폴더: kebab-case
- 컴포넌트: PascalCase 디렉토리 + index.tsx
- API/Hooks/유틸: kebab-case (api.ts, use-mobile.ts)
- Context: PascalCase (AuthContext.tsx)
- 테스트: `*.test.ts` (`.spec.ts` 금지)

## 코드

- 변수/함수: camelCase
- 상수: UPPER_SNAKE_CASE
- 타입/인터페이스: PascalCase
- boolean: `is`/`has`/`should`/`can` 접두사 필수
- API 함수: [동사][명사]Api (getMeApi, createUserApi)

## Import

- 디렉토리명으로 import (index 명시 금지)

## 금지

- 범용 네이밍: `data`, `info`, `temp`, `flag`, `result`
- 약어 남용: `btn`, `usr`, `msg` (풀네임 사용)
