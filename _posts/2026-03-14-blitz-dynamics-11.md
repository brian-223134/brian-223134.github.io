---
title: "[인턴] NestJS @Module 패턴으로 에러 모니터링 설정 주입하기"
excerpt: "NestJS의 모듈 시스템을 활용해 @Injectable 서비스에 기본 설정을 주입하는 구조로 개선한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, NestJS, Sentry]

permalink: /intern/company/blitz-dynamics/11/

toc: true
toc_sticky: true

date: 2026-03-14
last_modified_at: 2026-03-14
---

# NestJS @Module 패턴으로 에러 모니터링 설정 주입하기

## 이전 글 요약

[이전 글](/intern/company/blitz-dynamics/10/)에서는 에러를 "신호등 체계"로 분류하고,
공통 함수를 통해 일관된 에러 수집 구조를 만드는 방법을 정리했습니다.

```ts
type Signal = "RED" | "ORANGE" | "YELLOW" | "BLUE";

function reportOperationalError(
  error: Error,
  context: {
    signal: Signal;
    labels?: Record<string, string>;
    meta?: Record<string, unknown>;
  },
) {
  return sendToMonitoringSystem({ error, ...context });
}
```

이 방식은 에러를 일관되게 보내는 데 효과적이었지만, 몇 가지 한계가 있었습니다.

- 호출할 때마다 반복되는 설정(기본 레이블, 환경 정보, 서비스명 등)을 매번 전달해야 함
- 서비스마다 다른 기본값을 적용하기 어려움
- 테스트 시 모니터링 설정을 교체하기 번거로움

이번 글에서는 이 구조를 NestJS의 `@Module` + `@Injectable` 패턴으로 확장하여, **기본 설정을 선언적으로 주입**하는 방법을 정리합니다.

---

## NestJS 모듈 시스템이란

NestJS는 **모듈 기반 아키텍처**를 핵심 설계 원칙으로 가지고 있습니다.

- `@Module` 데코레이터로 관련 기능을 하나의 단위로 묶음
- `@Injectable` 데코레이터가 붙은 클래스는 DI(Dependency Injection) 컨테이너에 등록됨
- 모듈이 `providers`에 등록한 의존성은 해당 모듈 내부 또는 `exports`를 통해 외부에 공개 가능

이 구조 덕분에 **설정 객체를 토큰으로 등록하고, 서비스에 자동으로 주입**하는 패턴이 가능합니다.

NestJS 생태계에서 이미 익숙한 예시가 있습니다.

```ts
// ConfigModule은 forRoot()로 설정을 받아 전역 등록
ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' });

// TypeOrmModule도 forRoot()로 DB 연결 설정을 받음
TypeOrmModule.forRoot({ type: 'postgres', host: 'localhost', ... });
```

이 `forRoot()` 패턴을 에러 모니터링에도 동일하게 적용할 수 있습니다.

---

## 구현 구조

### 1) Injection Token과 옵션 인터페이스 정의

```ts
// monitoring.constants.ts
export const MONITORING_OPTIONS = Symbol("MONITORING_OPTIONS");
```

```ts
// monitoring.interfaces.ts
export type Signal = "RED" | "ORANGE" | "YELLOW" | "BLUE";

export interface MonitoringModuleOptions {
  serviceName: string;
  environment: string;
  defaultSignal?: Signal;
  defaultLabels?: Record<string, string>;
  enabled?: boolean;
}
```

`Symbol`을 Injection Token으로 사용하면, NestJS의 DI 컨테이너가 이 토큰을 키로 삼아 설정 객체를 찾아 주입합니다.

---

### 2) @Module 데코레이터로 동적 모듈 구성

```ts
// monitoring.module.ts
import { DynamicModule, Module } from "@nestjs/common";
import {
  MONITORING_OPTIONS,
  MonitoringModuleOptions,
} from "./monitoring.interfaces";
import { MonitoringService } from "./monitoring.service";

@Module({})
export class MonitoringModule {
  static forRoot(options: MonitoringModuleOptions): DynamicModule {
    return {
      module: MonitoringModule,
      providers: [
        {
          provide: MONITORING_OPTIONS,
          useValue: {
            defaultSignal: "YELLOW",
            enabled: true,
            ...options, // 호출 시 전달한 값이 기본값을 덮어씀
          },
        },
        MonitoringService,
      ],
      exports: [MonitoringService],
    };
  }
}
```

핵심은 `forRoot()` 내부에서 **기본값을 먼저 선언하고, 호출 시 전달받은 옵션으로 병합**하는 부분입니다.

- `defaultSignal`을 지정하지 않으면 `'YELLOW'`가 기본값
- `enabled`를 지정하지 않으면 `true`가 기본값
- `serviceName`, `environment`는 반드시 전달해야 하는 필수값

이 방식은 NestJS의 `ConfigModule.forRoot()`나 `TypeOrmModule.forRoot()`와 동일한 패턴입니다.

---

### 3) @Injectable 서비스에서 설정 주입받기

```ts
// monitoring.service.ts
import { Inject, Injectable } from "@nestjs/common";
import {
  MONITORING_OPTIONS,
  MonitoringModuleOptions,
  Signal,
} from "./monitoring.interfaces";

@Injectable()
export class MonitoringService {
  constructor(
    @Inject(MONITORING_OPTIONS)
    private readonly options: MonitoringModuleOptions,
  ) {}

  report(
    error: Error,
    context?: Partial<{
      signal: Signal;
      labels: Record<string, string>;
      meta: Record<string, unknown>;
    }>,
  ) {
    if (!this.options.enabled) return;

    const signal = context?.signal ?? this.options.defaultSignal;
    const labels = {
      service: this.options.serviceName,
      env: this.options.environment,
      ...this.options.defaultLabels,
      ...context?.labels,
    };

    return sendToMonitoringSystem({
      error,
      signal,
      labels,
      meta: context?.meta,
    });
  }
}
```

