---
title: "[인턴] 보는 사람을 위한 SRE 대시보드 — Pulumi IaC로 만든 Lambda 2단 파이프라인과 정적 웹"
excerpt: "여기저기 흩어진 관측 지표를 비개발자도 읽을 수 있는 한 페이지로 모으기 위해, Pulumi로 S3 + Lambda(collector/builder) + EventBridge 구조를 구성한 과정을 정리"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, SRE, Pulumi, IaC, Lambda, S3, EventBridge, Datadog, 대시보드]

permalink: /intern/company/blitz-dynamics/29/

toc: true
toc_sticky: true

date: 2026-04-21
last_modified_at: 2026-04-23
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/28/)에서는 프로덕션 백엔드가 하루에 세 번 같은 패턴으로 죽었던 사건을 추적했습니다. 그 과정에서 "로그는 CloudWatch, 지표는 CloudWatch/Datadog, 일부 비즈니스 수치는 백엔드 GraphQL, 미해결 요청 현황은 또 다른 페이지"라는 식으로 **관측 데이터가 여기저기 흩어져 있다**는 문제가 계속 걸렸습니다.

DevOps 팀은 매주 SRE 관점에서 서비스 상태를 리뷰하는 주간 미팅을 진행하는데, 매번 사람이 4~5개 탭을 돌면서 수치를 긁어오고 있었습니다. 미팅마다 기준 시간/단위가 조금씩 다르기도 했고, "지난주랑 비교하면 어때?"라는 질문이 나올 때마다 쿼리를 다시 돌려야 했습니다. 게다가 지표 자체가 SRE 친화적인 용어(p95, 5XX rate, `resource_name` …)로 되어 있어 **도메인 담당자(운영/사업팀)가 읽기 어렵다**는 불만도 있었습니다.

그래서 이번에는 **"보는 사람을 위한" SRE 대시보드**를 별도의 작은 서비스로 만들었습니다. 요구사항은 단순했습니다.

- 매 시간 자동으로 여러 데이터 소스에서 값을 당겨서 저장한다.
- 매 시간 집계를 다시 계산해서 정적 JSON으로 떨어뜨린다.
- 브라우저는 **정적 JSON만** 읽는다. 외부 API를 직접 호출하지 않는다.
- 지난 1시간(`current`)과 지난주(`last_week`)를 항상 같은 포맷으로 볼 수 있어야 한다.
- 비개발자도 한눈에 읽을 수 있어야 한다(임계값 판정까지 서버에서 해준다).

이 글은 그 구조를 Pulumi(IaC)로 어떻게 구성했는지에 대한 기록입니다.

> ⚠️ **민감 정보 마스킹**: 사내 클러스터/서비스명, GraphQL 엔드포인트, 구체적인 비즈니스 소스 명칭 등은 일반화하거나 `***`, `<MASKED>` 등으로 마스킹했습니다. 아키텍처와 모듈 구조 자체는 실제 배포본과 동일합니다.

---

## 1. 왜 이렇게 만들었나 (Purpose)

실시간 스트리밍까지 갈 필요는 없었습니다. 우리가 필요했던 건 **1시간 단위로 신뢰할 수 있는 한 페이지**였습니다. 그래서 새로운 관측 플랫폼을 도입하는 대신, **시간당 한 번 배치로 지표를 모아 정적 JSON으로 떨어뜨리고, 정적 웹에서 그리는 대시보드**를 선택했습니다.

의존성이 거의 없고(Lambda 2개 + S3 2개), 장애 영향도가 낮고, 무엇보다 **페이지를 여는 순간 느리거나 터질 시나리오가 구조적으로 제거**됩니다.

처음엔 BI 툴이나 Grafana 얹는 것도 검토했는데 과하게 느껴졌습니다.

