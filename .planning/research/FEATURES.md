# Feature Research

**Domain:** Client-side proposal builder/generator (no server, single-file constraint)
**Researched:** 2026-02-28
**Confidence:** MEDIUM — industry features verified via multiple sources; single-file/no-server constraint filters are inference from PROJECT.md constraints, not published research

---

## Context Note

This research covers the **builder side only** (the form/UI used to create a new proposal). The viewer side (signature, PDF export, pricing display) already exists. The constraints are hard: no server, no build step, output must be a self-contained HTML file. Every "table stakes" judgment below is made for this specific context — Kyle as the sole user generating proposals for clients — not for a SaaS product with thousands of users.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features that must exist or the builder is useless to Kyle.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Client info fields | Every proposal is different per client; name, company, contact are the minimum personalization points | LOW | Fields: client name, client company, client contact email — these become merge vars in the output HTML |
| Service selection with pricing | The existing proposal already has service selection; the builder must reproduce this on the input side | MEDIUM | Needs checkbox/toggle UI per service, each carrying its price; must match the data model already in the existing HTML |
| Proposal title / project name | Identifies what this proposal is for; appears on cover page and PDF filename | LOW | Single text field; also drives the output filename (e.g., "Drover-ClientName-2026-02.html") |
| Live preview or preview-on-demand | Kyle needs to verify the output looks right before sending; editing blind is error-prone | MEDIUM | Full live preview is complex (iframe rendering); preview-on-demand (button that opens generated HTML in new tab) is LOW complexity and sufficient |
| Generate + download output file | The primary deliverable of the builder: a standalone HTML file ready to deploy or send | LOW | `Blob` + `URL.createObjectURL` + `<a download>` — well-established browser pattern, no server needed |
| Date fields | Proposal date and expiry/validity date appear in every professional proposal | LOW | Two date inputs; formatted into the output template |
| Scope / project description | The problem statement and proposed solution are client-specific and must be editable | MEDIUM | Rich text is overkill; a `<textarea>` per section is sufficient; the builder is for Kyle, not clients |
| Section-level content fields | Cover page intro, scope statement, timeline summary, terms blurb — these vary per client | MEDIUM | One textarea per major section; pre-populated with sensible defaults so Kyle only edits what changes |

### Differentiators (Competitive Advantage)

Features that go beyond minimum; valuable for this specific use case but not blocking.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Auto-save to localStorage | Kyle may build a proposal across multiple sessions; data loss is frustrating | LOW | `localStorage.setItem` on every input event; restore on page load; clear on successful generate |
| Saved proposal templates / presets | If Kyle frequently proposes similar service bundles (e.g., "brand refresh package"), one-click presets reduce repetition | MEDIUM | Store named config objects in localStorage; load button per preset; save-as-preset from current form state |
| Output filename derived from inputs | Automatically suggests "Drover-[ClientName]-[Date].html" as the download name | LOW | Pure JS: `${clientName.replace(/\s+/g,'-')}-${date}.html` — minor UX polish with zero cost |
| Service package subtotal display | Show running total in the builder as Kyle selects services, matching what the client will see | LOW | Mirror of the existing pricing logic; gives Kyle confidence the math is right before generating |
| Inline default text placeholders | Pre-fill section textareas with the boilerplate from the current VPS proposal; Kyle deletes/edits what's client-specific | LOW | Hardcode the current proposal's standard text as `placeholder` or default `value`; massive time saver per new proposal |
| Multiple output variants (e.g., "light/dark" or "compact/full") | Future flexibility if Drover Insights wants different visual treatments per client tier | HIGH | Defer — adds output template management complexity without clear current need |

### Anti-Features (Commonly Requested, Often Problematic)

Features to explicitly NOT build given the single-file, no-server constraint.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| WYSIWYG rich text editor (Quill, TipTap, etc.) | Seems like good UX for editing proposal text | Adds heavy JS dependency (~200KB+), requires inline styles in output, increases build complexity substantially; overkill when Kyle is the only editor and the text structure is fixed | `<textarea>` with character count; formatting is controlled by the output template, not the input |
| Drag-and-drop section reordering | Proposal builder tools feature this; looks professional | The existing proposal has a fixed section order that was designed deliberately; reordering breaks the PDF page layout and the narrative flow | Pre-defined section order; if reordering is ever needed, add CSS `order` overrides in the output template |
| Database / file storage for proposals | "Save all proposals in one place" is appealing | Requires a server or cloud storage; violates the core constraint; adds auth complexity | Use the filesystem: each generated HTML file is the record. Name files clearly. |
| Cloud sync / team sharing | Multiple people editing the same proposal | No team — Kyle is the sole user; adds OAuth, API keys, CORS complexity for zero current benefit | If team grows, revisit. Not now. |
| CRM integration (HubSpot, Salesforce) | Industry standard for proposal tools | Requires API keys, server-side token handling, webhook endpoints; none of this is possible client-side without exposing credentials | Manual: Kyle copies client details from CRM into the form |
| AI-generated proposal copy | Trending feature in 2026 proposal tools | Requires API key handling (security problem in client-side code), non-deterministic output that still needs review, adds latency | Kyle knows his clients better than an LLM does; write good default text instead |
| Version history / revision tracking | Useful in multi-user SaaS tools | localStorage size limits (~5MB), no meaningful diff UI possible without a server; complexity-to-value ratio is very high | File-based versioning: regenerate and save with a new date in the filename |
| E-signature capture in the builder | Seen in tools like PandaDoc that combine builder and viewer | E-signature is already in the generated output (the viewer); the builder does not need it | The generated HTML handles this |
| Payment collection at proposal acceptance | HoneyBook, Better Proposals combine signing and payment | Requires Stripe/PayPal integration; impossible without a server-side webhook to handle payment confirmation securely | If payment collection is needed, add a payment link as a field in the builder that gets embedded in the output |

