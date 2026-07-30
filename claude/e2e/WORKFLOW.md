# Playwright E2E workflow contract

`/e2e`, `/e2e-gen`, `/e2e-heal`과 `e2e-*` 에이전트가 공유하는 계약이다.

## 역할

- **Planner**: 요구사항을 검증 가능한 시나리오로 정규화한다. 비즈니스 기대 결과를 만들지 않는다.
- **Executor**: 브라우저와 네트워크에서 계획의 실행 가능성을 탐색하고 증거를 남긴다. 테스트 코드를 수정하지 않는다.
- **Generator**: 승인된 계획과 실행 증거를 Playwright Test 코드로 변환한다.
- **Healer**: 재현된 실패의 원인을 분류하고 테스트 결함만 최소 수정한다. 제품 결함을 테스트 변경으로 숨기지 않는다.

## 세션 파일

모든 명령은 `[project root]/e2e/.session/current`가 가리키는 세션 디렉터리를 사용한다.

```text
[project root]/e2e/.session/<SESSION_ID>/
├── scenario.md       # Planner의 승인 가능한 테스트 계약
├── execution.md      # Executor의 액션·locator·network·assertion 증거
├── generation.md     # 생성 파일과 검증 결과
└── healing.md        # 실패 분류, 수정, 재실행 이력
```

세션 파일은 append-only 이력으로 사용한다. 원래 기대 결과와 실패 증거를 덮어쓰지 않는다.

spec 경로는 두 형태를 구분한다.

- `SPEC_PATH`: 저장소 루트 기준 경로. 예: `[project root]/e2e/login/smoke.spec.ts`
- `SPEC_TARGET`: `pnpm -C [project root]` 실행 기준 경로. 예: `e2e/login/smoke.spec.ts`

## 디렉터리와 fixture 소유권

spec은 사용자 기능 도메인 아래에 둔다. API mock 데이터는 시나리오 이름이 아니라 **도메인과 응답 상태**로 식별한다.

```text
[project root]/e2e/
├── <domain>/
│   ├── <scenario>.spec.ts
│   └── support/                 # 해당 도메인에서만 공유하는 helper
├── fixtures/
│   └── <domain>/
│       ├── <state>.json         # 크고 고정된 API 응답
│       └── <resource>.ts        # override가 필요한 builder/factory
└── support/                     # 둘 이상의 도메인이 실제로 공유할 때만 사용
```

예:

```text
e2e/
├── auth/login-from-main.spec.ts
└── fixtures/auth/login-success.json
```

fixture와 scenario는 1:1 관계가 아니다.

- 하나의 scenario가 여러 API fixture를 조합할 수 있다.
- 여러 scenario가 같은 응답 상태 fixture를 재사용할 수 있다.
- 파일명은 `getCouponList.json` 같은 호출명보다 `coupon/empty.json`, `coupon/available.json`처럼 의미 있는 상태를 사용한다.

데이터 표현 기준:

- 크고 안정적인 응답 스냅샷: `fixtures/<domain>/<state>.json`
- 일부 필드만 달라지는 응답: TypeScript builder/factory
- 한 scenario에서만 사용하는 작은 응답: spec 내부 inline object
- 실제 사용자·토큰·주문·주소 등 민감 데이터: 복사 금지. 명백한 테스트 값으로 작성

fixture를 추가하거나 재사용할 때 API path만 보고 선택하지 않는다. 해당 scenario가 필요로 하는 응답 상태와 oracle을 먼저 확인하고, 소비 코드의 최종 응답 schema와 대조한다. BFF를 intercept하면 외부 백엔드 응답이 아니라 **브라우저가 실제로 받는 BFF 최종 응답**을 사용한다.

사용하지 않는 legacy fixture를 보존하지 않는다. 후속 scenario에 필요해질 때 현재 schema를 다시 확인하고 추가한다.

## 필수 시나리오 계약

각 테스트 케이스에는 다음 항목이 모두 있어야 한다.

