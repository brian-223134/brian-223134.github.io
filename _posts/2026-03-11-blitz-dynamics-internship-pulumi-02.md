---
title: "[인턴] Pulumi 모니터링 코드 구조 리팩터링: SSM/Agent 분리와 기능별 디렉토리 구성"
excerpt: "기본 템플릿에 포함되어 있던 SSM·CloudWatch Agent 구성을 분리하고, Pulumi 코드를 기능별 디렉토리로 재구성한 과정을 정리합니다."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Pulumi, CloudWatch, SSM, IaC]

permalink: /intern/company/blitz-dynamics/pulumi-refactoring/01/

toc: true
toc_sticky: true

date: 2026-03-11
last_modified_at: 2026-03-11
---

# 들어가며

[이전 포스트](/intern/company/blitz-dynamics/pulumi-cloudwatch-slack/)에서는 Pulumi로 CloudWatch 대시보드, 알람, Slack 알림을 어떻게 구성할 수 있는지 전체 흐름을 정리했다.

이번 글에서는 그 다음 단계로, 실제 Pulumi 코드를 어떻게 더 읽기 쉽고 유지보수하기 쉬운 구조로 정리했는지를 다룬다.

초기에 전달받은 기본 포맷에는 이미 **SSM**과 **CloudWatch Agent** 관련 구성이 기본값처럼 포함되어 있었다. 빠르게 시작하기에는 좋은 구조였지만, 현재 인턴십 과제 범위와는 약간 어긋나는 부분이 있었다.

현재 우선순위는 다음과 같았다.

1. **AWS 기본 제공 메트릭 기반 모니터링**을 먼저 안정적으로 구축한다.
2. **CloudWatch Dashboard / Alarm / Slack 알림 흐름**을 명확하게 완성한다.
3. **CloudWatch Agent, SSM 연동은 선택적 기능**으로 분리해 이후 단계에서 확장 가능하게 만든다.

즉, 이번 리팩터링의 핵심은 단순히 파일을 나누는 것이 아니라, **기본 기능과 선택 기능의 경계를 분명히 하는 것**이었다.

---

# 1. 왜 기본 포맷을 그대로 쓰지 않았는가

## 1-1. 기본 포맷의 장점

사수님이 작성해주신 기본 포맷은 빠르게 구조를 이해하고 바로 개발을 시작하는 데 매우 유용했다.

- Pulumi 프로젝트의 기본 진입점이 이미 잡혀 있었다.
- 모니터링에 필요한 리소스들이 어떤 식으로 연결되는지 큰 그림을 볼 수 있었다.
- SSM, CloudWatch Agent까지 고려한 확장 방향이 미리 반영되어 있었다.

즉, "어디서부터 시작해야 하는가"를 고민할 필요가 없었다는 점에서 좋은 출발점이었다.

## 1-2. 현재 과제 범위와의 차이

문제는 **지금 당장 필요한 것**과 **나중에 붙일 기능**이 한 구조 안에 같이 들어가 있었다는 점이다.

이전 글에서 정리했듯이, 현재 단계에서 우선 사용하는 메트릭은 `AWS/EC2`, `AWS/RDS` 같은 **AWS 기본 제공 메트릭**이다. 반면 CloudWatch Agent는 EC2 내부 메모리, 디스크 사용률 같은 **추가 커스텀 메트릭**을 수집할 때 필요하다.

즉, 다음과 같이 성격이 달랐다.

| 영역 | 현재 필수 여부 | 설명 |
|---|---|---|
| CloudWatch Alarm | 필수 | 기본 메트릭 임계값 감지 |
| CloudWatch Dashboard | 필수 | 메트릭 시각화 |
| Slack Notification | 필수 | 운영 알림 전달 |
| SSM | 선택 | 에이전트 설치/명령 전달 자동화 |
| CloudWatch Agent | 선택 | 메모리/디스크 등 추가 메트릭 수집 |

필수 경로와 선택 경로가 섞이면, 코드를 처음 읽는 사람이 "이 기능이 반드시 필요한가?"를 헷갈리기 쉽다.

