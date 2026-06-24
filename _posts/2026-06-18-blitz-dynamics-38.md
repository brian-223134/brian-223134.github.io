---
title: "[인턴] Airbyte sync가 끝나면 dbt는 누가 깨우는가 (adnanh/webhook 트리거)"
excerpt: "Dagster나 Airflow 같은 오케스트레이터 없이, Go 단일 바이너리 하나로 Airbyte sync 성공과 dbt 실행을 이어 붙인 이야기. POST 한 방이 dbt까지 가닿는 경로를 따라가며, 그 과정에서 며칠씩 헤맸던 지점들을 정리했다."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, dbt, Airbyte, webhook, 오케스트레이션]

permalink: /intern/company/blitz-dynamics/38/

toc: true
toc_sticky: true

date: 2026-06-18
last_modified_at: 2026-06-18
---

## 들어가며

[지난 글](/intern/company/blitz-dynamics/37/)에서 `abctl`로 띄운 Airbyte가 사실 `kind`(Docker 안의 단일 노드 쿠버네티스) 클러스터였다는 이야기를 했다. 그 클러스터가 하는 일은 결국 하나다. 소스 MySQL의 데이터를 웨어하우스 PostgreSQL의 `raw_*` 테이블로 실어 나르는 것.

그런데 데이터가 `raw_*`까지 왔다고 끝이 아니다. 그 위에서 dbt가 staging과 marts로 변환을 돌려야 비용 대시보드도 BI 마트도 신선해진다. 그러니 남는 질문은 이거였다. **sync가 끝난 순간, 누가 dbt를 깨우지?**

Fivetran을 쓸 땐 이런 고민이 없었다. "sync 끝나면 transform" 한 줄이 managed로 들어 있었으니까. 그걸 걷어내고 나니 그 한 줄을 직접 이어야 했다. 처음엔 당연히 Dagster나 Airflow를 떠올렸다. 그런데 결국 고른 건 오케스트레이터도, 상주 데몬도, 우리가 짠 리스너도 없는 구성이었다. Go로 만든 단일 바이너리 하나가 HTTP POST를 받아 셸 스크립트를 실행하는 게 전부다.

이 글은 그 POST 한 방이 dbt까지 가닿는 경로를 따라가는 기록이다. 그리고 중간중간, `localhost`로 바인딩했다가 트리거가 영영 안 오던 일이나 `curl`이 200을 돌려줬는데 빌드는 안 돌던 일처럼 내가 실제로 며칠씩 헤맸던 지점들을 같이 적었다.

---

## 1. 전체 그림부터

세부로 들어가기 전에 경로 전체를 한 번 펼쳐 보자. Airbyte가 sync를 성공시킨 순간부터 dbt가 웨어하우스에서 SQL을 실행하기까지다.

```
Airbyte sync 성공
  → POST http://<EC2 사설IP>:9000/hooks/dbt-on-sync?token=<공유 시크릿>
  → adnanh/webhook (systemd 상주, URL의 token 매칭)
  → run-dbt.sh   : flock(중복 방지) → git pull --ff-only(코드 배포) → dbt-exec.sh에 위임
  → dbt-exec.sh  : docker compose up -d(컨테이너 보장) → 상주 컨테이너에 exec로 dbt 실행
  → dbt run --select "+mart_cost_a +mart_bi_b +mart_usage_c"   # 실제 모델명은 가림
```

화살표는 다섯 개지만 각 단계가 맡는 일은 단순하다. Airbyte는 sync가 끝나면 미리 등록해 둔 URL로 POST를 한 번 쏜다. adnanh/webhook은 그 POST의 토큰이 맞으면 정해진 커맨드를 실행한다. 그 커맨드가 `run-dbt.sh`고, 이 스크립트가 코드를 최신으로 당겨 와 dbt를 돌린다. 앞으로 이 화살표들을 위에서 아래로 하나씩 따라 내려갈 것이다.

