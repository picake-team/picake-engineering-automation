---
name: picake-web-performance
description: web-user의 로딩·반응 속도와 검색(SEO) 상태를 점검해 달라는 요청을 받으면 사용합니다. "웹 성능 어때", "사이트 느려졌어?", "로딩 속도 점검해줘", "웹 바이탈 보여줘", "라이트하우스 돌려줘", "SEO 점수 어때" 같은 요청에 이 skill을 참고해 PostHog 웹 바이탈(실사용)과 Lighthouse(PageSpeed Insights)를 함께 조회하고 해석 코멘트를 붙여 보여주세요.
---

# 웹 성능 점검

> **이 문서의 `apps/web-*` 경로는 [`picake-engineering-web`](https://github.com/picake-team/picake-engineering-web) 레포의 것입니다.** 매일 09:00 KST 클라우드 루틴("Picake 웹 성능 점검")이 이 레포를 체크아웃해 5번 **자동 실행 모드**로 실행합니다.

`web-user`(https://picakes.com)가 사용자에게 얼마나 빠른지, 검색엔진이 읽기 좋은 상태인지를 두 도구로 봅니다. 두 도구는 보는 대상이 다릅니다.

| 도구 | 보는 것 | 범위 | 강점 | 약점 |
| --- | --- | --- | --- | --- |
| PostHog 웹 바이탈 | **실제 사용자**가 겪은 로딩·반응 속도 | 표본이 있는 **모든 페이지**(경로별), 기기·지역별 | 진짜 체감에 가까움, 매일 쌓임 | 사용자가 적어 표본이 작음, 원인은 모름 |
| Lighthouse (PageSpeed Insights API) | 구글 서버에서 한 번 재현한 성능·SEO | **대표 페이지 3곳**(홈, 상품 상세, 스토어 상세)만 | 원인(개선 제안)을 알려줌, 표본 걱정 없음 | 돌릴 때마다 값이 흔들림, 구글 서버·가상 기기라 한국 사용자와 다를 수 있음 |

## 0. 범위와 알려진 한계

- 대상은 항상 **운영(production) `web-user`** 입니다. 검증(staging)은 사용자가 거의 없어 표본이 안 나오고 검색 차단 상태라 측정하지 않습니다.
- 측정하지 않는 것: 서버·백엔드 API 응답 시간, 클릭 후 화면 전환 시간, 코드·배포 이력. 필요해 보이면 사용자에게 별도 점검이 필요하다고 안내하되, 이 점검이 본 것처럼 쓰지 마세요.
- 합성(테스트) 접속은 UA·화면 폭으로 걸러내지만(2-1), 개발자가 실제 휴대폰으로 하는 수동 테스트는 걸러지지 않습니다.
- 앱(Flutter WebView)과 브라우저 접속이 섞여 있습니다. 구글의 사용자 데이터(CrUX)는 Chrome 방문만 집계하므로 WebView 사용자를 대표하지 않습니다.

## 1. 측정 기준 (가장 먼저 정하고, 리포트에 그대로 적는다)

**이 점검은 실시간이 아닙니다.** 실행 시점에 한 번 모은 값이고, PostHog는 **완료된 날짜(한국 시간 00:00 경계) 단위**로 자릅니다. 오늘 하루는 아직 끝나지 않았으므로 제외합니다(오늘의 변화는 다음 날 리포트에 반영됩니다).

먼저 기준 날짜를 계산합니다(실행 환경은 리눅스).

```bash
export TZ=Asia/Seoul
TODAY=$(date +%F)                     # 실행일
D1=$(date -d "$TODAY -1 day" +%F)     # 어제 (집계에 포함되는 마지막 날)
D7=$(date -d "$TODAY -7 day" +%F)     # 이번 구간 시작일
D14=$(date -d "$TODAY -14 day" +%F)   # 직전 구간 시작일
NOW_KST=$(date '+%Y-%m-%d %H:%M')     # 점검 시각
```

| 구간 | 범위 (KST 00:00 기준, 시작 포함·끝 제외) | 리포트 표기 예 |
| --- | --- | --- |
| 이번 구간 | `D7` ~ `TODAY` (완료된 7일) | `10/20 ~ 10/26 (7일)` |
| 직전 구간 | `D14` ~ `D7` (그 앞 7일) | `10/13 ~ 10/19 (7일)` |
| 어제 | `D1` ~ `TODAY` | `10/26 (1일)` |

리포트에 반드시 적을 **측정 기준** (대시보드 맨 위 카드와 Discord 필드 모두):

- 실사용(PostHog): 이번 구간·직전 구간·어제의 날짜 범위, "완료된 날짜 기준, 실시간 아님", 봇 제외, 점검 시각(`NOW_KST`).
- Lighthouse: 측정 시각(`NOW_KST`), 모바일 시뮬레이션, 페이지당 1회 측정, 구글 서버에서 실행.
- 구글 사용자 데이터(CrUX): 구글이 집계한 최근 28일(있을 때만).

### 1-1. 최근 운영 배포 이력과 배포 전후 비교 (자동 조회, 실패해도 계속 진행)

날짜 단위 구간만 보면 "이 값이 배포 전 코드의 값인지 후의 값인지" 알 수 없습니다. 웹 레포는 공개라서 **인증 없이** 공개 GitHub API로 운영 배포 이력을 읽을 수 있습니다. 호출이 실패하거나 한도에 걸리면 "배포 이력 확인불가"로 적고 이 단계만 건너뜁니다.

```bash
API=https://api.github.com/repos/picake-team/picake-engineering-web
# 최근 성공한 운영 배포 (created_at은 UTC)
curl -s -m 30 "$API/actions/workflows/deploy-web.yml/runs?per_page=30" \
  | jq -r '.workflow_runs[] | select(.conclusion=="success" and (.head_branch|startswith("web-user/production-"))) | "\(.created_at) \(.head_branch)"' | head -5
# 각 태그의 변경 요약(주석 태그 메시지 첫 줄)
SHA=$(curl -s "$API/git/ref/tags/$TAG" | jq -r '.object.sha')
curl -s "$API/git/tags/$SHA" | jq -r '.message' | head -1
```

- 시각은 KST `MM/DD HH:MM`으로 바꿔 적습니다. **점검 시각 기준 최근 14일의 운영 배포만 최대 5개** 보여 줍니다.
- 이번 구간 안에 운영 배포가 있으면, 구간 값에는 **배포 전 값이 섞여 있음**을 코멘트에 반드시 적습니다.

**배포 전후 비교** — 가장 최근 운영 배포가 14일 이내이고 배포 후 6시간 이상 지났을 때만 합니다.

- `{DEPLOY_UTC}` = 그 배포의 `created_at`(`YYYY-MM-DD HH:MM:SS`, UTC).
- 2-2 전체 p75 쿼리를 두 번 실행합니다. 배포 전 `[DEPLOY_UTC − 7일, DEPLOY_UTC)`, 배포 후 `[DEPLOY_UTC, 지금)`. 기간 조건은 `toDateTime('...', 'UTC')`로 넣고 나머지 필터(합성 접속 제외 포함)는 그대로 둡니다.
- 이 비교만 예외로 **날짜 단위가 아니라 "배포 후 ~ 점검 시각"의 실시간 구간**입니다. 리포트에 그렇게 명시합니다.
- 지표별 배포 후 표본이 3건 미만이면 "배포 후 표본 부족, 다음 리포트에서 다시"로 적고 비교하지 않습니다. 10건 미만이면 참고용입니다.
- **관찰과 원인을 구분**합니다. "배포 후 INP 값이 낮다(관찰)"는 쓰되 "배포 때문"이라고 단정하지 않습니다(표본 변동·다른 변화 가능).

## 2. PostHog 웹 바이탈 수집

이 레포의 `.mcp.json`에 `posthog` MCP가 연결되어 있습니다(project id `497749`). `mcp__posthog__exec` 도구로 `execute-sql`을 호출하세요. 이 프로젝트는 development/staging/production과 `web-user`/`web-seller`가 **하나로 공용**이라 아래 필터를 빠뜨리면 숫자가 틀립니다 (배경: [`picake-data-report`](../picake-data-report/SKILL.md) 1번 "환경 필터링").

### 2-1. 필수 필터 (모든 쿼리 공통)

- `properties.environment = 'production'` 그리고 `properties.app = 'web-user'`
- `(properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)`: 봇·크롤러 제외. 운영은 AI·검색 크롤러를 허용하고 있어 봇이 웹 바이탈에 섞일 수 있습니다.
- **합성(테스트) 접속 제외** — 아래 두 줄은 모든 쿼리에 이미 들어 있습니다. 빼지 마세요.
  - `$raw_user_agent LIKE '%moto g power (2022)%'`: **이 루틴의 Lighthouse(PageSpeed Insights)가 쓰는 가상 기기**입니다. 봇으로 표시되지 않고 실제 Android Chrome 사용자와 똑같이 집계되며, 매 실행마다 3건(홈·상품·스토어)이 느린 가상 환경 값으로 섞입니다.
  - `$device_type = 'Desktop'` 이면서 `$viewport_width <= 500`: 폭 500px 이하의 데스크톱 브라우저는 개발자 도구의 모바일 에뮬레이션이나 자동화 테스트(Playwright 등)입니다. 실제 데스크톱 사용자의 창이 이렇게 좁은 경우는 사실상 없습니다.
  - 표본이 작은 서비스라 합성 접속 몇 건만으로도 p75가 크게 흔들립니다. 이 규칙을 지켜도 실제 휴대폰으로 하는 개발자 수동 테스트는 걸러지지 않으니, 특정 시각에 몰린 이상한 값은 "내부 테스트 가능성"으로 언급하세요.
- 기간은 1번의 날짜로 자릅니다. 아래 쿼리의 `{START}`·`{END}`·`{D14}`·`{D7}`·`{TODAY}`는 `YYYY-MM-DD 00:00:00` 형태의 실제 날짜로 채워 `toDateTime('...', 'Asia/Seoul')`에 넣습니다.

### 2-2. 전체 p75 (핵심 지표 타일용)

같은 쿼리를 **구간 3번**(이번 구간 `[D7, TODAY)`, 직전 구간 `[D14, D7)`, 어제 `[D1, TODAY)`) 실행합니다.

```sql
SELECT
  countIf(properties.$web_vitals_LCP_value IS NOT NULL) AS lcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_LCP_value), properties.$web_vitals_LCP_value IS NOT NULL)) AS lcp_p75,
  countIf(properties.$web_vitals_FCP_value IS NOT NULL) AS fcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_FCP_value), properties.$web_vitals_FCP_value IS NOT NULL)) AS fcp_p75,
  countIf(properties.$web_vitals_INP_value IS NOT NULL) AS inp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p75,
  countIf(properties.$web_vitals_CLS_value IS NOT NULL) AS cls_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_CLS_value), properties.$web_vitals_CLS_value IS NOT NULL), 3) AS cls_p75
FROM events
WHERE event = '$web_vitals'
  AND properties.environment = 'production'
  AND properties.app = 'web-user'
  AND (properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)
  AND NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%')
  AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
  AND timestamp >= toDateTime('{START}', 'Asia/Seoul')
  AND timestamp < toDateTime('{END}', 'Asia/Seoul')
```

### 2-3. 페이지별 p75 (이번 구간 vs 직전 구간)

경로의 id 부분은 묶어서 봅니다(`/product/<id>` → `/product/*`).

```sql
SELECT
  replaceRegexpOne(properties.$pathname, '^/(product|store|order)/[^/]+', '/\\1/*') AS page,
  if(timestamp >= toDateTime('{D7}', 'Asia/Seoul'), 'this_week', 'prev_week') AS period,
  countIf(properties.$web_vitals_LCP_value IS NOT NULL) AS lcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_LCP_value), properties.$web_vitals_LCP_value IS NOT NULL)) AS lcp_p75,
  countIf(properties.$web_vitals_FCP_value IS NOT NULL) AS fcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_FCP_value), properties.$web_vitals_FCP_value IS NOT NULL)) AS fcp_p75,
  countIf(properties.$web_vitals_INP_value IS NOT NULL) AS inp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p75,
  countIf(properties.$web_vitals_CLS_value IS NOT NULL) AS cls_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_CLS_value), properties.$web_vitals_CLS_value IS NOT NULL), 3) AS cls_p75
FROM events
WHERE event = '$web_vitals'
  AND properties.environment = 'production'
  AND properties.app = 'web-user'
  AND (properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)
  AND NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%')
  AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
  AND timestamp >= toDateTime('{D14}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
GROUP BY page, period
ORDER BY lcp_n DESC
LIMIT 12
```

### 2-4. 기기(OS)별 p75 (이번 구간)

```sql
SELECT
  properties.$os AS os,
  properties.$device_type AS device,
  countIf(properties.$web_vitals_LCP_value IS NOT NULL) AS lcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_LCP_value), properties.$web_vitals_LCP_value IS NOT NULL)) AS lcp_p75,
  countIf(properties.$web_vitals_FCP_value IS NOT NULL) AS fcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_FCP_value), properties.$web_vitals_FCP_value IS NOT NULL)) AS fcp_p75,
  countIf(properties.$web_vitals_INP_value IS NOT NULL) AS inp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p75
FROM events
WHERE event = '$web_vitals'
  AND properties.environment = 'production'
  AND properties.app = 'web-user'
  AND (properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)
  AND NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%')
  AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
  AND timestamp >= toDateTime('{D7}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
GROUP BY os, device
ORDER BY lcp_n DESC
LIMIT 10
```

iOS/Android(모바일)와 데스크톱을 나눠 봅니다. 이 앱의 실사용자는 대부분 모바일이라 모바일 수치가 핵심입니다.

### 2-5. 지역(시·도)별 p75 (이번 구간)

```sql
SELECT
  properties.$geoip_subdivision_1_name AS region,
  countIf(properties.$web_vitals_LCP_value IS NOT NULL) AS lcp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_LCP_value), properties.$web_vitals_LCP_value IS NOT NULL)) AS lcp_p75,
  countIf(properties.$web_vitals_INP_value IS NOT NULL) AS inp_n,
  round(quantileIf(0.75)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p75
FROM events
WHERE event = '$web_vitals'
  AND properties.environment = 'production'
  AND properties.app = 'web-user'
  AND (properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)
  AND NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%')
  AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
  AND timestamp >= toDateTime('{D7}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
GROUP BY region
ORDER BY lcp_n DESC
LIMIT 6
```

지역은 IP로 추정한 값이라 대략적입니다(`null`은 위치 미상). 개인을 식별하는 값이 아니라 집계만 사용합니다.

### 2-6. 읽는 법 (실제 데이터 특성)

- `$web_vitals` 이벤트 하나에는 **그 시점에 준비된 지표만** 들어 있습니다. LCP·FCP·INP·CLS는 서로 다른 이벤트에 있어서 `*_n`(표본 수)이 지표마다 다릅니다. 지표별 표본 수를 따로 봅니다.
- INP는 **사용자가 무언가를 눌러야** 생깁니다. `null`은 "측정할 상호작용이 없었다"는 뜻이지 이상이 아닙니다. 눌렀는데 화면이 안 바뀌고 기다리는 시간도 INP에 잡힙니다.
- LCP·FCP는 페이지를 **새로 열 때만** 잽니다. 앱 안에서 화면을 눌러 이동하는 건 잡히지 않습니다.
- p75는 구글이 정한 Core Web Vitals 평가 기준입니다. PostHog 대시보드는 p90을 써서 숫자가 다를 수 있습니다.
- **배포 직후에는 구간 안에 과거 값이 섞입니다.** 7일 구간에는 개선 배포 이전 날짜도 들어 있어, 개선 효과가 구간 값에 천천히 반영됩니다. 그래서 "어제" 값을 함께 봅니다(어제 표본이 작으면 참고용).

### 2-7. 판정 기준 (Google Core Web Vitals)

| 지표 | 🟢 좋음 | 🟡 개선 필요 | 🔴 나쁨 |
| --- | --- | --- | --- |
| LCP (가장 큰 요소 표시) | ≤ 2,500ms | ≤ 4,000ms | > 4,000ms |
| FCP (첫 화면 표시) | ≤ 1,800ms | ≤ 3,000ms | > 3,000ms |
| INP (반응) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (화면 흔들림) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

### 2-8. 표본이 작을 때의 규칙 (사용자가 적은 서비스라 자주 해당)

- **지표별 표본이 10건 미만이면** 등급은 매기되 "표본 N, 참고용"을 반드시 붙입니다.
- 페이지·기기·지역 **한 구분의 표본이 3건 미만이면** 표에서 제외하고, 제외한 구분이 있으면 "표본 부족으로 제외: …"를 한 줄로 적습니다.
- 지역별은 표본 5건 이상인 지역만 따로 보여 주고, 나머지는 "기타(표본 부족)"로 묶습니다. 한 지역만 남으면 "지역 구분은 표본이 부족해 생략"이라고 한 줄로 적습니다. 억지로 표를 채우지 마세요.

## 3. Lighthouse 수집 (PageSpeed Insights API)

구글이 대신 Lighthouse를 돌려 주는 API입니다. 루틴 프롬프트의 `PSI_API_KEY`가 필요합니다(키 없이는 호출 한도 초과로 거절됩니다). **키가 없거나 호출이 실패하면 이 단계만 건너뛰고** 그 사실을 보고에 적습니다.

### 3-1. 대상 URL (모바일, 3곳)

1. 홈: `https://picakes.com/`
2. 상품 상세: 최신 상품 1개. 상품이 없으면 건너뜁니다.
3. 스토어 상세: 첫 번째 스토어 1개. 스토어가 없으면 건너뜁니다.

```bash
PRODUCT_ID=$(curl -s "https://api.picakes.com/v1/consumer/products?sortBy=latest&page=1&limit=1" \
  | jq -r '.data.data[0].id // empty')
STORE_ID=$(curl -s "https://api.picakes.com/v1/consumer/store?page=1&limit=1" \
  | jq -r '.data.data[0].id // empty')
# 상품: https://picakes.com/product/$PRODUCT_ID  /  스토어: https://picakes.com/store/$STORE_ID
```

대시보드에는 어떤 상품·스토어를 쟀는지 이름 대신 **경로 패턴만**(`/product/*`, `/store/*`) 적습니다.

### 3-2. 호출 (성능 + SEO를 한 번에)

```bash
curl -s -m 120 -G "https://www.googleapis.com/pagespeedonline/v5/runPagespeed" \
  --data-urlencode "url=$TARGET_URL" \
  --data-urlencode "strategy=mobile" \
  --data-urlencode "category=performance" \
  --data-urlencode "category=seo" \
  --data-urlencode "locale=ko" \
  --data-urlencode "key=$PSI_API_KEY" -o psi.json
```

- 한 번에 하나씩 순서대로 호출합니다(호출당 최대 1~2분). 실패하면 1회만 재시도합니다.
- 응답에 `error`가 있으면(예: 한도 초과, 403) 해당 URL은 "확인불가"로 표시하고 이유를 한 줄로 적습니다. 키 값은 로그·결과물에 남기지 않습니다.
- 데스크톱 점검은 하지 않습니다(앱이 모바일 WebView).

### 3-3. 뽑을 값

```bash
jq '{
  performance: (.lighthouseResult.categories.performance.score * 100 | round),
  seo: (.lighthouseResult.categories.seo.score * 100 | round),
  fcp:  .lighthouseResult.audits["first-contentful-paint"].numericValue,
  lcp:  .lighthouseResult.audits["largest-contentful-paint"].numericValue,
  tbt:  .lighthouseResult.audits["total-blocking-time"].numericValue,
  cls:  .lighthouseResult.audits["cumulative-layout-shift"].numericValue,
  ttfb: .lighthouseResult.audits["server-response-time"].numericValue,
  bytes: .lighthouseResult.audits["total-byte-weight"].numericValue,
  opportunities: ([ .lighthouseResult.audits[]
    | select(.details.type == "opportunity" and (.details.overallSavingsMs // 0) > 100)
    | {title, savingsMs: .details.overallSavingsMs} ]
    | sort_by(-.savingsMs) | .[0:3]),
  seoFailed: ([ .lighthouseResult.categories.seo.auditRefs[].id as $id
    | .lighthouseResult.audits[$id]
    | select(.score != null and .score < 1)
    | .title ]),
  crux: ((.loadingExperience // .originLoadingExperience)
    | if . == null or .metrics == null then null else {
        scope: (if .origin_fallback == true then "사이트 전체" else "해당 페이지" end),
        overall: .overall_category,
        lcp: .metrics.LARGEST_CONTENTFUL_PAINT_MS,
        inp: .metrics.INTERACTION_TO_NEXT_PAINT,
        cls: .metrics.CUMULATIVE_LAYOUT_SHIFT_SCORE
      } end)
}' psi.json
```

- `opportunities`: 절감 예상 시간이 큰 성능 개선 제안 상위 3개. 성능이 나쁜 지표의 **원인 후보**로만 연결해 씁니다.
- `seoFailed`: SEO 항목 중 통과하지 못한 것의 제목. 비어 있으면 "SEO 항목 모두 통과"입니다. SEO 점수는 메타 설명·대표 주소(canonical)·크롤 허용 여부 같은 **자동 점검 가능한 기본 항목**만 봅니다. 구조화 데이터의 유효성은 반영되지 않으므로 "SEO 만점 = 검색에 잘 노출된다"로 해석하지 마세요.
- `crux`(구글 실사용 데이터): 방문자가 적으면 **없는 게 정상**이고, 없으면 이 항목은 조용히 건너뜁니다(경고하지 않음). 각 지표의 `category`(`FAST`=좋음, `AVERAGE`=개선 필요, `SLOW`=나쁨)로 판정합니다. CLS `percentile`은 100배 값으로 올 수 있어 확신이 없으면 숫자 대신 `category`만 씁니다. Chrome 방문만 집계하므로 앱(WebView) 사용자와 다릅니다.

## 4. 코멘트 작성 규칙

숫자만 나열하지 말고 사람이 읽을 해석을 붙입니다. 3~6개 bullet로 아래 순서를 따릅니다. **모든 비교는 "어느 기간끼리"인지 날짜로 말합니다**(예: "10/13~10/19 대비").

1. **한 줄 총평**: 전반적으로 괜찮은지 / 신경 쓸 곳이 있는지.
2. **지난 구간 대비 눈에 띄는 변화**: 표본 10건 이상이면서 20% 넘게 변한 지표만. 없으면 "큰 변화 없음". 어제 값이 구간 값과 크게 다르면(표본 허용 범위에서) 개선·악화 추세로 언급.
   - 이번 구간 안에 운영 배포가 있었다면 "어느 날짜 값이 배포 전인지"를 함께 말하고, 1-1의 배포 전후 비교 결과(관찰)를 근거로 씁니다. 배포 직후라 구간 값이 배포 전 값에 끌려가고 있으면 그렇게 설명해 불필요한 경고를 피합니다.
3. **나쁨·개선 필요 지표**: 어느 페이지·기기의 어떤 지표인지, 표본이 작으면 그렇다고 명시. Lighthouse 제안 중 관련 있는 것을 "가능한 원인(추정)"으로 연결.
4. **SEO 상태**: 페이지별 SEO 점수와 미통과 항목. 이전과 달라졌거나 미통과 항목이 있을 때만 길게, 문제없으면 한 줄.
5. **표본·측정 한계**: 판단을 보류한 부분, 건너뛴 항목과 이유.
6. **다음에 볼 것** (선택): 다음 점검에서 확인할 한 가지.

### 두 도구의 결과가 어긋날 때

어느 한쪽을 무조건 믿지 말고 **그때그때 판단**하되, 판단 근거를 한 줄로 적습니다. 아래를 고려하세요.

- **표본 수**: PostHog 표본이 작으면(10건 미만) 한두 사용자의 느린 환경이 p75를 끌어올릴 수 있습니다. 반대로 표본이 충분하면 실사용 값이 더 믿을 만합니다.
- **측정 환경**: Lighthouse는 구글 서버와 가상 모바일(느린 네트워크·CPU)에서 한 번 재현한 값이고 호출마다 5~10점씩 흔들립니다. 한 번의 낮은 점수만으로 문제라고 단정하지 마세요.
- **방향성**: 두 도구가 같은 방향(둘 다 나쁨)이면 확신을 높이고, 어긋나면 "어긋남"을 사실대로 적고 어느 쪽을 더 신뢰하는지와 이유를 씁니다. 판단이 서지 않으면 "판단 보류"라고 씁니다.

### 공통 원칙

- 추측은 "추정"이라고 표시합니다. 이 점검은 코드 변경·배포 이력을 보지 않으므로 "어떤 배포 때문"이라고 단정하지 않습니다.
- 확인하지 못한 것을 확인한 것처럼 쓰지 않습니다(예: Lighthouse를 건너뛰었으면 원인 후보를 만들어 내지 않음).
- 이상이 없으면 "이상 없음"으로 짧게 끝냅니다. 문제를 억지로 찾지 않습니다.
- 개별 사용자를 식별할 수 있는 정보는 쓰지 않습니다(페이지·기기·지역 단위 집계만).

## 5. 자동 실행 모드 (클라우드 루틴)

루틴 프롬프트가 "자동 실행 모드"를 지정하면 위 절차를 아래처럼 바꿔 적용합니다. 다른 루틴(데이터 리포트 등)처럼 **상세 내용은 Artifact 대시보드, Discord에는 요약과 링크**를 보냅니다.

### 5-1. 실행 순서 (반드시 이 순서)

1. 측정 기준 날짜 계산 (1번)
2. PostHog 수집 (2번, **2-2~2-5 쿼리를 모두 실행**합니다. 표본 규칙으로 화면에서 생략하더라도 쿼리는 먼저 실행해 결과를 확인합니다) 그리고 1-1 배포 이력 조회
3. Lighthouse 수집 (3번, 키가 없으면 건너뜀)
4. 코멘트 작성 (4번)
5. **Artifact 대시보드 발행** — `dataviz` skill을 로드한 뒤 Artifact 도구를 호출합니다. 도구가 바로 보이지 않으면 `ToolSearch`로 `select:Artifact`를 먼저 로드하세요(PostHog 도구처럼 지연 로드될 수 있습니다). **이 단계를 생략하지 마세요.**
6. **Discord 전송** — 5번에서 받은 Artifact 링크를 포함해서 보냅니다.
7. 보고

### 5-2. 실패·예외 처리

- **질문하지 않음**: 환경은 항상 production, PostHog 필터는 `environment=production`, `app=web-user`, 봇 제외.
- **PostHog 실패 시**: 도구 호출이 계속 실패하면 "실사용 데이터를 가져오지 못했다"고 알리고 Lighthouse 결과만으로 짧게 보고합니다(color는 주황, 코멘트에 한계 명시).
- **Lighthouse 실패·키 없음**: 해당 항목을 "확인불가 (이유)"로 표시하고 나머지는 정상 진행합니다.
- **푸시 알림은 보내지 않습니다.** 결과 전달은 Discord 하나입니다.

### 5-3. 대시보드(Artifact) 구성

`dataviz` skill을 먼저 로드해서 카드·표·타일 스타일 가이드를 따르고 Artifact(HTML)로 렌더링합니다. title `Picake 웹 성능 점검 · {YYYY-MM-DD}`, icon `chart`. 매번 새 Artifact로 발행해도 됩니다. **모바일(폰)에서 가장 많이 열어 보므로 한 줄 폭 레이아웃이 기본**이고, 표는 카드 안에서 가로 스크롤합니다. 번호는 실제 순서와 맞게 유지하세요.

위에서 아래 순서:

1. **헤더**: 제목, 부제(`운영 web-user · 점검 시각 {NOW_KST} KST`), **종합 상태 배지**(🟢 이상 없음 / 🟠 확인 필요, 5-4의 색 기준과 동일).
2. **측정 기준 카드 (항상 헤더 바로 아래, 접지 않음)**: 1번의 표기를 그대로 사용합니다.
   - 실사용(PostHog): 이번 구간 `MM/DD ~ MM/DD (7일)` · 직전 구간 · 어제 · "완료된 날짜 기준, 실시간 아님" · 봇 제외
   - Lighthouse: 측정 시각 · 모바일 시뮬레이션 · 페이지당 1회 · 구글 서버
   - 구글 사용자 데이터: 최근 28일 (있을 때만)
   - 운영 배포: 최근 14일 안의 배포 최대 3개 `MM/DD HH:MM · 태그 · 한 줄 요약` (확인불가면 그 사실)
   - 합성 접속 제외: "Lighthouse 가상 기기·폭 500px 이하 데스크톱 제외"
3. **핵심 지표 타일 4개 (LCP · FCP · INP · CLS)**: 각 타일에 이번 구간 전체 p75, **등급 칩**(텍스트 `좋음`/`개선 필요`/`나쁨` + 색, 색만으로 구분하지 않음), 직전 구간 대비 증감(▲▼ %), 어제 값, 표본 수. 표본 10건 미만이면 "참고용" 표시.
4. **배포 전후 카드 (조건부)**: 1-1 조건을 만족할 때만. 배포 시각, `배포 전 7일` vs `배포 후 ~ 점검 시각`의 LCP·FCP·INP·CLS p75와 표본 수를 2열로 비교하고 "실시간 구간, 표본 작으면 참고용"을 명시. 조건을 만족하지 않으면 카드를 만들지 않습니다.
5. **코멘트**: 4번 규칙의 bullet를 소제목(총평 / 변화 / 주의 / SEO / 한계 / 다음에 볼 것)으로 나눠 짧게.
6. **페이지별 실사용**: **폰에서 한 화면에 다 보이도록 표 대신 페이지별 카드**로 만듭니다(7열 표는 가로로 잘려 읽기 어렵습니다). 카드 하나 = 페이지 경로 + 2×2 지표 칸(LCP·FCP·INP·CLS, 각각 등급 칩 + 값 + 표본 수) + 한 줄 `직전 구간 LCP ○○ · INP ○○`. 표본이 많은 순으로 정렬하고, 제외한 페이지는 목록 아래 한 줄로 알립니다.
7. **기기별**(OS별 카드 또는 3열 이내 표)과 **지역별**은 `<details>`로 접어 둡니다(표본 규칙 2-8 적용). 폭을 넘는 표가 불가피하면 첫 열을 고정하고 오른쪽에 "→ 스크롤" 안내를 붙입니다.
8. **Lighthouse 카드 3개 (홈 / 상품 상세 / 스토어 상세)**: 성능·SEO 점수, FCP·LCP·TBT·CLS, 개선 제안 상위 3개, SEO 미통과 항목. 페이지를 건너뛰었으면 카드에 "확인불가 (이유)".
9. **구글 실사용 데이터(CrUX)**: 데이터가 있을 때만 카드로 표시하고, 없으면 섹션 자체를 생략합니다.
10. **부록 (`<details>`로 접음)**: 판정 기준표(2-7)와 점검 범위·한계(0번).

스타일 규칙: 숫자는 `tabular-nums`, 표본 수는 흐린 보조 텍스트, 라이트/다크 모두 대응, 외부 라이브러리·이미지 없음, 개별 사용자 정보 없음, 날짜는 모두 `MM/DD` 한국 시간.

### 5-4. Discord 전송

jq로 아래 구조의 payload를 만들어 루틴 프롬프트의 웹훅 URL로 `curl` POST합니다. 숫자만 그날 값으로 채우고 구조·순서는 고정합니다.

```
{
  embeds: [{
    title: "⚡ Picake 웹 성능 점검 · {YYYY-MM-DD}",
    color: (아래 기준으로 초록 5763719 / 주황 15105570),
    description: "1줄 핵심 요약 (예: 실사용 속도 대체로 양호 · 모바일 반응 속도(INP)만 확인 필요)",
    fields: [
      { name: "🗓️ 측정 기준", value: "실사용 {D7 MM/DD}~{D1 MM/DD}(7일, 완료된 날짜 기준·실시간 아님) vs 직전 7일 · 어제 {D1 MM/DD}\nLighthouse {NOW_KST} 1회 측정(모바일)", inline: false },
      { name: "🚀 운영 배포", value: "최근 14일 배포 최대 3줄 `MM/DD HH:MM · v태그 · 요약` + 배포 전후 한 줄(조건 충족 시). 확인불가면 \"배포 이력 확인불가\"", inline: false },
      { name: "⚡ 실사용 속도 · 이번 7일 p75", value: "핵심 페이지 3~4줄. 예) 🟢 홈 · LCP 1.7초 · INP 448ms🟡 (표본 6, 참고용)", inline: false },
      { name: "📱 기기별", value: "iOS · Android 한 줄씩 (표본 부족이면 생략 사유)", inline: false },
      { name: "🔦 Lighthouse · 모바일 (참고용)", value: "홈 성능 N · SEO N / 상품 성능 N · SEO N / 스토어 성능 N · SEO N (확인불가면 이유)", inline: false },
      { name: "💬 코멘트", value: "핵심 2~3줄 (전체는 대시보드)", inline: false },
      { name: "📊 전체 대시보드", value: "[클릭해서 자세히 보기 →](ARTIFACT_URL)", inline: false }
    ],
    footer: { text: "Picake 웹 성능 점검 · 매일 09:00 KST · 실사용=PostHog, 참고=Lighthouse" },
    timestamp: (now | strftime("%Y-%m-%dT%H:%M:%SZ"))
  }]
}
```

- **color 기준**: 아래 중 하나라도 해당하면 주황, 아니면 초록입니다. ① 표본 10건 이상인 지표 중 🔴가 있음 ② 직전 구간 대비 20% 이상 나빠진 지표가 있음(표본 10건 이상) ③ Lighthouse 성능 50 미만 또는 SEO 90 미만 ④ 데이터 수집 실패. 표본이 10건 미만인 🔴는 코멘트에서만 언급하고 색에는 반영하지 않습니다. 빨간색은 쓰지 마세요(장애 알림이 아닙니다).
- 각 field의 `value`는 1024자 이하로 유지합니다(Discord 제한). 길면 표본이 많은 순으로 자릅니다.
- Artifact 링크는 반드시 `[클릭해서 자세히 보기 →](URL)` 마스킹 링크로 단독 필드에 넣습니다.
- **Artifact 도구를 실제로 호출했는데 오류가 났을 때만** 링크 필드를 빼고 footer에 "대시보드 발행 실패"를 적습니다. 호출하지 않았거나 도구를 찾지 못했다면 "발행 실패"라고 쓰지 말고, 호출하지 않은 이유(예: "Artifact 도구를 찾을 수 없음")를 그대로 적습니다. 시도하지 않은 것을 실패한 것처럼 보고하지 마세요.
- HTTP 상태코드가 2xx가 아니면 1회만 재시도하고, 실패하면 에러 내용을 최종 메시지에 남깁니다.
- 웹훅 URL과 `PSI_API_KEY`는 로그·결과물·Artifact에 노출하지 않습니다.

### 5-5. 보고

마지막에 측정 기준 날짜, PostHog 포함 여부, Lighthouse 포함 여부(건너뛴 이유), Artifact 링크, Discord 전송 결과(HTTP 상태코드)를 세션 로그에 짧게 남깁니다. 코드 수정·커밋은 하지 않습니다.

## 6. 수동 요청 모드

사용자가 직접 "웹 성능 어때"라고 물으면 위 1~4번을 그대로 수행하고 결과를 채팅에 정리해서 보여줍니다(측정 기준 날짜를 맨 앞에 적습니다). Discord 전송과 Artifact는 사용자가 요청할 때만 합니다. `PSI_API_KEY`가 없으면 Lighthouse는 건너뛰고 필요하면 키 발급이 필요하다고 안내합니다.
