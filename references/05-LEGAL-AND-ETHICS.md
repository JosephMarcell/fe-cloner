# 05 — Legal and Ethical Constraints

> **Not legal advice.** This is an engineering guardrail document, written by engineers, for engineers. It records the constraints that most often bite when you clone a page, and the workflow gates that keep a project defensible. Copyright, contract, trademark, database, and unfair-competition law are jurisdiction-specific and fact-specific. If the clone is going anywhere public, or anywhere near a brand you do not own, get a lawyer in the relevant jurisdiction.

The single rule that prevents ~95% of problems:

> **Only clone a page you own, have written permission to clone, or are capturing for private study/analysis — and never republish someone else's brand assets or content as your own.**

Everything below is elaboration on that rule.

---

## 1. Allowed vs not allowed

| Activity | Status | Why / notes |
|---|---|---|
| Cloning **your own** site to migrate it to a new stack | Allowed | You hold the copyright. Still check third-party embeds (fonts, stock images, analytics) — those licenses may not transfer. |
| Cloning a page as a **private reference** for study (layout, spacing, technique) | Generally allowed | Reading and learning from published work is normal engineering practice. Keep it local, keep it unindexed, do not ship it. |
| Cloning a page **under written permission / a client engagement** | Allowed | Get the permission in writing and record it in the provenance file. Scope it to what was granted. |
| Cloning a page to run an **accessibility or performance audit** of a site you do not own | Generally allowed (read-only, non-published) | Auditing is analysis, not republication. Do not publish the mirror itself; publish the findings. |
| Reproducing **your own** page pixel-for-pixel as a regression baseline | Allowed | This is the primary legitimate use of the whole toolchain. |
| Republishing a cloned page **on the public web** | Not allowed | Copyright in the HTML/CSS/JS, images, and copy. Also trademark exposure if brand marks travel with it. |
| Shipping a clone as a **template or theme for sale** | Not allowed | The visual design and assets are the site owner's. Rebuild the *technique*, not the *asset*. |
| Cloning a competitor's page to **pass it off** as your product | Not allowed | Passing off / unfair competition / trademark infringement. This is the phishing-adjacent end of the spectrum. |
| Cloning a login page, bank page, or checkout flow of a site you do not own | **Never** | Phishing. Criminal exposure in most jurisdictions regardless of intent. Stop immediately. |
| Bulk-mirroring a whole site behind a **paywall or login** | Not allowed | Breach of contract (ToS) even where the CFAA may not apply — see §5. |
| Scraping behind **Cloudflare bot protection** by defeating the challenge | Not allowed in practice | Circumventing an explicit technical access control. Also see §5. |
| Reusing a site's **web fonts** (.woff2) in your rebuild | Usually not allowed | Fonts are separately licensed; a site's self-hosted webfont is not a license grant to you. See §3. |
| Reusing a site's **logo / wordmark / product screenshots** | Not allowed | Trademark. Even "just for the demo" is risky once it is public. |
| Keeping a clone in a **private, non-deployed repo** for reference | Generally allowed | The practical safe harbour. Keep it unlisted and out of any published artifact. |

---

## 2. Preflight checklist (run before any capture)

Answer all of these in writing in the capture manifest. Any "no" is a stop condition.