- `case_id`: 세션 내 고유 ID
- `title`: 사용자 관점의 행위
- `project`: `chromium`, `mobile-webkit`, `both` 중 하나
- `preconditions`: 인증 상태, URL, viewport, 필요한 데이터
- `mocks`: 요청 method/pattern, 응답 상태 이름, status/fixture, mock 이유. 없으면 `none`
- `steps`: 사용자 행위만 포함한 Given/When 순서
- `oracles`: 관찰 가능한 최종 결과. URL, visible text/role, 값, 요청 등
- `cleanup`: 생성 데이터 정리 또는 `none`
- `source`: 사용자 입력 또는 코드 근거
- `uncertainties`: 미확정 사항 또는 `none`

## Planner 결정

Planner는 반드시 다음 중 하나만 반환한다.

- `READY`: 실행 가능한 시나리오와 oracle이 모두 확정됨
- `NEEDS_INPUT`: 비즈니스 의미를 사용자가 결정해야 함
- `RECORD_RECOMMENDED`: UI 조작 경로/locator 탐색이 어려워 codegen 기록이 유용함
- `BLOCKED`: 서버, 인증, 테스트 데이터 등 실행 전제 부족

다음은 코드에서 추론하지 말고 `NEEDS_INPUT`으로 반환한다.

- 성공/실패의 정확한 기대 결과
- 어떤 API 상태를 테스트해야 하는지
- 결제, 주문, 회원 정보 변경처럼 실제 데이터에 영향을 주는 동작의 허용 범위
- 사용할 계정·상품·쿠폰 등 테스트 데이터

질문은 한 번에 최대 3개, 선택에 따라 테스트가 달라지는 것만 묻는다.

## codegen 사용 기준

codegen은 시나리오나 oracle을 만드는 도구가 아니다. 아래 상황에서만 보조 입력으로 사용한다.

- canvas, drag-and-drop, 복잡한 iframe, 파일 업로드 등 MCP snapshot만으로 조작 경로가 불명확함
- 접근 가능한 role/name이 없고 고유 locator를 사람이 확인해야 함
- 로그인 상태를 사람이 직접 만든 뒤 기록해야 함
- 사용자가 명시적으로 `--record`를 요청함

실행 예:

```bash
pnpm -C [project root] exec playwright codegen \
  --target=playwright-test \
  --output=e2e/.session/<SESSION_ID>/codegen.spec.ts \
  "$BASE_URL"
```

로그인 상태 저장 파일에는 민감한 쿠키가 포함될 수 있으므로 `e2e/.auth/` 아래에만 저장하고 커밋하지 않는다.

실제 계정을 사용하는 시나리오는 ID/비밀번호를 spec, fixture, 세션 문서, 명령 인자에 넣지 않는다. 사람이 로그인한 브라우저 상태를 다음처럼 별도 저장한다.

```bash
pnpm -C [project root] exec playwright codegen \
  --save-storage=e2e/.auth/user.json \
  "$BASE_URL/user/login"
```

사용자가 브라우저에서 직접 로그인한 뒤 codegen 창을 닫으면 이후 Executor와 spec은 `e2e/.auth/user.json`을 로드한다. 저장 파일의 cookie/token 값은 읽거나 로그에 출력하지 않는다. 인증 상태 파일이 없거나 만료됐으면 실행 결과를 `BLOCKED`로 분류하고 재기록을 요청한다.

실제 서버의 변경 API를 호출하는 시나리오는 초기 상태를 먼저 관찰하고, 검증 후 가능한 경우 원래 상태로 복원한다. 복원 요청까지 oracle로 검증하며, 복원에 실패하면 테스트가 통과한 것으로 보고하지 않는다.

codegen 출력은 초안이다. Generator가 사용자 관점 locator, fixture, 격리, oracle을 다시 검증한 뒤 정식 spec으로 옮긴다.

## 생성 코드 품질 기준

