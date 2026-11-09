---
name: adr-writer
description: Write an Architecture Decision Record (ADR) as a Markdown file. Use when the user asks to create, draft, document or record an architecture/technical decision, or mentions "ADR", "decision record" or "registro de decisión de arquitectura".
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated combining Michael Nygard's minimal template (Status, Context, Decision, Consequences) with EdgeX Foundry's traceability sections (Submitters, Change Log, Use Cases, Proposed Design, Considerations, Related ADRs, References)
---

# ADR Writer

One ADR = one decision = one Markdown file.

## What an ADR must do

- Explain **why** the decision was made, not just what the final design looks like.
- Be understandable six months later without asking anyone: constraints, rejected options, trade-offs.

## Format (hybrid)

- **core** (always): Title, Status, Context, Decision, Consequences.
- **additions** (only when they add value): Submitters, Change Log, Referenced Use Cases / Requirements, Proposed Design, Considerations, Related ADRs, References.

## 1. Decide whether an ADR is warranted

- **Write one** when the decision is hard to reverse, affects multiple components or teams, sets a pattern others will follow, or someone will later ask "why did we do it this way?".
- **Examples:** database, auth strategy, deployment model, service boundary, framework, public API contract.
- **Skip it** for a bug fix, routine refactor or trivial dependency bump — say so in one sentence and offer a plain issue/PR description instead.
- **Bundled decisions** → one ADR per decision, or ask which to start with.

## 2. Gather inputs before writing

Collect from the request, the conversation and the repository:

- the decision (being made or already made);
- the context: problem, constraints, forces, deadlines, non-functional requirements;
- the options considered and why each was accepted or rejected;
- consequences the user already knows about;
- metadata: submitters, date, related ADRs, tickets/PRs, use cases.

Rules:

- Inspect the repo first (README, existing ADRs, config, docs, `git log`) to fill gaps instead of asking.
- Ask only **blocking** gaps in one short message — usually the decision itself or the alternatives. Never ask about optional metadata.
- Never fabricate names, organizations, URLs, ticket numbers, benchmarks, past dates or use cases. If unknown, omit the field or write `TBD` and flag it in the final message.
- Running unattended: use the most reasonable reading, set Status `Proposed`, and list your assumptions in the final message (not as invented facts in the ADR).

## 3. Locate the file and number it

1. A path or filename given by the user wins.
2. Else use an existing ADR directory: `docs/adr`, `docs/decisions`, `doc/adr`, `doc/architecture/decisions`, `adr`, `decisions`, `architecture/decisions` — follow its filename convention and numbering.
3. Number: next zero-padded 4-digit value in the global sequence (`0007`).
4. Name: `NNNN-short-kebab-case-title.md` (e.g. `0007-use-postgresql-for-orders.md`).
5. No directory and no path given: write `ADR.md` in the current directory with id `ADR-0001`.
6. Never overwrite an ADR. Accepted ADRs are immutable except for Status and Change Log (see section 6).

## 4. Template

Use this structure. Keep headings in English for portability; write content in the **language the user is writing in**.

```markdown
# ADR-NNNN: <Decision as a short noun phrase>

- **Status:** Proposed
- **Date:** YYYY-MM-DD
- **Submitters:**
  - Name (Organization)
- **Deciders:** <optional: who has authority to accept>

## Change Log

- [Proposed](<PR or issue URL, if any>) YYYY-MM-DD

## Referenced Use Cases / Requirements

- [Use case or requirement name](URL)

## Context

<Problem, forces, constraints, requirements, why now. Neutral tone: describe, do not advocate.>

## Proposed Design

<Optional. High-level design without implementation detail.>

- **Services/modules impacted:**
- **New services/modules:**
- **Data model / DTO impact:**
- **API impact:**
- **DevOps / infrastructure impact:**

## Considerations

<Alternatives, concerns, open questions. For each: what it is, pros, cons, verdict.>

## Decision

<The change being made, in full sentences and active voice: "We will ...". State the chosen option and the main reason it won.>

## Consequences

**Positive**
- <What becomes easier>

**Negative**
- <What becomes harder, costs, risks, debt introduced>

**Neutral / follow-ups**
- <Migrations, follow-up decisions this now requires>

## Related ADRs

- [ADR-NNNN: Title](relative/link.md) - how it relates (supersedes, is superseded by, depends on, refines)

## References

- [Title](URL)
```

