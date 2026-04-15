---
title: "[인턴] ECS Fargate Private Subnet 전환과 Deployment Circuit Breaker 적용기"
excerpt: "고정 IP가 필요한 외부 연동 환경에서 Fargate를 Private Subnet + NAT Gateway로 전환하고, 배포 실패 시 자동 롤백을 위한 Circuit Breaker를 세팅한 과정 정리"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, ECS, Fargate, NAT, VPC, CircuitBreaker, 배포]

permalink: /intern/company/blitz-dynamics/25/

toc: true
toc_sticky: true

date: 2026-04-15
last_modified_at: 2026-04-15
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/24/)에서는 ECS Fargate로의 회귀와 Source IP 기반 카나리 이전 전략을 정리했습니다.

이번 글에서는 Fargate 인프라를 실제로 세팅하면서 마주친 두 가지 문제와 해결 과정을 기록합니다.

1. **외부 서비스 IP Whitelist 환경에서 Fargate Task의 Public IP 변동 문제**
2. **배포 시 OOM / Container Failure로 서비스가 뜨지 않는 문제**

---

## 문제 1: Fargate Public IP 변동과 외부 서비스 IP Whitelist

### 상황

연동 중인 외부 서비스에서 **IP Whitelist 방식으로 접근을 제어**하고 있었습니다. 즉, 사전에 등록된 IP에서 오는 요청만 허용하는 구조입니다.

ECS Fargate를 Public Subnet에 배치하고 `assignPublicIp: ENABLED`로 설정하면, 각 Task에 Public IP가 할당됩니다. 문제는 이 IP가 **Task가 새로 뜰 때마다 바뀐다**는 점입니다.

### 왜 문제인가

Fargate Task는 다양한 상황에서 교체됩니다.

- 새 버전 배포 (Rolling Update)
- Auto Scaling에 의한 Task 추가/제거
- 헬스체크 실패로 인한 Task 재시작
- 인프라 유지보수로 인한 Task 재배치

Task가 교체될 때마다 새로운 Public IP가 할당되므로, 외부 서비스의 Whitelist에 등록된 IP와 달라지게 됩니다. 결과적으로 **배포할 때마다, 혹은 스케일링이 발생할 때마다 외부 서비스와의 통신이 끊기는** 상황이 발생합니다.

매번 새 IP를 외부 서비스에 등록 요청하는 것은 현실적이지 않습니다. 외부 서비스 측의 Whitelist 반영에도 시간이 걸리고, 그 사이에 서비스 장애가 발생합니다.

### 해결: Private Subnet + NAT Gateway + Elastic IP

ECS Cluster를 **Private Subnet으로 이동**하고, 인터넷으로 나가는 트래픽은 **NAT Gateway**를 경유하도록 변경했습니다. NAT Gateway에는 **Elastic IP(EIP)**를 할당하여 고정된 Outbound IP를 확보했습니다.

#### 변경 전 (Public Subnet)

```
[Fargate Task A] ──(Public IP: 변동)──→ 인터넷 → 외부 서비스
[Fargate Task B] ──(Public IP: 변동)──→ 인터넷 → 외부 서비스
```

Task마다 다른 Public IP로 외부에 요청을 보내므로, IP가 예측 불가능합니다.

#### 변경 후 (Private Subnet + NAT Gateway)

```
[Fargate Task A] ──→ NAT Gateway (EIP: 고정) ──→ 인터넷 → 외부 서비스
[Fargate Task B] ──→ NAT Gateway (EIP: 고정) ──→ 인터넷 → 외부 서비스
```

모든 Task의 Outbound 트래픽이 NAT Gateway를 경유하므로, 외부에서 보이는 IP는 항상 동일합니다.

#### 네트워크 구성

```
VPC
├── Public Subnet
│   ├── ALB (인바운드 트래픽 수신)
│   └── NAT Gateway + EIP (아웃바운드 트래픽 경유)
│
└── Private Subnet
    └── ECS Fargate Tasks (assignPublicIp: DISABLED)
        ├── Task A
        ├── Task B
        └── ...
```

- **Inbound**: 클라이언트 → ALB (Public Subnet) → Fargate Task (Private Subnet)
- **Outbound**: Fargate Task → NAT Gateway (Public Subnet) → 인터넷 → 외부 서비스

#### 핵심 설정

```json
{
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": ["subnet-private-a", "subnet-private-b"],
      "securityGroups": ["sg-ecs-tasks"],
      "assignPublicIp": "DISABLED"
    }
  }
}
```

Private Subnet의 라우팅 테이블에서 `0.0.0.0/0` 트래픽을 NAT Gateway로 향하도록 설정하면, Task에서 나가는 모든 인터넷 트래픽이 NAT Gateway의 EIP를 통해 나갑니다.

#### 외부 서비스에 등록할 IP

NAT Gateway에 할당된 EIP 하나만 외부 서비스의 Whitelist에 등록하면 됩니다. Task가 몇 개가 뜨든, 어떤 이유로 교체되든, Outbound IP는 항상 동일합니다.

#### 비용 고려

NAT Gateway는 시간당 과금 + 데이터 처리량 과금이 있어 무시할 수 없는 비용 항목입니다. 하지만 외부 서비스와의 통신이 끊기는 장애 비용과 비교하면, 충분히 합리적인 선택이었습니다.

---

## 문제 2: 배포 시 OOM / Container Failure로 서비스 미기동

### 상황

새 버전을 배포할 때 간헐적으로 **서비스가 아예 뜨지 않는** 문제가 발생했습니다. 원인은 크게 두 가지였습니다.

