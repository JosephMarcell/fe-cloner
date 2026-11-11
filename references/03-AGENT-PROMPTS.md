# 03 — Agent prompts

Copy-paste prompts for running this workflow with a coding agent. Each prompt is
self-contained: it names the inputs, the commands, the artifact to read, and the pass
condition. Agents fail on vague instructions ("make it look the same") and succeed on
measurable ones ("reduce the desktop mismatch from 1.4% to under 0.1%").

---

## 0. The standing brief

Paste this once at the start of a clone session, then add the phase prompt.

```text
You are reproducing a web page's frontend with near-100% visual fidelity.

Ground rules:
- The capture is ground truth. Never guess a value you can read from
  out/<slug>/capture/dom/nodes.json.
- Work in small, measurable steps. After every change, re-run the diff and the audit.
- Fix in this order: fonts -> layout -> paint -> media -> states.
- Never claim a number you did not read from a report file.
- Stop when the numbers pass, not when it "looks right".

Available commands (run from tools/):
  node capture.mjs --url <url> --out <dir>
  node analyze.mjs --capture <dir> --out <dir>
  node rewrite.mjs --capture <dir> --out <dir>
  node serve.mjs --dir <dir> --port 4173
  node diff.mjs --ref <png> --cand <png> --out <dir> --label <name>
  node audit.mjs --capture <dir> --candidate <url> --out <dir>
  node crop.mjs --in <png> --rect "x,y,w,h" --out <png>
  node run.mjs --url <url> --out <dir> [--stage ...]

Gates: desktop mismatch < 0.1%, mobile < 0.3%, audit score > 0.97,
zero missing assets, zero font mismatches.
```

---

## 1. Phase prompt — Recon & capture

```text
Task: capture the target page and prove the capture is trustworthy.

Target: <URL>
Output: out/<slug>/

Steps:
1. Run: node capture.mjs --url <URL> --out out/<slug> --viewports desktop:1440x900,mobile:390x844 --scroll --states --deep
2. Verify the capture quality gate and report PASS/FAIL for each:
   - screenshots/desktop.png exists, is not blank, dimensions are 1440x900
   - css/index.json: every sheet has bytes > 0 and a local file that exists
   - fonts/fonts.json: faces > 0 and every face has status loaded or missing (never error)
   - assets/asset-map.json: count > 0, warnings contain no 4xx for critical images
   - dom/nodes.json: node count plausible for the page type
3. Re-capture to out/<slug>-probe and compare dom/serialize.json html strings.
   Report STABLE or VARIES. If VARIES, list the differing elements and propose a
   neutralisation (block the experiment script, mask the region, or pick a fixed state).
4. Write out/<slug>/PROVENANCE.md: target URL, capture time, browser version,
   authorisation basis, what is dynamic.

Do not proceed to rebuild if any gate fails. Report the exact failing artifact and the
exact command output that proves it.
```

---

## 2. Phase prompt — Analyze & plan

```text
Task: turn the capture into a rebuild plan.

Input: out/<slug>/capture/
Steps:
1. Run: node analyze.mjs --capture out/<slug>/capture --out out/<slug>/analysis
2. Read analysis/inventory.md, analysis/plan.md, analysis/tokens.json,
   analysis/components.md.
3. Write analysis/BRIEF.md containing:
   - the ordered section list with crop rects (from plan.md)
   - the design token table (colours, font scale, spacing scale, radii, shadows)
   - the component list with occurrence counts
   - the interaction surface (hover/focus/keyframes counts)
   - the responsive matrix (breakpoints from breakpoints.json)
   - the hard cases you will scope out (canvas, iframes, third-party embeds)
4. Propose a build order. Justify it by visual area, not by DOM order.

Do not write any site code in this phase.
```

---

## 3. Phase prompt — Baseline

