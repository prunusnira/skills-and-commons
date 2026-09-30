## 목적

이 문서는 AI 에이전트가 이 저장소에서 Jest 테스트를 작성, 수정, 리뷰할 때 따라야 하는 기준을 정의한다.

대상 프로젝트는 기본적으로 다음 환경을 사용한다.

* Vite
* React
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


## Vite + React 테스트 규칙

### Jest 설정 확인

Vite 설정만으로 Jest가 동작한다고 가정하지 않는다. 테스트를 작성하기 전에 프로젝트의 Jest 설정에서 다음 항목을 확인한다.

* DOM 환경이 `jsdom`으로 설정되어 있는지
* TypeScript와 JSX를 어떤 transformer(`babel-jest`, `ts-jest`, SWC 등)로 처리하는지
* `setupFilesAfterEnv`에서 `@testing-library/jest-dom`을 불러오는지
* Vite alias와 Jest의 `moduleNameMapper`가 일치하는지
* CSS, CSS Module, 이미지, SVG 같은 정적 자산을 어떻게 처리하는지

프로젝트에 이미 설정과 test utility가 있다면 그것을 기준으로 삼고, 테스트 하나를 위해 새 설정 체계를 임의로 추가하지 않는다.

### React Router

React Router를 사용하는 컴포넌트는 라우팅 동작의 범위에 따라 실제 provider 또는 mock을 선택한다.

링크, 현재 경로, route parameter 등 라우팅 통합 동작이 중요하면 `MemoryRouter` 또는 프로젝트의 custom render를 우선 사용한다.

```tsx
render(
  <MemoryRouter initialEntries={['/products/1']}>
    <ProductPage />
  </MemoryRouter>,
);
```

이동 함수 호출만 외부 경계로 확인하면 충분한 경우에는 필요한 hook만 mock한다.

```ts
const mockNavigate = jest.fn();

jest.mock('react-router-dom', () => ({
  ...jest.requireActual<typeof import('react-router-dom')>('react-router-dom'),
  useNavigate: () => mockNavigate,
}));
```

router 전체를 빈 mock으로 대체하지 않는다. 테스트가 끝날 때 `mockNavigate`의 호출 상태가 다른 테스트로 새지 않도록 정리한다.

### Vite 환경 변수

`import.meta.env` 값에 암묵적으로 의존하지 않는다. 먼저 프로젝트의 Jest transformer가 `import.meta`를 어떻게 처리하는지 확인한다.

환경 변수를 별도 config 모듈에서 노출하는 구조라면 그 모듈을 외부 경계로 mock한다.

```ts
jest.mock('@/config/env', () => ({
  apiBaseUrl: 'https://example.test',
}));
```

transformer가 `import.meta.env`를 지원하지 않는다면 테스트 코드에서 임의의 전역 객체로 흉내 내지 않는다. 기존 Jest 설정에 맞는 env adapter나 transformer 구성이 필요한지 먼저 설명한다. production code 수정은 사용자가 요청한 경우에만 수행한다.

### 정적 자산과 CSS

Vite에서는 동작하지만 Jest가 직접 해석하지 못하는 CSS, 이미지, SVG import가 있을 수 있다. 이 경우 프로젝트의 기존 `moduleNameMapper`와 file mock을 재사용한다.

정적 자산의 실제 바이트나 CSS class name을 검증하지 않는다. 사용자에게 보이는 대체 텍스트, role, 상태 변화를 검증한다. SVG를 React 컴포넌트로 불러오는 방식은 플러그인마다 다르므로 실제 Vite 설정과 타입 선언을 확인한 뒤 필요한 경계만 mock한다.

### 브라우저 API

`matchMedia`, `ResizeObserver`, `IntersectionObserver`처럼 jsdom이 제공하지 않는 API는 실제 사용 범위만 최소한으로 mock한다.

```ts
const observe = jest.fn();
const disconnect = jest.fn();

beforeEach(() => {
  global.ResizeObserver = jest.fn(() => ({
    observe,
    unobserve: jest.fn(),
    disconnect,
  })) as unknown as typeof ResizeObserver;
});
```

공통 setup에 이미 mock이 있다면 테스트 파일에서 중복 정의하지 않는다.

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
