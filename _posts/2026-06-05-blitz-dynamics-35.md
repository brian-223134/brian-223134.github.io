---
title: "[인턴] Airbyte OSS로 CDC 파이프라인 옮기기 (4) 운영 런북 — 터널·API 인증·부분 재동기화·Job 모니터링"
excerpt: "공인망에 포트를 열지 않고 SSH 터널로만 접근하는 원칙, 짧은 수명의 API 토큰을 자동 재발급하는 함수, Merge refresh로 미달 스트림만 소탕하기, count 게이트로 진짜 적재를 판정하기까지 — 파일럿에서 실측 검증한 운영 절차를 런북으로 묶었다"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, Airbyte, 운영, 런북, Kubernetes, API]

permalink: /intern/company/blitz-dynamics/35/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

## 들어가며

지난 글(3편)에서는 Airbyte의 "succeeded"를 믿지 말아야 하는 이유와, 가짜 성공을 만들어내던 세 종류의 버그를 정리했다. 그 글을 쓰면서 한 가지가 분명해졌다. 버그를 안다고 끝나는 게 아니라, **그 버그들을 매번 안전하게 피해 가는 절차가 손에 익어 있어야** 파일럿이 굴러간다는 것.

처음에는 매 작업을 그때그때 기억에 의존해 처리했다. 그런데 같은 실수를 두 번, 세 번 반복하고 있었다. 터널이 죽은 걸 RDS 장애로 오진하거나, 토큰이 만료된 걸 코드 버그로 착각하거나, refresh를 Truncate로 날려버릴 뻔하거나. 그래서 이쯤에서 한 번, 흩어진 명령들을 **런북**으로 묶기로 했다. 나 자신이 다음 주에 똑같이 헤매지 않으려고, 그리고 다음 담당자가 이 글만 따라 해도 동일한 결과가 나오게 하려고.

이 글의 모든 명령은 파일럿 세션에서 실제로 돌려보고 검증한 것이다. 예시는 모두 `db_a` connection 기준으로 적었지만, 다른 connection에 적용할 때는 connection id만 갈아끼우면 된다(그게 그렇게 단순하지 않은 함정이 몇 군데 있는데, 그 부분은 §3-1에서 따로 짚는다).

런북이라 명령이 많다. 다만 명령을 나열만 하지 않고, 왜 그렇게 하는지를 같이 적어두려 한다. 절차의 "왜"가 빠지면 다음 사람이 변형 상황에서 그대로 응용하지 못하기 때문이다.

## 1. 전체 구조 한눈에

먼저 접근 경로를 그림으로 박아두는 게 디버깅에 도움이 됐다. 어디서 어디로 붙는지가 머릿속에 그려져 있어야, 연결이 끊겼을 때 "어느 구간이 죽었나"를 바로 가늠할 수 있기 때문이다.

```
[맥] ──SSH 터널 8000──> [EC2 localhost:8000] = Airbyte UI + API (kind 내부)
[맥] ──SSH 터널 5433──> [EC2] ──> [테스트 RDS 5432] (destination PG / 메타 DB)
[EC2] ──직결──> API(localhost:8000), 테스트 RDS, 소스 MySQL
```

핵심은 맥에서 직접 닿는 게 아무것도 없다는 점이다. UI도, destination PG도, 메타 DB도 전부 EC2를 경유한 SSH 터널 너머에 있다. Airbyte API는 kind 클러스터 안에서 돌고 EC2의 `localhost:8000`에만 노출되므로, API 작업의 **기본 위치는 EC2**다. EC2에 들어가 있으면 터널이 필요 없고 전부 localhost 직결이라 가장 단순하고 가장 안전하다.

이게 그냥 편의 문제가 아니라 원칙이었다.

> **공인망에 포트를 열지 않는다. 모든 외부 접근은 SSH 터널로만 한다.**

Airbyte UI나 API 포트를 보안그룹에서 열어두면 당장은 편하지만, 인증이 얇은 내부 API가 인터넷에 노출되는 셈이다. 그래서 8000번도 5433번도 공인망에는 닫아두고, 필요할 때만 터널을 띄워 내 맥의 `localhost`로 끌어왔다.

## 2. 포트와 SSH 터널

