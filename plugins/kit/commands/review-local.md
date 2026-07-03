---
description: Review local diff or branch changes against project standards (line-level, or --arch for a high-level architecture review) — prints findings in conversation, optionally publishes to GitHub if a PR exists
---

You are reviewing code changes. You will review the changes against React/frontend and general code-quality standards and print findings in the conversation. If a PR exists, you will offer to publish findings to GitHub.

**Input:** `$ARGUMENTS` — optional PR number or URL (e.g. `123`, `#123`, PR URL) and optional flags.

### Flags

| Flag | Meaning |
| --- | --- |
| `--ldiff` | **Local diff (default).** Review staged + working-tree changes only (`git diff HEAD`). |
| `--bdiff` | **Branch diff.** `--ldiff` plus committed changes vs the base branch (origin/main → master → develop). |
| `--full` | Review the **full content of each changed file**, not just the changed lines. Off by default. |
| `--arch` | **Architecture mode.** Run a high-level architecture & codebase-quality review (structure, component organization, consistency, duplication) instead of the standard line-level review. Defaults to `--bdiff` scope and writes the report to `ARCHITECTURE_REVIEW.md`. |

- `--ldiff` and `--bdiff` are mutually exclusive; if both are passed, `--bdiff` wins.
- `--full` is orthogonal — combine it with either scope (e.g. `--bdiff --full`).
- `--arch` is a mode toggle. When set, scope defaults to `--bdiff` (unless the user explicitly passes `--ldiff`), and the standard rulesets are skipped in favor of **Architecture Mode** (see below).

---

## Step 0: Parse flags and determine review target

### 0a. Parse flags

From `$ARGUMENTS`, extract:
- **Mode**: `--arch` present → Architecture Mode; otherwise standard review.
- **Scope**: `--bdiff` if present, else `--ldiff` (default). Exception: in Architecture Mode, scope defaults to `--bdiff` unless `--ldiff` is explicitly passed.
- **Depth**: `--full` present → whole changed files; otherwise changed lines only (default).
- **PR number/URL** if present (strip `#`, or parse from URL).

### 0b. Resolve the target branch (only if a PR number was given)

1. Get the PR's head branch: `gh pr view $PR_NUMBER --json headRefName -q .headRefName`
2. Get the current branch: `git rev-parse --abbrev-ref HEAD`
3. **If PR branch == current branch** → continue on the current branch.
4. **If PR branch != current branch**:
   - Check for uncommitted changes (`git diff HEAD`, `git diff --cached`).
   - If changes exist → ask: "Stash, commit, or cancel?" (stash → `git stash`; commit → prompt for a message and commit; cancel → stop).
   - Ask the user to check out the PR branch:
     > "Please run: `git checkout <branch>` and confirm when ready."
   - Wait for confirmation, then continue.

If no PR number was given, `$PR_NUMBER` is empty and you operate on the current branch.

### 0c. Compute `$DIFF` from the scope

- **`--ldiff` (default):**
  ```bash
  git diff HEAD
  ```
  Captures staged + unstaged changes.

- **`--bdiff`:** resolve the base branch `$BASE`, then diff from the merge-base to the working tree:
  ```bash
  # $BASE: first try the upstream tracking ref, else first existing of the candidates
  git rev-parse --abbrev-ref @{upstream} 2>/dev/null
  # otherwise, in priority order: origin/main, origin/master, origin/develop
  git diff $(git merge-base $BASE HEAD)
  ```
  Captures committed changes on this branch since it diverged from base **plus** staged + unstaged changes.

Use the result as `$DIFF`. If `$DIFF` is empty (nothing to review) — tell the user and stop.

### 0d. If `--full` — load whole changed files

Get the list of changed files for the same scope:
```bash
# --ldiff
git diff HEAD --name-only
# --bdiff
git diff $(git merge-base $BASE HEAD) --name-only
```

`Read` each changed file in full (skip deleted and binary files). In `--full` mode the reviewer evaluates the **entire content** of each changed file — using `$DIFF` to know what changed, but free to flag issues anywhere in those files. In default (non-`--full`) mode, restrict findings to changed lines only.

Tell the user which **scope** (`ldiff`/`bdiff`) and **depth** (lines / full-file) were used.

---

## Architecture Mode — only when `--arch` is set

When `--arch` is passed, **do not** run the standard rulesets (Steps 1–3). Instead:

1. Scope is `--bdiff` by default (unless the user explicitly passed `--ldiff`). Use the `$DIFF` and changed-file list from Step 0.
2. Delegate the review to the **`architecture-reviewer`** agent. Give it:
   - the changed-file list and `$DIFF`
   - explicit permission to read neighboring / related files and to Glob/Grep the wider codebase to detect duplication, "reinvented wheels", and inconsistency
3. Take the markdown report the agent returns and **write it to `ARCHITECTURE_REVIEW.md` in the project root**, overwriting any existing file.
4. Print a short summary in the conversation — verdict, finding counts by theme, and: *"Full report written to `ARCHITECTURE_REVIEW.md`."*
5. Ask: *"Want me to act on any of these, or leave the report as-is?"* — do **not** auto-refactor.

