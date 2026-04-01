---
title: "[인턴] path-filter로 인한 빌드 스킵 버그와 Docker 캐시 중심 파이프라인으로 재설계"
excerpt: "path-filter 기반 조건부 빌드의 치명적 약점을 발견하고, Docker 레이어 캐시를 단일 진실 공급원으로 삼아 파이프라인을 전면 재설계한 과정"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, AWS, ECR, GitHub Actions, CI/CD, Docker, BuildKit, 캐시]

permalink: /intern/company/blitz-dynamics/18/

toc: true
toc_sticky: true

date: 2026-04-01
last_modified_at: 2026-04-01
---

# path-filter로 인한 빌드 스킵 버그와 Docker 캐시 중심 파이프라인으로 재설계

## 들어가며

[이전 글](/intern/company/blitz-dynamics/16/)에서 `dorny/paths-filter`를 활용해 변경된 서비스만 선택적으로 빌드하는 파이프라인을 구성했다.

[그 다음 글](/intern/company/blitz-dynamics/17/)에서는 `build-common` job에 ECR 레지스트리 기반 Docker 레이어 캐시를 적용해 빌드 시간을 11분에서 3~4분으로 줄였다.

두 가지를 조합하면 꽤 괜찮은 파이프라인이 완성된 것처럼 보였다.

그런데 실제로 운영하다 보니, 생각보다 심각한 문제가 있다는 걸 발견했다.

**path-filter가 `false`를 반환해서 `build-common`이 스킵됐는데, 실제로는 반드시 재빌드했어야 하는 경우**가 있었다.

이번 글은 그 문제의 원인을 분석하고, 파이프라인을 근본적으로 재설계한 과정을 정리한 기록이다.

---

## 1부: 무엇이 문제였나

### path-filter의 동작 방식

`dorny/paths-filter`는 현재 push의 git diff를 분석해서, 지정한 경로에 변경이 있었는지 `true`/`false`로 반환한다.

이전 파이프라인에서 `build_common` 필터는 이렇게 설정되어 있었다.

```yaml
filters: |
  build_common:
    - 'Dockerfile'
    - 'package.json'
    - 'yarn.lock'
    - 'packages/**'
    - 'apps/*/package.json'
```

`build-common` job은 이 필터가 `true`일 때만 실제 빌드를 수행하고, `false`이면 ECR에 있는 기존 이미지를 그대로 사용했다.

```yaml
build-common:
  steps:
    - name: Check if rebuild is needed
      id: check
      run: |
        if [ "$CHANGED" == "true" ] || [ "$IMAGE_EXISTS" == "false" ]; then
          echo "needed=true" >> $GITHUB_OUTPUT
        else
          echo "needed=false" >> $GITHUB_OUTPUT
        fi

    - name: Build & push build-common
      if: steps.check.outputs.needed == 'true'
      run: docker build ...
```

직관적으로는 합리적인 설계처럼 보인다.

### 실제로 발생한 버그

문제는 **필터가 커버하지 못하는 변경이 존재한다**는 것이다.

대표적인 케이스 두 가지:

**케이스 1: 워크플로우 파일 자체의 변경**

`.github/workflows/build.yml`을 수정해서 빌드 인자나 환경변수를 바꿨다고 가정하자.

```yaml
- name: Build & push build-common
  run: |
    docker buildx build \
      --build-arg NEW_ENV=value \  # ← 새로 추가
      ...
```

이 경우 `Dockerfile`, `package.json`, `yarn.lock`, `packages/**` 중 어느 것도 변경되지 않았다. path-filter는 `build_common: false`를 반환한다.

결과: **build-common이 스킵되고, 새 빌드 인자가 반영되지 않은 구버전 이미지가 ECR에 남아있다.**

**케이스 2: 필터 경로에 누락된 의존성**

프로젝트 구조가 커지면서 새로운 공통 설정 파일이 생겼다고 가정하자.

```
config/
  tsconfig.base.json   # ← 새로 추가된 공통 설정
  ...
```

이 파일은 `build_common` 필터의 어느 경로에도 포함되지 않는다. `config/tsconfig.base.json`을 수정해도 path-filter는 여전히 `false`를 반환한다.

Dockerfile에서 이 파일을 COPY해서 사용하고 있다면?

