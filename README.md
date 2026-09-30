# Financial Statement Summarizer

An application planned to turn public annual-report PDFs into a concise, source-grounded summary, five financial ratios, and report-cited risks.

> **Status:** This repository currently contains the product requirements and planned architecture. The application has not been implemented yet.

## Planned Features

- Upload one text-based annual report in PDF format.
- Generate a one-page summary covering the business, revenue and profit trends, key events, and outlook.
- Calculate five financial ratios with formulas, source inputs, results, and plain-English explanations:
  - Current ratio
  - Debt-to-equity
  - Net profit margin
  - Return on equity
  - Operating margin or interest coverage
- List only risks stated in the report, with page references.
- Verify that every number in the written summary appears in the source text; retry summary generation once if verification fails.

## How It Is Planned to Work

1. The React frontend uploads a PDF to the FastAPI backend.
2. PyMuPDF extracts text and page numbers. Scanned PDFs without a readable text layer are rejected; OCR is out of scope for v1.
3. A LangChain pipeline uses the configured Google AI Studio model to extract report figures and risks, calculate ratios in Python, and draft the summary.
4. A Python verifier checks summary numbers against the source text before results are returned to the frontend.

The language model extracts information and writes explanations; Python performs the ratio calculations and source-number checks.

## Planned Technology

| Area | Technology |
| --- | --- |
| Backend API | Python, FastAPI |
| Frontend | React, TypeScript, Vite |
| PDF parsing | PyMuPDF |
| LLM integration | LangChain, `langchain-google-genai` |
| Model | `gemma-4-26b-a4b-it` via Google AI Studio |

## Planned API

- `POST /summarise` — accepts a PDF upload and returns a summary, ratios, risks, and warnings as JSON.
- `GET /health` — reports API health.

## Scope and Safeguards

- Text-based public annual reports only; no OCR in v1.
- Use the currency and units shown in the report; do not convert them.
- Do not use outside data or provide investment advice.
- Risks must be stated in the report and include page references.
- Keep the Google AI Studio API key in the backend `.env` file; never commit secrets.

## Development

Implementation and setup instructions will be added as the backend and frontend are built. See [prd.md](prd.md) for the full product requirements and planned project layout.