- 읽는 사람은 **주간 미팅 참석자 몇 명**이다. 실시간 쿼리가 필요한 게 아니다.
- 데이터 소스가 **CloudWatch, Datadog, 내부 GraphQL로 이미 정해져 있다**. 새 백엔드를 두는 게 아니라 "긁어오는 것"이 목적이다.
- 사내 데이터가 브라우저에서 직접 API를 치는 구조는 **피하고 싶었다**. 인증 토큰을 프론트에 실어야 하고, CORS/레이트리밋도 걸림돌.

---

## 2. 설계 원칙 (Design Principles)

모듈을 만들기 전에 다섯 가지 원칙을 먼저 적어두었습니다. 뒤에서 구현 선택을 정당화할 때 이 기준으로 되돌아옵니다.

1. **Pull-based batch, not push-based streaming**
   서비스 코드에 에이전트/SDK를 추가하지 않는다. 필요한 소스를 Lambda가 주기적으로 "당긴다".
2. **Collector ↔ Builder 분리 (Separation of concerns)**
   `collector`는 **외부 소스 → raw S3 저장**만. `builder`는 **raw S3 → 집계 JSON + 정적 자산**만. 집계 로직이 바뀌어도 원본 수집은 건드리지 않는다(재생성만 다시 돌리면 된다).
3. **Static-first frontend**
   브라우저는 S3 정적 웹호스팅에서 HTML/JS/CSS와 `data/*.json` **두 파일**만 읽는다. API 서버 의존 0.
4. **Fail-fast & observable**
   Collector는 소스 하나라도 실패하면 `RuntimeError`를 던져 EventBridge retry 대상이 된다. 다만 `maximumEventAgeInSeconds = 3600`으로 **1시간 이상 지난 이벤트는 재시도하지 않는다** — 실패가 무한히 누적되지 않도록 bound를 둔다.
5. **Everything as code (Pulumi)**
   S3, IAM, Lambda, EventBridge, CloudWatch Alarm까지 전부 Pulumi로 선언. `enableSreDashboard: false`이면 **리소스 자체가 생성되지 않는다**.

---

## 3. 전체 아키텍처

```
┌─────────────────┐   cron(5 * * * ? *)   ┌───────────────────────┐
│  EventBridge    │ ─────────────────────▶│ collector Lambda      │
└─────────────────┘                       │ (Python 3.12)         │
                                          └───────────┬───────────┘
                                                      │  GetMetricData / GraphQL / Datadog API
                                                      ▼
                            ┌──────────────────────────────────────────┐
                            │  raw S3 bucket (private)                 │
                            │  raw/source=cloudwatch/yyyy=…/…          │
                            │  raw/source=business_a/…                 │
                            │  raw/source=business_b/…                 │
                            │  raw/source=datadog/…                    │
                            │  runs/yyyy=…/…json (manifest)            │
                            └───────────────────┬──────────────────────┘
                                                │
┌─────────────────┐   cron(10 * * * ? *)        ▼
│  EventBridge    │ ──────────────────▶┌──────────────────────────┐
└─────────────────┘                    │ builder Lambda           │
                                       │ (aggregate + static)     │
                                       └──────────────┬───────────┘
                                                      ▼
                            ┌──────────────────────────────────────────┐
                            │  web S3 bucket (Static Website Hosting)  │
                            │  data/current.json                       │
                            │  data/last_week.json                     │
                            │  index.html / app.js / styles.css        │
                            └──────────────────────────────────────────┘
                                                      ▲
                                                      │ HTTP
                                                      │
                                               ┌──────┴──────┐
                                               │   Browser   │
                                               └─────────────┘
```

collector는 매시 5분(`cron(5 * * * ? *)`), builder는 매시 10분(`cron(10 * * * ? *)`)에 도는데, 5분의 간격은 **메트릭 도착 지연을 흡수**하고 **collector 완료를 builder가 기다리도록** 만들기 위함입니다.

---

## 4. 모듈 구성

Pulumi 레포 안에 `infra/src/sre-dashboard/`라는 모듈 하나로 격리했습니다. 나머지 인프라와 엮이지 않고, **토글 하나로 켜고 끌 수 있어야** 했기 때문입니다.

