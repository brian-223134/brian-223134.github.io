---
title: "[인턴] Sentry 운영 환경 리팩토링"
excerpt: "백엔드 코드의 문제점 파악 및 Trace 가능한 에러 방식으로 수정"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Sentry]

permalink: /intern/company/blitz-dynamics/10/

toc: true
toc_sticky: true

date: 2026-03-13
last_modified_at: 2026-03-13
---

# Sentry Alert Tier로 에러 알림을 운영 가능한 신호로 바꾸기

운영 환경에서 에러가 많아질수록 중요한 문제와 참고성 로그가 한 화면에 섞여 대응 우선순위가 흐려집니다.  
이 글은 **서비스 내부 비즈니스 로직을 노출하지 않고**, Sentry에 `alert_tier`(P0~P3)를 일관되게 부여해 알림 체계를 운영 가능한 형태로 정리한 방법을 소개합니다.

---

## 왜 Alert Tier가 필요한가

일반적으로 `captureException`만 사용하면 에러는 쌓이지만, 다음 질문에 즉답하기 어렵습니다.

- 이 이슈가 지금 즉시 대응 대상인가?
- 운영 알림을 누구에게, 어느 채널로 보내야 하는가?
- 반복 발생하는 경고를 장애처럼 취급하고 있지는 않은가?

핵심은 **"에러 수집"이 아니라 "우선순위 기반 대응"** 입니다.

---

## Tier 설계: P0~P3

| Tier | Sentry Level | 의미 | 권장 대응 시간 |
|------|--------------|------|----------------|
| P0 | `fatal` | 서비스 중단/핵심 흐름 완전 실패 | 즉시 |
| P1 | `error` | 핵심 기능 장애/심각한 병목 | 1시간 이내 |
| P2 | `warning` | 보조 기능 실패, 부분 기능 저하 | 업무시간 내 |
| P3 | `info` | 모니터링성 예외/개선 후보 | 다음 스프린트 |

> `alert_tier`는 Sentry의 내장 Priority를 대체하지 않습니다.  
> 내장 Priority는 Sentry가 자동 산정하고, `alert_tier`는 팀이 의도적으로 정의한 운영 태그입니다.

---

## 구현 전략

### 1) 단일 헬퍼 함수로 수집 경로 통일

모든 에러 보고 지점을 공통 헬퍼로 모으면 다음이 한 번에 해결됩니다.

- `alert_tier` 태그 강제
- tier별 severity 자동 매핑
- tags/extra 형식 일관성 유지

```ts
import * as Sentry from '@sentry/node';

export enum SentryAlertTier {
  P0 = 'P0',
  P1 = 'P1',
  P2 = 'P2',
  P3 = 'P3',
}

const ALERT_TIER_TO_LEVEL = {
  P0: 'fatal',
  P1: 'error',
  P2: 'warning',
  P3: 'info',
} as const;

export function captureWithAlertTier(
  error: Error | unknown,
  options: {
    alertTier: SentryAlertTier;
    tags?: Record<string, string>;
    extra?: Record<string, unknown>;
  },
): string {
  const { alertTier, tags = {}, extra = {} } = options;

  return Sentry.withScope(scope => {
    scope.setTag('alert_tier', alertTier);
    scope.setLevel(ALERT_TIER_TO_LEVEL[alertTier]);

    for (const [key, value] of Object.entries(tags)) scope.setTag(key, value);
    for (const [key, value] of Object.entries(extra)) scope.setExtra(key, value);

    return Sentry.captureException(error);
  });
}
```

### 2) 객체 전달 대신 `Error` 중심으로 전환

스택트레이스가 없는 객체 전달은 디버깅 난이도를 높입니다.  
가능하면 아래처럼 `new Error(...)`로 감싸고, 구조화 데이터는 `extra`로 보냅니다.

```ts
captureWithAlertTier(new Error('외부 연동 실패'), {
  alertTier: SentryAlertTier.P1,
  tags: { module: 'integration', action: 'sync' },
  extra: { partner: 'external-platform', retryCount: 2 },
});
```

### 3) 메시지성 이벤트도 Error 전환 고려

`captureMessage`는 빠르게 기록하기 좋지만, 장애 분석 관점에서는 스택트레이스가 부족할 수 있습니다.  
운영 이슈로 다뤄야 하는 경우라면 Error 이벤트로 통일하는 것이 유리합니다.

---

## 마이그레이션 체크리스트

기존 코드에서 아래 패턴을 우선적으로 정리하면 전환이 빠릅니다.

- 단순 `captureException(error)` 호출
- `captureException(..., { extra })` 형태
- 일반 객체를 직접 예외로 전달하는 코드
- `withScope + setTag/setExtra + captureException` 반복 패턴
- `captureMessage`를 장애 이벤트처럼 사용하는 코드

전환 원칙은 단순합니다.

1. 모든 이벤트에 `alert_tier`를 부여한다.
2. tier에 맞는 level을 자동 매핑한다.
3. Error/Tag/Extra 구조를 표준화한다.

---

## Alert Rule 운영 예시

Sentry Alert Rule을 tier 기준으로 분리하면 알림 피로도를 크게 줄일 수 있습니다.

- **P0**: 즉시 알림(Incident 채널 + 온콜)
- **P1**: 단기 집계 알림(예: 5분 내 N회)
- **P2**: 시간 단위 digest 알림
- **P3**: 알림 없이 대시보드 모니터링

환경은 `production` 기준으로 분리하고, 팀 온콜 정책과 연결해 실제 대응 체계를 완성합니다.

---

## 도입 효과

Alert Tier를 도입하면 기술적으로는 작은 변화지만, 운영에서는 큰 차이가 납니다.

- 장애 대응 우선순위가 명확해짐
- 알림 채널/간격 정책을 체계화할 수 있음
- 사후 분석 시 이벤트 맥락(tags/extra) 품질 향상
- 팀 내 "무엇이 진짜 장애인가"에 대한 합의 형성

결국 핵심은 **에러를 많이 모으는 것**이 아니라,  
**중요한 에러를 빠르게 구분하고, 올바른 사람에게, 올바른 타이밍에 전달하는 것**입니다.