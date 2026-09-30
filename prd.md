PRD: Financial Statement Summariser

Platform: GitHub Copilot | Level: Intermediate | Approach: Keep it simple

What it is

Upload a public annual report (PDF). Get back a one-page summary, five financial ratios with explanations, and the risks the report itself mentions.

Input
One annual public report, PDF only.
Scanned PDFs with no text layer: show "no readable text found, please upload a text-based PDF". (No OCR in v1.)
Output
One-page summary: business overview, revenue and profit trends, key events, outlook.
Five ratios, each with formula, inputs from the report, result, and a plain-English explanation:
Current ratio
Debt-to-equity
Net profit margin
Return on equity
Operating margin or interest coverage
Risks from the report: only risks the report itself states, with page references.
Rules
Use the currency and units shown in the PDF. No conversion.
No outside data, no investment advice.
Every number in the summary must appear in the source text.
Tech stack
Layer	Choice
Backend	Python
API	FastAPI
Frontend	React + TypeScript (Vite)
LLM	gemma-4-26b-a4b-it via Google AI Studio (free tier API key)
Framework	LangChain (langchain-google-genai)
PDF parsing	PyMuPDF
Architecture
React UI --upload PDF--> FastAPI --> PDF parser (text + page numbers)
                                          |
                                          v
                               LangChain pipeline
                    1. Extract   LLM returns figures + risks (Pydantic schema)
                    2. Compute   plain Python calculates the 5 ratios
                    3. Summarise LLM writes summary + ratio explanations
                    4. Verify    Python checks every number exists in source text
                                 (fail -> retry step 3 once, then flag result)
                                          |
React UI <--JSON: summary, ratios, risks, page refs

Core idea: the LLM only extracts and writes. Python does the math and the number check.

Keeping it simple
Only 2 LLM calls per report (extract, summarise), which suits free-tier rate limits.
No database, no login, no background jobs. One request in, one JSON response out.
Send only the relevant pages to the model (financial statements and risk factors, found by keyword search), not the whole report.
One retry on verification failure, then return the result with a warning.
One-page frontend: upload button, loading state, results.
API key lives in backend .env only.
API
POST /summarise: multipart PDF in, JSON out (summary, ratios[], risks[], warnings[])
GET /health
Project layout
financial-summariser/
├── backend/
│   ├── app/
│   │   ├── main.py          # FastAPI app + /summarise route
│   │   ├── pdf_parser.py
│   │   ├── extractor.py     # LangChain extraction (structured output)
│   │   ├── ratios.py        # pure Python
│   │   ├── summariser.py    # LangChain summary
│   │   ├── verifier.py      # number-in-source check
│   │   └── schemas.py       # Pydantic models
│   ├── tests/
│   └── requirements.txt
├── frontend/                # Vite + React + TS
├── prd.md
└── .env.example             # GOOGLE_API_KEY, MODEL_NAME
Build setup (GitHub Copilot)
MCP (during build): Fetch, to pull sample public annual reports for testing.
Stop hook: refuses to finish if any number in the summary is not found in the source text. Computed ratios are exempt, but their input figures must be found.
The same check runs inside the app as verifier.py.
Success
Every number traces back to the source.
Summary fits one page.
Ratios match a manual calculation on 3 test reports.
No made-up risks.
Stretch

Compare two years: upload two reports, show year-over-year changes in key figures and ratios.