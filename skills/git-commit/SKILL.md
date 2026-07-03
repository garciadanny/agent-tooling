---
name: git-commit
description: Commit staged or unstaged git changes with a well-crafted conventional commit message. Invoke whenever the user says "commit", "commit my changes", "commit this", "make a commit", "/commit", or anything implying they want to record their current work into git history. Also use when the user says "commit and push" — handle the commit first, then push. Always use this skill even if the user doesn't mention conventional commits or a message format.
---

# git-commit

Commit the current working changes with a conventional commit message that has a clear title and a descriptive body.

## Hard rules — never break these

- **Never include `Co-Authored-By` or any Claude/AI authorship lines** in any commit message, ever.
- **Always include a body.** A title-only commit is never acceptable. Every commit must have a blank line after the title followed by 1–2 paragraphs explaining the *why* and *what* of the change.
- **Always use conventional commit format** for the title.
- **Never force-push under any circumstances.** Rewriting pushed history is their call, not yours.

## Step 1 — Understand what changed

Run these in parallel:

```bash
git status
git diff          # unstaged changes
git diff --cached # staged changes
```

Read the full diff carefully. Your commit message is only as good as your understanding of the change.

## Step 2 — Apply any extra instructions

The user may have passed arguments like:
- `ignore test files` → don't stage files matching `spec/`, `test/`, `*.test.*`, etc.
- `only stage app/frontend` → only `git add` files under that path
- `skip migrations` → exclude `db/migrate/`

Honor these precisely. If the instruction is ambiguous, ask before staging anything.

## Step 3 — Stage the right files

By default, stage all unstaged changes:

```bash
git add -A
```

If extra instructions narrow the scope, use targeted `git add <path>` calls instead of `-A`. Never stage files the user explicitly excluded.

## Step 4 — Write the commit message

### Title format (conventional commits)

```
<type>[optional scope]: <description>
```

**Types:**
- `feat` — new feature or capability
- `fix` — bug fix
- `refactor` — restructuring with no behavior change
- `perf` — performance improvement
- `test` — adding or updating tests
- `docs` — documentation only
- `style` — formatting, whitespace, no logic change
- `build` — build system or dependency changes
- `ci` — CI/CD config changes
- `chore` — maintenance tasks that don't fit above

**Scope** is optional but useful — use the module, domain, or component name (e.g., `feat(auth):`, `fix(proposals):`).

**Breaking changes:** append `!` after type/scope (e.g., `feat(api)!:`), and add a `BREAKING CHANGE:` footer.

**Title rules:**
- Imperative mood ("add", "fix", "remove" — not "added", "fixes", "removing")
- 72 characters max
- No period at the end
- Lowercase after the colon

### Body (required — never skip)

One blank line after the title, then 1–2 paragraphs. The body should answer:
- **Why** was this change needed? (the problem or motivation)
- **What** does this change do at a meaningful level? (not just restating the title)
- Any non-obvious consequences, constraints, or context a future reader would want to know

Be specific. "Fix a bug" is not a body. "Removes the `is_custom_allocation_search` param from SRL calls because the API deprecated it in v3 and started returning 400s" is a body.

Keep each paragraph under ~4 sentences. Don't pad — if one tight paragraph covers it, that's fine.

### Footer (optional)

Add footers after a blank line following the body:
- `BREAKING CHANGE: <description>` for breaking changes
- `Fixes #123` or `Closes #456` for issue references
- Only include if genuinely applicable

## Step 5 — Commit

Always use a heredoc to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
<type>[scope]: <description>

<body paragraph 1>

<body paragraph 2 if needed>
EOF
)"
```

Then run `git status` to confirm the commit landed cleanly.

## Example commit messages

**Simple fix:**
```
fix(proposals): correct recommended model filtering in SRL results

The SRL search was returning unfiltered results because the recommended
model flag was being applied after pagination rather than as a query
constraint. Moved the filter upstream so only matching models are
counted and returned.
```

**Feature with scope:**
```
feat(trading): add tax-lot selection to account rebalancing

Rebalancing now respects tax-lot preferences when generating trades.
Users can specify FIFO, LIFO, or highest-cost-basis lot selection per
account, and the trade engine will honor those preferences when
deciding which lots to sell.

This required adding a `tax_lot_method` column to `trading_accounts`
and a corresponding UI control in the account settings panel.
```

**Breaking change:**
```
feat(api)!: remove deprecated is_custom_allocation_search param

The SRL API no longer accepts `is_custom_allocation_search` as of v3.
Calls including this param now return 400. Removed it from all SRL
request builders across the codebase.

BREAKING CHANGE: any external callers passing is_custom_allocation_search
must remove the param before upgrading.
```
