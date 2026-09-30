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
