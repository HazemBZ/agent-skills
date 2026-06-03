---
name: todo-workflow
description: A structured workflow for tracking multi-step migration and refactoring tasks via a persistable TODOS.md file.
---

# TODO Workflow Skill

A structured workflow for tracking multi-step migration and refactoring tasks via a persistable `TODOS.md` file.

## When to use

Use this skill whenever you are performing a sequence of related codebase changes (migrations, refactors, feature builds) that span multiple steps over multiple sessions. It keeps both the agent and the user aligned on what's done and what's next.

## Workflow Artifacts

Before starting a new workflow, any prior completed workflow is archived under `artifacts/todo-workflow-<timestamp>/` so the history is preserved. The archive contains:
- `TODOS.md` — final task list snapshot
- `ERRORS.md` — full error log
- `COMMITS.md` — log of commits made during the workflow

This lets multiple workflow sessions run sequentially without losing the record of what each session accomplished.

## Workflow

### 1. Initialize the list

Before creating or resetting `TODOS.md`, check if a previous workflow exists:

```bash
test -f TODOS.md && grep -q '\[x\]' TODOS.md && echo "Has completed tasks"
```

**If a prior completed workflow exists:**

1. Create a timestamped archive:
   ```bash
   ts=$(date +%Y%m%d-%H%M%S)
   mkdir -p "artifacts/todo-workflow-$ts"
   cp TODOS.md "artifacts/todo-workflow-$ts/TODOS.md"
   cp ERRORS.md "artifacts/todo-workflow-$ts/ERRORS.md"
   git log --oneline -20 -- TODOS.md ERRORS.md > "artifacts/todo-workflow-$ts/COMMITS.md"
   ```
2. Stage and commit the archive:
   ```bash
   git add "artifacts/todo-workflow-$ts/" TODOS.md ERRORS.md
   git commit -m "archive: snapshot completed workflow $ts"
   ```
3. Then reset `TODOS.md` for the new workflow.

**If no prior workflow exists**, create `TODOS.md` fresh:

```markdown
# Migration Tasks

- [ ] Task one
- [ ] Task two
- [ ] Task three
```

### 2. Keep both representations in sync

Every time you call the `todowrite` tool, immediately update `TODOS.md` to match. The two lists MUST be identical at all times — never let them drift.

Sync rules:
- When starting a task → `in_progress` + `- [ ]` → `- [/]`
- When completing a task → `completed` + `- [ ]` → `- [x]`
- When adding new tasks → add to both
- When reordering → reorder both identically

### 3. Log errors at task completion

After completing each task, log errors encountered in `ERRORS.md` at the project root. Use this format:

```markdown
### Task N: Title
- **Date:** YYYY-MM-DD
- **Error:** Brief description of what went wrong
- **Root cause:** What caused it
- **Resolution:** How it was fixed
- **Status:** Resolved / Workaround / Unresolved / No errors encountered
```

If no errors occurred, write `No errors encountered` and set status to `No errors encountered`. Add this instruction to `AGENTS.md`:

```markdown
## Errors File

The file `ERRORS.md` at the project root logs errors encountered during task execution. After completing each task, the agent MUST record any errors that occurred, including root cause and resolution.
```

### 4. Reference the files

Add this instruction to the project's `AGENTS.md` so the agent automatically keeps the file in sync:

```markdown
## TODOs File

The file `TODOS.md` at the project root tracks all outstanding and completed tasks. The agent MUST keep `TODOS.md` in sync with its in-session todo list whenever tasks progress.
```

### 5. Commit after every task

After completing each task (and logging errors to `ERRORS.md`), make a git commit. The commit must include `TODOS.md` and `ERRORS.md` alongside the code changes.

Commit workflow:
1. Run `pnpm run build` to verify no broken imports
2. Run `git status` to check all new/edited files are tracked
3. Stage everything: `git add -A`
4. Commit with a descriptive message referencing the task number and title

Add this instruction to `AGENTS.md` under the TODOs section:

```markdown
## Git & Commit Hygiene

Before every commit, the agent MUST run these checks:

1. **`git status`** — Verify all new files (directories, components, mocks) are staged. Untracked files (`??`) that are imported elsewhere will break CI.
2. **`pnpm run build`** — Confirm the production build succeeds with no `Could not resolve` errors. A local build pass guarantees the module graph is complete.
3. **Stage all related artifacts** — When adding a new feature directory (e.g., `features/MyFeature/`), ensure every file inside it is staged, not just the files that import it.
4. **Remove dead code** — If a directory is replaced (e.g., `FeaturesHub` → `FeatureLayout`), delete the old directory so it doesn't linger as untracked dead weight.
```

### 6. Reflect & generate follow-ups

After completing all tasks, run a reflection pass to identify leftover concerns that the migration may have introduced or ignored.

#### 6a. Survey for post-migration concerns

Check these categories systematically:

| Category | What to look for |
|---|---|
| **Stale references** | Old path aliases in `tsconfig.json` / `jsconfig.json`, outdated `README.md` docs, aged comments |
| **Dead dependencies** | Packages in `package.json` no longer imported anywhere (`rg` for each dep name across `src/`) |
| **Missing files** | Things imported by source code that don't exist (e.g., `src/lib/utils.js` for shadcn's `cn()`) |
| **Doc/impl gaps** | Components documented in `AGENTS.md` with no backing file, or files with no docs |
| **Lint/CI gaps** | `pnpm run lint` fails, missing ESLint config, test runner absent |
| **Unused patterns** | Packages that conflict with project conventions (e.g., `formik` installed but convention says `react-hook-form`) |
| **Version mismatches** | Deprecated APIs, stale shadcn registries, Tailwind v3 patterns in a v4 project |

Default reflection checks:
```bash
# 1. Stale path aliases
rg '@design-system' tsconfig.json 2>/dev/null || echo "Clean"

# 2. Dead deps — quick scan of package.json deps not imported in src/
# (run manually per dep or use a tool like depcheck)

# 3. Missing files imported by source
rg "from ['\"]@/lib/utils['\"]" src/components/ui/ | head -1
test -f src/lib/utils.js && echo "exists" || echo "MISSING"

# 4. Broken lint
pnpm run lint 2>&1 | tail -5

# 5. Build
pnpm run build 2>&1 | tail -5
```

#### 6b. Present considerations to the user

For each concern found, present it to the user as a **consideration**, not a demand. Example:

> **Consideration:** `tsconfig.json` still has `@design-system` path aliases (lines 32-33). This won't break anything since the directory is gone, but it could mask future import errors.
> 
> **Consideration:** `README.md` still references `@design-system/ui/` imports. Anyone cloning the template gets wrong docs.
> 
> **Consideration:** `formik` is installed but the project convention is `react-hook-form`.

#### 6c. Translate into new tasks (with user approval)

For each consideration the user wants to act on:

1. Add a new pending task to `TODOS.md`.
2. Add the corresponding section to `ERRORS.md` with status `Pending`.
3. Proceed through the workflow again (start → commit → reflect) until the list is empty.

If the user declines all considerations, mark the session complete:

1. **Produce the terminal archive** for the next workflow initialization to pick up:
   ```bash
   ts=$(date +%Y%m%d-%H%M%S)
   mkdir -p "artifacts/todo-workflow-$ts"
   cp TODOS.md "artifacts/todo-workflow-$ts/TODOS.md"
   cp ERRORS.md "artifacts/todo-workflow-$ts/ERRORS.md"
   git log --oneline -999 --skip=1 > "artifacts/todo-workflow-$ts/COMMITS.md"
   ```
2. **Stage everything** including the archive, `TODOS.md`, and `ERRORS.md`.
3. **Commit** with a terminal message like `"Migration complete"`.
4. The archived `TODOS.md` and `ERRORS.md` remain as historical records for the next session's artifact check.
