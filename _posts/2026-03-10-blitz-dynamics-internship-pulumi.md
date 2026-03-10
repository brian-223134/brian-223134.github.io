---
title: "[블리츠다이나믹스] Pulumi로 CloudWatch 모니터링 & Slack 알람 인프라 구축"
excerpt: "AWS CloudWatch 메트릭 수집부터 대시보드 구성, Slack 알람 연동까지 Pulumi IaC로 구현하는 전 과정을 정리합니다."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Pulumi, CloudWatch, Slack, IaC]

permalink: /intern/company/blitz-dynamics/pulumi-cloudwatch-slack/

toc: true
toc_sticky: true

date: 2026-03-10
last_modified_at: 2026-03-10
---

# 들어가며

[이전 포스트](/intern/company/blitz-dynamics/log-metric/)에서 로그·메트릭 수집 도구를 전반적으로 살펴봤다면, 이번에는 실제로 인턴십에서 맡은 과제를 수행하는 방법을 정리한다.

과제의 핵심은 다음 세 가지다.

1. **AWS CloudWatch 메트릭 수집 및 정리** — 어떤 메트릭이 있고, 무엇을 봐야 하는가
2. **CloudWatch 대시보드 구축** — 필요한 메트릭을 한 화면에서 확인
3. **Slack 알람 연동** — 임계값 초과 시 Slack으로 즉시 통보

그리고 이 모든 것을 **Pulumi (TypeScript)**로 코드화하는 것이 목표다.

---

# 1. AWS CloudWatch 메트릭 이해

## 1-1. CloudWatch 메트릭이란?

**CloudWatch 메트릭(Metric)**은 AWS 리소스의 상태를 나타내는 시계열 수치 데이터다. EC2, RDS, Lambda, ECS 등 거의 모든 AWS 서비스가 메트릭을 자동으로 CloudWatch에 전송한다.

메트릭은 다음 세 요소로 구성된다.

| 요소 | 설명 | 예시 |
|---|---|---|
| **Namespace** | 메트릭을 묶는 논리적 그룹 | `AWS/EC2`, `AWS/RDS` |
| **Metric Name** | 측정 항목의 이름 | `CPUUtilization`, `NetworkIn` |
| **Dimension** | 메트릭을 필터링하는 키-값 쌍 | `InstanceId=i-0abc123` |

## 1-2. EC2 주요 메트릭 목록

EC2 인스턴스가 기본으로 제공하는 메트릭은 다음과 같다. Namespace는 `AWS/EC2`다.

### 컴퓨팅

| Metric Name | 단위 | 설명 |
|---|---|---|
| `CPUUtilization` | % | CPU 사용률. 지속적으로 80% 이상이면 스펙 업그레이드 검토 필요 |
| `CPUCreditBalance` | Count | T 계열 인스턴스의 CPU 크레딧 잔량. 0에 가까우면 성능 제한 시작 |
| `CPUCreditUsage` | Count | 소비 중인 CPU 크레딧 수 |

### 네트워크

| Metric Name | 단위 | 설명 |
|---|---|---|
| `NetworkIn` | Bytes | 인스턴스로 수신된 바이트 수 |
| `NetworkOut` | Bytes | 인스턴스에서 송신된 바이트 수 |
| `NetworkPacketsIn` | Count | 수신 패킷 수 |
| `NetworkPacketsOut` | Count | 송신 패킷 수 |

### 디스크 (인스턴스 스토어 전용)

| Metric Name | 단위 | 설명 |
|---|---|---|
| `DiskReadBytes` | Bytes | 디스크에서 읽은 바이트 수 |
| `DiskWriteBytes` | Bytes | 디스크에 쓴 바이트 수 |
| `DiskReadOps` | Count | 디스크 읽기 작업 수 |
| `DiskWriteOps` | Count | 디스크 쓰기 작업 수 |

> **주의**: 기본 EC2 메트릭에는 **메모리 사용률과 디스크 사용률(EBS)**이 포함되지 않는다. 이를 수집하려면 **CloudWatch Agent**를 설치해야 한다.

