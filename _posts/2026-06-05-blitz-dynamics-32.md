---
title: "[인턴] Fivetran을 떠나기로 했다 — Airbyte OSS로 CDC 파이프라인 옮기기 (1) 배경과 설계"
excerpt: "무료로 쓰던 Fivetran을 Airbyte OSS(abctl)로 대체하기 위한 사전 테스트. 왜 옮기려 했고, MySQL→Postgres CDC 미러링을 어떻게 재현하려 했는지에 대한 설계 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 데이터엔지니어링, Airbyte, Fivetran, CDC, ELT]

permalink: /intern/company/blitz-dynamics/32/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

## 들어가며

회사 데이터 파이프라인에서 EL(Extract-Load) 구간은 그동안 **Fivetran**이 맡고 있었다. 운영 MySQL의 두 데이터베이스(`db_a`, `db_b`)를 분석용 Postgres로 CDC 미러링하는 구조였고, destination에는 각각 `raw_db_a`, `raw_db_b` 스키마로 떨어졌다. 분석 쪽 dbt 모델은 전부 이 `raw_*` 스키마를 입력으로 삼고 있었다. 다시 말해 Fivetran은 우리 분석 파이프라인의 맨 앞단을 조용히 받쳐주는 기반이었다.

문제는 우리가 Fivetran을 사실상 무료로 써오고 있었다는 점이었다. 그런데 이 무료 사용을 앞으로도 계속 기대하긴 어려워졌다. 비용과 한도 이슈가 겹치면서, "지금처럼 공짜로 굴러가는 게 언제까지 갈지 모른다"는 분위기가 생겼다. 그래서 받은 과제가 **오픈소스인 Airbyte OSS로 같은 파이프라인을 재현할 수 있는지** 사전 테스트, 즉 PoC를 해보는 일이었다.

이 시리즈는 그 PoC의 기록이다. 설계부터 시작해서 EC2에 Airbyte를 올리는 과정, Airbyte를 믿었다가 데인 버그들, 운영 런북, 마지막엔 dbt 이관까지를 다섯 편으로 나눠 정리했다. 그 첫 편인 이 글은 **"왜, 무엇을, 어떻게 옮기려 했는가"** 의 밑그림이다. 코드는 거의 없고 대부분 설계 이야기다.

미리 말해두면, 이 단계까지의 설계는 꽤 깔끔했다. 정작 고생은 그다음부터 시작됐다.

## 1. 목표는 "그대로 재현"이었다

PoC의 목표는 의외로 단순했다. 새로운 걸 만들자는 게 아니라, **Fivetran이 하던 일을 Airbyte OSS로 똑같이 재현하는 것**이었다. MySQL의 `db_a`, `db_b` 두 DB를 Postgres로 미러링하되, destination 스키마 이름(`raw_db_a`, `raw_db_b`)과 동작을 최대한 동일하게 맞추는 게 기준이었다.

"최대한 동일하게"가 중요했다. 분석 쪽 dbt 모델은 이미 Fivetran이 만들어둔 스키마 구조와 컬럼 형태에 맞춰져 있었다. 그러니 입력단을 Airbyte로 갈아끼우더라도 그 아래는 가능한 한 손대지 않는 게 안전했다. 새 도구로 갈아탔는데 다운스트림이 죄다 깨진다면 그건 이관이 아니라 재구축이다.

**그래서 이 PoC의 성공 기준은 "Airbyte가 Fivetran과 구분되지 않게 동작하느냐"였다.** 화려한 기능이 아니라, 티 안 나게 똑같이 굴러가는 것.

## 2. 가장 까다로운 제약 — soft delete

재현 목표 중에서 제일 신경 쓰인 건 **soft delete**였다.

Fivetran은 소스에서 행이 삭제되더라도 destination에서 그 행을 실제로 지우지 않는다. 대신 삭제 플래그를 세워 보존한다. 이게 soft delete다. 분석 입장에서는 "예전에 있었다가 지워진 데이터"도 기록으로 남아 있어야 의미가 있는 경우가 많아서, 이 동작이 은근히 중요했다.

Airbyte에서 이걸 재현하려면 destination 쪽의 **CDC deletion mode를 Soft delete로** 둬야 했다. 이렇게 하면 소스에서 삭제된 행이 destination에서 사라지지 않고, `_ab_cdc_deleted_at`이라는 타임스탬프 플래그가 채워진 채로 남는다. 삭제 시각이 찍히는 셈이다.

