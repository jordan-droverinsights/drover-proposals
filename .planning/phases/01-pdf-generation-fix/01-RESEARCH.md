# Phase 1: PDF Generation Fix - Research

**Researched:** 2026-02-28
**Domain:** Client-side HTML-to-PDF conversion, html2canvas, browser rendering fidelity
**Confidence:** HIGH

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PDF-01 | After clicking Submit, PDF captures all proposal sections (full document, not just the visible section) | Temporarily make all `.section` elements `display: block` before capture, restore after. See Pattern 1. |
| PDF-02 | PDF preserves the dark background and typography as it appears on the proposal webpage | Set `backgroundColor: '#080808'` in html2canvas opts + `print-color-adjust: exact` CSS. See Pitfall 1. |
| PDF-03 | Drawn signature is visible in the PDF at the correct location | Convert signature canvas to `<img>` via `toDataURL` before DOM capture to avoid tainted-canvas blank. See Pattern 3. |
| PDF-04 | Client name and signed date appear in the PDF (from the form fields) | Input values must be copied to visible text nodes before capture (html2canvas renders input values inconsistently). See Pattern 3. |
| PDF-05 | PDF downloads automatically with descriptive filename `VPS-Proposal-Signed-[ClientName].pdf` | Pass `filename` option to html2pdf. Name is already being built correctly in existing code. |
| PDF-06 | PDF generation runs entirely client-side — no server, no fetch to localhost | html2pdf.js is a pure browser library. Load from CDN or inline the bundle. No server call needed. |
| PDF-07 | Client validation still runs before PDF generation (name required, signature required) | Existing `showFieldError` + `signaturePad.isEmpty()` checks stay in place before the html2pdf pipeline starts. |
| PDF-08 | Submit button shows progress state while generating, re-enables when done | Existing `btn.disabled = true / btn.textContent = '⏳ Generating...'` pattern carries over; html2pdf returns a Promise so `await` + `finally` re-enables button. |
| IMPL-01 | Replace current `generatePDF()` (custom 11-page layout) with html2pdf.js page capture approach | The entire offscreen-wrap + makePage + 11-page-loop body is replaced with a ~40-line html2pdf capture block. |
| IMPL-02 | All proposal sections temporarily made visible for capture, then restored after | QuerySelectorAll `.section`, add `display: block` to each, html2pdf, then restore `.section` classes. |
| IMPL-03 | html2pdf.js loaded from CDN or inlined — no server dependency | CDN: `https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js`. Inline: copy raw bundle into a `<script>` tag. |
| IMPL-04 | Existing jsPDF and html2canvas inline libraries can be removed or replaced once html2pdf.js is in place | The html2pdf.bundle.min.js already includes both html2canvas and jsPDF internally. The ~300 KB inline blobs (lines 1587–1931) can be deleted. |
</phase_requirements>

---

## Summary

The existing `generatePDF()` function builds 11 custom offscreen template pages from scratch — it does NOT screenshot the actual rendered proposal sections. It constructs abbreviated content manually via JavaScript-assembled HTML strings in a hidden `div#pdf-render-wrap`, then renders each custom page through html2canvas one at a time. This explains why the PDF looks nothing like the proposal webpage and misses section content: it is a hand-rolled re-layout, not a page capture.

The replacement approach (per IMPL-01/02) is fundamentally different: temporarily reveal all 10 proposal sections (`s01`–`s10`) by overriding their `display: none` CSS, capture the live proposal DOM through html2pdf.js, then restore the display state. This produces a PDF that is a direct visual screenshot of the actual rendered page — dark background, exact typography, all section content — with zero manual page construction.

The main technical challenges are: (1) the section-visibility toggle must happen synchronously before capture and be reliably restored even on error, (2) Google Fonts loaded via CDN require `await document.fonts.ready` before html2canvas fires or fonts fall back to system monospace, (3) the signature `<canvas>` element must be replaced with an `<img>` tag in the cloned DOM before rendering to avoid tainted-canvas blank output, and (4) html2canvas must receive `backgroundColor: '#080808'` to prevent the white-background override that is html2canvas's default.

