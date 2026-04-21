---
title: "[인턴] DevOps 주간 미팅을 위한 SRE 대시보드를 IaC로 만든 기록: S3 정적 웹 + Lambda 2단 파이프라인"
excerpt: "여기저기 흩어져 있던 관측 지표를 하나의 정적 웹 대시보드로 모으기 위해, Pulumi로 S3 + Lambda(collector/builder) + EventBridge 구조를 구성한 과정을 정리"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, SRE, Pulumi, IaC, Lambda, S3, EventBridge, 대시보드]

permalink: /intern/company/blitz-dynamics/29/

toc: true
toc_sticky: true

date: 2026-04-21
last_modified_at: 2026-04-21
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/28/)에서는 프로덕션 백엔드가 하루에 세 번 같은 패턴으로 죽었던 사건을 추적했습니다. 그 과정에서 "로그는 CloudWatch, 지표는 CloudWatch, 일부 비즈니스 수치는 백엔드 GraphQL, 미해결 요청 현황은 또 다른 페이지"라는 식으로 **관측 데이터가 여기저기 흩어져 있다**는 문제가 계속 걸렸습니다.

DevOps 팀은 매주 SRE 관점에서 서비스 상태를 리뷰하는 주간 미팅을 진행하는데, 매번 사람이 4~5개 탭을 돌면서 수치를 긁어오고 있었습니다. 미팅마다 수치의 기준 시간/단위가 조금씩 다르기도 했고, "지난주랑 비교하면 어때?"라는 질문이 나올 때마다 쿼리를 다시 돌려야 했습니다.

그래서 이번에는 **주간 미팅용 SRE 대시보드**를 별도의 작은 서비스로 만들었습니다. 요구사항은 단순했습니다.

- 매 시간 자동으로 여러 데이터 소스에서 데이터를 가져와 저장한다.
- 매 시간 집계를 다시 계산해서 정적 JSON으로 떨어뜨린다.
- 브라우저는 **정적 JSON만** 읽는다. 외부 API를 직접 호출하지 않는다.
- 지난 1시간(`current`)과 지난주(`last_week`)를 항상 같은 포맷으로 볼 수 있어야 한다.

이 글은 그 구조를 Pulumi(IaC)로 어떻게 구성했는지에 대한 기록입니다.

> ⚠️ **민감 정보 마스킹**: 사내 클러스터/서비스명, GraphQL 엔드포인트, 구체적인 비즈니스 데이터 소스 명칭 등은 일반화하거나 `***`로 마스킹했습니다. 아키텍처와 모듈 구조 자체는 실제 배포본과 동일합니다.

---

## 1. 왜 "정적 웹 + 2단 Lambda" 구조인가

처음에는 BI 툴이나 Grafana를 얹는 것도 검토했는데, 요구사항이 아래 세 가지였기 때문에 과하게 느껴졌습니다.

- **읽는 사람은 주간 미팅 참석자 몇 명**이다. 실시간 쿼리가 필요한 게 아니다.
- 데이터 소스가 **이미 CloudWatch와 내부 GraphQL 두 군데로 정해져 있다**. 새 백엔드를 두는 게 아니라 긁어오는 게 목적이다.
- 사내 데이터가 브라우저에서 직접 API를 치는 구조는 **피하고 싶었다**. 인증 토큰을 프론트에 실어야 하고, CORS/레이트리밋도 걸림돌.

그래서 아래 구조로 정리했습니다.

```
EventBridge ──► collector Lambda ──► S3(raw)  : 원본 스냅샷 저장
                                       │
EventBridge ──► builder Lambda   ──► S3(web)  : 집계 JSON + 정적 에셋
                                       │
                                    브라우저   : 정적 JSON만 GET
```

핵심 아이디어는 두 가지입니다.

1. **수집(collector)과 집계(builder)를 Lambda 두 개로 분리**한다. 한쪽이 실패해도 다른 쪽은 독립적으로 재시도할 수 있고, 나중에 집계 로직을 바꿔도 raw 데이터는 그대로 남아서 과거 구간을 다시 계산할 수 있다.
2. **브라우저가 보는 것은 오직 S3에 올려둔 정적 JSON**이다. 내부 API 인증 토큰이 브라우저에 실릴 일이 없고, 페이지는 CDN/캐시 친화적이다.

결과적으로 "주간 미팅 직전에 브라우저로 URL 하나만 열면, 최신 집계가 이미 올려져 있는" 형태가 됩니다.

---

## 2. 전체 모듈 구성

Pulumi 레포 안에 `infra/src/sre-dashboard/`라는 모듈 하나로 격리했습니다. 나머지 인프라와 엮이지 않고, **토글 하나로 켜고 끌 수 있어야** 했기 때문입니다.

