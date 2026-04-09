---
title: "[인턴] Production 백엔드 배포를 위한 ECS 마이그레이션 및 카나리 배포 전략"
excerpt: "EC2 + Docker Compose 구조에서 ECS Fargate로 전환하며 설계한 Blue/Green·카나리 배포 전략과 DB 스키마 변경 충돌 해법"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, ECS, Fargate, 배포전략, 카나리, BlueGreen, DB마이그레이션, CodeDeploy, ALB]

permalink: /intern/company/blitz-dynamics/21/

toc: true
toc_sticky: true

date: 2026-04-09
last_modified_at: 2026-04-09
---

## 들어가며

preview 환경이 아닌 실제 production 환경에 백엔드를 배포하기 위한 전략을 수립해야 했다.
지금까지 내부 검증은 preview 환경에서만 이뤄졌는데, production 배포를 위해서는 무중단 배포 구조와 안전한 롤아웃 전략이 먼저 설계되어야 했다.

이 과정에서 현재 구조(EC2 + Docker Compose)의 한계를 정리하고, ECS Fargate 전환 시 무엇이 어떻게 달라지는지, 그리고 카나리 배포와 DB 스키마 변경이 어떻게 충돌하는지를 조사했다.

---

## 1. 현재 구조의 문제점

현재 production 배포 구조는 다음과 같다.

```
EC2 단일 인스턴스
  └── Docker Compose
        ├── service-blue  (내부 포트 N개)
        ├── service-green (내부 포트 N개)
        └── service-nginx (blue/green으로 직접 프록시)
```

`deploy.sh` 스크립트가 git pull → docker build → health check loop → nginx reload 순서로 동작한다.

이 구조에는 몇 가지 구조적 한계가 있다.

**스케일 아웃 불가**: EC2 1대에 고정이다. 트래픽이 급증해도 수평 확장 수단이 없다.

**빌드가 배포 시간에 포함**: EC2 내부에서 직접 이미지를 빌드한다. 빌드 시간(약 14분)이 고스란히 배포 다운타임 리스크로 이어진다.

**nginx가 단일 인스턴스 내부에서 포트 단위 분산**: 컨테이너 1개 안의 N개 포트로 직접 분산하는 구조다. 컨테이너가 죽으면 분산 대상 자체가 사라진다.

**수동 health check**: deploy.sh의 루프로 컨테이너 생존 여부만 확인한다. 실제 트래픽 품질을 보장하지 않는다.

---

## 2. ECS로 전환했을 때 달라지는 것

### 포트 기반 분산 → Task 기반 분산

가장 큰 구조 변화는 분산 단위가 바뀐다는 점이다.

**변경 전 (EC2 + Docker Compose)**

```
nginx :{NGINX_PORT}
  └── service-blue 컨테이너 1개
        ├── Node.js process :{PORT_1}
        ├── Node.js process :{PORT_2}
        ├── Node.js process :{PORT_3}
        └── Node.js process :{PORT_4}
        (3 vCPU / 8 GB 전체 공유)
```

**변경 후 (ECS Fargate)**

```
ALB
  └── Target Group
        ├── Task #1: Node.js :{APP_PORT} (1 vCPU / 2 GB)
        ├── Task #2: Node.js :{APP_PORT} (1 vCPU / 2 GB)
        ├── Task #3: Node.js :{APP_PORT} (1 vCPU / 2 GB)
        └── Task #4: Node.js :{APP_PORT} (1 vCPU / 2 GB)
```

nginx가 컨테이너 내부에서 포트 단위로 분산하던 역할을 ALB가 외부에서 Task 단위로 대체한다. 앱 코드는 포트 하나만 listen하면 된다(`process.env.PORT`). 총 리소스는 동일하지만(3 vCPU / 8 GB ≈ 1 vCPU × 4 Tasks / 2 GB × 4 Tasks), Task 단위로 독립적인 스케일 아웃/인이 가능해진다.

### 컨테이너 이전 매핑표

| 현재 (docker-compose) | ECS 전환 |
|---|---|
| service-blue / service-green | Backend ECS Service (Blue/Green Task Set) |
| service-nginx | ALB로 완전 대체 |
| service-cron | 별도 Cron ECS Service |
| datadog | Task 내 dd-agent sidecar |
| service-vector | Task 내 firelens sidecar (fluent-bit) |

