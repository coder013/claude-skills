---
name: commit-month
description: 대상 월의 작업 내용을 Obsidian Monthly Review로 집계한다. /commit-month 로 트리거한다.
---

# Commit Month 스킬

대상 월의 Daily Note를 집계하여 Monthly Review를 작성한다.

**트리거**: `/commit-month`

---

## 실행 절차

### Step 1 - 데이터 수집 (Bash 1회)

```bash
TODAY_DAY=$(date +%d)
LAST_DAY=$(date -v+1m -v1d -v-1d +%d)
if [ "$TODAY_DAY" = "$LAST_DAY" ]; then
  TARGET=$(date +%Y-%m)
else
  TARGET=$(date -v-1m +%Y-%m)
fi
DAILY="/Users/jrlee/dev/repo/coder013/devlog/daily"
WORK_EMAIL="wnssh607@colosseum.kr"
PERSONAL_EMAIL="wnssh607@gmail.com"
AUTHOR_PATTERN="${WORK_EMAIL}|${PERSONAL_EMAIL}"

echo "=== TARGET: $TARGET ==="
echo "=== WORK LOGS ==="
for f in "$DAILY"/${TARGET}-*.md; do
  [ -f "$f" ] || continue
  echo "### $(basename $f .md)"
  sed -n '/#### 🛠️ Work Log/,/^---/p' "$f" | grep -v "^---" | grep -v "🛠️" | grep -v "^$"
done
echo "=== COMMITS ==="
git -C "/Users/jrlee/dev/repo/colosseum/colo_global" log --after="${TARGET}-01 00:00:00" --before="${TARGET}-${LAST_DAY} 23:59:59" --format="%h %s %ae" --all 2>/dev/null | grep -E "$AUTHOR_PATTERN" | sed 's/ [^ ]*@[^ ]*$//' | head -50
```

### Step 2 - 분석

**추가 파일 Read 금지.** Step 1 출력만으로 분석한다.

### Step 3 - Monthly Note 작성

- 경로: `/Users/jrlee/dev/repo/coder013/devlog/Monthly/{TARGET}.md`
- 파일 없으면 Write로 생성, 있으면 전체 교체

**성과 서술**: `[한 일] → [왜 중요한가]` 구조, 수치 포함 시 우선.
