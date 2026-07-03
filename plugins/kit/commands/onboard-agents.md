---
description: Scaffold a project-agnostic agent set into .claude/agents/ — profile the project, pick archetypes, personalize, and write
---

Onboard a tailored set of agents into the current project's `.claude/agents/` folder. The bundle is **project-agnostic** — archetypes are filled with the user's stack details, not pre-baked for React or any other framework.

Perform the following steps.

---

## Step 1: Locate the bundled templates

The plugin ships archetype skeletons under:

```
${CLAUDE_PLUGIN_ROOT}/templates/agents/archetypes/
```

Confirm the archetype directory exists (`ls ${CLAUDE_PLUGIN_ROOT}/templates/agents/archetypes/`). If it doesn't, stop and tell the user the plugin is not properly installed.

The archetype files are:

| Archetype file | Role |
| --- | --- |
| `code-structure-researcher.md` | Maps the codebase slice for a feature. Read-only. |
| `spec-researcher.md` | Pulls requirements from tickets, wiki, RFCs, in-repo docs. Read-only. |
| `architecture-reviewer.md` | Reviews diff against project standards. Read-only. |
| `senior-developer.md` | Implements features and fixes in the chosen stack. |
| `team-lead-orchestrator.md` | Dispatches the other agents and consolidates results. |
| `e2e-tester.md` | Authors and runs end-to-end tests in the project's runner. |
| `design-researcher.md` | Translates design specs (Figma/etc.) into developer-ready brief. Read-only. |
| `project-driver.md` | The "agent who knows your UI/SDK." Drives the project's primary surface live. |

---

## Step 2: Auto-detect project profile

Scan the project root for hints. Run these checks in parallel (one Bash call each, or a single combined call) and remember the results:

- iOS: `Package.swift`, `*.xcodeproj`, `*.xcworkspace`, `Podfile`
- Android: `build.gradle`, `build.gradle.kts`, `AndroidManifest.xml`, `settings.gradle*`
- Web (SPA / SDK): `package.json`, `tsconfig.json`, `*.tsx`, `*.jsx`
- Backend / general Node: `package.json` with no `*.tsx`/`*.jsx`
- Rust: `Cargo.toml`
- Go: `go.mod`
- Python: `pyproject.toml`, `setup.py`, `requirements.txt`
- Monorepo signals: `nx.json`, `pnpm-workspace.yaml`, `lerna.json`, `turbo.json`
- Test framework: `playwright.config.*`, `jest.config.*`, `vitest.config.*`, `cypress.config.*`, `karma.conf.*`, `*.xctest`, `*.feature` (Cucumber), `pytest.ini`, `go test` patterns
- Design source: any `*.fig`, presence of `figma` strings in `package.json` deps

Also detect package manager from lock files: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm, `Cargo.lock` → cargo, `Pipfile.lock` → pipenv, etc. For non-Node projects, default to the project's primary build tool (xcodebuild, gradle, cargo, go, make, etc.).

---

## Step 3: Confirm the profile with the user

Use `AskUserQuestion` with the auto-detected hints pre-applied. Ask in **one batch** (1–4 questions max):

1. **Project kind** (single-select). Options based on hints: `iOS app/SDK`, `Android app/SDK`, `Web app (SPA)`, `Web SDK or library`, `Backend service`, `CLI / tool`, `Other`. Pre-select the strongest hint.
2. **Repo layout** (single-select): `single app`, `monorepo`, `multi-target`. Pre-select `monorepo` if any monorepo signal was detected.
3. **Has a design source you want an agent to read from?** (single-select): `Figma`, `Other design tool`, `No`.
4. **Has a project-specific UI/SDK an agent should drive?** (single-select): `Yes — describe in the next step`, `No`.

After the user answers, ask follow-up questions only if needed:
- If they said yes to a project-specific UI/SDK, ask for: short surface name (e.g. "the iOS demo app", "Storybook", "the SDK e2e harness", "the moderation page"), the launch command or hint, the path where its specs live, and one sentence describing what "success" looks like when an agent boots it.
- Confirm the **language(s)** and **test framework** if auto-detection was ambiguous — short free-form question.

Store the user's answers in a placeholder map for Step 5. Suggested placeholder values:

| Placeholder | Source |
| --- | --- |
| `{{PROJECT_NAME}}` | Directory name, or ask if it's unclear |
| `{{PROJECT_KIND}}` | Step 3 question 1 |
| `{{LANGUAGE}}` | Detected or asked (e.g. `Swift`, `Kotlin`, `TypeScript`, `Rust`) |
| `{{STACK}}` | One-liner like `Swift + SwiftUI + XCTest` or `Kotlin + Jetpack Compose + Espresso` |
| `{{FRAMEWORK}}` | Primary framework (e.g. `SwiftUI`, `Jetpack Compose`, `Express`, `FastAPI`, `React 18`) |
| `{{REPO_LAYOUT}}` | Step 3 question 2 |
| `{{TEST_RUNNER}}` | Detected or asked (e.g. `XCUITest`, `Playwright`, `pytest`) |
| `{{TEST_PATH}}` | Conventional path for tests in this project |
| `{{PACKAGE_MANAGER}}` | Detected (e.g. `pnpm`, `cargo`, `swift package`, `gradle`) |
| `{{BUILD_CMD}}` | Real command (e.g. `xcodebuild`, `cargo build`, `pnpm run build`) |
| `{{LINT_CMD}}` | Real command (e.g. `swiftlint`, `cargo clippy`, `pnpm run lint`) |
| `{{UI_SURFACE}}` | Step 3 question 4 follow-up |
| `{{DRIVER_NAME}}` | Suggested name for the project-driver agent — propose one based on the surface (e.g. `ios-app-driver`, `storybook-driver`, `sdk-driver`). Confirm with the user. |
| `{{DRIVER_PATH}}` | Where the surface's specs / harness live |
| `{{DRIVER_BOOTSTRAP}}` | Canonical bootstrap spec/file path |
| `{{LAUNCH_HINT}}` | Short hint like "the npm dev script", "the e2e harness target" |
| `{{SUCCESS_CRITERIA}}` | One-line success definition the user gave |

