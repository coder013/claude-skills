---
name: start-day
description: 하루를 시작하는 워크플로우. "start-day", "/start-day", "하루 시작", "출근", "오늘 시작" 요청 시 반드시 사용한다. brew/claude/git 업데이트 → Obsidian daily note 열기 → 어제 컨텍스트 자동 반영(어제 한일, 후속 과제) 순서로 실행한다.
---

# Start Day Workflow

하루를 시작하는 루틴이다. **두 단계**로 진행한다.

---

## Step 1: 개발 환경 업데이트

아래 명령어들을 순서대로 Bash 도구로 실행한다. 각 단계 완료 후 다음으로 진행한다.

### 1-1. Homebrew 업데이트

```bash
brew update && brew upgrade && brew cleanup
```

### 1-2. Claude CLI 업그레이드

```bash
if command -v claude >/dev/null 2>&1; then
  claude upgrade
else
  echo "Claude CLI not installed"
fi
```

### 1-3. 개인 레포 동기화

```bash
UNIVERSE="/Users/jrlee/dev/repo/coder013/devlog"

cd "$UNIVERSE" || { echo "⚠️ universe: directory not found"; exit 0; }

git fetch --prune

BRANCH=$(git branch --show-current)

# upstream 없으면 자동 설정 시도
if ! git rev-parse --abbrev-ref --symbolic-full-name @{u} >/dev/null 2>&1; then
  if git show-ref --verify --quiet "refs/remotes/origin/${BRANCH}"; then
    git branch --set-upstream-to="origin/${BRANCH}" "${BRANCH}"
  else
    echo "⚠️ universe: upstream not found. Skipping pull."
    exit 0
  fi
fi

git pull --ff-only && echo "✅ universe: updated" || echo "⚠️ universe: pull failed"
```

### 1-4. Obsidian Daily Note 열기

```bash
open "obsidian://daily?action=open"
```

각 단계의 결과를 간결하게 요약해서 보여준다.

---

## Step 2: 오늘 Daily Note 보강

Shell 작업이 완료되면 오늘 Daily Note에 어제의 컨텍스트를 반영한다.

### 2-1. 날짜 결정 (KST 기준)

```bash
DAILY_DIR="/Users/jrlee/dev/repo/coder013/devlog/daily"

# KST 기준 오늘 / 내일 / 요일
TODAY=$(TZ=Asia/Seoul date +%Y-%m-%d)
TOMORROW=$(TZ=Asia/Seoul date -v+1d +%Y-%m-%d)
DOW=$(TZ=Asia/Seoul date +%u)   # 1=월 2=화 ... 6=토 7=일

# KST 기준 지난 영업일 (주말이면 금요일로 소급)
# 월요일(1) → 3일 전(금), 일요일(7) → 2일 전(금), 나머지 → 1일 전
if [ "$DOW" = "1" ]; then
  PREV_WORKDAY=$(TZ=Asia/Seoul date -v-3d +%Y-%m-%d)
elif [ "$DOW" = "7" ]; then
  PREV_WORKDAY=$(TZ=Asia/Seoul date -v-2d +%Y-%m-%d)
else
  PREV_WORKDAY=$(TZ=Asia/Seoul date -v-1d +%Y-%m-%d)
fi

# 지난 영업일 파일 우선 탐색, 없으면 가장 최근 파일로 fallback
if [ -f "$DAILY_DIR/${PREV_WORKDAY}.md" ]; then
  YESTERDAY="$PREV_WORKDAY"
else
  YESTERDAY=$(ls "$DAILY_DIR"/*.md 2>/dev/null \
    | grep -v "Template" \
    | sed 's|.*/||; s|\.md||' \
    | sort \
    | awk -v today="$TODAY" '$0 < today' \
    | tail -1)
fi

echo "TODAY=$TODAY TOMORROW=$TOMORROW DOW=$DOW YESTERDAY=${YESTERDAY:-없음}"
```

### 2-2. 파일 읽기

Read 도구로 두 파일을 읽는다.

- 오늘 daily: `/Users/jrlee/dev/repo/coder013/devlog/daily/<TODAY>.md`
- 어제 daily: `/Users/jrlee/dev/repo/coder013/devlog/daily/<YESTERDAY>.md` — **YESTERDAY가 비어있으면 어제 파일 읽기를 건너뛰고 2-3~2-4도 생략한다.**

