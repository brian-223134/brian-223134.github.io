---
title: "[인턴] Airbyte OSS로 CDC 파이프라인 옮기기 (3) 'succeeded'를 믿지 마라 — 파일럿에서 만난 4대 버그와 운영 독트린"
excerpt: "33 스트림 파일럿 sync에서 만난 무단 스킵·가짜 succeeded·비대 JSON 컬럼 OOM·빈 state 전염. job status를 믿지 않고 destination count로만 판정하게 된 이유와, 대량 sync를 수렴 루프로 운영하게 된 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, Airbyte, CDC, 데이터엔지니어링, 트러블슈팅, OOM]

permalink: /intern/company/blitz-dynamics/34/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

## 들어가며

지난 글에서는 abctl로 EC2 위에 Airbyte OSS를 올리는 과정과 그 시행착오를 정리했다. 설치가 끝났으니 이제 데이터를 옮길 차례였는데, 처음부터 900개가 넘는 전체 스트림을 다 던질 생각은 없었다. 한 번도 운영해본 적 없는 도구에 대고 그러는 건 무모하다고 느꼈고, 그래서 **33개 스트림(약 8.5GB)으로 파일럿**을 먼저 돌렸다. db_a DB의 dbt가 실제로 필요로 하는 32개에, market_store 하나를 일부러 더 끼워 넣었다. market_store는 큰 테이블에만 적용되는 병렬 분할 읽기 경로를 타기 때문에, 그 경로를 실전에서 미리 밟아보고 싶었다.

결론부터 말하면 파일럿은 **33/33 정합으로 종결**했다. 하지만 그 "결국 맞췄다"는 문장 하나에 도달하기까지, 나는 Airbyte를 다시는 순진하게 믿지 않게 만든 버그들을 줄줄이 만났다. job은 분명 succeeded라고 말하는데 데이터는 절반이 비어 있고, 85행짜리 테이블을 옮기랬더니 30만 건을 끌고 와서 죽는 식이었다. 이 글은 그 과정에서 만난 이슈들과, 거기서 세운 운영 독트린의 기록이다.

미리 한 줄로 못박아 두자면, 이번 파일럿에서 내가 얻은 가장 비싼 교훈은 이거였다.

> job status / UI 표시 / state 기록 — 셋 다 단독으로는 신뢰할 수 없다. 적재의 진실은 오직 destination의 count뿐이다.

## 1. 파일럿 환경

먼저 어떤 조건에서 돌렸는지부터 정리해 둔다.

| 항목 | 값 |
|---|---|
| 플랫폼 | abctl 0.30.4 / 차트·App 2.1.0 (EC2 m7i.2xlarge, kind) |
| source-mysql | 3.52.3 (소스 MySQL 8.0.44, CDC, RDS) |
| destination-postgres | 3.0.13 (Direct Load — raw 테이블 없음, db `analytics`) |
| connection | db_a → Postgres, namespace `raw_db_a` |
| 파일럿 스코프 | 33 스트림 ≈ 8.5GB |
| sync mode | 전 스트림 Incremental \| Append+Dedup, CDC deletion = Soft delete, Invalid CDC Position = Fail sync |
| binlog retention | 168h |

옮기기 전에 가장 걱정했던 건 운영 OLTP에 주는 부하였다. CDC 초기 스냅샷이 운영 DB를 긁는 동안 서비스가 느려지면 곤란하니까. 그런데 파일럿 내내 소스 RDS를 실측해 보니 **CPU ~10%, ReadIOPS ~60/s, ReadLatency ~1ms**로, 운영 영향은 사실상 없었다. 스냅샷이 InnoDB를 락 없이 순차 스캔하는 방식이라 read-ahead로 읽기가 병합된 덕이었다. 이건 의외로 안심되는 포인트였다.

## 2. Job 타임라인 — 이게 다 한 connection에서 벌어진 일이다

이슈를 하나씩 풀기 전에, 전체 흐름을 시간순으로 깔아 두는 게 읽기 편할 것 같다. 시각은 KST 기준인데, 한 가지 함정이 있었다. **메타 DB와 EC2가 UTC로 돌고 있어서**, 로그 시각에 9시간을 더해야 내 시계와 맞았다. 이걸 모르고 한참 "왜 job 시간이 안 맞지" 하며 헤맸다.

