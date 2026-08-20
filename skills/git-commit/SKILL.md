---
name: git-commit
description: Commit staged or unstaged git changes with a well-crafted conventional commit message. Invoke whenever the user says "commit", "commit my changes", "commit this", "make a commit", "/commit", or anything implying they want to record their current work into git history. Also use when the user says "commit and push" — handle the commit first, then push. Always use this skill even if the user doesn't mention conventional commits or a message format.
---

# git-commit

Commit the current working changes with a conventional commit message that has a clear title and a descriptive body.

## Hard rules — never break these

- **Never include `Co-Authored-By` or any Claude/AI authorship lines** in any commit message, ever.
- **Keep the title at 72 characters or fewer.** This is GitHub's hard cutoff — GitHub truncates any summary line past 72 characters and hides the rest behind an ellipsis, so a longer title loses information in the UI. Count the characters and rewrite until it fits. Never exceed 72.
- **In commit messages, never use em dashes or dashes as punctuation.** No "—", no "–", and no spaced hyphen (" - ") used as a pause or aside. Rewrite with complete sentences, commas, or parentheses instead. (Ordinary hyphens inside compound words like "tax-lot" or "well-crafted" are fine.) This rule governs the commit messages you write, not this skill doc.
- **Write commit messages like a person, not a changelog bot.** Use plain, direct language. Say what changed and why the way you would explain it to a teammate. Cut filler, hedging, and restated obvious context.
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
- 72 characters max, counting the type and scope. This is GitHub's truncation cutoff, so treat it as a hard limit and count the characters before committing. Aim for around 50 when you can.
- No period at the end
- Lowercase after the colon
- No em dashes or dashes as punctuation in the message
- **Describe the user-visible outcome, not the internal mechanism.** Say what now works and for whom. Naming a policy object, filter, or scope that only makes sense once you've read the code is a rejected title.
- Test before you commit: would a teammate who has never opened these files understand the title? If not, rewrite it around the behavior.

```
❌ fix(proposals): exempt proposals-all-access from the advisor filter
✅ fix(proposals): load households for proposals-all-access users
```

### Body (required — never skip)

One blank line after the title, then 1–2 paragraphs. The body should answer:
- **Why** was this change needed? (the problem or motivation)
- **What** does this change do at a meaningful level? (not just restating the title)
- Any non-obvious consequences, constraints, or context a future reader would want to know

Be specific. "Fix a bug" is not a body. "Removes the `is_custom_allocation_search` param from SRL calls because the API deprecated it in v3 and started returning 400s" is a body.

**Body rules:**
- **Write it like a person would.** Plain, direct sentences. No em dashes or dashes as punctuation; use commas or parentheses instead. Skip filler, boilerplate summaries of the diff, and anything the reader can already see in the title or the code.
- **At most one identifier, and only if it's the thing you changed.** Threading several class, method, and exception names through prose forces the reader to hold a symbol table. Describe the behavior instead: "the household lookup still filtered to households they personally own" beats naming the scope that does the filtering.
- **Report, don't argue.** Cut sentences that defend or characterize the change, like "this is a consistency fix rather than a widening" or "note that…". State what is true and let the reader judge.
- **Read in order, never double back.** One idea per sentence, each following from the last. No mid-sentence justifications, no parentheticals that send the reader backwards.
- **Two paragraphs maximum, roughly four sentences each,** shorter when the change allows. If a sentence can go without losing information a future reader needs, cut it.

Don't pad. If one tight paragraph covers it, that's fine.

### Worked example — rejected vs. approved

The same change, written twice. The first version was rejected as "hard to follow, too wordy, and the title doesn't make sense."

**❌ Rejected**

```
fix(proposals): exempt proposals-all-access from the advisor filter

A user holding Persona-ProposalsAllAccess alongside a base advisor role could open any
proposal in the firm, because Proposals::ProposalPolicy::Scope drops the user filter for
that role, but could not load that proposal's household: TenantScope#user_scope narrows
anyone holding "advisor" to their own records, and roles are multi-valued. The household
lookup raised RecordNotFound, which the missing /404 route turned into a redirect, so the
frontend resolved a 200 with an unrelated body and crashed reading household.clients.

Adding the role to the early return drops the user filter while leaving tenant isolation
intact. This is a consistency fix rather than a widening: a user holding only
Persona-ProposalsAllAccess already had firm-wide visibility, since the role was absent from
user_scope and fell through to nil, so today the same persona sees the whole firm or just
its own records depending on which unrelated roles accompany it. Note the scope is shared,
so this also widens Proposals::AdvisoryFeeSchedulePolicy and the Advisor Assistant
household tools, and it restores the proposal's own household to the setup page's Client
Relationship dropdown.
```

Five identifiers plus an exception name; a title naming a filter no reader can picture; a
second paragraph that stops to defend itself mid-sentence.

**✅ Approved**

```
fix(proposals): load households for proposals-all-access users

Persona-ProposalsAllAccess users can open any proposal in their firm, but the household
lookup still filtered to households they personally own. Opening another advisor's proposal
therefore found no household, and the failed lookup reached the browser as a crash and an
error banner rather than a real error, because the app has no /404 route and Rails turns the
404 into a redirect the browser silently follows.

TenantScope#user_scope now skips the owner filter for this role, keeping tenant isolation
intact. The role already behaved this way when held on its own; the filter only applied
because these users also carry the base advisor role. The scope is shared, so advisory fee
schedules and the Advisor Assistant household tools widen the same way, and the proposal's
household reappears in the setup page dropdown.
```

One identifier, the method actually changed. Every consequence stated plainly, none
argued. The same information, read start to finish without backtracking.

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
