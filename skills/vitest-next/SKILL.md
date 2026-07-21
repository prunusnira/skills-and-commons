# Vitest Next.js 테스트 작성 Skill

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

## Server Action

Server Action을 테스트할 때는 UI 테스트와 서버 함수 테스트를 분리한다.

### Client Component 테스트

Client Component에서는 다음을 검증한다.

* form 입력
* 제출 동작
* pending 상태
* validation 메시지
* Server Action 호출에 전달되는 값
* 성공 또는 실패 결과에 따른 UI 변화

Server Action 내부의 데이터베이스 동작까지 Client Component 테스트에서 검증하지 않는다.

### Server Action 단위 테스트

Server Action은 일반 비동기 서버 함수처럼 테스트할 수 있지만 다음 외부 경계를 mock한다.

* 인증
* 데이터베이스
* 외부 API
* `revalidatePath`
* `revalidateTag`
* `redirect`

```ts
const { mockRevalidatePath } = vi.hoisted(() => ({
  mockRevalidatePath: vi.fn(),
}));

vi.mock('next/cache', () => ({
  revalidatePath: mockRevalidatePath,
}));
```

```ts
it('상품 수정 후 상품 상세 캐시를 무효화한다', async () => {
  // Given
  mockUpdateProduct.mockResolvedValue({
    id: 'product-1',
  });

  const formData = new FormData();
  formData.set('name', '수정 상품');

  // When
  await updateProduct('product-1', formData);

  // Then
  expect(mockUpdateProduct).toHaveBeenCalledWith('product-1', {
    name: '수정 상품',
  });
  expect(mockRevalidatePath).toHaveBeenCalledWith(
    '/products/product-1',
  );
});
```

Server Action은 Client Component에서 호출되더라도 서버에서 실행된다.

각 Server Action의 인증과 권한 검사를 테스트 대상에 포함한다.

---

## Route Handler

Route Handler는 `Request` 또는 `NextRequest`를 생성해 직접 호출할 수 있다.

```ts
import { GET } from './route';

describe('GET /api/products', () => {
  it('상품 목록을 반환한다', async () => {
    // Given
    mockGetProducts.mockResolvedValue([
      {
        id: 'product-1',
        name: '테스트 상품',
      },
    ]);

    const request = new Request(
      'https://example.test/api/products',
    );

    // When
    const response = await GET(request);
    const body = await response.json();

    // Then
    expect(response.status).toBe(200);
    expect(body).toEqual({
      products: [
        {
          id: 'product-1',
          name: '테스트 상품',
        },
      ],
    });
  });
});
```

Route Handler 테스트에서 우선 검증한다.

* HTTP status
* 응답 body
* validation
* 인증과 권한
* query parameter
* request body
* header
* cookie
* 외부 서비스 실패 처리

Next.js 서버 런타임 전체의 동작이 중요한 경우에는 E2E 또는 integration test를 사용한다.

---

## next/headers

`cookies()`, `headers()` 같은 `next/headers` API를 사용하는 로직은 직접 모듈 mock을 사용할 수 있다.

Next.js 버전에 따라 해당 API가 비동기일 수 있으므로 현재 프로젝트의 Next.js 버전과 실제 사용 형태를 먼저 확인한다.

```ts
const { mockCookies } = vi.hoisted(() => ({
  mockCookies: vi.fn(),
}));

vi.mock('next/headers', () => ({
  cookies: mockCookies,
}));
```

```ts
beforeEach(() => {
  mockCookies.mockResolvedValue({
    get: vi.fn((name: string) => {
      if (name === 'access-token') {
        return {
          name,
          value: 'test-token',
        };
      }

      return undefined;
    }),
  });
});
```

Next.js API가 동기인지 비동기인지 추측해서 mock하지 않는다.

---

## Metadata

`generateMetadata`가 순수한 데이터 변환에 가깝다면 함수 결과를 직접 검증할 수 있다.

다만 `generateMetadata` 내부에서 비동기 데이터 조회, `cookies()`, `headers()` 같은 서버 API를 사용한다면 단위 테스트 범위를 작게 유지한다.

```ts
it('상품명으로 metadata title을 생성한다', async () => {
  // Given
  mockGetProduct.mockResolvedValue({
    id: 'product-1',
    name: '테스트 상품',
  });

  // When
  const metadata = await generateMetadata({
    params: Promise.resolve({
      productId: 'product-1',
    }),
  });

  // Then
  expect(metadata.title).toBe('테스트 상품');
});
```

