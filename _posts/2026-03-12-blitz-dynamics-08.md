---
title: "[인턴] Config와 Pulumi import"
excerpt: "Pulumi의 config 기반 다중 stack yaml 기반 매개변수 관리 및 기존 AWS 리소스 풀링을 설명."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Pulumi, AWS, Stack]

permalink: /intern/company/blitz-dynamics/08/

toc: true
toc_sticky: true

date: 2026-03-12
last_modified_at: 2026-03-12
---

# Pulumi 실전 가이드: Config 기반 조건 분기와 pulumi import

> Pulumi에서 하나의 코드베이스로 여러 환경을 관리하는 방법과, 기존 AWS 리소스를 Pulumi로 편입시키는 import 기능을 실전 예시와 함께 정리합니다.

---

## Part 1. Config 기반 조건 분기

### 왜 필요한가

Pulumi 프로젝트에서 TypeScript 코드는 단 하나입니다. 그런데 현실에서는 dev 환경에는 RDS가 없고, staging에는 있고, prod에서는 Multi-AZ RDS가 필요한 것처럼 **환경마다 인프라 구성이 달라야** 합니다.

이 문제를 해결하는 방법이 Config 기반 조건 분기입니다. 각 Stack의 yaml 파일에 설정값을 넣어두고, 코드에서 그 값을 읽어 리소스를 생성할지 말지, 어떤 스펙으로 생성할지를 결정하는 패턴입니다.

---

### Config 시스템의 구조

Pulumi의 Config는 프로젝트 루트에 `Pulumi.<stack-name>.yaml` 파일로 관리됩니다.

```
my-infra/
├── index.ts
├── Pulumi.yaml              ← 프로젝트 메타 정보
├── Pulumi.dev-01.yaml       ← dev-01 stack 전용 설정
├── Pulumi.dev-02.yaml       ← dev-02 stack 전용 설정
└── Pulumi.prod.yaml         ← prod stack 전용 설정
```

config 값을 설정하는 방법은 두 가지입니다.

**CLI로 설정하는 방법:**

```bash
pulumi stack select dev-01
pulumi config set instanceType t3.medium
pulumi config set enableRds false
pulumi config set --secret dbPassword "P@ssw0rd"   # 민감 정보는 --secret
```

**yaml 파일을 직접 편집하는 방법:**

```yaml
# Pulumi.dev-01.yaml
config:
  my-infra:instanceType: t3.medium
  my-infra:enableRds: "false"
  aws:region: ap-northeast-2
```

여기서 `my-infra:` 접두사는 프로젝트 이름(Pulumi.yaml의 name)이며, `aws:`는 aws provider의 네임스페이스입니다.

---

### Config 값을 코드에서 읽는 방법

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();           // 프로젝트 네임스페이스의 config
const awsConfig = new pulumi.Config("aws");   // aws 네임스페이스의 config

// 기본 읽기
const instanceType = config.require("instanceType");       // 없으면 에러
const enableRds = config.requireBoolean("enableRds");      // boolean으로 파싱
const maxCount = config.getNumber("maxCount") ?? 3;        // 없으면 기본값 3

// Secret 읽기
const dbPassword = config.requireSecret("dbPassword");     // Output<string> 반환

// Stack 이름 자체도 활용 가능
const stack = pulumi.getStack();  // "dev-01", "dev-02", "prod" 등
```

`require`는 값이 없으면 즉시 에러를 발생시키고, `get`은 `undefined`를 반환합니다. 필수 설정에는 `require`, 선택적 설정에는 `get`을 사용하는 것이 관례입니다.

---

### 조건 분기 패턴 4가지

#### 패턴 1: 리소스 생성 여부를 on/off 하기

가장 기본적인 패턴입니다. 특정 리소스를 이 stack에서 만들 것인지 말 것인지를 boolean으로 제어합니다.

```yaml
# Pulumi.dev-01.yaml
config:
  my-infra:enableRds: "false"
  my-infra:enableRedis: "false"

