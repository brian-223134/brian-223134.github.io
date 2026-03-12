---
title: "[인턴] Pulumi의 State 관리와 배포 방법 정리"
excerpt: "Pulumi의 상태 관리 단위인 Stack의 관리 및 다중 Stack의 기초적인 관리 방식을 설명."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Pulumi, AWS, Stack]

permalink: /intern/company/blitz-dynamics/07/

toc: true
toc_sticky: true

date: 2026-03-12
last_modified_at: 2026-03-12
---


> 로컬 backend, `dev` stack, CloudWatch Alarm 수정 시나리오, multi-stack 운영까지

## 1. 들어가며

최근 AWS 모니터링 인프라를 Pulumi로 다루는 과정을 정리하면서, 단순히 `pulumi up` 명령만 아는 것으로는 부족하다는 점을 느꼈다. (사수님이 추가로 공부하라는 지시도 있었다.)  
특히 아래와 같은 질문들이 자연스럽게 따라왔다.

- Pulumi의 **state**는 정확히 무엇인가?
- 로컬 backend를 쓰고 있을 때 state는 어떻게 관리해야 하는가?
- 코드 리뷰 승인이 끝난 뒤 `pulumi up` 전에는 무엇을 확인해야 하는가?
- 이미 배포한 CloudWatch Alarm을 수정하거나 교체할 때 Pulumi는 어떻게 동작하는가?
- `dev` 하나만 쓰는 것이 아니라 `dev-01`, `dev-02`처럼 여러 stack을 운용할 때는 어떻게 구조를 잡아야 하는가?

이 글은 위 질문들에 대한 답을 한 번에 정리한 글이다.  
예시는 다음과 같은 모니터링 시나리오를 기준으로 설명한다.

- EC2의 `AWS/EC2` 기본 namespace metric 활용
- CloudWatch Alarm 생성
- SNS Topic으로 알림 전달
- Amazon Q Developer in chat applications(구 AWS Chatbot)을 통해 Slack으로 전달

---

## 2. Pulumi에서 State란 무엇인가

Pulumi에서 state는 현재 인프라를 관리하기 위한 **기준 정보**다.  
단순히 “배포 로그”가 아니라, Pulumi가 자신이 관리하는 리소스를 추적하기 위해 보관하는 메타데이터라고 이해하면 된다.

예를 들어 state에는 아래와 같은 정보가 반영될 수 있다.

- 어떤 리소스를 생성했는지
- 해당 리소스의 provider 정보
- 리소스 간 의존 관계
- 현재 stack의 output 값
- config 값
- secret 처리된 값

Pulumi는 `pulumi preview`나 `pulumi up` 실행 시,  
현재 코드가 선언한 desired state와 기존 state를 비교해서 다음 작업을 계산한다.

즉, Pulumi는 다음과 같은 흐름으로 동작한다.

1. 현재 stack의 state를 읽는다.
2. TypeScript 코드가 선언한 리소스 구성을 해석한다.
3. 둘의 차이를 비교한다.
4. create / update / delete / replace 계획을 계산한다.
5. `pulumi up`이 성공하면 새로운 state snapshot을 저장한다.

결국 Pulumi에서 state는 **다음 배포를 위한 기준점**이다.

---

## 3. Backend와 State 저장 방식

Pulumi는 stack별 state를 backend에 저장한다.  
여기서 backend는 state를 저장하고 관리하는 저장소라고 생각하면 된다.

대표적으로 다음과 같은 방식이 있다.

### 3.1 Pulumi Cloud
Pulumi가 제공하는 managed backend다.

장점:
- 팀 협업에 유리
- 감사 이력과 UI 제공
- backend 운영 부담 감소
- state 관리와 협업 흐름이 단순함

### 3.2 DIY Backend
직접 state 저장소를 관리하는 방식이다.

예:
- local filesystem
- AWS S3
- Azure Blob
- GCS
- PostgreSQL

현재처럼 `pulumi login --local`을 사용했다면 local backend를 쓰는 상태라고 보면 된다.

---

## 4. 로컬 backend를 사용하는 현재 상황 해석

내 상황을 정리하면 다음과 같다.

- 로컬 backend 사용 중
- `dev` stack 생성 완료
- TypeScript 코드 구현 완료
- 코드 리뷰 및 승인 완료
- 아직 `pulumi up`은 하지 않음