## 1-3. CloudWatch Agent로 추가 메트릭 수집 (추후 적용 예정)

> **현재 과제 범위 밖**: 이번 인턴십 과제에서는 AWS가 기본으로 제공하는 메트릭만 대상으로 한다. CloudWatch Agent 기반 메트릭은 현재 단계에서는 적용하지 않으며, 인프라가 안정화된 이후 추가로 도입할 예정이다.

EC2가 기본으로 보내는 메트릭에는 **메모리 사용률과 디스크 사용률이 포함되지 않는다.** 이를 수집하려면 EC2에 CloudWatch Agent를 별도 설치해야 하며, 수집된 메트릭은 `CWAgent` Namespace로 저장된다.

향후 수집 대상:

| Metric Name | 설명 |
|---|---|
| `mem_used_percent` | 메모리 사용률 (%) |
| `disk_used_percent` | 디스크 사용률 (%) |
| `disk_inodes_free` | 사용 가능한 inode 수 |
| `swap_used_percent` | 스왑 사용률 (%) |

## 1-4. 모니터링 핵심 메트릭 선정

모든 메트릭을 다 볼 필요는 없다. 실제 운영에서 중요한 메트릭을 추려내는 것이 먼저다. 블리츠다이나믹스 환경에서는 아래 메트릭을 우선 수집·모니터링한다.

```
[필수 모니터링 대상 - AWS 기본 제공 메트릭]
EC2 (Namespace: AWS/EC2):
  - CPUUtilization          → 서버 부하 파악
  - NetworkIn / NetworkOut  → 트래픽 이상 감지
  - NetworkPacketsIn/Out    → 패킷 수준 이상 감지
  - StatusCheckFailed       → 인스턴스/시스템 상태 이상 감지

RDS (사용 시, Namespace: AWS/RDS):
  - CPUUtilization
  - FreeStorageSpace
  - DatabaseConnections
  - ReadLatency / WriteLatency
```

---

# 2. CloudWatch 대시보드 구성

## 2-1. 대시보드 설계 원칙

좋은 대시보드는 **한눈에 서비스 상태를 파악**할 수 있어야 한다. 다음 기준으로 설계한다.

- **위에서 아래로**: 중요도가 높은 메트릭을 상단에 배치
- **그룹화**: EC2, RDS, 네트워크 등 리소스 유형별로 구분
- **시간 범위**: 기본 3시간, 최대 1주일까지 확인 가능하도록 설정

## 2-2. 대시보드 위젯 구성

CloudWatch 대시보드는 **위젯(Widget)**의 집합이다. 위젯 유형은 다음과 같다.

| 위젯 유형 | 용도 |
|---|---|
| `metric` | 시계열 그래프로 메트릭 시각화 |
| `alarm` | 알람 상태를 직관적으로 표시 |
| `text` | 마크다운 설명 추가 |
| `log` | CloudWatch Logs Insights 쿼리 결과 표시 |

---

# 3. Slack 알람 연동

## 3-1. 전체 알람 흐름

CloudWatch에서 Slack으로 알람을 보내는 흐름은 다음과 같다.

```
[CloudWatch Alarm]
        │ 상태 변경 (OK → ALARM)
        ▼
[SNS Topic]
        │ 메시지 발행
        ▼
[Lambda Function]
        │ Slack Webhook 호출
        ▼
[Slack Channel]
```

CloudWatch는 직접 Slack을 호출할 수 없기 때문에, **SNS(Simple Notification Service)**와 **Lambda**를 중간에 배치하는 구조를 사용한다.

## 3-2. SNS와 Lambda의 역할

- **SNS**: CloudWatch Alarm이 상태 변경 시 이벤트를 받아 구독자(Lambda)에게 전달하는 메시지 버스
- **Lambda**: SNS 이벤트를 받아 Slack Incoming Webhook URL로 HTTP POST 요청을 전송하는 함수

## 3-3. Slack 메시지 포맷

Lambda에서 Slack으로 보낼 메시지를 구조화하면 가독성이 높아진다.

