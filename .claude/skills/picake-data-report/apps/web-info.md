# web-info 모듈 — 사장님 유입 (정보·소개 사이트 + 입점 게이트)

> [`SKILL.md`](../SKILL.md)의 공통 규칙(0번 기간·시간대, 1번 환경 필터·합성 접속 제외, 3번 대시보드 그룹)을 그대로 따릅니다. 이 파일에는 web-info에만 해당하는 사실·지표·쿼리만 둡니다. `apps/web-*` 경로는 `picake-engineering-web` 레포의 것입니다.

| 항목 | 값 |
| --- | --- |
| 대상 | 정보·소개 사이트 `https://info.picakes.com` (`apps/web-info`) + web-user의 입점 게이트 `https://picakes.com/partner` |
| PostHog `app` | `web-info` (게이트만 `web-user`) |
| 수집되는 이벤트 | `$pageview`, `$pageleave`, `$autocapture`(클릭), `$web_vitals`. 커스텀 이벤트(`trackEvent`)는 없습니다. |
| 핵심 질문 | 사장님이 **어디서 와서 소개를 보고, 입점 설문까지 갔는가** |
| 대시보드 위치 | 그룹 C · ⑧ 사장님 유입 |

## 1. 해석에 필요한 사실

- **기본 흐름은 오프라인 QR → 게이트 → 소개 → 설문입니다.** 명함·인쇄물 QR은 `picakes.com/partner`(게이트, web-user)를 가리킵니다. 게이트에는 "입점 사전 수요조사 참여하기"(구글 설문)와 "소개페이지 바로가기" 버튼이 있고, 소개 버튼은 `picakes.com/partner/about` → `info.picakes.com/seller/about`으로 307 리다이렉트됩니다.
- **2026-10-27에 소개 페이지를 web-user(`/partner/about`)에서 web-info(`/seller/about`)로 옮겼습니다.** 그 이전 데이터는 `app = 'web-user'`, `$pathname = '/partner/about'`에 남아 있습니다. 30일 구간에 이전 날짜가 들어가는 동안(2026-11-26 리포트까지)은 두 경로를 **"사장님 소개" 한 페이지로 합산**하고 "10/27 정보 사이트로 이전"이라고 표기합니다. 아래 쿼리의 `-- 이전 전 합산` 줄이 이 규칙이며, 2026-11-27부터는 지워도 됩니다.
- **설문은 외부 구글 폼이라 제출 여부를 알 수 없습니다.** 볼 수 있는 것은 설문 링크 **클릭**까지입니다(`$autocapture`의 `$external_click_url`). 리포트에 "신청", "전환 완료"라고 쓰지 말고 "설문 클릭"이라고 씁니다.
- **DB의 "입점요청"과 다릅니다.** 그룹 A의 `입점요청`(store-entry-requests)은 고객이 아직 입점하지 않은 가게의 등록을 요청한 것이고, 여기의 "입점 설문 클릭"은 사장님이 직접 입점 의향을 남기려 한 것입니다. 둘을 같은 문장에서 비교하거나 합치지 말고, 항상 "입점 설문 클릭"이라고 풀어 씁니다.
- `info.picakes.com`과 `picakes.com`은 같은 상위 도메인이라 PostHog 방문자 식별자가 공유됩니다. 그래서 "게이트를 본 사람이 소개도 봤는지"를 **사람 단위(`person_id`)로** 이을 수 있습니다.
- **게이트를 거쳐 왔는지는 리퍼러로 판단하지 않습니다.** 이전 전에는 게이트 → 소개가 같은 앱 안의 화면 전환이라 리퍼러가 남지 않았습니다. 같은 기간에 게이트도 방문한 사람인지로 판단합니다(4-4).
- QR로 들어온 방문은 리퍼러가 없어 `$direct`로 잡힙니다. 카카오톡 같은 메신저 인앱 브라우저도 대개 `$direct`입니다. 그래서 "직접"이 많은 것은 정상입니다.
- 게이트(`/partner`) 방문은 web-user 리포트(그룹 B)의 방문자 수에도 포함돼 있습니다. 그룹 B와 이 섹션의 숫자를 더하지 마세요.

## 2. 페이지 목록은 하드코딩하지 않습니다

