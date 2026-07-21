---
description: 티켓(GitHub Issue 또는 Notion)을 분석해 구현 draft HTML 생성
argument-hint: <티켓ID 또는 URL>
---

> ### 적용하기 전에
> 본 내용은 프로젝트 설정마다, 개인 환경마다 상이한 내용을 담고 있으므로
> 실제로 적용하기에 앞서 자신의 현재 프로젝트 환경에 맞춰
> 내용 업데이트를 AI에게 한 번 요청해야 합니다

# /issue

입력: `$ARGUMENTS`

## 목적

이슈(ID 또는 URL)를 분석해 `.claude/drafts/{티켓ID}.html`에 구현 draft HTML을 생성한다. 이후 `/impl {티켓ID}`로 실제 코드를 구현한다.

## 1단계: 소스 판별 및 티켓 조회

`$ARGUMENTS` 형태로 소스를 자동 판별한다.

- `[PREFIX-number]` 또는 GitHub URL → GitHub Issue. `gh issue view <ID> --repo [myrepo] --json number,title,body,labels,state,assignees,comments` 로 조회. 인증 오류(`gh auth login` 필요) 시 사용자에게 안내 후 대기. 네트워크 오류 시 `[github_repo_url]/issues/<ID>`를 WebFetch로 폴백.
- Notion URL 또는 32자 UUID → Notion 페이지. Notion MCP가 활성화된 경우 `mcp__notion__*` 툴로 본문 조회. MCP가 없으면 사용자에게 본문을 직접 붙여넣어달라고 요청하고 대기.
- 그 외 형태 → MCP 연결 설정을 시도 한 후, 그래도 데이터를 받아올 수 없다면 사용자에게 소스(GitHub/Notion)와 본문을 명시적으로 물어본다.

GitHub Issue 번호만 숫자로 들어온 경우에도 `gh issue view <숫자>`로 시도. `gh`는 항상 프로젝트 루트에서 실행한다.

조회 실패 시 내용을 추정하지 말고, 실패 사실과 원인을 확인 가능한 범위에서 알린 뒤 다음 입력을 요청한다.

## 2단계: 코드베이스 컨텍스트 탐색

티켓 본문에서 추출한 키워드로 관련 파일을 탐색한다. 반드시 실제 존재하는 파일만 사용한다 (Glob/Grep로 검증).

- `feature/<도메인>/` 하위에서 동일/유사 기능 탐색
- `common/`, `lib/`에서 재사용 가능한 유틸 확인
- API 연동이 필요한 경우 `url/api.ts`, `app/api/` 라우트 확인
- 수정이 필요한 파일과 신규 생성해야 할 파일을 분리해서 정리

## 3단계: draft HTML 작성

`.claude/drafts/{티켓ID}.html` 파일을 생성한다. 단일 HTML 파일로 인라인 CSS를 사용하고 외부 의존성 없이 동작하게 한다. 한국어로 작성.

포함 항목:

1. **티켓 요약** - 제목, 설명, 인수조건(acceptance criteria), 원본 링크
2. **구현 범위** - 수정 파일 목록, 신규 파일 목록 (경로 + 한 줄 설명)
3. **구현 단계 체크리스트** - 순차적으로 실행 가능한 TODO 항목. 체크박스 형태
4. **리스크/확인 포인트** - 아래처럼 구현 방향이 갈리거나 되돌리기 어려워 사용자 확인이 필요한 사항
   - 디자인 링크가 없고 UI 구현 방향이 두 가지 이상인 경우
   - 기존 컴포넌트 수정과 신규 컴포넌트 생성 중 판단이 불명확한 경우
   - API를 BFF 경유로 연동할지 직접 호출할지 불명확한 경우
   - 파일 삭제나 덮어쓰기 등 되돌리기 어려운 작업이 필요한 경우
   - 코드 탐색 중 티켓 설명과 다른 구현 방식을 발견한 경우
5. **후속 작업** - 구현 완료 후 자동 수행:
   - `.claude/skill/` 폴더 내 vitest, storybook 관련 템플릿 파일을 참고하여 테스트 코드와 스토리 자동 작성
   - 해당 폴더/파일이 비어 있거나 없으면 사용자에게 템플릿이 없음을 알리고 먼저 세팅하도록 안내

상단에 티켓 ID와 원본 링크를 표시하고, 각 섹션은 명확한 헤더로 구분한다.

## 4단계: 보고

draft HTML 파일 경로와 핵심 요약(구현 범위, 리스크)만 짧게 출력하고, 전체 내용은 다시 출력하지 않는다.

### HTML draft 생성

아래 형식으로 HTML 파일을 생성한다.

저장 경로: `.claude/draft/{티켓ID}.html`

