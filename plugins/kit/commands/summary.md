---
description: Propose commit message options and a wrap-up comment summarizing the work done
---

Summarize the work in progress: propose conventional commit message options and produce a wrap-up review comment describing what was done. This command **only prints** — it never commits, pushes, or posts anything automatically.

**Input:** `$ARGUMENTS` — optional and all free-form:
- an explicit ticket/issue id (e.g. `SLW-6769`)
- a title override
- `--bdiff` to summarize the whole branch vs base (default is local uncommitted changes)

---

## Step 1: Gather the work

Determine the scope, mirroring `/kit:review-local`:

- **Default (local):** `git diff HEAD` (staged + unstaged) plus `git diff HEAD --stat` for the shape of the change.
- **`--bdiff`:** resolve the base (`@{upstream}`, else first existing of `origin/main`, `origin/master`, `origin/develop`), then `git diff $(git merge-base $BASE HEAD)` plus `git log $BASE..HEAD --oneline` for the commit list.

Read enough of the diff to understand **what** changed and **why** — the problem being solved, the mechanism, the key files/functions touched. Do not narrate line by line.

If there is nothing to summarize (empty diff and no commits), tell the user and stop.

---

## Step 2: Identify the ticket / title

- If `$ARGUMENTS` contains an explicit ticket id, use it.
- Otherwise try to extract one from the current branch name — pattern like `ABC-1234` (`git rev-parse --abbrev-ref HEAD`).
- If none is found, derive a short descriptive title from the change, or ask the user for one.

This value fills the header line: `Code Review — <TICKET or TITLE>`.

---

## Step 3: Determine checks

Fill the `Checks:` line with lint + type results when available:

1. Detect scripts (e.g. `package.json` → `lint`, `typecheck`/`tsc`, `build`), or project equivalents.
2. Offer to run them and capture the results (error counts). If the user declines, ask them to provide known results, or write `Checks: not run`.

Keep this lightweight — never block the summary on a long build.

---

## Step 4: Propose commit message options

Produce **2–3** options in conventional-commit format:

```
<type>(<scope>): <description>
```

Types: `feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `ci`, `perf`, `style`, `build`.

Vary them in emphasis/granularity (e.g. one terse, one that names the mechanism, one scoped differently) so the user can pick or edit. Prefix the subject with the ticket id in the scope or body if the project convention uses it.

---

## Step 5: Produce the wrap-up comment

Write a wrap-up comment in this exact structure (this is the template — match it):

```
Code Review — <TICKET or TITLE>

Verdict: <Approved | Approved with comments | Changes requested>.

Summary: <2–5 sentences in plain English: the problem being closed, the mechanism/how (name the key files, hooks, functions), any changes batched during the work, and any known residual or optional follow-up>.

Checks: <lint result, type result> (e.g. "ESLint 0 errors, TypeScript 0 errors").
```

Guidance:
- **Verdict** — default to `Approved` for a clean self-summary; use `Approved with comments` if there are minor caveats, `Changes requested` if something is unresolved.
- **Summary** — lead with what the change accomplishes for the user/QA, then the mechanism. Name the concrete pieces (files, hooks, components). Explicitly call out **known residuals** and anything deliberately left as a follow-up, so nothing looks hidden.
- **Checks** — reflect the real results from Step 3.

---

## Step 6: Output

Print two clearly separated, copy-pasteable blocks:

1. **Commit options** — the 2–3 candidates.
2. **Wrap-up comment** — the filled template from Step 5.

Then stop — let the user pick and copy whichever commit message and comment they want.

---

## Rules

- Print only — never commit, push, or post to GitHub automatically.
- Keep the Summary factual and grounded in the actual diff — do not invent changes or checks that weren't made.
- Always surface known residuals / follow-ups; never present partial work as complete.
