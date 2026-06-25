---
title: "[인턴] 같은 자물쇠를 두 트리거가 공유할 때 — 비대칭 flock으로 푼 동시성"
excerpt: "30분마다 오는 sync webhook과 하루 한 번 도는 systemd timer가 같은 run-dbt.sh를 공유한다. 둘이 겹칠 때 누가 양보하느냐를 비대칭으로 둔 이야기. flock 한 줄을 조건부로 바꾸고, 그 조건을 한쪽에만 흘리지 않으려 며칠을 신경 쓴 기록."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, dbt, systemd, flock, 동시성]

permalink: /intern/company/blitz-dynamics/39/

toc: true
toc_sticky: true

date: 2026-06-25
last_modified_at: 2026-06-25
---

## 들어가며

[지난 글](/intern/company/blitz-dynamics/38/)에서 Airbyte sync가 끝나면 POST 한 방이 `run-dbt.sh`를 깨워 dbt를 돌리는 경로를 따라갔다. 그 글의 마지막에 한 줄 흘려 둔 게 있다. "sync 훅은 `dbt run`(모델만)을 쓰고, 테스트를 포함한 전체 build는 따로 systemd 타이머가 맡는다"는.

그 한 줄이 이번 글의 시작이다. 트리거가 둘이 됐다. 하나는 sync가 끝날 때마다 오는 webhook이고, 다른 하나는 하루 한 번 정해진 시각에 도는 systemd timer다. 그런데 이 둘은 **같은 `run-dbt.sh`를, 따라서 같은 자물쇠(flock)를 공유한다.** 자물쇠가 하나뿐이니, 둘이 동시에 도달하면 한쪽은 반드시 기다리거나 포기해야 한다.

문제는 "누가 양보하느냐"였다. 처음엔 둘 다 똑같이 "겹치면 skip"으로 뒀는데, 그 대칭이 하필 제일 중요한 빌드를 조용히 날려 먹었다. 이 글은 그 대칭을 일부러 깨서 — 자주 오는 쪽은 양보하고 하루 한 번 오는 쪽은 기다리게 만들어 — 푼 이야기다. 코드로 보면 `flock` 한 줄을 조건부로 바꾼 게 전부지만, 그 조건이 엉뚱한 쪽으로 새지 않게 하느라 신경을 꽤 썼다.

---

## 1. 왜 트리거가 둘이어야 했나

먼저 트리거가 왜 둘이 됐는지부터. 하나로 안 됐던 이유가 있다.

sync webhook이 돌리는 건 전체 dbt가 아니라 **셀렉터로 고른 95개 모델뿐**이다. `dbt run --select "+mart_cost_a +mart_bi_b ..."` 처럼, 비용·BI 마트와 그 상류만 짚어서 돌린다. sync가 자주 끝나니 자주 돌아야 하는데, 매번 전체를 돌리면 무겁고 느리다. 그래서 "자주 갱신돼야 하는 신선도 핵심"만 추려 sync 주기에 묶었다.

그런데 그러면 **셀렉터 밖 모델들**은 누가 갱신하나? 두 종류가 셀렉터 밖에 있다.

- 비용·BI 마트가 아닌, 덜 급한 일반 모델들.
- `dockers/*` 커스텀 잡이 직접 INSERT하는 raw(`raw_kepco`·`raw_google_sheets` 등)에 의존하는 마트들. 이 raw들의 신선도는 Airbyte sync와 무관하게 각자의 잡 스케줄을 탄다. 그러니 "sync 끝남" 신호로는 갱신 타이밍을 잡을 수가 없다.

이것들까지 sync 셀렉터에 욱여넣을 수도 있었지만, 그러면 sync마다 도는 빌드가 점점 무거워지고 "자주·가볍게"라는 sync 훅의 본분이 흐려진다. 그래서 역할을 갈랐다. **신선도가 급한 것은 sync 주기에 자주(`run --select`), 나머지 전체는 하루 한 번 몰아서(`run`).** 그 "하루 한 번"을 맡은 게 systemd timer다.

