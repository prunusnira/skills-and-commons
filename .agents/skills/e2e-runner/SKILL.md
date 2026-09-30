---
name: e2e-runner
description: "승인된 E2E 계약을 브라우저 자동화로 실행하고 locator, network, assertion 근거를 기록한다."
---

먼저 `.agents/skills/e2e/references/WORKFLOW.md`를 읽고 모든 계약을 따른다.

## 입력

- `BASE_URL`
- `SCENARIO_PATH`
- `EXECUTION_PATH`

## 실행 전 게이트

- `scenario.md`가 `READY`, `approved: true`가 아니면 실행하지 않는다.
- oracle이 없는 case는 실행하지 않는다.
- mock fixture를 읽고 domain/state, JSON, status, request method/pattern, BFF 최종 응답 schema를 확인한다.
- 실제 계정 case가 지정한 auth state 파일이 없으면 `BLOCKED`로 중단한다. 파일의 cookie/token 값은 읽거나 출력하지 않는다.

## 실행

각 case를 독립된 상태로 실행한다.

1. 실제 계정 case는 navigate 전에 브라우저 저장 상태 복원 기능로 계획에 지정된 auth state를 복원한다. ID/비밀번호를 입력하거나 auth state 내용을 로그에 남기지 않는다.
2. navigate 전에 case의 mock을 적용한다. method 조건이 있으면 Playwright 브라우저 자동화 API의 `page.route()`에서 `route.request().method()`를 검사하고, 불일치 요청은 `route.fallback()`한다.
3. navigate 후 snapshot과 network request를 확인한다.
4. 각 액션 직전 locator가 고유하고 보이는지 확인한다.
5. 계획 locator가 틀리면 snapshot에서 후보를 찾되 사용자 관점 locator만 채택한다. 근거 없는 `nth`는 사용하지 않는다.
6. 각 액션 후 URL·snapshot·관련 network를 확인하고 oracle을 검증한다.
7. 실패 시 screenshot을 남기고 즉시 해당 case를 중단한다. 같은 액션의 무조건 재시도는 하지 않는다.

## `EXECUTION_PATH` 기록

case별로 append한다.

```md
## Case <case_id>
- result: PASS|FAIL_TESTABILITY|FAIL_PRODUCT|BLOCKED
- preconditions_applied:
- mocks_applied:
- action: <행위>
  planned_locator:
  actual_locator:
  result:
- network:
- oracle:
  expected:
  actual:
  result:
- last_url:
- screenshot:
- evidence:
```

민감한 입력값, cookie, token은 `[REDACTED]`로 기록한다.

## 반환

```text
result: PASS|FAIL_TESTABILITY|FAIL_PRODUCT|BLOCKED
cases_passed: <수>
cases_failed: <수>
execution: <EXECUTION_PATH>
failed_case: <없으면 none>
evidence: <핵심 증거 한 줄>
```
