# 00 — Quickstart

From a URL to a measured, offline, near-identical copy in about ten minutes.

## 1. Install and verify (2 min)

```powershell
cd tools
npm install
node doctor.mjs
```

`doctor.mjs` checks Node, the dependencies, that Chromium launches, network access, that
`out/` is writable, and that every script exists. All checks must pass.

## 2. Capture the target (1–2 min)

```powershell
node capture.mjs --url https://example.com --out ../out/example-com `
  --viewports desktop:1440x900,tablet:768x1024,mobile:390x844 `
  --scroll --states --deep
```

Writes `out/example-com/capture/`:

```
manifest.json          everything the pipeline knows about this capture
dom/index.html         the post-hydration DOM — your rebuild baseline
dom/nodes.json         per-element computed styles + geometry — ground truth
css/sheet-*.css        every stylesheet, cross-origin ones re-fetched
assets/                every image/font/media file, SHA-256 recorded
screenshots/           the reference images
breakpoints.json       the responsive test matrix
interactions.json      hover/focus/animation inventory
```

**Check the capture before trusting it** — [Gate 1](04-FIDELITY-CHECKLIST.md#gate-1--capture):

- `screenshots/desktop.png` is not blank
- `css/index.json` sheets all have bytes > 0
- `fonts.json` has faces and no `status: "error"`
- `warnings` has no critical 4xx

## 3. Analyze (10 s)

```powershell
node analyze.mjs --capture ../out/example-com/capture --out ../out/example-com/analysis
```

Read `analysis/plan.md` (build order, largest area first) and `analysis/tokens.json`
(the real colours, font scale, spacing scale).

## 4. Build the baseline (10 s)

```powershell
node rewrite.mjs --capture ../out/example-com/capture --out ../out/example-com/site --neutralize-links --strip-scripts
```

This is a self-contained offline copy of the captured page. It is already ~95–99%
pixel-identical. **Never start from a blank page.**

`--strip-scripts` matters: the captured DOM is already hydrated, so re-running the page's
own bundle can mutate it (a live site's script duplicated the entire body in our
integration test, turning a 0.0000% baseline into 2.19%). A visual baseline must be static.

## 5. Measure (30 s)

```powershell
node serve.mjs --dir ../out/example-com/site --port 4173
```

In a second terminal:

```powershell
node run.mjs --url https://example.com --out ../out/example-com --stage verify --candidate http://127.0.0.1:4173/
```

Read `out/example-com/REPORT.md`. You now have:

- a fidelity matrix (mismatch % and SSIM per viewport)
- an audit score with ranked hotspots
- `diff/desktop/diff.png` and `heatmap.png` — look at these
- `audit/report.md` — the exact CSS deltas to fix

## 6. Iterate

```text
fix top 3 hotspots → re-shoot → re-diff → re-audit → repeat
```

For each section, crop the reference and diff the crop — a 0.4% whole-page change is
invisible, a 4% section change is obvious:

```powershell
node crop.mjs --in ../out/example-com/capture/screenshots/desktop-full.png --rect "0,0,1440,900" --out ../out/example-com/crops/hero-ref.png
node crop.mjs --in ../out/example-com/shots/desktop-full.png --rect "0,0,1440,900" --out ../out/example-com/crops/hero-cand.png
node diff.mjs --ref ../out/example-com/crops/hero-ref.png --cand ../out/example-com/crops/hero-cand.png --out ../out/example-com/diff/hero --label hero
```

## 7. One command instead of seven

```powershell
node run.mjs --url https://example.com --out ../out/example-com
```

Runs capture → analyze → rewrite → serve → shoot → verify → `REPORT.md`.

## Done when

| Metric | Target |
| --- | --- |
| Desktop pixel mismatch | < 0.05% |
| Mobile pixel mismatch | < 0.2% |
| Audit score | > 0.98 |
| Missing assets | 0 |
| Font mismatches | 0 |

Then run [Gate 9](04-FIDELITY-CHECKLIST.md#gate-9--ship) and write
`FINAL-REPORT.md`.

## Where to go next

- Full procedure: [`01-PLAYBOOK.md`](01-PLAYBOOK.md)
- Prompts to hand an agent: [`03-AGENT-PROMPTS.md`](03-AGENT-PROMPTS.md)
- Something will not converge: [`06-TROUBLESHOOTING.md`](06-TROUBLESHOOTING.md)
- Before you clone anything: [`05-LEGAL-AND-ETHICS.md`](05-LEGAL-AND-ETHICS.md)
