# 01 — The Playbook

The complete, repeatable procedure for reproducing a target page's frontend with
near-100% visual fidelity and identical assets.

Everything here is written so an **agent** can execute it: each phase has a command, an
artifact, and a pass condition. Human judgement is required only at the decision points
marked **DECIDE**.

---

## 0. The fidelity model

"Pixel perfect" is not one property. It decomposes into six layers, and each layer has
its own failure mode, its own instrument, and its own fix. Debugging a pixel diff without
this model is guesswork.

| # | Layer | What it covers | Instrument | Typical failure |
| --- | --- | --- | --- | --- |
| 1 | **Structure** | DOM tree, nesting, order, text content | `audit` node coverage (`missing`/`extra`) | Wrong wrapper, missing pseudo-element, different tag |
| 2 | **Layout** | Box geometry: rects, margins, padding, gaps, grid/flex, stacking | `audit` bbox deltas | 1–4px drift from a padding or gap |
| 3 | **Paint** | Colours, gradients, borders, radii, shadows, opacity, blend, filters | `audit` style deltas | Off-by-one hex, wrong shadow blur |
| 4 | **Type** | Family, size, weight, line-height, tracking, rendering | font mismatch list + same-engine screenshots | Font substituted, line-height rounding |
| 5 | **Media** | Image bytes, intrinsic size, crop, `srcset` selection, SVG, video | SHA-256 asset match | Re-encoded image, wrong `srcset` candidate |
| 6 | **State** | Hover/focus/active, scroll effects, sticky, animations, breakpoints | breakpoint + state screenshots | Missing hover, wrong sticky offset |

Two rules follow from this table:

- **A pixel diff tells you *how much* is wrong; the audit tells you *why*.** Use them
  together. A 4% diff with a clean audit means media or rendering, not CSS.
- **Fix layers in order.** Structure → layout → paint → type → media → state. Fixing
  colour on an element that is 8px off is wasted work; the next layout fix will move it.

### What "100%" realistically means

- **Byte-identical assets**: achievable and worth enforcing. SHA-256 match, always.
- **Identical DOM structure**: achievable for static pages; not always desirable (you
  may want cleaner markup — then accept that the audit will report `extra`/`missing`
  nodes and rely on the pixel diff instead).
- **Pixel-identical rendering**: only achievable with the same browser engine, same OS,
  same font files, same DPR, and no animations. Cross-platform, expect 0.05–0.3% noise
  from font rasterisation and subpixel antialiasing alone.
- **Identical behaviour**: re-implement, do not copy. Bundled JS is usually licensed,
  minified, and coupled to a backend.

---

## 1. Phase 0 — Preflight

**Goal:** remove every source of non-determinism and confirm you are allowed to do this.

### 1.1 Legality and scope (do not skip)

Run the checklist in [`05-LEGAL-AND-ETHICS.md`](05-LEGAL-AND-ETHICS.md). At minimum:

- [ ] You own the site, have written permission, or this is private study/analysis.
- [ ] You will not republish the target's brand assets, logos, or copy.
- [ ] You checked `robots.txt` and the ToS for scraping restrictions.
- [ ] You noted the font licences — a webfont you cannot licence is a hard blocker for
      shipping, and a substitution for fidelity.

Record the outcome in `out/<slug>/PROVENANCE.md` (template in
[`templates/CLONE-SPEC.md`](../templates/CLONE-SPEC.md)).

### 1.2 Pin the environment

Determinism is the difference between a 0.05% diff and a 2% diff that never converges.

