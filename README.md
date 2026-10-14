# Mocka AI: LLM-Based Resume & Soft Skills Evaluation Framework

A Flask web app that compares a candidate's resume with a job description (both PDFs) and uses **LLaMA 3.1 8B (via the Groq API)** to produce a relevance score, a skill-gap analysis, and 10 tailored interview questions. Candidates can then record their spoken answers in the browser, and every analysis is saved to their account history.

> **Status:** working prototype, run locally. See [Limitations](#limitations) and [Roadmap](#roadmap).

## Features

- **Resume vs. job description analysis.** Upload two PDFs and get:
  - a **0-100 relevance score** with a short written reason
  - **technical gaps** and **soft-skill / experience gaps**
  - **10 interview questions** (5 technical, 3 behavioural using the STAR method, 2 situational)
- **Mock interview flow.** Questions are shown one at a time; the browser records your spoken answer (with an optional camera preview and a recording timer) and uploads it to the server.
- **User accounts.**
  - username + password registration (passwords hashed with Werkzeug)
  - **Google sign-in** (ID token verified with Google)
  - **OTP-based password reset** over Gmail SMTP (6-digit code, 10-minute expiry, single use)
- **Dashboard and history.** Total analyses, average score and best score, plus a list of past analyses.
- **Graceful AI failures.** If the Groq call fails, the app returns a safe default response (and a built-in question set) instead of crashing.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | Python, Flask, Flask-CORS |
| LLM | LLaMA 3.1 8B Instant via the Groq API |
| PDF parsing | PyMuPDF |
| Database | MySQL (mysql-connector-python) |
| Auth | Werkzeug password hashing, Flask sessions, Google Identity Services |
| Email | smtplib (Gmail SMTP with an App Password) |
| Frontend | Jinja2 templates, HTML/CSS, vanilla JavaScript, browser MediaRecorder API |

## How it works

```
Resume PDF + JD PDF
        │
        ▼
PyMuPDF text extraction  ->  clean_text() (whitespace, bullets, special characters)
        │
        ▼
Three prompts to llama-3.1-8b-instant on Groq
   • relevance score + reason      (temperature 0.3)
   • technical / soft-skill gaps   (temperature 0.3)
   • 10 interview questions        (temperature 0.7)
        │
        ▼
parse_score() extracts the 0-100 number  ->  saved to MySQL (chats table)
        │
        ▼
JSON response -> results page -> mock interview with in-browser recording
```

Inputs are truncated before being sent to the model (3,000 characters per document for scoring, 2,500 for gaps and questions) to stay within the model's context.

## Project structure

```
.
├── app.py                 # Flask app: routes, auth, /analyze, /upload_audio
├── ai_service.py          # Groq prompts: score, skill gaps, questions, answer evaluation
├── db.py                  # MySQL access: users, OTPs, chats, dashboard stats
├── pdf_utils.py           # PDF text extraction and cleaning (PyMuPDF)
├── mail_service.py        # OTP generation and Gmail SMTP sender
├── config.py              # Settings loaded from environment variables
├── resume_analyzer.sql    # Database schema
├── requirements.txt
└── templates/             # landing, login, register, forgot password, OTP,
                           # main app, dashboard, history
```

## Database schema

| Table | Purpose |
|---|---|
| `users` | username, email, hashed password (NULL for Google-only accounts), Google ID, avatar |
| `otp_codes` | password-reset codes with expiry and a `used` flag |
| `chats` | one row per analysis: file names, relevance output, skill gaps, questions. Linked to `users` with `ON DELETE CASCADE`. |

All queries use parameterised statements. The full schema is in `resume_analyzer.sql`.

## Getting started

### Prerequisites

- Python 3.9 or newer
- MySQL 8 (a local install or XAMPP both will work)
- A [Groq API key](https://console.groq.com/)
- Optional: a Gmail App Password (password-reset emails) and a Google OAuth client ID (Google sign-in)

### Setup

```bash
# 1. Clone
git clone https://github.com/abdullahk88/LLM-Based-Resume-and-Soft-Skills-Evaluation-Framework.git
cd LLM-Based-Resume-and-Soft-Skills-Evaluation-Framework

# 2. Create a virtual environment and install dependencies
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

# 3. Create the database (run the schema file in MySQL / phpMyAdmin)
mysql -u root -p < resume_analyzer.sql

# 4. Configure environment variables
cp .env.example .env            # then edit .env with your values

# 5. Run
python app.py
```

Open **http://127.0.0.1:5000** in your browser.

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `GROQ_API_KEY` | Yes | Groq API key |
| `SECRET_KEY` | Yes | Flask session secret. Use a long random string. |
| `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Yes | MySQL connection (defaults: `localhost`, `root`, empty, `resume_analyzer`) |
| `MAIL_USERNAME`, `MAIL_PASSWORD` | Optional | Gmail address and App Password for OTP emails |
| `MAIL_SERVER`, `MAIL_PORT` | Optional | Default `smtp.gmail.com` / `587` |
| `GOOGLE_CLIENT_ID` | Optional | Enables Google sign-in |

If mail is not configured, the OTP is printed to the server console (development mode), so the password-reset flow can still be tested locally.

## Routes

| Route | Method | Description |
|---|---|---|
| `/` | GET | Landing page |
| `/register`, `/login`, `/logout` | GET, POST | Account creation and session login |
| `/google_login` | POST | Verifies a Google ID token and signs the user in |
| `/forgot_password`, `/verify_otp` | GET, POST | OTP password reset |
| `/app` | GET | Main upload and interview page (login required) |
| `/analyze` | POST | Accepts `jd_pdf` + `resume_pdf`; returns score, gaps and questions as JSON and saves the analysis |
| `/get_relevance`, `/get_skill_gaps`, `/get_questions` | POST | Individual endpoints (kept for compatibility) |
| `/upload_audio` | POST | Saves a recorded answer as `.webm` in `recorded_answers/` |
| `/dashboard`, `/history` | GET | Stats and past analyses |

## Limitations

- **Recorded answers are stored but not yet evaluated.** An `evaluate_answer()` function exists in `ai_service.py`, but audio isn't transcribed, so it isn't connected to the UI.
- **Relevance scoring is LLM judgement**, not an embedding-based or deterministic metric, so scores can vary between runs.
- **Long documents are truncated** to the first 2,500-3,000 characters before analysis.
- **Text-based PDFs only.** Scanned resumes without a text layer are not supported (no OCR).
- Runs locally; not deployed. No automated tests yet.

## Roadmap

- [ ] Transcribe recorded answers (speech-to-text) and wire up `evaluate_answer()` for per-answer feedback
- [ ] Store the numeric score in its own integer column (currently stored as text)
- [ ] Docker Compose setup (Flask + MySQL) for one-command local runs
- [ ] Production hardening: disable debug mode, require a strong `SECRET_KEY`, verify the Google token audience, use the `secrets` module for OTP generation
- [ ] Unit tests for `parse_score`, PDF cleaning and the auth flow
- [ ] Deployment

## Author

**Abdullah Khan** · [GitHub](https://github.com/abdullahk88)
