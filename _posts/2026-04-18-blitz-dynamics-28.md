---
title: "[인턴] 프로덕션 백엔드가 같은 패턴으로 하루에 세 번 죽었다: NestJS AllExceptionsFilter × GraphQL Subscription, 그리고 빌드에서 누락된 정적 자산"
excerpt: "오늘 동일 stacktrace로 세 번 반복 crash한 ECS Fargate 태스크의 원인을 추적한 기록. AllExceptionsFilter가 GraphQL subscription 컨텍스트에서 Express req를 만지다가 죽는 패턴과, 빌드 산출물에서 빠진 .html 자산으로 ENOENT가 터진 두 가지 사건을 정리"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 백엔드, NestJS, GraphQL, Subscription, ExceptionFilter, ECS, Fargate, 장애대응]

permalink: /intern/company/blitz-dynamics/28/

toc: true
toc_sticky: true

date: 2026-04-18
last_modified_at: 2026-04-18
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/27/)에서는 프론트엔드 인프라 전환과 DNS 카나리 실패기를 정리했습니다. 그쪽이 일단락된 직후, 이번에는 백엔드 쪽이 하루 동안 같은 패턴으로 세 번 죽는 일이 있었습니다.

증상은 이렇습니다.

- 오늘 ECS Fargate 태스크가 동일한 형태로 반복 종료됨
  - `aee9...` → 14:13 사망
  - `3468...` → 14:44 사망
  - `918a...` → 15:24 사망 (`stopCode: EssentialContainerExited`, `exitCode: 1`)
- 모두 죽기 직전 로그에 **같은 stacktrace**가 찍힘
- 한 컨테이너가 죽으면 ECS가 새 태스크를 띄우지만, 같은 트리거가 또 들어와서 같은 식으로 다시 죽는 루프

이번 글은 그 trace를 따라가서 원인 두 개(주범 + 곁가지로 발견된 ENOENT)를 어떻게 격리하고 정리했는지에 대한 기록입니다.

> ⚠️ **민감 정보 마스킹**: 사내 서비스/모듈명, task definition revision, 일부 파일 경로 등은 일반화하거나 마스킹했습니다.

---

## 1. 증상: 같은 stacktrace로 반복 종료

ECS 콘솔에서 stopped tasks를 훑어보면 사인이 일관됩니다.

```
task aee9...   stoppedReason: Essential container in task exited
task 3468...   stoppedReason: Essential container in task exited
task 918a...   stoppedReason: Essential container in task exited
                stopCode: EssentialContainerExited
                exitCode: 1
```

`exitCode: 1`은 컨테이너 안의 Node 프로세스가 비정상 종료됐다는 뜻입니다. 즉 ECS가 죽인 게 아니라 **앱이 스스로 죽어서 컨테이너가 따라 죽은** 케이스입니다.

CloudWatch Logs에서 각 태스크의 마지막 수십 줄을 받아보면, 죽기 직전에 동일한 패턴이 찍혀 있었습니다.

```
TypeError: Cannot read properties of undefined (reading 'originalUrl')
    at ExpressAdapter.getRequestUrl
    at AllExceptionsFilter.catch (apps/backend/build/.../exception-filters/index.js:32:35)
    at ExternalExceptionsHandler.invokeCustomFilters
    ...
    at async executeSubscription (graphql/execution/subscribe.js:230:25)
    at async onMessage (@nestjs/graphql/.../graphql-ws/lib/server.js:191:51)
```

세 태스크 모두 정확히 같은 모양이었기 때문에, 어떤 일회성 결함이 아니라 **재현 가능한 코드 경로**가 있다는 뜻이었습니다.

---

## 2. 1차 원인: AllExceptionsFilter가 Subscription 컨텍스트에서 Express req를 만짐

stacktrace의 결정적 단서는 두 줄입니다.