### 추가로 얻는 것

- CodeDeploy Blue/Green 또는 카나리로 무중단 배포 자동화
- Auto Scaling으로 트래픽에 따른 탄력적 운영 (평상시 4개, 급증 시 8개 등)
- GitHub Actions → ECR 빌드 분리로 배포 시간이 배포 다운타임에 포함되지 않음

---

## 3. Blue/Green 배포

### 동작 방식

1. Green Task Set 4개 기동 (Blue 100% 유지 중)
2. ALB health check 통과 대기
3. ALB 가중치 Blue 100% → Green 100% 즉시 전환
4. Blue Task Set draining 후 종료

### Blue/Green의 한계

`/health` API는 "서버가 살아있는가"만 확인한다. 실제 서비스 안정성은 담보하지 않는다. 즉시 100% 전환이므로 문제가 있어도 전체 트래픽이 영향받는다. 롤백은 빠르지만, 전환 전 검증 수단이 부족하다.

이 한계를 보완하기 위해 카나리 배포를 검토했다.

---

## 4. 카나리 배포

### Blue/Green과의 핵심 차이

Task 수는 동일하다. Blue 4개 + Green 4개 = 총 8개를 동시에 운영한다. 트래픽 제어는 Task 수가 아니라 **ALB 가중치**로만 조정한다.

```
5% → 25% → 50% → 100%
```

### CodeDeploy 카나리 preset

| Config | 동작 |
|---|---|
| ECSCanary10Percent5Minutes | 10% → 5분 대기 → 100% |
| ECSLinear10PercentEvery1Minutes | 매 1분마다 10%씩 증가 |
| ECSLinear10PercentEvery3Minutes | 매 3분마다 10%씩 증가 |
| Custom | 직접 percentage / interval 설정 |

### 가중치 업데이트 판단 기준

**절대값이 아니라 Green vs Blue 비율로 비교해야 한다.**

```
# 틀린 방식
Green 5xx 에러율 > 1% → 롤백

# 올바른 방식
Green 5xx 에러율 / Blue 5xx 에러율 > 1.3 → 롤백
```

외부 요인(야간, DB 부하)으로 서비스 전체가 느려지면 절대값 기준은 오탐이 발생한다. Blue를 기준선(baseline)으로 삼아 상대적 열화를 탐지해야 한다.

### 지표 3계층

**1. 비즈니스 지표** (신호 느림, 50% 이상부터 신뢰도 상승)

전환율, 주문 성공률, 결제 오류율, 핵심 API 성공률

**2. 애플리케이션 지표** (핵심 신호)

5xx 에러율, p99 지연, RPS, 큐 적체. Green / Blue 비율: delta 0.5% 이내, p99 ratio 1.3× 이내

**3. 인프라 지표** (참고용, 단독 판단 비권장)

CPU, 메모리, 열린 커넥션, 네트워크 에러. False positive가 많아 이상 증폭 시 보조 참고용으로만 사용한다.

### 관찰 구간 설계

| 가중치 | 관찰 시간 | 이유 |
|---|---|---|
| 5% | 5분 | 빠른 크래시, 명백한 오류 탐지 |
| 25% | 10분 | 엣지 케이스, 부하 패턴 확인 |
| 50% | 15~20분 | 가장 긴 관찰. 메모리 리크 초기 징후 감지 |
| 100% | — | 완료 후 Blue draining |

비즈니스 지표는 5% 구간에서 통계적 유의성이 부족하다. 이 구간에서는 애플리케이션 지표만으로 판단하고, 비즈니스 지표는 25% 이상부터 보조 신호로 참고하는 것이 현실적이다.

### Datadog + CodeDeploy 자동화 예시

```javascript
// AfterAllowTraffic Lambda Hook
exports.handler = async (event) => {
  const ratio = await queryDatadog(`
    avg:trace.express.request.errors{version:green} /
    avg:trace.express.request.errors{version:blue}
  `);

  const status = ratio > 1.3 ? 'Failed' : 'Succeeded';

  await codedeploy.putLifecycleEventHookExecutionStatus({
    deploymentId: event.DeploymentId,
    lifecycleEventHookExecutionId: event.LifecycleEventHookExecutionId,
    status
  });
};
```

