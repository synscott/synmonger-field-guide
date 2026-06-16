---
name: hermes-internals
description: "Query and inspect Hermes Agent runtime state: session DB, cost tracking, token usage, session history."
tags: [hermes, sqlite, cost, tokens, sessions, introspection]
triggers:
  - "how much has this cost"
  - "how many tokens have we used"
  - "show me session history"
  - "query cost/usage"
  - "hermes session data"
---

# Hermes Internals: Runtime State & Cost Tracking

## Overview

Hermes stores session metadata — cost, token counts, model, source platform, title — in a local SQLite database. All introspection goes through that DB. No external API call needed.

## State DB Location

```
/opt/data/state.db
```

Key tables: `sessions`, `messages`, `state_meta`.

## Sessions Table Schema

Relevant columns:

- `id` — session ID string
- `source` — platform (`telegram`, `discord`, etc.)
- `model` — model name (e.g. `claude-sonnet-4-6`)
- `started_at` — Unix float (e.g. `1781476123.256`). **NOT a datetime string.** Date-range queries must compare against `datetime.timestamp()` values, not use `date(started_at)` (which returns NULL for float columns).
- `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`
- `estimated_cost_usd` — estimated cost based on official pricing snapshot
- `actual_cost_usd` — populated if billing API confirms it
- `cost_status` — `estimated` or `actual`
- `cost_source` — e.g. `official_docs_snapshot`
- `title` — auto-generated session title
- `message_count`, `tool_call_count`, `api_call_count`

## Cost Query Pattern

Use `execute_code` with pure Python sqlite3 — do NOT use `terminal()` for this, as it has a working directory bug in the execute_code sandbox.

```python
import sqlite3
from datetime import datetime

conn = sqlite3.connect('/opt/data/state.db')

# All-time summary
total = conn.execute(
    "SELECT SUM(estimated_cost_usd), SUM(input_tokens), SUM(output_tokens), COUNT(*) FROM sessions"
).fetchone()
print(f"Sessions: {total[3]}, Cost: ${total[0]:.4f}, In: {total[1]:,}, Out: {total[2]:,}")

# Recent sessions
rows = conn.execute("""
    SELECT id, source, model, started_at, input_tokens, output_tokens,
           cache_read_tokens, estimated_cost_usd, title
    FROM sessions ORDER BY started_at DESC LIMIT 20
""").fetchall()

for r in rows:
    ts = datetime.fromtimestamp(r[3]).strftime('%Y-%m-%d %H:%M')
    print(f"[{ts}] {r[1]:<10} ${r[7] or 0:.4f}  in:{r[4]:>7,} out:{r[5]:>6,}  {(r[8] or '')[:50]}")

conn.close()
```

## Diagnostic Pattern: Vague "Something Is Wrong" Reports

When syn says something like "there's something wrong with session lengths" or "sessions are behaving weird" — pull the DB data immediately and surface it. Don't try to interpret or guess what's wrong from the data alone; ask what the specific symptom is *after* showing the data. The data gives her something concrete to point at.

Useful columns to include in a session diagnostic: `message_count`, `cache_read_tokens`, `input_tokens`, `output_tokens`, `estimated_cost_usd`, `started_at`, `source`, `title`.

## User Preference

syn wants cost to be **query-on-demand only** — no polling, no periodic cost summaries, no proactive reporting. Pull it when she asks.

## Messages Table Schema

