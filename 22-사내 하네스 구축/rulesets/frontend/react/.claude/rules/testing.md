---
globs: ["src/**/*.test.*", "src/**/__tests__/**/*", "e2e/**/*", "tests/**/*"]
---

# 테스트 규칙

## 파일

- 테스트 파일: 대상과 같은 디렉토리
- 파일명: `.test.ts` (`.spec.ts` 금지)

## 단위 테스트 (Vitest)

- AAA 패턴: Arrange → Act → Assert
- beforeEach에서 vi.clearAllMocks()
- 외부 의존성만 모킹, 테스트 대상은 실제 코드
- renderHook으로 커스텀 훅 테스트
- 엣지 케이스: 빈 값, 경계값, 에러

## 컴포넌트 테스트 (Vitest + RTL)

- 셀렉터 우선순위: getByRole > getByLabelText > getByText > getByTestId
- CSS 클래스/태그로 찾기 금지
- userEvent 사용 (fireEvent 최소화)
- 요소 부재 확인: queryBy 사용 (getBy 금지)
- 비동기: waitFor 또는 findBy 사용
- Provider 필요 시 renderWithProviders

## API 모킹 (MSW)

- MSW v2로 API 가로채기
- 에러 시나리오 테스트 필수

## E2E (Playwright)

- Page Object Model 필수 (셀렉터 하드코딩 금지)
- waitForTimeout 금지, waitForResponse/waitForURL/toBeVisible 사용
- 인증 상태: global-setup.ts + storageState
- 테스트 독립성: 다른 테스트 결과에 의존 금지

## TDD

- Red → Green → Refactor, 한 번에 하나의 테스트만
- GREEN: 과도한 구현 금지 (최소한의 코드만)
- 커버리지 목표: 80% 이상
