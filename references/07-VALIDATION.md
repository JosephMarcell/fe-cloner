# 07 — Validation record

What was actually run, on what, with what numbers. This is the evidence that the toolkit
and the workflow work — not a claim, a transcript.

Environment: Windows, Node 24.22.2, Playwright 1.65.0, Chromium 158.0.8127.18,
DPR 1, locale `en-US`, timezone `UTC`, colour scheme light, reduced motion `reduce`.

---

## Test 1 — Capture a minimal static page

```powershell
node capture.mjs --url https://example.com --out out/example-com --viewports desktop:1440x900,mobile:390x844 --scroll --states --deep
```

| Metric | Result |
| --- | --- |
| Nodes | 14 (14 visible) |
| Stylesheets | 2 (both readable, non-empty) |
| Assets | 2 |
| Fonts | 0 (correct — the page uses system fonts) |
| Screenshots | desktop 1440×900, mobile 390×844, full-page both |
| Document size | 1440×900 desktop, 390×876 mobile |
| Warnings | none |

Verdict: capture gate passed.

## Test 2 — Capture a real site with a build pipeline

```powershell
node capture.mjs --url https://playwright.dev --out out/playwright-dev --viewports desktop:1440x900,mobile:390x844 --scroll --states --deep
```

| Metric | Result |
| --- | --- |
| Nodes | 246 (224 visible) |
| Stylesheets | 1 — `styles.3714015c.css`, 216,805 bytes |
| Assets | 38 |
| Breakpoints detected | 420, 576, 768, 966, 996, 997, 1200, 1300, 1400, 1440 |
| Document height | 2848 desktop, 5126 mobile |
| Warnings | none |

Verdict: capture gate passed. The 216KB stylesheet was fully retrieved and re-serialised
from the CSSOM.

## Test 3 — Analyze

```powershell
node analyze.mjs --capture out/playwright-dev/capture --out out/playwright-dev/analysis
```

| Metric | Result |
| --- | --- |
| Sections detected | 8, covering the full document |
| Component candidates | 8 |
| Distinct colours | 20 |
| Font sizes | 9 |
| Spacing scale | 21 values |

Section boundaries matched the page's real structure: `header.hero` (y=60, h=492),
`section.pathCards` (y=552, h=352), three `featureSection` blocks, `crossBrowser`,
`logosSection`, `footer`. A 1994px `main` element was automatically split into its
children because it owned 70% of the page.

Verdict: the plan is actionable — one build step per real section.

## Test 4 — Baseline rebuild

```powershell
node rewrite.mjs --capture out/playwright-dev/capture --out out/playwright-dev/site --neutralize-links
node run.mjs --url https://playwright.dev --out out/playwright-dev --stage verify
```

| Metric | Result |
| --- | --- |
| URLs rewritten | 109 |
| Files copied | 37 |
| Links neutralised | 30 |
| Integrity attributes removed | 1 |
| **Desktop pixel mismatch** | **0.0000%** (0 of 1,296,000 px) |
| **Mobile pixel mismatch** | **0.0000%** (0 of 329,160 px) |
| Desktop SSIM | 1.0 |
| **Audit score** | **0.9961** |
| Nodes matched | 246/246, 0 missing, 0 extra |
| bbox within 1px | 246/246, mean delta 0px |
| Style mismatches | 0 |
| Assets | 37/38 ok, 0 hash mismatches |

Verdict: the baseline-first strategy delivers a pixel-identical starting point before a
single line of rebuild code is written. This is the core claim of the playbook, and it
holds.

## Test 5 — The audit detects a real deviation

The baseline site was copied and a deliberate 2-property break injected:

```html
<style>h1{letter-spacing:2px !important}.navbar{padding-top:4px !important}</style>
```

| Metric | Baseline | Broken | Delta |
| --- | --- | --- | --- |
| Desktop mismatch | 0.0000% | **15.0327%** | +15.03 |
| Mobile mismatch | 0.0000% | **5.4715%** | +5.47 |
| Audit score | 0.9961 | **0.7398** | −0.2563 |
| bbox within 1px | 246/246 | 17/246 | −229 |
| Mean bbox delta | 0px | 58.34px | +58.34 |
| Style mismatches | 0 | 19 | +19 |
| Exit code | 0 | 1 | fails the gate |

Top hotspot reported:

```
html > div#__docusaurus_skipToContent_fallback > header > div > h1
  reason: bbox+styles
  props:  height, letter-spacing, transform-origin
```