```text
Task: produce the baseline site and verify the pipeline before writing any code.

Steps:
1. node rewrite.mjs --capture out/<slug>/capture --out out/<slug>/site --neutralize-links
2. node serve.mjs --dir out/<slug>/site --port 4173   (background)
3. node run.mjs --url <target> --out out/<slug> --stage verify
4. Read out/<slug>/diff/*/summary.json and out/<slug>/audit/audit.json.
5. Report: mismatch %, audit score, missing assets, font mismatches, unresolved URLs
   from rewrite-report.json, and the top 5 hotspots.

Pass condition: mismatch < 1.5% and audit score > 0.90.
If it fails, fix the capture/rewrite (not the site) and repeat. Do not refactor yet.
```

---

## 4. Phase prompt — Section rebuild loop

Run this once per section, in the order from `analysis/plan.md`.

```text
Task: rebuild section <N> — <label> — to budget.

Reference: out/<slug>/capture/screenshots/desktop-full.png
Section rect: x=<x>, y=<y>, w=<w>, h=<h>
Tokens: out/<slug>/analysis/tokens.json
Computed styles for this subtree: filter out/<slug>/capture/dom/nodes.json by
  path startsWith "<section path> > " or equals "<section path>"

Steps:
1. Crop the reference:
   node crop.mjs --in out/<slug>/capture/screenshots/desktop-full.png --rect "<x,y,w,h>" --out out/<slug>/crops/sec-<N>-ref.png
2. Write the component: markup + scoped CSS using only tokens. No magic hex values.
3. Re-shoot and crop the candidate the same way.
4. Diff the crops:
   node diff.mjs --ref out/<slug>/crops/sec-<N>-ref.png --cand out/<slug>/crops/sec-<N>-cand.png --out out/<slug>/diff/sec-<N> --label sec-<N>
5. Audit and filter hotspots to this section's paths.
6. Fix the top 3 hotspots by area. Re-measure. Repeat until the section mismatch < 0.1%.

Rules:
- Do not touch other sections. If a change affects them, report it.
- Do not move on while this section is above budget.
- If three fixes do not move the number, stop and report the section as blocked with
  the evidence (diff image path, audit hotspot, and what you tried).
```

---

## 5. Phase prompt — Hotspot fixer

The tightest loop. Use it when the page is mostly right and you are grinding the last 1%.

```text
Task: reduce the desktop mismatch from <current>% to <target>%.

Inputs:
- out/<slug>/diff/desktop/summary.json   (how much, where)
- out/<slug>/diff/desktop/regions.json   (which grid cells)
- out/<slug>/audit/audit.json            (which elements, which properties)

Steps:
1. Read the worst 3 regions. For each, read the hotspots whose bbox overlaps that region.
2. For each hotspot, state the hypothesis in one line and the exact CSS change.
   Example: "padding-top 64px -> 56px on .hero because bbox height is +8px and
   nodes.json says 56px."
3. Apply the changes one at a time, re-diffing after each. Keep a table:
   | change | mismatch before | after | delta |
4. Revert any change that does not reduce the number.
5. Stop at < 0.05% or after 3 non-improving iterations; then report.

Never batch more than 3 changes before measuring. Never change a value without a
nodes.json citation.
```

---

## 6. Phase prompt — Responsive pass

```text
Task: make the page match at every captured breakpoint.

Breakpoints: <from breakpoints.json, e.g. 640, 768, 1024, 1280>

Steps:
1. node capture.mjs --url <target> --out out/<slug> --viewports tablet:768x1024,mobile:390x844
2. node run.mjs --url http://127.0.0.1:4173 --out out/<slug> --stage verify
3. For each viewport, read the mismatch and the audit hotspots. Fix mobile-first:
   base styles, then min-width media queries at the captured breakpoints only.
4. Verify no horizontal scrollbar at 320px, no overlapping text, tap targets >= 44px.

Pass: mobile mismatch < 0.3%, tablet < 0.2%.
```

---

## 7. Phase prompt — States and interactions

```text
Task: reproduce interactive states.

Input: out/<slug>/capture/interactions.json
- hoverSelectors, focusSelectors, activeSelectors, keyframes, animatedSelectors

Steps:
1. For each hover selector, read the rule from the captured CSS, reproduce the delta.
2. For each focus selector, reproduce the ring and keep it visible (accessibility).
3. For each keyframe, copy the frames and timing; verify with animations enabled.
4. For scroll effects (sticky headers, reveal-on-scroll), reproduce with
   IntersectionObserver and verify by screenshotting at 0/25/50/75/100% scroll.
5. Screenshot each state and compare against the captured state screenshots in
   capture/screenshots/states/.

Pass: every interactive element has a visible hover and focus state; no layout shift
on hover.
```

