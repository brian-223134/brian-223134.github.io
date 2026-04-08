---
title: "[인턴] Docker build-common 제거로 CI 빌드 시간 14분 → 4분으로 단축하기"
excerpt: "모노레포 공통 빌드 이미지의 COPY ./ ./가 캐시를 무력화하는 구조적 원인을 파악하고, 자립형 Dockerfile과 ECR 외부 캐시로 전환한 과정"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Docker, CI, 빌드최적화, GitHub Actions, ECR, 캐시]

permalink: /intern/company/blitz-dynamics/20/

toc: true
toc_sticky: true

date: 2026-04-08
last_modified_at: 2026-04-08
---

## 들어가며

CI 파이프라인에서 백엔드 이미지 빌드가 평균 14분 걸리고 있었다.

특히 작은 비즈니스 로직 수정 한 줄을 반영하는데도 전체 빌드가 처음부터 다시 돌아가는 상황이 반복됐다. 캐시가 거의 동작하지 않는다는 의미였다. 원인을 파악하고 나서 구조를 뜯어고쳤고, 결과적으로 빌드 시간을 약 4분 이내로 단축했다. 이번 글은 그 과정을 기록한다.

---

## 기존 구조: build-common 공유 이미지

기존 파이프라인은 **build-common**이라는 공유 Docker 이미지를 중심으로 구성되어 있었다.

```
build-common (공통 빌드 베이스)
  ├── 전체 소스코드 복사 (COPY ./ ./)
  ├── 의존성 설치 (yarn install)
  └── 전체 빌드 (tsc)

          ↓ FROM build-common AS base

app-backend       app-frontend      app-...
  (run stage)       (run stage)       (run stage)
```

**아이디어 자체는 합리적**이다. 모노레포에서 의존성 설치와 TypeScript 빌드는 모든 앱에 공통이다. 한 번 만들어두고 재사용하면 효율적이어야 한다.

문제는 이 구조가 Docker 레이어 캐시와 궁합이 매우 나쁘다는 점이었다.

---

## 문제: COPY ./ ./가 캐시를 무효화하는 구조

Docker는 레이어 단위로 캐시를 관리한다. 특정 레이어의 입력이 바뀌면 **그 이후 레이어는 전부 캐시 미스**가 된다.

build-common의 핵심 레이어는 아래와 같다.

```dockerfile
# build-common/Dockerfile
FROM node:20-slim AS builder

WORKDIR /app

COPY ./ ./          # ← 문제의 레이어
RUN yarn install
RUN yarn build
```

`COPY ./ ./`는 **모노레포 루트 전체를 복사**한다.

모노레포 특성상 루트 아래에는 수많은 앱과 공통 패키지가 있다. **어느 앱의 파일 하나라도 변경되면** 이 `COPY` 레이어의 체크섬이 달라지고, 이후 `yarn install`, `yarn build`까지 전부 캐시 미스가 된다.

```
[캐시 무효화 흐름]

apps/frontend/src/some-component.tsx 수정
        ↓
COPY ./ ./ 레이어 → 체크섬 변경 → 캐시 MISS
        ↓
yarn install → 캐시 MISS (수백 개 패키지 재설치)
        ↓
yarn build (tsc) → 캐시 MISS (전체 재컴파일)
        ↓
app-backend / app-frontend / ... 전부 재빌드
```

백엔드만 배포하는 상황인데, 프론트엔드 파일이 바뀌어도 백엔드가 처음부터 다시 빌드됐다. 반대도 마찬가지였다. 공통 이미지 하나가 모든 앱의 캐시를 동시에 날리는 구조였다.

### 캐시가 동작하는 상황

아이러니하게도 build-common 캐시가 유효한 경우는 **아무것도 바뀌지 않았을 때**뿐이다.

| 변경 사항 | COPY ./ ./ 캐시 | 결과 |
|-----------|----------------|------|
| 변경 없음 | HIT | 빠름 ✅ |
| 앱 A 소스 수정 | MISS | 전체 재빌드 ❌ |
| 앱 B 소스 수정 | MISS | 전체 재빌드 ❌ |
| 패키지 추가 | MISS | 전체 재빌드 ❌ |

실무에서 "아무것도 안 바뀐 채로" 배포하는 경우는 없다. build-common은 사실상 항상 캐시 미스였다.

---

## 해결: 자립형 Dockerfile로 전환

접근 방식을 바꿨다. **build-common을 제거하고, 각 앱이 독립적으로 빌드**되도록 전환했다.

```
[이전]
build-common → app-backend
             → app-frontend
             → ...

[이후]
app-backend (자립형 빌드)
app-frontend (자립형 빌드)
```

각 앱의 `Dockerfile.no-common`은 build-common에 의존하지 않고 자체적으로 의존성을 설치하고 빌드한다.

### COPY 범위를 앱 단위로 좁히기