> 이 차이를 맞추는 게 사실상 이관의 핵심이었다. deletion mode를 잘못 두면 행이 진짜로 지워지고, 그러면 Fivetran 시절 데이터와 모양이 달라진다.

sync mode는 전 스트림을 **Incremental | Append + Dedup**으로 잡았다. dedup을 쓰면 보통 중복이 정리되니까 "삭제 행도 같이 정리되는 거 아니냐" 싶었는데, soft delete 모드와 함께 쓰면 삭제된 행은 그대로 보존된다. 이 조합이 Fivetran의 동작에 가장 가까웠다.

주기는 처음엔 일부러 **Manual**로 뒀다. 검증이 끝나기 전까지는 자동으로 돌리고 싶지 않았기 때문이다. 검증이 끝나면 1시간 주기(또는 cron 30분)로 올릴 계획이었다. 참고로 Fivetran은 30분 주기로 돌고 있었다.

dbt는 같은 EC2에서 돌릴 예정이었고, sync가 성공하면 webhook으로 dbt를 트리거하는 구조를 머릿속에 그려뒀다. 다만 이 시점에는 아직 손대지 않았다. dbt 이관은 이 시리즈의 마지막 편에서 따로 다룬다.

## 3. 절대 어기면 안 되는 순서 하나

설계 단계에서 스스로 못박아둔 안전 규칙이 하나 있었다.

> **Fivetran의 `db_a`/`db_b` 커넥터를 먼저 중지한 다음에 Airbyte를 켠다.**

이유는 단순하다. 두 도구가 같은 `raw_*` 스키마에 동시에 쓰면 안 되기 때문이다. Fivetran과 Airbyte가 같은 destination에 동시에 손을 대면 어느 쪽이 쓴 건지, 어느 게 진실인지 알 수 없는 상태가 된다. 그래서 이건 협상의 여지가 없는 순서였다. PoC 검증 동안에는 destination을 따로 두고 갔지만, 실제로 같은 `raw_*`를 노릴 때는 반드시 Fivetran을 먼저 끄고 Airbyte를 켜야 한다.

## 4. 아키텍처 — 어디에 무엇을 두었나

전체 그림은 이랬다.

- **소스**: 운영 RDS MySQL(MySQL 8.0.44). 이건 **라이브 운영 DB**다. 지금 이 순간에도 Fivetran이 여기 붙어서 CDC를 하고 있다. 테스트용 Airbyte도 **같은** DB에 붙는데, 어디까지나 읽기로만 붙는다. 즉 소스에 추가되는 건 읽기 부하뿐이다. 운영 DB에 테스트 도구를 붙인다는 게 좀 부담스러웠지만, CDC는 binlog를 읽는 방식이라 쓰기를 건드리지 않는다는 점이 그나마 마음을 놓이게 했다.
- **destination**: PoC 전용으로 새로 띄운 테스트용 RDS Postgres. 운영 destination을 건드리지 않기 위해 일부러 별도로 만들었다. 스펙은 아래와 같다.

| 항목 | 값 |
|---|---|
| 엔진 | PostgreSQL 16.8 |
| 인스턴스 | db.t4g.large |
| 스토리지 | gp3 300GB, autoscale 500GB |
| 접근 | 비공개(퍼블릭 액세스 off) |
| 암호화 | 활성화 |
| 가용성 | Single-AZ |

- **네트워크**: Airbyte를 올린 EC2, 테스트 RDS Postgres, 소스 MySQL이 **모두 같은 VPC** 안에 있었다. 통신은 보안그룹끼리 참조(SG-to-SG)하는 방식으로 열었다. IP를 직접 화이트리스트하는 대신 "이 보안그룹에서 오는 트래픽은 허용" 식으로 잡는 방법이다. 구체적으로는 소스 MySQL의 보안그룹 인바운드 3306을 EC2의 보안그룹(`<SG_ID>`)에 대해 열어줬다. 이렇게 해두면 EC2 IP가 바뀌어도 규칙을 다시 손댈 필요가 없다.

## 5. 테스트 RDS 내부 구조

테스트용 Postgres 안에는 DB를 셋으로 나눠뒀다.