Architecture Mode does **not** post to GitHub — its only output is the local `ARCHITECTURE_REVIEW.md`. Stop here; skip Steps 1–3.

---

## Step 1: Apply Review Rulesets *(standard mode — skip if `--arch`)*

Operate on `$DIFF` (and, in `--full` mode, the full contents of the changed files).

### Common rules (always applied)

#### Bugs & Logic

- Obvious bugs: null access, wrong comparisons, missing return, typos in variable names
- Missing error handling on new code
- If you SUSPECT a deeper logic problem but can't confirm — flag it as a comment, don't investigate further

#### Security

- Hardcoded tokens, keys, secrets, PII
- `dangerouslySetInnerHTML` without sanitization
- `eval()`, `new Function()`, dynamic script injection
- User input passed directly to DOM or URLs
- SQL injection, auth bypass vectors

#### Code Style & Consistency

- Naming doesn't match project conventions
- Dead code or unreachable branches introduced
- Leftover debug code (but do NOT flag console.logs in uncommitted work)
- Overly complex one-liners that hurt readability

---

### React / Frontend ruleset

Delegate detailed analysis to the `frontend-reviewer` agent. Pass `$DIFF` (and full file contents in `--full` mode) directly — do not have the agent re-fetch it.

Key checks:
- Missing or incorrect dependency arrays in useEffect/useMemo/useCallback
- State updates that could cause infinite re-renders
- Missing cleanup in useEffect (event listeners, timers, subscriptions)
- Unnecessary re-renders — missing memoization on expensive components
- Unstable references (inline objects/functions) passed to memoized children
- Missing key props in lists
- Accessibility issues: missing aria labels, non-semantic elements, keyboard navigation, focus management
- TypeScript: `any` escape hatches, unsafe `as` casts, unhandled nullables
- Component size — flag components over ~200 lines that should be split
- Bundle size — large imports for small features, missing code splitting / lazy loading

---

## Step 2: Print Findings in Conversation

Output a markdown report directly in the conversation. No GitHub API calls in this step.

**If no issues:**

> Self-review complete. No issues found.
>
> Scope: [ldiff | bdiff] · Depth: [changed lines | full files]

**If issues found:**

```
## Review Findings
Scope: bdiff · Depth: full files

### `path/to/file.tsx` line 42
**Bug**: description of issue

### `path/to/file.tsx` line 87
**React**: description of issue

---
**Summary**: Found N issues — X bugs, Y React, Z TypeScript, W style
```

Finding format — prefix with the category:

- **Bug**: [description]
- **Possible logic issue — double-check**: [description]
- **React**: [description]
- **TypeScript**: [description]
- **Performance**: [description]
- **Security**: [description]
- **Style**: [description]

---

## Step 3: Ask Developer

### If `$PR_NUMBER` is set:

> "Review complete. What next?"
> 1. **Publish findings** — post inline comments to the PR on GitHub
> 2. **Fix + publish** — fix issues, commit, push, then post a summary comment to the PR
> 3. **Leave as-is** — done

### If no `$PR_NUMBER`:

> "Review complete. Want me to fix the issues or leave as-is?"
> 1. **Fix** — fix issues, remind user to commit
> 2. **Leave as-is** — done

---

### On "Publish findings":

Get the latest commit SHA on the PR:
```bash
gh pr view $PR_NUMBER --json headRefOid -q .headRefOid
```

Get the repo owner/name:
```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

For each finding, post an inline comment:
```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments \
  -f body="<comment>" \
  -f commit_id="<headRefOid>" \
  -f path="<file_path>" \
  -F line=<line_number> \
  -f side="RIGHT"
```

Then post one summary comment:
```bash
gh pr comment $PR_NUMBER --body "<summary>"
```

Summary format mirrors Step 2 but condensed (counts only, with "See inline comments for details.").

**Note:** GitHub inline comments require the line to be part of the PR diff. In `--full` mode, findings on unchanged lines can't be posted inline — include them in the summary comment instead.

---

### On "Fix + publish":

1. Fix all issues on the branch
2. Commit:
   ```bash
   git commit -m "fix: self-review fixes (PR #$PR_NUMBER)"
   ```
3. Push: `git push`
4. Post a PR comment summarizing what was fixed and what needs manual verification:
   ```bash
   gh pr comment $PR_NUMBER --body "<summary>"
   ```

### On "Fix" (no PR):

1. Fix all issues on the branch
2. Remind user: "Fixes applied. Remember to commit and push when ready."

---

## Rules

- In default mode, review only changed lines. In `--full` mode, review the whole content of each changed file.
- Do NOT autofix without asking first
- Do NOT remove console.logs unless developer asks
- Be specific: always reference file and line
- If unsure whether something is a bug or intentional — post as a question, not as a bug
- Keep findings concise, one issue per finding
- Do not flag things that are clearly intentional project patterns
- When delegating to the reviewer agent, pass `$DIFF` directly — do not have it re-fetch
