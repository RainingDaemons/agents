---
name: penpot-workflow
description: "Manage design work in Penpot — create projects, organize pages, build components, export assets, review designs visually, and handle design-to-code handoff. Use when asked to create designs, manage Penpot projects, build components, export tokens, set up a new design project, review a design, or show what something looks like."
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  notes: Tool-agnostic. Uses only the Penpot MCP server (execute_code + export_shape). Product behavior summarized from the Penpot help center.
---

# Penpot Workflow

Workflows for managing design work in Penpot through its MCP server. The server
exposes three capabilities:

- **`execute_code`** — runs JavaScript against the Penpot Plugin API, with
  `penpot` and `penpotUtils` available.
- **`export_shape`** — renders a shape or page to PNG/SVG for visual review.
- **`high_level_overview` / `penpot_api_info`** — API documentation.

Tool names may carry a server prefix (for example `penpot_execute_code`). Use
the names your server actually exposes.

> **Companion skill.** Load **`penpot-frontend-design`** alongside this skill
> whenever the result must look intentionally designed rather than generic.
> That skill owns the *what* and *why* (direction, typography, color,
> composition, anti-slop rules); this one owns the *how*. Precedence, highest
> first: agent hard rules → `DESIGN.md` and the existing Penpot file →
> `penpot-frontend-design` → this skill's defaults.

## Start here — reference map

`references/` holds a snapshot of the Penpot help center. These documents
describe the product UI; the Plugin API mirrors much of it but not all of it.
Read the relevant file **before** writing code for an unfamiliar area, and
verify method names with `penpot_api_info` instead of guessing.

| Task | Read |
|---|---|
| Files, pages, layers panel, guides, comments, version history | `references/designing-workspace-basics.md` |
| Boards, shapes, paths, images, masks, booleans, constraints, layer styling | `references/designing-layers.md` |
| Fills, gradients, color palette, strokes, stroke per side, caps | `references/designing-color-stroke.md` |
| Text, typography options, RTL, custom fonts | `references/designing-text-typo.md` |
| Flex Layout and Grid Layout | `references/designing-flexible-layouts.md` |
| Assets, local library, organizing styles | `references/design-systems-assets.md` |
| Shared libraries, connecting and publishing, tokens source | `references/design-systems-libraries.md` |
| Main components, copies, overrides, annotations, swap | `references/design-systems-components.md` |
| Variants, properties, connected layers, combining | `references/design-systems-variants.md` |
| Design tokens, sets, themes, aliases, math, import/export | `references/design-systems-design-tokens.md` |
| Prototype triggers, actions, animations, flows | `references/prototyping-testing-prototyping.md` |
| View mode, sharing links, permissions, inspecting | `references/prototyping-testing-view-mode.md` |
| Inspect mode, CSS copy, code, export presets | `references/dev-tools.md` |

## Project Setup

When starting a new design project:

1. **Create the Penpot project** with a clear, matching name (same as the code repo)
2. **Set up standard pages:**
   - `Design System` — shared components, tokens, and styles
   - `Wireframes` — low-fidelity layouts and flows
   - One page per major screen or user flow (e.g., `Dashboard`, `Settings`, `Onboarding`)
3. **Define foundations first:**
   - Color palette (primary, secondary, neutral, semantic: success/warning/error/info)
   - Typography scale (headings h1-h6, body, caption, label, mono)
   - Spacing scale (aligned with Tailwind: 4px base unit)
   - Border radius tokens
   - Shadow/elevation tokens

Pages are tabs inside one file; a page named `---` with no content renders as a
separator, which is useful for grouping screens. Only the **active page** can be
created or edited through the API — call `await penpot.openPage(page)` first.

## Document model

A file contains pages; a page contains layers. Only **first-level boards** are
shown by default in View mode (nested boards are not), so treat first-level
boards as screens. Layers in the panel are listed bottom-to-top, and z-order
follows child order.

