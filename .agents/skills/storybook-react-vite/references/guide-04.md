## Storybook과 Vitest 테스트의 역할 분리

Storybook과 Vitest는 모두 UI 품질을 높이기 위한 도구지만, 검증 목적이 다르다.

Storybook은 컴포넌트의 상태, 변형, 사용 예시, 시각적 표현을 문서화하는 데 집중한다.

Vitest와 Testing Library는 비즈니스 로직, 조건 분기, 사용자 행위 결과, 예외 케이스를 검증하는 데 집중한다.

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

### Vitest가 담당하는 것

Vitest와 Testing Library는 다음 항목을 우선 담당한다.

- 조건 분기 로직
- 사용자 입력에 따른 상태 변경
- 이벤트 핸들러 호출 여부
- 유효성 검사
- API 응답에 따른 렌더링 결과
- 권한, 상태값, feature flag 등에 따른 분기
- 예외 상황 처리
- custom hook / util 함수 / domain logic 검증

Vitest의 핵심 질문은 다음과 같다.

> 사용자가 행동했을 때 의도한 결과가 발생하는가?

---

### Storybook play 함수의 위치

Storybook `play` 함수는 Vitest 테스트를 완전히 대체하지 않는다.

`play` 함수는 Story 안에서 보여주는 UI 상태가 실제로 가볍게 상호작용 가능한지 확인하는 용도로 사용한다.

적합한 예:

- 버튼 클릭 시 callback 호출 확인
- 드롭다운 열기
- 탭 전환
- 모달 닫기
- 체크박스 선택
- 간단한 입력 상태 확인

다음 검증은 Storybook `play` 함수에 넣지 않고 Vitest에서 처리한다.

- 복잡한 비즈니스 규칙
- 여러 API 응답 조합
- 권한별 분기
- 날짜 / 금액 / 할인율 계산
- 서버 상태와 캐시 동작
- 실패 재시도 로직
- custom hook 내부 동작
- util 함수의 경계값 검증

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

Vitest:

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

Vitest:

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

Vitest:

- 필수값 누락 시 에러가 표시되는지
- 잘못된 입력값을 막는지
- submit 시 올바른 payload가 전달되는지
- submit 실패 시 에러 메시지가 표시되는지

---


## AI 에이전트 작업 절차

AI 에이전트는 Story 작성 시 다음 순서로 작업한다.

1. 컴포넌트 props와 렌더링 조건을 확인한다.
2. 컴포넌트가 의존하는 Provider, Hook, API, React Router, Vite 기능을 확인한다.
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
- [ ] React Router, asset, env, Provider 의존성을 적절히 처리했는가?
- [ ] 불필요한 decorator나 mock을 추가하지 않았는가?
- [ ] Storybook에서 실제로 렌더링 가능한가?
- [ ] Vitest에서 검증해야 할 로직을 Storybook에 몰아넣지 않았는가?

---


## 좋은 Story의 기준

좋은 Story는 다음 질문에 답할 수 있어야 한다.

- 이 컴포넌트는 어떤 상황에서 쓰이는가?
- 어떤 props를 넘기면 어떤 모습이 되는가?
- 로딩, 에러, 빈 데이터 같은 상태는 어떻게 보이는가?
- 사용자가 상호작용하면 어떤 일이 일어나는가?
- 실제 서비스 데이터가 들어와도 UI가 깨지지 않는가?
- 디자이너나 QA가 확인할 수 있을 만큼 상태가 명확한가?

Storybook은 "컴포넌트 전시장"이 아니라 "컴포넌트 사용 설명서"처럼 작성한다.

Storybook은 보이는 상태를 설명하는 문서로 작성한다.

Vitest는 행동과 결과를 검증하는 테스트로 작성한다.

둘을 함께 사용할 때는 같은 컴포넌트를 보호하되, 같은 책임을 반복하지 않는다.