**Primary recommendation:** Replace the entire `generatePDF()` body with an html2pdf.js capture of `document.querySelector('.proposal-wrap')` (or `document.body`), using the `onclone` callback to make all sections visible, embed the signature as an image, and fill in client name / date as text. Load `html2pdf.bundle.min.js` from CDN and delete the inline jsPDF + html2canvas blobs.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| html2pdf.js | 0.10.1 (CDN) / 0.14.0 (npm) | Orchestrates html2canvas + jsPDF into a single chainable PDF pipeline | Purpose-built wrapper for HTML-to-PDF; handles page splitting, image quality, and filename download automatically |
| html2canvas | 1.4.1 (bundled inside html2pdf) | Rasterizes DOM to a canvas | Required by html2pdf internally; already present in the existing file as an inline blob |
| jsPDF | 2.5.x (bundled inside html2pdf 0.10.1) | Creates the PDF document and handles download | Required by html2pdf internally; already present in the existing file as an inline blob |
| signature_pad | 2.x (already inlined) | Captures drawn signature on `<canvas id="sigCanvas">` | Already present; no changes needed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| document.fonts.ready | Web API (native) | Promise that resolves when all fonts are loaded | Await before calling html2pdf to prevent Google Fonts falling back to system fonts |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| html2pdf.js | Raw html2canvas + jsPDF (current approach) | Current approach requires manual per-page layout; html2pdf automates page splitting and is the locked decision |
| CDN-loaded bundle | Inline the ~300KB bundle as base64 or `<script>` tag | Inline = works offline; CDN = smaller initial HTML but requires internet. Project STATE.md says "all libraries inlined" — consider inlining |
| html2pdf 0.14.0 (npm) | html2pdf 0.10.1 (CDN) | 0.14.0 has improved node cloning (v0.12.0), native Promises (v0.11.0), and delay-before-capture (v0.11.1). CDN only has 0.10.1. If inlining, prefer fetching 0.10.1 from CDN or check unpkg for newer |

**Installation (CDN — add to `<head>`, replaces inline jsPDF + html2canvas blobs):**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"
  integrity="sha512-GsLlZN/3F2ErC5ifS5QtgpiJtWd43JWSuIgh7mbzZ8zBps+dvLusV+eNQATqgA/HdeKFVgA5v3S/cIrLF7QnIg=="
  crossorigin="anonymous" referrerpolicy="no-referrer"></script>
```

**If inlining (offline support), fetch bundle and paste as `<script>` block replacing existing inline libraries.**

---

## Architecture Patterns

### Recommended Project Structure

This is a single-file HTML project. All changes happen inside:

```
vps-proposal-fixed__5__1 (1) (1).html
├── <head>
│   ├── <link> Google Fonts (line 7) — keep, but await document.fonts.ready before capture
│   └── [REPLACE] inline jsPDF + html2canvas blobs (lines ~1587-1931) with html2pdf CDN script tag
├── <body> markup — no changes
└── <script> block
    ├── SECTIONS array, goTo(), signaturePad init — no changes
    └── [REPLACE] generatePDF() body — entire offscreen-wrap approach replaced with html2pdf capture
```

### Pattern 1: Reveal All Sections for Capture

**What:** Before capture, add `display: block` inline style to every `.section` element. After capture (success or failure), remove those inline styles to restore normal navigation state.

**When to use:** Mandatory. The section CSS is `.section { display: none }` with `.section.active { display: block }`. html2canvas only renders what is visually present.

```javascript
// Reveal all sections (save original inline display values)
var sections = Array.from(document.querySelectorAll('.section'));
var savedDisplay = sections.map(function(s) { return s.style.display; });
sections.forEach(function(s) { s.style.display = 'block'; });

// ... capture ...

// Restore — always run, even on error
sections.forEach(function(s, i) { s.style.display = savedDisplay[i]; });
```

**Anti-pattern:** Toggling the `.active` class (that would change navigation state and trigger animations). Use inline `style.display` only.

### Pattern 2: html2pdf.js Capture with Dark Background

**What:** The html2pdf.js chainable API. Critical options: `backgroundColor` inside `html2canvas`, `image: {type: 'png'}` for transparency-safe output (JPEG will show white where CSS background-color is transparent), `jsPDF` unit and format.

**When to use:** This is the core PDF generation call.

```javascript
// Source: https://ekoopmans.github.io/html2pdf.js/
var opt = {
  margin:       0,
  filename:     'VPS-Proposal-Signed-' + safeName + '.pdf',
  image:        { type: 'jpeg', quality: 0.95 },
  html2canvas:  {
    scale:           2,
    useCORS:         true,
    allowTaint:      true,
    backgroundColor: '#080808',
    logging:         false,
    windowWidth:     816,
    imageTimeout:    0,
    onclone: function(clonedDoc) {
      // Make all sections visible in the clone
      clonedDoc.querySelectorAll('.section').forEach(function(s) {
        s.style.display = 'block';
      });
      // Replace signature canvas with img (tainted canvas workaround)
      var sigCanvas = clonedDoc.getElementById('sigCanvas');
      if (sigCanvas && sigDataURL) {
        var img = clonedDoc.createElement('img');
        img.src = sigDataURL;
        img.style.cssText = sigCanvas.style.cssText;
        img.style.width  = sigCanvas.style.width  || sigCanvas.width + 'px';
        img.style.height = sigCanvas.style.height || sigCanvas.height + 'px';
        sigCanvas.parentNode.replaceChild(img, sigCanvas);
      }
    }
  },
  jsPDF:        { unit: 'pt', format: 'letter', orientation: 'portrait', compress: true }
};