```dockerfile
COPY config/tsconfig.base.json ./config/   # 변경됐지만 캐시는 미스
```

**결과: 실제로는 재빌드가 필요한데 build-common이 스킵된다.**

### 왜 이게 치명적인가

단순히 "최신 상태가 아닌 이미지를 사용한다"는 것이 왜 치명적인지 설명할 필요가 있다.

`build-common:latest`는 앱 빌드들의 베이스 이미지다.

```
build-common:latest  →  build-backend  (FROM build-common:latest)
                    →  build-frontend (FROM build-common:latest)
                    →  build-ceo-proxy, ...
```

build-common이 구버전 상태로 ECR에 남아있으면, 이후 앱 빌드들이 모두 **구버전 베이스 이미지 위에 새 코드를 얹는다.**

새 환경변수가 반영되지 않은 채로, 또는 수정된 공통 설정이 없는 채로 이미지가 만들어진다.

```
[기대하는 상태]
build-common v2 (새 설정 반영) → build-backend v5 (정상)

[실제 발생하는 상태]
build-common v1 (구버전) → build-backend v5 (버그 포함)
```

개발자는 코드를 push했고, 파이프라인도 성공으로 표시됐다. 하지만 실제로 배포된 이미지는 의도한 상태가 아니다.

---

## 2부: 근본 원인 분석

### path-filter의 책임 범위

돌이켜보면, path-filter에 잘못된 책임을 맡겼던 것이 문제였다.

path-filter가 실제로 하는 일: **git diff 기반으로 파일 변경 여부를 감지한다.**

path-filter에 맡겼던 일: **Docker 이미지를 재빌드해야 하는지 판단한다.**

이 두 가지는 다르다. Docker 레이어 캐시 무효화는 파일 변경만으로 결정되지 않는다. 빌드 인자, 환경변수, Dockerfile의 변경, 베이스 이미지의 변경 등 다양한 요소가 영향을 준다.

그리고 가장 중요한 사실: **Docker 자체가 이미 캐시 무효화를 처리하는 메커니즘을 내장하고 있다.**

```dockerfile
COPY package.json yarn.lock ./   # ← 이 파일이 바뀌면 아래 레이어는 자동으로 캐시 미스
RUN yarn install                 # ← Docker가 자동으로 재실행
COPY . .                         # ← Docker가 자동으로 재실행
```

path-filter로 "빌드할지 말지"를 결정하는 건 Docker의 캐시 메커니즘과 충돌하는 구조다.

Docker에게 "변경이 없으니 빌드하지 마"라고 하는 건 결국 Docker보다 덜 정확한 판단을 Docker 앞에 추가하는 셈이다.

---

## 3부: 재설계 — Docker 캐시를 단일 진실 공급원으로

### 핵심 아이디어

해결 방향은 단순하다.

> **항상 빌드하되, Docker 레이어 캐시가 빠르게 처리하도록 한다.**

path-filter가 빌드 여부를 결정하는 게 아니라, Docker 자체의 캐시 메커니즘이 "무엇을 재실행할지"를 결정한다.

변경이 없으면? Docker 캐시 히트 → 빠르게 완료.  
변경이 있으면? 해당 레이어부터 Docker가 재실행 → 정확하게 반영.

이 방식에서는 Docker Dockerfile이 캐시 무효화의 **단일 진실 공급원 (Single Source of Truth)** 이 된다.

### build-common: 항상 실행

이전 파이프라인에서 `build-common`은 `detect-changes` job의 결과에 따라 조건부로 빌드했다.

새 파이프라인에서는 `detect-changes` job 자체가 없다.

```yaml
# 이전
build-common:
  needs: detect-changes
  steps:
    - name: Check if rebuild is needed
      # ... 복잡한 로직
    - name: Build & push
      if: steps.check.outputs.needed == 'true'
      run: docker buildx build ...

# 이후
build-common:
  runs-on: ubuntu-latest
  steps:
    - name: Build & push build-common
      run: |
        docker buildx build \
          --cache-from type=registry,ref=$ECR_REGISTRY/blitz-build-common:cache \
          --cache-to   type=registry,ref=$ECR_REGISTRY/blitz-build-common:cache,mode=max \
          -t $ECR_REGISTRY/blitz-build-common:latest \
          -t $ECR_REGISTRY/blitz-build-common:$IMAGE_TAG \
          --push \
          .
```

