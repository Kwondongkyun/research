---
globs: ["src/**/*.{ts,tsx}"]
---

# 코드 품질 체크리스트

## Null/Undefined

- 옵셔널 체이닝(`?.`) 누락 확인
- 빈 배열/객체 인덱스 접근 시 length 체크

## 비동기

- await 누락 확인
- Promise 에러 미처리 확인
- 언마운트 후 setState → cleanup 필수

## React 패턴

- useEffect 의존성: 객체/배열 직접 전달 금지
- Stale closure: 함수형 업데이터(`prev =>`) 사용
- 배열/객체 직접 mutation 금지 (불변 업데이트)
- 파생 상태를 useState + useEffect로 관리 금지 (렌더 중 계산)
- 조건부 렌더링 0 버그: `{count && <JSX />}` → `{count > 0 && <JSX />}`

## 에러 처리

- 빈 catch 블록 금지
- catch(error: any) 금지, 타입 명시
- catch: console.log만 금지, 사용자 피드백 필수

## 유지보수

- DRY 위반: 동일 패턴 3회 이상 반복 시 추출
- 순환복잡도: if 중첩 3단계 이상 → early return
- 범용 네이밍 금지 (`data`, `info`, `temp`)
- 매직 넘버/스트링 → 상수로 의미 부여