await html2pdf().set(opt).from(document.body).save();
```

**Note on `onclone` approach:** The `onclone` callback is the safest way to make sections visible because it operates on the cloned document — it does not affect the live page, so section navigation state is never disrupted even if an exception occurs during rendering.

### Pattern 3: Signature and Form Field Capture

**What:** The `<canvas id="sigCanvas">` containing the drawn signature will render blank if html2canvas considers it tainted (a known browser security behavior even for same-origin canvases in certain rendering contexts). Convert to `<img>` before capture.

**When to use:** Always — apply in the `onclone` callback.

```javascript
// Before calling html2pdf, capture signature data URL from the live canvas
var sigDataURL = signaturePad.toDataURL('image/png');

// In the onclone callback, swap the canvas for an img
// (sigDataURL is captured in closure)
onclone: function(clonedDoc) {
  var sigCanvas = clonedDoc.getElementById('sigCanvas');
  if (sigCanvas && sigDataURL) {
    var img = clonedDoc.createElement('img');
    img.src = sigDataURL;
    img.style.display = 'block';
    img.style.width = '100%';
    img.style.height = '140px';
    sigCanvas.parentNode.replaceChild(img, sigCanvas);
  }
  // Also copy form field values to visible text spans
  // (html2canvas renders input placeholder text, not input.value, on some browsers)
  var nameInput  = clonedDoc.getElementById('clientName');
  var dateInput  = clonedDoc.getElementById('clientDate');
  if (nameInput) {
    var span = clonedDoc.createElement('div');
    span.style.cssText = nameInput.style.cssText || '';
    span.style.color = '#F0EDE8';
    span.textContent = nameInput.value;
    nameInput.parentNode.insertBefore(span, nameInput);
    nameInput.style.display = 'none';
  }
  // Same pattern for date if needed
}
```

### Anti-Patterns to Avoid

- **Manipulating the live DOM then restoring:** Risk of navigation state corruption if the async capture throws. Use `onclone` for the section-reveal and canvas-swap instead.
- **Using `image: { type: 'jpeg' }` with transparent backgrounds:** JPEG has no alpha channel — any transparent areas render white. Use JPEG only if the background is explicitly set to a solid color on the container (`background: #080808`).
- **Calling `html2pdf` before `document.fonts.ready`:** Google Fonts served from CDN will fall back to the system monospace/sans-serif. The IBM Plex and Bebas Neue brand fonts will be missing.
- **Setting `scale` above 2 on mobile:** iOS Safari caps canvas area at ~3-5 megapixels. At scale 2 on an 816px-wide container, the canvas is approximately 1632×N px. Keep scale: 2.
- **Leaving existing inline jsPDF/html2canvas blobs in place after adding html2pdf bundle:** Creates a version conflict — html2pdf.bundle.min.js bundles its own jsPDF and html2canvas. The existing inline libraries (lines ~1587–1931) become dead weight and may shadow the bundled versions.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Page splitting across PDF pages | Custom page-height slicer, clipping divs to 1056px | html2pdf.js `pagebreak` option | Edge cases with content overflow, orphaned lines, partial images at page boundaries |
| PDF creation from canvas data | Manual `doc.addImage` + `doc.addPage` loop | html2pdf.js pipeline | Manages coordinate mapping, page dimensions, compression — already wired to html2canvas output |
| Download trigger | `<a download>` creation + `.click()` hack | `jsPDF.save(filename)` via html2pdf | jsPDF handles cross-browser download behavior including Safari |
| Font readiness | `setTimeout` delay | `await document.fonts.ready` | Deterministic; setTimeout is a guess that fails on slow connections |