```json
{
  "text": "🚨 CloudWatch 알람 발생",
  "attachments": [
    {
      "color": "danger",
      "fields": [
        { "title": "알람 이름", "value": "EC2-CPU-High", "short": true },
        { "title": "상태", "value": "ALARM", "short": true },
        { "title": "메트릭", "value": "CPUUtilization > 80%", "short": false },
        { "title": "발생 시각", "value": "2026-03-10 12:00:00 KST", "short": true }
      ]
    }
  ]
}
```

---

# 4. Pulumi로 전체 인프라 코드화

## 4-1. Pulumi란?

**Pulumi**는 TypeScript, Python, Go 등 일반 프로그래밍 언어로 인프라를 정의하는 **IaC(Infrastructure as Code)** 도구다.

AWS CloudFormation(YAML/JSON)이나 Terraform(HCL)과 달리, 익숙한 언어의 **조건문, 반복문, 함수, 패키지**를 그대로 활용할 수 있다는 것이 가장 큰 장점이다. 특히 TypeScript로 작성하면 **타입 안전성**이 보장되어 오타나 잘못된 파라미터를 컴파일 단계에서 잡을 수 있다.

### Pulumi vs Terraform vs CloudFormation

| 구분 | Pulumi | Terraform | CloudFormation |
|---|---|---|---|
| **언어** | TypeScript, Python, Go 등 | HCL (전용 언어) | YAML / JSON |
| **멀티 클라우드** | ✅ | ✅ | ❌ (AWS 전용) |
| **상태 관리** | Pulumi Cloud / S3 | S3 + DynamoDB | AWS 관리형 |
| **타입 안전성** | ✅ (TypeScript) | ❌ | ❌ |
| **재사용성** | 높음 (일반 언어 패턴) | 중간 (모듈) | 낮음 |

## 4-2. Pulumi 프로젝트 초기 설정

### 설치

```bash
# macOS
brew install pulumi

# 버전 확인
pulumi version
```

### 프로젝트 생성

```bash
# TypeScript 기반 Pulumi 프로젝트 생성
pulumi new aws-typescript --name blitz-monitoring --stack dev

# 생성된 구조
.
├── Pulumi.yaml          # 프로젝트 메타데이터
├── Pulumi.dev.yaml      # dev 스택 설정 (리전, 변수 등)
├── index.ts             # 인프라 정의 진입점
├── package.json         # Node.js 의존성
└── tsconfig.json        # TypeScript 설정
```

### 의존성 설치

```bash
npm install
# 또는
npm install @pulumi/pulumi @pulumi/aws
```

### AWS 인증 설정

```bash
# AWS CLI 프로필 설정 (이미 되어 있다면 생략)
aws configure

# Pulumi에 AWS 리전 설정
pulumi config set aws:region ap-northeast-2
```

### Slack Webhook URL을 시크릿으로 저장

```bash
# 평문으로 저장하지 않고 암호화된 시크릿으로 관리
pulumi config set --secret slackWebhookUrl https://hooks.slack.com/services/...
```

> `--secret` 플래그를 사용하면 Pulumi가 해당 값을 암호화하여 상태 파일에 저장한다. 코드나 Git에 노출되지 않는다.

## 4-3. SNS Topic 생성

CloudWatch Alarm이 발행할 SNS Topic을 먼저 만든다.

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

// SNS Topic: CloudWatch Alarm → Lambda 이벤트 전달
const alarmTopic = new aws.sns.Topic("alarm-topic", {
  name: "blitz-cloudwatch-alarm",
  tags: { Project: "blitz-monitoring" },
});
```

## 4-4. Lambda 함수 생성 (SNS → Slack)

Lambda 함수는 SNS 메시지를 받아 Slack Webhook을 호출한다.

### Lambda 핸들러 코드 작성

```typescript
// lambda/slackNotify.ts
import * as https from "https";

interface SnsRecord {
  Sns: { Message: string };
}