| job | 무슨 일이 있었나 |
|---|---|
| job 9 (첫 sync) | succeeded라고 보고. 실제론 33개 중 15개가 0건 + 2개 부분 적재 → ISSUE-1 |
| job 10~16 | 미달분을 부분 refresh로 복구 |
| job 11 (15스트림 일괄 refresh) | succeeded인데 orchestrator OOM, 9/15만 복구 → ISSUE-2, 3 |
| job 12 (site_invoice "단건" refresh) | 85행짜리인데 482MB / 305,901건을 옮기고 OOM → ISSUE-4 |
| job 17 (market_store 단독, 4Gi 적용 후) | 시작 14초 만에 OOM, count 동결 → ISSUE-3 |
| job 18 (market_store, menupan 제외 후) | 16.44MB / 22,437건 / 1분 1초 완주, count 22,433 = MySQL 정확 일치 → ISSUE-3 해결 |
| job 21 (첫 순수 증분 sync) | 55건 / 22.4kB / 38초 — 증분 핸드오버 검증 완료 |

이 표만 봐도 알겠지만, "succeeded"라는 글자가 얼마나 자주 거짓말을 했는지가 이 편의 전부다.

## 3. ISSUE-1 — 첫 sync에서 스트림이 통째로 스킵됐다 (그런데 job은 succeeded)

신규 connection의 첫 sync인 job 9가 succeeded로 끝났다. 그래서 처음엔 잘 굴러갔다고 생각했다. 그런데 destination count를 세어 보니 **33개 중 15개가 0건이었고, 2개는 부분 적재**(order_delivery 1건, market_store 1,385건)에 그쳐 있었다.

처음 든 의심은 "destination 쪽에서 뭔가 유실됐나"였다. 그런데 Airbyte UI의 스트림별 표시도 "0 loaded"였고, 그 숫자가 destination count와 정확히 일치했다. 즉 **destination이 받았다가 흘린 게 아니라, 소스를 애초에 읽지 않은 것**이었다. job 11에서도 6/15가 같은 패턴으로 재현됐으니 우연도 아니었다.

진단을 위해 먼저 `jobs/list`로 job 9가 이 connection의 최초이자 유일한 시도임을 확인했다. "이전에 돌린 잔재가 섞인 거 아니냐"는 가설을 먼저 죽여 둔 것이다. 그다음 메타 DB의 `state` 테이블을 들여다봤다.

```sql
-- 스트림별 state 마커 길이로 정상/스킵을 갈라본다
select stream_name, length(state::text) as state_bytes
from state
order by state_bytes;
```

정상적으로 초기적재를 마친 스트림은 71~99바이트짜리 완료 마커를 들고 있었는데, 스킵된 스트림은 **2바이트(`{}`)** 거나, "마커는 있는데 데이터는 비어 있는" 변종이 섞여 있었다. 부분 적재로 잡힌 1건, 1,385건도 스냅샷이 일부 남은 게 아니었다. 이건 해당 스트림이 CDC-only로 취급되면서 sync가 도는 동안 binlog에서 주워 담은 변경분의 지문이었다. 다시 말해 초기 스냅샷은 통째로 건너뛰고, 그 사이에 우연히 흘러간 변경분만 들어온 것이다.

실패한 스트림들 사이에 크기, 이름, 타입 어떤 공통점도 없었다. 그래서 나는 이걸 동시성 초기적재 플래너의 레이스로 추정했다. 같은 증상의 업스트림 이슈를 찾아봤지만 똑 떨어지는 건 없었다.

대응은 단순했다. 미달 스트림만 골라 부분 refresh(`refreshMode: Merge`)로 복구했다. 단, 여러 개를 한꺼번에 일괄로 돌리면 또 일부가 스킵됐다. 그래서 **count 검증을 내장한 단건 루프**로 한 스트림씩 채워 수렴시켰다.

> 근본 원인(업스트림 거동)은 끝내 해결하지 못했다. 그러니 나는 이걸 "재발한다"는 전제로 운영하기로 했다. 전체 확장 후에도 count 게이트는 선택이 아니라 필수다.