| Knob | Value to pin | Why |
| --- | --- | --- |
| Browser build | Playwright's bundled Chromium | Chrome version changes layout subtly |
| Viewport | Primary `1440x900`, plus `768x1024`, `390x844` | The reference must be reproducible |
| DPR | `1` | `srcset` selection and raster scaling both depend on it |
| Locale / timezone | `en-US` / `UTC` | Date formatting, number separators, A/B geo |
| `prefers-reduced-motion` | `reduce` | Stops marquees/parallax from moving between shots |
| `prefers-color-scheme` | `light` | Otherwise you capture the dark variant |
| Scrollbars | hidden | 15px of layout shift otherwise |
| Font rendering | `--font-render-hinting=none --disable-lcd-text` | Reduces AA noise between runs |
| Colour profile | `--force-color-profile=srgb` | Otherwise wide-gamut shifts every colour |
| Animations | frozen via injected CSS | A moving pixel is a false hotspot |
| Network | `networkidle` + fonts + images settled | Half-loaded pages capture half-laid-out |

`capture.mjs` applies all of these and records them in `manifest.env`. If you change any
of them, you must re-capture the reference — never compare across environments.

**The same launch flags must be used for the candidate screenshot.** Measured cost of
getting this wrong: a pixel-perfect clone scored **0.44% mismatch** when screenshotted
without `--font-render-hinting=none --disable-lcd-text --force-color-profile=srgb`,
purely from LCD subpixel antialiasing on text. That is 9× the 0.05% target, from nothing
but a missing launch flag.

### 1.3 Detect variants before capturing

A page that renders differently on two loads will never converge. Check:

```powershell
node capture.mjs --url <url> --out out/_probe1 --no-assets
node capture.mjs --url <url> --out out/_probe2 --no-assets
# compare
node -e "const a=require('./out/_probe1/capture/dom/serialize.json'),b=require('./out/_probe2/capture/dom/serialize.json');console.log(a.html===b.html?'STABLE':'VARIES')"
```

- **VARIES** → look for A/B testing scripts, personalisation, rotating banners, carousels
  starting at a random slide, cookie-consent gating, or a geo/pricing variant. Either
  neutralise the variation (block the experiment script with `--block`) or mask the
  region in the diff.
- **STABLE** → proceed.

Also decide, explicitly, which *state* you are cloning: logged out, no cookie banner,
first slide of the carousel, default tab. Write it down. "The page" is ambiguous.

**DECIDE:** primary viewport, target state, and whether dynamic regions get masked.

---

## 2. Phase 1 — Capture

**Goal:** a complete, offline, deterministic snapshot of the page as the browser
actually rendered it.

```powershell
node capture.mjs --url <url> --out out/<slug> `
  --viewports desktop:1440x900,tablet:768x1024,mobile:390x844 `
  --scroll --states --deep
```

### What you get, and what each artifact is for

| Artifact | Use |
| --- | --- |
| `dom/index.html` | Post-hydration DOM. The baseline for the rebuild. |
| `dom/nodes.json` | Per-element computed styles + geometry. **The ground truth.** |
| `dom/dom-snapshot.json` | CDP layout tree (`--deep`). Use when `nodes.json` is not enough. |
| `css/sheet-NNN.css` | Every stylesheet, including cross-origin ones re-fetched. |
| `assets/` + `asset-map.json` | Every image/font/media file with SHA-256. |
| `fonts/` + `fonts.json` | `@font-face` faces and their load status. |
| `fonts/rendered-fonts.json` | **The font the engine actually rendered** (CDP `CSS.getPlatformFontsForNode`). Substitutions are listed here — fix them first. |
| `screenshots/` | The reference images. The objective function. |
| `breakpoints.json` | Media-query widths — your responsive test matrix. |
| `interactions.json` | Hover/focus selectors, keyframes, animated selectors. |
| `network.json` | Full request log — find the API/data sources. |

### Why computed styles, not the CSS source

The stylesheet is not what the browser painted. The cascade, inheritance, media queries,
CSS custom properties, container queries, `:has()`, and JS-applied inline styles all
transform it. `nodes.json` records the *resolved* value of ~110 visual properties for
every element, plus `::before`/`::after` content and styles.

Practical consequence: **when the diff disagrees with your CSS, believe `nodes.json`.**
Search it for the element's path and read the actual value.

### The CDP escape hatch