interface AlarmMessage {
  AlarmName?: string;
  NewStateValue?: string;
  NewStateReason?: string;
}

export const handler = async (event: { Records: SnsRecord[] }) => {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL!;

  for (const record of event.Records) {
    const message: AlarmMessage = JSON.parse(record.Sns.Message);

    const alarmName = message.AlarmName ?? "Unknown";
    const newState = message.NewStateValue ?? "Unknown";
    const reason = message.NewStateReason ?? "";

    const color = newState === "ALARM" ? "danger" : "good";
    const emoji = newState === "ALARM" ? "🚨" : "✅";

    const payload = JSON.stringify({
      text: `${emoji} CloudWatch 알람: *${alarmName}*`,
      attachments: [
        {
          color,
          fields: [
            { title: "상태", value: newState, short: true },
            { title: "원인", value: reason, short: false },
          ],
        },
      ],
    });

    await new Promise<void>((resolve, reject) => {
      const url = new URL(webhookUrl);
      const req = https.request(
        { hostname: url.hostname, path: url.pathname, method: "POST",
          headers: { "Content-Type": "application/json" } },
        (res) => { res.on("data", () => {}); res.on("end", resolve); }
      );
      req.on("error", reject);
      req.write(payload);
      req.end();
    });
  }
};
```

### Pulumi로 Lambda 함수 정의

```typescript
// index.ts (계속)

const config = new pulumi.Config();
const slackWebhookUrl = config.requireSecret("slackWebhookUrl");

// Lambda 실행 역할
const lambdaRole = new aws.iam.Role("lambda-role", {
  assumeRolePolicy: JSON.stringify({
    Version: "2012-10-17",
    Statement: [{
      Effect: "Allow",
      Principal: { Service: "lambda.amazonaws.com" },
      Action: "sts:AssumeRole",
    }],
  }),
});

// 기본 Lambda 실행 권한 (CloudWatch Logs 쓰기)
new aws.iam.RolePolicyAttachment("lambda-basic-execution", {
  role: lambdaRole.name,
  policyArn: "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
});

// Lambda 함수
const slackNotifier = new aws.lambda.Function("slack-notifier", {
  name: "blitz-slack-notifier",
  runtime: aws.lambda.Runtime.NodeJS22dX,
  handler: "slackNotify.handler",
  role: lambdaRole.arn,
  code: new pulumi.asset.AssetArchive({
    ".": new pulumi.asset.FileArchive("./lambda"),  // lambda/ 디렉터리를 ZIP으로 패키징
  }),
  environment: {
    variables: { SLACK_WEBHOOK_URL: slackWebhookUrl },
  },
  timeout: 30,
  tags: { Project: "blitz-monitoring" },
});

// SNS가 Lambda를 호출할 수 있도록 권한 부여
new aws.lambda.Permission("sns-lambda-permission", {
  action: "lambda:InvokeFunction",
  function: slackNotifier.name,
  principal: "sns.amazonaws.com",
  sourceArn: alarmTopic.arn,
});

// SNS Topic → Lambda 구독 연결
new aws.sns.TopicSubscription("alarm-topic-subscription", {
  topic: alarmTopic.arn,
  protocol: "lambda",
  endpoint: slackNotifier.arn,
});
```

## 4-5. CloudWatch Alarm 생성

EC2 CPU 사용률 알람을 예시로 작성한다.

```typescript
// index.ts (계속)

const INSTANCE_ID = "i-0abc123def456"; // 실제 EC2 인스턴스 ID

// CPU 사용률 알람
const cpuAlarm = new aws.cloudwatch.MetricAlarm("cpu-high-alarm", {
  name: "EC2-CPU-High",
  comparisonOperator: "GreaterThanOrEqualToThreshold",
  evaluationPeriods: 2,           // 2번 연속 초과 시 알람
  metricName: "CPUUtilization",
  namespace: "AWS/EC2",
  period: 300,                    // 5분 단위 평균
  statistic: "Average",
  threshold: 80,                  // 80% 이상
  alarmDescription: "EC2 CPU 사용률이 80%를 초과했습니다.",
  alarmActions: [alarmTopic.arn],
  okActions: [alarmTopic.arn],    // 복구 시에도 알람
  dimensions: { InstanceId: INSTANCE_ID },
  treatMissingData: "notBreaching",
  tags: { Project: "blitz-monitoring" },
});

