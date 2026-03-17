---
title: "[인턴] Datadog으로 서비스 상태 모니터링하고 분석하기"
excerpt: "Datadog의 Metrics, Logs, APM, Dashboard, Monitor를 실제 업무에서 어떻게 활용했는지 정리한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Datadog, 모니터링]

permalink: /intern/company/blitz-dynamics/12/

toc: true
toc_sticky: true

date: 2026-03-17
last_modified_at: 2026-03-17
---

# Datadog으로 서비스 상태 모니터링하고 분석하기

## 이전 글 요약

[이전 글](/intern/company/blitz-dynamics/11/)에서는 NestJS의 `@Module` + `@Injectable` 패턴으로
에러 모니터링 설정을 주입하는 구조를 만들었습니다.

```ts
MonitoringModule.forRoot({
  serviceName: "api-server",
  environment: process.env.NODE_ENV ?? "development",
  defaultLabels: { team: "backend" },
});
```

이 구조 덕분에 에러를 **일관된 포맷**으로 모니터링 시스템에 전송할 수 있었습니다.
이번 글에서는 그 모니터링 시스템 중 하나인 **Datadog**을 실제로 어떻게 설정하고 분석에 활용했는지 정리합니다.

---

## Datadog이란

Datadog은 **클라우드 환경의 인프라, 애플리케이션, 로그를 통합 관리하는 관측성(Observability) 플랫폼**입니다.

핵심 구성 요소는 세 가지입니다.

- **Metrics**: 시계열 수치 데이터 (CPU, 메모리, 요청 수, 에러율 등)
- **Logs**: 애플리케이션에서 출력한 로그 텍스트
- **Traces**: 요청이 여러 서비스를 거치는 흐름 (APM)

이 세 가지를 하나의 UI에서 연결해서 볼 수 있다는 점이 Datadog의 가장 큰 강점입니다.

---

## Agent 설치와 기본 설정

Datadog은 서버에 **Agent**를 설치해서 데이터를 수집합니다.

### Docker 환경

```yaml
# docker-compose.yml
services:
  datadog-agent:
    image: gcr.io/datadoghq/agent:7
    environment:
      - DD_API_KEY=${DD_API_KEY}
      - DD_SITE=datadoghq.com
      - DD_LOGS_ENABLED=true
      - DD_APM_ENABLED=true
      - DD_DOGSTATSD_NON_LOCAL_TRAFFIC=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
```

### NestJS 앱에 APM 연동

```ts
// main.ts — 반드시 다른 import보다 먼저 실행
import tracer from "dd-trace";

tracer.init({
  service: "api-server",
  env: process.env.NODE_ENV,
  version: process.env.APP_VERSION,
  logInjection: true, // 로그에 trace_id 자동 주입
});
```

`logInjection: true`를 설정하면 로그에 `dd.trace_id`가 자동으로 포함되어,
나중에 로그와 트레이스를 연결해서 볼 수 있습니다.

---

## Metrics 수집과 활용

### 기본 제공 Metrics

Agent를 설치하면 아래 Metrics는 별도 설정 없이 자동 수집됩니다.

| Metric 이름              | 설명               |
| ------------------------ | ------------------ |
| `system.cpu.user`        | CPU 사용률         |
| `system.mem.used`        | 메모리 사용량      |
| `nginx.net.request_per_s`| 초당 요청 수       |
| `docker.cpu.usage`       | 컨테이너 CPU 사용률|

### 커스텀 Metrics (DogStatsD)

애플리케이션에서 직접 수치를 Datadog으로 전송할 수 있습니다.

```ts
import StatsD from "hot-shots";

const dogstatsd = new StatsD({
  host: process.env.DD_AGENT_HOST ?? "localhost",
  port: 8125,
  globalTags: {
    env: process.env.NODE_ENV,
    service: "api-server",
  },
});

// 카운터: 이벤트 발생 횟수
dogstatsd.increment("order.created", 1, { paymentMethod: "card" });

// 게이지: 현재 상태 값
dogstatsd.gauge("queue.pending_jobs", pendingCount);

// 히스토그램: 분포 측정 (p50, p95, p99 자동 계산)
dogstatsd.histogram("order.processing_time_ms", elapsedMs);
```

