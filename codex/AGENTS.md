# AGENTS.md

## Claude commands compatibility

이 프로젝트에는 기존 Claude Code용 명령어가 `.claude/commands/`에 남아 있다.

사용자가 claude commands에 정의된 command 이름을 언급하면, Codex는 먼저 `.claude/commands/<command-name>.md`를 찾아 읽고 해당 절차를 따른다.

Codex에서 명시적으로 command 호환 작업을 수행할 때는 `.agents/skills/claude-command/SKILL.md`를 사용한다.

`.claude/commands` 문서와 이 파일이 충돌하면 `AGENTS.md`를 우선한다.
