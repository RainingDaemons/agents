---
description: Generate changelog and specification docs from current changes
# model: <provider/model-id>
---

Generate two documents based on the current changes in this repository. Use a versioned filename pattern that
increments on each new commit: `YYYY-MM-DD-c<N>-changelog.md` and `YYYY-MM-DD-c<N>-specification.md`.

## Version resolution

List `.agents/changes/` to find the highest existing `c<N>` counter for today's date. If no file exists for
today, start at `c1`. If a new commit is generated during this session, the next document run must use `c<N+1>`.

## Git diff (HEAD)
!`git diff HEAD`

## Git status
!`git status`

## Git diff --stat
!`git diff --stat`

## Changes directory
!`ls .agents/changes/`

Read all modified files listed above, then:

1. Create `.agents/changes/YYYY-MM-DD-c<N>-changelog.md` containing:
   - **Resume**: short summary of the changes
   - **Affected files**: list of modified files
   - **TODO**: pending items

2. Create `.agents/changes/YYYY-MM-DD-c<N>-specification.md` containing:
   - **Problem solved**
   - **Solution implemented**
   - **Alternatives considered**
   - **Election reason**
   - **Consecuences**
   - **Future work**

Use today's date in YYYY-MM-DD format for the filenames. Create the `.agents/changes/` directory if it doesn't exist.