Layer kinds the workflow uses: board, rectangle, ellipse, text, path (bezier or
freehand curve), image, group, mask, and boolean operation. See
`references/designing-layers.md` for selection, grouping, masking, boolean
operators, resizing constraints, and per-layer styling.

Boards carry behavior worth setting deliberately:

- **Clip content** on/off — content outside the board can be hidden or shown.
- **Show in View mode** — decide which boards are screens.
- **Show fill in exports** — export the content without the board fill.
- **Resize to fit content**, board guides, "Set as thumbnail", "Copy link".

## Creating and Modifying Shapes

Write JavaScript and run it with `execute_code`. There is no separate per-shape
tool: one call can create, style, and arrange many shapes.

```js
const card = penpot.createBoard();
card.name = "Card";
card.x = 0;
card.y = 0;
card.resize(320, 200);
card.fills = [{ fillColor: "#14181C", fillOpacity: 1 }];
card.borderRadius = 12;
```

Rules that prevent most mistakes:

- `width` and `height` are **read-only**. Use `shape.resize(width, height)`.
- `x` and `y` are absolute page coordinates. For parent-relative positioning use
  `penpotUtils.setParentXY(shape, px, py)` or `shape.x = parent.x + px`.
- `fills`, `strokes`, and `shadows` arrays are **replace-only**: their contents
  are read-only, so assign a new array to change styling.
- Colors are hex strings in uppercase, e.g. `"#FFB454"`.
- Add children with `parent.appendChild(shape)`; z-order follows child order
  (first added = furthest back).
- Shapes can only be created or modified on the **active page**.

### Styling properties

Border radius, uniform or per corner:

```js
shape.borderRadius = 12;
shape.borderRadiusTopLeft = 8;
```

Multiple fills and gradients are allowed per layer, and so are multiple strokes
(order matters, the first stroke wins for tokenized widths). Shadow, stroke,
blur, opacity, and blend follow the same replace-array or value pattern:

```js
shape.shadows = [{
  style: "drop-shadow",
  color: { color: "#000000", opacity: 0.3 },
  offsetX: 0, offsetY: 4, blur: 12, spread: 0,
}];

shape.strokes = [{
  strokeColor: "#3A424B",
  strokeOpacity: 1,
  strokeWidth: 1,
  strokeAlignment: "inner",   // inner | outer | center
  strokeStyle: "solid",       // solid | dotted | dashed | mixed
}];

shape.blur = { value: 4 };            // layer blur
shape.backgroundBlur = { value: 4 };  // background (frosted) blur

shape.opacity = 0.9;
shape.blendMode = "multiply";
```

Other writable properties include `rotation`, `hidden`, `visible`, `blocked`,
`constraintsHorizontal`, `constraintsVertical`, `flipX`, and `flipY`. Check
`penpot_api_info` instead of guessing.

UI-only details from the docs: background blur, stroke-per-side, and stroke-to-path
need Penpot's WebGL renderer; stroke caps (arrows, round/square ends) apply to
open paths and are configured from the stroke section. If the API does not expose
them, note it instead of approximating with extra shapes.

### Text

```js
const title = penpot.createText("Signal Path");
title.fontFamily = "Space Grotesk";
title.fontSize = "40";
title.fontWeight = "600";
title.letterSpacing = "-0.02em";
title.fills = [{ fillColor: "#F2EFE9", fillOpacity: 1 }];
```

- `fontSize`, `fontWeight`, `lineHeight`, and `letterSpacing` are **strings**.
- `characters` holds the text content; changing it keeps the shape and styling.
- Style part of a text with `text.getRange(start, end)` (end exclusive).
- `resize()` sets `growType` to `"fixed"`. Set it back to `"auto-width"` or
  `"auto-height"` if you want the text to size itself.
- Text options in the product include case, horizontal and vertical alignment,
  decoration, and direction. RTL text is detected automatically; set direction
  explicitly when the design depends on it.

### Fonts

