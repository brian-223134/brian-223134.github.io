---
title: "[인턴] NestJS에서 Jest를 이용한 단위 테스트 작성하기"
excerpt: "spec.ts 파일 구조부터 Testing Module, Mocking, 실무 테스트 패턴까지 NestJS 백엔드 단위 테스트의 모든 것"

categories:
    - Company
tags:
    - [블리츠다이나믹스, 산학, 인턴, 백엔드, NestJS, Jest, 단위테스트, TypeScript]

permalink: /intern/company/blitz-dynamics/14/

toc: true
toc_sticky: true

date: 2026-03-23
last_modified_at: 2026-03-23
---

# NestJS + Jest: spec.ts 단위 테스트 완벽 정리

## 들어가며

처음 회사 프로젝트에 들어갔을 때, 나는 spec.ts가 뭔지도 모르고 그냥 기존 코드를 복사해서 붙여 넣고 있었다.
"테스트 왜 이렇게 복잡해? 그냥 손으로 테스트하면 되지 않나?" 이런 생각도 했었다.

하지만 3개월 정도 실무를 하다보니 깨달았다. **테스트 없이 대규모 리팩토링은 불가능**하다는 것을. 
코드를 바꾸고 배포했는데 갑자기 에러가 터지고, 2시간을 소모해서 원점복귀하고... 이런 일이 반복되었다.

이번 글에서는 내가 겪었던 시행착오와 배운 점들을 중심으로 NestJS + Jest 테스팅을 정리해본다.

---

## 1부: Jest와 spec.ts는 뭔가?

처음 팀장분이 "테스트 커버리지 60% 이상으로 올려야 해"라고 하셨을 때, 나는 멘붕했다.
 spec.ts 파일을 본 적도 없었기 때문이다.

### Jest란?

Jest는 Facebook에서 만든 테스트 프레임워크인데, NestJS 프로젝트를 생성하면 기본으로 탑재된다.
`jest.config.js`를 보면 자동으로 `**/*.spec.ts` 패턴을 찾아서 실행하도록 설정되어 있다.

처음엔 이게 왜 자동으로 실행되는지도 몰랐다. 나중에 `npm run test`를 치면 터미널에 테스트 결과가 뜨는데, 
그게 spec.ts 파일들을 Jest가 찾아서 실행한 결과라는 걸 깨달았다.

### spec.ts와 실제 로직

```
src/
├── user/
│   ├── user.service.ts       ← 실제 로직 (당신이 지금 짜는 코드)
│   ├── user.service.spec.ts  ← 테스트 코드 (이 파일이 spec.ts)
│   ├── user.controller.ts
│   └── user.controller.spec.ts
```

처음엔 혼동했다. "아, 같은 파일을 두 번 쓰는 건가?" 하고.

아니다. `user.service.ts`는 **실제 비즈니스 로직**이고, `user.service.spec.ts`는 
**그 로직이 정말 맞게 작동하는지 자동으로 확인해주는 코드**다.

처음엔 이 둘을 왜 나눠야 하는지 이해 안 됐는데,
나중에 "아, 리팩토링할 때 spec.ts가 있으면 기존 기능이 깨지지 않는지 바로 알 수 있구나"라고 깨달았다.

---

## 2부: Testing Module — 내 첫 spec.ts 경험

처음으로 작성한 spec.ts는 사수가 주는 템플릿을 그대로 복사한 거였다.
코드는 돌아갔지만, 이게 정확히 뭘 하는 건지는 전혀 몰랐다.

```typescript
// user.service.spec.ts

import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [UserService],
    }).compile();

    service = module.get<UserService>(UserService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

나중에야 이 코드가 뭘 하는지 이해했다:

- **describe()**: "UserService라는 주제로 여러 테스트를 모아둘 거야"라는 선언이다.
- **beforeEach()**: "각 테스트를 실행하기 전에, 먼저 이 코드를 실행해줘"라는 뜻이다.
  - `Test.createTestingModule()`: NestJS의 DI 컨테이너를 미니어처 버전으로 만드는 거다.
  - `providers: [UserService]`: "이 미니 컨테이너에 UserService를 넣어줘"
  - `.compile()`: "자, 이제 준비됐으니까 실제로 만들어줘"
- **module.get<UserService>()**: "미니 컨테이너에서 UserService 인스턴스를 꺼내줘"
- **it()**: "다음은 하나의 테스트 케이스다"
- **expect()**: "결과가 이거여야 해" 라는 주장이다.

맨 아래 `should be defined`는 사실 "서비스가 정상으로 생성되었는가"를 확인하는 가장 단순한 테스트다.
나도 처음엔 이게 뭐 하는 테스트인지 의미를 못 알았다.

---

## 3부: Mocking — 내가 가장 오래 헤맸던 부분

여기가 내가 가장 이해하기 힘들었던 부분이다. "왜 자꾸 Mock을 쓴대?" "Real 데이터로 하면 안 돼?" 이렇게 물었던 기억이 난다.

### 실제 상황: 이메일 테스트

내가 `UserService`를 만들었는데, 여기가 이메일을 발송하는 로직을 포함하고 있다:

```typescript
// user.service.ts