web-info에는 페이지가 계속 추가될 수 있습니다(예: 고객용 `/customer/*`, 기능 안내, 공지). 페이지 이름을 이 문서에 고정하지 않고 매번 아래에서 읽습니다.

- **운영 중인 페이지 = 운영 sitemap**
  ```bash
  curl -s -m 20 https://info.picakes.com/sitemap.xml \
    | grep -o '<loc>[^<]*</loc>' | sed -E 's#</?loc>##g; s#^https://info\.picakes\.com##'
  ```
  실패하면 "sitemap 확인불가"라고 적고 PostHog 데이터에 나온 경로를 그대로 씁니다.
- **대상 묶음 = 경로의 첫 단계**: `/seller/*` = 사장님, `/customer/*` = 고객, 그 외 = 기타. 데이터에 새 첫 단계가 나오면 그 이름으로 묶음을 추가합니다. 사장님 입점 흐름(4-4)은 `/seller/*`만 봅니다.
- **sitemap에 없는 경로**(루트 `/`, `/seller`처럼 페이지가 없는 주소)로 들어온 방문은 "없는 주소 방문 N건"으로 한 줄만 적습니다. 옛 링크나 오타를 찾는 용도입니다.

## 3. 공통 조건 (이 모듈의 모든 쿼리)

```sql
properties.environment = 'production'
AND NOT (properties.$raw_user_agent LIKE '%moto g power (2022)%')
AND NOT (properties.$device_type = 'Desktop' AND toFloat(properties.$viewport_width) <= 500)
```

웹 성능 점검 루틴의 Lighthouse는 web-info 페이지도 잽니다. 위의 `moto g power` 조건이 그 접속을 거르므로 빼지 마세요. 아래 쿼리에서는 `<공통 조건>`으로 줄여 씁니다.

## 4. 지표와 쿼리

`{START}`/`{END}`는 SKILL.md 0번 구간(어제·최근 7일·최근 30일)의 `YYYY-MM-DD 00:00:00`, `{D7}`·`{D30}`·`{TODAY}`도 같은 규칙입니다. 모두 `toDateTime('...', 'Asia/Seoul')`로 넣습니다.

### 4-1. 소개 사이트 방문 (어제 / 최근 7일 / 최근 30일)

같은 쿼리를 구간 3번 실행합니다.

```sql
SELECT
  uniq(person_id) AS visitors,
  count() AS pageviews
FROM events
WHERE event = '$pageview'
  AND timestamp >= toDateTime('{START}', 'Asia/Seoul')
  AND timestamp < toDateTime('{END}', 'Asia/Seoul')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))  -- 이전 전 합산
```

### 4-2. 페이지별 (최근 7일 · 최근 30일)

```sql
SELECT
  if(properties.app = 'web-user', '/seller/about', properties.$pathname) AS page,  -- 이전 전 합산
  uniqIf(person_id, timestamp >= toDateTime('{D7}', 'Asia/Seoul')) AS visitors_7d,
  countIf(timestamp >= toDateTime('{D7}', 'Asia/Seoul')) AS pageviews_7d,
  uniq(person_id) AS visitors_30d,
  count() AS pageviews_30d
FROM events
WHERE event = '$pageview'
  AND timestamp >= toDateTime('{D30}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))  -- 이전 전 합산
GROUP BY page
ORDER BY visitors_30d DESC
LIMIT 30
```

결과를 2번의 sitemap 목록과 대조해 묶음(사장님 / 고객 / 기타 / 없는 주소)으로 나눕니다. 경로는 쿼리스트링 없이 `$pathname`만 씁니다.

### 4-3. 유입 경로 (최근 30일)

```sql
SELECT
  multiIf(
    properties.$referring_domain IN ('picakes.com', 'www.picakes.com'), '서비스·게이트',
    properties.$referring_domain = 'info.picakes.com', 'info 내부 이동',
    properties.$referring_domain = '$direct' OR properties.$referring_domain IS NULL, '직접(QR·메신저·주소 입력)',
    match(properties.$referring_domain, '(^|\\.)(google|naver|daum|bing|yahoo)\\.'), '검색',
    properties.$referring_domain
  ) AS source,
  uniq(person_id) AS visitors
FROM events
WHERE event = '$pageview'
  AND timestamp >= toDateTime('{D30}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))  -- 이전 전 합산
GROUP BY source
ORDER BY visitors DESC
LIMIT 8
```

