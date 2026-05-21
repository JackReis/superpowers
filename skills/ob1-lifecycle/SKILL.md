---
name: ob1-lifecycle
description: Auto-brief (session start) and auto-drain (session end) for OB1 external brain and OBn vault-local memory. Queries OB1 for relevant context at startup, persists durable findings at shutdown.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [ob1, obn, brain, lifecycle, brief, drain, memory, persistence]
    related_skills: [session-briefing, agent-achievement-snapshots, coordination-folder-workflow]
---

# OB1/OBn Lifecycle — Auto-Brief and Auto-Drain

## When to Use

**Auto-Brief** — when starting a new session, resuming work, or the operator asks "brief me" / "what does OB1 know?"

**Auto-Drain** — when wrapping up a session, after a significant finding, or the operator says "drain" / "persist findings" / "save to brain"

## Architecture

```
Session Start → Auto-Brief → Read OBn file + Query OB1 → Inject context
Session End   → Auto-Drain  → Collect durable facts     → Append OBn + Upsert OB1 + Post #bots
```

## Auto-Brief Procedure

### Step 1: Read OBn (local vault memory)

```bash
cat ~/Documents/=notes/claude/memory/HERMES-MEMORY.md
```

Parse sections: Infrastructure, Decisions, User Preferences, Fleet Coordination.

### Step 2: Query OB1 (external brain)

If `BRAIN_KEY` is set in environment:

```python
# Recent curated thoughts — use MCP list_thoughts
import json, urllib.request, os

OB1_URL = "https://jhpuctiyosazlyrcnfuu.supabase.co/functions/v1/open-brain-mcp"
key = os.environ["BRAIN_KEY"]

# Recent thoughts
req = urllib.request.Request(
    OB1_URL,
    data=json.dumps({
        "jsonrpc": "2.0", "id": 1, "method": "tools/call",
        "params": {"name": "list_thoughts", "arguments": {"limit": 5, "source": "hermes"}}
    }).encode(),
    headers={"Content-Type": "application/json", "x-brain-key": key},
    method="POST"
)

# Topic-specific search — use MCP search_thoughts
req = urllib.request.Request(
    OB1_URL,
    data=json.dumps({
        "jsonrpc": "2.0", "id": 1, "method": "tools/call",
        "params": {"name": "search_thoughts", "arguments": {"query": "<topic>", "limit": 3}}
    }).encode(),
    headers={"Content-Type": "application/json", "x-brain-key": key},
    method="POST"
)
```

The `ob1-pull` CLI script wraps these same MCP calls for command-line use:
```bash
python3 ~/Documents/=notes/bin/ob1-pull --recent --limit 5
python3 ~/Documents/=notes/bin/ob1-pull --query "<topic>" --limit 3
```

If `BRAIN_KEY` is not set, skip OB1 and note that the external brain is unavailable.

### Step 3: Format briefing

Present as a compact block:

```
📋 OBn Briefing:
- [Infrastructure fact most relevant to current work]
- [Recent decision affecting current task]
- [User preference to honor this session]

📋 OB1 Recent:
- [Thought 1 summary]
- [Thought 2 summary]
```

Keep total briefing ≤15 lines. Prefer facts directly relevant to the current task over general knowledge.

### Step 4: Surface conflicts

If OBn and OB1 contain contradictory facts, surface the conflict:
```
⚠️ OBn/OB1 conflict: OBn says X, OB1 says Y — current working assumption: X
```

## Auto-Drain Procedure

### Step 1: Identify durable facts

From the session, extract only facts that will matter in 7+ days:
- ✅ Environment changes (new host, new port, new dependency)
- ✅ User corrections ("don't do X", "prefer Y")
- ✅ Architecture decisions ("use #bots for fleet comms")
- ✅ Tool quirks discovered ("ob1-pull needs PYTHONPATH")
- ❌ Task progress ("PR #123 merged")
- ❌ Session-specific state ("currently on step 3")
- ❌ Ephemeral observations ("it's raining")

### Step 2: Append to OBn

Use the `patch` tool to **surgically append** to `~/Documents/=notes/claude/memory/HERMES-MEMORY.md`.

- Add lines under the appropriate existing section header
- Create a new section header if no existing one matches
- **Never overwrite the whole file** — other agents and manual edits coexist
- Keep each fact to one line, declarative, no imperative mood

### Step 3: Upsert to OB1

If `BRAIN_KEY` is set, use the OB1 MCP `capture_thought` endpoint directly:

```python
import json, urllib.request, os

OB1_URL = "https://jhpuctiyosazlyrcnfuu.supabase.co/functions/v1/open-brain-mcp"
key = os.environ["BRAIN_KEY"]

for fact in durable_facts:
    req = urllib.request.Request(
        OB1_URL,
        data=json.dumps({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {
                "name": "capture_thought",
                "arguments": {
                    "content": fact,
                    "task_id": f"hermes/{namespace}",  # e.g., hermes/durable
                    "source": "hermes"
                }
            }
        }).encode(),
        headers={"Content-Type": "application/json", "x-brain-key": key},
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=30) as resp:
        # Parse SSE response to confirm capture
        pass
```

Namespace conventions (passed as `task_id`):
- `hermes/durable` — facts surviving across sessions
- `hermes/correction` — user corrections (high priority)
- `hermes/environment` — infra facts (hosts, ports, paths)

### Step 4: Post drain summary to #bots

Send a one-line summary to Discord #bots:

```
🧠 Drain: <1-sentence summary of what was persisted>
```

### Step 5: Verify persistence

Re-read the OBn file to confirm the append survived. If OB1 was written, query it back:

```bash
python3 ~/Documents/=notes/bin/ob1-pull --query "<fact-keyword>" --limit 1
```

## Pitfalls

| Pitfall | Prevention |
|---------|------------|
| OB1 MCP timeout | Auto-brief degrades gracefully — OBn local file always works |
| Stale OB1 data | Use `--recent --limit` with small limits; topic queries narrow scope |
| OBn race conditions | Append-only writes; never overwrite; use surgical `patch` |
| Too much context injection | Keep briefings ≤15 lines; respect session context budget |
| Session end drain skipped | AGENTS.md ritual mandates drain before "Stop" |
| OB1 upsert fails silently | Always verify by re-querying after upsert |
| `BRAIN_KEY` not in environment | Check env var first; skip OB1 ops gracefully with a note |
| `ob1-pull` not on PATH | Full path: `python3 ~/Documents/=notes/bin/ob1-pull` |

## Integration with Other Skills

- **session-briefing**: OB1 auto-brief should run *after* the daily note read, as part of the session startup ritual
- **agent-achievement-snapshots**: Auto-drain should run *before* the PDF audit trail, so persistent facts are captured first
- **coordination-folder-workflow**: If a session produces a coordination note, drain the key decision to OB1/OBn as well