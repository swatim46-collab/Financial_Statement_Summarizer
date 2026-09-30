# Data Model: Financial Statement Summarizer

## Design constraints

- All domain objects are request-scoped; the product persists no report, extraction, summary, ratio, or risk record.
- PDF page references are 1-based physical PDF page numbers, independent of printed page labels.
- Raw report labels, values, currency/units, periods, and source quotes are retained. Parsed numeric values use `Decimal` internally.
- Facts, figures, and risks returned by extraction must include page-level evidence. An invalid quote/page reference invalidates that extracted item; it must not be silently accepted.
- Missing or ambiguous inputs remain missing. The pipeline does not infer values or convert units.

## Request lifecycle

`Received → PDF validated → Page text extracted → Relevant pages selected → Facts extracted and validated → Ratios computed → Summary/explanations written → Narrative numbers verified → Response returned`

Any validation, parsing, provider, or schema failure terminates the request with a user-safe API error. A narrative-number verification failure is the exception: return the generated result with a warning and do not invoke the model again. There is no durable state transition or job record.

## Entities

### AnnualReportRequest

Ephemeral validated upload supplied to `POST /summarise`.

| Field | Type | Rules |
|---|---|---|
| `filename` | string | Metadata only; never trusted as proof of file type. |
| `content_type` | string | Must be `application/pdf` or otherwise pass server-side PDF validation. |
| `content` | bytes/stream | One PDF; maximum 25 MiB; no durable copy. |
| `page_count` | integer | 1–1,200 pages. |

Reject corrupt/non-PDF content, excessive size, or excessive page count before any model call. If no readable text exists on any page, return the exact unreadable-text message and make zero model calls.

### ReportPage

Internal text extracted locally from one physical PDF page.

| Field | Type | Rules |
|---|---|---|
| `page_number` | positive integer | 1-based PDF index. |
| `text` | string | Text-layer extraction only; no OCR. |
| `sections` | list of enum/string | Selector tags such as financial statements, risk factors, business, management discussion, events, or outlook. |
| `selected_for_model` | boolean | True only when page is within the relevance and input-budget rules. |

### ReportEvidence

A supporting passage from the uploaded report, attached to a fact, figure, or risk.

| Field | Type | Rules |
|---|---|---|
| `quote` | string | Non-empty excerpt copied from the extracted page; whitespace-normalized match must occur on the referenced page. |
| `page_number` | positive integer | Must identify an extracted `ReportPage`. |

Evidence is for provenance, not a source of external facts. A model-generated statement with no matching evidence is not eligible for the response.

### ReportFact

A structured qualitative fact used only to write the required summary.

| Field | Type | Rules |
|---|---|---|
| `category` | enum | `business_overview`, `revenue_profit_trend`, `key_event`, or `outlook`. |
| `claim` | string | Concise claim derived from the report; no new numeric claims. |
| `evidence` | `ReportEvidence` | Required; page quote validates locally. |

Facts can be absent when the report does not support a requested topic. The summary must not fill gaps with general knowledge.

### FinancialFigure

A source-reported amount used to calculate a ratio or support a financial trend.

| Field | Type | Rules |
|---|---|---|
| `metric` | enum | `current_assets`, `current_liabilities`, `total_debt`, `current_debt`, `long_term_debt`, `total_equity`, `opening_equity`, `closing_equity`, `net_income`, `revenue`, or `operating_income`. |
| `reported_label` | string | Exact or faithful report line-item label; ambiguous labels are not mapped. |
| `numeric_value` | `Decimal` | Parsed from report evidence; finite value only. Parentheses/negative notation is preserved in the reported value. |
| `reported_value` | string | Original display value and scale as shown in the report. |
| `unit` | string or null | Currency and scale/unit, e.g. the report's own currency/unit; never normalized or converted. |
| `period` | string | Exact fiscal year/period represented. |
| `reporting_scope` | string | Consolidated or entity scope where the report makes it explicit. |
| `evidence` | `ReportEvidence` | Quote/page must support the label and value. |

Select compatible figures from the same reporting entity and applicable period. For debt-to-equity, use explicitly reported total debt or clearly labeled current plus long-term interest-bearing borrowings; do not use total liabilities as a proxy. Return on equity requires source-backed opening and closing equity for the same entity. If these rules cannot be satisfied, the corresponding ratio is unavailable.

