---
description: Documents the change, creates a git flow branch, commits and pushes
mode: all
temperature: 0.7
permission:
  read: allow
  glob: allow
  grep: allow
  list: allow
  edit: ask
---

# `commit` mode

Closes a feature that has already been built and validated: generates the
documentation, creates the git flow branch, commits and pushes it to the remote.

## When to use it

At the end of the `build -> test -> review` flow, when the feature is ready to
enter the repository. It does not write to or modify the feature's code.

## Input

Always ask the user for these before running any command:

- `<branch-type>`: one of `feature`, `fix`, `hotfix`, `release`.
- `<description>`: commit subject in lowercase and present tense (e.g.
  `adds a persistent cart for authenticated users`). It must be non-empty.

Also collect `<name>`: kebab-case slug for the branch (e.g. `persistent-cart`).

If the user wants to skip the branch type, the description or the name, do not
invent it yet: continue to step 1, then auto-assign it by reviewing the
generated docs (see "Auto-assignment").

## Commit type by branch

`.gitlab-ci.yml` allows commit types
`feat|fix|hotfix|refactor|chore|docs|test|ci`. `release` is not a valid commit
type, so map the branch type to a commit type:

  | branch type | branch prefix | commit type |
  | ----------- | ------------- | ----------- |
  | `feature`   | `feature/`    | `feat`      |
  | `fix`       | `fix/`        | `fix`       |
  | `hotfix`    | `hotfix/`     | `hotfix`    |
  | `release`   | `release/`    | `chore`     |

## Auto-assignment

After `/document` finishes, if `<branch-type>`, `<description>` or `<name>` was
skipped, read `.agents/changes/YYYY-MM-DD-c<N>-changelog.md` and
`-specification.md` and choose:

- **branch type**: `fix` for a defect correction, `hotfix` for an urgent
  production fix, `release` for release preparation, otherwise `feature`.
- **description**: one-line, lowercase, present-tense summary of the change, fit
  for a commit subject.
- **name**: kebab-case slug derived from the description when it was not given.

Show the chosen values to the user before creating the branch or committing.

## Steps

Run the steps in order. If one fails, stop and report the error; do not
continue.

### 0. Pre-checks

1. `git status --short` and `git branch --show-current`.
2. Confirm there are changes to commit. If the working tree is clean, say so and
   finish.
3. Confirm the current branch is `develop` (the git flow base). If it is not,
   ask before continuing.
4. Review the file list from `git status`. If anything suspicious appears
   (`.env`, keys, credentials, dumps, `node_modules`, large binaries), stop and
   ask before running `git add .`.

### 1. Document

Run opencode's `/document` command to update the feature documentation. Wait for
it to finish. It must create **both** new files in `.agents/changes/`:

- `YYYY-MM-DD-c<N>-changelog.md`
- `YYYY-MM-DD-c<N>-specification.md`

`.gitlab-ci.yml`'s `validate:changes-docs` job requires a new, matching pair,
otherwise the pipeline fails. Verify them with
`git status --short .agents/changes/` before continuing; if either is missing,
stop and report it. Any documentation files it creates or modifies go into the
same commit.

### 2. Create the branch

```bash
git checkout -b <branch-type>/<name>
```

### 3. Commit

```bash
git add .
git commit -m "<commit-type>: <description>"
```

Before committing, show `git status --short` one last time so the summary is
faithful to what is being included. The commit subject must match
`.gitlab-ci.yml`'s
`^(feat|fix|hotfix|refactor|chore|docs|test|ci)(\(.+\))?: .+`.

### 4. Push

```bash
git push -u origin <branch-type>/<name>
```

The mode is not done until the branch is on the remote. Show the remote URL it
printed. If the push is rejected (permissions, protected branch, hooks), report
the error and stop; do not retry with `--force`.

## Rules

- Branch prefixes allowed by `.gitlab-ci.yml`: `feature/`, `fix/`, `hotfix/`,
  `release/`. Always ask the user which one to use.
- Commit type must be one of `feat`, `fix`, `hotfix`, `refactor`, `chore`,
  `docs`, `test`, `ci` (Conventional Commits), derived from the branch type as
  per the table above. Single-line message, always `<type>: <description>` with
  a space after the colon.
- Every commit between `origin/develop` and `HEAD` is validated by the pipeline,
  not just the last one. Check with `git log origin/develop..HEAD --format="%s"`
  that all subjects match the convention.
- Push the branch to the remote with `git push -u origin <branch-type>/<name>`.
  Do not open a PR and do not merge into `develop`. The mode ends after a
  successful push.
- Do not use `--no-verify` or `--force`. If a hook fails, report the error and
  stop.
- Do not modify code beyond what `/document` touches.

## Output

When done, reply only with:

- Branch created and pushed to `origin`
- Short hash and commit message
- Files included (summary)
- Suggested next step: open a PR towards `develop` (`feature/` and `fix/`) or
  `main` (`hotfix/` and `release/`)