| 경로 | 역할 |
|---|---|
| `index.ts` | `createSreDashboard()` — 전체 리소스를 순서대로 생성하는 엔트리 |
| `bucket.ts` | raw/web 두 개 S3 버킷 생성. web은 정적 호스팅, raw는 프라이빗 |
| `iam.ts` | collector/builder가 공용으로 쓰는 Lambda 실행 Role |
| `collector/` | collector Lambda 리소스 + Python 런타임 코드 아카이브 |
| `builder/` | builder Lambda 리소스 + Python 런타임 코드 아카이브 |
| `schedule.ts` | EventBridge 스케줄 및 Lambda invoke 권한 |
| `alarms.ts` | collector/builder 에러 알람 |
| `types.ts` | 모듈 입출력 타입 정의 |
| `web/` | 정적 웹 에셋 원본 (`index.html`, `app.js`, `styles.css`) |

엔트리 함수는 대략 이런 모양입니다.

```ts
// infra/src/sre-dashboard/index.ts
export function createSreDashboard(args: SreDashboardArgs): SreDashboardOutputs | undefined {
  if (!args.settings.enableSreDashboard) {
    return undefined; // 토글 off면 리소스를 아예 생성하지 않는다.
  }

  const { rawBucket, webBucket } = createBuckets(args);
  const role = createLambdaRole(args, { rawBucket, webBucket });
  const collector = createCollectorLambda(args, { role, rawBucket });
  const builder = createBuilderLambda(args, { role, rawBucket, webBucket });

  createSchedule(args, { collector, builder });
  createAlarms(args, { collector, builder });

  return { rawBucket, webBucket, collector, builder };
}
```

`enabled === false`면 `undefined`를 반환하도록 한 이유는, 스테이징/프로덕션 스택에서 **"지금은 안 쓴다"는 상태를 config 한 줄로 표현**하고 싶었기 때문입니다. 구성 키를 지우는 게 아니라 토글만 내려두는 식입니다.

---

## 3. 데이터 흐름 상세

### 3-1. collector Lambda: 원본을 그대로 S3에 떨어뜨린다

collector는 매시 `cron(5 * * * ? *)`에 깨어나 **직전 1시간 구간**(`[H-1, H)`)의 데이터를 긁어옵니다.

```python
# collector/runtime/handler.py (요지)
def handler(event, context):
    window = resolve_time_window(event)  # EventBridge time 기반. 수동 호출 시 start/end override
    run_id = make_run_id(window)

    results = {}
    for source in SOURCES:  # cloudwatch / business_metric_a / business_metric_b
        try:
            payload = source.collect(window)
            key = s3_key_for(source, window)
            put_gzipped_json(RAW_BUCKET, key, payload)
            results[source.name] = {"ok": True, "key": key}
        except Exception as e:
            results[source.name] = {"ok": False, "error": str(e)}

    put_manifest(RAW_BUCKET, run_id, window, results)

    if any(not r["ok"] for r in results.values()):
        raise RuntimeError(f"partial failure: {results}")
```

두 가지 설계 선택이 중요했습니다.

1. **소스별 실패를 독립 처리**한다. CloudWatch가 일시적으로 throttling되어도 다른 소스의 수집은 그대로 저장되도록, `try/except`로 분리해 각자의 결과를 기록한다.
2. **하나라도 실패하면 마지막에 `RuntimeError`를 던진다.** EventBridge의 재시도 정책(`maximumRetryAttempts`)이 이 Lambda를 다시 호출하도록 하기 위함이다. 성공한 소스는 이미 저장되어 있으므로, 다음 호출에서는 실패한 소스만 다시 시도되면 된다(멱등 키 덕분에 덮어쓰기 안전).

raw 버킷의 키 구조는 아래처럼 **파티션 친화적**으로 잡았습니다.

```
raw/source=cloudwatch/yyyy=2026/mm=04/dd=21/hh=14.json.gz
raw/source=business_metric_a/yyyy=2026/mm=04/dd=21/hh=14.json.gz
raw/source=business_metric_b/yyyy=2026/mm=04/dd=21/hh=14.json.gz
runs/yyyy=2026/mm=04/dd=21/hh=14.json
```

나중에 Athena를 붙이거나, 특정 시간대의 원본을 사람이 손으로 찾아보기 좋도록 `yyyy=/mm=/dd=/hh=` 포맷으로 유지했습니다. `runs/` 아래에는 **그 시각의 실행 결과 매니페스트**(어떤 소스가 성공/실패했는지)를 함께 남깁니다. 나중에 "이 시간 왜 비었지"를 역추적할 때 이게 가장 빠른 단서가 됩니다.

### 3-2. builder Lambda: 집계 JSON + 정적 에셋을 web 버킷에 쓴다

