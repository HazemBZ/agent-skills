---
name: project-pulse
description: Project-scoped monitoring and telemetry layer for phased delivery. Adds timestamps, event logs, webhook broadcasts, and export surfaces to project_state.json — enabling dashboards and cross-project visibility.
license: MIT
metadata:
  author: sisyphus
  version: "1.0.0"
  audience: agent
  workflow: monitoring
---

# Project-Pulse Skill

A sibling skill to `phase-tracking` — project-scoped, structured, and built for a
monitoring dashboard. Where `phase-tracking` manages *in-session* state, `project-pulse`
broadcasts it outward and records the full project lifecycle.

Load **both** skills side by side: `phase-tracking` writes the state, `project-pulse`
adds timestamps, identity, events, and export surfaces. They share `project_state.json`
and own different concerns.

## When to use

Use `project-pulse` alongside `phase-tracking` when:

- You want to **track progress across multiple projects** from a single dashboard
- You need **phase velocity metrics** (how long does each phase take?)
- You want **alerts on stalled phases or aging issues**
- You need a **structured event log** for compliance or post-mortems
- You want to **push project state to a collector** (via webhook or cron)
- You're building a **monitoring or reporting layer** on top of phased delivery

Do **not** use `project-pulse` alone — it reads `project_state.json` written by
`phase-tracking`. Always load `phase-tracking` first.

## Relationship with phase-tracking

| Concern | Owned by |
|---|---|
| State mutations (phase transitions, ADRs, issues) | `phase-tracking` |
| Schema v1.1 structured fields (timestamps, events, project_id) | Both |
| Webhook delivery | `project-pulse` |
| Export / snapshot | `project-pulse` |
| Schema validation + health checks | `project-pulse` |
| Monitoring dashboard ingest | `project-pulse` |

## Schema: `project_state.json` v1.1

The v1.1 schema extends `phase-tracking`'s v1.0 with **timestamps**, **structured
objects**, **project identity**, and an **event log**. Existing v1.0 files continue
to work — missing v1.1 fields produce warnings but no errors.

### Full schema

```json
{
  "schema_version": "1.1",
  "project_id": "my-project-slug",
  "project_name": "My Project",

  "current_phase": "Phase 3 — Auth & Permissions",
  "current_phase_started_at": "2026-06-10T08:00:00Z",

  "completed_phases": [
    {
      "name": "Phase 1 — Project scaffold & config",
      "started_at": "2026-06-01T09:00:00Z",
      "completed_at": "2026-06-03T17:30:00Z",
      "duration_hours": 32.5
    }
  ],

  "pending_phases": ["Phase 4 — API routes"],

  "architecture_decisions": [
    {
      "id": "ADR-001",
      "title": "Use row-level security with Postgres RLS",
      "status": "Accepted",
      "recorded_at": "2026-06-05T14:00:00Z"
    }
  ],

  "known_issues": [
    {
      "id": "ISS-001",
      "description": "Refresh token rotation not implemented",
      "severity": "medium",
      "status": "open",
      "opened_at": "2026-06-08T10:00:00Z"
    }
  ],

  "monitoring": {
    "webhook_url": "https://hooks.example.com/events",
    "webhook_secret": "whsec_...",
    "enabled_events": ["phase_started", "phase_completed", "issue_opened"]
  },

  "event_log": [
    { "type": "phase_started",     "phase": "Phase 1", "timestamp": "2026-06-01T09:00:00Z" },
    { "type": "phase_completed",   "phase": "Phase 1", "timestamp": "2026-06-03T17:30:00Z" },
    { "type": "adr_recorded",      "adr": "ADR-001",   "timestamp": "2026-06-05T14:00:00Z" }
  ],

  "next_steps": "Implement JWT verify middleware",
  "updated_at": "2026-06-13T10:00:00Z"
}
```

### Field reference

| Field | Type | Required in v1.1 | Notes |
|---|---|---|---|
| `schema_version` | string | yes | `"1.1"` for this schema |
| `project_id` | string | yes | Unique slug or UUID. Set once at init. |
| `project_name` | string | yes | Human-readable project name |
| `current_phase` | string | yes (inherited) | Same as v1.0 |
| `current_phase_started_at` | string (ISO 8601) | recommended | Set when starting a phase |
| `completed_phases` | array | yes (inherited) | Array of objects in v1.1 (strings also accepted) |
| `pending_phases` | array | yes (inherited) | Array of strings |
| `architecture_decisions` | array | yes (inherited) | Array of objects in v1.1 (strings also accepted) |
| `known_issues` | array | yes (inherited) | Array of objects in v1.1 (strings also accepted) |
| `monitoring` | object | optional | Webhook configuration block |
| `event_log` | array | recommended | Chronological list of transition events |
| `next_steps` | string | yes (inherited) | Same as v1.0 |
| `updated_at` | string (ISO 8601) | recommended | Last modification timestamp |

