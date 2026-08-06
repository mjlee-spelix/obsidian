---
tags: [개발, Git]
date: 2026-04-23
---
# Git - 기본 워크플로우 명령어

## 핵심
- Git의 가장 기본적인 흐름: **clone → branch → add → commit → push → pull**
- 이 흐름만 확실히 익히면 일상적인 협업의 80%는 커버 가능
- 각 명령어가 **어디에서 어디로** 데이터를 옮기는지 이해하는 것이 핵심

## 상세

### Git의 4가지 영역

```
원격 저장소 (Remote)    ← push / → pull,fetch
        ↕
로컬 저장소 (Local Repo) ← commit
        ↕
스테이징 영역 (Staging)  ← add
        ↕
작업 디렉토리 (Working)  ← 내가 편집하는 파일들
```

### git clone — 원격 저장소 복제

```bash
# HTTPS
git clone https://github.com/user/repo.git

# SSH (회사에서 주로 사용)
git clone ssh://git@192.168.10.200:22/spx-agent.git

# 특정 브랜치만 클론
git clone -b feature/branch-name https://github.com/user/repo.git
```

### git status — 현재 상태 확인

```bash
git status          # 변경된 파일, 스테이징 상태 전부 보기
git status -s       # 간단하게 보기 (M: 수정, A: 추가, ??: 미추적)
```

- 습관적으로 자주 치는 게 좋음 — add/commit 전후로 항상 확인

### git add — 스테이징에 올리기

```bash
git add 파일명.txt         # 특정 파일만
git add .                 # 현재 디렉토리 전체 (새 파일 포함)
git add -A                # 삭제된 파일까지 포함해서 전체
git add *.py              # 특정 확장자만
```

| 명령 | 새 파일 | 수정 파일 | 삭제 파일 |
|------|--------|----------|----------|
| `git add .` | ✅ | ✅ | ✅ |
| `git add -A` | ✅ | ✅ | ✅ |
| `git add -u` | ❌ | ✅ | ✅ |

### git commit — 변경사항 확정

```bash
git commit -m "메시지"              # 한 줄 메시지
git commit -m "제목" -m "상세 설명"   # 제목 + 본문
git commit --amend -m "수정 메시지"   # 직전 커밋 메시지 수정 (push 전에만!)
```

- 커밋 메시지 컨벤션 예시: `feat: 로그인 기능 추가`, `fix: 차트 렌더링 버그 수정`

### git push — 원격에 올리기

```bash
git push                          # 현재 브랜치 push
git push origin feature/my-work   # 특정 브랜치 push
git push -u origin feature/my-work # 최초 push 시 (-u로 추적 설정)
```

- `-u` (upstream): 한 번 설정하면 이후 `git push`만 쳐도 됨

### git pull — 원격에서 받기

```bash
git pull                  # 현재 브랜치의 원격 변경사항 받기
git pull origin main      # main 브랜치 내용 받기
```

- `git pull` = `git fetch` + `git merge` (자동으로 합침)
- 충돌이 날 수 있으니 pull 전에 내 변경사항 커밋 or stash 해두기

### git branch — 브랜치 관리

```bash
git branch                    # 브랜치 목록 보기 (* 표시가 현재 브랜치)
git branch feature/new-work   # 새 브랜치 생성 (이동은 안 함)
git branch -d feature/done    # 브랜치 삭제 (병합된 것만)
git branch -D feature/done    # 브랜치 강제 삭제
git branch -a                 # 원격 브랜치까지 전부 보기
```

### git checkout — 브랜치 이동

```bash
git checkout feature/my-work              # 해당 브랜치로 이동
git checkout -b feature/new-work          # 새 브랜치 생성 + 이동 (한 번에)
git checkout feature/keycloak-auth        # 다른 사람이 만든 브랜치로 이동
```

- 최신 Git에서는 `git switch`를 권장 (→ [[Git - 브랜치 전략 명령어]] 참고)

### 일반적인 작업 흐름 정리

```
1. git clone (최초 1회)
2. git checkout -b feature/my-work   ← 새 브랜치 생성
3. (코드 작업)
4. git status                        ← 뭐가 바뀌었는지 확인
5. git add .                         ← 스테이징
6. git commit -m "feat: 기능 추가"    ← 커밋
7. git push -u origin feature/my-work ← 원격에 push
8. (PR/MR 생성 → 코드 리뷰 → 머지)
9. git checkout main                 ← main으로 돌아가기
10. git pull                         ← 최신 반영
```

## 관련 노트
- [[Git - 브랜치 전략 명령어]]
- [[Git - 되돌리기 & 충돌 해결 명령어]]
