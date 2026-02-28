# Requirements: Drover Insights Proposal PDF

**Defined:** 2026-02-28
**Core Value:** Client signs the proposal and downloads a PDF that looks exactly like the webpage — dark design, all sections, signature captured — entirely in the browser.

## v1 Requirements

### PDF Generation

- [ ] **PDF-01**: After clicking Submit, PDF captures all proposal sections (full document, not just the visible section)
- [ ] **PDF-02**: PDF preserves the dark background and typography as it appears on the proposal webpage
- [ ] **PDF-03**: Drawn signature is visible in the PDF at the correct location
- [ ] **PDF-04**: Client name and signed date appear in the PDF (from the form fields)
- [ ] **PDF-05**: PDF downloads automatically with a descriptive filename (e.g. `VPS-Proposal-Signed-[ClientName].pdf`)
- [ ] **PDF-06**: PDF generation runs entirely client-side — no server, no fetch to localhost
- [ ] **PDF-07**: Client validation still runs before PDF generation (name required, signature required)
- [ ] **PDF-08**: Submit button shows progress state while generating, re-enables when done

### Implementation

- [ ] **IMPL-01**: Replace current `generatePDF()` (custom 11-page layout) with html2pdf.js page capture approach
- [ ] **IMPL-02**: All proposal sections temporarily made visible for capture, then restored after
- [ ] **IMPL-03**: html2pdf.js loaded from CDN or inlined — no server dependency
- [ ] **IMPL-04**: Existing jsPDF and html2canvas inline libraries can be removed or replaced once html2pdf.js is in place

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
| PDF-01 | Phase 1 | Pending |
| PDF-02 | Phase 1 | Pending |
| PDF-03 | Phase 1 | Pending |
| PDF-04 | Phase 1 | Pending |
| PDF-05 | Phase 1 | Pending |
| PDF-06 | Phase 1 | Pending |
| PDF-07 | Phase 1 | Pending |
| PDF-08 | Phase 1 | Pending |
| IMPL-01 | Phase 1 | Pending |
| IMPL-02 | Phase 1 | Pending |
| IMPL-03 | Phase 1 | Pending |
| IMPL-04 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 12 total
- Mapped to phases: 12
- Unmapped: 0 ✓

---
*Requirements defined: 2026-02-28*
*Last updated: 2026-02-28 after initial definition*
