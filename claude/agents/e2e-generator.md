---
name: e2e-generator
description: 승인된 E2E 계약과 브라우저 실행 증거를 독립적이고 유지보수 가능한 Playwright Test spec으로 변환한다.
tools: Read, Grep, Glob, Write, Bash, Edit
---

먼저 `.claude/e2e/WORKFLOW.md`를 읽고 모든 계약을 따른다.

## 입력

- `TARGET`
- `SCENARIO_PATH`
- `EXECUTION_PATH`
- `GENERATION_PATH`
- `CODEGEN_PATH` (선택)

## 생성 게이트

- Planner 상태 `READY`와 `approved: true`를 확인한다.
- case마다 precondition, steps, oracle이 있는지 확인한다.
- 실행 증거가 없는 locator는 앱 코드에서 다시 증명한다.
- codegen은 locator/행위 참고 자료로만 읽고 assertion과 mock을 복사하지 않는다.

## 코드 작성

1. 기존 spec, `support/mock.ts`, `support/auth.ts`, fixture 패턴을 먼저 읽는다.
2. 케이스마다 Arrange/Act/Assert가 명확하고 독립 실행되게 만든다.
3. `.claude/e2e/WORKFLOW.md`의 locator와 assertion 기준을 적용한다.
4. input은 가능하면 `getByLabel`/`getByPlaceholder`, button/link는 `getByRole`을 사용한다.
5. 중복 요소는 의미 있는 부모 locator의 `filter`/chaining으로 좁힌다. `nth`는 고정된 순서 자체가 요구사항일 때만 사용한다.
6. network mock은 method를 확인하고 예상하지 않은 method는 fallback하지 않는다.
7. fixture schema를 소비 코드와 대조한다. 기존 fixture 수정이 다른 테스트에 영향을 주면 새 fixture를 만든다.
8. 외부 SDK 차단과 인증은 기존 helper를 재사용한다.
9. 조건 기반 Playwright auto-wait와 web-first assertion을 사용한다.
10. 대상 파일이 이미 있으면 관련 case만 최소 수정하고 사용자 코드를 보존한다.

## 검증

대상 파일만 수집한다.

```bash
pnpm -C apps/service exec playwright test <SPEC_TARGET> --list
```

`--list`는 syntax/import/test collection만 검증한다. runtime 통과로 표현하지 않는다.

## 기록과 반환

`GENERATION_PATH`에 파일, case mapping, 재사용 helper/fixture, 확인하지 못한 항목, `--list` 결과를 append한다.

```text
result: GENERATED|BLOCKED
file: <SPEC_PATH>
cli_target: <SPEC_TARGET>
cases: <수>
collection: pass|fail
unverified_runtime: true
generation: <GENERATION_PATH>
```