- Penpot ships the Google Fonts catalog; teams can also upload custom fonts
  (TTF, OTF, WOFF, WOFF2). Those uploads are a team-level dashboard action, not
  an API or agent action.
- Discover what the file can actually use through `penpot.fonts`; each font
  exposes its available `variants` (weights, styles).
- Prefer `penpot.fonts.findByName("IBM Plex Mono").applyToText(text, variant)`
  so family, weight, and style are applied together.
- The Plugin API enumerates and applies fonts; it does not install font files.
  If the intended family is missing, fall back to an available family and
  **report the substitution**. Never name a font in the report that the design
  does not actually use.
- Read `DESIGN.md` and the existing file for intended families and weights
  before choosing one. Do not introduce a new family on a whim.

### Layout systems

Prefer board layout systems over manually positioning children. They are the
only reliable way to make a design reflow, and they map directly to CSS.

```js
const row = penpot.createBoard();
row.resize(1200, 64);
const flex = row.addFlexLayout();
flex.dir = "row";              // row | row-reverse | column | column-reverse
flex.alignItems = "center";
flex.justifyContent = "space-between";
flex.columnGap = 24;
```

- Flex properties mirror Flexbox: direction, wrap, align items, justify content,
  gaps, padding, and sizing (`fix` / `fit`).
- Check `board.flex` / `board.grid` before moving children; the layout system
  controls their position.
- When a container already has children, use
  `penpotUtils.addFlexLayout(container, "row" | "column")` so their visual order
  is preserved.
- Control children with `child.layoutChild` (margins, `verticalSizing`,
  `horizontalSizing`: `"fix" | "auto" | "fill"`), not absolute coordinates.
- Grid: `board.addGridLayout()`, then `board.grid.appendChild(shape, row, col)`.
  Grid units are `fr` (fraction), `auto`, and pixels; cells can be merged into
  named areas.
- Set `child.layoutChild.absolute = true` only for intentional overlap.
- See `references/designing-flexible-layouts.md` for the full property lists and
  the terms used in inspect output.

## Components

A component is a layer or group of layers reused across the file. The source is
the **main component**; each use is a **copy** (instance). Copies inherit
changes, and overrides let one copy diverge while staying linked. Deleting a
main removes the component from the library too; a copy can restore it.

```js
const component = penpot.library.local.createComponent([buttonBoard]);
component.name = "buttons/primary/default";
```

- Instantiate with `component.instance()`.
- Detach a copy with `shape.detach()` before editing it independently.
- Name with slash-separated groups (`buttons/primary/hover`); slashes create the
  library folder structure.
- Annotations on a main component are visible on every copy and in Inspect —
  use them for developer notes.

Follow atomic design methodology:

- **Atoms** — buttons (primary, secondary, ghost, destructive), inputs, labels,
  badges, icons (Lucide as base set), avatar, tooltip, separator.
- **Molecules** — form fields, search bars, cards, menu items.
- **Organisms** — navigation, data tables, modals, forms.

Naming: `category/component-name/variant` — `forms/text-input/filled`,
`navigation/sidebar/collapsed`.

## Variants

Variants group related components behind named **properties** (`Color`, `Size`,
`State`) with **values** (`Primary`, `Small`, `Hover`); each variant is one
combination. Users switch variants by changing properties, and overrides survive
the switch.

- Create via `penpotUtils.createVariantContainer([...])`, or from the UI with
  "Create variant" on a component.
- Overrides survive only between **connected** layers: same name, same type
  (rectangle, ellipse, path, boolean count as one type), same hierarchy level.
  Renaming or regrouping breaks the connection.
- Boolean-style properties must use a recognized pair to render as a toggle:
  `true/false`, `on/off`, `yes/no`.
- Variants must keep at least one property.
- Bulk: "Combine as variants" merges same-page components into one set and
  derives property names from component names.

See `references/design-systems-variants.md` for the full rules.

## Design Tokens