```
infra/src/sre-dashboard/
├── index.ts           # createSreDashboard() 오케스트레이터
├── bucket.ts          # raw/web S3 버킷 + 수명주기 + 정책
├── iam.ts             # collector/builder 공용 Lambda Role
├── schedule.ts        # EventBridge 스케줄 + Lambda invoke 권한
├── alarms.ts          # Lambda 에러 알람
├── types.ts           # 입출력 타입
├── collector/
│   ├── index.ts       # collector Lambda 리소스
│   ├── code.ts        # Python 코드 아카이빙
│   ├── environment.ts # 환경변수 조립
│   └── runtime/       # 실제 Python 런타임 (handler.py, *_fetcher.py, …)
├── builder/
│   ├── index.ts
│   ├── code.ts
│   ├── environment.ts
│   └── runtime/       # aggregate.py, source_loader.py, web_writer.py, …
└── web/               # index.html / app.js / styles.css (정적 자산 원본)
```

핵심은 두 가지입니다.

- **TypeScript(Pulumi) 쪽은 리소스 선언만**. 비즈니스 로직은 전부 Python runtime 디렉토리에.
- `web/`은 빌드 단계 없이 있는 그대로 S3에 업로드됩니다. builder Lambda가 `write_static_assets`로 배포.

---

## 5. 인프라 레이어 (Pulumi / TypeScript)

### 5.1 오케스트레이터

```typescript
export function createSreDashboard(
  params: SreDashboardParams
): SreDashboardResult | undefined {
  if (!settings.sreDashboard.enabled) {
    return undefined;
  }

  const { rawBucket, webBucket, websiteEndpoint } = createSreDashboardBucket();
  const { lambdaRole } = createSreDashboardLambdaRole([
    rawBucket.arn,
    webBucket.arn,
  ]);

  const collector = createSreDashboardCollectorLambda({ rawBucket, lambdaRole });
  const builder = createSreDashboardBuilderLambda({ rawBucket, webBucket, lambdaRole });

  createSreDashboardSchedules({
    collectorLambda: collector.lambda,
    builderLambda: builder.lambda,
  });
  createSreDashboardAlarms({
    collectorLambda: collector.lambda,
    builderLambda: builder.lambda,
    alarmActions: params.alarmActions,
  });

  return {
    rawBucketName: rawBucket.bucket,
    webBucketName: webBucket.bucket,
    collectorFunctionName: collector.lambda.name,
    builderFunctionName: builder.lambda.name,
    websiteEndpoint,
    // …
  };
}
```

`enabled === false`이면 `undefined`를 반환하도록 한 이유는, 스테이징/프로덕션 스택에서 **"지금은 안 쓴다"는 상태를 config 한 줄로 표현**하고 싶었기 때문입니다. 구성 키를 지우는 게 아니라 토글만 내려두는 식입니다.

### 5.2 두 종류의 버킷

- **raw bucket (private)** — `PublicAccessBlock` 전부 `true`, SSE-S3, 수명주기 정책으로 `raw/`, `runs/`, `aggregates/` prefix 자동 만료.
- **web bucket (public)** — `BucketPolicy`로 `s3:GetObject`를 `Principal: "*"`에 허용. 단, ACL 차원은 여전히 차단해서 "버킷 정책을 통한 공개"만 열어둡니다.

```typescript
new aws.s3.BucketPublicAccessBlock(`${resourceName}PublicAccessBlock`, {
  bucket: bucket.id,
  blockPublicAcls: true,
  blockPublicPolicy: false,      // 정책 기반 공개는 허용
  ignorePublicAcls: true,
  restrictPublicBuckets: false,
});
```

→ **민감 데이터는 raw 버킷(프라이빗)에만 저장되도록 builder 로직을 유지**하는 것이 운영상 규칙입니다. web 버킷에 올라가는 JSON은 이미 가공/익명화된 상태여야 합니다.

### 5.3 IAM Role — 최소 권한

