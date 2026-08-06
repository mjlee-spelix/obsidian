---
tags: [개발, Git]
date: 2026-04-23
---
# Git - 되돌리기 & 충돌 해결 명령어

## 핵심
- `reset`은 커밋을 **없었던 것처럼** 되돌림 (이력 삭제) — push 전에만 사용
- `revert`는 **취소 커밋을 새로 만들어서** 되돌림 (이력 보존) — push 후에도 안전
- `stash`는 작업 중인 변경사항을 **임시 저장** — 브랜치 이동할 때 필수
- merge conflict는 겁먹지 말고 **파일 열어서 직접 수정 → add → commit** 하면 됨

## 상세

### git stash — 임시 저장

작업 중인데 급하게 다른 브랜치로 가야 할 때

```bash
git stash                      # 현재 변경사항 임시 저장
git stash -m "작업 메모"         # 메모와 함께 저장
git stash list                 # 임시 저장 목록 보기
git stash pop                  # 가장 최근 stash 꺼내서 적용 + 삭제
git stash apply                # 꺼내서 적용 (삭제는 안 함)
git stash drop                 # 가장 최근 stash 삭제
git stash clear                # 전부 삭제
```

**흔한 사용 패턴:**
```bash
# 작업 중에 급하게 main 확인해야 할 때
git stash -m "기능 개발 중"    # 1. 임시 저장
git switch main               # 2. main으로 이동
# (확인 작업)
git switch feature/my-work     # 3. 다시 돌아와서
git stash pop                  # 4. 임시 저장 복원
```

### git reset — 커밋 되돌리기 (이력 삭제)

```bash
git reset --soft HEAD~1    # 커밋만 취소 (변경사항은 staged 상태로 유지)
git reset --mixed HEAD~1   # 커밋 + add 취소 (변경사항은 작업 디렉토리에 유지) ← 기본값
git reset --hard HEAD~1    # 커밋 + 변경사항 전부 삭제 ⚠️ 복구 어려움
```

| 모드 | 커밋 | 스테이징 | 작업 파일 | 용도 |
|------|-----|---------|----------|------|
| `--soft` | ❌ 취소 | ✅ 유지 | ✅ 유지 | 커밋 메시지만 바꾸고 싶을 때 |
| `--mixed` | ❌ 취소 | ❌ 취소 | ✅ 유지 | 커밋은 취소하되 코드는 남기고 싶을 때 |
| `--hard` | ❌ 취소 | ❌ 취소 | ❌ 삭제 | 전부 없었던 걸로 (위험!) |

```bash
# 스테이징 취소 (add 취소)
git reset HEAD 파일명.txt       # 특정 파일 unstage
git reset HEAD                 # 전체 unstage
```

> ⚠️ **중요**: `reset`은 push 전에만 사용! push 후에는 다른 사람에게 영향을 줌

### git revert — 커밋 취소 (이력 보존)

```bash
git revert abc1234             # 해당 커밋을 취소하는 새 커밋 생성
git revert HEAD                # 직전 커밋 취소
git revert HEAD~3..HEAD        # 최근 3개 커밋 취소
git revert --no-commit abc1234 # 취소는 하되 커밋은 안 만듦 (확인 후 직접 커밋)
```

```
 (before) A---B---C---D
 (after)  A---B---C---D---D'  ← D를 취소하는 커밋 D' 추가
```

### reset vs revert 비교

| 구분 | reset | revert |
|------|-------|--------|
| 이력 | 삭제됨 | 보존됨 (취소 커밋 추가) |
| push 후 사용 | ❌ 위험 | ✅ 안전 |
| 되돌리기 범위 | 해당 커밋 이후 전부 | 특정 커밋만 골라서 |
| 용도 | 로컬에서 실수 수정 | 이미 공유된 커밋 되돌리기 |

### .gitignore — 추적 제외

프로젝트 루트에 `.gitignore` 파일 생성:

```gitignore
# 환경 설정
.env
.env.local

# 의존성
node_modules/
__pycache__/
venv/

# 빌드 결과물
dist/
build/
*.pyc

# IDE 설정
.vscode/
.idea/

# OS 파일
.DS_Store
Thumbs.db

# 로그
*.log
logs/
```

```bash
# 이미 추적 중인 파일을 무시하려면
git rm --cached 파일명          # 추적 중지 (파일은 삭제 안 됨)
git rm --cached -r 폴더명/      # 폴더 전체 추적 중지
```

### Merge Conflict 해결

충돌이 나면 파일에 이런 표시가 생김:

```
<<<<<<< HEAD
내가 수정한 코드
=======
상대방이 수정한 코드
>>>>>>> feature/other-work
```

**해결 순서:**
```bash
# 1. 충돌 파일 확인
git status                    # "both modified" 표시된 파일

# 2. 파일 열어서 직접 수정
#    <<<, ===, >>> 마커를 삭제하고 최종 코드만 남기기

# 3. 해결 완료 후
git add 해결한파일.txt
git commit -m "merge: conflict 해결"
```

**충돌 해결 포기:**
```bash
git merge --abort              # merge 충돌 시 취소
git rebase --abort             # rebase 충돌 시 취소
git cherry-pick --abort        # cherry-pick 충돌 시 취소
```

### HEAD, origin, upstream 개념

| 용어 | 의미 | 예시 |
|------|------|------|
| `HEAD` | 현재 내가 보고 있는 커밋 (보통 현재 브랜치의 최신 커밋) | `HEAD~1` = 1개 전 커밋 |
| `origin` | 내가 clone한 원격 저장소 (기본 이름) | `origin/main` = 원격의 main |
| `upstream` | fork한 경우 원본 저장소 | 오픈소스 기여 시 사용 |
| `HEAD~n` | n개 전 커밋 | `HEAD~3` = 3개 전 |
| `HEAD^` | 부모 커밋 (merge에서 구분 시) | `HEAD~1`과 보통 같음 |

## 관련 노트
- [[Git - 기본 워크플로우 명령어]]
- [[Git - 브랜치 전략 명령어]]
