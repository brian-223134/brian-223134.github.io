---
title: "[인턴] ALB+EC2에서 CloudFront+S3로: 프론트엔드 정적 호스팅 전환과 GitHub Actions 자동 배포, 그리고 본사 IP 카나리 라우팅 실패기"
excerpt: "수동 배포하던 ALB+EC2 프론트 서버를 CloudFront+S3 정적 호스팅으로 이전하고 GitHub Actions로 배포를 자동화한 과정, 그리고 Route53 IP-based Routing 카나리가 본사 네트워크에서 실패한 원인(ECS 미전달)을 추적한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, CloudFront, S3, Route53, GitHub Actions, DNS, ECS, 카나리배포]

permalink: /intern/company/blitz-dynamics/27/

toc: true
toc_sticky: true

date: 2026-04-17
last_modified_at: 2026-04-17
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/26/)에서는 Fargate 태스크가 `ETIMEDOUT`으로 혼자 죽은 사건을 포스트모템 형태로 정리했습니다. 백엔드 쪽 장애 대응이 어느 정도 일단락되고 나서, 이번에는 **프론트엔드 쪽 인프라**를 손볼 차례가 되었습니다.

기존 프론트엔드는 **ALB + EC2** 위에 올라가 있었습니다. 빌드된 번들을 EC2에 직접 올려서 nginx로 서빙하는 형태였고, 배포는 전부 **수동**이었습니다. 정적 자산 덩어리 하나 올리려고 EC2에 SSH로 들어가서 `rsync`를 돌리고 있는 상황이었는데, 여러모로 비효율적이었습니다.

이 글은 아래 세 가지를 다룹니다.

1. **ALB + EC2 → CloudFront + S3 전환**: 왜 옮겼고, 어떻게 구성했는지
2. **GitHub Actions로 배포 자동화**: 수동 `rsync`를 없애고 PR merge 한 번으로 배포
3. **본사(HQ) IP 카나리 라우팅 실패 원인 분석**: Route53 IP-based Routing이 왜 본사 네트워크에서 먹히지 않았는지

특히 3번은 DNS 내부 동작에 대해서 한 번 제대로 깊게 파봐야 할 계기가 되었기 때문에 길게 정리했습니다.

> ⚠️ **민감 정보 마스킹**: 도메인, 본사 공인 IP 대역, CloudFront 배포 도메인, Route53 CIDR Collection ID, NS 서버명 등은 마스킹하거나 예시로 대체했습니다.

---

## 1. 왜 ALB + EC2를 떠나려고 했나

### 기존 구성의 문제

기존 프론트엔드는 대략 아래와 같았습니다.

```
[사용자] → Route53 → ALB → EC2(nginx + 정적 번들)
```

단순히 정적 SPA 번들을 서빙하기 위한 구성치고는 과한 부분이 많았습니다.

- **비용**: EC2 인스턴스를 24/7 띄워두고, ALB까지 앞단에 두고 있음. 실제로 하는 일은 정적 파일 서빙.
- **수동 배포**: 빌드 산출물을 EC2에 직접 업로드. 배포 휴먼 에러 가능성이 상시 존재.
- **캐시 부재**: ALB 앞단에 CDN이 없어서, 해외 사용자도 국내 리전 EC2에서 직접 받아감.
- **스케일링**: 트래픽이 튀면 EC2 하나로 버티거나, ASG를 엮어야 하는데 정적 파일에 이건 과설계.

정적 SPA에는 **Object Storage + CDN** 조합이 교과서적인 답이고, AWS에서는 **S3 + CloudFront**가 그 정답지입니다.

### 목표 구성

```
[사용자] → Route53 → CloudFront (OAC) → S3 (정적 번들)
```

- CloudFront: 글로벌 엣지 캐시, TLS 종단, SPA fallback(`/* → /index.html`)
- S3: 빌드 산출물(`dist/`) 저장. 버킷은 퍼블릭 차단, **Origin Access Control(OAC)** 로만 CloudFront가 접근
- Route53: 도메인을 CloudFront로 라우팅

