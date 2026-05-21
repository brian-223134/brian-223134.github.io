---
title: "[인턴] Agent가 코드를 쓰는 시대의 테스트 작성 관점 — coverage가 아니라 정책을 고정하기"
excerpt: "비개발자가 agent로 코드를 만들고 개발자가 검수하는 워크플로우에서, 테스트가 자꾸 빈약해지는 문제를 마주하며 다시 정리한 테스트 작성의 기준"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 테스트, 테스트전략, 시나리오테스트, 함수테스트, coverage, agent, AI코딩]

permalink: /intern/company/blitz-dynamics/31/

toc: true
toc_sticky: true

date: 2026-05-21
last_modified_at: 2026-05-21
---

## 들어가며

요즘 회사에서는 **비개발자가 agent 기반 코드 생성 툴로 소규모 프로덕션 코드를 만들고, 개발자가 2차로 검수·보완하는 워크플로우**를 운영한다. 처음에는 잘 굴러가는 듯 보였지만 시간이 지나면서 한 패턴이 반복적으로 눈에 띄었다.

**테스트가 자꾸 빈약해진다는 것.**

증상은 다양했다. 핵심 정책을 검증해야 할 자리에 단순 happy path 한두 줄만 들어 있거나, 함수가 호출됐는지만 보는 mock-heavy 테스트가 늘어나거나, coverage 수치는 높은데 정작 회귀가 발생했을 때 잡아주는 테스트가 없었다. agent도 사람 리뷰어도 "테스트가 있긴 있다"는 사실에 의존해 한 단계 검증을 건너뛰는 일이 잦았다.

이쯤에서 한 번 관점을 다시 잡을 필요가 있었다. 이 글은 그 정리다. 사람이 직접 쓰는 테스트뿐 아니라 **agent가 작성·수정하는 테스트에서도 무너지지 않는 기준**을 만드는 것이 목표였다.

> 예시 코드는 회사 내부에서 운영 중인 어떤 서비스의 일부 테스트 코드를 보고 일반화한 형태. 메트릭 prefix·도메인 용어는 모두 추상화했고, 작성 패턴과 의도 위주로 발췌한다.

## 1. 테스트가 실제로 하는 일

먼저 테스트가 코드에 무엇을 해주는지부터 정렬했다. 테스트는 단순히 "버그를 찾기 위한 도구"가 아니라 세 가지 일을 동시에 한다.

**1) 요구사항 고정**
사용자가 기대하는 동작, 도메인 정책, 실패 처리 방식을 *실행 가능한 형태로* 남긴다. 문서는 시간이 지나면 코드와 어긋나지만, 통과하는 테스트는 어긋나지 않는다.

**2) 회귀 방지**
나중에 사람이든 agent든 리팩터링을 하다가 중요한 동작을 깨면 즉시 알게 한다. 특히 agent는 "전체 의도"보다는 "지금 변경 대상"에 집중하기 때문에, 변경 범위 밖의 정책이 깨지는 사고가 자주 난다.

**3) 설계 피드백**
테스트하기 너무 어렵다면 그 코드가 과하게 결합되어 있거나, side effect와 domain logic이 섞여 있을 가능성이 크다. "테스트가 잘 안 짜진다"는 코드 냄새의 신호다.

그래서 좋은 테스트의 기준은 단순히 "많이 통과하는가"가 아니라 **"깨졌을 때 무엇이 깨졌는지 명확히 말해주는가"** 다.

## 2. 시나리오 테스트 vs 함수 테스트

기본값은 **시나리오 테스트**가 좋다. 특히 agent가 코드를 작성하는 워크플로우에서는 더 그렇다.

```typescript
it('buildSpawnOptions 실패 시 release(pending) 호출 후 원래 에러를 throw 한다', async () => {});
it('process tree 합산 중 scrape 사이에 사라진 descendant PID 는 skip 한다', () => {});
```

