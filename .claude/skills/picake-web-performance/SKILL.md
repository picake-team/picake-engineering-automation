---
name: picake-web-performance
description: web-user의 로딩·반응 속도와 검색(SEO) 상태를 점검해 달라는 요청을 받으면 사용합니다. "웹 성능 어때", "사이트 느려졌어?", "로딩 속도 점검해줘", "웹 바이탈 보여줘", "라이트하우스 돌려줘", "SEO 점수 어때" 같은 요청에 이 skill을 참고해 PostHog 웹 바이탈(실사용)과 Lighthouse(PageSpeed Insights)를 함께 조회하고 해석 코멘트를 붙여 보여주세요.
---

# 웹 성능 점검

> **이 문서의 `apps/web-*` 경로는 [`picake-engineering-web`](https://github.com/picake-team/picake-engineering-web) 레포의 것입니다.** 매일 09:00 KST 클라우드 루틴("Picake 웹 성능 점검")이 이 레포를 체크아웃해 5번 **자동 실행 모드**로 실행합니다.

`web-user`(https://picakes.com)가 사용자에게 얼마나 빠른지, 검색엔진이 읽기 좋은 상태인지를 두 도구로 봅니다. 두 도구는 보는 대상이 다릅니다.

| 도구 | 보는 것 | 강점 | 약점 |
| --- | --- | --- | --- |
| PostHog 웹 바이탈 | **실제 사용자**가 겪은 로딩·반응 속도 | 진짜 체감에 가까움, 매일 쌓임 | 사용자가 적어 표본이 작음, 원인은 모름 |
| Lighthouse (PageSpeed Insights API) | 구글 서버에서 한 번 재현한 성능·SEO | 원인(개선 제안)을 알려줌, 표본 걱정 없음 | 돌릴 때마다 값이 흔들림, 구글 서버·가상 기기 조건이라 한국 사용자와 다를 수 있음 |

## 0. 범위와 알려진 한계

- 대상은 항상 **운영(production) `web-user`** 입니다. 검증(staging)은 사용자가 거의 없어 표본이 안 나오고 검색 차단 상태라 측정하지 않습니다.
- 기간은 **최근 7일**, 비교 대상은 그 직전 7일입니다.
- 측정하지 않는 것: 서버·백엔드 API 응답 시간, 클릭 후 화면 전환 시간, 코드·배포 이력. 필요해 보이면 사용자에게 별도 점검이 필요하다고 안내하되, 이 점검이 본 것처럼 쓰지 마세요.
- 앱(Flutter WebView)과 브라우저 접속이 섞여 있습니다. 구글의 사용자 데이터(CrUX)는 Chrome 방문만 집계하므로 WebView 사용자를 대표하지 않습니다.

## 1. PostHog 웹 바이탈 수집

이 레포의 `.mcp.json`에 `posthog` MCP가 연결되어 있습니다(project id `497749`). `mcp__posthog__exec` 도구로 `execute-sql`을 호출하세요. 이 프로젝트는 development/staging/production과 `web-user`/`web-seller`가 **하나로 공용**이라 아래 필터를 빠뜨리면 숫자가 틀립니다 (배경: [`picake-data-report`](../picake-data-report/SKILL.md) 1번 "환경 필터링").

### 1-1. 필수 필터 (모든 쿼리 공통)

- `properties.environment = 'production'` 그리고 `properties.app = 'web-user'`
- `(properties.$virt_is_bot IS NULL OR properties.$virt_is_bot = false)`: 봇·크롤러 제외. 운영은 AI·검색 크롤러를 허용하고 있어 봇이 웹 바이탈에 섞일 수 있습니다.

### 1-2. 페이지별 p75 (지난 7일 vs 직전 7일)

경로의 id 부분은 묶어서 봅니다(`/product/<id>` → `/product/*`).

```sql
SELECT
  replaceRegexpOne(properties.$pathname, '^/(product|store|order)/[^/]+', '/\\1/*') AS page,
  if(timestamp >= now() - INTERVAL 7 DAY, 'this_week', 'prev_week') AS period,
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
  AND timestamp >= now() - INTERVAL 14 DAY
GROUP BY page, period
ORDER BY lcp_n DESC
LIMIT 12
```

전체(페이지 구분 없이) 값이 필요하면 같은 쿼리에서 `page` 그룹을 빼고 조회하세요.

### 1-3. 기기(OS)별 p75 (지난 7일)

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
  AND timestamp >= now() - INTERVAL 7 DAY
GROUP BY os, device
ORDER BY lcp_n DESC
LIMIT 10
```

iOS/Android(모바일)와 데스크톱을 나눠 봅니다. 이 앱의 실사용자는 대부분 모바일이라 모바일 수치가 핵심입니다.

### 1-4. 지역(시·도)별 p75 (지난 7일)

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
  AND timestamp >= now() - INTERVAL 7 DAY
GROUP BY region
ORDER BY lcp_n DESC
LIMIT 6
```

지역은 IP로 추정한 값이라 대략적입니다(`null`은 위치 미상). 개인을 식별하는 값이 아니라 집계만 사용합니다.

### 1-5. 읽는 법 (실제 데이터 특성)

- `$web_vitals` 이벤트 하나에는 **그 시점에 준비된 지표만** 들어 있습니다. LCP·FCP·INP·CLS는 서로 다른 이벤트에 있어서 `*_n`(표본 수)이 지표마다 다릅니다. 지표별 표본 수를 따로 봅니다.
- INP는 **사용자가 무언가를 눌러야** 생깁니다. `null`은 "측정할 상호작용이 없었다"는 뜻이지 이상이 아닙니다.
- LCP·FCP는 페이지를 **새로 열 때만** 잽니다. 앱 안에서 화면을 눌러 이동하는 건 잡히지 않습니다.
- p75는 구글이 정한 Core Web Vitals 평가 기준입니다. PostHog 대시보드는 p90을 써서 숫자가 다를 수 있습니다.

### 1-6. 판정 기준 (Google Core Web Vitals)

| 지표 | 🟢 좋음 | 🟡 개선 필요 | 🔴 나쁨 |
| --- | --- | --- | --- |
| LCP (가장 큰 요소 표시) | ≤ 2,500ms | ≤ 4,000ms | > 4,000ms |
| FCP (첫 화면 표시) | ≤ 1,800ms | ≤ 3,000ms | > 3,000ms |
| INP (반응) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (화면 흔들림) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

### 1-7. 표본이 작을 때의 규칙 (사용자가 적은 서비스라 자주 해당)

- **지표별 표본이 10건 미만이면** 등급은 매기되 "표본 N, 참고용"을 반드시 붙입니다.
- 페이지·기기·지역 **한 구분의 표본이 3건 미만이면** 표에서 제외합니다.
- 지역별은 표본 5건 이상인 지역만 따로 보여 주고, 나머지는 "기타(표본 부족)"로 묶습니다. 한 지역만 남으면 "지역 구분은 표본이 부족해 생략"이라고 한 줄로 적습니다. 억지로 표를 채우지 마세요.

## 2. Lighthouse 수집 (PageSpeed Insights API)

구글이 대신 Lighthouse를 돌려 주는 API입니다. 루틴 프롬프트의 `PSI_API_KEY`가 필요합니다(키 없이는 호출 한도 초과로 거절됩니다). **키가 없거나 호출이 실패하면 이 단계만 건너뛰고** 그 사실을 보고에 적습니다.

### 2-1. 대상 URL (모바일)

1. 홈: `https://picakes.com/`
2. 상품 상세: 최신 상품 1개의 주소. 상품이 없으면 이 항목만 건너뜁니다.

```bash
PRODUCT_ID=$(curl -s "https://api.picakes.com/v1/consumer/products?sortBy=latest&page=1&limit=1" \
  | jq -r '.data.data[0].id // empty')
```

### 2-2. 호출 (성능 + SEO를 한 번에)

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

### 2-3. 뽑을 값

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
    | .title ])
}' psi.json
```

- `opportunities`: 절감 예상 시간이 큰 성능 개선 제안 상위 3개. 성능이 나쁜 지표의 **원인 후보**로만 연결해 씁니다.
- `seoFailed`: SEO 항목 중 통과하지 못한 것의 제목. 비어 있으면 "SEO 항목 모두 통과"입니다.
- SEO 점수는 메타 설명, 대표 주소(canonical), 크롤 허용 여부, 링크 텍스트 같은 **자동 점검 가능한 기본 항목**만 봅니다. 구조화 데이터의 유효성은 이 점수에 반영되지 않으므로 "SEO 만점 = 검색에 잘 노출된다"로 해석하지 마세요.

### 2-4. 구글 실사용 데이터(CrUX, 있을 때만)

응답의 `loadingExperience`(해당 주소) 또는 `originLoadingExperience`(사이트 전체)에 Chrome 사용자 실측값이 들어 있습니다. 방문자가 적으면 **없는 게 정상**이고, 없으면 이 항목은 조용히 건너뜁니다(경고하지 않음).

```bash
jq '(.loadingExperience // .originLoadingExperience) | if . == null or .metrics == null then null else {
  scope: (if .origin_fallback == true then "사이트 전체" else "해당 페이지" end),
  overall: .overall_category,
  lcp: .metrics.LARGEST_CONTENTFUL_PAINT_MS,
  inp: .metrics.INTERACTION_TO_NEXT_PAINT,
  cls: .metrics.CUMULATIVE_LAYOUT_SHIFT_SCORE
} end' psi.json
```

- 각 지표에는 `percentile`(p75)과 `category`(`FAST`=좋음, `AVERAGE`=개선 필요, `SLOW`=나쁨)가 있습니다. 판정은 `category`를 우선 쓰고, 값이 있으면 PostHog 수치와 나란히 보여 줍니다.
- CLS의 `percentile`은 소수가 아니라 100배 값으로 올 수 있습니다(1보다 훨씬 크면 100으로 나눠 해석). 확신이 없으면 숫자 대신 `category`만 씁니다.
- Chrome 방문만 집계하므로 앱(WebView) 사용자와 다릅니다.

## 3. 코멘트 작성 규칙

숫자만 나열하지 말고 사람이 읽을 해석을 붙입니다. 3~6개 bullet로 아래 순서를 따릅니다.

1. **한 줄 총평**: 전반적으로 괜찮은지 / 신경 쓸 곳이 있는지.
2. **지난주 대비 눈에 띄는 변화**: 표본 10건 이상이면서 20% 넘게 변한 지표만. 없으면 "큰 변화 없음".
3. **나쁨·개선 필요 지표**: 어느 페이지·기기의 어떤 지표인지, 표본이 작으면 그렇다고 명시. Lighthouse 제안 중 관련 있는 것을 "가능한 원인(추정)"으로 연결.
4. **SEO 상태**: SEO 점수와 미통과 항목. 이전과 달라졌거나 미통과 항목이 있을 때만 길게 쓰고, 문제없으면 한 줄.
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

## 4. 수동 요청 모드

사용자가 직접 "웹 성능 어때"라고 물으면 위 1~3번을 그대로 수행하고 결과를 채팅에 정리해서 보여줍니다. Discord 전송과 Artifact는 사용자가 요청할 때만 합니다. `PSI_API_KEY`가 없으면 Lighthouse는 건너뛰고 필요하면 키 발급이 필요하다고 안내합니다.

## 5. 자동 실행 모드 (클라우드 루틴)

루틴 프롬프트가 "자동 실행 모드"를 지정하면 위 절차를 아래처럼 바꿔 적용합니다. 다른 루틴(데이터 리포트 등)처럼 **상세 내용은 Artifact 대시보드, Discord에는 요약과 링크**를 보냅니다.

### 실행 순서 (반드시 이 순서)

1. PostHog 수집 (1번)
2. Lighthouse 수집 (2번, 키가 없으면 건너뜀)
3. 코멘트 작성 (3번)
4. **Artifact 대시보드 발행** — `dataviz` skill을 로드한 뒤 Artifact 도구를 호출합니다. 도구가 바로 보이지 않으면 `ToolSearch`로 `select:Artifact`를 먼저 로드하세요(PostHog 도구처럼 지연 로드될 수 있습니다). **이 단계를 생략하지 마세요.**
5. **Discord 전송** — 4번에서 받은 Artifact 링크를 포함해서 보냅니다. 링크는 Discord 전송보다 먼저 있어야 하므로 4번이 5번보다 앞입니다.
6. 보고

- **질문하지 않음**: 환경은 항상 production, PostHog 필터는 `environment=production`, `app=web-user`, 봇 제외.
- **PostHog 실패 시**: 도구 호출이 계속 실패하면 "실사용 데이터를 가져오지 못했다"고 알리고 Lighthouse 결과만으로 짧게 보고합니다(color는 주황, 코멘트에 한계 명시).
- **Lighthouse 실패·키 없음**: 해당 항목을 "확인불가 (이유)"로 표시하고 나머지는 정상 진행합니다.
- **대시보드**: `dataviz` skill을 먼저 로드해서 카드/표/차트 스타일 가이드를 따르고 Artifact(HTML)로 렌더링합니다. title `Picake 웹 성능 점검 · {YYYY-MM-DD}`, icon `chart`. 매번 새 Artifact로 발행해도 됩니다. 섹션 구성(번호는 실제 순서와 맞게 유지):
  1. 요약과 코멘트 (맨 위, 3번 규칙 전체)
  2. 실사용 속도: 페이지별 표 (LCP·FCP·INP·CLS p75, 등급 이모지, 지난주 대비, 표본 수)
  3. 기기·지역별 (1-3, 1-4. 표본 규칙 적용)
  4. Lighthouse 홈·상품 상세: 성능·SEO 점수, 지표, 개선 제안 3개, SEO 미통과 항목
  5. 구글 실사용 데이터 (있을 때만)
  6. 점검 범위와 한계 (건너뛴 항목, 표본, 측정하지 않는 것)
- **Discord 전송**: jq로 아래 구조의 payload를 만들어 루틴 프롬프트의 웹훅 URL로 `curl` POST합니다. 숫자만 그날 값으로 채우고 구조·순서는 고정합니다.

```
{
  embeds: [{
    title: "⚡ Picake 웹 성능 점검 · {YYYY-MM-DD}",
    color: (아래 기준으로 초록 5763719 / 주황 15105570),
    description: "1줄 핵심 요약 (예: 실사용 속도 대체로 양호 · 모바일 반응 속도(INP)만 확인 필요)",
    fields: [
      { name: "⚡ 실사용 속도 · 지난 7일 p75", value: "핵심 페이지 3~4줄. 예) 🟢 홈 · LCP 1.7초 · INP 528ms🔴 (표본 5)", inline: false },
      { name: "📱 기기별", value: "iOS · Android 한 줄씩 (표본 부족이면 생략 사유)", inline: false },
      { name: "🔦 Lighthouse · 모바일 (참고용)", value: "홈 성능 N · SEO N / 상품 성능 N · SEO N (확인불가면 이유)", inline: false },
      { name: "💬 코멘트", value: "핵심 2~3줄 (전체는 대시보드)", inline: false },
      { name: "📊 전체 대시보드", value: "[클릭해서 자세히 보기 →](ARTIFACT_URL)", inline: false }
    ],
    footer: { text: "Picake 웹 성능 점검 · 매일 09:00 KST · 실사용=PostHog, 참고=Lighthouse" },
    timestamp: (now | strftime("%Y-%m-%dT%H:%M:%SZ"))
  }]
}
```

- **color 기준**: 아래 중 하나라도 해당하면 주황, 아니면 초록입니다. ① 표본 10건 이상인 지표 중 🔴가 있음 ② 지난주 대비 20% 이상 나빠진 지표가 있음(표본 10건 이상) ③ Lighthouse 성능 50 미만 또는 SEO 90 미만 ④ 데이터 수집 실패. 표본이 10건 미만인 🔴는 코멘트에서만 언급하고 색에는 반영하지 않습니다. 빨간색은 쓰지 마세요(장애 알림이 아닙니다).
- 각 field의 `value`는 1024자 이하로 유지합니다(Discord 제한). 길면 표본이 많은 순으로 자릅니다.
- Artifact 링크는 반드시 `[클릭해서 자세히 보기 →](URL)` 마스킹 링크로 단독 필드에 넣습니다.
- **Artifact 도구를 실제로 호출했는데 오류가 났을 때만** 링크 필드를 빼고 footer에 "대시보드 발행 실패"를 적습니다. 호출하지 않았거나 도구를 찾지 못했다면 "발행 실패"라고 쓰지 말고, 호출하지 않은 이유(예: "Artifact 도구를 찾을 수 없음")를 그대로 적습니다. 시도하지 않은 것을 실패한 것처럼 보고하지 마세요.
- HTTP 상태코드가 2xx가 아니면 1회만 재시도하고, 실패하면 에러 내용을 최종 메시지에 남깁니다.
- 웹훅 URL과 `PSI_API_KEY`는 로그·결과물·Artifact에 노출하지 않습니다.
- **보고**: 마지막에 PostHog 포함 여부, Lighthouse 포함 여부(건너뛴 이유), Artifact 링크, Discord 전송 결과(HTTP 상태코드)를 세션 로그에 짧게 남깁니다. 코드 수정·커밋은 하지 않습니다.