이건 업스트림(airbytehq) 제보 1순위로 정리했다.

## 4. ISSUE-2 — 가짜 succeeded 3형, job status를 못 믿게 된 결정적 계기

ISSUE-1을 겪고 나니 "그럼 succeeded는 도대체 무슨 뜻이냐"는 의문이 들었다. 그래서 세션 내내 "succeeded인데 데이터가 틀린" 케이스를 모아봤더니, 세 종류로 정리됐다.

| 형 | 사례 | 내용 |
|---|---|---|
| 스킵형 | job 9 | 소스를 안 읽고 완료로 기록 (ISSUE-1) |
| 종료단 OOM형 | job 11, 12 | replication은 완료 보고했는데, 정리 단계에서 orchestrator가 OOM. status는 succeeded |
| 진행중 OOM형 | job 17 | 스트림 처리 도중 OOM으로 미완인데 status는 succeeded |

세 경우 모두, job status만 보면 초록불이다. 그래서 내가 내린 독트린은 이거였다.

> job status도, UI 카드도, state 기록도 단독으로는 믿지 않는다. 판정은 오직 destination count로만 한다.

그리고 count 게이트 자체에도 함정이 있었다. 한번은 입력이 빈 상태로 게이트가 통과(PASS)해 버린 적이 있었다. 비교 대상이 비어 있으니 "0 == 0"으로 잠깐 맞다고 나온 가짜 PASS였다. 그래서 게이트 스크립트에 자기검증을 박았다 — 파이프 출력을 `wc -l`로 행 수까지 확인한 뒤에야 판정하도록.

"OOM이 났는데 succeeded로 보고한다"는 건 데이터 정합성에 대한 명백한 오보고라, 업스트림 제보 2순위로 정리했다.

## 5. ISSUE-3 — market_store OOM의 진짜 범인은 비대 JSON 컬럼(menupan)

market_store(4.9GB, 22,433행) 스냅샷이 든 job은 11, 12, 17 어느 것이든 orchestrator가 같은 자리에서 죽었다.

```text
java.lang.OutOfMemoryError: Java heap space
Killing orchestrator because of an Exception
```

job 17은 시작한 지 14초 만에 죽었다. 메모리가 부족하다는 건 알겠는데, 4.9GB 테이블을 옮기는 데 14초 만에 힙이 터진다는 게 이상했다. 스택 트레이스를 보니 `Jsons.serialize → ObjectMapper.writeValueAsString → ...`, 즉 destination으로 직렬화해 내보내는 길목에서 힙이 고갈되고 있었다.

테이블을 부검했다.

```sql
SHOW CREATE TABLE market_store;
```

`menupan json` 컬럼이 범인이었다. 이 컬럼을 정량으로 떠 봤더니 **평균 229KB/행, 최대 5.3MB**, 1MB를 넘는 행이 360개였다.

처음엔 "5.3MB짜리 독행(poison row) 한두 개가 통째로 힙을 먹는 거 아니냐"고 생각했는데, 그러기엔 max 5.3MB로는 단발 OOM 이론이 안 섰다. 결론은 누적형이었다. **평균 229KB짜리 행이 끊임없이 들어오면서, 행 수 기준으로 잡혀 있는 인플라이트 버퍼를 바이트 기준으로 초과**시키는 구조였다. 버퍼는 "몇 행"으로 한도를 잡는데, 한 행이 수백 KB면 같은 행 수라도 바이트로는 폭발한다.

자연스럽게 메모리를 키워보고 싶었다. connection 레벨로 컨테이너를 4Gi까지 올린 걸 파드 스펙에서 직접 확인하고 다시 돌렸다. 그래도 **똑같은 지점에서 OOM**이었다. 컨테이너 증설은 해법이 아니었다.

그래서 방향을 틀었다. Airbyte의 **필드 선택(field selection)** 으로 menupan 컬럼 자체를 빼기로 한 것이다. 그 전에 dbt 레포를 grep해서 menupan을 쓰는 모델이 하나도 없는 걸 확인했다 — 안 쓰는 컬럼이라면 마음 편히 버릴 수 있었다.