맥에서 작업해야 할 때 띄우는 터널은 둘이다. 하나는 브라우저로 UI를 보기 위한 8000, 다른 하나는 psql·dbt·pg_dump가 테스트 RDS에 붙기 위한 5433이다.

```bash
# UI를 맥 브라우저로
ssh -fN -i airbyte-oss.pem -L 8000:localhost:8000 ec2-user@<EC2_PUBLIC_IP>
# → http://localhost:8000

# 테스트 RDS를 맥에서 (psql / dbt / pg_dump 수신용)
ssh -fN -i airbyte-oss.pem -L 5433:<TEST_PG_HOST>:5432 ec2-user@<EC2_PUBLIC_IP>

# 터널 생존 확인
lsof -i :8000 ; lsof -i :5433     # ssh LISTEN 이 보여야 한다
```

`-fN`은 명령 실행 없이 백그라운드로 터널만 유지하라는 뜻이고, `-L`은 로컬 포트 포워딩이다. 5433을 굳이 쓰는 이유는 맥 로컬에 이미 PostgreSQL이 5432로 돌고 있을 수 있어서, 충돌과 오접속을 피하려고 일부러 한 칸 비켜둔 것이다. 이 한 칸이 나중에 나를 한참 헤매게 만들었다.

연결이 안 될 때 메시지를 읽는 법이 의외로 중요했다. 같은 "안 됨"이라도 원인이 완전히 다르기 때문이다.

- `connection refused` on 127.0.0.1 → **터널이 죽은 것**이다. RDS 문제가 아니다. 터널을 다시 띄우면 된다.
- `server does not support SSL` on 5432 → 포트를 잘못 지정해서 **맥 로컬 PG에 붙은 것**이다. 5433으로 붙고 있는지 확인한다.

이 두 오진을 구분하는 법을 알고 나서는 디버깅 시간이 눈에 띄게 줄었다. 처음에 나는 `connection refused`를 보고 RDS 보안그룹과 파라미터를 한참 뒤졌는데, 알고 보니 그냥 터널이 끊겨 있었다. 메시지가 127.0.0.1을 가리키면 그건 내 맥 안쪽 이야기고, RDS는 거기까지 오지도 못한 거다.

## 3. API 인증 — abtoken 함수

Airbyte OSS API는 짧은 수명의 Bearer 토큰을 요구한다. 문제는 이 토큰이 생각보다 빨리 만료된다는 거였다. 처음엔 토큰이 만료된 줄도 모르고, API 응답을 파싱하던 스크립트가 `KeyError`를 뱉는 걸 보고 코드를 의심했다. 한참 엉뚱한 곳을 들여다봤다.

그래서 **세션을 시작할 때마다, 그리고 만료 증상이 보일 때마다** 토큰을 재발급하는 함수를 EC2의 `~/.bashrc`에 넣어뒀다.

```bash
abtoken() {
  local creds=$(abctl local credentials 2>/dev/null | sed 's/\x1b\[[0-9;]*m//g')   # ANSI 제거 필수
  export TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/applications/token \
    -H "Content-Type: application/json" \
    -d "{\"client_id\":\"$(echo "$creds"|grep -i client-id|awk '{print $NF}')\",\"client_secret\":\"$(echo "$creds"|grep -i client-secret|awk '{print $NF}')\"}" \
    | python3 -c "import sys,json;print(json.load(sys.stdin).get('access_token',''))")
  echo "TOKEN set (${#TOKEN} chars)"
}
```

여기서 `sed`로 ANSI 색상 코드를 지우는 줄을 빼면 안 된다. `abctl local credentials` 출력에 색상 이스케이프가 섞여 있어서, 그걸 그대로 `awk`로 잘라 쓰면 client-id 끝에 보이지 않는 제어문자가 붙어 토큰 발급이 조용히 실패한다. 이걸 못 찾아서 또 시간을 버렸다.

**만료 증상**은 이렇게 나타난다.

- 응답 파서의 `KeyError: 'job'` 또는 `KeyError: 'jobs'`
- HTTP 401
- 빈 응답

이 중 하나라도 보이면 코드를 의심하기 전에 일단 `abtoken`을 다시 돌리고 재시도한다. 십중팔구 그게 원인이었다.

세션을 새로 열면 토큰 외에도 자주 쓰는 변수들을 같이 세팅해두면 편하다.

