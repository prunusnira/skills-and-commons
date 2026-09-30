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


## React + Vite 환경 규칙

### 1. `@storybook/react-vite`

React + Vite 프로젝트에서는 Storybook framework를 `@storybook/react-vite` 기준으로 확인한다.

```ts
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
};

export default config;
```

프로젝트가 이미 다른 Storybook framework를 사용 중이라면 임의로 변경하지 않는다.

---

### 2. Vite path alias

컴포넌트 import는 프로젝트의 기존 alias 규칙을 따른다.

```tsx
import { Button } from './Button';
```

또는

```tsx
import { Button } from '@/components/Button';
```

Storybook에서 alias가 동작하지 않는다면 `.storybook/main.ts`의 `viteFinal` 설정이나 프로젝트의 `vite.config.ts`, `tsconfig.json`을 먼저 확인한다.

Story 파일 작성을 위해 새로운 alias를 임의로 만들지 않는다.

---

### 3. 정적 asset

Vite public asset은 일반적으로 `/` 기준 경로를 사용한다.

```tsx
const mockProduct = {
  imageUrl: '/images/mock/product.png',
};
```

컴포넌트가 import된 asset을 받는 구조라면 기존 코드와 같은 방식을 따른다.

```tsx
import productImageUrl from './product.png';

export const WithImage: Story = {
  args: {
    imageUrl: productImageUrl,
  },
};
```

Storybook에서 이미지가 보이지 않는다면 asset 위치, Vite 설정, Storybook static directory 설정을 확인한다.

---

### 4. Vite 환경 변수

`import.meta.env`에 의존하는 컴포넌트는 Storybook 환경에서도 필요한 값이 존재하는지 확인한다.

Story에서 실제 서비스 환경값에 의존하지 않는다.

환경 변수 때문에 Story 렌더링이 깨진다면 다음 중 하나를 우선 검토한다.

- Storybook 실행 환경의 `.env` 설정
- `.storybook/main.ts`의 Vite define 설정
- 컴포넌트에 필요한 값을 props로 주입할 수 있는지
- MSW 또는 mock data로 외부 의존성을 대체할 수 있는지

---

### 5. React Router

React Router에 의존하는 컴포넌트는 `MemoryRouter` 같은 실제 router provider로 감싼다.

```tsx
import { MemoryRouter } from 'react-router-dom';

const meta = {
  component: ProductLink,
  decorators: [
    (Story) => (
      <MemoryRouter initialEntries={['/products/1']}>
        <Story />
      </MemoryRouter>
    ),
  ],
} satisfies Meta<typeof ProductLink>;
```

라우터 hook을 Story 파일에서 직접 mock하기보다 provider를 우선 사용한다.

프로젝트에 이미 전역 router decorator가 있다면 Story 파일에서 중복으로 추가하지 않는다.

---

### 6. CSS와 전역 스타일

컴포넌트가 전역 CSS, reset CSS, design token, theme에 의존한다면 `.storybook/preview.ts` 또는 `.storybook/preview.tsx`에서 이미 import되어 있는지 확인한다.

```ts
// .storybook/preview.ts
import '../src/styles/global.css';
```

개별 Story에서 전역 CSS를 중복 import하지 않는다.

CSS Modules, vanilla-extract, styled-components, emotion, Tailwind 등은 프로젝트의 기존 설정을 따른다.

---


## Storybook 설정 파일 기준

### `.storybook/main.ts`

```ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
};

export default config;
```

프로젝트가 이미 `@storybook/react-vite`가 아닌 다른 framework를 쓰고 있다면 기존 설정을 먼저 확인하고, React + Vite 전환이 명시적으로 필요한 경우에만 framework 변경을 제안한다.

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

Story 파일 작성을 위해 `.storybook/main.ts`, `.storybook/preview.tsx`, `tsconfig.json`, `vite.config.ts`를 수정해야 한다면 기존 설정과 충돌하지 않는지 먼저 확인한다.

특히 다음 설정은 임의로 변경하지 않는다.

- TypeScript `module`
- path alias
- SVGR 설정
- static directory 설정
- MSW 초기화 설정
- 전역 Provider decorator
- framework 종류
- addon 구성
- Vite plugin 구성

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
- Vitest에서 검증해야 할 복잡한 로직을 play 함수에 몰아넣은 Story

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
