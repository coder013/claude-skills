---
name: sync-skills
description: claude-skills 레포의 스킬들을 Claude 플러그인 디렉토리에 동기화한다. "sync-skills", "/sync-skills", "스킬 동기화", "스킬 등록" 요청 시 반드시 사용한다.
---

# Sync Skills Workflow

`claude-skills` 레포의 스킬들을 모든 Claude config 디렉토리(`~/.claude*`)에 동기화한다.
경로는 환경에서 자동 추론한다 — 하드코딩 없음.

---

## Step 1: 경로 추론 및 현재 상태 확인

```bash
# Source: $HOME 하위에서 claude-skills 레포 탐색
SRC=$(find "$HOME" -maxdepth 6 -type d -name "claude-skills" 2>/dev/null | head -1)/skills

echo "=== Source ==="
echo "$SRC"
ls "$SRC"

echo ""
echo "=== Target Claude config dirs ==="
find "$HOME" -maxdepth 1 -type d -name ".claude*" 2>/dev/null | sort
```

## Step 2: 동기화

각 Claude config 디렉토리에 대해 `local/plugins/jrlee-daily/skills/` 경로를 생성하고 rsync한다.
source에서 삭제된 스킬은 destination에서도 제거된다.
`plugin.json`이 없는 디렉토리에는 자동으로 생성한다.

```bash
SRC=$(find "$HOME" -maxdepth 6 -type d -name "claude-skills" 2>/dev/null | head -1)/skills
PLUGIN_JSON_SRC="$HOME/.claude/plugins/local/plugins/jrlee-daily/.claude-plugin/plugin.json"

# zsh에서 find 결과를 for 루프에 쓰면 한 덩어리로 처리되므로 while read 사용
while IFS= read -r CLAUDE_DIR; do
    DST="$CLAUDE_DIR/plugins/local/plugins/jrlee-daily/skills"
    echo ""
    echo ">>> Syncing to: $DST"
    mkdir -p "$DST"
    rsync -av --delete "$SRC/" "$DST/"

    # plugin.json 없으면 복사
    PLUGIN_JSON_DST="$CLAUDE_DIR/plugins/local/plugins/jrlee-daily/.claude-plugin/plugin.json"
    if [ ! -f "$PLUGIN_JSON_DST" ]; then
        mkdir -p "$(dirname "$PLUGIN_JSON_DST")"
        cp "$PLUGIN_JSON_SRC" "$PLUGIN_JSON_DST"
        echo ">>> Copied plugin.json to: $PLUGIN_JSON_DST"
    fi
done < <(find "$HOME" -maxdepth 1 -type d -name ".claude*" 2>/dev/null | sort)
```

## Step 3: 결과 보고

동기화된 Claude config 디렉토리 목록, 스킬 목록, 변경 내역(추가/수정/삭제)을 간결하게 출력한다.
