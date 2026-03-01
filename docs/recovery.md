# pi-parallel-agents recovery runbook

Practical recovery steps for common parallel-agent failures.

> Assumptions: run commands from the **parent repo root** (this directory), unless noted.

## Quick state map

- Registry: `.pi/parallel-agents/registry.json`
- Registry lock: `.pi/parallel-agents/registry.lock`
- Runtime files per agent: `.pi/parallel-agents/runtime/<agent-id>/`
  - `kickoff.md`, `backlog.log`, `exit.json`, `launch.sh`
- Worktree lock per agent slot: `../<repo>-agent-worktree-XXXX/.pi/active.lock`
- Merge critical-section lock (finish script): `.pi/parallel-agents/merge.lock`

---

## 1) Unknown / invalid agent IDs

### Symptoms

- `agent-check`: `{ "ok": false, "error": "Unknown agent id: a-9999" }`
- `agent-wait-any`: `{ "ok": false, "error": "Unknown agent id(s): ..." }` (fails fast on first poll)
- `agent-send`: `{ "ok": false, "message": "Unknown agent id: ..." }`

### Recovery

1. List known agents:
   - `/agents`
2. Retry with an existing id (`a-0001`, `a-0002`, ...).
3. If you expected a just-started agent, check registry directly:

```bash
node -e 'const fs=require("fs");const p=".pi/parallel-agents/registry.json";if(!fs.existsSync(p)){console.log("registry missing");process.exit(0)};const r=JSON.parse(fs.readFileSync(p,"utf8"));console.log(Object.keys(r.agents||{}).sort().join("\n"));'
```

4. If the id is gone but you need logs, inspect `.pi/parallel-agents/runtime/<id>/backlog.log` (if still present), then start a fresh agent.

---

## 2) Agent crashed / tmux window disappeared

### Symptoms

- Agent status becomes `crashed`
- Error often: `tmux window disappeared before an exit marker was recorded`
- `agent-send` returns `tmux window is not active`

### Recovery

1. Inspect current state (tool):

```text
agent-check { "id": "<id>" }
```

2. Inspect runtime artifacts:

```bash
ID="a-0001"
ls -la ".pi/parallel-agents/runtime/$ID"
tail -n 120 ".pi/parallel-agents/runtime/$ID/backlog.log"
[ -f ".pi/parallel-agents/runtime/$ID/exit.json" ] && cat ".pi/parallel-agents/runtime/$ID/exit.json"
```

3. If tmux window still exists, try graceful stop (tool):

```text
agent-send { "id": "<id>", "prompt": "!/quit" }
```

4. If tmux window is gone, treat as terminal crash:
   - collect backlog/error
   - spawn a replacement agent if needed (`/agent ...` or `agent-start`)
   - clean up old failed/crashed record (see section 5)

---

## 3) Stale registry lock contention (`registry.lock`)

### Symptoms

- Commands/tools fail with: `Timed out waiting for lock .../.pi/parallel-agents/registry.lock`

### Notes

- Registry writes use a file lock.
- Lock timeout is ~10s.
- Lock files older than ~30s are automatically considered stale and reaped.

### Recovery

1. Inspect lock file and timestamp:

```bash
LOCK=".pi/parallel-agents/registry.lock"
ls -l "$LOCK" 2>/dev/null || echo "no registry.lock"
[ -f "$LOCK" ] && cat "$LOCK"
```

2. If lock keeps timing out and no active parent operation is running, remove stale lock:

```bash
rm -f ".pi/parallel-agents/registry.lock"
```

3. Retry the failed action (`/agents`, `agent-check` tool, etc.).
4. If contention recurs, reduce concurrent parent sessions mutating the same registry.

---

## 4) Stale worktree `.pi/active.lock` files

### Symptoms

- Start warnings such as:
  - `Locked worktree is not tracked in registry: ...`
- New agents skip a slot that looks abandoned.

### Recovery

1. List lock files in worktree pool:

```bash
REPO_NAME="$(basename "$PWD")"
find .. -maxdepth 2 -type f -path "../${REPO_NAME}-agent-worktree-*/.pi/active.lock" -print
```

2. Inspect each lock and cross-check with `/agents` or `registry.json`.
3. Fast path: run `/agents` and confirm **Reclaim orphan worktree locks?** when offered.
   - This only targets orphan locks with no tracked registry agent and no live pid/tmux signal.
4. If you prefer manual cleanup, remove lock files directly:

```bash
LOCK="../<repo>-agent-worktree-0007/.pi/active.lock"
rm -f "$LOCK"
```

5. Before reusing/removing that worktree, check for uncommitted changes:

```bash
WT="../<repo>-agent-worktree-0007"
git -C "$WT" status --short
```

If non-empty, preserve work first (commit/patch) before destructive cleanup.

---

## 5) Failed/crashed agents cleanup flow

Use this when an agent is terminal (`failed` or `crashed`) and you want to tidy state.

1. Capture diagnostics first:
   - `agent-check` tool for `<id>`
   - save backlog/error if needed
2. Preserve any useful work in the agent worktree (`git -C <worktree> status`, patch/commit).
3. Remove registry entry (recommended path):
   - run `/agents`
   - confirm **Clean up failed agents?** prompt
4. Clean runtime and lock leftovers for that id:

```bash
ID="a-0001"
WT="../<repo>-agent-worktree-0001"   # use actual worktreePath from agent-check
rm -rf ".pi/parallel-agents/runtime/$ID"
rm -f "$WT/.pi/active.lock"
```

5. If tmux window still exists unexpectedly, close it:

```bash
tmux kill-window -t "@<window-id>" 2>/dev/null || true
```

---

## 6) Safe manual inspection commands (registry/runtime/locks)

### Read-only inspection

```bash
# Registry summary (id, status, tmux window, worktree)
node -e 'const fs=require("fs");const p=".pi/parallel-agents/registry.json";if(!fs.existsSync(p)){console.log("registry missing");process.exit(0)};const r=JSON.parse(fs.readFileSync(p,"utf8"));for(const id of Object.keys(r.agents||{}).sort()){const a=r.agents[id];console.log(`${id}\t${a.status}\t${a.tmuxWindowId??"-"}\t${a.worktreePath??"-"}`)}'

# Runtime dirs
find .pi/parallel-agents/runtime -maxdepth 2 -mindepth 2 -type d 2>/dev/null | sort

# Lock files
ls -l .pi/parallel-agents/*.lock 2>/dev/null || true
REPO_NAME="$(basename "$PWD")"; find .. -maxdepth 2 -type f -path "../${REPO_NAME}-agent-worktree-*/.pi/active.lock" -print
```

### Safe cleanup primitives

```bash
# Remove only known-stale lock files
rm -f .pi/parallel-agents/registry.lock
rm -f .pi/parallel-agents/merge.lock
rm -f ../<repo>-agent-worktree-XXXX/.pi/active.lock

# Remove runtime artifacts for one terminal agent
rm -rf .pi/parallel-agents/runtime/<agent-id>
```

Prefer `/agents` for registry record cleanup to avoid manual JSON edits while another session may be writing.
