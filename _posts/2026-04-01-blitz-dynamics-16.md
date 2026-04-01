---
title: "[인턴] GitHub Actions로 ECR 멀티 서비스 빌드 파이프라인 구성하기"
excerpt: "모노레포에서 변경된 서비스만 선택적으로 빌드하고 ECR에 푸시하는 CI/CD 파이프라인을 GitHub Actions로 구성한 과정 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, AWS, ECR, ECS, GitHub Actions, CI/CD, Docker]

permalink: /intern/company/blitz-dynamics/16/

toc: true
toc_sticky: true

date: 2026-04-01
last_modified_at: 2026-04-01
---

# GitHub Actions로 ECR 멀티 서비스 빌드 파이프라인 구성하기

## 들어가며

[이전 글](/intern/company/blitz-dynamics/15/)에서 EC2 + nginx 구조의 헬스체크 문제를 해결하기 위해 ECR + ECS로 마이그레이션했다는 내용을 정리했다.

그런데 ECS로 넘어오면서 자연스럽게 새로운 과제가 생겼다.

기존에는 배포를 할 때 EC2에 SSH로 들어가서 `git pull`하고 프로세스를 재시작했다.
ECS에서는 새 이미지를 ECR에 푸시해야 배포가 가능하다.

그리고 우리 서비스는 **모노레포**에 여러 앱이 함께 있는 구조였다.

```
apps/
  ├── backend/
  ├── frontend/
  ├── proxy-a/
  ├── proxy-b/
  └── web-browser/
packages/
  └── shared/  # 공통 패키지
```

문제는 이랬다.
- `frontend`만 수정했는데 `backend`까지 빌드해야 할까?
- 매번 전체를 빌드하면 시간이 너무 오래 걸린다.
- 그렇다고 수동으로 "이번엔 프론트만 빌드해"라고 관리하면 실수가 생긴다.

이 문제를 해결하기 위해 **변경된 서비스만 선택적으로 빌드**하는 GitHub Actions 파이프라인을 만들었다.
이번 글은 그 과정에서 고민했던 것들과 결과를 정리한 기록이다.

---

## 전체 파이프라인 구조

파이프라인은 크게 4단계로 구성된다.

```
Step 1: detect-changes    # 어떤 파일이 바뀌었는지 감지
Step 2: build-common      # 공통 베이스 이미지 빌드 (필요한 경우에만)
Step 3: build-*           # 각 앱 이미지 병렬 빌드 (변경된 것만)
Step 4: notify-slack      # 결과 Slack 알림
```

각 단계를 하나씩 살펴보자.

---

## Step 1: 변경 감지 (detect-changes)

핵심은 "어떤 서비스가 바뀌었는지"를 먼저 파악하는 것이다.

이를 위해 `dorny/paths-filter` 액션을 사용했다.

```yaml
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      build_common:
        - 'Dockerfile'
        - 'package.json'
        - 'yarn.lock'
        - 'packages/**'
        - 'apps/*/package.json'
      backend:
        - 'apps/backend/**'
      frontend:
        - 'apps/frontend/**'
```

이 액션은 현재 push에서 변경된 파일 목록을 분석해서, 각 필터에 해당하는 변경이 있었는지 `true`/`false`로 반환한다.

그리고 이 결과를 `outputs`으로 노출해서 이후 job들이 조건부로 실행될 수 있게 한다.

```yaml
outputs:
  build_common: ${{ steps.filter.outputs.build_common }}
  backend: ${{ steps.filter.outputs.backend }}
  frontend: ${{ steps.filter.outputs.frontend }}
  # ...
```

**`build_common` 필터가 하는 역할**이 중요하다.
`Dockerfile`, `package.json`, `yarn.lock`, 공통 패키지 등이 변경되면 사실상 모든 앱 이미지를 다시 빌드해야 한다.
이 경우를 별도로 감지해서 모든 앱 빌드를 트리거하는 신호로 사용한다.

---

## Step 2: 공통 베이스 이미지 (build-common)

이 단계가 이 파이프라인에서 가장 흥미로운 부분이었다.

### 왜 build-common이 필요한가?