- `executeSubscription` (graphql-ws)
- `ExpressAdapter.getRequestUrl` 안에서 `originalUrl`이 undefined

조합해서 읽으면 흐름이 다음과 같이 그려집니다.

1. 클라이언트가 **GraphQL Subscription**(WebSocket, `graphql-ws`)으로 접속
2. JWT 검증 실패 → `UnauthorizedException` 발생
3. NestJS가 글로벌 `AllExceptionsFilter.catch()`로 예외를 위임
4. 필터 안에서 `httpAdapter.getRequestUrl(ctx.getRequest())` 호출
5. **subscription 컨텍스트에는 Express의 req 객체가 없음** → `req`가 `undefined`
6. `ExpressAdapter.getRequestUrl`이 `req.originalUrl`을 읽으려다 `TypeError`
7. 이 TypeError는 **예외 필터 내부에서** 터진 것이라 또 다른 필터가 잡아주지 못하고, async 경로를 타고 **unhandled rejection**으로 빠져나감
8. Node 프로세스는 unhandled rejection 정책에 따라 `exit 1`

즉 "예외를 처리하려던 필터가, 처리 도중에 자기가 또 예외를 내고, 그 2차 예외를 아무도 못 잡아서 프로세스가 죽는" 고전적인 자기참조 장애였습니다.

### 코드는 이미 try-catch로 감싸져 있었다

문제의 필터에는 이미 이런 코드가 있었습니다.

```ts
// TODO: graphql 요청의 경우는 아래의 getRequestUrl과 httpAdapter.reply 함수에서
// 에러가 나는데, 이를 완전히 처리하는 것이 필요함
// 지금은 일단 catch문으로 잡아주기만 해도 응답엔 문제가 없어서, 이대로 두고 나중에 개선하기로 함
try {
  const responseBody = {
    // ...
    path: httpAdapter.getRequestUrl(ctx.getRequest()),
  };
  httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
} catch (e) {
  console.error(e);
}
```

코멘트만 보면 이미 손질된 것처럼 보입니다. 하지만 프로덕션 task definition `***-backend-prod:30`이 **이 try-catch가 무력화되는 경로**로 들어가면서 죽고 있는 게 확인됐습니다. stacktrace에 `AllExceptionsFilter.catch (...exception-filters/index.js:32:35)`가 그대로 노출돼 있었기 때문에, 해당 try-catch 블록이 실제 빌드 산출물에서 적용되어 있는지부터 검증이 필요했습니다.

가능한 원인 가설 두 가지였습니다.

- **(A)** 배포된 이미지가 try-catch가 들어가기 전 버전 (빌드 산출물이 소스와 어긋남)
- **(B)** try-catch는 들어가 있지만, **TypeError가 try 블록 바깥**(예: `responseBody` 생성 라인 자체나 그 위)에서 던져지고 있음

빌드된 `index.js:32` 라인을 직접 읽어보면 답이 나옵니다. 확인해보니 **(B)** 였습니다 — `getRequestUrl(ctx.getRequest())`이 실제 코드에서는 try의 더 바깥(예를 들어 로깅/메트릭 라인)에서 한 번 더 호출되고 있었고, 그쪽 호출이 try-catch 보호를 받지 못하고 있었습니다. "응답엔 문제 없어 보였던" 이유는 정상 트래픽(HTTP)에서는 ctx에 req가 항상 존재했기 때문입니다. **GraphQL subscription에서만** ctx의 형태가 달라서 폭발하는, 운영 중에야 드러나는 케이스였습니다.

### 진짜 고쳐야 할 방향

응급 처치는 try-catch 범위를 넓히는 것이지만, 본질적인 수정 방향은 아래 두 가지입니다.

