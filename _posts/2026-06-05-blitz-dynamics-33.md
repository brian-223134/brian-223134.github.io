---
title: "[인턴] Airbyte OSS로 CDC 파이프라인 옮기기 (2) abctl로 EC2에 올리며 만난 설치 함정들"
excerpt: "abctl(kind) + 외부 RDS 메타DB + S3 스토리지 구성으로 Airbyte OSS를 EC2에 올리는 과정. Helm 차트 V2 키 규칙, bootloader/temporal/IMDS/리소스 부족까지 실제로 밟은 함정과 그 해결을 순서대로 정리한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Airbyte, abctl, Kubernetes, AWS, Helm]

permalink: /intern/company/blitz-dynamics/33/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

## 들어가며

지난 글에서는 Fivetran이 하던 MySQL → Postgres CDC 미러링을 Airbyte OSS로 재현하기 위한 밑그림을 잡았다. 소스와 destination, 네트워크, soft delete 같은 제약까지 설계 단계에서 정리해 두었으니, 이번 편은 그 설계를 실제로 EC2 위에 올린 이야기다.

Airbyte OSS는 `abctl`이라는 CLI로 설치한다. 직접 쿠버네티스 클러스터를 운영하는 대신, abctl이 **EC2 위에 kind(도커 컨테이너 안에서 도는 쿠버네티스)** 를 띄우고 그 안에 Airbyte 플랫폼 전체를 올려주는 방식이다. 개념만 들으면 "명령어 한 줄이면 끝"처럼 들리고, 실제로 공식 가이드도 그렇게 적혀 있다.

결론부터 말하면, 한 번에 안 됐다. bootloader가 시작도 못 하고 죽고, temporal이 권한 때문에 줄줄이 쓰러지고, S3 자격증명을 못 받아오고, 마지막엔 리소스가 모자라 파드가 아예 스케줄조차 안 됐다. 막히는 지점마다 로그를 파고 원인을 찾아 하나씩 걷어낸 기록이 이 글이다.

**이 편의 진짜 교훈은 마지막에 가서야 또렷해지는데, 메타데이터와 스토리지를 외부로 빼두었기 때문에 이 모든 삽질을 install/uninstall을 반복하며 마음 편히 할 수 있었다는 점이다.** 그 이야기부터 시작한다.

## 1. 구성 개요 — 무엇을 외부로 뺐는가

설치한 버전과 인스턴스부터 정리한다.

| 항목 | 값 |
|---|---|
| abctl | v0.30.4 |
| Helm 차트 / App | 2.1.0 / 2.1.0 |
| EC2 (최종) | m7i.2xlarge (8 vCPU / 32 GB), AL2023 |
| region | ap-northeast-2 |

EC2는 처음엔 xlarge(4 vCPU)로 시작했는데, 뒤에서 리소스 부족에 걸려 2xlarge로 스케일업했다. 이 사연은 뒤에서 따로 다룬다.

여기서 가장 중요한 결정은 **메타데이터와 스토리지를 Airbyte 내장 컴포넌트가 아니라 외부 AWS 리소스로 빼둔 것**이다. abctl은 기본적으로 내장 Postgres와 MinIO를 같이 띄우는데, 나는 그걸 쓰지 않았다.

- **메타데이터:** 내장 Postgres 대신 외부 RDS의 `airbyte_meta` DB로.
- **스토리지(로그 / state / 출력):** 내장 MinIO 대신 S3 버킷 `<S3_BUCKET>` 하나로.

이렇게 외부화하면 abctl을 uninstall했다 다시 install해도 메타데이터와 CDC 커서가 그대로 보존된다. 설치 단계에서 수십 번을 갈아엎을 게 뻔한데, 그때마다 커넥터 설정과 동기화 커서가 날아간다면 디버깅 자체가 불가능했을 것이다. 그래서 이 구조가 핵심 이점이었다.

S3 버킷 하나에 `log`, `state`, `workloadOutput`, `activityPayload`를 전부 담았다. 이 중에서 **`state`가 CDC 커서**다. 절대 지우면 안 되는 데이터고, 이걸 S3에 둔 덕분에 클러스터를 통째로 날려도 커서가 살아남는다.

EC2가 S3에 접근하는 방식은 액세스 키가 아니라 **IAM 역할(인스턴스 프로파일)** 로 했다. 이 선택이 나중에 IMDS hop limit 함정으로 돌아오는데, 그건 4번에서.

## 2. OS 사전 준비

설치 명령을 치기 전에 EC2 쪽에서 먼저 깔아둔 것들이다. 별것 아닌 것 같아도 하나라도 빠지면 설치 도중에 묘하게 막힌다.

