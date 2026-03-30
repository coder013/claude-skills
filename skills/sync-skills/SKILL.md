---
name: sync-skills
description: claude-skills 레포의 스킬들을 Claude 플러그인 디렉토리에 동기화한다. "sync-skills", "/sync-skills", "스킬 동기화", "스킬 등록" 요청 시 반드시 사용한다.
---

# Sync Skills Workflow

`/Users/jrlee/dev/repo/coder013/claude-skills/skills/` 에 작성된 스킬들을
`/Users/jrlee/.claude/plugins/local/plugins/jrlee-daily/skills/` 로 복사한다.

---

## Step 1: 현재 상태 확인

```bash
SRC="/Users/jrlee/dev/repo/coder013/claude-skills/skills"
DST="/Users/jrlee/.claude/plugins/local/plugins/jrlee-daily/skills"

echo "=== Source skills ==="
ls "$SRC"

echo ""
echo "=== Destination skills ==="
ls "$DST"
```

## Step 2: 동기화

`rsync`로 source → destination 전체 복사한다. source에서 삭제된 스킬은 destination에서도 제거된다.

```bash
SRC="/Users/jrlee/dev/repo/coder013/claude-skills/skills"
DST="/Users/jrlee/.claude/plugins/local/plugins/jrlee-daily/skills"

rsync -av --delete "$SRC/" "$DST/"
```

## Step 3: 결과 보고

동기화된 스킬 목록과 변경 내역(추가/수정/삭제)을 간결하게 출력한다.
