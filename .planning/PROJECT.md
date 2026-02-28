# Drover Insights Proposal System

## What This Is

A self-contained, single-HTML proposal tool used by Drover Insights (Kyle Soria) to send signable, downloadable proposals to clients. Currently a polished dark-theme page with signature capture and PDF export that runs entirely client-side — no server required. The goal is to evolve this into a proposal builder: fill out a form, generate a customized proposal page for any client.

## Core Value

A client can read the proposal, sign it, and download a professional PDF — without Kyle needing a server, backend, or third-party tool.

## Requirements

### Validated

- ✓ Client-side PDF generation (html2canvas + jsPDF, 11-page output) — existing
- ✓ Signature pad capture embedded in PDF — existing
- ✓ Dark premium design with IBM Plex Mono + Bebas Neue typography — existing
- ✓ Service selection with live pricing calculation — existing
- ✓ PDF download without Puppeteer server — fixed in session 1

### Active

- [ ] Proposal builder: a form/UI that generates a customized proposal HTML for a new client
- [ ] Support multiple proposals (different clients, different service lines)
- [ ] Mobile experience improvements (layout, readability, touch interactions)
- [ ] Section redesign / rebrand capability (easy to update sections without rewriting the whole file)

### Out of Scope

- Server-side PDF generation — client-side works well and avoids hosting complexity
- Database or CMS — proposals stay as files
- Client portal / authentication — not needed, proposals are sent as direct links

## Context

- Single HTML file (~2700 lines) with all CSS, JS, and libraries inlined
- Libraries embedded inline: signature_pad.umd.min.js, jspdf.umd.min.js, html2canvas 1.4.1
- Deployed to a subdomain Kyle controls
- PDF generation builds 11 hidden pages (816×1056px), renders each with html2canvas at 2x scale, assembles with jsPDF
- Current proposal is specific to VPS (Vanguard Property Services) — names, prices, and content are hardcoded
- Fonts loaded from Google Fonts (IBM Plex Mono, IBM Plex Sans, Bebas Neue)

## Constraints

- **No server**: Everything must work as a static file opened in a browser
- **Single file**: Keep the proposal as one self-contained HTML file for easy deployment and sharing
- **No build tools required**: Changes should be editable in a text editor without a build step

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| html2canvas + jsPDF over Puppeteer | Client-side, no server dependency | ✓ Good — works reliably |
| Inline libraries (not CDN) | File works offline and without network | — Pending |
| Proposal builder as a separate form | Easier than editing HTML directly for new clients | — Pending |

---
*Last updated: 2026-02-28 after initialization*