## 5. Section guidance

- **Title**: one concrete decision as a noun phrase ("Use PostgreSQL for order storage"). No vague titles like "Improve the data architecture".
- **Status**: exactly one of `Proposed`, `Accepted`, `Rejected`, `Deprecated`, `Superseded by ADR-NNNN`. Start at `Proposed`; use `Accepted` only if the user says it is already approved.
- **Date**: ISO 8601 (`YYYY-MM-DD`), the date of the last status change. Take today's date from the environment; do not guess.
- **Submitters / Deciders**: only people the user named or the repo clearly shows; otherwise omit the block.
- **Change Log**: one line per state change, newest last. States: Proposed, Accepted, Amended, Deprecated, Superseded. Link a PR/issue only if it exists.
- **Referenced Use Cases / Requirements**: link the motivating requirement, issue or use case. Say what is left out if the ADR does not cover all of it. Omit if none exists.
- **Context**: lead with the problem and the forces (technical, business, team, time, cost, compliance) and why a decision is needed *now*. Two to five short paragraphs or a short list.
- **Proposed Design**: only when the decision has structural impact. Architecture level only — no code walkthroughs. Delete inapplicable sub-bullets.
- **Considerations**: at least two real options, including "do nothing" when plausible. If no alternatives exist and none can be inferred, say so instead of inventing weak ones.
- **Decision**: unambiguous, present or future tense ("We will use X"), tied to the forces in Context. Readable on its own.
- **Consequences**: record **both** positive and negative outcomes. No negatives is a red flag — look for cost, lock-in, operational burden, migration work, skills gaps, follow-up decisions. Be concrete (what, who, how much).
- **Related ADRs**: link superseding, superseded, dependent and sibling ADRs with a one-phrase relevance note.
- **References**: external docs, benchmarks, RFCs, tickets. Real URLs only.

### Omit rather than pad

- Mandatory: Title, Status, Context, Decision, Consequences.
- Every other section is included only if it has real content.
- No empty headings, no `<...>` placeholders and no "N/A" filler in the final file.
- Small, well-understood decision → lightweight form: Title, Status, Date, Context, Decision, Consequences.

## 6. Lifecycle rules

- Accepted ADRs are never rewritten. When context changes, create a **new** ADR that supersedes the old one.
- Superseding: in the old ADR set Status `Superseded by ADR-NNNN` and add a Change Log line; in the new ADR add a Related ADRs entry. Edit the old file only for these two changes.
- Keep rejected decisions (Status `Rejected`) — the reasoning is valuable.
- History beyond the Change Log lives in version control.

## 7. Writing style

- Concise and specific; usually one to two pages.
- Prefer concrete facts and numbers from the repo or user over adjectives.
- Active voice for the Decision, neutral voice for Context.
- Explain the *reason*, not the implementation steps.
- Use the user's language for prose; keep technical terms and identifiers as they are.
- Markdown only as the template shows (headings, bullets, links).

## 8. Quality checklist

- [ ] Title names one concrete decision.
- [ ] Context explains why the decision is needed now.
- [ ] At least two options considered (or the absence of alternatives stated).
- [ ] Decision is unambiguous and justified by the Context.
- [ ] Consequences include positive and negative items.
- [ ] Status and Date are valid; the number follows the existing sequence.
- [ ] No fabricated names, links, metrics or dates; no leftover placeholders or empty sections.
- [ ] Related ADR links resolve to real files.
- [ ] File written to the intended path without overwriting anything.

## 9. Final message to the user

- The path of the created file and its Status.
- Any assumptions made or fields left as `TBD`.
- One suggested next step if relevant (e.g. open a PR, update the ADR index/README).

Do not paste the whole ADR back unless the user asks.

## References

- [EdgeX decision record template](references/decision-record-template-by-edgex.md) — full section list for the EdgeX additions.
- [Michael Nygard decision record template](references/decision-record-template-by-michael-nygard.md) — source of the mandatory core.
