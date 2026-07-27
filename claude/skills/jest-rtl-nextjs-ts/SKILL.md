# Jest 테스트 작성 Skill

## 목적

이 문서는 AI 에이전트가 이 저장소에서 Jest 테스트를 작성, 수정, 리뷰할 때 따라야 하는 기준을 정의한다.

이 프로젝트는 기본적으로 다음 환경을 사용한다.

* Next.js
* TypeScript
* Jest
* React Testing Library
* `@testing-library/user-event`

목표는 테스트 개수를 늘리는 것이 아니다.
목표는 **읽기 쉽고, 안정적이며, 유지보수 가능한 테스트**를 작성하는 것이다.

---

## 핵심 원칙

### 1. Given-When-Then 구조를 반드시 유지한다

모든 테스트는 기본적으로 다음 구조를 따른다.

```ts
it('기대하는 동작을 설명한다', async () => {
  // Given
  // 초기 props, mock, fixture, 렌더링 대상 등을 준비한다

  // When
  // 사용자 행동을 수행하거나 테스트 대상 함수를 실행한다

  // Then
  // 관찰 가능한 결과를 검증한다
});
```

테스트가 매우 짧더라도 Given-When-Then 흐름이 드러나야 한다.

좋은 예:

```ts
it('필수 입력값이 비어 있으면 에러 메시지를 보여준다', async () => {});
```

나쁜 예:

```ts
it('setError를 호출한다', async () => {});
```

테스트 이름은 내부 구현이 아니라 **사용자 관점의 동작**을 설명해야 한다.

---

## 테스트 철학

테스트는 구현 세부사항이 아니라 **관찰 가능한 동작**을 검증해야 한다.

우선적으로 검증해야 하는 것:

* 화면에 보이는 텍스트
* 접근 가능한 role과 name
* form 입력값
* 페이지 이동 의도
* API 호출 결과
* 로딩, 성공, 실패 상태
* public callback의 결과
* 사용자에게 의미 있는 상태 변화
* 사용자가 인지하거나 조작할 수 있는 주요 UI의 올바른 표시

되도록 피해야 하는 것:

* 내부 state
* private 함수
* hook 내부 구현
* CSS class name
* DOM 구조의 세부 형태
* 큰 snapshot
* 구현 편의를 위한 과도한 mock

---

## React Testing Library Query 우선순위

컴포넌트 테스트에서는 다음 순서로 query를 선택한다.

1. `screen.getByRole(...)`
2. `screen.getByLabelText(...)`
3. `screen.getByPlaceholderText(...)`
4. `screen.getByText(...)`
5. `screen.getByDisplayValue(...)`
6. `screen.getByTestId(...)`

좋은 예:

```ts
screen.getByRole('button', { name: '저장' });
```

가능하면 피해야 하는 예:

```ts
screen.getByTestId('submit-button');
```

`data-testid`는 사용자 관점의 query로 찾기 어려운 경우에만 사용한다.

---

## 사용자 상호작용

사용자 행동은 기본적으로 `userEvent.setup()`을 사용한다.

```ts
const user = userEvent.setup();

await user.type(screen.getByLabelText('이름'), '홍길동');
await user.click(screen.getByRole('button', { name: '저장' }));
```

`fireEvent`는 `userEvent`로 표현하기 어려운 낮은 수준의 이벤트를 테스트할 때만 사용한다.

---

## 비동기 테스트 규칙

요소가 나타날 때까지 기다려야 한다면 `findBy*`를 사용한다.

```ts
expect(await screen.findByText('저장되었습니다')).toBeInTheDocument();
```

비동기 처리 후 특정 assertion을 기다려야 한다면 `waitFor`를 사용한다.

```ts
await waitFor(() => {
  expect(mockSubmit).toHaveBeenCalledTimes(1);
});
```

임의의 delay는 사용하지 않는다.

나쁜 예:

```ts
await new Promise((resolve) => setTimeout(resolve, 1000));
```

---

## Mock 규칙

Mock은 외부 경계에만 사용한다.

Mock해도 되는 대상:

* 네트워크 요청
* 브라우저 API
* Next.js router
* analytics
* 날짜와 시간
* localStorage / sessionStorage
* feature flag
* 외부 SDK

되도록 mock하지 말아야 하는 대상:

* 테스트 대상 컴포넌트 자체
* 단순한 child component
* 내부 유틸 함수
* assertion을 쉽게 만들기 위한 과도한 의존성

mock이 너무 많다면 실제 동작이 아니라 mock 설정을 테스트하고 있을 가능성이 높다.

---

