# Storybook 작성 스킬

## 목적

이 문서는 AI 에이전트가 Next.js + TypeScript 환경에서 Storybook stories를 일관성 있게 작성하도록 돕기 위한 기준이다.

Storybook은 단순히 컴포넌트를 나열하는 문서가 아니라, 다음 목적을 가진다.

- 컴포넌트의 사용 예시를 명확히 보여준다.
- 주요 상태와 변형을 빠짐없이 확인할 수 있게 한다.
- 디자이너, 개발자, QA가 UI 동작을 독립적으로 검토할 수 있게 한다.
- 시각적 회귀 테스트와 인터랙션 검증의 기반이 된다.

---

## 핵심 원칙

### 1. CSF 3.0 형식을 사용한다

Story는 최신 Component Story Format 3.0 형식으로 작성한다.

구형 render function 중심의 CSF 2.0 스타일은 사용하지 않는다.

```tsx
// ❌ CSF 2.0 스타일
export default {
  title: 'Components/Button',
  component: Button,
};

export const Primary = () => <Button variant="primary">Click me</Button>;
```

```tsx
// ✅ CSF 3.0 스타일
import type { Meta, StoryObj } from '@storybook/react';

import { Button } from './Button';

const meta = {
  component: Button,
  tags: ['autodocs'],
  args: {
    variant: 'primary',
    children: 'Click me',
  },
} satisfies Meta<typeof Button>;

export default meta;

type Story = StoryObj<typeof meta>;

export const Primary: Story = {};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
  },
};
```

---

### 2. Args 기반으로 Story를 작성한다

컴포넌트 props는 가능한 한 `args`로 정의한다.

`args`를 사용하면 Storybook Controls 패널에서 props를 인터랙티브하게 조작할 수 있다.

```tsx
// ❌ props를 render 안에 하드코딩
export const Disabled: Story = {
  render: () => <Button disabled>Disabled</Button>,
};
```

```tsx
// ✅ args 사용
export const Disabled: Story = {
  args: {
    disabled: true,
  },
};
```

---

### 3. 기본값은 `args`에 둔다

기본값은 `argTypes.defaultValue`가 아니라 `args`에 선언한다.

여러 Story에서 공통으로 필요한 값은 `meta.args`에 둔다.

개별 Story에서는 차이점만 override한다.

```tsx
// ❌ 같은 args 중복
export const Primary: Story = {
  args: {
    children: 'Click me',
    variant: 'primary',
  },
};

export const Secondary: Story = {
  args: {
    children: 'Click me',
    variant: 'secondary',
  },
};
```

```tsx
// ✅ meta.args에 공통값 선언
const meta = {
  component: Button,
  args: {
    children: 'Click me',
    variant: 'primary',
  },
} satisfies Meta<typeof Button>;

export default meta;

type Story = StoryObj<typeof meta>;

export const Primary: Story = {};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
  },
};

export const Disabled: Story = {
  args: {
    disabled: true,
  },
};
```

---

### 4. `title`은 기본적으로 생략한다

Storybook은 파일 경로를 기반으로 사이드바 계층을 자동 추론할 수 있다.

`title`을 문자열로 직접 명시하면 컴포넌트 이름이나 파일 경로가 변경되었을 때 Storybook 사이드바 구조와 실제 파일 구조가 어긋날 수 있다.

```tsx
// ❌ title 직접 명시
const meta = {
  title: 'Components/Button',
  component: Button,
} satisfies Meta<typeof Button>;
```

```tsx
// ✅ title 생략
const meta = {
  component: Button,
} satisfies Meta<typeof Button>;
```

단, 프로젝트에서 명시적인 Storybook 사이드바 구조를 관리하고 있다면 기존 프로젝트 규칙을 따른다.

---

### 5. 타입 안전한 Meta 정의를 사용한다

`Meta<typeof Component>`를 직접 타입 annotation으로 붙이기보다 `satisfies`를 사용한다.

`satisfies`는 타입 체크와 타입 추론을 함께 활용할 수 있다.

```tsx
// ❌ 타입 추론이 제한됨
const meta: Meta<typeof Button> = {
  component: Button,
};
```

