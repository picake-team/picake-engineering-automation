---
name: picake-data-report
description: 사용자 데이터, 전환율, PostHog 지표, 회원수 등 Picake의 종합 데이터를 보여달라는 요청을 받으면 사용합니다. "사용자 데이터 보여줘", "전환율 어때", "이번 주 지표 알려줘", "회원수 몇 명이야" 같은 요청에 이 skill을 참고해 PostHog(행동 데이터)와 관리자 통계(DB 데이터)를 함께 조회해 리포트를 보여주세요.
---

# 종합 데이터 리포트

> **이 문서의 `apps/web-*` 경로는 [`picake-engineering-web`](https://github.com/picake-team/picake-engineering-web), `apps/backend/...` 경로는 [`picake-engineering-backend`](https://github.com/picake-team/picake-engineering-backend) 레포의 것입니다.** 로컬에서는 형제 폴더(`../picake-engineering-web`, `../picake-engineering-backend`)에서 확인하세요. 매일 09:00 KST 클라우드 루틴("Picake 일일 데이터 리포트")이 이 레포를 체크아웃해 4번 **자동 실행 모드**로 실행합니다.

Picake의 데이터는 두 곳에 나뉘어 있습니다. **행동 데이터(페이지뷰·클릭·전환 퍼널)는 PostHog**, **DB가 원본인 지표(회원수·주문·매출·스토어)는 백엔드 관리자 통계 API**입니다 (`apps/backend/src/apis/admin/controllers/statistics.controller.ts` 주석에 명시된 설계). 이 skill은 둘을 한 번에 모아 예쁜 대시보드로 보여주는 절차입니다.

## 0. 범위 확인

사용자 요청에서 아래가 불명확하면 짧게 확인하되, 굳이 다 묻지 말고 합리적 기본값으로 진행합니다.

- **환경**: `production`(`https://api.picakes.com`) 또는 `staging`(`https://api-staging.picakes.com`). 언급 없으면 production 기준으로 진행. 사용자가 명확히 밝히지 않으면 두 환경 중 어디를 보여줄지, DB 통계는 관리자 토큰이 필요하다는 점을 먼저 물어봐도 됩니다.
  - **중요**: 여기서 고른 환경은 DB API의 base URL뿐 아니라 아래 1번의 PostHog `environment` 속성 필터에도 그대로 써야 합니다. 두 값이 어긋나면(예: production DB + 필터 없는 PostHog) 서로 다른 환경 데이터를 비교하는 리포트가 됩니다.
  - **출력 범위**: 사용자가 특정 환경(예: production)을 요청했다면 최종 리포트(채팅 요약 + 대시보드 Artifact 모두)는 그 환경만 다룹니다. 다른 환경의 이벤트 수·페이지뷰 수 등 구체적 수치나 이름(staging/development)을 리포트에 언급하지 마세요 — 원인 조사 과정에서 다른 환경 데이터를 내부적으로 대조해봤더라도, 사용자가 명시적으로 여러 환경 비교를 요청한 게 아니라면 결과물에는 넣지 않습니다.
- **기간·시간대 (모든 숫자의 기준, 가장 먼저 정하고 리포트에 적습니다)**: **한국 시간(KST), 어제까지 완료된 날짜**로 통일합니다. 오늘은 아직 끝나지 않았으므로 제외합니다(자동 실행은 09시라 오늘은 9시간뿐입니다). 실행 환경은 리눅스입니다.

  ```bash
  export TZ=Asia/Seoul
  TODAY=$(date +%F)                     # 실행일
  D1=$(date -d "$TODAY -1 day" +%F)     # 어제 (집계에 포함되는 마지막 날)
  D7=$(date -d "$TODAY -7 day" +%F)     # 최근 7일 시작일
  D30=$(date -d "$TODAY -30 day" +%F)   # 최근 30일 시작일
  NOW_KST=$(date '+%Y-%m-%d %H:%M')     # 점검 시각
  ```

  | 구간 | 날짜 범위 (KST 00:00 경계, 시작 포함·끝 제외) |
  | --- | --- |
  | 어제 | `D1` ~ `TODAY` (1일) |
  | 최근 7일 | `D7` ~ `TODAY` (완료된 7일) |
  | 최근 30일 | `D30` ~ `TODAY` (완료된 30일) |

  - **PostHog**: 모든 기간 조건은 `toDateTime('YYYY-MM-DD 00:00:00', 'Asia/Seoul')`로 넣습니다. 일별 집계는 `toDate(toTimeZone(timestamp, 'Asia/Seoul'))`로 묶습니다. **PostHog 프로젝트 시간대는 UTC라서 기본 `toDate(timestamp)`는 UTC 날짜(= KST 09:00 경계)입니다. 그대로 쓰면 "10/25"가 KST 10/25 09:00 ~ 10/26 09:00이 되어 다른 리포트와 하루가 9시간 어긋납니다.** 또 `now() - INTERVAL 1 DAY` 같은 **롤링 구간은 쓰지 않습니다**(실행 시각에 따라 값이 달라집니다).
  - **활성 사용자**: DAU = 어제 고유 방문자, WAU = 최근 7일 고유 방문자, MAU = 최근 30일 고유 방문자(위 구간에서 `uniq(distinct_id)`).
  - **DB 신규 건수**(가입·스토어·입점요청·주문): **`daily-trends`(Asia/Seoul)로 구간을 직접 합산**합니다. `startDate=D30&endDate=D1`로 한 번 받아 어제 / 최근 7일 / 최근 30일을 계산하세요. 통계 API의 `today`·`last7Days`·`last30Days` 필드는 **오늘(부분)을 포함한 서울 달력일 구간**(`last7Days` = 오늘 포함 7일)이라 위 구간과 맞지 않고 09시에는 오늘이 9시간뿐이라 쓰지 않습니다. 누적(`total`, `withdrawn`, `gmv`, `byStatus`)은 그대로 씁니다.
  - **"오늘"이라는 표현은 리포트에 쓰지 않습니다.** "어제(MM/DD)"를 씁니다.
  - 리포트 맨 위(Artifact 헤더·Discord 필드)에 **집계 기준**을 반드시 적습니다: `한국 시간(KST) · 어제 MM/DD까지 완료된 일자 · 점검 시각 {NOW_KST}`. 이번 7일/30일의 날짜 범위(`MM/DD ~ MM/DD`)도 함께 적습니다.
- 특정 카테고리만 요청한 경우(예: "회원가입 전환율만") 해당 섹션만 조회해도 됩니다.

## 1. PostHog 데이터 수집

이 레포의 `.mcp.json`에 `posthog` MCP가 연결되어 있습니다(루틴은 PostHog 커넥터 사용, project id `497749`, org "picake"). `mcp__posthog__exec` 도구로 아래를 조회하세요.

### 환경 필터링 (필수, 빠뜨리면 리포트가 틀립니다)

PostHog 프로젝트는 **development/staging/production이 하나로 공용**입니다 (`apps/web-user/.env.development`의 `NEXT_PUBLIC_POSTHOG_KEY` 주석에 "공용"이라고 명시, `NEXT_PUBLIC_POSTHOG_HOST`도 동일). 대신 `apps/web-user/src/common/config/posthog.config.ts`가 모든 이벤트에 super property `environment`(값: `production` / `staging` / `development`, `NEXT_PUBLIC_NODE_ENV`에서 옴)를 자동으로 붙이고 있어 이걸로 구분하도록 설계되어 있습니다.

**게다가 이 PostHog 프로젝트(497749)는 `web-user`뿐 아니라 `web-seller`도 같이 씁니다** (`apps/web-seller/src/common/config/posthog.config.ts`도 동일한 방식으로 super property를 등록, `app: "web-seller"`). 그래서 `environment` 필터만으로는 부족하고 **`app` 필터도 항상 같이 걸어야 합니다** — 안 그러면 web-seller 트래픽이 web-user 리포트에 섞여 들어갈 수 있습니다 (2026-08-03 기준 web-seller는 아직 development 트래픽뿐이라 지금 당장은 영향 없지만, 실사용이 시작되면 바로 문제가 됩니다). `web-admin`은 PostHog 연동 자체가 없습니다(내부 툴이라 의도적으로 제외된 것으로 보임).

**따라서 이 skill의 모든 PostHog 쿼리(query-trends, query-funnel, execute-sql 등)에는 0번에서 정한 환경으로 `properties.environment` 필터와 **`properties.app` 필터(리포트 대상이 web-user면 `"web-user"`)**를 반드시 같이 걸어야 합니다.** 필터를 빼먹으면 로컬/스테이징 테스트 트래픽이나 다른 앱(web-seller)의 트래픽이 production 숫자에 섞여 들어가고, 특히 표본이 작을 때 심하게 왜곡됩니다.

- `query-trends`/`query-funnel`: 최상위 `properties` 배열에 `{"key": "environment", "operator": "exact", "type": "event", "value": ["production"]}`와 `{"key": "app", "operator": "exact", "type": "event", "value": ["web-user"]}`를 함께 추가.
- `execute-sql`(HogQL): `WHERE properties.environment = 'production' AND properties.app = 'web-user'`를 다른 조건과 함께 걸기.
- 값이 실제로 뭐가 있는지 헷갈리면 `read-data-schema`(`kind: event_property_values`, `event_name: $pageview`, `property_name: environment` 또는 `app`)로 먼저 확인하세요.
- 기존 저장된 Insight/Dashboard(1번의 "이미 PostHog에 저장된 Insight" 부분)는 이 필터가 안 걸려 있을 수 있으니, 그대로 인용하지 말고 숫자를 재확인하세요.
- 리포트를 시작하기 전에 한 번씩 `SELECT properties.app, properties.environment, count() FROM events WHERE timestamp >= now() - INTERVAL 30 DAY GROUP BY 1, 2`로 전체 분포를 훑어보면, 새로운 앱/환경이 추가되거나 태깅이 누락된 이벤트가 있는지 빠르게 감지할 수 있습니다.

### 합성(테스트) 접속 제외 (필수, 환경 필터와 같이 항상 건다)

**봇으로 표시되지 않는 합성 접속이 실제 방문자처럼 집계됩니다.** 실제로 2026-10-26의 방문자(고유 사용자)가 73명으로 보였는데, 그중 61명이 합성 접속이었고 실제는 약 12명이었습니다. 원인은 두 가지입니다.

- **웹 성능 점검 루틴의 Lighthouse(PageSpeed Insights)**: 구글 서버에서 `moto g power (2022)`라는 가상 Android 기기로 접속하며, 실행할 때마다 페이지를 여러 번 새로 열어 매번 "새 방문자"가 됩니다.
- **개발자의 모바일 에뮬레이션·자동화 테스트(Playwright 등)**: 폭 500px 이하의 **데스크톱** 브라우저로 나타납니다. 실제 데스크톱 사용자의 창은 이렇게 좁지 않습니다.

이 둘을 거르는 **HogQL 조건식**(아래 한 줄)을 `environment`·`app` 필터와 함께 **모든 PostHog 조회에** 겁니다.

```
NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%') AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
```

- `query-trends`/`query-funnel` 등 타입이 있는 조회: 최상위 `properties` 배열에 `{"type": "hogql", "key": "<위 조건식>"}`를 환경·앱 필터와 함께 추가합니다. (실제로 2026-10-26 DAU가 68 → 12로 줄어드는 것을 확인했습니다.)
- `execute-sql`: `WHERE` 절에 `AND <위 조건식>`을 붙입니다.
- **저장된 Insight 재실행**(아래 "저장된 Insight 기반 확장 지표"): 환경 필터를 추가하는 것과 같은 방식으로 이 조건식도 추가한 뒤 재실행합니다. 저장된 정의에는 이 필터가 없습니다.
- 한 번이라도 이 필터를 빼면 방문자·페이지뷰·세션·기기·유입 지표가 모두 부풀려집니다. 특히 사용자가 적은 서비스라 합성 접속 몇 건이 비율(예: Android 비중)까지 바꿉니다.
- **걸러지지 않는 것**: 개발자·팀원이 실제 휴대폰으로 직접 써 본 접속(예: 2026-10-25에 페이지뷰 157건/방문자 10명)은 구분할 방법이 없습니다. 특정 날짜에 페이지뷰만 비정상적으로 많고 방문자가 적으면 "내부 테스트 가능성(추정)"으로 코멘트에 적으세요.
- DB 관리자 통계(2번)는 이 문제와 무관하지만, 팀 내부 테스트 계정이 회원·주문에 포함되어 있을 수 있습니다. 구분할 근거가 없으면 "내부 테스트 계정 포함 가능"이라고만 적고 임의로 빼지 마세요.

### $current_url 관련 PII 주의 (2026-08-03 완화됨)

이전에 `/auth/*` 콜백·가입 URL의 쿼리스트링에 실제 이메일·OAuth 코드가 그대로 노출되던 문제가 있었습니다(인기 페이지·유입경로 분석 중 발견). `apps/web-user`·`apps/web-seller`의 `posthog.config.ts`에 `before_send` 훅을 추가해 2026-08-03 배포분부터는 `$current_url`/`$referrer`에서 이런 값이 자동으로 제거됩니다(`/auth/*`는 쿼리스트링 전체 제거, 그 외 경로는 알려진 민감 키만 제거). **그 이전 시점 데이터(과거 pageview, 세션 리플레이)에는 여전히 원문이 남아있을 수 있으니**, 인기 페이지/유입경로 리포트에서 `$current_url`을 직접 노출할 때는 항상 도메인/경로만 쓰고 쿼리스트링은 표시하지 마세요. 세션 리플레이도 `/auth/*`에서는 이제 녹화 자체가 꺼집니다(`stopSessionRecording()`).

### 기본 제품 분석 지표 (PostHog 기본 기능)

- 활성 사용자: DAU / WAU / MAU — 정의는 0번 "기간·시간대"를 따릅니다(어제 / 최근 7일 / 최근 30일의 KST 완료 일자 고유 방문자)
- `$pageview` 총 건수, 신규 vs 재방문 사용자 비율
- 이미 PostHog에 저장된 Insight/Dashboard가 있다면 함께 조회해서 보여주세요 (팀이 이미 만들어둔 지표를 놓치지 않기 위함) — `insight`/`dashboard` 도메인 조회.

### 저장된 Insight 기반 확장 지표 (기본 포함 — 매번 조회)

팀이 이미 아래 Insight들을 만들어뒀습니다 (`system.insights`, PostHog 프로젝트 497749). DAU/WAU/MAU/Pageviews처럼 이미 위에서 직접 쿼리하는 것 말고, 아래는 **매 리포트마다 기본으로 함께 조회**해서 대시보드에 포함하세요 — 새로 설계할 필요 없이 팀이 만들어둔 정의를 그대로 재실행하면 됩니다.

| 묶음        | Insight                         | short_id   |
| ----------- | ------------------------------- | ---------- |
| 페이지·유입 | Most Popular Landing Pages      | `KUrqDhTL` |
| 페이지·유입 | Unique Users on Landing Page(s) | `t2MZ0Nxm` |
| 페이지·유입 | Referring Domains               | `Yg8lWuRy` |
| 페이지·유입 | Top referrers                   | `Af8TydUi` |
| 페이지·유입 | New & Returning Users           | `bfwMElLX` |
| 사용자 환경 | Unique Users by Device Type     | `L0xOeUjj` |
| 사용자 환경 | Unique Users by Browser         | `bm0V59iK` |
| 사용자 환경 | Which country are users from?   | `AeWquiKN` |
| 참여도      | Pages Per Session               | `OA1xCy7l` |
| 참여도      | Average Session Duration        | `376jcwtB` |
| 참여도      | Unique Sessions Trend           | `7hT84UAa` |
| 참여도      | Retention                       | `9bhduJEN` |

- 조회 방법: `system.insights`에서 `short_id`로 쿼리 정의를 가져오거나(`insight` 관련 도구로 `retrieve`), 이름으로 검색해도 됩니다. short_id는 UI가 바뀌어도 안정적이니 우선 사용하세요.
- **환경 필터는 저장된 정의에 없을 수 있습니다.** 위 "환경 필터링" 규칙이 여기에도 그대로 적용됩니다 — 원본 그대로 인용하지 말고, 쿼리 정의를 가져온 뒤 `environment` 속성 필터를 추가해 선택된 환경으로 재실행하세요.
- **기간도 저장된 정의 그대로 쓰지 않습니다.** 저장된 Insight의 기간(롤링 N일·UTC 기준)을 0번의 구간(KST 완료 일자)으로 바꿔서 재실행합니다.
- short_id가 바뀌었거나 삭제된 Insight가 있으면 조용히 건너뛰지 말고 이름으로 다시 찾아보고, 그래도 없으면 사용자에게 알리세요.

### 커스텀 이벤트 기반 전환 퍼널 (팀이 추가한 기능)

전체 이벤트 정의는 `apps/web-user/src/common/types/analytics.type.ts`에 있습니다 (`view_*`=화면 노출 `engage_*`=클릭 `request_*`=요청 전송 `success_*`/`fail_*`=서버 응답 네이밍 규칙). **이 파일은 도메인이 추가될 때마다 계속 늘어나므로 총 개수를 하드코딩해서 믿지 마세요** — 필요하면 `grep -oP '^\s+\K[a-z_]+(?=:\s*(\{|never))' apps/web-user/src/common/types/analytics.type.ts | sort -u | wc -l`로 매번 다시 세세요 (2026-08-03 기준 70개였음, 그 이후 더 늘었을 수 있음). 아래 퍼널 단위로 단계별 전환율을 HogQL/퍼널 쿼리로 계산하세요.

| 퍼널                 | 이벤트 순서                                                                                  |
| -------------------- | -------------------------------------------------------------------------------------------- |
| 예약(주문)           | `view_product_detail` → `engage_reservation` → `request_reservation` → `success_reservation` |
| 회원가입             | `view_login_entry` → `engage_social_select` → `request_signup` → `success_signup`            |
| 검색                 | `engage_search_bar` → `request_search` → `view_search_result`                                |
| 지도 탐색            | `view_map` → `engage_store_pin`                                                              |
| 미입금 리마인드 대응 | `view_payment_alarm` → `engage_easy_payment` → `success_payment_complete`                    |
| 리뷰 작성            | `engage_review_write` → `success_review_submit`                                              |
| 예약 취소            | `engage_cancel_reservation` → `request_cancel_reservation` → `success_cancel_reservation`    |
| 환불 정보 등록       | `request_refund_info` → `success_refund_info`                                                |

사용자가 위 목록에 없는 세부 질문(특정 이벤트의 기간별 추이, 특정 유저 세그먼트 등)을 하면 같은 MCP로 자유롭게 추가 질의하세요.

**주의**: 위 이벤트 이름은 `analytics.type.ts`의 타입 정의 기준이며, 실제로 `trackEvent(...)` 호출까지 코드에 있어도 PostHog에 단 한 번도 수집되지 않았을 수 있습니다 (버그로 전송이 안 됐거나, 아직 그 인터랙션을 아무도 실행하지 않은 경우). `read-data-schema`(`kind: events`)로 실제 존재하는 이벤트만 먼저 확인하고, 목록에 없는 이벤트는 퍼널에서 빼거나 실재하는 근접 이벤트로 대체하세요.

**`read-data-schema`(events)만으로는 부족합니다 — 최근에 추가된 이벤트가 taxonomy 캐시에 아직 안 잡혀서 "없음"으로 오판될 수 있습니다.** (실제 사례: `engage_social_select`/`request_social_auth`는 production에서 매주 발화 중이었는데 `read-data-schema`의 events 목록엔 안 떴고, `execute-sql`로 이벤트명을 직접 카운트하니 나왔음.) 그래서 이벤트 존재 여부를 확실히 판단하려면 `execute-sql`로 `SELECT event, count() FROM events WHERE event IN (...) GROUP BY event`를 직접 돌려서 카운트를 확인하세요. `execute-sql`이 모르는 이벤트명에는 `<taxonomy_warnings>`를 붙여주는데, 이건 프로젝트 전체(모든 환경) 기준으로 "한 번도 발생한 적 없음"을 뜻하는 신뢰할 수 있는 신호입니다.

이때 "이벤트가 없다"를 "아무도 안 눌러서 데이터가 없다"로 단정하지 말고, DB 통계(예: 주문 건수)와 대조해서 크게 어긋나면(예: DB엔 주문이 여러 건인데 `request_reservation`/`success_reservation`이 0건) 트래킹 누락 가능성으로 사용자에게 알려주세요.

### 이벤트 태깅 커버리지 감사 (사용자가 "태깅/텍소노미/트래킹 커버리지" 등을 물을 때만 — 기본 리포트엔 불필요)

사용자가 "이벤트 태깅 잘 되고 있어?", "새로 추가한 이벤트도 반영됐어?" 처럼 트래킹 자체의 완성도를 물으면, 위 8개 퍼널 대신 `analytics.type.ts`에 정의된 **전체** 이벤트를 감사하세요. "코드에 반영됐는지"와 "PostHog에 실제로 찍히는지"는 서로 다른 질문이라 반드시 둘 다 확인해야 합니다 — 둘 중 하나만 보면 오판합니다(실제로 코드 호출은 70개 전부 있었는데 PostHog엔 16개가 전혀 안 찍혀 있던 사례가 있었음, `analytics.type.ts`에 타입만 있고 `trackEvent()` 호출이 아예 없는 경우도 이론상 가능).

1. 위 grep 명령으로 전체 이벤트명 목록을 뽑는다.
2. **코드 호출 여부**: `trackEvent(` 호출부를 `apps/web-user/src` 전체에서 찾아(agent 위임 권장 — 개수가 많음) 각 이벤트명이 실제로 최소 1곳에서 호출되는지 file:line과 함께 확인. 타입만 있고 호출이 없으면 "타입은 정의됐지만 아무 데도 안 쓰이는 죽은 타입"이므로 최우선으로 보고.
3. **PostHog 발생 여부**: 선택된 환경(0번) 기준으로 `execute-sql`에 전체 목록을 `IN (...)`으로 넣어 `GROUP BY event`. 실행 결과에 안 나온 건 해당 환경에서 90일간 0건.
4. 같은 실행에서 뜨는 `<taxonomy_warnings>`가 그 이벤트명이 프로젝트 전체(모든 환경, 전체 기간) 기준으로 진짜 한 번도 없었는지 알려줌 — "코드 호출은 있는데 PostHog엔 전혀 안 찍힌" 이벤트를 걸러내는 확정 신호입니다.
5. 이벤트명이 여러 도메인/파일에서 재사용되고 있으면(예: 검색과 지도가 같은 이벤트명을 공유) 속성만으로 도메인을 구분할 수 있는지 확인하고, 안 되면 사용자에게 알려주세요 — 나중에 데이터가 쌓여도 분리 분석이 불가능해집니다. 반대 패턴(같은 동작인데 도메인마다 이름이 다름)도 일관성이 없다는 신호이니 같이 확인하세요. (실제 사례, 2026-08-03에 둘 다 정리함: ① 검색/지도 필터 적용 이벤트가 `success_filter_apply` 하나를 공유하던 걸 지도 쪽을 `success_map_filter_apply`로 분리, ② "필터 아이콘 클릭"이 홈/검색은 `engage_filter`인데 지도만 다른 이름(`engage_filter_open`)이던 걸 `engage_map_filter_open`으로 통일 — 둘 다 `success_map_area_search`처럼 지도 도메인은 `map_` 접두어를 쓰는 기존 규칙을 따름. 앞으로 새 이벤트를 추가할 때도 이 접두어 규칙을 유지하세요.)
6. 네 그룹으로 나눠 보고: (a) 선택 환경에서 발화 확인, (b) 코드 호출은 있고 프로젝트 어딘가엔 발생 이력이 있지만 선택 환경에서는 90일간 0건, (c) 코드 호출은 있지만 프로젝트 전체에서 한 번도 발화된 적 없음(taxonomy_warnings에 뜬 것), (d) 코드 호출 자체가 없음(있다면 최우선 보고 대상).
7. 사용자가 기획팀 원본 텍소노미 문서(스프레드시트를 복사한 텍스트 등)를 함께 주면, 그 문서와 `analytics.type.ts`도 대조하세요 — 문서에는 있는데 코드엔 없는 이벤트(미구현), 코드에 있는데 문서 갱신이 안 된 케이스(예: 백엔드 전용이라 프론트에서 계측 불가한 이벤트, 아직 UI가 mock이라 제외한 이벤트는 `analytics.type.ts` 파일 상단 주석에 이미 문서화되어 있음)를 구분해서 알려주세요.

## 2. 관리자(DB) 통계 수집

백엔드 관리자 통계 API를 호출합니다. 관리자 로그인은 아이디/비번 + Google OTP(TOTP)가 필수입니다. 자동화(루틴)는 전용 관리자 API 키를 `X-Admin-Api-Key` 헤더로 쓰고(4번), 사람이 요청한 경우에는 아래처럼 토큰을 받습니다. **저장된 비밀번호나 OTP secret을 요구하거나 대신 로그인을 시도하지 마세요** — 대신 사용자에게 해당 환경의 유효한 관리자 액세스 토큰(Bearer)을 요청하세요. (web-admin에 로그인한 뒤 브라우저 개발자도구 → localStorage 또는 네트워크 탭에서 확인 가능하다고 안내하면 됩니다.)

토큰을 받으면 아래 엔드포인트를 `Authorization: Bearer <token>`으로 호출합니다 (base URL은 0번에서 정한 환경 기준). **API에는 전역 prefix `/v1`이 붙습니다** (`apps/backend/src/common/constants/app.constants.ts`의 `API_PREFIX`) — `/admin/statistics/...`가 아니라 `/v1/admin/statistics/...`로 호출해야 합니다. prefix를 빼먹으면 404(Not Found)가 납니다.

| 메서드/경로                                                                                                                 | 응답 형태                                                                          |
| --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `GET /v1/admin/statistics/users`                                                                                            | `{ consumers: {total, today, last7Days, last30Days, withdrawn}, sellers: {...} }`  |
| `GET /v1/admin/statistics/orders`                                                                                           | `{ total, gmv, byStatus: [{status, count}] }`                                      |
| `GET /v1/admin/statistics/stores`                                                                                           | `{ stores: {total, today, last7Days, last30Days}, businessVerifiedStores: {...} }` |
| `GET /v1/admin/statistics/store-entry-requests`                                                                             | `{ storeEntryRequests: {total, today, last7Days, last30Days} }`                    |
| `GET /v1/admin/statistics/daily-trends?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD&metrics=signups,orders,stores,entryRequests` | `{ days: [...] }` (날짜는 Asia/Seoul 기준)                                         |

예: `https://api.picakes.com/v1/admin/statistics/users`, `https://api-staging.picakes.com/v1/admin/statistics/orders`.

토큰이 만료됐거나 401이 오면 사용자에게 재로그인 후 새 토큰을 요청하세요.

### 통계(statistics) 외 추가 관리자 데이터 (사용자가 "DB에 있는 거 다 보여줘" 등으로 명시 요청할 때만)

`apps/backend/src/apis/admin/`에는 `statistics` 말고도 12개 컨트롤러가 더 있습니다(2026-08-03 기준). 기본 리포트엔 위 5개 statistics 엔드포인트로 충분하지만, 사용자가 더 상세한 관리자 데이터를 요청하면 아래도 조회하세요:

| 컨트롤러                         | 주요 엔드포인트                            | 비고                                                                           |
| -------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------ |
| `member-management`              | `consumers`, `sellers` (목록)              | **개인정보 원문 포함** (이름·전화번호·이메일) — 아래 PII 규칙 필수             |
| `store-management`               | `stores`, `stores/:id`                     | 상세는 GMV·상태별 주문·`topProductsByRevenue` 포함 (매출 랭킹은 여기서만 나옴) |
| `store-entry-request-management` | `requests`, `requests/:id`                 | place 스냅샷(상호명·주소 등 공개된 사업자 정보) + 요청자 정보(PII)             |
| `seller-segment-management`      | `segments`                                 | 세그먼트 정의 + 인원수, PII 아님                                               |
| `admin-management`               | `accounts`, `requests`                     | 관리자 계정 자체에 대한 데이터라 사용자 대상 리포트와 무관, 보통 스킵          |
| 콘텐츠 CRUD                      | `home-banners`, `notices`, `qnas`, `terms` | 건수만 세면 됨, PII 아님                                                       |

**PII 규칙 (필수)**: `member-management/consumers`·`sellers`, `store-entry-request-management` 응답에는 실제 이름·전화번호·이메일이 원문으로 들어있습니다. **Artifact나 채팅 답변에 이 원문을 그대로 올리지 마세요** — Artifact는 나중에 공유될 수 있는 페이지입니다. 대신:

- 개수·비율 같은 집계만 표시 ("구매자 2명 조회됨 — 원문 미포함" 식으로 명시).
- 사업자명·주소처럼 원래 공개된 정보(카카오맵 등)는 노출해도 무방합니다 — 개인 연락처와는 다릅니다.
- 요청자 phone 필드가 `"REVIEW_ACCOUNT"`처럼 명백히 심사용 테스트 계정 마커인 경우는 그 사실 자체(진성 사용자가 아닐 수 있음)를 알려주는 게 유용하니 언급하세요.

## 3. 대시보드로 표시

1. `dataviz` skill을 먼저 로드해서 카드/스탯 타일/차트 스타일 가이드를 따르세요.
2. Artifact(HTML)로 대시보드를 렌더링합니다. 섹션 구성:
   - ① 회원/가입 현황 (구매자·판매자 총계, 어제/최근 7일/최근 30일 신규(daily-trends 합산), 탈퇴)
   - ② 주문·매출 (총 주문 수, GMV, 상태별 건수)
   - ③ 스토어·입점 현황 (스토어 총계, 사업자 인증 완료, 입점 요청)
   - ④ PostHog 행동 지표 (DAU/WAU/MAU, pageview) 및 전환 퍼널
   - ⑤ 일별 추이 (신규가입/주문/스토어/입점요청 라인 또는 바 차트)
   - ⑥ 페이지 · 유입 분석 (인기 페이지, 유입 경로/리퍼러, 신규 vs 재방문)
   - ⑦ 사용자 환경 · 참여도 (기기/브라우저/국가 분포, 세션당 페이지 수, 평균 세션 시간, 리텐션)

   이건 기본 템플릿입니다. 사용자가 명시적으로 더 요청하면 아래를 이어서 번호 매겨 추가하세요(매번 다 넣지 마세요 — 무거워집니다):
   - 관리자 상세 데이터 (위 "통계 외 추가 관리자 데이터" 참고 — DB에 있는 거 다 보여달라고 할 때)
   - 이벤트 텍소노미 전수 감사 (위 "이벤트 태깅 커버리지 감사" 참고 — 태깅/텍소노미를 물을 때)
   - 세션 리플레이 · 개인정보 보호 점검 (PII·세션 리플레이를 물을 때)

   섹션이 늘어나면 성격이 다른 그룹(비즈니스 현황/DB, 행동 데이터/PostHog, 엔지니어링 QA)끼리 시각적으로 묶어서 구분하세요 — DB 지표와 PostHog 지표를 순서 없이 섞어놓으면(예: ①②③ DB → ④ PostHog → ⑤ DB → ...) 산만해집니다. 섹션을 추가/재배치할 때마다 번호가 실제로 순서대로 매겨져 있는지(중간에 번호 빠진 무제목 섹션이 생기지 않는지) 반드시 다시 확인하세요.

3. 채팅 답변에는 대시보드와 별도로 핵심 하이라이트 3~5개(눈에 띄는 수치, 전일/전주 대비 변화 등)를 텍스트로 짧게 요약하세요.

## 4. 자동 실행 모드 (클라우드 루틴)

루틴 프롬프트가 "자동 실행 모드"를 지정하면 위 절차를 아래처럼 바꿔 적용합니다.

### 실행 순서와 필수 단계 (이 순서대로, 필수 단계는 생략하지 마세요)

1. 기간·시간대 기준 계산 (0번)
2. DB 통계 수집 (2번) — **필수**
3. PostHog 수집 (1번) — **필수: 활성 사용자·페이지뷰·일별 추이, 전환 퍼널 이벤트 집계(execute-sql로 이벤트별 건수·고유 사용자)**. 선택: 저장된 Insight 12개 재실행(타임아웃이 반복되거나 시간이 오래 걸리면 생략 가능하지만 **생략했다고 대시보드와 Discord에 적습니다**). **PostHog 쿼리는 한 번에 최대 2개씩만 보냅니다**(동시에 많이 보내면 60초 타임아웃이 반복됩니다).
4. **Artifact 대시보드 발행 — 필수.** `dataviz` skill을 로드한 뒤 Artifact 도구를 호출합니다. 도구가 바로 보이지 않으면 `ToolSearch`로 `select:Artifact`를 먼저 로드하세요(PostHog 도구처럼 지연 로드될 수 있습니다). 시간을 아끼려고 생략하지 마세요.
5. Discord 전송 — 4번에서 받은 Artifact 링크를 포함합니다.
6. 보고

필수 단계를 못 한 경우는 **실제로 호출했는데 오류가 났을 때만** 허용되며, 그때는 오류 내용을 Discord와 보고에 그대로 적습니다. 호출하지 않고 "생략했다"고 쓰는 것은 허용되지 않습니다. 푸시 알림은 따로 보내지 않습니다(루틴 설정이 처리합니다).

- **질문하지 않음**: 환경은 항상 production(`https://api.picakes.com`), PostHog 필터는 `environment=production`, `app=web-user`, 그리고 **합성(테스트) 접속 제외**(1번).
- **관리자 통계 인증**: 토큰을 요청하지 말고 루틴 프롬프트의 관리자 API 키를 `X-Admin-Api-Key: <키>` 헤더로 보냅니다. 이 키는 통계 외 admin API에도 접근 가능하므로 로그·결과물에 노출하지 마세요.
- **PostHog 실패 시**: 도구 호출이 계속 실패하면 PostHog 부분은 건너뛰고 DB 통계만으로 진행하며, 그 사실을 Discord와 최종 보고에 적습니다.
- **대시보드**: 3번 절차대로 `dataviz` skill을 먼저 로드하고 Artifact를 발행합니다. title `Picake 일일 리포트 · {YYYY-MM-DD}`, favicon 📊. 매번 새 Artifact로 발행해도 됩니다.
- **Discord 전송**: jq로 아래 구조의 payload를 만들어 루틴 프롬프트의 웹훅 URL로 `curl` POST합니다. 숫자만 그날 값으로 채우고 구조·순서는 고정합니다.

```
{
  embeds: [{
    title: "📊 Picake 일일 데이터 리포트 · {YYYY-MM-DD}",
    color: (특이사항 없으면 5763719 초록, 있으면 15105570 주황),
    description: "1줄 핵심 요약 (예: 구매자 X명·주문 Y건·특이사항 없음)",
    fields: [
      { name: "🗓️ 집계 기준", value: "한국 시간(KST) · 어제 {D1 MM/DD}까지 완료된 일자 · 최근 7일 {D7 MM/DD}~{D1 MM/DD} · 점검 {NOW_KST}", inline: false },
      { name: "👥 구매자", value: "총 N · 어제 +N", inline: true },
      { name: "🏪 판매자", value: "총 N · 어제 +N", inline: true },
      { name: "📦 주문", value: "총 N건 · GMV ₩N", inline: true },
      { name: "\u200b", value: "──────────────────────────────", inline: false },
      { name: "🏪 스토어", value: "총 N · 인증완료 N", inline: true },
      { name: "📋 입점요청", value: "총 N · 어제 +N", inline: true },
      { name: "👁️ PostHog", value: "DAU N · WAU N · MAU N (포함 안 됐으면 \"미포함\")", inline: true },
      { name: "\u200b", value: "──────────────────────────────", inline: false },
      { name: "🔍 추이·특이사항", value: "(2~3줄 bullet, 없으면 \"특이사항 없음\")", inline: false },
      { name: "📊 전체 대시보드", value: "[클릭해서 자세히 보기 →](ARTIFACT_URL)", inline: false }
    ],
    footer: { text: "Picake 일일 리포트 · 매일 09:00 KST · 한국 시간, 어제까지 완료된 일자 기준" },
    timestamp: (now | strftime("%Y-%m-%dT%H:%M:%SZ"))
  }]
}
```

- 구분자 필드(`name`이 제로폭 문자 `\u200b`)로 그룹을 나누고, 숫자 필드는 `inline: true` 3개씩, 문장형 필드는 `inline: false`.
- description은 숫자 나열이 아니라 한 줄 요약 문장.
- Artifact 링크는 반드시 `[클릭해서 자세히 보기 →](URL)` 마스킹 링크로 단독 필드에 넣습니다(맨 URL을 넣으면 링크가 없는 줄 안다는 피드백이 있었음).
- PostHog가 빠졌으면 해당 필드와 footer에 짧게 명시합니다.
- color는 데이터 리포트 표시용입니다 — 장애 알림처럼 빨간색을 쓰지 마세요.
- HTTP 상태코드가 2xx가 아니면 1회만 재시도하고, 실패하면 에러 내용을 최종 메시지에 남깁니다.
- **보고**: 마지막에 대시보드에 담은 내용(PostHog 포함 여부), Artifact 링크, Discord 전송 결과(HTTP 상태코드)를 세션 로그에 짧게 남깁니다. 코드 수정·커밋은 하지 않습니다.
