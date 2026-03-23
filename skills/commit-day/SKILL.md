---
name: commit-day
description: 오늘 작업한 내용을 Obsidian Daily Note에 기록한다. /commit-day 로 트리거한다.
---

# Commit Day 스킬

오늘 작업한 내용을 Obsidian Daily Note에 기록한다.
**목적**: Daily → Monthly → Yearly 집계 → 이력서 및 성과평가 자료.

**트리거**: `/commit-day`

---

## 실행 절차

### Step 1 - 데이터 수집 (Bash 1회)

```bash
TODAY=$(date +%Y-%m-%d)
WORK_EMAIL="wnssh607@colosseum.kr"
PERSONAL_EMAIL="wnssh607@gmail.com"
AUTHOR_PATTERN="${WORK_EMAIL}|${PERSONAL_EMAIL}"

echo "=== BRANCH ===" && git branch --show-current
echo "=== COMMITS ===" && git log --since="$TODAY 00:00:00" --format="%h %s %ae" --all | grep -E "$AUTHOR_PATTERN" | sed 's/ [^ ]*@[^ ]*$//'
echo "=== UNCOMMITTED ===" && git status --short
echo "=== UNIVERSE CHANGES ===" && git -C "/Users/jrlee/dev/repo/coder013/devlog" log --since="$TODAY 00:00:00" --author="$PERSONAL_EMAIL" --format="%h %s" --all
echo "=== UNIVERSE UNCOMMITTED ===" && git -C "/Users/jrlee/dev/repo/coder013/devlog" status --short
```

### Step 2 - 컨텍스트 분석

**파일 Read 금지.** 세션 대화 + Step 1 결과만으로 작업 내용을 파악한다.

- 업무 레포 커밋: 업무 계정 것만 집계
- 개인 레포 커밋: 개인 계정 것만 집계
- 개인 레포 변경사항은 **[개인 환경 작업]** 항목으로 별도 기록

### Step 3 - Daily Note 작성

파일 경로: `/Users/jrlee/dev/repo/coder013/devlog/daily/$TODAY.md`

1. Grep으로 `#### 🛠️ Work Log` 섹션 존재 여부만 확인
2. 기존 내용이 있으면 합산하여 하나의 Summary로 작성 (중복 제거)
3. 섹션이 있으면 Edit으로 해당 섹션만 교체, 없으면 파일 끝에 추가
4. 파일 자체가 없으면 Write로 생성

**출력 형식**:

```markdown
### [YYYY-MM-DD] Daily Summary

**[커밋]**
- `abc1234`: feat: 작업 내용

**[주요 작업 및 성과]**
- ✅ [한 일] → [왜 중요한가] 구조로 서술

**[기술 부채 / 후속 과제]**
- 🔲 다음에 해야 할 것

**[진행 상황]**
- CSP-XXXX: 현재 상태 / 다음 단계
```
