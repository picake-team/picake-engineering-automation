# web-user 모듈 — 고객 서비스 행동 데이터

> [`SKILL.md`](../SKILL.md)의 공통 규칙(0번 기간·시간대, 1번 환경 필터·합성 접속 제외, 3번 대시보드 그룹)을 그대로 따릅니다. 이 파일에는 web-user에만 해당하는 지표·Insight·퍼널만 둡니다. `apps/web-*` 경로는 `picake-engineering-web` 레포의 것입니다.

| 항목 | 값 |
| --- | --- |
| 대상 | 고객 서비스 `https://picakes.com` (`apps/web-user`, 앱 WebView 포함) |
| PostHog `app` | `web-user` |
| 핵심 질문 | 고객이 얼마나 들어와서, 찾고, 예약까지 가는가 |
| 대시보드 위치 | 그룹 B · ⑤ 활성 사용자·전환 퍼널, ⑥ 페이지·유입, ⑦ 사용자 환경·참여도 |
| Discord | `👁️ 고객 앱 (web-user)` 필드 (DAU · WAU · MAU) |

- 입점 게이트(`/partner`) 방문도 web-user 방문자에 포함됩니다(사장님이 QR로 들어온 방문). 사장님 유입은 [`web-info.md`](web-info.md)에서 따로 보고, 두 그룹의 숫자를 더하지 않습니다.
- 2026-10-27 이전의 `/partner/about`(사장님 소개) 방문도 web-user 데이터에 남아 있습니다. 인기 페이지에 보이면 "사장님 소개(10/27 정보 사이트로 이전)"라고 표기합니다.

### 기본 제품 분석 지표 (PostHog 기본 기능)

- 활성 사용자: DAU / WAU / MAU — 정의는 SKILL.md 0번 "기간·시간대"를 따릅니다(어제 / 최근 7일 / 최근 30일의 KST 완료 일자 고유 방문자)
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
- **환경 필터는 저장된 정의에 없을 수 있습니다.** SKILL.md 1번 "환경 필터링"·"합성 접속 제외" 규칙이 여기에도 그대로 적용됩니다 — 원본 그대로 인용하지 말고, 쿼리 정의를 가져온 뒤 `environment` 속성 필터를 추가해 선택된 환경으로 재실행하세요.
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
3. **PostHog 발생 여부**: 선택된 환경(SKILL.md 0번) 기준으로 `execute-sql`에 전체 목록을 `IN (...)`으로 넣어 `GROUP BY event`. 실행 결과에 안 나온 건 해당 환경에서 90일간 0건.
4. 같은 실행에서 뜨는 `<taxonomy_warnings>`가 그 이벤트명이 프로젝트 전체(모든 환경, 전체 기간) 기준으로 진짜 한 번도 없었는지 알려줌 — "코드 호출은 있는데 PostHog엔 전혀 안 찍힌" 이벤트를 걸러내는 확정 신호입니다.
5. 이벤트명이 여러 도메인/파일에서 재사용되고 있으면(예: 검색과 지도가 같은 이벤트명을 공유) 속성만으로 도메인을 구분할 수 있는지 확인하고, 안 되면 사용자에게 알려주세요 — 나중에 데이터가 쌓여도 분리 분석이 불가능해집니다. 반대 패턴(같은 동작인데 도메인마다 이름이 다름)도 일관성이 없다는 신호이니 같이 확인하세요. (실제 사례, 2026-08-03에 둘 다 정리함: ① 검색/지도 필터 적용 이벤트가 `success_filter_apply` 하나를 공유하던 걸 지도 쪽을 `success_map_filter_apply`로 분리, ② "필터 아이콘 클릭"이 홈/검색은 `engage_filter`인데 지도만 다른 이름(`engage_filter_open`)이던 걸 `engage_map_filter_open`으로 통일 — 둘 다 `success_map_area_search`처럼 지도 도메인은 `map_` 접두어를 쓰는 기존 규칙을 따름. 앞으로 새 이벤트를 추가할 때도 이 접두어 규칙을 유지하세요.)
6. 네 그룹으로 나눠 보고: (a) 선택 환경에서 발화 확인, (b) 코드 호출은 있고 프로젝트 어딘가엔 발생 이력이 있지만 선택 환경에서는 90일간 0건, (c) 코드 호출은 있지만 프로젝트 전체에서 한 번도 발화된 적 없음(taxonomy_warnings에 뜬 것), (d) 코드 호출 자체가 없음(있다면 최우선 보고 대상).
7. 사용자가 기획팀 원본 텍소노미 문서(스프레드시트를 복사한 텍스트 등)를 함께 주면, 그 문서와 `analytics.type.ts`도 대조하세요 — 문서에는 있는데 코드엔 없는 이벤트(미구현), 코드에 있는데 문서 갱신이 안 된 케이스(예: 백엔드 전용이라 프론트에서 계측 불가한 이벤트, 아직 UI가 mock이라 제외한 이벤트는 `analytics.type.ts` 파일 상단 주석에 이미 문서화되어 있음)를 구분해서 알려주세요.

## 자동 실행에서

- **필수**: 활성 사용자(DAU·WAU·MAU)·페이지뷰·일별 추이, 전환 퍼널 이벤트 집계(`execute-sql`로 이벤트별 건수·고유 사용자)
- **선택**: 저장된 Insight 12개 재실행. 타임아웃이 반복되거나 시간이 오래 걸리면 생략할 수 있지만 **생략했다고 대시보드와 Discord에 적습니다.**
