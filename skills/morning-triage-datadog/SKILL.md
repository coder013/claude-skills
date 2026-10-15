---
name: morning-triage-datadog
description: 지난 24시간 Datadog 에러 로그를 분류·요약해 Slack 채널로 모닝 리포트를 전송한다. "/morning-triage-datadog", "모닝 트리아지", "오늘 아침 에러", cron 자동 호출 시 사용한다.
---

# morning-triage-datadog

지난 24시간 동안 누적된 Datadog 에러 로그를 한 번에 훑어 분류하고, 출근 직전 Slack에 요약 리포트를 보낸다. 매일 아침 Datadog UI를 직접 뒤지는 시간을 0으로 만드는 게 목적.

**전제조건**:
- Datadog MCP 서버가 현재 세션에 등록되어 있어야 함
- 환경변수 `SLACK_TRIAGE_WEBHOOK`에 Slack incoming webhook URL이 있어야 함

---

## 실행 절차

### Step 1 - 환경변수 검증

Bash로 `echo $SLACK_TRIAGE_WEBHOOK` 실행. 비어있으면 사용자에게 알리고 중단.

### Step 1b - Datadog MCP 가용성 검증 (실패 시 Slack 알림)

**조용한 실패를 시끄러운 알림으로 바꾸는 단계.** headless(`claude -p`) 실행에서는 인터랙티브 OAuth 갱신이 불가능해, 토큰이 lapsed면 Datadog MCP 도구가 세션에 아예 로드되지 않고 triage가 조용히 멈춘다(아무 Slack도 안 옴). 이를 방지한다.

가벼운 프로브 1건으로 가용성을 확인: `search_datadog_logs`로 `status:error`, `from=now-15m`, `max_tokens=300` 검색.

다음 중 하나라도 해당하면 **정상 리포트를 만들지 말고**, 아래 실패 알림을 Step 6의 curl 메커니즘으로 Slack에 전송한 뒤 즉시 종료한다:
- Datadog MCP 도구(`search_datadog_logs` 등)가 세션에 등록되어 있지 않음
- 프로브가 auth / permission / connection / `Not logged in` 류 에러를 반환

실패 알림 메시지(한 줄, 명확하게):
```
⚠️ *Datadog Morning Triage 실패* — <YYYY-MM-DD (요일) HH:MM KST>
Datadog MCP에 연결하지 못해 triage를 건너뛰었습니다. 토큰이 lapsed 됐을 가능성이 높습니다.
복구: 인터랙티브 세션에서 `cd ~/dev/repo/datadog-mcp-poc && claude-colosseum` → `/mcp` → datadog 재인증 → `launchctl kickstart -p gui/$(id -u)/com.jrlee.morning-triage` 로 재실행.
```

프로브가 정상(에러 여부와 무관하게 결과/카운트 반환)이면 Step 2로 진행.

### Step 2 - 시간 윈도우 설정

KST 기준 **어제 09:00 ~ 오늘 09:00** (= 직전 24시간). cron이 9시에 돌든 사용자가 수동 호출하든 같은 윈도우를 쓰도록 "now - 24h" 대신 "지난 영업일 09시" 기준으로 잡는다.

`date` 명령으로 현재 시각 확인 후 윈도우 계산.

### Step 3 - Datadog 검색

**조회 대상 service 6개로 한정** (그 외 service는 무시):

| 리전 | 별칭 | Datadog `service` |
|---|---|---|
| 🇺🇸 미국 | `app` | `prod-us-ecs-cgs-app` |
| 🇺🇸 미국 | `oms` | `prod-us-ecs-cgs-oms` |
| 🇺🇸 미국 | `batch` | `prod-us-ecs-cgs-batch` |
| 🇸🇬 싱가폴 | `app` | `prod-sg-ecs-cgs-app` |
| 🇸🇬 싱가폴 | `js-app` | `prod-sps1-cgs-js-app` |
| 🇯🇵 일본 | `app` | `prod-apn1-cgs-jp-app` |

Datadog 검색 쿼리:
```
status:error service:(prod-us-ecs-cgs-app OR prod-us-ecs-cgs-oms OR prod-us-ecs-cgs-batch OR prod-sg-ecs-cgs-app OR prod-sps1-cgs-js-app OR prod-apn1-cgs-jp-app)
```

다음 집계까지 가능하면 한 번에:
- service별 카운트
- exception class / 에러 메시지 패턴별 카운트 (동적 토큰 정규화 후)
- 시간대별(시간 단위) 분포

