---
name: claude-skill-angular-upgrade
description: >
  Angular version upgrade skill — upgrades any legacy Angular project to the
  latest version, step by step. Use when asked to upgrade Angular, run ng update,
  fix post-upgrade errors, or modernize an Angular codebase. Handles Angular
  Material breaking changes, CSS/SCSS regressions, TypeScript strict mode errors,
  and dependency conflicts. Always analyzes the project first, upgrades one major
  version at a time, and asks the user before making any behavioral or dependency
  changes.
---

# Angular Upgrade Skill

Upgrade any Angular project from a legacy version to the latest, safely and
incrementally. This skill is cautious by design — it never changes behavior or
replaces a dependency without explicit user confirmation.

---

## Core Principles

1. **Analyze before acting** — always read the project files first.
2. **One major version at a time** — never skip versions (v9 → v10 → v11 ...).
3. **Verify after every step** — run build and tests before moving on.
4. **Ask before changing behavior** — never silently swap APIs, components, or dependencies.
5. **Report clearly** — summarize what was done and the outcome after each step.
6. **Be cautious** — when in doubt, ask the user rather than assume.

---

## Input Parameters

This skill takes one main parameter:

- **Target version** — the Angular major version to upgrade to (e.g. `18`, `20`).
  - If the user specifies it (e.g. "upgrade to Angular 18"), use that as the
    final target — do not go past it even if a newer version exists.
  - If the user does not specify it, default to the latest stable Angular
    release. Confirm this default with the user during Phase 0 before
    proceeding, since "latest" changes over time.

- **Current version** — never asked from the user. Always detect it by reading
  `@angular/core` in `package.json` during Phase 0.

---

## Phase 0 — Project Analysis (always run first)

Before doing anything, read and analyze the following files:

```
package.json       → current versions of all dependencies, package manager (via lockfile)
angular.json       → project structure, builders, budgets
nx.json            → present only in Nx workspaces
tsconfig.json      → TypeScript config, strict flags
tsconfig.app.json  → app-specific TS config
src/styles.scss    → global styles, imports
```

Also check which lockfile is present (`package-lock.json`, `yarn.lock`, or
`pnpm-lock.yaml`) to determine the package manager, since update/install
commands must match it.

If `nx.json` exists, this is an Nx workspace: use `nx migrate {TARGET}` instead
of `ng update` throughout Phase 2, and follow it with `nx migrate --run-migrations`
after reviewing the generated migrations. Confirm this with the user before
proceeding, since the update flow differs from a plain Angular CLI workspace.

Build a report covering:

- Current Angular version (from `@angular/core` in `package.json`) and target version (from the user, or latest if unspecified)
- Current Node.js version (check engines field or ask user to run `node -v`)
- Package manager in use (npm/yarn/pnpm, detected from lockfile)
- Whether this is an Nx workspace or a plain Angular CLI workspace
- Angular Material version (if present)
- RxJS version
- All third-party dependencies and their versions
- TypeScript version
- Test framework in use (Karma/Jasmine, Jest, Vitest)
- Whether `standalone` components are used
- Whether `ng update` schematics are available for each dependency
- Estimated number of upgrade steps (major version hops)

**Output format:**

```
📋 PROJECT ANALYSIS REPORT
==========================
Angular:          9.1.x  →  target: latest (20.x)
Angular Material: 9.x
RxJS:             6.x
TypeScript:       3.8.x
Node.js:          12.x   ⚠️  needs upgrade before Angular 17+
Test framework:   Karma + Jasmine

Upgrade path: v9 → v10 → v11 → v12 → v13 → v14 → v15 → v16 → v17 → v18 → v19 → v20
Estimated steps: 11 major version upgrades

⚠️  Issues detected:
- Node 12 is incompatible with Angular 17+ (requires Node 18+)
- Angular Material appearance="legacy" removed in v15
- RxJS 6 → 7 has breaking changes (manual migration needed)

Proceed? (yes / no)
```

Do not proceed until the user confirms.

---

## Phase 1 — Pre-Upgrade Checklist

Before the first `ng update`, confirm the following with the user:

- [ ] Git repository is clean (all changes committed or stashed)
- [ ] A backup branch exists before upgrading — if not, offer to create one
      (`git checkout -b backup/pre-upgrade`) and ask for confirmation before
      running it, per normal git safety rules
- [ ] Node.js version is compatible with the target Angular version
- [ ] The user understands upgrades happen one major version at a time

