---
globs: ["src/**/api.*", "src/**/hooks.*", "src/**/keys.*", "src/lib/api/**/*"]
---

# API 규칙

## Axios

- 인스턴스: lib/api/axios.ts에서 관리
- 요청 인터셉터: SSR 가드 (typeof window 확인) + 토큰 추가
- 응답 인터셉터: 401 시 refresh token 자동 갱신 (무한 루프 방지)
- API 함수 패턴: [동사][명사]Api (getMeApi, createUserApi)
- 에러 처리 3단계: Interceptor(401) → API 함수(404) → 컴포넌트(UI)

## TanStack Query

- useQuery: 데이터 조회 (캐싱, 백그라운드 갱신)
- useMutation: 데이터 변경 (생성/수정/삭제)
- QueryClient: useState로 생성 (컴포넌트 밖 금지)
- staleTime 필수 설정 (기본값 0은 위험)
- queryKey: keys.ts에서 상수 관리, 계층적 설계 (['users'], ['users', id])
- mutation 성공 후 invalidateQueries로 캐시 무효화
- 낙관적 업데이트: onMutate → onError 롤백 → onSettled 동기화
- enabled 옵션으로 불필요한 쿼리 실행 방지

## 금지

- queryKey 하드코딩
- 루프 안 API 호출 (배치 요청 사용)
