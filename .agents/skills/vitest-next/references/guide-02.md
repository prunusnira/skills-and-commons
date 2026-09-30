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
