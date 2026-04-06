---
title: "[인턴] AWS ECS Preview 환경 구축 — 트러블슈팅 기록"
excerpt: "NestJS 백엔드를 ECS Fargate에 브랜치별로 자동 배포하는 Preview 환경을 구축하면서 마주친 다섯 가지 문제와 해결 과정"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, AWS, ECS, Fargate, Aurora, ElastiCache, NestJS, 트러블슈팅]

permalink: /intern/company/blitz-dynamics/19/

toc: true
toc_sticky: true

date: 2026-04-06
last_modified_at: 2026-04-06
---

# AWS ECS Preview 환경 구축 — 트러블슈팅 기록

## 들어가며

NestJS 백엔드를 GitHub Actions + ECS Fargate로 브랜치별 Preview 환경에 자동 배포하는 파이프라인을 구축했다.

개념 자체는 단순하다. PR이 열리면 해당 브랜치를 전용 ECS 태스크로 띄우고, ALB 호스트 기반 라우팅으로 `{branch-slug}.api.preview.example.com` 형태의 URL을 제공한다. PR이 닫히면 태스크를 정리한다.

막상 구현해 보니, 단계마다 예상치 못한 문제들이 쌓였다. 각각은 작은 설정 하나였지만, 원인을 찾는 데 상당한 시간이 들었다. 이번 글은 그 과정을 증상 → 원인 → 해결 순서로 정리한 기록이다.

---

## 인프라 구성 개요

본론에 앞서 Preview 환경의 전체 구성을 간략히 소개한다.

```
GitHub PR 오픈
  │
  ├─ GitHub Actions
  │    ├─ Docker 이미지 빌드 & ECR 푸시
  │    ├─ Aurora DB: CoW Clone으로 브랜치 전용 격리 DB 생성
  │    ├─ ECS Task Definition 등록 (브랜치별 환경변수 주입)
  │    └─ ECS Service 생성 및 ALB 타깃 등록
  │
  └─ 접근: {branch-slug}.api.preview.example.com
```

- **ECS**: Fargate, 브랜치당 독립 Service
- **DB**: Aurora — CoW(Copy-on-Write) Clone으로 브랜치별 격리
- **Cache**: ElastiCache Redis — Preview 환경 전체 공유
- **Secrets**: Secrets Manager — `backend-preview/{branch-slug}` 경로로 브랜치별 시크릿 저장

---

## 1. Aurora Security Group — ETIMEDOUT (DB 연결 불가)

### 증상

ECS 태스크가 기동되자마자 DB 연결 시도 시 `ETIMEDOUT` 에러 발생. 앱 로그에는 이렇게 찍혔다.

```
Error: connect ETIMEDOUT {DB_HOST}:3306
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:1494:16)
```

### 원인

ECS Fargate 태스크는 자체 Security Group을 가진다. Aurora 클러스터의 Security Group 인바운드 규칙에 이 ECS Security Group이 포함되어 있지 않아 TCP 3306 포트가 차단된 상태였다.

네트워크 레벨의 문제라서 앱 코드나 환경변수와 무관하다. 호스트명은 DNS로 정상 해석되지만, TCP 연결 자체가 막혀있다.

```
[증상 흐름]
DNS 해석 → ✅ 성공 (호스트명 → IP)
TCP 연결 → ❌ 차단 (Security Group)
```

### 해결

Aurora Security Group 인바운드 규칙에 아래 허용 규칙을 추가했다.

```
ECS Security Group → Aurora Security Group : TCP 3306 허용
```

> **패턴**: "DNS는 된다, TCP가 안 된다" → Security Group 인바운드 규칙을 먼저 확인할 것.

---

## 2. `.env` 파일이 ECS 환경변수를 덮어쓰는 문제

### 증상

Task Definition에 주입한 환경변수(`DB_PASSWORD` 등)가 무시되고, 로컬 개발 설정값이 사용됨. DB 연결이 엉뚱한 호스트로 향하거나 인증 실패가 발생했다.

### 원인

`src/environments/index.ts`에서 `dotenv.config({ path: '.env.local', override: true })`를 호출하고 있었다.

```typescript
// src/environments/index.ts
dotenv.config({ path: '.env.local', override: true });
```

`override: true` 옵션은 이미 설정된 환경변수도 파일의 값으로 덮어쓴다.

Docker 이미지 빌드 시 `.env.*` 파일이 포함되면, ECS에서 Task Definition으로 주입한 환경변수가 이미지 내 파일 값으로 덮어씌워진다.

```
[우선순위 (의도)]
Task Definition 환경변수 > .env 파일

[실제 동작]
.env.local (override: true) → Task Definition 환경변수를 무력화
```

### 해결

`.dockerignore`에 `.env*` 패턴을 추가해 이미지 빌드 시 `.env` 파일 자체가 포함되지 않도록 했다.

```
# .dockerignore
.env*
.env.local
.env.development
.env.production
```

