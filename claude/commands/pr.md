---
description: 현재 브랜치 변경사항을 정리해서 PR 생성. 사용법: /pr [target_branch] [--from origin_branch] [--ocr]
argument-hint: [target_branch] [--from origin_branch] [--ocr]
---

> ### 적용하기 전에
> 본 내용은 프로젝트 설정마다, 개인 환경마다 상이한 내용을 담고 있으므로
> 실제로 적용하기에 앞서 자신의 현재 프로젝트 환경에 맞춰
> 내용 업데이트를 AI에게 한 번 요청해야 합니다
> 이 파일에서 이야기하는 ocr은 ocr.md 파일과 연계되어 있습니다
> ocr(=open-code-review)는 `@alibaba-group/open-code-review` 프로젝트입니다

# /pr

입력: `$ARGUMENTS`

## 목적

현재 브랜치(또는 지정 브랜치)의 변경사항을 분석해 GitHub PR을 자동 생성한다. 이슈 내용까지 요약해서 반영한다.

## 인자 파싱

`$ARGUMENTS`에서 아래 값을 추출한다:

- `target_branch`: 위치 인자 (지정하지 않으면 `main`)
- `--from {브랜치}`: 원본 브랜치. 지정하지 않으면 현재 브랜치 (`git rev-parse --abbrev-ref HEAD`)
- `--ocr`: PR 생성 전 로컬 OCR 리뷰를 실행하고 결과를 PR 본문에 요약 포함. `ocr` CLI가 설치/설정되어 있지 않으면 이 플래그는 무시하고 안내만 출력

예시:
- `/pr` → target=main, from=현재 브랜치
- `/pr develop` → target=develop, from=현재 브랜치
- `/pr main --from JOB-01` → target=main, from=JOB-01
- `/pr develop --ocr` → target=develop, OCR 로컬 리뷰 실행 후 결과 요약을 PR 본문에 포함

## 1단계: 사전 검증

아래 조건 중 하나라도 해당하면 중단하고 사실을 알린다.

1. **target_branch와 origin_branch가 같은 경우** - 동작하지 않음
2. **origin_branch가 target_branch인 경우** (예: 현재 브랜치가 main) - 동작하지 않음
3. **미커밋 변경사항이 있는 경우** - `git status --porcelain`이 비어있지 않으면 "커밋되지 않은 변경사항이 있습니다. 먼저 커밋하거나 stash하세요." 경고 후 사용자 확인 요청
4. **`gh` 미인증** - `gh auth status` 실패 시 인증 안내 후 대기

`gh`는 항상 프로젝트 루트에서 실행한다.

## 2단계: 변경사항 수집

병렬로 아래 명령을 실행한다:

- `git log {target_branch}..{origin_branch} --pretty=format:'%h %s' --no-merges` - 커밋 목록
- `git diff {target_branch}...{origin_branch} --stat` - 파일별 변경 통계
- `git diff {target_branch}...{origin_branch} --name-status` - 파일 생성/수정/삭제 분류
- `git diff {target_branch}...{origin_branch}` - 전체 diff (필요 시 범위 좁혀서 읽기)
- `gh repo view --json owner,name -q '.owner.login + "/" + .name'` - 리포지토리 식별자

origin_branch가 remote에 없으면 원격 저장소 상태를 임의로 변경하지 말고 `git push -u origin {origin_branch}`를 먼저 실행할지 사용자에게 확인한다.

## 3단계: 이슈 추적

origin_branch에서 티켓 ID를 추출한다. 일반적으로 브랜치명 자체가 `{JOB-숫자}` 형식.

- 브랜치명이 `JOB-\d+` 패턴이면 해당 티켓 조회
- 커밋 메시지에서 `[JOB-\d+]` 패턴을 추가로 검색해서 교차 검증
- 티켓 발견 시 `gh issue view {숫자} --repo [user/myrepo] --json number,title,body` 로 본문 조회

조회된 이슈 본문을 1~3문장으로 요약한다. 이슈 본문이 모호하면 내용을 추정하지 말고 "이슈 본문 확인 필요"로 표기한다.

이슈를 찾지 못한 경우 PR 본문에서 연결 이슈 섹션을 생략하거나 "연결 이슈 없음"으로 표기.

## 3.5단계: 로컬 OCR 리뷰 (`--ocr` 플래그 지정 시만)

`--ocr` 플래그가 있을 때만 실행. 없으면 이 단계 전체를 건너뛴다.

### 사전 검증

병렬로 아래 명령 실행:

- `which ocr` - CLI 설치 여부
- `cat ~/.opencodereview/config.json 2>/dev/null` - LLM 설정 여부 (내용 출력 금지)

둘 중 하나라도 없으면 "OCR을 생략합니다. `ocr` CLI 설치 또는 LLM 설정이 필요합니다." 만 출력하고 이 단계를 건너뛴다. PR 생성 자체는 계속 진행.

### 캐시 조회

`/ocr` 커맨드와 동일한 캐시 정책 사용:

- base SHA: `git rev-parse origin/{target_branch}`
- head SHA: `git rev-parse HEAD`
- 캐시 키: `range:{base_sha}:{head_sha}`
- `.claude/ocr/index.json`에서 조회 → hit면 CLI 실행 생략, 기존 HTML과 인덱스의 요약 사용
- miss면 아래 리뷰 실행으로 진행

### 리뷰 실행

`/ocr` 커맨드의 2~4단계와 동일한 절차로 로컬 리뷰를 실행한다:

- 모드: `range`, base=`origin/{target_branch}`, head=`HEAD`
- 명령: `ocr review --from origin/{target_branch} --to HEAD --format json --background "{최신 커밋 메시지}"`
- 결과: `/tmp/ocr-local-result.json`에 저장
- HTML 결과: `.claude/ocr/{timestamp}-range-{origin_branch}.html`에 저장 (한국어, 서버 구동 없이 브라우저로 바로 열 수 있게)
- 저장 후 `.claude/ocr/index.json`에 캐시 엔트리 갱신

### 결과 수집

`/tmp/ocr-local-result.json`에서 4단계 본문 작성을 위해 아래 값을 추출한다:

- 발견된 이슈 수 (comments.length)
- 심각도별 분류 (severity 필드가 있을 경우 집계, 없으면 미분류 N건)
- 주요 이슈 최대 5건: 파일 경로, 시작/끝 라인, 내용 1줄 요약
- 저장된 HTML 파일명
- warnings가 있으면 함께 기록

실행 실패 시 실패 사실만 기록하고 계속 진행 (PR 생성은 막지 않음).

## 4단계: PR 본문 작성

기존 PR 포맷(아래 템플릿)을 준수한다. 한국어로 작성.

```markdown
### 작업 내용

- {변경 내용 항목별 bullet. "무엇을, 왜"를 한 줄로. 3~7개 권장}

---

### 연결 이슈

- [{티켓ID}] #{이슈번호}

---

### 비고

- etc
```

### 작업 내용 작성 규칙

- 파일 단위가 아니라 **기능/변경 단위**로 묶는다 (예: "Toggle 컴포넌트 테스트 추가" - 파일 3개가 한 bullet)
- diff에서 파악한 핵심 변경만. 단순 import 정리, 포맷 변경은 묶어서 한 줄로
- 새로 추가된 의존성, 환경 설정 변경, 호환성 영향은 반드시 명시
- 이슈에서 요구한 사항과 실제 구현을 매핑 (이슈가 있을 때)

### 비고 섹션

아래 항목이 해당하면 기록, 아니면 `etc`로 유지:

- breaking change 여부
- 후속 티켓 필요 여부 (예: "페이지 컴포넌트 테스트는 별도 티켓에서 진행")
- 리뷰어가 특별히 봐야 할 부분
- 설정 마이그레이션 필요 (`pnpm install` 재실행 등)

### OCR 리뷰 섹션 (optional, `--ocr` 플래그 지정 시만)

아래 템플릿을 `### 비고` 위에 추가한다. OCR 실행 결과(`5.5단계`)에서 수집한 값을 반영한다.

```markdown
---

### AI 로컬 리뷰 (OCR)

- 발견 이슈: {N}건 (심각 {critical} / 경고 {warning} / 정보 {info})
- 주요 이슈:
  - {파일경로}:L{시작}-L{끝} — {이슈 요약 1줄}
  - {파일경리}:L{시작}-L{끝} — {이슈 요약 1줄}
- 상세: `{.claude/ocr/파일명.html}`
```

작성 규칙:
- 심각도 분류가 없으면 "발견 이슈: N건"으로만 표기
- 주요 이슈는 최대 5건. 그 이상은 상세 HTML로 유도
- 빈 결과(0건)면 섹션 전체를 "발견된 이슈 없음" 한 줄로 대체
- OCR 실행 자체를 실패한 경우 섹션을 생략하고 실패 사실만 비고에 1줄 기록

## 5단계: PR 제목 작성

형식: `[{브랜치명}] {변경 내용 요약}`

- 브랜치명은 origin_branch 그대로 (예: `JOB-01`)
- 요약은 20~40자, 핵심 변경 1개를 대표 명사로
- 예: `[JOB-01] 공통 컴포넌트 vitest 테스트코드 추가`
- 여러 변경이 균등하면 더 포괄적인 상위 개념으로 (예: "스킬 페이지 개선" ← 랭크 표시 + 푸터 추가)

## 6단계: 품질 게이트 (권장)

PR 생성 전 아래를 실행해서 실패하면 사용자에게 알리고 진행 여부 확인:

1. `pnpm --filter [monorepo_project] lint` - 린트 통과
2. `pnpm --filter [monorepo_project] test` - 테스트 통과

실패해도 강제는하지 않고, "테스트가 실패했습니다. PR 생성을 계속할까요?" 정도로 확인.

## 7단계: PR 생성

```bash
gh pr create \
  --base {target_branch} \
  --head {origin_branch} \
  --title "{제목}" \
  --body "$(cat <<'EOF'
{PR 본문}
EOF
)"
```

`--draft` 플래그는 사용자가 명시하지 않는 한 붙이지 않는다.

## 8단계: 보고

PR 생성 후 아래만 짧게 출력:

- PR URL (`gh`가 반환하는 URL)
- PR 번호
- 변경 파일 수 (신규 N / 수정 M / 삭제 K)
- 커밋 수
- `--ocr` 실행 시: OCR 발견 이슈 수 + 저장된 HTML 파일 경로

불필요한 반복을 피하기 위해 전체 PR 본문은 다시 출력하지 않는다.

## 유의사항

- force push, push, 브랜치 삭제 등 원격 상태나 이력에 영향을 주는 작업은 실행 전 사용자 확인 필수. force push는 사용하지 않으며 main/master에는 어떤 경우에도 force push하지 않음
- 이슈 본문 요약은 조회된 내용만 근거로 작성하고, 확인되지 않은 요구사항이나 의도는 추정하지 않음
- diff가 매우 큰 경우 통계와 name-status로 먼저 전체를 파악한 뒤 주요 파일만 선별적으로 읽는다
- 브랜치명에서 티켓 ID를 추출할 수 없으면 커밋 메시지 최신 것에서 추출 시도. 둘 다 실패하면 연결 이슈 없이 진행
