# SupportAI — AI-Powered Customer Support Chatbot

A production-ready FAQ chatbot built with **FastAPI**, **DistilBERT**, **SQLite**, and a clean HTML/JS frontend.

---

## Project Structure

```
ai-support-bot/
├── app.py              # FastAPI server — /chat and /logs endpoints
├── nlp_engine.py       # NLP logic: FAQ match → DistilBERT → fallback
├── database.py         # SQLAlchemy models + helpers for interaction logging
├── requirements.txt    # Python dependencies
├── static/
│   └── index.html      # Chat UI + Logs viewer (single-file frontend)
└── support_logs.db     # SQLite DB — auto-created on first run
```

---

## Quick Start

### 1. Create & activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

> **Note:** PyTorch (~2 GB) and the DistilBERT model (~250 MB) download on first run.
> Subsequent starts use the local Hugging Face cache (usually `~/.cache/huggingface`).

### 3. Run the server

```bash
uvicorn app:app --reload --port 8000
```

### 4. Open the UI

```
http://localhost:8000
```

Interactive API docs are auto-generated at:
- **Swagger UI** → `http://localhost:8000/docs`
- **ReDoc**       → `http://localhost:8000/redoc`

---

## API Reference

### `POST /chat`
Send a user message and receive a bot reply.

**Request body:**
```json
{ "message": "How do I reset my password?" }
```

**Response:**
```json
{
  "reply": "To reset your password, click 'Forgot Password'...",
  "confidence": 1.0,
  "source": "faq_exact",
  "status": "answered",
  "log_id": 42
}
```

| Field        | Description |
|-------------|-------------|
| `reply`      | Bot's answer or human-escalation message |
| `confidence` | Score in [0,1]; `null` when escalated to human |
| `source`     | `faq_exact` \| `faq_fuzzy` \| `model` \| `fallback` |
| `status`     | `answered` \| `fallback` |
| `log_id`     | Row ID in `interaction_logs` table |

---

### `GET /logs`
Retrieve paginated interaction logs.

**Query params:**

| Param       | Default | Description |
|------------|---------|-------------|
| `page`      | `1`     | Page number |
| `page_size` | `20`    | Rows per page (max 100) |
| `status`    | —       | Filter: `answered` or `fallback` |

---

## Configuration

### Tune the confidence threshold
In `nlp_engine.py`, change `CONFIDENCE_THRESHOLD` (default `0.20`).
- **Higher** → more escalations, fewer wrong answers.
- **Lower** → bot answers more, risk of low-quality replies.

### Extend the FAQ
Add entries to the `FAQ` dictionary in `nlp_engine.py`:

```python
FAQ: dict[str, str] = {
    "how do i reset my password": "...",
    "what are your shipping rates": "Standard shipping is free on orders over ₹500.",
    # add your own Q&A pairs here
}
```

### CORS
In production, update `allow_origins` in `app.py`:
```python
allow_origins=["https://yourapp.com"],
```

---

## Architecture Overview

```
Browser / Frontend
      │  POST /chat
      ▼
  FastAPI (app.py)
      │
      ├──▶ NLPEngine.answer()
      │         │
      │    1. Exact FAQ key match        → confidence 1.0
      │    2. Fuzzy keyword match        → confidence 0.85
      │    3. DistilBERT extractive Q&A  → model score
      │    4. Fallback (below threshold) → null confidence
      │
      └──▶ log_interaction() → SQLite (support_logs.db)
```

---

## Tech Stack

| Layer     | Technology |
|-----------|-----------|
| API       | FastAPI 0.111 |
| NLP Model | `distilbert-base-cased-distilled-squad` (Hugging Face) |
| Database  | SQLite via SQLAlchemy 2.0 |
| Frontend  | Vanilla HTML / CSS / JS |
| Server    | Uvicorn (ASGI) |
