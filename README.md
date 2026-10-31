# InternAI — AI-Powered Internship Assistant

InternAI is a complete, production-grade web application designed for university students to accelerate and protect their internship search. It provides structured AI job parsing, scam and risk indicator detection, semantic skill gap matching, dynamic weekly learning roadmaps, AI-tailored application generation, turn-by-turn mock interview coaching, and a Kanban application tracker.

---

## Complete Phase-by-Phase Architecture & Feature Walkthrough

### Phase 1: Project Scaffold, Database Layer, Auth, Profile & Base UI
- **Application Factory Pattern:** Constructed `create_app()` in `app/__init__.py` integrating Flask-WTF CSRF protection, Flask-Limiter rate limits, blueprint modularization, and centralized error handlers (400, 403, 404, 429, 500).
- **Dual Database Layer (`db.py`):**
  - Thin-Mode Oracle Database connection pool (`python-oracledb`) requiring zero Oracle client DLLs.
  - Automatic fallback to local SQLite (`internai.db`) for instant zero-config setup.
  - Parameterized bind queries exclusively (`query_one`, `query_all`, `execute`, `execute_returning_id`).
- **Database Schema (`schema.sql`):** 15 relational tables with constraints, identity columns, cascade deletion, and indexes.
- **Master Seed Catalog (`seed.py` / `seed.sql`):** 66 technical & soft skills across 6 categories and default administrator account.
- **Authentication (`app/blueprints/auth`):** Registration, login, logout, Werkzeug password hashing, session cookies, and `@login_required` / `@admin_required` decorators.
- **Student Profile (`app/blueprints/profile`):** University details, GPA, bio, skill proficiency ratings (Beginner/Intermediate/Advanced), and portfolio project manager.
- **Base UI:** Responsive custom CSS design system, sidebar navigation, toast notifications, and loading overlays.

### Phase 2: AI Job Description Analyzer & Explainable Skill Matcher
- **AI Service Foundation (`app/services/ai_service.py` & `prompts.py`):**
  - Google Gemini API integration with structured JSON output schemas (`response_mime_type="application/json"`).
  - Delimited prompt injection defenses (`<JOB_DESCRIPTION>`, `<STUDENT_PROFILE>`).
  - Exponential retry with backoff and deterministic heuristic fallbacks for offline testing.
- **Job Description Analyzer (`app/blueprints/analyzer`):**
  - Parses raw job postings into structured role summary, company, work type, location, stipend, duration, and requirements.
  - Persists data to `JOBS`, `JOB_REQUIREMENTS`, and `JOB_ANALYSIS`.
- **Explainable Skill Matcher (`app/blueprints/matcher`):**
  - Compares student profile and CV skills against job requirements.
  - Recognizes semantic equivalencies (e.g., ReactJS = React, Postgres = PostgreSQL).
  - **Deterministic Formula:** Computes match percentage mathematically in Python rather than letting AI hallucinate numbers.
  - Visual color-coded skill chips: Matched (Green), Critical Missing (Red), Nice-to-Have (Amber).

### Phase 3: AI Learning Roadmap & Dual-Engine Job Risk / Scam Analyzer
- **AI Learning Roadmap (`app/blueprints/roadmap`):**
  - Week-by-week prioritized syllabus generated from missing skills.
  - Tailored to student's weekly study time (e.g. 4 weeks @ 10 hrs/week).
  - Recommends real non-hallucinated resource types (official docs, courses, tutorials) and portfolio practice projects.
  - Interactive checkboxes with persistent completion progress saved to `ROADMAPS`.
- **Dual-Engine Job Risk Analyzer (`app/blueprints/risk`):**
  - **Deterministic Regex Engine:** Flags upfront payments/fees, Telegram/WhatsApp-only communications, personal banking/SSN requests, and free email domains (`@gmail.com`).
  - **Generative AI Risk Evaluator:** Flags vague company identity, missing interview steps, and pressure language.
  - Risk rating (`Safe`, `Low`, `Medium`, `High`) with quoted evidence from text, safety advice, and ethical disclaimers.
  - "Report this posting" workflow saving suspicious listings into `REPORTED_POSTINGS`.