The page's own CSSOM cannot read cross-origin stylesheets — `document.styleSheets[i].cssRules`
throws `SecurityError`. The Chrome DevTools Protocol is out-of-process from the
same-origin policy, so it can:

| Need | CDP call | Why it beats the in-page approach |
| --- | --- | --- |
| Cross-origin sheet text | `CSS.getStyleSheetText` / `CSS.styleSheetAdded` events | No CORS, includes constructed sheets |
| The cascade, not just the winner | `CSS.getMatchedStylesForNode` | Tells you *which* rule won, and the inherited chain |
| Layout + geometry in one call | `DOMSnapshot.captureSnapshot` | Includes iframes, flattened shadow DOM, paint order, `currentSourceURL` (the `srcset` winner) |
| **Which font actually rendered** | `CSS.getPlatformFontsForNode` | Returns `familyName`, `postScriptName`, `isCustomFont`, `glyphCount` |
| Authoritative breakpoints | `CSS.getMediaQueries` | Every query the engine parsed, with an `active` flag |
| Deterministic hover/focus capture | `CSS.forcePseudoState` | Better than synthesising mouse events |

`capture.mjs` uses the in-page extraction (portable, readable) and re-fetches cross-origin
CSS over the network; `--deep` writes the CDP `DOMSnapshot`. When you need the cascade or
the rendered font, go to CDP directly — see
[`02-RESEARCH-NOTES.md`](02-RESEARCH-NOTES.md) §2 for the exact request shapes.

### Font truth: declared vs rendered

`nodes.json` records `font-family` — what the CSS *asked for*. What actually rendered may
be a fallback, and that is the single most damaging failure mode in a clone: every text
box changes width, every line wraps differently, every section height shifts.

The only reliable check is `CSS.getPlatformFontsForNode`, which reports the real platform
font, whether it was a custom download, and how many glyphs it drew. Treat it as a
**binary gate**: if the rendered family is not the requested family, stop and fix the font
before measuring anything else. A 0.4% pixel diff caused by a substituted font will never
converge, because the fix is not in your CSS.

### Gotchas that silently ruin captures

| Symptom | Cause | Fix |
| --- | --- | --- |
| CSS files are empty | Cross-origin stylesheet; CSSOM blocked | `capture.mjs` re-fetches `href` — verify `css/index.json` shows `crossOriginBlocked: true` **with** non-zero bytes |
| Missing fonts | Fonts served from a CDN, `@font-face` in a cross-origin sheet | Re-parse fetched CSS text for `@font-face`; check `fonts.json` `status` |
| Images are placeholders | Lazy loading never triggered | `--scroll`; or set `loading="eager"` and `data-src` → `src` before capture |
| Wrong image variant | `srcset` picked a candidate based on DPR | Pin `--dpr 1` and record `currentSrc` |
| Page is half-rendered | SPA hydration not finished | Wait for a stable `nodeCount` across 500ms, not just `networkidle` |
| Cookie banner shifts layout | Consent script | Dismiss it before capture, or capture with it and reproduce it |
| Canvas is blank | WebGL/canvas content | `collectAssets()` snapshots `toDataURL()`; treat as media |
| iframe content missing | Cross-document | Capture each frame separately and rebuild as an embed |
| Shadow DOM missing | Closed shadow roots | Only open roots are reachable; note it as a hard case |

### Capture quality gate

Before rebuilding anything, verify the capture is trustworthy:

- [ ] `screenshots/desktop.png` is not blank and has the expected dimensions.
- [ ] `css/index.json` total bytes > 0 and every sheet has a local file.
- [ ] `fonts.json` faces > 0 and `status` is `loaded` for the primary family.
- [ ] `assets` count is plausible; `warnings` contains no `4xx` for critical images.
- [ ] `nodes.json` node count is plausible (a marketing page is usually 800–3000).
- [ ] Two consecutive captures produce the same DOM (see 1.3).

---

## 3. Phase 2 — Analyze

**Goal:** turn raw capture data into a rebuild plan: sections, components, tokens,
breakpoints, and the interaction list.

