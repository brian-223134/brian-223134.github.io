---
title: "[인턴] Docker 레이어 캐시로 ECR 빌드 시간 11분 → 3분으로 줄이기"
excerpt: "GitHub Actions에서 ECR 레지스트리 기반 Docker BuildKit 캐시를 적용해 build-common 빌드 시간을 11분에서 3~4분으로 단축한 과정"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, AWS, ECR, GitHub Actions, CI/CD, Docker, BuildKit, 캐시]

permalink: /intern/company/blitz-dynamics/17/

toc: true
toc_sticky: true

date: 2026-04-01
last_modified_at: 2026-04-01
---

# Docker 레이어 캐시로 ECR 빌드 시간 11분 → 3분으로 줄이기

## 들어가며

[이전 글](/intern/company/blitz-dynamics/16/)에서 모노레포에서 변경된 서비스만 선택적으로 빌드하는 GitHub Actions 파이프라인을 구성한 내용을 정리했다.

파이프라인을 구성하고 나서 실제로 돌려보니, 한 가지 병목이 눈에 띄었다.

`build-common` job이 실행될 때마다 **11분 정도** 걸렸다.

이 job은 yarn install과 공통 패키지 빌드 등 무거운 작업을 처리하는 베이스 이미지를 만드는 단계다. 여기서 시간이 오래 걸리면 이후 모든 앱 빌드가 대기해야 하기 때문에 전체 파이프라인 시간에 직접적인 영향을 준다.

원인은 명확했다. **GitHub Actions runner는 매번 새 VM에서 시작하기 때문에, 이전 실행에서 만든 Docker 레이어가 남아 있지 않는다.**

Docker 빌드에서 레이어 캐시가 없으면 `RUN yarn install` 같은 명령이 항상 처음부터 실행된다. 의존성이 바뀌지 않았더라도.

이를 해결하기 위해 **ECR 레지스트리 기반 캐시**를 적용했고, 빌드 시간이 **3~4분으로 줄었다.**

이번 글은 그 과정을 정리한 기록이다.

---

## Docker 레이어 캐시가 왜 중요한가

Docker 이미지는 레이어 단위로 구성된다.

```dockerfile
FROM node:20-alpine         # Layer 1
WORKDIR /app                # Layer 2
COPY package.json yarn.lock # Layer 3
RUN yarn install            # Layer 4  ← 시간이 오래 걸림
COPY . .                    # Layer 5
RUN yarn build              # Layer 6
```

레이어 캐시의 핵심 규칙은 이렇다: **어떤 레이어가 변경되면, 그 이후 레이어는 모두 다시 실행된다.**

반대로 말하면, 레이어가 변경되지 않았다면 이전에 만들어 둔 레이어를 재사용할 수 있다.

로컬 개발 환경에서는 이 캐시가 자동으로 작동한다. `package.json`이 바뀌지 않았다면 Layer 3까지는 캐시에서 가져오고, Layer 4의 `yarn install`도 다시 실행하지 않는다.

문제는 CI 환경이다. GitHub Actions의 runner는 매 실행마다 새 VM이 할당되기 때문에 **로컬 캐시가 없다.** 이전 빌드에서 만든 레이어가 하나도 남아 있지 않으므로, 항상 Layer 1부터 끝까지 전부 다시 실행된다.

`yarn install`이 매번 처음부터 돌아가는 이유가 이것이다.

---

## 해결 방법: 레지스트리 기반 캐시

로컬 캐시를 쓸 수 없다면, **외부 저장소에 캐시를 저장하고 다음 빌드에서 가져오면 된다.**

Docker BuildKit은 이를 위한 `--cache-from`과 `--cache-to` 옵션을 제공한다.

캐시 저장소로는 여러 옵션이 있다.

| 방식 | 설명 |
|------|------|
| `type=local` | 로컬 디렉토리에 저장. CI에선 별도 캐시 복원 액션이 필요 |
| `type=registry` | Docker 레지스트리(ECR, Docker Hub 등)에 저장. 별도 설정 없이 바로 사용 가능 |
| `type=gha` | GitHub Actions 캐시 저장소 사용. 용량 제한(10GB)이 있음 |

