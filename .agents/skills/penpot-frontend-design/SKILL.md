---
name: penpot-frontend-design
description: "Aesthetic direction for Penpot design work: commit to a distinctive, non-generic visual point of view and express it with what Penpot can actually render. Companion to penpot-workflow, which owns the mechanics. Use when creating or restyling boards, screens, or components in Penpot and the result must look intentional instead of AI-generated."
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  notes: Adapted version for Penpot workflow based in luo-kai frontend-design skill
---

# Penpot Frontend Design

This skill supplies the *what* and *why* of a design. `penpot-workflow` supplies
the *how* (MCP tools, shape creation, styling parameters, export, review).

Load both together when the deliverable is a design, not code. If only one is
loaded, expect either a technically correct but generic result (mechanics only)
or unrealizable intentions (aesthetics only).

## How the two skills combine

| Concern | Owner |
|---|---|
| Aesthetic direction, typography, color, composition, anti-generic rules | this skill |
| Tool calls, shape/frame creation, fills, radii, shadows, components, export | `penpot-workflow` |

**Precedence, highest first:**

1. The design agent's hard rules (safety, scope, no fabricated results).
2. `DESIGN.md` (or equivalent brand/design-system doc) and the existing Penpot
   file: tokens, fonts, spacing, conventions.
3. This skill: how to be distinctive *within* that system.
4. `penpot-workflow` defaults (Tailwind spacing, Source Sans Pro, atomic naming).

`DESIGN.md` always wins over "be bold". If the brand mandates Inter or a
specific palette, use it. Distinctiveness comes from composition, hierarchy,
detail, and constraint — not from overriding the brand. Only when no design
system exists do you choose the direction from scratch, and then you must state
that choice in the report.

## Step 1 — Read before drawing

1. Locate and read `DESIGN.md` (and `ARCHITECTURE.md` for product context).
2. Inspect the Penpot file: existing pages, boards, components, tokens, fonts
   already in use.
3. Identify: purpose, audience, tone, constraints, and the one thing the screen
   must communicate first.

If none of this exists, say so in one line and proceed with an explicit,
written assumption. Never silently invent a brand.

## Step 2 — Commit to a direction

Choose one clear conceptual direction and execute it with precision. Pick an
extreme, not a midpoint:

- brutally minimal, maximalist editorial, industrial/utilitarian,
  warm analog, retro-futuristic, luxury/refined, playful/toy-like,
  brutalist/raw, art-deco/geometric, soft/pastel, organic/natural.

**Bold maximalism and refined minimalism both work. The failure mode is a
design with no point of view.** Write the direction in one sentence before you
create a single shape, and report it. If `DESIGN.md` exists, restate its
direction instead of inventing a new one.

## Step 3 — Express it in Penpot

### Typography
- Pair a characterful display face with a dependable body face; never one
  generic family for everything if the brand allows a pairing.
- Use scale, weight, tracking, and case contrast for hierarchy. Establish one
  deliberate type moment (oversized headline, tight mono label, generous
  display line-height) rather than styling every text layer equally.
- Use only fonts the file actually exposes through `penpot.fonts` (see
  `penpot-workflow`). The Plugin API cannot install font files; if the intended
  family is missing, fall back to an available family and report the
  substitution honestly.

### Color
- One dominant field, one supporting tone, one sharp accent. Avoid evenly
  distributed palettes and timid pastels.
- Depth comes from value, border, and layering before shadows. Reserve shadows
  for genuine elevation, not decoration.
- Follow the brand's emphasis ratio if defined (e.g. 70/20/10). Respect contrast
  targets (WCAG AA) even when the palette is bold.

### Composition
- Prefer asymmetry, overlap, offset alignment, diagonal flow, or one
  grid-breaking element over a perfectly centered stack.
- Give negative space a job: separate arguments, not inflate the canvas.
- Build inside frames/auto-layout. Break the grid *within* the layout system;
  do not switch to absolute positioning just to place a decorative element.

### Background and detail
- Create atmosphere with layered fills, gradients, image fills, low-opacity
  shapes, hairlines, and geometry — not with unrelated decoration.
- One memorable detail per screen is enough: a distinctive divider, an
  oversized index number, a repeating instrument-panel mark, a textured band.

### States and motion
- Design hover/focus/disabled/error/empty/loading as separate frames or
  variants; a single resting state is an incomplete design.
- Motion, scroll effects, and cursor behavior cannot be rendered in Penpot.
  Describe them as handoff annotations; never fake an interactive state with a
  static illusion that misrepresents behavior.

## Translation table: web intent → Penpot reality

| Idea from frontend design | In Penpot |
|---|---|
| CSS variables / design tokens | Penpot tokens, shared styles, or a `Design System` page |
| Custom web fonts | Fonts exposed by the file (`penpot.fonts`); otherwise fallback + report |
| Gradient mesh, noise, grain, texture | Layered gradient/image fills, low-opacity shapes, SVG/PNG fills |
| Backdrop filter / glassmorphism | Not reliably supported; use solid or semi-transparent fills |
| Motion / scroll-triggered reveal | Handoff annotation only |
| Custom cursor | Out of scope; note it if user asked |
| Hover/focus microinteraction | Separate variant frames with explicit state names |
| Responsive behavior per breakpoint | Frame per breakpoint (mobile/tablet/desktop), per the workflow's scale |

## Anti-slop rules

Avoid unless the brand explicitly calls for it:

- Purple (or blue) gradient on a white background.
- Default look of `Inter`/`Roboto`/`Arial`/system fonts when the brand allows a
  deliberate pairing.
- Centered hero: eyebrow + split headline + two buttons + three equal feature
  cards.
- Glassmorphism, glowing blobs, generic mesh gradients.
- Emoji as icons, fake testimonials, invented logos, lorem-grade placeholder
  claims.
- Arbitrary radii/shadows that vary between similar components.

Prefer:

- A named point of view you could describe to someone.
- Type, color, and spacing used as an argument, not as defaults.
- Aligned irregularity: asymmetry that still sits on a grid.
- One intentional detail that survives a screenshot.

## Self-review before reporting

Run this alongside `penpot-workflow`'s visual review and screenshot step:

- Can I name the direction in one sentence? Would the design change if I renamed
  it? If not, it has no point of view.
- Would this be mistaken for a template or another AI output? What is the one
  thing that proves it was designed for this product?
- Is the type pairing and palette intentional, or the library default?
- Is hierarchy readable at a glance without color?
- Are the states present, named, and consistent?
- Did I deviate from `DESIGN.md`? If so, was it authorized and reported?
