# Feature Specification: Financial Statement Summarizer

**Feature Branch**: not created

**Created**: 2026-09-30

**Status**: Draft

**Input**: User description: "Upload one public annual-report PDF to receive a one-page summary, five financial ratios, and risks stated in the report."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Review an annual report summary (Priority: P1)

As a reader of annual reports, I want to submit one report and receive a concise summary, so I
can understand the business, its financial trends, key events, and outlook without reading the
entire report.

**Why this priority**: The summary is the primary user outcome and makes the report easier to
understand.

**Independent Test**: Submit a readable public annual report and verify that the returned
summary covers the requested topics, fits one page, and contains only source-supported facts.

**Acceptance Scenarios**:

1. **Given** a readable annual-report PDF, **When** the user submits it, **Then** the user can
   review a one-page summary covering the business overview, revenue and profit trends, key
events, and outlook.
2. **Given** a summary containing numeric claims, **When** each claim is checked against the
   uploaded report, **Then** every non-computed number is present in the source text.
3. **Given** a PDF with no readable text, **When** the user submits it, **Then** the exact message
   `no readable text found, please upload a text-based PDF` is shown.
4. **Given** a user without an account, **When** they submit one report, **Then** they can
   complete the report review without creating an account.
5. **Given** a user wants to review multiple reports together, **When** they submit more than one
   report for a single analysis, **Then** the product explains that v1 supports one report per
   analysis.
6. **Given** the report contains unrelated sections, **When** the summary is prepared, **Then**
   its claims are limited to facts relevant to the requested summary topics.

---

### User Story 2 - Understand the financial ratios (Priority: P1)

As a reader reviewing a company's financial position, I want to see five ratios with their
formulas, report inputs, results, and plain-English explanations, so I can understand how each
result relates to figures in the report.

**Why this priority**: Ratios make key financial relationships easier to compare while keeping
the underlying report figures visible.

**Independent Test**: Use a report with the required source figures, calculate the ratios
independently, and compare each displayed formula, input, and result.

**Acceptance Scenarios**:

1. **Given** a report with sufficient source figures, **When** ratios are produced, **Then** the
   results include current ratio, debt-to-equity, net profit margin, return on equity, and
   operating margin, each with its formula, source inputs, result, and plain-English explanation.
2. **Given** a report missing an input or containing a zero denominator or incompatible units,
   **When** the affected ratio is evaluated, **Then** it is marked unavailable with a reason and
   no value is guessed or converted.
3. **Given** a ratio input shown in the results, **When** the reader checks its provenance,
   **Then** its reported label, period, units, and source page can be identified.

---

### User Story 3 - Review stated risks and warnings (Priority: P2)

As a reader assessing the report's disclosures, I want to review only risks stated by the report
and see where each risk appears, so I can verify the disclosure in context.

**Why this priority**: Page references and source-only risks prevent unsupported claims and let
readers check the original disclosure.

**Independent Test**: Submit a report with risk disclosures and confirm that each displayed risk
matches a source passage and includes its PDF page reference; confirm that no extra risks are
added.

**Acceptance Scenarios**:

1. **Given** a report containing risk disclosures, **When** risks are listed, **Then** each item
   is stated in the report and includes its source page reference.
2. **Given** a report with no identified risk disclosures, **When** the results are shown,
   **Then** no risk is invented.
3. **Given** a numeric claim that cannot be verified against the report, **When** results are
   returned, **Then** the user sees a warning identifying the verification issue and summary
generation is not repeated.

### Edge Cases

- An image-only PDF with no readable text produces the exact unreadable-text message and is
  unsupported.
- A partially readable report may support some summary topics or ratios but not others. Unsupported
  topics are omitted or identified as unavailable; unsupported figures are not inferred.
- A missing figure, zero denominator, or incompatible reporting period or unit makes the affected
  ratio unavailable with a clear reason.
- When the report provides no risk disclosure that can be identified, the risk list remains empty
  rather than being supplemented with general or external risks.
- If source-number verification fails, the user receives a warning without a second attempt to
  regenerate the summary.
- A corrupt or non-PDF upload is rejected with a clear, actionable message.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The product MUST accept one public annual report in PDF format per request without
  requiring a user account.
- **FR-002**: The product MUST display `no readable text found, please upload a text-based PDF`
  when a submitted PDF has no readable text. PDFs without readable text are unsupported.
