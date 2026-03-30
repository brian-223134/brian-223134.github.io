---
title: "[인턴] CloudWatch ALB 헬스체크의 한계와 ECR/ECS 마이그레이션"
excerpt: "EC2 + nginx 환경에서 ALB 헬스체크 진단이 부정확한 원인을 분석하고, 이를 해결하기 위해 ECR + ECS로 마이그레이션한 과정 기록"

categories:
    - Company
tags:
    - [블리츠다이나믹스, 산학, 인턴, 인프라, AWS, ALB, CloudWatch, ECS, ECR, Docker, 마이그레이션]

permalink: /intern/company/blitz-dynamics/15/

toc: true
toc_sticky: true

date: 2026-03-31
last_modified_at: 2026-03-31
---

# CloudWatch ALB 헬스체크가 왜 믿기 어려웠나 — 그리고 ECR/ECS로 넘어간 이유

## 들어가며

"알람은 울렸는데 서비스는 멀쩡해요."

처음 이 말을 들었을 때, 나는 사수가 무슨 말을 하는 건지 바로 이해하지 못했다.
CloudWatch에서 ALB Target이 **Unhealthy**로 찍혔는데, 정작 사용자는 정상적으로 요청이 처리되고 있었다.

반대 상황도 있었다. CloudWatch에서는 아무런 이상이 없었는데, 특정 노드가 사실상 죽어있어서 일부 요청이 타임아웃 나는 경우였다.

"이게 왜 이런 거예요?" 물었더니, 사수가 "nginx 때문이야"라고 했다.
그때부터 나는 우리 인프라 구조를 처음부터 다시 들여다봤다.

이번 글은 그 과정에서 내가 이해하게 된 것들, 그리고 팀이 이를 해결하기 위해 ECR + ECS로 마이그레이션을 결정한 이유를 정리한 기록이다.

---

## 1부: 기존 아키텍처 — EC2 + nginx + ALB

### 전체 구조

기존 아키텍처는 대략 이런 모양이었다:

```
[Client]
   │
[ALB (Application Load Balancer)]
   │
   ├── EC2 인스턴스 A (nginx → 포트 포워딩 → Node.js 앱)
   ├── EC2 인스턴스 B (nginx → 포트 포워딩 → Node.js 앱)
   └── EC2 인스턴스 C (nginx → 포트 포워딩 → Node.js 앱)
```

ALB가 들어오는 트래픽을 EC2 인스턴스들에 나눠준다.
각 EC2 안에는 **nginx**가 있고, nginx가 실제 앱(Node.js)의 포트로 트래픽을 포워딩한다.

처음엔 이 구조가 자연스러워 보였다. "아, nginx가 리버스 프록시 역할을 하는구나." 정도로만 이해했다.

### ALB 헬스체크가 보는 것

ALB는 Target Group의 각 인스턴스가 살아있는지 주기적으로 헬스체크를 한다.

```
ALB → EC2:80 (또는 443) → HTTP 200 OK?
      └── 이게 헬스체크다
```

여기서 중요한 것: **ALB가 체크하는 건 nginx가 응답하는지 여부**다.
즉, ALB 입장에서 "EC2가 살아있다"는 건 "nginx가 HTTP 200을 반환했다"는 뜻이다.

---

## 2부: 문제의 정체 — nginx와 앱 프로세스는 다른 존재다

### Healthy인데 실제로는 죽어있는 경우

이게 내가 처음 접한 상황이었다. CloudWatch에서 보면 3개 인스턴스 모두 **Healthy**인데, 실제로는 특정 인스턴스의 Node.js 앱이 죽어서 응답을 못 하고 있었다.

왜?

```
ALB → EC2:80 (nginx) → nginx는 살아있음 → HTTP 200 OK ✅
                     → 하지만 nginx가 포워딩할 Node.js 앱(예: :3000)은 죽어있음 💀
```

nginx 자체는 멀쩡히 응답한다. ALB는 "아, 이 인스턴스 정상이네"라고 판단한다.
하지만 nginx가 실제 앱 포트로 포워딩하면 Connection refused 또는 타임아웃이 난다.

결과: ALB는 트래픽을 이 인스턴스에 계속 보내고, 사용자는 간헐적으로 타임아웃을 경험한다.

### Unhealthy인데 실제로는 살아있는 경우

반대 상황도 있었다. CloudWatch에서 특정 인스턴스가 **Unhealthy**로 찍혔는데, 실제 서비스는 정상으로 돌아가고 있었다.

이건 조금 다른 원인이었다. nginx 설정 중 일부가 헬스체크 경로(/health)에 대해 다르게 반응하거나, nginx reload 중에 일시적으로 응답을 못 하는 경우가 있었다.

```
ALB → EC2:80/health → nginx reload 중 → 503 반환 → Unhealthy 판정 ❌
                    → 하지만 실제 앱(:3000)은 정상 동작 중 ✅
```

ALB가 해당 인스턴스를 Target Group에서 제거하면?
남은 인스턴스들에 트래픽이 몰린다. 그게 과부하로 이어지는 경우도 있었다.

### 근본 원인 정리

