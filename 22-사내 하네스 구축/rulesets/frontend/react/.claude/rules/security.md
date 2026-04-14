---
globs: ["src/**/*.{ts,tsx}"]
---

# 보안 규칙

## 입력

- 사용자 입력: params로 전달 (직접 URL 삽입 금지)
- HTML 삽입 시 반드시 DOMPurify 등 새니타이저 라이브러리로 정화 후 사용
- href/src에 사용자 입력 시 javascript: 프로토콜 검증

## 인증

- 클라이언트에서만 권한 체크 금지 (서버 검증 필수)
- 토큰: HttpOnly 쿠키 우선 (localStorage 취약)
- CSRF: SameSite=Strict, HttpOnly, Secure 설정

## 노출 방지

- 에러 메시지: 서버 내부 구조 노출 금지
- console.log: 비밀번호, 토큰, 개인정보 금지
- 환경 변수: 공개되어도 괜찮은 것만 클라이언트에 노출
- 의존성: 알려진 취약점 버전 사용 금지