| 트리거 | 무엇을 도나 | 주기 | 겹치면 |
|---|---|---|---|
| sync webhook | `dbt run --select`(비용/BI 마트 95개) | sync 끝날 때마다(잦음) | **skip(양보)** |
| systemd timer (`dbt-daily`) | `dbt run`(셀렉터 밖 포함 전체) | 하루 1회, 15:00 KST | **최대 30분 대기** |

이 표의 마지막 칸이 이 글의 전부다. 둘이 같은 자물쇠를 공유하는데, 한쪽은 양보하고 한쪽은 기다린다. 왜 그렇게 갈랐는지를 풀어 보자.

---

## 2. 자물쇠가 하나뿐인 이유

트리거는 둘인데 진입점은 하나다. webhook도 timer도 결국 같은 `run-dbt.sh`를 부른다. 인자만 다를 뿐(`run --select ...` vs `run`), 실행되는 스크립트는 한 파일이다. 38편에서 본 그 "같은 진입점" 설계가 여기서도 그대로다. 덕분에 git pull도, 컨테이너 보장도, 로그도, Slack 알림도 한 곳에만 있으면 된다.

그런데 진입점이 하나라는 건, **둘이 동시에 들어오면 같은 dbt가 같은 웨어하우스에서 두 번 겹쳐 돈다**는 뜻이기도 하다. 같은 테이블을 두 빌드가 동시에 쓰면 충돌한다. 그래서 38편에서도 스크립트 맨 앞에 `flock` 한 줄을 두고 직렬화했다. 자물쇠를 못 잡으면 안 들어간다.

문제는 이 자물쇠가 **딱 하나**라는 데 있다. `/tmp/run-dbt.lock` 하나를 sync run도 timer run도 같이 노린다. 평소엔 부딪힐 일이 거의 없다. sync run은 ~10분이면 끝나고, timer는 하루에 딱 한 번 15:00에만 온다. 둘이 만나려면 하필 15:00 언저리에 sync run이 돌고 있어야 하는데, 그게 매일 일어나진 않는다.

하지만 "거의 안 일어남"은 "안 일어남"이 아니다. 그리고 하필 그 드문 충돌이, 둘 중 제일 잃으면 안 되는 쪽을 때렸다.

---

## 3. 대칭이 만든 함정 — 하루치 전체 run이 조용히 사라지다

처음 구성(글에서 A안이라 부르던 것)은 단순했다. **둘 다 똑같이 `flock -n`.** non-blocking, 즉 자물쇠가 잡혀 있으면 즉시 포기(skip)하고 빠진다.

sync run끼리는 이게 맞다. sync가 짧은 간격으로 연달아 성공하면 POST도 연달아 온다. 앞 run이 도는 중에 뒤 run이 오면 그냥 skip시키면 된다. 버려도 아쉽지 않다. dbt는 큐에 쌓인 작업을 처리하는 게 아니라 **매번 소스의 현재 상태를 새로 읽기** 때문이다. 한 번 걸렀어도 다음 sync의 run이 그새 쌓인 변경분까지 한꺼번에 갱신한다. 일종의 자가복구다.

함정은 timer였다. timer도 `flock -n`이면, **15:00 정각에 마침 sync run이 돌고 있을 때 그날의 전체 run이 그대로 skip된다.** 그냥 한 번 미뤄지는 게 아니라 **그날치가 통째로 사라진다.** 다음 기회는 내일 15:00이다. 그동안 셀렉터 밖 모델들과 `dockers/*` 의존 마트는 24시간을 묵은 채로 남는다.

게다가 증상이 지독하게 조용하다. skip은 에러가 아니다. 로그에 `[run-dbt] 이미 실행 중 — skip` 한 줄 찍히고 정상 종료(exit 0)다. Slack 알림도 안 간다(skip은 노이즈라 일부러 안 쏜다). 그러니 아무도 모른다. "어제 전체 run이 돌았겠지" 하고 믿고 있는데 실은 안 돈, 데이터만 조용히 하루 묵은 상태. 38편에서 git pull 충돌이 만들던 "트리거는 오는데 빌드는 없는" 침묵 실패와 똑같은 결의 함정이, 이번엔 동시성 쪽에서 났다.

