## 목적

이 문서는 AI 에이전트가 이 저장소에서 Vitest 기반 React 테스트를 작성, 수정, 리뷰할 때 따라야 하는 기준을 정의한다.

이 프로젝트는 기본적으로 다음 환경을 사용한다.

* React
* TypeScript
* Vitest
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
* 라우팅 또는 navigation 의도
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
* router 또는 navigation adapter
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


## Vitest 테스트 규칙

### vi 사용

Vitest에서는 Jest 전역 API 대신 `vi`를 사용한다.

```ts
import { vi } from 'vitest';

const mockSubmit = vi.fn();
```

Jest 전용 API를 그대로 사용하지 않는다.

나쁜 예:

```ts
const mockSubmit = jest.fn();
```

### Module mock

모듈 mock이 필요한 경우 테스트 파일 상단에서 `vi.mock(...)`을 사용한다.
`vi.mock(...)`은 hoisting되므로 mock factory에서 참조할 값은 `vi.hoisted(...)`로 준비한다.

```ts
const { mockNavigate } = vi.hoisted(() => ({
  mockNavigate: vi.fn(),
}));

vi.mock('@/lib/router', () => ({
  useRouter: () => ({
    navigate: mockNavigate,
  }),
}));
```

mock factory 안에서는 테스트마다 달라져야 하는 값을 과도하게 직접 만들지 않는다.
테스트별 동작 변경이 필요하면 `vi.mocked(...)`, mock 함수의 `mockResolvedValue`, `mockImplementation` 등을 사용한다.

### React Router

React Router navigation이 필요한 경우 필요한 동작만 mock하거나, 실제 router provider를 사용한다.

```ts
const { mockNavigate } = vi.hoisted(() => ({
  mockNavigate: vi.fn(),
}));

vi.mock('react-router-dom', async () => {
  const actual = await vi.importActual<typeof import('react-router-dom')>(
    'react-router-dom',
  );

  return {
    ...actual,
    useNavigate: () => mockNavigate,
  };
});
```

라우팅 통합 동작이 중요하다면 `MemoryRouter` 같은 실제 provider를 우선 고려한다.

### Vite 환경 변수

`import.meta.env`에 의존하는 코드는 실제 환경값에 암묵적으로 기대지 않는다.

필요하면 테스트 setup 또는 테스트 파일에서 명시적으로 stub한다.

```ts
vi.stubEnv('VITE_API_BASE_URL', 'https://example.test');

afterEach(() => {
  vi.unstubAllEnvs();
});
```

### jsdom / happy-dom

컴포넌트 테스트는 DOM 환경이 필요하다.

프로젝트 설정에서 `environment: 'jsdom'` 또는 `environment: 'happy-dom'` 중 무엇을 쓰는지 먼저 확인한다.

```ts
// vitest.config.ts
test: {
  environment: 'jsdom',
  setupFiles: './src/test/setup.ts',
}
```

테스트가 브라우저 layout 계산에 과도하게 의존한다면 unit test보다 E2E 테스트가 적절할 수 있다.

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
const mockSubmit = vi.fn<(values: FormValues) => Promise<void>>();
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

프로젝트가 `*.spec.tsx` 또는 `__tests__` 구조를 이미 사용한다면 기존 구조를 따른다.

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