// 인스턴스 상태 체크 알람
const statusAlarm = new aws.cloudwatch.MetricAlarm("status-check-alarm", {
  name: "EC2-StatusCheck-Failed",
  comparisonOperator: "GreaterThanOrEqualToThreshold",
  evaluationPeriods: 1,
  metricName: "StatusCheckFailed",
  namespace: "AWS/EC2",
  period: 60,
  statistic: "Maximum",
  threshold: 1,
  alarmDescription: "EC2 인스턴스 상태 체크가 실패했습니다.",
  alarmActions: [alarmTopic.arn],
  okActions: [alarmTopic.arn],
  dimensions: { InstanceId: INSTANCE_ID },
  treatMissingData: "notBreaching",
  tags: { Project: "blitz-monitoring" },
});
```

### 주요 파라미터 설명

| 파라미터 | 설명 |
|---|---|
| `evaluationPeriods` | 알람 발동 전 연속으로 임계값을 초과해야 하는 기간 수. 높을수록 일시적 스파이크에 덜 민감 |
| `period` | 메트릭 집계 간격 (초). 60, 300, 3600 등 |
| `statistic` | 집계 방식. `Average`, `Sum`, `Maximum`, `Minimum` 중 선택 |
| `threshold` | 알람 발동 기준값 |
| `treatMissingData` | 데이터 없을 때 처리 방식. `notBreaching`(정상 취급), `breaching`(위반 취급), `missing`, `ignore` |
| `alarmActions` | 알람 상태 전환 시 호출할 SNS Topic ARN 목록 |
| `okActions` | 정상 복구 시 호출할 SNS Topic ARN 목록 |

## 4-6. CloudWatch 대시보드 생성

```typescript
// index.ts (계속)

const dashboard = new aws.cloudwatch.Dashboard("monitoring-dashboard", {
  dashboardName: "blitz-server-monitoring",
  dashboardBody: pulumi.all([cpuAlarm.arn, memAlarm.arn, diskAlarm.arn])
    .apply(([cpuArn, memArn, diskArn]) =>
      JSON.stringify({
        widgets: [
          // 타이틀
          {
            type: "text",
            x: 0, y: 0, width: 24, height: 1,
            properties: { markdown: "## 블리츠다이나믹스 서버 모니터링 대시보드" },
          },
          // CPU 사용률
          {
            type: "metric",
            x: 0, y: 1, width: 8, height: 6,
            properties: {
              title: "CPU 사용률 (%)",
              metrics: [
                ["AWS/EC2", "CPUUtilization", "InstanceId", INSTANCE_ID,
                  { stat: "Average", period: 300 }],
              ],
              yAxis: { left: { min: 0, max: 100 } },
              annotations: {
                horizontal: [{ value: 80, color: "#ff0000", label: "임계값" }],
              },
              view: "timeSeries",
            },
          },
          // 메모리 사용률
          {
            type: "metric",
            x: 8, y: 1, width: 8, height: 6,
            properties: {
              title: "메모리 사용률 (%)",
              metrics: [
                ["CWAgent", "mem_used_percent", "InstanceId", INSTANCE_ID,
                  { stat: "Average", period: 300 }],
              ],
              yAxis: { left: { min: 0, max: 100 } },
              annotations: {
                horizontal: [{ value: 85, color: "#ff0000", label: "임계값" }],
              },
              view: "timeSeries",
            },
          },
          // 디스크 사용률
          {
            type: "metric",
            x: 16, y: 1, width: 8, height: 6,
            properties: {
              title: "디스크 사용률 (%)",
              metrics: [
                ["CWAgent", "disk_used_percent", "InstanceId", INSTANCE_ID,
                  "path", "/", "fstype", "xfs",
                  { stat: "Average", period: 300 }],
              ],
              yAxis: { left: { min: 0, max: 100 } },
              annotations: {
                horizontal: [{ value: 90, color: "#ff0000", label: "임계값" }],
              },
              view: "timeSeries",
            },
          },
          // 네트워크 트래픽
          {
            type: "metric",
            x: 0, y: 7, width: 12, height: 6,
            properties: {
              title: "네트워크 트래픽 (Bytes)",
              metrics: [
                ["AWS/EC2", "NetworkIn", "InstanceId", INSTANCE_ID,
                  { stat: "Sum", period: 300, label: "수신" }],
                ["AWS/EC2", "NetworkOut", "InstanceId", INSTANCE_ID,
                  { stat: "Sum", period: 300, label: "송신" }],
              ],
              view: "timeSeries",
            },
          },
          // 알람 상태
          {
            type: "alarm",
            x: 12, y: 7, width: 12, height: 6,
            properties: {
              title: "알람 상태",
              alarms: [cpuArn, memArn, diskArn],
            },
          },
        ],
      })
    ),
});
```

> **대시보드 그리드**: CloudWatch 대시보드는 24열 그리드 시스템을 사용한다. `x + width <= 24` 조건을 만족해야 한다.

## 4-7. 배포 및 확인

```bash
# 변경 사항 미리 보기 (실제 배포 전 diff 확인)
pulumi preview