## 1-3. 분리가 필요했던 이유

기본 포맷을 그대로 유지하면 다음 문제가 생길 수 있었다.

- 아직 사용하지 않는 Agent 관련 코드가 기본 모니터링의 일부처럼 보인다.
- SSM 변경이 알람/대시보드 영역과 같은 수준에서 섞여 읽힌다.
- 추후 기능 추가 시 의존 관계가 불분명해진다.
- 특정 기능만 수정하려고 해도 전체 파일을 함께 읽어야 한다.

결국 구조를 단순화하려면, "처음부터 모든 것을 한 번에 넣는 방식"보다 **현재 운영 범위에 맞는 최소 코어를 만들고, 확장 기능은 모듈로 분리하는 방식**이 더 적절했다.

---

# 2. SSM과 CloudWatch Agent를 기본값에서 분리하기

## 2-1. SSM과 CloudWatch Agent는 왜 따로 봐야 하는가?

두 기능은 서로 관련은 있지만, 역할이 완전히 같지는 않다.

- **SSM**: EC2 인스턴스에 명령을 전달하거나 설정을 배포하기 위한 관리 채널
- **CloudWatch Agent**: 인스턴스 내부 메트릭을 수집하여 CloudWatch로 보내는 에이전트

실무에서는 SSM을 통해 Agent를 설치하거나 설정 파일을 배포하는 흐름이 흔하지만, 그렇다고 해서 **"기본 모니터링 = SSM + Agent"**가 되는 것은 아니다.

현재 과제 기준의 기본 모니터링은 아래만으로도 충분히 동작한다.

```text
[기본 모니터링 경로]
EC2 / RDS 기본 메트릭
  -> CloudWatch Alarm
  -> SNS / Lambda
  -> Slack

  -> CloudWatch Dashboard
```

반면 Agent 기반 확장은 다음 단계의 과제다.

```text
[확장 모니터링 경로]
EC2 내부 메모리/디스크 메트릭
  -> CloudWatch Agent
  -> CWAgent Namespace
  -> 추가 Dashboard / Alarm
```

즉, Agent 경로는 "기본 경로 위에 덧붙는 확장 기능"으로 보는 편이 맞다.

## 2-2. 분리 후 얻는 장점

SSM과 CloudWatch Agent를 기본값에서 분리하면 다음 장점이 생긴다.

- **가독성 향상**: 지금 반드시 읽어야 하는 코드와 나중에 볼 코드를 구분할 수 있다.
- **점진적 확장**: 먼저 기본 메트릭만 배포하고, 이후 Agent를 붙이는 단계적 적용이 가능하다.
- **변경 범위 축소**: Agent 설정을 바꿔도 알람/대시보드 핵심 코드에 영향이 적다.
- **설계 의도 명확화**: 어떤 기능이 코어이고 어떤 기능이 옵션인지 코드 구조가 설명해 준다.

## 2-3. 설정값으로 선택 가능하게 만들기

Pulumi에서는 이런 선택적 기능을 `config`로 제어하는 방식이 자연스럽다.

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();

const enableSsm = config.getBoolean("enableSsm") ?? false;
const enableCloudWatchAgent = config.getBoolean("enableCloudWatchAgent") ?? false;

if (enableSsm) {
  // SSM 관련 리소스 생성
}

