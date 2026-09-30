# Financial Statement Summarizer Constitution

## Core Principles

### I. Source-Grounded Financial Outputs

Summaries and risks MUST be based only on the annual report uploaded for that request. Every
numeric claim in the summary and ratio explanations MUST match report text, except computed ratio
results. Each computed result MUST use a deterministic formula with every input traceable to
report figures. Each risk MUST be stated in the report and include its source page. The product
MUST NOT introduce outside data or investment advice. This makes each user-facing claim auditable
against its source.

### II. Deterministic Financial Calculations

The language model MUST be limited to structured extraction and prose generation. Python MUST
calculate ratios and verify source numbers. Each ratio MUST expose its formula, report inputs,
result, and plain-English explanation. If source figures are insufficient or incompatible, the
ratio MUST be marked unavailable rather than guessed. Reported currency, units, and labels MUST
be preserved without conversion. These rules keep calculations reproducible and prevent silent
assumptions.

### III. Minimal, Bounded Processing

Version 1 MUST accept one public annual-report PDF per request, extract text page-by-page with
PyMuPDF, and retain source page numbers. The model MUST receive only relevant financial-statement
and risk-factor pages plus the smallest necessary business, management, event, or outlook
passages for the required summary; the full report MUST NOT be sent. OCR is out of scope. Each
report MUST use at most two model calls: one extraction call and one summary call. If number
verification fails, the response MUST include a warning without another model call. This limits
unnecessary data exposure, latency, and free-tier usage.

### IV. Secure Server-Side Configuration

`GOOGLE_API_KEY` MUST remain in the backend `.env` and MUST NOT appear in frontend code or
committed files. `.env.example` MUST contain placeholders only. Credentials MUST NOT be
hard-coded. Server-side configuration keeps secrets out of client bundles and source control.

### V. Testable Contracts and Evidence

The backend response contract and frontend TypeScript types MUST remain synchronized. Changes to
the API MUST update both sides and include tests for the changed behavior. Deterministic tests MUST
mock model calls and cover ratio calculations, unavailable inputs, unreadable-PDF handling,
source-page references, number verification, warning behavior, and the two-call limit. Ratio
results MUST be checked against manual calculations on three reports. These checks make the
product's grounding and arithmetic requirements measurable.

## Product and Technical Constraints

- The v1 stack is Python and FastAPI, React with TypeScript and Vite, PyMuPDF, Pydantic, and
	LangChain with `langchain-google-genai` and `gemma-4-26b-a4b-it` via Google AI Studio.
- `POST /summarise` accepts a multipart PDF and returns `summary`, `ratios`, `risks`, and
  `warnings`. `GET /health` reports service health.
- The summary MUST fit one page and cover the business overview, revenue and profit trends, key
  events, and outlook. The five ratios MUST be current ratio, debt-to-equity, net profit margin,
  return on equity, and operating margin. Missing inputs MUST result in an unavailable ratio,
  not an inferred value.
- The upload, loading, and results experience MUST remain on one page. V1 MUST NOT add accounts,
  a database, background jobs, OCR, or multi-report comparison.
- Public annual reports MAY be fetched for development or testing only. External report data
  MUST NOT be used in a user's generated result.
- Any feature beyond v1, including the two-report comparison stretch goal, requires an explicitly
  approved feature specification before implementation.

## Engineering Workflow and Quality Gates

- Tests MUST demonstrate agreement with manual ratio calculations, including the three-report
  acceptance target. Model calls MUST be mocked in deterministic tests.
- Tests MUST check the exact unreadable-PDF message: `no readable text found, please upload a
  text-based PDF`.
- Before a change is accepted, reviewers MUST verify source-page provenance, source-backed inputs,
  narrative number checks, report-grounded risks, API/type consistency, and the two-call maximum.
- Verification failures MUST be visible in `warnings`; they MUST NOT trigger a summary retry.
- Changes MUST NOT expand product scope or weaken a grounding, security, or quality rule without
  an approved constitution amendment.

## Governance

This constitution governs project decisions. Product requirements MAY specify behavior in more
detail but MUST NOT weaken these principles. When requirements conflict with a principle, work
MUST pause until the requirements are reconciled or the constitution is formally amended.

Constitution amendments MUST include a rationale, affected principles, compatibility and test
impact, and project-owner approval. The amendment MUST update the version and last-amended date.
Versioning follows semantic versioning: MAJOR for backward-incompatible principle changes or
removals, MINOR for new principles or material expansions, and PATCH for clarifications or
non-semantic refinements. The ratified date records the initial adoption and MUST remain unchanged
by later amendments.

Every feature proposal and code review MUST check compliance with this constitution. Reviewers
MUST require test evidence for applicable quality gates; unresolved violations MUST be fixed or
addressed through an approved amendment before acceptance.

**Version**: 1.0.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-09-30