# Pulumi.dev-02.yaml
config:
  my-infra:enableRds: "true"
  my-infra:enableRedis: "true"
```

```typescript
const enableRds = config.requireBoolean("enableRds");
const enableRedis = config.requireBoolean("enableRedis");

let rdsInstance: aws.rds.Instance | undefined;
if (enableRds) {
    rdsInstance = new aws.rds.Instance(`db-${stack}`, {
        engine: "mysql",
        instanceClass: config.require("rdsInstanceClass"),
        allocatedStorage: 20,
        skipFinalSnapshot: stack !== "prod",  // prod만 final snapshot
        tags: { Environment: stack },
    });
}

let redisCluster: aws.elasticache.Cluster | undefined;
if (enableRedis) {
    redisCluster = new aws.elasticache.Cluster(`cache-${stack}`, {
        engine: "redis",
        nodeType: "cache.t3.micro",
        numCacheNodes: 1,
    });
}
```

이 패턴의 핵심은 **리소스를 담는 변수를 `let`으로 선언하고 조건부로 할당**하는 것입니다. 이후에 다른 리소스가 이 값을 참조할 때는 `undefined` 체크를 해야 합니다.

---

#### 패턴 2: 동일 리소스의 스펙만 변경하기

모든 환경에 EC2가 존재하지만 인스턴스 타입, 디스크 크기 등이 다른 경우입니다.

```yaml
# Pulumi.dev-01.yaml
config:
  my-infra:instanceType: t3.medium
  my-infra:volumeSize: "20"
  my-infra:instanceCount: "1"

# Pulumi.prod.yaml
config:
  my-infra:instanceType: c5.2xlarge
  my-infra:volumeSize: "100"
  my-infra:instanceCount: "3"
```

```typescript
const instanceType = config.require("instanceType");
const volumeSize = config.requireNumber("volumeSize");
const instanceCount = config.requireNumber("instanceCount");

for (let i = 0; i < instanceCount; i++) {
    new aws.ec2.Instance(`web-${stack}-${i}`, {
        ami: "ami-0abcdef1234567890",
        instanceType: instanceType,
        rootBlockDevice: {
            volumeSize: volumeSize,
            volumeType: "gp3",
        },
        tags: {
            Name: `web-${stack}-${i}`,
            Environment: stack,
        },
    });
}
```

---

#### 패턴 3: 구조화된 Config 객체 사용하기

설정 항목이 많아지면 개별 config key가 관리하기 어려워집니다. 이때 JSON 객체를 config에 넣을 수 있습니다.

```bash
pulumi config set --path 'ec2.instanceType' t3.xlarge
pulumi config set --path 'ec2.volumeSize' 50
pulumi config set --path 'rds.enabled' true
pulumi config set --path 'rds.instanceClass' db.t3.large
pulumi config set --path 'rds.multiAz' false
```

결과 yaml:

```yaml
config:
  my-infra:ec2:
    instanceType: t3.xlarge
    volumeSize: 50
  my-infra:rds:
    enabled: true
    instanceClass: db.t3.large
    multiAz: false
```

코드에서 읽을 때:

```typescript
// 인터페이스 정의
interface Ec2Config {
    instanceType: string;
    volumeSize: number;
}

interface RdsConfig {
    enabled: boolean;
    instanceClass: string;
    multiAz: boolean;
}

const ec2Conf = config.requireObject<Ec2Config>("ec2");
const rdsConf = config.requireObject<RdsConfig>("rds");

const server = new aws.ec2.Instance(`web-${stack}`, {
    instanceType: ec2Conf.instanceType,
    rootBlockDevice: { volumeSize: ec2Conf.volumeSize },
    // ...
});

