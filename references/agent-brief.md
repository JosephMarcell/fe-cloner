# Agent brief — <target page>

Paste this into a fresh agent session. Fill in the angle-bracket fields.

---

## Mission

Reproduce `<URL>`'s frontend at `1440x900` (plus `768x1024` and `390x844`) with
near-100% visual fidelity and byte-identical assets, using the playbook in
`docs/01-PLAYBOOK.md` and the toolkit in `tools/`.

## Non-negotiables

1. **The capture is ground truth.** Read resolved values from
   `out/<slug>/capture/dom/nodes.json`. Never guess a CSS value you can read.
2. **Measure after every change.** Run the diff and the audit; paste the numbers.
3. **Fix in order:** fonts → layout → paint → media → states.
4. **Never ship the target's JavaScript bundle.** Re-implement behaviour.
5. **Never claim a number you did not read from a report file.**
6. **Stop at the gates**, not when it "looks right".

## Gates

| Metric | Target | Hard stop |
| --- | --- | --- |
| Desktop mismatch | < 0.05% | < 0.1% |
| Mobile mismatch | < 0.2% | < 0.3% |
| Audit score | > 0.98 | > 0.95 |
| Missing assets | 0 | 0 |
| Font mismatches | 0 | 0 |

Three consecutive non-improving fixes on the same hotspot → stop and report it as
blocked with evidence.

## Environment

```powershell
cd tools
node doctor.mjs          # must be all PASS
```

## Pipeline

```powershell
node capture.mjs --url <URL> --out ../out/<slug> --viewports desktop:1440x900,tablet:768x1024,mobile:390x844 --scroll --states --deep
node analyze.mjs --capture ../out/<slug>/capture --out ../out/<slug>/analysis
node rewrite.mjs --capture ../out/<slug>/capture --out ../out/<slug>/site --neutralize-links
node serve.mjs --dir ../out/<slug>/site --port 4173
node run.mjs --url <URL> --out ../out/<slug> --stage verify --candidate http://127.0.0.1:4173/
```

## Deliverables

| File | Contents |
| --- | --- |
| `out/<slug>/PROVENANCE.md` | authorisation, capture metadata, what was copied |
| `out/<slug>/analysis/BRIEF.md` | section plan, tokens, components, hard cases |
| `out/<slug>/site/` | the rebuild |
| `out/<slug>/REPORT.md` | fidelity matrix + hotspots (generated) |
| `out/<slug>/FINAL-REPORT.md` | gate table with real numbers + known deviations |

## Scope

**In:** `<list>`

**Out:** backend, forms, auth, analytics, `<third-party embeds>`, `<canvas regions>`,
`<A/B variants>`.

## Known constraints

- Fonts: `<licence / substitution decision>`
- Dynamic regions to mask: `<selectors>`
- Target state: `<logged out, no banner, slide 1>`

## Reporting format

Every status update must contain:

```
viewport   mismatch   audit score   top hotspot
desktop    x.xxxx%    0.xxx         <path> — <property> <from> -> <to>
mobile     x.xxxx%    0.xxx         ...
```

No prose without numbers.
