## 목적

이 문서는 AI 에이전트가 이 저장소에서 Vitest 기반 Next.js 테스트를 작성, 수정, 리뷰할 때 따라야 하는 기준을 정의한다.

이 프로젝트는 기본적으로 다음 환경을 사용한다.

* Next.js
* React
* TypeScript
* Vitest
* React Testing Library
* `@testing-library/user-event`

프로젝트에서 App Router 또는 Pages Router 중 무엇을 사용하는지 먼저 확인한다.

목표는 테스트 개수를 늘리는 것이 아니다.
목표는 **읽기 쉽고, 안정적이며, 유지보수 가능한 테스트**를 작성하는 것이다.

---


## Next.js 테스트 범위

Next.js 컴포넌트를 테스트하기 전에 대상이 다음 중 어디에 해당하는지 확인한다.

* Client Component
* 동기 Server Component
* 비동기 Server Component
* Server Action
* Route Handler
* Middleware 또는 Proxy
* 일반 TypeScript 함수

### Client Component

`'use client'`가 선언된 컴포넌트는 React Testing Library를 사용해 일반 React 컴포넌트처럼 테스트한다.

우선적으로 검증한다.

* 사용자 입력
* 클릭과 키보드 동작
* 로딩 및 오류 상태
* navigation 요청
* Server Action 호출 의도
* 사용자에게 보이는 상태 변화

### 동기 Server Component

비동기 처리가 없는 Server Component는 Vitest와 React Testing Library로 렌더링할 수 있다.

```tsx
export default function Page() {
  return <h1>상품 목록</h1>;
}
```

```tsx
it('상품 목록 제목을 보여준다', () => {
  // Given
  render(<Page />);

  // When
  const heading = screen.getByRole('heading', {
    name: '상품 목록',
  });

  // Then
  expect(heading).toBeInTheDocument();
});
```

### 비동기 Server Component

`async` Server Component를 Vitest와 React Testing Library로 직접 렌더링하지 않는다.

```tsx
export default async function Page() {
  const products = await getProducts();

  return <ProductList products={products} />;
}
```

현재 Vitest는 Next.js의 비동기 Server Component 테스트를 완전히 지원하지 않는다.

다음 방식 중 하나를 선택한다.

1. 데이터 가공 로직을 순수 함수로 분리해 단위 테스트한다.
2. 하위 Client Component를 props 기반으로 테스트한다.
3. 데이터 접근 함수를 별도로 테스트한다.
4. 페이지 전체 동작은 Playwright 또는 Cypress 같은 E2E 테스트로 검증한다.

비동기 Server Component를 테스트하기 위해 비공식적인 렌더링 우회 코드를 추가하지 않는다.

---


## Server Component와 Client Component 경계

App Router에서는 page와 layout이 기본적으로 Server Component다.

컴포넌트 테스트 전에 다음을 확인한다.

* `'use client'` 선언 여부
* 브라우저 API 사용 여부
* React state 또는 Effect 사용 여부
* 서버 전용 모듈 사용 여부
* 비동기 데이터 조회 여부

Server Component 테스트에서 다음 서버 전용 로직을 무리하게 jsdom 환경에 재현하지 않는다.

* 데이터베이스 접근
* 인증 세션 조회
* 서버 환경변수 접근
* `cookies()`
* `headers()`
* 캐시 및 재검증
* React Server Component 스트리밍

이러한 기능은 작은 서버 함수로 분리하거나 통합 테스트 또는 E2E 테스트에서 검증한다.

---


## Next.js Navigation

### App Router

App Router에서는 `next/navigation`을 사용한다.

필요한 API만 mock한다.

주요 대상:

* `useRouter`
* `usePathname`
* `useSearchParams`
* `redirect`
* `notFound`

```tsx
const { mockPush, mockReplace, mockRefresh } = vi.hoisted(() => ({
  mockPush: vi.fn(),
  mockReplace: vi.fn(),
  mockRefresh: vi.fn(),
}));

vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: mockPush,
    replace: mockReplace,
    refresh: mockRefresh,
    back: vi.fn(),
    forward: vi.fn(),
    prefetch: vi.fn(),
  }),
  usePathname: () => '/products',
  useSearchParams: () => new URLSearchParams(),
}));
```

```tsx
it('상품을 선택하면 상세 페이지로 이동한다', async () => {
  // Given
  const user = userEvent.setup();

  render(<ProductCard product={product} />);

  // When
  await user.click(
    screen.getByRole('link', {
      name: '테스트 상품',
    }),
  );

  // Then
  expect(mockPush).toHaveBeenCalledWith('/products/product-1');
});
```

실제 `<Link>`를 클릭하는 컴포넌트라면 `useRouter` 호출 여부보다 최종 `href`를 검증하는 것을 우선한다.

```tsx
expect(
  screen.getByRole('link', {
    name: '상품 보기',
  }),
).toHaveAttribute('href', '/products/product-1');
```

`router.push` 호출은 명령형 navigation이 중요한 경우에만 검증한다.

### Pages Router

Pages Router를 사용하는 프로젝트에서는 `next/router`를 사용한다.