- [ ] **1. Authority.** Do I own this site, have written permission, or is this private study/analysis? Which one, exactly?
- [ ] **2. Destination.** Where will the output live? Private repo / local disk = fine. Public URL / npm / template marketplace = requires ownership or a license.
- [ ] **3. robots.txt.** Fetched and read? `https://<host>/robots.txt`. Note the applicable `User-agent` group and `Disallow` prefixes. Per RFC 9309 the rules are **not a form of access authorization**, and they bind *crawlers* — but ignoring them is a strong signal of bad faith if a dispute arises. ([RFC 9309 §1](https://www.rfc-editor.org/rfc/rfc9309.html))
- [ ] **4. Terms of service.** Is there a scraping / automated-access / reproduction clause? Quote it into the manifest. A ToS breach is actionable even when scraping itself is lawful (see §5).
- [ ] **5. Rate and load.** Am I fetching a handful of pages or crawling? Am I willing to put this behind a 1 req/sec limiter and a real User-Agent with contact info?
- [ ] **6. Bot protection.** Is the site behind Cloudflare Bot Fight Mode / Super Bot Fight Mode / Bot Management, or a CAPTCHA? If yes: **stop and ask for permission.** Do not solve the challenge. ([Cloudflare bot solutions](https://developers.cloudflare.com/bots/))
- [ ] **7. Assets with separate licenses.** Fonts, stock photos, icon sets, video, embedded maps, third-party widgets. For each: is it mine, is it open-licensed, or is it licensed to *them* only?
- [ ] **8. Personal data.** Does the page render names, emails, avatars, addresses, or user-generated content? If so, cloning it stores personal data — GDPR/CCPA exposure. Strip or mask it.
- [ ] **9. Brand marks.** Does the clone contain a logo, wordmark, or trade dress that will be visible in the output? Plan the replacement now.
- [ ] **10. Attribution.** If I am publishing anything derived (a case study, a diff report, a blog post), how do I credit the source?

### Stop conditions (hard stops, no exceptions)

1. The page is a **credential-collection surface** (login, password reset, payment form) you do not own.
2. The site's ToS explicitly prohibits automated access or reproduction **and** you do not own it.
3. You would need to **defeat a bot challenge, CAPTCHA, or paywall** to proceed.
4. The output would be **publicly deployed** and contains another party's brand assets or copy.
5. You cannot identify who owns the content.

---

## 3. Font and asset licensing

Fonts are the most common accidental license violation in a pixel-perfect clone, because they are the single biggest pixel-diff cause (see [02-RESEARCH-NOTES.md](./02-RESEARCH-NOTES.md) §4) and therefore the thing you are most tempted to copy verbatim.

| Font source | Can you reuse the .woff2 in your rebuild? | Notes |
|---|---|---|
| **SIL Open Font License (OFL) 1.1** | Yes, with conditions | Free/libre/open license designed for fonts. Allows use on websites, embedding in apps and software "subject to certain conditions"; modifications may require renaming if the font declares **Reserved Font Names**. ([openfontlicense.org](https://openfontlicense.org/)) |
| **Google Fonts (OFL/Apache families)** | Yes | Serve from your own origin or from Google; keep the license file. Verify the individual family's license — most are OFL, some are Apache-2.0. |
| **Adobe Fonts (Typekit)** | **No** — not transferable | Licensed to the subscriber's Creative Cloud account and bound to specific domains/sites. Web font use is governed by a separate webfont license from the desktop license. ([Adobe Fonts licensing FAQ](https://helpx.adobe.com/fonts/web/font-licensing/font-licensing.html)) |
| **Commercial foundry webfont (self-hosted)** | **No** | A webfont license is granted to the site owner for their domain. Copying the file is redistribution. |
| **System fonts** (`-apple-system`, `Segoe UI`, `Helvetica Neue`) | N/A — do not copy | Reference the stack by name; rendering will differ per OS, which is itself a pixel-diff cause. |
| **Icon fonts** (Font Awesome Pro, custom) | Depends | Free tiers are usually OFL/MIT/CC-BY; Pro tiers are per-seat. Check before copying the font file. |

**Practical rule:** if you cannot name the license of a font file, do not ship it. Substitute a metrically compatible open font and record the substitution in the manifest as a known pixel-diff source. That is an honest, defensible outcome; a copied commercial webfont is not.

Other asset classes:

- **Images / photography.** Copyright by default. Stock photos are licensed to the site owner, not to you. Placeholders or your own assets only.
- **SVG icons and illustrations.** Often the most-copied asset and the most clearly copyrightable. Check for an explicit license (many icon sets are MIT/CC0 — those are fine).
- **Video / poster frames.** Copyright plus, frequently, a separate CDN or stock license.
- **Third-party embeds** (YouTube, Vimeo, Stripe, Intercom, Google Maps). You are not copying these — you are loading someone else's service. Keep the embed, do not re-host the content, and expect the embed to be non-deterministic (see 02 §5).
- **Logos and wordmarks.** Trademark, not copyright — a different and in some ways stricter regime. Replace before publishing.
- **Page copy (text).** Copyright. For a private reference capture it is fine; for anything published, rewrite.

---

## 4. robots.txt and ToS handling

### robots.txt

- Located at `https://<host>/robots.txt`, UTF-8, `text/plain`. ([RFC 9309 §2.3](https://www.rfc-editor.org/rfc/rfc9309.html))
- Matching is **longest-match wins**, case-sensitive on the path, with `*` and `$` wildcards. A more specific `Allow` overrides a broader `Disallow`.
- Error handling: a **4xx** on robots.txt means "unavailable" → the crawler *may* access. A **5xx** means "unreachable" → the crawler *must* assume complete disallow. Do not treat a 5xx as permission.
- `Crawl-delay` is **not** part of RFC 9309. Absence of a directive is not permission to hammer the origin.
- **robots.txt is not an access-control mechanism** — the RFC says so explicitly, and it also says listing paths makes them discoverable. It is a signal of intent, and courts and counterparties read it that way.

**Our rule:** obey robots.txt for any automated multi-page capture, even though our tooling is a browser and not a crawler. If a path is disallowed and you do not own the site, do not fetch it.

### Terms of service

The important asymmetry, from the case law:

- **hiQ Labs v. LinkedIn** (9th Cir. 2019, reaffirmed on remand 2022) held that scraping *publicly available* data does not "exceed authorized access" under the CFAA. ([Wikipedia summary](https://en.wikipedia.org/wiki/HiQ_Labs_v._LinkedIn))
- **But** in November 2022 the N.D. Cal. held that hiQ had **breached LinkedIn's User Agreement**, and the case settled. ([same](https://en.wikipedia.org/wiki/HiQ_Labs_v._LinkedIn))

The lesson for a cloner: **the CFAA is not your problem; the contract is.** Scraping being lawful does not make breaching a ToS lawful, and it certainly does not grant you a copyright license in what you copied. Two independent gates, both must pass.

**Our rule:** if the ToS prohibits automated access or reproduction and you do not own the site, stop. Record the clause you read in the manifest so the decision is auditable.

### Bot protection

Cloudflare's stack (Bot Fight Mode, Super Bot Fight Mode, Bot Management) scores requests using signals including JA3/JA4 TLS fingerprints and JavaScript detections. ([Cloudflare bot solutions docs](https://developers.cloudflare.com/bots/)) A headless browser clone will be scored and challenged.

**Our rule:** a challenge is a "no". Do not tune the browser to pass it. Ask for allowlisting, use an official API, or pick a different reference site.

---

## 5. Recording provenance

Every capture run writes a `manifest.json` next to the artifacts. Minimum fields:

```json
{
  "capture_id": "2026-11-11T01-20-00Z_example-com_home",
  "source_url": "https://example.com/",
  "captured_at": "2026-11-11T01:20:00Z",
  "authority": {
    "basis": "own_site | written_permission | private_study",
    "evidence": "docs/permission-2026-10-01.pdf | n/a",
    "reviewed_by": "name"
  },
  "robots": { "url": "https://example.com/robots.txt", "sha256": "...", "verdict": "allowed" },
  "tos": { "url": "https://example.com/terms", "relevant_clause": "§4.2 no automated access", "verdict": "n/a - own site" },
  "bot_protection": "none | cloudflare_bot_fight | captcha",
  "viewport": { "width": 1440, "height": 900, "dpr": 1 },
  "user_agent": "Mozilla/5.0 ... Chrome/... (clone-research; contact@example.com)",
  "assets": [
    { "url": "https://example.com/f/Inter.woff2", "license": "OFL-1.1", "kept": true },
    { "url": "https://example.com/img/hero.jpg", "license": "unknown", "kept": false, "replaced_with": "placeholder" }
  ],
  "substitutions": [
    { "what": "font Inter", "reason": "license unverified" },
    { "what": "logo.svg", "reason": "trademark" }
  ],
  "not_for_publication": true
}
```

Why bother: it is the difference between "we did an engineering exercise" and "we cannot explain what we did". It also makes the clone *re-runnable* and the diff *interpretable* — every substitution in the manifest is a known, expected pixel-diff source.

---

## 6. If in doubt — decision tree

```
Do I own this site, or hold written permission?
├─ YES ──> Proceed. Still verify third-party asset licenses (fonts, stock, embeds).
└─ NO
   └─ Will the output be publicly deployed, sold, or shipped to a client?
      ├─ YES ──> STOP. You need a license or ownership. Not a technical question.
      └─ NO (private, local, unindexed study)
         └─ Does the page collect credentials or payment data?
            ├─ YES ──> STOP. Never. Phishing risk regardless of intent.
            └─ NO
               └─ Does robots.txt disallow the path, or does the ToS prohibit
                  automated access / reproduction?
                  ├─ YES ──> STOP. Pick a different reference or ask permission.
                  └─ NO
                     └─ Is it behind a bot challenge, CAPTCHA, or paywall?
                        ├─ YES ──> STOP. Do not defeat the control.
                        └─ NO
                           └─ Capture. Keep it local. Replace brand marks and
                              unlicensed assets. Write the manifest. Do not publish.
```

Three questions that resolve almost everything:

1. **Whose assets are these?** If you cannot answer, you cannot ship them.
2. **Would the owner be upset if they saw this?** If yes, do not do it.
3. **Am I copying the technique or the artifact?** Technique is learnable and reusable. The artifact is theirs.

---

## 7. Attribution

If you publish anything *derived* — a blog post, a diff report, a talk, a case study — attribute:

- The source URL, capture date, and that it was captured for analysis.
- Any tooling used, if the tool requires it (SingleFile is AGPL — see its license note; monolith is public domain).
- A clear statement that no affiliation or endorsement is implied.

Do not attribute in a way that implies a license you do not have. "Layout studied from example.com" is honest. "Built with example.com's design system" implies a relationship that does not exist.

---

## Sources

- [RFC 9309 — Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309.html)
- [SIL Open Font License](https://openfontlicense.org/)
- [Adobe Fonts — font licensing FAQs](https://helpx.adobe.com/fonts/web/font-licensing/font-licensing.html)
- [Cloudflare bot solutions](https://developers.cloudflare.com/bots/)
- [hiQ Labs v. LinkedIn (Wikipedia)](https://en.wikipedia.org/wiki/HiQ_Labs_v._LinkedIn)
- [CSSOM spec — CSSStyleSheet.cssRules](https://drafts.csswg.org/cssom/#dom-cssstylesheet-cssrules)
- [SingleFile (AGPL) — license section](https://github.com/gildas-lormeau/SingleFile)
- [monolith — public domain](https://github.com/Y2Z/monolith)
- [HTTrack](https://www.httrack.com/)