- **FR-003**: The product MUST provide a one-page summary covering the report's business overview,
  revenue and profit trends, key events, and outlook when those topics are supported by the
  report.
- **FR-004**: Every numeric claim in the summary and ratio explanations MUST be supported by the
  uploaded report, except for computed ratio results whose inputs are source-backed.
- **FR-005**: The product MUST provide current ratio, debt-to-equity, net profit margin, return on
  equity, and operating margin. Each ratio MUST include its formula, source inputs, result or
  unavailable status, and plain-English explanation.
- **FR-006**: The product MUST preserve the report's currency, units, labels, and periods without
  conversion. It MUST mark a ratio unavailable when its inputs cannot be compared as reported.
- **FR-007**: The product MUST list only risks stated in the uploaded report and MUST provide a
  source page reference for each listed risk. It MUST NOT provide outside data or investment
  advice.
- **FR-008**: The product MUST return a warning when a narrative number cannot be verified against
  the report. It MUST NOT repeat summary generation after a verification failure.
- **FR-009**: The upload, loading, and results experience MUST remain in one page. The results
  MUST present the summary, ratios, risks, and any warnings.
- **FR-010**: Analysis MUST use financial-statement and risk-factor content plus only the minimum
  report passages needed for the requested summary. Unrelated report content and external sources
  MUST NOT contribute to a user's results.
- **FR-011**: The v1 user journey MUST support one report at a time and MUST NOT require account
  creation or compare multiple reports. Features outside the v1 scope require an approved feature
  specification.

### Key Entities *(include if feature involves data)*

- **Annual Report**: The single public PDF supplied by the user, with readable content organized
  by PDF page.
- **Report Figure**: A report-stated financial value with its label, period, currency or unit, and
  source page.
- **Summary**: A concise account of the report's business, financial trends, key events, and
  outlook.
- **Ratio Result**: A named ratio with formula, report figures used as inputs, result or
  unavailable reason, and explanation.
- **Risk Disclosure**: A risk stated in the report and its source page reference.
- **Warning**: A notice that a requested result is unavailable or a narrative number could not be
  verified.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In the acceptance test set, 100% of numeric claims in summaries and ratio
  explanations match source text, except computed ratio results; every computed result has
  source-backed inputs.
- **SC-002**: Across three representative annual reports, every available ratio matches an
  independent manual calculation at the displayed precision, and every unsupported ratio is
  marked unavailable.
- **SC-003**: In the acceptance test set, 100% of displayed risks match a report disclosure and
  include a PDF page reference; no unsupported risk is displayed.
- **SC-004**: In every test using a PDF with no readable text, the product displays the exact
  required unreadable-text message.
- **SC-005**: The summary fits within one page while covering all report-supported required
topics.
- **SC-006**: At least 90% of first-time users in a usability check can upload a readable report
  and locate the summary, ratios, risks, and warnings without navigating to another page.
- **SC-007**: In every verification-failure test, the product displays a warning and does not
  regenerate the summary.

## Assumptions

- A supported input is one public annual report in PDF format with readable text. Image-only PDFs
  are unsupported and receive the specified unreadable-text message.
- Page references use the 1-based PDF page number, even if the report's printed page numbering
  differs.
- For ratios, the most recently completed fiscal year presented is used when its inputs are
  available. Earlier periods may be used to describe trends or calculate average equity.
- Standard formulas are used: current assets divided by current liabilities; total debt divided
  by total shareholders' equity; net income divided by revenue; net income divided by average
  shareholders' equity for return on equity; and operating income divided by revenue for
  operating margin. Total liabilities are not substituted for total debt. Current ratio and
  debt-to-equity are displayed as multiples; margins and return on equity are displayed as
  percentages.
- Return on equity requires source-backed opening and closing equity figures to calculate average
  equity. If the required values or an unambiguous report line item are missing, the ratio is
  unavailable rather than inferred.
- If source values use incompatible currencies, units, or periods and cannot be compared as
  reported, the affected ratio is unavailable; no conversion is performed.
- Current project governance resolves the PRD's conflicting retry note: failed number
  verification produces a warning without regenerating the summary.
- The 90% first-time task-completion target is an initial usability benchmark and may be revised
  after representative users are tested.
