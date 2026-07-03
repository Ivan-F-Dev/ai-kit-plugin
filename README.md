# kit — Personal Claude Plugin

A personal all-in-one plugin to set up project conventions, map a codebase, review React/frontend PRs, scaffold agents, and commit with conventional format — consistently across my repositories.

## Installation

```
/plugin marketplace add Ivan-F-Dev/ai-kit-plugin
/plugin install kit@ai-kit
```

For a private repo, use the full URL instead: `git@github.com:Ivan-F-Dev/ai-kit-plugin.git`.

## Commands

| Command                  | Description                                                    |
| ------------------------ | -------------------------------------------------------------- |
| `/kit:claudify`          | Interactive project setup — select languages, generate CLAUDE.md & CODEBASE.md, create tasks/ |
| `/kit:explore`           | Force refresh `CODEBASE.md` — re-analyze project structure and regenerate the structural map |
| `/kit:review <PR#>`      | React/frontend PR review — applies common + React rulesets, posts inline GitHub comments |
| `/kit:review-local [PR#]`| Local-first review with scope flags (`--ldiff`/`--bdiff`/`--full`), or `--arch` for a high-level architecture review written to `ARCHITECTURE_REVIEW.md` |
| `/kit:summary`           | Propose commit message options and a wrap-up comment summarizing the work done |
| `/kit:onboard-agents`    | Scaffold a project-agnostic agent set into `.claude/agents/` — profile the project, pick archetypes, personalize, and write |

## How it works

### `/kit:claudify` — Project Setup

1. Auto-detects your stack (Node.js/TS, Frontend/React, iOS/Swift, Android, Roku, DevOps)
2. Lets you confirm or adjust the selection
3. Generates `CLAUDE.md` with project conventions and a machine-readable config comment
4. Analyzes the codebase and generates `CODEBASE.md` — a structural map with directory descriptions, key files, and architecture relationships
5. Creates `tasks/` folder for session-local planning

### `/kit:review <PR#>` — PR Review

1. Fetches the PR diff
2. Applies common rules (bugs, security, style) + a React/frontend ruleset (via the `frontend-reviewer` agent)
3. Posts inline comments to GitHub
4. Posts a summary comment

### `/kit:review-local [PR#]` — Local-First Review

Reviews your changes locally first, prints findings in the conversation, then offers to publish.

**Scope flags:** `--ldiff` (default — staged + working-tree changes), `--bdiff` (adds committed changes vs the base branch), `--full` (review the whole content of each changed file, not just changed lines).

**Architecture flag:** `--arch` switches to a high-level architecture review — structure, component organization, consistency, and duplication ("reinvented wheels") — via the `architecture-reviewer` agent. Defaults to `--bdiff` scope, may scan the wider codebase for context, and writes the report to `ARCHITECTURE_REVIEW.md` in the project root (no GitHub posting).

1. Parses flags; if a PR number is given, resolves the target branch (stash/commit/checkout when needed)
2. Builds the diff for the chosen scope — in `--full` mode also loads the full changed files
3. Applies common rules + the React/frontend ruleset
4. Prints a markdown findings report in the conversation
5. Offers three options:
   - **Publish findings** — post inline comments to the PR on GitHub
   - **Fix + publish** — fix issues, commit, push, then post a summary comment
   - **Leave as-is** — done (or just "Fix / Leave as-is" when no PR number)

### `/kit:explore` — Codebase Map Refresh

1. Reads existing `CODEBASE.md` (if any) and notes custom sections
2. Re-analyzes the full project structure — directories, key files, configs, architecture
3. Regenerates `CODEBASE.md` with updated structural map (paths, one-line descriptions, relationships)
4. Preserves any custom sections from the previous version

`CODEBASE.md` is also auto-updated after tasks that change project structure. This command forces a full refresh.

### `/kit:summary` — Work Wrap-Up

Prints (never commits or posts) two things for the work in progress:

1. Gathers the diff — local uncommitted changes by default, or the whole branch vs base with `--bdiff`
2. Identifies a ticket/title (from `$ARGUMENTS` or the branch name)
3. Optionally runs lint/typecheck to fill a `Checks:` line
4. Proposes **2–3 conventional commit message options**
5. Produces a **wrap-up comment** in a fixed structure — `Code Review — <TICKET>` / `Verdict:` / `Summary:` / `Checks:` — with the mechanism named and known residuals called out

### `/kit:onboard-agents` — Agent Scaffolding

Installs a tailored set of agents into the current project's `.claude/agents/`. Project-agnostic — works for any stack (iOS, Android, web, backend, SDKs).

1. Auto-detects the project profile (language, repo layout, test framework, package manager) from files like `Package.swift`, `build.gradle`, `package.json`, `Cargo.toml`, `go.mod`, lock files, and test configs
2. Confirms the profile with the user via interactive questions
3. Recommends a curated set of **archetypes** — `code-structure-researcher`, `spec-researcher`, `architecture-reviewer`, `senior-developer`, `team-lead-orchestrator`, `e2e-tester`, `design-researcher`, `project-driver`
4. Personalizes each chosen archetype by filling stack-specific placeholders ({{LANGUAGE}}, {{STACK}}, {{TEST_RUNNER}}, etc.) with the user's answers
5. Writes the files to `.claude/agents/`, asking per-file when something already exists (overwrite / skip / show diff)
6. Optionally walks through a custom-agent wizard for anything the archetypes don't cover

The `project-driver` archetype is the headline "agent who knows your UI/SDK" — you name it, point at the launch path, and the wizard scaffolds an agent tailored to that surface.

## Updating

### Via UI

Open the plugin manager with `/plugin`, then navigate to the **Marketplaces** tab and select **Update marketplace** to refresh the registry. Then switch to the **Installed** tab, select the `kit` plugin and click **Update now**.

### Via Commands

```bash
# 1. Refresh the marketplace registry
/plugin marketplace update ai-kit

# 2. Update the plugin to the latest version
/plugin update kit@ai-kit
```

## Plugin Structure

```
plugins/kit/
  .claude-plugin/plugin.json
  commands/
    claudify.md          # Project setup with interactive language selection
    explore.md           # Force refresh CODEBASE.md structural map
    review.md            # React/frontend PR review
    review-local.md      # Local-first review (--ldiff/--bdiff/--full) — findings in conversation, optional GitHub publish
    summary.md           # Commit message options + wrap-up comment
    onboard-agents.md    # Scaffold project-agnostic agent set into .claude/agents/
  agents/
    frontend-reviewer.md     # React/frontend review specialist (bugs, hooks, perf, a11y)
    architecture-reviewer.md # Architecture & codebase-quality specialist (structure, consistency, duplication)
  templates/
    agents/
      archetypes/        # Project-agnostic agent skeletons with {{...}} placeholders
```

## Testing Locally

```
cd /path/to/this/repo
claude
/plugin marketplace add .
/plugin install kit@ai-kit
```

Then run `/kit:claudify` in any project to set up, `/kit:review <PR#>` or `/kit:review-local` to review, `/kit:summary` to wrap up work, and `/kit:onboard-agents` to scaffold project-specific agents.