| DB | 용도 |
|---|---|
| `analytics` | 동기화 데이터 적재 (`raw_db_b`, `raw_db_a`, Airbyte 내부 스키마가 여기 생긴다) |
| `airbyte_meta` | Airbyte 플랫폼 메타데이터 |
| `postgres` | RDS 기본 관리용 |

여기서 한 가지 미리 알아둬야 할 함정이 있다. `airbyte`라는 role을 만들면서 `LOGIN` 권한과 함께 **`CREATEDB`** 권한을 줬다. 처음엔 이게 왜 필요한지 몰랐다. Airbyte 내부의 temporal 컴포넌트가 런타임에 **자기 DB를 직접 만들기** 때문인데, role에 `CREATEDB`가 없으면 설치가 `permission denied to create database`로 죽어버린다. 이 함정에 정확히 한 번 빠졌고, 자세한 이야기는 다음 편에서 한다.

## 6. 소스 MySQL의 CDC 사전조건

CDC는 소스가 준비돼 있지 않으면 시작조차 못 한다. 그래서 운영 MySQL이 CDC를 받아줄 상태인지 먼저 확인했다.

```sql
-- binlog가 켜져 있는지, 포맷이 ROW인지
SHOW VARIABLES LIKE 'log_bin';        -- ON
SHOW VARIABLES LIKE 'binlog_format';  -- ROW
```

- `log_bin = ON`, `binlog_format = ROW` — 둘 다 확인됐다. CDC가 row 단위 변경을 읽으려면 ROW 포맷이어야 한다.
- 전용 유저는 새로 만들 필요가 없었다. **Fivetran이 쓰던 replication 유저를 그대로 재사용**했다. 권한은 `SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.*`였고, host가 `%`로 열려 있어 EC2에서도 접속이 됐다. 이미 운영에서 CDC용으로 검증된 유저였으니, 새 유저를 만들어 권한 시비를 다시 겪는 것보다 이게 안전했다.
- 마지막으로 확인한 건 **binlog retention**이었다. CDC 커서는 binlog 위치를 기준으로 따라가는데, binlog가 너무 빨리 지워지면 커서가 가리키던 위치가 사라져서 sync가 끊긴다. 확인 결과 168시간, 즉 7일이었다.

> binlog retention이 7일이라는 건, sync를 적어도 7일에 한 번 이상은 돌려야 커서가 살아 있다는 뜻이다. 테스트 중에 Manual로 며칠씩 방치하면 커서를 잃을 수 있으니 이 숫자를 기억해둘 필요가 있었다.

## 7. destination namespace를 Fivetran 모양으로 맞추기

마지막 설계 디테일은 스키마 이름이었다.

Fivetran은 소스 DB `db_b`를 destination에서 `raw_db_b`로, `db_a`를 `raw_db_a`로 적재한다. 이 네이밍을 그대로 맞춰야 다운스트림 dbt가 안 깨진다. Airbyte에서는 connection의 **Destination Namespace를 커스텀 포맷**으로 둬서 해결했다.

```text
raw_${SOURCE_NAMESPACE}
```

이렇게 두면 소스 DB `db_b`는 destination에서 `raw_db_b` 스키마로, `db_a`는 같은 패턴으로 `raw_db_a`로 떨어진다. 소스 이름 앞에 `raw_`만 붙이는 단순한 규칙인데, 이 한 줄 설정으로 Fivetran의 스키마 네이밍을 그대로 흉내 낼 수 있었다.

## 마치며

여기까지가 PoC의 밑그림이다. 정리해보면 "Fivetran이 하던 MySQL→Postgres CDC 미러링을, soft delete까지 포함해 Airbyte OSS로 똑같이 재현한다"가 전부였고, 아키텍처도 같은 VPC 안에서 EC2·테스트 RDS·운영 MySQL을 SG-to-SG로 묶는 깔끔한 구조였다.

문제는, 설계가 깔끔하다고 구축과 운영까지 깔끔한 건 아니라는 점이었다. 실제로 손을 대기 시작하니 예상 못 한 함정이 줄줄이 나왔다. `CREATEDB` 권한 하나로 설치가 죽는 것부터, "succeeded"라고 떠놓고 사실은 아무것도 안 한 가짜 성공까지.

> 설계 문서는 함정을 적어두지 않는다. 함정은 항상 그다음에, 직접 손을 댄 사람한테만 나타난다.

다음 편에서는 abctl로 EC2에 Airbyte OSS를 실제로 올리며 만난 설치 시행착오를 정리한다.