builder는 매시 `cron(10 * * * ? *)`에 깨어납니다. collector가 5분에 돌고 builder가 10분에 도는 이유는 단순합니다. **collector가 그 시간대의 원본을 다 쓴 뒤에** builder가 읽도록, 여유 5분을 준 것입니다.

builder는 두 기간을 각각 계산해서 두 개의 JSON을 만듭니다.

- `data/current.json` — 최근 1시간
- `data/last_week.json` — KST 기준 이전 ISO week 전체

```python
# builder/runtime/handler.py (요지)
def handler(event, context):
    now = resolve_now(event)

    current = aggregate(range_for_current(now))
    last_week = aggregate(range_for_last_iso_week_kst(now))

    put_json(WEB_BUCKET, "data/current.json", current)
    put_json(WEB_BUCKET, "data/last_week.json", last_week)

    # 정적 웹 에셋도 매 실행마다 동기화 — 이미지에 포함된 html/js/css를 기준으로 덮어쓴다.
    sync_static_assets(WEB_BUCKET, ["index.html", "app.js", "styles.css"])
```

`last_week`를 KST 기준 ISO week로 잡은 건 주간 미팅 스케줄과 맞추기 위해서였습니다. UTC로 잘라버리면 월요일 오전 미팅에서 "지난주"가 직관과 어긋났기 때문입니다.

집계 결과 JSON은 대략 이런 모양입니다(스키마는 축약).

```json
{
  "generated_at": "2026-04-21T05:10:00Z",
  "window": { "start": "...", "end": "...", "label": "current" },
  "summary": {
    "error_rate_pct": 0.42,
    "p95_latency_ms": 312,
    "open_requests": 17
  },
  "panels": {
    "ecs": { "cpu": [...], "memory": [...] },
    "alb":  { "target_5xx": [...], "rt_p95": [...] },
    "business_metric_a": { "recent": [...] },
    "business_metric_b": { "open": [...] }
  },
  "freshness": {
    "missing_source_hours": 0
  }
}
```

`freshness.missing_source_hours`를 스키마 최상단에 둔 이유는, **"이 대시보드가 지금 얼마나 믿을 만한가"** 를 미팅 참석자가 한눈에 보게 만들기 위해서였습니다. raw에 빠진 시간이 많으면 이 숫자가 올라가고, 페이지 상단에 경고 배너가 뜨도록 했습니다.

### 3-3. 브라우저는 정적 JSON만 읽는다

페이지 쪽 로직은 본질적으로 "fetch → render"가 전부입니다.

```js
// web/app.js (요지)
async function loadPanel(which) {
  const res = await fetch(`./data/${which}.json`, { cache: "no-cache" });
  if (!res.ok) {
    renderError(`failed to fetch ./data/${which}.json`);
    return;
  }
  const data = await res.json();
  renderSummary(data.summary);
  renderPanels(data.panels);
  renderFreshness(data.freshness);
}
```

이 단순함이 핵심이었습니다. 브라우저는 **인증된 API를 모른다**. 토큰도 가지지 않는다. 그냥 public S3 정적 웹의 JSON을 GET할 뿐이고, 사내 데이터는 collector/builder가 Lambda 실행 Role로 긁어서 이미 가공/익명화해둔 상태입니다.

---

## 4. 스케줄 / 권한 / 알람

### EventBridge 스케줄

```ts
// schedule.ts (요지)
new aws.cloudwatch.EventRule("sre-dashboard-collector-schedule", {
  scheduleExpression: args.collectorSchedule, // cron(5 * * * ? *)
});

new aws.cloudwatch.EventRule("sre-dashboard-builder-schedule", {
  scheduleExpression: args.builderSchedule,   // cron(10 * * * ? *)
});
```

각 규칙에는 `maximumRetryAttempts`(기본 2)를 걸어, 일시적인 실패는 EventBridge가 자동으로 재시도하게 했습니다. collector 코드가 부분 실패 시 `RuntimeError`를 던지는 이유가 여기에 있습니다.

### Lambda 실행 Role

하나의 Role을 collector/builder가 공유합니다. 필요한 최소 권한만 붙였습니다.

- `s3:GetObject/PutObject/ListBucket` on raw/web 버킷
- `cloudwatch:GetMetricData`
- `logs:*` on 해당 Lambda 로그 그룹

사내 GraphQL은 SigV4 대상이 아니어서 IAM 권한과 무관하고, **로그인 ID/비밀번호는 Pulumi secret**으로 주입해서 Lambda 환경변수로 내립니다(`--secret` 플래그).

### CloudWatch Alarm

MVP 단계에서는 단순하게 두 개만 걸었습니다.

- `sre-dashboard-collector-errors`: `Errors >= 1` (5분)
- `sre-dashboard-builder-errors`: `Errors >= 1` (5분)