이런 테스트는 **무슨 동작을 보존해야 하는지가 한 줄로 분명**하다. agent도 테스트명을 읽고 "아 이건 cleanup 호출 순서와 에러 전파 정책을 검증하는 거구나"를 빠르게 잡아낸다.

하지만 다음 종류의 코드는 **함수 정당성 테스트**가 추가로 필요하다.

- parser / formatter
- escaping / encoding
- scoring / ranking
- permission check
- 상태 전이 (state machine)
- tree / graph traversal
- quota / usage / money 계산
- date / time 계산
- retry / backoff 정책

예를 들면 이런 테스트.

```typescript
it('/proc stat comm 필드에 공백과 괄호가 있어도 마지막 닫는 괄호 기준으로 파싱한다', () => {});
it('Prometheus label 값의 quote, backslash, newline 을 escape 한다', () => {});
```

이건 시나리오 테스트만으로는 부족하다. 작은 함수지만 edge case가 많고, **틀리면 여러 상위 기능이 동시에 깨지기** 때문이다. comm 필드에 괄호가 들어간 케이스 하나만 놓쳐도 process tree 합산, 모니터링 지표, 알람 임계까지 줄줄이 잘못된 값을 띄울 수 있다.

실무적인 균형은 대략 이렇게 잡는다.

| 테스트 종류 | 무엇을 검증 | 비중 |
|---|---|---|
| 시나리오 | 핵심 user/domain behavior | 가장 많이 |
| 함수 | 작지만 복잡한 pure/domain logic | 위험 부위에 |
| integration | DB, filesystem, network, framework boundary | 경계마다 |
| E2E | 정말 중요한 대표 workflow | 소수만 |

## 3. 테스트 대상은 "가장 작은 안정적인 경계"로 잡는다

agent에게 "이 함수 테스트 짜줘"라고 시키면 종종 **private helper 하나하나에 단위 테스트를 붙이는** 결과물을 준다. coverage는 올라가지만, 이후 리팩터링을 시작하는 순간 거의 모든 테스트가 깨진다. 내부 구현을 그대로 박제했기 때문.

좋은 기준은 다음과 같다.

- 이 동작이 **외부에서 관찰 가능한 정책**인가? → 시나리오 테스트
- 이 함수가 작지만 복잡하고 **여러 곳에서 재사용**되는가? → 함수 테스트
- 이 코드가 **DB / FS / API / framework와 만나는 경계**인가? → integration / contract 테스트
- 단순 pass-through / private helper인가? → 직접 테스트하지 말고 **상위 테스트로 커버**

요점은 이거다. 테스트는 작을수록 좋은 게 아니라, **깨졌을 때 의미 있는 경계에 걸려 있어야** 한다.

## 4. Coverage를 보는 관점

가장 많이 오해받는 지표가 coverage다. coverage는 **목표가 아니라 탐색 도구**다.

- Line coverage → 어떤 줄이 실행됐는가
- Branch coverage → if / catch / edge 분기가 실행됐는가
- Function coverage → 어떤 함수가 아예 호출됐는가

coverage가 낮으면 "테스트가 부족할 수 있다"는 신호다. 하지만 coverage가 높다고 좋은 테스트라는 보장은 전혀 없다. 가짜 assert로 덮은 99% 도 99%다.

예를 들어 orchestration 레이어 테스트(`runner.test.ts`)에서 git 유틸이 0%로 나오는 건 문제라고 보기 어렵다. runner 테스트는 실제 git 호출을 검증하는 테스트가 아니라 **orchestration 흐름**을 검증하는 테스트이고, git은 mock하는 게 맞기 때문.

반대로 process tree 합산 코드에서 `/proc` reader path가 한 번도 안 지나갔다면, 그건 **의미 있는 구멍**이다. 실제 CPU / memory 계산 경로가 domain 기능의 핵심이기 때문.

즉 coverage는 이렇게 써야 한다.