---

## 2. CloudFront + S3 구성

### 2.1. S3 버킷

- 퍼블릭 액세스 전면 차단
- 버킷 정책은 **특정 CloudFront Distribution만** `s3:GetObject` 허용 (OAC의 Service Principal + `AWS:SourceArn` 조건)
- 버전 관리 활성화 (롤백 용)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::***-frontend-prod/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::***:distribution/E***"
      }
    }
  }]
}
```

### 2.2. CloudFront Distribution

핵심만 정리하면 아래와 같습니다.

- **Origin**: S3 버킷 (OAC 연결)
- **Default Root Object**: `index.html`
- **Custom Error Response**: `403`, `404` → `/index.html` (status 200)
  - SPA는 새로고침/직접 진입 시 해당 경로의 객체가 S3에 없으므로, CloudFront에서 `index.html`로 폴백시켜야 라우팅이 망가지지 않습니다.
- **TLS**: ACM 인증서 (us-east-1에서 발급한 것만 사용 가능, 놓치기 쉬운 포인트)
- **Alternate Domain Name(CNAME)**: `app.***.com` (예시)
- **Cache Policy**:
  - `index.html`은 **짧은 TTL** (또는 `Cache-Control: no-cache`)
  - 해시된 정적 자산(`assets/*.[hash].js`)은 **1년짜리 long TTL**
- **Compression**: 활성화 (gzip/brotli)

`index.html`을 짧게 잡지 않으면 새 번들을 올려도 엣지 캐시에 걸려서 **일부 사용자만 구버전**이 뜨는 고전적인 문제가 생깁니다. 반대로 해시된 자산은 immutable이므로 길게 잡아도 안전합니다.

### 2.3. 엣지 동작(SPA fallback) 체크

구성 후 반드시 확인해야 하는 케이스는 다음 세 가지입니다.

1. `/` 루트 진입 → `index.html` 정상
2. `/some/nested/route` 직접 진입 → 403/404가 `index.html`로 리라이트되어 SPA 라우터가 받음
3. 해시 자산 요청 → 캐시 적중, 오래된 해시 자산도 여전히 받을 수 있음 (배포 직후 이전 버전 사용자 보호)

---

## 3. GitHub Actions로 배포 자동화

수동 `rsync`를 없애는 게 이번 작업의 큰 동기 중 하나였습니다.

### 3.1. 자동화 방침

- `main` 브랜치에 merge되면 자동 빌드 → S3 업로드 → CloudFront Invalidation
- AWS 자격 증명은 **정적 Access Key 대신 OIDC(Assume Role)** 로
- 캐시 TTL 특성에 맞춰 **업로드 Cache-Control 헤더**를 객체별로 다르게

### 3.2. OIDC 기반 Role

GitHub Actions용 IAM Role에는 아래 정도의 권한만 줍니다.

- `s3:PutObject`, `s3:DeleteObject` on 특정 버킷
- `cloudfront:CreateInvalidation` on 특정 Distribution

Trust Policy에서는 `token.actions.githubusercontent.com`을 OIDC Provider로 두고, `sub` 조건으로 **특정 레포 + 특정 브랜치**만 AssumeRole 가능하도록 제한했습니다. 정적 키를 레포 Secret에 박아두는 것보다 훨씬 안전합니다.

### 3.3. 워크플로우 골자

```yaml
name: deploy-frontend

on:
  push:
    branches: [main]

permissions:
  id-token: write   # OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile
      - run: yarn build   # -> dist/

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::***:role/gha-frontend-deploy
          aws-region: ap-northeast-2

      # 1) 해시 자산: 1년짜리 immutable
      - name: Upload hashed assets
        run: |
          aws s3 sync dist/ s3://***-frontend-prod/ \
            --delete \
            --exclude "index.html" \
            --cache-control "public,max-age=31536000,immutable"

      # 2) index.html: 캐시 금지 (엣지도 매번 확인)
      - name: Upload index.html
        run: |
          aws s3 cp dist/index.html s3://***-frontend-prod/index.html \
            --cache-control "no-cache, no-store, must-revalidate" \
            --content-type "text/html; charset=utf-8"

      # 3) CloudFront Invalidation
      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id E*** \
            --paths "/index.html"
```

포인트는 **해시 자산을 먼저 올리고 `index.html`을 마지막에 올리는 것**입니다. 순서가 반대가 되면, 새 `index.html`이 아직 존재하지 않는 해시 자산을 참조하는 순간적인 불일치 구간이 생길 수 있습니다.

Invalidation도 `/*`로 통째로 날리는 대신 `/index.html`만 무효화했습니다. 해시 자산은 파일명 자체가 바뀌므로 invalidation이 필요 없고, `/*` invalidation은 요금이 붙습니다.

이 정도만 해두면 **PR merge → 약 2~3분 뒤 프로덕션 반영**까지 자동으로 흐릅니다. 예전의 `ssh + rsync` 루틴이 전부 사라졌습니다.

---

## 4. 전환 전략: Source IP 기반 카나리

전면 컷오버는 위험했습니다. 프론트엔드 번들이 바뀌는 건 아니지만, 원본이 S3로 바뀌고 캐시 계층(CloudFront)이 새로 들어오는 변화라, 최소한 **본사 내부 사용자들로 먼저 검증**한 뒤 외부 컷오버를 하고 싶었습니다.

### 카나리 구상

`app.***.com` 도메인에서, **본사(HQ) 네트워크**에서 접속한 요청만 CloudFront로 보내고, 나머지는 기존 ALB로 계속 라우팅하는 구성을 생각했습니다.

Route53에는 이 용도로 딱 맞아 보이는 기능이 있습니다.

- **CIDR Collection** + **IP-based Routing**
- 즉, 사용자의 Source IP CIDR에 따라 다른 타겟으로 라우팅 가능

### Route53 설정

- **CIDR Collection**: `****-****-****-****-************` (예시 ID)
- **Location**:
  - `HQ` = `xxx.xx.x.0/24` (본사 공인 IP 대역, **/24가 최소 단위** — 이보다 더 좁게는 안 됨)
