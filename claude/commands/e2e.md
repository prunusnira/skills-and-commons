---
description: 자연어 요구사항을 검증 가능한 E2E 계약으로 만들고 실제 브라우저에서 탐색한다. 사용법: /e2e [--record] {시나리오}
---

입력: `$ARGUMENTS`

이 명령은 `.claude/e2e/WORKFLOW.md`를 먼저 읽고 따른다.

목표는 “그럴듯한 동작 목록”이 아니라 Generator가 그대로 코드화할 수 있는 **승인된 테스트 계약과 실행 증거**를 만드는 것이다.

## 1. 입력과 실행 전제 확인

1. `$ARGUMENTS`가 비어 있으면 만들고 싶은 기능/행위와 기대 결과를 사용자에게 묻고 중단한다.
2. `--record` 유무를 분리하고 나머지를 `SCENARIO`로 사용한다.
3. `BASE_URL`은 `/tmp/tunnel.url`의 유효한 값이 있으면 사용하고, 아니면 `http://localhost:3000`을 사용한다.
4. HTTP 200~399가 아니면 `[project root]`에서 `pnpm dev`을 기동하고 PID와 로그 경로를 기록한다. 최대 120초 동안 준비를 확인한다.
5. 세션을 만든다.

```bash
SESSION_ID=$(date +%Y%m%d-%H%M%S)
SESSION_DIR="[project root]/e2e/.session/$SESSION_ID"
mkdir -p "$SESSION_DIR"
printf '%s\n' "$SESSION_DIR" > [project root]/e2e/.session/current
```

`scenario.md`, `execution.md`, `generation.md`, `healing.md`를 빈 파일로 만든다.

## 2. Planner

`e2e-planner` 에이전트에 다음을 전달한다.

- `SCENARIO`
- `BASE_URL`
- `SCENARIO_PATH = $SESSION_DIR/scenario.md`
- `RECORD_REQUESTED = true|false`

Planner 반환값에 따라 처리한다.

- `NEEDS_INPUT`: Planner의 질문만 사용자에게 묻는다. 답을 받아 같은 세션의 Planner를 다시 실행한다.
- `BLOCKED`: 누락된 전제를 보고하고 중단한다.
- `RECORD_RECOMMENDED`: 이유와 정확한 codegen 명령을 보여준다. `--record`가 있으면 즉시 codegen을 열고, 없으면 사용자가 기록할지 묻는다.
- `READY`: 케이스, domain, 필요한 응답 상태별 fixture, mock, oracle, 불확실성 요약을 사용자에게 보여주고 승인받는다.

사용자 승인 전 Executor를 실행하지 않는다. `uncertainties`가 남아 있거나 oracle이 없는 계획은 승인 가능한 것으로 취급하지 않는다.
승인받으면 `scenario.md`의 `approved: false`를 `approved: true`로 바꾸고 승인 시각을 `approved_at`으로 기록한다. Planner를 다시 실행할 때는 승인 상태를 다시 `false`로 되돌려 변경된 계획을 재승인받는다.

실제 계정이 필요하지만 `[project root]/e2e/.auth/user.json`이 없으면 비밀번호를 묻지 않는다. Planner는 `RECORD_RECOMMENDED`를 반환하고 `.claude/e2e/WORKFLOW.md`의 `--save-storage` 명령으로 사용자가 직접 로그인하도록 안내한다. 인증 상태가 생성된 뒤 Planner를 다시 실행하고 계획 승인을 받는다.

## 3. 선택적 codegen

codegen은 `.claude/e2e/WORKFLOW.md`의 사용 기준에 해당할 때만 사용한다. 출력은 `$SESSION_DIR/codegen.spec.ts`에 저장한다.

기록이 끝나면 Planner를 다시 실행해 codegen에서 확인된 조작 경로/locator만 `scenario.md`에 반영한다. 비즈니스 기대 결과는 codegen에서 추론하지 않는다.

로그인 상태만 기록하는 경우 codegen spec 출력은 만들지 않아도 된다. `e2e/.auth/user.json`의 존재만 확인하고 내용을 읽거나 세션 문서에 복사하지 않는다.

## 4. Executor

승인 후 `e2e-runner` 에이전트에 다음을 전달한다.

- `BASE_URL`
- `SCENARIO_PATH = $SESSION_DIR/scenario.md`
- `EXECUTION_PATH = $SESSION_DIR/execution.md`

Executor는 계획의 mock과 인증 조건을 먼저 적용한 뒤 각 케이스를 실행한다. mock은 URL과 HTTP method를 함께 확인하고 BFF 최종 응답 schema를 사용한다. 실제 locator, URL, 주요 network request, assertion 결과, 실패 screenshot을 기록한다.

## 5. 결과

- `PASS`: 실행된 케이스와 oracle을 요약하고 `/e2e-gen {domain}/{name}`을 안내한다.
- `FAIL_TESTABILITY`: locator/mock/환경 문제와 증거를 보고한다. 계획을 고칠 수 있으면 Planner→Executor를 한 번 더 수행한다.
- `FAIL_PRODUCT`: 기대 결과와 실제 결과를 함께 보고하고 자동으로 기대 결과를 바꾸지 않는다.
- `BLOCKED`: 사용자에게 필요한 정보나 조치를 구체적으로 요청한다.

이 명령에서 기동한 서버만 종료한다. 기존 서버는 종료하지 않는다.
