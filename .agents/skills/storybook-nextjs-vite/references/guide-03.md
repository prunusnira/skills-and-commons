## 필수 Story 기준

컴포넌트의 성격에 따라 가능한 상태를 빠짐없이 작성한다.

해당 상태가 없는 컴포넌트에 억지로 만들지는 않는다.

---

### 공통 상태

가능하면 다음 Story를 고려한다.

- `Default`
- `Disabled`
- `Loading`
- `Error`
- `Empty`
- `WithLongText`
- `Mobile`
- `Desktop`

---

### Button 계열

- `Primary`
- `Secondary`
- `Disabled`
- `Loading`
- `FullWidth`
- `WithIcon`

---

### Input / Form 계열

- `Default`
- `WithPlaceholder`
- `WithValue`
- `Disabled`
- `Error`
- `Required`
- `WithHelperText`
- `Submitting`

---

### Card 계열

- `Default`
- `WithImage`
- `WithoutImage`
- `WithLongTitle`
- `Loading`
- `Empty`
- `SoldOut`
- `Discounted`

---

### Modal / Dialog 계열

- `Open`
- `Confirm`
- `Alert`
- `WithLongContent`
- `Mobile`

Modal은 기본적으로 열린 상태를 보여주는 Story를 작성한다.

---

### List / Table 계열

- `Default`
- `Empty`
- `Loading`
- `Error`
- `WithManyItems`
- `WithSelectedItem`
- `Mobile`

---


## 반응형 Story 작성 규칙

반응형 UI가 중요한 컴포넌트는 viewport를 지정한다.

```tsx
export const Mobile: Story = {
  parameters: {
    viewport: {
      defaultViewport: 'mobile1',
    },
  },
};
```

```tsx
export const Desktop: Story = {
  parameters: {
    viewport: {
      defaultViewport: 'desktop',
    },
  },
};
```

프로젝트에서 정의한 viewport 이름이 있다면 그 값을 사용한다.

---


## MSW 사용 규칙

API 응답에 따라 UI가 달라지는 컴포넌트는 MSW를 사용할 수 있다.

다음 경우에는 MSW 사용을 고려한다.

- 로딩 상태
- 성공 상태
- 빈 데이터 상태
- 에러 상태
- 권한 없음 상태
- 페이지네이션 상태

```tsx
import { http, HttpResponse } from 'msw';

export const Success: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get('/api/products', () => {
          return HttpResponse.json({
            items: [
              {
                id: 1,
                name: '샘플 상품',
                price: 10000,
              },
            ],
          });
        }),
      ],
    },
  },
};
```

단순 presentational component에는 MSW를 사용하지 않는다.

props로 상태를 표현할 수 있다면 args를 우선 사용한다.

Storybook에서는 실제 API를 호출하지 않는다.

---


## Next.js 환경 규칙

### 1. 서버 컴포넌트

Storybook은 기본적으로 브라우저에서 컴포넌트를 렌더링하므로, 서버 컴포넌트를 직접 Story 대상으로 삼지 않는다.

서버 컴포넌트가 필요한 경우:

- UI 표현을 담당하는 Client Component를 분리한다.
- Story는 분리된 Client Component를 대상으로 작성한다.
- 데이터 fetch는 args나 mock data로 대체한다.

---

### 2. `next/image`

`next/image`를 사용하는 컴포넌트는 Storybook 환경에서 정상적으로 표시되도록 설정이 필요할 수 있다.

프로젝트에 이미 설정된 Storybook 설정을 우선 따른다.

필요한 경우 `.storybook/preview.tsx` 또는 `.storybook/main.ts`의 기존 설정을 확인한다.

동일 목적의 설정을 Story 파일마다 중복으로 추가하지 않는다.

---

### 3. `next/navigation`

`useRouter`, `usePathname`, `useSearchParams` 등 Next.js 라우팅 훅을 사용하는 컴포넌트는 mock 처리가 필요할 수 있다.