Tokens are the shared vocabulary between design and code, stored per file (or
per shared library) following the **W3C DTCG** format. Types: color, dimension,
spacing, sizing, border radius, stroke width, opacity, rotation, number,
typography (font family, size, weight, letter spacing, text case, decoration),
and composite typography and shadow tokens.

- **Aliases:** `{token.name}` inside a value. References are case sensitive.
- **Math:** numeric tokens accept `+ - * / % ^` and functions such as `round()`,
  `max()`, `min()`, `roundTo(x, decimals)`. Use spaces around operators.
- **Sets:** tokens live in sets; when two sets define the same name, the set
  **later in the list wins**, like CSS cascade. Sets can be disabled.
- **Themes:** named contexts (mode, brand, contrast, platform) built from sets.
  Several themes can be active at once; only one per theme group.
- **Groups:** dots in a name create nested groups automatically
  (`button.primary.default.background-color`).
- **Library tokens:** a file can use one tokens source at a time — its own or a
  connected shared library's. Edit them in the library, not the consuming file.
- **Import/export** is DTCG JSON, single file or multifile, optionally zipped.

Inside `execute_code`: `penpot.library.local.tokens` lists them, apply with
`shape.applyToken`, and summarize with `penpotUtils.tokenOverview()`. Tokens can
bind width/height/x/y, radius, gaps, padding, stroke width, shadows, blur, and
typography — not just colors. Applying a second token of the same type replaces
the first.

## Prototyping

Interactions connect a hotspot (any layer) to a destination board. Triggers: on
click, mouse enter, mouse leave, after delay (boards only). Actions: navigate to,
open overlay, toggle overlay, close overlay, previous screen, open URL.
Animations: dissolve, slide, push. Overlays support "close when clicking
outside", a 20% black backdrop, and "relative to" positioning for hover menus.
Flows define starting points and are shareable independently in View mode.
"Fix when scrolling" pins headers and floating buttons for the presentation
view.

Treat prototyping as a **handoff artifact**: document the intended trigger,
action, animation, and flow per screen. The Plugin API's interaction support is
limited — verify with `penpot_api_info` before promising code-created
interactions; if unsupported, record the spec in annotations instead of
inventing it.

## Visual Review

When you need to **review your own work** or the user asks you to **review a
design**, export the rendered result with the MCP export tool. Do not use
browser automation, share links, or temporary screenshot files.

1. **Identify the target**: a board/shape id, `"page"`, or `"selection"`.
2. **Export it** with `export_shape` (`format`: `"png"` or `"svg"`,
   `mode`: `"shape"` or `"fill"`).
3. **Inspect the returned image directly.**
4. If the export fails, report the tool, target, and error; never claim you
   inspected a design you did not see.

Review for layout, direction, hierarchy, consistency, completeness, accessibility,
and memorability. Use `references/prototyping-testing-view-mode.md` when the
deliverable is a presentable prototype rather than a static board.

## Inspect and Handoff

Inspect mode is a view-only read of the design: layer tree on one side, Info and
Code on the other. It exposes distances to neighbors and board edges, property
sections, per-side stroke widths as CSS logical properties, and export presets.

When preparing designs for developer handoff:

1. **Export design tokens** as JSON:
   ```json
   {
     "colors": { "primary-500": "#7c6ff0" },
     "spacing": { "1": "4px", "2": "8px" },
     "typography": { "heading-1": { "size": "36px", "weight": 700, "lineHeight": 1.2 } },
     "radii": { "sm": "4px", "md": "8px", "lg": "12px" },
     "shadows": { "sm": "0 1px 2px rgba(0,0,0,0.05)" }
   }
   ```
   Penpot tokens can be inspected with `penpotUtils.tokenOverview()`.
2. **Map components to shadcn-svelte** — document which Penpot component maps to
   which shadcn component and what customization is needed.
3. **Annotate specs** — spacing, interaction behavior, edge cases (empty, error,
   loading), accessibility notes (ARIA, focus order), and behavior Penpot cannot
   render (motion, scroll-triggered reveals, backdrop blur, custom cursors).
