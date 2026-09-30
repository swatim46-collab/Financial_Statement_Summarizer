# Tasks: Financial Statement Summarizer

**Input**: Design documents in `specs/001-financial-statement-summarizer/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/openapi.yaml](contracts/openapi.yaml), and [quickstart.md](quickstart.md).

**Tests**: Tests are included because the project constitution requires deterministic mocked-model coverage, source verification, and ratio acceptance checks. Within each story, test tasks precede implementation tasks.

**Organization**: Tasks are grouped by the three user stories in the specification. Setup and shared prerequisites precede the stories; final acceptance covers cross-cutting checks.

## Phase 1: Setup

**Purpose**: Create the initial backend/frontend scaffolds, safe configuration templates, and consistent requirements.

- [ ] T001 [P] Create root `.gitignore` entries for `backend/.env`, `backend/.venv/`, `frontend/node_modules/`, `frontend/dist/`, Python caches, and test caches; do not ignore `.env.example`.
- [ ] T002 [P] Create root `.env.example` with placeholders only for `GOOGLE_API_KEY`, `MODEL_NAME=gemma-4-26b-a4b-it`, frontend origin, upload/page limits, model-input budget, and model timeout.
- [ ] T003 [P] Create `backend/requirements.txt` with tested Python 3.14.4-compatible FastAPI, Pydantic v2, `pydantic-settings`, PyMuPDF, `python-multipart`, `langchain-google-genai`, pytest, and httpx dependencies; pin versions only after verifying installation and test compatibility.
- [ ] T004 [P] Scaffold the React/TypeScript Vite application in `frontend/`, including `frontend/package.json`, `frontend/package-lock.json`, `frontend/index.html`, `frontend/src/main.tsx`, and `frontend/src/App.tsx`.
- [ ] T005 Configure `frontend/package.json` and `frontend/vite.config.ts` with Vitest, React Testing Library, user-event, and `openapi-typescript`; add `test`, `build`, and `generate:api-types` scripts that generate `frontend/src/types/api.d.ts` from `specs/001-financial-statement-summarizer/contracts/openapi.yaml`.
- [ ] T006 [P] Reconcile the retry wording in `prd.md` and `README.md` with the governing rule: exactly one extraction call and one summary call maximum, with a warning and no retry after numeric-verification failure.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish shared models, parsing, request boundaries, provider injection, and deterministic fixtures before story work.

**⚠️ CRITICAL**: Complete this phase before integrating any user story. Keep uploaded report data request-scoped and model calls injectable for tests.

- [ ] T007 Create `backend/tests/conftest.py` fixtures that generate an in-memory text PDF, an image-only/no-text PDF, a multi-page PDF, and corrupt bytes, plus a fake model adapter that records invocation count and arguments without network access.
- [ ] T008 [P] Add settings validation tests in `backend/tests/unit/test_settings.py` for backend-only `GOOGLE_API_KEY`, configured model name, 25 MiB upload limit, 1,200-page limit, 80,000-character model-input budget, and finite model timeout.
- [ ] T009 [P] Add Pydantic model tests in `backend/tests/unit/test_schemas.py` for source evidence, financial figures, unavailable ratios, the fixed five-ratio order, the 350-word summary limit, and API error shape.
- [ ] T010 [P] Add PDF parser tests in `backend/tests/unit/test_pdf_parser.py` for one-file handling, the 25 MiB/1,200-page limits, page numbering, corrupt/non-PDF rejection, no OCR, and the exact no-text message `no readable text found, please upload a text-based PDF`.
- [ ] T011 [P] Add relevance-selection tests in `backend/tests/unit/test_page_selector.py` proving financial statements, risk sections, and only necessary summary pages are selected within 80,000 characters and unrelated pages are excluded.
- [ ] T012 [P] Add extractor/provider tests in `backend/tests/unit/test_extractor.py` proving structured output is Pydantic-validated, one extraction invocation is made, provider retries are disabled, and provider/schema failures do not trigger a second extraction call.
- [ ] T013 [P] Implement typed backend configuration in `backend/app/settings.py`; load `backend/.env`, keep credentials server-side, and provide defaults for the documented upload/page/model-input limits and timeout.
- [ ] T014 [P] Implement extraction and API Pydantic models in `backend/app/schemas.py`, including validators and schema descriptions for each constrained field. Preserve the following data-model constraints verbatim:
  - AnnualReportRequest: `filename` is “Metadata only; never trusted as proof of file type”; `content_type` “Must be `application/pdf` or otherwise pass server-side PDF validation”; `content` “One PDF; maximum 25 MiB; no durable copy”; `page_count` “1–1,200 pages”.
  - ReportPage: `page_number` “1-based PDF index”; `text` “Text-layer extraction only; no OCR”; `sections` are “Selector tags such as financial statements, risk factors, business, management discussion, events, or outlook”; `selected_for_model` “True only when page is within the relevance and input-budget rules”.
  - ReportEvidence: `quote` “Non-empty excerpt copied from the extracted page; whitespace-normalized match must occur on the referenced page”; `page_number` “Must identify an extracted `ReportPage`”.
  - ReportFact: `category` is one of `business_overview`, `revenue_profit_trend`, `key_event`, or `outlook`; `claim` is “Concise claim derived from the report; no new numeric claims”; `evidence` is required.
  - FinancialFigure: `metric` is one of `current_assets`, `current_liabilities`, `total_debt`, `current_debt`, `long_term_debt`, `total_equity`, `opening_equity`, `closing_equity`, `net_income`, `revenue`, or `operating_income`; `reported_label` is “Exact or faithful report line-item label; ambiguous labels are not mapped”; `numeric_value` is “Parsed from report evidence; finite value only” and “Parentheses/negative notation is preserved in the reported value”; `reported_value` is “Original display value and scale as shown in the report”; `unit` is “Currency and scale/unit, e.g. the report's own currency/unit; never normalized or converted”; `period` is “Exact fiscal year/period represented”; `reporting_scope` is “Consolidated or entity scope where the report makes it explicit”; evidence “Quote/page must support the label and value”.
  - RiskDisclosure: `description` is “Report-grounded wording; may not add an unstated risk or advice”; `evidence` is “At least one quote and page; include every page needed to support the disclosure”.
  - ExtractionResult: facts are “Only facts from selected source pages”; figures are “Only figures with source evidence and an unambiguous period/unit mapping”; risks are “Only explicit report disclosures with page evidence”. Invalid schema/evidence fails the request without a corrective model call.
  - RatioResult: `name` uses the five stable names; `formula` is “Exact formula used; must match the ratio name and inputs”; `inputs` preserve “Source-reported inputs, preserving labels, reported values, units, periods, and pages”; `available` is “True only if all required compatible inputs and a non-zero denominator are present”; `result` is “Rounded display value (two decimal places); null when unavailable”; `result_unit` is `multiple` or `percent`; `unavailable_reason` is “Non-empty when unavailable; null when available”; `explanation` is “Plain-English explanation from the single summary call; no numeric ratio result is repeated in prose, and any source number used must be verifiable”.
  - SummaryDraft: fields are `business_overview`, `revenue_profit_trends`, `key_events`, `outlook`, and `ratio_explanations`; explanations are “One plain-English explanation for each ratio; do not repeat computed result values, introduce unsupported numbers, or provide investment advice. Any source numeric claim must use digits copied from the report; do not spell out new numbers in prose”.
  - SummaryResponse: `summary` is “At most 350 words”; `ratios` are “Exactly five entries in stable order”; `risks` include “description, source quote(s), and page reference(s)”; `warnings` are “Empty when all checks pass; includes extraction truncation or numeric-verification warnings when applicable”.
  - ApiError: `error.code` is stable; `error.message` is safe to display; for an unreadable PDF, the message is exactly `no readable text found, please upload a text-based PDF`; never include provider credentials, stack traces, raw PDF text, or secret configuration.
- [ ] T015 Implement `backend/app/pdf_parser.py` using PyMuPDF; accept “One PDF; maximum 25 MiB; no durable copy”, enforce “1–1,200 pages”, retain “1-based PDF index” page numbers, extract “Text-layer extraction only; no OCR”, and close upload resources on all paths.
- [ ] T016 Implement `backend/app/page_selector.py` with local heading/keyword selection for financial statements, risk factors, business overview, management discussion, events, and outlook; set `selected_for_model` true only when the page is within relevance and input-budget rules, enforce 80,000 characters, and warn when relevant passages are truncated.
- [ ] T017 Implement the FastAPI app and `GET /health` in `backend/app/main.py`; return `{"status":"ok"}` without a model call, configure local frontend CORS, and map validation/provider failures to the documented user-safe JSON error envelope.
- [ ] T018 Implement the Google model adapter and one-call structured extraction in `backend/app/extractor.py`; use only selected page text, attach page evidence to facts/figures/risks, set provider `max_retries=0`, and fail without a corrective model call.
- [ ] T019 Implement request-scoped orchestration and a two-call budget in `backend/app/pipeline.py`; allow at most one extraction invocation plus one summary invocation, expose injectable fake adapters, and return a controlled error rather than retrying malformed/provider responses.

**Checkpoint**: Settings, schemas, page extraction/selection, provider adapter, health endpoint, and test doubles are ready. No user result is persisted.

---

## Phase 3: User Story 1 - Review an annual report summary (Priority: P1) 🎯 MVP

**Goal**: Let a reader submit one readable report and receive a source-grounded one-page summary through a single-page upload/loading/results flow.

**Independent Test**: Submit a generated readable PDF using fake model responses; verify the four supported summary topics, 350-word limit, exact source-number checks, exact two-call maximum, and one-page UI states without any live provider request.

### Tests for User Story 1

- [ ] T020 [P] [US1] Add `backend/tests/integration/test_summary_flow.py` for a successful multipart request, supported summary sections, no full-report model input, a maximum of two model invocations, and warning/no-retry behavior after an unsupported number.
- [ ] T021 [P] [US1] Add `backend/tests/unit/test_verifier.py` for source-quote/page matching, grouping separators, decimals, Unicode minus/accounting parentheses, numeric words, computed-result metadata exclusions, and warning output without another model call.
- [ ] T022 [P] [US1] Add `frontend/src/tests/upload-summary.test.tsx` covering single-PDF selection, file rejection feedback, loading/disabled state, summary rendering, and API error display.

### Implementation for User Story 1

- [ ] T023 [P] [US1] Implement source-quote and narrative-number verification in `backend/app/verifier.py`; enforce that each evidence quote is a “Non-empty excerpt copied from the extracted page; whitespace-normalized match must occur on the referenced page”, verify generated summary numbers against locally extracted report text without converting currency/scale, do not scan structured formula strings, page-number metadata, source quotes/raw input values, or deterministic ratio result fields, and return a warning instead of triggering another model call.
- [ ] T024 [P] [US1] Implement the single structured summary call in `backend/app/summariser.py`; cover business overview, revenue/profit trends, key events, and outlook, enforce `summary` “At most 350 words”, use only selected report facts, and do not generate outside data or investment advice.
- [ ] T025 [US1] Integrate extraction, summary generation, and verification in `backend/app/pipeline.py`; call each model stage at most once and keep the response contract stable with five explicitly unavailable ratio entries and an empty risk list until their story phases are complete, without presenting placeholders as calculated results.
- [ ] T026 [US1] Implement `POST /summarise` in `backend/app/main.py` for exactly one multipart field named `file`; enforce size/page limits, return the exact unreadable-text message for PDFs without text, and use the OpenAPI success/error response shapes.
- [ ] T027 [US1] Implement `frontend/src/api/client.ts` to send the selected file as browser `FormData` to `POST /summarise`, avoid manually setting multipart `Content-Type`, read the typed JSON error envelope, and use no secret frontend configuration.
- [ ] T028 [US1] Implement the one-page upload/loading/results flow in `frontend/src/App.tsx`, `frontend/src/components/UploadForm.tsx`, `frontend/src/components/SummaryView.tsx`, and `frontend/src/styles.css`; display the summary and warnings and keep file selection single-report only.

**Checkpoint**: A reader can independently upload a readable report and review a source-verified one-page summary. No OCR, account, or comparison feature is introduced.

---

## Phase 4: User Story 2 - Understand the financial ratios (Priority: P1)

**Goal**: Show all five deterministic ratios with formulas, source inputs, results or unavailable reasons, and plain-English explanations.

**Independent Test**: Provide source-backed fixture figures directly to the ratio module and compare every available result with manual calculations; verify unavailable results for missing/ambiguous inputs, zero denominators, and incompatible units/periods.

### Tests for User Story 2

- [ ] T029 [P] [US2] Add `backend/tests/unit/test_ratios.py` with manual expected results for current ratio, debt-to-equity, net profit margin, return on equity, and operating margin; cover missing values, zero denominators, ambiguous debt, incompatible scope/period/units, and two-decimal half-up rounding.
- [ ] T030 [P] [US2] Add `backend/tests/integration/test_ratio_response.py` to assert the API returns exactly five stable-ordered ratio entries with formula, raw report inputs, page provenance, result or unavailable reason, and no more than two model invocations.
- [ ] T031 [P] [US2] Add `frontend/src/tests/ratio-list.test.tsx` for available and unavailable ratios, formulas, reported inputs/units/pages, result units, explanations, and unavailable reasons.

### Implementation for User Story 2

- [ ] T032 [US2] Implement pure `Decimal` calculations in `backend/app/ratios.py` using the exact formulas: current assets ÷ current liabilities; total debt ÷ total shareholders' equity; net income ÷ revenue; net income ÷ average opening/closing shareholders' equity; operating income ÷ revenue. Require compatible periods/scope/units and a non-zero denominator, do not substitute total liabilities for debt or convert units, and set `result` to “Rounded display value (two decimal places); null when unavailable” with a “Non-empty when unavailable; null when available” reason.
- [ ] T033 [US2] Update `backend/app/pipeline.py` and `backend/app/summariser.py` so `ratio_explanations` contains “One plain-English explanation for each ratio”; “do not repeat computed result values, introduce unsupported numbers, or provide investment advice”; and “Any source numeric claim must use digits copied from the report; do not spell out new numbers in prose.” Calculate ratios before the single summary call, keep results in structured fields, and do not add a third invocation.
- [ ] T034 [P] [US2] Implement `frontend/src/components/RatioList.tsx` and `frontend/src/components/RatioCard.tsx` to display exactly five ratios with formulas, source inputs and page references, available results/units or unavailable reasons, and explanations.
- [ ] T035 [US2] Integrate the ratio list into `frontend/src/App.tsx` and the final API response assembly in `backend/app/pipeline.py`; replace every temporary placeholder with its calculated `RatioResult`, including a non-empty unavailable reason whenever the result is unavailable.

**Checkpoint**: Ratio results are deterministic, manually testable, source-backed, and shown beside their inputs without currency/unit conversion.

---

## Phase 5: User Story 3 - Review stated risks and warnings (Priority: P2)

**Goal**: Show only disclosed report risks with their evidence pages and visibly report unavailable or failed verification outcomes.

**Independent Test**: Feed selected page fixtures containing known and absent risk disclosures; verify only source-backed risk descriptions appear with matching 1-based page citations, and verification failures create warnings without another model call.

### Tests for User Story 3

- [ ] T036 [P] [US3] Add `backend/tests/unit/test_risk_evidence.py` for non-empty risk evidence, quote-to-page matching, 1-based page references, empty risk lists, and rejection of unsupported risk claims.
- [ ] T037 [P] [US3] Add `backend/tests/integration/test_risk_warning_response.py` to prove risks are report-stated, each quote/page pair is returned, invalid narrative numbers create a warning, and summary generation is never repeated.
- [ ] T038 [P] [US3] Add `frontend/src/tests/risk-warning-list.test.tsx` for report-page citations, empty risk results, warning visibility, and unavailable-ratio messaging on the same page.

### Implementation for User Story 3

- [ ] T039 [P] [US3] Extend `backend/app/verifier.py` to enforce that `RiskDisclosure.evidence` contains “At least one quote and page”, validate every quote against its cited extracted page, and scan risk descriptions for unsupported numbers without treating page metadata as narrative.
- [ ] T040 [US3] Map validated extraction risks and verification warnings into `backend/app/pipeline.py` using the OpenAPI `evidence` shape; return an empty risk list when no disclosure is found and never add outside risks or a retry.
- [ ] T041 [P] [US3] Implement `frontend/src/components/RiskList.tsx` and `frontend/src/components/Warnings.tsx`, then render them from `frontend/src/App.tsx` with accessible labels, report page references, and clear warning text.

**Checkpoint**: Readers can verify each listed risk in the report, and failures/unavailable values are visible without invented content or additional model calls.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify the full contract, provider readiness, stop-hook gate, usability target, and release acceptance evidence.

- [ ] T042 [P] Add `backend/tests/integration/test_api_contract.py` to verify `/health` and `/summarise` response fields and constraints against `specs/001-financial-statement-summarizer/contracts/openapi.yaml`; regenerate `frontend/src/types/api.d.ts` and require `npm run build` to pass after contract changes.
- [ ] T043 Implement `backend/scripts/verify_summary.py`, `.github/hooks/summary-verification.json`, and `backend/tests/integration/test_stop_hook.py`; have the Copilot Stop hook invoke the shared verifier for explicit source/summary artifacts (or the verifier test gate if the hook event cannot provide them), make no model calls, and do not duplicate numeric matching logic.
- [ ] T044 [P] Run a developer-key smoke test for `gemma-4-26b-a4b-it` and structured extraction using `backend/.env`; record availability, structured-output compatibility, and disabled retries in `specs/001-financial-statement-summarizer/research.md`. If unsupported, stop and obtain product-owner approval for a model change; do not silently fall back.
- [ ] T045 [P] Validate all available ratios manually against three approved public annual-report PDFs and record the source page, period, reported label/value/unit, manual result, and application result in `specs/001-financial-statement-summarizer/quickstart.md`; use the permitted Fetch workflow or user-approved fixtures, and do not use external report data in generated user results.
- [ ] T046 [P] Run a first-time-user check with at least 10 participants and record the completion rate in `specs/001-financial-statement-summarizer/quickstart.md`; meet the specification's 90% target (at least 9 of 10 users locate the summary, ratios, risks, and warnings without navigating to another page).
- [ ] T047 Run every automated scenario and command in `specs/001-financial-statement-summarizer/quickstart.md`, including `pytest`, `npm run test`, `npm run build`, and secret/config checks for `.gitignore` and `.env.example`; fix failures in the corresponding `backend/`, `frontend/`, and root configuration files.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies; tasks T001–T004 and T006 can run in parallel. T005 follows the Vite scaffold in T004.
- **Foundational (Phase 2)**: Begins after setup. T008–T012 are independent test files after T007 provides shared fixtures. Implementations follow their relevant tests and block all story integration.
- **User Stories (Phases 3–5)**: Start after the foundational API, schemas, parser, selector, extractor, and call-budget implementation.
- **Polish (Phase 6)**: Runs after all user stories; provider smoke test and three-report manual validation are acceptance gates.

### User Story Dependencies

- **US1 (P1)**: Depends on Setup and Foundation; establishes the user-facing summary flow and is the suggested MVP.
- **US2 (P1)**: Ratio calculations and UI can be developed/tested independently after Foundation. Final response integration depends on the shared `/summarise` skeleton from US1.
- **US3 (P2)**: Risk evidence validation and UI can be developed/tested independently after Foundation. Final response integration depends on the shared `/summarise` skeleton from US1.
- US2 and US3 can parallelize their isolated calculations/evidence and components; changes to shared `backend/app/pipeline.py` and `frontend/src/App.tsx` should be integrated sequentially to avoid conflicts.

### Parallel Opportunities

- Setup: T001–T004 and T006 touch separate files; T005 follows T004.
- Foundation tests: T008–T012 touch separate test files and can be authored in parallel after T007.
- US1: T020–T022 are separate backend/frontend test files; T023 and T024 are separate modules after Foundation.
- US2: T029–T031 are separate test files; `backend/app/ratios.py` and `frontend/src/components/RatioCard.tsx` can be implemented independently after their tests.
- US3: T036–T038 are separate test files; risk rendering can be developed separately from the verifier extension.
- Polish: T044–T046 are independent acceptance activities once all stories are integrated.

## Parallel Example: User Story 1

After Phase 2 is complete, run these independent workstreams in parallel:

- Backend verification: T021 → T023 in `backend/app/verifier.py`.
- Summary generation: T020 → T024 in `backend/app/summariser.py`.
- Frontend behavior: T022 → T027/T028 in `frontend/src/`.

Integrate the completed work through T025 and T026 after the parallel workstreams finish.

## Parallel Example: User Story 2

After the US1 response contract exists, run T029, T030, and T031 together. Then implement the pure ratio module in T032 and the ratio UI in T034 independently; integrate both through T033 and T035.

## Parallel Example: User Story 3

After the US1 response contract exists, run T036, T037, and T038 together. Implement evidence validation in T039 and risk/warning UI in T041 independently; integrate through T040.

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Phase 1 Setup and Phase 2 Foundation.
2. Complete Phase 3 US1: one report upload, summary generation, source-number checks, and one-page results UI.
3. Validate the MVP independently with mocked model calls, including unreadable-PDF handling and the two-call maximum.
4. Continue with US2 and US3 before declaring full v1 acceptance; the final product requires all five ratios and report-cited risks.

### Incremental Delivery

1. Setup and Foundation establish PDF provenance, typed contracts, and mocked model boundaries.
2. US1 delivers the source-grounded summary journey.
3. US2 adds deterministic ratios without adding a model call.
4. US3 adds disclosed risks and page references; warning behavior stays in the same response.
5. Polish verifies model readiness, three-report manual accuracy, the Stop hook, usability, and all quickstart checks.
