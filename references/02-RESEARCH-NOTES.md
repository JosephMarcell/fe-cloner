# 02 — Research Notes: Pixel-Perfect Frontend Cloning

Grounding notes for the playbook. Every non-obvious claim is linked. Research date: **2026-11-11**. All sources listed in §9 were actually fetched during this research pass.

---

## 1. TL;DR — the 10 decisions that matter most

| # | Decision | Why it dominates the outcome |
|---|---|---|
| 1 | **Capture from the live DOM via CDP, not by parsing HTML/CSS text.** Use `DOMSnapshot.captureSnapshot` for geometry + a computed-style whitelist, and `CSS.getMatchedStylesForNode` for the cascade. | Static HTML parsing cannot see the post-JS DOM, the cascade result, or the layout tree. The browser already computed the answer — read it, don't recompute it. ([DOMSnapshot.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/DOMSnapshot.pdl)) |
| 2 | **Get the real rendered font before writing a single line of CSS.** `CSS.getPlatformFontsForNode` returns the actual platform font, PostScript name, whether it was a custom (webfont) download, and a glyph count per font. | Font mismatch is the #1 pixel-diff cause and it silently cascades into every text box, line break, and element height. ([CSS.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl)) |
| 3 | **Copy the font *files*, not just the `font-family` name** — but only where licensed. | A `font-family: "Inter"` declaration on a machine without Inter renders in the fallback. See [05-LEGAL-AND-ETHICS.md](./05-LEGAL-AND-ETHICS.md) §3 before you copy any `.woff2`. |
| 4 | **Pin the render environment before the first screenshot.** Same browser build, same viewport, same DPR, same locale/timezone, same OS font set. | Playwright's own docs warn that rendering varies by OS, browser version, hardware, power source, and headless mode, and that snapshots must be generated in the same environment they are compared in. ([Visual comparisons](https://playwright.dev/docs/test-snapshots)) |
| 5 | **Freeze time and animation.** `page.clock.setFixedTime` / `install` + a determinism stylesheet. | `Date.now()`, `requestAnimationFrame`, `performance`, and `Event.timeStamp` all leak into layout and content. Playwright's clock API overrides exactly these. ([Clock](https://playwright.dev/docs/clock)) |
| 6 | **Rebuild section-by-section against per-section diffs, not whole-page at the end.** | A whole-page diff at 1440×8000 gives you a percentage, not an action. Per-section diffs localise the fix. |
| 7 | **Extract design tokens (CSS custom properties, spacing scale, type scale, palette) before writing components.** | Tokens are the site's actual design system. Rebuilding with hardcoded values guarantees drift on the second section. |
| 8 | **Treat every dynamic region as a named mask, declared in a manifest.** Cookie banners, carousels, ads, timestamps, A/B slots. | Unmasked dynamic regions produce diffs you can never close, which destroys the signal-to-noise ratio of the whole loop. |
| 9 | **Use `odiff` (or `pixelmatch` with `windowSize`) rather than a naive per-pixel counter.** | GPU dithering and sub-pixel AA scatter stray pixels; a real regression is a compact blob. Windowed counts and ignore-regions separate the two. ([pixelmatch](https://github.com/mapbox/pixelmatch), [odiff](https://github.com/dmtrKovalenko/odiff)) |
| 10 | **Write a capture manifest with provenance, and gate publication on it.** | Determinism, re-runnability, and legal defensibility are the same artifact. See 05 §5. |

---

## 2. The fidelity stack: capture → rebuild → verify

### Layer 0 — Determinism harness (before any capture)

Nothing downstream is trustworthy if the harness is not pinned.

```js
const context = await browser.newContext({
  viewport: { width: 1440, height: 900 },
  deviceScaleFactor: 1,
  locale: 'en-US',
  timezoneId: 'UTC',
  colorScheme: 'light',
  reducedMotion: 'reduce',
});
```

Playwright exposes `viewport`, `locale`, `timezoneId`, `isMobile`, `colorScheme`, and `permissions` as context/test options, and ships a device registry for UA/screen-size presets. ([Emulation](https://playwright.dev/docs/emulation)) Note that `timezoneId`/`locale` affect the *browser*, not the test runner — set `TZ` for the runner separately.

For time: prefer `page.clock.setFixedTime(new Date(...))`; fall back to `install()` + `pauseAt()` when timers must progress. `install()` must be called before any other clock call. ([Clock](https://playwright.dev/docs/clock))

Determinism stylesheet, injected before the capture screenshot:

```css
*, *::before, *::after {
  animation-duration: 0s !important;
  animation-delay: 0s !important;
  transition-duration: 0s !important;
  transition-delay: 0s !important;
  caret-color: transparent !important;
}
html { scroll-behavior: auto !important; }
```

For `toHaveScreenshot` the equivalent built-in is `stylePath`, which applies a custom stylesheet at screenshot time specifically to filter volatile elements. ([Visual comparisons](https://playwright.dev/docs/test-snapshots))

### Layer 1 — Capture

#### 1a. DOM + layout + style, in one shot

`DOMSnapshot.captureSnapshot` returns a flattened tree covering the root document **including iframes, template contents, and imported documents**, with shadow DOM flattened, plus:

- `NodeTreeSnapshot` — `parentIndex`, `nodeType`, `nodeName`, `attributes`, `backendNodeId`, `pseudoType`, `pseudoIdentifier`, `currentSourceURL` (**the selected URL for a `srcset`**), `originURL`, `isClickable`, `shadowRootType`.
- `LayoutTreeSnapshot` — `nodeIndex`, `styles` (indices into the whitelisted computed-style strings), `bounds` (absolute, **document coordinates, scroll offset ignored**), `text`, `stackingContexts`, `paintOrders`, and — with `includeDOMRects: true` — `offsetRects`, `scrollRects`, `clientRects`.
- `TextBoxSnapshot` — per-text-run `bounds` + `start` / `length`, i.e. where each run of characters actually landed.
- `DocumentSnapshot` — `contentWidth`, `contentHeight`, `scrollOffsetX/Y`.

Request shape:

```js
await cdp.send('DOMSnapshot.captureSnapshot', {
  computedStyles: [
    'display','position','top','left','right','bottom','z-index',
    'width','height','min-width','max-width','min-height','max-height',
    'margin-top','margin-right','margin-bottom','margin-left',
    'padding-top','padding-right','padding-bottom','padding-left',
    'box-sizing','flex-direction','flex-wrap','justify-content','align-items',
    'gap','row-gap','column-gap','grid-template-columns','grid-template-rows',
    'font-family','font-size','font-weight','font-style','line-height',
    'letter-spacing','word-spacing','text-align','text-transform',
    'color','background-color','background-image','background-size',
    'background-position','background-repeat','border-radius',
    'border-top-width','border-top-style','border-top-color',
    'box-shadow','opacity','overflow','transform','filter','backdrop-filter',
  ],
  includePaintOrder: true,
  includeDOMRects: true,
  includeBlendedBackgroundColors: true,
  includeTextColorOpacities: true,
});
```

`includeBlendedBackgroundColors` and `includeTextColorOpacities` are the two most under-used flags: they tell you the *effective* background behind text and the *effective* text opacity after stacking, which is exactly what you need when a design uses overlapping translucent layers. ([DOMSnapshot.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/DOMSnapshot.pdl))

Caveat from the protocol itself: the post-layout inline text positions are documented as **not stable across versions** — do not build a hard assertion on exact text box coordinates across Chrome upgrades.

#### 1b. Style: get the cascade, not just the computed result

`getComputedStyle` gives you the winning value but not *why*. `CSS.getMatchedStylesForNode` gives you `inlineStyle`, `attributesStyle`, `matchedCSSRules` (with matching selector indices), `pseudoElements`, `inherited` (the full ancestor chain), `inheritedPseudoElements`, `cssKeyframesRules`, `cssPropertyRules`, `cssPropertyRegistrations`, `cssAtRules`, `cssFunctionRules`, plus `parentLayoutNodeId`. ([CSS.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl))

That inheritance chain is what lets you rebuild *correctly* rather than by copying a flat property bag: you learn that `color` came from `body`, not from the element.

`CSS.getComputedStyleForNode(nodeId)` returns the full computed style as `{name, value}` pairs — useful as the ground-truth oracle for the verify loop.

#### 1c. The cross-origin stylesheet problem, and the CDP escape hatch

In-page, `document.styleSheets[i].cssRules` returns a live `CSSRuleList`, but for a stylesheet loaded from another origin the accessor throws a `SecurityError` per the CSSOM spec. ([MDN: cssRules](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet/cssRules), [CSSOM spec](https://drafts.csswg.org/cssom/#dom-cssstylesheet-cssrules))

Workarounds, best first:

1. **`CSS.getMatchedStylesForNode` / `CSS.getStyleSheetText` over CDP.** The CSS domain is out-of-process from the page's same-origin policy, so it reads cross-origin sheets that the page itself cannot. This is the correct answer and it also gives you the full text of every sheet, including `constructed` sheets created via `new CSSStyleSheet()` (flagged `isConstructed` in `CSSStyleSheetHeader`).
2. **`fetch()` the CSS URL** from Node (not from the page) — no CORS on the server side. Loses the cascade but gives you the text.
3. **Regex the raw HTML** — worst option, misses `@import`, `adoptedStyleSheets`, and anything injected by JS.

Use `CSS.styleSheetAdded` / `CSS.styleSheetRemoved` events to enumerate every sheet including ones injected after load.

#### 1d. Font truth: declared vs rendered

Two independent sources, and you need both:

```js
// declared faces + load status
const declared = await page.evaluate(() =>
  [...document.fonts].map(f => ({
    family: f.family, weight: f.weight, style: f.style,
    status: f.status,          // 'loaded' | 'unloaded' | 'loading' | 'error'
    unicodeRange: f.unicodeRange,
  }))
);
```

`document.fonts` is the document's `FontFaceSet`; `document.fonts.ready` resolves when loading and layout of all **used** fonts is done, and the set of used fonts can differ from the set of declared fonts (e.g. `font-display: optional` faces that never loaded). ([MDN: Document.fonts](https://developer.mozilla.org/en-US/docs/Web/API/Document/fonts))

```js
// what actually rendered
const fonts = await cdp.send('CSS.getPlatformFontsForNode', { nodeId });
// [{ familyName, postScriptName, isCustomFont, glyphCount }, ...]
```

`isCustomFont: true` means the glyph came from a downloaded webfont; `false` means it fell back to a platform font. A node whose `font-family` says "Inter" but whose `PlatformFontUsage` says `isCustomFont: false, familyName: "Arial"` is a fallback you must fix. `glyphCount` tells you which font won for which characters — essential for mixed-script and icon-font cases.

Also available: `CSS.fontsUpdated` fires with a `FontFace` payload (`fontFamily`, `fontStyle`, `fontVariant`, `fontWeight`, `fontStretch`, `fontDisplay`, `unicodeRange`, `src`, `platformFontFamily`, `fontVariationAxes`) — the `src` and `fontVariationAxes` fields are how you harvest the actual file URL and the variable-font axis ranges in one call.

#### 1e. Asset harvesting

| Asset class | How to enumerate | Gotcha |
|---|---|---|
| `<img>` | `img.currentSrc` (resolved `srcset` winner) + `img.src` | `DOMNode.currentSourceURL` gives the same thing out-of-band; never guess the srcset winner yourself |
| `<picture>` / `<source>` | `picture > source[srcset]` + `currentSrc` on the img | `media` / `type` conditions mean the chosen source is viewport- and format-dependent |
| CSS `background-image` | `getComputedStyle(el).backgroundImage` — and **repeat with `'::before'` / `'::after'`** | Pseudo-element backgrounds are invisible to a naive `querySelectorAll('*')` scan |
| `@font-face` | `document.fonts` + `CSS.fontsUpdated` + `FontFace.src` | Subsets: one family may be many files with different `unicodeRange` |
| Inline `<svg>` | `el.outerHTML` | `currentColor` and CSS vars inside the SVG change with context |
| SVG sprite | `<use href="#id">` → resolve the `<symbol>` in the sprite document | Cross-origin sprites may be unreadable |
| Icon fonts | `CSS.getPlatformFontsForNode` → `isCustomFont` + `glyphCount` | Icons are glyphs, not images; you must copy the font file (license!) |
| `<video>` / poster | `video.src`, `video.poster`, `<source>` children | Streaming (HLS/DASH) has no single file to copy |
| Favicon / manifest | `link[rel~="icon"]`, `link[rel="manifest"]` | Multiple sizes; the manifest points at more icons |
| Canvas / WebGL | `canvas.toDataURL()` (2D) — **fails for WebGL without `preserveDrawingBuffer`** | See §5 |
| Iframes | `DOMSnapshot` includes them; `page.frames()` for separate capture | Third-party frames are usually non-reproducible |
| Shadow DOM | `DOMSnapshot` flattens it; `el.shadowRoot` for open roots only | Closed shadow roots are only reachable via CDP |

A cheap completeness check: `performance.getEntriesByType('resource').map(e => e.name)` gives every URL the page actually fetched, which you diff against your harvested set to find misses.

#### 1f. Responsive: enumerate breakpoints from the engine

Do not regex the CSS for `@media`. `CSS.getMediaQueries` returns **all media queries parsed by the rendering engine**, each with `text`, `source` (`mediaRule` / `importRule` / `linkedSheet` / `inlineSheet`), `mediaList` expressions (`feature`, `value`, `unit`, `computedLength`) and an `active` flag for the current viewport. That is the authoritative breakpoint list, and `active` tells you which one you are currently in.

Container queries are separate: `CSSRule.containerQueries` (with `conditionText`, `name`, `physicalAxes`, `logicalAxes`) and the `CSSContainerQuery` type. `CSS.getLayersForNode` gives the `@layer` tree and its order, which matters because layer order changes the cascade without changing specificity.

Capture matrix: at minimum 3 viewports × 1 DPR. Recommended: `375×812 @3`, `768×1024 @2`, `1440×900 @1`, `1920×1080 @1`. Capture at DPR 1 for diffing (stable) and DPR 2 only for asset-variant discovery.

#### 1g. Interactive states

`CSS.forcePseudoState(nodeId, forcedPseudoClasses)` forces `:hover`, `:focus`, `:active`, `:focus-visible`, `:focus-within`, `:target` etc. on a node for style computation — this is how DevTools implements the `:hov` toggle, and it is far more reliable than synthesising mouse events. ([CSS.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl))

Playwright's own guidance is the complement: screenshots capture whatever hover state is live, so park the pointer at `(-1, -1)` before a baseline shot. ([Visual comparisons](https://playwright.dev/docs/test-snapshots))

Capture per interactive component: default, hover, focus, focus-visible, active, disabled, and any `[aria-expanded="true"]` variant.

### Layer 2 — Rebuild

**Token-first.** Before components, extract:

- **CSS custom properties.** Enumerate `CSSPropertyRegistration` via `CSS.getMatchedStylesForNode` (`propertyName`, `initialValue`, `inherits`, `syntax`) for `@property`-registered tokens, and read `:root` computed custom properties for the unregistered ones.
- **Type scale** — the distinct `(font-family, font-size, font-weight, line-height, letter-spacing)` tuples across the layout tree, sorted by frequency. The top 8 are the scale.
- **Spacing scale** — the distinct `margin`/`padding`/`gap` values in px, sorted by frequency. Round to the nearest 4 or 8 and you have the system.
- **Palette** — distinct `color` + `background-color` + `border-*-color` values, with `blendedBackgroundColors` to resolve translucency.
- **Radii / shadows / z-index ladder** — distinct values, again by frequency.

**Component inventory.** Cluster the layout tree by `(tagName, classList, boundingBox size bucket)`. Each cluster with >1 instance is a component; each with exactly 1 is a section-level one-off.

**Section-by-section rebuild.** Build top-to-bottom in document order, verifying each section before starting the next. A section is "done" when its own cropped diff is under target and its bbox deltas are zero.

**Reset discipline.** Match the original's reset. If the site uses `box-sizing: border-box` globally and you do not, every padded element is off by its padding. This single mismatch accounts for a large share of "everything is slightly wrong" diffs.

### Layer 3 — Verify

```
screenshot → diff → rank hotspots → attribute cause → fix → repeat
```

Hotspot ranking, in order of usefulness:

1. **Connected-component analysis on the diff mask.** `pixelmatch` returns a diff image; `odiff` can return `diffLines` + `diffCols`, which together give you the bounding rectangle of changes directly. ([odiff](https://github.com/dmtrKovalenko/odiff))
2. **Join hotspots to the DOM.** For each hotspot rect, find the `LayoutTreeSnapshot` nodes whose `bounds` intersect it. That converts "a red blob at y=1204" into "`.card__title` at (240,1204) is 3px too tall".
3. **Computed-style delta for the joined nodes.** Diff the captured whitelist against the rebuild's `CSS.getComputedStyleForNode`. The first differing property in cascade order is almost always the cause.
4. **Bounding-box delta table.** `{node, expected: {x,y,w,h}, actual: {x,y,w,h}, dx, dy, dw, dh}`. Sub-pixel deltas point at transform/rounding; whole-pixel deltas point at box model.

What to hand a coding agent (see §7 for metrics): the diff image, the hotspot list with joined selectors, the bbox delta table, and the computed-style delta — not just "the diff is 2.3%".

---

## 3. Tool matrix

| Tool | Purpose | Verdict / when to use | Limits | Link |
|---|---|---|---|---|
| **Playwright** | Browser automation, screenshots, emulation, clock control, CDP access | **Primary driver.** Best-in-class determinism controls (`locale`, `timezoneId`, `colorScheme`, `clock`, `stylePath`) and a first-class `CDPSession` | Chromium/Firefox/WebKit only; rendering varies by host OS | [playwright.dev](https://playwright.dev/docs/test-snapshots) |
| **Puppeteer** | Same class as Playwright, Chrome-only | Fine if you only need Chromium; weaker cross-browser story | Chrome/Chromium only | [pptr.dev](https://pptr.dev/) |
| **CDP `DOMSnapshot.captureSnapshot`** | DOM + layout tree + whitelisted computed styles in one call | **The single highest-value capture primitive.** Includes iframes, shadow DOM flattened, `currentSourceURL` for srcset, paint order, blended backgrounds | Experimental domain; text box positions documented as unstable across versions | [DOMSnapshot.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/DOMSnapshot.pdl) |
| **CDP `CSS.*` domain** | Cascade, computed styles, forced pseudo-states, media queries, platform fonts, rule-usage coverage | **Essential.** Reads cross-origin sheets the page cannot; `getPlatformFontsForNode` is the only reliable "what font actually rendered" | Experimental; `getMatchedStylesForNode` payloads are large | [CSS.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl) |
| **Playwright `toHaveScreenshot`** | Built-in visual assertion with baselines | Good for regression *after* the rebuild is close; not a capture tool | Uses pixelmatch; `threshold` default 0.2; baselines are browser+platform-scoped | [Visual comparisons](https://playwright.dev/docs/test-snapshots) |
| **pixelmatch** | JS pixel diff, OKLab + HyAB perceptual distance, AA detection | **Default diff engine.** `windowSize` for noise rejection, `ignoreMask` for dynamic regions, `checkerboard` for translucency | Single-threaded JS; slow on very large images | [pixelmatch](https://github.com/mapbox/pixelmatch) |
| **odiff** | Native (Zig, SIMD) pixel diff | **Fastest.** ~6× pixelmatch on a full cypress.io page in its own benchmark; `ignoreRegions`, `diffLines`/`diffCols` for bbox, `layout-diff` reason | Native binary per platform; MIT | [odiff](https://github.com/dmtrKovalenko/odiff) |
| **Resemble.js** | JS diff with bounding-box / ignored-box / color-key masking | Useful for its `ignoredBoxes` and `ignoreAreasColoredWith` masking model | **Ultra low-maintenance**; and it **skips pixels by default when width or height > 1200px** (`largeImageThreshold`) — a silent correctness trap for full-page shots | [Resemble.js](https://github.com/rsmbl/Resemble.js) |
| **ssim.js / image-ssim** | Structural similarity | Good as a *secondary* perceptual metric alongside pixel count | Not a localisation tool; says "how different", not "where" | [ssim.js](https://github.com/obartra/ssim) |
| **BackstopJS** | Scenario-driven visual regression with an HTML review UI | Good for multi-viewport, multi-selector scenario runs; `hideSelectors` / `removeSelectors` for dynamic regions | **Project needs a new maintainer** (stated in its README); default engine is Resemble.js; `misMatchThreshold` default 0.1 | [BackstopJS](https://github.com/garris/BackstopJS) |
| **reg-suit** | CLI visual regression with pluggable storage + CI notifiers | Good if you want snapshot storage in S3/GCS and PR comments; `x-img-diff-js` gives structural (inserted/moved) detection | Plugin ecosystem adds setup cost; thresholds are `thresholdRate` / `thresholdPixel` | [reg-suit](https://github.com/reg-viz/reg-suit) |
| **Lost Pixel** | OSS visual regression for Storybook/Ladle/Histoire/pages | **Do not adopt for new work** — the team announced they are joining Figma and are sunsetting the product | OSS core still works; platform is going away | [lost-pixel](https://github.com/lost-pixel/lost-pixel) |
| **Chromatic** | Hosted visual testing for Storybook/component libraries | Best if your rebuild is componentised and you want review UX | SaaS, paid at scale; tied to a component workflow | [chromatic.com](https://www.chromatic.com/docs/) |
| **Percy** | Hosted visual testing | Enterprise option; broad browser matrix | SaaS, paid | [percy.io](https://percy.io/) |
| **Argos** | Hosted visual testing, uses odiff under the hood | Modern alternative to Percy/Chromatic | SaaS | [argos-ci.com](https://argos-ci.com/docs) |
| **SingleFile** | Save a complete page into one self-contained HTML file | Best for a **human-readable reference artifact** — minifies and removes unused HTML/CSS, inlines assets | **AGPL** — commercial use needs a separate license; the CLI lives in a separate repo (`single-file-cli`) | [SingleFile](https://github.com/gildas-lormeau/SingleFile) |
| **monolith** | Bundle a page into one HTML file with assets as data URLs | Best for **offline reference + browser-rendered capture**: `-B` renders in a real browser first, `-F MHTML/WARC` for archive formats, `-j` freezes JS | No JS engine of its own (needs `-B`); public domain | [monolith](https://github.com/Y2Z/monolith) |
| **HTTrack** | Recursive site mirror to a local directory | Good for multi-page site structure; v3.51 (2026-10-28) now pulls every font and image a stylesheet names, and can mirror behind a login | GPL; static mirror only — no post-JS DOM, no computed styles | [httrack.com](https://www.httrack.com/) |
| **wget `--page-requisites`** | Single-page asset fetch | Zero-dependency fallback; `-p -k -E` for requisites + link conversion | No JS, no CSS `url()` resolution, no srcset | [GNU wget manual](https://www.gnu.org/software/wget/manual/wget.html) |
| **Chrome MHTML / Save Page WE** | Browser-native single-file archive | MHTML is the lowest-friction "what the browser had" snapshot | MHTML only fully supported in Chromium-based browsers | [MHTML](https://en.wikipedia.org/wiki/MHTML) |
| **web.archive.org** | Historical snapshots | Use when the live site has changed or is gone; also a legal safe-ish reference | Snapshots may be stale, partial, or missing assets; Wayback replay injects its own toolbar (mask it) | [web.archive.org](https://web.archive.org/) |
| **Chrome DevTools Coverage / `CSS.startRuleUsageTracking`** | Unused CSS/JS detection | **Use this to prune the rebuild's CSS.** `RuleUsage` gives `{styleSheetId, startOffset, endOffset, used}` per rule | Coverage is per-run; states you never triggered look "unused" | [Coverage panel](https://developer.chrome.com/docs/devtools/coverage), [CSS.pdl](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl) |
| **PostCSS / cssnano** | CSS transform + minify | Post-rebuild hygiene only | Do not run a minifier *before* diffing — it can change cascade order in edge cases | [postcss.org](https://postcss.org/), [cssnano.co](https://cssnano.co/) |

**2025–2026 landscape notes.** Two OSS staples are decaying: BackstopJS is explicitly looking for a maintainer, and Lost Pixel is sunsetting as its team joins Figma. Both are still usable but should not be the load-bearing choice for a new playbook. The durable stack is Playwright + CDP for capture and odiff/pixelmatch for diffing — everything else is a review-UI convenience on top.

---

## 4. Top 25 pixel-diff causes, ranked

Ranked by how often they are the actual cause in practice. "Detect" assumes you have the §2 capture artifacts.

| # | Cause | Detect | Fix |
|---|---|---|---|
| 1 | **Wrong font family** — webfont failed to load, fell back | `CSS.getPlatformFontsForNode` → `isCustomFont: false`; `document.fonts` status `error`/`unloaded` | Ship the licensed font file; check `@font-face` `src` URL and CORS |
| 2 | **Font weight/size not exactly matched** — 500 vs 600, 15px vs 15.2px | Font tuple diff across the layout tree | Copy the exact tuple; do not "round" to a familiar scale |
| 3 | **`letter-spacing` / `word-spacing` drift** | Computed-style delta on text nodes | Copy exact values, including `-0.01em`-style negatives |
| 4 | **`line-height` `normal` vs numeric** | Computed `line-height` is `normal` on one side, a px value on the other | Resolve `normal` to the computed px and hardcode it |
| 5 | **Font subset missing glyphs** — some characters fall back | `glyphCount` split across multiple `PlatformFontUsage` entries on one node | Capture all `unicodeRange` subsets for the family |
| 6 | **Box model: `box-sizing` mismatch** | Every padded element off by its padding | Match the original's global reset |
| 7 | **Missing CSS reset** — UA default margins/headings | Diff concentrated on `h1`–`h6`, `p`, `ul`, `body` | Port the original reset (normalize/modern-reset) |
| 8 | **Root font-size / `rem` base mismatch** | All `rem`-derived sizes scale wrong together | Set `html { font-size: ... }` to the captured value |
| 9 | **Spacing scale drift** — 24px vs 25.6px from a different scale | Distinct spacing-value frequency table | Use the extracted scale, not a guessed one |
| 10 | **Image asset mismatch** — different crop, compression, or DPR variant | Per-asset checksum comparison | Copy the exact bytes, or record the substitution as an accepted diff |
| 11 | **Wrong `srcset` candidate chosen** | `img.currentSrc` vs your rebuild's `currentSrc` | Match `sizes` + `srcset` + viewport exactly |
| 12 | **Flex/grid `gap` and alignment defaults** | bbox deltas cluster on flex/grid children | Copy `justify-content`, `align-items`, `gap`, `flex-basis` explicitly |
| 13 | **Color space / notation drift** — `#fff` vs `rgb()` vs `oklch()`, gamma | Computed color string differs, or matches but renders differently | Compare computed `rgb()` values, not source notation |
| 14 | **Opacity / `backdrop-filter` / blend mode** | `includeTextColorOpacities` + `blendedBackgroundColors` differ | Copy the full stacking of translucent layers, not just the top one |
| 15 | **`border-radius` + `overflow` clipping** | Corner-region diffs on cards/images | Copy both `border-radius` and `overflow`; `overflow: hidden` is load-bearing |
| 16 | **`box-shadow` spread/blur/alpha** | Soft-edged halo diffs | Copy the full shorthand; shadows are 4 numbers and a color |
| 17 | **Gradient angle / color-stop / interpolation space** | Banded diffs inside gradient regions | Copy stops and the interpolation method (`in oklab` etc.) |
| 18 | **Pseudo-element content** — `::before`/`::after` icons, counters, quotes | Node count mismatch; capture misses pseudo nodes | Enumerate `CSS.getMatchedStylesForNode(...).pseudoElements` and copy `content` |
| 19 | **z-index / stacking context / paint order** | Overlap regions show the wrong layer on top | Use `LayoutTreeSnapshot.paintOrders` + `stackingContexts` as ground truth |
| 20 | **Sub-pixel `transform` / fractional layout** | bbox deltas of 0.5–1px, or text 1px shifted | Copy `transform` exactly; avoid "cleaning up" fractional values |
| 21 | **Scrollbar width affecting layout** | Content width differs by ~15px on scrollable vs not | Match `overflow` and consider `scrollbar-gutter`; capture with the same scrollbar mode |
| 22 | **Sticky/fixed header offset captured mid-scroll** | Header appears at wrong y | Capture at `scrollY = 0` and at a known scroll offset; mask sticky elements when they must move |
| 23 | **Animation/transition captured mid-flight** | Diff appears only sometimes; vanishes on re-run | Determinism stylesheet + `page.clock` (§2 Layer 0) |
| 24 | **Lazy-loaded images never triggered** (`IntersectionObserver`) | Blank boxes at the bottom of a full-page shot | Scroll the page to the bottom and back before capture; wait for `networkidle` |
| 25 | **Third-party embed / iframe not loaded or non-deterministic** | Whole-region diff inside an iframe rect | Mask the region; see §5 |

Two more that are really "cause #1 in disguise": **DPR mismatch** (captured at DPR 2, rebuilt at DPR 1 → everything is half-size or blurry) and **dark mode** (`prefers-color-scheme` differs between capture and rebuild). Both are prevented by Layer 0, not fixed in Layer 2.

---

## 5. Hard cases and how to scope them

| Hard case | Why it is hard | Recommendation |
|---|---|---|
| **Canvas / WebGL charts, 3D** | No DOM to inspect; WebGL `toDataURL()` returns blank unless the context was created with `preserveDrawingBuffer: true` | **Scope out the pixels, capture the frame.** Screenshot the canvas element as a static image, mark it `dynamic: true` in the manifest, and rebuild with a placeholder of identical dimensions. Do not attempt to reimplement the chart. |
| **Maps (Google/Mapbox)** | Tiles are licensed, key-gated, and geolocation-dependent | Placeholder with identical bbox. Never re-host tiles. |
| **Video** | Streaming manifests, DRM, autoplay policy | Capture the poster frame; rebuild with the poster + a play affordance. |
| **YouTube / Vimeo / Stripe / Intercom embeds** | Third-party origin, own fonts and styles, non-deterministic content | Keep the embed as an embed. Mask it in diffs. Never re-host the content. |
| **A/B tests and experiments** | Two visitors see different DOMs; you may capture variant B | Capture twice from clean contexts and diff the two captures against each other *first* to detect experiment variance. If they differ, freeze the variant via cookie/localStorage or scope the section out. |
| **Geo / currency / pricing variants** | Server-side or client-side locale branching | Pin `locale` + `timezoneId` + `Accept-Language`, and capture from a single egress IP. Record the IP/region in the manifest. |
| **Cookie banners / consent managers** | Often injected late, often A/B'd, often overlay-blocking | Handle in the harness: either dismiss deterministically or mask the region. Prefer masking — dismissal changes layout. |
| **Service workers** | Serve cached/stale assets; can make a "fresh" capture non-reproducible | Capture in a fresh context with SW disabled, or clear storage between runs. Record the policy in the manifest. |
| **CSP / SRI** | A rebuild that self-hosts assets will break SRI hashes; a strict CSP may block your injected determinism stylesheet | For the *capture*, do not fight CSP — use CDP (out-of-band). For the *rebuild*, recompute SRI hashes or drop the attributes. |
| **SPA hydration** | The DOM before hydration differs from after; screenshots taken too early capture the skeleton | Wait for a hydration signal (a `readySelector`, a console event, or `networkidle`), then add a fixed settle delay. |
| **Closed shadow DOM** | `el.shadowRoot` is `null` | CDP `DOMSnapshot` flattens shadow trees regardless — this is a reason to prefer CDP over in-page JS. |
| **Cross-origin iframes** | Cannot read into them from the page | `page.frames()` + a separate CDP target per frame; or mask. |
| **`@layer`, `@scope`, `@container`, `@starting-style`** | Modern cascade features that a naive property-copy rebuild loses | Capture the layer tree (`CSS.getLayersForNode`), container queries, and `@starting-style` via `getMatchedStylesForNode`; reproduce the *structure*, not a flattened property bag. |

**Scoping principle:** every hard case gets one of three verdicts, recorded in the manifest — **rebuild** (fully reproducible), **placeholder** (same bbox, static asset), or **mask** (excluded from diff). Never leave a hard case in the "just diff it and hope" bucket; it will poison your metrics forever.

---

## 6. Determinism checklist for screenshot comparison

Environment:
- [ ] Same browser build and channel on capture and verify (pin the Playwright version; pin the Chromium revision).
- [ ] Same OS and OS font set. Playwright explicitly warns that rendering varies by host OS, and BackstopJS ships a Docker mode for exactly this reason. ([Visual comparisons](https://playwright.dev/docs/test-snapshots), [BackstopJS](https://github.com/garris/BackstopJS))
- [ ] `viewport` pinned; `deviceScaleFactor` pinned to 1 for diffing.
- [ ] `locale` + `timezoneId` pinned; `TZ` set for the runner process.
- [ ] `colorScheme` pinned (`light`), and `prefers-color-scheme` verified in the capture.
- [ ] `reducedMotion: 'reduce'` set, and the site's `prefers-reduced-motion` branch verified — note this *changes* the design, so record which branch you captured. ([MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion))

Timing:
- [ ] `page.clock.setFixedTime(...)` or `install()` before navigation. ([Clock](https://playwright.dev/docs/clock))
- [ ] Determinism stylesheet injected (animations/transitions/caret/scroll-behavior).
- [ ] `await document.fonts.ready` before screenshot. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Document/fonts))
- [ ] All `<img>` `complete === true` and `naturalWidth > 0`.
- [ ] Scroll to bottom, wait, scroll back to top (triggers `IntersectionObserver` lazy loads).
- [ ] `networkidle` (or a `readySelector`) plus a fixed settle delay.
- [ ] Pointer parked at `(-1, -1)` so no hover state is live. ([Visual comparisons](https://playwright.dev/docs/test-snapshots))

Content:
- [ ] Dynamic regions masked via a declared list — `stylePath` for `toHaveScreenshot`, `ignoreMask` for pixelmatch, `ignoreRegions` for odiff. ([Visual comparisons](https://playwright.dev/docs/test-snapshots), [pixelmatch](https://github.com/mapbox/pixelmatch), [odiff](https://github.com/dmtrKovalenko/odiff))
- [ ] Cookie banner handled by a recorded, deterministic rule.
- [ ] Timezone/locale-dependent text (dates, currency) either pinned or masked.
- [ ] Service worker policy recorded.
- [ ] Full-page height stable across two consecutive captures (capture twice, diff the two captures — if they differ, you are not deterministic yet).

Harness self-test:
- [ ] Capture the same page twice and assert **0 differing pixels**. If this fails, fix determinism before touching the rebuild.

---

## 7. Verification metrics

| Metric | What it measures | How to compute | Realistic target |
|---|---|---|---|
| **Pixel mismatch %** | Raw area difference | `diffCount / (width*height)` from pixelmatch or odiff `diffPercentage` | Tier 1 (fonts+assets perfect): **< 0.1%**. Tier 2 (good rebuild): **< 1%**. Tier 3 (structurally right): **< 5%** |
| **Windowed diff count** | Whether differences are *clustered* (real) or *scattered* (noise) | `pixelmatch(..., { windowSize: 16 })` — returns max diff pixels in any 16×16 window | A real regression produces a high windowed count in one place; AA noise does not. ([pixelmatch](https://github.com/mapbox/pixelmatch)) |
| **SSIM** | Perceptual structural similarity | `ssim.js` on grayscale | **> 0.99** for a good rebuild; **> 0.995** for near-perfect |
| **Per-region diff** | Localised mismatch | Diff each section crop independently | Each section **< 0.2%**; no section above 2% |
| **Bbox delta** | Geometry correctness, independent of paint | `LayoutTreeSnapshot.bounds` (document coords) vs rebuild's, joined by selector | **dx/dy/dw/dh = 0** for > 95% of nodes; max abs delta ≤ 1px |
| **Computed-style delta** | CSS correctness | `CSS.getComputedStyleForNode` whitelist diff, per matched node | **0 differing properties** on the whitelist for structurally matched nodes |
| **Font match** | The #1 cause, isolated | `CSS.getPlatformFontsForNode` → compare `{postScriptName, isCustomFont}` sets | **Exact match** on every text node. This is a binary gate, not a percentage. |
| **Asset checksum match** | Byte-identical assets | SHA-256 of each harvested asset vs rebuild's | **100%** for assets you are licensed to copy; every miss must be a declared substitution |
| **Text box delta** | Per-line text placement | `TextBoxSnapshot.bounds` diff | ≤ 1px per line. Treat as advisory across Chrome versions (the protocol documents these as unstable). |

**Important caveat on thresholds.** Playwright's `toHaveScreenshot` `threshold` (default 0.2) is a **per-pixel perceptual color difference**, not a share of the image — to allow a percentage of the image to differ you compare the returned count against `width * height`. Conflating the two is the most common way teams set a "1% tolerance" that actually means something else entirely. ([pixelmatch docs](https://github.com/mapbox/pixelmatch), [Visual comparisons](https://playwright.dev/docs/test-snapshots))

**Also:** if you use Resemble.js anywhere in the pipeline, set `largeImageThreshold: 0`. Its default of 1200 means it **skips pixels entirely** on images wider or taller than 1200px — which is every full-page screenshot. ([Resemble.js](https://github.com/rsmbl/Resemble.js))

---

## 8. Open questions / what stays manual

1. **Font licensing is never automatable.** You can detect the font; you cannot detect whether you may copy it. The substitution decision is human. See [05](./05-LEGAL-AND-ETHICS.md) §3.
2. **"Which pixels matter" is a judgement call.** A 0.4% diff concentrated on a hero headline is worse than a 2% diff spread across a masked carousel. Metrics rank; humans decide.
3. **Variable-font axis matching.** `FontVariationAxis` gives you the axis ranges, but mapping `font-variation-settings` from a design tool to the rendered instance is still manual.
4. **Canvas/WebGL reimplementation** is out of scope for a cloning playbook. Capture the frame, move on.
5. **Cross-browser fidelity.** Everything here is Chromium-centric. Firefox and WebKit render text and some layout differently; a Chromium-perfect clone is not a Safari-perfect clone. Multi-engine verification is a separate, much harder project.
6. **A/B experiment detection** has no reliable automated signal. Two clean-context captures diffed against each other is the best available heuristic.
7. **Whether the rebuild should preserve the original's DOM structure or be idiomatic.** Preserving structure makes the diff converge faster; idiomatic markup is better engineering. The playbook should probably say: preserve structure during convergence, refactor after the diff is closed.
8. **Long-term tool risk.** BackstopJS is unmaintained and Lost Pixel is sunsetting. Any playbook that hard-codes them will rot. Prefer Playwright + CDP + a diff library.
9. **The `@layer` / `@scope` / `@container` reproduction problem.** Capturing the structure is solved (`getMatchedStylesForNode`, `getLayersForNode`); emitting it faithfully in a hand-written rebuild is not.
10. **No verified 2025–2026 "agentic visual QA" standard has emerged.** The tooling that exists is hosted visual-regression SaaS with an agent-friendly API, not an agent-native feedback protocol. Treat "give the agent a diff image + bbox table + style delta" as the current best practice, not a settled standard.

---

## 9. Sources

All fetched during this research pass (2026-11-11).

**Chrome DevTools Protocol (authoritative, from the protocol source)**
- [CDP browser_protocol.pdl (domain index)](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/browser_protocol.pdl)
- [CDP CSS domain — `getComputedStyleForNode`, `getMatchedStylesForNode`, `forcePseudoState`, `getMediaQueries`, `getPlatformFontsForNode`, `startRuleUsageTracking`, `getLayersForNode`, `FontFace`, `RuleUsage`](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/CSS.pdl)
- [CDP DOMSnapshot domain — `captureSnapshot`, `LayoutTreeSnapshot`, `TextBoxSnapshot`, `currentSourceURL`](https://raw.githubusercontent.com/ChromeDevTools/devtools-protocol/master/pdl/domains/DOMSnapshot.pdl)

**Playwright**
- [Visual comparisons (`toHaveScreenshot`, pixelmatch, `threshold`, `stylePath`, hover caveat, cross-environment warning)](https://playwright.dev/docs/test-snapshots)
- [Emulation (viewport, locale, timezone, colorScheme, devices)](https://playwright.dev/docs/emulation)
- [Clock (`setFixedTime`, `install`, `pauseAt`, overridden globals)](https://playwright.dev/docs/clock)
- [Screenshots (fullPage, element, buffer)](https://playwright.dev/docs/screenshots)

**Diffing / visual regression**
- [pixelmatch — OKLab/HyAB, `windowSize`, `ignoreMask`, `checkerboard`, CLI exit codes](https://github.com/mapbox/pixelmatch)
- [odiff — Zig/SIMD, `ignoreRegions`, `diffLines`/`diffCols`, `layout-diff`, benchmarks](https://github.com/dmtrKovalenko/odiff)
- [Resemble.js — `largeImageThreshold` 1200 default, `ignoredBoxes`, low-maintenance status](https://github.com/rsmbl/Resemble.js)
- [ssim.js](https://github.com/obartra/ssim)
- [BackstopJS — maintainer-needed notice, `misMatchThreshold`, `hideSelectors`, Docker mode](https://github.com/garris/BackstopJS)
- [reg-suit — thresholds, `x-img-diff-js` structural diff, plugins](https://github.com/reg-viz/reg-suit)
- [Lost Pixel — Figma acquisition / sunset notice](https://github.com/lost-pixel/lost-pixel)
- [Chromatic docs](https://www.chromatic.com/docs/)
- [Argos docs](https://argos-ci.com/docs)
- [Percy](https://percy.io/)

**Archivers**
- [SingleFile — AGPL, format comparison table, CLI in separate repo](https://github.com/gildas-lormeau/SingleFile)
- [monolith — `-B` browser render, `-F MHTML/WARC`, `-j`, scroll limit, public domain](https://github.com/Y2Z/monolith)
- [HTTrack 3.51-0 (2026-10-28) — stylesheet font/image capture, login/CAPTCHA mirroring, GPL](https://www.httrack.com/)
- [GNU wget manual](https://www.gnu.org/software/wget/manual/wget.html)
- [MHTML (Wikipedia)](https://en.wikipedia.org/wiki/MHTML)
- [web.archive.org](https://web.archive.org/)
- [Chrome DevTools Coverage panel](https://developer.chrome.com/docs/devtools/coverage)
- [PostCSS](https://postcss.org/) · [cssnano](https://cssnano.co/)

**Web platform**
- [MDN: CSSStyleSheet.cssRules](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet/cssRules)
- [CSSOM spec: `dom-cssstylesheet-cssrules`](https://drafts.csswg.org/cssom/#dom-cssstylesheet-cssrules)
- [MDN: Document.fonts / FontFaceSet](https://developer.mozilla.org/en-US/docs/Web/API/Document/fonts)
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion)

**Legal / ethics (detail in 05)**
- [RFC 9309 — Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309.html)
- [SIL Open Font License](https://openfontlicense.org/)
- [Adobe Fonts licensing FAQ](https://helpx.adobe.com/fonts/web/font-licensing/font-licensing.html)
- [Cloudflare bot solutions](https://developers.cloudflare.com/bots/)
- [hiQ Labs v. LinkedIn](https://en.wikipedia.org/wiki/HiQ_Labs_v._LinkedIn)

**Method note.** `web_search` was unavailable in this session (search endpoint returned HTTP 401), so all grounding came from direct `web_fetch` of primary sources: the CDP protocol `.pdl` files, vendor documentation, and project READMEs. That is arguably better provenance than search snippets, but it means the tool landscape was sampled from known projects rather than discovered by search — a follow-up pass with working search should check for 2026-era entrants not covered here.
