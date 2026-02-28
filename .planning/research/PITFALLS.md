# Pitfalls Research

**Domain:** Client-side HTML proposal builder — single-file, no build step, html2canvas + jsPDF
**Researched:** 2026-02-28
**Confidence:** HIGH (critical pitfalls verified via official docs + GitHub issues; architecture pitfalls MEDIUM from multiple community sources)

---

## Critical Pitfalls

### Pitfall 1: iOS Safari Canvas Memory Limit Kills PDF Generation Silently

**What goes wrong:**
On iOS devices, Safari enforces a hard canvas area limit: 3 megapixels on devices with less than 256 MB RAM, 5 megapixels on devices with more. An 11-page PDF rendered at 816×1056px at 2x scale creates a canvas area of roughly 1,729×11,616px = ~20 million pixels — far over the limit. The result is a blank or partially rendered PDF with no useful error message. The failure is silent or shows a cryptic console warning: "Total canvas memory use exceeds the maximum limit."

**Why it happens:**
The existing implementation renders all 11 hidden pages individually (816×1056 at scale 2), which keeps each canvas under ~3.5MP — but if mobile layout shift causes any page to expand beyond that, or if the approach changes to render a single tall canvas, iOS will silently fail. Developers test on desktop and miss this entirely.

**How to avoid:**
- Keep the per-page canvas approach (render each PDF page separately, not as one tall canvas)
- Cap html2canvas scale at 2 for mobile, not 3 or 4
- Explicitly set `windowWidth` and `windowHeight` in html2canvas options to match the fixed-width page container, not the mobile viewport
- Add a mobile detection guard that shows a warning before PDF generation if canvas area would exceed 4MP

**Warning signs:**
- PDF download button produces an empty PDF on iPhone/iPad
- Console shows "Total canvas memory use exceeds the maximum limit"
- Works on desktop Chrome/Firefox, fails silently on iOS Safari

**Phase to address:** Mobile experience improvements phase (before shipping mobile layout fixes, validate PDF generation works end-to-end on iOS)

