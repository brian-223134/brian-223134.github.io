---
title: "[인턴] ALB Target Group cutover에 묻혀 있던 잠재 결함 — SRE Dashboard 데이터 누락 디버깅 회고"
excerpt: "토요일 새벽 알람과 Summary 카드의 빈 값이라는 두 증상을 추적하다가, ECS Fargate cutover 이후 collector가 옛 TG suffix를 그대로 보고 있었음을 발견한 디버깅 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, SRE, CloudWatch, ALB, ECS, Fargate, Pulumi, 디버깅]

permalink: /intern/company/blitz-dynamics/30/

toc: true
toc_sticky: true

date: 2026-04-27
last_modified_at: 2026-04-27
---

## TL;DR

- 토요일 새벽 알람이 떴고 [SRE Dashboard](/intern/company/blitz-dynamics/29/)의 Summary 카드(서비스 성능/안정성/트래픽)가 `-`/`0`으로 보였다. 두 증상은 사실 별개 문제였다.
- 알람의 원인은 한 fetcher의 일회성 502 — 5분간 자동 복구되었고 데이터 누락은 없었다.
- 카드의 원인은 ECS Fargate cutover로 트래픽이 새 TG(`ecs-tg`)로 이동했는데, SRE collector는 여전히 옛 TG(`legacy-ec2-tg`) suffix를 CloudWatch dimension으로 들고 있었다.
- 핵심 통찰: ALB의 Target 단위 메트릭(`TargetResponseTime`, `HTTPCode_Target_*`)은 LoadBalancer 단독 dimension으로는 **발행조차 되지 않는다**. 반드시 `[LoadBalancer + TargetGroup]` 조합 발행. 따라서 TG suffix가 옛 값이면 메트릭 4개가 통째로 0/empty가 된다.
- README에 "ECS production 모듈이 활성화되면 자동 주입된다"고 적혀 있었지만 Pulumi caller에서 wire-up이 빠져 있었다. fix는 5파일짜리 prop drilling.

## 1. 첫 신호 — 두 가지 섞인 증상

> "AWS CloudWatch 지표를 토요일부터 못 가져오는 것 같다. 에러 알람이 왔다."

동시에 SRE Dashboard 화면을 보니 Summary 4개 카드 중 3개가 `-`/`0`으로 떠 있고 인프라 자원(CPU) 카드만 정상이었다. 처음에는 같은 원인이라고 가정했지만 실제로는 두 증상이 별개였다.

## 2. Lambda 로그 분석 — 알람의 진짜 원인

`/aws/lambda/sre-dashboard-collector` 로그 검색.

```text
2026-04-25T06:05:08Z urllib.error.HTTPError: HTTP Error 502: Bad Gateway
  File "request_***_fetcher.py", line 116
2026-04-25T06:05:08Z [ERROR] RuntimeError: collector failed for sources ['request_***']
```

CloudWatch가 아니라 한 비즈니스 fetcher가 GraphQL 본 쿼리에서 502를 받은 것. CloudWatch Alarm history도 확인:

```text
2026-04-25T15:06:56 KST  OK → ALARM
2026-04-25T15:11:56 KST  ALARM → OK   (5분간)
```

알람은 1번만 발생, 자동 복구. 모든 시간대의 manifest가 `status: ok`로 정상 적재 → **데이터 누락 없음**.

handler 코드를 읽어보니, 5개 소스 중 단 하나라도 실패하면 전체를 `RuntimeError`로 던지는 구조였다. 한 소스의 일회성 5xx가 곧장 알람으로 직결되는 구조. 이것은 별도 이슈로 정리.

## 3. 진짜 문제 — Summary 카드 3개의 `-`/`0`

S3 `data/last_week.json`을 까보니 카드 값은 정상이었다.

```json
{ "id": "alb_p99_max",    "value": 157.625, "unit": "초",  "status": "red"   }
{ "id": "alb_error_rate", "value": 0.003,   "unit": "%",   "status": "green" }
{ "id": "alb_rps_avg",    "value": 3.344,   "unit": "req/s" }
{ "id": "ecs_cpu_avg",    "value": 20.733,  "unit": "%",   "status": "green" }
```

그런데 `current.json`과 `this_week.json`은:

```text
alb_p99_max     value=null
alb_error_rate  value=null
alb_rps_avg     value=0.0
ecs_cpu_avg     value=23.479   ← 정상
```

ALB 관련 3개만 비어있고 ECS CPU만 정상. 처음 가설은 "트래픽이 적은 새벽 시간대라 ALB latency 메트릭이 발행 안 됨"이었지만 사용자가 부정: "새벽에도 트래픽이 원래 있었음."

## 4. CloudWatch CLI로 가설 검증

같은 시간 같은 ALB를 직접 호출.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/<ALB-NAME>/<LB-SUFFIX> \
  --start-time 2026-04-26T23:00:00Z --end-time 2026-04-27T00:00:00Z \
  --period 300 --statistics Sum
```

결과: 5분당 수천 건 요청. 정상이다. 그런데 collector가 같은 시간에 받은 ALB payload는 60 points 모두 `0.0`.

차이는 dimension 한 줄이었다.

```python
# CLI 호출 (성공)
Dimensions=[{"Name": "LoadBalancer", "Value": "..."}]