if (enableCloudWatchAgent) {
  // CloudWatch Agent 관련 리소스 생성
}
```

이렇게 하면 같은 코드베이스를 유지하면서도, 스택이나 환경에 따라 필요한 기능만 켤 수 있다.

---

# 3. 기능별 디렉토리로 구조 재정리하기

## 3-1. 디렉토리를 나눈 기준

이번 리팩터링에서는 코드를 다음과 같은 기능 단위로 나누었다.

- `agent`
- `alarms`
- `common`
- `dashboard`
- `instance`
- `notifications`

핵심 기준은 **AWS 서비스 기준**이 아니라 **역할 기준**이다. 예를 들어 SNS와 Lambda는 AWS 서비스로는 다르지만, 이 프로젝트에서는 둘 다 "알림 전달"이라는 하나의 기능에 속한다. 그래서 `notifications`로 묶는 편이 더 읽기 쉽다.

## 3-2. 리팩터링 후 디렉토리 예시

```text
.
├── index.ts
├── agent/
│   ├── cloudwatch-agent.ts
│   └── ssm.ts
├── alarms/
│   ├── ec2-alarms.ts
│   └── rds-alarms.ts
├── common/
│   ├── config.ts
│   ├── tags.ts
│   └── naming.ts
├── dashboard/
│   └── monitoring-dashboard.ts
├── instance/
│   └── monitored-instance.ts
└── notifications/
    ├── slack-lambda.ts
    ├── sns-topic.ts
    └── subscription.ts
```

이 구조에서 `index.ts`는 모든 세부 구현을 품는 거대한 파일이 아니라, 각 기능 모듈을 조합하는 **오케스트레이션 계층**의 역할만 담당한다.
(해당 구조는 일부 네이밍을 변경하여 작성하였기 때문에 약간 다르다)

## 3-3. 디렉토리별 역할

### `common`

반복되는 설정과 공통 로직을 둔다.

- Pulumi `Config` 로딩
- 공통 태그
- 리소스 이름 규칙
- 환경별 상수

이 영역을 분리하면 하드코딩이 줄고, 여러 모듈에서 같은 값을 일관되게 사용할 수 있다.

### `instance`

어떤 EC2 인스턴스를 모니터링 대상으로 삼는지, 어떤 Dimension을 사용하는지를 정리한다.

예를 들어 인스턴스 ID, 이름, 환경 태그 같은 값을 이 영역에서 관리하면 알람과 대시보드가 같은 대상을 바라보게 만들기 쉽다.

### `notifications`

알람이 발생했을 때 외부로 전달되는 경로를 담당한다.

- SNS Topic 생성
- Lambda 함수 생성
- SNS-Lambda 구독 연결
- Slack Webhook 메시지 포맷 구성

즉, "알림을 어디로 어떻게 보낼 것인가"에 대한 관심사를 한 곳에 모은다.

### `alarms`

CPU, 네트워크, 상태 체크, RDS 지표 등 실제 알람 규칙을 생성하는 영역이다.

이 영역은 메트릭, 임계값, 평가 기간 같은 **운영 정책**이 반영되는 곳이므로, 다른 영역과 섞이지 않도록 분리하는 것이 좋다.

### `dashboard`

CloudWatch 대시보드 위젯 조합을 담당한다.

알람 생성 코드와 대시보드 생성 코드를 한 파일에 섞으면 읽을 때 맥락 전환이 잦아진다. 둘 다 CloudWatch를 사용하더라도, 실제 관심사는 다르다.

- 알람: 이상 탐지와 대응
- 대시보드: 시각화와 상태 확인

그래서 별도 디렉토리로 분리했다.

### `agent`

현재는 선택 기능으로 두는 영역이다.

- CloudWatch Agent 설정 파일 관리
- SSM 문서 또는 명령 실행
- Agent 설치 및 재시작 자동화

앞으로 메모리 사용률, 디스크 사용률까지 모니터링 범위를 넓힐 때 이 디렉토리가 중심 역할을 하게 된다.

---

# 4. 리팩터링 이후 `index.ts`의 역할

리팩터링 전에는 `index.ts`가 모든 리소스를 직접 생성하는 중심 파일이 되기 쉽다. 하지만 파일이 커질수록 "무엇을 생성하는지"보다 "어디에 무엇이 섞여 있는지"를 먼저 파악해야 하는 상태가 된다.

리팩터링 후에는 `index.ts`를 다음처럼 얇게 유지하는 것이 목표다.

```typescript
import { loadConfig } from "./common/config";
import { createNotificationResources } from "./notifications/sns-topic";
import { createEc2Alarms } from "./alarms/ec2-alarms";
import { createDashboard } from "./dashboard/monitoring-dashboard";
import { setupCloudWatchAgent } from "./agent/cloudwatch-agent";

