# Shared agent assets

이 디렉터리에는 여러 에이전트에서 재사용하는 skill과 프로젝트 지침 템플릿이 있다. 에이전트가 `.agents/`를 자동으로 발견하지 않는 경우가 있으므로, 작업 지침에서 이 인덱스를 직접 읽도록 안내한다.

## 내용

- `skills/<name>/SKILL.md`: 작업 유형별 절차. 각 skill의 `name`과 `description`을 살펴 요청과 맞는 것만 읽는다. skill이 참조하는 `references/` 자료는 필요한 경우 함께 읽는다.
- `profiles/<stack>/AGENTS.md`: Android, Next.js, React, Unity 스택별 프로젝트 지침 템플릿. 해당 스택 작업에서만 참고하고, 프로젝트에 적용하려면 대상 프로젝트의 적절한 경로에 복사한다.
- `work/`: 작업 중 생성하는 임시 리포트와 산출물의 기본 위치

저장소 루트 `AGENTS.md`는 작업을 시작할 때 이 파일을 읽고, 요청과 맞는 skill을 선택하도록 안내한다. Claude Code는 프로젝트 skill을 `.claude/skills/`에서 검색하므로, 저장소의 해당 항목들은 이 디렉터리의 `skills/` 하위 폴더를 가리키는 심볼릭 링크다. 원본 skill은 이곳에서만 관리한다.