### Phase 4: AI CV Analyzer & Application Assistant
- **CV Extraction Service (`app/services/cv_service.py`):**
  - Multi-format text extraction from PDF (`PyPDF2` / `pdfplumber`) and DOCX (`python-docx`).
  - Validates file type, MIME headers, and 5MB upload limit using `secure_filename`.
- **AI CV Analyzer (`app/blueprints/cv`):**
  - Evaluates resume sections and calculates a 0–100 score across 4 rubrics: Clarity, Relevance, Impact, and Formatting.
  - Suggests action-oriented rewritten bullet points with quantified outcomes.
  - "Import extracted skills to profile" with candidate confirmation.
- **Application Assistant (`app/blueprints/assistant`):**
  - Generates tailored cover letter drafts, professional outreach emails, and tailored interview prep questions.
  - Tone selector (`formal`, `friendly`, `confident`).
  - Strict anti-hallucination prompting instructions (never fabricates unlisted experience).
  - Copy-to-clipboard, in-browser editor, and persistence in `GENERATED_DOCUMENTS`.

### Phase 5: Turn-by-Turn AI Mock Interview & Kanban Internship Tracker
- **AI Mock Interview Coach (`app/blueprints/interview`):**
  - Interactive simulator for Technical, Behavioral, or Mixed formats (3, 5, or 8 questions).
  - Chat-like room with live word counter and STAR method coaching guidelines.
  - Turn-by-turn AI coaching evaluation: 1–10 score, strengths, improvements, and benchmark 10/10 model answers.
  - De-duplicated question progression and session resumption.
  - Performance report with overall readiness score, executive verdict, and printable transcript.
- **Internship Kanban Tracker (`app/blueprints/tracker`):**
  - 5-column board (`Saved` -> `Applied` -> `Interview` -> `Offer` -> `Closed`).
  - Vanilla JS HTML5 drag-and-drop with AJAX sync (`POST /tracker/update-status/<id>`).
  - Deadline urgency alerts (Overdue, Due Today, Due in 3d, Upcoming).
  - Real-time client-side search filter and full modal CRUD.
  - One-click "+ Add to Tracker" integrated directly into analyzed job pages.

### Phase 6: Admin Panel, Moderation, Hardening & Verification
- **Admin Center (`app/blueprints/admin`):**
  - Role-protected dashboard (`@admin_required`).
  - System KPI cards and Chart.js interactive visualizations.
  - User management: activate, deactivate, or delete accounts.
  - Moderation queue: review student scam reports and mark as reviewed or dismissed.
- **Test Suite:** 39 unit and integration tests across 5 test modules passing 100%.

---

## Database ERD & Schema Overview

```text
+------------------+       1:1       +--------------------+
|      USERS       |---------------->|  STUDENT_PROFILES  |
+------------------+                 +--------------------+
        |
        | 1:N
        +----------------------------+--------------------+--------------------+
        |                            |                    |                    |
        v                            v                    v                    v
+------------------+         +---------------+    +----------------+   +---------------+
|  STUDENT_SKILLS  |         |   PROJECTS    |    |  APPLICATIONS  |   |  ROADMAPS     |
+------------------+         +---------------+    +----------------+   +---------------+
        |                                                 |
        | N:1                                             |
        v                                                 |
+------------------+                                      |
|      SKILLS      |                                      |
+------------------+                                      |
                                                          |
+------------------+       1:N       +--------------------+
|      JOBS        |<----------------|
+------------------+
        |
        +---- 1:1 ----> JOB_ANALYSIS (Risk score, match %, JSON summary)
        |
        +---- 1:N ----> JOB_REQUIREMENTS (Normalized skills & importance)
        |
        +---- 1:N ----> CV_ANALYSIS (Extracted CV metrics & section breakdown)
        |
        +---- 1:N ----> INTERVIEW_SESSIONS
                               |
                               +---- 1:N ----> INTERVIEW_MESSAGES (Per-turn feedback)
        |
        +---- 1:N ----> GENERATED_DOCUMENTS (Cover letters, emails, prep)
        |
        +---- 1:N ----> REPORTED_POSTINGS (Scam/fraud flags for admin review)
```