우리 프로젝트 루트의 `Dockerfile`은 **공통 빌드 레이어**를 만드는 용도다.
`yarn install`처럼 시간이 오래 걸리는 작업을 미리 처리해서, 각 앱 빌드 시 이 레이어를 재사용할 수 있게 한다.

```
루트 Dockerfile (build-common) : yarn install + 공통 패키지 빌드
apps/backend/Dockerfile         : FROM build-common:latest → backend 빌드
apps/frontend/Dockerfile        : FROM build-common:latest → frontend 빌드
```

앱별 Dockerfile이 `FROM build-common:latest`를 베이스로 시작하면, Docker 레이어 캐시 덕분에 yarn install을 다시 실행하지 않아도 된다.

### "항상 실행, 하지만 빌드는 조건부"

`build-common` job은 **항상 실행**되지만, **실제 빌드 작업은 필요한 경우에만** 수행한다.

왜 항상 실행해야 하냐면, 이후 단계에서 `build-common:latest` 이미지가 ECR에 있는지 없는지를 먼저 확인해야 하기 때문이다.

```bash
ECR_OUTPUT=$(aws ecr describe-images \
    --repository-name my-build-common \
    --image-ids imageTag=latest 2>&1) && IMAGE_EXISTS=true || IMAGE_EXISTS=false

if [ "$CHANGED" == "true" ] || [ "$IMAGE_EXISTS" == "false" ]; then
  NEEDED=true
else
  NEEDED=false
fi

echo "needed=$NEEDED" >> $GITHUB_OUTPUT
```

- `build_common` 필터가 `true`이면 → 빌드 필요
- ECR에 이미지가 없으면 → 빌드 필요 (처음 실행하는 경우)
- 둘 다 아니면 → 기존 이미지를 그대로 사용

이 판단 결과를 `steps.check.outputs.needed`로 저장하고, 이후 step들이 이 값을 조건으로 사용한다.

```yaml
- name: Free disk space
  if: steps.check.outputs.needed == 'true'
  uses: jlumbroso/free-disk-space@v1.3.1

- name: Checkout repository
  if: steps.check.outputs.needed == 'true'
  uses: actions/checkout@v4
```

빌드가 필요 없을 때는 checkout조차 하지 않는다.
GitHub Actions의 runner는 디스크가 제한적이기 때문에, 불필요한 작업을 줄이는 것이 의외로 중요하다.

### 이미지 태그 전략

이미지를 두 개의 태그로 푸시한다.

```bash
docker build \
  -t $ECR_REGISTRY/my-build-common:latest \
  -t $ECR_REGISTRY/my-build-common:$IMAGE_TAG \
  .
```

`IMAGE_TAG`는 `$(date +%Y%m%d).${{ github.run_number }}` 형태로 생성된다.
예: `20260401.42`

- `latest` 태그: 항상 가장 최신 이미지를 가리킴. 이후 앱 빌드 step에서 pull할 때 사용.
- 날짜.번호 태그: 어느 빌드에서 만들어진 이미지인지 추적 가능. 롤백 시 유용.

---

## Step 3: 앱별 병렬 빌드

각 앱의 빌드 job은 아래 두 조건을 `AND`로 연결한다.

```yaml
build-backend:
  needs: [detect-changes, build-common]
  if: |
    needs.build-common.result == 'success' &&
    (needs.detect-changes.outputs.backend == 'true' ||
     needs.detect-changes.outputs.build_common == 'true')
```

1. `build-common`이 성공해야 한다 (베이스 이미지가 준비됐다는 전제)
2. `apps/backend/` 파일이 변경됐거나, 공통 레이어가 변경됐거나

두 번째 조건에서 `build_common == 'true'`가 포함된 이유:
베이스 이미지가 바뀌면, 그걸 사용하는 모든 앱 이미지도 다시 빌드해야 하기 때문이다.

### 각 앱 빌드의 흐름

앱 빌드 job들은 거의 동일한 구조를 가진다.

```yaml
steps:
  - name: Checkout repository
  - name: Configure AWS credentials (OIDC)
  - name: Login to Amazon ECR
  - name: Set image tag
  - name: Pull build-common           # ← ECR에서 베이스 이미지 가져오기
  - name: Build & push {app}          # ← 앱 이미지 빌드 & 푸시
```