- **A 레코드 `app.***.com` 2개 (IP-based routing)**
  - `SetId = cf-hq` → Location `HQ` → **CloudFront ALIAS** (`d***.cloudfront.net`)
  - `SetId = alb-default` → Location `*` (default) → **기존 ALB ALIAS**

구성 자체는 정상적으로 들어갔고, authoritative NS에 `+subnet` 옵션을 주고 직접 물어보면 의도대로 응답이 돌아옵니다.

```bash
# 본사 IP 대역을 ECS로 붙여서 authoritative NS에 직접 질의
$ dig @ns-***.awsdns-**.com app.***.com +subnet=xxx.xx.x.129/24

;; ANSWER SECTION:
app.***.com.  60  IN  A  3.170.221.xx   # CloudFront IP
```

Route53 단에서 IP-based routing은 제대로 동작한다는 증거입니다.

---

## 5. 그런데 본사 네트워크에서는 여전히 ALB로 갔다

카나리 검증하러 본사 데스크에서 직접 쳐봤습니다.

```bash
# 본사 사내망 PC에서
$ dig app.***.com
;; ANSWER SECTION:
app.***.com.  60  IN  A  52.79.x.x      # ALB IP
app.***.com.  60  IN  A  54.116.x.x     # ALB IP
app.***.com.  60  IN  A  3.36.x.x       # ALB IP
```

CloudFront IP(`3.170.221.x` 대역)이 아니라 **기존 ALB IP(`52.79.x`, `54.116.x`, `3.36.x`)** 가 반환됩니다. 즉, 본사에서 접속해도 카나리 분기가 안 걸리고 전부 기존 ALB로 흘러가고 있다는 뜻입니다.

설정은 분명 정상인데, 왜 본사에서만 안 맞을까.

---

## 6. 원인: Route53 IP-based Routing과 ECS(EDNS Client Subnet)

### 핵심 오해

