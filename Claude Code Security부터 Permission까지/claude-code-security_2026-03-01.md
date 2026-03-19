# Claude Code Security부터 Permission까지: AI 코딩 도구의 보안 체계

> 최종 수정: 2026-03-01

## TL;DR

Claude Code의 Permission 시스템(Allow/Ask/Deny)으로 위험 명령을 제어하고, 별도 제품인 Claude Code Security는 AI가 코드 보안 취약점을 찾아 패치까지 제안하는 서비스다. 두 개는 완전히 다른 것.

## 목차

1. Claude Code의 Permission 시스템
2. Claude Code Security란?
3. Claude Code Security 신청 방법

---

## 1. Claude Code의 Permission 시스템

Claude Code에는 보안을 위한 **다층 Permission 시스템**이 내장되어 있다. 설정 화면에서 Allow / Ask / Deny / Workspace 4개 탭으로 관리한다.

### Ask (확인 후 실행)

Claude가 실행 전 **항상 사용자에게 확인을 요청**하는 명령어들이다.

| 규칙 | 왜 Ask인가 |
|------|-----------|
| `Bash(*chmod *)` | 파일 권한 변경 — 보안 영향 |
| `Bash(*chown *)` | 파일 소유자 변경 — 시스템 영향 |
| `Bash(*crontab *)` | 예약 작업 — 지속적 시스템 변경 |
| `Bash(*curl *)` | 네트워크 요청 — 외부 통신 |
| `Bash(*git clean -f*)` | 추적 안 된 파일 삭제 — 비가역적 |
| `Bash(*git push *)` | 원격 저장소 변경 — 다른 사람에게 영향 |
| `Bash(*git reset --hard*)` | 변경사항 강제 초기화 — 작업 손실 가능 |
| `Bash(*launchctl *)` | macOS 서비스 관리 — 시스템 영향 |
| `Bash(*pkill *)` | 프로세스 강제 종료 — 데이터 손실 가능 |

### Deny (완전 차단)

Claude가 **절대 실행할 수 없는** 명령어들이다.

| 규칙 | 위험도 | 이유 |
|------|--------|------|
| `Bash(*dd *)` | 치명적 | 디스크 직접 쓰기 — 데이터 완전 파괴 가능 |
| `Bash(*diskutil erase*)` | 치명적 | 디스크 포맷 |
| `Bash(*diskutil partitionDisk*)` | 치명적 | 파티션 변경 — 데이터 손실 |
| `Bash(*fdisk*)` | 치명적 | 디스크 파티션 편집 |
| `Bash(*git push --force*)` | 높음 | 원격 히스토리 강제 덮어쓰기 |
| `Bash(*killall *)` | 높음 | 모든 프로세스 일괄 종료 |
| `Bash(*mkfs *)` | 치명적 | 파일시스템 생성 (기존 데이터 파괴) |
| `Bash(*npm publish*)` | 높음 | 패키지 공개 배포 — 되돌리기 어려움 |
| `Bash(*poweroff*)` | 높음 | 시스템 종료 |
| `Bash(*reboot*)` | 높음 | 시스템 재부팅 |
| `Bash(*rm -rf *)` | 치명적 | 재귀적 강제 삭제 — 복구 불가 |
| `Bash(*shutdown*)` | 높음 | 시스템 종료 |
| `Bash(*sudo *)` | 치명적 | 루트 권한 실행 — 모든 보안 우회 |
| `Read(./.env.*)` | 높음 | 환경변수 파일 (API 키, DB 비밀번호 등) |
| `Read(./.env)` | 높음 | 환경변수 파일 |
| `Read(./secrets/**)` | 높음 | 시크릿 디렉토리 전체 |
| `Read(~/.ssh/**)` | 높음 | SSH 키 — 서버 접근 자격증명 |

### 적용 범위

| 탭 | 범위 | 저장 위치 |
|----|------|----------|
| **Allow** | 전체 프로젝트 | `~/.claude/settings.json` |
| **Ask** | 전체 프로젝트 | `~/.claude/settings.json` |
| **Deny** | 전체 프로젝트 | `~/.claude/settings.json` |
| **Workspace** | 현재 프로젝트만 | `.claude/settings.json` (프로젝트 루트) |

Allow / Ask / Deny는 **전역(global) 설정**으로 모든 프로젝트에 적용된다. Workspace만 현재 프로젝트에만 적용된다.

### 전체 구조 요약

```
Allow  → 자동 실행 (안전한 읽기/검색 등)
Ask    → 확인 후 실행 (git push, curl, chmod 등)
Deny   → 완전 차단
         ├── 시스템 파괴 (dd, mkfs, rm -rf, diskutil)
         ├── 시스템 제어 (sudo, poweroff, reboot, shutdown)
         ├── 비가역 배포 (npm publish, git push --force)
         └── 민감 파일 읽기 (.env, secrets, .ssh)
```