`.claude/draft/` 디렉터리가 없으면 먼저 생성한다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>[{티켓ID}] {티켓 제목}</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, sans-serif; background: #f5f5f5; color: #1a1a1a; padding: 32px; }
    .container { max-width: 900px; margin: 0 auto; }
    h1 { font-size: 22px; font-weight: 700; margin-bottom: 6px; }
    .badge { display: inline-block; padding: 3px 10px; border-radius: 12px; font-size: 12px; font-weight: 600; margin-bottom: 24px; }
    .badge.new { background: #dbeafe; color: #1d4ed8; }
    .badge.change { background: #fef9c3; color: #854d0e; }
    .section { background: #fff; border: 1px solid #e5e7eb; border-radius: 10px; padding: 24px; margin-bottom: 20px; }
    .section h2 { font-size: 15px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.05em; margin-bottom: 16px; }
    .file-list { list-style: none; display: flex; flex-direction: column; gap: 10px; }
    .file-item { display: flex; align-items: flex-start; gap: 12px; padding: 12px 14px; border-radius: 8px; border: 1px solid #e5e7eb; }
    .file-item.create { border-left: 3px solid #22c55e; background: #f0fdf4; }
    .file-item.modify { border-left: 3px solid #f59e0b; background: #fffbeb; }
    .file-item.delete { border-left: 3px solid #ef4444; background: #fef2f2; }
    .file-tag { font-size: 11px; font-weight: 700; padding: 2px 7px; border-radius: 4px; white-space: nowrap; margin-top: 1px; }
    .tag-create { background: #dcfce7; color: #15803d; }
    .tag-modify { background: #fef3c7; color: #d97706; }
    .tag-delete { background: #fee2e2; color: #dc2626; }
    .file-info { flex: 1; }
    .file-path { font-size: 13px; font-family: monospace; font-weight: 600; margin-bottom: 4px; word-break: break-all; }
    .file-desc { font-size: 13px; color: #6b7280; line-height: 1.5; }
    .checklist { list-style: none; display: flex; flex-direction: column; gap: 8px; }
    .checklist li { display: flex; align-items: flex-start; gap: 8px; font-size: 14px; line-height: 1.5; }
    .checklist li::before { content: "☐"; font-size: 16px; margin-top: -1px; flex-shrink: 0; }
    .summary-box { background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 8px; padding: 14px 16px; font-size: 14px; line-height: 1.7; color: #334155; }
    .meta { font-size: 13px; color: #9ca3af; margin-bottom: 4px; }
    .legend { display: flex; gap: 16px; margin-bottom: 16px; font-size: 12px; }
    .legend-item { display: flex; align-items: center; gap: 5px; }
    .legend-dot { width: 10px; height: 10px; border-radius: 2px; }
    .design-link { display: inline-flex; align-items: center; gap: 8px; padding: 10px 16px; background: #f0f4ff; border: 1px solid #c7d7fd; border-radius: 8px; color: #3b5bdb; font-size: 13px; font-weight: 600; text-decoration: none; margin-bottom: 12px; }
    .design-link:hover { background: #e0e9ff; }
    .design-notes { font-size: 13px; color: #475569; line-height: 1.7; margin-top: 12px; background: #f8fafc; border-radius: 6px; padding: 12px 14px; border: 1px solid #e2e8f0; white-space: pre-line; }
    .design-tag { display: inline-block; font-size: 10px; font-weight: 700; padding: 1px 6px; border-radius: 4px; background: #e0e9ff; color: #3b5bdb; margin-left: 6px; vertical-align: middle; }
    .ref-list { list-style: none; display: flex; flex-direction: column; gap: 8px; }
    .ref-item { display: flex; align-items: flex-start; gap: 10px; padding: 10px 14px; border-radius: 8px; border: 1px solid #e5e7eb; background: #fafafa; }
    .ref-item.ref-jira { border-left: 3px solid #0052cc; }
    .ref-item.ref-confluence { border-left: 3px solid #0052cc; }
    .ref-item.ref-github { border-left: 3px solid #24292f; }
    .ref-item.ref-comment { border-left: 3px solid #7c3aed; }
    .ref-item.ref-etc { border-left: 3px solid #9ca3af; }
    .ref-tag { font-size: 11px; font-weight: 700; padding: 2px 7px; border-radius: 4px; white-space: nowrap; margin-top: 1px; background: #e0e7ff; color: #4338ca; }
    .ref-info { flex: 1; }
    .ref-link { font-size: 13px; font-weight: 600; color: #1d4ed8; text-decoration: none; word-break: break-all; }
    .ref-link:hover { text-decoration: underline; }
    .ref-summary { font-size: 12px; color: #6b7280; line-height: 1.5; margin-top: 4px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="meta">{티켓ID} · {이슈타입} · {날짜}</div>
    <h1>{티켓 제목}</h1>
    <span class="badge {new|change}">{신규 작업 | 변경 작업}</span>

    <!-- 디자인 참조 (Zeplin/Figma 링크가 있을 경우에만 렌더링) -->
    <div class="section">
      <h2>디자인 참조 <span class="design-tag">ZEPLIN</span></h2>
      <a class="design-link" href="{디자인_링크_URL}" target="_blank">
        🎨 {화면명 또는 링크 제목} — 디자인 열기
      </a>
      <div class="design-notes">{WebFetch로 추출한 화면 구성 정보, 컴포넌트명, 레이아웃 힌트 등. 추출 불가 시 "디자인 확인 후 구현 사항을 파악하세요." 로 대체}</div>
    </div>

    <!-- 참조 링크 (Step 1.3에서 수집한 링크가 있을 경우에만 렌더링) -->
    <div class="section">
      <h2>참조 링크</h2>
      <ul class="ref-list">
        <!-- 각 링크마다 아래 패턴 반복. 섹션별로 그룹핑 -->
        <li class="ref-item ref-{jira|confluence|github|comment|etc}">
          <span class="ref-tag">{JIRA|기획서|PR|댓글|링크}</span>
          <div class="ref-info">
            <a class="ref-link" href="{URL}" target="_blank">{제목 또는 URL}</a>
            <div class="ref-summary">{가져온 내용 요약. 현재 작업과 관련된 핵심 정보만. 가져올 수 없으면 생략}</div>
          </div>
        </li>
      </ul>
    </div>

    <!-- 요약 -->
    <div class="section">
      <h2>작업 요약</h2>
      <div class="summary-box">{티켓 설명을 2~4문장으로 요약. 무엇을 왜 만드는지 또는 무엇을 왜 바꾸는지}</div>
    </div>

    <!-- 파일 목록 -->
    <div class="section">
      <h2>파일 작업 목록</h2>
      <div class="legend">
        <span class="legend-item"><span class="legend-dot" style="background:#22c55e"></span>신규 생성</span>
        <span class="legend-item"><span class="legend-dot" style="background:#f59e0b"></span>수정</span>
        <span class="legend-item"><span class="legend-dot" style="background:#ef4444"></span>삭제</span>
      </div>
      <ul class="file-list">
        <!-- 각 파일마다 아래 패턴 반복 -->
        <li class="file-item {create|modify|delete}">
          <span class="file-tag {tag-create|tag-modify|tag-delete}">{신규|수정|삭제}</span>
          <div class="file-info">
            <div class="file-path">{실제 파일 경로}</div>
            <div class="file-desc">{이 파일에서 구체적으로 무엇을 할지. 예: ProductCard 컴포넌트에 discount 배지 props 추가, 조건부 렌더링 분기 처리}</div>
          </div>
        </li>
      </ul>
    </div>

    <!-- 체크리스트 -->
    <div class="section">
      <h2>작업 체크리스트</h2>
      <ul class="checklist">
        <!-- 실제 구현 순서에 맞게 작성 -->
        <li>{구체적인 작업 항목}</li>
      </ul>
    </div>

    <!-- 승인 기준 (Acceptance Criteria가 있을 경우) -->
    <div class="section">
      <h2>완료 기준</h2>
      <ul class="checklist">
        <li>{AC 항목}</li>
      </ul>
    </div>
  </div>
</body>
</html>
```

파일 목록 작성 규칙:
- 실제로 존재 확인한 파일만 "수정"으로, 새로 만들어야 하는 파일만 "신규"로 표시
- 경로는 프로젝트 루트 기준 상대경로로 작성 (`apps/...`)
- 각 파일 설명은 "무엇을" 구체적으로 작성 (추상적인 "로직 구현" 금지)
- 관련 없는 파일은 포함하지 않는다
- **디자인 링크가 있는 경우**: 디자인에서 확인된 컴포넌트명, 레이아웃 구조, 색상 토큰, 간격값 등을 파일 설명에 반영한다. 예: "Figma에 정의된 내용을 기반으로 width 100px, 이미지 비율 4:3, 이름에 사용할 항목 typography `title-2`"
- **디자인 참조 섹션**: 링크가 없으면 해당 `<div class="section">` 블록 전체를 생략한다
- **참조 링크 섹션**: Step 1.3에서 수집한 링크가 없으면 해당 블록 전체를 생략한다. 있으면 섹션 출처(본문/댓글/연결이슈)를 `ref-tag`로 구분해 표시한다
- **참조 링크 활용**: Step 1.3 요약에서 파악한 선행 작업, 스펙 변경, API 변경 등을 파일 설명과 체크리스트에 구체적으로 반영한다. 예: "[PREFIX-number]에서 특정 기능의 API가 변경됨 — 해당 변경으로 인한 필드 추가 필요"

완성된 HTML을 위 경로에 저장하고, 저장된 전체 경로와 파일 목록 개수(신규 N개, 수정 M개)를 간략히 알려준다.


## 유의사항

- 파일 경로는 Glob/Grep 등으로 실제 존재 여부를 확인하며 추정해서 작성하지 않는다
- 티켓 본문이 모호하면 빈칸을 채우지 말고 "확인 필요"로 표시한다
- draft는 확정된 명세가 아니라 `/impl` 단계에서 검증되는 설계안이다