히스토그램은 평균값 대신 **백분위수(percentile)** 기준으로 분석할 수 있어,
일부 느린 요청이 평균을 왜곡하는 문제를 피할 수 있습니다.

---

## Logs 분석

### 로그 수집 설정

```yaml
# /etc/datadog-agent/conf.d/app.d/conf.yaml
logs:
  - type: docker
    service: api-server
    source: nodejs
    tags:
      - env:production
```

### 구조화 로그 출력

Datadog에서 로그를 필터링하려면 **JSON 포맷**으로 출력하는 것이 효과적입니다.

```ts
import winston from "winston";

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(), // JSON 포맷으로 출력
  ),
  transports: [new winston.transports.Console()],
});

// 출력 예시
// {"level":"error","message":"DB connection failed","orderId":"abc123","timestamp":"2026-03-17T10:00:00.000Z","dd.trace_id":"12345"}
logger.error("DB connection failed", { orderId: "abc123" });
```

### Log Explorer에서 분석하기

Datadog의 **Log Explorer**에서 쿼리 문법으로 로그를 검색할 수 있습니다.

```
# 에러 레벨 로그 중 특정 서비스 필터링
service:api-server status:error

# 특정 필드 값으로 필터링
service:api-server @orderId:abc123

# 최근 1시간 내 에러 로그 수 집계
service:api-server status:error | stats count by @http.status_code
```

`@` 접두사는 JSON 로그의 커스텀 필드를 의미합니다.
`status`, `service` 등 기본 필드는 `@` 없이 사용합니다.

---

## APM (Application Performance Monitoring)

APM은 요청이 여러 서비스와 함수를 거치는 **전체 흐름(Trace)** 을 추적합니다.

### 자동 계측

`dd-trace`는 Express, NestJS, Prisma, Redis 등 주요 라이브러리를 **자동으로 계측**합니다.
별도 코드 없이 HTTP 요청, DB 쿼리, 캐시 호출이 모두 트레이스로 기록됩니다.

### 커스텀 Span 추가

자동 계측이 되지 않는 내부 비즈니스 로직에는 직접 Span을 추가할 수 있습니다.

```ts
import tracer from "dd-trace";

async function processOrder(orderId: string) {
  // 커스텀 Span 생성
  const span = tracer.startSpan("order.process", {
    tags: {
      "order.id": orderId,
      "order.type": "standard",
    },
  });

  try {
    const result = await doHeavyWork(orderId);
    span.setTag("order.result", "success");
    return result;
  } catch (error) {
    span.setTag("error", true);
    span.setTag("error.message", error.message);
    throw error;
  } finally {
    span.finish(); // 반드시 종료 처리
  }
}
```

### Trace와 Log 연결

`logInjection: true` 설정 시, 로그에 `dd.trace_id`가 자동 삽입됩니다.
APM의 특정 트레이스에서 **"View related logs"** 버튼을 누르면,
해당 요청 중 발생한 로그만 바로 필터링해서 볼 수 있습니다.

---

## Dashboard 구성

Dashboard는 여러 Metrics와 로그 집계를 한 화면에 배치해 **서비스 상태를 한눈에 파악**하는 도구입니다.

### 실용적인 위젯 구성

| 위젯 유형        | 활용 예시                          |
| ---------------- | ---------------------------------- |
| Timeseries       | 시간대별 요청 수, 에러율 추이      |
| Top List         | 에러가 많이 발생한 엔드포인트 순위 |
| Query Value      | 현재 활성 사용자 수 (단일 수치)    |
| Heatmap          | 응답 시간 분포                     |
| Log Stream       | 실시간 에러 로그 스트림            |

