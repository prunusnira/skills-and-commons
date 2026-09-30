## 테스트 작성이 어려운 경우

테스트 작성이 어렵다면 무리하게 Next.js API mock을 늘리지 않는다.

먼저 다음을 설명한다.

1. 테스트가 어려운 이유
2. Server Component, Server Action, Route Handler 중 어떤 경계에서 문제가 발생하는지
3. 어떤 Next.js API 또는 외부 의존성 때문에 문제가 생기는지
4. Vitest 단위 테스트로 검증 가능한 범위
5. integration 또는 E2E 테스트가 더 적절한 범위
6. production code를 개선한다면 어떤 구조가 좋은지

비동기 Server Component, streaming, hydration, middleware, caching처럼 Next.js 런타임 의존성이 큰 기능은 E2E 테스트를 우선 검토한다.

단, production code 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

---


## 완료 전 체크리스트

작업을 마치기 전에 다음을 확인한다.

* [ ] Given-When-Then 구조를 따른다
* [ ] 테스트 이름이 동작을 설명한다
* [ ] App Router와 Pages Router를 구분했다
* [ ] Server Component와 Client Component를 구분했다
* [ ] 비동기 Server Component를 무리하게 Vitest로 렌더링하지 않았다
* [ ] 사용자 행동은 `userEvent`를 사용한다
* [ ] query는 접근 가능한 selector를 우선한다
* [ ] 비동기 처리는 `findBy*` 또는 `waitFor`를 사용한다
* [ ] mock은 외부 경계에만 제한적으로 사용한다
* [ ] `next/navigation`, `next/headers`, `next/cache` mock은 필요한 API만 포함한다
* [ ] `next/link`는 가능하면 실제 `href`를 검증한다
* [ ] `next/image` 내부 구현에 assertion이 결합되지 않았다
* [ ] 환경변수는 `process.env` 기준으로 처리한다
* [ ] Server Action의 인증과 권한 검사를 고려했다
* [ ] Route Handler의 status와 response body를 검증했다
* [ ] 중요한 에러 케이스를 포함한다
* [ ] 테스트 간 상태가 격리되어 있다
* [ ] `.only`, `.skip`, 임의 delay가 없다
* [ ] TypeScript 타입을 보존한다
* [ ] 테스트를 읽는 사람이 과도한 mock setup 없이 이해할 수 있다
* [ ] Vitest보다 E2E가 적합한 범위를 구분했다
* [ ] 실제 동작이 깨지면 테스트도 실패한다

---


## AI 에이전트의 최종 응답 형식

AI 에이전트가 테스트를 작성하거나 수정한 뒤에는 다음 내용을 요약한다.

1. 어떤 동작을 테스트했는지
2. 테스트 대상이 Client Component, Server Component, Server Action, Route Handler 중 무엇인지
3. 어떤 mock을 추가했고 왜 필요한지
4. 어떤 edge case를 포함했는지
5. Vitest로 검증하기 어려워 E2E가 필요한 영역이 있는지
6. 실행한 Vitest, type-check, lint 또는 build 결과