| 준비 | 메모 |
|---|---|
| Docker 25.x | `ec2-user`를 docker 그룹에 추가. 안 하면 권한 에러로 재로그인해야 한다 |
| sysctl inotify | `fs.inotify.max_user_watches=524288`, `fs.inotify.max_user_instances=512` |
| Swap 16GB | `Text file busy`가 나면 이미 swapfile이 있다는 뜻 |
| 클라이언트 | kubectl, psql, mysql |

inotify 한도를 올린 건 kind가 파일 워처를 굉장히 많이 쓰기 때문이다. 기본값으로 두면 파드들이 watch를 못 잡아 이상하게 죽는다.

swap에서 한 번 헷갈렸는데, swapfile을 새로 만들려다 `Text file busy` 에러를 만났다. 알고 보니 **이미 swapfile이 존재한다는 신호**였고, 새로 만들 필요 없이 `swapon`만 걸어주면 됐다.

## 3. 설치 명령과 secret/values 구성

준비가 끝나면 EC2 홈에 `secret.yaml`과 `values.yaml`을 두고 한 줄로 설치한다.

```bash
abctl local install --secret secret.yaml --values values.yaml
```

여기서 의도적으로 `--host`를 빼고 설치했다. `--host` 없이 깔면 Airbyte는 **localhost 전용**으로 뜬다. 외부 IP로 직접 접속하는 길이 막히는 대신, UI는 반드시 SSH 터널을 거쳐야 하는 구조가 된다. 보안상 이 편이 낫다고 봤고, 터널 운영은 4편 런북에서 자세히 다룬다.

### secret.yaml

```yaml
stringData:
  database-user: airbyte
  database-password: "<RDS_PASSWORD>"
  CONFIG_DATABASE_REPLICA_USER: airbyte          # 차트 2.1.0이 요구
  CONFIG_DATABASE_REPLICA_PASSWORD: "<RDS_PASSWORD>"
```

처음엔 위쪽 두 줄(`database-user`, `database-password`)만 넣었다. 그런데 차트 2.1.0은 **read replica 자격증명 키**까지 secret에서 찾는다. 우리 RDS엔 read replica가 따로 없으니, primary와 똑같은 값을 replica 키에 넣어줬다. 어차피 같은 곳을 가리키니 무해하다. 이게 첫 번째 함정으로 이어지는데, 트러블슈팅에서 다시 보자.

### values.yaml

전체 덤프는 길어서 핵심만 옮긴다.

- `global.database`: type `external`, host `<TEST_PG_HOST>`, port `5432`, **`name: "airbyte_meta"`**.
- `global.storage`: type `s3`, 버킷 `<S3_BUCKET>`, `authenticationType: instanceProfile`.

여기서 가장 강조하고 싶은 건 **차트 V2의 키가 전부 camelCase라는 점**이다. V1 시절의 하이픈 표기(`workload-launcher` 같은)를 그대로 쓰면, 에러도 안 나고 그냥 **조용히 무시된다.** 이게 사람을 제일 미치게 하는 종류의 함정이다. 분명히 값을 줬는데 안 먹는데, 왜 안 먹는지 로그에 안 나온다. 올바른 키는 `workloadLauncher`다.

설치를 정리하다가 추가로 알게 된 두 가지가 있었다.

