## D1: Report Architecture

**Choice:** Report Service + Template pattern — one service per report type producing a ReportModel POJO, shared ReportRenderer for JSON/PDF output, single ReportResource JAX-RS class.
**Alternatives:**
- Single monolithic ReportService — simpler but unwieldy as report complexity grows
- Command pattern with ReportGenerator interface — over-engineered for 3 known report types
**Rationale:** Clean separation between data aggregation (services) and rendering (shared pipeline). Each service independently testable. Renderer reusable for new report types.
**Trade-offs:** Three service classes instead of one — marginal complexity for significant testability gain.
**Sources:** Existing PatientComplianceResource pattern, issue #162 requirements
**Exploration:** quick
**Status:** captured

## D2: PDF rendering library

**Choice:** Use existing platform-pdf — PdfGenerator SPI (platform-api) + OpenHtmlToPdfGenerator (platform-pdf, openhtmltopdf + PDFBox, PDF/A-2B compliant, Liberation fonts embedded). No new library needed.
**Alternatives:**
- OpenPDF — would duplicate what platform-pdf already provides
- Playwright/headless Chrome — heavy, unnecessary given platform-pdf exists
- Flying Saucer — another XHTML renderer, but platform already chose openhtmltopdf
**Rationale:** Platform already solved this. OpenHtmlToPdfGenerator produces PDF/A-2B (regulatory archival standard) with embedded fonts. No reason to introduce a second PDF library.
**Trade-offs:** None — strictly better than adding a new dependency.
**Sources:** casehub-platform/platform-pdf/OpenHtmlToPdfGenerator.java, casehub-platform-api/pdf/PdfGenerator.java
**Exploration:** quick
**Status:** captured

## D3: Extraction boundary — platform vs ledger vs clinical

**Choice:** Three-layer split — platform-pdf (exists, PdfGenerator SPI), new casehub-ledger-reporting module (compliance/audit reports + template rendering), clinical (IND safety report only).
**Alternatives:**
- Build everything in clinical — faster but locks reusable capability into one app
- New casehub-platform-reporting module — platform-pdf + PdfGenerator SPI already cover the rendering; adding Qute + content negotiation there would work but compliance logic belongs with compliance data (ledger)
**Rationale:** ComplianceSupplement, LedgerVerificationService, LedgerProvExportService all live in casehub-ledger. The reports that aggregate this data belong next to the data. Rendering SPI is already in platform. Clinical only owns the IND report (FDA-specific).
**Trade-offs:** Clinical#162 depends on ledger#211 shipping first. Can stub/mock the ledger-reporting services for initial development.
**Sources:** casehubio/ledger#211, casehubio/platform#383 (closed — SPI exists), PdfGenerator SPI
**Exploration:** quick
**Depends on:** D2 (PDF rendering library)
**Status:** captured