### Migration from v1.0

When loading an existing v1.0 `project_state.json`, the agent:

1. Detects missing v1.1 fields
2. Logs a warning listing which fields are absent
3. Prompts the user: *"This project uses the v1.0 schema. Add v1.1 fields (project_id, timestamps, event_log) for monitoring support? [y/N]"*
4. If yes, writes default values (`project_id` from repo name, `event_log: []`, etc.)
5. Proceeds in either case — no hard error

### Completed phases — object format

Each completed phase object:

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Phase title |
| `started_at` | ISO 8601 | recommended | When phase work began |
| `completed_at` | ISO 8601 | yes | When phase was marked complete |
| `duration_hours` | number | recommended | `(completed_at - started_at)` in hours |

If `started_at` is missing, `duration_hours` is omitted.

### Architecture decisions — object format

Each ADR object:

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | `"ADR-NNN"` — monotonic |
| `title` | string | yes | Short decision title |
| `status` | string | recommended | `"Proposed"`, `"Accepted"`, `"Deprecated"`, `"Superseded"` |
| `recorded_at` | ISO 8601 | recommended | When the ADR was recorded |

### Known issues — object format

Each issue object:

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | `"ISS-NNN"` — monotonic |
| `description` | string | yes | Actionable problem description |
| `severity` | string | yes | `"low"`, `"medium"`, `"high"`, `"critical"` |
| `status` | string | yes | `"open"`, `"in_progress"`, `"resolved"`, `"wontfix"` |
| `opened_at` | ISO 8601 | recommended | When the issue was created |
| `resolved_at` | ISO 8601 | nullable | When the issue was closed (null if open) |

## Event log

The `event_log` is a chronological array of project lifecycle events. It is the
**source of truth for monitoring** — every transition generates one event.

### Event types

| Type | Trigger | Payload fields |
|---|---|---|
| `phase_started` | A new phase begins | `phase`, `timestamp` |
| `phase_completed` | A phase is marked complete | `phase`, `timestamp`, `duration_hours` |
| `phase_skipped` | A phase is skipped/deprioritized | `phase`, `timestamp`, `reason` |
| `adr_recorded` | An ADR is created | `adr` (ID), `title`, `timestamp` |
| `issue_opened` | A known issue is created | `issue` (ID), `description`, `severity`, `timestamp` |
| `issue_resolved` | An issue is closed | `issue` (ID), `timestamp` |
| `project_created` | Project is initialized | `project_id`, `timestamp` |

### Event format

```json
{
  "type": "phase_completed",
  "phase": "Phase 2 — Database schema & migrations",
  "timestamp": "2026-06-08T17:30:00Z",
  "duration_hours": 32.5
}
```

### Ordering guarantees

- Events are appended in chronological order (newest at end)
- Timestamps should be monotonic — a new event MUST NOT have a timestamp older than the last event
- If the agent's clock cannot be trusted, use `updated_at` as the event timestamp

## Workflow

### 1. Initialize

Run this once when setting up a new project. Typically done by `phase-tracking`'s
Initialize step with `project-pulse` adding monitoring fields.

1. Ask the user for the complete list of phases.
2. Ask for `project_id` (default: repo directory name) and `project_name`.
3. Write `project_state.json`:
   - Set `schema_version: "1.1"`
   - Set `project_id` and `project_name`
   - Set `current_phase` to Phase 1, `current_phase_started_at` to now
   - `completed_phases: []`
   - All phases in `pending_phases`
   - `event_log: []`
   - `monitoring: {}` (no webhook by default)
   - Optionally configure a webhook if the user provides one
4. Append a `project_created` event to `event_log`.
5. Create the first todo breakdown for Phase 1 using `todowrite`.

After every `project_state.json` write, run the validation gate (see Health checks
section).

> **If `project_state.json` is missing**: treat as fresh start (same as `phase-tracking`).
> Prompt for `project_id` and `project_name` before creating the file.

### 2. Start a phase

When beginning work on a phase:

1. Read `project_state.json` to confirm current phase.
2. Set `current_phase_started_at` to the current timestamp (ISO 8601).
3. Break the phase into atomic tasks using `todowrite`.
4. Append a `phase_started` event to `event_log`.
5. Optionally fire the webhook (see Webhook Integration).
6. Mark the first todo `in_progress` and proceed.

### 3. Complete a phase

When all todos for the current phase are marked `completed`:

1. Calculate `duration_hours` from `current_phase_started_at` to now.
2. Move the phase to `completed_phases` as an object:
   ```json
   {
     "name": "Phase N — Title",
     "started_at": "<current_phase_started_at>",
     "completed_at": "<now>",
     "duration_hours": <calculated>
   }
   ```
3. Set `current_phase` to the next phase in `pending_phases` (shift from front).
4. Set `current_phase_started_at` to the current timestamp for the new phase.
5. Append a `phase_completed` event to `event_log`.
6. Optionally fire the webhook.
7. If no phases remain, set `current_phase` to `"Complete"` and `current_phase_started_at` to null.
8. Update `next_steps` with the next phase goal.
9. Run full project verification (all tests, lint, typecheck).

### 4. Record an ADR

When an important architectural decision is made:

1. Write the full ADR as a dedicated file at `docs/adr/NNN-title-of-decision.md` with
   context, decision, consequences, and status.
2. Append an object to `architecture_decisions`:
   ```json
   {
     "id": "ADR-NNN",
     "title": "Title of decision",
     "status": "Accepted",
     "recorded_at": "<now>"
   }
   ```
3. Append an `adr_recorded` event to `event_log`.
4. Increment the ADR number (maintain monotonic ordering).

If no `docs/adr/` directory exists, store the full ADR inline under
`architecture_decisions` as an object with a `details` field. Prefer dedicated files
once the project has more than 5 ADRs.

### 5. Manage known issues

When a problem is identified:

1. Assign the next `ISS-NNN` ID.
2. Append an object to `known_issues` with all required fields.
3. Append an `issue_opened` event to `event_log`.
4. Optionally fire the webhook.

When an issue is resolved:

1. Set `status` to `"resolved"` and `resolved_at` to now.
2. Append an `issue_resolved` event to `event_log`.
3. Optionally fire the webhook.

### 6. Session handoff

When ending a session mid-phase:

1. Ensure all v1.1 fields are set (timestamps, event_log entries appended).
2. Update `updated_at` to the current timestamp.
3. Update `next_steps` to precisely describe what comes next.

When **resuming** after a session gap:

1. Read `project_state.json` first. Check `schema_version` to decide which fields to expect.
2. If v1.0, offer to upgrade to v1.1 (see Migration from v1.0).
3. Create fresh todos from `next_steps` + `current_phase`.

## Webhook integration

Project-pulse can fire HTTP webhooks on state transitions. This enables real-time
integration with Slack, Datadog, CI systems, or a custom monitoring dashboard.

### Configuration

Add a `monitoring` block to `project_state.json`:

```json
"monitoring": {
  "webhook_url": "https://hooks.example.com/events",
  "webhook_secret": "whsec_your-secret-here",
  "enabled_events": ["phase_started", "phase_completed", "issue_opened"]
}
```

- `webhook_url` — where to POST events
- `webhook_secret` — used to sign the payload (HMAC-SHA256)
- `enabled_events` — which event types trigger a webhook (omit = all events)
- If `webhook_url` is absent or empty, no webhook is fired

### Payload format

```json
{
  "event": "phase_completed",
  "project_id": "my-project-slug",
  "project_name": "My Project",
  "phase": "Phase 2 — Database schema & migrations",
  "duration_hours": 32.5,
  "timestamp": "2026-06-08T17:30:00Z",
  "schema_version": "1.1"
}
```

### Delivery

Use `curl` with HMAC signing:

```bash
payload=$(cat <<'JSON'
{
  "event": "phase_completed",
  "project_id": "my-project-slug",
  "project_name": "My Project",
  "phase": "Phase 2 — Database schema & migrations",
  "duration_hours": 32.5,
  "timestamp": "2026-06-08T17:30:00Z",
  "schema_version": "1.1"
}
JSON
)
sig=$(echo -n "$payload" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | cut -d' ' -f2)
curl -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-Signature-256: $sig" \
  -d "$payload"
```

### Error handling

- **2xx response** → success. Log nothing.
- **4xx/5xx response** → log a warning, append to `known_issues` if repeated.
- **Timeout / network error** → retry up to 3 times with exponential backoff (1s, 2s, 4s).
- **Fire and forget** — the webhook is best-effort. The `event_log` is the authoritative
  record. State never depends on webhook delivery.

## Export & snapshot

These patterns push project state to a monitoring collector without needing a server
or daemon on the project side.

### Snapshot to archive

```bash
mkdir -p .progress
cp project_state.json ".progress/project_state-$(date +%Y%m%d-%H%M%S).json"
```

### Push to collector (curl)

```bash
curl -X POST https://monitor.example.com/ingest \
  -d @project_state.json \
  -H "Content-Type: application/json"
```

### Push to collector (Python)