`cloudwatch:GetMetricData`와 대상 S3 버킷에 대한 Get/Put/Delete/ListBucket만 허용합니다. Lambda 두 개가 같은 role을 공유합니다.

```typescript
new aws.iam.RolePolicy("sreDashboardLambdaPolicy", {
  role: lambdaRole.id,
  policy: pulumi.jsonStringify({
    Version: "2012-10-17",
    Statement: [
      { Effect: "Allow", Action: ["cloudwatch:GetMetricData"], Resource: "*" },
      { Effect: "Allow", Action: ["s3:ListBucket"], Resource: bucketArns },
      {
        Effect: "Allow",
        Action: ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        Resource: bucketArns.map((arn) => `${arn}/*`),
      },
    ],
  }),
});
```

내부 GraphQL은 SigV4 대상이 아니어서 IAM 권한과 무관하고, **로그인 ID/비밀번호는 Pulumi secret**으로 주입해 Lambda 환경변수로 내립니다.

### 5.4 EventBridge 스케줄

```typescript
new aws.cloudwatch.EventTarget("sreDashboardCollectorTarget", {
  rule: collectorSchedule.name,
  arn: args.collectorLambda.arn,
  retryPolicy: {
    maximumRetryAttempts: settings.sreDashboard.collectorMaxRetry, // 기본 2회
    maximumEventAgeInSeconds: 3600,                                // 1시간 bound
  },
});
```

- collector: 매시 5분(`cron(5 * * * ? *)`) — 메트릭 도착 지연 흡수.
- builder:   매시 10분(`cron(10 * * * ? *)`) — collector 완료를 기다림.

---

## 6. Collector Lambda — 외부 소스를 "당기기"

### 6.1 시간 창(window)

기본은 EventBridge `time`을 기준으로 **직전 1시간 `[H-1, H)`** 을 수집합니다. `event.start / event.end`를 수동으로 넣으면 임의 구간 재수집(백필)도 가능합니다.

```python
def resolve_window(event: dict) -> tuple[datetime, datetime]:
    if event.get("start") and event.get("end"):
        return parse_utc_datetime(event["start"]), parse_utc_datetime(event["end"])

    event_time = (
        parse_utc_datetime(event["time"])
        if isinstance(event.get("time"), str) and event["time"].strip()
        else datetime.now(timezone.utc)
    )
    end = event_time.replace(minute=0, second=0, microsecond=0)
    return end - timedelta(hours=1), end
```

### 6.2 수집 소스 4종

| 소스 | 내용 | S3 저장 경로 |
|---|---|---|
| CloudWatch | `AWS/ECS` CPUUtilization, `AWS/ApplicationELB` RequestCount / 5XX / TargetResponseTime p95·p99 등 | `raw/source=cloudwatch/yyyy=…/mm=…/dd=…/hh=….json.gz` |
| 내부 GraphQL (business_a) | 비즈니스 지표 A 요약 | `raw/source=business_a/…` |
| 내부 GraphQL (business_b) | 개발팀 미해결 요청 목록 | `raw/source=business_b/…` |
| Datadog Spans Analytics Aggregate | 주요 API 2-call probe (pc95/pc99 toplist + count timeseries) | `raw/source=datadog/…` |

그리고 run 단위 매니페스트도 남깁니다.

```
runs/yyyy=…/mm=…/dd=…/hh=….json
```

### 6.3 handler 구조 — 소스별 실패를 manifest에 기록

```python
def lambda_handler(event, context):
    settings = load_settings()
    window_start, window_end = resolve_window(event if isinstance(event, dict) else {})
    sources: dict = {}

    try:
        items = fetch_cloudwatch_items(window_start, window_end, build_metric_queries(settings))
        object_key = write_source_payload(...)
        sources["cloudwatch"] = {"status": "ok", "object_key": object_key, "item_count": len(items)}
    except Exception as error:
        traceback.print_exc()
        sources["cloudwatch"] = {"status": "failed", "error": str(error)}

    # business_a / datadog / business_b 동일 패턴 …

    manifest_key = write_run_manifest(bucket=settings.s3_bucket, start=window_start,
                                      end=window_end, sources=sources)

    failed = [name for name, r in sources.items() if r.get("status") != "ok"]
    if failed:
        raise RuntimeError(f"collector failed for sources {failed} …")

    return {"ok": True, "manifest_key": manifest_key, "sources": sources, ...}
```

설계 의도는 두 가지입니다.

1. **소스별 실패를 독립적으로 S3에 기록**합니다. 어느 하나가 throttling되어도 다른 소스는 저장됩니다.
2. **마지막에 예외를 던져 EventBridge가 재시도**하게 합니다. 성공한 소스는 이미 저장되어 있으므로, 다음 호출에서 실패한 소스만 다시 쓰이면 됩니다(키 덮어쓰기 안전).

raw 버킷 키를 `yyyy=/mm=/dd=/hh=` 파티션 형태로 유지한 건, 나중에 Athena를 붙이거나 사람이 손으로 찾아보기 편하도록 하기 위함입니다. `runs/` 아래의 매니페스트는 "이 시간 왜 비었지?"를 역추적할 때 가장 빠른 단서입니다.

### 6.4 CloudWatch 지표 쿼리 예시

```python
def build_metric_queries(settings: CollectorSettings) -> list[dict]:
    ecs_dimensions = [
        {"Name": "ClusterName", "Value": settings.cluster_name},
        {"Name": "ServiceName", "Value": settings.service_name},
    ]
    alb_dimensions = [
        {"Name": "TargetGroup", "Value": settings.target_group_suffix},
        {"Name": "LoadBalancer", "Value": settings.load_balancer_suffix},
    ]
    return [
        _metric("ecs_cpu_utilization", "AWS/ECS", "CPUUtilization", "Average", 60, ecs_dimensions),
        _metric("alb_request_count",   "AWS/ApplicationELB", "RequestCount", "Sum", 60, alb_dimensions),
        _metric("alb_target_5xx",      "AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "Sum", 60, alb_dimensions),
        _metric("alb_target_response_time_p95", "AWS/ApplicationELB", "TargetResponseTime", "p95", 60, alb_dimensions),
        _metric("alb_target_response_time_p99", "AWS/ApplicationELB", "TargetResponseTime", "p99", 60, alb_dimensions),
        # …
    ]
```

> ALB/TG 차원 suffix는 **현재 운영 중인 ALB/TG에 하드코딩**(예: `app/<MASKED>/<HASH>`)되어 있습니다. 재생성 시 바뀌므로 Pulumi config로 override해야 합니다.

### 6.5 Datadog "2-call probe" 설계

주요 API를 이름으로 지정한 뒤, 두 번의 `/api/v2/spans/analytics/aggregate` 호출로 필요한 수치를 얻습니다.

- call 1: **scalar (toplist)** — `pc95 / pc99 @duration`, `group_by: resource_name (limit 10)`
- call 2: **timeseries 1m** — `count`

```python
TRACKED_RESOURCE_NAMES: tuple[str, ...] = (
    "graphql <OperationA>",
    "graphql <OperationB>",
    # … 주요 도메인 GraphQL operation 다수
    "POST /<masked-rest-endpoint>/:<param>",
)

def _build_tracked_filter_query() -> str:
    resource_or = " OR ".join(f'resource_name:"{name}"' for name in TRACKED_RESOURCE_NAMES)
    return f"env:production service:backend AND ({resource_or})"
```

> Datadog 응답은 요청 body의 `compute[*].id`를 그대로 내려주지 않습니다. 그래서 `compute_labels[index]`의 **순서가 곧 응답 `c0 / c1`의 key**가 됩니다. 이 "순서 매핑"을 코드에 명시적으로 문서화해둔 게 포인트였습니다.

---

## 7. Builder Lambda — raw S3를 집계 JSON으로

### 7.1 두 개의 기간(period)

- `current`: 최근 1시간
- `last_week`: **KST 기준** 이전 ISO 주 전체 (월~일)

`last_week`는 168개의 hourly raw payload를 합쳐 평균/합계를 냅니다. Datadog toplist처럼 **"최신 한 스냅샷만 의미 있는"** 지표는 `current`에만 노출합니다. `last_week`의 평균 근사는 정확도 손실이 있어 **의도적으로 뺐습니다.**

`last_week`를 KST 기준 ISO week로 잡은 건 주간 미팅 스케줄과 맞추기 위해서였습니다. UTC로 잘라버리면 월요일 오전 미팅에서 "지난주"가 직관과 어긋났기 때문입니다.

### 7.2 handler

```python
DATA_KEYS = {"current": "data/current.json", "last_week": "data/last_week.json"}

def lambda_handler(event, context):
    settings = load_settings()
    reference_time    = resolve_reference_time(event if isinstance(event, dict) else {})
    latest_window_end = resolve_latest_window_end(reference_time)
    periods           = resolve_periods(latest_window_end)

    latest_business_b_payload = load_latest_business_b_payload(
        bucket=settings.raw_bucket, latest_window_end=latest_window_end,
    )

    for period in periods:
        payloads, missing = load_period_payloads(settings.raw_bucket, period.start, period.end)
        business_a, _     = load_business_a_payloads(settings.raw_bucket, period.start, period.end)
        datadog, _        = load_datadog_payloads(settings.raw_bucket, period.start, period.end)

        dashboard_payload = build_dashboard_payload(
            period=period,
            payloads=payloads,
            business_a_payloads=business_a,
            business_a_recent_limit=settings.business_a_recent_limit,
            latest_business_b_payload=latest_business_b_payload,
            business_b_recent_limit=settings.business_b_recent_limit,
            datadog_payloads=datadog,
            missing_hours=missing,
            reference_time=reference_time,
            latest_window_end=latest_window_end,
        )
        write_dashboard_json(settings.web_bucket, DATA_KEYS[period.name], dashboard_payload)

    write_static_assets(settings.web_bucket)
```

### 7.3 집계 payload 스키마

대시보드가 읽는 최종 JSON은 아래처럼 단순화되어 있습니다.

```json
{
  "schema_version": 2,
  "generated_at": "2026-04-23T03:10:00Z",
  "display_timezone": "Asia/Seoul",
  "period": { "name": "current", "start": "...", "end": "..." },
  "freshness": {
    "last_successful_window_end": "...",
    "data_delay_minutes": 12,
    "missing_source_hours": 0
  },
  "summary_cards": [
    { "id": "alb_p99_max",      "label": "응답이 가장 느렸을 때 (p99)", "value": 0.84, "unit": "초", "status": "green" },
    { "id": "alb_error_rate",   "label": "오류율 (5XX / 전체 요청)",
      "value": 0.12, "unit": "%",
      "secondary_value": 3, "secondary_unit": "건",
      "status": "green" },
    { "id": "alb_rps_avg",      "label": "평균 요청 수 (RPS)", "value": 42.1, "unit": "req/s", "status": "none" },
    { "id": "ecs_cpu_avg",      "label": "서버 평균 CPU 사용률", "value": 38.4, "unit": "%", "status": "green" }
  ],
  "panels": [
    { "id": "ecs_cpu_line", "kind": "line", "title": "서버 CPU 사용률", "unit": "%",
      "series": [ { "query_id": "ecs_cpu_utilization", "label": "CPU",
                    "points": [ { "timestamp_ms": 1714000000000, "value": 35.2 } ] } ],
      "threshold": 70.0 },
    { "id": "alb_response_time_line", "kind": "line", "title": "서버 응답 시간", "unit": "초",
      "series": [ /* p95, p99 2-series */ ] },
    { "id": "api_latency_ranking", "kind": "ranking", "title": "주요 API 지연 랭킹 (pc95 / pc99)",
      "unit": "ms", "entries": [ /* { resource_name, pc95_ms, pc99_ms } */ ] },
    { "id": "business_a_recent_list", "kind": "list",
      "title": "최근 비즈니스 지표 A", "items": [ /* … */ ] },
    { "id": "business_b_unresolved_list", "kind": "list", "list_variant": "site_time",
      "title": "개발팀 미해결 요청 (최근)", "items": [ /* … */ ] }
  ],
  "data_quality": { "failed_sources": [], "warnings": [] }
}
```

**설계 포인트**

- `summary_cards`의 card는 `value / unit` + 선택적 `secondary_value / secondary_unit` 조합으로 "0.12% / 3건"처럼 **한 줄 병기**를 자연스럽게 지원합니다.
- line panel의 `threshold` 필드가 있으면 차트에 가로 빨간 실선을 그리고, 필요하면 y축 최대값도 threshold를 포함하도록 자동 확장합니다.
- `freshness.data_delay_minutes`, `data_quality.warnings`를 payload에 박아두어 UI가 별도 계산 없이 **"데이터가 얼마나 최신인지"** 를 그대로 표시할 수 있게 했습니다.

### 7.4 임계값(threshold) 판정은 서버에서

프론트가 계산을 떠안지 않도록, **status를 서버에서 결정**해 `green / yellow / red / none`으로 내려보냅니다.

```python
def _threshold_status(value, green_max, yellow_max):
    if value is None:
        return "none"
    if value <= green_max:
        return "green"
    if value <= yellow_max:
        return "yellow"
    return "red"