```bash
# web_backend로 해당 스트림만 필드 선택 켜고 나머지 컬럼만 남긴다
# (_ab_cdc_* 메타필드는 유지)
curl -s -X POST "$AIRBYTE_API/web_backend/connections/update" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
        "connectionId": "<CONN_ID>",
        "syncCatalog": { "...": "market_store stream: fieldSelectionEnabled=true, menupan 제외" }
      }'
```

결과는 극적이었다. 전송량이 **4.9GB에서 16.44MB로** 줄었고, 1분 1초 만에 완주했으며, count는 22,433으로 MySQL과 정확히 일치했다(ISSUE-3 해결).

다만 이건 운영 전환 문서에 반드시 적어 둬야 할 차이를 만든다. **Fivetran raw에는 menupan이 있고, Airbyte raw에는 없다.** 운영 PG의 raw_db_a가 76GB인데, 그중 상당량이 이 컬럼의 이력으로 추정된다. 그러니 전체 확장 전에 **동종 비대 컬럼을 사전 스캔**하는 건 필수다. (참고로 가장 큰 테이블 whale은 행당 ~123B라 이 리스크는 없었다.)

> 메모리를 키워서 풀려는 본능을 버리고, 원인(비대 컬럼)을 제거하는 게 정답이었다. 행 수 기준 버퍼가 비대 레코드에서 힙을 초과한다는 건 바이트 기준 백프레셔의 부재라, 이걸 업스트림 제보 3순위로 정리했다.

## 6. ISSUE-4 — 빈 state(`{}`)의 전염성, "스냅샷 빚"이라 부르기로 했다

job 12가 가장 어이없었다. site_invoice는 85행짜리 테이블인데, 그 "단건" refresh가 **482MB / 305,901건**을 옮기고 OOM으로 죽었다. 85행을 옮기랬는데 30만 건이라니, 한참을 들여다봤다.

메커니즘은 이랬다. refresh를 요청하면 플랫폼은 대상 스트림의 state를 `{}`로 비우고, 스냅샷이 끝나면 다시 완료 마커를 적는다. 그런데 이 `{}`는 "이 job에 소속된 명령"이 아니라 **스트림에 영속적으로 붙어 있는 상태**다. 그래서 다음 job이 무엇이든, 소스는 `{}`를 들고 있는 스트림 전부의 스냅샷을 함께 떠안는다.

당시 market_store를 비롯한 몇몇 스트림의 `{}`가 살아 있었다. 그래서 site_invoice만 refresh한다고 트리거한 job 12에, 그 비대한 스트림들이 전부 동승했고, 결국 menupan을 든 행이 orchestrator를 죽인 것이다. 다시 말해, **OOM으로 죽는 스트림이 `{}`를 들고 있는 한, 그 connection의 모든 후속 job이 같은 지뢰를 밟는다.** 1회성 실패가 아니라 전염성 실패였다.

여기서 운영 규칙 두 개를 세웠다.

1. 문제 스트림은 "완료"시키거나 카탈로그에서 **deselect로 격리**하기 전까지, 어떤 sync도 트리거하지 않는다. (deselect하면 `{}`가 남아 있어도 작업 목록에서는 빠진다.)
2. state가 `{}`인 스트림을 쿼리하는 걸 idle 시점의 "빚 장부"로 삼는다.

이 사건은 menupan을 제외해 market_store가 정상 완료되면서, `{}`가 0행이 되어 빚이 전액 청산됐다.

## 7. ISSUE-5 — 리소스 설정은 connection 레벨만 먹힌다

ISSUE-3에서 컨테이너를 4Gi로 올린 얘기를 했는데, 사실 그 "어디서 올리느냐"부터가 한 판 싸움이었다.

처음엔 values.yaml의 `workloadLauncher.extraEnv`에 리소스 env를 박아 뒀다. 그런데 정작 replication 파드는 플랫폼 기본값(orchestrator 2Gi, source·destination 각 1Gi)으로 떴다. `CONNECTOR_SPECIFIC_RESOURCE_DEFAULTS_ENABLED: "false"`까지 줬는데도 그대로였다.

