---
title: "[인턴] ECS Fargate 태스크 사망 원인 추적기: Uncaught ETIMEDOUT과 Essential Container"
excerpt: "프로덕션 Fargate 태스크 하나가 ETIMEDOUT으로 자살한 사건을 ECS API → CloudWatch → 소스코드 순으로 역추적하며 정리한 포스트모템 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, ECS, Fargate, CloudWatch, NodeJS, 포스트모템, 장애대응]

permalink: /intern/company/blitz-dynamics/26/

toc: true
toc_sticky: true

date: 2026-04-16
last_modified_at: 2026-04-16
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/25/)에서 Fargate를 Private Subnet으로 옮기고 Deployment Circuit Breaker를 붙이며 운영 안전장치를 한 단계 올린 이야기를 정리했습니다.

Circuit Breaker는 **배포 실패**에 대한 안전장치입니다. 그런데 오늘은 **런타임에 정상적으로 돌아가던 태스크가 혼자 죽는** 사건이 발생했습니다.

오후 5시 30분 즈음, Datadog 알림으로 프로덕션 백엔드 태스크 하나가 `STOPPED` 상태로 빠졌다는 사실을 확인했습니다. 다행히 ECS 서비스가 30초 이내에 새 태스크를 띄워주어 실제 사용자 영향은 거의 없었지만, **왜 죽었는지**는 반드시 짚고 넘어가야 할 문제였습니다.

이번 글은 그 사망 원인을 역추적한 과정을 포스트모템 형태로 정리한 기록입니다. AWS API로 메타데이터부터 긁어내고, CloudWatch 로그를 파고, 결국 우리 소스코드 안쪽까지 들어가서 범인을 찾아내는 과정이 꽤 교과서적이었기에 기록으로 남깁니다.

> ⚠️ **민감 정보 마스킹**: 클러스터명, 태스크 ID, 일부 엔드포인트 경로는 마스킹했습니다.

---

## 1단계: 태스크 메타데이터 조회 (AWS ECS API)

가장 먼저 확인해야 할 것은 **"ECS가 이 태스크를 왜 STOPPED로 기록했는가"** 입니다. ECS는 각 태스크에 대해 `stopCode`, `stoppedReason`, 컨테이너별 `exitCode` 같은 메타데이터를 남깁니다.

```bash
aws ecs describe-tasks \
  --cluster ***-prod \
  --tasks be03d6da... \
  --region ap-northeast-2 \
  --query 'tasks[0].[stoppedReason,stopCode,stoppedAt,startedAt,
           containers[*].[name,exitCode,reason,lastStatus]]' \
  --output json
```

### 확인한 핵심 필드

| 필드 | 값 | 의미 |
|------|------|------|
| `stopCode` | `EssentialContainerExited` | essential 컨테이너 중 하나가 죽어 태스크 전체 종료 |
| `stoppedReason` | `"Essential container in task exited"` | ECS가 cascading shutdown 트리거 |
| `containers[backend].exitCode` | `1` | backend 컨테이너가 비정상 종료 (**범인**) |
| `containers[datadog-agent].exitCode` | `0` | datadog는 SIGTERM 받고 정상 종료 (**피해자**) |
| `startedAt → stoppedAt` | 09:47:43 → 17:21:22 | 약 7시간 33분 가동 후 사망 |

### 알아낸 것

- **backend 컨테이너가 먼저 exit 1로 죽었고**, ECS의 `essential: true` 규칙에 의해 같은 태스크에 묶여 있던 datadog-agent까지 연쇄적으로 내려갔습니다.
- 즉, **datadog이 backend를 죽인 게 아니라 backend가 자살하면서 datadog을 함께 데려간 것**입니다.

이 단계에서 "backend가 죽은 이유"로 수사 범위가 좁혀졌습니다.

---

## 2단계: CloudWatch Logs 스트림 위치 파악

backend가 exit 1로 죽었다는 사실은 확인했지만, **왜** 죽었는지는 애플리케이션 로그를 봐야 알 수 있습니다.