핵심을 다시 보면 이렇다. 같은 `flock -n`이라는 **대칭 규칙**이, 성격이 정반대인 두 트리거에 똑같이 적용된 게 문제였다. sync run은 버려도 되는 것이고 timer run은 버리면 안 되는 것인데, 대칭은 그 차이를 모른다.

---

## 4. 비대칭이라는 답 — 누가 양보할지를 정한다

그래서 대칭을 깼다. 규칙을 한 줄로 줄이면 이렇다.

> **자주 오는 쪽은 양보하고(skip), 하루 한 번 오는 쪽은 기다린다(wait).**

문 앞에서 두 사람이 동시에 마주쳤다고 해 보자. 서로 "먼저 가세요"만 반복하면 아무도 못 지나가고(대칭의 교착), 반대로 둘 다 먼저 들어가려 하면 부딪힌다. 깔끔한 답은 **누가 양보할지를 미리 정해 두는 것**이다. 여기선 그 기준이 분명했다. 자주 오는 손님은 다음에 또 오니 이번엔 비켜 주면 되고, 하루에 딱 한 번 오는 손님은 이번을 놓치면 내일까지 기다려야 하니 먼저 보낸다.

이걸 코드로 옮기면 `flock` 호출 한 줄을 **조건부**로 바꾸는 일이 된다. 환경변수 `LOCK_WAIT`이 있느냐 없느냐로 갈린다.

```bash
# run-dbt.sh — 동시 실행 방지(가이드 §5.3)
# 기본은 non-blocking(skip): webhook이 연달아 와도 안 겹친다.
# LOCK_WAIT(초)이 설정되면 그만큼 대기(blocking): 하루 1회 전체 run이
# 진행 중인 webhook run에 밀려 스킵되지 않게 큐잉시키는 용도.
exec 9>/tmp/run-dbt.lock
if [ -n "${LOCK_WAIT:-}" ]; then
  flock -w "$LOCK_WAIT" 9 || {            # LOCK_WAIT초 동안 락을 기다린다
    notify_slack "🔴" "일일 dbt 시작 실패 — ${LOCK_WAIT}s 대기 후 락 미획득" \
                 "다른 run-dbt가 과도 점유 중 — 이번 예약 실행 건너뜀"
    exit 1
  }
else
  flock -n 9 || {                         # 즉시 포기(skip)
    echo "[run-dbt] 이미 실행 중 — skip"
    exit 0
  }
fi
```

읽는 법은 이렇다.

- **`LOCK_WAIT`이 없으면** → `flock -n` → 자물쇠가 잡혀 있으면 즉시 skip하고 exit 0. 이게 **sync webhook**이 타는 길이다. 양보하는 쪽.
- **`LOCK_WAIT`이 있으면**(예: `1800`) → `flock -w 1800` → 자물쇠가 풀릴 때까지 **최대 1800초(30분) 기다린다.** 그 안에 잡으면 진행, 못 잡으면 🔴 Slack 알림 후 exit 1. 이게 **timer**가 타는 길이다. 기다리는 쪽.

같은 스크립트, 같은 자물쇠다. 환경변수 하나가 있느냐 없느냐로 "양보"와 "대기"가 갈린다. 비대칭은 코드를 두 벌로 나눠서 만든 게 아니라, **호출하는 쪽이 `LOCK_WAIT`을 넘기느냐 마느냐**로 만들어진다.

---

## 5. 그 비대칭이 왜 맞는가

규칙은 단순한데, 양쪽 가지가 각각 왜 맞는지를 짚어 둘 가치가 있다. 동시성 코드는 "왜 이 방향으로 양보하는가"가 흐려지면 나중에 거꾸로 고쳐 놓기 쉽다.

**sync 훅이 양보(skip)해도 되는 이유.** 3장에서 말한 그대로다. 버려도 손해가 거의 없다. dbt run은 큐에 쌓인 일감을 비우는 게 아니라 매번 소스의 현재 상태를 통째로 다시 읽는다. 그래서 한 번의 sync run을 걸러도 데이터가 영구히 누락되지 않는다. 다음 sync(보통 멀지 않다)의 run이 그새 쌓인 분까지 같이 따라잡는다. skip은 손실이 아니라 지연일 뿐이고, 그 지연은 다음 트리거가 자동으로 메운다.

