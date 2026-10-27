---
name: picake-data-report
description: 사용자 데이터, 전환율, PostHog 지표, 회원수, 사장님 입점 유입(소개 사이트·설문 클릭) 등 Picake의 종합 데이터를 보여달라는 요청을 받으면 사용합니다. "사용자 데이터 보여줘", "전환율 어때", "이번 주 지표 알려줘", "회원수 몇 명이야", "소개 페이지 몇 명 봤어?", "입점 설문 클릭 있었어?" 같은 요청에 이 skill을 참고해 PostHog(행동 데이터)와 관리자 통계(DB 데이터)를 함께 조회해 리포트를 보여주세요.
---

# 종합 데이터 리포트

> **이 문서의 `apps/web-*` 경로는 [`picake-engineering-web`](https://github.com/picake-team/picake-engineering-web), `apps/backend/...` 경로는 [`picake-engineering-backend`](https://github.com/picake-team/picake-engineering-backend) 레포의 것입니다.** 로컬에서는 형제 폴더(`../picake-engineering-web`, `../picake-engineering-backend`)에서 확인하세요. 매일 09:00 KST 클라우드 루틴("Picake 일일 데이터 리포트")이 이 레포를 체크아웃해 4번 **자동 실행 모드**로 실행합니다.

Picake의 데이터는 두 곳에 나뉘어 있습니다. **행동 데이터(페이지뷰·클릭·전환 퍼널)는 PostHog**, **DB가 원본인 지표(회원수·주문·매출·스토어)는 백엔드 관리자 통계 API**입니다 (`apps/backend/src/apis/admin/controllers/statistics.controller.ts` 주석에 명시된 설계). 이 skill은 둘을 한 번에 모아 예쁜 대시보드로 보여주는 절차입니다.

### 파일 구성 (공통 코어 + 앱별 모듈)

PostHog 행동 데이터는 **앱마다 성격이 달라 모듈 파일로 나눠 둡니다.** 이 파일(`SKILL.md`)에는 모든 앱에 공통인 규칙(기간·시간대, 환경 필터, 합성 접속 제외, DB 통계, 대시보드·Discord 구조, 자동 실행)만 두고, 앱별 지표·쿼리는 모듈에 둡니다.

| 모듈 | PostHog `app` | 다루는 것 | 대시보드 그룹 |
| --- | --- | --- | --- |
| [`apps/web-user.md`](apps/web-user.md) | `web-user` | 고객 서비스 행동(활성 사용자, 저장된 Insight, 전환 퍼널, 태깅 감사) | B |
| [`apps/web-info.md`](apps/web-info.md) | `web-info` (+ web-user의 입점 게이트 `/partner`) | 사장님 유입(소개 사이트 방문, 게이트 → 소개 → 설문 클릭) | C |
| (아직 없음) `apps/web-seller.md` | `web-seller` | 판매자 앱 행동 — 5번 "앱 모듈 추가" 참고 | D(예정) |

리포트를 만들 때는 **이 파일을 먼저 읽고, 이어서 모든 모듈 파일을 읽습니다.** 사용자가 특정 앱만 물으면(예: "소개 페이지 몇 명 봤어?") 해당 모듈만 읽어도 됩니다.

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

**게다가 이 PostHog 프로젝트(497749)는 `web-user`뿐 아니라 `web-seller`·`web-info`도 같이 씁니다** (`apps/web-seller/src/common/config/posthog.config.ts`도 동일한 방식으로 super property를 등록, `app: "web-seller"`). 그래서 `environment` 필터만으로는 부족하고 **`app` 필터도 항상 같이 걸어야 합니다** — 안 그러면 web-seller 트래픽이 web-user 리포트에 섞여 들어갈 수 있습니다 (2026-08-03 기준 web-seller는 아직 development 트래픽뿐이라 지금 당장은 영향 없지만, 실사용이 시작되면 바로 문제가 됩니다). `web-info`(정보·소개 사이트, `app: "web-info"`)도 같은 방식입니다. `web-admin`은 PostHog 연동 자체가 없습니다(내부 툴이라 의도적으로 제외된 것으로 보임).

**따라서 이 skill의 모든 PostHog 쿼리(query-trends, query-funnel, execute-sql 등)에는 0번에서 정한 환경으로 `properties.environment` 필터와 **`properties.app` 필터(각 앱 모듈 머리의 `app` 값. 예: web-user 모듈이면 `"web-user"`)**를 반드시 같이 걸어야 합니다.** 필터를 빼먹으면 로컬/스테이징 테스트 트래픽이나 다른 앱(web-seller·web-info)의 트래픽이 production 숫자에 섞여 들어가고, 특히 표본이 작을 때 심하게 왜곡됩니다.

- `query-trends`/`query-funnel`: 최상위 `properties` 배열에 `{"key": "environment", "operator": "exact", "type": "event", "value": ["production"]}`와 `{"key": "app", "operator": "exact", "type": "event", "value": ["web-user"]}`를 함께 추가.
- `execute-sql`(HogQL): `WHERE properties.environment = 'production' AND properties.app = 'web-user'`를 다른 조건과 함께 걸기.
- 값이 실제로 뭐가 있는지 헷갈리면 `read-data-schema`(`kind: event_property_values`, `event_name: $pageview`, `property_name: environment` 또는 `app`)로 먼저 확인하세요.
- 기존 저장된 Insight/Dashboard(web-user 모듈의 "저장된 Insight")는 이 필터가 안 걸려 있을 수 있으니, 그대로 인용하지 말고 숫자를 재확인하세요.
- 앱을 넘는 지표(예: web-info 모듈의 "게이트 → 소개 → 설문")는 모듈에 적힌 조건으로만 `app`을 여러 개 겁니다. 그 외에는 앱 하나만 겁니다.
- 리포트를 시작하기 전에 한 번씩 `SELECT properties.app, properties.environment, count() FROM events WHERE timestamp >= now() - INTERVAL 30 DAY GROUP BY 1, 2`로 전체 분포를 훑어보면, 새로운 앱/환경이 추가되거나 태깅이 누락된 이벤트가 있는지 빠르게 감지할 수 있습니다.

### 합성(테스트) 접속 제외 (필수, 환경 필터와 같이 항상 건다)

**봇으로 표시되지 않는 합성 접속이 실제 방문자처럼 집계됩니다.** 실제로 2026-10-26의 방문자(고유 사용자)가 73명으로 보였는데, 그중 61명이 합성 접속이었고 실제는 약 12명이었습니다. 원인은 두 가지입니다.

- **웹 성능 점검 루틴의 Lighthouse(PageSpeed Insights)**: 구글 서버에서 `moto g power (2022)`라는 가상 Android 기기로 접속하며, 실행할 때마다 페이지를 여러 번 새로 열어 매번 "새 방문자"가 됩니다. web-user뿐 아니라 web-info 페이지도 잽니다.
- **개발자의 모바일 에뮬레이션·자동화 테스트(Playwright 등)**: 폭 500px 이하의 **데스크톱** 브라우저로 나타납니다. 실제 데스크톱 사용자의 창은 이렇게 좁지 않습니다.

이 둘을 거르는 **HogQL 조건식**(아래 한 줄)을 `environment`·`app` 필터와 함께 **모든 PostHog 조회에** 겁니다.

```
NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%') AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
```

- `query-trends`/`query-funnel` 등 타입이 있는 조회: 최상위 `properties` 배열에 `{"type": "hogql", "key": "<위 조건식>"}`를 환경·앱 필터와 함께 추가합니다. (실제로 2026-10-26 DAU가 68 → 12로 줄어드는 것을 확인했습니다.)
- `execute-sql`: `WHERE` 절에 `AND <위 조건식>`을 붙입니다.
- **저장된 Insight 재실행**(web-user 모듈의 "저장된 Insight 기반 확장 지표"): 환경 필터를 추가하는 것과 같은 방식으로 이 조건식도 추가한 뒤 재실행합니다. 저장된 정의에는 이 필터가 없습니다.
- 한 번이라도 이 필터를 빼면 방문자·페이지뷰·세션·기기·유입 지표가 모두 부풀려집니다. 특히 사용자가 적은 서비스라 합성 접속 몇 건이 비율(예: Android 비중)까지 바꿉니다.
- **걸러지지 않는 것**: 개발자·팀원이 실제 휴대폰으로 직접 써 본 접속(예: 2026-10-25에 페이지뷰 157건/방문자 10명)은 구분할 방법이 없습니다. 특정 날짜에 페이지뷰만 비정상적으로 많고 방문자가 적으면 "내부 테스트 가능성(추정)"으로 코멘트에 적으세요.
- DB 관리자 통계(2번)는 이 문제와 무관하지만, 팀 내부 테스트 계정이 회원·주문에 포함되어 있을 수 있습니다. 구분할 근거가 없으면 "내부 테스트 계정 포함 가능"이라고만 적고 임의로 빼지 마세요.

### $current_url 관련 PII 주의 (모든 앱, 2026-08-03 완화됨)

이전에 `/auth/*` 콜백·가입 URL의 쿼리스트링에 실제 이메일·OAuth 코드가 그대로 노출되던 문제가 있었습니다(인기 페이지·유입경로 분석 중 발견). `apps/web-user`·`apps/web-seller`의 `posthog.config.ts`에 `before_send` 훅을 추가해 2026-08-03 배포분부터는 `$current_url`/`$referrer`에서 이런 값이 자동으로 제거됩니다(`/auth/*`는 쿼리스트링 전체 제거, 그 외 경로는 알려진 민감 키만 제거). **그 이전 시점 데이터(과거 pageview, 세션 리플레이)에는 여전히 원문이 남아있을 수 있으니**, 인기 페이지/유입경로 리포트에서 `$current_url`을 직접 노출할 때는 항상 도메인/경로만 쓰고 쿼리스트링은 표시하지 마세요. 세션 리플레이도 `/auth/*`에서는 이제 녹화 자체가 꺼집니다(`stopSessionRecording()`).

### 앱별 지표

여기부터는 앱마다 다릅니다. 위 공통 규칙을 지킨 채 각 모듈 파일([`apps/web-user.md`](apps/web-user.md), [`apps/web-info.md`](apps/web-info.md))의 지표·쿼리를 수집합니다.

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
2. Artifact(HTML)로 대시보드를 렌더링합니다. **성격이 다른 데이터는 그룹으로 나눠 보여 줍니다.** 그룹 순서와 섹션 번호는 고정입니다.

   | 그룹 | 섹션 | 출처 |
   | --- | --- | --- |
   | **A. 비즈니스 현황** | ① 회원/가입 현황 (구매자·판매자 총계, 어제/최근 7일/최근 30일 신규(daily-trends 합산), 탈퇴) · ② 주문·매출 (총 주문 수, GMV, 상태별 건수) · ③ 스토어·입점 현황 (스토어 총계, 사업자 인증 완료, 입점 요청) · ④ 일별 추이 (신규가입/주문/스토어/입점요청 라인 또는 바 차트) | DB (관리자 통계 API) |
   | **B. 고객 서비스 (web-user)** | ⑤ 활성 사용자(DAU/WAU/MAU)·페이지뷰·전환 퍼널 · ⑥ 페이지 · 유입 분석 (인기 페이지, 유입 경로/리퍼러, 신규 vs 재방문) · ⑦ 사용자 환경 · 참여도 (기기/브라우저/국가 분포, 세션당 페이지 수, 평균 세션 시간, 리텐션) | PostHog `app=web-user` — [`apps/web-user.md`](apps/web-user.md) |
   | **C. 사장님 유입 (정보 사이트 web-info + 입점 게이트)** | ⑧ 사장님 유입 (소개 사이트 방문, 게이트 → 소개 → 설문 클릭, 페이지별, 유입 경로) | PostHog `app=web-info` (+ web-user `/partner`) — [`apps/web-info.md`](apps/web-info.md) |

   - 그룹마다 머리에 **그룹 이름과 출처**(DB / PostHog · 어느 앱)를 한 줄로 적어, 어떤 숫자가 어디서 왔는지 바로 보이게 합니다.
   - **앱끼리 방문자 수를 더하지 않습니다.** 같은 사람이 web-user와 web-info를 모두 방문할 수 있어 합치면 중복됩니다. 앱을 넘는 지표는 모듈에 정의된 사람 단위 쿼리로만 냅니다.
   - 그룹 C는 방문이 0이어도 숨기지 않고 한 줄로 남깁니다(web-info 모듈 5번).
   - 새 앱 모듈을 추가하면 C 다음에 그룹(D, E …)을 이어 붙이고 섹션 번호를 이어서 매깁니다(5번).

   이건 기본 템플릿입니다. 사용자가 명시적으로 더 요청하면 아래를 이어서 번호 매겨 추가하세요(매번 다 넣지 마세요 — 무거워집니다):
   - 관리자 상세 데이터 (2번 "통계 외 추가 관리자 데이터" 참고 — DB에 있는 거 다 보여달라고 할 때)
   - 이벤트 텍소노미 전수 감사 (web-user 모듈의 "이벤트 태깅 커버리지 감사" 참고 — 태깅/텍소노미를 물을 때)
   - 세션 리플레이 · 개인정보 보호 점검 (PII·세션 리플레이를 물을 때)

   추가 섹션은 성격이 맞는 그룹 끝에 붙이거나, 엔지니어링 점검성 섹션이면 맨 뒤에 별도 그룹(예: "Z. 점검")으로 묶습니다. 섹션을 추가/재배치할 때마다 번호가 실제로 순서대로 매겨져 있는지(중간에 번호 빠진 무제목 섹션이 생기지 않는지) 반드시 다시 확인하세요.

3. 채팅 답변에는 대시보드와 별도로 핵심 하이라이트 3~5개(눈에 띄는 수치, 전일/전주 대비 변화 등)를 텍스트로 짧게 요약하세요. 사장님 유입은 방문이나 설문 클릭이 있었을 때만 한 줄 넣습니다.

## 4. 자동 실행 모드 (클라우드 루틴)

루틴 프롬프트가 "자동 실행 모드"를 지정하면 위 절차를 아래처럼 바꿔 적용합니다.

### 실행 순서와 필수 단계 (이 순서대로, 필수 단계는 생략하지 마세요)

1. 기간·시간대 기준 계산 (0번)
2. DB 통계 수집 (2번) — **필수**
3. PostHog 수집 (1번 공통 규칙 + **모든 앱 모듈**) — 각 모듈의 "자동 실행에서" 항목을 따릅니다. **필수**: web-user 모듈의 필수 항목(활성 사용자·페이지뷰·일별 추이, 전환 퍼널 이벤트 집계)과 web-info 모듈의 필수 항목(sitemap 조회, 소개 사이트 방문, 페이지별, 입점 흐름). 선택 항목(저장된 Insight 12개 등)은 타임아웃이 반복되거나 시간이 오래 걸리면 생략할 수 있지만 **생략했다고 대시보드와 Discord에 적습니다.** **PostHog 쿼리는 한 번에 최대 2개씩만 보냅니다**(동시에 많이 보내면 60초 타임아웃이 반복됩니다).
4. **Artifact 대시보드 발행 — 필수.** `dataviz` skill을 로드한 뒤 Artifact 도구를 호출합니다. 도구가 바로 보이지 않으면 `ToolSearch`로 `select:Artifact`를 먼저 로드하세요(PostHog 도구처럼 지연 로드될 수 있습니다). 시간을 아끼려고 생략하지 마세요.
5. Discord 전송 — 4번에서 받은 Artifact 링크를 포함합니다.
6. 보고

**DB와 PostHog를 비교해 이상 징후를 쓸 때는 같은 기간끼리만 비교합니다.** DB의 누적(`total`)이나 오래된 건과 PostHog의 최근 30일 이벤트를 비교하면 "트래킹 누락"이라는 오경보가 납니다(실제 사례: DB 주문 누적 1건과 PostHog 30일 `request_reservation` 0건을 비교했지만, DB의 최근 30일 주문(`daily-trends` 합산)은 0건이라 누락이 아니었습니다). 이벤트는 코드에 구현돼 있는지(`apps/web-user`의 `trackEvent`)와 같은 기간의 DB 건수를 먼저 확인하고, 기간이 맞지 않으면 이상 징후로 쓰지 말고 "기간이 달라 비교 불가"라고 적습니다.

필수 단계를 못 한 경우는 **실제로 호출했는데 오류가 났을 때만** 허용되며, 그때는 오류 내용을 Discord와 보고에 그대로 적습니다. 호출하지 않고 "생략했다"고 쓰는 것은 허용되지 않습니다. 푸시 알림은 따로 보내지 않습니다(루틴 설정이 처리합니다).

- **질문하지 않음**: 환경은 항상 production(`https://api.picakes.com`), PostHog 필터는 `environment=production`, `app`은 각 모듈의 값, 그리고 **합성(테스트) 접속 제외**(1번).
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
      { name: "👁️ 고객 앱 (web-user)", value: "DAU N · WAU N · MAU N (포함 안 됐으면 \"미포함\")", inline: true },
      { name: "\u200b", value: "──────────────────────────────", inline: false },
      { name: "🧑‍🍳 사장님 유입 (소개·게이트)", value: "web-info 모듈 6번 형식 한 줄 (예: 소개 방문 7일 N명 · 30일 N명 / 설문 클릭 7일 N명 · 30일 N명 / 게이트 7일 N명)", inline: false },
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
- PostHog가 빠졌으면 해당 필드(고객 앱·사장님 유입)와 footer에 짧게 명시합니다.
- 사장님 유입 필드는 방문이 0이어도 빼지 않습니다(`최근 7일 방문 없음 (30일 …)`).
- color는 데이터 리포트 표시용입니다 — 장애 알림처럼 빨간색을 쓰지 마세요.
- HTTP 상태코드가 2xx가 아니면 1회만 재시도하고, 실패하면 에러 내용을 최종 메시지에 남깁니다.
- **보고**: 마지막에 대시보드에 담은 내용(PostHog 포함 여부), Artifact 링크, Discord 전송 결과(HTTP 상태코드)를 세션 로그에 짧게 남깁니다. 코드 수정·커밋은 하지 않습니다.

## 5. 앱 모듈 추가 (예: web-seller)

새 앱의 행동 데이터를 리포트에 넣을 때는 **이 파일에 지표를 늘리지 말고 `apps/<앱>.md`를 하나 추가합니다.** 공통 규칙(기간·환경 필터·합성 접속 제외·출력 형식)을 바꿀 때만 이 파일을 고칩니다.

모듈 머리에 반드시 적을 것 (기존 모듈과 같은 표 형식):

1. 대상(도메인·앱 경로)과 PostHog `app` 값 — 앱 코드의 `src/common/config/posthog.config.ts`의 `SUPER_PROPERTIES`에서 확인
2. 핵심 질문 한 줄 — 이 앱 데이터로 무엇을 판단하는지
3. 대시보드 그룹·섹션 번호, Discord 필드 이름(필드 1개 이내)

본문에 적을 것: 해석에 필요한 사실, 지표와 쿼리(페이지 경로는 하드코딩하지 말고 데이터·sitemap에서 읽기), 표본이 작을 때 규칙, 대시보드 배치, 자동 실행의 필수/선택 항목.

추가한 뒤에는 1번 "파일 구성" 표, 3번 그룹 표, 4번 Discord 구조에 한 줄씩 등록합니다.

- **web-seller (아직 모듈 없음, 2026-10-27 기준)**: `app: "web-seller"`로 페이지뷰만 쌓이고 커스텀 이벤트(`trackEvent`)는 아직 없습니다. production 실사용이 생기면(1번의 `properties.app, properties.environment` 분포로 확인) `apps/web-seller.md`를 만들고, 판매자 쪽 DB 지표(판매자 가입·스토어·주문 처리)와 함께 그룹 D로 붙입니다.