1. `postgresql.enabled: false`와 `minio.enabled: false`를 명시하지 않으면, 메타를 RDS로 보내는데도 **내장 Postgres와 MinIO 파드가 같이 뜰 수 있다.** 실제 데이터는 외부로 가니 동작엔 무해하지만, 안 그래도 모자란 자원을 낭비한다.
2. `workloadLauncher.extraEnv`에 리소스 캡을 걸어도, 정작 **replication 파드에는 그 값이 안 먹는다.** 이건 3편(버그 편)에서 본격적으로 파는데, GitHub issue [#68162](https://github.com/airbytehq/airbyte/issues/68162)와 직접 얽혀 있다.

둘 다 다음 재설치 때 정리 대상으로 메모해 뒀다.

## 4. 트러블슈팅 — 실제로 밟은 순서대로

여기가 이 글의 본론이다. 증상 → 원인 → 해결 순서로, 내가 실제로 막힌 순서 그대로 적는다.

### (1) bootloader가 replica 키를 못 찾는다

```text
couldn't find key CONFIG_DATABASE_REPLICA_USER
```

설치하자마자 bootloader 파드가 이걸 뱉고 죽었다. 앞서 적은 대로 차트 2.1.0이 secret에서 replica 자격증명을 찾는데, 내 secret엔 그 키가 없었던 것. `secret.yaml`에 `CONFIG_DATABASE_REPLICA_USER`/`PASSWORD`를 primary와 같은 값으로 추가하니 통과했다.

### (2) bootloader: 존재하지 않는 DB `db-airbyte`

```text
FATAL: database "db-airbyte" does not exist
```

이번엔 bootloader가 `db-airbyte`라는, 내가 만든 적도 없는 DB를 찾았다. 내 RDS에 있는 메타 DB는 `airbyte_meta`인데 말이다.

원인은 또 키 이름이었다. values에서 DB 이름을 `database:` 키로 줬더니 **그 키가 무시되고 차트 기본값 `db-airbyte`가 붙은 것**이다. V2의 올바른 키는 `global.database.name`이다.

```yaml
global:
  database:
    name: "airbyte_meta"   # database: 가 아니라 name:
```

이렇게 고치니 해결됐다. 1번과 2번을 연달아 겪고 나서야 "이 차트는 camelCase 정식 키가 아니면 에러도 없이 무시한다"는 감을 확실히 잡았다.

### (3) temporal: 데이터베이스를 만들 권한이 없다

```text
pq: permission denied to create database
```

이게 제일 헷갈렸다. 겉으로는 server와 launcher 파드가 `connection refused`로 줄줄이 죽는 것처럼 보였는데, 한참 로그를 거슬러 올라가 보니 **진짜 원인은 temporal이었다.**

Airbyte 내부의 temporal은 런타임에 자기가 쓸 DB들을 직접 만든다. 그런데 `airbyte` role에 `CREATEDB` 권한이 없으니 DB 생성이 막혔고, temporal이 못 뜨자 거기 의존하던 server·launcher가 connection refused로 연쇄적으로 쓰러진 거였다.

master 권한으로 들어가서 권한을 한 줄 줬더니 풀렸다.

```sql
ALTER ROLE airbyte CREATEDB;
```

지난 글의 설계에서 "`airbyte` role에 CREATEDB가 필요하다"고 미리 적어둔 게 바로 이 함정 때문이다. 그땐 예고로만 적었는데, 실제로 여기서 밟았다.

### (4) destination 파드가 S3 자격증명을 못 받는다 (IMDS)

```text
Unable to persist the job output, check the document store credentials
```

설치는 됐는데, 막상 동기화를 돌리니 destination 파드가 출력을 S3에 못 쓰고 이 에러를 냈다. "document store credentials를 확인하라"는데, 나는 액세스 키를 쓰는 게 아니라 인스턴스 프로파일을 쓰고 있었다. 키를 안 쓰는데 키를 확인하라니.

원인은 IMDS였다. EC2의 인스턴스 프로파일 자격증명은 **IMDS(인스턴스 메타데이터 서비스)** 를 통해 받아오는데, 우리 파드는 `EC2 → 도커 → kind → 파드`로 한 홉 더 깊은 곳에 있다. 기본 IMDS hop limit이 너무 짧아 파드에서 메타데이터까지 패킷이 못 닿았던 것이다.

hop limit을 늘려주고, 관련 deployment를 rollout restart 했다.

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id <EC2_INSTANCE_ID> \
  --http-put-response-hop-limit 3 \
  --http-tokens required
```

검증은 단순하게 했다. 클러스터 안에 임시 파드를 하나 띄워서 `aws s3 ls`가 도는지 확인했다. 파드 안에서 S3가 보이면 자격증명이 제대로 전달된 거다.

### (5) 리소스 부족 — replication 파드가 스케줄조차 안 된다

```text
ResourceConstraintException: Unable to start the REPLICATION pod
Pending: Insufficient cpu
```

여기서 제일 오래 막혔다. replication 파드는 단일 파드 안에서 cpu를 합쳐 4를 요청한다(orchestrator 2 + source 1 + destination 1, init 컨테이너가 별도로 2). 그런데 xlarge(4 vCPU)의 allocatable은 시스템 오버헤드를 빼면 약 3.8밖에 안 돼서, 4를 요구하는 파드는 영원히 `Pending`에 걸린다.

처음엔 당연히 "helm으로 리소스 요청을 낮추면 되겠지" 하고 한참 매달렸다. 그런데 **그 시도가 전부 실패했다.** 위에서 말한 V2 camelCase 키 규칙과 값 우선순위 문제 때문에, 내가 설정한 리소스 캡이 replication 파드에는 끝내 적용되지 않았다. (이 싸움의 전말은 3편에서 GitHub issue와 함께 자세히.)

리소스를 낮추는 길이 막히니, 결국 정공법으로 인스턴스를 키웠다. **m7i.2xlarge(8 vCPU)로 스케일업.**

```bash
# stop → modify → start
aws ec2 stop-instances  --instance-ids <EC2_INSTANCE_ID>
aws ec2 modify-instance-attribute \
  --instance-id <EC2_INSTANCE_ID> \
  --instance-type m7i.2xlarge
aws ec2 start-instances --instance-ids <EC2_INSTANCE_ID>
```

여기서 다시 한번 외부화 구조의 덕을 봤다. 메타와 스토리지를 RDS·S3로 빼뒀으니 인스턴스를 stop했다 start해도 **데이터 손실이 없었고, 탄력 IP도 유지**됐다. 만약 내장 Postgres/MinIO를 썼다면 인스턴스 스토어가 날아가며 전부 잃었을 것이다.

### (6) Init:Error — 플랫폼이 Ready 되기 전에 sync가 들어옴

```text
Init:Error — workload-api-server-svc:8001 connection refused
```

인스턴스를 재시작한 직후에 만난 타이밍 문제다. 재시작하면 플랫폼 파드들이 다시 뜨는데, 특히 `workload-api-server`가 Ready가 되기 전에 동기화가 들어가면 init 컨테이너가 connection refused로 exit해버린다.

이건 버그라기보다 순서 문제라, 대응도 단순했다.

1. 플랫폼 파드가 **전부 Running**인지 먼저 확인한다.
2. 어정쩡하게 떠 있던 `replication-job-*` 파드를 삭제한다.
3. sync를 다시 돌린다.

### (7) 재설치 시 `PG_VERSION: permission denied`

```text
... PG_VERSION: permission denied
```

재설치 과정에서 만난 잔재 문제다. 앞선 시도 중 잠깐 내장 Postgres가 떴던 적이 있었는데, 그때 만들어진 볼륨 디렉터리가 **root 소유로 남아 있었다.** 우리는 메타를 외부 RDS로 쓰니 이 데이터는 애초에 불필요하다. 해당 볼륨 디렉터리를 삭제하고 다시 설치하니 깨끗하게 풀렸다.

## 5. 자격증명은 매번 새로 발급된다

설치가 끝나면 UI/API에 쓸 자격증명을 이 명령으로 조회한다.

```bash
abctl local credentials
```

여기서 꼭 기억해야 할 점. **재설치할 때마다 password·client-id·client-secret이 새로 발급된다.** 그래서 어딘가 문서에 적어둔 값을 믿고 쓰면 안 되고, 매번 이 명령으로 새로 조회해야 한다. 설치를 수십 번 반복하는 동안 옛날 자격증명을 붙여 넣고 "왜 인증이 안 되지" 하며 헤맨 적이 있어서, 이건 몸으로 배웠다.

한 가지 더. 이 명령의 출력에는 **ANSI 색상코드(ESC 문자)가 섞여 들어온다.** 사람 눈엔 안 보이지만, 출력을 그대로 스크립트에 넘기면 JSON 파서가 `CTRL-CHAR code 27` 같은 400 에러를 낸다. 제어문자가 값 안에 끼어 있어서다. 그래서 항상 한 단계 거쳐 제어문자를 벗겨낸다.

```bash
abctl local credentials | sed 's/\x1b\[[0-9;]*m//g'
```

이 `sed` 트릭은 4편 런북에서 API 자동화를 짤 때 또 등장한다.

## 6. UI 접속과 React #185 (다음 편 예고)

마지막으로 짧게. 앞서 `--host` 없이 설치했다고 했으니, UI는 외부 IP로 직접 들어가면 안 되고 **SSH 터널을 뚫은 뒤 `http://localhost:8000`** 으로만 접속해야 한다.

이걸 모르고 외부 IP로 들어가면 화면이 곧장 죽는다. `/api/oauth/access_token`이 400을 뱉고, 프론트가 그걸 무한 재시도하다가 결국 **React #185("Sorry, something went wrong")** 로 흰 화면이 된다. 백엔드는 멀쩡한데 화면만 죽는 거라 한참 백엔드를 의심하며 헤맸는데, 알고 보니 이 버전(2.1.0)의 알려진 프론트 버그였다.

대응의 핵심은 "시크릿 창 + 딥링크로 직접 진입, 그래도 안 되면 UI를 포기하고 API로 작업"이다. 이 부분은 4편 런북에서 본격적으로 다룬다.

## 마치며

설치 한 줄이면 끝날 줄 알았던 게 bootloader → temporal → IMDS → 리소스 부족까지, 막힐 수 있는 거의 모든 지점에서 막혔다. 그래도 끝까지 install/uninstall을 두려움 없이 반복할 수 있었던 건, 처음에 메타데이터를 RDS로, 스토리지를 S3로 빼두었기 때문이다.

> 디버깅이 길어질 게 뻔한 시스템일수록, **상태(state)를 인프라 밖으로 먼저 빼라.** 그러면 본체는 언제든 갈아엎을 수 있는 소모품이 된다. 이번 설치에서 다시 확인한 결론이었다.

설치는 이렇게 끝났다. 그런데 정작 데이터를 옮기기 시작하니 더 깊은 함정이 기다리고 있었다. 다음 편에서는 **"succeeded라고 떠 있는데 destination엔 데이터가 없는"** 가짜 성공과, 그 뒤에 숨어 있던 세 개의 버그를 파고든다.
