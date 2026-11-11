# fe-cloner

Agent skill for reproducing a web page frontend with near-100% visual fidelity.

## What it is

A guidance-only skill (no scripts executed): the capture (rendered DOM plus resolved
computed styles plus byte-identical assets) is ground truth, and a measured diff
loop drives the remaining few percent.

## Contents

- `SKILL.md` — entry point: when to use, core pattern, fidelity gates, references
- `references/` — full playbook, quickstart, agent prompts, fidelity checklist,
  legal and ethics, troubleshooting, validation record, clone spec, agent brief

## Source

Distilled from `dsh-web-cloner-research` (playbook + toolkit docs).

## Fidelity gates

| Metric | Target | Hard stop |
| --- | --- | --- |
| Desktop mismatch | < 0.05% | < 0.1% |
| Mobile mismatch | < 0.2% | < 0.3% |
| SSIM | > 0.999 | > 0.995 |
| Audit score | > 0.98 | > 0.95 |
| Missing assets, font mismatches | 0 | 0 |

## Use

Only clone pages you own, have written permission for, or capture for private
study. Never republish someone else's brand assets, fonts, or copy. See
`references/05-LEGAL-AND-ETHICS.md`.