---

## Step 4: Recommend archetypes

Show a multi-select with `AskUserQuestion` (or list + confirmation) for which archetypes to install. Pre-check based on the profile:

- **Always pre-checked**: `code-structure-researcher`, `spec-researcher`, `architecture-reviewer`, `senior-developer`, `team-lead-orchestrator`.
- **Pre-checked if a test framework was detected/named in Step 3**: `e2e-tester`.
- **Pre-checked if a design source was named**: `design-researcher`.
- **Always offered, never pre-checked**: `project-driver` — explain in plain language: *"Want an agent who knows how to operate {{UI_SURFACE}}? It will boot your surface, exercise features, and report what it sees."*

Let the user toggle individual entries.

---

## Step 5: Personalize each selected archetype

For each archetype the user picked, do the following:

1. Read the template from `${CLAUDE_PLUGIN_ROOT}/templates/agents/archetypes/<name>.md`.
2. Substitute every `{{PLACEHOLDER}}` token using the map from Step 3. Leave a sensible default in place (and warn the user) if a placeholder has no value — never write a literal `{{PLACEHOLDER}}` to disk.
3. For `project-driver` specifically:
   - The `name:` frontmatter must become `{{DRIVER_NAME}}` (e.g. `ios-app-driver`).
   - The file should be saved under `<cwd>/.claude/agents/{{DRIVER_NAME}}.md` — NOT `project-driver.md`.
4. For other archetypes, save to `<cwd>/.claude/agents/<archetype-name>.md` (matching the source filename).

If `<cwd>/.claude/agents/` doesn't exist, create it before writing.

---

## Step 6: Conflict policy — ask per file

For every target file that already exists, **stop before writing** and use `AskUserQuestion` to ask:

- `Overwrite` — replace with the freshly generated version.
- `Skip` — leave the existing file alone.
- `Show diff and decide` — print a unified diff (`diff -u <existing> <new-temp-file>`), then ask the same question again without the diff option.

Apply the chosen action and continue. Keep a running tally (`installed`, `skipped`, `overwritten`) for the final report.

---

## Step 7: Custom-agent wizard

After scaffolding, ask: *"Create another custom agent for this project?"*

If yes, prompt for:

- `name` (kebab-case, no extension)
- `description` (one or two sentences describing when to use the agent — this is what the orchestrator sees)
- `model` — optional, single-select: `opus`, `sonnet`, `haiku`, `(leave unset)`
- `tools` — optional, comma-separated (e.g. `Read, Grep, Glob, Bash`). Leave blank for "all tools."
- `color` — optional, free-form
- `role` — one paragraph the wizard pastes verbatim into the body
- Pick one starter shape: `researcher (read-only)`, `implementer`, `reviewer`, `driver/operator`, `blank`.

Write `<cwd>/.claude/agents/<name>.md` with this layout:

```markdown
---
name: <name>
description: <description>
[model: <model>]
[tools: <tools>]
[color: <color>]
---

<role>

## Responsibilities
- <bullet>

## How to work
1. <step>
2. <step>

## Output
<what the agent returns to the caller>

## Rules
- <bullet>
```

Brackets denote optional lines — omit when unset. Apply the same conflict policy from Step 6 if the file exists.

Loop until the user says they're done.

---

## Step 8: Final report

Tell the user, in a tight summary:

- Project profile detected (`{{PROJECT_KIND}}`, `{{LANGUAGE}}`, `{{REPO_LAYOUT}}`, etc.)
- Agents installed (full list with paths)
- Agents skipped / overwritten (if any)
- Any custom agents created
- One-liner reminder: agents are auto-loaded by Claude Code from `.claude/agents/` — they can be invoked via the `Task` tool with `subagent_type: "<name>"`, or by `@<name>` mention in chat.
- Suggest committing `.claude/agents/` to source control so teammates pick up the same set.

---

## Rules

- **Never write a file containing a literal `{{PLACEHOLDER}}` token.** Fill all placeholders or warn the user.
- **Project-agnostic by default.** Do not assume React, Redux, Nx, Playwright, or any specific stack — read the profile from Step 3 and respect it.
- Respect the per-file conflict policy. Don't silently overwrite.
- Keep the wizard fast: pre-fill from auto-detection, batch questions where possible, only prompt for what isn't already known.
