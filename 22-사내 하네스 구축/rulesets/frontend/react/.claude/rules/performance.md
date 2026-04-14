---
globs: ["src/**/*.{ts,tsx}"]
---

# 성능 규칙

## 렌더링

- Props로 인라인 함수/객체/배열 전달 금지 (useCallback/useMemo)
- 무거운 자식 컴포넌트: React.memo 적용
- 동일 계산 반복: useMemo 캐싱
- 파생 상태를 useState + useEffect로 관리 금지 (렌더 중 계산)

## 네트워크

- 루프 안 API 호출 금지 (배치 요청)
- 동일 데이터 중복 요청 금지 (캐싱 레이어)

## 번들

- import * 금지, 필요한 것만 import
- 배럴 파일 경유 import 금지 (직접 경로)
- 대형 컴포넌트: dynamic import로 지연 로딩

## 런타임

- 이벤트 리스너/타이머 정리 (cleanup 필수)
- 중첩 루프(O(n²)) 금지, Set/Map 활용
- 터치/스크롤 이벤트: { passive: true }
- content-visibility: auto로 오프스크린 렌더링 지연
