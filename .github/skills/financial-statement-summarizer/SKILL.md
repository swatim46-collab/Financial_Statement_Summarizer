---
name: financial-statement-summarizer
description: "Use when building, testing, debugging, or changing the Financial Statement Summarizer: annual-report PDF ingestion, PyMuPDF, FastAPI, React/TypeScript, LangChain and Google AI Studio, financial ratios, source-grounded summaries and risks, or related tests."
---

# Financial Statement Summarizer

## Goal and sources of truth

Build and maintain the v1 product defined in the [PRD](../../../prd.md), following the [project instructions](../../instructions/financial-summarizer.instructions.md). The project instructions are stricter than the PRD where they conflict: in particular, make at most two LLM calls per report and do not retry a failed summary verification.

## Required implementation skills

- **PDF processing and provenance:** extract text page-by-page with PyMuPDF, preserve page numbers, and identify only relevant report sections.
- **API and data validation:** implement FastAPI endpoints and validate model output and API data with Pydantic.
- **Grounded LLM integration:** use LangChain with the configured Google AI Studio model for structured extraction and prose, not arithmetic or outside research.
- **Financial calculations:** compute ratios in deterministic, pure Python and make missing or incompatible inputs explicit.
- **Source verification:** check narrative numbers against the uploaded report and surface verification failures as warnings.
- **Frontend integration:** keep the React/TypeScript upload, loading, and results flow on one page and synchronize frontend types with the API.
- **Testing and secret handling:** mock model calls in deterministic tests; keep credentials on the backend.

## Workflow

1. **Inspect the existing implementation first.** Keep changes aligned with the PRD and these project instructions. Do not add the stretch comparison feature or other product scope without confirmation.
2. **Preserve the contract and v1 boundaries.** `POST /summarise` accepts one multipart PDF and returns `summary`, `ratios`, `risks`, and `warnings`; `GET /health` reports service health. V1 has no accounts, database, background jobs, OCR, or multi-report comparison.
3. **Parse and select source pages.** Use PyMuPDF and retain 1-based report page references. If no readable text is extracted, return exactly: `no readable text found, please upload a text-based PDF`. Do not add OCR. Select relevant financial-statement and risk-factor pages, plus only supporting business, management discussion, event, or outlook passages needed for the requested summary. Never send the entire report to the model.
4. **Extract structured facts.** Use one model call for extraction. Validate its output with Pydantic. Keep each figure tied to its reported label, period, currency/units, and source page; retain source text or an equivalent trace where practical. Include only risks stated in the report and attach their page references.
5. **Calculate ratios in Python.** Do not ask the model to calculate. Use source-backed inputs and expose each ratio's formula, inputs, result, and plain-English explanation:
   - Current ratio = current assets ÷ current liabilities.
   - Debt-to-equity = total debt ÷ total shareholders' equity; do not silently substitute total liabilities for debt.
   - Net profit margin = net income ÷ revenue.
   - Return on equity = net income ÷ average shareholders' equity, with average equity calculated from source-backed opening and closing balances.
   - Operating margin = operating income ÷ revenue. This is the required fifth ratio.

   Express margins and return on equity as percentages and current ratio/debt-to-equity as multiples. Preserve the report's currency, units, and labels in the inputs; do not convert currencies or units. If required figures are missing, a denominator is zero, or the inputs cannot be compared without conversion or an unsupported substitution, mark that ratio unavailable with a clear reason instead of guessing.
6. **Write and verify the response.** Use one model call for the one-page summary and ratio explanations, covering the business overview, revenue/profit trends, key events, and outlook. Ground all prose and risks only in the uploaded report; provide no outside data or investment advice. Verify each number in generated narrative against source text. Computed ratio results are exempt from direct source matching only when their input figures are source-backed. If verification fails, return a warning and do not make another model call. The extraction plus summary pipeline must never exceed two LLM calls total.
7. **Keep frontend and backend aligned.** Maintain the one-page upload/loading/results experience. Update TypeScript types whenever the response contract changes, and do not add screens or features outside v1.
8. **Protect credentials.** Keep `GOOGLE_API_KEY` in the backend `.env`; never put it in frontend code or commit it. Use placeholders in `.env.example` and keep model configuration server-side.
9. **Test the behavior, not the live model.** Mock LLM calls in deterministic tests. Cover manual results for all five ratios, unavailable ratios, unreadable-PDF handling and its exact message, page references, source-number verification, warning behavior, API contract, and the two-call maximum. Check ratio calculations against three reports when suitable public samples are available.

## Data and scope rules

- Use only facts from the uploaded report in a user's result. MCP Fetch may be used only to obtain public annual reports for development or testing; external data must never enter a user's generated summary.
- Do not invent figures, reinterpret missing values, fabricate risks, or omit provenance. Keep source page references with extracted facts and risks.
- Do not normalize or convert reported currency/units. If source figures are not comparable as reported, make the ratio unavailable rather than hiding a conversion.
- Treat model output as untrusted until schema validation and source verification have passed. A failed numeric check produces a warning, not a retry.

## Completion checklist

- [ ] One public annual-report PDF per request; no OCR or out-of-scope product features.
- [ ] Relevant pages only are sent to the model, with page provenance retained.
- [ ] All five ratio slots expose formula, report inputs, result or unavailable reason, and explanation.
- [ ] Narrative numbers and risks are report-grounded; verification failures appear in `warnings`.
- [ ] At most two model calls occur per report, including failure paths.
- [ ] API, frontend types, and deterministic tests agree on the response contract.
- [ ] No secret is exposed or committed.
