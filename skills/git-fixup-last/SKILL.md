---
name: git-fixup-last
description: Fold the current changes into the previous commit (the last commit) without changing its message, using git fixup + autosquash. Invoke whenever the user says "add this to the last commit", "amend the previous commit", "fixup", "fold this into the last commit", "squash this into HEAD", "/fixup", or anything implying they want to absorb their current work into the most recent commit rather than create a new one. If the user instead wants a brand-new commit, use the git-commit skill.
disable-model-invocation: true
---

# git-fixup-last

Absorb the current working changes into the **previous commit** (`HEAD`) while keeping that commit's original message intact. This is the "I forgot something in my last commit" workflow, done cleanly with `git commit --fixup` followed by an autosquash rebase — so the fix lands in the right commit instead of trailing behind as a separate "oops" commit.

## When to use this vs. git-commit

- Use **git-fixup-last** when the changes belong in the commit you _just made_ — a missed file, a typo fix, a review nit — and you don't want a new commit or a changed message.
- Use **git-commit** when the changes deserve their own commit with their own message.

## Step 1 — Stage the right changes (delegate to git-commit)

Understanding what changed and staging exactly the right files is identical to a normal commit, so **reuse the git-commit skill's staging logic** rather than reinventing it. Read `../git-commit/SKILL.md` and follow it up to and including staging the right files.

Then stop and come back here. git-commit continues on to write a message and create a new commit — you must **not** do that, because you're going to fold these changes into the existing commit instead (don't do steps 4 or 5).

If nothing is staged after this step, there's nothing to fold in — tell the user and stop.

## Step 2 — Identify the target commit

The target is the current tip:

```bash
git rev-parse HEAD          # sha to fix up
git log -1 --oneline        # confirm it's the commit the user means
```

Confirm this is genuinely the commit the changes belong to. If the user said "the last commit" this is it; if they named a different commit, use that sha instead (the autosquash mechanics below still work, you'll just widen the rebase range in Step 4).

### Safety check: is the target already pushed?

Rewriting a commit that's already on a remote means anyone who pulled it will diverge, and you'll need a force-push. Check:

```bash
git branch -r --contains HEAD
```

If the target commit already exists on a remote branch, **warn the user** before proceeding: this rewrites published history and will require a `git push --force-with-lease`. Let them decide whether to continue.

## Step 3 — Create the fixup commit

```bash
git commit --fixup HEAD
```

This creates a commit whose message is `fixup! <original title>`. It's a marker: the autosquash step in the next step will melt it into the target and discard the marker message. (If you're targeting a commit other than the tip, use `git commit --fixup <sha>` with that sha.)

## Step 4 — Autosquash into the target

```bash
git rebase --autosquash HEAD~2
```

`--autosquash` finds the `fixup!` marker and folds it into the target. `HEAD~2` covers the two commits involved: the original target and the fixup you just made. **Widen the range** if you targeted an older commit — the range must reach back one commit _before_ the target. For example, to fix up the commit that is 3 back, use `HEAD~4`.

**Edge case — the target is the repo's very first commit** (`HEAD~2` fails with "unknown revision"). In that case there's no parent to rebase onto, so rebase from the root:

```bash
git rebase --autosquash --root
```

## Step 5 — Verify

```bash
git log -2 --oneline
git status
```

Confirm that:

- No `fixup!` commit remains (it should have been absorbed).
- The target commit's original message is unchanged.
- The working tree is clean and no rebase is in progress.

If the rebase stopped on a conflict, `git status` will say so. In this case, do `git rebase --abort` to back out entirely — the fixup commit will still exist and can be reset away), then
flag this to the user.

If the target was already pushed (Step 2), remind the user they'll need:

```bash
git push --force-with-lease
```

## Hard rules

- **Never change the target commit's message.** The whole point is to preserve it. `--fixup` + autosquash guarantees this; don't substitute `git commit --amend` with a new message.
- **Never include `Co-Authored-By` or any Claude/AI authorship lines** — consistent with git-commit.
- **Never force-push under any circumstances.** Rewriting pushed history is their call, not yours.
