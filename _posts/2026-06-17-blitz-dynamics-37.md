---
title: "[인턴] 실제 Airbyte 배포를 해부하며 배우는 쿠버네티스 워크로드 객체 (Pod·Deployment·Job·Service·ReplicaSet·StatefulSet)"
excerpt: "abctl로 띄운 진짜 Airbyte 클러스터의 kubectl 출력을 한 줄씩 뜯어보며, Pod·Deployment·Job·Service·ReplicaSet·StatefulSet이 서로 어떻게 연결되는지, 그리고 'Deployment와 Job은 같은 층인가'라는 질문의 답을 ReplicaSet 출력으로 증명한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Kubernetes, Airbyte, Pod, Deployment]

permalink: /intern/company/blitz-dynamics/37/

toc: true
toc_sticky: true

date: 2026-06-17
last_modified_at: 2026-06-17
---

## 들어가며

지난 시리즈에서 Fivetran을 걷어내고 **Airbyte OSS**로 CDC 파이프라인을 옮기는 작업을 다섯 편에 걸쳐 정리했다. 그 Airbyte는 EC2 한 대 위에 `abctl`로 설치돼 있는데, `abctl`은 내부적으로 [kind](https://kind.sigs.k8s.io/)(Kubernetes in Docker) 기반의 단일 노드 쿠버네티스 클러스터를 띄우고 그 위에 Airbyte를 올린다. 즉 내가 운영하던 Airbyte는 사실 **쿠버네티스 클러스터 하나**였다.

그래서 어느 날 무심코 `kubectl get pods`를 쳤다가, 화면 가득 찬 길고 복잡한 이름들을 보고 멈칫했다.

```
airbyte-abctl-server-6658446d68-zk6dh
```

왜 이름이 이렇게 길까? `airbyte-abctl-server`까지는 알겠는데 뒤의 `6658446d68`은 뭐고 `zk6dh`는 또 뭔가. 그런데 이 의문을 파고들다 보니, **이 한 화면 안에 쿠버네티스 워크로드 객체들의 관계가 전부 박혀 있다는 것**을 알게 됐다.

이 글은 교과서의 추상적인 정의를 외우는 글이 아니다. **실제로 돌아가는 Airbyte 클러스터의 `kubectl` 출력을 한 줄씩 해부하면서**, Pod·Deployment·Job·Service·ReplicaSet·StatefulSet이 어떻게 서로 연결되는지를 증거로 보여주려 한다. 글에 나오는 모든 명령어 출력은 가공하지 않은 진짜 데이터고, 독자도 자기 클러스터에서 똑같이 따라 칠 수 있는 명령어를 함께 적었다.

> 참고: `abctl`은 로컬/단일 노드 설치용 도구다. 여러 대의 노드에 걸쳐 도는 프로덕션 멀티노드 클러스터와는 운영의 결이 다르다. 하지만 "워크로드 객체들이 어떻게 연결되는가"라는 본질은 단일 노드든 멀티 노드든 동일하므로, 학습용으로는 더없이 좋은 해부 대상이다.

---

## 1. 먼저 전체 그림: Pod 목록 한 장

본론에 들어가기 전에, 우리가 앞으로 계속 들여다볼 "해부 대상"을 먼저 펼쳐 놓자.

```bash
kubectl get pods -n airbyte-abctl
```

```
NAME                                                 READY   STATUS      RESTARTS      AGE
airbyte-abctl-bootloader                             0/1     Completed   0             14d
airbyte-abctl-cron-8578657bcd-bhnlm                  1/1     Running     1 (14d ago)   14d
airbyte-abctl-manifest-server-658c87b6c7-4b4gb       1/1     Running     1 (14d ago)   14d
airbyte-abctl-server-6658446d68-zk6dh                1/1     Running     2 (14d ago)   14d
airbyte-abctl-temporal-547db8cf96-4ldkb              1/1     Running     1 (14d ago)   14d
airbyte-abctl-worker-7494c78855-xsmz4                1/1     Running     1 (14d ago)   14d
airbyte-abctl-workload-api-server-7849b89f9b-49x6j   1/1     Running     1 (14d ago)   14d
airbyte-abctl-workload-launcher-7bb7cb6d6b-x8t5q     1/1     Running     3 (14d ago)   14d
airbyte-db-0                                         1/1     Running     1 (14d ago)   14d
replication-job-157-attempt-0                        0/3     Completed   0             9m47s
```

이 한 화면을 키워드별로 분류해 보면 이미 많은 게 보인다.

- **`STATUS`가 `Running`인 것들** (`server`, `cron`, `temporal`, `worker`, `manifest-server`, `workload-api-server`, `workload-launcher`) — 계속 떠 있어야 하는 **상시 서비스**. 뒤에서 보겠지만 이들은 **Deployment**가 관리한다.
- **`STATUS`가 `Completed`인 것들** (`bootloader`, `replication-job-157-attempt-0`) — 한 번 실행되고 끝나는 **일회성 작업**. 이들은 **Job**이 만든다. 여기서 `Completed`는 오류가 아니라 **정상적으로 성공해서 끝났다는 뜻**이다.
- **`airbyte-db-0`** — 이름 끝에 `-0`이 붙은 특이한 녀석. 이게 바로 **StatefulSet**의 흔적이다 (Airbyte의 내부 PostgreSQL).

그리고 이름의 생김새가 세 부류로 나뉜다는 것도 눈여겨보자.

| 이름 형태 | 예시 | 정체 |
|---|---|---|
| `이름-해시-식별자` | `airbyte-abctl-server-6658446d68-zk6dh` | Deployment가 만든 Pod |
| `이름-attempt-N` | `replication-job-157-attempt-0` | Job이 만든 Pod |
| `이름-숫자` | `airbyte-db-0` | StatefulSet이 만든 Pod |
| (해시 없음) | `airbyte-abctl-bootloader` | Job이 만든 Pod |

이름만 봐도 누가 이 Pod를 만들었는지 짐작할 수 있다. 이 글의 목표는 이 "짐작"을 **증거로 못 박는 것**이다. 그러려면 먼저 객체 하나하나를 알아야 한다.

---

## 2. Pod — 실제로 일하는 작업자 한 명

쿠버네티스에서 컨테이너가 **실제로 실행되는 가장 작은 단위**가 Pod다. Pod는 IP를 하나 받고, CPU와 메모리를 써서 진짜로 일을 한다. 위 목록의 `airbyte-abctl-server-...`는 실제로 Airbyte의 API 서버 프로세스가 돌고 있는 실체다.

비유하자면 Pod는 **현장에서 실제로 일하는 작업자 한 명**이다. 그런데 이 작업자에게는 치명적인 약점이 하나 있다.

> **Pod는 자기 관리 능력이 없다.**

Pod가 어떤 이유로든 죽으면, 그냥 죽은 채로 남는다. 스스로 재시도하지도, 부활하지도 못한다. 작업자 한 명이 쓰러지면 그 자리는 그냥 빈자리가 되는 것과 같다. 누군가 옆에서 "쓰러졌네, 새 사람 투입해"라고 챙겨 주지 않으면 일이 멈춘다.

그래서 현실에서는 **Pod를 직접 만들지 않는다.** 대신 Pod를 대신 만들고, 죽으면 다시 띄워 주고, 개수를 관리해 주는 **상위 컨트롤러**에게 맡긴다. 그 컨트롤러가 누구냐에 따라 Pod의 성격이 갈린다. 끝나면 안 되는 서비스라면 **Deployment**가, 끝나는 게 정상인 작업이라면 **Job**이 맡는다.

---

## 3. Deployment vs Job — 상시 서비스와 일회성 작업

Pod를 관리하는 두 대표 컨트롤러는 **성격이 정반대**다. 이 대비를 이해하면 절반은 끝난 셈이다.

### Deployment — "항상 N개를 유지하라"

Deployment는 "이 Pod를 항상 N개 떠 있는 상태로 유지하라"고 책임지는 관리자다.

- Pod가 죽으면 즉시 새로 띄워서 원하는 개수(replica)를 계속 맞춘다.
- 버전을 올릴 때는 기존 Pod를 한 번에 죽이지 않고 **무중단으로 점진 교체**(rolling update)하며, 문제가 생기면 **롤백**한다.
- **끝나면 안 되는 상시 서비스**용이다. 그래서 정상 상태가 곧 `Running`이다.

비유하자면, **항상 N명이 일하도록 인원을 유지하고, 사람이 바뀌어도 업무가 끊기지 않게 무중단으로 교대시키는 관리자**다.

위 Pod 목록에서 계속 `Running`인 `server`, `cron`, `temporal`, `worker` 등이 모두 Deployment가 관리하는 상시 서비스다. 실제 Deployment 목록을 보면 이렇다.

```bash
kubectl get deployment -n airbyte-abctl
```

```
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
airbyte-abctl-cron                   1/1     1            1           14d
airbyte-abctl-manifest-server        1/1     1            1           14d
airbyte-abctl-server                 1/1     1            1           14d
airbyte-abctl-temporal               1/1     1            1           14d
airbyte-abctl-worker                 1/1     1            1           14d
airbyte-abctl-workload-api-server    1/1     1            1           14d
airbyte-abctl-workload-launcher      1/1     1            1           14d
```

`READY 1/1`은 "원하는 1개가 실제로 1개 떠 있다"는 뜻이다. 여기서 한 가지 짚고 넘어가자. **Deployment는 Pod를 직접 만들지 않는다.** 이건 이 글의 가장 중요한 정확성 포인트인데, 자세한 건 5장에서 ReplicaSet과 함께 다룬다.

> Airbyte의 `workload-api-server`, `workload-launcher` 계열 구성은 버전에 따라 바뀌는 과도기 아키텍처다. 여기 출력은 **이 글을 쓰던 시점의 버전 기준**이며, 버전이 다르면 구성이 조금 다를 수 있다.

### Job — "이 작업을 끝까지 완수시켜라"

Job은 Deployment와 정반대다. 목표가 **"이 작업이 성공할 때까지 책임지고, 끝나면 종료"**다.

- Pod가 도중에 실패하면 새 Pod를 띄워 재시도한다.
- 작업이 완료되면 Pod는 `Completed` 상태로 끝나고, 다시 띄우지 않는다.
- **끝나는 게 정상인 일회성 작업**용이다.

비유하자면, **"이 일을 끝까지 완수시켜라"는 지시서를 든 현장 감독**이다. 작업자가 도중에 쓰러지면 새 작업자를 투입해서라도 일을 끝낸다. 그리고 일이 끝나면 감독도 작업자도 해산한다.

위 Pod 목록의 `Completed` 두 줄이 정확히 이것이다.

```
airbyte-abctl-bootloader        0/1   Completed   0   14d        ← 설치 초기화 작업 (1회)
replication-job-157-attempt-0   0/3   Completed   0   9m47s      ← 데이터 동기화 작업
```

- `bootloader`는 Airbyte 설치 시 DB 스키마를 초기화하는 1회성 작업이다. 14일 전에 성공적으로 끝났고, 그 `Completed`가 그대로 남아 있다.
- `replication-job-157-attempt-0`은 9분 전에 돈 **실제 데이터 동기화 작업**이다. 이름의 `attempt-0`이 흥미롭다. "0번째 시도"라는 뜻으로, Job이 처음부터 "실패하면 재시도한다"는 전제로 설계됐음을 이름 자체가 드러낸다.

> `Completed`를 빨간불처럼 보지 말자. Job과 bootloader의 `Completed`는 **정상이고, 성공을 의미한다.** 오히려 Job 계열이 `Running`인 채로 오래 머물러 있다면 그게 더 의심스러운 신호다.

### Deployment vs Job

| | Deployment | Job |
|---|---|---|
| 목표 | 항상 N개 떠 있게 유지 | 작업을 끝까지 완수 후 종료 |
| 정상 상태 | `Running` (계속) | `Completed` (끝) |
| Pod가 죽으면 | 즉시 새로 띄움 | (작업 미완 시) 재시도 |
| 용도 | 상시 서비스 | 일회성 작업 |
| Airbyte 예 | server, cron, worker | bootloader, replication-job |


---

## 4. Service — 변하는 Pod들에 고정 주소를 주기

지금까지의 셋(Pod, Deployment, Job)이 "Pod를 만들고 관리"하는 쪽이라면, **Service**는 결이 다르다. Service는 "그 Pod에 어떻게 접속할 것인가"를 책임진다.

문제는 이것이다. **Pod는 죽고 다시 뜰 때마다 IP가 바뀐다.** Deployment가 Pod를 교체하면 새 Pod는 새 IP를 받는다. 만약 누군가 Pod의 IP를 직접 바라보고 있었다면, 그 Pod가 교체되는 순간 연결이 끊긴다.

Service는 이 문제를 **변하지 않는 가상 IP와 DNS 이름**을 하나 만들어 두는 것으로 푼다. 클라이언트는 그 고정 주소만 바라보고, Service가 뒤에 있는 (수시로 바뀌는) 실제 Pod들로 요청을 자동 분배(로드밸런싱)한다.

비유하자면 Service는 **건물 호수가 바뀌어도 변하지 않는 대표 전화번호**다. 직원이 몇 호 방으로 옮기든 대표번호로 걸면 연결된다.

실제로 내가 Airbyte UI에 접속할 때 쓰는 포트포워딩 명령이 이 Service를 대상으로 한다.

```bash
kubectl port-forward svc/airbyte-abctl-airbyte-server-svc 8000:80 -n airbyte-abctl
```

`svc/...`로 시작하는 이름이 바로 Service다. 뒤의 Airbyte 서버 Pod가 재시작돼 IP가 바뀌어도, 나는 이 Service 이름만 바라보면 되기 때문에 포트포워딩이 깨지지 않는다.

---

## 5. "Deployment와 Job은 같은 Plane 인가?"

여기서부터가 이 글의 클라이맥스다. 다음 질문을 던져 보자.

> **Deployment와 Job은 같은 계위(plane)에 있는가?**

직관적으로는 "둘 다 내가 YAML로 선언해서 다루는 워크로드 컨트롤러니까 같은 층 아닌가?"라고 답하기 쉽다. 그런데 정답은 "보는 관점에 따라 다르다"이다. 이 양면성을 이해하는 게 핵심이다. 그리고 그 열쇠는 지금까지 한 번도 직접 다루지 않은 숨은 객체, **ReplicaSet**에 있다.

### ReplicaSet의 등장

Deployment를 설명할 때 "Deployment는 Pod를 직접 만들지 않는다"고 흘리듯 말했다. 그 진실이 이것이다.

> **Deployment는 ReplicaSet을 만들고, 그 ReplicaSet이 Pod를 만든다.**

ReplicaSet은 Deployment와 Pod **사이에 있는 중간 컨트롤러**다. 실제로 "Pod 개수를 원하는 만큼 맞춰서 생성·유지하는" 궂은일을 하는 건 ReplicaSet이다. Deployment는 한 단계 위에서 "어떤 버전의 Pod를 몇 개" 원하는지 선언하고, 그에 맞는 ReplicaSet을 만들어 부린다.

비유하자면, Deployment가 "항상 N명 유지"를 책임지는 관리자라면, ReplicaSet은 그 관리자가 **"실제로 인원수를 N명으로 맞춰 놔"라고 시키는 중간 반장**이다. 사용자가 ReplicaSet을 직접 만들 일은 거의 없다. Deployment가 알아서 만들고 관리하기 때문이다.

### 두 가지 관점

이제 처음 질문으로 돌아가자.

**관점 A — 사용자가 다루는 최상위 워크로드 객체 (실용적 관점)**

우리가 직접 YAML로 선언하고 `kubectl get`으로 다루는 최상위 객체는 **Deployment**와 **Job**이다. ReplicaSet은 Deployment가 뒤에서 알아서 만드는 것이라 우리 손이 거의 안 닿는다. 이 관점에서는 **Deployment와 Job이 같은 층**이다.

**관점 B — 누가 Pod를 직접 만드는가 (내부 메커니즘 관점)**

Pod를 **직접** 생성하는 컨트롤러가 누구인지를 기준으로 보면 그림이 달라진다.

- **ReplicaSet**이 Pod를 직접 만든다.
- **Job**도 Pod를 직접 만든다. (Job은 ReplicaSet을 거치지 않는다!)
- 반면 **Deployment**는 Pod를 직접 만들지 않는다. 한 단계 위에서 ReplicaSet을 만들 뿐이다.

그래서 이 관점에서 **Job과 같은 층에 있는 것은 Deployment가 아니라 ReplicaSet**이다.

### 계층도

두 관점을 하나의 그림으로 정리하면 이렇다.

```
[ 상위 컨트롤러 ]      Deployment            (Job은 여기 없음)
   (사용자가 선언)          │
                            ▼
[ 하위 컨트롤러 ]      ReplicaSet  ◀── 같은 층 ──▶  Job
   (Pod 직접 생성)          │                        │
                            ▼                        ▼
[ 실행 단위 ]             Pod                      Pod
                       (Running)                (Completed)
```

핵심은 **경로의 단계 수가 다르다**는 것이다.

- **Deployment → ReplicaSet → Pod** : 3단계
- **Job → Pod** : 2단계 (ReplicaSet을 건너뛴다)

관점 A는 맨 윗줄(사용자가 선언하는 객체)에서 Deployment와 Job을 나란히 본 것이고, 관점 B는 가운데 줄(Pod를 직접 만드는 층)에서 ReplicaSet과 Job을 나란히 본 것이다. 둘 다 맞다. 그래서 "같은 층인가?"에 단답하면 부정확하다.

---

## 6. ReplicaSet

지금까지는 "이렇다더라"는 설명이었다. 이제 **진짜 출력으로 증명**해 보자. ReplicaSet 목록에 내용이 남아 있다.

```bash
kubectl get replicaset -n airbyte-abctl
```

```
NAME                                           DESIRED   CURRENT   READY   AGE
airbyte-abctl-cron-8578657bcd                  1         1         1       14d
airbyte-abctl-manifest-server-658c87b6c7       1         1         1       14d
airbyte-abctl-server-6658446d68                1         1         1       14d
airbyte-abctl-temporal-547db8cf96              1         1         1       14d
airbyte-abctl-worker-7494c78855                1         1         1       14d
airbyte-abctl-workload-api-server-7849b89f9b   1         1         1       14d
airbyte-abctl-workload-launcher-78bbfb5664     0         0         0       14d
airbyte-abctl-workload-launcher-7bb7cb6d6b     1         1         1       14d
```

이 한 장에서 세 가지가 동시에 증명된다.

### ① — 이름에 계위가 통째로 박혀 있다

`server` Pod와 그 ReplicaSet을 나란히 놓아 보자.

```
ReplicaSet :  airbyte-abctl-server-6658446d68
Pod        :  airbyte-abctl-server-6658446d68-zk6dh
                                   └────────┘ └───┘
                                   RS 해시     Pod 식별자
```

Pod 이름 `airbyte-abctl-server-6658446d68-zk6dh`의 구조는 정확히 **`[Deployment 이름]-[ReplicaSet 해시]-[Pod 식별자]`**다.

- `airbyte-abctl-server` → Deployment 이름
- `6658446d68` → ReplicaSet 해시 (위 ReplicaSet 목록의 이름과 **정확히 일치**한다)
- `zk6dh` → 이 Pod만의 고유 식별자

도입부에서 "왜 이름이 이렇게 길까?"라고 물었던 그 이름이, 알고 보니 **Deployment → ReplicaSet → Pod의 3단 계위를 그대로 새겨 놓은 것**이었다. 이름 하나가 곧 족보다.

### ② — Job에는 ReplicaSet이 없다

ReplicaSet 목록을 다시 훑어보자. 그리고 1장의 Pod 목록과 대조해 보자. 분명히 `Completed`였던 두 Job 계열 Pod —

```
airbyte-abctl-bootloader
replication-job-157-attempt-0
```

— 이들의 ReplicaSet이 위 목록에 **단 하나도 없다.** `bootloader`도 없고 `replication-job-157`도 없다.

이것이 바로 5장 "Job은 ReplicaSet을 거치지 않고 Pod를 직접 만든다"는 주장의 결정적 증거다. Job이 만든 Pod는 중간에 ReplicaSet을 두지 않기 때문에, ReplicaSet 목록에 흔적을 남기지 않는다. 그래서 Pod 이름도 `...-attempt-0`처럼 **해시 없이** 붙는 것이다. 반면 Deployment가 만든 Pod는 모두 중간 해시(=ReplicaSet)를 달고 있다.

이름의 생김새 차이가 곧 메커니즘의 차이였다.

### ③ — workload-launcher의 ReplicaSet이 2개인 이유 (rolling update의 흔적)

가장 흥미로운 부분이다. 목록에서 `workload-launcher`만 ReplicaSet이 **두 개**다.

```
airbyte-abctl-workload-launcher-78bbfb5664   0   0   0   ← 옛 버전 (DESIRED 0, 비워짐)
airbyte-abctl-workload-launcher-7bb7cb6d6b   1   1   1   ← 새 버전 (DESIRED 1, 활성)
```

하나는 `DESIRED 1`로 활성 상태고, 다른 하나는 `DESIRED 0`으로 텅 비어 있다. 이건 고장이 아니라 **rolling update(무중단 점진 교체)가 지나간 흔적**이다.

Deployment가 새 버전으로 업데이트되면 어떻게 동작하느냐 —

1. 기존 Pod를 한꺼번에 죽이지 **않는다.**
2. 대신 **새 ReplicaSet**(`7bb7cb6d6b`)을 추가로 만들어 새 버전 Pod를 띄운다.
3. 새 Pod가 정상이라는 게 확인되면, **옛 ReplicaSet**(`78bbfb5664`)의 DESIRED를 0으로 줄여 옛 Pod를 거둔다.

그 결과 옛 ReplicaSet은 `DESIRED 0`인 빈 껍데기로 남는다. 그런데 왜 **삭제하지 않고 남겨 둘까?** 답은 **롤백 대비**다. 새 버전에 문제가 생기면, 옛 ReplicaSet의 DESIRED를 다시 1로 올리기만 하면 즉시 이전 버전으로 되돌릴 수 있다. 화석처럼 남은 `DESIRED 0` ReplicaSet은 사실 **언제든 꺼내 쓸 수 있는 복구 지점**인 셈이다.

이 보존 개수는 Deployment의 `revisionHistoryLimit`(기본값 10)으로 제어된다. 즉 기본 설정이라면 최근 10개 버전까지 이런 빈 ReplicaSet을 롤백용으로 쟁여 둔다.

업데이트 이력은 다음 명령으로 직접 확인할 수 있다.

```bash
kubectl rollout history deployment/airbyte-abctl-workload-launcher -n airbyte-abctl
```

### ④ — ownerReferences로 lineage를 코드로 확인하기

이름과 출력으로 추론하는 것을 넘어, 쿠버네티스가 객체에 직접 새겨 둔 **소유 관계(ownerReferences)**를 꺼내 보면 계위를 확인할 수 있다.

```bash
kubectl get pod airbyte-abctl-server-6658446d68-zk6dh -n airbyte-abctl \
  -o jsonpath='{.metadata.ownerReferences}'
```

이 명령을 치면 해당 Pod의 주인이 `kind: ReplicaSet`, `name: airbyte-abctl-server-6658446d68`이라고 찍힌다. 반대로 Job이 만든 Pod에 같은 명령을 치면 주인이 `kind: Job`으로 나온다 — ReplicaSet이 아니라. ①과 ②에서 이름으로 추론한 계위가, 객체 메타데이터 레벨에서 그대로 확인되는 것이다.

---

## 7. StatefulSet과 `airbyte-db-0`

마지막으로 1장에서 미뤄 둔 수수께끼 하나. `airbyte-db-0`의 `-0`은 뭘까?

이건 **StatefulSet**이 만든 Pod다. StatefulSet은 Deployment의 사촌격으로, **상태(데이터)를 보존해야 하는 Pod**를 관리한다. Airbyte의 내부 PostgreSQL이 여기에 해당한다 — DB는 당연히 데이터를 잃으면 안 되니까.

Deployment가 만드는 Pod 이름이 `...-6658446d68-zk6dh`처럼 랜덤 해시로 끝나는 것과 달리, StatefulSet은 Pod에 **`-0`, `-1`, `-2`...처럼 안정적이고 순서가 있는 이름**을 부여한다. Pod가 죽었다 다시 떠도 `airbyte-db-0`은 계속 `airbyte-db-0`이고, 자기 전용 저장소(볼륨)에 다시 연결된다. 이름이 곧 정체성이자 자기 데이터로 가는 주소인 셈이다.

비유하자면 StatefulSet은 **각자 고정된 사물함 번호와 자리를 가진 직원들을 관리하는 관리자**다. 직원이 바뀌어도 "3번 사물함은 3번 자리 직원의 것"이라는 규칙은 변하지 않는다.

StatefulSet은 그 자체로 한 편을 따로 쓸 만한 주제(순서 보장 기동, 헤드리스 Service, PVC 템플릿 등)라, 여기서는 `-0` 접미어가 단서라는 것만 짚고 넘어간다. 다음에 기회가 되면 따로 해부해 보려 한다.

---

## 8. 정리

처음의 의문 — "`kubectl get pods`를 쳤는데 이름이 왜 이렇게 길까?" — 에서 시작해, 실제 Airbyte 클러스터를 해부하며 워크로드 객체들의 관계를 추적했다.

- **Pod**는 실제로 일하는 작업자 한 명. 자기 관리 능력이 없어 상위 컨트롤러가 필요하다.
- **Deployment**는 상시 서비스를 `Running`으로 유지하는 관리자. **Job**은 일회성 작업을 `Completed`로 완수시키는 감독. 성격이 정반대다.
- **Service**는 IP가 바뀌는 Pod들 앞에 고정 주소를 세워 주는 대표 전화번호.
- **ReplicaSet**은 Deployment와 Pod 사이의 숨은 중간 반장. 실제로 Pod를 직접 만드는 건 ReplicaSet과 Job이다.
- "Deployment와 Job은 같은 층인가?"의 답은 **관점에 따라 다르다.** 사용자가 다루는 최상위 객체로는 Deployment·Job이 같은 층, Pod를 직접 만드는 메커니즘으로는 ReplicaSet·Job이 같은 층이다.
- 그리고 이 모든 주장이 **Pod 이름의 구조, ReplicaSet 목록, ownerReferences**라는 실제 출력 안에 증거로 박혀 있었다. Job에 ReplicaSet이 없다는 사실, workload-launcher의 빈 ReplicaSet이 rolling update의 화석이라는 사실까지.

쿠버네티스 객체는 추상적인 개념이 아니라, 내가 매일 운영하던 Airbyte 클러스터 안에서 **이름과 상태로 살아 움직이는 실체**였다. 다음에 `kubectl get pods`를 칠 일이 있다면, 긴 이름을 그냥 넘기지 말고 한 번 뜯어보길 권한다. 그 안에 클러스터의 족보가 들어 있다.

### 직접 따라 해 볼 명령어 모음

```bash
# 모든 네임스페이스의 Pod 보기
kubectl get pods -A

# 특정 네임스페이스 Pod
kubectl get pods -n airbyte-abctl

# Deployment 목록
kubectl get deployment -n airbyte-abctl

# Deployment 뒤에 숨은 ReplicaSet 확인
kubectl get replicaset -n airbyte-abctl

# rolling update 이력(리비전) 확인
kubectl rollout history deployment/airbyte-abctl-workload-launcher -n airbyte-abctl

# 한 Pod의 소유 관계(ownerReferences)까지 들여다보기 — 계위 직접 확인용
kubectl get pod <POD_NAME> -n airbyte-abctl -o jsonpath='{.metadata.ownerReferences}'
```