이미지 안에 `.env` 파일이 없으면 `dotenv.config()`는 아무 동작도 하지 않고, Task Definition에서 주입한 환경변수만 남는다.

> **교훈**: `override: true`는 환경에 따라 의도와 정반대로 동작할 수 있다. Docker 이미지에는 `.env` 파일을 포함시키지 않는 것이 원칙이다.

---

## 3. AI 에이전트 모듈 초기화 크래시 — `permission denied for schema public`

### 증상

Preview 환경에서 앱 기동 시 전체 부트스트랩이 실패했다. 로그에는 아래와 같은 에러가 찍혔다.

```
ERROR [ExceptionHandler] permission denied for schema public
```

### 원인

일부 NestJS 모듈의 `onModuleInit()`에서 벡터스토어 초기화를 시도하고 있었다.

```typescript
async onModuleInit() {
  this.vectorStore = await VectorStore.initialize(embeddings, config);
  // 내부적으로 CREATE TABLE IF NOT EXISTS 실행
}
```

`VectorStore.initialize()` 내부에서 `CREATE TABLE`을 시도하는데, Preview 환경의 Analytics DB 계정은 **읽기 전용**이라 DDL 권한이 없었다.

NestJS의 `OnModuleInit`에서 처리되지 않은 예외가 발생하면 **전체 앱이 시작 불가 상태**가 된다. 그 결과 관련 없는 API 엔드포인트조차 전혀 응답하지 못하게 됐다.

```typescript
// 문제 코드 — 예외 전파 시 앱 전체 크래시
async onModuleInit() {
  this.vectorStore = await VectorStore.initialize(embeddings, config); // ← 여기서 크래시
}
```

### 해결

`onModuleInit()` 내부를 `try/catch`로 감싸 초기화 실패를 non-fatal로 처리했다.

```typescript
async onModuleInit() {
  try {
    this.vectorStore = await VectorStore.initialize(embeddings, config);
    this.logger.log('VectorStore initialized');
  } catch (err) {
    this.logger.warn(
      `onModuleInit failed (non-fatal): ${(err as Error).message}`
    );
    // vectorStore는 undefined 상태로 유지 — 해당 기능만 비활성화
  }
}
```

같은 패턴을 체크포인터(Checkpointer) 관련 provider의 `setup()` 호출에도 적용했다.

```typescript
// provider factory
{
  provide: CHECKPOINTER,
  useFactory: async (dataSource: DataSource) => {
    try {
      const saver = new PostgresSaver(dataSource);
      await saver.setup();
      return saver;
    } catch (err) {
      logger.warn(`Checkpointer setup failed (non-fatal): ${(err as Error).message}`);
      return null;
    }
  },
  inject: [DataSource],
}
```

이렇게 하면 벡터스토어와 체크포인터가 초기화되지 않더라도 앱은 정상 기동하고, 해당 기능을 사용하는 API만 선택적으로 비활성화된다.

> **교훈**: Preview 환경의 외부 의존성(DB, 벡터스토어 등)은 읽기 전용이거나 권한이 제한적일 수 있다. `onModuleInit`의 I/O 작업은 항상 실패 가능성을 고려해야 한다.

---

## 4. JWT 로그인 오류 — 두 단계 트러블슈팅

### 증상 (1차)

Preview 환경에서 로그인 시도 시 401 응답과 함께 아래 에러가 발생했다.

```
Error: secretOrPrivateKey must be a symmetric key when using HS256
```

### 원인 (1차)

Task Definition의 `secrets` 배열에서 JWT 시크릿으로 특정 AWS Secrets Manager 값을 참조하고 있었다.

문제는 그 시크릿이 **RSA 키 쌍**(비대칭키)을 담고 있다는 점이었다. 백엔드 코드는 `JwtModule.register({ secret: SECRET_JWT })`으로 **HS256(대칭키)** 알고리즘을 사용하도록 설정되어 있었다.

| 항목 | 내용 |
|------|------|
| 주입된 키 형식 | RSA Private Key (비대칭) |
| 백엔드 알고리즘 | HS256 (대칭, HMAC-SHA256) |
| 결과 | 알고리즘 불일치 → 서명 실패 |

Production 환경은 RS256을 사용하기 때문에 문제가 없었다. Preview 환경은 Production Task Definition을 복사해서 만들다 보니 이 불일치가 그대로 따라왔다.

### 1차 시도 (실패)

`secrets` 배열에서 `SECRET_JWT` 항목을 완전히 제거했다.

### 증상 (2차)

```
JwtStrategy requires a secret or key
TypeError: JwtStrategy requires a secret or key
    at new JwtStrategy (passport-jwt/lib/strategy.js:45:15)
```

앱 부트스트랩 자체가 실패했다.

### 원인 (2차)

`SECRET_JWT`를 제거하니 `process.env.SECRET_JWT`가 `undefined`가 됐다.

`passport-jwt`의 `Strategy` 생성자는 `secretOrKey`가 falsy이면 즉시 예외를 던진다.