1. **요청 컨텍스트 종류를 먼저 분기**한 뒤 그에 맞는 어댑터를 사용한다.

   ```ts
   if (host.getType<GqlContextType>() === 'graphql') {
     const gqlHost = GqlArgumentsHost.create(host);
     // subscription/query/mutation 모두 여기서 처리.
     // GraphQL 응답은 throw 또는 GraphQLError로 표현되므로,
     // httpAdapter.reply를 절대 호출하지 않는다.
     return this.handleGraphqlException(exception, gqlHost);
   }
   // 여기 아래로는 진짜 HTTP 요청만 도달함
   const ctx = host.switchToHttp();
   const req = ctx.getRequest();
   const res = ctx.getResponse();
   // ...
   ```

   요점은 **컨텍스트 타입을 먼저 좁힌 뒤** Express adapter를 쓰는 것입니다. 그래야 subscription/HTTP/RPC가 같은 필터를 통과해도 각자 안전하게 처리됩니다.

2. **subscription에서는 `httpAdapter.reply` 자체를 부르지 않는다.** WebSocket 연결로 흐르는 응답은 HTTP 응답이 아니므로 `reply`를 호출할 객체 자체가 존재하지 않습니다. GraphQL subscription에서의 인증 실패는 `GraphQLError` 또는 `UnauthorizedException`을 그대로 throw해서 graphql-ws 레이어가 close frame을 보내도록 위임하면 됩니다.

이 두 가지를 적용해야 같은 패턴이 다시 일어나도 프로세스가 죽지 않습니다.

---

## 3. 곁가지로 같이 잡힌 문제: 빌드에 누락된 `.html` 자산 (ENOENT)

원인을 파면서 같은 컨테이너의 다른 시점 로그도 같이 훑었는데, 별도의 에러도 한 줄 박혀 있었습니다.

```
Error: ENOENT: no such file or directory,
open '/usr/src/app/apps/backend/src/modules/***_printer/style.html'
```

(*모듈명은 마스킹*)

### 핵심 원인

문제의 코드는 대략 이렇습니다.

```ts
// apps/backend/src/modules/***_printer/***_printer.service.ts:115
const stylePath = path.join(
  process.cwd(),
  '/src/modules/***_printer/style.html',
);
```

이 코드에는 두 개의 약점이 겹쳐 있습니다.

1. **`src/` 경로를 런타임에 직접 읽고 있다.** 프로덕션/Docker 이미지에는 `src/`가 포함되지 않거나, `.html` 같은 비-TS 자산이 빌드 산출물(`dist/`)로 복사되지 않습니다.
2. **`process.cwd()` 기반 경로**라 컨테이너의 실행 디렉터리가 바뀌면 즉시 깨집니다.

에러 경로(`/usr/src/app/apps/backend/src/modules/...`)에서 알 수 있는 것은, 컨테이너의 `process.cwd()`가 `/usr/src/app/apps/backend`라는 것입니다. 즉 경로 조합 자체는 의도대로 만들어졌지만, **만들어진 경로에 실제 파일이 없는** 상태입니다.

확인해보니 `apps/backend/nest-cli.json`에는 `compilerOptions.assets` 설정이 없어서, `nest build` 시 `.html` 파일이 빌드 산출물로 복사되지 않고 있었습니다. 로컬 개발(`yarn start:dev`)에서는 `src/`가 그대로 존재하므로 멀쩡하게 보였고, 프로덕션 이미지에서만 깨지는 전형적인 "dev에서는 잘 됨" 케이스였습니다.

### 권장 수정 방향 (승인 후 진행)

1. **nest-cli.json에 자산 복사 규칙을 명시**한다.

   ```json
   {
     "collection": "@nestjs/schematics",
     "sourceRoot": "src",
     "compilerOptions": {
       "assets": [
         { "include": "**/*.html", "outDir": "build/dist" }
       ],
       "watchAssets": true
     }
   }
   ```