# collector 호출 (0/empty) — metrics.py:31-34
alb_dimensions = [
    {"Name": "TargetGroup",  "Value": settings.target_group_suffix},
    {"Name": "LoadBalancer", "Value": settings.load_balancer_suffix},
]
```

## 5. 핵심 통찰 — CloudWatch ALB 메트릭 발행 차원 규칙

AWS/ApplicationELB 네임스페이스의 메트릭은 **차원 조합 별로 따로 발행**된다.

| 메트릭 | LoadBalancer 단독 발행 | LoadBalancer + TargetGroup 발행 |
|---|---|---|
| `RequestCount` | ✅ ALB 전체 요청 | ✅ 그 TG로 forward된 요청 |
| `HTTPCode_ELB_*` | ✅ ALB 자체 4xx/5xx | — |
| `HTTPCode_Target_5XX_Count` | — | ✅ Target 메트릭 |
| `TargetResponseTime` (p95/p99) | — | ✅ Target 메트릭 |
| `UnHealthyHostCount` | — | ✅ TG 단위 |

요점은 **`TargetResponseTime`, `HTTPCode_Target_*`, `UnHealthyHostCount`는 LoadBalancer 단독 dimension으로는 발행조차 되지 않는다**는 것. Target 단위 메트릭이라 항상 `[LB + TG]` 조합으로만 잡힌다. `RequestCount`는 둘 다 발행되지만 의미가 다르다.

collector는 `[LB + TG=legacy-ec2-tg]` 조합으로 호출 중. 옛 TG로 트래픽이 0이면 4개 메트릭이 통째로 0/empty가 된다.

## 6. ALB Listener Rule 분석

```text
ALB <ALB-NAME> (변경 없음)
└─ Listener :443 (HTTPS)
   ├─ rule prio 10000~10650 (마지막은 0.0.0.0/0)
   │   └─ forward → ecs-tg (<ECS-TG-SUFFIX>)
   └─ default
       └─ forward → legacy-ec2-tg  ← 도달 안 함
```

priority 10650에 `0.0.0.0/0` rule이 있어 모든 트래픽이 새 TG(`ecs-tg`)로 forward된다. ALB는 그대로지만 트래픽 목적지가 바뀐 것.

분 단위 트래픽 추이로 cutover 시점을 좁히면:

| 시각 (KST) | legacy-ec2-tg | ecs-tg |
|---|---|---|
| 4/24 14:00 | 1,232 | 44,629 |
| 4/24 22:00 | 3,304 | 57,266 |
| 4/24 22:30 | **3,684** | 63,745 |
| **4/24 23:00** | **0** ⬅ | 61,749 |
| 이후 | 계속 0 | 50k~70k 유지 |

4/24 22:30 → 23:00 KST 사이에 cutover 100% 완료. 사용자가 인지한 "토요일부터" 와 정확히 일치한다.

## 7. "전엔 잘 나왔다"는 인지의 실체

git log로 추적해보니, collector는 4/21 SRE dashboard 프로토타입 시점부터 줄곧 `legacy-ec2-tg`를 가리키도록 작성되어 있었다 — 변경 없음. 그럼 4/21~4/24 동안 어떻게 데이터가 들어와 있었나? 토요일 직전 시간대의 raw payload를 까보면:

| 시각 (UTC) | RPS sum/시간 | p99 max |
|---|---|---|
| 4/22 06시 | 5,536 | 5,793 ms |
| 4/22 12시 | 5,534 | **157,625 ms** |
| 4/23 12시 | 6,762 | **157,100 ms** |
| 4/24 06시 | 3,907 | 6,751 ms |
| 4/24 12시 | 10,319 | 17,474 ms |
| **4/24 18시** | **0** | **EMPTY** |

ALB 전체(시간당 약 350,000건)의 **1~3% 잔류 트래픽**만 옛 TG에 흘러 들어오고 있었다. 카드는 0이 아니므로 정상으로 보였지만 실제로는 production의 일부만 보여주는 부분 집계였다. last_week.json의 빨간색 status `p99=157초`도 잔류 트래픽 한두 건의 슬로우 요청에서 그어진 값.

**즉 "전엔 잘 나왔다"는 인지는 절반만 맞았다.** 처음부터 잘못된 TG를 보고 있었지만 잔류 트래픽 덕에 0이 아니었을 뿐. ECS cutover 100% 완료가 잠재 결함을 표면화시킨 것.

## 8. 코드 수정 — wire-up이 빠져있던 5 파일

이미 README에 의도가 명시되어 있었다.

> ECS production 모듈이 활성화되어 있으면 SRE collector는 `ecsProduction.targetGroupArnSuffix`를 사용해 ALB 지표를 조회한다. ECS 전환 후 옛 EC2 TG가 아닌 current ECS TG를 보게 하기 위함이다.

그러나 caller wire-up이 빠진 채 fallback이 옛 TG로 떨어진 상태였다. fix는 `cloudWatchTargetGroupSuffix` prop을 5단계 하향 전달:

```typescript
// infra/src/index.ts
const sreDashboard = createSreDashboard({
  alarmActions: notification.alarmActions ?? [],
+ cloudWatchTargetGroupSuffix: ecsProduction?.targetGroupArnSuffix,
});
```

```typescript
// infra/src/sre-dashboard/types.ts
export interface SreDashboardParams {
  alarmActions: pulumi.Input<string>[];
+ cloudWatchTargetGroupSuffix?: pulumi.Input<string>;
}
```

```typescript
// infra/src/sre-dashboard/index.ts
const collector = createSreDashboardCollectorLambda({
  rawBucket,
  lambdaRole,
+ cloudWatchTargetGroupSuffix: params.cloudWatchTargetGroupSuffix,
});
```

```typescript
// infra/src/sre-dashboard/collector/index.ts
interface CollectorArgs {
    rawBucket: aws.s3.Bucket;
    lambdaRole: aws.iam.Role;
+   cloudWatchTargetGroupSuffix?: pulumi.Input<string>;
}
// ...
buildCollectorEnvironment({
    rawBucket: args.rawBucket,
+   cloudWatchTargetGroupSuffix: args.cloudWatchTargetGroupSuffix,
})
```

```typescript
// infra/src/sre-dashboard/collector/environment.ts
- ALB_TARGET_GROUP_SUFFIX: settings.sreDashboard.targetGroupSuffix,
+ ALB_TARGET_GROUP_SUFFIX:
+     args.cloudWatchTargetGroupSuffix ??
+     settings.sreDashboard.targetGroupSuffix,
```

`pulumi up` 후 환경변수 검증.

```bash
aws lambda get-function-configuration --function-name sre-dashboard-collector \
  --query 'Environment.Variables.ALB_TARGET_GROUP_SUFFIX' --output text
