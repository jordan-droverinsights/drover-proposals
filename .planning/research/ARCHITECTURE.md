# Architecture Research

**Domain:** Client-side proposal builder — single-HTML-file static tool
**Researched:** 2026-02-28
**Confidence:** HIGH (patterns grounded in existing codebase + verified web fundamentals)

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   PROPOSAL BUILDER (builder.html)            │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Builder Form                                       │      │
│  │  - Client fields (name, company, date, tagline)    │      │
│  │  - Service selection + pricing toggles             │      │
│  │  - Content overrides (scope, terms, risk notes)    │      │
│  └────────────────┬───────────────────────────────────┘      │
│                   │  serializes to                            │
│  ┌────────────────▼───────────────────────────────────┐      │
│  │  ProposalConfig (JS object / JSON)                  │      │
│  │  { client, services[], prices, content, date }     │      │
│  └────────────────┬───────────────────────────────────┘      │
│                   │  two output paths                         │
│        ┌──────────┴──────────┐                               │
│        ▼                     ▼                               │
│  ┌──────────────┐  ┌──────────────────────────────────┐      │
│  │ Download     │  │ Open in new tab (proposal viewer) │      │
│  │ proposal.html│  │ proposal.html?data=<encoded>      │      │
│  └──────────────┘  └──────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                PROPOSAL VIEWER (proposal.html)               │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Data Loader                                        │      │
│  │  reads ProposalConfig from embedded <script> tag   │      │
│  │  OR from URL param ?data= (base64 JSON)            │      │
│  └────────────────┬───────────────────────────────────┘      │
│                   │                                           │
│  ┌────────────────▼───────────────────────────────────┐      │
│  │  Render Engine (template literals + innerHTML)     │      │
│  │  - Fills client name, company, date                │      │
│  │  - Renders selected services and pricing table     │      │
│  │  - Injects content sections from config            │      │
│  └────────────────┬───────────────────────────────────┘      │
│                   │                                           │
│  ┌────────────────▼───────────────────────────────────┐      │
│  │  Proposal Page (existing dark-theme UI)            │      │
│  │  - Signature pad (signature_pad)                   │      │
│  │  - Service toggle + live pricing calc              │      │
│  │  - PDF generator (html2canvas + jsPDF)             │      │
│  └────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| Builder Form | Collects all client-specific inputs | HTML form, vanilla JS event listeners |
| ProposalConfig | Canonical data object describing one proposal | Plain JS object / JSON schema |
| Data Serializer | Encodes config for transport between files | JSON.stringify + btoa (base64) |
| Render Engine | Hydrates proposal template with config data | Template literals, innerHTML, data- attributes |
| Proposal Viewer | Presents the signed/downloadable proposal | Existing HTML, hydrated at page load |
| PDF Generator | Renders 11 hidden pages → jsPDF output | Existing html2canvas + jsPDF pipeline (unchanged) |

---

## Recommended Project Structure

```
HTML SITE EDIT/
├── builder.html               # New: proposal builder form (separate file)
├── proposal-template.html     # New OR refactored: the viewer/template
│                              # (replaces the hardcoded vps-proposal file)
├── vps-proposal-fixed...html  # Existing file — keep untouched as reference
└── .planning/
    └── research/
```

### Structure Rationale

- **builder.html:** Entirely separate from the viewer. No shared HTML is needed; they communicate only through a data payload. Keeping it separate lets Kyle open it locally without affecting deployed proposals.
- **proposal-template.html:** The existing 2700-line file refactored so all hardcoded VPS content is replaced with data- injection points. The file remains self-contained (libraries still inlined); only the client-specific strings change.
- **Existing file untouched:** The VPS proposal is live and working. The refactored template is built alongside it, not replacing it until verified.

---

## Architectural Patterns

### Pattern 1: Config-Driven Render

**What:** A single JS object (`ProposalConfig`) at the top of the proposal file (or injected into it) contains all client-specific data. All content strings, service arrays, and prices come from this object. The rest of the file is pure layout and library code.

**When to use:** Always — this is the core pattern for the whole system.

**Trade-offs:** Simple to understand and edit in a text editor. No reactivity or virtual DOM needed. The file is still one monolith, but it becomes data-driven rather than content-hardcoded.

**Example:**
```javascript
// At the top of proposal-template.html, inside a <script> tag
const ProposalConfig = {
  client: {
    company: "Vanguard Property Services",
    contact: "Jay Noyola",
    date: "2/27/2026",
    tagline: "VPS is ready to win more multifamily work"
  },
  services: [
    { id: "website", label: "Website Build", price: 3000, type: "onetime", selected: true },
    { id: "logo",    label: "Logo System",   price: 500,  type: "onetime", selected: true }
  ]
};
// Then document.addEventListener('DOMContentLoaded', () => hydrate(ProposalConfig));
```

### Pattern 2: Builder → Downloadable File (Generate-and-Download)

**What:** The builder form collects inputs, builds the ProposalConfig object in memory, injects it into the proposal template HTML string (fetched or inlined), and triggers a download of the resulting `.html` file via a Blob URL.

