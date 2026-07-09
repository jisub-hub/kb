---
tags:
  - collaboration
  - git
  - workflow
  - branching
  - pr
created: 2026-06-16
---

# Git Workflow — 브랜치 전략 & PR 프로세스

> [!summary] 한 줄 요약
> 팀 규모·배포 주기에 맞는 브랜치 전략을 선택하고, PR·Merge 프로세스를 표준화한다.

---

## 1. 브랜치 전략 선택

| 전략 | 팀 규모 | 배포 주기 | 특징 |
|------|---------|----------|------|
| **GitHub Flow** | 소~중 | 상시 배포 (CD) | main + feature 2단계. 단순. |
| **Git Flow** | 중~대 | 정기 릴리스 | develop/release/hotfix 다단계. 복잡. |
| **Trunk-based** | 대형 | 상시 배포 + Feature Flag | main만 사용. Feature Toggle로 격리. |

---

## 2. GitHub Flow (권장 — 스타트업·중소팀)

```
main ──────●─────────────────────●── (항상 배포 가능)
            \                   /
             feat/order-cancel ●─●
```

```bash
# 1. main 최신화
git switch main && git pull origin main

# 2. feature 브랜치 생성 (issue 번호 포함)
git switch -c feat/123-order-cancel

# 3. 작업 후 커밋 (Conventional Commits)
git commit -m "feat(order): 주문 취소 API 추가"

# 4. PR 생성 → 리뷰 → Squash Merge
gh pr create --title "feat(order): 주문 취소 API" --body "..."
```

### 브랜치 네이밍 규칙

```
feat/<issue-no>-<short-desc>     새 기능
fix/<issue-no>-<short-desc>      버그 수정
refactor/<short-desc>            리팩터링
hotfix/<short-desc>              프로덕션 긴급 수정
chore/<short-desc>               설정, 의존성
```

---

## 3. Git Flow (정기 릴리스 팀)

```
main     ─────●────────────────────●────
              ↑  release/1.2.0     ↑
develop  ──●──●──●──●──●──●──●──●──●──
              \                 /
               feat/order-search
```

| 브랜치 | 역할 | merge 대상 |
|--------|------|-----------|
| `main` | 프로덕션 릴리스 태그 | ← release, hotfix |
| `develop` | 다음 릴리스 통합 | ← feature |
| `feature/*` | 기능 개발 | → develop |
| `release/*` | 릴리스 준비 (QA, 버전 범프) | → main + develop |
| `hotfix/*` | 프로덕션 긴급 패치 | → main + develop |

---

## 4. PR 프로세스 & 리뷰 규칙

### PR 크기
```
✅ 이상적인 PR: 400 라인 이하, 한 가지 목적
❌ 거대 PR: 1000+ 라인 → 리뷰어가 맥락 파악 불가
→ 크다면 "기반 PR → 기능 PR" 으로 분리
```

### 리뷰어 지정 기준
```
- 최소 1명의 Approver 필요 (소규모 팀)
- 변경 파일 오너십(CODEOWNERS) 있는 경우 필수 리뷰
- 자기 PR을 자기가 Merge하지 않는다 (긴급 hotfix 제외)
```

### Merge 전략
| 방식 | 언제 | 히스토리 |
|------|------|----------|
| **Squash Merge** | feature → main (GitHub Flow) | 깔끔, 커밋 하나로 |
| **Merge Commit** | release → main (Git Flow) | 병합 이력 보존 |
| **Rebase Merge** | 선호에 따라 | 선형 히스토리 |

---

## 5. 충돌 해결 프로세스

```bash
# main 변경사항을 내 브랜치에 반영
git fetch origin
git rebase origin/main

# 충돌 발생 시
git status            # 충돌 파일 확인
# 파일 편집 후
git add <resolved-file>
git rebase --continue

# 취소
git rebase --abort
```

---

## 6. 태그 & 릴리스

```bash
# Semantic Versioning: MAJOR.MINOR.PATCH
# MAJOR: 하위 호환 불가 변경
# MINOR: 하위 호환 새 기능
# PATCH: 버그 수정

git tag -a v1.2.0 -m "Release 1.2.0: 주문 취소 기능 추가"
git push origin v1.2.0

# GitHub Release 생성 (CHANGELOG 자동)
gh release create v1.2.0 --title "v1.2.0" --generate-notes
```

---

## 7. Git 설정 & 도움이 되는 alias

```bash
# ~/.gitconfig
[alias]
  st = status -s
  sw = switch
  lg = log --oneline --graph --decorate --all
  pushf = push --force-with-lease   # 안전한 force push

# 필수 전역 설정
git config --global pull.rebase true      # pull 시 rebase
git config --global init.defaultBranch main
git config --global core.autocrlf input   # LF 통일 (Mac/Linux)
```

---

## 8. .gitignore 핵심 항목

```gitignore
# 빌드 산출물
target/
build/
*.class
*.jar

# IDE
.idea/
.vscode/
*.iml

# 환경 설정 (절대 커밋 금지)
.env
.env.local
application-local.yml
application-secret.yml

# OS
.DS_Store
Thumbs.db

# 로그·임시 파일
*.log
*.tmp
```

---

## 9. 관련
- [[Coding-Convention]] — 커밋 메시지, 코드 스타일
- [[Code-Review]] — PR 리뷰 기준, 피드백 방법
- [[Team-Communication]] — 브랜치 전략 결정 배경, 팀 합의 방식
- [[../../infra/ssh-cicd/GitLab-CICD]] — CI/CD 파이프라인 연동
