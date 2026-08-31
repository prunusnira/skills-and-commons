---
description: 로컬에서 OpenCodeReview CLI로 현재 변경사항 리뷰 후 HTML로 정리. 사용법: /ocr [target_branch|@<commit>] [--preview] [--json]
argument-hint: [target_branch|@<commit>] [--preview] [--json]
---

> ### 적용하기 전에
> 본 내용은 프로젝트 설정마다, 개인 환경마다 상이한 내용을 담고 있으므로
> 실제로 적용하기에 앞서 자신의 현재 프로젝트 환경에 맞춰
> 내용 업데이트를 AI에게 한 번 요청해야 합니다
> ocr(=open-code-review)는 `@alibaba-group/open-code-review` 프로젝트입니다

# /ocr

입력: `$ARGUMENTS`

## 공통 리뷰 기준

리뷰 결과의 중요도 분류와 리포트 작성 방식은 [review.md](review.md)를 따른다. OCR 고유의 JSON 필드에 심각도 값이 있더라도, 최종 리포트에는 `review.md`의 1~4단계 기준으로 정규화해 기록한다.

## 목적

로컬 환경에서 `ocr` CLI를 실행해 현재 브랜치(또는 워크스페이스) 변경사항을 AI 코드 리뷰한 뒤, 결과를 한국어 HTML로 `.claude/ocr/`에 저장한다. GitHub Actions 워크플로우(`open-code-review.yml`) 없이 커밋 전에 빠르게 피드백을 받을 때 사용.

참고: https://github.com/alibaba/open-code-review

## 인자 파싱

`$ARGUMENTS`에서 아래 값을 추출한다:

- `target_branch`: 위치 인자. 지정하지 않으면 `develop`. `@<commit>` 형태면 단일 커밋 모드로 동작
- `--preview` / `-p`: LLM 호출 없이 검사 대상 파일만 미리보기
- `--json` / `-f json`: 터미널에 원본 JSON 그대로 출력 (HTML 생성은 유지)

예시:
- `/ocr` → develop..HEAD 리뷰 후 HTML 저장
- `/ocr main` → main..HEAD 리뷰 후 HTML 저장
- `/ocr @abc1234` → 단일 커밋 리뷰 후 HTML 저장
- `/ocr --preview` → 검사 대상 파일만 미리보기
- `/ocr develop --json` → develop..HEAD 리뷰, 터미널에 JSON 원본도 출력

## 1단계: 사전 검증

병렬로 아래 명령을 실행한다:

- `which ocr` - CLI 설치 여부. 없으면 "ocr이 설치되어 있지 않습니다. `npm install -g @alibaba-group/open-code-review` 로 설치하세요." 출력 후 중단
- `ocr version` - 설치된 버전 확인
- `cat ~/.opencodereview/config.json 2>/dev/null` - LLM 설정 여부. 없으면 설정 상태를 추정하지 않고 "LLM 설정이 없습니다. `ocr config provider` 또는 `ocr config set llm.url/auth_token/model` 로 먼저 설정하세요." 출력 후 중단. 내용 출력 금지 (API 키 포함)
- `git rev-parse --is-inside-work-tree` - Git 저장소 여부
- `git rev-parse --abbrev-ref HEAD` - 현재 브랜치명

`ocr`가 설치되어 있지 않거나 LLM 설정이 없으면 사용자에게 사실을 알리고 대기. 임의로 설치나 설정을 진행하지 않는다.

## 2단계: 리뷰 대상 결정

`target_branch`가 `@<commit>` 형태인 경우 단일 커밋 모드:

- `git rev-parse <commit>`로 커밋 존재 검증. 실패하면 중단
- 모드: `commit`
- base SHA: `git rev-parse <commit>^`
- head SHA: `git rev-parse <commit>`
- 캐시 키: `{mode}:{base_sha}:{head_sha}`
- 명령: `ocr review --commit <commit> --format json`

그 외에는 브랜치 범위 모드:

- `git rev-parse --verify origin/<target_branch>` - remote 브랜치 존재 확인. 실패하면 원격 상태를 임의로 변경하지 말고 `git fetch origin <target_branch>` 실행 여부를 사용자에게 확인
- 현재 브랜치가 `target_branch`와 같으면 "현재 브랜치가 {target_branch}입니다. 리뷰할 변경사항이 없습니다." 출력 후 중단
- 모드: `range`
- base SHA: `git rev-parse origin/<target_branch>`
- head SHA: `git rev-parse HEAD`
- 캐시 키: `{mode}:{base_sha}:{head_sha}`
- 명령: `ocr review --from origin/<target_branch> --to HEAD --format json`

## 2.5단계: 캐시 조회

`.claude/ocr/index.json`에서 캐시 키로 조회. 파일이 없으면 빈 객체로 간주.

- 키가 존재하고 매핑된 HTML 파일이 실제 디스크에 있으면 캐시 hit:
  - OCR CLI 실행, HTML 생성 생략
  - "캐시된 리뷰 결과를 사용합니다: {HTML 파일 경로}" 출력
  - 5단계(결과 보고)로 바로 이동. 이때 저장된 HTML 파일을 다시 파싱하거나, 인덱스에 저장된 요약(이슈 수, 심각도별 집계, 주요 이슈)을 사용
- 키가 없거나 매핑된 파일이 없으면 캐시 miss → 3단계로 진행

`--preview` 모드는 캐시를 사용하지 않고 항상 실시간으로 검사 대상 파일을 나열한다.

인덱스 구조:
```json
{
  "range:abc1234:def5678": {
    "file": "20260623-153012-range-JOB-01.html",
    "base_sha": "abc1234...",
    "head_sha": "def5678...",
    "base_ref": "origin/develop",
    "head_ref": "JOB-01",
    "timestamp": "2026-06-23T15:30:12",
    "comment_count": 5,
    "warning_count": 0,
    "ocr_version": "1.3.13"
  }
}
```

## 3단계: 리뷰 실행

프로젝트 루트에서 `ocr review`를 실행한다. 결과는 항상 `--format json`으로 받아 `/tmp/ocr-local-result.json`에 리다이렉트하고 stderr는 `/tmp/ocr-local-stderr.log`로 분리. 플래그 조합:

- `--preview` 지정 시 `--preview` 추가 (LLM 호출 생략, HTML 저장 생략 - 미리보기만 터미널 출력)
- `--json` 지정 시 터미널에 원본 JSON 내용도 같이 출력
- `--audience human` 기본 (진행률 표시)
- `--background`에 최근 커밋 메시지를 자동으로 전달:
  - 단일 커밋 모드: `git log -1 --pretty=%s <commit>`
  - 브랜치 범위 모드: `git log -1 --pretty=%s HEAD`

실행 전 커맨드를 사용자에게 보여주고, 타임아웃은 600000ms (10분).

실행 중 실패하면 확인되지 않은 원인을 추정하지 않는다. stderr에서 API 키 등 민감 정보를 마스킹한 뒤 오류 내용을 보여주고 사용자가 판단할 수 있게 한다.

## 4단계: HTML 결과 저장

`--preview` 모드가 아니면 `/tmp/ocr-local-result.json`을 파싱해 한국어 HTML을 작성한다.

### 파일 위치

`.claude/ocr/` 디렉터리가 없으면 먼저 생성한다.

파일명 규칙: `{YYYYMMDD-HHMMSS}-{mode}-{ref}.html`
- timestamp: `date +%Y%m%d-%H%M%S`
- mode: `commit` 또는 `range`
- ref: 단일 커밋 모드면 SHA 앞 7자리, 브랜치 범위 모드면 현재 브랜치명 (`/`는 `-`로 치환)

예: `.claude/ocr/20260623-153012-range-JOB-01.html`

### HTML 구조

단일 HTML 파일로 인라인 CSS만 사용 (외부 의존성 없이 서버 없이 브라우저로 바로 열 수 있도록). 한국어 작성. issue 커맨드의 draft HTML과 동일한 디자인 언어 사용 (동일한 색상 토큰, 레이아웃 패턴).

포함 섹션:

1. **헤더**
   - 날짜, 모드(range/commit), base..head ref, ocr 버전
   - 리뷰된 커밋 수, 변경 파일 수 (사전 수집한 통계에서)