```

| Card | green / yellow / red |
|---|---|
| `alb_p99_max` | ≤1.0초 / ≤3.0초 / 그 외 |
| `alb_error_rate` | ≤1.0% / ≤5.0% / 그 외 |
| `ecs_cpu_avg` | ≤50% / ≤70% / 그 외 |

나머지 카드(RPS, 업로드 건수, 미해결 요청 수 등)는 **상태 없음(`"none"`)** 으로 표기합니다. "임계를 정의하기 어려운 지표는 임계를 붙이지 않는다"를 명시적인 규칙으로 가져갔습니다.

---

## 8. Web 레이어 — 정적 호스팅으로 충분한 이유

```html
<main class="layout">
  <header class="header">
    <h1>SRE Dashboard</h1>
    <p id="meta" class="meta">데이터를 불러오는 중...</p>
  </header>

  <section class="period-switch" aria-label="Period selector">
    <button type="button" data-period="last_week" class="period-button is-active">Last Week</button>
    <button type="button" data-period="current"   class="period-button">Current</button>
  </section>

  <section>
    <h2>Summary</h2>
    <div id="summary" class="summary-grid"></div>
  </section>

  <section>
    <h2>Panels</h2>
    <div id="panels" class="panel-list"></div>
  </section>

  <script src="./app.js" defer></script>