## Next.js 테스트 규칙

### Router

Next.js navigation이 필요한 경우 필요한 동작만 mock한다.

```ts
const mockPush = jest.fn();

jest.mock('next/navigation', () => ({
  useRouter: () => ({
    push: mockPush,
    replace: jest.fn(),
    refresh: jest.fn(),
  }),
  usePathname: () => '/current-path',
  useSearchParams: () => new URLSearchParams(),
}));
```

### Server Component

async Server Component를 억지로 Jest unit test로 검증하지 않는다.

이 경우 다음 방향을 우선한다.

* 순수 로직을 별도 함수로 분리해서 테스트한다
* 하위 Client Component를 별도로 테스트한다
* 전체 동작은 E2E 테스트에서 검증한다

### Next Image

`next/image` 때문에 테스트가 불필요하게 복잡해지는 경우 Jest setup에서 일반 `img`처럼 mock할 수 있다.

```tsx
jest.mock('next/image', () => ({
  __esModule: true,
  default: (props: React.ImgHTMLAttributes<HTMLImageElement>) => {
    // eslint-disable-next-line @next/next/no-img-element
    return <img {...props} />;
  },
}));
```

---

## TypeScript 규칙

테스트 코드에서도 `any`는 되도록 사용하지 않는다.

좋은 예:

```ts
const mockUser: User = {
  id: 'user-1',
  name: '홍길동',
};
```

나쁜 예:

```ts
const mockUser: any = {};
```

mock 함수도 가능하면 타입을 지정한다.

```ts
const mockSubmit = jest.fn<Promise<void>, [FormValues]>();
```

타입이 너무 복잡하다면 `any`로 약화하기보다 fixture builder를 만든다.

```ts
const createMockUser = (override: Partial<User> = {}): User => ({
  id: 'user-1',
  name: '홍길동',
  email: 'test@example.com',
  ...override,
});
```

---

## Fixture 규칙

fixture는 작고 명확하게 작성한다.

좋은 예:

```ts
const createMockProduct = (
  override: Partial<Product> = {},
): Product => ({
  id: 'product-1',
  name: '테스트 상품',
  price: 10000,
  isSoldOut: false,
  ...override,
});
```

피해야 하는 예:

```ts
import { hugeMockProduct } from '@/test/fixtures/product';
```

테스트를 읽는 사람이 어떤 데이터가 중요한지 바로 이해할 수 있어야 한다.

---

## 테스트 파일 구조

가능하면 테스트 대상 파일 가까이에 테스트 파일을 둔다.

```txt
components/
  ProductCard.tsx
  ProductCard.test.tsx
```

도메인 유틸 함수는 다음처럼 둔다.

```txt
features/
  product/
    utils/
      calculateDiscount.ts
      calculateDiscount.test.ts
```

파일 확장자는 다음 기준을 따른다.

* React 컴포넌트 테스트: `.test.tsx`
* 일반 TypeScript 함수 테스트: `.test.ts`

---

## 테스트 이름 규칙

테스트 이름은 동작을 설명해야 한다.

좋은 예:

```ts
describe('LoginForm', () => {
  it('이메일이 비어 있으면 validation 메시지를 보여준다', async () => {});
  it('form이 유효하면 이메일과 비밀번호를 제출한다', async () => {});
});
```

나쁜 예:

```ts
describe('handleSubmit', () => {
  it('works', async () => {});
});
```

---

## 컴포넌트 테스트 기본 형태

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('form이 유효하면 이메일과 비밀번호를 제출한다', async () => {
    // Given
    const user = userEvent.setup();
    const onSubmit = jest.fn();

    render(<LoginForm onSubmit={onSubmit} />);

    // When
    await user.type(screen.getByLabelText('이메일'), 'test@example.com');
    await user.type(screen.getByLabelText('비밀번호'), 'password123');
    await user.click(screen.getByRole('button', { name: '로그인' }));

    // Then
    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123',
      });
    });
  });
});
```

---

## 순수 함수 테스트 기본 형태

```ts
import { calculateDiscountRate } from './calculateDiscountRate';

