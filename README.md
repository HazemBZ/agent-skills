# agent-skills

A collection of reusable agent skills. Each top-level folder is a self-contained skill that any agent can load by reading its `SKILL.md`.

## Available skills

| Skill | What it does | Best for |
|---|---|---|
| [`todo-workflow`](./todo-workflow/SKILL.md) | Tracks multi-step migrations/refactors via a persisted `TODOS.md` + `ERRORS.md` workflow with inter-session archival. | Long-running refactors, migrations, and feature builds that span multiple sessions. |
| [`ui-migration`](./ui-migration/SKILL.md) | Tracks component provenance (shadcn / custom / wrapper / legacy) during UI library migrations. | Migrating a project between UI libraries (e.g. legacy design system → shadcn/ui). |

## Installation

Each skill is installed into `~/.agents/skills/` (the standard agent skills directory). Pick the install method that fits your workflow.

### A. Via `npx skills` (recommended)

The [`skills`](https://www.npmjs.com/package/skills) CLI handles cloning, copying, and symlinking in one step. This is the right choice when you just want to use a skill.

```bash
# Install a single skill
npx skills add HazemBZ/agent-skills@todo-workflow
npx skills add HazemBZ/agent-skills@ui-migration

# Or install all of them
npx skills add HazemBZ/agent-skills
```

Add the `-g` flag for a global (user-level) install and `-y` to skip confirmation prompts:

```bash
npx skills add HazemBZ/agent-skills@todo-workflow -g -y
```

### B. Clone + symlink (best for local development)

Use this when you want edits in the repo to be live in your agent without re-running the CLI. It works well if you plan to contribute back or maintain your own fork.

```bash
git clone https://github.com/HazemBZ/agent-skills.git ~/code/agent-skills
ln -s ~/code/agent-skills/todo-workflow ~/.agents/skills/todo-workflow
ln -s ~/code/agent-skills/ui-migration  ~/.agents/skills/ui-migration
```

After this, `git pull` in `~/code/agent-skills` updates your installed skills automatically.

### C. Plain copy (one-off, no live updates)

Use this if you want a frozen snapshot.

```bash
git clone https://github.com/HazemBZ/agent-skills.git /tmp/agent-skills
cp -r /tmp/agent-skills/todo-workflow ~/.agents/skills/
cp -r /tmp/agent-skills/ui-migration  ~/.agents/skills/
rm -rf /tmp/agent-skills
```

## Usage

Once installed, the agent's skill loader picks up skills from `~/.agents/skills/` automatically — you don't need to "register" them. To use a skill mid-session, ask the agent to load it (the exact phrasing depends on your agent client; e.g. in opencode, skills are listed in the system prompt).

## Adding a new skill

Each skill lives in its own top-level directory and is just a `SKILL.md` file. The repo layout is intentionally minimal so it's easy to fork and extend.

### 1. Create the directory and author `SKILL.md`

```bash
cd ~/code/agent-skills
mkdir my-skill
$EDITOR my-skill/SKILL.md
```

A good `SKILL.md` follows this skeleton:

```markdown
# My Skill Name

One-line summary of what the skill does.

## When to use

Describe the trigger conditions: what kind of task should the agent load this skill for?
Be specific. Examples help: "When migrating a React project from Formik to react-hook-form".

## Workflow / Conventions

The body of the skill. Step-by-step instructions, conventions, examples, and any
guardrails the agent should follow while executing the task.
```

Conventions to follow:
- File name: exactly `SKILL.md` (capital `SKILL`, lower `.md`).
- No code, scripts, or extra assets in the skill folder unless the skill explicitly needs them.
- Use H1 for the skill title, H2 for major sections.
- Keep the document scannable — agents load it into context, so brevity matters.

### 2. Add it to this README

Append a row to the **Available skills** table above so newcomers can find it.

### 3. Commit and push

```bash
git add my-skill/
git commit -m "feat(skill): add my-skill"
git push
```

Anyone can now install it with:

```bash
npx skills add HazemBZ/agent-skills@my-skill
```

## Updating a skill

- **Symlink installs** (method B): `cd ~/code/agent-skills && git pull`. The symlinked folder updates automatically.
- **`npx skills` installs** (method A): re-run `npx skills add HazemBZ/agent-skills@<skill> -g -y` to overwrite, or run `npx skills check && npx skills update`.

When you change a skill's behavior, bump a short note in this README under that skill's row so users know what's new.

## License

[MIT](./LICENSE) © 2026 HazemBZ.
