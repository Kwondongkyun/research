# Git Worktree — 하나의 저장소, 여러 브랜치 동시 작업

> 최종 수정: 2026-03-10

## TL;DR

Git Worktree는 하나의 git 저장소에서 **여러 브랜치를 각각 다른 폴더로 동시에 열어두는 기능**이다. 브랜치 전환(`checkout`) 없이 폴더만 이동하면 되므로, stash/unstash 지옥에서 벗어날 수 있다.

---

## 1. 핵심 개념: Branch vs Worktree — 같은 것인가?

**다르다.** 서로 다른 계층의 개념이다.

### Branch (브랜치) = 논리적 포인터

브랜치는 **특정 커밋을 가리키는 포인터**다. 물리적 실체가 없다.

```
main        → commit abc1234
feature/auth → commit def5678
hotfix       → commit ghi9012

실체: .git/refs/heads/ 안의 40글자 해시값 텍스트 파일
```

```bash
cat .git/refs/heads/main
# abc1234... (40자 SHA-1 해시)
```

브랜치는 그냥 **"이 커밋이 최신이야"를 표시하는 이름표**일 뿐이다.

### Worktree (워크트리) = 물리적 작업 공간

워크트리는 **브랜치의 코드가 실제로 펼쳐져 있는 물리적 폴더**다.

```
~/my-project/           ← main 브랜치의 워크트리 (기본)
~/my-project-hotfix/    ← hotfix 브랜치의 워크트리 (추가)
```

### 비유로 이해하기

```
Branch  = 책의 "목차 항목" (3장, 5장, 부록 A)
Worktree = 책을 실제로 펼쳐놓은 "책상"

일반 git: 책상이 1개 → 한 번에 한 장만 펼칠 수 있음
          3장 보다가 5장 보려면 → 3장 닫고 5장을 펼침 (checkout)

Worktree: 책상이 여러 개 → 각 책상에 다른 장을 펼쳐둠
          3장은 왼쪽 책상, 5장은 오른쪽 책상 → 돌아보기만 하면 됨
```

또는:

```
Branch  = TV 채널 (1번, 2번, 3번)
Worktree = TV 자체

일반 git: TV 1대 → 채널 전환해가며 봄 (한 번에 1개만)
Worktree: TV 여러 대 → 각 TV에 다른 채널 틀어놓음 (동시에 여러 개)
```

### 관계 정리

```
저장소 (Repository)
  └── .git/              ← 1개 (공유)
       ├── objects/      ← 커밋, 트리, 블롭 (모든 워크트리가 공유)
       ├── refs/heads/   ← 브랜치 포인터들 (모든 워크트리가 공유)
       └── worktrees/    ← 추가 워크트리 메타데이터
            ├── hotfix/
            └── feature/

  └── Worktree 1 (기본)  ← main 브랜치의 물리적 폴더
  └── Worktree 2 (추가)  ← hotfix 브랜치의 물리적 폴더
  └── Worktree 3 (추가)  ← feature 브랜치의 물리적 폴더
```

**핵심: Branch는 "무엇을", Worktree는 "어디서"를 정의한다.**

| | Branch | Worktree |
|--|--------|----------|
| **정의** | 커밋을 가리키는 포인터 | 브랜치의 코드가 펼쳐진 폴더 |
| **성격** | 논리적 (이름표) | 물리적 (디렉토리) |
| **실체** | `.git/refs/heads/` 안의 해시값 | 파일 시스템의 실제 폴더 |
| **개수** | 제한 없음 (수백 개 가능) | 기본 1개 + 추가 생성 |
| **동시 사용** | 한 번에 1개만 checkout | **여러 개 동시 사용** |
| **비유** | 채널 번호 | TV 자체 |
| **관계** | 워크트리에 의해 "열린다" | 브랜치를 "열어서 보여준다" |

**Branch 없이 Worktree는 의미 없고, Worktree 없이 Branch는 볼 수 없다.** 상호 보완 관계.

---

## 2. 내부 원리: Worktree는 어떻게 작동하는가?

### .git 디렉토리 공유 구조

일반적으로 git 저장소를 clone하면 이런 구조다:

```
my-project/
  ├── .git/          ← 저장소 메타데이터 (커밋, 브랜치, 설정 전부 여기)
  ├── src/           ← 작업 파일
  ├── package.json
  └── ...
```

Worktree를 추가하면:

```
my-project/                          ← 메인 워크트리
  ├── .git/                          ← 진짜 .git (모든 데이터의 원본)
  │     ├── objects/                 ← 모든 커밋/파일 객체 (공유)
  │     ├── refs/                    ← 모든 브랜치 포인터 (공유)
  │     ├── config                   ← 설정 (공유)
  │     └── worktrees/               ← 추가 워크트리 메타데이터
  │           ├── hotfix/
  │           │     ├── HEAD         ← 이 워크트리가 가리키는 브랜치
  │           │     ├── index        ← 이 워크트리의 스테이징 영역
  │           │     └── gitdir       ← 워크트리 폴더 위치
  │           └── feature/
  │                 ├── HEAD
  │                 ├── index
  │                 └── gitdir
  ├── src/
  └── ...

my-project-hotfix/                   ← 추가 워크트리 1
  ├── .git                           ← 파일! (폴더가 아님)
  │                                     내용: "gitdir: ../my-project/.git/worktrees/hotfix"
  │                                     → 진짜 .git으로 연결하는 심볼릭 참조
  ├── src/                           ← hotfix 브랜치의 파일들
  └── ...

my-project-feature/                  ← 추가 워크트리 2
  ├── .git                           ← "gitdir: ../my-project/.git/worktrees/feature"
  ├── src/                           ← feature 브랜치의 파일들
  └── ...
```