# 배포 실행
pulumi up

# 배포된 리소스 목록 확인
pulumi stack output

# 스택 전체 제거
pulumi destroy
```

### `pulumi preview` 출력 예시

```
Previewing update (dev)

     Type                          Name                      Plan
 +   pulumi:pulumi:Stack           blitz-monitoring-dev      create
 +   ├─ aws:sns:Topic              alarm-topic               create
 +   ├─ aws:iam:Role               lambda-role               create
 +   ├─ aws:lambda:Function        slack-notifier            create
 +   ├─ aws:sns:TopicSubscription  alarm-topic-subscription  create
 +   ├─ aws:cloudwatch:MetricAlarm cpu-high-alarm            create
 +   ├─ aws:cloudwatch:MetricAlarm mem-high-alarm            create
 +   ├─ aws:cloudwatch:MetricAlarm disk-high-alarm           create
 +   └─ aws:cloudwatch:Dashboard   monitoring-dashboard      create

Resources:
    + 9 to create
```

---

# 5. Pulumi 핵심 개념 정리

Pulumi를 처음 사용할 때 이해하고 넘어가야 할 개념들이다.

## 5-1. 스택(Stack)

**스택**은 동일한 Pulumi 프로그램의 독립적인 배포 인스턴스다. `dev`, `staging`, `prod`처럼 환경을 분리하는 데 사용한다.

```bash
pulumi stack init prod        # prod 스택 생성
pulumi stack select dev       # dev 스택으로 전환
pulumi stack ls               # 스택 목록 확인
```

스택별로 설정값을 다르게 줄 수 있다.

```bash
# dev 스택: CPU 임계값 80%
pulumi config set cpuThreshold 80 --stack dev

# prod 스택: CPU 임계값 70% (더 민감하게)
pulumi config set cpuThreshold 70 --stack prod
```

## 5-2. Output과 Input

Pulumi 리소스의 속성값은 **`Output<T>`**라는 비동기 타입으로 반환된다. 리소스가 생성된 후에야 실제 값이 결정되기 때문이다. TypeScript에서는 이 타입이 명시적으로 드러나 IDE 자동완성과 타입 체크의 혜택을 그대로 누릴 수 있다.

```typescript
// SNS Topic ARN은 생성 전에는 알 수 없음 → Output<string> 타입
const alarmTopic = new aws.sns.Topic("alarm-topic");
const topicArn: pulumi.Output<string> = alarmTopic.arn;