if (rdsConf.enabled) {
    new aws.rds.Instance(`db-${stack}`, {
        instanceClass: rdsConf.instanceClass,
        multiAz: rdsConf.multiAz,
        // ...
    });
}
```

Config 항목이 5개를 넘어가기 시작하면 이 패턴으로 전환하는 것이 좋습니다. 타입 안전성도 확보되고 관련 설정끼리 묶이므로 가독성이 올라갑니다.

---

#### 패턴 4: Stack 이름 자체로 분기하기

간단한 경우에는 config 없이 stack 이름만으로 분기할 수도 있습니다. 다만 이 방식은 stack 이름에 로직이 결합되므로 유연성이 떨어집니다. 환경 수가 적고 구분이 명확할 때만 사용하세요.

```typescript
const stack = pulumi.getStack();

const isProd = stack === "prod";
const isStaging = stack.startsWith("staging");

const instance = new aws.ec2.Instance(`web-${stack}`, {
    instanceType: isProd ? "c5.2xlarge" : "t3.medium",
    monitoring: isProd,   // prod만 상세 모니터링
    // ...
});

// prod에서만 Multi-AZ RDS
if (isProd) {
    new aws.rds.Instance(`db-${stack}`, {
        multiAz: true,
        backupRetentionPeriod: 7,
        // ...
    });
}
```

---

### Config 분기 시 주의사항

**1) 코드 변경 후 반드시 모든 활성 stack에서 preview를 실행하세요.**

```bash
for stack in dev-01 dev-02 staging prod; do
    echo "=== Checking $stack ==="
    pulumi stack select $stack
    pulumi preview 2>&1 | tail -5
    echo ""
done
```

dev-02를 위해 추가한 코드가 dev-01에서 예상치 못한 변경을 일으킬 수 있습니다.

**2) Config 기본값을 코드에 하드코딩하지 마세요.**

```typescript
// 나쁜 예: 코드에서 기본값을 숨김
const instanceType = config.get("instanceType") || "t3.medium";

// 좋은 예: 명시적으로 필수 값으로 관리
const instanceType = config.require("instanceType");
```

기본값이 코드에 묻히면 나중에 "이 stack에 왜 t3.medium이 떠있지?"라는 상황이 발생합니다. config에 명시적으로 넣어두는 것이 추적에 유리합니다.

**3) 조건부 리소스의 export 처리:**

```typescript
// 나쁜 예: rds가 undefined이면 런타임 에러
export const rdsEndpoint = rdsInstance.endpoint;

// 좋은 예: 조건부 export
export const rdsEndpoint = rdsInstance?.endpoint ?? "N/A";
```

---

---

## Part 2. pulumi import — 기존 AWS 리소스 가져오기

### 왜 필요한가

현실에서는 인프라가 항상 IaC로 시작하지 않습니다. AWS 콘솔에서 수동으로 만든 EC2, 운영팀이 CLI로 띄운 RDS, 이전 팀이 CloudFormation으로 만들었지만 이제 Pulumi로 전환하고 싶은 리소스 등 **이미 존재하는 리소스를 Pulumi의 state로 편입**시켜야 할 때가 있습니다.

`pulumi import`는 이 작업을 수행합니다. 기존 AWS 리소스를 삭제하거나 재생성하지 않고, Pulumi의 state 파일에 "이 리소스는 내가 관리한다"라고 등록하는 것입니다.

---

### import의 작동 원리

```
┌─────────────────────┐
│  AWS에 실제 존재하는  │
│  리소스 (EC2 등)     │
└─────────┬───────────┘
          │  pulumi import
          ▼
┌─────────────────────┐
│  Pulumi State 파일   │  ← 리소스 정보가 기록됨
│  (stack 상태 저장소)   │
└─────────┬───────────┘
          │  코드 작성 필요
          ▼