이 상태는 **실제 AWS에 리소스가 생성된 상태가 아니라**,  
Pulumi가 “이 코드를 배포하면 어떤 리소스를 만들지” 계산할 수 있는 상태다.

즉:

- `pulumi preview` → 계획만 본다
- `pulumi up` → 실제로 AWS에 적용한다
- 적용 성공 후 → state가 기록된다

아직 `up`을 하지 않았다면, Pulumi 입장에서는 해당 stack에 대한 배포 기록이 실제로 쌓이지 않은 상태라고 볼 수 있다.

---

## 5. `pulumi preview`와 `pulumi up`의 차이

두 명령은 매우 자주 같이 언급되지만 역할은 분명히 다르다.

### 5.1 `pulumi preview`
현재 state와 코드의 차이를 계산해서 **무슨 일이 일어날지 미리 보여주는 명령**이다.

예를 들어 아래처럼 볼 수 있다.

- 새 Alarm 생성 예정
- SNS Topic 생성 예정
- Slack 연동 리소스 생성 예정
- 기존 Alarm 삭제 예정
- 특정 리소스 replace 예정

즉, 실제 반영 없이 계획만 확인한다.

### 5.2 `pulumi up`
preview로 계산한 내용을 실제 클라우드에 적용하는 명령이다.

적용이 끝나면 Pulumi는 새로운 state snapshot을 저장한다.

---

## 6. 코드 리뷰 승인 이후 `up` 전에 하면 좋은 것

코드 리뷰 승인이 끝났다면, 바로 `pulumi up`을 치는 것보다 아래 흐름이 더 안정적이다.

```bash
pulumi stack select dev
pulumi preview --diff
pulumi up
```

조금 더 엄격하게 관리하려면 **update plan**을 사용하는 방식도 좋다.

```bash
pulumi stack select dev
pulumi preview --diff --save-plan=dev.plan.json
pulumi up --plan=dev.plan.json
```

이 방식의 장점은 다음과 같다.

- 리뷰 시점의 변경 계획을 고정할 수 있음
- 승인된 diff만 배포한다는 의미를 갖기 쉬움
- 팀 단위 검증 흐름에 잘 맞음

즉, 리뷰가 끝났지만 아직 배포 전인 상황에서는  
단순 preview보다 **plan 기반 배포**가 더 설명력 있는 운영 방식이 될 수 있다.

---

## 7. `refresh`는 왜 중요한가

Pulumi는 항상 클라우드의 현재 상태를 자동으로 전부 다시 읽어오지 않는다.  
기본적으로는 **자신이 저장해 둔 state를 기준**으로 동작한다.

문제는 다음과 같은 상황이다.

- 누군가 AWS Console에서 Alarm을 수동 수정함
- 다른 도구가 SNS 설정을 바꿈
- Slack 연동 리소스를 콘솔에서 변경함

이 경우 Pulumi의 state와 실제 AWS 상태가 어긋날 수 있다.

이럴 때 사용하는 것이 `pulumi refresh`다.

```bash
pulumi refresh
```

중요한 점은:

- `refresh`는 **state를 실제 리소스 상태에 맞게 동기화**
- 하지만 **TypeScript 코드를 수정하지는 않음**

따라서 외부에서 바뀐 값을 앞으로도 유지하고 싶다면  
refresh만으로 끝내면 안 되고 코드도 같이 맞춰야 한다.

---

## 8. Alarm → SNS → Slack 구조를 Pulumi로 다룰 때

모니터링 알림 구조를 단순화하면 보통 아래와 같다.

1. CloudWatch Alarm 생성
2. Alarm action으로 SNS Topic 연결
3. Amazon Q Developer in chat applications을 Slack 채널과 연결
4. SNS Topic을 통해 Slack 알림 전달

이 구조를 Pulumi로 관리하면, 결국 아래와 같은 리소스들이 stack state의 관리 대상이 된다.

- CloudWatch Metric Alarm
- SNS Topic
- SNS Topic subscription 또는 관련 연결 구성
- Amazon Q Developer / Slack 채널 연동 설정
- IAM role 또는 권한 설정(필요 시)