// Output 값을 다른 리소스에 전달할 때는 그냥 넘기면 됨
new aws.cloudwatch.MetricAlarm("cpu-alarm", {
  alarmActions: [alarmTopic.arn],  // Output<string>을 직접 사용
  // ...
});
```

여러 Output 값을 조합해야 할 때는 `pulumi.all()` 또는 `.apply()`를 사용한다.

```typescript
// 여러 Output을 묶어서 처리
const combined = pulumi.all([topicArn, lambdaArn]).apply(
  ([topic, lambda]) => `Topic: ${topic}, Lambda: ${lambda}`
);

// 단일 Output 변환
const shortArn = topicArn.apply((arn) => arn.split(":").pop()!);
```

## 5-3. 상태 파일(State)

Pulumi는 **현재 배포된 인프라의 상태**를 상태 파일에 저장한다. 다음 번에 `pulumi up`을 실행하면 코드와 상태 파일을 비교하여 변경이 필요한 것만 적용한다.

상태 파일 저장 위치는 세 가지 중 선택한다.

| 백엔드 | 명령 | 특징 |
|---|---|---|
| Pulumi Cloud | 기본값 | GUI 대시보드 제공, 팀 협업 가능 |
| S3 | `pulumi login s3://bucket-name` | 자체 관리, 추가 비용 없음 |
| 로컬 | `pulumi login --local` | 개인 개발용, 공유 불가 |

## 5-4. Config와 Secret

환경별로 다른 값을 주입하거나 민감한 정보를 안전하게 관리할 때 사용한다.

```bash
# 일반 설정값
pulumi config set instanceId i-0abc123def456

# 암호화된 시크릿
pulumi config set --secret slackWebhookUrl https://hooks.slack.com/...
```

코드에서 읽는 방법:

```typescript
const config = new pulumi.Config();

const instanceId = config.require("instanceId");               // 없으면 에러
const slackUrl = config.requireSecret("slackWebhookUrl");      // Output<string> 타입 반환
const threshold = config.getNumber("cpuThreshold") ?? 80;      // 없으면 기본값
```

---

# 6. 전체 인프라 흐름 요약

```
[Pulumi 코드 (TypeScript)]
  ├── SNS Topic (alarm-topic)
  │       └── Subscription → Lambda (slack-notifier)
  │
  ├── Lambda Function (Node.js 22)
  │       ├── 환경변수: SLACK_WEBHOOK_URL (Secret)
  │       └── 코드: SNS 메시지 파싱 → Slack POST
  │
  ├── CloudWatch Alarms
  │       ├── EC2-CPU-High       (CPUUtilization ≥ 80%)
  │       ├── EC2-Memory-High    (mem_used_percent ≥ 85%)
  │       └── EC2-Disk-High      (disk_used_percent ≥ 90%)
  │               └── alarmActions → SNS Topic
  │
  └── CloudWatch Dashboard
          ├── CPU / 메모리 / 디스크 사용률 그래프
          ├── 네트워크 트래픽 그래프
          └── 알람 상태 위젯

[이벤트 흐름]
EC2 메트릭 임계값 초과
  → CloudWatch Alarm (ALARM 상태)
  → SNS Topic에 메시지 발행
  → Lambda 트리거
  → Slack Webhook 호출
  → Slack 채널에 알람 메시지 수신
```

---

# 마무리

이번 포스트에서는 AWS CloudWatch 메트릭 수집 → 대시보드 구성 → Slack 알람 연동의 전체 흐름을 Pulumi TypeScript 코드로 구현하는 방법을 정리했다.

핵심 포인트를 한 줄로 요약하면 다음과 같다.

- **CloudWatch Alarm** → 임계값 초과 감지
- **SNS + Lambda** → 이벤트 라우팅 및 변환
- **Slack Webhook** → 팀에게 즉시 알림
- **Pulumi (TypeScript)** → 위 모든 것을 타입 안전한 코드로 관리, 재현 가능한 인프라 보장

다음 단계로는 여러 EC2 인스턴스에 대한 알람을 `for...of` 반복문으로 일괄 생성하거나, Pulumi `ComponentResource`를 활용하여 재사용 가능한 모니터링 모듈을 만드는 것을 고려해볼 수 있다.
