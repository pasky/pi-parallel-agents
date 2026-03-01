# pi-parallel-agents

**Code in sprints** (asynchronous agents), **not in a marathon** (serial task-by-task flow).

Instead of keeping a text backlog and waiting for one item to finish before starting the next, fire tasks off immediately into single-use child agents. Each child runs in its own **tmux window** and **git worktree**, so you can keep moving without cross-task contamination.

This extension automates the tmux/worktree lifecycle for you, and can also be used by an agent that orchestrates its own flock of subagents.

> ⚠️ Parallelism boosts throughput and model usage. Pick sensible default models and pace your review loop.

## What it does

- `/agent [-model ...] <task>`: spawn a child Pi agent in the background.
- `/agents`: inspect active agents and clean up stale runtime state.
- Statusline summary of active agents (including tmux window references).
- Shared runtime registry at `.pi/parallel-agents/registry.json`.
- Parent-orchestration tools:
  - `agent-start`
  - `agent-check`
  - `agent-wait-any`
  - `agent-send`
- `agent-setup` skill to scaffold project-specific lifecycle scripts.

## Quick start

1. Run setup once in your repo:
   - `/skill:agent-setup`
2. Spawn work as it appears:
   - `/agent investigate weirdMethod behavior`
   - `/agent -model gpt-5.3-codex add regression tests for auth`
3. Keep coding in your main session while children run.
4. Check and steer children:
   - statusline (`parallel-agents`)
   - `/agents`
   - switch to child tmux window when one is waiting
5. When a child is done, review and approve with **“LGTM, merge”**.
   - child lands changes, then you can `/quit` it.

## Workflow in one line

Spawn early, review often, merge small, keep your main flow unblocked.

## Requirements

- `tmux`
- Git repository (worktrees enabled)
- Pi configured/authenticated

## Development

```bash
npm run test:unit
npm run test:integration
```

## Docs

- Architecture: `docs/architecture.md`
- Recovery/runbooks: `docs/recovery.md`
- Implementation notes: `docs/todo.md`
