---
description: Review a PR against project standards — React/frontend focused, posts inline GitHub comments
---

You are reviewing a pull request. You will review the changed lines against React/frontend and general code-quality standards, then post findings to GitHub.

**Input:** $ARGUMENTS (PR number, e.g. `#123` or `123`)

---

## Step 1: Get PR Changes

```bash
gh pr diff $PR_NUMBER
gh pr view $PR_NUMBER --json title,body,baseRefName
```

Review ONLY the changed lines. Do not analyze full files or trace app-wide logic.

---

## Step 2: Apply Review Rulesets

### Common rules (always applied)

#### Bugs & Logic (changed lines only)

- Obvious bugs: null access, wrong comparisons, missing return, typos in variable names
- Missing error handling on new code
- If you SUSPECT a deeper logic problem but can't confirm from the diff alone — flag it as a comment, don't investigate further

#### Security (changed lines only)

- Hardcoded tokens, keys, secrets, PII
- `dangerouslySetInnerHTML` without sanitization
- `eval()`, `new Function()`, dynamic script injection
- User input passed directly to DOM or URLs
- SQL injection, auth bypass vectors

#### Code Style & Consistency (changed lines only)

- Naming doesn't match project conventions
- Dead code or unreachable branches introduced
- Leftover debug code (but do NOT flag console.logs in uncommitted work)
- Overly complex one-liners that hurt readability

---

### React / Frontend ruleset

Delegate detailed analysis to the `frontend-reviewer` agent. Pass the PR diff directly — do not have the agent re-fetch it.

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

## Step 3: Post Inline Comments to GitHub

For each issue found, post an inline comment on the specific line:

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments \
  -f body="<comment>" \
  -f commit_id="<commit_sha>" \
  -f path="<file_path>" \
  -F line=<line_number> \
  -f side="RIGHT"
```

Comment format — prefix with the category that found it:

- **Bug**: [description]
- **Possible logic issue — double-check**: [description]
- **React**: [description]
- **TypeScript**: [description]
- **Performance**: [description]
- **Security**: [description]
- **Style**: [description]

---

## Step 4: Summary Comment

After all inline comments, post ONE summary comment on the PR:

```bash
gh pr comment $PR_NUMBER --body "<summary>"
```

**If no issues:**

> Self-review complete. No issues found.

**If only minor/uncertain issues:**

> Self-review complete. No critical issues, but double-check:
>
> - [brief list of what to look at and why]

**If issues found:**

> Self-review complete. Found [N] issues:
>
> - [count] bugs
> - [count] React/frontend
> - [count] TypeScript
> - [count] performance
> - [count] security
> - [count] style
>
> See inline comments for details.

---

## Rules

- ONLY review changed lines — never review unchanged code
- Do NOT remove console.logs unless developer asks
- Be specific: always reference file and line
- If unsure whether something is a bug or intentional — post as a question, not as a bug
- Keep comments concise, one issue per comment
- Do not post comments for things that are clearly intentional project patterns
- When delegating to the reviewer agent, pass the PR diff — do not have it re-fetch