```bash
python3 -c "
import json, requests
data = json.load(open('project_state.json'))
requests.post('https://monitor.example.com/ingest', json=data)
"
```

> The `requests` library is not always installed. Prefer `curl` when running in
> constrained environments.

### CI integration (GitHub Actions)

```yaml
- name: Push project state to monitor
  run: |
    curl -X POST ${{ secrets.MONITOR_URL }} \
      -d @project_state.json \
      -H "Content-Type: application/json"
```

### Cron pattern

```bash
# Every hour, push state to the monitoring collector
0 * * * * curl -X POST https://monitor.example.com/ingest \
  -d @/path/to/project/project_state.json \
  -H "Content-Type: application/json"
```

## Health checks & validation

Run these commands to verify `project_state.json` integrity.

### v1.1 validation gate

Run after every `project_state.json` write:

```bash
python3 -c "
import json, sys
d = json.load(open('project_state.json'))
assert d.get('schema_version'), 'schema_version is required'
assert d.get('project_id'), 'project_id is required'
assert d.get('project_name'), 'project_name is required'
assert isinstance(d.get('event_log'), list), 'event_log must be a list'
assert isinstance(d.get('completed_phases'), list), 'completed_phases must be a list'
assert isinstance(d.get('pending_phases'), list), 'pending_phases must be a list'
print('project-pulse: state is valid (v1.1)')
"
```

### Assertions that must hold

- `schema_version` is present and non-empty
- `project_id` is a non-empty string
- `completed_phases` and `pending_phases` have no overlapping phase names
- `completed_phases` is in chronological order by `completed_at`
- `pending_phases` is in chronological order
- ADR numbers are monotonic and sequential
- `current_phase` is never in both `completed_phases` and `pending_phases`
- Event log entries have `type`, `timestamp` and type-specific required fields
- No events reference phases not present in `completed_phases` or `pending_phases`
- Timestamps are valid ISO 8601 strings

### Health endpoint pattern

A monitoring system can validate `project_state.json` remotely. The expected response
from a simple health endpoint:

```json
{
  "status": "healthy",
  "project_id": "my-project-slug",
  "project_name": "My Project",
  "schema_version": "1.1",
  "current_phase": "Phase 3 — Auth & Permissions",
  "phase_count": 5,
  "completed_count": 2,
  "open_issues": 1,
  "last_updated": "2026-06-13T10:00:00Z"
}
```

This can be served by a static file host, a small Python HTTP server, or injected
into a Grafana JSON API data source.

## Integration with subagents

When delegating work for a `project-pulse`-monitored phase to a subagent, include
this context in the prompt:

```
CONTEXT: project-pulse monitoring active for [project_id].
State schema version: [1.0|1.1].
Webhook configured: [yes|no].
Current phase: [current_phase].
All state transitions MUST be recorded in event_log with ISO 8601 timestamps.
```

This keeps subagents aligned with the monitoring requirements even though they don't
have access to the full session history.

## Loading order

When using both skills, load `phase-tracking` **first** (it creates `project_state.json`),
then `project-pulse` **second** (it manages monitoring fields).

```typescript
// Correct loading order
// 1. phase-tracking — manages state mutations
// 2. project-pulse — manages monitoring, events, webhooks
```

No locking is needed — agents operate sequentially within a single session. The two
skills write to different sets of fields in the same JSON file.

## Monitoring dashboard concepts

Metrics a dashboard could surface from `project_state.json` data:

| Metric | How to compute | What it tells you |
|---|---|---|
| **Phase velocity** | `sum(duration_hours) / count(completed_phases)` per project | Average phase throughput |
| **Active vs completed** | Count of projects where `current_phase != "Complete"` vs `== "Complete"` | Overall delivery capacity |
| **ADR density** | `len(architecture_decisions) / len(completed_phases)` | Decision complexity per phase |
| **Issue aging** | Avg days since `opened_at` for issues where `status = "open"` | How long problems stay unresolved |
| **Project health** | Composite: stalled phase (>2x expected), aging issues (>30d), missed milestones | At-a-glance project risk |
| **Bottlenecks** | `avg(duration_hours)` grouped by phase name across projects | Which phases consistently take longest |

Suggested visualization tools:
- **Grafana** — JSON API plugin consumes `project_state.json` directly
- **Datadog** — custom metrics via webhook or API
- **Simple dashboard** — static HTML/JS reading the JSON file (zero dependencies)

## Verification gates

After every `project_state.json` write, run:

```bash
python3 -c "import json; data = json.load(open('project_state.json')); assert 'schema_version' in data; assert 'project_id' in data"
```

Then run the full v1.1 validation gate (see Health checks section above).