---

## Feature Dependencies

```
[Client info fields]
    └──required by──> [Generate + download output]
                          └──required by──> [Live preview / preview-on-demand]

[Service selection with pricing]
    └──required by──> [Service package subtotal display]
    └──required by──> [Generate + download output]

[Generate + download output]
    └──enhances──> [Output filename derived from inputs]

[Auto-save to localStorage]
    └──enhances──> [Saved proposal templates / presets]

[Section-level content fields]
    └──enhanced by──> [Inline default text placeholders]
```

### Dependency Notes

- **Generate + download output requires client info + service selection:** You cannot produce a meaningful proposal HTML without at minimum a client name and at least one selected service. These are blockers.
- **Live preview requires generate logic:** The preview is just the generator running and opening the result in a tab. Build generate first, preview is nearly free afterward.
- **Saved presets require auto-save:** Both use localStorage; build the auto-save persistence layer first, then presets reuse it.
- **Inline placeholders are independent:** They can be added at any time; they are content decisions, not code decisions. But they dramatically reduce Kyle's effort per proposal, so include them in v1.

---

## MVP Definition

### Launch With (v1)

Minimum needed for Kyle to stop editing HTML directly.

- [ ] Client info fields (name, company, contact email) — personalizes the cover and header
- [ ] Proposal title + date + expiry date — identifies the proposal and appears on the cover
- [ ] Service selection with per-service pricing checkboxes — maps to the existing service model
- [ ] Service subtotal display — sanity check before generating
- [ ] Section content textareas with pre-filled defaults (cover intro, scope, timeline summary, terms) — editable boilerplate
- [ ] Generate + download output as a standalone HTML file — the core deliverable
- [ ] Preview-on-demand (open in new tab) — verify before sending
- [ ] Auto-save to localStorage — survive accidental tab close

### Add After Validation (v1.x)

- [ ] Output filename derived from client name + date — add after confirming generate works correctly
- [ ] Saved presets — add once Kyle has generated 2-3 proposals and knows which bundles recur

### Future Consideration (v2+)

- [ ] Multiple output visual variants (dark/light, compact/full) — only if Drover Insights needs differentiated client-facing aesthetics
- [ ] Payment link field in the builder — only if Kyle wants to embed a payment URL in generated proposals
- [ ] Mobile-friendly builder layout — Kyle presumably builds proposals at a desk; defer

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Client info fields | HIGH | LOW | P1 |
| Service selection + subtotal | HIGH | MEDIUM | P1 |
| Section textareas with defaults | HIGH | LOW | P1 |
| Generate + download HTML | HIGH | LOW | P1 |
| Preview-on-demand | HIGH | LOW | P1 |
| Auto-save to localStorage | MEDIUM | LOW | P1 |
| Output filename from inputs | LOW | LOW | P2 |
| Saved presets | MEDIUM | MEDIUM | P2 |
| Multiple output variants | LOW | HIGH | P3 |
| Payment link field | LOW | LOW | P2 |

**Priority key:**
- P1: Must have for launch — builder is unusable without these
- P2: Should have, add after core is validated
- P3: Nice to have, future consideration

---

## Competitor Feature Analysis

| Feature | PandaDoc / Proposify (SaaS) | HoneyBook (Agency-focused) | Our Approach |
|---------|---------------------------|---------------------------|--------------|
| Proposal creation UI | Web app editor with drag-and-drop blocks | Guided form with template sections | HTML form with textareas; no drag-and-drop needed |
| Service/pricing selection | Interactive pricing table builder | Packages with upsells | Checkbox list mirroring the existing proposal's service model |
| Client data input | CRM integration auto-populates fields | Manual input or contact import | Manual input — acceptable for low volume |
| Draft saving | Cloud-synced across sessions | Cloud-synced | localStorage — sufficient for single user |
| Output format | Hosted web page + PDF | Hosted web page + PDF | Downloadable standalone HTML file |
| Preview | Live preview in editor | Send test to self | Open generated file in new tab |
| Templates / reuse | Team content library | Saved templates | localStorage presets — sufficient for single user |

**Key difference:** Every competitor stores data server-side and delivers proposals as hosted URLs. This builder generates a portable file instead. That is a feature for this use case, not a limitation — proposals work offline and need no hosting other than Kyle's existing subdomain.

---

## Sources

- [Better Proposals — pricing table features](https://help.betterproposals.io/en/articles/5203752-pricing-table-basics) — MEDIUM confidence (official product docs)
- [Better Proposals — marketing agency template sections](https://betterproposals.io/proposal-templates/marketing-agency-proposal-template) — MEDIUM confidence
- [Fresh Proposals — builder UI redesign patterns](https://www.freshproposals.com/proposal-software-ui-redesign/) — MEDIUM confidence
- [Qwilr — branding proposal template 2026](https://qwilr.com/blog/how-to-write-a-branding-proposal/) — MEDIUM confidence
- [Guideflow — best proposal software 2026](https://www.guideflow.com/blog/best-proposal-software) — LOW confidence (review aggregator)
- [Oneflow — best proposal management software 2026](https://oneflow.com/blog/best-proposal-management-software/) — LOW confidence (review aggregator)
- [Smashing Magazine — pricing table UX](https://www.smashingmagazine.com/2022/07/designing-better-pricing-page/) — MEDIUM confidence (design reference)
- [Raymond Camden — localStorage draft saving](https://www.raymondcamden.com/2011/09/11/Using-Local-Storage-to-keep-a-draft-of-form-data) — HIGH confidence (well-established browser API pattern, date not relevant)

---
*Feature research for: Client-side proposal builder/generator (Drover Insights)*
*Researched: 2026-02-28*