- 각 테스트는 독립 실행 가능해야 하며 다른 테스트의 실행 순서나 상태에 의존하지 않는다.
- 사용자에게 보이는 행위와 결과를 검증한다.
- locator 우선순위는 `getByRole`/`getByLabel`/`getByPlaceholder`/`getByText` → `getByTestId` → 기존 `data-cy`다.
- CSS class, XPath, DOM 깊이, 무근거 `nth()`는 사용하지 않는다.
- Playwright locator와 web-first assertion을 사용한다.
- `waitForTimeout`, 임의 sleep, 조건 없는 retry loop를 사용하지 않는다.
- `force: true`, timeout 증가는 원인을 증명한 경우가 아니면 사용하지 않는다.
- mock은 테스트가 소유한 API에만 적용하고, 요청 method와 fixture schema를 앱 코드에서 검증한다.
- URL만 일치하는 다른 HTTP method를 가로채지 않는다. 예상 method가 아니면 `route.fallback()` 또는 `route.continue()`한다.
- fixture 경로는 `fixtures/<domain>/<state>.json`을 기본으로 하고, scenario명과 API 함수명만으로 이름 짓지 않는다.
- fixture는 브라우저가 받는 최종 응답 schema여야 한다. BFF 내부 upstream 응답을 그대로 반환하지 않는다.
- fixture와 mock helper는 실제 spec에서 사용되는 것만 생성한다.
- 성공 경로에는 최소 1개의 핵심 결과 oracle이 있어야 한다. 단순히 “페이지가 열림”만 확인하지 않는다.
- 민감한 값과 실제 사용자 계정을 spec, fixture, trace, 로그에 기록하지 않는다.

## 검증과 안정화

로컬 Next 개발 서버의 최초 route compile을 고려해 Playwright의 test, expect, action, navigation 기본 제한은 60초로 사용한다. `waitForTimeout`을 추가하거나 실패를 무조건 재시도하는 근거로 사용하지 않으며, 60초를 초과하면 기존과 동일하게 원인을 분류한다.

생성 또는 수정 후 아래 순서로 검증한다.

1. 대상 파일 수집:
   `pnpm -C [project root] exec playwright test <SPEC_TARGET> --list`
2. 대표 프로젝트 단일 실행:
   `pnpm -C [project root] exec playwright test <SPEC_TARGET> --project=chromium --retries=0`
3. 안정성 실행:
   `pnpm -C [project root] exec playwright test <SPEC_TARGET> --project=chromium --retries=0 --repeat-each=3`
4. `project: both`면 `mobile-webkit`도 1회 실행한다.

재시도 없이 안정화하는 이유는 flaky failure를 retry로 가리지 않기 위해서다.

## Healing 정책

실패 분류:

- `TEST_LOCATOR`
- `TEST_WAIT_OR_ASSERTION`
- `TEST_MOCK_OR_DATA`
- `ENVIRONMENT`
- `PRODUCT_REGRESSION`
- `FLAKY_UNPROVEN`
- `UNKNOWN`

자동 수정 허용 범위:

- 대상 spec
- 대상 domain의 `support/**`
- `[project root]/e2e/support/**` 중 대상 spec이 실제 사용하는 공통 helper
- 해당 domain의 fixture/builder

앱 코드는 자동 수정하지 않는다. `PRODUCT_REGRESSION`, `UNKNOWN`, 기대 결과 변경이 필요한 경우 중단하고 근거를 보고한다.

Healer는 한 호출에서 최대 3회만 수정·재실행한다. 같은 원인이 두 번 반복되거나 새 증거가 없으면 조기 중단한다. 통과 후에도 안정성 실행을 통과해야 `healed`다.

금지된 healing:

- assertion 삭제·완화, 기대 문자열 변경, 테스트 skip/fixme 처리
- locator를 근거 없이 `.first()`, `.last()`, `.nth()`로 변경
- `waitForTimeout`, 무조건적 reload/retry 추가
- timeout을 늘려서만 통과시키기
- 제품 결함을 fixture나 mock으로 우회하기