```powershell
node analyze.mjs --capture out/<slug>/capture --out out/<slug>/analysis
```

Produces:

- `analysis/inventory.md` — the page split into vertical sections with their bounding
  boxes, backgrounds, and element counts. This is your build order.
- `analysis/tokens.json` — the design tokens actually used: colour palette by frequency,
  font scale, spacing scale, radii, shadows, container widths.
- `analysis/components.md` — repeated element signatures (same tag+class+structure
  appearing 3+ times) = your component list with occurrence counts.
- `analysis/plan.md` — a suggested section-by-section build order, largest visual area
  first.

### How to read the inventory

Build **top to bottom, largest area first**. Rationale: the diff is dominated by large
blocks, so fixing them moves the number fastest, and each section is independently
verifiable with a cropped diff.

For each section, note:

1. Its bounding box (`x, y, width, height`) — this becomes the crop rect for
   section-level diffs.
2. Its background (colour, gradient, image, video).
3. Its layout primitive: centred container, full-bleed, 2-col grid, 3-col card grid,
   flex row, absolute-positioned collage.
4. Its interactive elements: links, buttons, tabs, carousel, accordion.
5. Its responsive behaviour: read `breakpoints.json` and check which rules touch it.

### Design tokens: extract, do not guess

`tokens.json` gives you the real values. Use them to write your stylesheet once, in
token form, instead of hardcoding hex values section by section. This is what makes the
result maintainable and makes section 7 converge fast.

**DECIDE:** rebuild target — static HTML/CSS, or a framework. See §4.1.

---

## 4. Phase 3 — Rebuild

**Goal:** a local site that renders the same page, built from the capture.

### 4.1 Choose the rebuild strategy

| Strategy | When | Fidelity speed | Maintainability |
| --- | --- | --- | --- |
| **Baseline-first** (start from `rewrite.mjs` output, then refactor) | Almost always | Fastest — starts at 95–99% | Grows into clean code |
| **Blank-page rebuild** (write markup from scratch from the spec) | You want clean semantic markup from turn one, or the target markup is unusable | Slow — every section starts at 0% | Best |
| **Framework port** (Astro/Next + components) | Target is itself a framework site, or you need routing/SSR | Medium | Best for production |

**The single most important trick: never start from a blank page.**

`rewrite.mjs` produces a self-contained, offline copy of the captured DOM with local
assets. That is a ~95–99% pixel match *before you write a line of code*. It validates
your capture pipeline, gives you a working reference to diff against, and turns the
rebuild into a series of small, measurable refactors instead of a big-bang guess.

Then refactor **one section at a time**, replacing captured markup with your own
components and captured CSS with your own token-based CSS, running the diff after each
step. Any regression is immediately attributable to the last change.

> Licensing note: the baseline is the target's markup. Use it as a scaffold for
> private/authorised work. For anything you will publish, replace it with your own
> implementation — that is the whole point of the refactor loop.

### 4.2 The baseline step

```powershell
node rewrite.mjs --capture out/<slug>/capture --out out/<slug>/site --neutralize-links --strip-scripts
node serve.mjs --dir out/<slug>/site --port 4173      # leave running
node run.mjs --url <target> --out out/<slug> --stage verify
```

Expected: diff < 1%, audit score > 0.95, zero missing assets. If not, **stop and fix the
capture or the rewrite** — do not start refactoring on a broken baseline.

`--strip-scripts` is not optional for a baseline. The captured DOM is already hydrated;
re-running the page's own bundle can mutate it. We measured a live site whose script
duplicated its entire body in the offline copy, taking the baseline from 0.0000% to
2.19% mismatch and adding 11 phantom nodes. Static baseline, always.

Common baseline gaps and their causes:

| Gap | Cause | Fix |
| --- | --- | --- |
| Backgrounds missing | `url()` inside a cross-origin sheet wasn't rewritten | Check `rewrite-report.json` `unresolved` |
| Fonts wrong | Font files not downloaded, or `@font-face` lost | Check `fonts/` and the rewritten CSS |
| Layout broken | CSS `@import` chain not flattened | `--inline-css` |
| Page blank | JS-dependent rendering; the captured DOM needs the app's JS | Keep the inline scripts, or re-implement |
| Body duplicated / extra nodes | The page's own bundle re-runs and re-renders | `--strip-scripts` |
| Page taller than the reference | Same cause, or lazy images loading | `--strip-scripts`, then `--scroll` on capture |

### 4.3 The refactor loop (per section)

For each section in `analysis/plan.md`, in order:

1. **Crop the reference.** Take the section's bbox from the inventory and crop
   `screenshots/desktop-full.png` to it. That is the section's target image.
2. **Write the component.** Markup + scoped CSS using only tokens from `tokens.json`.
3. **Diff the crop.** Section-level diff, not whole-page — a 0.4% whole-page change is
   invisible, a 4% section change is actionable.
4. **Read the audit hotspots** for that section's paths and fix the top 5 by area.
5. **Commit** when the section diff is under budget (see §8).

Do not move to the next section while the current one is above budget. Errors compound:
a 3px padding error in the hero shifts everything below it.

### 4.4 What to copy, what to rewrite, what to drop

| Thing | Action |
| --- | --- |
| Markup structure | Reproduce the box structure; improve semantics if you like, but keep the boxes |
| CSS values | Copy the *resolved* values from `nodes.json`, expressed as tokens |
| Images/SVG | Copy bytes exactly; verify SHA-256 |
| Fonts | Copy only if licensed; otherwise substitute and accept the diff |
| Inline SVG | Copy as-is — it is part of the DOM |
| Icon fonts | Prefer converting to inline SVG for control |
| JS interactions | **Re-implement.** Do not ship the target's bundle |
| Analytics/trackers | **Drop.** They are noise and a liability |
| Third-party embeds | Re-embed, or replace with a static placeholder and mask it |

### 4.5 Interactions

`interactions.json` lists every `:hover`, `:focus`, `:active` selector, every `@keyframes`
name, and every element with a transition or animation. Work through them:

- Hover states: reproduce the property deltas (usually colour, transform, shadow).
- Focus states: reproduce the ring; keep it accessible.
- Scroll effects: sticky headers, reveal-on-scroll, parallax. Reproduce with
  `IntersectionObserver`; verify by screenshotting at scroll offsets.
- Carousels/tabs: reproduce the *initial* state exactly; behaviour second.
- Animations: copy the keyframes and timings; verify with animations disabled.

---

## 5. Phase 4 — Verify (the convergence loop)

**Goal:** drive the numbers down with the smallest possible changes.

```
screenshot candidate ──► diff (how much) ──► audit (why) ──► fix top 5 ──┐
        ▲                                                                │
        └────────────────────────────────────────────────────────────────┘
```

### 5.1 The two instruments

**`diff.mjs` — the objective function.** Pixel-level. Answers "how much is wrong and
where". Produces `diff.png`, a magenta `heatmap.png`, a per-region grid, mismatch %, and
SSIM.

Two numbers, two meanings — do not confuse them:

- **`threshold`** is a *per-pixel colour distance* (0–1). It decides whether a single
  pixel counts as different.
- **`mismatchPercent`** is *the share of the image that differs*. This is the number you
  optimise.

A "1% tolerance" written as `threshold: 0.01` is not a 1% tolerance — it is a very strict
per-pixel colour test. Always gate on `mismatchPercent` against `width × height`.

**`audit.mjs` — the explanation.** DOM-level. Answers "which element, which property,
by how much". Produces ranked hotspots with the exact CSS deltas, plus missing/extra
nodes, bbox deltas, asset hash mismatches, and font mismatches.

### 5.2 Reading a hotspot

An audit hotspot looks like:

```
html > body > main > section:nth-of-type(2) > div:nth-of-type(1)
  bbox: +12.5px height, -0.0px width
  props: padding-top 64px → 48px | font-family "Inter" → "Helvetica"
```

