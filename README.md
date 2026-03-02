# pi-side-agents

**Code in sprints** (using agents *asynchronously*), **not in a marathon** (*synchronous* task-by-task flow).

Instead of waiting for one backlog item to finish before starting the next, spin tasks out into single-use child agents as soon as they occur to you. Each child runs in its own **tmux window** and **git worktree**, so you can keep shipping in parallel while maintaining isolation. Each child is a one-off and lives and dies with its short topic branch and tmux window—no "teams of long-running agents messaging each other" or "role-based subagents" complexity. The workflow is unified, simple, and deterministic.

This extension automates the full tmux/worktree/merge lifecycle for you. Parallel agents can also be spawned and controlled by another agent to orchestrate its own flock of subagents.

**Warning:** You will build a lot more, which means you may run out of context windows and need to take better care of your wellbeing between sprints. Also, for the community's sake, please don't max out Claude subscriptions with Pi—use a Codex model (or APIs) by default.

## What it does

- Adds `/agent [-model ...] <task>` to spawn a background child Pi agent.
- Adds `/agents` to inspect current agents and clean up stale state.
- Shows active-agent summary with tmux window numbers in the statusline.
- Includes `agent-setup` skill to scaffold project-specific lifecycle scripts.
- Tracks runtime state in `.pi/side-agents/registry.json`.
- Exposes orchestration tools for parent agents:
  - `agent-start`
  - `agent-check`
  - `agent-wait-any`
  - `agent-send`

## Quick start

1. **Run setup once** in your project: `/skill:agent-setup`
   - If you want to change the setup later, or are upgrading this skill and want to get new setup goodies, just re-run the skill with a short prompt.
2. **Spawn asynchronous work items at any point** during your work:
   - `/agent wait, why is weirdMethod doing something-weird?`
   - `/agent -model gpt-5.3-codex add regression tests for auth`
   - Keep firing new items as they appear. As a rule of thumb, start all new work via `/agent`, but you can also use it only for ad hoc side questions.
3. **Check progress and attend** to the baby agents:
   - Check the statusline for which agents (by branch and tmux window) are waiting for you.
   - Use `/agents` to get a detailed overview of what's being done right now.
   - Steer the waiting children and work with them as normal Pi instances—just switch tmux windows.
4. If an agent is done, review its work and once happy, confirm by **LGTM, merge**.
   - Recommended: Write `commit your work when done` in your `AGENTS.md`. (You can always tell the agent to amend.)
   - Quickest way to review: ctrl+z, `git show`, `fg` to go back to the baby agent's Pi.
   - Your main worktree should be clean at this point; avoid editing in the main tree while side agents are active.
   - You can also tell your Pi to open GitHub PRs instead of merging locally, if that's what you prefer.
5. The agent will merge its work into your main repo. **Just type `/quit` and move on.**
   - Old worktrees are kept around and reused and updated by new agents.
   - Old branches are auto-pruned during reuse by a new agent.
   - You can pause your work on a topic—if you `/quit` before work is merged, the branch will stay around.

## Requirements

- `tmux`
- Git repository (worktrees enabled)
- Pi configured/authenticated

## Development

Run tests:

```bash
npm run test:unit
npm run test:integration
```

## Docs

- Architecture: `docs/architecture.md`
- Recovery/runbooks: `docs/recovery.md`
- Implementation notes: `docs/todo.md`