describe('calculateDiscountRate', () => {
  it('정가가 판매가보다 크면 할인율을 반환한다', () => {
    // Given
    const tagPrice = 10000;
    const realPrice = 8000;

    // When
    const result = calculateDiscountRate(tagPrice, realPrice);

    // Then
    expect(result).toBe(20);
  });
});
```

---

## 에러 케이스 규칙

중요한 실패 케이스를 반드시 고려한다.

예시:

* 빈 입력값
* 잘못된 입력값
* API 실패
* 로딩 상태
* 권한 없음
* optional 데이터 누락
* 경계값
* 품절 또는 비활성화 상태
* 로그인하지 않은 상태

happy path만 테스트하지 않는다.

---

## Timer와 Date 규칙

날짜나 timer를 테스트할 때는 실제 현재 시간에 의존하지 않는다.

필요하면 fake timer를 사용하고, 테스트 이후 반드시 원복한다.

```ts
beforeEach(() => {
  jest.useFakeTimers();
});

afterEach(() => {
  jest.useRealTimers();
});
```

---

## 테스트 격리 규칙

각 테스트는 서로 독립적이어야 한다.

다음 상태가 테스트 사이에 새면 안 된다.

* mock call history
* mock implementation
* fake timer
* localStorage
* sessionStorage
* 공유 mutable state

필요하면 다음처럼 정리한다.

```ts
afterEach(() => {
  jest.clearAllMocks();
  localStorage.clear();
  sessionStorage.clear();
});
```

mock 구현까지 초기화해야 하는 경우에만 `jest.resetAllMocks()`를 사용한다.

---

## Snapshot 규칙

큰 snapshot은 만들지 않는다.

snapshot은 작고 안정적인 결과물을 의도적으로 검증할 때만 사용한다.

나쁜 예:

```ts
expect(container).toMatchSnapshot();
```

좋은 예:

```ts
expect(screen.getByRole('heading', { name: '주문 완료' })).toBeInTheDocument();
expect(screen.getByText('결제가 정상적으로 완료되었습니다.')).toBeInTheDocument();
```

---

## AI 에이전트가 테스트 작성 전에 확인해야 할 것

테스트를 작성하거나 수정하기 전에 반드시 다음을 확인한다.

1. 테스트 대상 컴포넌트 또는 함수
2. public input과 output
3. 사용자에게 보이는 동작
4. 외부 의존성
5. 주변 테스트 파일의 스타일
6. 프로젝트의 Jest 설정
7. custom render utility 존재 여부
8. 이미 정의된 mock 존재 여부

기존 테스트 유틸이 있다면 새로 만들지 말고 재사용한다.

---

## AI 에이전트가 하면 안 되는 것

다음 행동은 하지 않는다.

* 테스트를 쉽게 만들기 위해 production code를 임의로 크게 수정하기
* 단순히 render만 확인하는 의미 없는 테스트 추가하기
* 과도하게 mock하기
* private 구현 세부사항 테스트하기
* `any`를 습관적으로 사용하기
* class name이나 DOM 구조에 강하게 의존하는 assertion 작성하기
* 임의의 `setTimeout`으로 기다리기
* 큰 snapshot 생성하기
* 테스트 파일의 TypeScript 오류 무시하기
* `it.skip`, `describe.skip` 남기기
* `.only` 남기기

---

## 테스트 작성이 어려운 경우

테스트 작성이 어렵다면 무리하게 mock을 늘리지 않는다.

먼저 다음을 설명한다.

1. 테스트가 어려운 이유
2. 어떤 의존성 때문에 문제가 생기는지
3. Jest unit test로 검증 가능한 범위
4. E2E 테스트가 더 적절한 범위
5. production code를 개선한다면 어떤 구조가 좋은지

단, production code 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

---

## 완료 전 체크리스트

작업을 마치기 전에 다음을 확인한다.

* [ ] Given-When-Then 구조를 따른다
* [ ] 테스트 이름이 동작을 설명한다
* [ ] 사용자 행동은 `userEvent`를 사용한다
* [ ] query는 접근 가능한 selector를 우선한다
* [ ] 비동기 처리는 `findBy*` 또는 `waitFor`를 사용한다
* [ ] mock은 외부 경계에만 제한적으로 사용한다
* [ ] 중요한 에러 케이스를 포함한다
* [ ] 테스트 간 상태가 격리되어 있다
* [ ] `.only`, `.skip`, 임의 delay가 없다
* [ ] TypeScript 타입을 보존한다
* [ ] 테스트를 읽는 사람이 과도한 mock setup 없이 이해할 수 있다
* [ ] 실제 동작이 깨지면 테스트도 실패한다

---

## AI 에이전트의 최종 응답 형식

AI 에이전트가 테스트를 작성하거나 수정한 뒤에는 다음 내용을 요약한다.

1. 어떤 동작을 테스트했는지
2. 어떤 mock을 추가했고 왜 필요한지
3. 어떤 edge case를 포함했는지
4. Jest로 검증하기 어려워 E2E가 필요한 영역이 있는지