---
tags: [개발, Git]
date: 2026-04-23
---
# Git - 브랜치 전략 명령어

## 핵심
- `switch`/`restore`는 `checkout`의 역할을 분리한 최신 명령어 — 섞어 써도 되지만 용도 구분이 명확해짐
- `merge`는 이력을 보존하고, `rebase`는 이력을 깔끔하게 정리 — 팀 컨벤션에 따라 선택
- `cherry-pick`으로 특정 커밋만 골라서 가져올 수 있음
- `fetch`는 원격 정보만 가져오고 합치지는 않음 — pull 전에 확인용으로 유용

## 상세

### git switch / git restore — checkout 대체

`checkout`이 "브랜치 이동"과 "파일 되돌리기" 두 가지를 다 해서 헷갈렸음 → Git 2.23부터 분리됨

```bash
# 브랜치 이동 (checkout 대체)
git switch main                    # main 브랜치로 이동
git switch feature/my-work         # 다른 브랜치로 이동
git switch -c feature/new-work     # 새 브랜치 생성 + 이동 (checkout -b 대체)

# 파일 되돌리기 (checkout -- 파일 대체)
git restore 파일명.txt              # 수정한 파일을 마지막 커밋 상태로 되돌리기
git restore --staged 파일명.txt     # 스테이징 취소 (add 취소)
```

| 옛날 방식 | 새 방식 | 용도 |
|----------|--------|------|
| `git checkout branch` | `git switch branch` | 브랜치 이동 |
| `git checkout -b branch` | `git switch -c branch` | 브랜치 생성+이동 |
| `git checkout -- file` | `git restore file` | 파일 되돌리기 |
| `git checkout HEAD -- file` | `git restore --source HEAD file` | 특정 커밋에서 파일 복원 |

### git fetch — 원격 정보만 가져오기

```bash
git fetch                  # 원격의 모든 브랜치 정보 업데이트 (합치진 않음)
git fetch origin           # origin의 정보 가져오기
git fetch --prune          # 원격에서 삭제된 브랜치 정보도 정리
```

```
git fetch  → 원격 정보만 다운로드 (내 코드 변경 없음)
git pull   → git fetch + git merge (자동으로 합침)
```

- 동료가 새 브랜치 만들었는데 안 보일 때 → `git fetch` 먼저!

### git merge — 브랜치 합치기 (이력 보존)

```bash
# main에 feature 브랜치 합치기
git switch main
git merge feature/my-work

# 합치고 나서 feature 브랜치 삭제
git branch -d feature/my-work
```

```
       A---B---C  feature/my-work
      /         \
 D---E---F---G---H  main  (merge commit H 생성)
```

- merge commit이 생겨서 "언제 합쳤는지" 이력이 남음
- 충돌 나면 수동으로 해결 후 `git add` → `git commit`

### git rebase — 브랜치 합치기 (이력 깔끔)

```bash
# feature 브랜치에서 main의 최신 변경사항 반영
git switch feature/my-work
git rebase main

# rebase 후 push (이력이 바뀌었으므로 force 필요)
git push --force-with-lease
```

```
 (before)        A---B---C  feature/my-work
                /
           D---E---F---G  main

 (after)                A'---B'---C'  feature/my-work
                       /
           D---E---F---G  main
```

| 구분 | merge | rebase |
|------|-------|--------|
| 이력 | 합친 기록(merge commit) 남음 | 일직선으로 깔끔 |
| 안전성 | 안전함 (이력 변경 없음) | push 후 rebase는 위험 ⚠️ |
| 사용 시점 | PR 머지할 때 | 작업 중 main 최신화할 때 |
| 팀 규칙 | 대부분 기본 | 팀 컨벤션 확인 필요 |

> ⚠️ **주의**: 이미 push한 커밋은 rebase 하지 않는 게 원칙 (다른 사람이 혼란)

### git cherry-pick — 특정 커밋만 가져오기

```bash
# 다른 브랜치의 특정 커밋 하나만 현재 브랜치에 적용
git cherry-pick abc1234

# 여러 커밋 가져오기
git cherry-pick abc1234 def5678

# 충돌 시
git cherry-pick --continue   # 충돌 해결 후 계속
git cherry-pick --abort      # 취소
```

- 예시: hotfix 브랜치의 버그 수정 커밋을 develop에도 적용하고 싶을 때

### git log — 커밋 이력 보기

```bash
git log                        # 전체 이력 (q로 나가기)
git log --oneline              # 한 줄씩 간단하게
git log --oneline --graph      # 브랜치 그래프까지
git log --oneline -10          # 최근 10개만
git log --author="이름"         # 특정 작성자 커밋만
```

### git diff — 변경 내용 비교

```bash
git diff                       # 작업 디렉토리 vs 스테이징 (add 전 변경사항)
git diff --staged              # 스테이징 vs 마지막 커밋 (add 후, commit 전)
git diff main feature/my-work  # 브랜치 간 비교
git diff HEAD~3                # 최근 3개 커밋과 비교
```

## 관련 노트
- [[Git - 기본 워크플로우 명령어]]
- [[Git - 되돌리기 & 충돌 해결 명령어]]
