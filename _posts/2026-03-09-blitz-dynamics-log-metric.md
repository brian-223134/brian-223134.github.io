---
title: "[블리츠다이나믹스] 로그 & 메트릭 수집 툴 정리 및 NestJS 적용"
excerpt: "다양한 로그·메트릭 수집 도구를 비교 정리하고, NestJS 백엔드에서 실제로 활용하기 좋은 조합을 소개합니다."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, NestJS, 로그, 메트릭, 모니터링]

permalink: /intern/company/blitz-dynamics/log-metric/

toc: true
toc_sticky: true

date: 2026-03-09
last_modified_at: 2026-03-09
---

# 들어가며

서비스가 운영되기 시작하면 **"지금 서버가 잘 돌아가고 있는가?"** 라는 질문에 빠르게 답할 수 있어야 한다.
이를 위한 핵심 수단이 바로 **로그(Log)**와 **메트릭(Metric)**이다.

- **로그**: 특정 시점에 어떤 일이 일어났는지를 기록한 텍스트 데이터 (예: "유저 A가 로그인했다", "DB 쿼리가 실패했다")
- **메트릭**: 시간에 따라 변화하는 수치 데이터 (예: CPU 사용률, API 응답 시간, 에러 발생 횟수)

두 가지는 상호 보완적이다. 메트릭으로 이상 징후를 감지하고, 로그로 원인을 추적하는 흐름이 일반적이다.

이 글에서는 다양한 로그·메트릭 수집 도구를 정리하고, 블리츠다이나믹스 백엔드 프레임워크인 **NestJS**에서 실용적으로 활용할 수 있는 조합을 설명한다.

---

# 1. 로그·메트릭 수집 툴 전체 정리

## 1-1. 로그 수집·관리 도구

### ELK Stack (Elasticsearch + Logstash + Kibana)

가장 널리 알려진 로그 관리 스택이다.

| 구성 요소 | 역할 |
|---|---|
| **Elasticsearch** | 로그 데이터를 저장하고 빠르게 검색하는 분산 검색 엔진 |
| **Logstash** | 다양한 소스에서 로그를 수집·파싱·변환하는 파이프라인 |
| **Kibana** | Elasticsearch의 데이터를 시각화하는 대시보드 |

- **장점**: 풍부한 생태계, 강력한 전문 검색, 커스텀 대시보드
- **단점**: 자원 소모가 크고 구축·운영 난이도가 높음
- **적합한 상황**: 대용량 로그를 상세하게 분석해야 할 때

---

### EFK Stack (Elasticsearch + Fluentd + Kibana)

Logstash 대신 **Fluentd**를 사용하는 스택이다.

- **Fluentd**: Ruby 기반의 경량 로그 수집기. 플러그인 생태계가 풍부하고 메모리 사용량이 적다.
- **Fluent Bit**: Fluentd의 경량 버전. C로 작성되어 컨테이너·임베디드 환경에 적합하다.
- **적합한 상황**: Kubernetes 환경에서 각 Pod의 로그를 중앙화할 때

---

### Loki (Grafana Loki)

Prometheus와 잘 통합되는 **로그 집계 시스템**이다. Grafana Labs에서 개발했다.

- 로그를 인덱싱하지 않고 **레이블(label) 기반으로 압축 저장**하기 때문에 저장 비용이 매우 낮다.
- Grafana와 네이티브로 연동되어 메트릭과 로그를 한 화면에서 함께 볼 수 있다.
- **장점**: 저렴한 운영 비용, Prometheus 생태계와의 자연스러운 통합
- **단점**: 전문 검색(full-text search)이 약함 (레이블 기반 필터링 위주)
- **적합한 상황**: Prometheus + Grafana 스택을 이미 사용 중인 환경

---

### Datadog Logs

**Datadog**은 SaaS 형태의 올인원 모니터링 플랫폼으로, 로그 수집 기능도 포함한다.

- 에이전트 설치만으로 로그·메트릭·트레이스를 한꺼번에 수집
- 머신러닝 기반의 이상 탐지, 실시간 알림 기능 제공
- **장점**: 구축 부담이 거의 없고, APM·인프라 메트릭과 완벽 통합
- **단점**: 비용이 높음 (데이터 볼륨에 따라 급격히 상승)
- **적합한 상황**: 빠른 도입이 필요하거나 인프라 운영 인력이 적을 때

---

### CloudWatch Logs (AWS)

AWS 네이티브 로그 관리 서비스다.

