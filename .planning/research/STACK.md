# Stack Research

**Domain:** Client-side HTML proposal builder (no server, no build tools, single HTML file)
**Researched:** 2026-02-28
**Confidence:** HIGH (core libraries verified against npm/jsDelivr/GitHub releases; patterns verified against official docs)

---

## Context: What's Already Solved

The existing proposal file uses this proven stack, which must NOT be changed:

| Library | Embedded Version | Status |
|---------|-----------------|--------|
| html2canvas | 1.4.1 | Keep — latest stable, still maintained (Jan 2025 release) |
| jsPDF | 2.x (UMD inline) | Upgrade candidate — 4.2.0 is current; see upgrade section |
| signature_pad | UMD inline | Upgrade candidate — 5.1.3 is current |

The builder adds a **separate form layer** on top. It does not replace or rewrite the existing PDF generation pipeline.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Vanilla JavaScript (ES2020+) | Native (no version) | Builder logic, template injection, DOM manipulation | No build tool required; template literals are native; all target browsers support ES2020 fully. Avoids framework overhead when the feature is a single-page form that produces a string. |
| HTML5 Template Literals | Native | Dynamic content injection — client name, services, pricing | The correct primitive for this use case. A JS function takes a data object and returns an HTML string. No library needed for simple field substitution. |
| localStorage API | Native | Persist form state between sessions | Built into every browser; 5–10MB storage; survives browser close. Works correctly when file is served over HTTP/HTTPS (the deployed subdomain case). Note: behavior is undefined for `file://` protocol — irrelevant since the file is deployed to a subdomain. |
| Blob + URL.createObjectURL | Native | "Generate proposal" output — produce a downloadable/openable HTML file | Standard approach for generating a new HTML document from JavaScript without a server. `new Blob([htmlString], {type:'text/html'})` → `URL.createObjectURL(blob)` → `window.open(url)`. Zero dependencies. |

### Supporting Libraries (for builder UI only)

| Library | Version | CDN URL | Purpose | When to Use |
|---------|---------|---------|---------|-------------|
| signature_pad | 5.1.3 | `https://cdn.jsdelivr.net/npm/signature_pad@5.1.3/dist/signature_pad.umd.js` | Signature capture in generated proposals | Already in use — keep the same library, upgrade version for security and bug fixes |
| jsPDF | 4.2.0 | `https://cdn.jsdelivr.net/npm/jspdf@4.2.0/dist/jspdf.umd.min.js` | PDF assembly in generated proposals | Already in use — upgrade from 2.x; v4.2.0 fixes three security vulnerabilities including PDF injection in AcroForm. API is backwards compatible for the addImage workflow. |
| html2canvas | 1.4.1 | `https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js` | HTML-to-canvas rendering for PDF pages | Keep at current version — 1.4.1 is still the latest stable release (no breaking changes since project creation) |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Text editor (VS Code, Notepad++) | Direct HTML/JS editing | No build step, no transpilation. Changes are live on save + browser refresh. |
| Browser DevTools | Debugging, localStorage inspection, network | Built-in to Chrome/Firefox. Use Application tab to inspect/clear localStorage during development. |
| Live Server (VS Code extension) | Local HTTP server for `file://` → `http://localhost` | Required during development to avoid localStorage quirks on `file://` protocol. One click in VS Code. Not needed for production (deployed to subdomain). |

---

## How the Builder Works (Architecture Choice)

**Recommended pattern: In-page builder form → Blob-generated output HTML**

The builder lives in a separate HTML file (e.g., `builder.html`). Kyle fills out a form — client name, company, services, pricing, dates. Clicking "Generate Proposal" assembles a complete proposal HTML string using template literals, wraps it in a Blob, and opens it in a new tab via `URL.createObjectURL`. The opened file is a complete, self-contained proposal page with all libraries inlined — functionally identical to the existing VPS proposal file.

```
builder.html (form UI)
  └── onSubmit:
        const proposalHTML = buildProposal(formData)  // template literal function
        const blob = new Blob([proposalHTML], { type: 'text/html' })
        window.open(URL.createObjectURL(blob))
        // user can Save As to get their client-specific .html file
```

This approach requires zero dependencies beyond what's already in the project.

---

## Installation

No npm install. No build step. All libraries are inlined in the HTML output.

For development with localhost (recommended to avoid `file://` quirks):

```bash
# If using VS Code Live Server extension — just click "Go Live" in status bar
# OR use any static server:
npx serve .
# OR
python -m http.server 8080
```

For library updates, fetch UMD builds from jsDelivr and inline them:

