# Session lifecycle and orphan cleanup

This page covers how to inventory the agent sessions tracked in the MCP server's DuckDB control plane, identify ones that have been orphaned, and clean them up.

## Why orphans happen

A *session* is a row in the `agent_sessions` table, created when an MCP client calls `ensure_agent_session` (directly, or via the `zulipchat-mcp-hook` SessionStart hook). It binds one MCP client process to one Zulip topic and stays at `status = "active"` until something explicitly closes it via `close_agent_session`.

Sessions become orphaned when the client process exits without calling close — terminal closed, host shut down, SIGKILL, crash. The row stays `active`, no one is polling for it, but the rest of the system can't tell the difference between "live but quiet" and "gone."

Nothing breaks immediately, but over time:

- `list_sessions` shows orphans as live, which is misleading
- Outstanding `agent_requests` tied to them never resolve
- `session_events` accumulates event history for sessions no one reads

Periodic inventory and cleanup is operational hygiene, not a correctness requirement.

## Running the queries

The DuckDB control-plane database lives at the path given by the `ZULIPCHAT_DB_PATH` environment variable (default `.mcp/zulipchat/zulipchat.duckdb` relative to the server's working directory). All the `python3 -c "..."` snippets below assume you're running them in an environment that can reach that file — directly, via `uv run python3`, `docker exec`, an SSH shell on the host, etc. Adapt the invocation to wherever your DB lives.

## Listing sessions

### Via MCP tool (preferred)

From any MCP client connected to the server:

```python
list_sessions(include_closed=False)   # active only
list_sessions(include_closed=True)    # everything ever bound
```

### Via direct DuckDB query

When the MCP server isn't reachable, or you want SQL flexibility:

```bash
python3 -c "
import duckdb, os
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'], read_only=True)
print(conn.execute('''
    SELECT
        session_id,
        host,
        topic_name,
        project_dir,
        status,
        created_at,
        updated_at,
        ended_at
    FROM agent_sessions
    ORDER BY updated_at DESC
''').fetchdf())
"
```

### Quick "what's active right now?" one-liner

Useful for monitoring shells and status scripts:

```bash
python3 -c "
import duckdb, os
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'], read_only=True)
rows = conn.execute(
    '''SELECT topic_name, host, updated_at FROM agent_sessions
       WHERE status = \"active\" ORDER BY updated_at ASC'''
).fetchall()
for topic, host, ts in rows:
    print(f'{ts.isoformat()} {host:14s} {topic}')
"
```

## Identifying orphans

Three independent signals. Any one is suspicious; together they're conclusive.

### 1. The host is gone

`agent_sessions.host` is the hostname (`socket.gethostname()`) of the process that bound the topic. What "gone" looks like depends on how the MCP client runs:

- Containerised clients: `host` is the container's short ID. Check with `docker ps -a` (or `podman ps -a`); if it's not listed, the agent is gone.
- Direct processes: `host` is the machine hostname, plus you'd typically look for a corresponding live `uvx`/`python` process on that host.

For long-quiet deployments the host check on its own can be misleading — the same hostname may be reused across reboots — so combine it with signals 2 and 3 before acting.

### 2. `updated_at` is stale

`updated_at` only advances when the session is touched (rebind, poll, agent_message, status change). A live session in steady operation touches it regularly. An `updated_at` from days ago without an `ended_at` is almost certainly orphaned.

### 3. No recent events

A joined query that surfaces orphans by sorting active sessions oldest-first alongside their event activity:

```bash
python3 -c "
import duckdb, os
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'], read_only=True)
print(conn.execute('''
    SELECT
        s.session_id,
        s.host,
        s.topic_name,
        s.updated_at,
        (SELECT MAX(created_at) FROM session_events e
            WHERE e.session_id = s.session_id) AS last_event_at,
        (SELECT COUNT(*) FROM session_events e
            WHERE e.session_id = s.session_id) AS event_count
    FROM agent_sessions s
    WHERE s.status = 'active'
    ORDER BY s.updated_at ASC
''').fetchdf())
"
```

A session with `event_count = 1` (just the initial outbound *session bound* announcement) and a `last_event_at` from days ago has never been driven and is safe to close.

## Cleaning up

### Single session via MCP tool (recommended for one-offs)

```python
# Quiet close — flips status to "cancelled" and sets ended_at, no Zulip post.
close_agent_session(
    session_id="00000000-0000-0000-0000-000000000000",
    status="cancelled",
)

# With a closing message visible to the topic owner.
close_agent_session(
    session_id="00000000-0000-0000-0000-000000000000",
    status="cancelled",
    summary="Closed by operator — bound session host no longer running.",
)
```

The `status` argument should be one of `completed`, `failed`, or `cancelled`. The database manager automatically sets `ended_at` whenever status moves to one of those terminal values, so row lifecycle stays consistent without manual fields.

### Bulk cleanup via direct SQL

For pruning many stale sessions at once. The cutoff below is 7 days; tune to your operational tolerance.

```bash
python3 -c "
import duckdb, os
from datetime import datetime, timezone, timedelta
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'])
cutoff = datetime.now(timezone.utc) - timedelta(days=7)
now = datetime.now(timezone.utc)
before = conn.execute(
    'SELECT COUNT(*) FROM agent_sessions WHERE status = ? AND updated_at < ?',
    ['active', cutoff],
).fetchone()[0]
conn.execute(
    '''UPDATE agent_sessions
       SET status = ?, ended_at = ?, updated_at = ?
       WHERE status = ? AND updated_at < ?''',
    ['cancelled', now, now, 'active', cutoff],
)
conn.commit()
print(f'Closed {before} stale sessions')
"
```

Real interactive sessions touch `updated_at` on every poll, message, or rebind, so a 24-hour threshold is usually conservative enough. Use a longer window if you have sessions that legitimately go quiet for days.

## Adjacent tables

| Table | What it holds | When to clean |
|---|---|---|
| `agent_profiles` | Stable agent identity (one row per agent name + owner + agent type). Long-lived, deliberately persistent. | Almost never. Profiles outlive sessions by design. |
| `agent_requests` | Pending `request_user_input` approval prompts. Has its own `status` (`pending` / `answered` / `timed_out`). | Closing a session does **not** auto-cancel its outstanding requests. Worth pruning when you accumulate stale `pending` rows. |
| `session_events` | Append-only event log — every poll, every message, both directions. Grows unbounded. | Periodic pruning when size becomes a concern. Joinable on `session_id` for trail-of-evidence queries. |

### Pruning event history for long-closed sessions

```bash
python3 -c "
import duckdb, os
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'])
result = conn.execute('''
    DELETE FROM session_events
    WHERE session_id IN (
        SELECT session_id FROM agent_sessions
        WHERE ended_at IS NOT NULL
          AND ended_at < CURRENT_TIMESTAMP - INTERVAL 30 DAY
    )
''')
conn.commit()
print(f'Pruned {result.rowcount} event rows from sessions ended >30d ago')
"
```

### Cancelling abandoned input requests

```bash
python3 -c "
import duckdb, os
from datetime import datetime, timezone
conn = duckdb.connect(os.environ['ZULIPCHAT_DB_PATH'])
now = datetime.now(timezone.utc)
result = conn.execute('''
    UPDATE agent_requests
    SET status = ?, responded_at = ?
    WHERE status = ?
      AND session_id IN (
          SELECT session_id FROM agent_sessions WHERE ended_at IS NOT NULL
      )
''', ['cancelled', now, 'pending'])
conn.commit()
print('Cancelled pending requests on closed sessions')
"
```

## Related docs

- [Agents API reference](../api-reference/agents.md)
- [Troubleshooting](../TROUBLESHOOTING.md)