┌─────────────────────┐
│  index.ts 코드       │  ← import 후 직접 작성하거나
│                     │     자동 생성된 코드를 추가
└─────────────────────┘
```

중요한 점은 import가 **두 가지를 동시에 하지 않는다**는 것입니다. import는 state에 리소스를 등록하고, 대응하는 코드를 생성해주지만 코드를 프로젝트에 자동으로 삽입하지는 않습니다. 생성된 코드를 개발자가 직접 프로젝트에 넣어야 합니다.

---

### 방법 1: CLI를 통한 개별 리소스 import

가장 기본적인 방법입니다. 리소스 하나씩 import합니다.

#### 기본 문법

```bash
pulumi import <pulumi-resource-type> <logical-name> <cloud-resource-id>
```

각 인자의 의미는 다음과 같습니다.

- `pulumi-resource-type`: Pulumi에서의 리소스 타입 (예: `aws:ec2/instance:Instance`)
- `logical-name`: Pulumi 코드에서 사용할 리소스 이름 (개발자가 자유롭게 지정)
- `cloud-resource-id`: AWS에서의 실제 리소스 ID

#### 실전 예시: EC2 인스턴스 import

먼저 import할 리소스의 ID를 확인합니다.

```bash
# AWS CLI로 기존 인스턴스 ID 확인
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=my-web-server" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text

# 결과: i-0abc123def456789
```

import를 실행합니다.

```bash
pulumi import aws:ec2/instance:Instance my-web-server i-0abc123def456789
```

실행하면 Pulumi가 AWS API를 호출해서 해당 인스턴스의 모든 속성을 조회한 후, 다음 두 가지를 수행합니다.

**첫째, state에 리소스를 등록합니다.** 이후 `pulumi stack export`로 확인하면 해당 리소스가 state에 포함되어 있습니다.

**둘째, 대응하는 TypeScript 코드를 터미널에 출력합니다.**

```
Importing (dev-02)

     Type                      Name            Status
 +   pulumi:pulumi:Stack       my-infra-dev-02 created
 =   aws:ec2/instance:Instance my-web-server   imported (1s)

Resources:
    = 1 imported
    1 unchanged
```

아래와 같은 코드가 자동 생성되어 출력됩니다:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const myWebServer = new aws.ec2.Instance("my-web-server", {
    ami: "ami-0abcdef1234567890",
    instanceType: "t3.medium",
    subnetId: "subnet-0123456789abcdef0",
    vpcSecurityGroupIds: ["sg-0123456789abcdef0"],
    keyName: "my-keypair",
    rootBlockDevice: {
        volumeSize: 20,
        volumeType: "gp3",
        encrypted: true,
    },
    tags: {
        Name: "my-web-server",
    },
});
```

이 출력된 코드를 `index.ts`에 복사해 넣어야 합니다. 그래야 다음 `pulumi up` 때 state와 코드가 일치해서 "no changes"가 나옵니다.

#### 다른 리소스 타입 import 예시

```bash
# RDS 인스턴스
pulumi import aws:rds/instance:Instance my-database my-db-instance-id

# S3 버킷
pulumi import aws:s3/bucket:Bucket my-bucket my-bucket-name

# Security Group
pulumi import aws:ec2/securityGroup:SecurityGroup my-sg sg-0123456789abcdef0

# VPC
pulumi import aws:ec2/vpc:Vpc my-vpc vpc-0123456789abcdef0

# SNS Topic
pulumi import aws:sns/topic:Topic my-alerts arn:aws:sns:ap-northeast-2:123456789012:my-alerts

# CloudWatch Alarm
pulumi import aws:cloudwatch/metricAlarm:MetricAlarm my-alarm my-cpu-alarm
```

리소스 타입별로 ID 형식이 다릅니다. EC2는 `i-xxxxx`, SNS는 ARN, RDS는 인스턴스 식별자 등입니다. 각 리소스의 ID 형식은 Pulumi 공식 문서의 해당 리소스 페이지 하단 Import 섹션에서 확인할 수 있습니다.

---

### 방법 2: 코드에서 import 옵션 사용하기

CLI 대신 코드에 `import` 옵션을 명시하고 `pulumi up`을 실행하는 방법도 있습니다.