즉, “알람 하나만 수정하는 것 같아도” 실제로는 연결된 리소스 영향 범위를 함께 봐야 한다.

---

## 9. 배포 후 기존 Alarm을 지우고 다른 Alarm으로 교체하려는 경우

이제 가장 중요한 시나리오를 정리해보자.

가정:
- `dev` stack에서 EC2 관련 CloudWatch Alarm을 만들었다.
- 이후 기존 알람을 제거하고 다른 metric 기반의 알람으로 교체하고 싶다.

이 경우를 두 가지로 나눠서 봐야 한다.

---

## 10. 같은 `dev` stack에서 수정하는 경우

이 경우는 **기존 state를 그대로 이어받아 수정하는 것**이다.

예를 들어 기존에 아래 리소스를 관리하고 있었다고 하자.

- `cpuHighAlarm`
- `alarmTopic`
- `slackChannelConfig`

이후 코드에서:

- `cpuHighAlarm` 제거
- `networkOutAlarm` 추가

이렇게 바꾸면 Pulumi는 기존 `dev` stack state와 새 코드를 비교해서 아래 중 하나로 계산한다.

- 기존 Alarm 삭제 + 새 Alarm 생성
- update 가능하면 update
- immutable 속성이 있으면 replace

즉, 같은 stack에서는 “이전에 내가 만들었던 리소스”를 알고 있기 때문에  
그것을 **수정 / 삭제 / 교체**하는 식으로 동작한다.

실무 흐름은 대체로 아래와 같다.

```bash
pulumi stack select dev
pulumi preview --diff --refresh
pulumi up
```

### 같은 stack에서 주의할 점

#### 1) 논리 이름 변경
리소스의 이름이나 경로를 바꾸면 Pulumi가 기존 리소스를 다른 것으로 판단할 수 있다.  
이 경우 단순 rename이 아니라 replace처럼 보일 수 있으므로 주의해야 한다.

#### 2) exact name 사용
CloudWatch Alarm처럼 실제 물리 이름을 직접 지정하는 경우,  
새 리소스를 먼저 만들고 나중에 기존 리소스를 지우는 전략이 항상 가능하지 않을 수 있다.  
이름 충돌 때문에 삭제 후 생성 순서가 필요할 수도 있다.

#### 3) preview 확인 필수
같은 stack에서 수정할 때는 “내가 생각한 수정”이 아니라  
Pulumi가 “삭제 / 생성 / replace” 중 무엇으로 해석했는지를 preview로 확인해야 한다.

정리하면, 같은 stack에서의 변경은  
**기존 stack state를 기반으로 한 변경 배포**다.

---

## 11. 다른 stack에서 진행하는 경우

“다른 stack에서 작업한다”는 말은 사실 두 의미로 나뉜다.

### 11.1 별도 환경으로 새로 배포하는 경우
예:
- `dev`는 그대로 두고
- `dev-02` 또는 `staging`에서 새로운 알람 구성을 시험

이 경우 stack은 완전히 분리된 배포 단위다.

즉:

- `dev` state는 그대로 유지
- `dev-02` state는 별도로 존재
- 같은 코드라도 다른 설정으로 독립 배포 가능

이 방식은 실험용, 검증용, 개발자별 sandbox 용도로 좋다.

다만 이 경우에도 실제 AWS에서 **물리 리소스 이름 충돌**은 피해야 한다.  
예를 들어 같은 계정/리전에서 AlarmName을 동일하게 쓰면 충돌하거나 의도치 않은 중복 관리 문제가 생길 수 있다.

따라서 보통은 stack 이름을 리소스 이름에 반영한다.

예:
- `ec2-cpu-high-dev-01`
- `ec2-cpu-high-dev-02`

---

### 11.2 기존 리소스의 관리 주체를 다른 stack으로 옮기고 싶은 경우
이건 단순히 “다른 stack에서 테스트한다”와는 다르다.

예를 들어:

- 원래 `dev` stack이 Alarm을 관리하고 있었는데
- 앞으로는 `monitoring-dev` stack이 그 Alarm을 관리하게 하고 싶다

이 경우에는 단순 설정 파일만 바꾸는 것으로는 부족할 수 있다.  
왜냐하면 기존 state의 소유권이 여전히 `dev` stack에 있기 때문이다.

이 상황에서는 다음과 같은 개념이 필요하다.

