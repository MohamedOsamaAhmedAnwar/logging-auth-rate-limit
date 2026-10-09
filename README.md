# InsightAPI

A practical FastAPI service demonstrating:

- API-key authentication
- Per-user sliding-window rate limiting
- Rotating file and terminal logging
- API usage reporting
- OpenAI-compatible model endpoints

The application was configured and tested with the `qwen2.5:7b` model. It can also use Ollama, Groq, Inception, real OpenAI, or a fully offline deterministic client.

---

## Project structure

```text
app/
├── config.py                  # Application configuration
├── main.py                    # FastAPI routes
├── schemas.py                 # Pydantic request/response models
├── services/
│   ├── fake_llm.py            # Offline deterministic LLM client
│   └── llm.py                 # OpenAI-compatible LLM client
└── utils/
    ├── logging_config.py      # Rotating-file and terminal logging
    ├── rate_limit.py          # Sliding-window limiter and dependency
    └── security.py            # API-key authentication
logs/
└── insight_api.log            # Created automatically at startup
```

---

## Requirements

- Python 3.12 or newer
- `uv` or `pip`

---

## Installation

### Option A: `uv` — recommended

```powershell
uv sync
```

### Option B: `pip`

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If PowerShell blocks the activation script, run:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

On macOS or Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## Configuration

Copy `.env.example` to `.env`, then select one LLM backend.

```powershell
Copy-Item .env.example .env
```

### Offline fake client

The simplest option requires no model or network access:

```env
OPENAI_API_KEY=fake
OPENAI_BASE_URL=fake
```

### Ollama

Install and pull the model:

```powershell
ollama pull qwen2.5:7b
```

Configure:

```env
OPENAI_API_KEY=ollama
OPENAI_BASE_URL=http://localhost:11434/v1
```

### Groq

```env
OPENAI_API_KEY=your_groq_key
OPENAI_BASE_URL=https://api.groq.com/openai/v1
```

Add the selected Groq model to `allowed_models` in `app/config.py`.

### Inception

```env
OPENAI_API_KEY=your_inception_key
OPENAI_BASE_URL=https://api.inceptionlabs.ai/v1
```

### Real OpenAI

```env
OPENAI_API_KEY=your_openai_key
OPENAI_BASE_URL=
```

---

## Authentication

Pass one of the development API keys in the `X-API-Key` header.

| API key       | User  |
|---------------|-------|
| `dev-key-123` | `ahmed` |
| `dev-key-456` | `ali`   |
| `dev-key-789` | `mona`  |

The API returns:

- `401 Unauthorized` when the header is missing
- `403 Forbidden` when the API key is invalid
- Internal user IDs rather than raw API keys in downstream processing

FastAPI’s interactive documentation provides an **Authorize** button for entering an API key.

---

## Rate limiting

The service uses a per-user sliding window of one minute.

| User  | Limit             |
|-------|-------------------|
| `ahmed` | 10 requests/minute |
| `ali`   | 5 requests/minute  |
| `mona`  | 3 requests/minute  |

Once a user reaches the configured limit, additional requests return:

```http
HTTP/1.1 429 Too Many Requests
```

The limits are configured by `rate_limit_per_minute` and `rate_limits` in `app/config.py`.

Calling `/usage` reports rate-limit status but does **not** consume a request slot.

---

## Logging

Logging is configured at application startup and includes:

- Timestamp
- Logger name
- Log level
- Message

Logs are sent to both:

- `logs/insight_api.log`
- The terminal

The file logger rotates according to the following settings in `app/config.py`:

- `log_max_bytes`
- `log_backup_count`
- `log_level`

The logging setup is idempotent: repeated calls do not attach duplicate handlers.

---

## Running the server

With `uv`:

```powershell
uv run uvicorn app.main:app --reload
```

With an activated pip environment:

```powershell
uvicorn app.main:app --reload
```

Open the interactive API documentation:

```text
http://localhost:8000/docs
```

---

## Endpoints

### `GET /`

Returns application and endpoint information.

```powershell
curl.exe http://localhost:8000/
```

### `GET /health`

Returns the service health state.

```powershell
curl.exe http://localhost:8000/health
```

Response:

```json
{
  "status": "ok"
}
```

### `GET /usage`

Returns the authenticated user’s rate-limit state without consuming a request slot.

```powershell
curl.exe `
  -H "X-API-Key: dev-key-123" `
  http://localhost:8000/usage
```

Example response:

```json
{
  "user": "ahmed",
  "limit": 10,
  "remaining": 10
}
```

### `POST /generate`

Generates a response using the configured model.

```powershell
curl.exe `
  -X POST "http://localhost:8000/generate" `
  -H "X-API-Key: dev-key-123" `
  -H "Content-Type: application/json" `
  -d '{
    "text": "Explain FastAPI in one sentence.",
    "model": "qwen2.5:7b",
    "temperature": 0.7
  }'
```

Example offline response:

```json
{
  "endpoint": "/generate",
  "user": "ahmed",
  "input_text": "Explain FastAPI in one sentence.",
  "output_text": "[fake:qwen2.5:7b] Explain FastAPI in one sentence.",
  "model_used": "qwen2.5:7b",
  "processing_time_ms": 0.12,
  "success": true
}
```

### `POST /summarize`

Summarizes text using the configured model.

```powershell
curl.exe `
  -X POST "http://localhost:8000/summarize" `
  -H "X-API-Key: dev-key-123" `
  -H "Content-Type: application/json" `
  -d '{
    "text": "FastAPI is a modern Python framework for building APIs. It provides automatic validation, interactive documentation, and high performance.",
    "model": "qwen2.5:7b",
    "temperature": 0.2,
    "max_words": 50
  }'
```

---

## Authentication verification

### Missing API key

```powershell
curl.exe -i http://localhost:8000/usage
```

Expected:

```http
HTTP/1.1 401 Unauthorized
```

### Invalid API key

```powershell
curl.exe -i `
  -H "X-API-Key: incorrect-key" `
  http://localhost:8000/usage
```

Expected:

```http
HTTP/1.1 403 Forbidden
```

---

## Rate-limit verification

Mona’s key is configured for three requests per minute. Send four requests:

```powershell
1..4 | ForEach-Object {
    curl.exe -i `
      -X POST "http://localhost:8000/generate" `
      -H "X-API-Key: dev-key-789" `
      -H "Content-Type: application/json" `
      -d '{
        "text": "test",
        "model": "qwen2.5:7b",
        "temperature": 0.7
      }'
}
```

Expected statuses:

```text
200
200
200
429
```

Check Mona’s remaining requests:

```powershell
curl.exe `
  -H "X-API-Key: dev-key-789" `
  http://localhost:8000/usage
```

Expected response:

```json
{
  "user": "mona",
  "limit": 3,
  "remaining": 0
}
```

Wait for the one-minute sliding window to expire before making additional Mona requests.

---

## Viewing logs

### PowerShell

```powershell
Get-Content .\logs\insight_api.log -Wait
```

### macOS/Linux

```bash
tail -f logs/insight_api.log
```

Typical log entries include application startup, successful rate-limit checks, exceeded limits, LLM calls, and endpoint activity.
