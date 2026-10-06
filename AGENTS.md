# AGENTS.md

이 저장소는 여러 코딩 에이전트에서 함께 사용하는 instruction profile과 skill 모음이다.

## 작업 시작 시 `.agents` 확인

- 먼저 `README.md`와 `.agents/README.md`를 읽는다. Claude Code, Claude Desktop을 포함해 에이전트가 `.agents/`를 자동 탐색하지 않을 수 있으므로, 이 지침에 따라 파일 도구로 직접 확인한다.
- `.agents/skills/`의 디렉터리와 `SKILL.md` 파일 목록을 확인한 뒤, 현재 요청에 맞는 skill만 골라 해당 `SKILL.md`와 그 문서가 참조하는 자료를 읽고 따른다. 모든 skill 문서를 무조건 한꺼번에 읽지는 않는다.
- 스택별 지침은 개별 프로젝트 내에 있다. 각 프로젝트 폴더 내의 `AGENTS.md` 파일을 참조한다 작업 대상 스택에 맞는 프로필만 참고한다. 프로필은 다른 프로젝트에 복사해 쓸 템플릿이며 이 카탈로그 저장소에 자동 적용되는 규칙이 아니다.

## 공통 규칙

- E2E 작업은 `e2e` skill과 역할별 `e2e-planner`, `e2e-runner`, `e2e-generator`, `e2e-healer` skill을 사용한다. 별도 agent 실행 기능이 없으면 같은 절차를 현재 작업에서 순서대로 따른다.
- GitHub 이슈와 PR의 원격 정보 및 작업은 가능한 경우 연결된 GitHub plugin/integration을 우선 사용한다. 사용할 수 없으면 저장소의 CLI와 셸 도구를 이용하고, 실패 시 확인 가능한 오류를 보고한다.
- 확인하지 않은 파일, API, 프로젝트 규칙을 추측하지 않는다.
- 별도의 Git worktree를 생성하거나 다른 worktree로 전환하지 않는다.
- 사용자가 현재 열어 둔 작업 디렉터리에서 직접 작업하여 로컬 변경사항을 바로 확인할 수 있게 한다.
- 현재 작업 디렉터리에 있는 기존 로컬 변경사항은 유지하며, 요청과 무관한 변경을 수정하거나 되돌리지 않는다.

## 필수 규칙

@.agents/limits/attention-kind (alexgreensh/attention-span)
@.agents/limits/karpathy (multica-ai/andrej-karpathy-skills)
@.agents/limits/verification-before-completion (obra/superpowers)

위 3개의 지침은 다른 어떤 지침보다 우선하며 반드시 따른다