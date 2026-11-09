---
name: refactoring
description: Restructure code while preserving behavior. Use for extraction, consolidation, ownership moves, or cleanup where verification must bracket structural edits.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# Refactoring

Define the behavior-preservation boundary and establish verification before any structural edit. Align every change with the repository's own coding standards.

## 1. Load repository context first

Before touching code, look for `CODING_STANDARDS.md`:

- Search the repo root first, then `docs/`, `.github/`, and any directory that contains the code being refactored (nested standards files override the root one for their subtree).
- If found, read it fully and treat it as the authority on naming, structure, error handling, testing, dependencies, and architectural boundaries.
- Note any rule that constrains the refactor (e.g. layering, public API conventions, forbidden patterns, test requirements) and follow it in every edit.
- If it is not found, say so briefly, infer conventions from the surrounding code (style, tests, lint/format config), and continue. Do not invent standards or create the file unless asked.
- If the standards conflict with the requested refactor, or the existing code violates them, stop and flag it rather than silently choosing. Do not widen scope to fix unrelated violations.

## 2. Establish the safety net

- Identify the behavior that must not change: public interfaces, failure behavior, ordering, side effects, and compatibility.
- Find the existing verification (tests, type checks, linters, build) and run it before editing to record the baseline.
- If coverage of the affected code is missing, add characterization tests that capture current behavior before restructuring, following the testing rules in `CODING_STANDARDS.md`.

## 3. Refactor in small, verified steps

- Keep feature changes and bug fixes outside the refactor; report them instead of folding them in.
- Move one ownership boundary at a time.
- Preserve public interfaces, failure behavior, ordering, and compatibility unless explicitly scoped otherwise.
- Keep every intermediate state buildable and testable; re-run verification after each step.
- Avoid dependency or configuration growth unless correctness requires it, and respect any dependency policy in `CODING_STANDARDS.md`.
- Follow the repo's conventions for naming, file layout, and comments in all new or moved code.

## 4. Prove and finish

Run the same proof used for the baseline after the change. Stop when behavior matches and the requested structure is achieved. Summarize: what moved, which standards from `CODING_STANDARDS.md` applied, verification results, and any out-of-scope issues noticed but left untouched.
