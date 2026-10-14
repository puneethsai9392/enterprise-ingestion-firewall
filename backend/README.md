# Enterprise Ingestion Firewall — Backend Setup Guide (Windows)

This guide explains how to configure, test, run, and troubleshoot the Enterprise Ingestion Firewall project on Windows using PowerShell.

The project uses **FastAPI, Groq, Instructor, and Pydantic** to classify user messages and return schema-validated JSON responses.

## 1. Configure the Groq API Key

Open PowerShell in the `backend` directory.

Create your local environment file from the example:

```powershell
Copy-Item .env.example .env
notepad .env
```

Add your Groq API key and model name to `.env`:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

Replace the example API key with your actual key from [Groq Console](https://console.groq.com/keys).

**Important:**
- Do not add quotation marks around the values.
- Do not add spaces around `=`.
- Never commit `.env` or publish your API key.
- Keep `.env.example` as a template without real secrets.

**Model configuration:** The model name must be supported by Groq and available to your account. If the configured model is unavailable, choose one from the models returned by the command in the troubleshooting section.

## 2. Create the Virtual Environment

Run these commands from the `backend` directory:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

After activation, the terminal prompt should start with `(.venv)`.

If PowerShell blocks script execution, allow activation scripts for the current PowerShell process:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
```

This process-scoped setting does not permanently change the execution policy for your account.

## 3. Run the Tests

Make sure the virtual environment is active and run:

```powershell
python -m pytest -q
```

Expected output:

```text
4 passed
```

These four tests check the Pydantic schemas. They do not test the live Groq API or the complete HTTP endpoint.

## 4. Start the Backend API

Run this command from the `backend` directory:

```powershell
python -m uvicorn app.main:app --reload --port 8000
```

Keep this terminal open while using the application.

Check the health endpoint:

[http://127.0.0.1:8000/health](http://127.0.0.1:8000/health)

A successful response should look like this:

```json
{
  "status": "ok",
  "message": "Ingestion Firewall API is running"
}
```

The health endpoint confirms that the API is running. It does not verify Groq connectivity.

## 5. Start the Frontend

Open a **second PowerShell terminal**.

Move to the project root directory, which contains both `backend` and `frontend`.

For example:

```powershell
cd "C:\Users\Hp\Videos\INTENT"
```

Start the frontend server:

```powershell
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

Keep this terminal open too.

Your project should have a structure similar to:

```text
INTENT/
├── backend/
│   ├── .env
│   ├── .venv/
│   ├── app/
│   ├── tests/
│   └── requirements.txt
└── frontend/
    ├── index.html
    ├── style.css
    └── script.js
```

## 6. Open the Application

Open the frontend:

[http://127.0.0.1:5500/index.html](http://127.0.0.1:5500/index.html)

Open the interactive API documentation:

[http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)

You can also open these pages using PowerShell:

```powershell
Start-Process "http://127.0.0.1:5500/index.html"
Start-Process "http://127.0.0.1:8000/docs"
```

Enter a message in the frontend and click **Analyze message**. The application should display the structured JSON returned by the backend.

## 7. Test the API Directly

You can test the backend without using the frontend.

Run this command in another PowerShell terminal:

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8000/api/analyze" `
  -ContentType "application/json" `
  -Body '{"message":"Where is my order?"}'
```

A successful response should contain fields similar to:

```json
{
  "intent": "order_tracking",
  "risk_profile": "medium",
  "priority": "medium",
  "requires_action": true,
  "summary": "User is asking for the status of their order.",
  "token_metadata": {
    "estimated_input_tokens": 5,
    "note": "Estimated locally; may differ from provider token counts."
  }
}
```

The exact classification and summary can vary between requests. The example is illustrative, not a guaranteed response.

## 8. Troubleshooting a 502 LLM Request Error

A `502` response means the backend could not complete the request successfully. Possible causes include an invalid API key, an unavailable model, a network failure, or an LLM response that failed validation.

### Step 1: Check the API key and model configuration

Confirm that `backend/.env` exists and contains valid values:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

Make sure the file is named `.env`, not `.env.txt`.

### Step 2: Test the LLM service directly

With the virtual environment active, run:

```powershell
python -c "from app.services.llm_service import analyze_message; print(analyze_message('Hi'))"
```

This command assumes the service module is named `llm_service.py` and exports `analyze_message`. If your actual filenames or function names differ, use those names instead.

### Step 3: Identify the cause

| Error | Likely cause | What to do |
|---|---|---|
| `401` or invalid API key | Invalid, revoked, or incorrect key | Check the key in `.env` |
| `404 model_not_found` | Model unavailable or incorrect name | Select a supported model |
| Missing API key | `.env` missing or not loaded | Check the file location and configuration |
| Connection or timeout | Network issue or provider outage | Check connectivity and provider status |
| Validation failure | Model response does not match the schema | Inspect backend logs and validation errors |

### Step 4: List available Groq models

Run this from the `backend` directory:

```powershell
python -c "from groq import Groq; from app.config import GROQ_API_KEY; [print(m.id) for m in Groq(api_key=GROQ_API_KEY).models.list().data]"
```

This requires a valid API key and a working connection. Select a model that is available to your account.

### Step 5: Change the model temporarily

To override the configured model for the current PowerShell session:

```powershell
$env:GROQ_MODEL = "openai/gpt-oss-120b"
python -m uvicorn app.main:app --reload --port 8000
```

Stop the previous Uvicorn process before starting another one on the same port. Restart the backend after changing `.env` values.

## API Reference

### `GET /health`

Checks whether the FastAPI application is running. It does not call the LLM.

### `POST /api/analyze`

Accepts a JSON request:

```json
{
  "message": "My payment was deducted twice."
}
```

The `message` field must contain between 1 and 5000 characters after trimming whitespace. Whitespace-only messages are rejected. Additional request fields are rejected if the request schema forbids them.

### Response Fields

| Field | Description |
|---|---|
| `intent` | `greeting`, `order_tracking`, `payment_issue`, `current_time`, `general_question`, or `unknown` |
| `risk_profile` | `low`, `medium`, `high`, or `unknown` |
| `priority` | `low`, `medium`, or `high` |
| `requires_action` | Boolean indicating whether the classifier considers a follow-up action necessary |
| `summary` | Short description of the user message |
| `token_metadata.estimated_input_tokens` | Local estimate of input tokens; not provider billing data |

### HTTP Status Codes

| Code | Meaning |
|---|---|
| `200` | Request processed and a valid response returned |
| `422` | Request body failed input validation |
| `502` | LLM processing failed or a valid structured response could not be obtained |

Actual error handling depends on the implementation in `app/main.py`.

## Code Structure

| File | Responsibility |
|---|---|
| `app/config.py` | Loads the Groq API key and model configuration |
| `app/schemas/ingestion.py` | Defines request and response Pydantic models |
| `app/services/llm_service.py` | Sends prompts to the LLM and processes structured responses |
| `app/utils/token_counter.py` | Estimates input tokens locally |
| `app/main.py` | Creates the FastAPI application and exposes API endpoints |
| `tests/test_ingestion.py` | Tests the request and response schemas |

## Known Limitations

- Schema validation checks structure and field constraints; it does not guarantee that the AI classification is factually correct.
- Token counts are estimates and may differ from the provider's reported usage.
- The health endpoint does not verify that Groq is reachable.
- The current tests cover schema validation only, not the full API or live LLM integration.
- If the application maps all LLM errors to `502`, inspect backend logs to find the underlying cause.
- Permissive CORS settings may be acceptable for local development but should be restricted before public deployment.
- The demo classifies messages. It does not independently retrieve order information, perform refunds, or fetch the current time.

## Quick Startup Checklist

- [ ] `.env` contains a valid Groq API key and supported model.
- [ ] Virtual environment is activated and dependencies are installed.
- [ ] Tests pass.
- [ ] Backend is running on port `8000`.
- [ ] Frontend is running on port `5500`.
- [ ] Frontend and API documentation open successfully.
- [ ] A test request returns a schema-validated JSON response.