Route53의 IP-based routing은 이름 때문에 오해하기 쉬운데, **"DNS 질의를 보낸 사용자의 IP"로 매칭하는 게 아닙니다.**

실제로는 **"resolver가 DNS 질의에 붙여 보낸 EDNS Client Subnet(ECS) 정보"** 로 매칭합니다.

### 왜 그런가: DNS 질의 흐름

사용자가 브라우저에서 `app.***.com`을 입력했을 때의 실제 흐름을 펴보면 이렇습니다.

```
[사용자 PC: xxx.xx.x.129 (HQ)]
      │
      │ "app.***.com A?" (DNS 질의)
      ▼
[회사/ISP Resolver: 예: KT 168.126.63.1]   ← 여기가 ECS를 붙여야 함
      │
      │ "app.***.com A? (client_subnet=xxx.xx.x.0/24)"
      ▼
[Route53 Authoritative NS]
```

Route53 권한 서버가 보는 상대는 **사용자 PC가 아니라 resolver**입니다. Route53 입장에서는 쿼리 출처 IP가 KT resolver IP(`168.126.63.x`)로 보이고, 사용자 PC의 `xxx.xx.x.129`라는 정보는 애초에 패킷 안에 없습니다.

사용자 subnet 정보가 Route53까지 전달되려면, 중간의 resolver가 **EDNS0 Client Subnet(ECS)** 옵션에 `xxx.xx.x.0/24`를 명시적으로 담아 보내야 합니다.

### Resolver별 ECS 정책

| Resolver               | ECS 전달 여부 | 본사에서 쿼리 시 결과 |
|------------------------|---------------|----------------------|
| Google `8.8.8.8`       | ✅ 전달        | CloudFront IP        |
| Cloudflare `1.1.1.1`   | ❌ 프라이버시 정책상 제거 | ALB IP |
| KT `168.126.63.1`      | ❌ 미전달      | ALB IP ← 현재 본사 상황 |

- Google Public DNS: **ECS를 기본으로 전달**합니다. 그래서 Google을 쓰면 Route53이 사용자 subnet을 알게 되고, HQ 규칙에 맞춰 CloudFront IP를 돌려줍니다.
- Cloudflare 1.1.1.1: **프라이버시를 이유로 ECS를 일부러 제거**합니다. 사용자 위치 노출을 최소화하는 것이 제품 방침.
- KT 등 국내 ISP resolver: **ECS를 전달하지 않음**. 본사 인터넷 회선이 이 resolver에 물려 있으면, Route53은 본사 subnet을 알 수 없습니다.

ECS가 없으면 Route53은 어느 Location에도 매칭할 수 없어서, **default(`*` = ALB)** 로 fallback합니다. 본사에서 ALB IP가 튀어나온 이유가 정확히 이것입니다.

### 3중 크로스체크로 확정

원인을 확정하기 위해 세 가지 경로로 쿼리해봤습니다.

**① authoritative NS에 직접 ECS 주입** — Route53 정책이 정상인지 확인

```bash
$ dig @ns-***.awsdns-**.com app.***.com +subnet=xxx.xx.x.129/24
;; ANSWER: 3.170.221.xx   # CloudFront
```

→ Route53 설정 자체는 정상. HQ CIDR이 들어오면 CloudFront를 반환.

**② Google DNS로 쿼리** — ECS 전달 경로가 정상적으로 동작하는지 확인

```bash
$ dig @8.8.8.8 app.***.com
;; ANSWER: 3.170.xx.xx   # CloudFront
```

→ ECS를 전달해주는 resolver를 쓰면 본사에서도 CloudFront IP가 나옴.

**③ Cloudflare / 기본(KT) resolver로 쿼리** — ECS 미전달 시 동작 확인

```bash
$ dig @1.1.1.1 app.***.com
;; ANSWER: 52.79.x.x, 54.116.x.x, 3.36.x.x   # ALB

$ dig app.***.com    # 기본 resolver (KT)
;; ANSWER: 52.79.x.x, 54.116.x.x, 3.36.x.x   # ALB
```

