---
globs: ["src/**/*form*", "src/**/*Form*", "src/features/**/schemas.*"]
---

# 폼 규칙

## 스택

- React Hook Form + Zod + shadcn/ui Form 조합

## 스키마

- 위치: features/[domain]/schemas.ts
- Zod z.infer<>로 타입 추출 (별도 interface 금지)

## 구현

- Submit: form.handleSubmit() 사용 (e.preventDefault 직접 금지)
- 로딩 상태: form.formState.isSubmitting (별도 useState 금지)
- 필드 에러: FormMessage 자동 표시
- 서버 에러: form.setError('fieldName' 또는 'root')

## 금지

- register() 직접 사용 (FormField + FormControl 필수)
- 폼 상태 별도 useState로 관리