```tsx
const { mockPush } = vi.hoisted(() => ({
  mockPush: vi.fn(),
}));

vi.mock('next/router', () => ({
  useRouter: () => ({
    push: mockPush,
    pathname: '/products',
    query: {},
    asPath: '/products',
    isReady: true,
  }),
}));
```

App Router와 Pages Router mock을 혼용하지 않는다.

---


## Search Params

`useSearchParams()`를 사용하는 컴포넌트는 실제 `URLSearchParams`와 유사한 객체를 제공한다.

```tsx
vi.mock('next/navigation', () => ({
  usePathname: () => '/products',
  useSearchParams: () =>
    new URLSearchParams({
      keyword: '키보드',
      page: '2',
    }),
}));
```

테스트별 query parameter가 다르다면 mock 값을 전역 상수로 고정하지 않는다.

```tsx
const { mockSearchParams } = vi.hoisted(() => ({
  mockSearchParams: vi.fn(),
}));

vi.mock('next/navigation', () => ({
  useSearchParams: () => mockSearchParams(),
}));
```

```tsx
beforeEach(() => {
  mockSearchParams.mockReturnValue(new URLSearchParams());
});
```

---


## redirect와 notFound

`redirect()`와 `notFound()`는 정상적으로 반환하는 함수처럼 다루지 않는다.

해당 호출 의도가 중요한 서버 로직이라면 모듈을 mock하고 호출 여부를 검증한다.

```tsx
const { mockRedirect } = vi.hoisted(() => ({
  mockRedirect: vi.fn(() => {
    throw new Error('NEXT_REDIRECT');
  }),
}));

vi.mock('next/navigation', () => ({
  redirect: mockRedirect,
}));
```

```ts
it('로그인하지 않은 사용자를 로그인 페이지로 이동시킨다', async () => {
  // Given
  mockGetSession.mockResolvedValue(null);

  // When
  const result = loadProtectedPage();

  // Then
  await expect(result).rejects.toThrow('NEXT_REDIRECT');
  expect(mockRedirect).toHaveBeenCalledWith('/login');
});
```

Next.js 내부 오류 문자열 자체를 비즈니스 요구사항처럼 과도하게 검증하지 않는다.

가능하면 redirect 여부와 대상 URL을 검증한다.

---


## next/link

`next/link`는 기본적으로 실제 링크 동작과 `href`를 검증한다.

단순히 테스트 편의를 위해 항상 mock하지 않는다.

```tsx
render(
  <Link href="/products/product-1">
    상품 보기
  </Link>,
);

expect(
  screen.getByRole('link', {
    name: '상품 보기',
  }),
).toHaveAttribute('href', '/products/product-1');
```

특정 테스트 환경에서 `next/link`가 문제가 되는 경우에만 최소한으로 mock한다.

```tsx
vi.mock('next/link', () => ({
  default: ({
    href,
    children,
  }: {
    href: string;
    children: React.ReactNode;
  }) => <a href={href}>{children}</a>,
}));
```

---


## next/image

`next/image`는 이미지 최적화 자체보다 사용자에게 전달되는 의미를 검증한다.

우선 검증한다.

* 접근 가능한 `alt`
* 조건에 따른 이미지 표시 여부
* 이미지 source를 결정하는 비즈니스 로직
* fallback 이미지

Next.js의 최종 최적화 URL 구조나 내부 속성은 검증하지 않는다.

테스트 환경에서 `next/image` mock이 필요한 경우 일반 `img`로 최소한 변환한다.

```tsx
vi.mock('next/image', () => ({
  default: ({
    src,
    alt,
    ...props
  }: React.ImgHTMLAttributes<HTMLImageElement>) => (
    <img src={String(src)} alt={alt ?? ''} {...props} />
  ),
}));
```

다음과 같은 assertion은 피한다.

```tsx
expect(image).toHaveAttribute(
  'src',
  '/_next/image?url=%2Fproduct.png&w=640&q=75',
);
```

Next.js 내부 이미지 최적화 구현에 테스트를 결합하지 않는다.

---


## 환경 변수

Next.js 환경변수는 `import.meta.env`가 아니라 `process.env`를 사용한다.

클라이언트에 노출되는 환경변수는 `NEXT_PUBLIC_` 접두사를 사용한다.

```ts
const previousApiUrl = process.env.NEXT_PUBLIC_API_BASE_URL;

beforeEach(() => {
  process.env.NEXT_PUBLIC_API_BASE_URL = 'https://example.test';
});

afterEach(() => {
  process.env.NEXT_PUBLIC_API_BASE_URL = previousApiUrl;
});
```

환경변수 모듈이 import 시점에 값을 읽는 경우에는 mock 설정 후 모듈을 다시 import해야 할 수 있다.

```ts
beforeEach(() => {
  vi.resetModules();
  process.env.NEXT_PUBLIC_API_BASE_URL = 'https://example.test';
});

it('환경변수의 API 주소를 사용한다', async () => {
  // Given
  const { getApiBaseUrl } = await import('./env');

  // When
  const result = getApiBaseUrl();

  // Then
  expect(result).toBe('https://example.test');
});
```

테스트에서 실제 `.env` 파일 값에 암묵적으로 의존하지 않는다.

서버 전용 환경변수는 Client Component 테스트에 노출하지 않는다.

---
