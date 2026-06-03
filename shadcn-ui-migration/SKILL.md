---
name: shadcn-ui-migration
description: A step-by-step playbook for migrating a React project from a hand-rolled/custom design system to shadcn/ui. Covers Tailwind v3→v4, shadcn init, component-by-component replacement, theme reconciliation, lint recovery, and cleanup.
---

# shadcn/ui Migration Skill

A step-by-step playbook for migrating a React project from a hand-rolled/custom design system to shadcn/ui. Covers Tailwind v3→v4, shadcn init, component-by-component replacement, theme reconciliation, lint recovery, and cleanup.

## When to use

Use this skill when:
- Migrating from a custom `@design-system` directory to shadcn/ui
- Upgrading Tailwind CSS from v3 to v4 alongside a shadcn migration
- Replacing hand-rolled UI components (Button, Card, Input, Dialog, Select, etc.) with shadcn equivalents
- Cleaning up after a migration (dependencies, docs, stale aliases)

## Dependencies

This skill builds on two companion skills in this repository:

| Skill | Purpose |
|---|---|
| [`ui-migration`](../ui-migration/SKILL.md) | Component provenance tracking — catalog what was shadcn, custom, or legacy |
| [`todo-workflow`](../todo-workflow/SKILL.md) | Multi-step task tracking via `TODOS.md` + `ERRORS.md` with commit-after-each-task discipline |

If both are loaded, after migration each component move is recorded in `COMPONENT_CATALOG.md`, and tasks are tracked in `TODOS.md`.

---

## Phase 0: Preflight Checks

Before touching any component:

1. **List every component** in the old design system directory. Note which are actually imported. Use `rg` across `src/` for import strings.
2. **Check `package.json`** for every dependency imported by the old system — these may become removable.
3. **Run both `build` and `lint`** before starting. Record the baseline. A broken linter will hide regressions introduced during migration.
4. **Read the project's `AGENTS.md`** — it likely documents phantom components that never existed.
5. **Note the CSS framework version.** Tailwind v3 and v4 require completely different approaches.

---

## Phase 1: Tailwind v3 → v4 (if applicable)

### Key differences

| v3 | v4 |
|---|---|
| `tailwind.config.js` | `@theme` directives in CSS |
| PostCSS plugin | Native Vite plugin (`@tailwindcss/vite`) |
| `@apply` still works | `@apply` still works |
| `darkMode: "class"` | `@custom-variant dark (&:is(.dark *));` |

### Steps

1. **Remove PostCSS**: delete `postcss.config.js`, uninstall `postcss` and `autoprefixer`.
2. **Install Tailwind v4 + Vite plugin**: `pnpm add tailwindcss @tailwindcss/vite`
3. **Update `vite.config.js`**: add `tailwindcss()` to plugins list.
4. **Migrate config to CSS**: copy all `theme.extend.colors` etc. into CSS `@theme { ... }` block.
5. **Remove `tailwind.config.js`** entirely.
6. **Remove `@tailwind base/components/utilities`** directives — Tailwind v4 auto-injects these.
7. **Install `tw-animate-css`** if you need animation utilities (shadcn needs them).

### Common pitfall

Tailwind v4's `@theme` uses `--color-*` variable naming. Old config `colors: { primary: { 500: '#3b82f6' } }` becomes:

```css
@theme {
  --color-primary-500: #3b82f6;
}
```

Test that `text-primary-500` still resolves after migration.

---

## Phase 2: shadcn Init

1. **`pnpm dlx shadcn@latest init --defaults`** — creates `components.json`, `src/lib/utils.ts` (with `cn()`), and `src/components/ui/`.
2. **Check `components.json`** — ensure `paths.ui` resolves to `src/components/ui`, not `src/*/components/ui`.
3. **If shadcn created `src/*/components/ui/` with a literal `*` directory**, move files up and remove the asterisk dir.
4. **Update `components.json`** to the correct aliases.
5. **Ensure `package.json` has `"type": "module"`** — shadcn's generated files use ESM imports.