값 자체가 이상한 경우(예: `error_rate_pct`가 튀는 경우)는 이 대시보드의 알람으로 잡지 않기로 했습니다. 이건 어디까지나 **"관측용 리포트 페이지"** 이고, 실제 프로덕션 알람은 별도 시스템이 담당하기 때문입니다. 여기서는 **대시보드가 갱신되지 않는 것** 만 알람의 책임으로 좁혔습니다.

---

## 5. Pulumi 설정 키로 켜고 끄기

모듈 전체가 config 키로 제어되도록 했습니다. 주요 키 몇 개만 옮기면 이렇습니다.

| 설정 키 | 기본값/필수 | 설명 |
|---|---|---|
| `enableSreDashboard` | `false` | 모듈 전체 토글 |
| `sreDashboardCollectorSchedule` | `cron(5 * * * ? *)` | collector 스케줄 |
| `sreDashboardBuilderSchedule` | `cron(10 * * * ? *)` | builder 스케줄 |
| `sreDashboardRawRetentionDays` | `180` | raw/runs 보관 일수 |
| `sreDashboardLambdaMemoryMb` | `512` | Lambda 메모리 |
| `sreDashboardLambdaTimeoutSeconds` | `300` | Lambda 타임아웃 |
| `sreDashboardBackendLoginId` | 필수(secret) | 내부 GraphQL 로그인 ID |
| `sreDashboardBackendLoginPassword` | 필수(secret) | 내부 GraphQL 로그인 비밀번호 |

활성화 자체는 명령 세 줄이면 끝납니다.

```bash
cd infra
pulumi config set blitz-core-infra:enableSreDashboard true
pulumi config set --secret blitz-core-infra:sreDashboardBackendLoginId '***'
pulumi config set --secret blitz-core-infra:sreDashboardBackendLoginPassword '***'
pulumi up
```

배포 직후에는 아래 output들을 확인합니다.

- `sreDashboardRawBucketName`
- `sreDashboardWebBucketName`
- `sreDashboardCollectorFunctionName`
- `sreDashboardBuilderFunctionName`
- `sreDashboardWebsiteEndpoint`

---

## 6. 운영 중 확인하는 포인트

주간 미팅 전에 대시보드가 비어 있으면 미팅이 반쯤 날아갑니다. 그래서 초반 몇 주는 아래 순서로 점검했습니다.

1. 페이지에 `failed to fetch .../data/*.json`이 뜨면 → **builder Lambda 로그**부터 확인 (`/aws/lambda/sre-dashboard-builder`).
2. builder는 성공했는데 숫자가 비어 있으면 → **해당 시간대 `runs/...json` 매니페스트**를 본다. `sources` 필드에서 실패한 소스를 바로 특정할 수 있다.
3. `freshness.missing_source_hours`가 주기적으로 튀면 → collector가 특정 시간대에 반복적으로 실패하는 패턴인지 확인(예: 특정 소스의 외부 API 점검 시간대).
4. 둘 다 성공인데 여전히 이상하면 → raw 오브젝트를 직접 열어보고, 원본 자체가 기대한 스키마인지 확인.

핵심은 **raw에 원본이 항상 남아 있다는 것**입니다. 집계 로직이 틀려도 이미 수집된 과거 데이터를 기반으로 builder만 다시 돌리면 복구됩니다. "수집과 집계를 분리한다"의 값은 이 운영 시점에서 가장 확실하게 돌아왔습니다.

---

## 7. 돌아보며

대시보드 자체는 UI상으로는 단순한 페이지이지만, 만들면서 크게 세 가지를 정리하게 됐습니다.

1. **"관측 데이터를 한 곳에 모은다"는 요구는 BI 툴이 아니라 파이프라인 설계 문제다.** 어디서 어떤 주기로 긁을지, 실패 시 어느 입도로 재시도할지, 언제 집계할지를 먼저 정해야 한다. 툴은 그 다음이다.
2. **정적 JSON + 정적 웹이 주는 단순함의 값이 크다.** 미팅용 페이지처럼 읽는 사람이 한정되고 읽기 전용인 화면은, 굳이 브라우저에서 인증 API를 치지 않고 Lambda가 미리 가공한 JSON만 보여주는 편이 보안/운영 양쪽으로 이득이었다.
3. **수집과 집계의 분리는 수정 비용을 낮춘다.** 집계 스키마를 바꾸거나 패널을 추가할 때마다 외부 API를 다시 때릴 필요가 없다. raw만 있으면 builder 한쪽에서 반복 실험이 가능하다.

다음 글에서는 이 대시보드 위에 얹을 예정인 **"지난주 대비 회귀 탐지 로직"**(builder 단에서 간단한 룰 기반 비교를 넣고, 임계를 넘은 항목만 상단에 띄우는 형태)을 정리할 계획입니다.