1. 먼저 의미 있는 테스트를 쓴다
2. coverage를 본다
3. 빠진 줄 / 분기가 **중요한 정책인지** 판단한다
4. 중요하면 테스트를 추가한다
5. 중요하지 않거나 다른 테스트의 책임이면 제외하거나 무시한다

agent에게 "coverage를 80%까지 올려줘"라고 시키면 거의 항상 가짜 테스트가 생긴다. 시키지 말아야 할 명령에 가깝다.

## 5. Mock과 fake의 원칙

Mock은 **외부 세계와의 경계**에만 쓰는 게 좋다.

**좋은 mock 대상**

- git command
- network 호출
- clock / random
- filesystem 일부
- DB pool
- process manager
- third-party API

**조심해야 할 mock 대상**

- 내가 검증하려는 domain logic 자체
- 중요한 branch를 결정하는 함수
- 테스트 대상 내부의 대부분의 helper

mock이 많아질수록 테스트는 **"진짜 동작"이 아니라 "내가 상상한 호출 순서"** 를 검증하게 된다. agent가 작성하는 테스트에서 특히 흔한 실패 모드 — 구현체를 보고 mock을 그대로 거울처럼 깔아두기 때문.

가능하면 pure logic은 mock 없이 테스트하고, side effect는 **dependency injection으로 바깥에서 주입**하는 구조가 좋다. 실제로 가짜 `/proc` 디렉터리를 mkdtemp로 만들어 통째로 넘기는 패턴이 잘 동작했다.

```typescript
function withFakeProc<T>(fn: (procRoot: string) => T): T {
  const procRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'proc-test-'));
  try {
    return fn(procRoot);
  } finally {
    fs.rmSync(procRoot, { recursive: true, force: true });
  }
}
```

`/proc` 자체를 mock하는 게 아니라 **현실적인 디렉터리 구조를 만들어 같은 코드로 읽게** 한다. domain logic은 손대지 않고 환경만 격리하는 방식.

## 6. 좋은 테스트명의 기준

테스트명은 거의 **요구사항 문장처럼** 써야 한다.

좋은 형태

- 상황 + 기대 동작
- 상황 + 정책 + 이유
- 실패 조건 + 복구 동작

예시

```typescript
it('attempt 기록 실패는 본 작업 spawn 을 막지 않는다', async () => {});
it('root PID resource 를 못 읽으면 child 만으로 합산하지 않는다', () => {});
it('process tree scrape 실패해도 기존 parent-only metric 은 계속 출력한다', () => {});
```

나쁜 형태

```typescript
it('works', () => {});
it('handles error', () => {});
it('case 1', () => {});
```

agent workflow에서는 테스트명이 특히 중요하다. 테스트명이 모호하면 agent가 **"통과만 하는 구현"으로 우회**하기 쉽다. 반대로 테스트명이 정책을 분명히 말하고 있으면, agent도 그 정책을 우회하는 구현을 쓰기가 어려워진다.

## 7. Assert는 구현이 아니라 정책에 건다

좋은 assert는 **관찰 가능한 결과**를 본다.

```typescript
expect(result).toEqual({
  cpuSecondsTotal: 45,
  memoryRssBytes: 4500,
  processCount: 9,
});

expect(metrics).toContain('process_tree_scrape_error{db_id="50",pid="5000",state="working"} 1');
expect(metrics).not.toContain('process_tree_cpu_seconds_total{db_id="50"');
```

조심해야 할 assert

```typescript
expect(fn).toHaveBeenCalledTimes(3);
expect(mock.calls[0][0]).toEqual(...);
```

이런 assert도 필요할 때가 있다. 호출 횟수 자체가 정책인 경우(예: 재시도 정책, 캐시 hit/miss)라면 정당하다. 하지만 **호출 횟수나 순서가 진짜 정책이 아닐 때** 거기에 assert를 걸면, 내부 구현을 살짝 정리만 해도 테스트가 우수수 깨지는 brittle한 테스트가 된다.