1. **OOM (Out of Memory)**: Task Definition에 설정된 메모리 한도를 초과하여 컨테이너가 kill됨
2. **Container Failure**: 애플리케이션 초기화 중 에러로 컨테이너가 비정상 종료됨

### ECS의 기본 배포 동작과 문제점

ECS의 Rolling Update 방식은 기본적으로 다음과 같이 동작합니다.

1. 새 Task를 시작
2. 새 Task가 `RUNNING` 상태가 되고 헬스체크를 통과하면, 기존 Task를 종료
3. 모든 Task가 교체될 때까지 반복

문제는 **새 Task가 계속 실패하면 어떻게 되는가**입니다. ECS는 기본적으로 실패한 Task를 **무한히 재시도**합니다. 새 버전의 컨테이너가 OOM으로 죽으면, ECS는 다시 새 Task를 띄우고, 또 OOM으로 죽고, 다시 띄우고... 이 과정이 반복됩니다.

이 동안 기존 Task는 유지되므로 **서비스 자체가 다운되지는 않지만**, 배포가 영원히 완료되지 않는 상태에 빠집니다. 배포가 진행 중인 상태로 멈춰 있으면:

- 다음 배포를 실행할 수 없습니다
- ECS가 계속 새 Task를 시도하면서 불필요한 리소스를 소비합니다
- 수동으로 배포를 취소하고 롤백해야 합니다
- 새벽이나 주말에 발생하면 대응이 늦어집니다

### 해결: Deployment Circuit Breaker

ECS의 **Deployment Circuit Breaker**를 활성화하여, 배포 실패 시 자동으로 롤백되도록 설정했습니다.

#### Circuit Breaker란

전기 회로의 차단기(Circuit Breaker)에서 따온 개념입니다. 과전류가 흐르면 차단기가 자동으로 회로를 끊듯이, **배포 실패가 반복되면 자동으로 배포를 중단하고 이전 버전으로 롤백**합니다.

#### 설정

```json
{
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```

- `enable: true`: Circuit Breaker를 활성화합니다
- `rollback: true`: Circuit Breaker가 발동되면 자동으로 이전 안정 버전으로 롤백합니다

#### 동작 방식

Circuit Breaker가 활성화되면 ECS는 배포 중 실패 임계값을 추적합니다.

1. 새 Task를 시작
2. Task가 `RUNNING` 상태에 도달하지 못하거나 헬스체크에 실패
3. 재시도 → 또 실패
4. **실패 횟수가 임계값을 초과하면 Circuit Breaker 발동**
5. 배포를 중단하고, 마지막으로 정상 동작했던 Task Definition 버전으로 자동 롤백

임계값은 ECS가 서비스의 `desiredCount`를 기반으로 자동 계산합니다. 별도 설정 없이도 합리적인 수준에서 발동됩니다.

#### 적용 전 vs 적용 후

**적용 전:**
```
새 버전 배포 → Task 실패 (OOM) → 재시도 → 또 실패 → 무한 반복
→ 배포 stuck 상태 → 수동 개입 필요 → 새벽이면 장애 장기화
```

**적용 후:**
```
새 버전 배포 → Task 실패 (OOM) → 재시도 → 또 실패 → Circuit Breaker 발동
→ 자동 롤백 → 이전 안정 버전 유지 → 알림 확인 후 원인 분석
```

핵심은 **사람이 개입하지 않아도 서비스가 안정 상태로 돌아간다**는 것입니다.

#### 알림 연동

Circuit Breaker가 발동되면 ECS가 EventBridge로 이벤트를 발생시킵니다. 이를 활용하여 배포 실패 시 알림을 받도록 구성했습니다.

```
ECS Deployment State Change Event
→ EventBridge Rule
→ SNS / Slack 알림
```

자동 롤백이 되더라도 **왜 실패했는지**는 확인해야 하므로, 알림은 필수입니다.

---

## 두 문제의 연결점

두 문제는 독립적으로 보이지만, 근본적으로 **Fargate 운영 안정성**이라는 같은 축에 있습니다.

| 문제 | 원인 | 해결 | 목적 |
|------|------|------|------|
| IP 변동 | Fargate Task 교체 시 Public IP 변경 | Private Subnet + NAT + EIP | 외부 연동 안정성 |
| 배포 실패 | OOM / Container Failure로 Task 미기동 | Deployment Circuit Breaker | 배포 안정성 |

EC2 단일 인스턴스 시절에는 IP가 고정되어 있고, 배포 실패하면 그냥 SSH 접속해서 수동으로 복구하면 됐습니다. 하지만 ECS Fargate로 넘어오면서 **Task가 동적으로 생성·소멸되는 환경**에 맞는 대응 방식이 필요해졌고, 이 두 가지가 그 첫 번째 세팅이었습니다.

---

## 마치며

인프라를 세팅하면서 느낀 건, **"동작하는 것"과 "안정적으로 운영되는 것"은 완전히 다른 차원의 문제**라는 점입니다.

Fargate Task를 Public Subnet에 띄우면 외부 통신이 "동작"합니다. 하지만 Task가 교체될 때마다 IP가 바뀌면 "운영"이 안 됩니다. 새 버전을 배포하면 코드는 "동작"할 수 있습니다. 하지만 메모리가 부족해서 Task가 뜨지 않으면 "운영"이 안 됩니다.

이번 세팅은 Fargate를 프로덕션에 올리기 전에 반드시 갖춰야 할 **운영 안전장치**였습니다. 다음 글에서는 실제 Stage 1 검증 과정과 그 결과를 정리할 예정입니다.