Translate it directly:

- `bbox` delta with matching `padding`/`margin`/`gap` deltas → layout layer, fix the box.
- `bbox` delta with no style delta → the cause is on an ancestor or a sibling; walk up.
- Style delta on `color`/`background`/`box-shadow` → paint layer, one-line fix.
- `font-family` delta → **stop everything and fix fonts first**; it changes every text
  box on the page and invalidates every other measurement.
- Asset `hashMismatch` → you re-encoded or resized an image. Restore the original bytes.

### 5.3 Prioritisation rule

Fix by **pixel area affected × magnitude**, not by hotspot count:

1. Fonts (global, invalidates everything else).
2. Large-section layout drift (> 4px on a region > 10% of the viewport).
3. Large-section paint (wrong background, missing image).
4. Small layout drift (1–4px).
5. Text rendering and AA noise (usually unfixable; mask or accept).

### 5.4 When diff and audit disagree

| Situation | Meaning | Action |
| --- | --- | --- |
| Diff high, audit clean | Media or rendering, not CSS | Check asset hashes, image crop/`object-fit`, font files, DPR |
| Diff low, audit noisy | Invisible differences (DOM order, unused props) | Ignore the audit; the pixels are right |
| Diff high, audit blames a tiny element | That element is huge in visual weight (a full-bleed overlay) | Fix it; area is not the only weight |
| Audit blames off-canvas elements | Carousel slides, off-screen menus, `translateX(-100%)` panels | Filter hotspots to the viewport rect before chasing them |
| Diff concentrated in one region, audit clean there | An animated or dynamic component (typewriter hero, marquee, video) | Mask the region, or pin the clock so both sides render the same frame |
| Diff won't move after 3 fixes | Structural: wrong box model or a missing wrapper | Re-read `nodes.json` for the subtree; compare `display` and `position` first |

### 5.5 Convergence expectations

| Stage | Desktop mismatch | What is left |
| --- | --- | --- |
| Baseline (`rewrite.mjs`) | 0.2–1.5% | URL rewriting gaps, font loading |
| Tokens + layout done | 0.1–0.5% | Paint details, spacing |
| Paint + type done | 0.02–0.15% | AA noise, subpixel |
| Finished static page | **< 0.05%** | Nothing meaningful |

If you are stuck above 1% after two full loops, the problem is almost always fonts or
images — not CSS.

---

## 6. Phase 5 — Harden

**Goal:** the page holds up outside the single reference screenshot.

### 6.1 Responsive

`breakpoints.json` gives the real widths. Screenshot and diff at each:

```powershell
node capture.mjs --url <target> --out out/<slug> --viewports tablet:768x1024,mobile:390x844
node run.mjs --url http://127.0.0.1:4173 --out out/<slug> --stage verify --viewports tablet:768x1024,mobile:390x844
```

Gate: mobile mismatch < 0.3%, no horizontal scrollbar, no overlapping text, tap targets
≥ 44px.

### 6.2 States

Screenshot each hover/focus state and diff against a captured state screenshot
(`--states`). Gate: every interactive element has a visible hover and focus state.

### 6.3 Accessibility

Cloning a page does not clone its accessibility. Check: alt text on every image,
semantic landmarks, heading order, contrast ratios, keyboard reachability, visible focus
rings, `lang` attribute, form labels.

### 6.4 Performance and hygiene

- Remove analytics, trackers, chat widgets, and A/B scripts. They are not part of the page.
- Keep the original asset bytes for verification; optimise a *copy* for production and
  re-run the diff to confirm the optimisation is visually lossless.
- Self-host fonts, preload the primary face, `font-display: swap` only if the target uses it.
- Check for leftover `http://` references to the origin — the page must render offline.

---

## 7. Phase 6 — Ship

### 7.1 Final gate

All of these must be true:

- [ ] Desktop mismatch **< 0.1%**, mobile **< 0.3%**.
- [ ] Audit score **> 0.97**.
- [ ] **Zero** missing assets; every asset SHA-256 matches the capture.
- [ ] **Zero** font-family mismatches.
- [ ] All breakpoints pass.
- [ ] No network requests to the target origin.
- [ ] Accessibility pass.
- [ ] Provenance recorded, licences checked.

### 7.2 Provenance

Keep `PROVENANCE.md` with: target URL, capture timestamp, browser build, what was copied
verbatim (assets, markup), what was rewritten, font licences, and the authorisation basis.
This is your defence if anyone asks where the code came from.

---

## 8. Convergence rules (the numbers)

Put these in the agent's instructions verbatim. They are the difference between a loop
that terminates and one that spins.

| Metric | Target | Hard stop |
| --- | --- | --- |
| Desktop pixel mismatch | < 0.05% | < 0.1% |
| Mobile pixel mismatch | < 0.2% | < 0.3% |
| SSIM | > 0.999 | > 0.995 |
| Audit score | > 0.98 | > 0.95 |
| Missing assets | 0 | 0 |
| Font mismatches | 0 | 0 |
| Iterations without improvement | — | **3 → stop and re-capture** |

**The 3-strike rule:** if three consecutive fix attempts do not reduce the mismatch,
the model of the problem is wrong. Re-capture, re-read `nodes.json`, or scope the region
out. Do not keep guessing.

**Diminishing returns:** below 0.05% you are chasing antialiasing. Stop.

---

## 9. Anti-patterns

| Anti-pattern | Why it fails |
| --- | --- |
| Starting from a blank page | You throw away the only ground truth you have |
| Copying the minified CSS bundle | Unmaintainable, and it will not match because JS mutates styles |
| Copying the JS bundle | Licensing, coupling to a backend, unreadable, and it will not run |
| Eyeballing instead of diffing | The eye misses 2px and 1% colour shifts |
| Comparing screenshots from different browsers/OS | 1–3% noise floor, unfixable |
| Fixing colour before layout | The element moves; you redo the colour |
| Ignoring fonts | Every text box is wrong; nothing else can converge |
| Re-encoding images "to optimise" | Breaks the byte-identical requirement |
| Capturing with animations running | Every capture has different hotspots |
| One giant fix per iteration | You cannot attribute the regression |
| Chasing 0.01% | Antialiasing is not a bug |

---

## 10. Scaling to many pages

Once one page converges, the marginal cost of the next drops sharply:

1. **Shared shell first.** Capture every page; the header/footer/nav is identical. Build
   it once, verify once.
2. **Component extraction.** `analyze.mjs` component signatures across pages reveal the
   real design system. Build components, not pages.
3. **Per-page diff matrix.** One row per page, one column per viewport, mismatch % in
   each cell. The matrix is the project status.
4. **Regression gate.** Any change re-runs the whole matrix; nothing ships red.
5. **Capture once, rebuild forever.** Captures are immutable evidence; keep them.

---

## 11. Command reference

```powershell
# environment
node doctor.mjs

# capture
node capture.mjs --url <url> --out out/<slug> --viewports desktop:1440x900,mobile:390x844 --scroll --states --deep

# analyze
node analyze.mjs --capture out/<slug>/capture --out out/<slug>/analysis

# rebuild baseline
node rewrite.mjs --capture out/<slug>/capture --out out/<slug>/site --neutralize-links

# serve
node serve.mjs --dir out/<slug>/site --port 4173

# screenshot the candidate (via the pipeline)
node run.mjs --url http://127.0.0.1:4173 --out out/<slug> --stage verify

# measure
node diff.mjs  --ref out/<slug>/capture/screenshots/desktop.png --cand out/<slug>/shots/desktop.png --out out/<slug>/diff --label desktop
node audit.mjs --capture out/<slug>/capture --candidate http://127.0.0.1:4173 --out out/<slug>/audit

# full pipeline
node run.mjs --url <url> --out out/<slug>
```

See [`06-TROUBLESHOOTING.md`](06-TROUBLESHOOTING.md) when a number will not move.
