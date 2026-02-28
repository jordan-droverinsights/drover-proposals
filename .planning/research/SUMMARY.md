# Project Research Summary

**Project:** Drover Insights Proposal Builder
**Domain:** Client-side HTML proposal builder — single-file, no-server tool
**Researched:** 2026-02-28
**Confidence:** HIGH

## Executive Summary

This project adds a proposal builder on top of an existing, working client-side HTML proposal file. The builder is a separate internal tool (builder.html) that Kyle uses to fill out a form and generate a fully self-contained, client-specific proposal HTML file — no server, no build step, no framework. The existing file (a 2700-line single-HTML proposal for VPS) already contains the complete PDF generation pipeline using html2canvas + jsPDF and a signature capture using signature_pad. The builder's only job is to collect client-specific data and produce a new instance of that file with the correct content injected. The recommended approach is config-driven rendering: a ProposalConfig JS object at the top of the proposal template holds all client-specific data, and a hydrate() function fills the DOM at page load time. The builder populates this object, injects it into the template string, and triggers a Blob download.

The recommended stack is entirely native: vanilla JavaScript with template literals for HTML generation, localStorage for draft persistence, and the Blob + URL.createObjectURL pattern for file output. No new libraries are needed. The three existing libraries (html2canvas 1.4.1, jsPDF upgraded to 4.2.0, signature_pad upgraded to 5.1.3) are sufficient. The architecture separates concerns cleanly — builder.html is the internal form tool, proposal-template.html is the refactored viewer/deliverable — and the two files communicate only through a data payload. This is a well-understood pattern with no exotic dependencies.

The most important risks are all known and preventable. Template literal injection via unescaped user input is the first task to solve before any template generation code is written. The PDF generation pipeline (11 hidden pages, html2canvas, jsPDF) is the most brittle part of the system and must be treated as a black box — hydration happens before it runs, and the generator code itself should not be touched. iOS Safari canvas limits and Google Fonts CORS failures are real risks for any phase that modifies PDF rendering or adds new proposal sections. All five critical pitfalls have clear, low-cost prevention steps documented in PITFALLS.md.

---

## Key Findings

### Recommended Stack

The existing file already uses the right libraries. The builder requires no new dependencies. All new logic uses native browser APIs: ES2020 template literals for HTML generation, localStorage for form persistence, and Blob + URL.createObjectURL for the download mechanism. The libraries used in generated proposals (html2canvas, jsPDF, signature_pad) should be upgraded to their current versions (jsPDF 4.2.0 fixes three security vulnerabilities; signature_pad 5.1.3 is the current stable release). html2canvas stays at 1.4.1 — it is still the latest stable release and the PDF pipeline is tightly coupled to it. All libraries are inlined in the generated proposal HTML so proposals work offline and without CDN dependencies.