또 한 가지, **민감한 값이 metric/log에 노출되지 않는다** 같은 negative assert는 직접 명시하는 것이 좋다.

```typescript
expect(metrics).not.toContain('hidden-from-metrics');
```

이런 한 줄이 "session 식별자를 외부 노출 metric에 절대 싣지 않는다"는 정책을 코드화해 둔다. 시간이 지나도 누군가 무심코 label에 끼워 넣는 순간 빨간 줄이 뜬다.

## 8. Agent 시대에 특히 중요한 작성법

agent가 테스트를 보고 구현하는 워크플로우에서는 **테스트가 사실상 spec**이다. 그래서 다음이 중요하다.

- 테스트명에 **의도**를 분명히 쓴다
- fixture는 **작지만 현실적인** 데이터로 만든다
- "왜 이 edge case가 중요한지"는 **짧은 주석**으로 남긴다
- snapshot 남발을 피한다 — agent는 snapshot을 무지성으로 업데이트하기 쉽다
- random / time / path / network를 고정한다
- **실패 메시지만 봐도** 무엇이 깨졌는지 알 수 있게 한다
- coverage를 보고 빠진 domain path를 확인한다
- 테스트가 구현을 과하게 고정하지 않게 한다

특히 좋은 테스트는 **agent가 꼼수로 통과시키기 어렵다**. 예를 들어 "함수가 호출됐다"가 아니라 "실패해도 본업은 계속 진행된다" 같은 정책을 검증하도록 짠다.

```typescript
it('attempt 기록 실패는 본 작업 spawn 을 막지 않는다', async () => {
  let started = false;

  await runJob(input, makeDeps({
    attemptsStore: {
      countAttemptsForParent: async () => {
        throw new Error('db temporarily unavailable');
      },
      insertAttempt: async () => 1,
    },
    sessionManager: {
      start: async () => {
        started = true;
        return { taskId: 'started' };
      },
    },
  }));

  expect(started).toBe(true);
});
```

이 테스트의 핵심은 coverage가 아니라 **정책**이다.

> 관측 / 모니터링 실패는 본 작업 시작을 막지 않는다.

이 정책이 코드로 박혀 있는 한, 누가 어떻게 리팩터링해도 "DB가 잠깐 흔들렸을 때 본 작업이 통째로 멈추는" 회귀는 막을 수 있다.

## 9. 자주 마주친 테스트 냄새

리뷰하면서 반복해서 마주친 패턴들.

- 테스트명이 요구사항을 설명하지 못함
- implementation을 그대로 복사해서 expected를 만듦
- mock이 너무 많아서 실제 domain logic이 거의 안 돎
- private helper만 억지로 테스트함
- coverage 숫자를 올리기 위한 무의미한 테스트
- 큰 snapshot을 agent가 계속 업데이트함
- 시간 / random / network 때문에 flaky함
- 테스트가 실패했는데 원인을 알기 어려움
- **성공 케이스만 있고 실패 / 경계 / 권한 케이스가 없음**

마지막 항목이 특히 많았다. agent는 자연스럽게 happy path를 생성한다. 권한 거부, 외부 의존성 실패, 입력 깨짐, 동시성 race 같은 케이스는 사람이 명시적으로 요구해야 들어온다.

## 10. 작성 순서 — 실무에서 쓰는 체크리스트

내가 가져가는 순서는 이렇다.

1. **이 변경의 핵심 정책을 한 문장으로 쓴다**
2. 그 정책을 대표하는 **시나리오 테스트**를 먼저 쓴다
3. 내부에 parser / 계산 / 상태전이 같은 복잡한 함수가 있으면 **함수 테스트**를 추가한다
4. 외부 의존성은 **mock / fake로 격리**한다
5. **실패 / edge / security / race 케이스**를 최소한으로 추가한다
6. targeted test를 돌린다
7. coverage를 보고 **중요한 미실행 branch**가 있는지 확인한다
8. 관련 test suite 또는 전체 test를 돌린다