```tsx
// ✅ 타입 체크와 추론을 함께 활용
const meta = {
  component: Button,
  args: {
    size: 'md',
    variant: 'primary',
  },
} satisfies Meta<typeof Button>;

export default meta;

type Story = StoryObj<typeof meta>;
```

---

### 6. 하나의 Story는 하나의 상태나 목적만 표현한다

여러 상태를 하나의 Story에 과하게 섞지 않는다.

```tsx
// ❌ 여러 상태를 한 Story에 섞음
export const ButtonExamples = () => (
  <>
    <Button variant="primary">확인</Button>
    <Button variant="secondary">취소</Button>
    <Button disabled>비활성</Button>
  </>
);
```

```tsx
// ✅ 상태별 Story 분리
export const Primary: Story = {};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
  },
};

export const Disabled: Story = {
  args: {
    disabled: true,
  },
};
```

다만 디자인 비교가 목적인 경우에는 `Variants`, `Sizes`, `States` 같은 비교용 Story를 별도로 작성할 수 있다.

비교용 Story는 개별 Story를 대체하지 않는다.

---

### 7. 상호작용과 그 결과를 확인 가능하게 작성한다

상호작용이 있는 컴포넌트는 정적인 상태만 보여주는 것으로 끝내지 않고, 사용자가 요소에 가하는 상호작용(클릭, 입력, 선택, 토글 등)과 그 결과(콜백 호출, 상태 변화, UI 전환)를 Storybook에서 직접 확인할 수 있도록 작성한다.

이벤트 핸들러는 `fn()` 또는 actions로 연결해 Actions 패널에서 결과를 볼 수 있게 하고, 상호작용 흐름이 중요한 컴포넌트는 `play` 함수로 실제 동작을 재현한다.

상호작용이 가능한 컴포넌트를 정적인 모습만 보여주는 Story로 남기지 않는다.

---

## 권장 Story 구조

```tsx
import type { Meta, StoryObj } from '@storybook/react';

import { Button } from './Button';

const meta = {
  component: Button,
  tags: ['autodocs'],
  parameters: {
    layout: 'centered',
  },
  args: {
    children: 'Button',
    size: 'md',
    variant: 'primary',
  },
  argTypes: {
    children: {
      control: 'text',
    },
  },
} satisfies Meta<typeof Button>;

export default meta;

type Story = StoryObj<typeof meta>;

export const Primary: Story = {};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
  },
};

export const Disabled: Story = {
  args: {
    disabled: true,
  },
};

export const Loading: Story = {
  args: {
    loading: true,
  },
};

export const WithLongText: Story = {
  args: {
    children: '매우 긴 버튼 텍스트가 들어갔을 때의 표시를 확인합니다',
  },
};
```

---

## 파일 작성 규칙

### 1. Story 파일 위치

컴포넌트와 가까운 위치에 작성한다.

```txt
components/
  Button/
    Button.tsx
    Button.stories.tsx
    Button.test.tsx
```

프로젝트가 stories를 분리하는 구조라면 기존 구조를 따른다.

```txt
components/
  Button/
    Button.tsx

stories/
  components/
    Button.stories.tsx
```

---

### 2. 파일명

Story 파일명은 다음 형식을 따른다.

```txt
<ComponentName>.stories.tsx
```

예:

```txt
Button.stories.tsx
ProductCard.stories.tsx
EmailForm.stories.tsx
```

---

## Args 작성 규칙

### 1. 의미 있는 값을 사용한다

Story에 들어가는 값은 실제 서비스에서 사용될 법한 값으로 작성한다.

```tsx
// ❌ 의미 없는 값
args: {
  title: 'test',
  description: 'description',
}
```

```tsx
// ✅ 실제 서비스에 가까운 값
args: {
  title: '여름 신상품 특가',
  description: '오늘 하루만 적용되는 한정 할인 상품입니다.',
}
```

---

### 2. 긴 텍스트는 의도가 있을 때만 사용한다

긴 텍스트 상태를 테스트할 때만 의도적으로 긴 텍스트를 사용한다.

```tsx
export const WithLongTitle: Story = {
  args: {
    title:
      '이 상품명은 두 줄 이상 노출될 수 있는 매우 긴 상품명 예시입니다',
  },
};
```

---

### 3. 실제 API 응답처럼 보이는 mock data를 사용한다