</main>
```

페이지 쪽 로직은 본질적으로 "fetch → render"가 전부입니다.

```js
async function loadPanel(which) {
  const res = await fetch(`./data/${which}.json`, { cache: "no-cache" });
  if (!res.ok) {
    renderError(`failed to fetch ./data/${which}.json`);
    return;
  }
  const data = await res.json();
  renderSummary(data.summary_cards);
  renderPanels(data.panels);
  renderFreshness(data.freshness);
}
```

이 단순함이 핵심이었습니다.

- **외부 API 호출 0** — 브라우저 ↔ S3만의 흐름.
- 브라우저는 **인증된 API를 모른다**. 토큰도 가지지 않습니다. 그냥 public S3 정적 웹의 JSON을 GET할 뿐이고, 사내 데이터는 collector/builder가 Lambda 실행 Role로 긁어서 이미 가공/익명화해둔 상태입니다.
- "스키마 버전"과 "panel kind" 개념을 JSON에 박아두어 새로운 panel 종류(line / list / ranking)는 payload에 `kind`만 추가하면 됩니다.

---

## 9. 보관 정책 / 알람

**raw 버킷 수명주기**

- `raw/`, `runs/` → `rawRetentionDays`일 후 삭제 (기본 180일)
- `aggregates/` → `aggregateRetentionDays`일 후 삭제 (기본 730일)

**CloudWatch 알람 (MVP)**

- `sre-dashboard-collector-errors`: Lambda `Errors >= 1` (5분)
- `sre-dashboard-builder-errors`: Lambda `Errors >= 1` (5분)

값 자체가 이상한 경우(예: `alb_error_rate`가 튀는 경우)는 이 대시보드의 알람으로 잡지 않기로 했습니다. 이건 어디까지나 **"관측용 리포트 페이지"** 이고, 실제 프로덕션 알람은 별도 시스템이 담당하기 때문입니다. 여기서는 **대시보드가 갱신되지 않는 것**만 알람의 책임으로 좁혔습니다.

---

## 10. Pulumi 설정 키로 켜고 끄기

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

## 11. 운영 체크리스트

주간 미팅 전에 대시보드가 비어 있으면 미팅이 반쯤 날아갑니다. 그래서 초반 몇 주는 아래 순서로 점검했습니다.

1. 페이지에 `failed to fetch .../data/*.json`이 뜨면 → **builder Lambda 로그**부터 확인 (`/aws/lambda/sre-dashboard-builder`).
2. builder는 성공했는데 숫자가 비어 있으면 → **해당 시간대 `runs/...json` 매니페스트**의 `sources` 필드에서 실패한 소스를 특정.
3. 집계 JSON의 `freshness.missing_source_hours`가 증가하면 → collector가 특정 시간대에 반복적으로 실패하는 패턴인지 확인(예: 특정 소스의 외부 API 점검 시간대).
4. 둘 다 성공인데 여전히 이상하면 → raw 오브젝트를 직접 열어보고, 원본 자체가 기대한 스키마인지 확인.
5. 로그 그룹: `/aws/lambda/sre-dashboard-collector`, `/aws/lambda/sre-dashboard-builder`.
6. web 버킷은 공개 읽기 구성이므로, **민감 데이터는 반드시 raw 버킷(프라이빗)에만** 쓰도록 builder 로직을 유지.

핵심은 **raw에 원본이 항상 남아 있다는 것**입니다. 집계 로직이 틀려도 이미 수집된 과거 데이터를 기반으로 builder만 다시 돌리면 복구됩니다. "수집과 집계를 분리한다"의 값은 이 운영 시점에서 가장 확실하게 돌아왔습니다.

---

## 12. 이 설계에서 배운 것

- **"우리에게 필요한 건 실시간이 아니라 1시간 단위의 신뢰할 만한 한 페이지"** 라는 전제를 먼저 정하면, 요구하는 인프라의 크기가 극적으로 줄어듭니다. BI 툴이나 Grafana까지 갈 필요 없이 Lambda 2개 + S3 2개로 끝났습니다.
- **Collector / Builder 분리**는 처음엔 과해 보이지만, "집계 포맷을 바꾸고 싶다"는 요구가 오면 가치가 바로 체감됩니다. 재수집 없이 builder만 돌리면 되니까요. raw 보관 기간(180일) 동안은 과거 어느 시점이든 집계 로직을 바꿔가며 다시 그릴 수 있습니다.
- 비개발자용 지표는 **서버에서 상태를 결정해서 내려주고**, 프론트는 단순히 렌더만 하도록 만들면 변경/리뷰가 훨씬 쉬워집니다. 임계값을 JS에 박아두지 않고 payload에 `status: green|yellow|red|none`으로 실어 보내는 게 결과적으로 옳았습니다.
- S3 정적 호스팅은 "대시보드"처럼 **읽기 편향** 시스템과 궁합이 아주 좋습니다. 장애 영향도가 낮고, 인증 토큰을 프론트에 실을 일이 없고, CDN/캐시 친화적입니다.

다음 글에서는 이 대시보드 위에 얹을 예정인 **"지난주 대비 회귀 탐지 로직"**(builder 단에서 간단한 룰 기반 비교를 넣고, 임계를 넘은 항목만 상단에 띄우는 형태)을 정리할 계획입니다.