import { EmailService } from './email.service';

@Injectable()
export class UserService {
  constructor(private emailService: EmailService) {}

  async createUser(email: string): Promise<void> {
    // 유저 생성...
    await this.emailService.sendVerification(email); // 요기서 이메일 발송!
  }
}
```

"그럼 이거 테스트할 때마다 진짜 이메일을 보낼 건가?" 이게 나의 첫 질문이었다.
당신이 `npm run test`할 때마다 실제로 이메일이 발송된다면?
- 테스트할 때마다 수십 개의 스팸 이메일이 날아간다 ❌
- 이메일 서버가 느리면 테스트도 느려진다 ❌  
- 이메일 서버가 다운되면 테스트도 실패한다 ❌
- 진짜 고객 데이터를 쓰면 개인정보 문제도 생긴다 ❌

이게 Mock을 써야 하는 이유다.

### 해결책: Mock으로 EmailService를 페이크로 바꾼다

```typescript
// user.service.spec.ts

describe('UserService', () => {
  let service: UserService;
  let emailService: EmailService;

  beforeEach(async () => {
    // EmailService를 통째로 가짜로 바꾼다
    const mockEmailService = {
      sendVerification: jest.fn().mockResolvedValue(undefined),
      // ↑ 진짜 이메일을 보내는 대신, 호출만 기록하는 페이크 함수
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: EmailService,
          useValue: mockEmailService, // "뭐, EmailService 달라고? 자, 이거 써"
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    emailService = module.get<EmailService>(EmailService);
  });

  it('should send verification email when user is created', async () => {
    // 테스트 실행
    await service.createUser('test@example.com');

    // 진짜 이메일이 발송되지 않았지만, 
    // 함수가 호출되었는지는 추적할 수 있다
    expect(emailService.sendVerification).toHaveBeenCalledWith('test@example.com');
  });
});
```

이렇게 하면?
- 실제 이메일은 발송되지 않는다 ✅
- 하지만 "sendVerification이 호출되었나"는 추적할 수 있다 ✅
- 테스트가 빠르다 ✅

**내가 깨달은 핵심**:

- `jest.fn()`: "이 함수가 진짜로 뭘 하는지는 상관없고, 그냥 호출됐는지만 추적해줘"
- `.mockResolvedValue()`: "이 함수는 Promise를 반환해야 해. 아무 값이나 반환하고 즉시 resolve 해줘"
- `toHaveBeenCalledWith()`: "정말로 이 함수가 이 인자로 호출됐는지 확인해줘"

---

## 4부: 실무 테스트 패턴 —코드 리뷰로 배운 것들

처음 내가 쓴 테스트는 정말 엉망이었다. 모킹도 제대로 안 했고, 검증도 엉성했다.
사수가 코드 리뷰를 해주면서 "AAA 패턴을 써봐"라고 조언해줬다. 그때부터 테스트가 정말 읽기 쉬워졌다.

### 패턴 1: Happy Path (정상 흐름)

```typescript
it('should return user when ID exists', async () => {
  const userId = 1;
  const expectedUser = { id: 1, email: 'test@example.com' };

  // Arrange: 테스트 준비하기
  jest.spyOn(userRepository, 'findOne')
    .mockResolvedValue(expectedUser);
    // ↑ "DB에서 조회하면 expectedUser를 반환해줘"

  // Act: 실제로 함수 호출하기
  const result = await service.getUserById(userId);

  // Assert: 결과 검증하기
  expect(result).toEqual(expectedUser);
  // ↑ "반환된 결과가 정확히 expectedUser여야 해"
});
```

이게 AAA 패턴이다:
- **Arrange**: "테스트할 데이터와 환경을 준비해"
- **Act**: "실제로 함수를 호출해"
- **Assert**: "결과를 검증해"

이렇게 세 부분을 명확히 나누면, 테스트를 읽을 때 "아, 이 테스트가 뭘 하는 건지 한눈에 알겠네"라고 생각할 수 있다.

### 패턴 2: Exception Handling (에러 케이스도 테스트하자)

처음엔 나도 "정상 케이스만 테스트하면 되지 않나?"라고 생각했다.
하지만 사수가 "버그의 90%는 에러 처리에서 난다"고 했을 때, 깨달았다.

```typescript
it('should throw NotFoundException when user does not exist', async () => {
  // Arrange: DB에는 없는 데이터
  jest.spyOn(userRepository, 'findOne')
    .mockResolvedValue(null);

  // Act & Assert: 함수가 정확히 에러를 던지는가?
  await expect(service.getUserById(999))
    .rejects
    .toThrow(NotFoundException);
});
```

이렇게 하면:
- 없는 사용자를 조회할 때, 앱이 제대로 에러를 던지는가? ✅
- 앱이 갑자기 종료되거나 이상한 결과를 반환하지 않는가? ✅

### 패턴 3: Controller 테스트 (API 엔드포인트가 제대로 작동하나?)

Service만 테스트하면 안 되나? 처음엔 그렇게 생각했는데, 사수가 "Controller도 테스트해야 해"라고 했다.
API 요청이 들어왔을 때, 제대로 Service를 호출하고 응답을 반환하는지 확인해야 한다는 뜻이었다.

```typescript
// user.controller.spec.ts

describe('UserController', () => {
  let controller: UserController;
  let service: UserService;

  beforeEach(async () => {
    // Service는 Mock으로 대체
    const mockUserService = {
      findAll: jest.fn().mockResolvedValue([
        { id: 1, email: 'user1@example.com' },
        { id: 2, email: 'user2@example.com' },
      ]),
    };

    const module: TestingModule = await Test.createTestingModule({
      controllers: [UserController],
      providers: [
        {
          provide: UserService,
          useValue: mockUserService,  // Service는 Mock으로 대체
        },
      ],
    }).compile();

    controller = module.get<UserController>(UserController);
    service = module.get<UserService>(UserService);
  });

  it('should return array of users when GET /users called', async () => {
    // Controller의 메서드 호출
    const result = await controller.findAll();

    // 결과가 정말 2개의 사용자를 반환했는가?
    expect(result).toHaveLength(2);
    // Service의 findAll이 정말 호출되었는가?
    expect(service.findAll).toHaveBeenCalled();
  });
});
```

이렇게 하면 API 엔드포인트 자체가 제대로 작동하는지 확인할 수 있다.

---

## 5부: 실전 — 복잡한 의존성이 있을 때

실제 프로젝트에서는 한 개의 Service가 여러 다른 Service에 의존하는 일이 흔하다.
주문 생성할 때 DB 저장, 결제 처리, 알림 발송이 모두 필요하다면?

```typescript
// order.service.spec.ts

describe('OrderService', () => {
  let service: OrderService;
  let paymentService: PaymentService;
  let notificationService: NotificationService;
  let orderRepository: Repository<Order>;

  beforeEach(async () => {
    // 여러 의존성을 모두 Mock으로 대체
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderService,
        {
          provide: PaymentService,
          useValue: {
            processPayment: jest.fn().mockResolvedValue({ transactionId: 'tx123' }),
            refund: jest.fn().mockResolvedValue(true),
          },
        },
        {
          provide: NotificationService,
          useValue: {
            sendOrderConfirmation: jest.fn().mockResolvedValue(undefined),
          },
        },
        {
          provide: 'ORDER_REPOSITORY',
          useValue: {
            save: jest.fn(),
            findOne: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<OrderService>(OrderService);
    paymentService = module.get<PaymentService>(PaymentService);
    notificationService = module.get<NotificationService>(NotificationService);
    orderRepository = module.get<Repository<Order>>('ORDER_REPOSITORY');
  });

  it('should create order, process payment, and send notification in correct order', async () => {
    // Arrange: 테스트 데이터 준비
    const createOrderDto = { items: [{ sku: 'ABC123', qty: 2 }], total: 100 };
    const savedOrder = { id: 1, ...createOrderDto, status: 'PENDING' };

    orderRepository.save.mockResolvedValue(savedOrder);

    // Act: 주문 생성
    const result = await service.createOrder(createOrderDto);

    // Assert: 세 가지를 모두 확인
    // 1. DB에 저장되었나?
    expect(orderRepository.save).toHaveBeenCalledWith(expect.objectContaining({
      items: createOrderDto.items,
    }));
    // 2. 결제 처리되었나?
    expect(paymentService.processPayment).toHaveBeenCalledWith(savedOrder.id, 100);
    // 3. 알림이 발송되었나?
    expect(notificationService.sendOrderConfirmation).toHaveBeenCalledWith(savedOrder.id);
    // 4. 최종 상태가 CONFIRMED인가?
    expect(result.status).toBe('CONFIRMED');
  });

  it('should refund order if payment fails', async () => {
    // Arrange: 결제가 실패하는 상황
    paymentService.processPayment.mockRejectedValue(new Error('Card declined'));

    // Act & Assert: 에러가 제대로 발생하는가?
    await expect(service.createOrder({ items: [], total: 100 }))
      .rejects
      .toThrow('Card declined');

    // 결제가 실패했으므로, 환불 함수는 호출되지 않아야 함
    expect(paymentService.refund).not.toHaveBeenCalled();
  });
});
```

---

## 6부: Jest Matcher — 자주 쓰는 것들

처음엔 matcher가 이렇게 많은 줄 몰랐다. 고민하다가 사수에게 "이거 뭐 고르지?"라고 물어봤더니 웃으면서
"상황에 맞는 걸 쓰면 돼. 코드가 더 읽기 쉬워질 거야"라고 했다.

| Matcher | 설명 | 언제 쓸까? |
| --- | --- | --- |
| `toBe()` | 정확히 같음 (===) | 숫자나 불린 비교: `expect(count).toBe(5)` |
| `toEqual()` | 깊은 비교 (객체 비교) | 객체 비교: `expect(user).toEqual({ id: 1 })` |
| `toBeTruthy()` / `toBeFalsy()` | 참/거짓 확인 | 뭔가 참인가만 확인: `expect(isActive).toBeTruthy()` |
| `toHaveLength()` | 배열/문자열 길이 | 배열 크기 확인: `expect([1,2,3]).toHaveLength(3)` |
| `toContain()` | 배열에 요소 포함 | 특정 요소가 있나: `expect(users).toContain(newUser)` |
| `toThrow()` | 예외 발생 확인 | 에러가 던져지는가: `expect(() => fn()).toThrow()` |
| `toHaveBeenCalled()` | Mock 함수 호출 확인 | 함수가 호출됐나: `expect(fn).toHaveBeenCalled()` |
| `toHaveBeenCalledWith()` | 특정 인자로 호출 | 특정 값으로 호출됐나: `expect(fn).toHaveBeenCalledWith(5)` |
| `toResolve()` | Promise 성공 | Promise가 성공했나: `await expect(promise).resolves.toBe(true)` |
| `toReject()` | Promise 실패 | Promise가 실패했나: `await expect(promise).rejects.toThrow()` |

---

## 7부: 테스트 실행과 커버리지 — 배포 전 필수 확인

### 테스트 실행하는 방법

처음 내가 한 실수: "그냥 한 두 번 돌려봤으니 괜찮겠지"라고 생각했다.
하지만 사수의 피드백: "모든 케이스를 다 돌려봐야지. 그래야 놓친 게 없어"

```bash
# 모든 테스트 한 번에 실행
npm run test

# 특정 파일만 테스트 (빠르게 피드백 받고 싶을 때)
npm run test -- user.service.spec.ts

# watch 모드 (코드 수정할 때마다 자동으로 테스트 실행)
# 개발 중에 이걸 켜두면 실시간으로 피드백을 받을 수 있음
npm run test:watch

# 커버리지 리포트 생성 (어느 부분이 테스트 안 됐는지 보기)
npm run test:cov
```

### 커버리지 목표 — 현실적으로 생각하기

처음엔 "100% 커버리지를 목표로 해야 하지 않나?"라고 생각했다.

```
Statement   : 함수/명령문이 실행되는가?
Branch      : if/else 모든 경로가 테스트되는가?
Function    : 모든 함수가 호출되는가?
Line        : 코드의 모든 줄이 실행되는가?
```

우리 팀의 목표: **statement 70~80%, branch 60~70%**

왜 100%이 아닌가?
- 에러 처리 부분 중 일부는 테스트하기 너무 어려움 (외부 서버 장애 등)
- 매번 릴리스할 때마다 테스트를 업데이트하는 비용이 높음
- 균형이 중요함 (테스트 시간 vs 커버리지)

따라서 70~80% 정도면 충분히 안정적인 코드라고 팀에서 판단하고 있다.
(실제와 약간 다른 수치를 작성하였습니다.)

---

## 8부: 실무 팁 — 동료들과의 코드 리뷰로 배운 것

### Tip 1: describe를 중첩해서 구조화하면 가독성이 올라간다

처음엔 모든 it()을 한 describe 아래에 썼다. 그런데 코드 리뷰에서 "테스트가 너무 길어 읽기 힘들어"라고 받았다.

```typescript
describe('UserService', () => {
  describe('createUser', () => {
    it('should create user with valid email', () => { /* ... */ });
    it('should throw error with invalid email', () => { /* ... */ });
  });

  describe('deleteUser', () => {
    it('should delete user by ID', () => { /* ... */ });
    it('should throw NotFoundException if user not found', () => { /* ... */ });
  });
});
```

이렇게 하면 "어떤 메서드의 어떤 케이스를 테스트하는 건지" 한눈에 보인다.

### Tip 2: beforeAll과 beforeEach의 차이를 몰라서 헷갈렸다

```typescript
// beforeAll: 전체 테스트 시작 전에 딱 1번만 실행
// 예: DB 연결, 서버 시작 등 무거운 작업
beforeAll(async () => {
  await initializeDatabase();
});

// beforeEach: 각 테스트 전에 매번 실행
// 예: 테스트 데이터 리셋, Mock 초기화
beforeEach(() => {
  jest.clearAllMocks(); // 이전 테스트의 호출 기록을 지워줌
});
```

beforeEach에서 jest.clearAllMocks()를 꼭 해줘야 한다.
한 테스트의 Mock 호출 기록이 다음 테스트에 영향을 주면 안 되거든.

### Tip 3: Mock 함수 초기화를 안 하면 나중에 디버깅이 정말 힘들다

내가 한 실수: beforeEach를 안 썼다. 그러니까 테스트 순서에 따라 결과가 바뀌었다.
"분명 아까는 통과했는데 지금은 왜 실패하지?" 하다가 2시간을 낭비했다.

```typescript
beforeEach(() => {
  jest.clearAllMocks(); // 모든 Mock의 호출 기록과 상태를 초기화
});
```

### Tip 4: 환경 변수를 테스트할 때도 Mock을 써야 한다

```typescript
it('should use correct API key from env', () => {
  // 임시로 환경 변수 설정
  process.env.API_KEY = 'test-key-123';

  const service = new ConfigService();
  expect(service.getApiKey()).toBe('test-key-123');
  
  // 테스트 후에는 원래대로 복구하는 것이 좋음
  // (다른 테스트에 영향을 주지 않도록)
  delete process.env.API_KEY;
});
```

---

## 마치며: 3개월 뒤 나의 변화

처음엔 "왜 이런 걸 다 써야 해?"라고 투덜거렸다.
하지만 3개월이 지난 지금, 나는 **테스트 없이는 코드를 건드릴 수 없는 사람**이 되었다.

왜?

**리팩토링이 무섭지 않다.**

예를 들어, UserService의 복잡한 로직을 깔끔하게 정리하고 싶었다.
예전 나라면, "혹시 모르니까 일단 건드리지 말아야지" 했을 거다.

```bash
npm run test
```

모든 테스트가 통과한다. 그럼 내가 뭘 바꿨든 기존 기능은 안 깨진 거다.
그 두 줄의 명령어가 나에게 확신을 준다.

그리고 **버그를 빨리 찾는다.**

지금은 `npm run test`를 치는 순간, 빨간 글씨로 내 실수를 알려준다.

**결론**: 처음엔 번거롭지만, **신뢰할 수 있는 리팩토링**과 **빠른 버그 발견** 때문에 필수다.

실제 업무에서 테스트를 충분히 작성하면, 배포할 때 "혹시 뭔가 깨지면 어쩌지?"라는 불안감 대신
"완벽하게 테스트됐으니 괜찮아"라는 확신을 가질 수 있다.

그게 바로 테스트의 진정한 가치인것 같다.