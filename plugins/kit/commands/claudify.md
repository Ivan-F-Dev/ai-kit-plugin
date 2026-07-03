---
description: Initialize project conventions — interactive language selection, CLAUDE.md, tasks/, and review workflow
---

Set up this project with standard conventions. Perform the following steps:

## Step 1: Check for Existing CLAUDE.md

Before anything else, check if `CLAUDE.md` already exists in the project root.

**If `CLAUDE.md` does NOT exist** — skip this step entirely and proceed to Step 1.

**If `CLAUDE.md` exists** — ask the user how to handle it. Present a single-select with these options:

- **Overwrite (Recommended)** — Replace the existing `CLAUDE.md` entirely with freshly generated content
- **Safe Merge** — Backs up your current `CLAUDE.md` to `claude_prev.md` and creates a merged version combining both rule sets, so your customizations are never lost

### If user selects "Safe Merge":

Store a flag so that after Step 3 generates the new `CLAUDE.md`, the following merge procedure runs:

1. **Before Step 3 writes anything**, copy the existing `CLAUDE.md` to `claude_prev.md` in the project root
2. **Step 3 proceeds normally** — writes the new `CLAUDE.md` with fresh content
3. **After Step 3 completes**, merge `claude_prev.md` into the new `CLAUDE.md`:
   - The new template structure and sections take priority (headings, config comment, section ordering)
   - Preserve any **custom sections** from `claude_prev.md` that don't exist in the new template — append them at the end
   - For sections that exist in both: keep the new template version but **merge in any extra rules/items** the user had added within those sections
   - The `<!-- kit:config -->` line must always come from the new template
   - Keep `claude_prev.md` in the project root so the user can review what changed

### If user selects "Overwrite":

Proceed normally — Step 3 will write `CLAUDE.md` from scratch, replacing any existing content.

---

## Step 2: Interactive Language Selection

Ask the user to select which language stacks this project uses. Present a multi-select with these options:

- **Node.js/TypeScript** — backend services, APIs, TypeScript libraries
- **Frontend/React** — React apps, component libraries, SPAs
- **iOS/Swift** — Swift mobile apps
- **Android** — Kotlin/Java mobile apps
- **ROKU** — Roku apps
- **DevOps** — DevOps tools, scripts, or services
- **Other** — Any other SDK, apps, or services

**Auto-detect fallback:** Before asking, scan the project root for hints and pre-select:
- `package.json`, `tsconfig.json`, `*.ts` files -> pre-select Node.js/TypeScript
- `*.tsx`, `*.jsx`, `*.css`, `*.scss` -> pre-select Frontend/React
- `*.xcodeproj`, `*.swift`, `Podfile`, `Package.swift` -> pre-select iOS/Swift
- `AndroidManifest.xml`, `*.java`, `*.kt` -> pre-select Android
- `*.roku`, `*.xml` -> pre-select ROKU
- `*.yml`, `*.yaml`, `Dockerfile`, `docker-compose.*` -> pre-select DevOps

Present the auto-detected selection to the user and let them confirm or adjust.

Store the selection as a comma-separated list for the config comment. Mapping:
- Node.js/TypeScript -> `node`
- Frontend/React -> `frontend`
- iOS/Swift -> `ios`
- Android -> `android`
- ROKU -> `roku`
- DevOps -> `devops`
- Other -> `other`

---

## Step 3: Documentation Folder Setup

Ask the user which folder to use for project documentation. Default is `docs`. Let them confirm or provide a custom path.

