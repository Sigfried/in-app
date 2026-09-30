# Help/Tour packages — plan

**How to read this for review.** Boxes marked **DECIDE** need a call from
Siggie; boxes marked **PROPOSED** are Claude's recommendation, waiting for a
yes/no. Everything else is either decided (§1) or a fact about the current code.

---

## 1. Goal and what is decided

Pull the help/tour system out of dmvd into **npm packages**, and use them first
in **vs-hub** (`../personal/vs-hub`, the TermHub revival). The work happens in a
new monorepo, `personal/in-app` (§4), into which dmvd and vs-hub are absorbed
temporarily and split back out when stable.

Decided (Siggie, 2026-09-28):

- **Help and tour are separate packages.** They do different jobs: a tour walks
  the viewer through steps in order and changes app state as it goes; context
  help explains whatever the viewer points at.
- **The prose dialect is its own package**, usable for anything that renders
  authored text — the legend today, not just help and tour.
- **A fourth package holds anchoring and the popover**, which help and tour
  both need (§2).
- **vs-hub needs tours only.** So the first release is three packages:
  markdown, anchor/popover, tour. The help package comes later (§6).
- **Published to npm under `@sigfried`**, named `@sigfried/in-app-*` (§4).
  While the apps sit in the monorepo they use the packages as workspace
  dependencies; they consume the published versions once split back out.
- **The live sites freeze** while their apps are in the monorepo. vs-hub
  (GitHub Actions) and dmvd (`npm run deploy`) are not redeployed until each is
  split back out to its own repo.

---

## 2. The four packages

Short names in this doc: **markdown**, **anchor**, **tour**, **help**.

| package | npm name | what it does | depends on |
|---|---|---|---|
| **markdown** | `@sigfried/in-app-markdown` | Renders authored markdown with the extensions below, on any string, in any component. | react-markdown, remark-directive |
| **anchor** | `@sigfried/in-app-anchor` | Finds the element a piece of content points at, and draws a popover beside it: placement, mount points, dragging, highlight/spotlight. | markdown |
| **tour** | `@sigfried/in-app-tour` | Tours: parsing steps and beats, the prev/next chrome, the tour map, driving app state through host callbacks. | markdown, anchor |
| **help** | `@sigfried/in-app-help` | Help mode: hint dots, click-an-element-to-see-its-entry. | markdown, anchor |

### What the markdown dialect is

These are the extensions dmvd's prose already uses. Each is specified in
[`src/help/FORMAT.md`](../apps/dmvd/src/help/FORMAT.md):

- `{{kind:arg}}` placeholders, filled by **host-supplied text resolvers** (dmvd
  uses them for live counts and names from the model).
- Inline widgets — an image whose URL is `widget:name:arg` — drawn by **host-supplied widget
  renderers**.
- `:s[text]{color=… size=…}` and `:::s{…} … :::` for styling a span or a block
  ([`styleDirectives.ts`](../apps/dmvd/src/help/styleDirectives.ts)).
- `{{target:replace}}` on a link: open in place instead of a new tab
  ([`linkTarget.ts`](../apps/dmvd/src/help/linkTarget.ts)).
- Alerts: `>` blockquotes, optionally dismiss-once.

Today's entry point is `<HelpMarkdown>`
([`HelpMarkdown.tsx`](../apps/dmvd/src/help/HelpMarkdown.tsx)); dmvd's legend already
renders through it.

> **DECIDE — who owns the content-file format?** *Provisionally answered with
> the PROPOSED option below (Claude, 2026-09-30, unattended); see §3.1.
> Siggie to confirm.* Today one file
> ([`help-content.md`](../apps/dmvd/src/explore/help-content.md)) holds `## sections` of
> `### entries`, each with `- **Field:** value` lines, and **one entry can be a
> help topic and a tour step at once**. With separate packages, something has
> to own that section/entry/field structure.
>
> **PROPOSED:** the markdown package owns the *document* structure (sections,
> entries, fields, `Description:` blocks) as a generic parser; tour and help
> each declare the fields they read (`Change:`, `Anchor:`, beats, …). The
> alternative —
> each package parses its own file — is simpler per package but means an
> element that is both explained and toured is written twice.

---

## 3. The extraction is a disentangling, not a move

The in-app code was written on the premise that help and tour are **one
registry with two navigation modes** (header of
[`HelpProvider.tsx`](../apps/dmvd/src/help/HelpProvider.tsx)). Splitting them reverses
that, so most large files have to be cut apart:

| current file | lines | goes to |
|---|---|---|
| [`parseHelpContent.ts`](../apps/dmvd/src/help/parseHelpContent.ts) | 1447 | split: document structure + placeholders → markdown; `parseAnchor`, highlight/placement fields → anchor; tours, beats, positions → tour |
| [`HelpLayer.tsx`](../apps/dmvd/src/help/HelpLayer.tsx) | 1389 | split: popover + placement → anchor; prev/next chrome, address readout → tour; hint dots → help |
| [`HelpProvider.tsx`](../apps/dmvd/src/help/HelpProvider.tsx) | 694 | split: anchor resolution → anchor; tour state + the host callbacks (`onPushChange`, `onPopChange`, `onJumpChanges`, `onTourStart`, `onTourEnd`) → tour; help-mode toggle, `?` key → help |
| [`helpContext.ts`](../apps/dmvd/src/help/helpContext.ts) | 182 | split per package; each gets its own context/hook |
| [`help.css`](../apps/dmvd/src/help/help.css) | 1061 | split along the same lines |
| [`FORMAT.md`](../apps/dmvd/src/help/FORMAT.md) | 1339 | split into one spec per package, **fully generic**: no dmvd or LinkML vocabulary, neutral examples. Expect it much shorter — 41 mentions are dmvd's, and whole tables list dmvd's resolver kinds, widgets, colors, anchor kinds and URL params. Those tables move to a dmvd authoring reference. What a vs-hub one contains: decide after seeing the generic spec. |
| [`HelpMarkdown.tsx`](../apps/dmvd/src/help/HelpMarkdown.tsx), [`markdownParts.tsx`](../apps/dmvd/src/help/markdownParts.tsx), [`styleDirectives.ts`](../apps/dmvd/src/help/styleDirectives.ts), [`linkTarget.ts`](../apps/dmvd/src/help/linkTarget.ts) | ~410 | markdown, whole |
| [`mountPoints.ts`](../apps/dmvd/src/help/mountPoints.ts), [`useDragged.ts`](../apps/dmvd/src/help/useDragged.ts) | ~200 | anchor, whole; `useDragged` is **exported** (dmvd's legend frame uses it) |
| [`TourMap.tsx`](../apps/dmvd/src/help/TourMap.tsx) | 291 | tour, whole |

What already holds and makes this tractable: nothing under `src/help/` imports
from dmvd, and dmvd's content, anchor tags, resolvers and style overrides all
live in `src/explore/`.

**Order of work.** Because dmvd is absorbed into the monorepo, the split
happens there with dmvd's app and tests to check against:

1. Create the monorepo; absorb dmvd and vs-hub (§4). *Done 2026-09-29.*
2. Carve `apps/dmvd/src/help/` into `packages/in-app-{markdown,anchor,tour}`;
   dmvd imports them by package name. dmvd's tests and build stay green at each
   step. Stages and design: §3.1.
3. vs-hub adopts `in-app-tour`, and its first tour gets written.
4. `in-app-help` later (§6).

### 3.1 Design of the split

Worked out by Claude on 2026-09-30, while Siggie had said to run unattended,
after reading all of `src/help/`, its dmvd call sites and its tests.
**Everything here is a proposal Siggie has not reviewed**, including the
answer to §2's DECIDE box. Each stage is its own commit, so any of it can be
revisited without unpicking the rest.

#### What goes where

**markdown** (`packages/in-app-markdown`)

- Whole files: `styleDirectives.ts`, `linkTarget.ts`, `markdownParts.tsx`
  (the component table, `widgetImg`, `urlTransform`, `remarkPluginsFor`).
- From `parseHelpContent.ts`: `TextResolver`, `fillPlaceholders`,
  `placeholdersIn`.
- `WidgetRenderer` (now in `helpContext.ts`) and `ColorMap`.
- `<HelpMarkdown>` becomes **`<Prose>`**. The package is no longer "help", and
  `Markdown` would clash with react-markdown's default import in callers.
- **The generic content-document parser** (§2's DECIDE, answered with its
  PROPOSED option): `parseDocument(markdown, { skipSections })` returns
  sections with their body lines, and entries with `id`, file `order` and raw
  `lines`. The field readers move with it: `fieldOf` (bold-optional,
  case-insensitive, `~~parked~~`), `readField`, `readBlockField`,
  `readBulletList`, `isMarginField`, and an `unknownFields` check. dmvd's
  `TODO` section name becomes a host option rather than a constant.

**anchor** (`packages/in-app-anchor`)

- Whole files: `mountPoints.ts`, `useDragged.ts` (still exported; dmvd's
  legend frame uses it).
- From `parseHelpContent.ts`: the anchor type (renamed `Anchor`),
  `parseAnchor`, `parseSpotlight`, and the placement types and parsers
  (`Highlight`, `PopoverSide`, `Offset`, `parseWidth`, `parseHighlight`,
  `parsePosition`, `parseOffset`).
- `resolveAnchor` becomes a plain function. It is a `querySelectorAll` on
  `data-help-id`, and never needed to be on a context.
- From `HelpLayer.tsx`: an **`<AnchoredPopover>`** component, driven by props.
  It owns anchor tagging with its `MutationObserver`, the scroll-into-view,
  spotlight tagging and rings, the `data-help-scrim` attribute, mounting in a
  registered mount point, the drag handle, the wait-for-anchor delay, and the
  placement style (`popoverPosition`, `offsetStyle`, `autoWidth`). It takes
  `centerOn` as a prop and measures the region itself.
- **`Once:` handling** (`stripAlerts`, the dismiss-once storage, the checkbox
  alert) goes here. Any popover body can carry a once-alert, and the width
  has to be computed from the text after stripping.

**tour** (`packages/in-app-tour`)

- The entry, beat, meta and position types; `DEFAULT_TOUR`, `tourNames`,
  `tourSlug`, `tourBySlug`, `tourSteps`, `tourPositions`, `positionOfStep`,
  `stepAddressOf`; `extractBeats`.
- `parseTourContent(markdown, options)` composes the other packages:
  `parseDocument`, then per entry the markdown field readers, anchor's
  parsers, and its own tour fields.
- **The `Beats:` cut.** Before any reader sees an entry's lines, tour splits
  them at the first `Beats:` header, parked or not, into the entry's own
  field lines and the beats block. Today `extractField` stops at `beats` by
  name. Cutting first means no reader in markdown or anchor has to know that
  beats exist, and the output is identical: beat fields are indented, and a
  block field already ends at the next margin field.
- `<TourProvider>`: the tour state (running tour, position index, the
  overview), the host callbacks, the `?`/Escape/arrow keys during a tour, the
  authoring-aid toggle, and parse-then-fill of the content.
- `<TourLayer>`: renders `<AnchoredPopover>` for the current position, plus the
  tour chrome (tour label, action band, counter, reveal dots, nav buttons,
  address tag) and `navMinWidth`.
- `TourMap.tsx`, whole.
- A small `<EntryExtras>` component for the `Interactions:`, `Shortcut:` and
  `Context:` fields, exported so dmvd's help popover can reuse it.

**Staying in dmvd's `src/help/` until the help package (§6)**

- Help mode, hint dots, and the **single-entry popover the Help menu opens**
  (`showEntry`). That popover is live code: with help mode off, the menu is
  the only way into a help-only entry. It is context help, so it belongs to
  the future help package, not to tour.
- It reads the parsed `content` from the tour context rather than parsing the
  file a second time. That dependency is temporary and dmvd-internal.
- **One behaviour change:** picking a help entry while a tour runs ends the
  tour. Today the two share one `activeId`, which gives a popover with one
  entry's title over another position's body. After the split they would be
  two popovers.

#### Contexts

- **`<ProseProvider textResolvers widgets colors>`**, from markdown, holds the
  enrichments. It also owns the registered-resolvers state and exposes
  `setTextResolvers`, which dmvd needs because its resolvers exist only once
  the model has loaded. **`useProse()` returns empty options** outside a
  provider rather than throwing: without enrichments, markdown still renders.
- **`<TourProvider>` sits inside `<ProseProvider>`** and reads the effective
  resolvers from it to fill the content. So `textResolvers`, `widgets` and
  `colors` leave the tour provider's props.
- anchor has no context: `<AnchoredPopover>` is all props.
- dmvd's tree becomes `ProseProvider > TourProvider > HelpProvider (dmvd) >
  app, TourLayer, HelpLayer (dmvd)`.

#### CSS

[`help.css`](../apps/dmvd/src/help/help.css) splits into one stylesheet per
package, each imported by that package's components the way `HelpLayer`
imports `help.css` today:

| file | rules |
|---|---|
| markdown | `.help-prose` output styles (headings, paragraphs, lists, code, links), the alert (`.help-popover-alert*`), `.help-inline-widget`, `.help-styled` |
| anchor | the `anchor-name` rules, spotlight rings and scrim, the popover shell and its `@position-try` fallbacks, the title and drag handle, body scroll, popover links, the `Once:` checkbox |
| tour | the inline beat title and tour label, action band, nav row, reveal dots, address tag, the map, `.help-popover-interactions`/`-shortcut`/`-context`, `.help-beat-past` |
| dmvd `src/help/` | hint dots, `body.help-mode` |
| dmvd `explore/` | `.help-inline-relation`, which styles a dmvd resolver's output |

- The popover body gets the `help-prose` class, so the doubled
  `.help-popover-body …, .help-prose …` selectors collapse to one.
- **Order matters** where rules of equal specificity override each other,
  for example `.help-popover-title-inline` over `.help-popover-title`, and
  `.help-popover-alert p` over `.help-prose p`. Import order gives markdown,
  then anchor, then tour, because each package's entry imports its
  dependencies first. dmvd's `helpTheme.css` must still load last.
- The parked dark-mode block splits along the same lines.
- Class names keep their `help-` prefix for now (see Deferred).

#### Workspace mechanics

- Each package's `package.json` `exports` points at its **TypeScript source**
  while in the workspace. Vite and vitest compile linked workspace packages
  from source, and dmvd's `tsc -b` typechecks them through the imports.
  `publishConfig` swaps in a built `dist` (JS plus `.d.ts`) at publish time,
  which is what the vs-hub CHECK in §5 needs once it consumes a published
  version.
- `react`/`react-dom` are `peerDependencies`, and also `devDependencies` so a
  package typechecks on its own. The lockfile has one React, 19.3.0, so pnpm
  gives every importer the same copy.
- Each package gets a `typecheck` script with its own `tsconfig.json`, at
  least as strict as dmvd's (`noUnusedLocals`, `verbatimModuleSyntax`,
  `erasableSyntaxOnly`), since dmvd's typecheck compiles the package source
  under dmvd's options anyway.
- **One `pnpm install`** after all three package skeletons and dmvd's
  `workspace:*` dependencies exist, rather than one per stage.
- **dmvd's tests stay in dmvd**, with imports updated. They are the evidence
  that each stage changed nothing. Moving the pure-package tests (style
  directives, link targets, placeholders, the parser) into the packages is
  deferred.

#### Stages

Each stage is committed with dmvd's `typecheck`, `build` and `vitest run`
green, run through nx.

1. **markdown.** Includes `<ProseProvider>`, dmvd's switch to it, and dmvd's
   `parseHelpContent` using the package's document parser and the `Beats:`
   cut.
2. **anchor.** Includes `<AnchoredPopover>` with dmvd's `HelpLayer` rendering
   through it.
3. **tour.** `TourProvider`, `TourLayer` and `TourMap` move; what is left in
   dmvd's `src/help/` is the help-mode remainder described above.
4. **FORMAT.md** split into one generic spec per package plus a dmvd authoring
   reference (the §3 table's row).

#### Deferred

- Renaming `help-*` classes and `data-help-*` attributes to package-neutral
  names. Wanted before publishing; it touches dmvd's theme, its e2e specs and
  its tests.
- A real build (`dist`, `.d.ts`, the CSS copied) and publishing.
- Moving tests into the packages.


---

## 4. The monorepo

Decided (Siggie, 2026-09-28): repo `personal/in-app`, pnpm + Nx, npm scope
`@sigfried`, package names `@sigfried/in-app-*`, apps dmvd and vs-hub absorbed
for now. How the repo is laid out, the rules that keep the apps extractable,
the hard limits and the known traps are in [CLAUDE.md](../CLAUDE.md).

## 5. Seams the extraction must keep

These are the places where the package deliberately knows nothing about the
host app. A change that "simplifies" one of them usually does so by teaching
the package something about dmvd.

| seam | package | contract |
|---|---|---|
| **anchor kinds** | anchor | Content says `Anchor: kind:arg` (e.g. `entity-row:Specimen`). The package matches that string against a `data-help-id` the host wrote on the element, and never interprets it. The host decides what each kind means by choosing which elements to tag. So a checkbox inside a row gets its **own** tag rather than being found as "the input inside the row" — the latter would put knowledge of rows into the package. |
| **text resolvers, widgets, colors** | markdown | `{{kind:arg}}`, `widget:name:arg` and color names are all looked up in tables the host passes in. |
| **`onPushChange` / `onPopChange` / `onJumpChanges` / `onTourStart` / `onTourEnd`** | tour | The tour hands the host each position's `Change:` query to push, asks it to pop one when going back, hands it a whole run for a jump, and says when a tour starts and ends. It counts frames and never interprets a query, so it never learns what a selection is. dmvd composes them against the URL. |
| **`centerOn`** | anchor | Where a popover with no anchor is centred. dmvd passes nothing (viewport-centred). |
| **`--help-font-size`** | anchor | Everything is sized in `em` off this one CSS custom property; a host overrides it. dmvd's override is [`helpTheme.css`](../apps/dmvd/src/explore/helpTheme.css). |

> **CHECK for vs-hub** (Claude, before starting): vs-hub is plain JSX, not
> TypeScript, so the packages must ship compiled JS plus `.d.ts`. It already
> uses React 19 and react-markdown 10, which matches. Whether its app state is
> in the URL — which decides how much work the push/pop callbacks are there — is not yet
> checked.
>
> Possible later (Siggie): convert vs-hub to TypeScript, and move its app from
> `frontend/` to the repo root — though a backend could come back someday,
> which is the case for keeping `frontend/`.

---

## 6. Help mode — later, in the help package

Help mode is switched off in dmvd (`HELP_MODE_ENABLED = false` in
[`helpContext.ts`](../apps/dmvd/src/help/helpContext.ts)), which hides its toggle and
disables the `?` key. The code still works. It has the defects below, so
**turning the flag back on is not enough**; they get fixed when the help
package is built, in this order:

1. **Exit affordance.** Make the toggle read `✕ exit help` and let its click
   through. Drop exit-on-window-blur; make one Escape exit, not two.
2. **`(i)` info buttons instead of `?` hints** (Siggie's call), leaving `?` for
   the keyboard shortcut only.
3. **Tag rows and controls and write their entries** — mostly content work.
4. **Decide what help mode does about clicks it has to block** (defect 2).

### The defects

1. **No visible way out.** Escape, `?`, clicking an untagged element and
   window blur all exit, and none is discoverable. Closing the popover closes
   the popover, not the mode, so the viewer's next click is silently eaten.
   Almost nothing is untagged to click on. And the toggle itself was disabled,
   because it sits inside a tagged element.
2. **It blocks what its own text describes.** An entry says "tick a checkbox
   to put an entity on the diagram," and help mode swallows that click.
3. **Clicks resolve far too coarsely.** A click shows the entry of the nearest
   tagged ancestor, and no row, checkbox or disclosure arrow is tagged — so
   every row shows the whole panel's entry.
4. **Coverage is accidental.** The tags were placed for the tour. Nothing
   explains attribute rows, edge types, or how to read the diagram.
5. **Hint dots ignore the viewport.** A dot is drawn for any element that
   exists, including ones scrolled out of view, so dots pile up at the edge.
6. **`?` means three things** — hint glyph, shortcut, button label. Fixed by
   step 2.

`isInputFocused` (4 lines, [`HelpProvider.tsx`](../apps/dmvd/src/help/HelpProvider.tsx))
exists only for the `?` shortcut, so it goes with help, inlined.

---

## 7. Settled constraints

- **Current browsers only** (Siggie, 2026-09-08). Placement is CSS anchor
  positioning, with no `@supports` guard and no measured fallback. Adding one
  would need new evidence of a support problem.
- **No native `title` on anything that opens a hover panel.** The browser's
  tooltip draws on top of the panel. Use `aria-label` and put the words in the
  panel. This one keeps getting reintroduced.
- **No `title`-swapping to show which elements have help.** Hint dots (or info
  buttons) do that job.

## 8. Worth a look, not planned

- **Two platform features** could replace hand-written popover code:
  `popover="hint"` (a popover that doesn't close other popovers) and interest
  invokers (`interestfor`), which would replace most of the
  hover/leave/pinned logic.
- **Backdrop:** plain `::backdrop` dim, or a spotlight cutout around the
  current element? The cutout needs an SVG mask or four rectangles recomputed
  on scroll/resize.