```tsx
const mockProduct = {
  id: 1,
  name: '오가닉 코튼 티셔츠',
  price: 29000,
  imageUrl: '/images/mock/product.png',
};
```

```tsx
// ❌ 피한다
const mockProduct = {
  id: 1,
  name: 'test',
  price: 123,
};
```

여러 Story에서 재사용하는 mock data는 별도 파일로 분리할 수 있다.

```txt
ProductCard.mock.ts
ProductCard.stories.tsx
```

---

## ArgTypes 작성 규칙

### 1. 자동 추론을 우선한다

Storybook은 컴포넌트의 TypeScript 타입을 기반으로 argTypes를 자동 추론한다.

타당한 이유 없이 `argTypes`를 직접 지정하지 않는다.

```tsx
// ❌ 불필요한 argTypes 수동 지정
const meta = {
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'tertiary'],
    },
    size: {
      control: 'radio',
      options: ['sm', 'md', 'lg'],
    },
    disabled: {
      control: 'boolean',
    },
  },
} satisfies Meta<typeof Button>;
```

```tsx
// ✅ 자동 추론에 맡김
const meta = {
  component: Button,
} satisfies Meta<typeof Button>;
```

---

### 2. 수동 지정은 필요한 경우에만 사용한다

다음 경우에는 `argTypes`를 수동 지정할 수 있다.

- `ReactNode` 타입이지만 Controls에서 텍스트 입력이 필요한 경우
- 자동 추론된 Control이 부적절한 경우
- 특정 Story에서 prop을 고정해야 하는 경우
- compound component pattern처럼 자동 추론이 어려운 경우
- 이벤트 핸들러를 Actions 패널에 연결해야 하는 경우

```tsx
const meta = {
  component: Button,
  argTypes: {
    children: {
      control: 'text',
    },
    opacity: {
      control: {
        type: 'range',
        min: 0,
        max: 1,
        step: 0.1,
      },
    },
    onClick: {
      action: 'clicked',
    },
  },
} satisfies Meta<typeof Button>;
```

---

### 3. 특정 Story에서 prop을 고정할 수 있다

```tsx
export const Horizontal: Story = {
  args: {
    orientation: 'horizontal',
  },
  argTypes: {
    orientation: {
      control: false,
    },
  },
};
```

---

## Parameters 작성 규칙

`parameters`는 Storybook의 표시 방식이나 addon 동작을 조정할 때 사용한다.

```tsx
const meta = {
  component: Button,
  parameters: {
    layout: 'centered',
    backgrounds: {
      default: 'light',
      values: [
        { name: 'light', value: '#ffffff' },
        { name: 'dark', value: '#333333' },
      ],
    },
  },
} satisfies Meta<typeof Button>;
```

개별 Story에서 override할 수 있다.

```tsx
export const OnDark: Story = {
  parameters: {
    backgrounds: {
      default: 'dark',
    },
  },
};
```

---

### 자주 사용하는 parameters

```tsx
parameters: {
  layout: 'centered' | 'fullscreen' | 'padded',

  backgrounds: {
    default: 'light',
    values: [
      { name: 'light', value: '#ffffff' },
      { name: 'dark', value: '#333333' },
    ],
  },

  actions: {
    argTypesRegex: '^on[A-Z].*',
  },

  docs: {
    description: {
      component: '컴포넌트 설명',
    },
  },
}
```

---

### layout 선택 기준

- 버튼, 아이콘, 태그: `centered`
- 카드, 폼, 리스트 아이템: `padded`
- 페이지, 레이아웃, 모달 화면 전체: `fullscreen`

---

## Decorators 작성 규칙

컴포넌트가 특정 Provider, context, layout wrapper에 의존한다면 decorator를 사용한다.

```tsx
export const WithTheme: Story = {
  decorators: [
    (Story) => (
      <ThemeProvider theme="dark">
        <Story />
      </ThemeProvider>
    ),
  ],
};
```

공통 wrapper가 필요하면 meta 수준에 작성한다.

```tsx
const meta = {
  component: Button,
  decorators: [
    (Story) => (
      <div style={{ padding: '3rem' }}>
        <Story />
      </div>
    ),
  ],
} satisfies Meta<typeof Button>;
```

