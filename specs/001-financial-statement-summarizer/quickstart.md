# Quickstart and Validation: Financial Statement Summarizer

This guide is for implementing and validating the planned v1 application. The workspace currently
contains planning artifacts only; commands become runnable after `/speckit-implement` creates the
backend and frontend.

## Prerequisites

- Linux development environment.
- Python 3.14.4, or the explicitly approved compatible Python target selected during dependency
  installation checks.
- Node.js 22.22.2 or a compatible Vite-supported Node release, npm 10.9.7.
- A Google AI Studio API key for a manual model smoke test and interactive demo. Use a placeholder
  in `.env.example`; keep the actual key in backend `.env` only.
- Confirmation that `gemma-4-26b-a4b-it` is available to the configured AI Studio account and
  supports the selected structured-output mode. Do not substitute another model without approval.

## Start the backend

From the repository root:

```sh
cd backend
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp ../.env.example .env
```

Set `GOOGLE_API_KEY` and `MODEL_NAME=gemma-4-26b-a4b-it` in `backend/.env`. The root
`.env.example` contains placeholders only. Do not print the key, commit the backend `.env`, or add
the key to frontend/Vite environment settings. Add the chosen local frontend origin and any
documented request limits as non-secret backend settings.

Start the API:

```sh
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Check liveness at `http://127.0.0.1:8000/health`; expected JSON is `{"status":"ok"}`. The health
check does not call the model. API details and response schemas are in
[contracts/openapi.yaml](contracts/openapi.yaml).

## Start the frontend

In another terminal from the repository root:

```sh
cd frontend
npm ci
npm run generate:api-types
npm run dev -- --host 127.0.0.1
```

Open the Vite URL printed in the terminal. Configure only a non-secret API base URL for the local
backend. Select one text-based annual-report PDF and submit it. The same page should show the
loading state, then the summary, five ratios, risks, and any warnings. Browser `FormData` should
set the multipart boundary; do not manually set the request `Content-Type` header.

## Automated validation

Backend tests (run from `backend/` with its virtual environment active):

```sh
pytest
```

Frontend tests and production build (run from `frontend/`):

```sh
npm run test
npm run build
```

Tests must mock all model invocations. The deterministic suite should cover:

- Manual expected results for all five ratios, plus missing figures, ambiguous mappings, zero
  denominators, incompatible periods/units, and decimal rounding.
- PDF opening, page numbering, malformed/non-PDF rejection, the upload/page limits, and the exact
  unreadable-PDF message.
- Relevance page selection and proof that the complete report is not passed to the model.
- Evidence quote/page validation, risk page references, source-number verification, and warning
  behavior.
- The hard maximum of two provider invocations, including invalid model output, provider errors,
  and verification failure; automatic SDK retries must be disabled.
- `/health`, multipart `/summarise`, success/error response shapes, and generated frontend types.
- Frontend single-page file selection, disabled/loading behavior, summary/ratio/risk/warning
  rendering, and actionable error display.

## End-to-end API smoke test

With the API running and an approved text-based public annual report available locally:

```sh
curl -sS -F "file=@/path/to/annual-report.pdf;type=application/pdf" \
  http://127.0.0.1:8000/summarise
```

Expected success is HTTP 200 JSON with `summary`, exactly five `ratios`, `risks`, and `warnings`.
Each ratio includes its formula, source inputs and pages, result or unavailable reason, result
unit, and explanation. Each risk includes source text and page reference(s). Narrative numeric
claims must match source text; a verification failure returns a warning without summary retry.

Also validate error paths with a non-PDF/corrupt file and a PDF generated in tests with no text
layer. The latter must return an error whose display message is exactly:

`no readable text found, please upload a text-based PDF`

It must cause zero model calls.

## Three-report manual ratio acceptance

Before declaring ratio acceptance complete, validate three representative public annual reports:

1. Record the source page, reporting period, label, raw value, currency/unit/scale, and reporting
   scope for each required ratio input.
2. Calculate all available ratios independently using the formulas in [data-model.md](data-model.md).
3. Compare values at the same two-decimal display precision used by the application.
4. Confirm unavailable ratios are not guessed, source values are not converted, and each result's
   inputs and page references are traceable.
5. Keep external report figures exclusively in test/acceptance materials; never use them in a
   user's generated result.

The workspace currently has no three-report fixture set and its configured MCP server does not
provide Fetch. Do not claim this manual acceptance criterion has passed until the permitted
public-report test materials are available and reviewed.