```bash
abtoken
export CONN_ID="<CONN_ID>"
export CONN_PG="host=<TEST_PG_HOST> port=5432 dbname=analytics user=airbyte sslmode=require"
export PSQL_META="host=<TEST_PG_HOST> port=5432 dbname=airbyte_meta user=airbyte sslmode=require"
export PGPASSWORD='<공유 비번>'
```

### 3-1. CONN_ID 찾기와 치환의 함정

`CONN_ID`는 connection의 UUID다. connection 단위 API는 전부 이 값을 요구하고, 다른 connection으로 작업을 옮길 때 바꿔야 하는 건 사실상 이 값 하나다.

찾는 법은 세 가지가 있었다.

1. API `connections/list`로 이름↔UUID 매핑을 받는다(workspace id가 필요하다).
2. UI 주소창의 `.../connections/<id>/...`에서 눈으로 긁는다.
3. 메타 DB에서 `SELECT id, name, status FROM connection;`으로 본다. **이때도 SELECT만.**

여기까지는 쉬운데, 진짜 함정은 셸 치환에 있었다. curl 본문을 어떤 따옴표로 감쌌느냐에 따라 변수가 치환되기도 하고 안 되기도 한다.

- **쌍따옴표 본문 + `\"` 이스케이프**면 `$CONN_ID`가 정상적으로 치환된다.
- **홑따옴표 본문**에서는 변수가 치환되지 않는다. 이 경우엔 값을 직접 타이핑해야 한다.

그리고 문서에 적힌 `<...>` 표기는 **placeholder**라는 점. 이걸 꺾쇠째 그대로 붙여넣고 400을 맞은 적이 실제로 있다. 꺾쇠를 포함해 실제 값으로 바꿔야 한다.

마지막으로 다른 connection에 적용할 때 빼먹기 쉬운 것 — refresh body의 `streamNamespace`는 connection마다 다르다. `db_a`는 `"db_a"`지만 다른 connection은 다른 값이다. 옮기기 전에 `web_backend/connections/get`으로 카탈로그를 받아 `stream.namespace`를 확인하고 가는 게 안전하다.

## 4. 트리거 전 안전 점검

이 절이 사실 런북의 심장이다. 3편에서 본 버그들 때문에, **sync든 refresh든 트리거하기 전에 항상 같은 점검을 먼저 돌리는 습관**이 필요했다. 트리거를 누르는 건 쉽지만, 잘못 누르면 오염된 결과를 진짜로 착각하게 된다.

세 가지를 본다.

```sql
-- ① 스냅샷 빚({} state) — 출력이 있으면 그 스트림이 모든 job에 무임승차한다.
SELECT stream_name FROM state
WHERE connection_id='<CONN_ID>' AND octet_length(state::text) <= 2;

-- ② refresh 대기열 — 잔존 시 sync 트리거가 refresh job으로 승격된다
SELECT stream_name, refresh_type FROM stream_refreshes WHERE connection_id='<CONN_ID>';
```

```bash
# ③ 도는 job 없는지
kubectl get pods -n airbyte-abctl | grep replication | grep -v Completed
```

①은 state가 빈 객체(`{}`)로 남아 있는 스트림을 찾는 쿼리다. `{}`는 2바이트라서 `octet_length`이 2 이하인 걸 거르면 빚이 있는 스트림이 잡힌다. 이게 하나라도 나오면, 그 스트림은 이후 어떤 job에도 슬쩍 끼어들어 같이 재스냅샷된다(3편 ISSUE-4의 그 동작이다). ②는 refresh 대기열에 잔여 항목이 있는 경우인데, 이게 남아 있으면 내가 순수 sync를 트리거해도 Airbyte가 그걸 refresh job으로 승격시켜버린다. ③은 단순하다 — 이미 도는 job이 있으면 새 트리거가 꼬인다.

> 빚 스트림이 OOM류 문제와 얽혀 있다면, 해결하거나 deselect하기 전까지는 **어떤 트리거도 금지**다. 문제 스트림 하나가 모든 job을 오염시킨다.

## 5. Sync와 부분 Refresh

용어부터 정리해두자. 둘은 비용도 동작도 다르다.

| 작업 | 의미 | 비용 |
|---|---|---|
| sync | connection 전체 증분 1회(CDC 변경분만), 스트림 지정 불가 | 초경량(파일럿 55건/38초) |
| refresh(부분) | 지정 스트림만 재스냅샷. `Merge`=기존 행 유지하며 병합 | 스트림 크기 비례 |