전역 Provider가 이미 `.storybook/preview.tsx`에 설정되어 있다면 Story 파일에서 중복으로 감싸지 않는다.

---

### Decorator 패턴 예시

```tsx
// 스타일 래퍼
(Story) => (
  <div style={{ padding: '3rem' }}>
    <Story />
  </div>
)
```

```tsx
// Theme Provider
(Story) => (
  <ThemeProvider theme="dark">
    <Story />
  </ThemeProvider>
)
```

```tsx
// Router Provider
(Story) => (
  <MemoryRouter initialEntries={['/']}>
    <Story />
  </MemoryRouter>
)
```

```tsx
// 다국어 Provider
(Story) => (
  <I18nProvider locale="ko">
    <Story />
  </I18nProvider>
)
```

```tsx
// 전역 상태 Provider
(Story) => (
  <Provider store={mockStore}>
    <Story />
  </Provider>
)
```

---

## Actions 작성 규칙

이벤트 핸들러는 Storybook actions 또는 `fn()`을 사용한다.

상호작용 테스트가 필요하다면 `@storybook/test`의 `fn()`을 우선 사용한다.

```tsx
import { fn } from '@storybook/test';

const meta = {
  component: Button,
  args: {
    onClick: fn(),
  },
} satisfies Meta<typeof Button>;
```

Actions 패널에 표시만 하면 되는 경우에는 `argTypes`를 사용할 수 있다.

```tsx
const meta = {
  component: Button,
  argTypes: {
    onClick: {
      action: 'clicked',
    },
  },
} satisfies Meta<typeof Button>;
```

---

## Play 함수 작성 규칙

사용자 상호작용이 중요한 컴포넌트는 `play` 함수를 작성한다.

```tsx
import { expect, fn, userEvent, within } from '@storybook/test';

const meta = {
  component: Button,
  args: {
    onClick: fn(),
    children: '클릭',
  },
} satisfies Meta<typeof Button>;

export default meta;

type Story = StoryObj<typeof meta>;

export const Clickable: Story = {
  play: async ({ canvasElement, args }) => {
    const canvas = within(canvasElement);

    await userEvent.click(canvas.getByRole('button', { name: '클릭' }));

    await expect(args.onClick).toHaveBeenCalled();
  },
};
```

`play` 함수는 다음 경우에 우선 작성한다.

- 버튼 클릭
- 체크박스 / 라디오 선택
- 입력값 변경
- 드롭다운 열기
- 모달 열기 / 닫기
- 탭 변경
- 토글 상태 변경

복잡한 비즈니스 로직 검증은 Storybook `play`가 아니라 Jest 테스트에서 처리한다.

---

## 접근성 기준

Story나 play 함수에서 요소를 찾을 때는 가능한 한 사용자가 인식하는 방식으로 찾는다.

```tsx
canvas.getByRole('button', { name: '확인' });
canvas.getByLabelText('이메일');
canvas.getByText('상품명');
```

`data-testid`는 접근 가능한 role, label, text로 찾기 어려운 경우에만 사용한다.

```tsx
// ❌ 우선 사용하지 않는다
canvas.getByTestId('submit-button');
```

---

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

## Storybook과 Jest 테스트의 역할 분리

Storybook과 Jest는 모두 UI 품질을 높이기 위한 도구지만, 검증 목적이 다르다.

Storybook은 컴포넌트의 상태, 변형, 사용 예시, 시각적 표현을 문서화하는 데 집중한다.

Jest와 Testing Library는 비즈니스 로직, 조건 분기, 사용자 행위 결과, 예외 케이스를 검증하는 데 집중한다.

---

### Storybook이 담당하는 것

Storybook은 다음 항목을 우선 담당한다.

- 컴포넌트의 기본 사용 예시
- props 조합에 따른 시각적 변화
- 로딩, 에러, 빈 데이터, 비활성 상태
- 긴 텍스트, 이미지 없음, 데이터 없음 같은 UI edge case
- 모바일 / 데스크톱 반응형 상태
- 디자이너, QA, 기획자가 확인할 수 있는 UI 문서화
- 간단한 사용자 상호작용 흐름

Storybook의 핵심 질문은 다음과 같다.

> 이 컴포넌트가 어떤 상태에서 어떻게 보이는가?

---

### Jest가 담당하는 것