# → targetgroup/ecs-tg/<ECS-TG-SUFFIX>
```

## 9. 백필 — 50시간 매시간 분할 호출

`window.py`가 `event.start`/`event.end`를 함께 받으면 그 윈도우로 직접 처리한다. cloudwatch만 다시 수집하도록 `event.sources=["cloudwatch"]` 명시하고, S3 키가 `hh` 단위라 1시간씩 분할해 호출.

```bash
python3 -c "
from datetime import datetime, timedelta, timezone
s = datetime(2026, 4, 24, 23, tzinfo=timezone.utc)
e = datetime(2026, 4, 27,  1, tzinfo=timezone.utc)
cur = s
while cur < e:
    nxt = cur + timedelta(hours=1)
    print(cur.strftime('%Y-%m-%dT%H:%M:%SZ'), nxt.strftime('%Y-%m-%dT%H:%M:%SZ'))
    cur = nxt
" | while read ws we; do
  printf '→ %s ~ %s  ' "$ws" "$we"
  aws lambda invoke \
    --function-name sre-dashboard-collector \
    --payload "{\"start\":\"$ws\",\"end\":\"$we\",\"sources\":[\"cloudwatch\"]}" \
    --cli-binary-format raw-in-base64-out \
    /tmp/collector-out.json >/dev/null \
    && jq -r '.sources.cloudwatch.status' /tmp/collector-out.json
done

# builder 1회 트리거하여 last_week.json 즉시 재생성
aws lambda invoke \
  --function-name sre-dashboard-builder \
  --payload '{}' --cli-binary-format raw-in-base64-out \
  /tmp/builder-out.json
```

## 10. 회고 — 무엇을 배웠나

**1. CloudWatch dimension의 발행 규칙은 메트릭마다 다르다.**
`RequestCount`만 LB 단독 발행이 가능하고, 나머지 Target 메트릭은 LB+TG 조합 필수. CLI로 dimension 한 줄만 빼고 호출하면 결과가 수천 RPS인지 0인지 갈린다.

**2. 점진적 cutover는 잠재 결함을 가린다.**
잔류 트래픽 1~3%만 흘러도 카드는 0이 아니므로 정상으로 보인다. 100% cutover가 완료되어야 결함이 표면화된다. **카드가 "보이느냐"가 아니라 "값의 절대 크기가 production 규모인가"를 봐야 한다.**

**3. 알람 이름 ≠ 진짜 원인.**
`sre-dashboard-collector-errors` 알람이 떴다고 collector 전체가 망가진 게 아니다. 한 소스 실패가 RuntimeError로 던져져 알람을 트리거하는 구조 때문. handler가 부분 실패와 전체 실패를 구분하지 않으면 알람의 신호 대 잡음 비가 떨어진다.

**4. 설계 의도와 wire-up은 별개의 문제다.**
README에 "활성화되면 자동 주입"이라고 적혀 있어도, caller에서 인자를 전달하지 않으면 fallback으로 떨어진다. 문서의 약속과 실제 코드 wire-up은 별도로 검증해야 한다.

**5. "잘 나오던 게 갑자기 안 나온다"의 함정.**
처음부터 잘못된 것이지만 환경적 우연으로 가려져 있다가 환경 변화로 표면화될 수 있다. "이전엔 정상이었다"는 인지는 디버깅의 출발점일 뿐, 결론으로 가져가면 안 된다.