이미 프로젝트에 라우터 mock 설정이 있다면 반드시 재사용한다.

Story 파일 안에서 임시 mock을 남발하지 않는다.

---

### 4. `@storybook/nextjs-vite`

Next.js 프로젝트에서 `@storybook/nextjs-vite`를 사용하는 경우, 설정 파일의 framework는 프로젝트 설정을 따른다.

```ts
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/nextjs-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: {
    name: '@storybook/nextjs-vite',
    options: {},
  },
};

export default config;
```

React Vite 프로젝트가 아니라 Next.js 프로젝트라면 `@storybook/react-vite`로 임의 변경하지 않는다.

---


## Storybook 설정 파일 기준

### `.storybook/main.ts`

```ts
import type { StorybookConfig } from '@storybook/nextjs-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: {
    name: '@storybook/nextjs-vite',
    options: {},
  },
};

export default config;
```

프로젝트가 `@storybook/react-vite`, `@storybook/nextjs`, `@storybook/nextjs-vite` 중 무엇을 쓰는지 먼저 확인하고 기존 설정을 따른다.

---

### `.storybook/preview.ts`

```ts
import type { Preview } from '@storybook/react';

const preview: Preview = {
  parameters: {
    actions: {
      argTypesRegex: '^on[A-Z].*',
    },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/,
      },
    },
  },
};

export default preview;
```

전역 decorator가 필요하다면 `preview.tsx`로 변경할 수 있다.

```tsx
import type { Preview } from '@storybook/react';

const preview: Preview = {
  decorators: [
    (Story) => (
      <div style={{ fontFamily: 'Arial, sans-serif' }}>
        <Story />
      </div>
    ),
  ],
};

export default preview;
```

전역 decorator를 추가할 때는 모든 Story에 영향을 주므로 기존 Story가 깨지지 않는지 확인한다.

---


## Storybook 설정 변경 규칙

Story 파일 작성을 위해 `.storybook/main.ts`, `.storybook/preview.tsx`, `tsconfig.json`, Vite 설정을 수정해야 한다면 기존 설정과 충돌하지 않는지 먼저 확인한다.

특히 다음 설정은 임의로 변경하지 않는다.

- TypeScript `module`
- path alias
- SVGR 설정
- Next.js image 설정
- MSW 초기화 설정
- 전역 Provider decorator
- framework 종류
- addon 구성

설정 변경이 필요한 경우, Story 파일 작성과 분리해서 별도 변경으로 제안한다.

---


## 작성하면 안 되는 Story

다음과 같은 Story는 작성하지 않는다.

- 실제로 존재하지 않는 UI 상태
- props 타입을 무시한 억지 상태
- 의미 없는 더미 데이터만 들어간 Story
- 테스트를 위해서만 존재하는 비현실적인 Story
- 컴포넌트 내부 구현에 의존하는 Story
- 실제 API를 호출하는 Story
- 모든 props 조합을 기계적으로 나열한 Story
- Jest에서 검증해야 할 복잡한 로직을 play 함수에 몰아넣은 Story

---


## import 규칙

상대 경로 또는 프로젝트 alias는 기존 프로젝트 규칙을 따른다.

```tsx
import { Button } from './Button';
```

또는

```tsx
import { Button } from '@/components/Button';
```

새로운 alias를 임의로 만들지 않는다.

---


## TypeScript 규칙

`Meta`, `StoryObj`를 사용한다.

```tsx
import type { Meta, StoryObj } from '@storybook/react';
```

`any`는 사용하지 않는다.

가능하면 `satisfies Meta<typeof Component>`를 사용한다.

```tsx
const meta = {
  component: Button,
} satisfies Meta<typeof Button>;
```

컴포넌트 props 타입을 별도로 가져와야 하는 경우에만 import한다.

```tsx
import type { ButtonProps } from './Button';
```

---