항상 빌드하지만, ECR에 저장된 캐시 덕분에 실질적인 변경이 없을 때는 빠르게 완료된다.

```yaml
# build-common job 주석
# paths-filter 없이 항상 빌드 — Dockerfile이 캐시 무효화의 단일 진실 공급원
# yarn install 레이어: package.json/yarn.lock 변경 시에만 캐시 miss (가장 무거운 레이어)
# COPY ./ ./ 레이어: 소스 변경 시 캐시 miss (단순 복사라 빠름)
```

### 앱 빌드: 모든 이미지에 캐시 적용

이전 파이프라인에서는 `build-common`에만 ECR 캐시를 적용했다.
([17번 글](/intern/company/blitz-dynamics/17/)에서 작업한 내용)

앱별 빌드(backend, frontend 등)는 `docker build`만 사용했고 캐시가 없었다.

새 파이프라인에서는 **모든 이미지**에 ECR 레지스트리 캐시를 적용했다.

```yaml
# 이전: 앱 빌드에 캐시 없음
- name: Build & push backend
  run: |
    docker build \
      -f apps/backend/Dockerfile \
      -t $ECR_REGISTRY/blitz-backend:latest \
      --push \
      .

# 이후: 앱 빌드에도 캐시 적용
- name: Build & push backend
  run: |
    docker buildx build \
      --build-context build-common=docker-image://$ECR_REGISTRY/blitz-build-common:latest \
      --cache-from type=registry,ref=$ECR_REGISTRY/blitz-backend:cache \
      --cache-to   type=registry,ref=$ECR_REGISTRY/blitz-backend:cache,mode=max \
      -f apps/backend/Dockerfile \
      -t $ECR_REGISTRY/blitz-backend:latest \
      -t $ECR_REGISTRY/blitz-backend:$IMAGE_TAG \
      --push \
      .
```

여기서 주목할 옵션이 하나 있다.

**`--build-context build-common=docker-image://...`**

이전에는 앱 빌드 전에 `docker pull`로 build-common 이미지를 받아서 로컬에 태그했다.

```bash
# 이전 방식
docker pull $ECR_REGISTRY/blitz-build-common:latest
docker tag $ECR_REGISTRY/blitz-build-common:latest build-common:latest
```

새 방식은 `--build-context`로 직접 참조한다. pull 없이 BuildKit이 필요한 레이어만 가져온다.

### 앱 빌드도 항상 실행

이전에는 각 앱 빌드 job에 `if:` 조건이 붙어 있었다.

```yaml
# 이전
build-backend:
  needs: [detect-changes, build-common]
  if: |
    needs.build-common.result == 'success' &&
    (needs.detect-changes.outputs.backend == 'true' ||
     needs.detect-changes.outputs.build_common == 'true')
```

새 파이프라인에서는 `if:`가 없다.

```yaml
# 이후
build-backend:
  needs: build-common
  runs-on: ubuntu-latest
```

`build-common`이 완료되면 모든 앱 빌드가 항상 실행된다. 하지만 각 앱의 Dockerfile 내용이 바뀌지 않았다면 캐시 히트로 빠르게 끝난다.

---

## 4부: 실행 흐름 비교

### 이전 파이프라인

```
push 발생
  │
  ├─ detect-changes
  │    backend: true / false
  │    build_common: true / false
  │
  ├─ build-common
  │    build_common=false + ECR에 이미지 있음 → 빌드 스킵
  │    → 구버전 이미지가 그대로 남아있을 수 있음 ⚠️
  │
  ├─ build-backend   ← backend=true일 때만 실행
  ├─ build-frontend  ← frontend=true일 때만 실행
  └─ ...
```

### 새 파이프라인

```
push 발생
  │
  ├─ build-common (항상 실행)
  │    ECR 캐시 참조
  │    변경 없음 → Docker 캐시 히트 → 빠르게 완료 ✅
  │    변경 있음 → 해당 레이어 재실행 → 정확하게 반영 ✅
  │
  ├─ build-backend   (항상 실행, 병렬)
  │    ECR 캐시 참조
  │    변경 없음 → 캐시 히트
  ├─ build-frontend  (항상 실행, 병렬)
  ├─ build-ceo-proxy (항상 실행, 병렬)
  └─ ...
```

