---
name: improve-codebase-architecture
description: Scan a codebase for deepening opportunities, present them as a visual HTML report, then grill through whichever one you pick.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated from "improve-codebase-architecture" mattpocock skill and updated using own architecture standards.
---

# Improve Codebase Architecture

Surface architectural friction and propose **deepening opportunities**: refactors that turn shallow modules into deep ones. The aim is testability and AI-navigability.

This command is _informed_ by the project's requirements, standards, domain model and recorded decisions. It is built on a shared design vocabulary:

- Call the Skill tool with "architect-glossary" for the architecture vocabulary (**module**, **interface**, **depth**, **seam**, **adapter**, **leverage**, **locality**) and its principles (the deletion test, "the interface is the test surface", "one adapter = hypothetical seam, two = real"). Use these terms exactly in every suggestion, and don't drift into "component," "service," "API," or "boundary."
- The project's domain glossary gives names to good seams.
- Architecture Decision Records record decisions this command should not re-litigate.

## Sources

A repo may keep some or all of these. **Locate them, don't assume a path**: search the repo (root, `docs/`, `internal/`, `conventions/`, `architecture/`) for files with these names and roles. Read the ones that exist before proposing anything; each gives a different baseline for judging the code. If one is absent, skip it and say so in the report.

| Source | What it tells you |
|---|---|
| Product requirements (e.g. `PRD.md`, `REQUIREMENTS.md`, `docs/prd.md`) | Which features were required, prioritized and scoped, versus how the code actually implements them. Treat it as the spec baseline; a requirement the code skips, half-implements, or scatters across shallow modules is a candidate. |
| Coding standards (e.g. `CODING_STANDARDS.md`, `CONTRIBUTING.md`, `docs/conventions/`, `STYLEGUIDE.md`) | The language- and framework-level best practices the project expects, and how it structures its areas and directories. A deviation that adds friction is a candidate. |
| Architecture Decision Records (e.g. `adr/`, `docs/adr/`, `docs/decisions/`) | Decisions already made. Skip them unless the friction is real enough to reopen the ADR. |
| Domain glossary (e.g. `GLOSSARY.md`, `CONTEXT.md`, `docs/glossary.md`) | The domain terms that name seams. |

**Glossary is missing:** do not create one silently. When no glossary exists, ask the user first, and offer to derive it from the project's coding standards and conventions (its area sections and naming rules). If they agree, create it alongside those convention docs and use it for the rest of the run. If they decline, work without it and use the requirements document's vocabulary instead.

## Process

### 1. Explore

**Scope before you scan: YAGNI.** Deepening a module pays off by making future changes to it easier, so put extra weight on the parts of the codebase that have recently changed. Decide *where* to look before you look:

- If the user named a direction (a module, a subsystem, a pain point), take it, and skip the inference below.
- Otherwise, walk back a good stretch of the commit history (`git log --oneline`) to find the codebase's hot spots, the files and areas that keep coming up, and let those paths pull your attention first. If the changes are scattered with no clear hot spot, widen the net.

Read the sources above first, at least the parts that cover the area you're touching.

Then spawn a sub-agent with a fresh context window to walk the codebase. Don't follow rigid heuristics; explore organically and note where you experience friction. Compare what the code does against what the requirements document demands and what the coding standards expect:

- Where does understanding one concept require bouncing between many small modules?
- Where does a required feature live in the code as a tangle of small modules instead of one **deep module**?
- Where does the implementation diverge from the coding standards in a way that costs **locality** or testability?
- Where are modules **shallow**, with an interface nearly as complex as the implementation?
- Where have pure functions been extracted just for testability, but the real bugs hide in how they're called (no **locality**)?
- Where do tightly-coupled modules leak across their seams?
- Which parts of the codebase are untested, or hard to test through their current interface?

Apply the **deletion test** to anything you suspect is shallow: would deleting it concentrate complexity, or just move it? A "yes, concentrates" is the signal you want.

### 2. Present candidates as an HTML report

Write a self-contained HTML file to the OS temp directory so nothing lands in the repo. Resolve the temp dir from `$TMPDIR`, falling back to `/tmp` (or `%TEMP%` on Windows), and write to `<tmpdir>/architecture-review-<timestamp>.html` so each run gets a fresh file. Open it for the user (`xdg-open <path>` on Linux, `open <path>` on macOS, `start <path>` on Windows) and tell them the absolute path.

The report uses **Tailwind via CDN** for layout and styling, and **Mermaid via CDN** for diagrams where a graph/flow/sequence reliably communicates the structure. Mix Mermaid with hand-crafted CSS/SVG visuals: use Mermaid when relationships are graph-shaped (call graphs, dependencies, sequences), and hand-built divs/SVG when you want something more editorial (mass diagrams, cross-sections, collapse animations). Each candidate gets a **before/after visualisation**. Be visual.

For each candidate, render a card with:

- **Files**: which files/modules are involved
- **Requirement intent vs actual**: only when it matters, what the requirements document asks for and how the code currently implements it
- **Problem**: why the current architecture is causing friction
- **Solution**: plain English description of what would change
- **Benefits**: explained in terms of locality and leverage, and how tests would improve
- **Before / After diagram**: side-by-side, custom-drawn, illustrating the shallowness and the deepening
- **Recommendation strength**: one of `Strong`, `Worth exploring`, `Speculative`, rendered as a badge

End the report with a **Top recommendation** section: which candidate you'd tackle first and why.

**Use the project glossary's vocabulary for the domain, and the `/architect-glossary` vocabulary for the architecture.** If the glossary defines "Order," talk about "the Order intake module," not "the FooBarHandler," and not "the Order service."

**ADR conflicts**: if a candidate contradicts an existing ADR, only surface it when the friction is real enough to warrant revisiting the ADR. Mark it clearly in the card (e.g. a warning callout: _"contradicts ADR-0007, but worth reopening because…"_). Don't list every theoretical refactor an ADR forbids.

See [HTML-REPORT.md](HTML-REPORT.md) for the full HTML scaffold, diagram patterns, and styling guidance.

Do NOT propose interfaces yet. After the file is written, ask the user: "Which of these would you like to explore?"

### 3. Grilling loop

Once the user picks a candidate, call the Skill tool with "grilling" to walk the decision tree with them: constraints, dependencies, the shape of the deepened module, what sits behind the seam, what tests survive.

Side effects happen inline as decisions crystallize; call the Skill tool with "domain-modeling" to keep the domain model current as you go:

- **Naming a deepened module after a concept not in the glossary?** Add the term to the project's glossary. If no glossary exists yet, ask the user first (derive it from the coding standards as described in Sources).
- **Sharpening a fuzzy term during the conversation?** Update the glossary right there.
- **User rejects the candidate with a load-bearing reason?** Offer an ADR in the project's ADR directory, framed as: _"Want me to record this as an ADR so future architecture reviews don't re-suggest it?"_ Only offer when the reason would actually be needed by a future explorer to avoid re-suggesting the same thing; skip ephemeral reasons ("not worth it right now") and self-evident ones.
- **Want to explore alternative interfaces for the deepened module?** Call the Skill tool with "architect-glossary" and use its design-it-twice parallel sub-agent pattern.
