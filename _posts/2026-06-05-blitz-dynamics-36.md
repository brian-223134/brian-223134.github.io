---
title: "[인턴] Airbyte OSS로 CDC 파이프라인 옮기기 (5) dbt 이관 — Fivetran raw와 Airbyte raw의 차이를 staging에서 흡수하기"
excerpt: "EL을 Airbyte로 옮기고 나니 dbt 모델이 읽던 raw 스키마가 달라져 있었다. Fivetran(소문자)과 Airbyte(camelCase 인용)의 컬럼 규칙, _fivetran_deleted와 _ab_cdc_deleted_at의 차이를 staging 한 겹에서 흡수하며 run을 ERROR 0으로 만들어 간 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, dbt, Airbyte, 데이터엔지니어링, ELT, Postgres]

permalink: /intern/company/blitz-dynamics/36/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

## 들어가며

지난 글에서는 Airbyte 운영 런북을 정리했다. SSH 터널로 안전하게 PG에 붙고, API로 부분 재동기화를 돌리고, 모니터링으로 가짜 succeeded를 거르는 절차까지. 그 단계를 마치며 나는 "EL(Extract & Load)은 이제 끝났다"고 생각했다. db_a 33/33 정합을 확인했고 증분 핸드오버까지 검증했으니, 데이터는 Fivetran 시절과 같은 모양으로 잘 들어오고 있었다.

그런데 끝이 아니었다. **EL을 종결하고 보니, 그 위에서 도는 dbt 모델들이 읽던 raw 스키마가 Fivetran 시절과 미묘하게 달라져 있었기** 때문이다. 같은 MySQL 소스에서 같은 데이터를 뽑아도, 그걸 Postgres에 적재하는 도구가 Fivetran에서 Airbyte로 바뀌면 raw 테이블의 *표면*이 달라진다. 컬럼 이름을 짓는 규칙도 다르고, 삭제를 표시하는 플래그도 다르다.

이 글은 그 차이를 dbt **staging 레이어에서 흡수**해, 기존 마트들이 한 줄도 깨지지 않고 그대로 돌게 만든 이관 기록이다. EL을 옮기는 일보다 이쪽이 더 까다로웠고, 결과적으로 이번 시리즈에서 "데이터 정합"이 왜 가장 어려운 층인지를 가장 잘 보여준 작업이기도 했다.

다섯 편을 이어 온 이 시리즈의 마지막 편이다.

## 1. 실행 환경 — 함정부터 밟았다

본격적인 모델 수정에 들어가기 전에, 우습게도 **dbt 바이너리부터 잘못 잡고 한참을 헤맸다.**

이 프로젝트에서 dbt 명령은 전부 프로젝트 루트에서 poetry 환경으로 돌려야 한다.

```bash
poetry run dbt --profiles-dir . <command>
```

poetry 환경에 박혀 있는 건 **dbt-core 1.9.8 + dbt-postgres 1.9.1**이다. 그런데 시스템 PATH에 잡히는 `dbt`는 **dbt-fusion 2.0.0-preview**, 즉 Rust로 재작성된 전혀 다른 바이너리였다. 이걸 모르고 그냥 `dbt run`을 치면 `Target not found`가 뜨거나, BrokenPipe panic으로 죽거나, 플래그가 비호환이라 엉뚱한 에러를 뱉었다.

세션 초기에 쏟아진 에러의 대부분이 사실 이거였다. "왜 멀쩡한 명령이 안 되지" 하고 profiles.yml과 터널을 몇 번이나 다시 들여다봤는데, 원인은 코드도 설정도 아니고 그냥 **다른 프로그램이 실행되고 있었던 것.** 허탈했다.

정리해 두면 좋은 건 dbt 명령이 DB 연결을 필요로 하는지 여부다. 이걸 알면 "터널이 죽어서 안 되는 건지, 코드가 틀려서 안 되는 건지"를 빠르게 가른다.

| 명령 | DB 연결 | 비고 |
|---|---|---|
| `parse` / `ls` / `compile` | 불필요 | 파일 파싱만. 단 profiles에 타겟 이름은 존재해야 함 |
| `debug` / `run` / `seed` / `test` | 필요 | SSH 터널 필수 |

