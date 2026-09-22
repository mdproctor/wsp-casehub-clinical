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