→ ECS 없으면 default로 fallback. 본사에서 관측된 증상과 정확히 일치.

### 결론

- Route53 설정은 문제없음
- CloudFront 구성도 문제없음
- **본사가 쓰는 DNS resolver(KT)가 ECS를 전달하지 않는 것**이 원인
- Route53은 Client Subnet을 식별할 수 없어서 기본값(`*` = ALB)으로 fallback하고 있었던 것

처음에는 당연히 Route53 설정 오류나 CloudFront Alternate Domain Name 등록 문제를 의심했는데, 그쪽은 모두 정상이었고 문제는 **사내 네트워크가 쓰는 resolver의 정책**이라는, 우리가 직접 건드릴 수 없는 레이어에 있었습니다.

---

## 7. 해결 선택지

현실적으로 고른 옵션들을 정리하면 이렇습니다.

| 방법 | 적용 범위 | 트레이드오프 |
|------|-----------|--------------|
| **HQ 네트워크 DNS를 `8.8.8.8`로 변경** | HQ 전체, 영구 | 사내 인프라 변경이라 IT 담당자/정책 결정 필요 |
| **개인 PC DNS를 `8.8.8.8`로 변경** | 개인 | 테스트/얼리 유저용으로만 실용적 |
| **`/etc/hosts`에 CloudFront IP 고정** | 개인 PC | CloudFront IP는 바뀔 수 있어 임시 테스트용 |
| **HQ 카나리 포기 → ALIAS 전면 전환** | 전체 트래픽 | 카나리 단계 생략, 즉시 컷오버 |

이번 건은 **"HQ 카나리가 DNS resolver 정책상 동작 보장이 안 된다"** 는 사실 자체가 가장 큰 발견이었습니다. 카나리를 굳이 Source IP 레이어에서 하려면 HQ의 DNS 방향을 Google로 옮기거나, 아예 다른 방식 — 예를 들어 **URL/쿠키 기반 카나리(예: 내부 전용 별도 도메인 `app-canary.***.com` → CloudFront 100%)** — 를 택하는 편이 훨씬 안정적입니다.

최종적으로는 **별도 내부 도메인을 CloudFront에 박아두고, 본사 인원이 그 도메인으로 먼저 충분히 검증한 뒤, 원 도메인을 ALIAS로 한 번에 CloudFront로 컷오버**하는 방향으로 전환했습니다. DNS 레이어의 불확실성을 우회한 겁니다.

---

## 8. 얻은 교훈

1. **DNS 기반 IP 카나리는 resolver가 협조해야만 성립한다.** Route53 IP-based Routing은 "클라이언트 IP"가 아니라 "ECS"를 본다. ECS는 resolver의 자발적 기능이고, 국내 ISP resolver와 Cloudflare는 대체로 안 보내준다.
2. **authoritative NS에 직접 `+subnet`을 주고 물어보는 게 가장 빠른 원인 분리 방법이다.** Route53 설정 문제 / CloudFront 문제 / resolver 문제를 한 번에 갈라낼 수 있다.
3. **정적 SPA는 CDN + Object Storage가 정답.** ALB + EC2는 유연하지만, 정적 호스팅에는 과설계. 비용·운영 복잡도·캐시 모두 S3 + CloudFront 쪽이 압도적으로 낫다.
4. **GitHub Actions OIDC + AssumeRole은 기본값으로 가져가자.** 정적 Access Key를 Secret에 박는 방식은 더 이상 쓸 이유가 없다.
5. **`index.html`은 `no-cache`, 해시 자산은 `immutable`.** SPA 배포에서 구버전이 일부 사용자에게만 보이는 고전적인 문제는 대부분 이 TTL 설정을 대충 두는 데서 온다.

CloudFront + S3 전환과 배포 자동화는 성공적으로 마무리됐고, 의도치 않게 DNS 내부 동작을 제대로 들여다볼 기회까지 얻었습니다. 다음 글에서는 전환 이후 실제 엣지 캐시 히트율과 TTFB가 어떻게 달라졌는지 측정한 내용을 정리할 예정입니다.