평소 운영은 sync로 충분하다. CDC 변경분만 따라가니 가볍다. 파일럿에서는 55건을 38초에 처리했다. refresh는 특정 스트림이 어떤 이유로 누락됐을 때, 그 스트림만 골라 다시 떠오는 용도다.

전체 증분 sync는 이렇게 친다.

```bash
curl -s -X POST http://localhost:8000/api/v1/connections/sync \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"connectionId\":\"$CONN_ID\"}"
# 응답의 configType을 반드시 읽을 것: sync=증분, refresh=큐 잔존분 승격(§4-②)
```

응답의 `configType`을 꼭 읽으라고 적어둔 이유가 있다. 내가 sync를 의도했어도 §4-②의 대기열이 남아 있으면 실제로는 refresh로 승격돼 돌아간다. 그러면 비용도 결과도 예상과 다르다. 그래서 트리거를 친 직후 응답에서 이게 정말 `sync`인지 `refresh`인지 확인하는 게 절차에 포함됐다.

특정 스트림만 다시 떠야 할 때는 부분 refresh를 쓴다. 이게 미달 스트림 소탕의 표준 도구였다.

```bash
curl -s -X POST http://localhost:8000/api/v1/connections/refresh \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"connectionId\":\"$CONN_ID\",\"refreshMode\":\"Merge\",\"streams\":[{\"streamName\":\"site_invoice\",\"streamNamespace\":\"db_a\"}]}"
```

여기서 절대 틀리면 안 되는 게 `refreshMode`다.

- enum 값은 **`Merge` / `Truncate`**다. 첫 글자만 대문자다.
- **항상 `Merge`를 쓴다.** `Truncate`는 기존 행을 비우고 다시 채우므로 운영에서는 금지다.

이 enum 표기를 추측으로 `MERGE`나 `merge`라고 쓰면 그냥 거부당한다. 스펙 원문을 확인하라는 교훈이 여기서 나왔다.

부분 refresh의 좋은 점은 **resumable**이라는 것이다. PK 기준으로 체크포인트를 잡기 때문에, 중간에 실패하거나 내가 취소해도 진행분이 보존된다. 그래서 cancel이 안전하다. 다만 §4에서 본 `{}` 빚이 있는 다른 스트림이 이 job에 **동승해서 함께 재스냅샷**되는 경우가 있다. 그래서 site_invoice 하나만 refresh했는데 records 수가 예상보다 커 보일 수 있는데, 이건 빚 스트림이 묻어 온 정상 동작일 수 있다.

### 5-1. 미달 스트림 소탕 — count 검증을 내장한 루프

3편의 교훈이 여기서 코드가 된다. succeeded를 믿지 않고, **실제 count가 소스의 `COUNT(*)` 기준치에 도달해야만 통과**로 치는 루프다. 전체 스크립트는 길어서 핵심 로직만 보인다.

```bash
declare -A EXPECT=( [site_invoice]=85 [user]=605 )   # 소스 정밀치로 채울 것
for S in "${!EXPECT[@]}"; do
  # refresh 트리거 → job 상태 폴링(succeeded/failed) → 실제 count 확인 → 미달이면 재시도(최대 3회)
  CNT=$(psql "$CONN_PG" -Atq -c "SELECT count(*) FROM raw_db_a.\"$S\";")
  # [ "$CNT" -ge "${EXPECT[$S]}" ] 이면 통과
done
# 같은 스트림 3회 연속 미달 = 레이스가 아닌 결정론 → 개별 분석
```

`EXPECT` 맵의 기댓값은 반드시 소스에서 직접 센 정밀치로 채워야 한다(이유는 §6의 count 게이트에서 다시). 루프는 스트림마다 refresh를 트리거하고, job 상태를 폴링하고, **job이 succeeded라고 말해도 거기서 멈추지 않고 destination에서 실제 count를 다시 센다.** 기준치에 못 미치면 재시도하되 최대 3회로 끊는다.

3회 연속으로 같은 스트림이 미달이라면, 그건 일시적 race가 아니라 결정론적 문제다. 그땐 루프를 더 돌릴 게 아니라 그 스트림을 따로 까서 봐야 한다.

