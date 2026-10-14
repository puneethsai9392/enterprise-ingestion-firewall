# Enterprise Structured Ingestion Firewall

A beginner-friendly AI application that classifies customer-support messages using a cloud-based Large Language Model (LLM) and validates the structured output with Pydantic before returning it as JSON.

The backend is built with **FastAPI**, while the optional frontend uses **HTML, CSS, and JavaScript**. The application integrates **Groq, Instructor, Pydantic, and tiktoken** to demonstrate structured LLM outputs, schema validation, and API integration.

The core idea behind the “firewall” is to enforce a predefined schema on the model's output. Responses that fail validation are rejected or retried according to the application's error-handling logic.

## How It Works

```text
User enters a message in the browser
                |
                v
Frontend sends POST /api/analyze
                |
                v
FastAPI validates the request
                |
                v
IngestionRequest (Pydantic)
                |
                v
Groq LLM + Instructor
Classifies the message
                |
                v
IngestionResult (Pydantic)
Validates the structured response
                |
                v
Token counter estimates input tokens
                |
                v
FastAPI returns JSON to the frontend
```

### Example

**Input message:**

```text
My payment was deducted twice. Please help urgently.
```

**Illustrative response:**

```json
{
  "intent": "payment_issue",
  "risk_profile": "high",
  "priority": "high",
  "requires_action": true,
  "summary": "User reports a duplicate payment deduction and requests urgent assistance.",
  "token_metadata": {
    "estimated_input_tokens": 10,
    "note": "Estimated locally; may differ from provider token counts."
  }
}
```

The exact classification, summary, and token estimate may vary. The example demonstrates the response structure rather than guaranteeing a particular result.

## Features

- **AI-based message classification:** Classifies messages into predefined intent categories.
- **Structured LLM outputs:** Uses Instructor to obtain output matching a Pydantic model.
- **Schema validation:** Checks field types, allowed values, and other configured constraints.
- **REST API:** Exposes the message-analysis endpoint using FastAPI.
- **Token estimation:** Uses tiktoken to estimate input tokens locally.
- **Interactive frontend:** Allows users to enter messages, view JSON responses, and copy results.
- **Error handling:** Returns API errors when request validation or LLM processing fails.
- **Automated schema tests:** Includes pytest tests for the Pydantic models.

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Main programming language |
| FastAPI | Backend API and routing |
| Groq | Cloud LLM inference |
| Instructor | Structured LLM output integration |
| Pydantic | Request and response validation |
| tiktoken | Local token estimation |
| HTML | Frontend structure |
| CSS | Frontend styling |
| JavaScript | API requests and UI interactions |
| pytest | Automated testing |

## Requirements