---

## Directory Structure

```text
internai/
├── app/
│   ├── __init__.py                # Flask application factory, limiter, CSRF, error handlers
│   ├── blueprints/
│   │   ├── auth/                  # Register, login, logout, decorators
│   │   ├── profile/               # Student profile, skills CRUD, projects CRUD
│   │   ├── analyzer/              # AI Job Description Analyzer
│   │   ├── matcher/               # Explainable Skill Matcher
│   │   ├── roadmap/               # Learning Roadmap Generator
│   │   ├── risk/                  # Job Risk & Fraud Detector
│   │   ├── cv/                    # CV Text Extraction & Bullet Polisher
│   │   ├── assistant/             # Cover Letter & Email Generator
│   │   ├── interview/             # AI Turn-by-Turn Mock Interview
│   │   ├── tracker/               # Kanban Internship Tracker
│   │   └── admin/                 # Admin Dashboard & Moderation
│   ├── services/
│   │   ├── ai_service.py          # Gemini AI API calls, Pydantic schemas, heuristics
│   │   ├── cv_service.py          # PDF & DOCX text extraction
│   │   └── prompts.py             # Prompt engineering templates
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css          # Design system & responsive layout
│   │   └── js/
│   │       └── main.js            # CSRF AJAX helper, toasts, modals, search
│   └── templates/
│       ├── base.html              # Base shell with sidebar, header, toast container
│       ├── index.html             # Landing page & authenticated student dashboard
│       ├── auth/                  # Login & registration views
│       ├── profile/               # Profile, skills catalog, and project cards
│       ├── analyzer/              # Job analysis input & detailed review
│       ├── matcher/               # Match percentage & gap breakdown
│       ├── roadmap/               # Weekly study plan & interactive topic checklist
│       ├── risk/                  # Job risk assessment & scam reporting
│       ├── cv/                    # CV upload, scoring rubrics, and bullet editor
│       ├── assistant/             # Cover letter, outreach email, & prep kit generator
│       ├── interview/             # Lobby, interactive room, & summary report
│       ├── tracker/               # 5-column Kanban board with drag-and-drop
│       ├── admin/                 # Admin dashboard, user management, & review queue
│       └── errors/                # 400, 403, 404, 429, 500 pages
├── tests/
│   ├── test_phase1.py             # Auth & profile tests
│   ├── test_phase2.py             # Analyzer & matcher tests
│   ├── test_phase3.py             # Roadmap & risk tests
│   ├── test_phase4.py             # CV & application assistant tests
│   └── test_phase5.py             # Mock interview & Kanban tracker tests
├── uploads/                       # CV & attachment upload storage
├── .env.example                   # Environment configuration template
├── config.py                      # Application configurations
├── db.py                          # Oracle Thin mode connection pool & DB helpers
├── requirements.txt               # Dependencies
├── run.py                         # Application runner
├── schema.sql                     # Full Oracle Database DDL script
├── seed.py                        # Database seed runner (Skills catalog & Admin)
└── seed.sql                       # Pure Oracle SQL insert script
```

---

## Local Setup & Quickstart Guide

### 1. Install Dependencies
```powershell
pip install -r requirements.txt
```

### 2. Configure Environment (`.env`)
Copy `.env.example` to `.env`:
```powershell
Copy-Item .env.example .env
```
Ensure your Gemini API key is configured if you wish to query live AI models:
```text
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-2.5-flash
DB_BACKEND=auto
```

### 3. Seed Database
```powershell
python seed.py
```
This populates the 66 master skills catalog and initializes the default administrator account.

### 4. Run the Application
```powershell
python run.py
```
Open your browser at `http://127.0.0.1:5000`.

### 5. Default Credentials
- **Admin:** `admin@internai.local` | `Admin@123456`
- **Student:** Register a new student account at `http://127.0.0.1:5000/auth/register`.

---

## Running Automated Tests
```powershell
python -m unittest discover tests
```
All 39 tests across Phases 1 through 5 pass with zero errors.