**When to use:** When Kyle needs to email or deploy a static proposal file per client.

**Trade-offs:** Each proposal is a standalone file — fully self-contained, works offline, no server needed. The cost is that the template HTML must be accessible from the builder (either fetched via `fetch()` from the same origin, or embedded as a string in builder.html).

**Example:**
```javascript
// In builder.html
async function generateProposal(config) {
  const template = await fetch('./proposal-template.html').then(r => r.text());
  const output = template.replace(
    '/* PROPOSAL_CONFIG_PLACEHOLDER */',
    'const ProposalConfig = ' + JSON.stringify(config, null, 2) + ';'
  );
  const blob = new Blob([output], { type: 'text/html' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = config.client.company.replace(/\s+/g, '-') + '-Proposal.html';
  a.click();
}
```

### Pattern 3: URL-Param Pass-Through (Preview Without Download)

**What:** The builder serializes ProposalConfig as base64-encoded JSON, appends it to the proposal-template URL as `?data=<encoded>`, and opens that URL in a new tab. The template reads the param on load and hydrates itself.

**When to use:** For live preview during building, or when Kyle wants to share a link without hosting a separate file.

**Trade-offs:** URL length limit (~2000 chars in some browsers, ~8000 in modern Chrome/Firefox) can be hit if services list or content is long. For short proposals this is reliable. For full proposals, the download-file approach is safer.

**Example:**
```javascript
// In builder.html — preview button
function previewProposal(config) {
  const encoded = btoa(JSON.stringify(config));
  window.open('./proposal-template.html?data=' + encoded, '_blank');
}

// In proposal-template.html — on load
function loadConfig() {
  const param = new URLSearchParams(window.location.search).get('data');
  if (param) return JSON.parse(atob(param));
  return ProposalConfig; // fallback to embedded default
}
```

---

## Data Flow

### Builder → Proposal Flow

```
User fills Builder Form
    │
    ▼
FormData → ProposalConfig object (JS)
    │
    ├── [Preview] → JSON.stringify → btoa → URL param → proposal-template.html?data=
    │                                                          │
    │                                                    atob → JSON.parse → hydrate()
    │
    └── [Download] → inject config into template string → Blob → .html file download
                                                              │
                                                    Client opens file → hydrate()
```

### Proposal Viewer Hydration Flow

```
Page load
    │
    ▼
loadConfig()  ← reads URL param OR uses embedded ProposalConfig
    │
    ▼
hydrate(config)
    │
    ├── Fill text nodes: client name, company, date, tagline
    ├── Render service rows via template literals → innerHTML
    ├── Set data-price attributes for existing pricing calc
    └── Inject any content overrides into section elements
    │
    ▼
Existing proposal UI runs as-is
    └── recalcTotals(), signaturePad init, generatePDF() — all unchanged
```

### Key Data Flows

1. **Config injection:** The ProposalConfig object is the single source of truth. It is written once (by builder or by hand) and read once (by the viewer on load). No live sync needed.
2. **PDF generation data path:** The existing PDF generator reads from the live DOM — it does not need to know about ProposalConfig at all. Hydration happens before generatePDF() is ever called, so the PDF always captures the correct client content.
3. **Service selection:** Selected services in config pre-check the `data-toggle-check` elements during hydration. The existing `recalcTotals()` function still runs unchanged against checked items.

---

## Build Order (Phase Dependencies)

```
Phase 1: Extract config schema
  Define ProposalConfig shape by auditing all hardcoded strings in the existing file.
  No code changes yet — just specification.
  Nothing depends on this being done, but everything else requires it first.
        │
        ▼
Phase 2: Refactor proposal-template.html
  Replace hardcoded VPS strings with hydrate() injection points.
  ProposalConfig embedded at top with VPS defaults (so VPS proposal still works).
  PDF generator and signature pad are NOT touched.
        │
        ▼
Phase 3: Build builder.html
  Form that produces a ProposalConfig object.
  Preview button (URL param method) for instant validation.
  Download button (blob method) for final file output.
        │
        ▼
Phase 4: Mobile + polish
  Builder form mobile layout.
  Proposal viewer mobile improvements.
  Section redesign capability if needed.
```

**Rationale for this order:**
- You cannot build the builder until you know what data it needs to collect (Phase 1 first).
- The template refactor (Phase 2) must be done and verified before the builder output is meaningful (otherwise downloading a file just gets hardcoded VPS content).
- The builder (Phase 3) is pure input UI — it can be built quickly once the template is solid.
- Mobile and polish (Phase 4) is independent of the core data flow and deferred safely.

---

## Key Architectural Decision: Separate Files vs Single File

**Decision: Two separate files — builder.html + proposal-template.html.**

**Rationale:**

The constraint is "single file per proposal" — not "single file for the whole system." The builder is a tool Kyle uses internally; it never goes to the client. The proposal file is what the client receives.