- `info 내부 이동`은 표시하지 않습니다(페이지 사이 이동이지 유입이 아님).
- 도메인 이름은 그대로 보여도 됩니다. `$referrer` 전체 URL은 쓰지 않습니다(쿼리스트링에 개인정보가 있을 수 있음).
- 1번 설명대로 "서비스·게이트"는 실제보다 적게 잡힙니다. 게이트 경유 규모는 4-4의 "게이트도 방문" 숫자로 말합니다.

### 4-4. 사장님 입점 흐름 (최근 7일 · 최근 30일, 사람 단위)

같은 쿼리를 7일·30일로 두 번 실행합니다. 서브쿼리의 기간도 바깥과 같게 바꿉니다.

```sql
SELECT
  uniqIf(person_id, event = '$pageview'
    AND properties.app = 'web-user' AND properties.$pathname = '/partner') AS gate_visitors,
  uniqIf(person_id, event = '$pageview'
    AND ((properties.app = 'web-info' AND properties.$pathname LIKE '/seller/%')
      OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))) AS about_visitors,  -- 이전 전 합산
  uniqIf(person_id, event = '$pageview'
    AND ((properties.app = 'web-info' AND properties.$pathname LIKE '/seller/%')
      OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))
    AND person_id IN (
      SELECT person_id FROM events
      WHERE event = '$pageview' AND properties.app = 'web-user' AND properties.$pathname = '/partner'
        AND properties.environment = 'production'
        AND timestamp >= toDateTime('{START}', 'Asia/Seoul') AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
    )) AS about_also_gate,
  uniqIf(person_id, event = '$autocapture'
    AND properties.$external_click_url LIKE 'https://docs.google.com/forms/%') AS survey_clickers,
  uniqIf(person_id, event = '$autocapture'
    AND properties.$external_click_url LIKE 'https://docs.google.com/forms/%'
    AND properties.app = 'web-user' AND properties.$pathname = '/partner') AS survey_from_gate,
  uniqIf(person_id, event = '$autocapture'
    AND properties.$external_click_url LIKE 'https://docs.google.com/forms/%'
    AND (properties.app = 'web-info'
      OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))) AS survey_from_about  -- 이전 전 합산
FROM events
WHERE timestamp >= toDateTime('{START}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
  AND event IN ('$pageview', '$autocapture')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname IN ('/partner', '/partner/about')))
```

- 이 숫자는 엄격한 순서 퍼널이 아닙니다. 소개 방문자는 게이트 없이 검색·공유 링크로도 오므로 게이트 방문자보다 많을 수 있습니다. "게이트 N명 → 소개 N명(그중 게이트도 방문 N명) → 설문 클릭 N명(게이트에서 바로 N · 소개에서 N)" 형태로 씁니다.
- 게이트에서 설문을 바로 누르는 사람도 있으므로, 설문 클릭은 두 곳을 모두 셉니다. 한 사람이 양쪽에서 누르면 합계는 1명입니다.

### 4-5. 클릭 (최근 30일, 선택)

**외부 링크 목적지**

```sql
SELECT
  multiIf(
    properties.$external_click_url LIKE 'https://docs.google.com/forms/%', '입점 설문',
    properties.$external_click_url LIKE 'https://seller.picakes.com%', '판매자 페이지',
    match(properties.$external_click_url, '^https://(www\\.)?picakes\\.com/app'), '앱 다운로드',
    domain(properties.$external_click_url)
  ) AS destination,
  uniq(person_id) AS people,
  count() AS clicks
FROM events
WHERE event = '$autocapture'
  AND properties.$external_click_url IS NOT NULL
  AND timestamp >= toDateTime('{D30}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname IN ('/partner', '/partner/about')))
GROUP BY destination
ORDER BY people DESC
```

새 목적지는 도메인 이름으로 자동으로 나옵니다. 자주 보이는 도메인이 생기면 위 `multiIf`에 이름을 붙여 추가하세요.

**페이지 안에서 많이 누른 것** (FAQ 질문 등, 관심사 파악용)