> 명령이 안 될 때 제일 먼저 의심할 것은 코드가 아니라 "지금 내가 부르는 게 진짜 그 바이너리가 맞나"였다.

## 2. profiles.yml — airbyte_test 타겟

새 raw를 가리키는 타겟을 하나 새로 만들었다. 이름은 `airbyte_test`다.

```yaml
    airbyte_test:
      type: postgres
      threads: 4
      host: 127.0.0.1      # SSH 터널 경유 → 터널 죽으면 connection refused (RDS 문제 아님)
      port: 5433           # 누락 시 5432 → 맥 로컬 PG에 붙는 사고
      user: airbyte
      pass: "{{ env_var('DBT_TEST_PG_PASS') }}"
      dbname: analytics
      schema: my_schema
      sslmode: require
```

여기에도 함정이 둘 숨어 있었다. `host`가 `127.0.0.1`인 건 SSH 터널을 경유하기 때문인데, 터널이 죽으면 `connection refused`가 뜬다. 이걸 보고 "RDS가 죽었나" 하고 엉뚱한 데를 보면 안 된다. **터널이 끊긴 것일 뿐 RDS는 멀쩡하다.**

`port`는 더 고약하다. 5433을 빼먹으면 기본값 5432로 붙는데, 그러면 터널 너머 테스트 PG가 아니라 **맥 로컬에 깔린 Postgres에 조용히 붙는 사고**가 난다. 다행히 `sslmode: require`가 로컬 PG에서는 보통 맞지 않아 거기서 한 번 걸려준 덕에 일찍 알아챘다.

비밀번호는 환경변수로 뺐다. 쓰기 전에 한 번 export 해야 한다.

```bash
export DBT_TEST_PG_PASS='<공유 비번>'
```

`dev` 타겟은 운영 PG(즉 Fivetran raw)를 가리키므로, 이번 작업 내내 **출력 diff의 비교군(A/B의 A)**으로 썼다. 테스트 타겟에서 나온 결과가 운영 결과와 같아야 이관이 무해하다는 걸 증명할 수 있으니까.

## 3. 소스 분석 — 무엇을 고쳐야 하는지부터 셌다

무작정 모델을 고치기 전에 **이번에 실제로 영향받는 소스가 어디까지인지**부터 추적했다. 최종 deliverable은 세 모델이었다.

- `order_with_platform_costs_new`
- `weekly_cost_analysis_bi`
- `daily_material_usage`

이 셋의 의존성을 거슬러 올라가며 소스를 전부 세니 **43개**가 나왔다. 내역은 이렇다.

| 소스군 | 개수 |
|---|---|
| db_a | 29 |
| db_b.store | 1 |
| 외부(raw_platform_cost 6, raw_ext1 3, raw_google_sheets 2, common 2) | 13 |
| 합계 | 43 |

다행히 db_a·db_b 쪽은 **이미 전부 Airbyte로 동기화가 끝나 있었다.** 분석 파일럿 33개 + db_b connection 21개가 이미 돌고 있었으니, EL 쪽에서 추가로 동기화할 일은 없었다. 이번 편은 순수하게 dbt 모델만의 문제였다.

소스 분석에서 한 번 멈칫한 지점이 있다. yml에 `source` 이름이 `raw_raw_db_a`, `raw_raw_db_b`처럼 `raw`가 두 번 붙어 보여서 "이게 잘못된 건가" 싶었는데, 알고 보니 **yml 명명 컨벤션("raw_" 접두사 + 소스명)일 뿐**이고 실제 `schema:`는 `raw_db_a`, `raw_db_b`였다. "ppp" 디렉터리의 모델 3개와 `ext_platform_cost`도 전부 `raw_db_b`를 읽고 있었으니, 이들 역시 이미 Airbyte 관리 영역이었다.

## 4. camelCase 변환 — 이 편의 핵심

