# 06 — Troubleshooting

Symptom → cause → fix. Split into tooling problems and fidelity problems.

---

## Tooling

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot find package 'playwright'` | `npm install` not run in `tools/` | `cd tools; npm install` |
| `Executable doesn't exist at ...chrome.exe` | Playwright browser not downloaded for this version | `npx playwright install chromium` |
| `browserType.launch: Target page, context or browser has been closed` | Missing OS deps or a crashed GPU process | add `--disable-dev-shm-usage`; retry; check `doctor.mjs` |
| Capture hangs forever | Page never reaches `networkidle` (long-poll, websocket) | it falls back after the timeout; lower `--timeout`, or `--block` the offending script |
| `EADDRINUSE` on `serve.mjs` | Port already taken | `--port 4174`, or kill the old server |
| `diff.mjs` reports 100% mismatch | Images are different sizes (candidate is not full-page) | compare like with like: viewport shot vs viewport shot, or `--full` both |
| `diff.mjs` crashes on a PNG | 16-bit or interlaced PNG | re-export as 8-bit RGBA |
| `audit.mjs` finds 0 matched nodes | Candidate DOM uses different paths (framework wrapper) | audit is path-based; rebuild with the same box structure, or rely on the pixel diff |
| `rewrite.mjs` leaves `unresolved` URLs | Asset was never downloaded (JS-injected, or blocked) | add it to the capture, or replace it manually and note it |
| Page blank after rewrite | JS-dependent rendering; the captured DOM needs the app's JS | keep inline scripts, or re-implement the render |
| `ERR_SSL_PROTOCOL_ERROR` | Corporate proxy / TLS interception | `--ignore-https-errors`, or `--proxy` |
| `fs` errors on Windows paths | Backslashes in generated HTML | always emit forward slashes in URLs |

---

## Fidelity

### Fonts (fix these first — they invalidate every other measurement)

| Symptom | Cause | Fix |
| --- | --- | --- |
| All text slightly wider/narrower | Font substituted (fallback rendered) | download the real `woff2`; check `fonts.json` `status` |
| Text looks bolder/thinner | Wrong `font-weight` file (variable font vs static) | capture the variable font; set `font-variation-settings` |
| Letter spacing wrong | `letter-spacing` not copied | read it from `nodes.json` |
| Line height off by 1–2px | `line-height: normal` vs a number | copy the resolved px value, not the keyword |
| Text baseline shifted | Different font metrics / `ascent-override` | use the same font file; avoid synthetic overrides |
| Emoji/icons render differently | System emoji font | use the same icon font or inline SVG |

### Layout

| Symptom | Cause | Fix |
| --- | --- | --- |
| Everything below a section is shifted | One section's height is off | fix top-down; never fix a lower section first |
| 1px gaps between blocks | Fractional heights + `line-height` | match the exact computed height; avoid `height: auto` where the target has a fixed value |
| Content wider than the viewport | Missing `max-width` / `box-sizing` | copy `max-width` and `box-sizing` from `nodes.json` |
| Grid columns uneven | Wrong `grid-template-columns` | copy the resolved value, not the shorthand |
| Element overlaps another | `position` / `z-index` mismatch | compare `position`, `top`, `z-index` |
| Sticky header not sticking | `overflow` on an ancestor | remove the `overflow` on the ancestor |
| Scrollbar appears only in the clone | Content 1px too tall | use `overflow-x: clip` carefully; find the 1px |

### Paint

| Symptom | Cause | Fix |
| --- | --- | --- |
| Colour slightly off | `opacity` or a gradient overlay | check `opacity`, `background-blend-mode`, `mix-blend-mode` |
| Shadow looks wrong | Blur/spread/colour mismatch | copy the full `box-shadow` string |
| Border radius wrong | Per-corner values | copy all four corner values |
| Background image missing | `url()` in a cross-origin sheet | check `rewrite-report.json` `unresolved` |
| Gradient banding | Different colour space | copy the exact gradient string |
| Backdrop blur missing | `backdrop-filter` not supported/copied | copy the property; check browser support |

### Media

| Symptom | Cause | Fix |
| --- | --- | --- |
| Image blurry | Wrong `srcset` candidate (DPR mismatch) | pin DPR 1; copy `currentSrc` |
| Image cropped differently | `object-fit` / `object-position` | copy both |
| Image re-encoded | Optimisation pass | restore original bytes; verify SHA-256 |
| SVG wrong size | `viewBox` / `width`/`height` attributes | copy the element as-is |
| Icons missing | Icon font not loaded | download the font, or convert to inline SVG |
| Video poster missing | `poster` not captured | add it to the asset list |

### Diff noise (not bugs)

| Symptom | Cause | Fix |
| --- | --- | --- |
| 0.01–0.05% mismatch that never moves | Antialiasing / subpixel rendering | stop; below the noise floor |
| Diff changes between runs on the same build | Animations, carousels, video | freeze animations; mask the region |
| A band of diff at the bottom | Different page height | compare full-page to full-page |
| Diff only on text | Different font hinting | capture with the same flags (`--font-render-hinting=none`) |
| Diff only in images of people/products | Rotating/personalised content | mask or neutralise |

---

## Process

| Symptom | Cause | Fix |
| --- | --- | --- |
| Mismatch stuck above 1% after two loops | Fonts or images, not CSS | run the font and asset checks first |
| Each fix breaks another section | Global CSS leaking | scope styles per component; use tokens |
| Numbers move randomly between runs | Non-deterministic environment | re-capture; pin DPR/locale/timezone/motion |
| Agent keeps "improving" without the number moving | No measurement in the loop | force it to paste `summary.json` numbers after each change |
| Audit score high but pixels wrong | Invisible differences only | trust the diff; the audit is a diagnostic, not the goal |
| Diff low but page looks wrong to a human | Wrong state (hover/scroll) | capture the state you actually care about |
| Works on desktop, broken on mobile | Only one viewport captured | capture and diff every breakpoint |
