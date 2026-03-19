# Findings (2026-03-18)

## Claude Code v2.1.60~v2.1.77 업데이트 조사

### 주요 새 기능
- `/loop` 커맨드 (v2.1.71): 반복 프롬프트 스케줄링
- Cron 스케줄링 도구 (v2.1.71)
- MCP Elicitation (v2.1.76): MCP 서버가 작업 중간에 사용자 입력 요청 가능
- `/effort` 슬래시 커맨드 (v2.1.76)
- `/context` 개선 (v2.1.74): 컨텍스트 무거운 도구, 메모리 블로트, 용량 경고 식별
- `modelOverrides` 설정 (v2.1.73): Bedrock/Vertex/Foundry용 모델 ID 매핑
- `worktree.sparsePaths` (v2.1.76): 대규모 모노레포에서 필요한 디렉토리만 체크아웃
- `/plan` 설명 인수 (v2.1.72): `/plan fix the auth bug` 형태
- ExitWorktree 도구 (v2.1.72)
- Voice STT 10개 언어 추가 (v2.1.69)
- `/copy N` 인덱스 지정 + `w` 키로 파일 저장 (v2.1.77, v2.1.72)

### 설정/훅/권한 변경
- `allowRead` 샌드박스 설정 (v2.1.77): denyRead 내 읽기 재허용
- `PostCompact` 훅 (v2.1.76): 컴팩션 완료 후 실행
- `Elicitation`/`ElicitationResult` 훅 (v2.1.76)
- `InstructionsLoaded` 훅 (v2.1.69)
- `autoMemoryDirectory` 설정 (v2.1.74): 자동 메모리 저장 디렉토리 커스터마이즈
- `includeGitInstructions` 설정 (v2.1.69): git 워크플로우 지시사항 제거 가능
- PreToolUse 훅 allow가 deny 권한규칙 우회하던 버그 수정 (v2.1.77)
- managed policy ask 규칙이 user allow로 우회되던 버그 수정 (v2.1.74)
- `CLAUDE_CODE_DISABLE_CRON` 환경변수 (v2.1.72)
- `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 환경변수 (v2.1.74)
- `sandbox.enableWeakerNetworkIsolation` 설정 (v2.1.69)
- `-n`/`--name` CLI 플래그 (v2.1.76): 시작 시 표시 이름 설정

### 성능 개선
- Opus 4.6 기본 max output 64k, 상한 128k (v2.1.77)
- macOS 시작 ~60ms 단축: keychain 병렬 읽기 (v2.1.77)
- `--resume` 최대 45% 빨라짐, ~100-150MB 메모리 절감 (v2.1.77)
- 프롬프트 렌더링 ~74% 감소 (v2.1.70)
- 시작 메모리 ~426KB 감소 (v2.1.70)
- 번들 크기 ~510KB 감소 (v2.1.72)
- prompt cache 무효화 수정 → 입력 토큰 12배 절감 (v2.1.72)
- 자동 컴팩션 3회 실패 시 서킷 브레이커 (v2.1.76)
- 진행 메시지 컴팩션에서 생존하던 메모리 릭 수정 (v2.1.77)
- 자동 업데이터 중복 다운로드로 수십 GB 메모리 사용 버그 수정 (v2.1.77)

### 새 플래그/옵션
- `-n`/`--name <name>`: 표시 이름 설정
- `--effort`: CLI에서 effort 레벨 설정
- `/branch` (구 `/fork`)
- `/loop 5m check the deploy`: 반복 실행
- `/effort low|medium|high`: effort 레벨 변경
- `/reload-plugins`: 플러그인 변경 활성화
- effort 레벨 단순화: low/medium/high (○ ◐ ●), max 제거

## 현재 설정 대비 미활용 기능 분석

### 즉시 적용 권장
1. **PostCompact 훅**: PreCompact는 있는데 PostCompact가 없음. compact 후 컨텍스트 복구 검증에 활용 가능
2. **autoMemoryDirectory**: 자동 메모리 저장 위치를 프로젝트별로 관리 가능
3. **allowRead 샌드박스**: 세밀한 파일 접근 제어
4. **InstructionsLoaded 훅**: CLAUDE.md 로딩 추적/로깅에 유용
5. **`/effort` 슬래시 커맨드**: effortLevel 설정은 있지만 대화 중 동적 변경 활용
6. **worktree.sparsePaths**: Agent Teams로 대규모 레포 작업 시 디스크/시간 절약

### 검토 후 적용 권장
7. **MCP Elicitation**: Playwright 등 MCP 서버에서 중간 입력 요청 지원
8. **`/loop` + Cron**: 반복 작업 자동화 (빌드 모니터링, 테스트 실행 등)
9. **modelOverrides**: Bedrock/Vertex 사용 시 모델 ID 매핑
10. **includeGitInstructions: false**: 커스텀 git 워크플로우 스킬 사용 시 내장 지시사항 제거
11. **`/plan` 설명 인수**: `/plan fix the auth bug` 형태로 계획 즉시 생성
12. **`/copy N` + `w` 키**: 특정 응답 복사/파일 저장

### 주요 버그 수정 (현재 설정에 영향)
- **PreToolUse 훅 allow가 deny 우회하던 버그 수정 (v2.1.77)**: PreToolUse 훅 사용 중이므로 중요
- **SessionStart 훅 --resume/--continue 시 2회 실행 버그 수정 (v2.1.73)**: SessionStart 훅 사용 중
- **CJK 문자 UI 출혈 수정 (v2.1.77)**: 한국어 사용자
- **compound bash "Always Allow" 버그 수정 (v2.1.77)**: Bash(*) allow 사용 중
- **tmux 관련 다수 수정**: iTerm2/tmux 사용 시

## 주의할 점
- Opus 4, 4.1은 first-party API에서 제거됨 (v2.1.68), 자동으로 4.6으로 이동
- `/output-style` deprecated → `/config` 사용 (v2.1.73)
- effort 레벨에서 `max` 제거됨, low/medium/high만 존재 (v2.1.72)
- Agent tool의 `resume` 파라미터 제거 → `SendMessage({to: agentId})` 사용 (v2.1.77)
- `--plugin-dir`가 하나의 경로만 받음, 복수는 반복 사용 (v2.1.76)

## 참고 링크
- https://github.com/anthropics/claude-code/releases
- https://code.claude.com/docs/en/settings