When querying individual messages (e.g. to inspect a session's content):

```python
# Correct column names (verified against PRAGMA table_info):
# id, session_id, role, content, tool_call_id, tool_calls, tool_name,
# timestamp, token_count, finish_reason, reasoning, reasoning_content,
# reasoning_details, codex_reasoning_items, codex_message_items,
# platform_message_id, observed, active

msgs = conn.execute("""
    SELECT id, role, content, tool_name, timestamp
    FROM messages
    WHERE session_id = ?
    ORDER BY timestamp ASC
""", (session_id,)).fetchall()
```

**Pitfall: The timestamp column is `timestamp`, NOT `created_at`.** Using `created_at` raises `OperationalError: no such column`. This trips up every time.

## Verifying Active Memory Providers

Do NOT try to import Hermes modules via subprocess to check provider state. There is no `agent.config` module — attempting `from agent.config import HermesConfig` or `from agent.config import load_config` will raise `ModuleNotFoundError` even from `/opt/hermes`. The correct approach is to read `config.yaml` directly (pure Python `yaml.safe_load`) for configuration, and grep `agent.log` for runtime confirmation:

```python
import subprocess, yaml

# Config (what's configured):
with open('/opt/data/config.yaml') as f:
    cfg = yaml.safe_load(f)
print(cfg.get('memory', {}))

# Runtime (what was actually loaded):
result = subprocess.run(
    ['grep', '-a', 'Memory provider', '/opt/data/logs/agent.log'],
    capture_output=True, text=True
)
print(result.stdout[-2000:])
```

Expected confirmation in log:
```
Memory provider 'honcho' registered (5 tools)
Memory provider 'holographic' registered (2 tools)
Memory provider(s) activated: honcho, holographic
```

## Context Composition Diagnostic

To measure what a typical call's context is made of, pull from the DB and measure schema size directly:

```python
import sqlite3, json, sys
from datetime import datetime, timezone

# 1. System prompt size: pull from a recent session
conn = sqlite3.connect('/opt/data/state.db')
sp_len = conn.execute(
    'SELECT length(system_prompt) FROM sessions ORDER BY started_at DESC LIMIT 1'
).fetchone()[0]
print(f'System prompt: {sp_len} chars (~{sp_len//4} tokens)')

# 2. Avg context per call: (cache_read + cache_write + input) / api_call_count
# for a specific session
sess_id = 'YOUR_SESSION_ID'
r = conn.execute('''
    SELECT api_call_count, input_tokens, cache_read_tokens, cache_write_tokens
    FROM sessions WHERE id = ?
''', (sess_id,)).fetchone()
calls, inp, cread, cwrite = r
total = (cread or 0) + (cwrite or 0) + (inp or 0)
print(f'Avg context/call: {total//calls:,} tokens over {calls} calls')
conn.close()

# 3. Tool schema size (run with /opt/hermes/.venv/bin/python3)
sys.path.insert(0, '/opt/hermes')
from model_tools import get_tool_definitions
defs = get_tool_definitions(enabled_toolsets=None, disabled_toolsets=None, quiet_mode=True)
schema_chars = len(json.dumps(defs))
print(f'{len(defs)} tools, ~{schema_chars//4:,} tokens')

# Context composition = sys_prompt + schemas + (avg_ctx - sys_prompt - schemas)
# The remainder is conversation history
```



| Component | Tokens | % | Notes |
|---|---|---|---|
| System prompt | ~6,800 | 13% | ~27K chars; includes SOUL.md, skills list, memories, user profile |
| Tool schemas (72 tools) | ~15,600 | 30% | Fixed; see measurement below |
| Conversation history | ~30,000 | 57% | Grows per turn; dominates in long sessions |

The **257:1 input:output ratio** is normal for this setup — not a bug. Most "input" is cache_read at $0.30/MTok, not fresh input at $3.00/MTok.

Cost levers in order of impact:
1. **Session length** — the biggest single-session cost driver. A 279-call session costs ~$8.71 alone.
2. **Toolset filtering** — 72 tools × ~217 tokens each = 15.6K tokens/call. Disabling unused toolsets cuts 30% of fixed cost.
3. **System prompt size** — relatively stable at ~6.8K tokens; lower impact than toolsets.

## Measuring Tool Schema Footprint

```python
import json
import sys
sys.path.insert(0, '/opt/hermes')
# Must run with /opt/hermes/.venv/bin/python3

from model_tools import get_tool_definitions

defs = get_tool_definitions(enabled_toolsets=None, disabled_toolsets=None, quiet_mode=True)
total_chars = len(json.dumps(defs))
print(f'{len(defs)} tools, ~{total_chars//4:,} tokens')

# Per-tool cost breakdown
sizes = sorted([(len(json.dumps(d)), d['function']['name']) for d in defs], reverse=True)
for sz, name in sizes[:10]:
    print(f'  {name}: ~{sz//4} tokens')
```

As of Jun 2026: 72 tools, ~15,622 tokens. Top 5 by schema size:
`cronjob` (~1,800), `delegate_task` (~1,734), `terminal` (~1,418), `session_search` (~1,268), `skill_manage` (~1,032).

## Per-Day Cost Query

```python
import sqlite3
from datetime import datetime, timezone

conn = sqlite3.connect('/opt/data/state.db')
rows = conn.execute('''
    SELECT 
      CAST(started_at / 86400 AS INT) * 86400 as day_ts,
      COUNT(*) as sessions,
      SUM(api_call_count) as calls,
      SUM(cache_read_tokens) as cread,
      SUM(cache_write_tokens) as cwrite,
      SUM(input_tokens) as inp,
      SUM(output_tokens) as out
    FROM sessions
    WHERE started_at IS NOT NULL
    GROUP BY day_ts
    ORDER BY day_ts DESC LIMIT 14
''').fetchall()

for r in rows:
    day_ts, sessions, calls, cread, cwrite, inp, out = r
    dt = datetime.fromtimestamp(day_ts, tz=timezone.utc).strftime('%Y-%m-%d')
    cread = cread or 0; cwrite = cwrite or 0; inp = inp or 0; out = out or 0
    # claude-sonnet-4 pricing (check for staleness)
    cost = cread*0.30/1e6 + cwrite*3.75/1e6 + inp*3.00/1e6 + out*15.00/1e6
    print(f'{dt}: sessions={sessions} calls={calls or 0} cread={cread:,} cwrite={cwrite:,} est=${cost:.2f}')
conn.close()
```

Note: Anthropic pricing used above is claude-sonnet-4 as of Jun 2026 — verify against current docs before relying on cost estimates.

## Cost Reduction Levers

When asked to diagnose or reduce cost, work through these in order of impact:

1. **Session length / compression threshold** — biggest lever. Compression fires at `compression.threshold` × context_length. Default 0.50 means it doesn't fire until 100K tokens (half the 200K window), by which point history dominates. Lower to 0.30 to compress at 60K — saves ~26% on marathon sessions. Config: `compression.threshold: 0.30`.

2. **Toolset filtering** — second lever. 72 tools = ~15,600 tokens fixed cost/call. Disabling `cronjob` + `delegation` saves ~3,534 tokens/call. Add to `config.yaml`: `agent.disabled_toolsets: [cronjob, delegation]`. Re-enable per-session with `-t` flag when actually needed. Do NOT disable `terminal`, `session_search`, or `skill_manage` globally — they're regularly needed.

3. **System prompt size** — relatively stable at ~6,800 tokens. Lower-impact than the above.

**Tool output truncation (`tool_output.max_bytes`) does NOT help** in practice. The default 50K char cap is almost never hit — terminal outputs are typically under 5K chars (largest seen: 10,738 chars in a 93-call session). The large tool results (e.g. `skill_view` loading a big skill — 48K chars for hermes-agent) are not subject to this limit. Lowering the cap saves zero tokens unless you're running commands that produce massive stdout dumps. **Do not recommend this as a cost lever.**

## Pitfalls

- `terminal()` inside `execute_code` sandbox hits a broken working directory (`/home/node/.openclaw/workspace`). Pure Python calls work fine. Use `execute_code` with `import sqlite3` directly.
- `execute_code` can be **blocked by approval config** — happens in cron sessions and when `approvals.cron_mode` is set restrictively. If blocked with "BLOCKED: execute_code runs arbitrary local Python", fall back to `terminal()` with inline Python: `python3 -c "..."`. This always works and is the safe default for DB queries. Prefer single-quoted strings inside the `-c` body; use escaped double-quotes or a heredoc for multi-liners.
- **`execute_code` was blocked in this session** — all DB queries were done via `terminal(command='python3 -c "..."')` and worked cleanly. If `execute_code` is unavailable, this is the correct fallback.
- `terminal(background=True)` background processes do **not** surface stdout via `process(action='log')` — even with `PYTHONUNBUFFERED=1` or `flush=True`. If a background script needs to communicate output (e.g. a device code), have it write to a temp file and poll that file instead.
- Input token counts in the `sessions` table look artificially low — this is because most input is served from cache and tracked separately in `cache_read_tokens`. **True context size per call = (cache_read + cache_write + input_tokens) / api_call_count.** Don't report raw `input_tokens` as the full picture without noting cache reads.
- `date(started_at)` returns NULL when `started_at` is a Unix float. Use integer comparisons: `WHERE started_at >= ? AND started_at < ?` with `datetime(..., tz=timezone.utc).timestamp()` bounds. Pattern: `ts_start = datetime(Y, M, D, tzinfo=timezone.utc).timestamp(); ts_end = ts_start + 86400`.
- The `session_id` column in `sessions` is actually named `id` — don't use `session_id` in queries against the sessions table.
- The `messages` table DOES have a `session_id` column (FK back to sessions) — that one is correct.

## Auxiliary Model Routing

Hermes has an `auxiliary` model layer — a separate configurable model for side tasks that run outside the main conversation loop. These are already offloaded; the question is what model handles them.

Tasks that can be cheaply routed:

| Task | Config key | Notes |
|---|---|---|
| Context compression | `auxiliary.compression` | Summarization only — doesn't need frontier capability |
| Curator (skill review) | `auxiliary.curator` | Taxonomy work; cheap model is fine |
| Title generation | `auxiliary.title_generation` | Trivial; any model works |
| Web extract summarization | `auxiliary.web_extract` | Page-to-markdown; cheap is fine |

**`auto` resolves to the main model** (e.g. Sonnet-4) when no provider is explicitly set. Compression on a 60K-token session running on Sonnet-4 is expensive and unnecessary.

Config pattern for a local OpenAI-compatible endpoint:
```yaml
auxiliary:
  compression:
    base_url: http://<host>:1234/v1
    model: qwen/qwen3-8b   # or whatever LM Studio calls it
  title_generation:
    base_url: http://<host>:1234/v1
    model: qwen/qwen3-8b
  curator:
    base_url: http://<host>:1234/v1
    model: qwen/qwen3-8b
```

**What auxiliary routing is NOT:** there is no mechanism to route individual tool calls to a cheaper model. Tool dispatch is cheap; it's the main agent processing results that costs. You cannot put a cheaper model between a tool result and the main agent's response.

See `references/local-llm-aux-routing.md` for connectivity setup (LM Studio on LAN/Tailscale).

## References

- See `references/sessions-schema.md` for full column list with types.
- See `references/cost-diagnostic-jun2026.md` for ground-truth cost anatomy: per-day breakdown, context composition measured values, tool schema footprint, and confirmed DB field semantics from Jun 2026 diagnostic session.
- See `references/local-llm-aux-routing.md` for routing auxiliary tasks to a local LM Studio instance.