**Node.js compatibility reference (approximate — verify exact minimums against
[angular.dev/reference/versions](https://angular.dev/reference/versions)
before relying on them, since this table can go stale):**

| Angular | Min Node   |
|---------|------------|
| v9–v10  | 10.13      |
| v11–v12 | 12.14      |
| v13     | 12.20      |
| v14     | 14.15      |
| v15     | 14.20      |
| v16     | 16.14      |
| v17     | 18.13      |
| v18–v19 | 18.19      |
| v20+    | 20.19      |

If Node.js needs upgrading, stop and ask the user to upgrade Node first. Do not
attempt to upgrade Node automatically.

---

## Phase 2 — Step-by-Step Version Upgrade

Repeat the following cycle for each major version hop. If this is an Nx
workspace (detected in Phase 0), substitute `nx migrate` for `ng update` in
Step A and follow the Nx migration flow instead.

### Step A — Preview, then run ng update

Always preview before applying, so conflicts and schematics are visible up
front instead of surprising the user:

```bash
ng update @angular/core@{TARGET} @angular/cli@{TARGET} --dry-run
```

Report what the dry run shows (files to be touched, peer dependency warnings,
any conflicts) and get confirmation before applying it for real:

```bash
ng update @angular/core@{TARGET} @angular/cli@{TARGET}
```

If Angular Material is installed:

```bash
ng update @angular/material@{TARGET} --dry-run
# then, after confirmation:
ng update @angular/material@{TARGET}
```

Never pass `--force` or `--allow-dirty` to skip a conflict without explicitly
asking the user first — those flags exist to bypass safety checks the user
should see.

### Step B — Verify build

```bash
ng build
```

Report the result:

```
✅ Build passed — Angular {VERSION} upgrade successful
   Next step: upgrade to v{NEXT}

❌ Build failed — {NUMBER} errors found
   Stopping. See errors below before proceeding.
```

### Step C — Verify tests (if test suite exists)

```bash
ng test --watch=false
```

### Step D — Report and confirm

After each version hop, report:

```
📊 STEP REPORT: v{FROM} → v{TO}
================================
Status:        ✅ Success / ❌ Failed
Build:         ✅ Passed / ❌ {N} errors
Tests:         ✅ Passed / ⚠️  {N} failures / ⏭️  skipped
Schematics:    Auto-migrated {N} files
Manual fixes:  {list any manual changes made}

Proceed to v{NEXT}? (yes / no / stop)
```

If the step succeeded, offer to commit the working tree before moving on:

```bash
git add -A
git commit -m "chore: upgrade Angular v{FROM} -> v{TO}"
```

This gives every version hop its own recoverable commit, so a failure two
steps later can be rolled back to the last known-good version instead of all
the way to `backup/pre-upgrade`. Ask before committing, same as any other git
write.

Never proceed to the next version without user confirmation.

---

## Phase 3 — Angular Material Migration

Angular Material has significant breaking changes across versions. Always handle
these explicitly.

### appearance="legacy" removal (v15)

`appearance="legacy"` was removed in Angular Material v15. When encountered:

1. **Stop and inform the user:**

```
⚠️  BREAKING CHANGE DETECTED — User confirmation required
================================================
`appearance="legacy"` has been removed in Angular Material v15.

All mat-form-field components using appearance="legacy" must be migrated.
Files affected: {list files}
Components affected: {count}

Available options:
  1. appearance="fill"    → filled background, no border
  2. appearance="outline" → bordered field (closest to legacy visually)
  3. appearance="standard" → underline only (deprecated, may be removed later)

⚠️  Important: Each option changes the visual appearance and sizing of your
form fields. This WILL affect your UI layout and may require CSS adjustments.

Which appearance would you like to use? (fill / outline / standard)
Or would you like to review each component individually?
```

2. Do not change any files until the user answers.
3. After migration, report all files changed.

### mat-form-field sizing and float label changes

Angular Material v15+ changed the default density and sizing of form fields.
Common symptoms:

- Float label overlaps border or disappears
- Input height changes
- Padding/spacing differs from previous version

When these issues are detected or reported by the user:

```
⚠️  FORM FIELD APPEARANCE ISSUE DETECTED
=========================================
Angular Material v15+ introduced a new design system (MDC-based components)
that changes the default sizing, density, and float label behavior of
mat-form-field.

Common causes:
  1. MDC migration changed component internals — CSS overrides may no longer work
  2. Float label now uses a different positioning strategy
  3. Default density changed (more compact)

Options to investigate:
  A. Add global density override in styles.scss:
     @include mat.form-field-density(-1);  ← adjust -1 to -2, -3 as needed
  B. Review and update custom CSS overrides targeting mat-form-field
  C. Use Angular Material theming mixins instead of direct CSS overrides

Would you like me to:
  1. Check your styles.scss for conflicting overrides?
  2. Apply a density override globally?
  3. List all affected components for manual review?
```

Do not apply any fix without user confirmation.

### Key Angular Material breaking changes by version

| Version | Breaking change |
|---------|----------------|
| v13 | `MatTableDataSource` moved to `@angular/material/table` |
| v14 | MDC-based components introduced (opt-in) |
| v15 | MDC components default, `legacy` appearance removed, `MatLegacy*` exports added as bridge |
| v16 | `MatLegacy*` bridge exports deprecated |
| v17 | `MatLegacy*` bridge exports removed |
| v17+ | `standalone: true` is default for new components |
| v19 | `standalone: true` removed from decorators (it is implicit) |

---

## Phase 4 — RxJS Migration

RxJS has its own breaking changes independent of Angular versions.

### RxJS 6 → 7

```
⚠️  RxJS MIGRATION REQUIRED
============================
RxJS 7 has breaking changes from RxJS 6.

Key changes:
  - `throwError()` now requires a factory function: throwError(() => new Error(...))
  - Some operators renamed or removed
  - `toPromise()` deprecated → use `firstValueFrom()` or `lastValueFrom()`

Would you like me to:
  1. Scan the codebase for deprecated RxJS 6 patterns?
  2. Apply automatic migration where safe?
  3. List all occurrences for manual review?
```

---

## Phase 5 — TypeScript Strict Mode

When TypeScript strict errors appear after upgrade:

1. List all errors grouped by type
2. Do not auto-fix — present to user first:

```
⚠️  TYPESCRIPT ERRORS DETECTED
================================
{N} errors found after upgrade. Summary:

  - {N} × "Object is possibly null"          → requires null checks
  - {N} × "Property does not exist on type"  → type definitions changed
  - {N} × "Argument not assignable"          → stricter generic types

Options:
  A. Fix errors one by one (recommended — safest)
  B. Temporarily disable strict checks in tsconfig (not recommended)
  C. Show all errors grouped by file

How would you like to proceed?
```

---

## Phase 6 — CSS/SCSS Migration

### @import → @use (Sass deprecation)

Angular v17+ uses Sass `@use` syntax. `@import` will show deprecation warnings
and will eventually fail.

```
⚠️  SCSS MIGRATION REQUIRED
============================
{N} files use deprecated @import syntax.

@import will be removed in a future Sass version. The replacement is @use/@forward.

This change WILL affect how variables and mixins are accessed in your stylesheets
and may require updating variable references throughout your SCSS files.

Would you like me to:
  1. List all files using @import?
  2. Migrate automatically where safe (simple cases only)?
  3. Skip for now and revisit after Angular upgrade is complete?
```

---

## Mandatory Confirmation Rules

The following changes **always** require explicit user confirmation before
proceeding. Never apply these silently:

| Change | Why confirmation is required |
|--------|------------------------------|
| Replacing a dependency with a different one | Changes behavior, API, and bundle size |
| Changing `appearance` on `mat-form-field` | Changes visual layout of the entire UI |
| Modifying `tsconfig.json` strict flags | Affects type safety across the whole project |
| Changing state management approach | Architectural decision |
| Replacing test framework | Affects all test files |
| Upgrading Node.js | Environment change outside the project |
| Changing SCSS from `@import` to `@use` | May break variable/mixin references |
| Any change affecting more than 10 files | High risk of unintended side effects |

**Confirmation format:**

```
⚠️  CONFIRMATION REQUIRED
==========================
Action:  {describe exactly what will be changed}
Affects: {N} files / {scope of change}
Reason:  {why this change is needed}
Risk:    {what could go wrong}

Question 1: {specific question}
Question 2: {specific question if needed}

Type YES to confirm, NO to skip, or DETAILS for more information.
```

---

## Rollback Instructions

If any step fails and cannot be recovered:

```bash
# Reset to the last successful per-version commit (see Phase 2, Step D)
git log --oneline -10   # find the commit for the last good version hop
git reset --hard {COMMIT_HASH}

# Or, if no per-step commits exist, return to the backup branch created before upgrade
git checkout backup/pre-upgrade
```

Always remind the user of this option when a step fails. Never run
`git reset --hard` or `git checkout` without the user's confirmation, since
both discard uncommitted work.

---

## Post-Upgrade Report

After all versions have been upgraded successfully:

```
🎉 UPGRADE COMPLETE
====================
From:    Angular {START_VERSION}
To:      Angular {END_VERSION}
Steps:   {N} major version hops
Duration: {estimated}

Changes made:
  ✅ Angular core + CLI upgraded
  ✅ Angular Material migrated
  ✅ {N} files auto-migrated by schematics
  ⚠️  {N} manual changes applied (see list below)

Manual changes:
  - {file}: {description of change}
  - ...

Recommended next steps:
  1. Run full test suite: ng test
  2. Run e2e tests if available
  3. Review all ⚠️  items flagged during upgrade
  4. Check Angular Material theming and form field appearance visually
  5. Update any CI/CD pipeline Node.js version requirements
```

---

## References

- Angular update guide: https://update.angular.io
- Angular Material migration: https://material.angular.io/guide/mdc-migration
- RxJS migration: https://rxjs.dev/guide/v7/migration
- Angular changelog: https://github.com/angular/angular/blob/main/CHANGELOG.md

---

## When to apply

Use this skill when the user asks to:

- Upgrade an Angular project to a newer or the latest version
- Run or troubleshoot `ng update`
- Migrate an Angular codebase across major versions
- Update Angular Material and fix related breaking changes (e.g. `appearance="legacy"`, MDC migration, form field sizing)
- Fix errors that appear after an Angular upgrade (build failures, TypeScript strict mode errors, RxJS breaking changes, SCSS `@import`/`@use` issues)
- Modernize a legacy Angular codebase (e.g. adopt standalone components)

Do not use this skill for unrelated Angular work such as new feature development, styling unrelated to a migration, or general debugging that isn't tied to a version upgrade.