### Known bug

shadcn may install via `npx` and not respect `pnpm`. If you get dependency errors, run the pnpm equivalent manually:

```bash
pnpm add class-variance-authority clsx tailwind-merge lucide-react @radix-ui/react-slot
```

---

## Phase 3: Component-by-Component Migration

### Ordering

Replace in dependency order: `Button → Card → Input → Select → Dialog → DropdownMenu → Sheet → ...`

### General pattern

For each old component:

1. **Install the shadcn equivalent**: `pnpm dlx shadcn@latest add <name>`
2. **Read the generated file** to understand the API.
3. **Find all callers**: `rg "from '.*old-component'" src/`
4. **Replace each caller** with the new import path and API.
5. **Delete the old component file.**
6. **Run build + lint** before moving to the next component.

### Component-specific notes

#### Button
- shadcn Button uses `class-variance-authority` for variants.
- No built-in `loading` — compose with `disabled` + a spinner icon.
- Old `icon` / `iconPosition` props: replace by manually placing `<Icon>` children inside Button.

#### Card
- shadcn Card is a pure Tailwind + `cn()` composition: `Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardAction`, `CardContent`, `CardFooter`.
- No `title` prop on Card — compose `CardHeader + CardTitle` manually.

#### Input / Textarea
- shadcn Input is a primitive `<input>`. No wrappers for "with unit" or "search" styling.
- For search: compose `div.relative > SearchIcon + Input.pl-8`.
- `InputNumber`, `InputSearch`, `InputWithUnit` were custom wrappers — decide if they're needed or inline composition suffices.

#### Select / Dropdown
- shadcn **Select** replaces custom single-select use cases.
- shadcn **DropdownMenu** replaces for menu/action use cases (click-to-open menu with items).
- Old MultiSelect: shadcn has none — use Checkbox groups or a dedicated library.

#### Dialog
- Wraps `@radix-ui/react-dialog`. Exports: `Dialog`, `DialogTrigger`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogDescription`, `DialogFooter`.
- Pass `open` + `onOpenChange` for controlled usage.

#### Table
- Pure Tailwind + `cn()`. Composable: `Table`, `TableHeader`, `TableBody`, `TableRow`, `TableHead`, `TableCell`.
- No built-in sort/filter — keep existing state management.

#### Badge
- Uses `class-variance-authority`. Variants: `default`, `secondary`, `destructive`, `outline`, `ghost`, `link`.
- Map old status/role classes to variants; use `className` for custom colors.

#### Avatar
- Composes: `Avatar`, `AvatarImage`, `AvatarFallback`, `AvatarBadge`.
- Replace hand-rolled initials div with `AvatarFallback`.

#### Breadcrumb
- Composable: `Breadcrumb`, `BreadcrumbList`, `BreadcrumbItem`, `BreadcrumbLink`, `BreadcrumbPage`, `BreadcrumbSeparator`.
- `BreadcrumbLink` uses the `render` prop pattern.

#### Tooltip
- Must wrap in `TooltipProvider` at the app root.
- Composition: `Tooltip > TooltipTrigger + TooltipContent`.

#### Sheet
- Wraps `@radix-ui/react-dialog` for slide-out panels. Props: `side` (top/right/bottom/left).

#### Sonner (Toasts)
- Import `Toaster` from `@/components/ui/sonner`, mount once in app root.
- Use `toast.success/error/info/loading` from `sonner` directly.
- ~16 KB savings vs `react-toastify`.

---

## Phase 4: The `render` Prop

Many shadcn components accept a `render` prop instead of `asChild`:

```jsx
// ❌ Wrong — nested <button> inside <a>
<Button render={<Link to="/dashboard" />}>Dashboard</Button>

