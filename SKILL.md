---
name: fe-cloner
description: Use when reproducing a web page frontend with near-100% visual fidelity, diagnosing pixel diffs, or auditing computed-style, layout, and asset mismatches
---

# fe-cloner

## Overview

Frontend cloning is a convergence problem, not copy-paste. The capture (rendered DOM plus resolved computed styles plus byte-identical assets) is ground truth; a measured diff loop drives the remaining few percent.

## When to Use

- Reproducing a page frontend at near-100% visual fidelity
- Diagnosing why a rebuild differs: pixels, geometry, styles, fonts, assets
- Setting fidelity gates for a clone task
- Not for: login or payment pages of sites you do not own, defeating CAPTCHA or bot checks, republishing someone else's brand assets or copy (see `references/05-LEGAL-AND-ETHICS.md`)

## Core Pattern

Six layers, fixed in order: structure, layout, paint, type, media, state. Fix fonts first: a substituted font invalidates every textbox measurement. Work top-down, largest area first, one section at a time, re-measuring after every change. Three non-improving fixes on one hotspot means stop, re-capture, re-read ground truth.

## Quick Reference

| Metric | Target | Hard stop |
| --- | --- | --- |
| Desktop mismatch | < 0.05% | < 0.1% |
| Mobile mismatch | < 0.2% | < 0.3% |
| SSIM | > 0.999 | > 0.995 |
| Audit score | > 0.98 | > 0.95 |
| Missing assets, font mismatches | 0 | 0 |

## Implementation

- Full procedure: `references/01-PLAYBOOK.md`; 10-minute path: `references/00-QUICKSTART.md`
- Agent prompts per phase: `references/03-AGENT-PROMPTS.md`; gates: `references/04-FIDELITY-CHECKLIST.md`
- Symptom to fix tables: `references/06-TROUBLESHOOTING.md`; evidence: `references/07-VALIDATION.md`
- Start every clone from `references/CLONE-SPEC.md`; paste `references/agent-brief.md` into a fresh session

## Common Mistakes

- Guessing a CSS value readable from ground truth (the rendered DOM's resolved computed styles)
- Fixing paint before layout (the element moves, then paint is redone)
- Starting from a blank page instead of the captured-DOM baseline
- Comparing screenshots across browsers, viewports, or DPRs
- Chasing sub-0.05% noise (antialiasing, not a bug)
