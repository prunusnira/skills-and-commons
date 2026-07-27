---
name: claude
description: Claude Code의 .claude/commands 안에 있는 명령 문서를 참고해서 동일한 워크플로우로 작업해야 할 때 사용한다.
---

이 skill은 기존 Claude Code commands를 Codex에서 참고하기 위한 어댑터다.

작업 방식:

1. 사용자가 Claude command 이름을 언급하면 `.claude/commands/`에서 같은 이름 또는 유사한 이름의 `.md` 파일을 찾는다.
2. 해당 command 문서를 먼저 읽는다.
3. command 문서의 목적, 입력 형식, 작업 절차, 검증 명령을 현재 Codex 작업에 적용한다.
4. `.claude/commands`의 지침과 `AGENTS.md`가 충돌하면 `AGENTS.md`를 우선한다.
5. command 문서가 없거나 이름이 모호하면 가능한 후보를 짧게 제시하고 확인을 요청한다.

명령어 형식

.claude/commands/[command].md