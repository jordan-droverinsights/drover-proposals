# Roadmap: Drover Insights Proposal PDF

## Overview

A single focused delivery: replace the existing custom 11-page PDF layout with an html2pdf.js-based capture approach that reliably outputs the full proposal with dark styling, embedded signature, client name, and signed date — running entirely in the browser, no server required. All 12 v1 requirements converge on this one outcome. Once this phase ships, the proposal tool works correctly for any client Kyle sends it to.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: PDF Generation Fix** - Replace custom 11-page PDF layout with html2pdf.js capture that outputs the full proposal faithfully

## Phase Details

### Phase 1: PDF Generation Fix
**Goal**: Client clicks Submit, signs the proposal, and downloads a PDF that captures the entire page — dark background, all sections, embedded signature, client name, and signed date — with no server required
**Depends on**: Nothing (first phase)
**Requirements**: PDF-01, PDF-02, PDF-03, PDF-04, PDF-05, PDF-06, PDF-07, PDF-08, IMPL-01, IMPL-02, IMPL-03, IMPL-04
**Success Criteria** (what must be TRUE):
  1. After signing and clicking Submit, the downloaded PDF contains all proposal sections — not just the active section visible on screen
  2. The PDF renders with the dark background and typography matching the proposal webpage (no white background, no font fallback)
  3. The drawn signature appears in the PDF at the correct location with the client name and signed date beneath it
  4. The PDF filename includes the client name (e.g. `VPS-Proposal-Signed-[ClientName].pdf`) and downloads automatically without a save dialog requiring extra steps
  5. The Submit button shows a generating state while the PDF builds and re-enables when the download completes; the browser makes no fetch request to localhost or any server during generation
**Plans**: 1 plan

Plans:
- [ ] 01-01-PLAN.md — Inline html2pdf bundle and replace generatePDF() with DOM-capture approach

## Progress

**Execution Order:**
Phases execute in numeric order: 1

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. PDF Generation Fix | 0/1 | Not started | - |