env에 넣은 값과 실제 파드 스펙이 다른 걸 jsonpath로 실증한 뒤, 업스트림 #68162에서 같은 보고를 확인했다. 공식 우선순위는 이랬다.

> **Connection > Connector(actor) > 커넥터 정의 > env.** 위쪽이 이긴다.

즉 env는 사다리의 맨 아래라, 위에서 기본값이 깔리면 무조건 진다. 그래서 사다리 최상층인 **connection 레벨**을 API로 설정했다.

```bash
# connections/update의 resourceRequirements로 직접 지정
curl -s -X POST "$AIRBYTE_API/connections/update" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{ "connectionId": "<CONN_ID>",
        "resourceRequirements": { "cpu_request": "...", "memory_limit": "4Gi" } }'
```

이후 job 파드에서 orchestrator/source/destination 세 컨테이너 모두에 3Gi/4Gi가 적용된 걸 jsonpath로 직접 확인했다. **connection 레벨이 유일하게 검증된 리소스 레버**였고(orchestrator 포함), values.yaml의 extraEnv 리소스 블록은 효과가 없는 게 실증됐다. 다음 재설치 때 이 블록은 지우는 게 낫다. 물론 ISSUE-3에서 봤듯, 메모리를 올린다고 만능은 아니지만.

## 8. ISSUE-6 — refresh enum은 RETAIN도 MERGE도 아니었다

부분 refresh를 API로 던지면서 `refreshMode`에 `"RETAIN"`을 넣었더니 400이 떨어졌다. UI 라벨에 "retain records"가 있길래 그대로 쓴 거였는데. 그럼 `"MERGE"`겠지 하고 바꿨더니 또 400(`Unexpected value`)이었다.

추측을 멈추고 airbyte-platform 레포의 OpenAPI 스펙 원문을 직접 열었다. enum은 **`Truncate` / `Merge`**, 첫 글자만 대문자였다. UI 라벨 매핑은 "retain records" = `Merge`, "remove records" = `Truncate`였다.

> UI 라벨로 API enum을 추측하지 말 것. 스펙 원문이나 에러의 모델명으로 확인할 것.

사소해 보이지만, 이런 데서 30분씩 새는 게 실전이다.

## 9. ISSUE-7 — 대형 INT PK 병렬 분할 버그(#63776)를 미리 피하고, 실전에서도 통과시켰다

