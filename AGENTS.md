# AGENTS quick context

- Package: `pi-parallel-agents` (Pi extension + skill for parallel child-agent orchestration).
- Core code: `extensions/parallel-agents.ts`; setup skill: `skills/agent-setup/SKILL.md`.
- Public contract to preserve: `/agent`, `/agents`, tools `agent-start`, `agent-check`, `agent-wait-any`, `agent-send`.
- Runtime assumptions: `tmux` + git worktrees; shared registry at `.pi/parallel-agents/registry.json`.
- Validation: `npm run test:unit` and (real-runtime) `npm run test:integration`.
- Always commit completed work before reporting the task done.
