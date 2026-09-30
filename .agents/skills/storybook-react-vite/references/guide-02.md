## ArgTypes 작성 규칙

### 1. 자동 추론을 우선한다

Storybook은 컴포넌트의 TypeScript 타입을 기반으로 argTypes를 자동 추론한다.

타당한 이유 없이 `argTypes`를 직접 지정하지 않는다.

```tsx
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
// React Router Provider
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

복잡한 비즈니스 로직 검증은 Storybook `play`가 아니라 Vitest 테스트에서 처리한다.

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
// 우선 사용하지 않는다
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