이미 ECR을 사용하고 있는 상황에서 `type=registry`가 가장 자연스러운 선택이었다. ECR에 캐시 전용 태그를 하나 만들고, 빌드 시 그걸 참조하면 된다.

---

## 적용 방법

### docker buildx 사용

`--cache-from`과 `--cache-to` 옵션은 **BuildKit**을 사용하는 `docker buildx build`에서만 지원된다. 기존의 `docker build`로는 동작하지 않는다.

워크플로우에 Buildx 설정을 추가한다.

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```

`docker/setup-buildx-action`은 BuildKit 기반의 `buildx` 빌더를 설치하고 활성화하는 공식 액션이다.

### 빌드 명령 변경

기존 `docker build` 명령을 `docker buildx build`로 교체하고 캐시 옵션을 추가한다.

```bash
# 변경 전
docker build \
  -t $ECR_REGISTRY/my-build-common:latest \
  -t $ECR_REGISTRY/my-build-common:$IMAGE_TAG \
  .

# 변경 후
docker buildx build \
  --cache-from type=registry,ref=$ECR_REGISTRY/my-build-common:cache \
  --cache-to   type=registry,ref=$ECR_REGISTRY/my-build-common:cache,mode=max \
  -t $ECR_REGISTRY/my-build-common:latest \
  -t $ECR_REGISTRY/my-build-common:$IMAGE_TAG \
  --push \
  .
```

옵션 하나씩 살펴보면:

**`--cache-from type=registry,ref=.../my-build-common:cache`**
- 빌드 시작 전에 ECR에서 `:cache` 태그 이미지를 가져온다.
- 이 이미지에 저장된 레이어를 캐시로 사용한다.
- 처음 실행 시(캐시 이미지가 없을 때)는 그냥 무시하고 빌드가 진행된다.

**`--cache-to type=registry,ref=.../my-build-common:cache,mode=max`**
- 빌드 완료 후 레이어 캐시를 ECR에 저장한다.
- `mode=max`는 **중간 레이어를 포함한 모든 레이어**를 저장하는 옵션이다.

**`--push`**
- `docker buildx build`는 기본적으로 빌드된 이미지를 로컬에 저장하지 않는다.
- 레지스트리로 직접 푸시하려면 `--push`를 명시해야 한다.

### `mode=max` vs `mode=min`

`--cache-to`의 `mode` 옵션은 어느 범위의 레이어를 캐시에 저장할지를 결정한다.

```dockerfile
FROM node:20-alpine         # Layer 1
WORKDIR /app                # Layer 2
COPY package.json yarn.lock # Layer 3
RUN yarn install            # Layer 4  ← 핵심 캐시 대상
COPY . .                    # Layer 5
RUN yarn build              # Layer 6  ← 최종 레이어
```

| mode | 저장 범위 |
|------|----------|
| `mode=min` (기본값) | 최종 이미지에 포함된 레이어만 저장 |
| `mode=max` | 빌드 과정의 **모든 중간 레이어** 저장 |

`mode=min`은 저장 용량이 작지만, multi-stage build에서 최종 스테이지에 포함되지 않는 중간 레이어는 캐시에 저장되지 않는다. 다음 빌드에서 그 중간 레이어를 다시 실행해야 한다.

`mode=max`는 저장 용량이 크지만, 중간 레이어까지 전부 캐시하기 때문에 캐시 히트율이 더 높다.

`yarn install`처럼 오래 걸리는 중간 레이어가 있는 경우라면 `mode=max`가 적합하다.

---

## 캐시가 동작하는 흐름

**첫 번째 실행 (캐시 없음):**

```
빌드 시작
  │
  ├─ --cache-from: ECR에 :cache 태그 없음 → 캐시 없이 진행
  │
  ├─ Layer 1: FROM node:20-alpine    (실행)
  ├─ Layer 2: WORKDIR /app           (실행)
  ├─ Layer 3: COPY package.json ...  (실행)
  ├─ Layer 4: RUN yarn install       (실행) ← 오래 걸림
  ├─ Layer 5: COPY . .               (실행)
  └─ Layer 6: RUN yarn build         (실행)
  │
  └─ --cache-to: 모든 레이어를 ECR :cache 태그에 저장