한 가지만 미리 짚어 두자. 이 경로 어디에도 "상시 떠서 이벤트를 기다리는, 우리가 짠 코드"는 없다. adnanh/webhook은 검증된 오픈소스 바이너리고, `run-dbt.sh`는 트리거 때만 잠깐 떴다 사라지는 셸 스크립트다. 이게 이 설계의 전부이자 핵심이다.

---

## 2. 왜 Dagster가 아니었나

먼저 답해야 할 건 "왜 멀쩡한 오케스트레이터를 두고 단일 바이너리를 골랐나"다. Dagster나 Airflow는 의존성 그래프, 재시도, 백필, 스케줄 UI까지 다 준다. 그걸 마다한 이유는 기능이 모자라서가 아니라, 우리 상황엔 그 기능들이 과했기 때문이다.

결정의 1순위 기준은 따로 있었다. **인턴이 떠난 뒤 이 파이프라인은 두 명이 간헐적으로만 들여다본다.** 이 조건이 모든 판단을 바꿨다.

Dagster를 올리면 그 자체가 데몬·웹서버·DB로 이뤄진 상주 서비스 한 벌이 되고, 곧 새로운 운영 대상이 된다. 평소엔 잘 돌다가 정작 몇 달 뒤 누가 손볼 때쯤이면 버전이 밀려 있고 왜 안 도는지 아무도 기억 못 하는 상태가 되기 십상이다. 풀타임 데이터 팀이 상주하는 조직이면 그 복잡도가 곧 힘이지만, 가끔 들르는 운영 체제에서는 그 복잡도가 그대로 부채가 된다. 같은 이유로 FastAPI 리스너를 직접 짜는 선택지도 접었다. 커스텀 코드는 짜는 사람에겐 투명해도 나중에 읽는 사람에겐 블랙박스다.

따지고 보면 트리거가 하는 일은 "POST가 오면 정해진 커맨드를 실행한다" 한 줄이다. 그 한 줄을 위해 상주 데몬 한 벌을 떠안을 이유가 없었다. 필요한 건 정문에 달린 초인종 하나지, 경비실 한 채와 상주 경비원이 아니었다.

게다가 EC2는 Airbyte 때문에 어차피 24시간 떠 있다. 그 위에 systemd 서비스 하나를 더 얹는 비용은 사실상 0이다. 이미 켜진 서버에 초인종 하나 다는 셈이라, 비용으로 봐도 제일 가벼운 답이었다.

---

## 3. adnanh/webhook이라는 초인종