### 템플릿 변수 활용

Dashboard 상단에 **템플릿 변수**를 추가하면, 환경이나 서비스를 드롭다운으로 전환할 수 있습니다.

```
# Dashboard 설정 예시
템플릿 변수: $env (값: production, staging, development)
위젯 쿼리: avg:system.cpu.user{env:$env}
```

하나의 Dashboard로 환경별 상태를 전환하며 비교할 수 있어 유용합니다.

---

## Monitor와 Alert 설정

Monitor는 Metrics나 로그가 특정 조건을 만족하면 **자동으로 알림을 보내는 기능**입니다.

### Metric Monitor 예시

```
# 에러율 기준 알림 설정
쿼리: sum:trace.express.request.errors{service:api-server}.as_count() /
      sum:trace.express.request.hits{service:api-server}.as_count() * 100

경고(Warning): > 1 (에러율 1% 초과)
위험(Critical): > 5 (에러율 5% 초과)
알림 대상: Slack #alerts 채널
```

### Log Monitor 예시

```
# 특정 에러 메시지가 5분 내 10회 이상 발생 시 알림
쿼리: service:api-server status:error "DB connection failed"
조건: count > 10 (last 5 minutes)
알림: PagerDuty on-call 담당자
```

### 알림 메시지 템플릿

```
{{#is_alert}}
🔴 [Critical] {{service.name}} 에러율 급등
- 현재 에러율: {{value}}%
- 임계값: {{threshold}}%
- 확인 링크: {{[APM Service](https://app.datadoghq.com/apm/service/{{service.name}})}}
{{/is_alert}}

{{#is_recovery}}
✅ [Recovered] {{service.name}} 에러율 정상화
{{/is_recovery}}
```

---

## 실제 분석 흐름 예시

아래는 "특정 시간대에 응답 지연이 발생했다"는 상황에서 실제로 사용한 분석 순서입니다.

### 1단계: Dashboard에서 이상 구간 포착

- Timeseries 위젯에서 `p95` 응답 시간이 평소보다 높아진 구간 확인
- 같은 시간대에 DB 쿼리 수가 급증한 것도 함께 확인

### 2단계: APM에서 느린 Trace 탐색

```
APM > Service > api-server
정렬: Duration 내림차순
필터: duration > 3s
```

- 특정 엔드포인트(`POST /orders`)에서 느린 트레이스 발견
- Trace 상세 보기에서 어느 Span이 오래 걸렸는지 확인 → DB 쿼리 Span이 2.8초 소요

### 3단계: Log Explorer에서 해당 요청 로그 확인

```
@dd.trace_id:1234567890 service:api-server
```

- Trace ID로 해당 요청의 로그를 바로 필터링
- "table scan detected" 경고 로그 발견 → 인덱스 누락이 원인

---

## 정리

Datadog을 실제 업무에 적용하면서 얻은 핵심 배움을 정리합니다.

1. **Agent + APM 설정**으로 인프라 Metrics와 애플리케이션 트레이스를 모두 수집 가능
2. **JSON 구조화 로그** + `logInjection`으로 로그와 트레이스를 연결하면 디버깅 속도가 크게 향상
3. **커스텀 Metrics**는 비즈니스 관점의 수치(주문 수, 대기열 크기 등)를 직접 측정하는 데 유용
4. **Dashboard 템플릿 변수**로 환경별 비교가 용이해져 배포 전후 상태 확인에 활용
5. **Monitor + 알림**은 사람이 직접 보지 않아도 이상 상태를 자동으로 감지하는 핵심 장치

이전 글들에서 만든 에러 분류 체계와 NestJS 모듈 패턴이 **"무엇을, 어떻게 보낼 것인가"** 에 대한 답이었다면,
이번 글의 Datadog 활용은 **"보내진 데이터를 어떻게 보고, 분석하고, 대응할 것인가"** 에 대한 답이라고 생각합니다.
