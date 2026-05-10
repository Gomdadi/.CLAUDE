# TypeScript/JavaScript Unit Test Reference (Jest)

## 목차
1. [설치 및 설정](#설치-및-설정)
2. [기본 테스트 구조](#기본-테스트-구조)
3. [Matcher 패턴](#matcher-패턴)
4. [Mock 패턴](#mock-패턴)
5. [비동기 테스트](#비동기-테스트)
6. [React 컴포넌트 테스트](#react-컴포넌트-테스트)
7. [NestJS 테스트](#nestjs-테스트)
8. [테스트 라이프사이클](#테스트-라이프사이클)

---

## 설치 및 설정

**TypeScript 프로젝트**
```bash
npm install -D jest @types/jest ts-jest
```

`jest.config.ts`:
```ts
export default {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.ts', '**/*.spec.ts', '**/*.test.ts'],
};
```

**Next.js / React**
```bash
npm install -D jest jest-environment-jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

---

## 기본 테스트 구조

```ts
// order.service.test.ts 또는 order.service.spec.ts
import { OrderService } from './order.service';

describe('OrderService', () => {
  let service: OrderService;

  beforeEach(() => {
    service = new OrderService();
  });

  describe('createOrder', () => {
    it('금액이 양수이면 주문을 생성한다', () => {
      // given
      const amount = 1000;

      // when
      const order = service.createOrder(amount);

      // then
      expect(order).toBeDefined();
      expect(order.amount).toBe(1000);
    });

    it('금액이 0이면 에러를 던진다', () => {
      expect(() => service.createOrder(0)).toThrow('금액은 양수여야 합니다');
    });
  });
});
```

**파일 명명:** 테스트 대상 파일과 같은 위치에 `.test.ts` 또는 `.spec.ts` 확장자 사용

---

## Matcher 패턴

```ts
// 동등성
expect(result).toBe(42);           // 원시값 (===)
expect(result).toEqual({ a: 1 }); // 객체 깊은 비교
expect(result).toStrictEqual(obj); // undefined 프로퍼티 포함 엄격 비교

// null / undefined
expect(result).toBeNull();
expect(result).toBeUndefined();
expect(result).toBeDefined();

// 불리언
expect(flag).toBeTruthy();
expect(flag).toBeFalsy();

// 숫자
expect(price).toBeGreaterThan(0);
expect(price).toBeLessThanOrEqual(1000);
expect(0.1 + 0.2).toBeCloseTo(0.3);

// 문자열
expect(name).toContain('길동');
expect(name).toMatch(/^홍/);
expect(name).toHaveLength(3);

// 배열 / iterable
expect(list).toHaveLength(3);
expect(list).toContain('apple');
expect(list).toContainEqual({ id: 1 });
expect(list).toEqual(expect.arrayContaining(['a', 'b']));

// 예외
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(Error);
expect(() => fn()).toThrow('에러 메시지');
expect(() => fn()).toThrowError(/패턴/);

// 비동기 예외
await expect(asyncFn()).rejects.toThrow('에러');
await expect(asyncFn()).resolves.toEqual(expected);
```

---

## Mock 패턴

### 함수 Mock

```ts
// jest.fn() — 단순 mock 함수
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockReturnValueOnce(1).mockReturnValueOnce(2);
mockFn.mockResolvedValue({ data: 'ok' }); // Promise 반환
mockFn.mockRejectedValue(new Error('실패'));
mockFn.mockImplementation((x) => x * 2);

// 호출 검증
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith(1, 'hello');
expect(mockFn).toHaveBeenLastCalledWith(expect.any(Number));
expect(mockFn).not.toHaveBeenCalled();
```

### 모듈 Mock

```ts
// 모듈 전체 mock
jest.mock('./payment-client');
jest.mock('../utils/logger', () => ({
  log: jest.fn(),
  error: jest.fn(),
}));

// 특정 함수만 mock
jest.spyOn(orderRepository, 'findById').mockResolvedValue(order);
jest.spyOn(console, 'error').mockImplementation(() => {});

// 테스트 후 원복
afterEach(() => {
  jest.restoreAllMocks();
});
```

### 의존성 주입 방식 Mock

```ts
import { OrderService } from './order.service';
import { OrderRepository } from './order.repository';

jest.mock('./order.repository');
const MockOrderRepository = OrderRepository as jest.MockedClass<typeof OrderRepository>;

describe('OrderService', () => {
  let service: OrderService;
  let mockRepo: jest.Mocked<OrderRepository>;

  beforeEach(() => {
    mockRepo = new MockOrderRepository() as jest.Mocked<OrderRepository>;
    mockRepo.findById.mockResolvedValue(null);
    service = new OrderService(mockRepo);
  });
});
```

---

## 비동기 테스트

```ts
// async/await (권장)
it('사용자를 조회한다', async () => {
  const user = await userService.findUser(1);
  expect(user.name).toBe('홍길동');
});

// Promise 반환
it('사용자를 조회한다', () => {
  return userService.findUser(1).then((user) => {
    expect(user.name).toBe('홍길동');
  });
});

// resolves / rejects matcher
it('사용자를 조회한다', async () => {
  await expect(userService.findUser(1)).resolves.toMatchObject({ name: '홍길동' });
});

it('존재하지 않는 사용자 조회 시 에러', async () => {
  await expect(userService.findUser(999)).rejects.toThrow('사용자를 찾을 수 없습니다');
});
```

---

## React 컴포넌트 테스트

```ts
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('텍스트가 렌더링된다', () => {
    render(<Button>확인</Button>);
    expect(screen.getByText('확인')).toBeInTheDocument();
  });

  it('클릭 시 핸들러가 호출된다', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>확인</Button>);

    await userEvent.click(screen.getByRole('button', { name: '확인' }));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('비활성화 상태에서 클릭되지 않는다', async () => {
    const handleClick = jest.fn();
    render(<Button disabled onClick={handleClick}>확인</Button>);

    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).not.toHaveBeenCalled();
  });
});
```

**쿼리 우선순위** (접근성 기반 권장 순서):
1. `getByRole` — 가장 권장
2. `getByLabelText` — 폼 요소
3. `getByPlaceholderText`
4. `getByText`
5. `getByTestId` — 마지막 수단

---

## NestJS 테스트

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';

describe('UserService', () => {
  let service: UserService;
  let repository: jest.Mocked<UserRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: {
            findById: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get(UserRepository);
  });

  it('사용자를 반환한다', async () => {
    const user = { id: 1, name: '홍길동' };
    repository.findById.mockResolvedValue(user);

    const result = await service.getUser(1);
    expect(result).toEqual(user);
    expect(repository.findById).toHaveBeenCalledWith(1);
  });
});
```

---

## 테스트 라이프사이클

```ts
describe('Suite', () => {
  beforeAll(() => {
    // 스위트 시작 전 1회
  });

  afterAll(() => {
    // 스위트 종료 후 1회
  });

  beforeEach(() => {
    // 각 테스트 전
    jest.clearAllMocks(); // mock 호출 기록 초기화
  });

  afterEach(() => {
    // 각 테스트 후
    jest.restoreAllMocks(); // spyOn 원복
  });
});
```

**Mock 초기화 메서드 차이:**
- `jest.clearAllMocks()` — 호출 기록, 반환값 초기화 (구현은 유지)
- `jest.resetAllMocks()` — 구현까지 초기화
- `jest.restoreAllMocks()` — `spyOn` 원복