여기서부터가 진짜 이번 편의 본론이다. 변경된 타겟으로 처음 `run`을 돌렸더니 **ERROR 32**가 떴다. 그리고 그 32개에는 공통점이 있었다.

원인은 두 도구가 MySQL의 camelCase 컬럼을 적재하는 방식이 정반대였기 때문이다.

| | Fivetran(운영 raw) | Airbyte(테스트 raw) |
|---|---|---|
| MySQL camelCase 컬럼 처리 | 인용 없이 적재 → PG가 소문자로 접음 (`siteid`) | **원형 보존 + 인용 부여** (`"siteId"`, 대소문자 구분) |

Fivetran은 `siteId`를 그냥 `siteid`로 내려보냈다. Postgres는 인용 없는 식별자를 소문자로 접으니까. 그래서 기존 staging은 자연스럽게 `siteid as site_id`처럼 **소문자를 전제로** 작성돼 있었다. 그런데 Airbyte는 같은 컬럼을 `"siteId"`로, 큰따옴표까지 붙여 대소문자를 구분해 적재한다. 그러니 Airbyte 테이블에서 `siteid`를 찾으면 `column "siteid" does not exist`가 뜬다.

흥미로웠던 건 에러가 **완벽하게 이분되었다는 점**이다.

- **ERROR 32** = Airbyte raw를 읽는 모델 전부
- **PASS 17** = 그 외(Airbyte와 무관한) 모델 전부

증상이 이렇게 깨끗하게 갈리면 오히려 마음이 편하다. 원인이 하나라는 뜻이니까.

### DB를 진실의 원천으로 삼은 변환

변환을 어떻게 할지가 고민이었다. 손으로 컬럼명을 일일이 `siteid → "siteId"`로 바꾸는 건 위험하다. 어떤 컬럼이 camelCase인지를 사람이 추측하는 순간 실수가 섞인다. 그래서 `fix_camelcase.py`라는 일회성 스크립트를 만들되, **추측을 완전히 배제하는** 방향으로 설계했다.

핵심 아이디어는 이렇다. **테스트 DB의 `information_schema`에서 실제 컬럼명을 직접 읽어**, 모델별로 `lowercase → "camelCase"` 매핑을 자동 생성하고 치환한다. DB에 실제로 `"siteId"`가 있으면 그렇게 바꾸고, 없으면 건드리지 않는다. 휴리스틱도 추측도 없다. DB가 진실의 원천이다.

이 설계의 부수 효과가 마음에 들었다. 레거시(legacy) 복사 테이블은 전부 소문자라 매핑이 비어 있으니 **자동으로 무변경 처리**된다. 따로 예외를 코딩하지 않아도 안전했다는 뜻이다.

스크립트에서 챙긴 디테일 몇 가지:

- `source()` 호출을 정규식으로 매칭해 대상 모델을 잡는다.
- 소스 스키마 매핑은 앞서 본 명명 컨벤션을 그대로 반영했다.

```python
SOURCE_SCHEMA = {
    'raw_raw_db_a': 'raw_db_a',
    'raw_raw_db_b': 'raw_db_b',
}
```

- psycopg2가 필요해 한 줄 설치했다.

```bash
poetry run pip install psycopg2-binary
```

스크립트는 프로젝트 루트에 두되 **레포에는 커밋하지 않았다.** 일회성 도구가 PR diff에 섞이면 리뷰어를 혼란스럽게 하니까, `.git/info/exclude`에 등록해 추적에서 빼고 본문은 따로 보존했다.

> 자동 변환의 신뢰성은 "내가 어떻게 변환할지 잘 적었는가"가 아니라 "변환의 근거를 어디서 가져왔는가"에서 나온다. information_schema를 근거로 삼은 게 이번 작업에서 가장 잘한 결정이었다.

## 5. soft delete 차이 흡수

두 번째 차이는 삭제 표시 방식이었다. Fivetran은 boolean 플래그 하나로 삭제를 표시했지만, Airbyte는 CDC 삭제 시각을 timestamp로 남긴다.

