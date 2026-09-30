# Research and Design Decisions: Financial Statement Summarizer

**Date**: 2026-09-30  
**Scope**: Resolve implementation choices for the v1 specification and the project constitution.

## Decisions

### 1. Runtime and package management

**Decision**: Target the workspace's observed Python 3.14.4 and Node.js 22.22.2/npm 10.9.7 toolchains. Use the PRD's `backend/requirements.txt`, `frontend/package.json`, and a committed npm lockfile. Pin the Python dependencies to versions that install and pass tests on the chosen interpreter; verify wheel availability before committing the set. Use Vite's React/TypeScript template and npm scripts for development, build, and tests.

**Rationale**: These are the active local toolchains, and current Vite documentation requires Node 20.19+ or 22.12+, which Node 22.22.2 satisfies. A simple requirements file and npm lockfile fit the existing PRD layout without introducing another package manager.

**Alternatives considered**: A separate Python environment manager or monorepo tool could improve larger-workspace management, but would add setup complexity to this small app. A Python downgrade is not planned unless a required dependency fails compatibility testing; if that happens, document the specific package constraint before changing the runtime target.

**Evidence**: [Vite Getting Started](https://vite.dev/guide/).

### 2. Upload and PDF handling

**Decision**: Accept one multipart `file` field. Use FastAPI `UploadFile` and enforce a 25 MiB limit while reading; reject files over the limit or reports over 1,200 pages. Check the PDF signature and let PyMuPDF validate/open the content as a PDF. Open the bounded bytes as an in-memory stream, extract text locally page by page, and close the request upload on every success/error path. Do not write a durable copy. A corrupt or non-PDF upload receives a clear client error; a valid PDF with no readable text receives the exact required message and does not reach the model.

**Rationale**: FastAPI's `UploadFile` uses a spooled file interface and avoids declaring an upload as an unbounded in-memory `bytes` body. PyMuPDF can open document bytes as a stream. The explicit limits bound memory and processing while covering typical public annual reports; they are configurable implementation limits, not a change to the one-report product scope.

**Alternatives considered**: Saving uploads into a report store or using a database is prohibited by v1. Passing the entire PDF to Google's file API is rejected because v1 requires local text extraction and sending only relevant extracted pages. OCR is explicitly out of scope.

**Evidence**: [FastAPI Request Files](https://fastapi.tiangolo.com/tutorial/request-files/); [PyMuPDF Opening Files](https://pymupdf.readthedocs.io/en/latest/how-to-open-a-file.html).

### 3. Page extraction and relevance selection

**Decision**: Extract every page locally with PyMuPDF, retain the 1-based PDF page number (`page.number + 1`), and use sorted text extraction as the initial reading-order heuristic. Build a local keyword/heading selector for financial statements, risk factors, business description, management discussion, key events, and outlook. Send only selected pages, prefixed with page number and section label, to the extraction call. Apply an 80,000-character budget; prioritize financial statements and risk-factor pages, then the smallest necessary passages for the required summary. If relevant content is omitted by the budget or layout cannot be reliably interpreted, include a warning and leave unsupported facts/ratios unavailable rather than infer them. Use only text-layer content; do not invoke OCR.

**Rationale**: The PRD and constitution explicitly restrict model input to relevant pages. PyMuPDF documents that plain extraction may not match visual reading order and offers sorted text, block/word extraction, and table-finding support. Starting with sorted text plus source quotes is the smallest v1 approach; ambiguous layouts must fail safely.

**Alternatives considered**: Sending the complete report is prohibited. Adding a vector database, embeddings, a general RAG stack, or OCR would expand the scope and data handling. Add table reconstruction only if acceptance fixtures demonstrate that ordinary sorted page text cannot preserve a required statement value.

**Evidence**: [PyMuPDF Text Extraction](https://pymupdf.readthedocs.io/en/latest/recipes-text.html); [PyMuPDF, LLM and RAG](https://pymupdf.readthedocs.io/en/latest/rag.html).

### 4. Structured extraction and model compatibility

**Decision**: Use a small LangChain provider adapter backed by `ChatGoogleGenerativeAI`, configure `MODEL_NAME` on the backend, and validate one structured extraction response with Pydantic. The extraction response contains report facts, financial figures, risks, and exact source quotes/page numbers. Use one summary invocation for the one-page prose and five qualitative ratio explanations. Pass the summary call only extracted facts, supported figures, and deterministic ratio results—not raw/full report text. Keep computed ratio result values in structured fields; ratio explanation prose must not repeat those computed numbers. Set provider retry behavior to zero and do not retry malformed output, provider errors, or verification failures. Unit and API tests mock the adapter.

The required identifier remains `gemma-4-26b-a4b-it` from the PRD. Before integration is accepted, perform a developer-key smoke test confirming that this identifier is available to the AI Studio account and that the chosen structured-output mode works. The official Google model catalogue inspected for this plan did not visibly list that Gemma identifier, and the LangChain structured-output examples do not establish its specific compatibility. If the smoke test fails, block release and ask the product owner to approve a replacement; do not silently fall back to another model.

**Rationale**: LangChain's Google integration documents Pydantic/JSON-schema structured output; Google recommends validating semantic content in the application even when JSON schema is used. The adapter isolates provider-specific behavior and makes the strict call limit testable. Provider-level automatic retries must also be disabled because a retry would violate the two-invocation cap.

**Alternatives considered**: Directly uploading PDFs to Google's files API, adding Google Search grounding, or letting the model calculate ratios are rejected. A different model is not selected without user approval. A second attempt to repair bad JSON or summary prose is rejected by the constitution.

**Evidence**: [LangChain ChatGoogleGenerativeAI](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai); [Google Gemini Models](https://ai.google.dev/gemini-api/docs/models); [Google Structured Outputs](https://ai.google.dev/gemini-api/docs/structured-output).

### 5. Financial figure mapping and deterministic ratios

**Decision**: Keep parsed financial values as Python `Decimal`; retain the exact reported label/value string, unit/currency/scale, period, reporting scope, 1-based source page, and source quote beside each numeric value. Calculate the fixed ratios in pure Python:

- Current ratio = current assets ÷ current liabilities.
- Debt-to-equity = total debt ÷ total shareholders' equity. Use a clearly reported total debt, or sum clearly identified current and long-term interest-bearing borrowings; do not substitute total liabilities.
- Net profit margin = net income ÷ revenue.
- Return on equity = net income ÷ average opening/closing shareholders' equity for the same reporting entity and period.
- Operating margin = operating income (or an unambiguous equivalent reported line item) ÷ revenue.

Use comparable reporting periods and reporting scope. Preserve source units and do no currency/unit conversion. A missing/ambiguous input, incompatible period/scope/unit, or zero denominator yields an unavailable result and an explicit reason. Round only the displayed ratio to two decimal places using decimal half-up rounding; keep full precision for intermediate arithmetic. Return the five ratios in a stable order.

**Rationale**: The formulas and fifth ratio are fixed by the constitution and the instructions. `Decimal` avoids binary floating-point drift. Preserving raw display strings plus source data supports auditability and avoids silent scale changes.

**Alternatives considered**: Calculating ratios in the model, treating total liabilities as debt, using ending equity instead of average equity, converting reported figures, or inferring a missing line item are rejected.

### 6. Numeric verification and evidence policy

**Decision**: Verify numeric tokens in generated summary prose, ratio explanations, and risk descriptions against the locally extracted report text. Prompt the model to use digits copied from the source for numeric claims; do not spell newly generated numbers out as words or create numeric trend percentages. If a number word or ordinal is emitted, accept it only when its numeric phrase is present in the source text. Require model-extracted facts/figures/risks to carry a source quote and page; validate that the normalized quote occurs on the cited page. Normalize only representation differences such as thousands separators, whitespace, and Unicode minus/accounting parentheses. Do not equate different currencies, scales, or units. Ratio result values, formulas, source-page metadata, and raw input evidence are structured fields, not generated narrative, and are excluded from the narrative-number scan. A failed narrative check leaves the result in place with a warning and causes no additional model invocation.

**Rationale**: This enforces the source-grounding rule without rejecting a computed ratio merely because the result does not appear verbatim in the report. Keeping ratio results separate from prose makes the exemption narrow and auditable.

**Alternatives considered**: Fuzzy numeric matching across scales/units could accept unsupported conversions and is rejected. Retrying summary generation conflicts with the strict two-call clarification and constitution.

### 7. API and frontend contract

**Decision**: Define the public contract in `contracts/openapi.yaml`. `POST /summarise` accepts exactly one multipart field named `file`; successful responses contain `summary`, exactly five `ratios`, `risks`, and `warnings`. Each ratio has its name, formula, report inputs, result or unavailable status/reason, result unit, and plain-English explanation. Each report input and each risk evidence quote are paired with a 1-based PDF page reference. `GET /health` is a liveness endpoint returning `{"status":"ok"}` without calling the model. Use a consistent JSON error envelope with a code and user-safe message; for unreadable text, the message value is exactly the required string. Generate frontend TypeScript types from the OpenAPI contract and test the FastAPI response against it.

The frontend uses one accessible form, browser `FormData`, and explicit initial/loading/success/error states. Do not manually set the multipart `Content-Type` header; the browser must provide its boundary. Use a non-secret `VITE_API_BASE_URL` only; never put model credentials in Vite settings.

**Rationale**: A single contract source reduces backend/frontend drift. FastAPI response models can validate and document responses. Browser `FormData` is the standard multipart path for sending a selected file.

**Alternatives considered**: Persisted result URLs/history, multi-page wizard, generated PDF export, user accounts, or a second report flow exceed v1. Maintaining duplicate handwritten API types without contract tests risks drift.

**Evidence**: [FastAPI Response Models](https://fastapi.tiangolo.com/tutorial/response-model/); [FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/); [MDN FormData](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_FormData_Objects); [React input](https://react.dev/reference/react-dom/components/input).

### 8. Storage, secrets, and test strategy

**Decision**: No report/result database, persistent cache, authentication, or background job. Keep `GOOGLE_API_KEY` and model settings backend-only in `.env`; commit placeholders in `.env.example`; do not log PDF contents, extracted passages, or credentials. Use pytest and mocked model calls for deterministic tests. Generate small text-based PDFs in test setup for parser/API tests; manually validate ratio outcomes against three public annual reports when sample reports are available. Use Vitest/React Testing Library with a mocked API for frontend state and rendering.

**Rationale**: This directly follows the one-request-in/one-response-out product and minimizes personal data retention and test flakiness.

**Alternatives considered**: Live LLM calls in CI, committing credentials, adding a database, or making deterministic tests depend on external report hosts are rejected.

**Test-fixture limitation**: No report PDFs are present in the workspace and the configured MCP server does not provide the PRD's Fetch capability. Use generated synthetic PDFs for deterministic tests; acquire three approved public report fixtures through the permitted development/testing workflow before claiming the three-report acceptance criterion is complete.

### 9. Copilot Stop-hook verification

**Decision**: Keep `backend/app/verifier.py` as the only source-number verification implementation. Provide a small deterministic CLI adapter for explicit source/summary test artifacts and configure the PRD's Copilot Stop hook to invoke that adapter or its test gate. The hook makes no model calls and must not implement a second numeric-matching algorithm. The application continues to call the verifier on every live request regardless of hook behavior. During implementation, confirm the supported Stop-hook event can supply the source/result artifact paths; if it cannot, the hook must run the verifier test suite while the in-request check remains the authoritative live-result guard.

**Rationale**: The PRD requests both an application verifier and a build-time Stop hook. Sharing the checker prevents their behavior from drifting and keeps the strict model-call budget unaffected. The current workspace has no hook configuration or verifier artifact interface, so the wrapper contract must be established when adding the hook.

**Alternatives considered**: Duplicating number parsing inside a hook, making a model call from the hook, or replacing the in-app verifier with a development-only gate are rejected.

## Risks and Release Gates

1. **Specified Gemma identifier/provider support**: verify model availability and structured output with a developer key before integration acceptance; no silent fallback.
2. **Three-report manual acceptance**: obtain three suitable public reports and independently transcribe the required figures; synthetic tests alone do not satisfy this criterion.
3. **PDF table ordering**: validate sorted text/quoted figures against representative statement tables; ambiguous figures remain unavailable rather than guessed.
4. **Model-grounded prose**: constrain the summary call to extracted facts, validate its schema, check numeric claims, and surface a warning instead of retrying.
5. **Stop-hook context**: the workspace has no configured Copilot hook or source/summary artifact convention; confirm its event payload before wiring the deterministic adapter.