```

**두 번째 실행 이후 (캐시 있음, package.json 변경 없음):**

```
빌드 시작
  │
  ├─ --cache-from: ECR에서 :cache 태그 이미지 가져오기
  │
  ├─ Layer 1: FROM node:20-alpine    → 캐시 히트 ✅
  ├─ Layer 2: WORKDIR /app           → 캐시 히트 ✅
  ├─ Layer 3: COPY package.json ...  → 캐시 히트 ✅ (내용 동일)
  ├─ Layer 4: RUN yarn install       → 캐시 히트 ✅  ← 여기가 핵심
  ├─ Layer 5: COPY . .               → 소스 변경 있으면 재실행
  └─ Layer 6: RUN yarn build         → 재실행
  │
  └─ --cache-to: 변경된 레이어만 업데이트해서 저장
```

`yarn install`이 캐시에서 바로 나오니까 시간이 크게 줄어든다.

---

## 적용 결과

| 구분 | 소요 시간 |
|------|----------|
| 캐시 적용 전 | 약 11분 |
| 캐시 적용 후 (첫 실행) | 약 11분 (캐시 없어서 동일) |
| 캐시 적용 후 (이후 실행) | 약 3~4분 |

캐시가 쌓인 이후에는 `build-common`이 3~4분 안에 완료된다. 약 70%의 시간이 단축됐다.

`package.json`이나 `yarn.lock`이 변경된 경우에는 Layer 3 이후가 캐시 미스가 나서 다시 오래 걸릴 수 있다. 하지만 의존성이 매번 바뀌지 않는 이상, 대부분의 빌드에서 캐시 히트를 기대할 수 있다.

---

## ECR에 캐시 태그가 생성됨

한 가지 주의할 점이 있다.

빌드 후 ECR 리포지토리를 보면 `:cache` 태그로 이미지가 하나 더 생성된다. 이 이미지는 실제 앱 이미지가 아니라 BuildKit 캐시 메타데이터가 저장된 이미지다.

ECR 비용은 저장 용량 기준이기 때문에, 캐시 이미지가 차지하는 용량도 비용에 포함된다. `mode=max`로 모든 레이어를 저장하면 이미지 크기가 꽤 커질 수 있다.

주기적으로 오래된 캐시 이미지를 정리하거나, ECR의 Lifecycle Policy를 설정해서 자동으로 관리하는 것이 좋다.

```json
{
  "rules": [{
    "rulePriority": 10,
    "description": "cache 태그는 5개만 유지",
    "selection": {
      "tagStatus": "tagged",
      "tagPrefixList": ["cache"],
      "countType": "imageCountMoreThan",
      "countNumber": 5
    },
    "action": { "type": "expire" }
  }]
}
```

---

## 마치며

GitHub Actions에서 Docker 빌드 시간을 줄이는 방법으로 레이어 캐시를 외부 레지스트리에 저장하는 방식을 써봤다.

핵심을 정리하면:

- GitHub Actions runner는 매번 새 VM이라서 로컬 Docker 캐시가 없다.
- `docker buildx build`의 `--cache-from`과 `--cache-to`로 ECR에 캐시를 저장하고 재사용할 수 있다.
- `mode=max`는 중간 레이어까지 저장하므로 `yarn install` 같은 무거운 레이어에 효과적이다.
- 단, 캐시 이미지가 ECR 저장 용량을 차지하므로 Lifecycle Policy로 관리가 필요하다.

"왜 매번 yarn install을 다시 하지?"라는 단순한 의문에서 시작해서, Docker 레이어 캐시의 동작 원리와 BuildKit 캐시 타입까지 자연스럽게 이해하게 된 작업이었다.