실제 `<head>` 병합 결과나 전체 Next.js metadata 처리 과정은 E2E 테스트 대상으로 본다.

---

## Suspense와 Loading UI

`loading.tsx`는 일반 동기 컴포넌트라면 직접 렌더링해 테스트할 수 있다.

```tsx
it('상품 목록 로딩 상태를 보여준다', () => {
  // Given
  render(<Loading />);

  // When
  const status = screen.getByRole('status');

  // Then
  expect(status).toHaveAccessibleName('상품 목록 불러오는 중');
});
```

Suspense 경계가 포함된 Client Component는 fallback과 완료 상태를 검증할 수 있다.

다만 React Server Component 스트리밍과 Next.js 라우트 단위 loading 처리 전체는 Vitest가 아니라 E2E 테스트를 우선한다.

---

## Vitest 설정

Next.js 공식 설정 형태를 우선 사용한다.

```ts
// vitest.config.mts
import react from '@vitejs/plugin-react';
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    tsconfigPaths(),
    react(),
  ],
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
  },
});
```

TypeScript path alias를 사용하는 경우 `vite-tsconfig-paths` 또는 프로젝트의 기존 alias 설정이 Vitest에도 적용되는지 확인한다.

Next.js의 Webpack 또는 Turbopack 설정이 Vitest에 자동으로 적용된다고 가정하지 않는다.

다음 항목은 별도의 Vitest 설정이 필요할 수 있다.

* path alias
* SVG component import
* CSS module
* 정적 asset
* custom transform
* monorepo package resolution

Next.js 빌드 성공과 Vitest 실행 성공은 별도의 검증으로 취급한다.

---

## jsdom 환경의 한계

Vitest의 `jsdom`은 Next.js 서버 런타임을 재현하지 않는다.

다음 기능은 jsdom 기반 컴포넌트 테스트만으로 완전히 검증할 수 없다.

* React Server Component 렌더링
* 서버와 클라이언트 경계
* streaming
* hydration 전체 과정
* Next.js caching
* route segment 처리
* middleware 또는 proxy
* 실제 redirect 응답
* 실제 image optimization
* production bundling
* Edge Runtime
* 브라우저 navigation 전체 흐름

이러한 기능은 필요에 따라 다음으로 검증한다.

* 순수 함수 단위 테스트
* Route Handler 단위 테스트
* Server Action 단위 테스트
* integration test
* Playwright 또는 Cypress E2E 테스트
* `next build`

---

## Mock 규칙

Mock은 외부 경계에만 사용한다.

Mock해도 되는 대상:

* 네트워크 요청
* 브라우저 API
* `next/navigation`
* `next/headers`
* `next/cache`
* 인증 모듈
* 데이터베이스 접근 계층
* analytics
* 날짜와 시간
* localStorage / sessionStorage
* feature flag
* 외부 SDK

상황에 따라 mock 가능한 대상:

* `next/image`
* `next/link`
* Server Action
* Next.js 전용 Provider

`next/image`와 `next/link`는 테스트 환경에서 실제 사용이 가능한 경우 불필요하게 mock하지 않는다.

되도록 mock하지 말아야 하는 대상:

* 테스트 대상 컴포넌트 자체
* 단순한 child component
* 내부 유틸 함수
* 모든 Server Component
* assertion을 쉽게 만들기 위한 과도한 Next.js API mock

Next.js API mock이 너무 많아지면 unit test보다 E2E 테스트가 더 적절한지 검토한다.

---

## 테스트 파일 구조

Next.js App Router에서는 테스트 파일을 대상 파일 가까이에 둘 수 있다.

```txt
app/
  products/
    page.tsx
    page.test.tsx
    loading.tsx
    loading.test.tsx
```

Route Handler는 다음처럼 둘 수 있다.

```txt
app/
  api/
    products/
      route.ts
      route.test.ts
```

Server Action은 다음처럼 둘 수 있다.

```txt
app/
  products/
    actions.ts
    actions.test.ts
```

프로젝트가 `__tests__` 구조를 사용한다면 기존 규칙을 따른다.

테스트 파일이 Next.js route로 인식되지 않는지는 현재 프로젝트 설정과 빌드 결과를 확인한다.

---

## AI 에이전트가 테스트 작성 전에 확인해야 할 것

테스트를 작성하거나 수정하기 전에 반드시 다음을 확인한다.