Lambda Hook이 Datadog에서 Green/Blue 비율을 조회하고, 임계값을 초과하면 배포를 자동으로 롤백한다.

---

## 5. DB 스키마 변경과 카나리의 충돌

### 문제 상황

카나리 배포는 v1(Blue)과 v2(Green)이 같은 DB를 공유한다. Breaking migration(컬럼 삭제, 컬럼 이름 변경 등)이 실행되는 순간, v1은 존재하지 않는 컬럼을 참조하게 되어 계속 fail이 발생한다. 카나리의 이점이 사라진다.

### 해법: Expand-Contract 패턴

마이그레이션 자체를 3단계로 쪼갠다.

#### Phase 1: Expand 마이그레이션 (v1 배포 중 실행)

```sql
-- additive 변경만 실행 (v1에 영향 없음)
ALTER TABLE some_table ADD COLUMN col_new VARCHAR(255);
UPDATE some_table SET col_new = col_old;
```

새 컬럼을 추가하되, 기존 컬럼은 그대로 유지한다. v1은 아무 영향 없이 계속 정상 운영된다.

#### Phase 2: v2 카나리 배포 (스키마 변경 없음)

DB 스키마는 Phase 1 상태 그대로다.

- v1: `SELECT col_old`, `UPDATE col_old = ?` → 정상
- v2: `SELECT col_new`, `UPDATE col_old=?, col_new=?` (dual-write) → 정상

두 버전이 공존해도 충돌이 없으므로 카나리가 정상적으로 진행된다.

#### Phase 3: Contract 마이그레이션 (v2 100% + Blue 완전 종료 확인 후)

```sql
-- v1이 완전히 종료된 것을 확인한 뒤에만 실행
ALTER TABLE some_table DROP COLUMN col_old;
```

### Expand-Contract의 비용

- Dual-write 구간 동안 v2 코드 복잡도가 증가한다
- 배포 사이클이 최소 3번 필요하다 (Expand → 카나리 → Contract)
- 빠르게 움직여야 하는 상황에서 부담이 될 수 있다

### 마이그레이션 유형별 전략

| 유형 | 예시 | 전략 |
|---|---|---|
| Additive | nullable 컬럼 추가, 테이블 추가, 인덱스 | 그냥 배포 |
| 구조 변경 | 컬럼 이름/타입 변경, 컬럼 병합 | Expand-Contract |
| 즉시 파괴적 | v1이 물리적으로 실행 불가능한 변경 | 카나리 포기 → 즉시 B/G |

### 즉시 파괴적 마이그레이션 시 위험 최소화

Expand-Contract도 불가능한 경우에는 카나리를 포기하고 아래 절차로 위험을 줄인다.

1. 스테이징에서 마이그레이션 + 전 버전 배포 완전 검증
2. 트래픽이 가장 낮은 시간대 선택
3. Blue/Green 즉시 전환 (cutover window 최소화)
4. DB 백업 + 마이그레이션 롤백 스크립트 사전 준비
5. on-call 대기

---

## 6. 정리

> **마이그레이션의 성격이 배포 전략을 결정한다.**
> 코드 변경보다 스키마 변경이 더 먼저 설계되어야 한다.

이번 조사의 핵심 결론은 세 가지다.

**Breaking migration이 없다면** 카나리는 강력한 안전망이다. ALB 가중치 단계적 증가 + Datadog Lambda Hook 자동 판단으로 문제를 조기에 탐지하고 롤백할 수 있다.

**Breaking migration이 있다면** Expand-Contract로 마이그레이션 자체를 무해하게 만들어야 한다. 스키마 변경을 additive 단계로 분리하면 카나리와의 충돌이 사라진다.

**그것도 불가능하다면** 카나리를 포기하고 Blue/Green + 충분한 사전 검증으로 위험을 줄인다. 배포 전략보다 스키마 변경 전략이 먼저 설계되어야 하는 이유다.