```typescript
const myServer = new aws.ec2.Instance("my-web-server", {
    ami: "ami-0abcdef1234567890",
    instanceType: "t3.medium",
    subnetId: "subnet-0123456789abcdef0",
    vpcSecurityGroupIds: ["sg-0123456789abcdef0"],
    tags: {
        Name: "my-web-server",
    },
}, {
    import: "i-0abc123def456789",   // ← AWS 리소스 ID
});
```

```bash
pulumi up
# 출력: = aws:ec2/instance:Instance my-web-server imported
```

import가 완료된 후에는 `import` 옵션을 코드에서 제거합니다. 남겨두면 다음 `pulumi up` 때 에러가 발생합니다(이미 state에 존재하므로).

```typescript
// import 완료 후 정리
const myServer = new aws.ec2.Instance("my-web-server", {
    ami: "ami-0abcdef1234567890",
    instanceType: "t3.medium",
    subnetId: "subnet-0123456789abcdef0",
    vpcSecurityGroupIds: ["sg-0123456789abcdef0"],
    tags: {
        Name: "my-web-server",
    },
    // import 라인 삭제
});
```

이 방법은 코드를 개발자가 미리 직접 작성해야 하므로 리소스 속성을 정확히 알고 있을 때 적합합니다. AWS 콘솔에서 속성을 하나하나 확인해야 하는 번거로움이 있지만 코드와 state가 동시에 정리되는 장점이 있습니다.

---

### 방법 3: Bulk Import (JSON 파일 기반 대량 import)

리소스가 많을 때는 JSON 파일에 목록을 작성하고 한 번에 import할 수 있습니다.

```json
// resources.json
{
    "resources": [
        {
            "type": "aws:ec2/vpc:Vpc",
            "name": "main-vpc",
            "id": "vpc-0123456789abcdef0"
        },
        {
            "type": "aws:ec2/subnet:Subnet",
            "name": "public-subnet-a",
            "id": "subnet-0aaa111bbb222ccc3"
        },
        {
            "type": "aws:ec2/subnet:Subnet",
            "name": "public-subnet-b",
            "id": "subnet-0ddd444eee555fff6"
        },
        {
            "type": "aws:ec2/securityGroup:SecurityGroup",
            "name": "web-sg",
            "id": "sg-0123456789abcdef0"
        },
        {
            "type": "aws:ec2/instance:Instance",
            "name": "web-server",
            "id": "i-0abc123def456789"
        },
        {
            "type": "aws:rds/instance:Instance",
            "name": "main-db",
            "id": "my-database-instance"
        }
    ]
}
```

```bash
pulumi import -f resources.json
```

모든 리소스에 대한 코드가 한 번에 생성됩니다. 이 방법은 기존 인프라를 Pulumi로 전환하는 마이그레이션 프로젝트에서 특히 유용합니다.

---

### import 후 필수 작업: 코드-State 동기화

import 직후 가장 중요한 작업은 코드와 state를 일치시키는 것입니다.

```bash
# import 직후 확인
pulumi preview
```

이 preview에서 나올 수 있는 결과는 세 가지입니다.

**"no changes"** — 이상적인 상태입니다. 코드가 실제 리소스와 완벽히 일치합니다.

**"update"가 표시됨** — 코드의 속성값이 실제 리소스와 다릅니다. 이 경우 두 가지 선택지가 있습니다.

```
~ aws:ec2/instance:Instance my-web-server update
    ~ instanceType: "t3.medium" => "t3.large"
```

- **선택지 A**: 코드를 실제 리소스에 맞추기 — 기존 상태를 유지하고 싶다면 코드에서 `t3.medium`을 `t3.large`로 수정합니다.
- **선택지 B**: 리소스를 코드에 맞추기 — 스펙 변경이 목적이었다면 그대로 `pulumi up`을 실행합니다.

