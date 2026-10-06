# Project settings

여러 코딩 에이전트에서 함께 쓰는 instruction profile, Agent Skills, GitHub workflow 모음이다.

## 구조

- `AGENTS.md`: 이 카탈로그 저장소의 공통 지침. 작업 시작 시 `.agents/README.md`와 요청에 맞는 skill을 읽도록 안내한다.
- `.agents/README.md`: 공통 에이전트 자료의 인덱스와 사용법
- `.agents/profiles/<stack>/AGENTS.md`: Android, C++/CMake, Next.js, React, Unity 프로젝트에 적용할 수 있는 지침 템플릿. 대상 프로젝트에 맞는 파일을 복사해 루트 또는 해당 하위 경로의 `AGENTS.md`로 사용한다.
- `.agents/skills/<name>/SKILL.md`: 공통 Agent Skills 형식으로 정리한 작업 절차와 참고 문서
- `.claude/skills/<name>`: Claude Code가 프로젝트 skill로 불러오도록 `.agents/skills/<name>`을 가리키는 심볼릭 링크. 원본 문서는 `.agents/skills`에서만 관리한다.
- `.agents/skills/e2e/references/WORKFLOW.md`: E2E skill 역할들이 공유하는 계약
- `.agents/work/`: issue draft, review, OCR 등 실행 중 생성하는 리포트와 임시 산출물의 기본 위치

## Claude 호환성

Claude Code v2.1.277 이상은 작업 디렉터리와 상위 경로에 `CLAUDE.md` 또는 `CLAUDE.local.md`가 없으면 `AGENTS.md`를 프로젝트 지침으로 읽는다. Claude Code는 프로젝트 skill을 `.claude/skills/`에서 찾으므로, 이 저장소는 해당 경로에서 `.agents/skills/`로 연결되는 심볼릭 링크를 제공한다. Claude Code는 skill 폴더의 심볼릭 링크를 지원한다.

작업 디렉터리나 상위 경로에 `CLAUDE.md` 또는 `CLAUDE.local.md`가 있으면 Claude Code 기본 설정은 `AGENTS.md`를 건너뛴다. 두 파일을 함께 읽어야 하면 Claude Code의 `/config`에서 **Project instructions**를 `claude-md-and-agents-md`로 설정한다. v2.1.280 이상에서는 `/memory`에서 지침 파일이 불러와졌는지 확인할 수 있다.

Claude Desktop의 Code 탭은 Claude Code와 같은 엔진과 프로젝트 설정을 사용하므로 이 저장소의 `.claude/skills/` 링크를 사용할 수 있다. 일반 Chat/Cowork 화면은 로컬 저장소의 skill 폴더를 자동 탐색하지 않으며, Cowork는 계정에 활성화한 skill을 사용한다.

## 출처

- 직접 작성
- Claude Marketplaces의 skills
- skills.sh
- 달레 스터디
- AI-generated custom Markdown
- 기타

E2E 자료는 Playwright 사용을 위해 작성했다.