1. Next.js 버전
2. App Router 또는 Pages Router 사용 여부
3. 테스트 대상이 Server Component인지 Client Component인지
4. 테스트 대상이 동기인지 비동기인지
5. public input과 output
6. 사용자에게 보이는 동작
7. 외부 의존성
8. 주변 테스트 파일의 스타일
9. 프로젝트의 Vitest 설정
10. DOM test environment 설정
11. custom render utility 존재 여부
12. 이미 정의된 Next.js mock 또는 setup 파일 존재 여부
13. path alias 설정
14. Server Action 또는 Route Handler 사용 여부
15. Vitest보다 E2E 테스트가 적절한 대상인지

기존 테스트 유틸과 Next.js mock이 있다면 새로 만들지 말고 재사용한다.

---

## AI 에이전트가 하면 안 되는 것

다음 행동은 하지 않는다.

* 비동기 Server Component를 억지로 React Testing Library에서 렌더링하기
* Vitest로 Next.js 서버 런타임 전체를 재현하려고 하기
* 모든 Next.js 컴포넌트를 일괄 mock하기
* `next/link`의 navigation 의도 대신 내부 구현을 검증하기
* `next/image`의 최적화 URL 형식을 검증하기
* App Router 프로젝트에서 `next/router`를 사용하기
* Pages Router 프로젝트에서 `next/navigation`을 사용하기
* Server Action 테스트에서 인증과 권한 검사를 생략하기
* `process.env` 대신 `import.meta.env`를 사용하기
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
* Jest 전용 API를 Vitest 테스트에 섞어 쓰기

---

## 테스트 작성이 어려운 경우

테스트 작성이 어렵다면 무리하게 Next.js API mock을 늘리지 않는다.

먼저 다음을 설명한다.

1. 테스트가 어려운 이유
2. Server Component, Server Action, Route Handler 중 어떤 경계에서 문제가 발생하는지
3. 어떤 Next.js API 또는 외부 의존성 때문에 문제가 생기는지
4. Vitest 단위 테스트로 검증 가능한 범위
5. integration 또는 E2E 테스트가 더 적절한 범위
6. production code를 개선한다면 어떤 구조가 좋은지

비동기 Server Component, streaming, hydration, middleware, caching처럼 Next.js 런타임 의존성이 큰 기능은 E2E 테스트를 우선 검토한다.

단, production code 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

---

## 완료 전 체크리스트

작업을 마치기 전에 다음을 확인한다.

* [ ] Given-When-Then 구조를 따른다
* [ ] 테스트 이름이 동작을 설명한다
* [ ] App Router와 Pages Router를 구분했다
* [ ] Server Component와 Client Component를 구분했다
* [ ] 비동기 Server Component를 무리하게 Vitest로 렌더링하지 않았다
* [ ] 사용자 행동은 `userEvent`를 사용한다
* [ ] query는 접근 가능한 selector를 우선한다
* [ ] 비동기 처리는 `findBy*` 또는 `waitFor`를 사용한다
* [ ] mock은 외부 경계에만 제한적으로 사용한다
* [ ] `next/navigation`, `next/headers`, `next/cache` mock은 필요한 API만 포함한다
* [ ] `next/link`는 가능하면 실제 `href`를 검증한다
* [ ] `next/image` 내부 구현에 assertion이 결합되지 않았다
* [ ] 환경변수는 `process.env` 기준으로 처리한다
* [ ] Server Action의 인증과 권한 검사를 고려했다
* [ ] Route Handler의 status와 response body를 검증했다
* [ ] 중요한 에러 케이스를 포함한다
* [ ] 테스트 간 상태가 격리되어 있다
* [ ] `.only`, `.skip`, 임의 delay가 없다
* [ ] TypeScript 타입을 보존한다
* [ ] 테스트를 읽는 사람이 과도한 mock setup 없이 이해할 수 있다
* [ ] Vitest보다 E2E가 적합한 범위를 구분했다
* [ ] 실제 동작이 깨지면 테스트도 실패한다

---

## AI 에이전트의 최종 응답 형식

AI 에이전트가 테스트를 작성하거나 수정한 뒤에는 다음 내용을 요약한다.

1. 어떤 동작을 테스트했는지
2. 테스트 대상이 Client Component, Server Component, Server Action, Route Handler 중 무엇인지
3. 어떤 mock을 추가했고 왜 필요한지
4. 어떤 edge case를 포함했는지
5. Vitest로 검증하기 어려워 E2E가 필요한 영역이 있는지
6. 실행한 Vitest, type-check, lint 또는 build 결과