```sql
SELECT
  properties.$el_text AS label,
  uniq(person_id) AS people
FROM events
WHERE event = '$autocapture'
  AND properties.$external_click_url IS NULL
  AND properties.$el_text IS NOT NULL
  AND timestamp >= toDateTime('{D30}', 'Asia/Seoul')
  AND timestamp < toDateTime('{TODAY}', 'Asia/Seoul')
  AND <공통 조건>
  AND (properties.app = 'web-info'
       OR (properties.app = 'web-user' AND properties.$pathname = '/partner/about'))  -- 이전 전 합산
GROUP BY label
ORDER BY people DESC
LIMIT 5
```

`$el_text`는 버튼·질문 제목 같은 **화면 문구**입니다. 이메일·전화번호처럼 보이는 값이 섞여 있으면 그 행은 표시하지 않습니다.

### 4-6. 기기 (최근 30일, 선택)

`$device_type`(Mobile / Desktop / Tablet)별 고유 방문자. 방문자 10명 미만이면 생략합니다.

## 5. 표본이 작을 때 (당분간 거의 항상 해당)

- **방문자 10명 미만**이면 비율(%)·증감률을 쓰지 않고 "N명 중 M명"으로만 씁니다.
- **0명이어도 섹션을 숨기지 않습니다.** "최근 7일 방문 없음 (30일 N명)"처럼 한 줄로 남깁니다. 0도 정보입니다.
- **배포일 근처의 소수 방문**은 팀의 배포 확인일 수 있습니다. 예: 2026-10-27(이전 배포일)의 방문은 "배포 확인 방문 포함 가능(추정)"이라고 적습니다. 걸러낼 근거가 없으니 숫자에서 빼지는 않습니다.
- 어제와 7일 평균 비교, 직전 30일 대비 증감은 30일 방문자가 30명 이상이 된 뒤에만 씁니다.

## 6. 출력

### 대시보드 (그룹 C · ⑧ 사장님 유입)

위에서 아래 순서입니다.

1. **그룹 머리**: `C. 사장님 유입 · PostHog (정보 사이트 web-info + 입점 게이트)`
2. **타일 3개**: 소개 사이트 방문자(어제 · 7일 · 30일), 설문 클릭 인원(7일 · 30일), 게이트 방문자(7일 · 30일)
3. **입점 흐름 카드**: 게이트 → 소개(그중 게이트도 방문) → 설문 클릭의 3단을 사람 수로(30일 기준, 7일 병기). "게이트에서 바로 N · 소개에서 N"을 아래에 적습니다. 단계 사이 비율은 5번 규칙을 따릅니다.
4. **페이지별**: 2번 묶음(사장님 / 고객 / 기타)마다 카드 하나, 카드 안에 페이지 행(7일·30일 방문자·페이지뷰). 없는 주소 방문은 맨 아래 한 줄. 이전 합산 기간에는 "10/27 정보 사이트로 이전(이전 전 /partner/about 합산)"을 표시합니다.
5. **유입 경로** (30일)
6. **클릭 목적지와 많이 누른 것** — `<details>`로 접어 둡니다.
7. **한 줄 메모**: "설문 제출은 외부 구글 폼이라 클릭까지만 집계"

### Discord (`🧑‍🍳 사장님 유입` 필드 1개)

`소개 방문 7일 N명 · 30일 N명 / 설문 클릭 7일 N명 · 30일 N명 / 게이트 7일 N명`. 최근 7일 방문이 0이면 `최근 7일 방문 없음 (30일 소개 N명 · 설문 클릭 N명)`.

### 특이사항(`🔍 추이·특이사항`)으로 올릴 것

- 어제 설문 클릭이 1명 이상 있었을 때 (사장님 쪽 가장 중요한 신호)
- 없는 주소 방문이 생겼을 때 (옛 링크·오타 가능)
- sitemap에 새 페이지가 생겼을 때 ("새 페이지 집계 시작: /경로")
- 30일 방문자 30명 이상이 된 뒤의 큰 증감 (5번 규칙)

## 7. 자동 실행에서

- **필수**: 2번 sitemap 조회, 4-1(어제·7일·30일), 4-2, 4-4(7일·30일)
- **선택**: 4-3, 4-5, 4-6. 타임아웃이 반복되면 생략할 수 있지만 생략했다고 대시보드에 적습니다.
- 쿼리는 SKILL.md 규칙대로 **한 번에 최대 2개씩** 보냅니다.
