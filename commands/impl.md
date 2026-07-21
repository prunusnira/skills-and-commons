---
description: HTML draft를 분석해서 실제 코드를 구현합니다. 사용법: /impl PTUN-1234
---

> ### 적용하기 전에
> 본 내용은 프로젝트 설정마다, 개인 환경마다 상이한 내용을 담고 있으므로
> 실제로 적용하기에 앞서 자신의 현재 프로젝트 환경에 맞춰
> 내용 업데이트를 AI에게 한 번 요청해야 합니다

티켓 ID: $ARGUMENTS

---

## 작업 디렉터리 원칙

- 별도의 Git worktree를 생성하거나 다른 worktree로 전환하지 않는다.
- 사용자가 현재 열어 둔 작업 디렉터리에서 직접 작업하여 로컬 변경사항을 바로 확인할 수 있게 한다.
- 현재 작업 디렉터리에 있는 기존 로컬 변경사항은 유지하며, 요청과 무관한 변경을 수정하거나 되돌리지 않는다.

---

## Step 1. Draft HTML 읽기

아래 경로에서 draft HTML 파일을 찾아 읽는다.

- `.claude/draft/$ARGUMENTS.html`

없으면 중단하고 "draft 파일을 찾을 수 없습니다. /issue $ARGUMENTS 를 먼저 실행하세요." 라고 알린다.

HTML에서 아래 정보를 추출한다.

**파일 목록** (`.file-item` 요소들):
- `.file-item.create` → 신규 생성 파일
- `.file-item.modify` → 수정 파일
- `.file-item.delete` → 삭제 파일
- `.file-path` 텍스트 → 실제 파일 경로
- `.file-desc` 텍스트 → 구현 지침

**디자인 스펙** (`.spec-card` 요소들):
- 레이아웃 수치 (width, height, padding 등)
- 타이포그래피 (font-size, font-weight)
- 컬러 값

**작업 요약** (`.summary-box` 텍스트)

---

## Step 2. 코드베이스 탐색

파일 목록에 있는 각 경로를 실제로 확인한다.

- **수정 파일**: 반드시 먼저 `Read`로 현재 내용을 읽는다
- **신규 파일**: 같은 디렉터리의 인접 파일을 읽어 패턴(import 경로, 컴포넌트 구조, CSS 방식 등)을 파악한다

파일 설명에 언급된 컴포넌트명, 훅, 타입, 유틸이 실제로 어디 있는지 `grep`으로 확인한다.

---

## Step 3. 코드 구현

파일 목록 순서대로 구현한다. 각 파일마다:

### 신규 생성 (`create`)

- 같은 디렉터리 인접 파일의 컨벤션을 그대로 따른다
  - import 경로 스타일 (`@/features/...` 또는 상대경로)
  - CSS 방식 (Tailwind `@apply`, CSS Module, inline)
  - 컴포넌트 선언 방식 (`export const Foo = () => {}`)
  - TypeScript interface 위치 (파일 상단)
- draft 설명의 수치를 코드에 직접 반영한다
  - 예: "width: 270px, 비선택 44px" → CSS `.wrapper { width: 270px }` + `.wrapperInactive { width: 44px }`
  - 예: "16px / w600 / #ffffff" → `font-size: 16px; font-weight: 600; color: #ffffff`

### 수정 (`modify`)

- 읽어온 파일 내용을 기반으로 **최소한의 변경**만 적용한다
- 기존 로직, 스타일, import를 불필요하게 건드리지 않는다
- draft 설명에 명시된 변경 사항만 반영한다

### 삭제 (`delete`)

- 파일 삭제 전 해당 파일을 참조하는 import가 없는지 `grep`으로 확인한다
- 안전하면 삭제한다

---

## Step 4. 구현 결과 보고

모든 파일 작업 완료 후 아래 형식으로 보고한다.

```
✅ 구현 완료: $ARGUMENTS

신규 생성 (N개):
  - apps/.../Foo.tsx
  - apps/.../Foo.module.css

수정 (M개):
  - apps/.../Bar.tsx

주요 변경 내용:
  {핵심 구현 내용 2~3줄 요약}

확인 필요:
  {구현 중 판단이 어려웠던 부분, 확인이 필요한 부분}
```