```
ALB 헬스체크 → nginx 상태만 본다
                     ≠
실제 앱 상태 → nginx 뒤에 있는 프로세스들
```

EC2 위에서 nginx가 중간에 끼어 있으니, **ALB가 관찰하는 것과 실제 앱의 상태가 분리**되는 구조다.
ALB는 nginx를 통해 간접적으로만 앱 상태를 볼 수 있고, 그 결과 진단 정보가 부정확해진다.

---

## 3부: 왜 이 문제가 CloudWatch 알람과 연결되나

### CloudWatch에서 ALB 메트릭 보기

CloudWatch에서는 ALB의 Target Group 상태를 메트릭으로 볼 수 있다:

```
AWS/ApplicationELB 네임스페이스

- HealthyHostCount : 헬시한 타겟 수
- UnHealthyHostCount : 언헬시한 타겟 수
- TargetResponseTime : 타겟 응답 시간
- HTTPCode_Target_5XX_Count : 5XX 에러 수
```

우리가 걸어둔 알람은 `UnHealthyHostCount >= 1` 이면 SNS를 통해 Slack에 알람을 보내는 구조였다.

처음엔 이게 충분해 보였다. "Unhealthy가 1개라도 생기면 알람이 오니까, 빠르게 대응할 수 있겠지"라고 생각했다.

### 실제로는 이랬다

- **Unhealthy 알람이 왔는데 서비스는 정상**: 알람에 반응해서 확인해보면 멀쩡함. 수십 번 이런 일이 반복되면 알람에 대한 신뢰도가 떨어진다. (알람 피로감)
- **알람이 안 왔는데 서비스 장애**: nginx는 살아있으니 Healthy로 잡혔지만 앱이 죽어있었던 경우. 이때는 사용자 신고를 받고서야 알게 되는 경우가 생겼다.

결국 **CloudWatch 알람이 인프라 상태를 정확하게 반영하지 못하는 상황**이 됐고, 팀 내에서 근본적인 해결책이 필요하다는 이야기가 나오기 시작했다.

---

## 4부: 해결책 — ECR + ECS로의 마이그레이션

### 왜 ECS인가?

팀에서 논의한 결과, **ECR + ECS** 기반으로 마이그레이션하기로 결정했다.
내가 처음 이 결정을 들었을 때, "도커로 옮기면 뭐가 달라지죠?"라고 물었다.

핵심은 이거다:

**ECS는 컨테이너 단위로 헬스체크를 한다.**

기존 구조에서는 ALB가 EC2(nginx)를 헬스체크했다. ECS 구조에서는 ALB가 ECS Task(컨테이너)를 직접 헬스체크한다.

```
[기존]
ALB → EC2(nginx) → 헬스체크
       └── 실제 앱은 별도 프로세스

[ECS 이후]
ALB → ECS Task(컨테이너) → 헬스체크
       └── 앱 자체가 컨테이너
```

nginx를 통한 포워딩 레이어가 사라지고, ALB가 앱 컨테이너를 직접 바라보게 된다.

### ECR이란?

ECR (Elastic Container Registry) 은 AWS가 제공하는 컨테이너 이미지 저장소다.

```
개발자 → docker build → docker push → ECR (이미지 저장)
                                         ↓
ECS → ECR에서 이미지 pull → 컨테이너 실행
```

Docker Hub처럼 이미지를 저장하고 관리하는데, AWS 내부 네트워크에서 동작하기 때문에 IAM 권한으로 접근 제어가 된다.

### ECS의 핵심 개념

ECS를 처음 접했을 때 용어들이 많아서 헷갈렸다. 정리하면:

| 개념 | 설명 | 비유 |
| --- | --- | --- |
| Cluster | ECS 리소스들의 논리적 그룹 | 서버실 |
| Task Definition | 컨테이너 실행 설정 (이미지, CPU, 메모리, 포트 등) | 설계도 |
| Task | Task Definition에 따라 실제 실행된 컨테이너 | 실제 서버 |
| Service | Task를 원하는 수만큼 유지해주는 관리자 | 자동 복구 시스템 |

서비스가 "항상 Task가 3개 실행되도록 유지해줘"라고 설정하면, Task 하나가 죽었을 때 자동으로 새 Task를 띄워준다.

### ECS에서 헬스체크가 어떻게 달라지나

ECS에는 두 종류의 헬스체크가 있다:

**1. Task 자체의 헬스체크 (컨테이너 레벨)**

Task Definition에 직접 헬스체크를 정의할 수 있다:

```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
```

이건 ECS 에이전트가 컨테이너 내부에서 직접 헬스체크를 실행한다.
nginx가 아니라, 앱이 `/health` 응답을 잘 하는지 직접 확인한다.

**2. ALB Target Group 헬스체크 (로드밸런서 레벨)**

ECS Service를 ALB Target Group에 연결하면, ALB가 각 ECS Task(컨테이너)를 직접 헬스체크한다.

```
ALB → ECS Task :3000/health → HTTP 200?
      └── 앱 컨테이너가 직접 응답
```