| 도구 | 삭제 표시 |
|---|---|
| Fivetran | `_fivetran_deleted` (boolean) |
| Airbyte | `_ab_cdc_deleted_at` (timestamp) |

staging에서 이걸 통일했다. 삭제 여부를 boolean으로 노출하되, 판정은 timestamp의 존재 여부로 한다.

```sql
-- 노출 컬럼
is_deleted = (_ab_cdc_deleted_at is not null)

-- 기존 조건 치환
-- Fivetran:  where _fivetran_deleted is true
-- Airbyte:   where _ab_cdc_deleted_at is not null
```

이런 조건이 코드 곳곳에 흩어져 있어서 grep으로 전수 조사했다. **33곳 전부 변환을 끝냈고**, 잔존 0을 다시 grep으로 확인하고 나서야 넘어갔다.

## 6. run 성적을 ERROR 0으로 몰아가기

작업의 진척은 `run` 결과 숫자로 또렷이 보였다.

**1차** (손대기 전)

```
PASS 17 / ERROR 32 / SKIP 47
```

ERROR 32는 전부 앞서 본 camelCase 케이스였다.

**2차** (케이스 변환 후)

```
PASS 67 / ERROR 3 / SKIP 26
```

camelCase를 흡수하니 32개가 한꺼번에 살아났다. 남은 ERROR 3은 또 다른 부류였다.

남은 셋은 **OLTP에서는 이미 떨어져 나갔지만 Fivetran raw에만 화석처럼 살아 있던 디폴트 컬럼 3개**였다. Fivetran이 과거 어느 시점 스키마를 그대로 들고 있었던 것이다. Airbyte raw에는 현재 스키마를 반영해 이 컬럼들이 아예 없으니 missing column 에러가 났다.

| 모델 | 컬럼 | 타입 |
|---|---|---|
| order | parentid | integer |
| purchasing_order_detail | expecteddeliverydate | timestamp |
| user | title | varchar |

처리는 간단했다. staging에서 해당 컬럼을 `null::타입`으로 패딩해 모양만 맞춰 줬다. 어차피 OLTP에서 사라진 컬럼이니 값은 의미가 없고, 마트가 기대하는 컬럼 이름과 타입만 유지하면 된다.

```sql
-- stg_db_a__order.sql:          parentid as parent_id          → null::integer  as parent_id
-- stg_db_a__...detail.sql:      expecteddeliverydate as ...    → null::timestamp as expected_delivery_date
-- stg_db_a__user.sql:           title                          → null::varchar  as title
```

**3차** (패딩 후) 의 기대치는 ERROR 0이었다.

다만 방심하지 않으려 한다. 마지막 층, 즉 하위 마트의 캐스팅 단계에서 새 이슈가 튀어나올 수 있기 때문이다. 대표적으로 MySQL `tinyint(1)`을 **Fivetran은 boolean으로, Airbyte는 int로 적재**하는 차이가 있다. 그러면 `where is_active`처럼 boolean 문맥을 기대하는 자리에서 타입 에러가 날 수 있다. 나오면 모델별 캐스팅 패치로 막을 생각이다.

## 7. 외부 의존성 복사 — pg_dump의 함정

세 deliverable은 db_a·db_b 말고도 외부 스키마(`common`, `raw_ext1`, `raw_platform_cost`, `raw_google_sheets`)를 참조한다. 테스트 타겟에서 run을 끝까지 돌리려면, 그리고 A/B diff를 뜨려면, 이 외부 스키마들을 운영 PG에서 테스트 PG로 **1회 스냅샷 복사**해야 했다.

여기서 `pg_dump`의 고전적인 함정을 하나 밟았다. **`-t`(테이블) 옵션이 하나라도 있으면 `-n`(스키마) 옵션이 무시된다.** 테이블 모드가 스키마 모드를 덮어쓰는 것이다. 그래서 "스키마 통째로"와 "특정 테이블만"을 한 번에 섞어서 던지면 의도와 다른 결과가 나온다. 정답은 **둘을 2회로 분리 실행**하는 것이었다.