---

## 8. Phase prompt — Final QA gate

```text
Task: run the release gate and produce the fidelity report.

Steps:
1. node run.mjs --url http://127.0.0.1:4173 --out out/<slug> --stage verify
2. Fill in the gate table from docs/04-FIDELITY-CHECKLIST.md with real numbers.
3. Verify offline: grep the built site for the target origin; report any remaining
   external requests.
4. Verify assets: every asset SHA-256 in the manifest matches the file in site/.
5. Accessibility pass: alt text, landmarks, heading order, contrast, keyboard, focus.
6. Write out/<slug>/FINAL-REPORT.md with: gate table, mismatch per viewport, audit
   score, asset integrity, known deviations and why, provenance summary.

Do not declare done if any gate is red. Report the red gate and the smallest next step.
```

---

## 9. Multi-agent decomposition

For a large page, one agent doing everything serialises badly. Split by **section**,
with a shared verifier.

```
Lead / orchestrator
├── capture agent        owns out/<slug>/capture  (runs once)
├── analyze agent        owns out/<slug>/analysis (runs once)
├── section agents ×N    each owns one component file + its own CSS partial
│     section-1 (hero), section-2 (features), section-3 (pricing), ...
└── verifier agent       owns out/<slug>/diff + out/<slug>/audit, runs after each merge
```

Rules that make this work:

1. **Disjoint write scopes.** One section agent = one component file + one CSS partial.
   No two agents edit the same file. The Lead owns the page shell and the token file.
2. **The verifier is read-only.** It runs the diff/audit and reports; it never edits.
3. **Fixed contract.** Tokens, the section rects, and the build order are frozen before
   section agents start. Changes go through the Lead.
4. **Merge then measure.** After each section merges, the verifier re-runs the whole-page
   diff. A regression is attributed to the last merge.
5. **One owner for fonts.** Font loading is global; a single agent fixes it before any
   section work starts.

Verifier prompt:

```text
You are the verifier. You never edit code.
After each merge, run:
  node run.mjs --url http://127.0.0.1:4173 --out out/<slug> --stage verify
Report: per-viewport mismatch, audit score, the 5 worst hotspots with their exact
property deltas, and whether the last merge improved or regressed the number.
If the number regressed, name the section that was merged and the hotspots that moved.
```

---

## 10. Prompt anti-patterns

| Don't write | Write instead |
| --- | --- |
| "Make it pixel perfect" | "Reduce desktop mismatch from 1.4% to under 0.1%; read diff/summary.json" |
| "Copy the CSS" | "Read the resolved values from nodes.json for these paths and express them as tokens" |
| "Fix the layout" | "bbox height is +8px on `.hero`; nodes.json says padding-top is 56px, you have 64px" |
| "Check it looks right" | "Run diff.mjs and audit.mjs; paste the numbers" |
| "Do the whole page" | "Do section 3 only; crop rect is 0,1200,1440,900; budget 0.1%" |
| "Improve performance" | "Optimise a copy of the assets, then re-run the diff to prove it is lossless" |

## 11. Context budget

`nodes.json` is large. Never paste it into a prompt. Instead:

```powershell
# one element's resolved styles
node -e "const d=require('./out/slug/capture/dom/nodes.json');const n=d.nodes.find(x=>x.path==='html > body > main > section:nth-of-type(1)');console.log(JSON.stringify(n.styles,null,1))"

# every node under a section, compact
node -e "const d=require('./out/slug/capture/dom/nodes.json');for(const n of d.nodes.filter(x=>x.path.startsWith('html > body > main > section:nth-of-type(1)')))console.log(n.path,'|',n.rect.width+'x'+n.rect.height,'|',n.styles['display'],n.styles['padding-top'],n.styles['background-color'])"
```

Give the agent a *query*, not a dump.