const app = loadConfig();
const notifications = createNotificationResources(app);
const alarms = createEc2Alarms(app, notifications);

createDashboard(app, alarms);

if (app.enableCloudWatchAgent) {
  setupCloudWatchAgent(app);
}
```

핵심은 `index.ts`에서 세부 구현을 설명하지 않고, **인프라가 어떤 단계로 조합되는지만 보여주는 것**이다.

이렇게 하면 처음 코드를 읽는 사람도 다음 순서로 빠르게 이해할 수 있다.

1. 설정을 읽는다.
2. 알림 리소스를 만든다.
3. 알람을 만든다.
4. 대시보드를 만든다.
5. 필요하면 Agent를 붙인다.

이 순서는 실제 운영 관점의 사고 흐름과도 거의 같다.

---

# 5. 이번 구조 변경에서 중요했던 설계 기준

이번 리팩터링에서 가장 중요하게 본 기준은 세 가지다.

## 5-1. 기능의 생명주기가 같은가?

항상 같이 생성되고 같이 수정되는 리소스는 같은 모듈에 둘 수 있다. 반대로, 어떤 기능은 자주 바뀌고 어떤 기능은 거의 바뀌지 않는다면 분리하는 편이 낫다.

- Slack 알림 메시지 포맷은 비교적 자주 바뀔 수 있다.
- EC2 대상 인스턴스 목록도 환경에 따라 자주 바뀔 수 있다.
- SSM/Agent 도입 여부는 현재 시점에서는 아예 꺼져 있을 수 있다.

이 차이를 무시하고 한 파일에 몰아넣으면 변경 비용이 높아진다.

## 5-2. 읽는 사람이 한 번에 이해할 수 있는가?

좋은 구조는 작성자보다 **다음에 읽을 사람**에게 친절해야 한다.

예를 들어 알람 정책을 수정하려는 사람이 있다면 `alarms/`만 보면 충분해야 한다. Slack 메시지를 바꾸려는 사람은 `notifications/`만 보면 충분해야 한다.

구조가 이렇게 동작하면 코드 탐색 비용이 크게 줄어든다.

## 5-3. 나중에 확장하기 쉬운가?

현재는 EC2 중심이지만, 이후에는 다음 같은 확장이 자연스럽게 생길 수 있다.

- 여러 EC2 인스턴스 일괄 모니터링
- RDS 알람 세분화
- Agent 기반 메모리/디스크 알람 추가
- Datadog 또는 다른 알림 채널 연동

기능별 디렉토리 구조는 이런 확장을 "기존 구조를 깨지 않고" 수용하기 좋다.

---

# 6. Pulumi 공식 베스트 프랙티스와 연결해서 보기

Pulumi 공식 블로그의 [IaC Best Practices: Structuring Pulumi Projects](https://www.pulumi.com/blog/iac-best-practices-structuring-pulumi-projects/)를 읽어보면, 프로젝트 구조를 잡을 때 중요한 기준이 꽤 명확하게 정리되어 있다.

이 글의 핵심은 "무조건 하나의 프로젝트가 좋다" 혹은 "무조건 여러 프로젝트로 쪼개야 한다"가 아니라, **조직과 인프라의 맥락에 따라 구조가 달라져야 한다**는 것이다.

## 6-1. Pulumi가 제시한 주요 판단 기준

Pulumi는 프로젝트를 나누거나 구조를 설계할 때 다음 요소를 보라고 제안한다.

| 기준 | 의미 |
|---|---|
| **Use case** | 서로 다른 애플리케이션/목적이라면 분리할 수 있음 |
| **Team structure** | 담당 팀이 다르면 프로젝트 분리가 유리할 수 있음 |
| **Security** | 권한 경계와 RBAC를 어디까지 나눌지 고려 |
| **Resource relationship** | 강하게 의존하는 리소스는 함께 관리하는 편이 좋음 |
| **Resource lifecycle** | 같이 생성/수정/삭제되는 리소스는 같은 단위로 묶기 좋음 |
| **Change rate** | 자주 바뀌는 리소스와 거의 안 바뀌는 리소스를 분리할 수 있음 |
| **Blast radius** | 실수했을 때 영향을 받는 범위를 줄이는 구조가 유리 |

이 기준들은 "프로젝트 개수"를 정할 때만 쓰이는 게 아니라, **하나의 Pulumi 프로젝트 내부를 어떻게 모듈화할지**를 결정할 때도 그대로 적용할 수 있다.

## 6-2. 이번 리팩터링과 연결되는 지점

이번 구조 변경은 아직 **멀티 프로젝트 분리**까지 간 것은 아니다. 하지만 Pulumi가 말하는 기준을 현재 코드베이스 안에 먼저 적용한 셈이다.

- **Resource lifecycle**: 알람/대시보드와 Agent는 같은 시점에 항상 변경되지 않는다.
- **Change rate**: Slack 메시지 포맷, 알람 기준, Agent 설정은 변경 속도가 서로 다르다.
- **Blast radius**: Agent 설정을 바꾸다가 기본 알람 경로까지 건드리는 구조는 피하고 싶다.
- **Resource relationship**: SNS, Lambda, Slack Webhook은 모두 알림이라는 하나의 기능으로 보는 편이 자연스럽다.

즉, 이번 리팩터링은 "프로젝트를 쪼개기 전에 먼저 내부 구조부터 경계가 드러나게 만든 작업"이라고 볼 수 있다.

## 6-3. 앞으로의 확장 방향

Pulumi 글에서는 조직이 커지고 책임 영역이 분리되면, 하나의 프로젝트에서 여러 프로젝트로 나누는 방향도 자연스럽다고 설명한다.

현재 수준에서는 기능별 디렉토리 분리만으로도 충분하지만, 앞으로 모니터링 범위가 더 커지면 다음 단계도 가능하다.

- 공용 인프라 프로젝트
- 모니터링 프로젝트
- 애플리케이션 프로젝트

그리고 프로젝트 간에는 `StackReference`로 필요한 출력값을 공유하는 방식으로 발전시킬 수 있다.

또 하나 인상적이었던 점은, **코드의 파일 시스템 위치를 바꾸는 것 자체는 Pulumi 리소스에 직접적인 영향을 주지 않는다**는 설명이다. 즉, Pulumi 프로젝트 이름과 스택 이름이 유지된다면, 내부 파일 구조를 더 좋은 방향으로 리팩터링하는 것은 충분히 안전하게 시도할 수 있다.

이번 디렉토리 재구성도 바로 그런 종류의 개선이라고 볼 수 있다.

---

# 7. 마무리

이번 리팩터링의 핵심은 두 가지다.

- **SSM / CloudWatch Agent를 기본 기능에서 분리하여 선택 기능으로 만든 것**
- **Pulumi 코드를 기능별 디렉토리로 나누어 책임 경계를 분명히 한 것**

정리하면, 처음 받은 기본 포맷은 좋은 출발점이었고, 이번 작업은 그 포맷을 현재 과제 범위에 맞게 더 실용적으로 다듬는 과정이었다.

특히 Pulumi 공식 베스트 프랙티스 글을 함께 읽고 나니, 이번 작업은 단순한 폴더 정리가 아니라 다음 원칙을 코드에 반영한 것에 가깝다는 생각이 들었다.

- 같이 변하는 것은 같이 둔다.
- 지금 필요 없는 것은 기본 경로에서 뺀다.
- 기능 단위로 구조를 드러내서 읽는 비용을 줄인다.
- 이후 프로젝트 분리까지도 가능한 방향으로 정리한다.

다음 단계에서는 이 구조를 바탕으로 `ComponentResource`를 도입하거나, 여러 인스턴스/환경에 공통 적용할 수 있는 재사용 가능한 모니터링 모듈로 발전시켜볼 수 있다.