2. **요약 박스** (`summary-box` 클래스)
   - 발견된 이슈 수와 1~4단계별 집계 (comments.length 기준)
   - 경고 수 (warnings.length)
   - 0건이면 "발견된 이슈가 없습니다. 코드가 양호해 보입니다." 문구

3. **이슈 목록** (comments 배열, 4단계부터 1단계 순서)
   - 각 이슈는 파일별 카드로 표시
   - 항목: 파일 경로(`.file-path`), 라인 범위, `review.md` 기준의 단계, 내용(content), 영향과 권장 조치
   - suggestion_code와 existing_code가 있으면 `<details><summary>제안 변경</summary>` 로 접기/펼치기. before/after 코드블록 형식
   - 라인 정보가 없는 코멘트는 "파일 전체"로 표시

4. **검사된 파일 목록** (optional - result에 reviewed_files 또는 files 필드가 있을 경우)
   - 파일 경로와 검사 상태만 표 형태

5. **미확인 사항·경고** (warnings가 있을 경우만 렌더링)
   - 원인과 조치 제안을 한국어로 표기. 단순 stderr 복사 금지

6. **메타데이터 푸터**
   - 실행 커맨드 (API 키 등 민감 정보는 제외)
   - base SHA, head SHA
   - 커밋 메시지 (`--background`로 전달한 값)

### HTML 이스케이프 규칙

OCR JSON의 모든 문자열을 HTML에 안전하게 삽입:
- `&` → `&amp;`
- `<` → `&lt;`
- `>` → `&gt;`
- `"` → `&quot;`
- 코드 블록 내부도 동일하게 이스케이프

### 빈 결과 처리

comments가 빈 배열이거나 JSON 파싱에 실패한 경우에도 HTML을 생성한다:
- 파싱 실패: "결과 파싱 실패" 에러 페이지. stderr를 `<pre>`로 표시 (민감 정보 마스킹)
- 빈 배열: "발견된 이슈 없음" 페이지. 검사된 파일 목록은 표시

### 인덱스 갱신

HTML 저장 후 `.claude/ocr/index.json`에 2단계에서 만든 캐시 키로 엔트리를 추가/갱신한다. 기존 키가 있으면 덮어쓰기. 파싱 실패로 에러 페이지를 생성한 경우에는 인덱스에 추가하지 않는다 (캐시 무효).

## 5단계: 결과 보고

터미널에는 아래 결과만 짧게 출력한다:

- 저장된 HTML 파일 경로
- 발견된 이슈 수 (1~4단계별 집계)
- 파일별 이슈 분포 (상위 3개 파일)
- 경고 수

`--json` 지정 시 원본 JSON도 함께 출력.

`--preview` 모드는 HTML을 저장하지 않고 터미널에 검사 대상 파일 목록만 출력.

## 6단계: 후속 안내

리뷰 완료 후 옵션 안내 (사용자가 요구하지 않으면 실행하지 않음):

- 저장된 HTML을 브라우저로 열기: `open {파일경로}` (macOS)
- ocr 자체 세션 뷰어: `ocr viewer` (localhost:5483, 서버 구동 필요)
- 워크스페이스 모드(staged/unstaged/untracked 전체)로 다시 리뷰하려면 `ocr review` 직접 실행
- CI에 동일 리뷰를 적용하려면 `.github/workflows/open-code-review.yml`이 이미 있음

## 유의사항

- `ocr` CLI 설치, LLM 설정, fetch처럼 환경이나 원격 상태를 바꾸는 작업은 사용자 확인 없이 진행하지 않는다
- ocr 설정 파일(`~/.opencodereview/config.json`)에는 API 키가 포함될 수 있으므로 출력 시 auth_token, api_key, token 값은 마스킹
- HTML에는 API 키, LLM URL의 인증 정보, 로컬 절대 경로 중 민감 정보는 포함하지 않는다
- HTML은 PR을 만들기 전 사전 점검 용도의 개인 메모. CI를 대체하지 않는다
- `.claude/ocr/` 디렉터리의 HTML 파일들은 누적되므로 사용자가 주기적으로 정리한다. 되돌리기 어려운 자동 삭제는 하지 않는다