"변경됐는데 스킵되는" 케이스가 구조적으로 불가능해진다.

---

## 5부: 부수적 개선 — YAML 앵커로 중복 제거

각 job마다 반복되는 공통 step들이 있었다.

```yaml
# build-backend, build-frontend, build-ceo-proxy ... 모두 동일
- name: Configure AWS credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::075088103317:role/github-actions-ecr
    aws-region: ${{ env.AWS_REGION }}
```

YAML 앵커(`&`)와 별칭(`*`)을 활용해서 중복을 제거했다.

```yaml
jobs:
  build-common:
    steps:
      - &step-aws-auth          # ← 앵커 정의
        name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::075088103317:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}

      - &step-ecr-login         # ← 앵커 정의
        name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - &step-setup-buildx      # ← 앵커 정의
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - &step-set-image-tag     # ← 앵커 정의
        name: Set image tag
        run: echo "IMAGE_TAG=$(date +%Y%m%d).${{ github.run_number }}-${GITHUB_SHA:0:7}" >> $GITHUB_ENV

  build-backend:
    steps:
      - *step-aws-auth          # ← 별칭으로 재사용
      - *step-ecr-login
      - *step-setup-buildx
      - *step-set-image-tag
      - name: Build & push backend
        run: ...
```

앵커는 처음 정의되는 곳(`build-common`)에서 선언하고, 이후 job들에서 별칭으로 참조한다.

GitHub Actions가 YAML을 파싱할 때 별칭이 자동으로 원본으로 치환된다.

코드 중복은 줄었지만, 각 job이 실제로 어떤 step을 실행하는지는 달라지지 않는다.

---

## 6부: 이미지 태그 전략 변경

이전에는 이미지 태그를 `$(date +%Y%m%d).${{ github.run_number }}` 형태로 생성했다.

```
20260401.42
```

새 파이프라인에서는 Git SHA를 포함했다.

```yaml
echo "IMAGE_TAG=$(date +%Y%m%d).${{ github.run_number }}-${GITHUB_SHA:0:7}" >> $GITHUB_ENV
```

```
20260401.42-a3f9c1d
```

날짜와 실행 번호만으로는 "어느 커밋에서 만들어진 이미지인지"를 추적하려면 GitHub Actions 로그를 직접 봐야 했다.

SHA를 포함하면 이미지 태그만으로 커밋을 바로 특정할 수 있다.

---

## 결과

파이프라인이 재설계된 후 달라진 점을 정리하면:

| 항목 | 이전 | 이후 |
|------|------|------|
| 빌드 여부 결정 | path-filter (git diff 기반) | Docker 레이어 캐시 (Dockerfile 기반) |
| build-common 빌드 | 조건부 (필터 결과에 따라) | 항상 실행 |
| 앱 빌드 | 변경된 서비스만 | 항상 실행 (병렬) |
| 앱 이미지 캐시 | 없음 | ECR 레지스트리 캐시 |
| build-common 참조 | docker pull + tag | --build-context |
| 이미지 태그 | 날짜.번호 | 날짜.번호-SHA |
| 잘못된 빌드 스킵 | 발생 가능 | 구조적으로 불가능 |

---

## 마치며

"항상 빌드하면 느리지 않나?"라는 의문이 처음에 있었다.

그런데 실제로 실행해보니, 변경이 없는 이미지는 Docker 레이어 캐시로 1~2분 안에 끝난다.

이전에 path-filter로 빌드를 스킵했을 때와 실제 소요 시간 차이가 크지 않다. 스킵은 0초지만, 캐시 히트도 충분히 빠르다.

반면 얻는 것은 분명하다. "이 빌드에 내 변경이 반영됐나?"를 의심하지 않아도 된다.

path-filter 방식이 문제없이 동작하는 경우가 대부분이다. 하지만 잘못됐을 때는 조용히 잘못된다. 파이프라인은 성공으로 표시되고, 이미지도 ECR에 올라가지만, 실제 변경은 반영되지 않은 채로.

Docker 캐시를 단일 진실 공급원으로 삼는 방식은 더 단순하고, 더 정확하다.

복잡한 조건부 로직보다, 도구가 원래 잘하는 일을 맡기는 게 낫다는 걸 이번에 다시 확인했다.