오늘 daily가 없으면 아래 형식으로 Write 도구로 생성한다. `TODAY`, `YESTERDAY`, `TOMORROW`는 2-1에서 구한 실제 값으로 치환한다:

```
 📅 <TODAY> (<요일>)

#### ✅ To Do

[]
-

[Meeting]
-

[ETC]
-

---
#### 🛠️ Work Log

-

---
#### 💡Notes

-

---
####  🧭 **Navigation**

 ⬅️ 이전 기록: [[<YESTERDAY>]]
 ➡️ 다음 기록: [[<TOMORROW>]]

---
```

### 2-3. 어제 daily에서 추출 (YESTERDAY가 있을 때만)

**추출 1 — 어제 TODO 전체**

`#### ✅ To Do` ~ 다음 `---` 직전까지의 내용을 그대로 가져온다. (빈 줄, 내부 구조 포함)

**추출 2 — 후속 과제** (우선순위 순으로 탐색)

1. Work Log의 `기술 부채 / 후속 과제` 블록에서 `🔲`로 시작하는 항목
2. 없으면 `진행 상황` 블록의 `다음:` 이후 내용
3. 없으면 어제 TODO에서 미완료 항목(`[ ]`로 시작하거나 체크되지 않은 것) 중 최대 3개

각 항목은 간결하게 1줄로 유지한다. 세 경우 모두 없으면 후속 과제 섹션을 생략한다.

### 2-4. 오늘 daily 수정

Edit 도구로 오늘 daily를 수정한다.

**수정 1 — Yesterday's Work 섹션 삽입**

`#### ✅ To Do` 바로 위에 삽입한다:

```
#### 📋 Yesterday's Work

<어제 TODO 섹션 내용 그대로>

---
```

**수정 2 — 후속 과제 추가**

오늘 daily의 `[ETC]` 블록 바로 위에 추가한다:

```
[Follow-up from Yesterday]
- 🔲 <후속 과제 1>
- 🔲 <후속 과제 2>

```

후속 과제가 없으면 생략한다.

**Obsidian 파일 충돌 대응**

Edit 도구가 `File has been modified since read` 오류로 실패하면, Obsidian이 파일을 동시에 수정 중인 것이다.
이 경우 아래 python3 스크립트로 원자적으로 쓴다:

```bash
python3 - <<'EOF'
import os, tempfile

filepath = "/Users/jrlee/dev/repo/coder013/devlog/daily/<TODAY>.md"

with open(filepath, "r", encoding="utf-8") as f:
    content = f.read()

# 수정 1: Yesterday's Work 삽입
insert_before = "#### ✅ To Do"
yesterday_block = """#### 📋 Yesterday's Work

<어제 TODO 내용>

---
"""
if "#### 📋 Yesterday's Work" not in content:
    content = content.replace(insert_before, yesterday_block + insert_before, 1)

# 수정 2: Follow-up 삽입
followup_block = """[Follow-up from Yesterday]
- 🔲 <후속 과제>

"""
if "[Follow-up from Yesterday]" not in content and "[ETC]" in content:
    content = content.replace("[ETC]", followup_block + "[ETC]", 1)

# 원자적 쓰기 (임시 파일 → rename)
dirpath = os.path.dirname(filepath)
with tempfile.NamedTemporaryFile("w", encoding="utf-8", dir=dirpath, delete=False) as tmp:
    tmp.write(content)
    tmp_path = tmp.name

os.replace(tmp_path, filepath)
print("✅ 파일 수정 완료")
EOF
```

`<TODAY>`, `<어제 TODO 내용>`, `<후속 과제>` 는 실제 값으로 치환 후 실행한다.

### 2-5. 완료 보고

- YESTERDAY 날짜 및 반영된 TODO 항목 수
- 삽입된 후속 과제 수 (출처: 기술부채 / 진행상황 / 미완료 TODO 중 어느 경로인지도 표기)
- 오늘 daily 파일 경로
- YESTERDAY가 없었으면 "이전 Daily Note를 찾지 못해 오늘 노트만 생성했습니다" 보고
