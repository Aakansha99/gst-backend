# GST Statement Parser — Backend API

A FastAPI service that parses bank statement PDFs into structured JSON. Designed to work with the [gst-statement-processor](https://github.com/Aakansha99/gst-statement-processor) React frontend.

## What it does

Upload any Indian bank statement PDF → get back structured transaction data (date, description, signed amount, running balance) as JSON.

**Supported banks** (tested and verified):
- ICICI Bank (detailed statement + summary statement)
- HDFC Bank
- YES Bank
- Allahabad Bank (Indian Overseas Bank)
- Any bank with a standard tabular layout

**How it works** — the parser doesn't use hardcoded column names. It identifies columns by data shape:
- A column of dates → date column
- A column of decimals → money column
- The money column whose values track incrementally → running balance
- Debit vs credit → classified by correlating amounts with balance deltas

This means new banks usually work without code changes.

## Prerequisites

- Python 3.10 or later
- pip

## Quick Start

```bash
# Clone the repo
git clone https://github.com/Aakansha99/gst-backend.git
cd gst-backend

# Create virtual environment and install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Start the server
uvicorn app.main:app --reload --port 8000
```

You should see:
```
INFO:gst-statement-api:gst-statement-api startup — exception handler active
INFO:     Uvicorn running on http://127.0.0.1:8000
```

## API Endpoints

### `GET /api/health`

Health check / readiness probe.

```bash
curl http://127.0.0.1:8000/api/health
# → {"status": "ok"}
```

### `POST /api/parse`

Upload a PDF bank statement and get parsed transactions back.

**Request:** `multipart/form-data` with a `file` field containing the PDF.

```bash
curl -X POST -F "file=@statement.pdf" http://127.0.0.1:8000/api/parse
```

**Success response (200):**

```json
{
  "statementPeriod": {
    "startDate": "01/04/2025",
    "endDate": "30/06/2025"
  },
  "transactionGroups": [
    {
      "account": {
        "accountNumber": "987654321012",
        "accountName": "MR. SAMPLE CUSTOMER"
      },
      "transactions": [
        {
          "date": "02/04/2025",
          "description": "UPI/AMAZON/9876543210/PAYMENT",
          "amount": -1499.00,
          "balance": 23501.00
        }
      ]
    }
  ],
  "warnings": []
}
```

**Error responses:**

| Status | Meaning |
|--------|---------|
| 400 | Not a PDF file, or file is empty |
| 413 | File exceeds 10 MB limit |
| 422 | PDF parsed but no transactions found |
| 500 | Internal parser error (details in response body) |

All errors return JSON:
```json
{ "error": "human-readable error message" }
```

## Running with the Frontend

The React frontend ([gst-statement-processor](https://github.com/Aakansha99/gst-statement-processor)) is configured to proxy `/api/*` requests to `http://127.0.0.1:8000` during development.

**Terminal 1 — Backend:**
```bash
cd gst-backend
source .venv/bin/activate
uvicorn app.main:app --reload --port 8000
```

**Terminal 2 — Frontend:**
```bash
cd gst-statement-processor
npm install
npm run dev
```

Open `http://localhost:5173` → upload a PDF → see transactions.

## Running Tests

```bash
source .venv/bin/activate
pytest tests/ -v
```

Tests use the fixture PDFs in `tests/fixtures/` — no external files needed.

## Project Structure

```
gst-backend/
├── app/
│   ├── __init__.py
│   ├── main.py                # FastAPI app, CORS, routes, error handling
│   ├── parser_adapter.py      # Converts ParseResult → frontend JSON shape
│   └── parser/
│       ├── __init__.py
│       └── parse_statement.py # Generic bank statement parser (pdfplumber)
├── tests/
│   ├── __init__.py
│   ├── test_api.py            # API integration tests
│   └── fixtures/
│       ├── sample_bank_statement.pdf
│       └── sample_bank_statement_zero_filled.pdf
├── .gitignore
├── requirements.txt
└── README.md
```

## How the Parser Works

The parser pipeline:

1. **Table extraction** — tries multiple pdfplumber strategies (auto, lines, text, hybrid) per page and picks the one with the best density-weighted score (favoring strategies where date-bearing rows make up a high proportion of total rows, not just the raw count)
2. **Column profiling** — counts what fraction of each column's cells look like dates, decimals, or text
3. **Column identification** — date column = highest date ratio; money columns = decimal-bearing; description = longest text; balance = the money column with incremental continuity
4. **Debit/credit classification** — correlates each money column's values with balance deltas to determine which is debit and which is credit
5. **Sign inference** — uses the classification to sign each transaction amount (negative = debit, positive = credit)
6. **Fuzzy date fallback** — if strict regex matching finds no dates, retries with `dateutil` for unusual formats

## Privacy

- PDFs are uploaded to the server for processing
- Files are processed **in memory only** — nothing is written to disk or stored
- No logging of file contents; only filename and content-type are logged

## Tech Stack

- [FastAPI](https://fastapi.tiangolo.com/) — async Python web framework
- [pdfplumber](https://github.com/jsvine/pdfplumber) — PDF text and table extraction
- [python-dateutil](https://dateutil.readthedocs.io/) — fuzzy date parsing fallback
- [uvicorn](https://www.uvicorn.org/) — ASGI server
- [pytest](https://docs.pytest.org/) + [httpx](https://www.python-httpx.org/) — testing

## License

MIT
