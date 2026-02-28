# Requirements: Drover Insights Proposal PDF

**Defined:** 2026-02-28
**Core Value:** Client signs the proposal and downloads a PDF that looks exactly like the webpage — dark design, all sections, signature captured — entirely in the browser.

## v1 Requirements

### PDF Generation

- [x] **PDF-01**: After clicking Submit, PDF captures all proposal sections (full document, not just the visible section)
- [x] **PDF-02**: PDF preserves the dark background and typography as it appears on the proposal webpage
- [x] **PDF-03**: Drawn signature is visible in the PDF at the correct location
- [x] **PDF-04**: Client name and signed date appear in the PDF (from the form fields)
- [x] **PDF-05**: PDF downloads automatically with a descriptive filename (e.g. `VPS-Proposal-Signed-[ClientName].pdf`)
- [x] **PDF-06**: PDF generation runs entirely client-side — no server, no fetch to localhost
- [x] **PDF-07**: Client validation still runs before PDF generation (name required, signature required)
- [x] **PDF-08**: Submit button shows progress state while generating, re-enables when done

### Implementation

- [x] **IMPL-01**: Replace current `generatePDF()` (custom 11-page layout) with html2pdf.js page capture approach
- [x] **IMPL-02**: All proposal sections temporarily made visible for capture, then restored after
- [x] **IMPL-03**: html2pdf.js loaded from CDN or inlined — no server dependency
- [x] **IMPL-04**: Existing jsPDF and html2canvas inline libraries can be removed or replaced once html2pdf.js is in place

## v2 Requirements

### Builder

- **BUILD-01**: Proposal builder form — fill in client details, services, pricing to generate a new proposal HTML file
- **BUILD-02**: Download generated proposal as a self-contained HTML file
- **BUILD-03**: Save service bundles as presets for reuse

### Proposal UI

- **UI-01**: Mobile-responsive layout for proposal viewer
- **UI-02**: Section redesign / visual refresh

## Out of Scope

| Feature | Reason |
|---------|--------|
| Server-side PDF generation | Client-side is sufficient; eliminates hosting complexity |
| Puppeteer / headless rendering | Server dependency; client-side approach is the goal |
| Database or CMS | Not needed for single-user, file-based workflow |
| AI copy generation | Requires server-side API key handling |
| WYSIWYG editor | Heavy, unnecessary for this use case |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| PDF-01 | Phase 1 | Complete |
| PDF-02 | Phase 1 | Complete |
| PDF-03 | Phase 1 | Complete |
| PDF-04 | Phase 1 | Complete |
| PDF-05 | Phase 1 | Complete |
| PDF-06 | Phase 1 | Complete |
| PDF-07 | Phase 1 | Complete |
| PDF-08 | Phase 1 | Complete |
| IMPL-01 | Phase 1 | Complete |
| IMPL-02 | Phase 1 | Complete |
| IMPL-03 | Phase 1 | Complete |
| IMPL-04 | Phase 1 | Complete |

**Coverage:**
- v1 requirements: 12 total
- Mapped to phases: 12
- Unmapped: 0 ✓

---
*Requirements defined: 2026-02-28*
*Last updated: 2026-02-28 after initial definition*