**"create" + "delete"가 표시됨** — 코드의 리소스 이름이 state와 매칭되지 않습니다. import 시 지정한 logical name과 코드의 리소스 이름이 동일한지 확인하세요.

---

### 실전 시나리오: dev-01의 SNS를 dev-02로 가져오기

dev-01에서 Pulumi로 배포한 SNS Topic을 dev-02 stack에서도 동일하게 사용하고 싶은 경우입니다.

**방법 A: import로 기존 리소스를 dev-02에 편입**

```bash
# dev-01에서 SNS ARN 확인
pulumi stack select dev-01
pulumi stack output snsTopicArn
# 출력: arn:aws:sns:ap-northeast-2:123456789012:alerts-dev-01

# dev-02에서 해당 리소스를 import
pulumi stack select dev-02
pulumi import aws:sns/topic:Topic alerts-dev-02 \
  arn:aws:sns:ap-northeast-2:123456789012:alerts-dev-01
```

단, 이 방법은 **하나의 리소스를 두 stack이 관리하는 상황**을 만들므로 위험합니다. dev-01에서 `pulumi destroy`를 하면 dev-02가 참조하는 SNS가 삭제됩니다.

**방법 B: Stack Reference (권장)**

리소스를 공유할 때는 import보다 Stack Reference가 안전합니다.

```typescript
// dev-02의 코드
const dev01Ref = new pulumi.StackReference("my-org/my-infra/dev-01");
const sharedSnsArn = dev01Ref.getOutput("snsTopicArn");

// dev-01의 SNS를 alarm action으로 사용
const alarm = new aws.cloudwatch.MetricAlarm(`cpu-alarm-${stack}`, {
    // ...
    alarmActions: [sharedSnsArn],
});
```

---

### import 주의사항 정리

**1) import는 state만 변경하고, 실제 AWS 리소스를 수정하지 않습니다.** 안전한 작업이지만, 이후 `pulumi up`에서 코드와 실제 리소스가 다르면 변경이 발생할 수 있습니다.

**2) import 후 반드시 `pulumi preview`로 drift를 확인하세요.** "no changes"가 나올 때까지 코드를 조정하는 것이 안전합니다.

**3) `protect` 옵션을 적극 활용하세요.** import한 리소스가 실수로 삭제되는 것을 방지합니다.

```typescript
const importedDb = new aws.rds.Instance("legacy-db", {
    // ... 속성들
}, {
    protect: true,  // pulumi destroy 시에도 이 리소스는 삭제하지 않음
});
```

**4) `retainOnDelete` 옵션도 고려하세요.** Pulumi state에서 제거하되 실제 AWS 리소스는 남기고 싶을 때 사용합니다.

```typescript
const importedDb = new aws.rds.Instance("legacy-db", {
    // ... 속성들
}, {
    retainOnDelete: true,  // state에서만 제거, AWS 리소스는 유지
});
```

**5) 리소스 간 의존성 순서를 지켜서 import하세요.** VPC → Subnet → Security Group → EC2 순서로 import해야 참조 관계가 올바르게 설정됩니다.

---

## 한눈에 보는 비교 요약

| 항목 | Config 기반 조건 분기 | pulumi import |
|---|---|---|
| **목적** | 하나의 코드로 여러 환경을 관리 | 기존 AWS 리소스를 Pulumi로 편입 |
| **사용 시점** | 새 Stack 생성 시 환경별 차이 반영 | 수동 생성된 리소스를 IaC로 전환 |
| **변경 대상** | Config yaml + 코드 내 분기 로직 | Pulumi state + 대응 코드 추가 |
| **위험도** | 낮음 (preview로 사전 확인 가능) | 중간 (코드-state 불일치 주의) |
| **함께 쓰는 경우** | import 후 config 분기로 관리하면 가장 이상적 |

이 두 가지 기능을 조합하면 "기존 AWS 리소스를 import로 가져온 뒤, config 분기로 stack별 차이를 관리"하는 완전한 IaC 전환이 가능합니다.