기존처럼 nginx를 거치지 않는다.
앱이 죽으면 → Task 헬스체크 실패 → ECS가 Task 교체 → ALB Target에서도 제거 → 새 Task가 올라오면 다시 등록.

이 흐름이 기존 nginx 레이어가 있던 구조보다 훨씬 정확하고 빠르게 동작한다.

---

## 5부: 마이그레이션 진행 과정

### Dockerfile 작성

기존 Node.js 앱을 컨테이너화하는 작업부터 시작했다.

```dockerfile
FROM node:20-alpine

WORKDIR /app

# 의존성 먼저 복사 (캐시 레이어 활용)
COPY package*.json ./
RUN npm ci --only=production

COPY . .

RUN npm run build

EXPOSE 3000

# 헬스체크 엔드포인트가 있는 앱
CMD ["node", "dist/main.js"]
```

### ECR에 이미지 푸시

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin \
  [account-id].dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드 & 태그
docker build -t my-app .
docker tag my-app:latest \
  [account-id].dkr.ecr.ap-northeast-2.amazonaws.com/my-app:latest

# 푸시
docker push [account-id].dkr.ecr.ap-northeast-2.amazonaws.com/my-app:latest
```

### 앱에 헬스체크 엔드포인트 추가

기존 앱에는 `/health` 엔드포인트가 없거나, 단순히 200만 반환하는 형태였다.
ALB와 ECS가 제대로 된 헬스체크를 할 수 있도록, **앱 내부 상태를 반영하는 헬스체크**로 개선했다.

```typescript
// health.controller.ts (NestJS 예시)

@Controller('health')
export class HealthController {
  @Get()
  check() {
    // DB 연결 상태, 메모리 등을 확인할 수 있음
    return { status: 'ok', timestamp: new Date().toISOString() };
  }
}
```

이제 ALB는 nginx가 아니라 이 엔드포인트를 직접 호출해서 상태를 확인한다.

### CloudWatch에서 ECS 메트릭 보기

ECS로 넘어오면 CloudWatch에서 볼 수 있는 메트릭도 더 세밀해진다:

```
AWS/ECS 네임스페이스

- CPUUtilization : 클러스터/서비스 CPU 사용률
- MemoryUtilization : 메모리 사용률
- RunningTaskCount : 실행 중인 Task 수
- PendingTaskCount : 대기 중인 Task 수
```

특히 `RunningTaskCount`를 모니터링하면, Task가 죽어서 재시작 중인 상황을 즉시 알 수 있다.
기존처럼 "nginx는 살아있어서 Healthy로 보이지만 실제 앱은 죽어있는" 상황이 사라진다.

---

## 6부: 마이그레이션 후 달라진 점

### 헬스체크 정확도

기존에는 "Unhealthy인데 서비스는 정상" 또는 "Healthy인데 서비스 장애" 상황이 종종 있었다.
ECS로 넘어온 후, 헬스체크 상태와 실제 서비스 상태가 훨씬 일치하게 됐다.

컨테이너 헬스체크가 앱 프로세스를 직접 확인하기 때문이다.

### 장애 복구

기존에는 EC2 안의 Node.js 프로세스가 죽으면:
1. 누군가 직접 SSH로 들어가서
2. 프로세스 상태 확인하고
3. 재시작

이 과정이 필요했다. ECS에서는:
1. Task 헬스체크 실패 감지
2. ECS가 자동으로 Task 교체

사람이 개입할 일이 줄었다.

### 배포 방식

기존에는 배포할 때 각 EC2에 SSH로 들어가서 `git pull` 후 재시작하는 방식이었다.
ECS에서는 새 이미지를 ECR에 푸시하고, Task Definition을 업데이트하면 ECS가 Rolling Update로 배포한다.

```
새 이미지 ECR 푸시
    ↓
Task Definition 새 버전 등록 (새 이미지 태그 반영)
    ↓
ECS Service 업데이트 (새 Task Definition 사용)
    ↓
ECS가 Rolling Update 수행:
  → 새 Task 실행 → 헬스체크 통과 → 구 Task 종료
```

---

## 마치며

이 경험에서 가장 크게 배운 건, **인프라 계층이 많아질수록 관찰 가능성(Observability)이 떨어질 수 있다**는 점이다.

EC2 위에 nginx, nginx 뒤에 앱, 이렇게 레이어가 쌓이면 ALB가 실제 앱 상태를 정확히 보기가 어려워진다.
CloudWatch 알람이 울린다고 해서 그게 실제 서비스 장애를 의미하지 않을 수도 있고,
반대로 알람이 없다고 해서 서비스가 정상이라고 확신할 수도 없게 된다.

컨테이너 기반으로 넘어오면서, ALB와 실제 앱 사이에 불필요한 중간 레이어가 줄었다.
헬스체크가 앱 프로세스를 직접 바라보게 되고, CloudWatch 메트릭이 실제 상태를 더 정확하게 반영한다.

아직 마이그레이션이 완전히 끝난 건 아니지만, 지금까지의 과정에서 "왜 이렇게 설계하는지"를 직접 부딪히며 이해하게 된 것 같다. 그게 이 경험에서 얻은 가장 큰 수확이다.