`Pull build-common` step이 핵심이다.

```bash
docker pull $ECR_REGISTRY/my-build-common:latest
docker tag $ECR_REGISTRY/my-build-common:latest build-common:latest
```

ECR에서 공통 이미지를 받아서 로컬에 `build-common:latest`라는 이름으로 태그한다.
앱 Dockerfile에서 `FROM build-common:latest`를 선언했다면, 이 이미지를 베이스로 사용하게 된다.

각 앱 job은 **독립적인 VM**에서 실행되기 때문에, 앱 간 빌드가 완전히 병렬로 진행된다.

```
build-common 완료 후
    ├── build-backend    (병렬)
    ├── build-frontend   (병렬)
    ├── build-proxy-a    (병렬)
    └── build-proxy-b    (병렬)
```

---

## OIDC 기반 AWS 인증

예전에는 AWS 인증을 위해 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 GitHub Secrets에 저장했다.
이 방식은 키가 유출될 위험이 있고, 주기적으로 교체해야 하는 번거로움이 있다.

OIDC (OpenID Connect) 방식은 이 문제를 해결한다.

```yaml
- name: Configure AWS credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
    aws-region: ap-northeast-2
```

동작 원리는 이렇다.

```
GitHub Actions runner
    → GitHub OIDC Provider에 토큰 요청
    → JWT 토큰 발급
    → AWS STS에 AssumeRoleWithWebIdentity 요청 (JWT 포함)
    → IAM이 신뢰 정책(Trust Policy) 검증
    → 임시 자격증명 발급 (최대 1시간)
```

장점:
- **장기 자격증명 없음**: Access Key 대신 단기 임시 자격증명 사용
- **자동 만료**: 사용 후 자동으로 만료되므로 유출돼도 피해가 제한됨
- **IAM Trust Policy로 제어**: 어느 레포지토리, 어느 브랜치에서만 이 Role을 assume할 수 있는지 제한 가능