Keeping them separate gives:
- Clean separation of concerns (build tool vs deliverable)
- Builder can be iterated on without touching live proposal files
- Each client still receives a single self-contained `.html` file
- No build tooling needed — both files are plain HTML/JS editable in any text editor

A truly single-file approach (builder embedded in the proposal) would require the client-facing file to include builder UI that clients should never see, and it would make the signature/PDF flow more complex. Reject this approach.

**Template approach:** Vanilla JS template literals with a placeholder replacement pattern. No Handlebars, no Mustache, no dependency additions. The template is a standard HTML file with one `<script>` block containing a `ProposalConfig` constant. The builder replaces that constant with the new client's config via string replacement before download. This is the simplest possible approach that satisfies all constraints.

---

## Anti-Patterns

### Anti-Pattern 1: Editing the HTML Directly Per Client

**What people do:** Copy the entire 2700-line file, do find-and-replace for "VPS"/"Vanguard"/prices.
**Why it's wrong:** Error-prone (missed replacements), no audit trail of what changed, diverging files that can never be updated in sync.
**Do this instead:** All client-specific content lives in ProposalConfig. Generating a new proposal means providing a new config object only.

### Anti-Pattern 2: Using URL Params as the Primary Delivery Mechanism

**What people do:** Builder encodes the config in the URL and shares that URL with the client.
**Why it's wrong:** URL length limits can truncate data silently. The URL looks ugly and unprofessional. If the proposal-template.html URL ever changes, the link breaks.
**Do this instead:** Use URL params only for internal preview. Primary delivery is always a downloaded `.html` file — self-contained, always works, can be emailed or hosted anywhere.

### Anti-Pattern 3: Touching the PDF Generator During Refactor

**What people do:** Refactor the file structure and also modernize the pdf generation code in the same pass.
**Why it's wrong:** The PDF generator is the highest-value, most brittle piece of the system. It is working now. Any change to how pages are constructed risks breaking the 11-page capture pipeline.
**Do this instead:** Treat the PDF generator as a black box. Hydration happens before it runs. The generator reads from the live DOM and doesn't need to know about ProposalConfig. Leave it alone until it breaks.

### Anti-Pattern 4: Framework Adoption

**What people do:** Add Vue, React, or Alpine.js to "make state management easier."
**Why it's wrong:** Violates the no-build-tools constraint. Adds CDN dependencies. Makes the file harder to edit in a text editor. The state management need here is minimal — one config object read once at load time.
**Do this instead:** Vanilla JS hydration function. Ten lines of code. No imports needed.

---

## Integration Points

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| builder.html ↔ proposal-template.html | Blob download (file) or URL param (preview) | No runtime link after delivery |
| ProposalConfig ↔ hydrate() | Direct function call on DOMContentLoaded | Config must be fully loaded before hydrate() runs |
| hydrate() ↔ existing UI (recalcTotals, signaturePad) | DOM attributes and element state | hydrate() sets data-price attrs and checked classes; existing functions run afterward unchanged |
| hydrate() ↔ generatePDF() | None direct — both read from DOM | PDF generator is downstream consumer of hydrated DOM; order of execution guarantees correctness |

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Google Fonts | `<link>` in `<head>` | Already present; builder.html should load same fonts for accurate preview |
| signature_pad | Inlined UMD bundle | No change needed |
| jsPDF | Inlined UMD bundle | No change needed |
| html2canvas | Inlined UMD bundle | No change needed |

---

## Scaling Considerations

This system is a personal tool for one operator (Kyle). "Scaling" means handling more clients and more proposal variants, not user load.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1-10 proposals | Current approach — one file per client, config at top |
| 10-50 proposals | Consider a saved-configs panel in builder.html backed by localStorage |
| 50+ proposals | A simple JSON manifest file listing all proposals, loaded by a dashboard page — still no server needed |

### Scaling Priorities

1. **First bottleneck:** Managing many generated files becomes confusing. Fix with a localStorage-backed "recent proposals" list in the builder.
2. **Second bottleneck:** If service catalog grows large, the builder form becomes unwieldy. Fix with a separate services-catalog.json file loaded by the builder, keeping catalog changes decoupled from builder UI changes.

---

## Sources

- Existing codebase audit: `vps-proposal-fixed__5__1 (1) (1).html` (2700 lines, direct inspection)
- Project constraints: `.planning/PROJECT.md`
- JavaScript template literals pattern: [MDN Web Docs — Template Literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)
- Client-side data passing: [JavaScript.info — localStorage](https://javascript.info/localstorage)
- Zero-dependency templating: [Templating in JavaScript, From Zero Dependencies on Up — Jim Nielsen](https://blog.jim-nielsen.com/2021/javascript-templating/)
- Vanilla JS SPA pattern: [Building Modern SPAs with Vanilla JavaScript — DEV Community](https://dev.to/moseeh_52/building-modern-spas-with-vanilla-javascript-a-beginners-guide-9a3)

---
*Architecture research for: Client-side proposal builder — Drover Insights*
*Researched: 2026-02-28*