Once confirmed, create the following structure (if it doesn't already exist):

| Folder              | Purpose                                                        |
| ------------------- | -------------------------------------------------------------- |
| `$DOCS/changelog/`  | Release notes and change logs — updated on every non-bugfix task |
| `$DOCS/wiki/`       | Feature documentation — created for major features             |
| `$DOCS/guide/`      | Integration guides — created when adding or changing integrations |
| `$DOCS/dev/`        | Developer processes — updated when workflows or tooling change |

Replace `$DOCS` with the chosen folder (e.g. `docs`).

### Auto-documentation rules (add to CLAUDE.md)

Every completed task **except pure bug fixes** must generate or update the relevant doc:

- **New feature / major change** → create or update `$DOCS/wiki/<feature>.md`
- **Integration work** (third-party APIs, SDKs, webhooks) → create or update `$DOCS/guide/<integration>.md`
- **Dev process change** (CI, tooling, conventions) → update `$DOCS/dev/<topic>.md`
- **Any shipped change** → append entry to `$DOCS/changelog/<version-or-date>.md`

Bug fixes only need a changelog entry when they are significant enough to mention in release notes.

---

## Step 4: Create or update CLAUDE.md

Write the following content to `CLAUDE.md` in the project root. Replace `$LANGUAGES` with the comma-separated language list from Step 1 (e.g. `node,frontend`).

**IMPORTANT:** The `<!-- kit:config -->` comment on line 1 records the project's language stacks as a lightweight machine-readable marker. Keep it — do not remove or reformat it.

```markdown
<!-- kit:config languages=$LANGUAGES -->

## Codebase Navigation

- Before exploring the filesystem, read `CODEBASE.md` first.
- Only explore files directly if `CODEBASE.md` doesn't cover what you need.
- If you explored files not covered by `CODEBASE.md` during a task, append them to the relevant section.
- After completing any task that adds, removes, or moves files or changes architecture, update `CODEBASE.md` to reflect the changes.
- Use `/kit:explore` to force a full refresh of `CODEBASE.md`.

## Project Commands

$PACKAGE_MANAGER_SECTION

## Documentation

$DOCS_SECTION

## Workflow Orchestration

### 1. Plan First

- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- Write plan to `.claude/plans/<task-name>.md` with checkable items
- If something goes sideways, STOP and re-plan — don't keep pushing
- Check in with user before starting implementation on complex tasks

### 2. Subagent Strategy

- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- One task per subagent for focused execution
- For complex problems, throw more compute at it via parallel subagents

### 3. Verification Before Done

- Run build to verify nothing is broken
- Ask yourself: "Would a staff engineer approve this?"
- Diff behavior between main and your changes when relevant

### 4. Self-Improvement Loop

- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules that prevent the same mistake class
- Review lessons at session start for the relevant project

## Code Standards

### Simplicity & Minimal Impact

- Make every change as simple as possible — touch only what's necessary
- Find root causes. No temporary fixes. Senior developer standards
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky, step back and implement the clean solution
- Skip elegance overthinking for simple, obvious fixes

### Respect User's Code

- **Do NOT remove console.logs** from uncommitted changes unless user explicitly asks
- **Do NOT remove comments**, TODO markers, or debugging helpers unless asked
- **Do NOT refactor** adjacent code that isn't part of the current task
- **Do NOT change formatting/style** of untouched code (respect existing patterns)
- Preserve import order conventions, naming patterns, and file structure

### Dependencies & Libraries

- Always check `package.json` if `node` or `frontend` is in the selected languages for current dependency versions before using any API
- Use Context7 MCP to look up documentation for the **exact version** in use
- Never assume API signatures — verify against the installed version
- When adding dependencies, prefer what's already in the project ecosystem

### Autonomous Bug Fixing

- When given a bug report: just fix it — don't ask for hand-holding
- Point at logs, errors, failing tests — then resolve them
- Go fix failing CI tests without being told how
- Zero context switching required from the user

## Task Management

All session metadata lives in the `./claude/` folder and should be tracked in Git.

| File / Folder              | Purpose                                            |
| -------------------------- | -------------------------------------------------- |
| `./claude/plans/*.md`      | Plans with checkable items + review section        |
| `./claude/tasks/lessons.md`   | Patterns learned from corrections                  |
| `./claude/tasks/decisions.md` | Architectural decisions and reasoning (optional)   |

### Workflow

1. **Plan** -> Write to `.claude/plans/<task-name>.md`
2. **Confirm** -> Check in before starting (for complex tasks)
3. **Execute** -> Mark items complete as you go, explain changes at each step
4. **Verify** -> Lint, build, test
5. **Document** -> Add review section to the plan file, update `./claude/tasks/lessons.md` if needed.

## Communication

- Give high-level summaries, not line-by-line narration
- When presenting options, give a clear recommendation with reasoning
- If blocked or unsure, say so immediately — don't guess and break things
- When a task is done, summarize: what changed, what was tested, any caveats

## PR Code Review

Use `/kit:review <PR#>` to run a self-review on any pull request (React/frontend focused).

Use `/kit:summary` to draft commit message options and a wrap-up comment for the work done.
```


### Template rules for `$PACKAGE_MANAGER_SECTION`:

**If `node` or `frontend` is in the selected languages**, include this section:

```markdown
### Package Manager Detection (ALWAYS do this first)

Detect the package manager by lock file before running ANY command:

| Lock file           | Package manager | Run scripts with    |
| ------------------- | --------------- | ------------------- |
| `pnpm-lock.yaml`    | pnpm            | `pnpm run <script>` |
| `yarn.lock`         | yarn            | `yarn <script>`     |
| `package-lock.json` | npm             | `npm run <script>`  |

**Never default to npm** — always check the lock file first.

- **Build**: `<pm> run build` (or relevant build script from package.json)
- **Lint**: `<pm> run lint` — use `<pm> run lint -- --fix` for autofixing
- **Test**: `<pm> run test` — **only when user explicitly asks**
- **Always check package.json** for available scripts before guessing commands
```

### Template rules for `$DOCS_SECTION`:

Replace `$DOCS` with the folder chosen in Step 2 (default `docs`).

```markdown
Every completed task **except pure bug fixes** must generate or update relevant docs:

| Change type                        | Action                                                  |
| ---------------------------------- | ------------------------------------------------------- |
| New feature / major change         | Create or update `$DOCS/wiki/<feature>.md`              |
| Integration (APIs, SDKs, webhooks) | Create or update `$DOCS/guide/<integration>.md`         |
| Dev process (CI, tooling, conventions) | Update `$DOCS/dev/<topic>.md`                       |
| Any shipped change                 | Append entry to `$DOCS/changelog/<version-or-date>.md`  |

Bug fixes only need a changelog entry when significant enough for release notes.
```

---

## Step 5: Generate CODEBASE.md

Analyze the project structure and generate a `CODEBASE.md` file in the project root. This is a lightweight structural map — think table of contents, not a book.

**How to generate:**

1. Use glob patterns to discover the directory tree and key files (entry points, configs, main modules)
2. Read a sample of important files (max ~20) to understand their purpose — do NOT dump full contents
3. Identify architectural patterns, component relationships, and module boundaries

**Write `CODEBASE.md` with the following structure:**

```markdown
# Codebase Map

> Auto-generated by `/kit:claudify`. Keep updated — see CLAUDE.md for rules.

## Project Overview

<!-- One paragraph: what this project is, what it does, core tech stack -->

## Directory Structure

<!-- Top-level and important nested directories with one-line descriptions -->

| Path | Description |
| ---- | ----------- |
| `src/` | Main application source code |
| `src/api/` | REST API route handlers |
| ... | ... |

## Key Files

<!-- Important files: entry points, configs, core modules -->

| File | Description |
| ---- | ----------- |
| `src/index.ts` | Application entry point |
| `package.json` | Dependencies and scripts |
| ... | ... |

## Architecture

<!-- How components connect, data flow, key patterns. Brief prose, not code dumps. -->

```

**Rules:**
- List EVERY top-level directory and its purpose
- Go one level deeper for directories with distinct sub-modules
- Include all config files (tsconfig, eslint, docker, CI, etc.)
- Describe relationships: "X calls Y", "A depends on B", "C is the entry point for D"
- Do NOT include file contents, code snippets, or line-by-line explanations
- Keep descriptions to one line each — concise and scannable
- If the project is very large (100+ files), focus on the most important paths and note what was omitted

---

## Step 6: Configure allowed permissions

Add the following permissions to `./claude/settings.json`:

Bash(git log:*), Bash(gh pr:*), Bash(pnpm build:*), Bash(git add:*), Bash(git push:*), Bash(git pull:*), Bash(git commit:*), Bash(gh repo:*), Bash(gh api:*), Bash(git diff:*), Bash(git status:*), Bash(git:*)

---

## Step 7: Create tasks/, plans/ folder structure

Create the following files if they don't already exist:

- `./claude/plans/` — add a `.gitkeep` file
- `./claude/tasks/lessons.md` — with header `# Lessons Learned`
- `./claude/tasks/decisions.md` — with header `# Architectural Decisions`

Also create the documentation folder structure from Step 2:

- `$DOCS/changelog/` — add a `.gitkeep` file
- `$DOCS/wiki/` — add a `.gitkeep` file
- `$DOCS/guide/` — add a `.gitkeep` file
- `$DOCS/dev/` — add a `.gitkeep` file

---

## Step 8: Git tracking

Add `CODEBASE.md` to Git tracking.
Add `./claude/tasks/` folder to Git tracking.
Add `./claude/plans/` folder to Git tracking.
Add `./claude/settings.json` folder to Git tracking.
Add `$DOCS/` folder to Git tracking.

---

## Step 9: Confirm

Tell the user:

- CLAUDE.md has been created/updated
  - If **Safe Merge** was used: mention that `claude_prev.md` contains the original backup and custom rules were merged in
  - If **Overwrite** was used: mention that the previous CLAUDE.md was replaced entirely
- CODEBASE.md has been generated with structural map of the project
- Selected languages: [list the selected stacks]
- `./claude/tasks/` folder is ready
- `$DOCS/` documentation structure is ready:
  - `$DOCS/changelog/` — change logs updated on every shipped change
  - `$DOCS/wiki/` — feature docs created for major features
  - `$DOCS/guide/` — integration guides for APIs, SDKs, webhooks
  - `$DOCS/dev/` — developer process docs for CI, tooling, conventions
  - **Auto-docs rule:** every task (except pure bug fixes) will generate or update the relevant doc automatically
- Available commands:
  - `/kit:review <PR#>` — review a pull request (React/frontend focused)
  - `/kit:summary` — draft commit message options and a wrap-up comment
  - `/kit:explore` — force refresh `CODEBASE.md` with latest project structure