**Key insight:** The current code already uses `doc.addImage` + `doc.addPage` correctly — the problem is not in the PDF assembly loop but in what content the loop renders. Replacing the manual page-builder with `html2pdf().from(element).save()` eliminates the root cause: hand-reconstructed page content that diverges from the actual proposal.

---

## Common Pitfalls

### Pitfall 1: White Background Override
**What goes wrong:** html2canvas defaults `backgroundColor` to `#ffffff`. If not overridden, every captured page will have a white background, completely destroying the dark design.
**Why it happens:** html2canvas needs an explicit background since the DOM root's `background-color` may be `transparent` or inherited from a CSS custom property that the renderer doesn't resolve.
**How to avoid:** Always pass `backgroundColor: '#080808'` (the proposal's root background) in the html2canvas options block. Additionally, add `* { -webkit-print-color-adjust: exact !important; print-color-adjust: exact !important; }` to the document CSS to force browsers to include background colors in rendered output.
**Warning signs:** PDF has white pages with dark text floating on it.

### Pitfall 2: Google Fonts Not Loaded at Capture Time
**What goes wrong:** The PDF renders with system monospace/sans-serif instead of IBM Plex Mono and Bebas Neue.
**Why it happens:** Google Fonts are loaded asynchronously via the `<link>` tag at line 7. html2canvas captures the DOM before the fonts finish downloading.
**How to avoid:** `await document.fonts.ready` before calling html2pdf. This is a native browser API that returns a Promise resolving when all web fonts are ready.
**Warning signs:** PDF shows generic monospace text with incorrect letter-spacing and character widths; Bebas Neue display headings show as a different typeface.

### Pitfall 3: Signature Canvas Renders Blank
**What goes wrong:** The drawn signature does not appear in the PDF — its canvas area is blank or entirely absent.
**Why it happens:** html2canvas may treat `<canvas>` elements as tainted if they contain data that was not drawn via browser-rendered content (signature_pad draws programmatically via pointer events). The security model can block `toDataURL` on the cloned canvas.
**How to avoid:** Before calling html2pdf, call `signaturePad.toDataURL('image/png')` to capture the signature as a data URL. In the `onclone` callback, replace the canvas element with an `<img src="[dataURL]">` element of identical size.
**Warning signs:** PDF shows the signature box border/label but the signature drawing area is empty.

### Pitfall 4: Hidden Sections Not Captured
**What goes wrong:** The PDF only contains the single active section (e.g., section 10 "Next Steps") if the user happens to be on that section when submitting.
**Why it happens:** Non-active sections have `display: none` — html2canvas skips hidden elements.
**How to avoid:** Use the `onclone` callback to apply `display: block` to every `.section` in the cloned document. The live page is never touched.
**Warning signs:** PDF has one page of content instead of all 10 sections.

### Pitfall 5: Input Field Values Not Captured
**What goes wrong:** The "Client Name" and "Date" input fields appear blank or show placeholder text in the PDF.
**Why it happens:** html2canvas renders the computed visual state of inputs, which may show placeholder rather than value on some browsers (especially when `font-color` is a CSS variable or the input styling uses `:placeholder`).
**How to avoid:** In the `onclone` callback, read `.value` from each input and either replace the input with a visible `<div>` containing the value text, or force `.setAttribute('value', inputEl.value)` on the cloned input. The latter does not guarantee rendering but the former always works.
**Warning signs:** Signature block in PDF shows blank name line and/or blank date.

### Pitfall 6: Version Conflict Between Inline Libraries and html2pdf Bundle
**What goes wrong:** PDF generation silently fails or produces unexpected output.
**Why it happens:** The existing HTML has jsPDF and html2canvas inlined as ~300KB minified blobs (approximately lines 1587–1931). If html2pdf.bundle.min.js is loaded via `<script>` tag, it brings its own versions of both libraries. Whichever loads last wins on `window.jspdf` / `window.html2canvas`, creating inconsistent behavior.
**How to avoid:** Remove the existing inline jsPDF and html2canvas blobs entirely when adding html2pdf.bundle.min.js. The bundle contains both.
**Warning signs:** `html2canvas is not a function` or `jsPDF is not defined` errors in the console.

---

## Code Examples

### Complete Replacement generatePDF() Function

```javascript
// Source: html2pdf.js docs (https://ekoopmans.github.io/html2pdf.js/) +
//         html2canvas docs (https://html2canvas.hertzen.com/configuration)
async function generatePDF() {
  var clientName  = (document.getElementById('clientName')  || {}).value || '';
  var clientEmail = (document.getElementById('clientEmail') || {}).value || '';
  var clientDate  = (document.getElementById('clientDate')  || {}).value || '';
  var st = document.getElementById('pdfStatus');

  // Guard: library loaded
  if (typeof html2pdf === 'undefined') {
    if (st) { st.textContent = '⚠ PDF library not loaded.'; st.style.color = 'rgba(230,170,80,0.95)'; st.style.display = 'block'; }
    return;
  }
  // Guard: validation
  if (!clientName.trim()) {
    showFieldError('clientName', 'Enter your full name above before submitting.');
    return;
  }
  if (!signaturePad || signaturePad.isEmpty()) {
    var wrap = document.getElementById('sigPadWrap');
    if (wrap) { wrap.style.borderColor = 'rgba(230,170,80,0.8)'; setTimeout(function(){ wrap.style.borderColor=''; }, 3000); wrap.scrollIntoView({behavior:'smooth', block:'center'}); }
    return;
  }

  // Capture signature data URL BEFORE cloning (canvas not accessible after clone)
  var sigDataURL = signaturePad.toDataURL('image/png');
  var safeName   = clientName.replace(/[^a-zA-Z0-9\s-]/g, '').replace(/\s+/g, '-').trim() || 'Client';

  // Wait for Google Fonts to be ready
  await document.fonts.ready;

  var opt = {
    margin:       0,
    filename:     'VPS-Proposal-Signed-' + safeName + '.pdf',
    image:        { type: 'jpeg', quality: 0.95 },
    html2canvas:  {
      scale:           2,
      useCORS:         true,
      allowTaint:      true,
      backgroundColor: '#080808',
      logging:         false,
      windowWidth:     816,
      imageTimeout:    0,
      onclone: function(clonedDoc) {
        // Make ALL sections visible for capture
        clonedDoc.querySelectorAll('.section').forEach(function(s) {
          s.style.display = 'block';
        });
        // Replace signature canvas with img (avoids tainted-canvas blank)
        var sigEl = clonedDoc.getElementById('sigCanvas');
        if (sigEl) {
          var img = clonedDoc.createElement('img');
          img.src = sigDataURL;
          img.style.display = 'block';
          img.style.width   = '100%';
          img.style.height  = '140px';
          img.style.objectFit = 'contain';
          sigEl.parentNode.replaceChild(img, sigEl);
        }
        // Bake input field values into visible elements
        ['clientName', 'clientDate'].forEach(function(id) {
          var input = clonedDoc.getElementById(id);
          if (input && input.value) {
            var div = clonedDoc.createElement('div');
            div.className = input.className;
            div.style.cssText = window.getComputedStyle(input).cssText || '';
            div.style.color = '#F0EDE8';
            div.textContent = input.value;
            input.parentNode.insertBefore(div, input.nextSibling);
            input.style.display = 'none';
          }
        });
      }
    },
    jsPDF: { unit: 'pt', format: 'letter', orientation: 'portrait', compress: true }
  };

  await html2pdf().set(opt).from(document.body).save();
}
```

### Removing Old Inline Library Blobs

The inline jsPDF blob starts around line 1539 with `/* jsPDF...` and the html2canvas blob occupies a separate large `<script>` block. Both can be identified by:

```
grep -n "jsPDF\|html2canvas" file.html | head -20
```

Replace both inline `<script>` blocks with a single CDN tag in `<head>`:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"
  integrity="sha512-GsLlZN/3F2ErC5ifS5QtgpiJtWd43JWSuIgh7mbzZ8zBps+dvLusV+eNQATqgA/HdeKFVgA5v3S/cIrLF7QnIg=="
  crossorigin="anonymous" referrerpolicy="no-referrer"></script>
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Custom hand-rolled 11-page offscreen layout | html2pdf.js DOM-capture (`from(element).save()`) | This phase | Eliminates maintenance of per-section content extraction; output matches webpage exactly |
| `new jsPDF()` + `html2canvas()` manual pipeline | `html2pdf().set(opt).from(el).save()` | This phase | Reduces pdf-generation code from ~600 lines to ~40 lines |
| jsPDF + html2canvas as separate inline blobs | html2pdf.bundle.min.js (contains both, versioned together) | This phase | Removes ~300KB of inline library code, single CDN dependency |

**html2pdf.js version history (relevant):**
- **0.10.1** (CDN-available): Stable, widely tested. The cdnjs version.
- **0.11.0** (2025-08): Native Promises, improved test coverage.
- **0.11.1** (2025-08): Added delay before capture to allow content to render.
- **0.12.0** (2025-09): Improved node cloning using @zumer/snapdom deepClone — fixes CSS cloning bugs.
- **0.14.0** (2026-01): Latest; sanitize text sources. Not yet on cdnjs CDN.

**Note:** If using the CDN approach (0.10.1), the `onclone` workarounds in this research are necessary. Version 0.12.0+ improves CSS cloning but is not on CDN; if inlining is chosen, consider fetching 0.12.0+ from unpkg.

---

## Open Questions

1. **Inline vs CDN for html2pdf.js bundle**
   - What we know: STATE.md says "All libraries inlined (not CDN) — proposals work offline." IMPL-03 says "loaded from CDN or inlined."
   - What's unclear: The existing file already has jsPDF and html2canvas inlined (~300KB). html2pdf.bundle.min.js is a similar size. Inlining would maintain offline capability.
   - Recommendation: Inline the bundle. Fetch `https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js`, paste as a `<script>` block, replacing the existing two inline blobs. Net file size change is roughly neutral.

2. **Page breaks between sections**
   - What we know: html2pdf.js auto-splits content across letter-size pages. Ten sections of varying height will result in page breaks that may cut through section content mid-element.
   - What's unclear: Whether this is acceptable (PDF is a faithful capture, not a perfectly paginated document) or whether page-break control is needed.
   - Recommendation: Accept natural page breaks for v1 (success criteria do not require specific page break positions). Add `pagebreak: { avoid: '.section' }` if mid-section cuts are unacceptable in practice.

3. **iOS Safari canvas memory limit**
   - What we know: STATE.md notes iOS Safari caps canvas at 3-5MP per canvas; scale 2 on a 816px-wide `document.body` is safe per-canvas but full-page height may exceed limits.
   - What's unclear: The total height of all 10 sections visible simultaneously at 816px width. This needs a hardware test.
   - Recommendation: If iOS testing shows a blank/truncated PDF, reduce scale from 2 to 1.5. Add a note in the implementation to test on an actual iOS device before marking done.

---

## Sources

### Primary (HIGH confidence)
- https://ekoopmans.github.io/html2pdf.js/ — configuration options, CDN URL, version 0.10.1 confirmed
- https://raw.githubusercontent.com/eKoopmans/html2pdf.js/master/README.md — defaults for all config options
- https://html2canvas.hertzen.com/configuration — all html2canvas options with defaults verified
- https://github.com/eKoopmans/html2pdf.js/releases — release history confirmed; v0.14.0 is latest as of 2026-01-12

### Secondary (MEDIUM confidence)
- https://github.com/eKoopmans/html2pdf.js/issues/405 — backgroundColor not passed to html2canvas correctly in some versions; workaround: set `html2canvas.backgroundColor` explicitly
- https://github.com/niklasvh/html2canvas/issues/1940 — Fonts not loaded in time; confirmed pattern: await document.fonts.ready before capture
- https://www.bennadel.com/blog/4262-generating-pdf-signatures-with-google-fonts-and-html2canvas-in-javascript.htm — Google Fonts + html2canvas + signature PDF pattern

### Tertiary (LOW confidence)
- Community reports of `onclone` callback issues in html2pdf.js (github.com/eKoopmans/html2pdf.js/issues/413) — unverified whether this affects 0.10.1 specifically; mitigation is to test and fall back to live-DOM section toggle if onclone proves unreliable

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — html2pdf.js CDN version confirmed, html2canvas options verified from official docs
- Architecture: HIGH — current code fully analyzed via grep; replacement pattern is directly derived from the actual DOM structure
- Pitfalls: HIGH — backgroundColor default, font loading, canvas taint, and section visibility are all verified from official sources and documented issues
- iOS canvas limits: MEDIUM — documented in STATE.md as a concern; specific pixel thresholds are browser-version-dependent

**Research date:** 2026-02-28
**Valid until:** 2026-03-28 (html2pdf.js is actively developed but the 0.10.1 CDN version is stable)