### RiskDisclosure

A risk stated by the uploaded report.

| Field | Type | Rules |
|---|---|---|
| `description` | string | Report-grounded wording; may not add an unstated risk or advice. |
| `evidence` | list of `ReportEvidence` | At least one quote and page; include every page needed to support the disclosure. |

No report disclosure means an empty risk list, not an inferred/general risk.

### ExtractionResult

Pydantic-validated output of the first and only extraction invocation.

| Field | Type | Rules |
|---|---|---|
| `facts` | list of `ReportFact` | Only facts from selected source pages. |
| `figures` | list of `FinancialFigure` | Only figures with source evidence and an unambiguous period/unit mapping. |
| `risks` | list of `RiskDisclosure` | Only explicit report disclosures with page evidence. |

Invalid schema or evidence must fail the request; do not issue a corrective model call.

### RatioResult

Deterministic Python result for one of the five required ratios.

| Field | Type | Rules |
|---|---|---|
| `name` | enum | Stable values: `current_ratio`, `debt_to_equity`, `net_profit_margin`, `return_on_equity`, `operating_margin`. |
| `formula` | string | Exact formula used; must match the ratio name and inputs. |
| `inputs` | list of `FinancialFigure` | Source-reported inputs, preserving labels, reported values, units, periods, and pages. |
| `available` | boolean | True only if all required compatible inputs and a non-zero denominator are present. |
| `result` | decimal string or null | Rounded display value (two decimal places); null when unavailable. Internal calculations remain `Decimal`. |
| `result_unit` | enum | `multiple` for current ratio/debt-to-equity; `percent` for the margins and return on equity. |
| `unavailable_reason` | string or null | Non-empty when unavailable; null when available. |
| `explanation` | string | Plain-English explanation from the single summary call; no numeric ratio result is repeated in prose, and any source number used must be verifiable. |

Ratios are returned in the stable order listed above. Calculation formulas:

1. Current ratio = current assets ÷ current liabilities.
2. Debt-to-equity = total debt ÷ total shareholders' equity.
3. Net profit margin = net income ÷ revenue.
4. Return on equity = net income ÷ ((opening equity + closing equity) ÷ 2).
5. Operating margin = operating income ÷ revenue.

Margins and return on equity are multiplied by 100 for percentage display. Inputs must be comparable as reported; no currency or unit conversions are allowed. Zero denominators, insufficient inputs, incompatible units/periods/scope, or ambiguous labels yield unavailable results.

### SummaryDraft

Pydantic-validated output of the second and final model invocation.

| Field | Type | Rules |
|---|---|---|
| `business_overview` | string | Report-supported prose only. |
| `revenue_profit_trends` | string | Report-supported description of trends; no calculated trend percentage unless the number is itself stated in the report. |
| `key_events` | string | Report-supported events only. |
| `outlook` | string | Report-supported outlook only. |
| `ratio_explanations` | map/list keyed by `RatioName` | One plain-English explanation for each ratio; do not repeat computed result values, introduce unsupported numbers, or provide investment advice. Any source numeric claim must use digits copied from the report; do not spell out new numbers in prose. |

The four summary parts are combined into `summary`, with a 350-word maximum as the implementation's one-page guard. If a section is unsupported, leave it empty or state that the report does not provide the information; do not invent content.

### SummaryResponse

Successful public response for the one-page client.

| Field | Type | Rules |
|---|---|---|
| `summary` | string | At most 350 words; covers report-supported required summary topics. |
| `ratios` | list of `RatioResult` | Exactly five entries in stable order. |
| `risks` | list of `RiskDisclosure` | Each item includes `description` and one or more evidence objects pairing a source quote with its page reference. |
| `warnings` | list of strings | Empty when all checks pass; includes extraction truncation or numeric-verification warnings when applicable. |

The verifier scans generated narrative (`summary`, ratio explanations, and risk descriptions). It does not scan structured formula strings, page-number metadata, source quotes/raw input values, or deterministic ratio result fields. On verification failure, include a warning and return the result without another LLM call.

### ApiError

User-safe error response for a failed request: `error.code` is a stable machine-readable string and `error.message` is safe for display. For an unreadable PDF, the message must be exactly `no readable text found, please upload a text-based PDF`. Never include provider credentials, stack traces, raw PDF text, or secret configuration.
