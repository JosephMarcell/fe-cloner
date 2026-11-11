# 04 — Fidelity checklist

Gate checklists for each phase. Copy this file into `out/<slug>/CHECKLIST.md` and tick
as you go. A phase is not done until every box is checked and every number is read from
a report file.

---

## Gate 0 — Preflight

- [ ] Authorisation basis recorded (own site / written permission / private study).
- [ ] `robots.txt` and ToS checked; no scraping restriction violated.
- [ ] Font licences identified; any unlicensable font flagged as a known deviation.
- [ ] Target state chosen and written down (logged out, no banner, slide 1, tab 1).
- [ ] Two captures compared → **STABLE**.
- [ ] Environment pinned: browser build, viewport, DPR 1, locale, timezone, reduced
      motion, colour scheme.
- [ ] Primary viewport chosen (default `1440x900`).
- [ ] Dynamic regions identified and either neutralised or scheduled for masking.
- [ ] `PROVENANCE.md` created.

---

## Gate 1 — Capture

- [ ] `screenshots/desktop.png` exists, is 1440×900, and is not blank.
- [ ] `screenshots/desktop-full.png` height matches `documentSize.height` ± 2px.
- [ ] `css/index.json`: every sheet has `bytes > 0` and its `local` file exists.
- [ ] Cross-origin sheets show `crossOriginBlocked: true` **and** non-empty CSS.
- [ ] `fonts/fonts.json`: `faces > 0`; no face has `status: "error"`.
- [ ] `assets/asset-map.json`: no `4xx` on any `img[src]` or `css:background-image`.
- [ ] `dom/nodes.json`: node count plausible; `truncated: false`.
- [ ] `breakpoints.json`: `widths` non-empty (or the page is genuinely fluid).
- [ ] `interactions.json`: hover/keyframe lists populated.
- [ ] `network.json`: no failed document/stylesheet request.
- [ ] `capture/report.md` written and warnings reviewed.

**Stop if:** any critical asset is missing, or fonts are `error`, or the DOM varies
between runs.

---

## Gate 2 — Analyze

- [ ] `analysis/sections.json` covers the whole document height (last section ends at
      `documentSize.height` ± 20px).
- [ ] `analysis/tokens.json`: colour palette, font scale, spacing scale extracted.
- [ ] `analysis/components.md`: repeated signatures listed.
- [ ] `analysis/BRIEF.md` written with build order and hard cases.
- [ ] Hard cases scoped out explicitly (canvas, iframes, embeds, closed shadow DOM).

---

## Gate 3 — Baseline

- [ ] `rewrite.mjs` produced `site/index.html` with zero unresolved critical URLs.
- [ ] Baseline desktop mismatch **< 1.5%**.
- [ ] Baseline audit score **> 0.90**.
- [ ] Zero missing assets in the baseline.
- [ ] Page renders offline (no requests to the target origin).

**Stop if:** baseline is worse than 1.5%. The capture or rewrite is broken; refactoring
on a broken baseline wastes the whole loop.

---

## Gate 4 — Rebuild (per section)

For each section in `analysis/plan.md`, largest area first:

- [ ] Section crop reference created.
- [ ] Markup uses only tokens from `tokens.json` (no magic hex values).
- [ ] Section mismatch **< 0.1%**.
- [ ] Section audit hotspots: zero layout deltas > 1px.
- [ ] No regression in previously completed sections (whole-page mismatch did not rise).
- [ ] Interactive elements in the section have hover + focus states.

---

## Gate 5 — Verify (whole page)

| Metric | Target | Hard stop |
| --- | --- | --- |
| Desktop pixel mismatch | < 0.05% | < 0.1% |
| Tablet pixel mismatch | < 0.1% | < 0.2% |
| Mobile pixel mismatch | < 0.2% | < 0.3% |
| SSIM (desktop) | > 0.999 | > 0.995 |
| Audit score | > 0.98 | > 0.95 |
| Missing assets | 0 | 0 |
| Asset SHA-256 mismatches | 0 | 0 |
| Font-family mismatches | 0 | 0 |
| Nodes missing | 0 | < 2% |
| bbox delta > 5px | 0 | 0 |

- [ ] Full-page diff reviewed; every remaining hotspot has a written explanation.
- [ ] Remaining hotspots classified as: AA noise / dynamic content / known deviation.
- [ ] Three-strike rule not exceeded on any hotspot.

---

## Gate 6 — Responsive

- [ ] Every captured breakpoint screenshotted and diffed.
- [ ] No horizontal scrollbar at 320px.
- [ ] No overlapping or clipped text at any breakpoint.
- [ ] Tap targets ≥ 44px on mobile.
- [ ] Images use the same `srcset` candidates as the target at each DPR.
- [ ] Mobile nav / hamburger reproduced and functional.

---

## Gate 7 — States and motion

- [ ] Every `:hover` selector from `interactions.json` reproduced.
- [ ] Every `:focus` state visible and accessible.
- [ ] All `@keyframes` reproduced with matching timing.
- [ ] Scroll effects (sticky, reveal, parallax) reproduced.
- [ ] Carousel/tabs start in the captured state.
- [ ] `prefers-reduced-motion` respected.

---

## Gate 8 — Accessibility

- [ ] Every `<img>` has meaningful `alt` (or `alt=""` if decorative).
- [ ] Landmarks: `header`, `nav`, `main`, `footer` present.
- [ ] Heading order is sequential (single `h1`, no skipped levels).
- [ ] Contrast ≥ 4.5:1 for body text, ≥ 3:1 for large text.
- [ ] Keyboard: all interactive elements reachable, visible focus ring.
- [ ] `lang` attribute set.
- [ ] Form inputs have labels.
- [ ] No `tabindex > 0`.

---

## Gate 9 — Ship

- [ ] All gates above green.
- [ ] `FINAL-REPORT.md` written with the gate table and real numbers.
- [ ] Provenance complete: target, date, browser, what was copied, licences.
- [ ] Trackers/analytics removed.
- [ ] No external requests to the target origin.
- [ ] Production assets optimised **and** the diff re-run to prove losslessness.
- [ ] Known deviations documented with the reason and the measured cost.

---

## Gate table template

```markdown
| Gate | Metric | Target | Actual | Status |
| --- | --- | --- | --- | --- |
| 3 | Baseline mismatch | < 1.5% | | |
| 5 | Desktop mismatch | < 0.05% | | |
| 5 | Tablet mismatch | < 0.1% | | |
| 5 | Mobile mismatch | < 0.2% | | |
| 5 | SSIM | > 0.999 | | |
| 5 | Audit score | > 0.98 | | |
| 5 | Missing assets | 0 | | |
| 5 | Font mismatches | 0 | | |
| 6 | Breakpoints passing | all | | |
| 7 | States reproduced | all | | |
| 8 | A11y issues | 0 | | |
| 9 | External origin requests | 0 | | |
```