IAM Trust Policy 예시:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/*"
      }
    }
  }]
}
```

`StringLike`와 `*` 와일드카드를 사용하면 어떤 브랜치에서도 이 Role을 assume할 수 있다.
`StringEquals`로 특정 브랜치만 허용하는 것도 가능하다.

---

## Concurrency: 중복 실행 방지

같은 브랜치에 연속으로 push가 들어오면, 이전 실행이 완료되기 전에 새 실행이 시작된다.
두 실행이 동시에 ECR에 push하면 의도치 않은 순서 문제가 생길 수 있다.

이를 방지하기 위해 `concurrency` 설정을 사용했다.

```yaml
concurrency:
  group: build-ecr-${{ github.ref_name }}
  cancel-in-progress: true
```

- 같은 브랜치(`github.ref_name`)에서 실행 중인 워크플로우가 있으면, 새 실행이 시작될 때 기존 실행을 **자동으로 취소**한다.
- `group`을 브랜치 이름 기반으로 설정했기 때문에, 서로 다른 브랜치의 실행은 영향을 주지 않는다.

---

## Step 4: Slack 알림

모든 빌드 job이 끝나면 결과를 Slack으로 알린다.

```yaml
notify-slack:
  needs: [build-common, build-backend, build-frontend, ...]
  if: always()  # 성공/실패/스킵 상관없이 항상 실행
```

`if: always()`가 없으면, 이전 job들 중 하나라도 실패했을 때 이 job이 실행되지 않는다.
실패했을 때도 알림을 받아야 하므로 `always()`를 명시했다.

### 셸 인젝션 방지

알림 메시지에서 각 job의 결과를 표시할 때, 처음에는 이렇게 작성하려고 했다.

```yaml
# ❌ 이렇게 하면 안 됨
run: |
  if [[ "${{ needs.build-backend.result }}" == "failure" ]]; then
    ...
  fi
```

GitHub Actions 표현식을 직접 쉘 스크립트에 삽입하면, job 결과값에 예상치 못한 문자(따옴표, 세미콜론 등)가 포함될 경우 셸 인젝션 취약점이 생길 수 있다.

이를 방지하기 위해 `env`를 통해 환경변수로 분리했다.

```yaml
- name: Determine overall status
  id: status
  env:
    R_COMMON: ${{ needs.build-common.result }}
    R_BACKEND: ${{ needs.build-backend.result }}
    R_FRONTEND: ${{ needs.build-frontend.result }}
  run: |
    ALL="$R_COMMON $R_BACKEND $R_FRONTEND"
    if echo "$ALL" | grep -q "failure"; then
      echo "result=failure" >> $GITHUB_OUTPUT
    fi
```

GitHub 표현식 결과를 먼저 환경변수(`R_COMMON`, `R_BACKEND` 등)에 담고,
쉘 스크립트에서는 그 환경변수만 참조한다.
환경변수는 셸 파싱 전에 치환되기 때문에, 인젝션 위험이 없다.

### Slack 메시지 구조

Slack 알림은 Block Kit 포맷으로 구성했다.

```
┌────────────────────────────────────────┐
│ ✅ ECR Build success                   │
│                                        │
│ Branch: main    Actor: kim             │
│ Commit: [abc1234]                      │
│                                        │
│ Image Results                          │
│ • build-common — success               │
│ • backend      — success               │
│ • frontend     — skipped               │
│ • proxy-a      — success               │
│                                        │
│ [View Run]                             │
└────────────────────────────────────────┘
```

각 job의 결과가 `success`, `failure`, `skipped`, `cancelled` 중 하나로 표시된다.
특히 `skipped`는 path filter에 의해 빌드가 실행되지 않은 경우인데, 이를 통해 "이번 빌드에서 어떤 서비스가 포함됐는지"를 한눈에 확인할 수 있다.

---

## 전체 실행 흐름 정리

`apps/backend/` 파일만 수정해서 push한 경우를 예시로 보면:

```
push 발생
  │
  ├─ detect-changes
  │    backend: true  ✅
  │    frontend: false ❌
  │    build_common: false ❌
  │
  ├─ build-common
  │    ECR에 이미지 있음 + build_common=false → 빌드 스킵
  │    (AWS 인증 + ECR 확인만 하고 종료)
  │
  ├─ build-backend    ← backend=true → 실행
  ├─ build-frontend   ← frontend=false → 스킵
  ├─ build-proxy-a    ← 변경 없음 → 스킵
  └─ build-proxy-b    ← 변경 없음 → 스킵
  │
  └─ notify-slack
       build-common: success
       backend: success
       frontend: skipped
       proxy-a: skipped
```

`packages/shared/` 파일을 수정했다면:

```
build_common: true  →  build-common 실행
backend: false      →  하지만 build_common=true이므로 build-backend도 실행
frontend: false     →  build_common=true이므로 build-frontend도 실행
...모든 앱 빌드 실행
```

---

## 마치며

이 파이프라인을 구성하면서 가장 많이 고민했던 건 **"무엇을 기준으로 빌드를 트리거할 것인가"** 였다.

단순히 "push마다 전부 빌드"는 너무 느리고,
"개발자가 직접 판단해서 빌드"는 실수가 생긴다.

path filter를 활용해서 변경된 파일 기반으로 자동 판단하고, build-common이라는 공통 레이어 개념을 추가해서 의존 관계를 명시적으로 처리한 것이 핵심이었다.

부수적으로 배운 것들도 있었다.

- OIDC 인증은 처음 설정이 복잡하지만, 장기 자격증명 관리의 부담이 없어서 장기적으로 훨씬 깔끔하다.
- GitHub Actions의 `concurrency`는 단순해 보이지만, 실제로 연속 push 상황에서 이상한 결과를 막아주는 중요한 설정이다.
- 셸 인젝션 방지를 위해 `env`로 분리하는 패턴은 처음에는 번거롭게 느껴졌지만, 보안 측면에서 올바른 습관이라는 걸 이해하고 나서는 자연스럽게 따르게 됐다.

ECS 마이그레이션으로 인프라 구조가 바뀌었고, 이 파이프라인으로 배포 흐름도 자동화됐다.
"코드를 push하면 이미지가 ECR에 올라간다"는 단순한 흐름이 되기까지 꽤 많은 것들이 맞물려 있었다는 걸, 직접 구성해보고 나서야 제대로 이해했다.