`@Inject(MONITORING_OPTIONS)`을 통해 모듈 등록 시 정의한 설정 객체가 **생성자에 자동으로 주입**됩니다.

이제 `report()` 호출 시:

- **서비스명, 환경, 기본 레이블**은 모듈 설정에서 자동으로 채워짐
- 호출 시점에서는 **신호 단계와 추가 컨텍스트**만 전달하면 됨
- 신호 단계를 생략하면 `forRoot()`에서 설정한 `defaultSignal`이 사용됨

---

### 4) 앱 모듈에서 등록하기

```ts
// app.module.ts
import { Module } from "@nestjs/common";
import { MonitoringModule } from "./monitoring/monitoring.module";

@Module({
  imports: [
    MonitoringModule.forRoot({
      serviceName: "api-server",
      environment: process.env.NODE_ENV ?? "development",
      defaultLabels: {
        team: "backend",
      },
    }),
  ],
})
export class AppModule {}
```

앱 전체에서 `MonitoringService`를 주입받으면,  
등록 시점에 설정한 기본값이 **모든 에러 리포팅에 자동 적용**됩니다.

---

## 사용 예시

### 비즈니스 로직에서의 사용

```ts
@Injectable()
export class OrderService {
  constructor(private readonly monitoring: MonitoringService) {}

  async processOrder(orderId: string) {
    try {
      // 주문 처리 로직...
    } catch (error) {
      // signal과 추가 labels만 전달하면 됨
      // serviceName, environment, team 등은 자동 포함
      this.monitoring.report(error, {
        signal: "RED",
        labels: { orderId },
      });
      throw error;
    }
  }
}
```

### 테스트에서의 활용

```ts
// 테스트 모듈에서는 모니터링을 비활성화하거나 별도 설정 적용
const moduleRef = await Test.createTestingModule({
  imports: [
    MonitoringModule.forRoot({
      serviceName: "test",
      environment: "test",
      enabled: false, // 테스트 시 실제 전송 비활성화
    }),
  ],
}).compile();
```

---

## 이전 방식과의 비교

| 항목          | 함수 호출 방식 (이전) | @Module 주입 방식 (현재)             |
| ------------- | --------------------- | ------------------------------------ |
| 기본 설정     | 매 호출마다 수동 전달 | 모듈 등록 시 1회 설정                |
| 설정 변경     | 모든 호출 지점 수정   | `forRoot()` 옵션만 수정              |
| 테스트        | mock 함수 직접 구성   | 테스트 모듈에서 설정 교체            |
| 서비스별 분리 | 어려움                | 모듈별 다른 설정 가능                |
| DI 통합       | 없음                  | NestJS DI 컨테이너와 자연스럽게 통합 |

---

## 확장: forRootAsync 패턴

설정값을 환경 변수나 외부 서비스에서 비동기로 가져와야 하는 경우,  
`forRootAsync()` 패턴으로 확장할 수 있습니다.

```ts
@Module({})
export class MonitoringModule {
  static forRoot(options: MonitoringModuleOptions): DynamicModule {
    // ... (위와 동일)
  }

  static forRootAsync(optionsFactory: {
    imports?: any[];
    useFactory: (
      ...args: any[]
    ) => MonitoringModuleOptions | Promise<MonitoringModuleOptions>;
    inject?: any[];
  }): DynamicModule {
    return {
      module: MonitoringModule,
      imports: optionsFactory.imports ?? [],
      providers: [
        {
          provide: MONITORING_OPTIONS,
          useFactory: optionsFactory.useFactory,
          inject: optionsFactory.inject ?? [],
        },
        MonitoringService,
      ],
      exports: [MonitoringService],
    };
  }
}
```

```ts
// ConfigService에서 비동기로 설정을 가져오는 예시
MonitoringModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (config: ConfigService) => ({
    serviceName: config.get("SERVICE_NAME"),
    environment: config.get("NODE_ENV"),
    enabled: config.get("MONITORING_ENABLED") === "true",
  }),
  inject: [ConfigService],
});
```

이 패턴은 NestJS 공식 모듈들(`TypeOrmModule`, `JwtModule` 등)이 사용하는 표준적인 방식입니다.

---

## 정리

이전 글에서 만든 공통 에러 리포팅 함수를 NestJS의 모듈 시스템으로 감싸면서 얻은 변화를 정리합니다.

1. **@Module의 `forRoot()` 패턴**으로 기본 설정을 한 곳에서 선언적으로 관리
2. **@Injectable 서비스**가 `@Inject` 토큰을 통해 설정을 자동으로 받아 사용
3. 호출 시점에서는 **신호 단계와 부가 정보만 전달**하면 되므로 코드 중복 감소
4. 테스트 시 **모듈 단위로 설정을 교체**할 수 있어 격리된 테스트 구성이 용이
5. `forRootAsync()`로 확장하면 **비동기 설정 로딩**에도 대응 가능

결국 NestJS의 모듈 시스템은 단순한 코드 구조화 도구가 아니라,  
**"기본값과 설정을 선언적으로 관리하고 주입하는 패턴"**으로 활용할 수 있다는 것이 가장 큰 배움이었습니다.

이전 글의 "신호 분류 체계"가 **무엇을 보낼 것인가**에 대한 답이었다면,  
이번 글의 모듈 패턴은 **어떻게 일관되게 보낼 것인가**에 대한 답이라고 생각합니다.