Verdict: the feedback loop works. A 2px letter-spacing change is detected as a pixel
regression, attributed to the correct element, and named as the correct CSS property.
This is the mechanism the whole workflow depends on.

## Test 6 — Diff tool self-check

| Case | Expected | Observed |
| --- | --- | --- |
| Identical images | 0.0000%, pass | 0.0000%, SSIM 1.0, pass |
| Shifted content | fail with a hotspot in the shifted cell | fail, hotspot in the correct grid cell |
| Different image sizes | `size-mismatch`, exit 1, no crash | `size-mismatch`, exit 1 |

## Test 7 — Crop for section-level diffs

```powershell
node crop.mjs --in out/example-com/capture/screenshots/desktop.png --rect "0,0,720,450" --out out/example-com/crops/test.png
```

Result: 720×450 crop written from a 1440×900 source at origin 0,0. Correct.

---

## Test 8 — A JS-heavy site with an animated hero (the honest failure case)

```powershell
node capture.mjs --url https://tailwindcss.com --out out/tailwindcss-com --viewports desktop:1440x900 --states
node rewrite.mjs --capture out/tailwindcss-com/capture --out out/tailwindcss-com/site --neutralize-links
node run.mjs --url https://tailwindcss.com --out out/tailwindcss-com --stage verify --candidate http://127.0.0.1:4195/
```

| Metric | Result |
| --- | --- |
| Nodes | 2294 (2073 visible) |
| Stylesheets | 2, 674KB + 3KB |
| Fonts | 10 faces, 11 files (Inter variable 352KB + italic 388KB, IBM Plex Mono ×6, Source, Ubuntu Mono) |
| Assets | 95 / 9.8MB |
| **Desktop mismatch** | **5.4276%** (70,342 of 1,296,000 px) |
| Audit score | 0.9919 |
| Nodes matched | 2293/2294 |
| Style mismatches | 8 |
| Assets | 93/95 ok |

Diagnosis, using the tools:

- `diff/desktop/desktop.regions.json` put the worst cells at x=1080–1260, y=675–900
  (73% and 70% mismatch) — the bottom-right of the hero.
- Querying `nodes.json` for text elements in that rect returned exactly one:
  a `span` reading `"Class Warfare"` in the Inter font.
- The region is the hero's **animated code preview** (a typewriter/carousel). The capture
  froze it at one frame; the rebuilt page renders a different frame.
- The audit's bbox hotspots, meanwhile, were dominated by **off-canvas carousel slides**
  (`x = -690` in the reference, `x = 56` in the candidate) — real differences, but not
  visible pixels.

Verdict: this is the expected behaviour for a page whose hero is animated, and it is the
single most important lesson in the whole validation set:

1. The diff tells you *where the visible damage is* (viewport coordinates).
2. The audit tells you *what changed structurally* (document coordinates, including
   off-canvas elements).
3. Neither is wrong — but you must filter audit hotspots to the viewport rect before
   chasing them, and you must mask or clock-pin animated regions before expecting a
   static page to converge.

Fix for this case: `--mask` the hero preview region in `diff.mjs`, or pin
`page.clock.setFixedTime()` in the capture so both sides render the same frame. Both are
documented in the playbook; neither is a CSS bug.

## Test 9 — `--strip-scripts` on a Next.js site

| Build | Desktop mismatch | Scripts stripped |
| --- | --- | --- |
| With app scripts | 5.4276% | 0 |
| `--strip-scripts` | 5.4276% | 23 |

Verdict: stripping the app bundle is safe (identical result) and removes the hydration
risk and the service-worker 404 noise. It did not fix the animated hero, because the
hero's state is baked into the captured DOM, not re-rendered by JS. Use it by default on
SPA/Next baselines.

---

## Test 10 — The script re-render trap (found by integration testing)

Running the whole pipeline from the repo root on `https://example.com` initially failed:

| Metric | With app scripts | `--strip-scripts` |
| --- | --- | --- |
| Candidate nodes | 25 (ref 14) | 14 |
| Document height | 1332px (ref 900px) | 900px |
| Desktop mismatch | 2.1915% | **0.0000%** |
| Mobile mismatch | 8.0876% | **0.0000%** |
| Audit score | 0.6934 | **1.0** |
| Exit code | 1 | 0 |

Cause: the captured page carried a `<script src="…/s.js">`. That script runs again in the
offline baseline and **duplicates the whole body** — 11 extra nodes, an extra `<svg>`, and
a page 432px taller. The captured DOM was already hydrated; re-running the bundle mutated
it.