**Confidence:** HIGH — documented in html2canvas official FAQ, multiple GitHub issues (#1350, #3169), and Apple Developer Forums

---

### Pitfall 2: Template Literal HTML Generation Without Escaping — XSS and Rendering Corruption

**What goes wrong:**
The proposal builder will generate HTML by populating a template with user-supplied values (client name, pricing, service descriptions). Using raw template literals like `` `<h1>${clientName}</h1>` `` inserted via `innerHTML` is dangerous and fragile. A client name containing `&`, `<`, `>`, or `"` will corrupt the HTML structure. A value containing `</script>` or `<img onerror=...>` executes arbitrary JavaScript.

**Why it happens:**
Template literals are natural for HTML generation. Developers write the template, it works with normal input, and the XSS/encoding issue is invisible until a real edge case appears. Single-file apps have no framework's auto-escaping to protect them.

**How to avoid:**
- Create a minimal `escapeHTML()` helper at the top of the file before any template generation code:
  ```javascript
  function escapeHTML(str) {
    return String(str)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');
  }
  ```
- Call `escapeHTML()` on every user-supplied value before embedding in a template literal
- For numeric fields (pricing), validate as numbers before formatting — never embed raw numeric strings without type checking
- Never use `eval()` or `Function()` constructor with user data

**Warning signs:**
- Client names with `&` render as literal `&` in the output
- Apostrophes in client names break string attribute values
- Any form field that accepts free text being used directly in innerHTML

**Phase to address:** Proposal builder phase — the escaping helper must be the very first function written before any template generation logic

**Confidence:** HIGH — verified via MDN XSS documentation, OWASP DOM XSS prevention cheat sheet, multiple security sources

---

### Pitfall 3: Google Fonts Fail to Render in PDF Due to CORS

**What goes wrong:**
The proposal uses IBM Plex Mono, IBM Plex Sans, and Bebas Neue loaded from Google Fonts. When html2canvas captures pages for PDF generation, it re-fetches resources including fonts. Google Fonts CDN serves fonts cross-origin. If the font fetch is blocked (CORS policy, or the font hasn't fully loaded when html2canvas runs), the PDF renders with fallback system fonts — destroying the premium typographic design that defines the proposal's brand.

**Why it happens:**
html2canvas clones the DOM and attempts to load referenced resources. Web fonts loaded via `@import` or `<link>` in the page may not automatically be available in the cloned context. The failure mode is silent — the PDF generates but with wrong fonts.

**How to avoid:**
- Set `useCORS: true` in every html2canvas call
- Add `crossOrigin: 'anonymous'` to any `<link>` tags loading Google Fonts
- Use `document.fonts.ready` promise before triggering pdf generation to ensure all fonts are loaded:
  ```javascript
  await document.fonts.ready;
  const canvas = await html2canvas(element, { useCORS: true });
  ```
- Long-term: inline the font files as base64 data URIs in the `<style>` block to eliminate the CORS dependency entirely (acceptable for 3 fonts totaling ~200KB)

**Warning signs:**
- PDF uses a different font than what's shown on screen
- Monospace text appears in a generic system mono font instead of IBM Plex Mono
- Heading text appears in a sans-serif instead of Bebas Neue

**Phase to address:** Any phase that modifies PDF generation or adds new template sections with text

**Confidence:** HIGH — documented in html2canvas GitHub issues (#1544, #3020), Google Fonts troubleshooting docs, multiple community reports

---

### Pitfall 4: PDF Content Cut Off at Page Boundaries When Dynamic Content Changes Height

**What goes wrong:**
The current implementation uses 11 fixed-height hidden pages (816×1056px each). Adding a proposal builder means content length becomes variable — a client with 5 services has different content height than one with 2. If any hidden page overflows its container, the content at the bottom gets clipped in the PDF with no warning.

**Why it happens:**
The existing approach assumes fixed content. Once the builder generates content dynamically, a service description longer than expected, or a pricing table with more rows, will push content below the 1056px page boundary. html2canvas captures exactly what fits in the element — overflow is silently ignored.

**How to avoid:**
- Set `overflow: hidden` on each hidden page container so overflow is at least visually obvious during development
- Add a development-mode height validator that checks all hidden pages after generation and warns in console if any exceeds its target height:
  ```javascript
  document.querySelectorAll('.pdf-page').forEach((page, i) => {
    if (page.scrollHeight > 1056) {
      console.warn(`PDF page ${i+1} overflows: ${page.scrollHeight}px`);
    }
  });
  ```
- Design template sections with conservative maximum content assumptions
- Provide character/item limits in the builder form that correspond to known safe content volumes per page

**Warning signs:**
- Sections appear complete on screen but the bottom portion is missing in the PDF
- Services list or pricing table truncates at an odd point
- Proposal looks correct on screen; PDF reviewer reports missing content

**Phase to address:** Proposal builder phase — validate all template variants at form design time, not after

**Confidence:** HIGH — documented in jsPDF GitHub issues (#3485, #3438), multiple community discussions

---

### Pitfall 5: File Size Explosion from Scale + Inline Libraries + Multiple Pages

**What goes wrong:**
The file is already ~2700 lines with three large libraries inlined. Increasing html2canvas `scale` from 2 to 3 for quality quadruples canvas memory and substantially increases JPEG output size per page. Combined with 11 pages, the resulting in-memory PDF can exceed 50–100MB before jsPDF compresses it. On mobile, this triggers tab crashes or out-of-memory errors. On desktop, the PDF download stalls.

**Why it happens:**
Scale is a multiplier, not a quality clamp. `scale: 2` doubles linear dimensions, producing 4x the pixels. `scale: 3` produces 9x the pixels. For an 816×1056px page: scale 2 = ~3.5MP per page × 11 = 38.5MP total canvas data. Scale 3 = ~7.8MP per page × 11 = 86MP. Memory consumption is proportional.

**How to avoid:**
- Keep `scale: 2` as the ceiling for multi-page PDF generation
- Use `quality: 0.92` (not 1.0) in jsPDF `addImage` — the difference is imperceptible in print but cuts JPEG file size by ~40%
- Do not increase scale without testing memory usage on a mid-range Android device
- The HTML file itself (with inlined libraries) approaching 5MB+ is a genuine problem for mobile users on slow connections — document the current baseline size and monitor it

**Warning signs:**
- PDF file size exceeds 15MB for an 11-page proposal
- Browser tab freezes or crashes during PDF generation on mobile
- `html2canvas` call never resolves (no rejection, no resolution — tab ran out of memory)

**Phase to address:** Any phase that modifies PDF rendering quality settings

**Confidence:** HIGH — confirmed via html2canvas scale documentation, jsPDF GitHub issues (#2766, #762), and official FAQ

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcode all content strings in the template | Fast to build first proposal | Every new client requires manually diffing and editing 2700-line file | Never — the whole point of the builder is to eliminate this |
| Use `innerHTML` with raw template string (no escaping) | Faster development | First client with `&` in their name breaks layout; XSS vector exists | Never |
| Copy-paste the entire HTML to create a second proposal variant | Works immediately | Two files diverge; fixing a PDF bug means fixing it twice | Only for one-off emergency; must be planned away |
| Increase html2canvas scale to 3 for better print quality | Sharper PDF text | 9x pixel data, mobile crashes, 50MB+ PDFs | Never for multi-page generation |
| Load fonts from Google Fonts (network) instead of inlining | Smaller HTML file | CORS failures in PDF, fonts fail offline, dependency on Google CDN | Only if offline use is explicitly out of scope AND CORS is verified |
| Store proposal data in `window` globals between builder form and proposal template | Simple, no module system needed | State is implicit, hard to debug, breaks if page order changes | Acceptable for MVP if namespaced under a single object (e.g., `window.ProposalData`) |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| html2canvas + Google Fonts | Call html2canvas before fonts finish loading | `await document.fonts.ready` before any html2canvas call |
| html2canvas + dark background | Dark background renders correctly on screen but PDF comes out black/inverted | Explicitly set `backgroundColor: '#0a0a0a'` (or the exact page background) in html2canvas options; never rely on inherited CSS background |
| jsPDF `addImage` + canvas | Pass canvas element directly; assume auto-sizing | Always calculate `imgWidth` and `imgHeight` explicitly from canvas dimensions and PDF page dimensions; never use defaults |
| Signature pad + PDF | Call `signaturePad.toDataURL()` without checking if empty | Check `signaturePad.isEmpty()` first; empty signature renders as blank in PDF without error |
| Template generation + `innerHTML` | Set innerHTML of the proposal body then immediately call html2canvas | Allow one event loop tick (`await new Promise(r => setTimeout(r, 100))`) after innerHTML update for images and fonts to settle |
| Builder form + localStorage | Save form state to localStorage for draft persistence | localStorage is domain-scoped; works fine on subdomains but test on the actual deployment subdomain, not `file://` — localStorage is partitioned on `file://` protocol |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Rendering all 11 PDF pages in series with html2canvas | PDF generation takes 15–30 seconds | This is expected and acceptable; do NOT try to parallelize — multiple simultaneous html2canvas calls on overlapping DOM compete for resources | Immediate on mobile if parallelized |
| Inline libraries totaling 3–5MB in a single HTML file | Page load time on mobile 3G is 10–20 seconds | Document library sizes; only add new libraries if strictly necessary; consider lazy-loading non-critical libraries after page paint | When file exceeds ~4MB on slow connections |
| Re-generating the full proposal preview on every form keystroke | UI becomes unresponsive; CPU fan spins up | Debounce preview regeneration to 500ms after last keystroke; use targeted DOM updates for fields that don't affect layout | Immediately with any live-preview implementation |
| Using `scale: 2` + `quality: 1.0` in jsPDF for all 11 pages | PDF download takes 30+ seconds; file is 20–40MB | Use `quality: 0.92` — imperceptible visual difference, substantial size reduction | With any scale >= 2 and quality = 1.0 |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Embedding user input directly in `innerHTML` via template literal | XSS — attacker-controlled content in a client name or service description executes JavaScript | Always pass user values through `escapeHTML()` before inserting into any HTML string |
| Logging form field values to console during development and forgetting to remove | Client data (company names, pricing, contact info) visible in DevTools console for anyone who opens it | Remove all console.log statements referencing form data before deployment; use a `DEBUG` flag |
| Storing sensitive proposal data in URL hash or query params for "share link" feature | Business-sensitive pricing and terms visible in browser history, server logs, and referrer headers | If sharing is needed, use a server-side share token (out of scope), or accept that proposals are shared as downloaded HTML files |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No feedback during PDF generation (10–20 second wait) | Kyle thinks the button is broken; clicks it again; triggers double-generation; browser crashes | Show a spinner or progress indicator immediately on click; disable the button during generation |
| Signature required before PDF download is enforced | Client skips signing, downloads blank signature PDF, sends it; Kyle has unsigned agreement | Show clear warning if signature is empty when download is clicked; do not silently embed a blank signature box |
| Mobile layout uses desktop-designed fixed widths | Proposal text overflows or becomes illegible on phones; Kyle's client gives up reading | Test every template section on 375px viewport width specifically; use `max-width` not fixed `width` for content containers |
| Builder form with no validation produces broken proposals | Empty client name, zero pricing, missing sections generate a malformed PDF | Validate all required fields before allowing proposal generation; show inline errors, not alerts |
| PDF page design assumes 816×1056px but mobile viewport is narrower | Hidden pages used for PDF render reflowed at mobile viewport width | Force hidden PDF pages to always render at 816px width regardless of viewport; use `width: 816px; position: absolute; left: -9999px` |

---

## "Looks Done But Isn't" Checklist

- [ ] **PDF Generation on iOS Safari:** Works on desktop Chrome — verify on an actual iPhone before calling done; canvas limits are browser-and-RAM-specific
- [ ] **Fonts in PDF:** Proposal looks correct on screen — open the PDF in a separate PDF viewer and verify Bebas Neue and IBM Plex Mono appear correctly, not system fallbacks
- [ ] **Page overflow:** All template sections render within page height — check `scrollHeight` of all hidden pages in DevTools after populating with maximum-length realistic test content
- [ ] **Special characters in client data:** Test with a client name like `O'Brien & Associates <VIP>` — verify it renders correctly in the proposal and doesn't corrupt HTML or PDF
- [ ] **Signature in PDF:** Signature appears on screen — verify the captured image actually appears in the downloaded PDF, at the correct position, with legible quality
- [ ] **Empty state handling:** Builder form submitted with no signature, empty service list, or zero pricing — verify graceful error, not a broken PDF
- [ ] **File loads offline:** HTML file sent to client via email as attachment — verify it opens and renders correctly when opened from local filesystem (no CDN dependencies that fail offline)

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| iOS PDF generation fails (canvas limit) | MEDIUM | Switch to per-page rendering if not already; add scale detection that drops to `scale: 1.5` on iOS |
| Client name corrupts HTML template | LOW | Add `escapeHTML()` helper and run through all template insertion points — one-pass fix |
| Fonts missing from PDF | LOW-MEDIUM | Add `await document.fonts.ready` before html2canvas calls; if still failing, inline font files as base64 |
| File size too large for email attachment (>10MB) | MEDIUM | Reduce html2canvas quality from 1.0 to 0.85–0.92; if still too large, compress inlined library code |
| Template content overflows PDF page | HIGH | Requires redesigning template sections with overflow budgets; may need to split sections across pages differently — do not defer this discovery to after a client uses it |
| Two proposal files diverge after copy-paste | HIGH | Must reconcile manually or choose one as canonical; prevents this by building the generator correctly in the first place |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| iOS canvas limit on PDF generation | Mobile improvements phase | Test PDF download on iPhone Safari before marking phase complete |
| Template literal XSS / encoding | Proposal builder phase (first task) | Test with `O'Brien & Associates <Test>` as client name; PDF renders correctly |
| Google Fonts CORS in PDF | Proposal builder phase (PDF generation integration) | Open generated PDF in standalone viewer; verify custom fonts render |
| Dynamic content overflowing PDF page height | Proposal builder phase (template design) | Populate all fields with maximum realistic content; measure hidden page scrollHeight in DevTools |
| File size explosion from scale settings | Any phase touching PDF generation | Measure output PDF file size; keep under 10MB for a standard 11-page proposal |
| Missing validation in builder form | Proposal builder phase (form implementation) | Submit form with empty required fields; verify no broken PDF is produced |
| Dark background rendering black in PDF | Any phase touching html2canvas config | Generate PDF; verify background color matches design (#0a0a0a or equivalent), not black or white |

---

## Sources

- [html2canvas FAQ — official known limitations](https://html2canvas.hertzen.com/faq.html) — HIGH confidence
- [html2canvas GitHub Issue #3169 — canvas area exceeds limit](https://github.com/niklasvh/html2canvas/issues/3169) — HIGH confidence
- [html2canvas GitHub Issue #1544 — CORS font loading](https://github.com/niklasvh/html2canvas/issues/1544) — HIGH confidence
- [jsPDF GitHub Issue #3485 — content cut off multipage](https://github.com/parallax/jsPDF/issues/3485) — HIGH confidence
- [jsPDF GitHub Issue #3876 — iOS 17/18 failures](https://github.com/parallax/jsPDF/issues/3876) — HIGH confidence
- [PQINA: Canvas area exceeds maximum limit](https://pqina.nl/blog/canvas-area-exceeds-the-maximum-limit) — HIGH confidence
- [PQINA: Total canvas memory use exceeds maximum limit (iOS Safari)](https://pqina.nl/blog/total-canvas-memory-use-exceeds-the-maximum-limit) — HIGH confidence
- [MDN: Cross-site scripting (XSS) prevention](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS) — HIGH confidence
- [OWASP: DOM-based XSS prevention cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html) — HIGH confidence
- [Joyfill: Creating PDFs from HTML+CSS — what actually works](https://joyfill.io/blog/creating-pdfs-from-html-css-in-javascript-what-actually-works) — MEDIUM confidence
- [html2canvas scale implementation guide](https://copyprogramming.com/howto/html2canvas-scale-issue) — MEDIUM confidence
- [Galah Cyber: JS Template Literal Injection](https://www.galahcyber.com.au/practical-cybersecurity-insights/understanding-and-preventing-js-template-literal-injection-attacks/) — MEDIUM confidence

---
*Pitfalls research for: client-side HTML proposal builder (single-file, no build step)*
*Researched: 2026-02-28*
