# Implementation Plan: Financial Statement Summarizer

**Branch**: `001-financial-statement-summarizer` | **Date**: 2026-09-30 | **Spec**: [spec.md](spec.md)

**Input**: [spec.md](spec.md), [prd.md](../../../prd.md), and [constitution.md](../../../.specify/memory/constitution.md)

## Summary

Deliver a single-page annual-report review for one readable public PDF per request. The React
client uploads one PDF to FastAPI. The backend extracts page-numbered text locally, sends only
selected financial-statement, risk, and summary-supporting pages to the configured Google model
for structured extraction, calculates the five required ratios in deterministic Python, then
makes one summary call. Pydantic validates extracted and returned data; Python verifies narrative
numbers against the full locally extracted source and returns warnings on failure without a
retry. The UI presents the one-page summary, ratio formulas and evidence, report-stated risks,
and warnings. No accounts, persistence, OCR, background jobs, or report comparison are included.

## Technical Context

**Language/Version**: Python 3.14.4 (active local interpreter); TypeScript with Node.js 22.22.2 and npm 10.9.7 (observed local toolchain). Pin tested dependencies and verify Python wheel compatibility before implementation is considered ready.

**Primary Dependencies**: FastAPI, Pydantic v2, `pydantic-settings`, PyMuPDF, `python-multipart`, LangChain `langchain-google-genai`, React, TypeScript, Vite, and `openapi-typescript` for generated frontend types. Use `pytest` and `httpx` for backend tests; Vitest and React Testing Library for frontend tests.

**Storage**: No persistent storage or database. Process upload bytes, extracted pages, facts, and results only for the request; close/clean up upload resources at request completion. Do not log report text or secrets.

**Testing**: `pytest` with FastAPI `TestClient` and mocked model adapters; Vitest/React Testing Library for upload, loading, success, and error states; frontend production build and type-check. Keep the three-report manual calculation check as an explicit acceptance activity.

**Target Platform**: Local Linux development with the API and Vite dev server running separately in a modern browser. Production hosting and deployment are outside this feature plan.

**Project Type**: Small full-stack web application: one React page and one synchronous FastAPI service.

**Performance Goals**: No external latency SLO is specified. Bound requests to one PDF, at most 25 MiB and 1,200 pages, at most 80,000 characters of selected model input, and at most two model invocations. Use a finite 60-second timeout per invocation; show a loading state while processing.

**Constraints**: Only one text-based PDF per request; no OCR, accounts, database, background jobs, or multi-report comparison. The model sees selected page text only, never the full report or external search results. Disable provider retries and summary retries. Keep `GOOGLE_API_KEY` on the backend. Preserve source currency, units, labels, periods, and 1-based PDF page references.

**Scale/Scope**: One report analysis at a time, one response with `summary`, exactly five ratio entries, report-cited `risks`, and `warnings`; one-page upload/loading/results experience.

## Constitution Check

*GATE: Checked before Phase 0 research and re-checked after Phase 1 design.*

| Gate | Design evidence | Result |
|---|---|---|
| Source-grounded output | Each extracted figure/fact/risk retains its source quote and PDF page; only selected report passages go to the model; narrative number verification runs locally before response. | PASS |
| Deterministic calculations | Python `Decimal` calculations implement the five fixed formulas; incompatible/missing values and zero denominators yield unavailable ratios, not estimates. | PASS |
| Minimal, bounded processing | One PDF, no OCR or persistent data; one extraction and one summary invocation at most; provider-level automatic retries disabled. | PASS |
| Server-side secrets | Backend settings read `GOOGLE_API_KEY` from `.env`; frontend receives only a non-secret API base URL; `.env.example` contains placeholders. | PASS |
| Contracts and evidence | Pydantic response models define the API; an OpenAPI contract drives/generated frontend types; deterministic tests mock calls and cover provenance, warnings, and the call cap. | PASS |
| PRD completion hook | The app verifier is the single source of truth; a Copilot Stop hook calls its deterministic wrapper/tests and makes no model calls. Every live request is still checked inside the app. | PASS |
| Scope and no-retry conflict | Follow the stricter constitution/instructions: a verification failure returns a warning and never triggers a third call. The PRD/README retry wording remains a documentation reconciliation item, not an implementation exception. | PASS |

**Pre-design external readiness notes**: The specified model identifier and its structured-output
support must pass a one-time developer-key smoke test; the inspected Google model catalogue did
not visibly list the PRD's Gemma identifier. Do not silently substitute another model. The
workspace currently has no annual-report test fixtures or configured Fetch MCP, so three-report
manual acceptance requires approved public sample reports to be made available. These are
release-readiness checks, not unresolved architecture choices or constitution violations.

## Project Structure

### Documentation (this feature)

```text
specs/001-financial-statement-summarizer/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── openapi.yaml
├── checklists/
│   └── requirements.md
└── tasks.md                         # created in Phase 2 by /speckit-tasks
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── main.py                      # FastAPI app, /health, /summarise
│   ├── settings.py                  # backend-only environment settings
│   ├── schemas.py                   # Pydantic extraction and API models
│   ├── pipeline.py                  # one-request orchestration and call budget
│   ├── pdf_parser.py                # local text extraction and page numbers
│   ├── page_selector.py             # relevant-page keyword selection and budget
│   ├── extractor.py                 # one structured extraction call
│   ├── ratios.py                    # pure Decimal ratio calculations
│   ├── summariser.py                # one summary/explanation call
│   └── verifier.py                  # source quote and numeric verification
├── scripts/
│   └── verify_summary.py            # CLI adapter for the shared verifier/Stop hook
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py                  # generated text-PDF and mocked model fixtures
└── requirements.txt

frontend/
├── src/
│   ├── App.tsx                      # single-page upload/loading/results flow
│   ├── api/client.ts                # multipart request and typed errors
│   ├── types/api.d.ts               # generated from OpenAPI contract
│   ├── components/                  # upload, summary, ratio, risk, warning views
│   ├── tests/                        # mocked API interaction tests
│   └── styles.css
├── package.json
├── package-lock.json
└── vite.config.ts

.env.example                             # root placeholders only, per PRD
.gitignore                               # excludes backend/.env and generated artifacts
.github/hooks/summary-verification.json # Stop hook invoking the deterministic verifier gate
```

**Structure Decision**: Use the backend/frontend split and module names from the PRD, adding
small focused modules for settings, page selection, and the request pipeline. Keep ratio math
and verification isolated and pure where possible. The repository currently contains planning
documents only; the paths above are the planned implementation layout.

## Complexity Tracking

No constitution violations or additional architectural projects are proposed. The two-process
development setup is required by the chosen React frontend and FastAPI API; no database,
queue, or deployment stack is introduced.

**Post-design recheck**: PASS. The generated data model, OpenAPI contract, and quickstart preserve
the source-grounding, deterministic-ratio, two-call, secret-handling, and v1-scope gates above.