This is the general rule, not an example.com quirk: **a visual baseline must be static.**
`rewrite.mjs --strip-scripts` removes external bundles and service-worker registration, and
`run.mjs` now passes it by default in the rewrite stage. If you rewrite manually, pass it
yourself.

## Test 11 — Rendered-font truth and substitution detection

`capture/fonts/rendered-fonts.json` records what the engine actually rendered, via CDP
`CSS.getPlatformFontsForNode`. On tailwindcss.com, 9 groups:

| Requested stack | Weight | Rendered `familyName` | PostScript | Custom | Glyphs |
| --- | --- | --- | --- | --- | --- |
| `inter, "inter Fallback", system-ui` | 400 | Inter Variable | `InterVariable-Text` | yes | 6 |
| `inter, …` | 500 | Inter Variable | `InterVariable-Text-Medium` | yes | 4 |
| `inter, …` | 600 | Inter Variable | `InterVariable-Text-SemiBold` | yes | 11 |
| `plexMono, …` | 400 | IBM Plex Mono | `IBMPlexMono` | yes | 52 |
| `plexMono, …` | 500 | IBM Plex Mono Medm | `IBMPlexMono-Medm` | yes | 4 |
| `plexMono, …` | 600 | IBM Plex Mono SmBld | `IBMPlexMono-SmBld` | yes | 8 |

`counts.substitutedFonts = 0`. The `… Fallback` faces Next.js injects are ignored by the
matcher, so `plexMono` → `IBM Plex Mono` is correctly *not* a false positive.

The negative case was proven by re-capturing with `--block InterVariable`: the rendered
family became `Arial`, `isCustomFont: false`, `substitutedFonts = 5`, and five
`font substitution: requested … rendered "Arial" at <path>` warnings.

Verdict: font substitution — the single most damaging fidelity failure — is now a
detected, named, binary gate rather than a mystery 0.4% diff.

## Test 12 — Engine media queries

`breakpoints.json.engineQueries` comes from `CSS.getMediaQueries`. tailwindcss.com: 31
queries, 13 active at 1440px, sources including `mediaRule` with the originating
`sourceURL`. Merged widths: `[320, 600, 640, 712, 768, 877, 1024, 1280, 1536]` — a
superset of what the regex found, including range syntax the regex missed.

## Test 13 — The 0.44% launch-flag trap

A pixel-perfect clone (audit score 1.0, zero style deltas) was screenshotted **without**
the capture's text-rendering launch flags:

| Candidate screenshot flags | Mismatch |
| --- | --- |
| `--font-render-hinting=none --disable-lcd-text --force-color-profile=srgb` | 0.0000% |
| Browser defaults | **0.44%** |

Same DOM, same CSS, same assets, same fonts. The entire 0.44% was LCD subpixel
antialiasing on text — 9× the 0.05% target, from a missing launch flag. Any screenshot
fed to `diff.mjs` must use the same flags as `capture.mjs` (and `run.mjs` does).

---

## Known gaps (as of this validation)

| Gap | Impact | Mitigation |
| --- | --- | --- |
| Webfont capture was validated only after the first pass (getbootstrap.com, tailwindcss.com) | The earliest captures reported `fonts: 0` | Re-capture any target whose manifest shows 0 fonts; verify `fonts[].status` |
| Service worker registration survives the rewrite | 404 noise in the console, no visual effect | `--strip-scripts` on `rewrite.mjs` |
| SPA JS chunks 404 on the offline baseline | Console errors, no visual effect (DOM is already hydrated) | Expected; the baseline is a visual reference, not a running app |
| Closed shadow DOM not reachable | Elements inside are invisible to capture | Out of scope; note as a hard case |
| Canvas content captured as a PNG snapshot, not live | Static image instead of a live chart | Correct trade-off for fidelity; re-render or embed if interactivity is needed |

## Reproduce it

```powershell
cd tools
node doctor.mjs
node capture.mjs --url https://playwright.dev --out out/playwright-dev --viewports desktop:1440x900,mobile:390x844 --scroll --states --deep
node analyze.mjs --capture out/playwright-dev/capture --out out/playwright-dev/analysis
node rewrite.mjs --capture out/playwright-dev/capture --out out/playwright-dev/site --neutralize-links
node run.mjs --url https://playwright.dev --out out/playwright-dev --stage verify
```

Then inject a break into `out/playwright-dev/site/index.html`, re-run the last command,
and confirm the mismatch and audit score move.