### Task Definition에서 로그 설정 확인

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/***-backend-prod",
      "awslogs-region": "ap-northeast-2",
      "awslogs-stream-prefix": "backend"
    }
  }
}
```

### 스트림 이름 규칙

`awslogs-stream-prefix`와 컨테이너명, 태스크 ID를 조합해 **스트림 이름을 직접 구성**할 수 있습니다.

```
{prefix}/{container-name}/{task-id}
→ backend/backend/be03d6da...
```

이 규칙을 모르면 CloudWatch 콘솔에서 UI로 스트림을 뒤져야 하지만, 규칙만 알면 CLI 한 번으로 바로 접근할 수 있습니다.

---

## 3단계: 사망 시점 직전 로그 추출

사망 시점 직전의 로그를 봐야 하므로, **로그 스트림의 가장 최근 200줄**만 가져오도록 CLI 옵션을 맞췄습니다.

```bash
aws logs get-log-events \
  --log-group-name /ecs/***-backend-prod \
  --log-stream-name backend/backend/be03d6da... \
  --no-start-from-head \
  --limit 200 \
  --region ap-northeast-2
```

### 시행착오

처음엔 `--start-from-head false` 형태로 호출했다가 `Unknown options` 에러가 떴습니다. AWS CLI에서 boolean은 **플래그 그 자체가 값**이라서, `--no-start-from-head` 형태로 써야 정상 동작합니다.

> 📝 **팁**: AWS CLI에서 boolean 옵션은 `--option true/false`가 아니라 `--option` / `--no-option` 형태를 사용합니다.

### 발견한 결정적 로그

사망 시각(17:19:37 KST) 바로 직전 라인에서 원인이 그대로 찍혀 있었습니다.

```
Error: read ETIMEDOUT
    at TCP.onStreamRead (node:internal/stream_base_commons:218:20)
  errno: -110,
  code: 'ETIMEDOUT',
  syscall: 'read',
  source: 'socket'

[Nest] 1  - 2026.04.16. 17:19:37.017 ERROR [Process] Uncaught exception: read ETIMEDOUT
```

이 로그가 알려주는 것은 명확합니다.

- Node.js 런타임이 어떤 **TCP 소켓에서 read를 시도하다가 ETIMEDOUT**을 받았습니다.
- 이 에러가 try/catch에 잡히지 않아 **`uncaughtException` 핸들러까지 올라갔습니다.**
- 그 핸들러가 로그를 남긴 직후, 프로세스는 `exit 1`로 종료되었습니다.

범인은 확실하지만, **어떤 호출이 ETIMEDOUT을 던졌는지(=call site)**는 이 스택만으로는 알 수 없습니다. `TCP.onStreamRead`는 Node 내부 함수라서 애플리케이션 코드의 어느 줄에서 발생했는지 드러나지 않습니다.

---

## 4단계: 코드 레벨 원인 추적

이 시점부터는 AWS가 아니라 **우리 코드**를 파고들어야 했습니다.

### 4-1. uncaughtException 핸들러 검사

NestJS 부트스트랩 파일을 열어 봤습니다.

```typescript
// main.ts (L190-198)
process.on('uncaughtException', (err: any) => {
  if (err.code === 'EPIPE' || err.code === 'ECONNRESET') {
    Logger.log(`[STDOUT] Ignored uncaught socket error: ${err.code}`);
  } else {
    Logger.error(`Uncaught exception: ${err.message}`, '', 'Process');
    process.exit(1); // ← 여기서 자기 자신을 죽임
  }
});
```

이 핸들러는 이렇게 동작합니다.

- `EPIPE` / `ECONNRESET`은 **화이트리스트**에 있어 무시됩니다. (클라이언트가 먼저 연결을 끊었을 때 나는 흔한 에러)
- 그 외 모든 uncaught exception은 **의도적으로 `process.exit(1)`**.

즉, **ETIMEDOUT은 화이트리스트에 없으므로 코드가 설계대로 자살한 것**입니다. 버그가 아니라 의도된 동작이었습니다.

### 4-2. 사망 직전 80초 트래픽 분석

"그럼 어떤 코드가 ETIMEDOUT을 던졌나" 를 좁히기 위해 CloudWatch 로그에서 **사망 직전 80초간의 요청 패턴**을 봤습니다.

결과는 꽤 명확했습니다.

```
POST /xxx/site/*  ≈ 50회 (초당 0.6회)
```

특정 디바이스 폴링용 엔드포인트 한 개가 로그의 거의 전부를 차지하고 있었습니다.

### 4-3. 해당 모듈 코드 점검

해당 엔드포인트의 컨트롤러/서비스 코드를 열어 확인해 본 결과:

- 이 엔드포인트는 Redis 큐만 건드리고, **직접 TCP 소켓이나 외부 HTTP를 때리지 않습니다.**
- 즉, 이 폴링 엔드포인트 자체가 ETIMEDOUT을 던졌다고 보기는 어렵습니다.

**그렇다면 진범은 이 폴링과 **같은 시간대에 병렬로 돌고 있던 다른 비동기 작업**일 가능성이 큽니다.** 후보는 다음과 같습니다.

- **Redis 연결 끊김**: `queueCount` / `queuePop` 중 소켓 read timeout
- **Postgres TCP keepalive**: DB 연결 풀의 유휴 소켓이 NAT/방화벽에 의해 끊긴 후 read 시도
- **외부 HTTP 호출**: Slack / Naver / OpenAI 등의 아웃바운드 — iptables로 특정 IP가 차단되어 있으면 ECONNREFUSED가 아니라 **ETIMEDOUT** 형태로 늦게 드러날 수 있음

이 중 어느 쪽이 진짜 범인이었는지는 **지금 수집된 스택만으로는 식별 불가**합니다. 현재 핸들러가 `err.message`만 찍고 `err.stack`을 남기지 않기 때문입니다.

---

## 5단계: 최종 인과사슬 정리

지금까지의 단서를 시간순으로 이으면 다음과 같습니다.

```
[17:19:37] 외부 소켓 read ETIMEDOUT (call site 미상, stack trace 미수집)
    ↓
try/catch에 잡히지 않음
    ↓
process.on('uncaughtException') 트리거
    ↓
err.code !== 'EPIPE' && err.code !== 'ECONNRESET'
    ↓
process.exit(1)  ← backend 자살
    ↓
ECS: stopCode = EssentialContainerExited
    ↓
[17:21:22] datadog-agent에 SIGTERM → 정상 종료 (exit 0)
    ↓
태스크 전체 STOPPED
    ↓
ECS 서비스 스케줄러가 desiredCount 유지를 위해 새 태스크 기동
```

### 결론

| 질문 | 답 |
|------|------|
| 범인 | backend의 **미처리 ETIMEDOUT** |
| 방아쇠 | `main.ts:195`의 `process.exit(1)` (의도된 동작) |
| datadog-agent 사망은? | essential container cascading의 **부수효과** |
| call site 식별 | ❌ 불가 — 스택 트레이스를 안 남기고 있었음 |

즉, **ECS 입장에선 설계대로 잘 동작했고, Node 프로세스도 설계대로 자살했지만, "왜 ETIMEDOUT이 났는지"는 우리가 볼 수 있는 형태로 기록되지 않았다**는 것이 이번 사건의 핵심입니다.

---

## 후속 조치

이번 사건으로 드러난 운영상 공백을 기간별로 정리했습니다.

| 우선순위 | 조치 | 기대 효과 |
|----------|------|-----------|
| **단기** | `main.ts`의 `uncaughtException` 핸들러에 `err.stack` 로깅 추가 | 다음 발생 시 **call site 즉시 특정** |
| **중기** | CloudWatch Metric Filter로 `Uncaught exception` 패턴 카운트 + 알람 | 발생 빈도 추적, 조기 감지 |
| **장기** | 외부 호출 site에 **명시적 timeout + try/catch** 도입 | 자살 자체를 방지 |

### 단기 조치: 한 줄 수정이 주는 가치

```typescript
// Before
Logger.error(`Uncaught exception: ${err.message}`, '', 'Process');

// After
Logger.error(`Uncaught exception: ${err.message}`, err.stack, 'Process');
```

이 한 줄만 추가되어도 다음 번에 같은 사건이 터졌을 때, **어느 모듈의 어느 라인에서 ETIMEDOUT이 났는지** 즉시 알 수 있습니다. 장애 분석 비용이 하루에서 5분으로 줄어드는 차이입니다.

### 중기 조치: 조용히 자주 죽는 걸 감지

이번엔 30초만에 새 태스크가 뜨면서 체감 영향이 적었지만, **같은 패턴이 하루에 여러 번 반복되고 있을 가능성**을 배제할 수 없습니다. Metric Filter로 "Uncaught exception" 문자열 카운트를 계측하고, 특정 임계값 이상일 때 Slack으로 알림을 보내도록 붙일 예정입니다.

### 장기 조치: 자살 자체를 막기

근본적으로 `process.exit(1)`에 도달하지 않도록 **외부 호출의 경계를 모두 try/catch와 timeout으로 감싸는 것**이 정석입니다.

- HTTP Client: axios의 `timeout` 설정 + interceptor에서 에러 normalize
- DB: pg Pool의 `idleTimeoutMillis` / `connectionTimeoutMillis`
- Redis: ioredis의 `commandTimeout` / `keepAlive`

이건 한번에 다 손보기보다는, 로그를 보면서 **실제로 자주 터지는 call site부터 하나씩** 막아나갈 계획입니다.

---

## 이번 사건에서 배운 것

### 1. 관찰 가능성(Observability)은 "사고 전"에 갖춰야 한다

사고가 나야 "아, `err.stack`을 안 찍고 있었구나"를 깨닫습니다. 사고 후에 고쳐도 다음 사고 대응은 빨라지지만, **이번 사고 자체는 원인 특정 불가**라는 대가를 치르게 됩니다. 관찰 가능성 코드는 **지금 당장은 아무 쓸모 없어 보이지만, 장애 한 번에 가치가 드러나는 투자**입니다.

### 2. Essential Container 설정의 연쇄성

`essential: true`로 묶인 컨테이너 중 하나만 죽어도 태스크 전체가 내려갑니다. datadog-agent가 "정상 종료"된 것처럼 보이지만, 실제로는 **backend 사망의 연쇄 피해자**였습니다. 이 연쇄 규칙을 모르면 로그만 보고 엉뚱한 범인을 쫓게 됩니다.

### 3. "설계대로 동작한 자살"도 장애다

`process.exit(1)`은 분명 의도된 코드입니다. 하지만 그 트리거인 ETIMEDOUT이 **"정말 회복 불가능한 에러였는가"**는 전혀 다른 질문입니다. 단순한 네트워크 일시 장애를 **치명적 에러로 간주하고 전체 프로세스를 내리는 것**은 과잉 반응일 수 있습니다. 에러의 종류별로 **회복 전략**을 달리 설계하는 것이 장기 조치의 핵심 방향입니다.

---

## 마치며

이번 사건은 **"Fargate + Circuit Breaker로 배포는 안전해졌지만, 런타임 자살은 아직 막지 못했다"** 는 걸 드러낸 사례였습니다.

AWS 콘솔 → CLI → CloudWatch → 소스코드 순으로 한 겹씩 벗겨가며 원인을 좁히는 경험 자체는 값진 훈련이었고, 동시에 **"다음 사고에서는 이 단계를 더 짧게 줄일 수 있어야 한다"**는 숙제를 분명히 남겼습니다.

다음 글에서는 위에서 정리한 단기/중기 조치를 실제로 반영하고, 재발 여부를 관찰한 결과를 이어서 정리할 예정입니다.