// ✅ Correct — render replaces the button element with the Link
<Button render={<Link to="/dashboard" />} />
```

The `render` prop replaces the rendered element entirely. If the child component needs children, put them inside the render prop element:

```jsx
<SidebarMenuButton render={<Link to="/dashboard" />}>
  Dashboard
</SidebarMenuButton>
```

- `render` = element replacement (shadcn/base-ui approach)
- `asChild` = Radix UI approach (used by some Radix primitives shadcn wraps)

---

## Phase 5: The shadcn Sidebar Block

The shadcn `sidebar` block is a full composable UI system installed via:

```bash
pnpm dlx shadcn@latest add sidebar
```

Generates `src/components/ui/sidebar.tsx` (~723 lines, 25 exports) and `src/hooks/use-mobile.ts`.

### Architecture

```
SidebarProvider (context — open/close state)
  Sidebar (desktop: persistent, mobile: Sheet overlay)
    SidebarHeader
    SidebarContent
      SidebarGroup > SidebarGroupLabel > SidebarGroupContent
        SidebarMenu > SidebarMenuItem
          SidebarMenuButton    (uses render prop, accepts tooltip)
          SidebarMenuSub > SidebarMenuSubItem > SidebarMenuSubButton
    SidebarFooter > SidebarMenu > ...
  SidebarRail     (desktop collapse handle)
  SidebarInset    (main content area)
  SidebarTrigger  (mobile toggle button)
```

### Critical: SidebarProvider Placement

`SidebarProvider` must wrap both the `<Sidebar>` and the content area. `useSidebar()` throws if called outside a provider. If multiple layouts each manage their own sidebar state, place a `SidebarProvider` in each layout.

### Mobile behavior

- On mobile (<768px), the sidebar renders as a shadcn `Sheet` — hidden by default, slides in via `SidebarTrigger`.
- On desktop, persistent panel with optional icon-only collapse via `SidebarRail`.
- Show `SidebarTrigger` with `className="md:hidden"` to only appear on mobile.

### The Collapsible Submenu Pattern (Critical)

For sidebar submenus, shadcn uses `Collapsible` from `@base-ui/react/collapsible`. **CRITICAL**: `Collapsible` is a namespace object, not a component.

```jsx
import { Collapsible } from '@base-ui/react/collapsible'

// ❌ BROWS — "Element type is invalid: expected a string or function but got: object"
<Collapsible defaultOpen={true}>...</Collapsible>

// ✅ CORRECT — use Collapsible.Root
<Collapsible.Root defaultOpen={true}>
  <Collapsible.Trigger>...</Collapsible.Trigger>
  <Collapsible.Panel>...</Collapsible.Panel>
</Collapsible.Root>
```

Props for `Collapsible.Root`: `defaultOpen`, `open`, `onOpenChange`, `disabled`, `className`.

---

## Phase 6: Theme Reconciliation

shadcn's default `--primary` color is near-black (`oklch(0.145 0 0)`). Most projects need to override it.

### Where to override

In `src/index.css`, after the shadcn-generated `:root` block:

```css
:root {
  --primary: oklch(0.546 0.215 262.881);  /* blue-600 */
  --primary-foreground: oklch(1 0 0);
  --ring: oklch(0.546 0.215 262.881);
  --sidebar-primary: oklch(0.546 0.215 262.881);
  --sidebar-primary-foreground: oklch(1 0 0);
  --sidebar-ring: oklch(0.546 0.215 262.881);
}