market_store를 파일럿에 일부러 끼워 넣은 진짜 이유가 이거였다. db_a 최대 테이블인 whale(11.9GB / 1억 행)의 PK가 int인데, source-mysql은 ~3GB를 넘는 테이블에 PK 기반 병렬 분할 읽기를 적용한다. 그런데 이 경로에 INT PK ClassCastException(#63776, 3.50.3)과 division-by-zero(#64139) 버그가 보고된 적이 있었다.

먼저 GitHub에서 수정 버전을 확인했다. #63776은 3.50.5에서 fix였고, 내가 설치한 건 3.52.3이라 3.50.5보다 높으니 노출되지 않는다고 판정했다. 그래도 머리로만 안심하긴 찜찜해서, 같은 병렬 분할 경로를 타는 market_store(4.9GB)를 파일럿에 넣어 실전으로 프로브했다.

job 17 로그에서 분할 경로가 예외 없이 도는 걸 확인했다.

```text
Sampled 1024 rows → estimated 5039 MiB
Target partition 3000 MiB → 파티션 생성, 예외 없음
```

이걸로 whale 본 적재의 분할 경로 리스크는 해소했다. (참고로 같은 job 17에서 뒤따른 OOM은 분할이 아니라 ISSUE-3, 즉 menupan이 원인인 별개 사건이다.)

## 10. ISSUE-8 — 자잘하지만 운영에 박아 둔 잔이슈들

큰 사건 사이사이에 부딪힌 작은 것들도, 모이면 운영 규칙이 된다.

| 항목 | 내용 / 결론 |
|---|---|
| API 토큰 만료 | 수명이 짧아 세션 중에 만료 → `KeyError: 'job'` 류로 오진. bashrc 함수로 재발급을 표준화하고 파서에 키 방어 추가 (다음 편 런북) |
| 자격증명 ANSI | credentials 출력에 ESC 코드가 섞여 JSON을 깨뜨림 → sed로 제거하는 걸 내장 |
| 로그 위치 | source 컨테이너 로그는 빈 껍데기였고, **replication 로그 본체는 orchestrator 컨테이너**에 있었다. 파드가 사라진 뒤엔 `attempt/get_for_job`이 S3에서 읽어준다 |
| n_live_tup 지연 | 벌크 적재 직후엔 통계가 안 갱신돼 0으로 보임 → 판정은 `count(*)`로 하고, 적재 후 ANALYZE를 관행화 |
| sync→refresh 승격 | 카탈로그 변경이 `stream_refreshes` 큐에 자동 등록 → 큐가 남아 있으면 sync 트리거가 refresh로 승격됨(job 20) |
| 정합성 기준 | information_schema 추정치는 20~60% 과소 사례가 많음 → 기준치는 반드시 `COUNT(*)` 정밀치로, 비교 시 라이브 드리프트(+수십~수백)는 허용 |

## 11. ISSUE-9 (미확정) — 0건 스트림은 destination 테이블을 만드는가

핸드오프 초기에 나는 "0건이면 테이블을 안 만든다"고 적어 뒀었다. 그런데 사후 검증을 해 보니, 이번 파일럿 카탈로그는 33개(빈 테이블 미포함)였고, 그래서 **이 가설은 이번엔 실증되지 않았다.**

이게 왜 중요하냐면, db_a엔 빈 테이블이 37개 있고, dbt staging이 빈 테이블을 참조하는데 그 relation이 안 만들어지면 run이 missing relation으로 죽는다. 그러니 **전체 확장 전에 빈 테이블 1개로 단독 검증**을 꼭 해야 한다. 만약 안 만들어지면 수동 DDL을 박거나 해당 모델을 비활성화하는 결정을 내려야 한다.

## 12. 전체 확장에 들고 갈 것

이번 파일럿이 내게 가르쳐 준 운영 모델은 한 단어로 **수렴 루프**다.

> 본 sync → count 게이트 → 미달분 부분 refresh → 재게이트.

"야간에 한 번 돌리면 끝"을 보장하지 않는 것으로 계획을 짰다. 한 번에 안 들어오는 게 정상이라는 전제다. 그리고 전체 확장 전 체크리스트도 이번 경험에서 그대로 뽑혀 나왔다.

- 비대 컬럼 사전 스캔 (ISSUE-3)
- 빈 테이블 동작 검증 (ISSUE-9)
- state `{}` 빚 0 확인 (ISSUE-4)
- connection 레벨 리소스 설정 유지 (ISSUE-5)
- binlog 168h 안에 들어오도록 sync 주기 관리

업스트림에 제보할 후보도 정리해 뒀다.

| 우선순위 | 내용 |
|---|---|
| 1 | 다중 스트림 sync/refresh에서 초기적재가 무단 스킵되는데 succeeded로 보고 (ISSUE-1) |
| 2 | orchestrator OOM이 났는데 job status는 succeeded (ISSUE-2) |
| 3 | 행 수 기준 인플라이트 버퍼가 비대 레코드에서 힙을 초과 — 바이트 기준 백프레셔 부재 (ISSUE-3) |

## 마치며

이번 파일럿에서 가장 크게 배운 건 결국 이 한 문장이다. **분산 EL 도구의 "성공"은 데이터의 성공이 아니다.** succeeded는 그저 "프로세스가 예외 없이 종료됐다"는 뜻일 뿐, "데이터가 다 들어왔다"는 보장이 전혀 아니었다. 진실은 destination에서 직접 센 숫자에만 있었다.

> job status도 UI도 state도 다 거짓말을 할 수 있다. 그래서 나는 모든 자동화에 count 게이트를 박았다. 적재의 진실은 오직 destination의 count뿐이다.

다음 편(4편)에서는, 이 모든 작업을 안전하게 반복하기 위해 정리한 운영 런북 — 터널, API, 부분 재동기화, 모니터링 — 을 풀어 보려 한다.
