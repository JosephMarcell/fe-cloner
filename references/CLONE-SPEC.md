# Clone spec — <page name>

Fill this in before capturing. It is the single source of truth for scope, and it is
what stops a clone from drifting into "while I'm here I'll also…".

---

## 1. Target

| Field | Value |
| --- | --- |
| URL | |
| Page name | |
| Site / product | |
| Captured at | |
| Browser build | |
| Primary viewport | `1440x900` |
| DPR | `1` |
| Locale / timezone | `en-US` / `UTC` |
| Colour scheme | light |
| Reduced motion | reduce |

## 2. Authorisation

| Field | Value |
| --- | --- |
| Basis | own site / written permission / private study |
| Reference / proof | |
| May republish markup? | yes / no |
| May republish images? | yes / no |
| May republish fonts? | yes / no — licence: |
| May reuse brand name/logo? | no (default) |

## 3. Scope

**In scope**

- [ ] Desktop layout at 1440×900
- [ ] Tablet at 768×1024
- [ ] Mobile at 390×844
- [ ] Hover / focus states
- [ ] Scroll effects
- [ ] Animations
- [ ] Dark mode
- [ ] …other

**Out of scope (explicitly)**

- Backend / forms / auth
- Analytics and trackers
- Third-party embeds: …
- Canvas / WebGL regions: …
- A/B variants: …
- Locale variants: …

## 4. Target state

| Question | Answer |
| --- | --- |
| Logged in or out? | |
| Cookie banner shown? | |
| Carousel slide | |
| Selected tab | |
| Scroll position at capture | top |
| Dynamic regions to mask | |

## 5. Rebuild decisions

| Decision | Value | Why |
| --- | --- | --- |
| Strategy | baseline-first / blank-page / framework | |
| Framework | none / Astro / Next / other | |
| CSS approach | plain CSS + tokens / Tailwind / CSS modules | |
| Component boundaries | from `analysis/components.md` | |
| JS interactions | re-implement | never ship the target bundle |
| Assets | copy bytes, verify SHA-256 | |

## 6. Fidelity gates

| Metric | Target |
| --- | --- |
| Desktop mismatch | < 0.05% |
| Tablet mismatch | < 0.1% |
| Mobile mismatch | < 0.2% |
| SSIM | > 0.999 |
| Audit score | > 0.98 |
| Missing assets | 0 |
| Font mismatches | 0 |

## 7. Known deviations

| # | Deviation | Reason | Measured cost |
| --- | --- | --- | --- |
| 1 | | | |
| 2 | | | |

## 8. Provenance

- Capture directory: `out/<slug>/capture/`
- Capture manifest SHA-256: `…`
- Copied verbatim: assets, fonts, inline SVG, markup (until refactored)
- Rewritten: URLs, links, forms
- Re-implemented: JavaScript behaviour
- Removed: analytics, trackers, third-party widgets

## 9. Sign-off

| Gate | Status | Evidence |
| --- | --- | --- |
| Capture | | `capture/report.md` |
| Baseline | | `REPORT.md` |
| Rebuild | | `diff/` |
| Responsive | | `diff/*/summary.json` |
| States | | `capture/screenshots/states/` |
| A11y | | audit notes |
| Ship | | `FINAL-REPORT.md` |