```bash
# Download for inlining (Windows PowerShell):
curl -o signature_pad.umd.min.js https://cdn.jsdelivr.net/npm/signature_pad@5.1.3/dist/signature_pad.umd.min.js
curl -o jspdf.umd.min.js https://cdn.jsdelivr.net/npm/jspdf@4.2.0/dist/jspdf.umd.min.js
curl -o html2canvas.min.js https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Template literals (native) | Mustache.js 4.2.0 | Only if logic-less templates with partials become important (e.g., reusable proposal sections). Mustache adds ~9KB and a `Mustache.render(template, data)` API. Unnecessary for a single-document builder with predictable structure. |
| Template literals (native) | Handlebars.js | Only if conditional blocks and helpers are needed at scale. Adds ~90KB. Overkill for this use case. |
| Blob + URL.createObjectURL | Building a second copy of the proposal inline | Avoids the Blob pattern but duplicates the entire proposal HTML in builder.html. Harder to maintain — two places to update design. The Blob approach means the proposal template is single-source. |
| localStorage | URL parameters (query string) | URL params work but are limited in length (~2000 chars) and can't store rich text or multi-service configurations reliably. localStorage handles the full form state trivially. |
| html2canvas 1.4.1 | html2canvas-pro | html2canvas-pro is a fork with better CSS color function support (e.g., `color()`) and minor rendering fixes. The existing proposal uses html2canvas 1.4.1 and produces working output — switching is only warranted if specific CSS rendering bugs are found in the proposal. |
| html2canvas 1.4.1 | html-to-image | html-to-image is well-maintained and has better modern CSS support, but the existing 11-page PDF pipeline is tightly coupled to html2canvas. Switching requires rewriting the entire PDF generation system — high risk, no benefit for this project. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| React / Vue / Svelte | Requires build tools (Vite, Webpack, or CLI). Violates the "no build tools" constraint. The feature scope (one form) doesn't justify framework overhead. | Vanilla JS + template literals |
| Alpine.js | Tempting because it's a CDN-friendly micro-framework, but it adds reactive state complexity to what is essentially a form-to-string operation. | Native DOM + template literals |
| EJS / Pug | Server-side template engines. EJS can run in browser but is designed for Node.js; not the right abstraction for client-side HTML string assembly. | Template literals (native) |
| jsPDF `.html()` method | This method uses html2canvas internally via a different code path and requires DOM access. The existing project uses a custom html2canvas pipeline for precise control over each of the 11 pages — don't replace it with `.html()`. | Existing html2canvas → addImage pipeline |
| IndexedDB | Overkill for proposal form state — structured query complexity, async API, much larger learning surface than localStorage. | localStorage |
| CDN-linked libraries in final proposal output | Proposals need to work offline and without network dependencies. The existing approach of inlining libraries is correct and must be maintained in generated proposals. | Inline all libraries as `<script>` blocks in the generated HTML string |

---

## Stack Patterns by Variant

**If the builder needs to manage multiple saved proposals (different clients):**
- Use localStorage with a keyed object: `proposals[clientId] = { ...formData }`
- List saved proposals in the builder UI for quick regeneration
- No library needed — native JSON.stringify/parse + localStorage

**If proposal sections become modular (different layouts per client):**
- Break the template literal into section functions: `buildCoverPage(data)`, `buildPricingPage(data)`, etc.
- Compose the final HTML string by calling section functions
- Still no library needed — just organized vanilla JS functions

**If the builder needs to be distributed to Kyle without a localhost server:**
- Use the `file://` protocol caveat: localStorage may behave unexpectedly
- Mitigation: encode form state as a base64 query param in the Blob URL (not localStorage)
- Or: deploy builder.html to the same subdomain as the proposals

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| jsPDF 4.2.0 | html2canvas 1.4.1 | The addImage(canvas.toDataURL()) workflow is unchanged in v4. No breaking changes for browser-side use. |
| jsPDF 4.2.0 | signature_pad 5.1.3 | Signature is captured as toDataURL() and passed to addImage() — no version dependency between these libraries. |
| signature_pad 5.1.3 | html2canvas 1.4.1 | Independent libraries, no shared interface. Both work on separate canvas elements. |
| html2canvas 1.4.1 | Modern Chrome/Firefox/Safari/Edge | Stable. Known issue: iOS 17/18 has rendering quirks with jsPDF + html2canvas. Flagged for testing. |

---

## Sources

- https://github.com/parallax/jsPDF/releases — jsPDF v4.2.0 release date (Feb 19, 2025), security fixes confirmed
- https://www.jsdelivr.com/package/npm/jspdf — jsPDF 4.2.0 CDN URL verified
- https://github.com/szimek/signature_pad/releases — signature_pad v5.1.3 confirmed latest
- https://www.jsdelivr.com/package/npm/signature_pad — CDN URL for signature_pad 5.1.3 UMD build verified
- https://github.com/niklasvh/html2canvas/releases — html2canvas 1.4.1 confirmed latest stable (Jan 2025)
- https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage — localStorage file:// protocol caveat (HIGH confidence — MDN official)
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals — Template literals (HIGH confidence — MDN official)
- https://github.com/parallax/jsPDF/issues/3876 — iOS 17/18 compatibility issue documented (MEDIUM confidence)
- WebSearch: "html2canvas abandoned maintenance 2025" — html2canvas-pro fork context (MEDIUM confidence)

---

*Stack research for: Drover Insights Proposal Builder (client-side, no-server, single HTML file)*
*Researched: 2026-02-28*