2. **경로 기준을 `process.cwd()` → `__dirname`으로 변경**한다.

   ```ts
   const stylePath = path.join(__dirname, 'style.html');
   ```

   `__dirname`은 컴파일된 `.js`가 위치한 디렉터리를 가리키므로, assets 복사 규칙으로 같은 디렉터리에 들어온 `style.html`을 안전하게 가리킵니다. 컨테이너 워킹 디렉터리에 의존하지 않게 됩니다.

이 두 가지가 함께 가야 합니다. 자산을 복사해두지 않으면 `__dirname` 기반으로 바꿔도 여전히 ENOENT이고, 자산만 복사하고 `process.cwd()`를 그대로 두면 워킹 디렉터리가 바뀌는 환경에서 다시 깨집니다.

---

## 4. 두 사건의 공통 교훈

이번 두 건은 직접적인 코드 위치는 다르지만, 메시지의 결은 같습니다.

1. **"호출 컨텍스트가 항상 같다"는 가정이 가장 자주 깨진다.**
   - AllExceptionsFilter는 HTTP만 들어오는 게 아니다. GraphQL query/mutation, subscription, WebSocket, RPC가 모두 같은 필터를 통과한다. `ctx.getRequest()`가 Express req를 돌려준다는 보장은 어디에도 없다.
   - 자산 경로도 마찬가지. `process.cwd()`가 항상 프로젝트 루트라는 가정은 dev 환경에서만 우연히 맞는다.

2. **try-catch는 "예외를 잡는다"가 아니라 "어떤 라인까지를 보호하는가"의 문제다.**
   - 코멘트로 "일단 catch로 잡아두자"고 적힌 코드가 실제 운영에서는 **try 블록 바깥에서** 같은 함수가 한 번 더 호출되고 있었다. 보호 범위를 코드로 정확히 그려보지 않으면 안전망은 종이 상자다.

3. **예외 필터 내부에서 또 예외가 나면 프로세스가 죽는다.** 글로벌 예외 필터는 "마지막 방어선"이므로, 그 안에서 발생할 수 있는 예외를 가장 먼저, 가장 보수적으로 막아야 한다. 어떤 입력이 와도 throw하지 않는 코드여야 한다.

4. **반복 crash 패턴 자체가 진단 정보다.** 같은 stacktrace로 N번 죽었다면, 트리거가 트래픽 어딘가에서 자동으로 재생산되고 있다는 뜻이다. 이번처럼 만료/실패한 토큰을 재시도하는 클라이언트가 한 명 있으면, 그 한 명이 백엔드를 무한히 죽일 수 있다. 인증 실패 → subscription 종료까지의 경로가 **항상 안전하게 닫혀야** 한다.

---

## 5. 다음 액션

| 항목 | 우선순위 | 방향 |
|------|----------|------|
| AllExceptionsFilter에서 GraphQL 컨텍스트 분기 처리 | 높음(긴급) | `host.getType()`으로 분기, subscription에서는 `httpAdapter.reply` 미호출 |
| `getRequestUrl` 호출 전체를 try-catch로 보호 | 높음(응급) | 위 영구 수정이 머지되기 전 임시 방어선 |
| nest-cli.json에 `.html` assets 복사 추가 | 중간 | 빌드 산출물에 정적 자산 포함 |
| `***_printer` 모듈의 `process.cwd()` 경로를 `__dirname` 기반으로 교체 | 중간 | 워킹 디렉터리 의존 제거 |
| 동일 클라이언트가 만료 토큰으로 subscription을 무한 재시도하는지 확인 | 중간 | 트리거 측 차단 / 백오프 |

응급 패치(글로벌 try-catch 보강)는 오늘 안에 PR로 올리고, 컨텍스트 분기 처리와 자산 복사 설정은 별도 PR로 분리해서 리뷰받기로 했습니다. 다음 글에서는 실제 PR 변경 내용과, 동일 패턴 재발 방지를 위해 추가한 통합 테스트(특히 subscription 컨텍스트에서 예외 필터가 throw하지 않음을 검증하는 테스트)를 정리할 예정입니다.