### 뭐가 공유되고, 뭐가 독립인가?

| 요소 | 공유 / 독립 | 설명 |
|------|------------|------|
| `.git/objects/` | **공유** | 모든 커밋, 파일 데이터 → 디스크 절약 |
| `.git/refs/` | **공유** | 브랜치 목록, 태그 → 어디서든 같은 브랜치 보임 |
| `.git/config` | **공유** | remote URL, user 설정 등 |
| `.git/hooks/` | **공유** | pre-commit, post-commit 등 |
| `HEAD` | **독립** | 각 워크트리가 다른 브랜치를 가리킴 |
| `index` (스테이징) | **독립** | 각 워크트리에서 독립적으로 add/commit |
| 작업 파일 | **독립** | 각 폴더에 별도의 소스 코드 |
| `node_modules/` | **독립** | 각 워크트리에서 별도 install 필요 |

### 비유: 도서관의 책

```
.git/objects/ = 도서관의 서고 (원본 책들이 보관된 곳)
               → 1벌만 존재, 모든 열람실이 공유

Worktree     = 열람실 (책을 꺼내서 펼쳐놓는 곳)
               → 여러 개 존재 가능, 각각 다른 책을 펼쳐놓음

Branch       = 책의 제목 (어떤 책을 펼쳤는지)
               → 열람실 A는 "3장", 열람실 B는 "5장"을 펼쳐놓음
```

서고(objects)는 1개인데, 열람실(worktree)을 추가해서 동시에 여러 책(branch)을 볼 수 있는 구조.

### clone vs worktree

저장소를 **clone**하는 것과 **worktree를 추가**하는 건 다르다:

```
git clone → 완전히 독립된 저장소 복사본
            .git/이 별도 → 커밋 히스토리도 별도 → push/pull 별도
            용량: 전체 저장소 크기

git worktree add → 같은 저장소의 추가 작업 공간
                   .git을 공유 → 커밋 히스토리 공유 → 한쪽에서 커밋하면 다른쪽에서 보임
                   용량: 작업 파일만 (objects는 공유)
```

| | `git clone` | `git worktree add` |
|--|------------|-------------------|
| .git | 별도 복사 | **공유** |
| 커밋 히스토리 | 독립 (push/pull 필요) | **즉시 공유** |
| 브랜치 | 독립 | **공유** |
| 디스크 사용량 | 전체 저장소 크기 | 작업 파일만 |
| 용도 | 완전히 다른 환경 | 같은 저장소에서 병렬 작업 |

---

## 3. 왜 필요한가?

일반적인 git 브랜치 작업 흐름:

```
feature/auth 작업 중...
  → 긴급 hotfix 요청!
  → git stash
  → git checkout main
  → hotfix 작업 + 커밋
  → git checkout feature/auth
  → git stash pop
  → "어... 뭐 하고 있었지?"
```

**문제점:**
- 브랜치 전환할 때마다 파일이 바뀐다
- stash가 쌓이면 어떤 게 어떤 건지 헷갈린다
- 빌드 캐시가 날아가서 다시 빌드해야 한다
- 컨텍스트 스위칭 비용이 크다

### Worktree는 이걸 해결한다

```
my-project/                    ← main 브랜치 (원본)
my-project-hotfix/             ← hotfix 브랜치 (worktree)
my-project-feature-auth/       ← feature/auth 브랜치 (worktree)
```

**같은 저장소**인데 브랜치마다 **물리적으로 다른 폴더**가 생긴다. 각 폴더는 완전히 독립적이다:
- 파일 변경이 서로 영향을 주지 않는다
- 빌드 캐시가 각각 유지된다
- 터미널 탭만 바꾸면 된다

---

## 4. 기본 명령어

### 워크트리 생성

```bash
# 기존 브랜치로 워크트리 생성
git worktree add ../my-project-hotfix hotfix

# 새 브랜치 만들면서 워크트리 생성
git worktree add -b feature/payment ../my-project-payment

# 경로만 지정하면 폴더명이 브랜치명이 됨
git worktree add ../hotfix
```

### 워크트리 확인

```bash
git worktree list
# /Users/me/my-project            abc1234 [main]
# /Users/me/my-project-hotfix     def5678 [hotfix]
# /Users/me/my-project-payment    ghi9012 [feature/payment]
```

### 워크트리 제거

```bash
# 워크트리 삭제 (디렉토리도 함께 제거)
git worktree remove ../my-project-hotfix

# 삭제된 워크트리 정리
git worktree prune
```