```javascript
// passport-jwt/lib/strategy.js:45
if (!options.secretOrKey && !options.secretOrKeyProvider) {
  throw new TypeError('JwtStrategy requires a secret or key');
}
```

빈 문자열 `''`은 통과하지 못하고, `undefined`도 마찬가지다. NestJS는 `JwtModule`에 넘긴 `secret`을 `passport-jwt`의 `secretOrKey`로 그대로 전달하기 때문에, 환경변수가 없으면 Strategy 생성 단계에서 앱 전체가 죽는다.

### 최종 해결

`secrets` 배열에서 RSA 키를 제거하는 대신, Task Definition의 `environment` 배열에 더미 문자열을 직접 주입했다.

```json
{
  "name": "SECRET_JWT",
  "value": "preview-jwt-dummy-secret"
}
```

| 시도 | 값 | 결과 |
|------|-----|------|
| 1차 | RSA 키 (비대칭) | HS256 알고리즘 불일치 에러 |
| 2차 | 미주입 (`undefined`) | JwtStrategy 생성자 에러, 앱 기동 실패 |
| 최종 | 더미 문자열 | HS256 서명 성공 ✅ |

더미 문자열은 HS256으로 서명 가능하고, Strategy 생성자도 통과한다.

> **주의**: 더미 키로 서명된 JWT는 보안상 취약하다. Preview는 실 사용자 데이터와 무관한 테스트 환경이므로 허용하는 구성이다. Production에는 절대 적용하지 않는다.

---

## 5. Redis(ElastiCache) 연결 ETIMEDOUT

### 증상

앱 기동 시 Redis 연결이 `ETIMEDOUT`으로 실패했다.

```
Error: connect ETIMEDOUT {REDIS_HOST}:6379
```

### 원인

1번 Aurora 문제와 동일한 패턴이었다.

```
DNS 해석 → ✅ 성공
TCP 연결 → ❌ 차단 (Security Group)
```

ElastiCache Security Group의 인바운드 규칙에 ECS Security Group이 포함되어 있지 않아 TCP 6379 포트가 차단된 상태였다.

Aurora와 ElastiCache는 **별개의 Security Group**을 가지므로, Aurora 문제를 해결했어도 ElastiCache는 따로 설정해야 한다.

### 해결

ElastiCache Security Group 인바운드 규칙에 허용 규칙을 추가했다.

```
ECS Security Group → ElastiCache Security Group : TCP 6379 허용
```

Aurora(3306)와 ElastiCache(6379) — 같은 패턴, 다른 Security Group, 다른 포트.

---

## 공통 패턴 정리

다섯 가지 문제를 겪으면서 공통적인 패턴이 보였다.

| 증상 | 먼저 확인할 것 |
|------|---------------|
| `ETIMEDOUT` (DB, Redis 등) | Security Group 인바운드 규칙 (DNS ↔ TCP 구분) |
| 환경변수가 무시됨 | Docker 이미지 내 `.env` 파일 포함 여부 |
| 앱이 기동조차 안 됨 | `onModuleInit` 내 uncaught exception |
| 알고리즘 불일치 에러 | 시크릿 값의 형식 (대칭키 vs 비대칭키) — `secrets`에서 제거 후 `environment`에 더미 문자열 주입 |

### Security Group 트러블슈팅 체크리스트

ETIMEDOUT이 발생했을 때 확인 순서:

```
1. DNS 해석 성공 여부 확인
   nslookup {host} 또는 curl --resolve 테스트

2. 대상 리소스의 Security Group 인바운드 규칙 확인
   - ECS Task의 Security Group ID 확인
   - 대상(Aurora, ElastiCache 등) Security Group에 위 SG가 포함되어 있는지 확인
   - 포트 번호 일치 여부 확인 (MySQL: 3306, Redis: 6379)

3. 허용 규칙 추가
   ECS SG → 대상 SG : TCP {port} 허용
```

"DNS는 된다, TCP가 안 된다"는 Security Group 문제의 전형적인 시그니처다.

---

## 마치며

Preview 환경 구축 자체보다 운영 중에 발견한 문제들이 더 공부가 됐다.

문제 하나하나는 "그냥 설정 하나 빠진 것"이었지만, 원인을 찾기까지는 시간이 걸렸다. Security Group처럼 익숙한 개념도 여러 리소스가 얽히면 놓치기 쉬웠고, `.env` override 같은 경우는 로컬에서는 문제없이 동작했기 때문에 의심 자체를 늦게 했다.

결국 공통점은 하나다. **"환경이 달라지면, 로컬에서 당연하게 됐던 것들을 다시 의심해야 한다."**

Preview 환경은 Production과 다르다. DB 계정 권한이 제한적이고, 시크릿 구성도 다를 수 있고, Security Group도 별도로 설정해야 한다. 이 사실을 미리 체크리스트로 만들어두면 다음 번에는 훨씬 빠르게 해결할 수 있을 것이다.