4. **Export assets** — `export_shape` with `format: "svg"` for icons and
   illustrations, PNG for raster previews.
5. For code references inside `execute_code`: `penpot.generateStyle(shapes,
   { type: "css" })` and `penpot.generateMarkup(shapes, { type: "html" | "svg" })`.

## Runbook — from an empty file to a reviewed design

1. Read `DESIGN.md` and the existing file: pages, boards, components, tokens,
   fonts. Load `penpot-frontend-design` and state the aesthetic direction.
2. Create pages and foundations (palette, type scale, spacing, radii, shadows).
   Persist them as tokens/styles so screens reuse them.
3. Build screens as first-level boards with flex or grid layouts; keep spacing
   on the scale.
4. Extract repeated elements into components, group them, and add variants for
   states and sizes.
5. Wire flows: starting boards, triggers, actions, animations; annotate what the
   prototype cannot express.
6. Export a preview with `export_shape` and review it critically.
7. Hand off tokens, component mapping, annotations, and assets.

## Correctness and performance tips

- Batch related creation and styling into as few `execute_code` calls as sensible.
- After changes that affect layout, `await penpot.waitForLayoutUpdate()` before
  reading positions, sizes, or exporting.
- Search with `penpotUtils.findShape` / `findShapes` / `findShapeById`; do not
  reimplement traversal.
- `storage` persists between calls: keep shape IDs and reusable helpers there.
- Never add text that merely repeats a layer's name; Penpot already shows names.
- Z-order is child order: add backgrounds first, or use `bringToFront`,
  `sendToBack`, `setParentIndex`.
- Do not fight a layout system with absolute coordinates; if you need an
  exception, mark that child absolute and say why.

### What the Plugin API supports

| Feature | Support | Notes |
|---|---|---|
| Position & size | ✅ | `x`, `y` writable; `resize()` for size |
| Fills | ✅ | Solid, gradient, image; multiple per layer |
| Border radius | ✅ | Uniform and per corner |
| Shadows | ✅ | Multiple; drop or inner |
| Blur | ✅ | `blur` and `backgroundBlur` |
| Strokes | ✅ | Alignment, style, per-side widths on boards/rectangles |
| Stroke caps / stroke to path | ⚠ | UI features; verify API exposure first |
| Text | ✅ | Characters, ranges, font variants, alignment, spacing, direction |
| Opacity / rotation / blend modes | ✅ | Writable shape properties |
| Flex / grid layout | ✅ | Preferred over absolute positioning |
| Components | ✅ | `penpot.library.local.createComponent`, `instance()`, `detach()` |
| Variants | ✅ | Via `penpotUtils` and component properties |
| Tokens | ✅ | `penpot.library.local.tokens`, `applyToken` |
| Pages | ✅ | `openPage` to activate before editing |
| Prototype interactions | ⚠ | UI-first; confirm API support before relying on it |
| Export / inspect code | ✅ | `export_shape`, `generateStyle`, `generateMarkup` |

## Checklist Before Handoff

- [ ] Aesthetic direction stated and applied; no default-template look (see `penpot-frontend-design`)
- [ ] All components use the design system (no one-off styles)
- [ ] States documented (default, hover, active, focus, disabled, error, loading)
- [ ] Responsive layouts at all breakpoints
- [ ] Color contrast passes WCAG AA (4.5:1 text, 3:1 large text/UI)
- [ ] Focus indicators visible on all interactive elements
- [ ] Empty states, error states, and loading states designed
- [ ] Design tokens exported as JSON, with sets and themes named
- [ ] Component-to-shadcn mapping documented
- [ ] Spacing and typography use the defined scale (no magic numbers)
- [ ] Flow starting points, triggers, actions, and animations documented
- [ ] Non-renderable behavior (motion, scroll, blur, cursors) documented as annotations
- [ ] Requested fonts are either available and in use, or the substitution was reported
- [ ] Final design exported and reviewed through `export_shape`