- Python 3.10 or newer; Python 3.12 is the tested version for this project.
- A Groq API key from [Groq Console](https://console.groq.com/keys).
- Internet connectivity for cloud LLM requests.
- PowerShell on Windows, or a terminal on macOS/Linux.

## Quick Start — Windows PowerShell

### 1. Set up the backend

From the project root, run:

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

If PowerShell blocks activation, allow scripts for the current process and activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
```

### 2. Configure environment variables

Create your local environment file:

```powershell
Copy-Item .env.example .env
notepad .env
```

Set the following values in `backend/.env`:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

Replace the example key with your real key. Confirm that the model is currently available to your Groq account.

**Security:** Never commit `.env` or expose your API key in screenshots, logs, or public repositories.

### 3. Run the tests

From the `backend` directory:

```powershell
python -m pytest -q
```

The included tests check the Pydantic schemas. They do not necessarily exercise the live Groq service or the complete API endpoint.

### 4. Start the backend

Still inside `backend`, run:

```powershell
python -m uvicorn app.main:app --reload --port 8000
```

Keep the terminal open.

Check the health endpoint:

- [API health](http://127.0.0.1:8000/health)
- [Interactive API documentation](http://127.0.0.1:8000/docs)

The health endpoint checks whether the application is running; it does not guarantee that the Groq API is reachable.

### 5. Start the frontend

Open a second PowerShell terminal. Navigate to the project root, one level above `backend`.

For example:

```powershell
cd "C:\Users\Hp\Videos\INTENT"
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

Keep this terminal open.

Open the application in your browser:

[http://127.0.0.1:5500/index.html](http://127.0.0.1:5500/index.html)

Make sure the frontend's API URL points to the backend:

```javascript
const API_URL = "http://127.0.0.1:8000";
```

### macOS/Linux

Activate the virtual environment with:

```bash
source .venv/bin/activate
```

Use `cp .env.example .env` instead of `Copy-Item`, and `python -m uvicorn app.main:app --reload --port 8000` to start the backend.

## API Reference

### `GET /health`

Checks whether the FastAPI application is running.

Example response:

```json
{
  "status": "ok",
  "message": "Ingestion Firewall API is running"
}
```

### `POST /api/analyze`

Classifies a user message and returns a structured response.

**Request body:**

```json
{
  "message": "Where is my order?"
}
```

The message must contain 1–5000 characters after trimming whitespace. Extra fields are rejected if the request schema is configured to forbid them.

**Possible intent values:**

- `greeting`
- `order_tracking`
- `payment_issue`
- `current_time`
- `general_question`
- `unknown`

The response also contains `risk_profile`, `priority`, `requires_action`, `summary`, and `token_metadata`. Refer to `backend/app/schemas/ingestion.py` for the authoritative field constraints.

### Common HTTP Status Codes

| Status | Meaning |
|---|---|
| `200` | Request processed successfully |
| `422` | Request validation failed |
| `502` | LLM processing failed or a valid response could not be produced |

Exact error behavior depends on the implementation in `backend/app/main.py`.

## Project Structure

```text
enterprise-ingestion-firewall/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── schemas/
│   │   │   └── ingestion.py
│   │   ├── services/
│   │   │   └── llm_service.py
│   │   └── utils/
│   │       └── token_counter.py
│   ├── tests/
│   │   └── test_ingestion.py
│   ├── .env.example
│   ├── requirements.txt
│   └── README.md
├── frontend/
│   ├── index.html
│   ├── style.css
│   ├── script.js
│   └── README.md
└── README.md
```

### Key Files

| File | Responsibility |
|---|---|
| `app/main.py` | FastAPI application and API routes |
| `app/config.py` | Environment and model configuration |
| `app/schemas/ingestion.py` | Pydantic request and response models |
| `app/services/llm_service.py` | Prompt construction and Groq/Instructor integration |
| `app/utils/token_counter.py` | Local token estimation |
| `tests/test_ingestion.py` | Schema validation tests |
| `frontend/index.html` | Page structure |
| `frontend/style.css` | UI design |
| `frontend/script.js` | API communication and frontend interactions |

## Troubleshooting

### `502` — LLM request failed

Possible causes include:

- Invalid or revoked Groq API key.
- Model name is incorrect or unavailable.
- Network or provider connectivity issues.
- The LLM output fails the required schema validation.

Check your `.env` configuration, inspect the backend terminal for errors, and verify that the configured model is available to your account.

If the error persists, test the LLM service directly using the actual module and function names in your code.

### Frontend loads, but the design looks different

- Confirm that `style.css` and `script.js` are in the `frontend` directory.
- Check that the paths in `index.html` match the filenames.
- Hard-refresh the browser with `Ctrl + F5`.
- Check the browser developer console for JavaScript errors.
- Verify that the frontend is calling `http://127.0.0.1:8000`.

### Submit button does not work

- Confirm that the backend is running on port `8000`.
- Open `/docs` and verify that `POST /api/analyze` exists.
- Check the browser console and backend terminal for errors.
- Confirm that the API URL and request body match the backend implementation.

## What This Project Does and Does Not Do

**It does:**
- Classify customer-support messages into predefined categories.
- Produce structured output using an LLM.
- Validate the output against a Pydantic schema.
- Return JSON through a FastAPI endpoint.
- Estimate input token counts locally.

**It does not:**
- Retrieve actual order or payment records.
- Process refunds or perform transactions.
- Fetch the current time from a live time service.
- Guarantee that the model's classification is factually correct.
- Guarantee protection against prompt injection merely through schema validation.

No database is required for the current demo.

## Future Improvements

- Add endpoint and integration tests.
- Improve logging and error reporting for LLM failures.
- Add authentication and rate limiting.
- Apply stricter CORS settings before deployment.
- Add business-rule validation between fields.
- Integrate real customer-support services when needed.

## License

Add your chosen license here if you intend to distribute this project publicly.