**Core technologies:**
- Vanilla JS (ES2020) + template literals: builder logic and HTML generation — no build tools, no framework, works in any text editor
- localStorage API: form draft persistence — native, 5-10MB, survives tab close; deploy to subdomain (not file://) to avoid protocol quirks
- Blob + URL.createObjectURL: proposal file output — zero dependencies, standard browser API, produces a downloadable self-contained HTML file
- jsPDF 4.2.0: upgrade from 2.x in generated proposals — backwards-compatible addImage API, security fixes included
- signature_pad 5.1.3: upgrade in generated proposals — same UMD build pattern, drop-in replacement
- html2canvas 1.4.1: keep as-is — latest stable, PDF pipeline depends on it, switching is high risk with no benefit

### Expected Features

The builder has a well-defined, narrow feature scope. Kyle is the sole user. The feature set is small enough that the entire P1 list can ship as a single working version.

**Must have (table stakes):**
- Client info fields (name, company, contact email) — every proposal is different; these are the minimum personalization points
- Proposal title + date + expiry date — appears on cover; drives output filename
- Service selection with per-service pricing checkboxes — must match the existing proposal's service data model
- Service subtotal display — real-time sanity check before generating
- Section content textareas with pre-filled defaults (cover intro, scope, timeline, terms) — boilerplate Kyle can edit rather than write from scratch
- Generate + download as standalone HTML file — the core deliverable
- Preview-on-demand (open in new tab) — verify before sending to client
- Auto-save to localStorage — protect against accidental tab close

**Should have (competitive):**
- Output filename derived from client name + date — zero-cost UX polish (add after core generate is confirmed working)
- Saved presets / named templates — add after Kyle has generated 2-3 real proposals and knows which service bundles recur

**Defer (v2+):**
- Multiple output visual variants (dark/light, compact/full) — no clear current need; adds template management complexity
- Mobile-friendly builder layout — Kyle builds proposals at a desk; not a current constraint
- Payment link field embedded in proposals — only if Kyle wants to link to Stripe/PayPal from within a proposal

### Architecture Approach

The system consists of two separate HTML files that communicate through a data payload. builder.html is the internal form tool — it collects ProposalConfig data, previews via URL param (base64 JSON), and downloads via Blob injection into the template string. proposal-template.html is the refactored viewer — it reads ProposalConfig from an embedded script block (or URL param for preview), calls hydrate() on DOMContentLoaded, and then runs all existing interactive logic (recalcTotals, signaturePad, generatePDF) unchanged. The existing VPS proposal file is kept untouched as a reference until the refactored template is verified.

**Major components:**
1. Builder Form (builder.html) — collects all client-specific inputs, serializes to ProposalConfig JS object, exposes preview and download actions
2. ProposalConfig — canonical data object: client info, services array with prices, section content strings, dates; single source of truth for one proposal
3. hydrate() function (proposal-template.html) — reads ProposalConfig, fills text nodes, renders service rows, sets data-price attributes; runs once at page load before any existing logic
4. Proposal Viewer / PDF Generator (proposal-template.html) — the existing interactive proposal UI and 11-page PDF pipeline; unchanged, downstream consumer of hydrated DOM
5. Data Serializer — JSON.stringify + btoa for URL param preview; string injection of ProposalConfig into template for download

### Critical Pitfalls

1. **Template literal XSS and HTML corruption** — Any user-supplied value embedded raw in a template literal via innerHTML can corrupt layout (special chars) or execute arbitrary JS. Write an escapeHTML() helper as the very first function in any template generation code, before any other logic. Test with `O'Brien & Associates <VIP>` as client name before shipping.

2. **PDF content cut off at page boundaries with dynamic content** — The existing 11 hidden pages assume fixed content height. Dynamic content (variable service counts, long descriptions) can overflow the 1056px page boundary; overflow is silently clipped by html2canvas. Set overflow: hidden on all hidden page containers during development, add a scrollHeight validator that warns in console when any page exceeds 1056px, and enforce content limits in the builder form.

3. **Google Fonts CORS failure in PDF** — html2canvas re-fetches resources including fonts when capturing pages. If fonts have not fully loaded or the CORS header is missing, the PDF silently uses system fallback fonts. Always await document.fonts.ready before any html2canvas call, and set useCORS: true on every call. Long-term, inline fonts as base64 data URIs to eliminate the network dependency.

4. **iOS Safari canvas memory limit kills PDF generation silently** — An 11-page PDF at scale 2 creates ~38MP of canvas data; iOS Safari enforces a 3-5MP per-canvas limit. The existing per-page rendering approach keeps each canvas under the limit, but any change to rendering approach or scale must be validated on actual iOS hardware. Never increase scale beyond 2. Add a mobile warning before PDF generation if canvas dimensions would exceed 4MP.

5. **File size explosion from scale and quality settings** — Scale is a squared multiplier. scale: 3 produces 9x the pixels of scale: 1. For 11 pages this can produce 50-100MB of in-memory canvas data before jsPDF compresses. Keep scale at 2 as a hard ceiling; use quality: 0.92 (not 1.0) in jsPDF addImage — imperceptible visual difference, ~40% smaller file size.

---

## Implications for Roadmap

Based on combined research, the architecture research already provides a validated phase ordering. The suggested structure below maps directly to the build-order dependency chain in ARCHITECTURE.md with pitfall prevention integrated.

### Phase 1: Config Schema Definition
**Rationale:** You cannot build a builder or refactor a template until you know what data it needs to carry. This is a specification exercise, not a code change — audit all hardcoded strings in the existing VPS proposal file and define the ProposalConfig schema. Everything else depends on this being correct and complete first.
**Delivers:** ProposalConfig schema document; list of all injection points in the existing file; confirmed service data model
**Addresses:** The "hardcode all content" anti-pattern — by defining the schema first, every subsequent phase works against it
**Avoids:** Schema drift between builder form and template hydration
**Research flag:** Standard — no external research needed; this is a codebase audit task

### Phase 2: Proposal Template Refactor
**Rationale:** The builder output is meaningless if the template still has hardcoded VPS content. The template refactor must be done and verified before the builder can produce a meaningful output. The existing VPS proposal file stays untouched; the refactored template is a parallel file built alongside it.
**Delivers:** proposal-template.html — a fully data-driven version of the proposal that renders correctly from any ProposalConfig; VPS proposal still works via embedded defaults
**Addresses:** All template stakes features that appear in the generated output (client name, services, pricing, dates, section content)
**Implements:** Config-driven render pattern; hydrate() function; data- attribute injection for existing recalcTotals and signaturePad
**Avoids:** Anti-pattern 1 (editing HTML directly per client); Anti-pattern 3 (touching PDF generator during refactor — treat it as a black box)
**Pitfalls to prevent:** Google Fonts CORS (add document.fonts.ready + useCORS: true); dark background rendering (set explicit backgroundColor in html2canvas); PDF page overflow (add scrollHeight validator, test with max realistic content)
**Research flag:** Standard — all patterns are documented in ARCHITECTURE.md with code examples

### Phase 3: Builder Form + Generate/Download
**Rationale:** With the template solid, the builder is pure input UI. This is the highest-value phase for Kyle — it eliminates the need to edit HTML directly. Preview-on-demand is nearly free once generate logic exists (it's the same code path opening a Blob URL in a new tab instead of triggering a download).
**Delivers:** builder.html — a complete form that produces a client-specific, downloadable proposal HTML file; preview button; auto-save to localStorage
**Addresses:** All P1 features from FEATURES.md: client info fields, service selection + subtotal, section textareas with defaults, generate + download, preview-on-demand, auto-save
**Uses:** Vanilla JS template literals, Blob + URL.createObjectURL, localStorage API
**Avoids:** Anti-pattern 4 (no framework adoption); CDN-linked libraries in output (all libraries inlined in generated file)
**Pitfalls to prevent:** XSS/encoding (escapeHTML() is the FIRST function written in this phase, before any template generation code); form validation (no broken proposals from empty required fields); localStorage on file:// (develop and test on HTTP via Live Server)
**Research flag:** Standard — well-documented patterns, no novel integrations

### Phase 4: Polish and Saved Presets
**Rationale:** After Kyle has used the builder for 2-3 real proposals, he will know which service bundles recur and where the form is friction-heavy. Presets and output filename polish are added after core validation, not before.
**Delivers:** Saved named presets in localStorage; output filename derived from client name + date; any UX friction fixes discovered during real use
**Addresses:** P2 features from FEATURES.md: saved presets, output filename from inputs
**Research flag:** Standard — localStorage preset pattern is a straightforward extension of Phase 3 auto-save

### Phase 5: Mobile and iOS Validation
**Rationale:** Mobile layout and iOS PDF generation are deferred until core functionality is verified. iOS Safari canvas limits are a real risk but only manifest on mobile — and Kyle builds proposals at a desk. This phase validates the generated proposals work on client devices (phones) and improves mobile display of the proposal viewer.
**Delivers:** Verified PDF generation on iOS Safari; mobile-responsive proposal viewer layout; mobile warning UI if PDF generation would exceed canvas limits
**Avoids:** Shipping a proposal that clients cannot view on phones
**Pitfalls to prevent:** iOS canvas memory limit (test on actual iPhone before calling done); PDF page design at 816px (force hidden pages to always render at 816px regardless of viewport)
**Research flag:** Needs validation — iOS Safari behavior should be tested on real hardware in this phase; no emulator substitution

### Phase Ordering Rationale

- Phase 1 before everything: the schema is the interface contract between builder and template; defining it first prevents rework
- Phase 2 before Phase 3: a working, data-driven template is required for the builder output to be meaningful; cannot validate the builder without it
- Phase 3 is the MVP: after Phase 3, Kyle can stop editing HTML directly — the builder works end-to-end
- Phase 4 after real use: presets are only useful once Kyle knows what recurs; adding them before first use is premature optimization
- Phase 5 last: iOS and mobile are correctness concerns for the client-facing experience; the builder itself is desktop-only

### Research Flags

Phases needing deeper research during planning:
- **Phase 5 (Mobile/iOS):** iOS Safari canvas behavior should be validated on actual hardware — emulators do not reproduce the memory limit accurately. If the existing per-page rendering approach is maintained, risk is LOW, but any change to scale or rendering strategy requires device testing.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Config Schema):** Codebase audit — no external research needed
- **Phase 2 (Template Refactor):** All patterns documented in ARCHITECTURE.md with working code examples
- **Phase 3 (Builder Form):** Blob, localStorage, template literals — all well-documented native browser APIs
- **Phase 4 (Polish/Presets):** Direct extension of Phase 3 patterns; no novel integrations

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Core libraries verified against npm/jsDelivr/GitHub releases; native APIs verified against MDN official docs; no speculative recommendations |
| Features | MEDIUM | Table stakes verified via competitor analysis and domain research; filtering to the no-server constraint is inference from PROJECT.md, not published research; scope is narrow enough that this uncertainty is low-risk |
| Architecture | HIGH | Grounded in direct inspection of the existing 2700-line codebase; all patterns are web fundamentals with MDN-level documentation; build order is dependency-driven, not arbitrary |
| Pitfalls | HIGH | All five critical pitfalls verified via official html2canvas/jsPDF GitHub issues, MDN, and OWASP; prevention strategies are concrete and code-level, not advisory |

**Overall confidence:** HIGH

### Gaps to Address

- **iOS Safari PDF generation on actual hardware:** The per-page rendering approach should be safe, but this cannot be verified without a real iPhone. Flag this explicitly in Phase 5 planning and require hardware testing before marking the phase complete.
- **ProposalConfig schema completeness:** The schema shape is documented in ARCHITECTURE.md with a working example, but the full audit of all hardcoded strings in the 2700-line file must happen in Phase 1. If there are injection points that were missed, Phase 2 will need to loop back — this is the most likely source of rework.
- **Service data model match:** The builder's service selection UI must exactly match the data attributes and structure already used by the existing recalcTotals() function. This needs to be confirmed by reading the existing file's service toggle implementation in Phase 1, not assumed.
- **HTML file size baseline:** The existing file's inlined library size is not documented. Before adding anything, measure the current file size — if it is already approaching 4-5MB, the constraint on adding new libraries is real, not theoretical.

---

## Sources

### Primary (HIGH confidence)
- Existing codebase: `vps-proposal-fixed__5__1 (1) (1).html` — direct inspection by architecture researcher
- https://github.com/parallax/jsPDF/releases — jsPDF v4.2.0 release, security fixes confirmed
- https://github.com/szimek/signature_pad/releases — signature_pad v5.1.3 confirmed latest
- https://github.com/niklasvh/html2canvas/releases — html2canvas 1.4.1 confirmed latest stable
- https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage — localStorage file:// caveat
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals — template literals
- https://html2canvas.hertzen.com/faq.html — canvas memory limits, known limitations
- https://github.com/niklasvh/html2canvas/issues/3169 — iOS canvas area limit
- https://github.com/niklasvh/html2canvas/issues/1544 — CORS font loading failure
- https://github.com/parallax/jsPDF/issues/3485 — content cut off at page boundaries
- https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS — XSS prevention
- https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html — DOM XSS

### Secondary (MEDIUM confidence)
- https://blog.jim-nielsen.com/2021/javascript-templating/ — zero-dependency templating pattern
- https://betterproposals.io/proposal-templates/marketing-agency-proposal-template — competitor feature reference
- https://qwilr.com/blog/how-to-write-a-branding-proposal/ — proposal content structure reference
- https://www.smashingmagazine.com/2022/07/designing-better-pricing-page/ — pricing table UX
- https://github.com/parallax/jsPDF/issues/3876 — iOS 17/18 compatibility

### Tertiary (LOW confidence)
- https://www.guideflow.com/blog/best-proposal-software — review aggregator, competitor landscape
- https://oneflow.com/blog/best-proposal-management-software/ — review aggregator, competitor landscape

---
*Research completed: 2026-02-28*
*Ready for roadmap: yes*
