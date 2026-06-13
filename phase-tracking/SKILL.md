---
name: phase-tracking
description: Dual-layer phase planning and state management for multi-phase projects. Uses a project_state.json (persistent between sessions) combined with in-session todowrite/todoread for atomic task tracking. Handles initialization, phase advancement, ADR recording, and session handoff.
license: MIT
metadata:
  author: Sisyphus
  version: "1.0.0"
---

# Phase Tracking Skill

Two-layer planning & state management for projects built incrementally across multiple sessions.

- **Layer 1 — `project_state.json`**: Persistent JSON file at the project root. Survives between sessions. Records completed phases, current phase, pending phases, architecture decision records (ADRs), known issues, and next steps.
- **Layer 2 — `todowrite`/`todoread`**: In-session atomic task decomposition. Each phase is broken into granular todos that get marked `in_progress` → `completed` as work proceeds. Does not survive sessions — resets each time.

## When to use

Use this skill for any project that follows a phased development plan (e.g. "20 phases," "milestone-based delivery," "incremental build"). It keeps both the agent and the user aligned on what's done, what's next, and what decisions were made — across multiple sessions, multiple users, and any number of subagents.

## Artifacts

### `project_state.json` (project root)

The single source of truth for project progress. Schema:

```json
{
  "current_phase": "Phase N — Title",
  "completed_phases": [
    "Phase 1 — Title",
    "Phase 2 — Title"
  ],
  "pending_phases": [
    "Phase N+1 — Title"
  ],
  "architecture_decisions": [
    "ADR-001: Title of decision"
  ],
  "known_issues": [
    "Description of any known issues or regressions"
  ],
  "next_steps": "Brief description of what to start on next."
}
```

`known_issues` should be empty (`[]`) when clean. Every known issue should be a single actionable sentence.

#### Example

```json
{
  "current_phase": "Phase 3 — Auth & Permissions",
  "completed_phases": [
    "Phase 1 — Project scaffold & config",
    "Phase 2 — Database schema & migrations"
  ],
  "pending_phases": [
    "Phase 4 — API routes & controllers",
    "Phase 5 — Frontend integration"
  ],
  "architecture_decisions": [
    "ADR-001: Use row-level security with Postgres RLS",
    "ADR-002: JWT stored in httpOnly cookies, not localStorage"
  ],
  "known_issues": [
    "Refresh token rotation not yet implemented — tokens never expire after refresh"
  ],
  "next_steps": "Implement JWT verify middleware and attach user context to req"
}
```

## Workflow

### 1. Initialize

When starting a new phased project, create `project_state.json` at the project root:

1. Ask the user for the complete list of phases.
2. Write the file with `current_phase` set to Phase 1, `completed_phases` empty, and all phases in `pending_phases`.
3. Create the first todo breakdown for Phase 1 using `todowrite`.

After every `project_state.json` write, run `python3 -c "import json; json.load(open('project_state.json'))"` (or equivalent) to verify the file is valid JSON.

> **If `project_state.json` is missing**: treat this as a fresh start. Ask the user for the list of phases, re-create the file with `current_phase` set to Phase 1, `completed_phases` empty, and all phases in `pending_phases`. Prompt the user to confirm before overwriting if a legacy version is found elsewhere.

### 2. Start a phase

When beginning work on a phase (including the initial Phase 1):

1. Read `project_state.json` to confirm current phase.
2. Break the phase into atomic, completable tasks using `todowrite`. Each todo must follow the format:
   `"[file/module]: [action] to [why] — expect [result]"`
3. Mark the first todo `in_progress`.
4. Proceed through each todo: delegate to subagents in parallel where possible, mark `in_progress` before starting, `completed` immediately after finishing.
5. After each todo, run verification (lint, typecheck, tests) on changed files.

### 3. Complete a phase

When all todos for the current phase are marked `completed`:

1. Update `project_state.json`:
   - Move `current_phase` from `completed_phases` (append) to `completed_phases`.
   - Set `current_phase` to the next phase in `pending_phases` (shift from front).
   - If no phases remain, set `current_phase` to `"Complete"`.
   - Update `next_steps` with the next phase goal.
   - Clear `known_issues` or migrate unresolved ones forward.
2. Run full project verification (all tests, lint, typecheck).
3. Present a summary to the user: what was built, test results, any known issues.
4. Create todos for the next phase.

### 4. Record an ADR

When an important architectural decision is made:

1. Write the full ADR as a dedicated file at `docs/adr/NNN-title-of-decision.md` (or `./adr/` relative to the project root) with context, decision, consequences, and status.
2. Append a reference entry to `architecture_decisions` in `project_state.json`. Format: `"ADR-NNN: Title of decision"`.
3. Increment the ADR number (maintain monotonic ordering).
4. Note the ADR in any relevant todo or summary.

If the project does not have a `docs/adr/` directory, store the full ADR text inline under the `architecture_decisions` array as an object with a `details` field. The `project_state.json` approach trades formality for portability — prefer dedicated files once the project has more than 5 ADRs.

### 5. Session handoff (cross-session continuation)

When ending a session mid-phase:

1. Ensure `project_state.json` reflects the true current state.
2. Update `next_steps` to precisely describe what comes next (which file, what change, what to verify).
3. Record any `known_issues` that are unresolved.
4. Optionally write a brief handoff note — but `project_state.json` + the todo list is the canonical handoff.

When **resuming** after a session gap:

1. Read `project_state.json` first. This tells you the exact state.
2. Read `project_state.json`'s `completed_phases` and `architecture_decisions` — these are immutable unless explicitly amended.
3. Create fresh todos from `next_steps` + `current_phase`. Do NOT rely on stale in-session todos — they were lost at session end.
4. Proceed.

## Integration with subagents

When delegating work for a phase to a subagent, include a reference to the current phase and the relevant `project_state.json` context in the prompt:

```
CONTEXT: This is part of [current_phase]. project_state.json shows phases [completed_count] complete, [pending_count] remaining.
```

This keeps subagents aligned with the project's overall trajectory even though they don't have access to the full session history.

## Verification gates

After every `project_state.json` change:
```bash
python3 -c "import json; data = json.load(open('project_state.json')); assert 'current_phase' in data; assert 'completed_phases' in data; assert 'pending_phases' in data"
```

Assertions that must hold:
- No duplicate phase names across `completed_phases`, `current_phase`, and `pending_phases`.
- `completed_phases` is in chronological order.
- `pending_phases` is in chronological order.
- ADR numbers are monotonic and sequential.
- `current_phase` is never in both `completed_phases` and `pending_phases` simultaneously.