그 초인종이 [adnanh/webhook](https://github.com/adnanh/webhook)이다. 한 문장으로 줄이면 "POST를 받아 규칙에 맞으면 선언된 커맨드를 실행하는, 그게 전부인 Go 단일 바이너리"다.

단일 바이너리라는 점이 중요하다. 런타임도 패키지 매니저도 필요 없이, 실행 파일 하나를 `/usr/local/bin/webhook`에 떨궈 두면 된다. Amazon Linux 2023의 `dnf`나 EPEL에는 패키지가 없어서 GitHub 릴리스 바이너리(v2.8.3)를 직접 받아 설치했다.

동작도 코드가 아니라 선언으로 정한다. `hooks.yaml`에 "이런 POST가 오면 이 커맨드를 실행하라"를 적어 두면 바이너리가 그대로 따른다. 우리 훅은 이렇게 생겼다.

```yaml
# /etc/webhook/hooks.yaml (시크릿·실제 모델명·실제 경로는 가림)
- id: dbt-on-sync
  execute-command: /usr/local/bin/run-dbt.sh
  command-working-directory: /opt/<dbt-repo>/dbt
  pass-arguments-to-command:                 # 이 훅은 모델만(run) — 전체 build는 timer 몫
    - { source: string, name: run }
    - { source: string, name: --select }
    - { source: string, name: "+mart_cost_a +mart_bi_b +mart_usage_c" }
  trigger-rule:
    match:
      type: value
      value: "<shared-secret>"
      parameter: { source: url, name: token }
```

읽는 법은 어렵지 않다. `id: dbt-on-sync`라 URL은 `/hooks/dbt-on-sync`로 들어오고, 매칭되면 `run-dbt.sh`를 `run --select "+...셀렉터..."` 인자와 함께 실행한다. 단, 실행 조건은 `trigger-rule.match` 하나뿐이다. URL 쿼리의 `token` 값이 `hooks.yaml`에 적힌 값과 같아야 한다. 위 예시의 모델명(`mart_cost_a` 등)은 가린 이름이고, 실제로는 이 셀렉터가 비용·BI 마트 95개를 가리킨다. sync가 끝날 때마다 이 95개가 갱신된다.

운영하면서 한 번 데인 게 있다. `hooks.yaml`은 webhook이 **시작할 때 딱 한 번만** 읽는다. 그래서 훅을 고쳐도 바로 반영되지 않는다. 무중단으로 다시 읽히려면 시그널을 보낸다.

```bash
sudo systemctl kill -s USR1 webhook    # hooks.yaml만 다시 읽음 (진행 중인 빌드엔 무영향)
```

`restart`로도 되긴 하지만, 빌드가 도는 중에 restart하면 systemd가 자식 프로세스(`run-dbt.sh`)까지 같이 죽인다. 그래서 평소엔 `USR1` reload를 쓰고 restart는 빌드가 없는 틈에만 쓴다.

---

## 4. localhost로 바인딩하면 트리거가 안 온다

이 글에서 내가 제일 오래 붙들고 있던 지점이다. 그리고 그 원인이 하필 37편에서 다룬 "Airbyte가 kind 안에서 돈다"는 사실이었다.

수신기를 systemd로 띄울 때 별생각 없이 `127.0.0.1:9000`에 바인딩했다. 외부로 안 새니 안전해 보였다. 그런데 Airbyte에서 webhook 테스트를 눌러도 트리거가 도착하지 않았다. journald엔 아무것도 안 찍혔다. 한참 뒤에야 원인을 알았다.

Airbyte는 kind Pod 안에서 돈다. 그 Pod에게 `localhost`는 EC2 호스트가 아니라 Pod 자기 자신이다. 37편에서 본 그 긴 이름의 Pod들을 떠올려 보면, Airbyte 서버도 그중 하나 안에서 실행된다. 그 Pod가 `127.0.0.1:9000`으로 POST를 쏘면 호스트의 9000번이 아니라 Pod 자신의 9000번을 두드리는 셈이고, 거기엔 아무도 없다. POST는 그냥 허공으로 사라진다. 한 세대 안에서 "1층 현관 초인종 눌러"라고 했더니 정작 누른 게 그 세대 안 인터폰이었던 격이다.

그래서 두 가지를 바꿨다. 하나는 바인딩을 `0.0.0.0:9000`으로 여는 것(`-ip 0.0.0.0 -port 9000`). 그래야 호스트의 모든 인터페이스로 들어오는 요청을 받는다. 다른 하나는 Airbyte가 POST를 보낼 목적지를 `localhost`가 아니라 호스트의 사설 IP(`172.31.x`)나 docker0 게이트웨이(`172.17.0.1`)로 잡는 것이다.

"0.0.0.0으로 열면 외부에 노출되는 거 아니냐"는 걱정이 자연히 따라오는데, 이건 보안 그룹(SG)에서 9000 인바운드를 그냥 안 열어 두는 것으로 해결된다. kind에서 호스트로 가는 트래픽은 Docker 내부 브리지를 타서 SG의 적용을 받지 않기 때문이다. 결과적으로 안에서는 닿고 밖에서는 막힌다.

목적지 IP는 둘 중 무엇을 써도 실측으로 동작했고, 트레이드오프만 다르다.

| 후보 | 정체 | 장점 / 단점 |
|---|---|---|
| `172.31.61.157` (사설 IP) | 호스트의 VPC 주소 | 명확함("이 EC2") / 인스턴스 교체 시 바뀜 → URL 갱신 필요 |
| `172.17.0.1` (docker0 게이트웨이) | 컨테이너에서 본 호스트 | 인스턴스 교체에 불변 / 덜 직관적·Docker 재설정 시 바뀔 수 있음 |

다만 Pod 주소(예 `172.18.0.2`)나 kind 게이트웨이(`172.18.0.1`)는 목적지로 쓰면 안 된다. 이건 "호스트로 나가는 문"이 아니라 보낸 쪽 자신의 주소라, 또 허공으로 사라진다.

---

## 5. 공유 시크릿 하나가 전부인 인증

POST가 호스트 9000번에 도착했다고 끝이 아니다. webhook은 아무 POST에나 dbt를 돌리지 않는다. 4장 `hooks.yaml`의 `trigger-rule.match`가 문지기다. URL의 `?token=<값>`이 `hooks.yaml`에 적힌 값과 정확히 같을 때만 커맨드를 실행한다. 이 토큰이 webhook의 유일한 인증 장치다. 값은 `openssl rand -hex 32`로 만들었다.

헷갈리기 쉬운 건, 이 시크릿이 "빌드를 트리거할 권한"일 뿐 DB 접근 권한은 아니라는 점이다. 시스템에는 성격이 다른 시크릿 셋이 따로 산다.

| 시크릿 | 위치 | 용도 |
|---|---|---|
| webhook token | `/etc/webhook/hooks.yaml` | 빌드 트리거 인증 |
| DB 비번 (`DBT_PASSWORD`) | `/opt/<dbt-repo>/dbt/.env` | dbt → 웨어하우스 RDS 접속 |
| GitHub Deploy Key | `~ec2-user/.ssh/id_ed25519` | `git pull` 인증 (레포 1개 한정 read-only) |

토큰이 새도 공격자가 할 수 있는 건 dbt 빌드를 트리거하는 것까지다(그것도 막아야 하지만). DB가 직접 열리진 않는다. 권한을 이렇게 잘게 쪼개 두면 사고가 나도 폭발 반경이 줄어든다. Deploy Key를 레포 하나에만 read-only로 묶어 둔 것도 같은 생각이다. 그 키가 새도 다른 레포나 개인 계정엔 손이 닿지 않는다.

로테이트는 새 값 생성 → `hooks.yaml` 치환 → webhook 재시작 → Airbyte URL 갱신 → curl 검증 순서로 한다. EC2와 Airbyte 양쪽 값이 동시에 맞아야 트리거가 살아 있으니, 둘 중 하나만 바꾸면 그 순간부터 모든 트리거가 조용히 막힌다는 걸 잊으면 안 된다.

---

## 6. run-dbt.sh가 실제로 하는 일

토큰 검증을 통과하면 드디어 `run-dbt.sh`가 돈다. 이게 트리거 체인의 실질적인 일꾼인데, dbt를 곧장 부르지 않고 안전하게 부르기 위한 준비를 차례로 밟는다.

```
run-dbt.sh
  ① flock              : 이미 빌드가 돌고 있으면 겹치지 않게 직렬화
  ② git pull --ff-only : 최신 dbt 코드를 받아 옴 (= 코드 배포)
  ③ dbt-exec.sh 위임   : 컨테이너 보장 후 dbt 실행
```

**flock부터.** Airbyte sync가 짧은 간격으로 연달아 성공하면 POST도 연달아 온다. 그러면 dbt가 동시에 두 번 돌 수 있고, 같은 웨어하우스의 같은 테이블을 두 빌드가 같이 쓰면 충돌한다. 그래서 스크립트 맨 앞에 `flock` 한 줄로 직렬화한다. 이미 도는 중이면 새 트리거는 조용히 skip된다. 그러니 로그에 `[run-dbt] 이미 실행 중 — skip`이 찍혀도 고장이 아니다. 앞 빌드가 끝난 뒤 다시 트리거하면 된다.

**그다음이 git pull인데, 이 발상이 이 시스템에서 제일 영리한 부분이다.** 매 트리거마다 `git pull --ff-only`로 최신 dbt 코드를 받는다. 즉 dbt 코드의 배포 행위 자체가 `git pull`이고, 별도 빌드·배포 파이프라인이 없다. 모델 SQL을 고치고 싶으면 EC2에 SSH로 들어갈 필요 없이 pipeline 브랜치에 push만 하면 된다. 다음 트리거의 `git pull`이 알아서 가져온다. (왜 이게 되는지는 다음 장에서.)

대신 여기에 이 시스템 최대의 함정이 숨어 있다. **EC2의 git checkout에서 추적 파일을 직접 고치면 안 된다.** EC2에서 vim으로 모델을 직접 손대면 로컬 수정이 생기고, 다음 `git pull --ff-only`가 충돌로 실패하면서 `run-dbt.sh`가 거기서 죽는다. 그 뒤로는 트리거는 오는데 빌드는 안 도는, 데이터만 조용히 멈추는 상태가 된다. 증상이 미묘해서 한참 못 알아챈다. 처방은 `cd /opt/<dbt-repo> && git status`로 수정분을 확인하고 `git checkout -- .`로 버린 뒤 다시 트리거하는 것이다. 교훈은 하나다. 수정은 늘 로컬에서 고쳐 push하는 경로로. EC2에서 직접 만져도 되는 파일은 git 밖에 있는 것들(`.env`, `hooks.yaml`, systemd 유닛, `profiles.yml`)뿐이다. 그리고 push는 반드시 pipeline 브랜치로 — checkout이 `--single-branch pipeline`이라 main에 push한 건 EC2에 영영 오지 않는다.

**마지막으로** `run-dbt.sh`는 `dbt-exec.sh`에 실행을 넘긴다. 이쪽이 `docker compose up -d`로 컨테이너가 떠 있는지 보장한 뒤, 상주 컨테이너 안 dbt에 `exec`로 명령을 꽂고, dbt가 끝날 때까지 살아서 로그를 `tee`로 날짜별 파일에 남긴다.

---

## 7. "build"라는 말의 세 가지 뜻

dbt 실행 단계를 제대로 보려면, 이 시스템에서 "build"가 서로 다른 세 가지를 가리킨다는 걸 먼저 갈라 둬야 한다. 나는 초기에 이 셋을 뭉뚱그렸다가 한참 헷갈렸다.

| "build"의 정체 | 무슨 뜻 | run-dbt.sh가 매번 하나? |
|---|---|---|
| docker 이미지 빌드 | 런타임 굽기 (`docker compose build`) | ✗ (이미지가 아예 없을 때 최초 1회만) |
| 컨테이너 기동 | `docker compose up -d` (보장 단계) | △ 떠 있으면 no-op(~0.5초), 죽었을 때만 |
| `dbt build` / `dbt run` | 웨어하우스에서 SQL 실행 | ✓ 이게 본업 |

매 트리거의 본업은 세 번째뿐이다. 첫 번째 이미지 빌드는 dbt 버전(1.9.8)을 동결해 둬서 거의 안 일어나고, 두 번째 컨테이너 기동은 이미 떠 있으면 사실상 즉시 끝난다.

이 구분이 왜 중요하냐면, "SQL을 고쳤다"와 "런타임을 고쳤다"가 완전히 다른 일이기 때문이다. 핵심은 컨테이너에 git checkout 전체가 바인드 마운트되어 있다는 데 있다. 컨테이너는 "python + dbt 1.9.8 + dbt_packages"라는 실행기 껍데기일 뿐이고, 실제 SQL 코드는 컨테이너 밖(호스트의 git checkout)에 있고 마운트로만 들여다본다. 그래서 코드의 수명주기(git push/pull)와 런타임의 수명주기(이미지/컨테이너)가 분리된다.

악보와 연주자로 보면 쉽다. 컨테이너가 연주자, SQL 코드가 악보다. 곡을 바꾸려고 연주자를 새로 교육시킬 필요는 없다. 악보만 갈아 끼우면(`git pull`) 같은 연주자가 새 곡을 친다. 연주자 자체를 바꿔야 할 때, 그러니까 dbt 버전을 올릴 때만 이미지를 다시 굽는다. 덕분에 변경별 작업량이 깔끔하게 갈린다.

| 변경 대상 | 할 일 | 이미지 재빌드 |
|---|---|---|
| 모델 SQL · schema.yml · seeds | pipeline 브랜치에 push만 | 불필요 |
| packages.yml (dbt_utils 등) | pull 후 `dbt deps` 1회 | 불필요 |
| Dockerfile (dbt 버전) | `docker compose build` + recreate | 유일한 재빌드 케이스 |

사실 처음엔 매 트리거마다 `docker run --rm`으로 컨테이너를 띄웠다 버리는 ephemeral 방식이었다. 깔끔해 보였는데 두 가지가 발목을 잡았다. 하나는 `--rm`이 종료할 때 로그까지 같이 지워서, dbt가 실패해도 사후에 들여다볼 게 안 남았다는 것. 다른 하나는 코드가 이미지에 동결돼서 SQL 한 줄 고칠 때마다 이미지를 다시 굽고 재배포해야 했다는 것. 상주 + 마운트로 바꾸니 둘 다 풀렸다. 로그는 호스트에 `tee`로 남고, 코드 배포는 `git pull`로 끝난다. 덤으로 dbt의 partial parse 캐시가 컨테이너 안에 유지돼서 매번 전체를 다시 파싱하는 일도 없어졌다.

한 가지 더. sync 훅은 `dbt run`(모델만)을 쓰고, 테스트를 포함한 전체 `dbt build`는 따로 systemd 타이머(하루 1회)가 맡는다. `dbt build`는 테스트가 실패하면 하류 마트를 SKIP시키는 게이트라, 그걸 sync 주기마다 두면 데이터 품질 이슈 하나가 비용 대시보드 신선도를 매번 막아 버린다. 그래서 신선도(자주, `run`)와 품질 검증(하루 1회, `build`)을 나눴다. 다만 이 전체 build 타이머는 글을 쓰는 시점(2026-06)엔 의도적으로 보류 중이라, 지금 자동 갱신되는 건 sync 훅의 95개뿐이고 셀렉터 밖 모델·테스트는 수동 `run-dbt.sh`(무인자)가 유일한 갱신 경로다.

---

## 8. HTTP 200은 성공이 아니다

트리거를 처음 검증할 때 제일 크게 데었다. `curl`로 훅을 직접 때렸더니 `200 OK`가 떴다. "됐다" 싶었는데 데이터는 갱신되지 않았다.

```bash
curl -X POST "http://127.0.0.1:9000/hooks/dbt-on-sync?token=<시크릿>"
# 200이 떠도 빌드 성공이 아니다
```

생각해 보면 당연하다. webhook은 커맨드를 실행시킨 그 순간 응답을 돌려준다. `dbt run`은 몇 분씩 걸리는데 그게 끝날 때까지 HTTP 연결을 붙들고 있을 순 없다. 그래서 200은 "훅이 매칭됐고 커맨드 실행을 시작했다"는 뜻이지, "빌드가 성공했다"가 아니다. 초인종이 "딩동" 울렸다고 손님이 들어와 일까지 마친 건 아닌 것과 같다.

그래서 판정은 늘 로그로 한다. 로그가 두 종류인데 쓰임이 다르다.

```bash
journalctl -u webhook -f                  # 수신·실행 이벤트 (커맨드 출력은 종료 후 일괄)
tail -f /var/log/dbt/dbt_$(date +%F).log  # 빌드 진행 실시간 (tee)
```

webhook의 journald엔 커맨드 출력이 종료 후 한꺼번에 찍힌다. 그래서 빌드가 도는 동안 실시간으로 보려면 `run-dbt.sh`가 `tee`로 남기는 날짜별 파일을 봐야 한다. "출력이 실시간으로 안 보인다"고 당황했던 것도 이 차이 때문이었다.

여기서 한발 더 나가서, 200이 떠도 결과를 모른다는 사각지대를 메우려고 `run-dbt.sh`에 Slack 완료 알림을 붙였다. dbt가 끝나면 성공/실패를 요약줄·소요시간과 함께 Slack으로 쏜다. webhook은 트리거까지만 알지만 `run-dbt.sh`는 dbt가 끝날 때까지 살아 있으니, 완료를 알리기엔 여기가 맞는 자리다. `SLACK_WEBHOOK_URL`이 설정 안 돼 있으면 아무 일도 안 하므로, 안 쓰던 환경의 동작을 깨지도 않는다.

---

## 9. 일부러 안 한 것 — 자동 재시도

마지막으로, 이 시스템이 일부러 하지 않는 걸 분명히 해 둔다. Dagster를 안 쓴 대가이자 동시에 의도한 단순함이다.

dbt가 실패하면 그냥 실패로 끝난다. 자동으로 다시 돌지 않는다. 복구는 사람이 `run-dbt.sh`를 손으로 다시 돌리거나 Airbyte UI에서 sync를 재실행하는 식이다. 재시도 로직은 2장에서 말한 간헐 운영 조건에서는 또 하나의 부채라, 의식적으로 뺐다.

Slack 알림으로도 못 잡는 사각지대가 있다. 이건 숨기지 않고 적어 두는 게 맞다. 트리거 자체가 안 온 경우(webhook 다운이나 토큰 불일치)는 `run-dbt.sh`가 실행조차 안 되니 스크립트 안의 알림으로는 못 잡는다. dbt가 멈춰서 끝나지 않는 경우도 스크립트가 안 끝나니 완료 알림이 안 나간다. 이 둘은 Airbyte UI의 Sync Failure 알림으로 보완하고, 별도 watchdog은 두지 않았다. watchdog 자체가 또 유지보수 대상이 되기 때문이다. 완벽한 자동 감시 대신 단순함과 인지된 사각지대를 택한 것이고, 이건 누락이 아니라 트레이드오프다.

---

## 정리

"sync가 끝난 순간 누가 dbt를 깨우나"에서 시작해, POST 한 방이 dbt까지 가닿는 경로를 따라가 봤다. 돌아보면 핵심은 몇 줄로 추려진다.

두 명이 가끔 들여다보는 파이프라인에는 오케스트레이터 한 채보다 초인종 하나가 맞았다. 그래서 adnanh/webhook(Go 단일 바이너리)만 systemd로 상주시키고 커스텀 코드는 두지 않았다. 그 과정에서 발을 헛디딘 곳은 대개 "겉으로 멀쩡해 보이는데 실은 아닌" 지점들이었다. `localhost` 바인딩은 kind Pod 안에선 호스트가 아니었고(37편의 그 kind가 여기서 발목을 잡았다), `git pull --ff-only`는 EC2 checkout을 직접 건드리는 순간 모든 트리거를 침묵 속에 멈췄고, HTTP 200은 트리거가 걸렸다는 뜻이지 빌드가 됐다는 뜻이 아니었다. 그래서 판정은 로그로, 완료 통지는 Slack으로, 못 잡는 건 Airbyte Sync Failure로 메웠다.

코드 배포가 곧 `git pull`이고, 컨테이너는 악보를 바꿔 끼우는 연주자라는 구조 덕에, SQL 수정은 push 한 번으로 끝난다. 화려한 오케스트레이터를 세우는 대신 이미 켜진 서버에 초인종 하나를 단 이 가벼운 구성이, 우리 상황에는 오히려 더 튼튼했다.

### 자주 쓰는 명령어

```bash
# 수동 트리거 (HTTP로 — 셀렉터 빌드)
curl -X POST "http://127.0.0.1:9000/hooks/dbt-on-sync?token=<시크릿>"

# 수동 실행 (HTTP 없이 같은 진입점)
run-dbt.sh                          # 전체 dbt build
run-dbt.sh run --select +모델명      # 임의 셀렉터

# 로그로 진짜 결과 판정
journalctl -u webhook -f                    # 수신·실행 이벤트
tail -f /var/log/dbt/dbt_$(date +%F).log    # 빌드 진행 실시간

# hooks.yaml 수정 반영 (무중단)
sudo systemctl kill -s USR1 webhook
journalctl -u webhook | tail -5             # "found N hook(s)"로 반영 확인

# git pull 충돌로 트리거가 멈췄을 때 복구
cd /opt/<dbt-repo> && git status             # 로컬 수정분 확인
git checkout -- .                            # 버리고 재트리거
```