**timer가 기다려야(wait) 하는 이유.** 반대로 timer run은 버리면 그냥 손실이다. 다음 기회가 24시간 뒤라, "다음 트리거가 따라잡는" 자가복구가 안 통한다. 그래서 양보 대신 대기를 시킨다. 15:00에 sync run이 돌고 있으면 그게 끝날 때까지 자물쇠 앞에서 기다렸다가, 풀리는 즉시 잡아서 전체 run을 돌린다. sync run이 ~10분이니 보통은 몇 분 기다리다 잡는다.

**그럼 30분(`LOCK_WAIT=1800`)이라는 한도는 왜 두나.** 무한정 기다리게 둘 수도 있었지만, 그건 또 다른 침묵 실패의 씨앗이다. 어떤 run이 비정상으로 자물쇠를 30분 넘게 물고 있으면(행이 걸렸거나 비정상적으로 긴 쿼리), timer가 거기 영영 매달려 버린다. 그래서 상한을 두고, 그 안에 못 잡으면 **포기하되 조용히는 아니게** — 🔴 Slack 알림을 쏘고 exit 1로 끝낸다. "오늘 일일 run은 락을 못 잡아 못 돌았다"가 명시적으로 보이게. 3장에서 우리를 데인 게 바로 "조용한 skip"이었으니, 같은 실수를 비대칭 쪽에서 반복하지 않으려는 장치다.

정리하면, 비대칭의 방향은 **"잃었을 때 누가 스스로 복구되는가"**로 정해진다. 자가복구되는 쪽(잦은 sync)이 양보하고, 복구가 안 되는 쪽(하루 한 번 timer)이 자리를 지킨다.

---

## 6. 최대 함정 — `LOCK_WAIT`이 한쪽으로만 새야 한다

여기까지면 깔끔한데, 이 설계 전체를 무너뜨리는 함정이 하나 있다. 그리고 그게 코드가 아니라 **설정이 어디에 놓이느냐**의 문제라 더 미묘하다.

비대칭의 핵심은 `LOCK_WAIT`이 **timer에게만** 보이고 **webhook에게는 안 보이는** 것이다. 그런데 둘은 같은 `run-dbt.sh`를 부른다. 만약 `LOCK_WAIT`이 webhook이 읽는 환경에까지 새 버리면, **webhook run도 `flock -w` 가지로 빠져서 같이 기다리게 된다.** 그 순간 비대칭이 무너진다.

무너지면 어떻게 되나. 양보할 줄 알았던 sync run이 이제 안 양보하고 기다린다. 30분마다 줄줄이 들어오는 sync POST들이 자물쇠 앞에 큐로 쌓이고, 하나가 끝나면 다음이 잡고… 신선도가 핵심이던 sync 훅이 도리어 굼떠진다. 자주·가볍게 돌려고 만든 webhook이 timer의 대기 성질을 물려받아 버리는 거다.

그래서 **`LOCK_WAIT`은 `dbt-daily.service`의 인라인 `Environment=`에만** 둔다. 절대 webhook.service의 `Environment=`나 `EnvironmentFile=`, 또는 양쪽이 공유하는 env 파일에 넣지 않는다.

```ini
# /etc/systemd/system/dbt-daily.service   — timer만 LOCK_WAIT을 본다
[Service]
Type=oneshot
User=ec2-user
Environment=LOCK_WAIT=1800                 # ← 이 한 줄이 비대칭의 전부
ExecStart=/usr/local/bin/run-dbt.sh run
# ⚠️ EnvironmentFile은 두지 않는다. LOCK_WAIT은 위 인라인으로만 주입.
#    (webhook.service에는 이 줄이 없다 → webhook run은 LOCK_WAIT 미상속 → flock -n)
```

`LOCK_WAIT`을 systemd 유닛 안에 인라인으로 박은 건 멋이 아니라 **격리**다. 유닛별 `Environment=`는 그 유닛이 띄운 프로세스에만 상속된다. webhook.service에는 이 줄이 없으니 webhook이 부른 `run-dbt.sh`는 `LOCK_WAIT`을 못 받고, 자연히 `else` 가지(`flock -n`, skip)로 간다. timer가 부른 쪽만 받는다. 한 변수가 정확히 한쪽에만 보이게 하는 게 이 설계의 생명줄이다.

