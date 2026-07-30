---
description: 승인된 E2E 계약과 실행 증거로 Playwright spec을 생성하고 실행·자가 치유·안정화한다. 사용법: /e2e-gen {domain}/{name}
---

대상: `$ARGUMENTS`

이 명령은 `.claude/e2e/WORKFLOW.md`를 먼저 읽고 따른다.

## 1. 사전 조건

1. 대상은 정확히 `{domain}/{name}` 형식이어야 하며 `..`, 절대 경로, 빈 segment를 허용하지 않는다.
   - `SPEC_PATH = [project root]/e2e/{domain}/{name}.spec.ts`
   - `SPEC_TARGET = e2e/{domain}/{name}.spec.ts`
   - fixture 기본 경로는 `[project root]/e2e/fixtures/{domain}/{state}.json`
2. `[project root]/e2e/.session/current`에서 현재 세션을 찾는다.
3. 해당 세션의 `scenario.md`와 `execution.md`를 읽는다.
4. 다음 중 하나라도 해당하면 생성하지 않는다.
   - Planner 상태가 `READY`가 아님
   - 사용자 승인 기록이 없음
   - oracle 또는 필수 precondition이 없음
   - Executor가 실행하지 못한 케이스인데 미실행 이유가 정리되지 않음

대상 파일이 이미 존재하면 내용을 읽고 사용자 변경을 보존한다. 전면 덮어쓰지 말고 케이스 단위로 추가·수정한다.
기존 flat/legacy fixture를 편의상 복사하지 않는다. 승인된 case가 요구하는 응답 상태만 현재 소비 schema로 생성한다.

## 2. Generator

`e2e-generator` 에이전트에 전달한다.

- `TARGET = $ARGUMENTS`
- `SCENARIO_PATH = <현재 세션>/scenario.md`
- `EXECUTION_PATH = <현재 세션>/execution.md`
- `GENERATION_PATH = <현재 세션>/generation.md`
- `CODEGEN_PATH = <현재 세션>/codegen.spec.ts` (있을 때만)

Generator는 대상 spec을 만들고 대상 파일에 한정해 `--list` 검증을 수행한다.

## 3. 실제 실행

Generator가 성공하면 아래 조건으로 대상 spec을 실행한다.

```bash
pnpm -C [project root] exec playwright test <SPEC_TARGET> --project=chromium --retries=0
```

- 통과하면 안정성 검증으로 이동한다.
- 실패하면 `e2e-healer`를 호출한다.
- 컴파일/수집 실패도 Healer 입력으로 전달하되 브라우저 실패와 구분한다.

## 4. 제한된 자가 치유

`e2e-healer`에 대상 spec, 현재 세션 경로, 실패 산출물을 전달한다.

Healer는 최대 3회의 진단→최소 수정→재실행만 수행한다. `PRODUCT_REGRESSION`, `UNKNOWN`, 기대 결과 변경 필요, 동일 원인 반복이면 즉시 중단한다.

## 5. 안정화

단일 실행 통과 후:

```bash
pnpm -C [project root] exec playwright test <SPEC_TARGET> \
  --project=chromium --retries=0 --repeat-each=3
```

`scenario.md`의 project가 `both`면 `mobile-webkit`도 1회 실행한다.

## 6. 결과

다음을 보고한다.

- 생성/수정된 파일과 test case 수
- 생성/재사용한 domain fixture와 각 응답 상태
- 단일 실행 결과
- healing 횟수와 각 원인의 증거
- 3회 안정성 실행 결과
- mobile 결과(해당 시)
- 남은 `PRODUCT_REGRESSION`, `ENVIRONMENT`, `UNKNOWN`

수집만 성공하고 실제 실행하지 않은 코드는 완료로 보고하지 않는다.