agent에게 시킬 때도 동일하다. "테스트 짜줘"가 아니라 "**이 변경의 핵심 정책은 X다. X를 보존하는 시나리오 테스트 먼저 짜고, parser는 함수 테스트로 정당성 증명해**"처럼 단계와 의도를 함께 넘기면 결과물 품질이 눈에 띄게 올라간다.

## 11. 한 가지 작은 예시 — 정책을 어떻게 테스트로 박는가

요점만 잡힌 패턴을 하나 보여주는 게 가장 빠르다. 아래는 "session resource metric을 Prometheus 포맷으로 렌더링한다"는 작은 모듈의 테스트 일부를 일반화한 예시다.

```typescript
it('PID 가 아직 없는 세션은 resource metric 에서 제외한다', () => {
  const metrics = renderSessionResourceMetrics(
    [makeHandle(makeInfo({ dbId: 43, pid: undefined }))],
    () => { throw new Error('should not read pidless process'); },
    () => { throw new Error('should not read pidless process tree'); },
  );

  expect(metrics).not.toContain('db_id="43"');
});
```

이 테스트가 보존하는 정책은 두 가지다.

1. **PID가 아직 없으면 metric 라인 자체가 나가지 않는다** → 잘못된 값으로 alert가 울리지 않게
2. **PID 없는 세션에 대해서는 process scrape를 호출조차 하지 않는다** → 불필요한 syscall / 노이즈 방지

두 번째는 `throw`로 박아두는 게 핵심이다. `expect(fn).not.toHaveBeenCalled()` 보다 강하게, **함수 자체가 호출되면 테스트가 폭발**하도록 강제한다. 누군가 무심코 "PID 없어도 일단 한번 읽어보자" 같은 코드를 넣으면 즉시 빨갛게 변한다.

```typescript
it('process tree scrape 실패해도 기존 parent-only metric 은 계속 출력한다', () => {
  const metrics = renderSessionResourceMetrics(
    [makeHandle(makeInfo({ dbId: 50, pid: 5000, state: 'working' }))],
    () => ({ cpuSecondsTotal: 10, memoryRssBytes: 1000 }),
    () => undefined,
  );

  expect(metrics).toContain('session_resource_scrape_error{db_id="50",pid="5000",state="working"} 0');
  expect(metrics).toContain('session_cpu_seconds_total{db_id="50",pid="5000",state="working"} 10');
  expect(metrics).toContain('session_process_tree_scrape_error{db_id="50",pid="5000",state="working"} 1');
  expect(metrics).not.toContain('session_process_tree_cpu_seconds_total{db_id="50"');
});
```

이 테스트가 박는 정책은 더 분명하다.

> 새 process-tree metric이 실패해도, 기존 dashboard / PromQL이 의존하는 parent-only series는 그대로 유지된다.

이 한 줄이 보존되는 한, 새로 추가하는 metric의 실패가 기존 monitoring을 통째로 깨는 사고는 막을 수 있다. 그리고 누군가가 "에러 났으니 metric 다 빼버리자"는 식으로 손대는 순간 즉시 빨갛게 된다.

## 12. 결론

테스트에 대한 관점은 결국 한 줄로 압축된다.

> 테스트는 coverage 숫자를 만들기 위한 코드가 아니다.
> 테스트는 **중요한 동작을 실행 가능한 언어로 고정**하는 장치다.

- **시나리오 테스트**로 의도를 고정하고,
- **함수 테스트**로 복잡한 domain logic의 정당성을 증명하고,
- **coverage**로 빠진 위험 지점을 찾는다.

이 균형을 잡으면 사람에게도 좋고, agent에게도 좋은 테스트 코드가 된다. 특히 비개발자 + agent + 개발자 리뷰가 섞인 워크플로우에서는, **사람이 만든 "정책 중심의 시나리오 테스트"가 agent의 한계를 가장 잘 막아주는 안전망**이라는 것이 이번에 다시 확인한 결론이었다.