검증도 그래서 양쪽을 같이 본다.

```bash
systemctl show dbt-daily.service -p Environment   # LOCK_WAIT=1800 이 보여야 정상
systemctl show webhook          -p Environment   # 비어 있어야 정상(누수 없음)
```

webhook 쪽 출력이 비어 있지 않으면 그게 바로 누수 신호다. "sync run까지 큐잉되며 굼떠졌다" 싶으면 십중팔구 이 변수가 공유 env 파일로 흘러든 것이다.

---

## 7. timer 자체의 잔가지들 — UTC와 소유자

비대칭 락이 본론이지만, timer를 실제로 세우면서 발이 걸린 두 가지를 같이 적어 둔다. 둘 다 systemd timer를 처음 다룰 때 흔히 밟는 돌부리다.

**하나는 시각이 UTC라는 것.** EC2는 UTC로 돈다. "15:00 KST에 돌려라"를 OnCalendar에 그대로 `15:00`이라 적으면 한국 시간 자정에 돈다. KST는 UTC+9니까 **15:00 KST = 06:00 UTC**로 환산해 박아야 한다.

```ini
# /etc/systemd/system/dbt-daily.timer
[Timer]
OnCalendar=*-*-* 06:00:00      # 06:00 UTC = 15:00 KST
Persistent=true                # 그 시각에 꺼져 있었으면 부팅 후 보충 실행
[Install]
WantedBy=timers.target
```

`Persistent=true`는 작지만 중요하다. 06:00 UTC에 인스턴스가 꺼져 있었다면, 켜진 직후에 놓친 실행을 한 번 보충해 준다. 하루치 전체 run을 재부팅 타이밍 때문에 통째로 잃지 않게 하는 안전장치다. 환산이 헷갈리면 AL2023의 systemd(252+)는 `OnCalendar=*-*-* 15:00:00 Asia/Seoul`처럼 타임존을 직접 박는 것도 받아 주니, 의도를 더 분명히 하고 싶으면 그쪽이 낫다.

**다른 하나는 실행 소유자.** timer가 부른 `run-dbt.sh`는 `git pull`을 한다. 그런데 서비스가 root로 돌면, ec2-user 소유인 `/opt/blitz-analytics` 레포에서 git이 `detected dubious ownership`으로 거부한다. 그러면 pull이 실패하고 — 38편의 그 함정과 똑같이 — 빌드 없이 끝난다. 그래서 `[Service]`에 `User=ec2-user`를 명시한다(webhook.service엔 이미 있었다). 여기서 헷갈리기 쉬운데, root용 `safe.directory`를 추가하는 건 오답이다. root가 pull로 만든 파일이 root 소유가 되어 나중에 ec2-user 실행을 또 깨뜨린다. 정답은 git을 **레포 소유자(ec2-user)로 돌리는 것**이다.

마지막으로 운영 명령 한 줄. timer는 `enable` 대상이 아니라 `.timer`를 켜고 끈다(`.service`는 `[Install]`이 없어 timer가 호출한다).

```bash
systemctl list-timers dbt-daily.timer        # NEXT=06:00 UTC 인지 확인
sudo systemctl start dbt-daily.service       # 스케줄과 무관하게 즉시 1회(테스트)
sudo systemctl disable --now dbt-daily.timer # 일일 자동 실행만 끄기(webhook은 계속 돎)
```

타이머를 꺼도 webhook은 그대로 돈다. 둘은 독립이다. 끄면 "셀렉터 밖 모델·전체 run"만 멈추고, sync 신선도는 유지된다.

---

## 8. 일부러 비워 둔 칸 — timer는 `build`가 아니라 `run`이다

비대칭 락을 세우고 나서 한 가지를 의식적으로 비워 뒀다. 적어 두지 않으면 "왜 테스트가 안 도냐"는 질문이 반드시 따라올 자리다.