Jest와 Testing Library는 다음 항목을 우선 담당한다.

- 조건 분기 로직
- 사용자 입력에 따른 상태 변경
- 이벤트 핸들러 호출 여부
- 유효성 검사
- API 응답에 따른 렌더링 결과
- 권한, 상태값, feature flag 등에 따른 분기
- 예외 상황 처리
- custom hook / util 함수 / domain logic 검증

Jest의 핵심 질문은 다음과 같다.

> 사용자가 행동했을 때 의도한 결과가 발생하는가?

---

### Storybook play 함수의 위치

Storybook `play` 함수는 Jest 테스트를 완전히 대체하지 않는다.

`play` 함수는 Story 안에서 보여주는 UI 상태가 실제로 가볍게 상호작용 가능한지 확인하는 용도로 사용한다.

적합한 예:

- 버튼 클릭 시 callback 호출 확인
- 드롭다운 열기
- 탭 전환
- 모달 닫기
- 체크박스 선택
- 간단한 입력 상태 확인

다음 검증은 Storybook `play` 함수에 넣지 않고 Jest에서 처리한다.

- 복잡한 비즈니스 규칙
- 여러 API 응답 조합
- 권한별 분기
- 날짜 / 금액 / 할인율 계산
- 서버 상태와 캐시 동작
- 실패 재시도 로직
- custom hook 내부 동작
- util 함수의 경계값 검증

---

### 중복 작성 기준

Storybook과 Jest에서 일부 시나리오가 겹칠 수 있다.

겹쳐도 되는 경우:

- 핵심 UI 상태를 Storybook에 문서화하고, 같은 조건의 동작 결과를 Jest에서 검증하는 경우
- 사용자에게 매우 중요한 플로우를 Storybook `play`와 Jest 양쪽에서 가볍게 확인하는 경우
- 회귀 위험이 큰 UI 상태를 시각적 문서와 테스트 양쪽으로 보호하는 경우

피해야 하는 경우:

- Storybook에 이미 있는 모든 상태를 Jest에서 단순 render 테스트로 반복하는 경우
- Jest에서 검증한 모든 로직 케이스를 Storybook Story로 억지로 만드는 경우
- Storybook `play` 함수에 Jest 수준의 복잡한 테스트를 몰아넣는 경우

---

### 역할 분리 예시

#### Button 컴포넌트

Storybook:

- `Primary`
- `Secondary`
- `Disabled`
- `Loading`
- `WithIcon`
- `FullWidth`

Jest:

- disabled 상태에서는 클릭 핸들러가 호출되지 않는지
- loading 상태에서는 중복 submit이 막히는지
- 접근 가능한 button name이 올바른지

---

#### ProductCard 컴포넌트

Storybook:

- `Default`
- `SoldOut`
- `Discounted`
- `WithoutImage`
- `WithLongName`
- `Loading`

Jest:

- 할인율 계산이 올바른지
- 품절 상품 클릭 시 이동하지 않는지
- 가격 포맷이 올바른지
- 상품 링크 href가 올바른지

---

#### Form 컴포넌트

Storybook:

- `Default`
- `WithInitialValues`
- `WithValidationError`
- `Disabled`
- `Submitting`

Jest:

- 필수값 누락 시 에러가 표시되는지
- 잘못된 입력값을 막는지
- submit 시 올바른 payload가 전달되는지
- submit 실패 시 에러 메시지가 표시되는지

---

#### Modal 컴포넌트

Storybook:

- `Open`
- `Confirm`
- `Alert`
- `WithLongContent`
- `Mobile`

Jest:

- ESC 입력 시 닫히는지
- overlay 클릭 시 닫히는지
- confirm 클릭 시 callback이 호출되는지
- focus trap이 유지되는지

---

## AI 에이전트 작업 절차

AI 에이전트는 Story 작성 시 다음 순서로 작업한다.