### 명령어 요약

| 명령어 | 설명 |
|--------|------|
| `git worktree add <path> <branch>` | 기존 브랜치로 워크트리 생성 |
| `git worktree add -b <branch> <path>` | 새 브랜치 + 워크트리 생성 |
| `git worktree list` | 모든 워크트리 목록 |
| `git worktree remove <path>` | 워크트리 제거 |
| `git worktree prune` | 삭제된 워크트리 참조 정리 |

---

## 5. 실무 활용 시나리오

### 1. 긴급 핫픽스

기능 개발 중 긴급 버그 수정이 필요할 때:

```bash
# 현재 feature/auth 작업 중 — 그대로 두고
git worktree add -b hotfix/critical-bug ../hotfix main

# 별도 폴더에서 핫픽스 작업
cd ../hotfix
# ... 수정, 커밋, 푸시 ...

# 끝나면 워크트리 정리
cd ../my-project
git worktree remove ../hotfix
```

기존 작업을 stash할 필요가 없다.

### 2. PR 리뷰

동료 PR을 검토할 때 현재 작업을 중단하지 않고:

```bash
git worktree add ../review-pr-123 origin/feature/user-profile
cd ../review-pr-123
# 코드 확인, 테스트 실행
```

### 3. 여러 기능 동시 개발

```bash
git worktree add -b feature/auth ../auth
git worktree add -b feature/payment ../payment
git worktree add -b feature/dashboard ../dashboard
```

각 폴더에서 독립적으로 개발하고, 각각의 터미널/IDE 창에서 작업하면 된다.

---

## 6. 일반 브랜치 전환 vs Worktree vs Clone 종합 비교

| | 일반 브랜치 전환 | Worktree | Clone |
|--|-----------------|----------|-------|
| 동시 작업 | 한 번에 1개만 | **여러 브랜치 동시** | 여러 저장소 동시 |
| 전환 방법 | `git checkout` (파일 변경) | 폴더 이동 (파일 그대로) | 별도 폴더 |
| stash 필요 | 자주 필요 | **불필요** | 불필요 |
| 빌드 캐시 | 전환 시 무효화 가능 | 각 폴더 독립 유지 | 각 폴더 독립 유지 |
| 커밋 공유 | 즉시 (같은 .git) | **즉시 (같은 .git)** | push/pull 필요 |
| 디스크 사용 | 최소 | 작업 파일만 추가 | **전체 저장소 복사** |
| .git 공유 | 하나 | **하나** (공유) | 별도 |
| 컨텍스트 전환 | 높음 | **낮음** (탭 전환) | 낮음 (탭 전환) |
| 사용 시점 | 간단한 브랜치 이동 | 병렬 작업, 임시 작업 | 완전히 분리된 환경 |

---

## 7. 주의사항 및 Best Practices

### 주의사항

- **같은 브랜치를 두 워크트리에서 체크아웃할 수 없다** — git이 차단한다
- 워크트리는 **작업 파일을 복사**하므로 디스크 공간을 차지한다 (`.git`은 공유)
- 사용이 끝난 워크트리는 **반드시 정리**해야 한다 (`git worktree remove`)
- 추가 워크트리에서 `node_modules/` 등은 **별도로 install** 해야 한다

### Best Practices

- 워크트리 디렉토리 이름을 **의미 있게** 짓자 (e.g., `../hotfix-login-bug`)
- 장기 워크트리보다 **단기 목적별**로 만들고 바로 정리하자
- `git worktree list`로 주기적으로 확인하자
- 팀에서 사용할 때는 워크트리 위치 컨벤션을 정하자

---

## 8. Claude Code에서의 Worktree

Claude Code에서 Agent Team과 Worktree를 조합하면 **에이전트별로 격리된 브랜치에서 병렬 작업**이 가능하다:

```
main 브랜치
.claude/worktrees/
  ├── feat-auth/        ← 에이전트 A 전용
  ├── feat-todo/        ← 에이전트 B 전용
  └── feat-dashboard/   ← 에이전트 C 전용
```

- 파일 충돌이 **물리적으로 원천 차단**된다
- 각 에이전트가 독립된 환경에서 작업한다
- 작업 완료 후 main에 머지한다

자세한 조합 방식은 [Agent Team부터 Ralph Loop까지](../Agent%20Team부터%20Ralph%20Loop까지/claude-code-agent-team_2026-03-01.md)의 조합 B, D를 참고.

---

## 레퍼런스

- [Git 공식 문서 — git-worktree](https://git-scm.com/docs/git-worktree)
- [Git Worktree 완벽 가이드](https://jonny-cho.github.io/git/2025-07-02-git-worktree-complete-guide/)
- [Git Worktree Tutorial — DataCamp](https://www.datacamp.com/tutorial/git-worktree-tutorial)
- [Git Tower — Git Worktree FAQ](https://www.git-tower.com/learn/git/faq/git-worktree)
- [git worktree를 활용해서 일하기](https://rein.kr/posts/2025-05-04-work-with-git-worktree/)