- EC2, Lambda, ECS 등 AWS 서비스와 자동 연동
- **CloudWatch Logs Insights**로 SQL과 유사한 문법의 로그 쿼리 가능
- **장점**: AWS 환경에서 별도 설정 없이 사용 가능
- **단점**: 시각화 기능이 부족하고 대용량 쿼리 비용이 높을 수 있음
- **적합한 상황**: AWS 인프라 기반 서비스의 기본 로그 수집

---

## 1-2. 메트릭 수집·모니터링 도구

### Prometheus

현재 메트릭 모니터링의 **사실상 표준(de facto standard)**이다.

- **Pull 방식**: Prometheus가 주기적으로 애플리케이션의 `/metrics` 엔드포인트를 호출하여 수집
- **PromQL**: 강력한 시계열 쿼리 언어로 다양한 집계·연산 가능
- **Alertmanager**: 임계값 초과 시 Slack, 이메일 등으로 알림 발송
- **장점**: 오픈소스, 거대한 생태계 (exporter가 수백 종 존재), Kubernetes와 궁합이 좋음
- **단점**: 장기 데이터 보존을 위해 별도 스토리지(Thanos, Cortex) 연동 필요
- **적합한 상황**: 컨테이너 기반 서비스의 메트릭 수집

---

### Grafana

**시각화 전문 도구**로, 다양한 데이터 소스(Prometheus, Loki, CloudWatch, Datadog 등)를 연결하여 대시보드를 구성할 수 있다.

- Prometheus와 함께 사용하는 것이 가장 일반적인 조합
- 알림(Alert) 기능도 내장
- **장점**: 다양한 데이터 소스 지원, 풍부한 시각화 옵션, 무료 오픈소스
- **적합한 상황**: 메트릭·로그·트레이스를 통합하여 한 화면에서 확인할 때

---

### Datadog APM & Metrics

로그뿐 아니라 **인프라 메트릭과 APM(Application Performance Monitoring)**을 통합 제공한다.

- 분산 트레이싱(Distributed Tracing)으로 마이크로서비스 간 요청 흐름 추적
- **NPM(Network Performance Monitoring)**으로 서비스 간 네트워크 성능 가시화
- **적합한 상황**: 빠른 도입이 필요한 스타트업, 인프라+앱+로그를 단일 플랫폼으로 관리하고 싶을 때

---

### AWS CloudWatch Metrics

EC2, RDS, Lambda 등 AWS 서비스의 메트릭을 자동으로 수집·저장한다.

- **CloudWatch Alarms**: 메트릭 임계값 초과 시 SNS, Lambda 등으로 자동 대응
- **CloudWatch Dashboards**: 기본적인 시각화 제공
- **적합한 상황**: AWS 인프라의 기본 메트릭 수집 및 경보 설정

---

### StatsD + Graphite

전통적인 메트릭 수집 스택이다.

- **StatsD**: UDP로 애플리케이션에서 카운터·타이머 데이터를 수집하는 경량 데몬
- **Graphite**: 시계열 데이터를 저장하고 그래프로 시각화
- **현황**: Prometheus의 등장 이후 신규 도입은 줄고 있으나 레거시 시스템에서 여전히 사용됨

---

### New Relic

Datadog과 유사한 SaaS 형태의 풀스택 모니터링 플랫폼이다.

- APM, 인프라 메트릭, 로그, 합성 모니터링(Synthetic Monitoring)을 통합 제공
- **무료 티어**가 비교적 넉넉하여 소규모 팀에서도 사용 가능
- **적합한 상황**: Datadog보다 비용을 줄이고 싶을 때

---

### OpenTelemetry (OTel)

로그·메트릭·트레이스 수집을 위한 **벤더 중립 표준**이다.

- 특정 모니터링 플랫폼에 종속되지 않고, 동일한 코드로 Datadog, Prometheus, Jaeger 등 여러 백엔드로 데이터를 내보낼 수 있다.
- **장점**: 벤더 락인(vendor lock-in) 방지, 업계 표준으로 자리잡는 중
- **단점**: 설정 복잡도가 있고 아직 일부 기능이 성숙하지 않음
- **적합한 상황**: 향후 모니터링 플랫폼 교체 가능성을 고려할 때

---

## 1-3. 도구 비교 요약

