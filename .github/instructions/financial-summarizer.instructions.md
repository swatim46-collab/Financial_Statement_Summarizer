---
description: "Use when implementing or changing the Financial Statement Summarizer backend, frontend, PDF parsing, ratios, LLM pipeline, API, or tests. Follow the PRD's source-grounding rules and v1 scope."
applyTo:
  - "backend/**/*.py"
  - "frontend/**/*.ts"
  - "frontend/**/*.tsx"
---

# Financial Statement Summarizer Instructions

- Treat [prd.md](../../prd.md) as the product source of truth. Keep v1 to one public annual-report PDF per request, with no accounts, database, or background jobs.
- Preserve the API contract: `POST /summarise` accepts a multipart PDF and returns `summary`, `ratios`, `risks`, and `warnings`; `GET /health` reports service health. Keep frontend and backend types in sync when changing the contract.
- Parse PDFs with PyMuPDF and retain source page numbers. If no readable text is extracted, return the exact message: `no readable text found, please upload a text-based PDF`. Do not add OCR to v1.
- Send only relevant financial-statement and risk-factor pages to the model, not the entire report.
- Use the LLM for structured extraction and prose only. Validate extracted data with Pydantic; calculate ratios in deterministic, pure Python code.
- Each ratio must expose its formula, report inputs, result, and plain-English explanation. Preserve the report's currency and units; do not convert them.
- Use operating margin as the fifth ratio. If the report does not provide enough source figures to calculate a ratio, mark it unavailable rather than guessing.
- Ground results only in the uploaded report: do not add outside data or investment advice. Include only risks stated in the report, with page references.
- Verify that each number in generated narrative appears in the source text. Computed ratios may be new values, but their input figures must be source-backed. If verification fails, return a warning without another model call.
- Make at most two LLM calls per report total: one for extraction and one for summary. This strict call cap follows the user's clarification and supersedes the PRD's summary-retry behavior. Avoid sending irrelevant report content to the model.
- Keep the upload, loading, and results experience to one page. Do not add product features outside the PRD without confirmation.
- Keep `GOOGLE_API_KEY` on the backend in `.env`; never expose or commit secrets. Use placeholders in `.env.example`.
- Test ratio calculations against manual results, plus unreadable-PDF handling, page references, source-number verification, warning behavior, and the two-call maximum. Mock model calls in deterministic tests; the PRD's acceptance target is agreement with manual calculations on three reports.
- Use MCP Fetch only to obtain public annual reports for development/testing when needed; never use external report data in a user's generated summary.
