## 컴포넌트 테스트 기본 형태

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('form이 유효하면 이메일과 비밀번호를 제출한다', async () => {
    // Given
    const user = userEvent.setup();
    const onSubmit = vi.fn<(values: LoginValues) => Promise<void>>();

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


## 순수 함수 테스트 기본 형태

```ts
import { describe, expect, it } from 'vitest';
import { calculateDiscountRate } from './calculateDiscountRate';

describe('calculateDiscountRate', () => {
  it('정가가 판매가보다 크면 할인율을 반환한다', () => {
    // Given
    const tagPrice = 10000;
    const realPrice = 8000;

    // When
    const result = calculateDiscountRate(tagPrice, realPrice);

    // Then
    expect(result).toBe(20);
  });
});
```

---


## 에러 케이스 규칙

중요한 실패 케이스를 반드시 고려한다.

예시:

* 빈 입력값
* 잘못된 입력값
* API 실패
* 로딩 상태
* 권한 없음
* optional 데이터 누락
* 경계값
* 품절 또는 비활성화 상태
* 로그인하지 않은 상태

happy path만 테스트하지 않는다.

---


## Timer와 Date 규칙

날짜나 timer를 테스트할 때는 실제 현재 시간에 의존하지 않는다.

필요하면 fake timer를 사용하고, 테스트 이후 반드시 원복한다.

```ts
beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});
```

`userEvent`와 fake timer를 함께 사용할 때는 timer advance 설정이 필요한지 확인한다.

```ts
const user = userEvent.setup({
  advanceTimers: vi.advanceTimersByTime,
});
```

---


## 테스트 격리 규칙

각 테스트는 서로 독립적이어야 한다.

다음 상태가 테스트 사이에 새면 안 된다.

* mock call history
* mock implementation
* fake timer
* stubbed env
* stubbed global
* localStorage
* sessionStorage
* 공유 mutable state

필요하면 다음처럼 정리한다.

```ts
afterEach(() => {
  vi.clearAllMocks();
  vi.unstubAllEnvs();
  vi.unstubAllGlobals();
  localStorage.clear();
  sessionStorage.clear();
});
```

mock 구현까지 초기화해야 하는 경우에만 `vi.resetAllMocks()`를 사용한다.

---


## Snapshot 규칙

큰 snapshot은 만들지 않는다.

snapshot은 작고 안정적인 결과물을 의도적으로 검증할 때만 사용한다.

나쁜 예:

```ts
expect(container).toMatchSnapshot();
```

좋은 예:

```ts
expect(screen.getByRole('heading', { name: '주문 완료' })).toBeInTheDocument();
expect(screen.getByText('결제가 정상적으로 완료되었습니다.')).toBeInTheDocument();
```

---


## AI 에이전트가 테스트 작성 전에 확인해야 할 것

테스트를 작성하거나 수정하기 전에 반드시 다음을 확인한다.

1. 테스트 대상 컴포넌트 또는 함수
2. public input과 output
3. 사용자에게 보이는 동작
4. 외부 의존성
5. 주변 테스트 파일의 스타일
6. 프로젝트의 Vitest 설정
7. DOM test environment 설정
8. custom render utility 존재 여부
9. 이미 정의된 mock 또는 setup 파일 존재 여부

기존 테스트 유틸이 있다면 새로 만들지 말고 재사용한다.

---


## AI 에이전트가 하면 안 되는 것

다음 행동은 하지 않는다.

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

테스트 작성이 어렵다면 무리하게 mock을 늘리지 않는다.

먼저 다음을 설명한다.

1. 테스트가 어려운 이유
2. 어떤 의존성 때문에 문제가 생기는지
3. Vitest unit test로 검증 가능한 범위
4. E2E 테스트가 더 적절한 범위
5. production code를 개선한다면 어떤 구조가 좋은지

단, production code 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

---


## 완료 전 체크리스트

작업을 마치기 전에 다음을 확인한다.

* [ ] Given-When-Then 구조를 따른다
* [ ] 테스트 이름이 동작을 설명한다
* [ ] 사용자 행동은 `userEvent`를 사용한다
* [ ] query는 접근 가능한 selector를 우선한다
* [ ] 비동기 처리는 `findBy*` 또는 `waitFor`를 사용한다
* [ ] mock은 외부 경계에만 제한적으로 사용한다
* [ ] `vi.fn`, `vi.mock`, `vi.useFakeTimers` 등 Vitest API를 사용한다
* [ ] 중요한 에러 케이스를 포함한다
* [ ] 테스트 간 상태가 격리되어 있다
* [ ] `.only`, `.skip`, 임의 delay가 없다
* [ ] TypeScript 타입을 보존한다
* [ ] 테스트를 읽는 사람이 과도한 mock setup 없이 이해할 수 있다
* [ ] 실제 동작이 깨지면 테스트도 실패한다

---


## AI 에이전트의 최종 응답 형식

AI 에이전트가 테스트를 작성하거나 수정한 뒤에는 다음 내용을 요약한다.

1. 어떤 동작을 테스트했는지
2. 어떤 mock을 추가했고 왜 필요한지
3. 어떤 edge case를 포함했는지
4. Vitest로 검증하기 어려워 E2E가 필요한 영역이 있는지
