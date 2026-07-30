---
description: 실패한 Playwright spec을 증거 기반으로 진단하고 허용 범위 안에서 최대 3회 자가 치유한다. 사용법: /e2e-heal [스펙 경로|도메인]
---

대상: `${ARGUMENTS:-<직전 실패 spec>}`

이 명령은 `.claude/e2e/WORKFLOW.md`를 먼저 읽고 따른다.

## 1. 대상 확정

- 인수가 있으면 `[project root]/e2e/` 아래의 정확한 spec 또는 domain으로 해석한다.
- 인수가 없으면 현재 세션의 `generation.md`와 `[project root]/test-results/`에서 직전 실패 spec을 찾는다.
- 둘 이상의 후보가 있고 직전 실행과 연결할 근거가 없으면 사용자에게 대상을 묻는다.
- spec이 `[project root]/e2e/` 밖이면 자동 수정하지 않는다.
- 파일 수정에는 저장소 루트 기준 `SPEC_PATH`를, `pnpm -C [project root]` 명령에는 `e2e/`로 시작하는 `SPEC_TARGET`을 사용한다.
- fixture 수정은 대상 spec의 domain fixture/builder로 제한한다. 다른 domain이 공유하는 fixture를 즉시 변경하지 않는다.

## 2. 재현

현재 세션이 있으면 `scenario.md`의 원래 oracle과 precondition을 함께 읽는다. 다음 명령으로 retry 없이 실패를 재현한다.

```bash
pnpm -C [project root] exec playwright test <SPEC_TARGET> \
  --project=chromium --retries=0 --trace=on --screenshot=only-on-failure
```

실패하지 않으면 같은 조건으로 `--repeat-each=3`을 실행해 flaky 여부를 확인한다.

## 3. Healer

`e2e-healer`에 전달한다.

- `SPEC_PATH`
- `SPEC_TARGET`
- `SCENARIO_PATH` (있을 때)
- `HEALING_PATH = <현재 세션>/healing.md` (현재 세션이 없으면 새 진단 파일)
- `MAX_ATTEMPTS = 3`

Healer는 trace, error-context, screenshot, stdout, network와 앱 코드를 대조한다. 매 시도마다 원인·증거·수정·재실행 결과를 append한다.

## 4. 종료 조건

- `healed`: 단일 재실행과 `--repeat-each=3`이 모두 통과
- `product_regression`: 앱 동작이 승인된 oracle과 다름. spec을 수정하지 않음
- `environment_blocked`: 서버/인증/테스트 데이터/외부 시스템 문제
- `needs_decision`: 기대 결과나 허용 수정 범위를 사용자가 결정해야 함
- `unhealed`: 3회 소진, 동일 실패 반복, 새 증거 없음

최종 보고에는 원래 assertion, 실제 결과, 증거 파일, 변경 파일, 시도별 결과를 포함한다.