- `pulumi state move`
- 필요 시 `pulumi import`

즉, 다른 stack으로 넘어간다는 것은 경우에 따라  
**새로 만들기**인지, **기존 리소스의 state ownership을 옮기기**인지 구분해야 한다.

---

## 12. 여러 stack을 쓰는 경우: `dev-01`, `dev-02`는 어떻게 관리해야 하나

IaC를 하다 보면 항상 같은 stack에서만 작업하지는 않는다.  
예를 들어 다음과 같은 상황이 생길 수 있다.

- 개발자 A는 `dev-01`
- 개발자 B는 `dev-02`
- 공통 검증은 `staging`
- 실제 운영은 `prod`

이런 구조는 자연스럽고, Pulumi도 이를 잘 지원한다.

핵심은 다음이다.

> **같은 TypeScript 코드 + stack별 다른 설정 + stack별 별도 state**

즉, 보통은 아래처럼 이해하면 된다.

- **코드베이스는 동일**
- **`Pulumi.<stack>.yaml` 설정 파일은 stack마다 별도**
- **state도 stack마다 독립적**

예시 구조:

```text
monitoring/
├── Pulumi.yaml
├── index.ts
├── Pulumi.dev-01.yaml
├── Pulumi.dev-02.yaml
├── Pulumi.staging.yaml
└── Pulumi.prod.yaml
```

---

## 13. 같은 코드, 다른 설정 파일이라는 이해는 맞는가?

결론부터 말하면 **대체로 맞다.**

Pulumi에서 stack은 **같은 프로그램의 독립적이고 개별 설정 가능한 인스턴스**다.  
그래서 보통은 같은 TypeScript 코드에 대해 stack마다 다른 설정 파일을 사용한다.

예를 들어 코드에서는 설정만 읽는다.

```ts
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();

const region = config.require("region");
const alarmThreshold = config.requireNumber("cpuAlarmThreshold");
const slackChannel = config.require("slackChannel");
```

그리고 stack별 설정은 아래처럼 달라질 수 있다.

```yaml
# Pulumi.dev-01.yaml
config:
  aws:region: ap-northeast-2
  monitoring:cpuAlarmThreshold: "70"
  monitoring:slackChannel: "#dev-01-alerts"
```

```yaml
# Pulumi.dev-02.yaml
config:
  aws:region: ap-northeast-2
  monitoring:cpuAlarmThreshold: "85"
  monitoring:slackChannel: "#dev-02-alerts"
```

즉, **리소스 구조는 같고 값만 다르면**  
같은 코드 + 다른 설정 파일 구조가 가장 자연스럽다.

---

## 14. 그렇다면 언제 stack만 늘리면 되고, 언제 project를 분리해야 하나

여기서 중요한 운영 판단이 필요하다.

### stack만 늘리면 되는 경우
- 리소스 구조는 동일함
- 환경별로 threshold, region, channel, instanceId 정도만 다름
- 같은 팀이 같은 배포 단위를 관리함

예:
- `dev-01`, `dev-02`, `staging`, `prod`

### project를 분리하는 게 더 나은 경우
- 리소스 구조 자체가 많이 다름
- 코드에서 stack 이름별 분기가 지나치게 많아짐
- 공통 인프라와 서비스 인프라의 책임 팀이 다름
- 배포 주기가 크게 다름
- preview/up의 영향 범위가 너무 커짐

예를 들어 아래처럼 쪼갤 수 있다.

- `base-infra` 프로젝트
- `monitoring` 프로젝트
- `service-app` 프로젝트

즉, **stack은 배포 인스턴스 단위**,  
**project는 코드와 책임 범위 단위**라고 이해하면 관리가 편하다.

---

## 15. multi-stack 운영 시 추천 원칙

### 15.1 stack 이름 규칙 통일
예:
- `dev-01`
- `dev-02`
- `staging`
- `prod`

혹은 팀/목적 기반:
- `feature-a-dev`
- `feature-b-dev`
- `monitoring-dev`
- `monitoring-prod`

### 15.2 리소스 이름에 stack suffix 반영
같은 계정/리전에서 여러 stack이 공존할 수 있으므로  
가능하면 리소스 이름에도 stack 정보를 넣는 것이 좋다.

예:

```ts
const stack = pulumi.getStack();

const topicName = `cw-alarm-topic-${stack}`;
const alarmName = `ec2-cpu-high-${stack}`;
```

### 15.3 공통 코드는 유지하고 설정만 분리
구조가 동일하다면 하나의 TypeScript 코드베이스를 유지하는 편이 관리가 쉽다.

### 15.4 `preview`를 배포 전 기본 습관으로
stack을 자주 바꿔가며 작업할수록  
현재 선택된 stack이 맞는지, 어떤 리소스를 수정하려는지 확인이 중요하다.

```bash
pulumi stack select dev-02
pulumi preview --diff
pulumi up
```

### 15.5 state ownership을 명확히 구분
같은 실제 리소스를 여러 stack이 동시에 관리하려 하면 문제가 생긴다.  
“이 리소스는 어느 stack이 소유하는가”를 명확히 해야 한다.

---

## 16. 내 상황에 대한 현실적인 추천 구조

현재 상황에서 가장 자연스러운 구조는 아래다.

### 프로젝트
- `monitoring`

### stack
- `dev-01`
- `dev-02`
- `staging`
- `prod`

### 코드베이스
- `index.ts` 또는 관련 Pulumi TypeScript 코드 하나 유지

### stack별 설정으로 분리할 것
- AWS region
- 대상 인스턴스 혹은 태그
- CloudWatch alarm threshold
- SNS topic 이름
- Slack 채널
- 필요 시 enable / disable 옵션

즉, 현재 단계에서는 **같은 TypeScript 코드 + stack별 설정 파일 분리** 전략이 가장 적절하다.

---

## 17. 배포/수정 시 추천 명령 흐름 정리

### 최초 배포 전
```bash
pulumi stack select dev
pulumi preview --diff --save-plan=dev.plan.json
pulumi up --plan=dev.plan.json
```

### 배포 후 수정 시
```bash
pulumi stack select dev
pulumi preview --diff --refresh
pulumi up
```

### state가 어긋난 것 같을 때
```bash
pulumi refresh
```

### state 백업이 필요할 때
```bash
pulumi stack export --file backup.json
```

### 여러 stack 운영 시
```bash
pulumi stack select dev-01
pulumi preview
pulumi up

pulumi stack select dev-02
pulumi preview
pulumi up
```

---

## 18. 결론

이번 내용을 정리하면 핵심은 다음과 같다.

### 1) Pulumi의 핵심은 state다
Pulumi는 현재 stack state와 코드의 desired state를 비교해 동작한다.  
따라서 배포와 수정은 항상 state 관점에서 이해해야 한다.

### 2) 같은 stack에서 수정하는 것과 다른 stack에서 작업하는 것은 완전히 다르다
- 같은 stack: 기존 리소스를 수정/삭제/교체
- 다른 stack: 별도의 독립 배포
- ownership 이동: 필요 시 `state move`나 `import` 고려

### 3) multi-stack 운영은 자연스러운 방식이다
`dev-01`, `dev-02`, `staging`, `prod`처럼 운영하는 것은 일반적이다.  
보통은 같은 TypeScript 코드에 stack별 설정 파일만 달리 두면 된다.

### 4) 단, 구조가 너무 달라지면 project 분리를 고려해야 한다
stack은 환경/배포 인스턴스 구분에 좋지만,  
책임 범위와 리소스 구조까지 달라지면 project 분리가 더 낫다.

### 5) 지금 내 상황에서는
현재처럼 CloudWatch Alarm + SNS + Slack 연동 구조를 다루는 단계에서는  
우선 **단일 Pulumi 프로젝트 + multi-stack 운영**이 가장 합리적이다.

---

## 참고 메모

실무에서는 아래 질문을 항상 먼저 던지는 것이 좋다.

- 지금 수정하려는 리소스는 어느 stack의 소유인가?
- 이 작업은 기존 리소스의 수정인가, 새 환경 배포인가?
- preview 결과가 정말 내가 의도한 diff인가?
- 같은 리소스를 여러 stack이 중복 관리하고 있지 않은가?
- stack만 늘리면 되는가, 아니면 project를 분리해야 하는가?

이 질문들이 정리되면 Pulumi의 state 관리와 배포 흐름도 훨씬 명확해진다.