job을 취소해야 할 때는 이렇게 한다.

```bash
curl -s -X POST http://localhost:8000/api/v1/jobs/cancel \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"id\": <N>}"
```

Merge refresh의 취소는 안전하다. 체크포인트가 보존되니 데이터가 깨지지 않는다. 단 하나, 취소된 대상 스트림의 state가 `{}`로 남을 수 있다. 그러면 그게 §4의 빚이 되어 다음 트리거에 동승하니, 취소 후에는 다시 §4 점검을 도는 게 맞다.

## 6. Job 모니터링

job이 도는 동안과 끝난 뒤를 보는 방법은 API, kubectl, 그리고 가장 중요한 count 게이트로 나뉜다.

### API로 이력·상태 보기

`jobs/list`(configId=CONN_ID)로 이력을, `jobs/get`(id)로 단건 상태를 본다. 시각은 epoch UTC라 KST로 읽으려면 +9시간 해야 한다. 그리고 파서에는 `'job'`/`'jobs'` 키가 없을 때를 방어하는 로직을 넣어뒀다. 토큰이 만료되면 응답 구조가 달라져서 `KeyError`가 나는데(§3), 그걸 토큰 만료 신호로 잡아 `abtoken` 후 재시도하게 했다.

### kubectl로 실시간 들여다보기

```bash
export KUBECONFIG=~/.airbyte/abctl/abctl.kubeconfig   # SSH 세션마다
kubectl get pods -n airbyte-abctl | grep replication
# 실시간 로그 — 본체는 orchestrator 컨테이너 (source 컨테이너는 빈 껍데기)
kubectl logs -n airbyte-abctl <replication-pod> -c orchestrator --follow \
  | grep -iE "OutOfMemory|Killing|estimated|partition|ERROR|Sync summary"
# 리소스 적용 검증 (connection rr 변경 후 의무 — 기대: 3컨테이너 모두 3Gi/4Gi)
kubectl get pod -n airbyte-abctl <pod> \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.resources.limits.memory}{"\n"}{end}'
```

`KUBECONFIG`는 SSH 세션을 새로 열 때마다 export해야 한다. 자꾸 까먹어서 `kubectl`이 안 붙길래 또 클러스터를 의심했던 적이 있다.

로그를 볼 때 핵심은 **본체가 orchestrator 컨테이너**라는 점이다. replication 파드 안에 source 컨테이너도 있지만 그건 사실상 빈 껍데기라, 거기 로그를 따라가면 아무것도 안 나온다. 실제 sync 진행, OOM, 파티션 추정치 같은 의미 있는 로그는 전부 orchestrator에 있다.

connection 리소스를 바꾼 뒤에는 마지막 명령으로 **3개 컨테이너 모두에 기대한 메모리(3Gi/4Gi)가 박혔는지** 검증하는 게 의무였다. 변경했다고 적용된 게 아니라, 적용됐는지 눈으로 확인해야 했다.

파드가 이미 소멸한 뒤의 로그는 S3에 남으므로, API `attempt/get_for_job`에 `{"jobId": <N>, "attemptNumber": 0}`을 던져 가져온다.

### 진짜 판정 — count 게이트

여기가 이 시리즈 전체를 관통하는 독트린이다. **job status도, UI도, state도 단독으로는 믿지 않는다.** 3편에서 본 가짜 succeeded가 세 종류나 있었기 때문이다. 그래서 대량 job이 끝날 때마다 destination에서 직접 행을 세는 게이트를 통과시킨다.

```bash
psql "$CONN_PG" -Atq -c "SELECT format('SELECT %L AS tbl, count(*) AS cnt FROM raw_db_a.%I;', tablename, tablename) FROM pg_tables WHERE schemaname='raw_db_a' ORDER BY tablename" | psql "$CONN_PG" -At | tee /tmp/db_a_counts_vN.txt
wc -l < /tmp/db_a_counts_vN.txt        # 스트림 수와 일치 — 자기검증(빈 입력=가짜 PASS 방지)
grep '|0$' /tmp/db_a_counts_vN.txt || echo PASS
```

첫 줄은 `raw_db_a` 스키마의 모든 테이블에 대해 `count(*)` 쿼리를 SQL로 생성한 다음, 그걸 다시 psql에 흘려보내 한 번에 세는 패턴이다. 결과를 파일로 떨군다.