| 도구 | 유형 | 운영 형태 | 강점 | 약점 |
|---|---|---|---|---|
| ELK Stack | 로그 | 자체 운영 | 강력한 검색, 풍부한 생태계 | 높은 자원 소모 |
| Loki + Grafana | 로그 | 자체 운영 | 저렴한 비용, Prometheus 통합 | 전문 검색 취약 |
| Datadog | 로그+메트릭+APM | SaaS | 올인원, 빠른 도입 | 비용 높음 |
| CloudWatch | 로그+메트릭 | AWS 관리형 | AWS 네이티브 통합 | 시각화 부족 |
| Prometheus | 메트릭 | 자체 운영 | 표준, 강력한 쿼리 | 장기 저장 별도 구성 필요 |
| Grafana | 시각화 | 자체 운영 | 다양한 소스 통합 | 단독으로 수집 불가 |
| New Relic | 로그+메트릭+APM | SaaS | 무료 티어, 올인원 | Datadog 대비 생태계 협소 |
| OpenTelemetry | 표준/SDK | - | 벤더 중립 | 설정 복잡 |

---

# 2. NestJS에서 활용하기 좋은 로그·메트릭 도구

블리츠다이나믹스의 백엔드는 **NestJS**로 구축되어 있다. NestJS는 TypeScript 기반의 Node.js 프레임워크로, 모듈화된 구조와 데코레이터 패턴을 특징으로 한다.

---

## 2-1. 로그: Winston + nest-winston

### 왜 Winston인가?

NestJS의 기본 Logger는 기능이 단순하여 프로덕션 환경에서는 부족하다. **Winston**은 Node.js 생태계에서 가장 널리 사용되는 로깅 라이브러리로, 다음을 지원한다.

- **로그 레벨** 구분 (error, warn, info, http, debug 등)
- **다양한 Transport** (파일, 콘솔, HTTP, 외부 서비스 등으로 전송)
- **JSON 형식 출력** (로그 수집기가 파싱하기 좋은 구조화된 형태)

**nest-winston**은 Winston을 NestJS의 Logger 인터페이스에 맞게 통합해주는 패키지다.

### 설치

```bash
npm install winston nest-winston
```

### 기본 설정 예시

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

@Module({
  imports: [
    WinstonModule.forRoot({
      transports: [
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.colorize(),
            winston.format.printf(({ timestamp, level, message, context }) => {
              return `${timestamp} [${context}] ${level}: ${message}`;
            }),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/combined.log',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
      ],
    }),
  ],
})
export class AppModule {}
```

### 사용 예시

```typescript
// some.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';

@Injectable()
export class SomeService {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}

  doSomething() {
    this.logger.info('작업 시작', { context: 'SomeService' });
    try {
      // ...
    } catch (error) {
      this.logger.error('작업 실패', { context: 'SomeService', error });
    }
  }
}
```

---

## 2-2. HTTP 요청 로그: Morgan 미들웨어

API 서버에서는 **모든 HTTP 요청·응답 정보를 자동으로 로깅**하는 것이 중요하다. **Morgan**은 Express(NestJS의 기반) 생태계에서 표준적으로 사용되는 HTTP 요청 로거다.

### 설치

```bash
npm install morgan
npm install -D @types/morgan
```

### NestJS에서 Morgan + Winston 연결

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { WINSTON_MODULE_NEST_PROVIDER } from 'nest-winston';
import * as morgan from 'morgan';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const logger = app.get(WINSTON_MODULE_NEST_PROVIDER);
  app.useLogger(logger);

  // Morgan 로그를 Winston으로 전달
  app.use(
    morgan('combined', {
      stream: {
        write: (message: string) => logger.log(message.trim(), 'HTTP'),
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

---

## 2-3. 메트릭: Prometheus + @willsoto/nestjs-prometheus

NestJS에서 Prometheus 메트릭을 노출하는 데 가장 많이 사용되는 패키지가 **@willsoto/nestjs-prometheus**다. `prom-client` 라이브러리 위에 NestJS 친화적인 래퍼를 제공한다.

### 설치

```bash
npm install @willsoto/nestjs-prometheus prom-client
```

### 기본 설정

```typescript
// app.module.ts
import { PrometheusModule } from '@willsoto/nestjs-prometheus';

@Module({
  imports: [
    PrometheusModule.register({
      path: '/metrics',       // Prometheus가 수집할 엔드포인트
      defaultMetrics: {
        enabled: true,        // Node.js 기본 메트릭 자동 수집 (힙 메모리, CPU 등)
      },
    }),
  ],
})
export class AppModule {}
```

### 커스텀 메트릭 예시 (HTTP 요청 카운터)

```typescript
// http-metrics.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { Counter, Histogram } from 'prom-client';
import { Observable, tap } from 'rxjs';

