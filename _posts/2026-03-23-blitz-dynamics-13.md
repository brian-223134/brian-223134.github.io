---
title: "[인턴] .env와 TypeScript environments 디렉토리의 역할 분리"
excerpt: ".env는 값 주입, environments/*.ts는 검증·변환·캐싱을 담당하는 이유를 실무 관점에서 정리한 글"

categories:
    - Company
tags:
    - [블리츠다이나믹스, 산학, 인턴, 백엔드, Nodejs, Typescript, Environment-Variables]

permalink: /intern/company/blitz-dynamics/13/

toc: true
toc_sticky: true

date: 2026-03-23
last_modified_at: 2026-03-23
---

# .env vs TypeScript environments/ — 환경 변수 관리의 두 계층

## 이전 글 요약

[이전 글](/intern/company/blitz-dynamics/12/)에서는 Datadog으로 서비스 상태를 관측하고,
Metrics·Logs·APM을 연결해 운영 지표를 해석하는 방법을 정리했습니다.

이번 글에서는 모니터링 못지않게 중요한 운영 안정성 주제,
즉 **환경 변수를 어떻게 안전하게 관리할 것인가**를 다룹니다.

---

## 배경: 환경 변수는 왜 두 곳에 있는가?

Node.js 백엔드 프로젝트를 보면 보통 아래 두 파일이 함께 존재합니다.

```text
.env.production          ← 값을 주입
src/environments/app.ts  ← 값을 검증·변환·캐싱
```

겉보기에는 둘 다 "환경 변수 파일"처럼 보이지만, 실제 책임은 완전히 다릅니다.

---

## .env 파일: 값을 "주입"하는 계층

`.env` 파일은 런타임에 `process.env`로 올라갈 **원본 문자열(raw string)** 을 저장합니다.

```env
# .env.production
APP_PORT=3000
TRACE_ENABLED=true
TRACE_SAMPLE_RATE=0.5
CAPTURE_PAYLOAD=false
```

핵심 특징은 다음과 같습니다.

- 모든 값이 문자열입니다. (`TRACE_ENABLED=true`도 실제로는 `"true"`)
- 값이 잘못돼도 파일 로드는 성공합니다. (`TRACE_SAMPLE_RATE=abc` 같은 오타 포함)
- 자체 검증이 없어서 오류가 런타임까지 조용히 흘러갈 수 있습니다.
- 환경별로 분리 관리합니다. (`.env.local`, `.env.staging`, `.env.production`)

즉 `.env`는 **무엇을 넣을지 정의하는 입력 레이어**입니다.

---

## environments/app.ts: 값을 "검증·변환·캐싱"하는 계층

TypeScript의 `environments` 디렉토리는 `process.env` 값을 안전한 타입으로 정제해
애플리케이션 전체에서 일관되게 사용하도록 만드는 방어 계층입니다.

```ts
// src/environments/app.ts

const TRACE_ENABLED = envToBoolean("TRACE_ENABLED", false);
const TRACE_SAMPLE_RATE = envToNumber("TRACE_SAMPLE_RATE", 1, { min: 0, max: 1 });

const API_KEY = process.env.API_KEY;
if (!API_KEY) throw new Error("API_KEY is required");

export function getTraceEnabled(): boolean {
    return TRACE_ENABLED;
}

export function getTraceSampleRate(): number {
    return TRACE_SAMPLE_RATE;
}
```

핵심 특징은 다음과 같습니다.

- 모듈 로드 시 1회 실행되어 서버 시작 시점에 검증이 끝납니다. (fail-fast)
- 잘못된 값이면 즉시 에러를 던져 서버 시작을 막습니다.
- 호출부는 `string`이 아닌 `boolean`, `number` 타입을 받습니다.
- 모듈 수준 캐싱으로 반복 파싱 비용이 없습니다.

---

## 두 계층의 차이 한눈에 보기

| 항목 | `.env` 파일 | `environments/app.ts` |
| --- | --- | --- |
| 역할 | 값 주입 | 검증 + 변환 + 캐싱 |
| 값의 타입 | 항상 string | boolean, number, string |
| 검증 시점 | 없음 | 서버 시작 시(모듈 로드) |
| 잘못된 값 처리 | 무시되거나 런타임 버그 | 즉시 에러, 서버 시작 실패 |
| 접근 방식 | `process.env.TRACE_ENABLED` | `getTraceEnabled()` |
| 주체 | DevOps / 배포 담당자 | 개발자 |

---

## 왜 getter 패턴으로 감싸는가?

```ts
// ❌ 직접 접근: 타입 없음, 검증 분산
const enabled = process.env.TRACE_ENABLED === "true";

// ✅ getter 접근: 타입 보장, 단일 진입점, 캐싱
const enabled = getTraceEnabled(); // boolean
```

getter 패턴을 쓰면 다음 이점이 있습니다.

- 타입 안전: `process.env.*`의 `string | undefined`를 도메인 타입으로 고정
- 단일 진입점: 검증 로직이 한 곳에만 있어 변경 영향 관리가 쉬움
- 테스트 용이: getter mock으로 테스트 격리가 쉬움
- 캐싱 효과: 이미 계산된 상수를 재사용

---

## 실제 실행 흐름

```text
서버 시작
        │
        ├── dotenv.config() 실행
        │       └── .env.production → process.env 적재
        │
        ├── import "src/environments/app.ts"
        │       ├── envToBoolean("TRACE_ENABLED") → "true" → true ✅
        │       ├── envToNumber("TRACE_SAMPLE_RATE", 1, { min: 0, max: 1 })
        │       │       └── "abc" → ❌ Error thrown → 서버 시작 실패
        │       └── 캐싱 완료
        │
        └── 이후 getTraceEnabled() 호출 → 캐시된 값 즉시 반환
```

---

## 요약

`.env`는 **값을 주입**하는 역할이고,
`environments/*.ts`는 그 값이 올바른지 **검증·변환·캐싱**하는 역할입니다.

`.env`만 사용하면 타입 부재와 검증 누락으로 런타임 장애 위험이 커집니다.
반면 `environments/*.ts`를 두면 서버 시작 시점에 문제를 조기에 차단하고,
애플리케이션 전역에서 타입 안전한 설정값을 일관되게 사용할 수 있습니다.