timer의 `ExecStart`는 `run-dbt.sh build`가 아니라 `run-dbt.sh run`이다. `dbt build`였다면 모델을 만들면서 `schema.yml`의 테스트(not_null·unique·relationships 등)까지 같이 돌고, 테스트가 실패하면 하류 마트를 SKIP시키는 게이트가 된다. 그런데 지금은 sync 훅도 `run`, 일일 timer도 `run`이라 **dbt 테스트가 자동으로 도는 곳이 어디에도 없다.**

이건 빠뜨린 게 아니라 고른 것이다. 38편에서 sync 훅을 `run`으로 둔 이유와 같다. 테스트 실패 하나가 비용 대시보드 신선도를 막아 버리는 게이트를, 자동 주기에 두고 싶지 않았다. 다만 sync 훅은 "셀렉터만"이라는 핑계라도 있었는데, 일일 전체 run마저 `run`이면 정말로 테스트가 도는 데가 없어진다. 그래서 이건 더 또렷하게 트레이드오프로 적어 둔다.

테스트를 자동화하고 싶어지면 손댈 곳은 한 줄이다. `dbt-daily.service`의 `ExecStart`를 `run-dbt.sh build`로 바꾸면, 하루 한 번 도는 전체 run이 곧 테스트 게이트가 된다. sync의 잦은 신선도는 `run`으로 가볍게 유지하면서 품질 검증만 하루 1회로 분리하는, 38편에서 말한 "신선도와 품질을 나눈다"는 그림이 그제야 완성된다. 그 전까진 데이터 검증은 수동 `run-dbt.sh build --select ...` 소관이다. 지금 굳이 안 켠 건, 게이트가 SKIP을 만들기 시작하면 그걸 지켜볼 사람이 상주해야 하는데 — 2장에서 말한 "2명이 간헐적으로 보는" 운영 조건에선 아직 그 부담을 질 때가 아니라고 봤기 때문이다.

---

## 정리

트리거가 둘이 되면서 같은 자물쇠를 공유하게 됐고, "겹칠 때 누가 양보하느냐"가 문제였다. 돌아보면 핵심은 짧다.

둘 다 똑같이 `flock -n`으로 두는 대칭은, 성격이 정반대인 트리거에 같은 규칙을 씌운 탓에 제일 잃으면 안 되는 일일 전체 run을 조용히 날렸다. 그래서 대칭을 일부러 깼다. **자가복구되는 쪽(잦은 sync)은 양보(skip)하고, 복구가 안 되는 쪽(하루 한 번 timer)은 기다린다(최대 30분).** 코드로는 `flock`을 `LOCK_WAIT` 유무로 가르는 조건부 한 덩이고, 그 비대칭이 깨지지 않으려면 `LOCK_WAIT`이 **timer 유닛 인라인에만** 있고 webhook 쪽엔 절대 새지 않아야 한다. 한 변수가 정확히 한쪽에만 보이게 하는 게 설계의 생명줄이었다.

곁가지로 timer를 세우며 UTC 환산(06:00 UTC = 15:00 KST)과 실행 소유자(`User=ec2-user`)에 발이 걸렸고, timer를 `build`가 아니라 `run`으로 둬서 테스트는 아직 자동으로 돌지 않는다는 칸은 일부러 비워 뒀다. 화려한 동시성 제어 대신 `flock` 한 줄을 비대칭으로 바꾼 이 가벼운 답이, 두 트리거가 한 자물쇠를 다투는 우리 상황에는 충분했다.

### 자주 쓰는 명령어

```bash
# 비대칭이 살아 있는지 — 양쪽을 같이 본다
systemctl show dbt-daily.service -p Environment   # LOCK_WAIT=1800 보여야 정상
systemctl show webhook          -p Environment   # 비어 있어야 정상(누수 없음)

# 일일 타이머 상태·즉시 실행·끄기
systemctl list-timers dbt-daily.timer             # NEXT=06:00 UTC(=15:00 KST)
sudo systemctl start   dbt-daily.service          # 스케줄 무관 즉시 1회(테스트)
journalctl -u dbt-daily.service -f                # 일일 run 로그
sudo systemctl disable --now dbt-daily.timer      # 일일 자동 실행만 끄기(webhook은 계속)

# 그날 무엇이 돌았든 한곳에서 — tee 파일은 유닛별로 안 갈린다
tail -f /var/log/dbt/dbt_$(date +%F).log
```
