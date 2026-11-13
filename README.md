# Multi-Country Invoice Reader API

[![Python 3.13+](https://img.shields.io/badge/python-3.13%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![Ollama](https://img.shields.io/badge/Ollama-000000?logo=ollama&logoColor=white)](https://ollama.com/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A high-throughput, asynchronous invoice extraction REST API that processes **UAE**, **USA**, and **India** invoices. It uses local **PaddleOCR** for text extraction (GPU-accelerated) and **Ollama** with **Gemma** for structured JSON output — no cloud API keys, no per-page billing, everything runs on your own hardware.

---

## Highlights

- **Multi-country support** — UAE (VAT/TRN), USA (sales tax/EIN), India (GST/PAN/HSN) with country-specific validation rules
- **Fully local pipeline** — PaddleOCR + Ollama Gemma; no data leaves your machine
- **4–5 concurrent extractions** — async semaphore-based flow control optimized for GPU (T4/A100)
- **Structured JSON output** — Pydantic-validated schemas with human-readable field names, enforced via Ollama's native JSON Schema
- **Batch processing** — prepare, submit, poll, and retrieve results for up to 100 invoices per batch
- **Docker-ready** — single `docker compose up` with health checks, auto-reload for development, and unprivileged container user
- **API key authentication** — separate client and admin keys for extraction vs. system operations

---

## Architecture

```
Client (Postman / Web App / Script)
  │
  │  POST /api/v1/{country}/invoices/extract
  │  multipart/form-data: file (PDF / JPG / PNG)
  ▼
Authentication & Concurrency Gate (asyncio.Semaphore)
  ▼
File Validation (signature, dimensions, page count)
  ▼
PaddleOCR Extraction (GPU via asyncio.to_thread)
  ├── PDF  → PyMuPDF renders pages at 200 DPI → PaddleOCR
  └── Image → PaddleOCR directly
  ▼
Text Compaction & Line-Order Reconstruction
  ▼
Ollama / Gemma (structured JSON via Pydantic schema)
  ▼
Validated JSON Response (with execution metrics)
```

The codebase follows a **modular country architecture** under `src/uae_invoice_reader/countries/`, with each country (`uae`, `usa`, `india`) containing its own `models / dto / controllers / routes / services`. Shared components (OCR, validation, middleware, security) live under `common/`. See [`ARCHITECTURE.md`](ARCHITECTURE.md) for full dependency boundaries.

---

## Tech Stack

| Component | Technology |
|---|---|
| Framework | FastAPI (Python 3.13) with native async/await |
| OCR Engine | PaddleOCR PP-OCRv5 (GPU/CPU) |
| PDF Rendering | PyMuPDF (fitz) |
| LLM | Ollama → Gemma 2B/9B |
| Structured Output | Ollama native JSON Schema + Pydantic |
| Config | Pydantic Settings with `.env` |
| Containerization | Docker + Docker Compose |

---

## Quick Start

### Prerequisites

- **Python 3.13+** with [uv](https://docs.astral.sh/uv/) package manager
- **Ollama** running locally with a Gemma model pulled (`ollama pull gemma2:2b`)
- GPU recommended (NVIDIA T4 16GB+ for concurrent processing)

### 1. Clone & configure

```bash
git clone https://github.com/<your-username>/multi-country-invoice-reader.git
cd multi-country-invoice-reader
cp .env.example .env
# Edit .env — set SAVVY_API_KEY and ADMIN_API_KEY
```

### 2. Run locally

```bash
uv sync
uv run uvicorn main:app --host 0.0.0.0 --port 8000
```

### 3. Or run with Docker

```bash
docker compose up -d --build
docker compose ps       # verify container is healthy
docker compose logs -f api
```

> Docker publishes host port **4207** → container port **8000**.

---

## API Endpoints

All extraction endpoints require the `SAVVY_API_KEY` header.

### Health Check

```
GET /health
```

### Invoice Extraction

| Endpoint | Country | Description |
|---|---|---|
| `POST /api/v1/invoices/extract` | UAE | Extract structured data from a UAE invoice |
| `POST /api/v1/invoices/complete` | UAE | Force full PaddleOCR on every page |
| `POST /api/v1/usa/invoices/extract` | USA | Extract with US tax/EIN fields |
| `POST /api/v1/india/invoices/extract` | India | Extract with GST/PAN/HSN fields |

### Batch Processing

```
POST /api/v1/invoices/batch/prepare     # OCR a single invoice (returns text + metadata)
POST /api/v1/invoices/batch/submit      # Submit 1–100 prepared texts for async extraction
GET  /api/v1/invoices/batch/status/{id} # Poll batch progress
GET  /api/v1/invoices/batch/result/{id} # Retrieve completed extractions
```

### System

```
GET  /api/v1/system/processing-status   # Live concurrency & recent errors
POST /api/v1/system/restart             # Safe container restart (requires ADMIN_API_KEY)
```

---

## Example Response

```json
{
  "success": true,
  "country": "uae",
  "data": {
    "invoice_number": "INV-2026-0042",
    "invoice_date": "2026-03-15",
    "vendor_name": "Gulf Tech Supplies LLC",
    "vendor_trn_no": "100234567800003",
    "buyer_name": "Al Amal Trading Co.",
    "net_amount": 1000.00,
    "vat_rate": 5.0,
    "total_tax": 50.00,
    "grand_total": 1050.00,
    "items": [
      {
        "item": "Network Switch 24-Port",
        "qty": 2.0,
        "unit_price": 500.00,
        "net_amount": 1000.00,
        "line_total": 1050.00
      }
    ]
  },
  "execution_time_ms": 1420
}
```

---

## Configuration

All settings are environment-driven via `.env`. See [`.env.example`](.env.example) for the full list.

| Variable | Default | Description |
|---|---|---|
| `OLLAMA_MODEL` | `gemma2:2b` | Ollama model for extraction |
| `MAX_CONCURRENT_EXTRACTIONS` | `4` | Parallel GPU extraction slots |
| `PADDLE_DEVICE` | `gpu` | `gpu` or `cpu` for PaddleOCR |
| `MAX_UPLOAD_MB` | `15` | Max upload file size |
| `MAX_PDF_PAGES` | `20` | Max pages per PDF |
| `AUTO_RELOAD` | `false` | Enable Uvicorn hot-reload (dev) |

---

## Testing

```bash
uv run pytest
```

---

## Postman Collections

Pre-built collections are included in the `postman/` directory for all three countries. Import them into Postman, set the `apiKey` variable, and start testing.

---

## Project Structure

```
├── main.py                  # FastAPI app entry point
├── src/uae_invoice_reader/
│   ├── api.py               # Router aggregation
│   ├── common/              # Shared: OCR, validation, middleware, security
│   │   ├── services/        # PaddleOCR, Ollama client, text compaction
│   │   ├── controllers/     # System status, restart
│   │   ├── dto/             # Base response schemas
│   │   └── models/          # Document models
│   └── countries/
│       ├── uae/             # UAE-specific models, routes, prompts
│       ├── usa/             # USA-specific (EIN, sales tax, freight)
│       └── india/           # India-specific (GST, PAN, HSN/SAC, IRN)
├── postman/                 # Postman collections per country
├── Dockerfile
├── docker-compose.yml
└── pyproject.toml
```

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## Author

**Vikas Pande** — [GitHub](https://github.com/vikaspande18)