1. 컴포넌트 props와 렌더링 조건을 확인한다.
2. 컴포넌트가 의존하는 Provider, Hook, API, Next.js 기능을 확인한다.
3. 기존 Storybook 설정과 프로젝트 convention을 확인한다.
4. CSF 3.0 형식으로 기본 Story 구조를 작성한다.
5. 공통 args는 meta.args에 둔다.
6. 개별 Story에서는 차이점만 override한다.
7. argTypes는 자동 추론을 우선하고 필요한 경우만 수동 지정한다.
8. 주요 상태별 Story를 추가한다.
9. 필요한 경우 mock data를 작성한다.
10. 이벤트가 있으면 `fn()` 또는 actions를 추가한다.
11. 사용자 상호작용이 중요하면 `play` 함수를 추가한다.
12. API 의존성이 있으면 MSW 사용을 검토한다.
13. 타입 에러가 없는지 확인한다.
14. Story가 실제 사용 예시로 의미 있는지 검토한다.

---

## AI 에이전트가 임의로 하면 안 되는 것

AI 에이전트는 다음 작업을 임의로 수행하지 않는다.

- 컴포넌트의 public props를 변경하지 않는다.
- Story 작성을 위해 컴포넌트 내부 구현을 변경하지 않는다.
- 실제 API 호출 코드를 Story 안에 넣지 않는다.
- 프로젝트에 없는 Provider를 새로 만들지 않는다.
- 기존 Storybook 설정을 임의로 크게 변경하지 않는다.
- framework를 임의로 변경하지 않는다.
- 불필요하게 MSW, addon, decorator를 추가하지 않는다.
- 타입 에러를 피하기 위해 `any`를 사용하지 않는다.
- 모든 props 조합을 무작정 Story로 만들지 않는다.
- `argTypes`를 불필요하게 수동 지정하지 않는다.
- `title`을 임의로 직접 명시하지 않는다.

컴포넌트 수정이 반드시 필요하다면, 먼저 이유를 설명하고 최소 변경만 제안한다.

---

## Story 작성 체크리스트

Story 작성 후 다음 항목을 확인한다.

- [ ] CSF 3.0 형식을 사용했는가?
- [ ] `Meta`, `StoryObj` 타입을 사용했는가?
- [ ] `satisfies Meta<typeof Component>`를 사용했는가?
- [ ] 특별한 이유 없이 `title`을 직접 명시하지 않았는가?
- [ ] `Default` 또는 기본 Story가 있는가?
- [ ] 공통값은 `meta.args`에 두었는가?
- [ ] 개별 Story에서는 차이점만 override했는가?
- [ ] 기본값을 `argTypes.defaultValue`가 아니라 `args`에 두었는가?
- [ ] 주요 상태가 Story로 분리되어 있는가?
- [ ] 의미 있는 args를 사용했는가?
- [ ] 실제 서비스에서 나올 법한 mock data를 사용했는가?
- [ ] argTypes는 자동 추론을 우선했는가?
- [ ] 수동 argTypes는 필요한 경우에만 작성했는가?
- [ ] 이벤트 핸들러는 `fn()` 또는 actions로 처리했는가?
- [ ] 필요한 경우 `play` 함수가 있는가?
- [ ] 접근 가능한 selector를 사용했는가?
- [ ] API 호출이 필요한 경우 MSW를 사용했는가?
- [ ] Next.js 라우터, 이미지, Provider 의존성을 적절히 처리했는가?
- [ ] 불필요한 decorator나 mock을 추가하지 않았는가?
- [ ] Storybook에서 실제로 렌더링 가능한가?
- [ ] Jest에서 검증해야 할 로직을 Storybook에 몰아넣지 않았는가?

---

## 좋은 Story의 기준

좋은 Story는 다음 질문에 답할 수 있어야 한다.

- 이 컴포넌트는 어떤 상황에서 쓰이는가?
- 어떤 props를 넘기면 어떤 모습이 되는가?
- 로딩, 에러, 빈 데이터 같은 상태는 어떻게 보이는가?
- 사용자가 상호작용하면 어떤 일이 일어나는가?
- 실제 서비스 데이터가 들어와도 UI가 깨지지 않는가?
- 디자이너나 QA가 확인할 수 있을 만큼 상태가 명확한가?

Storybook은 “컴포넌트 전시장”이 아니라 “컴포넌트 사용 설명서”처럼 작성한다.

Storybook은 보이는 상태를 설명하는 문서로 작성한다.

Jest는 행동과 결과를 검증하는 테스트로 작성한다.

둘을 함께 사용할 때는 같은 컴포넌트를 보호하되, 같은 책임을 반복하지 않는다.