민감 파일 읽기까지 Deny에 넣으면 Claude가 실수로라도 시크릿을 컨텍스트에 올리는 것을 원천 차단할 수 있다.

---

## 2. Claude Code Security란?

Claude Code의 Permission 시스템과는 **완전히 다른 별도 제품**이다.

| | Permission 시스템 | Claude Code Security |
|--|------------------|---------------------|
| **목적** | Claude가 위험한 명령 실행 못하게 제한 | 코드베이스의 **보안 취약점을 AI가 찾고 패치까지 제안** |
| **대상** | Claude Code 사용자 | Enterprise/Team 고객, 오픈소스 관리자 |

### 핵심 요약

2026년 2월 20일 출시된 제한적 연구 미리보기(Limited Research Preview)로, Claude Opus 4.6 기반이다. Anthropic 내부 Frontier Red Team이 연구한 결과물로, 오픈소스 프로덕션 코드에서 수십 년간 미탐지된 취약점 500개 이상을 발견했다. 코드를 인간 보안 연구자처럼 이해·추론하며, 발견부터 패치 제안까지 하나의 루프로 압축한다. 단, 모든 패치 적용에는 개발자의 최종 승인이 필요하다.

### 3단계 작동 방식: Scan → Validate → Patch

**Step 1 — Scan**
기존 도구는 파일 단위로 패턴을 매칭한다. Claude Code Security는 코드를 읽고 추론한다.

| 항목 | 기존 SAST | Claude Code Security |
|------|----------|---------------------|
| 분석 방식 | 알려진 패턴 매칭 | 코드를 읽고 추론 (인간 연구자처럼) |
| 컴포넌트 관계 | 파악 어려움 | 컴포넌트 간 상호작용 이해 |
| 데이터 흐름 | 제한적 | 애플리케이션 전체 데이터 흐름 추적 |

**Step 2 — Validate**
모든 발견사항이 적대적 검증 과정(adversarial verification pass)을 거치며, 표면화되기 전에 스스로 결과를 검증하여 오탐(False Positive)을 최소화한다.

**Step 3 — Patch**
취약점을 찾는 데서 멈추지 않고, 기존 코드 스타일을 유지하면서 구체적인 패치 코드를 제안한다. 단, 개발자의 최종 승인 없이는 어떤 패치도 적용되지 않는다.

### 실제 발견 사례

- **GhostScript**: Git 커밋 이력을 분석해서 미패치 취약점 추론
- **OpenSC**: 퍼저가 못 찾은 버퍼 오버플로우 발견
- **CGIF**: LZW 압축 알고리즘의 설계 가정 오류 발견 (100% 코드 커버리지에서도 미탐지)

### 시장 충격

발표 당일 CrowdStrike(-8%), Cloudflare(-8%) 등 보안주가 급락했다. 기존 보안 도구가 "찾고 리포트"에서 끝났다면, Claude Code Security는 "찾고 패치까지" 한 루프로 만든 것이 핵심이다.

---

## 3. Claude Code Security 신청 방법

### 현재 상태

**Limited Research Preview** 단계로 일반 사용자는 아직 사용 불가.

### 신청 경로

**[claude.com/contact-sales/security](https://claude.com/contact-sales/security)** 에서 "Join the waitlist" 대기자 등록.

### 대상별 안내

| 대상 | 비용 | 비고 |
|------|------|------|
| Enterprise / Team 고객 | 요금 미공개 (sales 문의) | Sales 통해 신청 |
| 오픈소스 관리자 | 무료 (변경 가능) | 우선 접근(Expedited Access) |

### 오픈소스 관리자 자격 요건

- GitHub Stars 5,000+ 이상인 공개 저장소의 주요 관리자/코어 팀 멤버
- 월간 NPM 다운로드 1M+ 이상인 패키지 관리자
- 최근 3개월 내 커밋, 릴리즈, PR 리뷰 활동이 있어야 함
- 위 기준에 안 맞아도 생태계에서 중요한 의존성을 관리한다면 신청 가능 (사유 설명 필요)

### 회사 계정으로 신청 시 주의

- 회사 코드가 외부 AI 서비스로 나감 — 보안 사고로 간주될 수 있음
- 승인 없이 사용 시 사내 보안 정책 위반 가능성
- 신청 폼에 회사명/직책 입력 — 회사 대표로 문의한 것처럼 보일 수 있음
- **보안팀 또는 상급자에게 먼저 공유 후 공식적으로 신청**하는 것을 권장

개인적으로 궁금하면 사이드 프로젝트나 개인 계정(개인 이메일)으로 대기자 등록만 해두는 것이 안전하다.

---

## 레퍼런스

- [Introducing Claude Code Security](https://www.anthropic.com/news/claude-code-security)
- [Claude Code Security 신청](https://claude.com/contact-sales/security)
- [Claude Code Permissions](https://docs.anthropic.com/en/docs/claude-code/security)