부수적으로, 클라이언트는 18.x인데 서버는 16.8이었다. 버전이 안 맞아 걱정했는데 **상위 클라이언트로 하위 서버를 dump하는 건 정상**이다(하위호환). 반대 방향만 조심하면 된다.

복사가 끝난 뒤 `raw_db_a`에는 **Airbyte 관리 테이블 33개 + Fivetran 포맷 레거시 복사본 8개가 공존**하는 상태가 됐다. 여기서 다시 한 번 information_schema 기반 변환의 설계가 빛을 봤다. 레거시 staging은 Fivetran 컬럼(소문자, `_fivetran_deleted`)을 그대로 읽으므로 애초에 변환 대상이 아니었고, 매핑이 비어 자연스럽게 정합을 유지했다.

## 8. raw 스키마 차이 목록 — 운영 전환 문서로 남기다

마지막으로, 이번에 마주한 Fivetran raw와 Airbyte raw의 차이를 운영 전환 문서로 정리했다. 나중에 실제 cutover를 할 사람이 같은 함정을 두 번 밟지 않도록.

| # | 차이 | 처리 |
|---|---|---|
| 1 | 컬럼 케이스: Fivetran 소문자 vs Airbyte `"camelCase"` | staging에서 흡수 완료 |
| 2 | 삭제 플래그: `_fivetran_deleted` bool vs `_ab_cdc_deleted_at` timestamp | 변환 완료 |
| 3 | 메타컬럼: Fivetran 2개 vs Airbyte 9개(`_airbyte_*` 4 + `_ab_cdc_*` 5) | `select *` 모델 주의 |
| 4 | menupan: market_store에서 Airbyte 측 의도적 제외(3편) | BI/애드혹 사용 여부 확인 중 |
| 5 | 디폴트 3컬럼: Airbyte raw에 부재 | staging에서 null 패딩 |

3번이 특히 조용한 지뢰다. Airbyte는 메타컬럼이 9개나 붙으니, `select *`로 다음 레이어에 넘기는 모델이 있으면 컬럼이 줄줄이 새어 나간다. 명시적으로 컬럼을 골라 쓰는 습관이 여기서 중요해진다.

## 마치며

이번 편에서 건진 교훈은 한 문장으로 압축된다. **EL 도구를 바꾸면 데이터는 같아 보여도 raw의 표면은 다르다.** 컬럼 케이스, 삭제 플래그, 메타컬럼, 화석 컬럼까지 — 같은 소스에서 같은 값을 뽑아도 적재 도구가 다르면 테이블의 모양이 달라진다.

그래서 핵심 전략은 **그 차이를 staging 한 겹에서 전부 흡수**하는 것이었다. staging에서 막아내면 그 위 수십 개의 마트는 한 줄도 손대지 않고 살릴 수 있다. 이관의 충격을 한 레이어에 가두는 것, 그게 전부였다.

그리고 그 흡수를 손이 아니라 **DB의 information_schema를 진실의 원천으로 삼아 자동화한 것**이 이번 작업에서 가장 잘한 결정이었다. 사람의 추측을 한 군데도 끼워 넣지 않았더니, 레거시 복사본 같은 예외 케이스조차 별도 코딩 없이 안전하게 처리됐다.

> 이관의 충격은 staging 한 겹에 가둔다. 그리고 변환의 근거는 추측이 아니라 DB에서 가져온다.

다섯 편을 이어 온 시리즈를 닫는다. Fivetran에서 Airbyte OSS로의 이관은 돌아보면 **"설치 < 운영 < 데이터 정합"의 순서로 어려웠다.** abctl로 EC2에 올리는 설치가 가장 쉬웠고, 가짜 succeeded와 부분 재동기화를 다루는 운영이 그다음이었고, raw의 표면 차이를 흡수하는 데이터 정합이 가장 까다로웠다.

그리고 이 다섯 편 내내 가장 많이 배운 건 도구 그 자체가 아니었다. 도구가 "succeeded"라고 보고하는 화면이 아니라, **목적지 테이블에서 내가 직접 센 숫자를 믿는 습관** — 그게 이번 PoC가 내게 남긴 가장 값진 것이었다.