**주의**: 로그가 매우 많으면 (1만 건+) 우선 집계 결과만 가져오고, 본문은 Top N(20개)만 샘플로 확인.

### Step 4 - 분류

다음 휴리스틱으로 그룹화. 애매하면 **Actionable로 분류**(보수적).

**🔴 Actionable** (코드 수정 후보):
- NullPointerException, IllegalStateException, IndexOutOfBoundsException, ClassCastException
- StaleObjectStateException, OptimisticLockException, ObjectOptimisticLockingFailureException
- SQLIntegrityConstraintViolationException (Duplicate entry 등)
- IllegalArgumentException 중 비즈니스 로직 위반으로 보이는 것

**🟡 Infrastructure** (인프라/외부 의존성):
- ConnectException, SocketTimeoutException, TimeoutException, ReadTimeoutException
- 5xx 응답·Gateway timeout 관련
- Redis/RabbitMQ/Kafka/외부 API 호출 실패
- Container OOM, deploy/rolling restart, health check 관련

**🟢 Noise** (저절로 해소되거나 알려진 일시적 문제):
- Health check 단발 실패 (회복됨)
- Known rate limit 정상 동작
- 만료 토큰/세션의 정상 실패
- 1회만 발생하고 회복된 일시적 에러

### Step 5 - Slack 메시지 빌드

Slack mrkdwn 호환 형식. **리전 → service → 카테고리** 3-레벨 그룹화.

**리전 순서 고정**: 🇺🇸 미국 → 🇸🇬 싱가폴 → 🇯🇵 일본 (지리적 일관성).

**Service 표시는 별칭과 풀네임 둘 다** — `*app*` `prod-us-ecs-cgs-app` 형태.

**정렬·생략 규칙**:
- 리전 내 service: Actionable 보유 우선 → 그 다음 총 건수 내림차순
- 카테고리 내 항목: 건수 내림차순
- 0건인 카테고리 / 0건인 service / 0건인 리전 모두 표시 생략
- 6개 service 모두 0건이면 `🎉 *지난 24시간 신규 에러 없음*` 한 줄로 종료

**형식 예시**:

```
🌅 *Datadog Morning Triage*
_2026-05-20 (수) · 지난 24h_

📊 *총 N건* · 활성 service M / 6


🇺🇸 *미국* — N건
━━━━━━━━━━━━━━━━━━━━━━━━
*app* `prod-us-ecs-cgs-app` — N건
   🔴 *Actionable* · M건
      • `예외클래스` — N건 — `엔드포인트` — 첫 HH:MM
      • `예외클래스` — N건 — `엔드포인트` — 첫 HH:MM
   🟡 *Infrastructure* · M건
      • `예외클래스` — N건
   🟢 *Noise* · N건

*oms* `prod-us-ecs-cgs-oms` — N건
   🔴 *Actionable* · M건
      • ...


🇸🇬 *싱가폴* — N건
━━━━━━━━━━━━━━━━━━━━━━━━
*app* `prod-sg-ecs-cgs-app` — N건
   ...


🇯🇵 *일본* — N건
━━━━━━━━━━━━━━━━━━━━━━━━
*app* `prod-apn1-cgs-jp-app` — N건
   ...


🔗 <Datadog UI 링크|24h 전체 보기>
```

**길이 축약 정책** (Slack 4000자 초과 시 단계적으로):
1. Service당 카테고리별 Top 5만 → 초과는 `_… (+N건 더)_`
2. Noise는 service당 건수만 표시(항목 생략)
3. 그래도 초과하면 Infrastructure도 Top 3으로 축소

### Step 6 - Slack 전송

Bash로 curl 호출:

```bash
curl -X POST -H 'Content-type: application/json' \
  --data "$(cat <<'EOF'
{"text": "<여기에 빌드한 메시지>"}
EOF
)" \
  "$SLACK_TRIAGE_WEBHOOK"
```

응답이 `ok` 가 아니면 에러 로그하고 stderr로 알림.

### Step 7 - 결과 출력

수동 호출 시 화면에도 동일 리포트를 출력해 사용자가 바로 확인할 수 있게 한다. cron 호출 시엔 stdout으로 로그만 남기면 됨.

---

## 출력 톤

- 한국어 (단, 예외 클래스명·service명 등 식별자는 원문 그대로)
- 보수적 분류 (애매하면 Actionable)
- 단정 없이 "~으로 보입니다" 톤
- 발견사항이 0건이면 "🎉 지난 24시간 신규 에러 없음" 한 줄로 마무리
