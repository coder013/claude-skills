---
name: end-day
description: 하루를 마무리하는 워크플로우. "end-day", "/end-day", "퇴근", "하루 마무리", "오늘 끝" 요청 시 반드시 사용한다. JaykingUniverse 레포의 변경사항을 자동으로 커밋 & 푸시한다.
---

# End Day Workflow

하루를 마무리하는 루틴이다.

---

## Step 1: JaykingUniverse 커밋 & 푸시

아래 로직을 Bash 도구로 실행한다.

```bash
UNIVERSE="/Users/jrlee/dev/repo/coder013/devlog"

cd "$UNIVERSE" || { echo "⚠️ universe: directory not found"; exit 1; }

BRANCH=$(git branch --show-current)
if [ -z "$BRANCH" ]; then
  echo "⚠️ universe: cannot determine current branch"
  exit 0
fi

# upstream 없으면 자동 설정 시도
if ! git rev-parse --abbrev-ref --symbolic-full-name @{u} >/dev/null 2>&1; then
  if git show-ref --verify --quiet "refs/remotes/origin/${BRANCH}"; then
    git branch --set-upstream-to="origin/${BRANCH}" "${BRANCH}"
  else
    echo "⚠️ universe: no upstream and origin/${BRANCH} not found. Skipping push."
    exit 0
  fi
fi

# 원격 최신화
echo "🔄 Fetching remote..."
git fetch --prune

# ahead/behind 계산
read -r AHEAD BEHIND < <(git rev-list --left-right --count HEAD...@{u} | awk '{print $1, $2}')
AHEAD=${AHEAD:-0}
BEHIND=${BEHIND:-0}

# 원격이 앞서 있으면 rebase pull 먼저
if [ "$BEHIND" -gt 0 ]; then
  echo "🔽 Remote has ${BEHIND} newer commit(s). Pulling with rebase..."
  git pull --rebase --autostash || { echo "❌ git pull --rebase failed. Resolve manually."; exit 0; }
fi

# 변경 감지
if [ -z "$(git status --porcelain)" ]; then
  echo "✅ universe: no changes"
  exit 0
fi

# 스테이징
echo "🧾 Changes detected. Staging..."
git add -A

if [ -z "$(git diff --cached --name-only)" ]; then
  echo "✅ universe: nothing staged. Skipping."
  exit 0
fi

# 커밋
TODAY_KST=$(TZ=Asia/Seoul date +%F)
COMMIT_MSG="Universe updated ${TODAY_KST} KST"

echo "📝 Committing: ${COMMIT_MSG}"
git -c user.email='wnssh607@gmail.com' commit -m "${COMMIT_MSG}"

# 푸시
echo "🚀 Pushing..."
git push && echo "✅ universe: pushed" || echo "❌ universe: push failed"
```

---

## Step 2: 완료 보고

결과를 간결하게 요약한다:

- 커밋 메시지 및 push 성공 여부
- 변경사항 없어 스킵된 경우
- 오류 내용 (있을 경우)