@Injectable()
export class HttpMetricsInterceptor implements NestInterceptor {
  constructor(
    @InjectMetric('http_requests_total')
    private readonly requestCounter: Counter<string>,
    @InjectMetric('http_request_duration_seconds')
    private readonly requestDuration: Histogram<string>,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const end = this.requestDuration.startTimer({
      method: req.method,
      route: req.route?.path ?? req.url,
    });

    return next.handle().pipe(
      tap(() => {
        const res = context.switchToHttp().getResponse();
        this.requestCounter.inc({
          method: req.method,
          route: req.route?.path ?? req.url,
          status: String(res.statusCode),
        });
        end({ status: String(res.statusCode) });
      }),
    );
  }
}
```

```typescript
// app.module.ts (메트릭 등록 추가)
import {
  makeCounterProvider,
  makeHistogramProvider,
} from '@willsoto/nestjs-prometheus';

@Module({
  imports: [PrometheusModule.register({ path: '/metrics' })],
  providers: [
    makeCounterProvider({
      name: 'http_requests_total',
      help: '총 HTTP 요청 수',
      labelNames: ['method', 'route', 'status'],
    }),
    makeHistogramProvider({
      name: 'http_request_duration_seconds',
      help: 'HTTP 요청 처리 시간 (초)',
      labelNames: ['method', 'route', 'status'],
      buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
    }),
  ],
})
export class AppModule {}
```

---

## 2-4. 로그 수집 백엔드 연동: Loki Transport

Winston으로 수집한 로그를 **Grafana Loki**로 직접 전송할 수 있다. `winston-loki` 패키지를 사용한다.

### 설치

```bash
npm install winston-loki
```

### Winston Transport에 Loki 추가

```typescript
import LokiTransport from 'winston-loki';

WinstonModule.forRoot({
  transports: [
    new LokiTransport({
      host: 'http://localhost:3100', // Loki 서버 주소
      labels: { app: 'blitz-dynamics-api' },
      json: true,
      format: winston.format.json(),
    }),
    // 기존 콘솔·파일 transport도 유지
  ],
})
```

이렇게 하면 Grafana에서 Loki 데이터소스를 추가하여 애플리케이션 로그와 Prometheus 메트릭을 **한 대시보드에서 함께 확인**할 수 있다.

---

## 2-5. 권장 조합 정리

블리츠다이나믹스 환경(AWS + NestJS)에서 현실적으로 추천하는 스택은 다음과 같다.

### 단기 (빠른 도입)

```
로그:    Winston + nest-winston → CloudWatch Logs (AWS 네이티브)
메트릭:  AWS CloudWatch Metrics (EC2 기본 메트릭)
시각화:  AWS CloudWatch Dashboards
```

> 별도 인프라 없이 AWS 기능만으로 빠르게 가시성을 확보할 수 있다.

---

### 중기 (운영 안정화 단계)

```
로그:    Winston + nest-winston → Loki
메트릭:  @willsoto/nestjs-prometheus → Prometheus
시각화:  Grafana (Loki + Prometheus 통합 대시보드)
알림:    Grafana Alerting 또는 Alertmanager → Slack
```

> 비용 효율적이고 자체 운영 가능한 오픈소스 스택. Prometheus + Grafana + Loki의 조합은 현재 가장 널리 채택되는 구성이다.

---

### 장기 (확장 고려)

```
표준화:  OpenTelemetry SDK (NestJS에서 로그·메트릭·트레이스를 OTel로 계측)
백엔드:  Prometheus (메트릭) + Loki (로그) + Tempo (트레이스)
시각화:  Grafana
```

> 특정 벤더에 종속되지 않고, 향후 Datadog이나 다른 플랫폼으로 전환할 때도 애플리케이션 코드 변경 없이 백엔드만 교체 가능하다.

---

# 3. 전체 흐름 요약

```
[NestJS 애플리케이션]
  ├── Winston (로그)
  │     ├── Console Transport    → 개발 환경 디버깅
  │     ├── File Transport       → 로컬 파일 저장
  │     └── Loki Transport       → Grafana Loki
  │
  └── prom-client (메트릭)
        └── /metrics 엔드포인트  → Prometheus (Pull)
                                       └── Grafana 대시보드 + 알림

[AWS 인프라]
  ├── CloudWatch Logs   (기본 로그 백업)
  └── CloudWatch Metrics (EC2 인프라 메트릭)
```

로그와 메트릭을 동시에 수집하는 구조를 갖추면, **문제 발생 시 메트릭으로 먼저 이상을 감지하고 로그로 원인을 추적**하는 빠른 대응 사이클을 만들 수 있다.