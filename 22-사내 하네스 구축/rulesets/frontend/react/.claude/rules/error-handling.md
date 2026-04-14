---
globs: ["src/**/*.{ts,tsx}"]
---

# React 에러 처리

## 3계층 구조

1. **API 계층**: Axios 인터셉터 (401 자동 갱신)
2. **폼 계층**: React Hook Form + Zod (검증 에러)
3. **렌더링 계층**: ErrorBoundary (런타임 에러)

## ErrorBoundary

- react-error-boundary 사용 또는 직접 구현
- 페이지/기능 단위로 배치
- fallback에 재시도 버튼 제공

## 에러 메시지

- 사용자 친화적 메시지만 표시
- 기술 정보(스택 트레이스, 서버 구조) 숨김
- 에러 로깅: onError 콜백에서 외부 서비스로 전송