`wc -l`을 일부러 끼운 이유가 있다. 게이트 자체가 빈 입력을 받아 조용히 PASS해버리면 그게 또 하나의 가짜 성공이다. 그래서 출력 줄 수가 실제 스트림 수와 일치하는지를 먼저 확인해 **게이트의 자기검증**을 한다. 그다음 `|0$`(count가 0인 테이블)이 하나도 없으면 PASS로 친다.

기준치는 반드시 소스에서 직접 센 `COUNT(*)` 정밀치여야 한다. `information_schema`의 추정치를 쓰면 안 된다 — 실측해보니 20~60%까지 과소평가했다. 그 값을 기준으로 삼았다면 절반만 적재됐는데도 통과로 착각했을 거다. 다만 라이브 소스라 적재 중에도 행이 늘어나니, 기준치보다 수십~수백 건 더 많은 드리프트는 정상이다. 그리고 적재 후 `ANALYZE`를 돌려 통계를 갱신하는 걸 관행으로 삼았다.

## 7. 자주 쓰는 API 정리

connection 카탈로그와 리소스를 만질 때 쓰는 엔드포인트는 이 세 개로 거의 정리됐다.

| 목적 | 엔드포인트 |
|---|---|
| 카탈로그/설정 조회(백업) | `web_backend/connections/get` (변경 전 항상 conn.json 백업) |
| 카탈로그/필드선택 변경 | `web_backend/connections/update` (변경 시 stream_refreshes 자동 큐잉) |
| connection 리소스(유일 유효 레버) | `connections/update` + `resourceRequirements` |

특히 카탈로그를 바꾸기 전에는 `web_backend/connections/get`으로 현재 상태를 `conn.json`에 백업하고 들어간다. update가 합집합이 아니라 덮어쓰기 성격이라, 백업 없이 만지면 기존 선택을 날릴 수 있다. 그리고 카탈로그를 update하면 `stream_refreshes`가 자동으로 큐잉된다는 걸 기억해야 한다 — 그게 §4-②의 대기열이 되어 다음 sync를 refresh로 승격시킨다.

## 8. 금지·주의 목록

런북의 마지막은 하지 말아야 할 것들의 목록이다. 하나하나 다 직접 데었거나, 델 뻔한 것들이다.

1. **Truncate / Clear data 금지.** 데이터와 state가 함께 소실된다. 부분 재동기화는 항상 Merge.
2. **`{}` 빚이 안 풀린 상태에서 트리거 금지.** 문제 스트림 하나가 모든 job을 오염시킨다.
3. **메타 DB는 SELECT만.** 특히 `state` 테이블은 CDC 커서 그 자체다. 잘못 건드리면 커서가 날아간다.
4. **UI로 카탈로그를 대량 조작하지 않는다.** API의 합집합 패턴으로 기존 selected·필드선택·리소스를 보존한다.
5. **API enum을 추측하지 않는다.** 스펙 원문을 확인한다(refreshMode 교훈).
6. **명령 블록에 주석을 섞어 통째로 붙여넣지 않는다**(zsh·psql 둘 다). 한 줄씩 친다.
7. **binlog retention은 168h.** 어떤 sync든 7일 안에 최소 한 번은 돌려야 커서가 유효 구간 안에 남는다.
8. **helm/abctl 재적용은 running job이 없을 때만.**

## 마치며

런북을 다 적고 나서 보니, 이 절차들의 본질이 한 줄로 모였다.

> 이 런북은 결국 **"Airbyte의 상태 보고를 믿지 않는 절차를 자동화한 것"**이다.

안전 점검 → 트리거 → count 게이트. 이 루프를 손에 익히고 나서야, 비로소 "오늘 적재가 진짜로 됐다"고 말할 수 있게 됐다. job이 succeeded라고 말하는 것과, 내가 destination에서 세어 본 행 수가 소스와 맞는 것은 전혀 다른 사실이라는 걸 매번 분리해서 다뤘다.

다음 편(5편)에서는 이렇게 옮긴 raw 데이터를 dbt가 읽도록 모델을 이관한 이야기를 정리한다. Fivetran이 쓰던 컬럼 표기(camelCase)와 soft delete 처리가 Airbyte와 미묘하게 달라서, 그 차이를 dbt 레이어에서 흡수해야 했던 작업이다.
