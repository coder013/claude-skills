---
name: commit-year
description: 대상 연도의 작업 내용을 Obsidian Yearly Review로 집계한다. /commit-year 로 트리거한다.
---

# Commit Year 스킬

대상 연도의 Monthly Note(없으면 Daily)를 집계하여 Yearly Review를 작성한다.

**트리거**: `/commit-year`

---

## 실행 절차

### Step 1 - 데이터 수집 (Bash 1회)

```bash
TODAY=$(date +%m-%d)
if [ "$TODAY" = "12-31" ]; then
  TARGET_YEAR=$(date +%Y)
else
  TARGET_YEAR=$(date -v-1y +%Y)
fi
MONTHLY="/Users/jrlee/dev/repo/coder013/devlog/Monthly"
DAILY="/Users/jrlee/dev/repo/coder013/devlog/daily"

echo "=== TARGET YEAR: $TARGET_YEAR ==="
echo "=== MONTHLY NOTES ==="
MONTHLY_COUNT=0
for f in "$MONTHLY"/${TARGET_YEAR}-*.md; do
  [ -f "$f" ] || continue
  MONTHLY_COUNT=$((MONTHLY_COUNT+1))
  echo "### $(basename $f .md)"
  cat "$f"
done

if [ "$MONTHLY_COUNT" -eq 0 ]; then
  echo "=== MONTHLY NOT FOUND — FALLBACK TO DAILY ==="
  for f in "$DAILY"/${TARGET_YEAR}-*.md; do
    [ -f "$f" ] || continue
    echo "### $(basename $f .md)"
    sed -n '/#### 🛠️ Work Log/,/^---/p' "$f" | grep -v "^---" | grep -v "🛠️" | grep -v "^$"
  done
fi

echo "=== WORK DAY COUNT ==="
ls "$DAILY"/${TARGET_YEAR}-*.md 2>/dev/null | wc -l
```

### Step 2 - 분석

**추가 파일 Read 금지.** Step 1 출력만으로 분석한다.
Top 성과 3~5개는 비즈니스 임팩트 > 기술적 난이도 순으로 선별한다.

### Step 3 - Yearly Note 작성

- 경로: `/Users/jrlee/dev/repo/coder013/devlog/Yearly/{TARGET_YEAR}.md`
- 파일 없으면 Write로 생성, 있으면 전체 교체

**성과 서술**: `[한 일] → [왜 중요한가]` 구조, 수치 포함 시 우선.