.dark {
  --primary: oklch(0.623 0.214 259.815);
  --ring: oklch(0.623 0.214 259.815);
  --sidebar-primary: oklch(0.623 0.214 259.815);
}
```

The `@theme inline { --color-primary: var(--primary); }` block wires CSS variables to Tailwind classes. Override the variables and `bg-primary`, `text-primary-foreground`, etc. follow automatically.

**Don't forget `--sidebar-primary`** — the sidebar block uses its own `--sidebar-*` variables.

---

## Phase 7: The Lint Trap

After a large migration, lint will be broken even if build passes.

| Symptom | Root Cause | Fix |
|---|---|---|
| `eslint: command not found` | ESLint not installed | `pnpm add -D eslint` |
| `.jsx` files ignored | `eslint.config.js` ignores wrong dir | Fix `ignores: ["build/**"]` not `"dist/**"` |
| `'React' is defined but never used` | React 19 JSX transform | Remove `import React` from all `.jsx` files |
| `'X' is defined but never used` | Orphaned imports from migration | Search with `rg` and remove |
| `__dirname is not defined` | ESM module | Replace with `fileURLToPath(import.meta.url)` |

### Setup commands

```bash
pnpm add -D eslint @eslint/js globals eslint-plugin-react-hooks eslint-plugin-react-refresh
pnpm run lint
```

---

## Phase 8: Delete the Old Design System

Before deleting:

1. **Verify all imports** are updated — `rg "from '@design-system'" src/` should return nothing.
2. **Check `vite.config.js`** for stale aliases (`@design-system`).
3. **Check `tsconfig.json`** for stale path aliases.
4. **Remove the directory**: `rm -rf src/design-system/`
5. **Run build + lint** to confirm nothing is broken.

---

## Phase 9: Documentation Cleanup

1. **`AGENTS.md`**: Remove old component references, update UI component table, remove outdated patterns.
2. **`README.md`**: Remove `@design-system` references, update dependency lists.
3. **Create `COMPONENT_CATALOG.md`**: Track provenance — which components are shadcn, which are custom, which were removed. (See the `ui-migration` skill for the catalog format.)

---

## Phase 10: Dependency Cleanup

### Likely removals

`formik`, `react-helmet`, `react-portal`, `react-resizable`, `react-router-hash-link`, `@fortawesome/*`, `@tanstack/eslint-plugin-query`, `react-toastify`, `classnames`/`clsx`, `postcss`/`autoprefixer`, and any component library wrappers no longer imported.

### Move shadcn to devDependencies

```bash
pnpm add -D shadcn
```

### Check peer dep warnings

```bash
pnpm ls --depth=0
```

---

## Common Runtime Errors

| Error | Cause | Fix |
|---|---|---|
| `Element type is invalid: expected a string or a class/function but got: object` | Using `Collapsible` directly instead of `Collapsible.Root` | Use `Collapsible.Root` |
| `useSidebar must be used within a SidebarProvider` | Component using `useSidebar()` outside provider | Wrap in `SidebarProvider` |
| `Module not found: Can't resolve '@/components/ui/X'` | Component not installed or wrong path | `pnpm dlx shadcn@latest add <name>` |
| Tailwind classes not applying (v4) | CSS variables not wired via `@theme inline` | Ensure `@theme inline` block maps variables |

---

## Verification Checklist

After every phase, run:

```bash
# 1. Build produces no errors
pnpm run build

# 2. Lint produces no errors
pnpm run lint

# 3. No remaining imports from old design system
rg "from '@design-system'" src/

# 4. No unused dependencies
pnpm ls --depth=0
```

---

## Hardest Lessons

1. **Lint matters more than build.** A green build can hide months of code rot.
2. **`@base-ui/react`'s `Collapsible` namespace is the biggest footgun.** Every agent hits this.
3. **Documentation debt is invisible but expensive.** Phantom components in agent instructions cause wasted cycles.
4. **shadcn's Sidebar block (723 generated lines) acts like a framework.** Read the entire file before using it.
5. **Theme variables must be overridden in both `:root` and `.dark`, and for both `--primary` and `--sidebar-primary`.**
6. **Move fast but validate often.** Build+lint after every single component swap.
7. **The real output is the process artifacts.** `TODOS.md`, `ERRORS.md`, `COMPONENT_CATALOG.md` make the migration reproducible. Without them, the next agent starts from zero.
