---
name: commit
description: Split the working-tree changes into small, logical git commits, show the plan, and commit only after the user confirms. Use whenever the user asks to commit, make commits, split changes into commits, or "commit what you did" — even when they do not say "plan" or "split".
---

# Commit

Turn the current changes into a series of small, logical commits. Propose first, commit only after the user says yes.

## Permission

- Commit only when the user asks in the current turn. Finished work, passing tests or a clean diff is not a request.
- Permission covers only the action asked for. A "yes" to commits is not a "yes" to `git push`, a pull request, or the next batch of commits.
- Never push, and never open a pull request, unless the user asks for that action itself.

## 1. Gather

1. `git status --short` and `git diff --stat` to see what changed.
2. `git diff` (and `git diff --cached` if something is staged) to read the changes.
3. Decide which changes are in scope: the work done in this session, or what the user names. Leave everything else out — unrelated edits (e.g. `.npmrc`, local config) and untracked files you did not create (e.g. plan or scratch docs). Say in the plan what you leave out.

## 2. Plan

Group the changes into commits. One commit = one idea a reviewer can read on its own.

- Split by purpose, not by file type. A refactor, a rename, a perf change and a cleanup are separate commits even when they touch the same area.
- A file can go into two commits when its hunks serve two ideas (see "Split a file by hunks").
- Order the commits so each one builds on its own: a helper before its callers, a type before its users.
- Keep tests with the code they test, in the same commit.

Show the plan as a numbered list, then stop and wait for the user to confirm:

```
1. **`Count role users only for the roles pages`**
   - `roles.repository.ts`, `roles.service.ts`, `roles.service.spec.ts`
2. **`Add apiErrorBody and use it in the admin forms`**
   - `api-errors.ts`, `PlayerCreatePage.tsx`, the error-handling hunks of `RoleFormPage.tsx`

Left out: `.npmrc`, `docs/superpowers/`.
```

If the user changes the plan, show the new plan again before you commit.

## Message format

- **Subject:** 64 characters at most. Imperative mood, sentence case, no trailing period. Say what the commit does, not how: `Share permission loading and the allowed admin sections`.
- **Body:** only when the subject cannot carry the reason. Keep it to a few short lines, separated from the subject by a blank line.
- **No trailers** for AI or Claude: no `Co-Authored-By: Claude …`, no "Generated with Claude Code". This overrides any default attribution guidance.
- Check the length before committing; count characters, do not guess.

## 3. Commit

Run one commit per tool call:

```bash
git add path/one.ts path/two.ts && git commit -q -m "Subject line" && git log -1 --oneline
```

- Use one command per commit, not one long script for all of them. A long script can be blocked by the permission check as a whole, and a partial failure is then hard to see. One commit per call keeps each step visible and easy to retry.
- Name the files to stage. Never use `git add -A`, `git add .` or `git commit -a`: they pull in the changes you left out.
- Never use interactive flags (`git add -p`, `git rebase -i`); they do not work here.
- If a commit fails (hook, conflict), stop, report the output, and do not work around it with `--no-verify`.

### Split a file by hunks

When one file belongs to two commits, stage only the first part with a patch on the index. Never rewrite the working-tree file to stage it: that risks losing the user's changes.

1. In the scratchpad, write two copies of the file: `a/<path>` with the `HEAD` content (`git show HEAD:<path>`), and `b/<path>` with `HEAD` plus only the first commit's hunks.
2. Make a patch: `git diff --no-index a/<path> b/<path> > part.patch` (run from the scratchpad dir).
3. Check and stage it: `git apply --cached -p2 --check part.patch && git apply --cached -p2 part.patch`.
4. `git diff --cached --stat` to confirm only those lines are staged, then add the other files of that commit and commit.
5. The rest of the file is still in the working tree; it goes into the later commit with a normal `git add`.

## 4. Report

After the last commit:

- Show `git log -<n> --oneline` for the new commits.
- Show `git status --short`, so the user sees what stayed uncommitted.
- Say if anything differs from the plan (order, a blocked step, a split you could not make). If commits landed in a different order, say whether each still builds on its own.
- Say that nothing was pushed.