가장 중요한 변경은 `COPY` 범위를 앱과 관련된 파일만으로 제한한 것이다.

```dockerfile
# Dockerfile.no-common (변경 후)
FROM node:20-slim AS builder

WORKDIR /app

# 패키지 파일만 먼저 복사 (캐시 최적화)
COPY package.json yarn.lock ./
COPY apps/backend/package.json apps/backend/
COPY packages/*/package.json packages/  # 의존하는 공통 패키지만

# 의존성 설치 (package.json이 바뀌지 않으면 캐시 HIT)
RUN yarn install --frozen-lockfile

# 소스코드 복사
COPY apps/backend/ apps/backend/
COPY packages/ packages/

# 빌드
RUN yarn workspace @app/backend build
```

`package.json` / `yarn.lock` 레이어를 소스코드와 분리했다. 의존성이 바뀌지 않는 한 `yarn install` 레이어는 캐시가 유지된다.

```
[변경 후 캐시 동작]

apps/frontend 파일 수정
        ↓
apps/backend Dockerfile: COPY 범위 밖 → 캐시 유지 ✅
apps/backend: yarn install → 캐시 HIT ✅
apps/backend: yarn build → 캐시 HIT ✅
```

---

## ECR 외부 캐시로 yarn install 최적화

자립형 전환만으로도 캐시 미스가 줄었지만, `yarn install`은 여전히 병목이었다. GitHub Actions 러너는 매번 새 환경에서 시작해 로컬 캐시가 없다.

이를 해결하기 위해 **ECR을 외부 캐시 저장소로 활용**했다.

```yaml
# GitHub Actions workflow
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    file: apps/backend/Dockerfile.no-common
    push: true
    tags: ${{ env.ECR_REGISTRY }}/app-backend:${{ env.IMAGE_TAG }}
    cache-from: type=registry,ref=${{ env.ECR_REGISTRY }}/app-backend:cache
    cache-to: type=registry,ref=${{ env.ECR_REGISTRY }}/app-backend:cache,mode=max
```

`mode=max`는 최종 이미지뿐 아니라 **빌더 스테이지의 모든 레이어**를 캐시에 저장한다. `yarn install` 레이어도 포함된다.

```
[ECR 캐시 동작]

1차 빌드: cache MISS → ECR에 레이어 저장 (cache-to)
2차 빌드 (소스만 변경): ECR에서 yarn install 레이어 HIT → 재설치 생략
```

빌더 스테이지와 실행 스테이지를 분리하는 멀티스테이지 빌드와 함께 사용하면 효과가 배가된다. 실행 이미지에는 `node_modules`와 빌드 산출물만 포함되므로 이미지 크기도 줄어든다.

```dockerfile
# Run stage
FROM node:20-slim AS runner

WORKDIR /app

# 빌더 스테이지에서 필요한 것만 복사
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/apps/backend/build ./build
# src/ 미포함 — 런타임에 불필요
```

---

## 결과

| 구분 | 이전 | 이후 |
|------|------|------|
| 빌드 방식 | build-common 공유 이미지 | 앱별 자립형 Dockerfile |
| 캐시 유효 조건 | 전체 소스 무변경 | 해당 앱/의존성 무변경 |
| 캐시 저장소 | (없음 / 로컬만) | ECR (mode=max) |
| 빌드 시간 (캐시 미스) | 약 14분 | 약 8분 |
| 빌드 시간 (캐시 HIT) | 약 14분 (사실상 항상 미스) | 약 3분 53초 |
| 앱 간 영향 | 한 앱 변경 → 전체 재빌드 | 변경된 앱만 재빌드 |

소스 수정 후 재빌드하는 일반적인 케이스(캐시 HIT 상황)에서 약 14분에서 약 3분 53초로 단축됐다.

---

## 정리

이번 작업의 핵심은 두 가지다.

**1. `COPY` 범위를 최소화하라**

Docker 캐시 무효화는 항상 `COPY`/`ADD` 레이어에서 시작된다. 모노레포에서 `COPY ./ ./`는 사실상 캐시 무효화 보장이다. `package.json` 먼저, 소스코드 나중 — 이 순서와 최소 범위 원칙을 지키면 캐시가 살아난다.

**2. 공유 베이스 이미지는 의존성 고정에만 써라**

"한 번 만들어두고 재사용"이라는 아이디어 자체는 틀리지 않지만, 가변적인 소스코드가 섞이는 순간 캐시 전략이 붕괴된다. 공유 이미지는 변경이 거의 없는 Node 버전, OS 설정, 글로벌 패키지 정도에만 쓰는 것이 맞다.

멀티스테이지 빌드 + 외부 캐시 레지스트리 조합은 CI 환경처럼 stateless한 곳에서 Docker 레이어 캐시를 유지하는 실용적인 방법이다. `mode=max`로 빌더 레이어까지 캐시에 올려두면, 소스 변경이 있어도 `yarn install`은 건너뛸 수 있다.
