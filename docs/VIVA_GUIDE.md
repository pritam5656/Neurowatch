# NeuroWatch Viva Preparation Guide

This guide is written for final-year viva preparation. It explains the project as an examiner would inspect it: architecture, files, execution flow, frontend, backend, database, security, AI pipeline, tests, deployment, computer-science concepts, important code walkthroughs, and viva questions.

Quick project identity:

- Project name: NeuroWatch.
- Type: production-oriented full-stack web application.
- Frontend: Vite, React, TypeScript, React Router, Tailwind CSS, shadcn/Radix-style UI primitives.
- Backend: FastAPI, SQLAlchemy 2, Alembic, Pydantic v2.
- Auth: secure httpOnly JWT cookie, local username/password flow, optional Firebase token exchange.
- AI: optional OpenAI analysis enhancement and Whisper-based voice transcription; deterministic local fallback always exists.
- Database: SQLite for local development, Postgres/Neon-ready for production.
- Deployment: AWS Lightsail Ubuntu VM, nginx, systemd, Gunicorn with Uvicorn workers, GitHub Actions.

## Part 1 - Project Overview

### What This Project Does

NeuroWatch collects small behavioral signals from a user through guided digital tasks:

1. Typing rhythm.
2. Reaction timing.
3. Memory recall and recognition.
4. Voice pacing and articulation.

It converts those signals into metrics, compares them with the user's previous baseline when enough sessions exist, creates a structured risk-style awareness report, stores it, and shows it back in a dashboard.

Very important viva line: NeuroWatch is not a diagnostic medical device. It is an early behavioral awareness and longitudinal monitoring tool.

### Real-World Problem Solved

Neurological changes such as motor slowing, reaction delay, memory decline, or speech changes may appear gradually. People usually notice them late because there is no simple daily tool for structured self-check-ins. NeuroWatch provides a calm, repeatable, low-friction way to collect comparable behavioral data over time.

### Main Objective

The main objective is to build a secure, production-ready web application that:

- Authenticates users.
- Collects behavioral metrics.
- Stores session history.
- Builds personal baselines.
- Analyzes change over time.
- Generates patient-friendly and doctor-friendly reports.

### Target Users

- Patients or older adults who want regular self-monitoring.
- Caregivers who want structured check-in information.
- Doctors who need a summarized report before consultation.
- Students/researchers demonstrating a full-stack health-tech workflow.

### Core Features

- Account signup, login, logout, and session restoration.
- Protected dashboard and session routes.
- Typing, reaction, memory, and voice data collection.
- Optional voice audio upload.
- Baseline creation after seven sessions.
- Deterministic local analysis even without OpenAI.
- Optional OpenAI JSON analysis.
- Session list, detail, trends, and doctor report generation.
- PDF report download.
- Production deployment scripts.
- Backend unit and integration tests.

### End-to-End Workflow

User opens site -> React app loads -> AuthProvider checks `/api/v1/auth/me` -> user logs in or signs up -> secure JWT cookie is set -> user starts session -> frontend computes typing/reaction/memory metrics -> optional audio is recorded -> frontend sends multipart request to `/api/v1/sessions` -> backend validates metrics -> transcribes audio if OpenAI key exists -> builds baseline from previous sessions -> analyzes current session -> stores raw metrics and analysis in DB -> response returns analysis -> frontend navigates to result page -> user views domain scores, insights, reports, and trends.

### High-Level Architecture

The project uses a two-tier architecture:

- Frontend tier: browser SPA built with React and Vite.
- Backend tier: FastAPI app providing REST APIs, auth, database access, analysis, and PDF generation.

Production adds:

- nginx as reverse proxy and static file server.
- Gunicorn as process manager.
- Uvicorn workers to run FastAPI ASGI app.
- systemd for service supervision.
- Postgres/Neon as managed production database.

### Frontend-Backend Interaction

The frontend calls relative URLs like `/api/v1/auth/login`. In development, Vite proxies `/api` to `http://127.0.0.1:8000`. In production, nginx proxies `/api/` to FastAPI on `127.0.0.1:8000`. Cookies are sent by `fetch(..., credentials: "include")`.

### Why The Project Is Useful

It demonstrates:

- Real full-stack engineering.
- Secure authentication.
- Data validation.
- Longitudinal metrics.
- AI integration with fallback.
- PDF report generation.
- Production deployment.

This is stronger than a simple CRUD project because it has a real workflow, domain-specific analysis, and production concerns.

### Unique Features

- Four behavioral domains instead of one simple form.
- Baseline-driven comparison after seven sessions.
- Explicit missing-data handling.
- Patient-friendly wording and medical disclaimer.
- Optional OpenAI support but no hard dependency on AI.
- Doctor report and PDF generation.
- Local JWT and Firebase auth seam.

### Limitations

- It does not medically diagnose Parkinson's, dementia, or any disease.
- The scoring thresholds are heuristic, not clinically validated.
- Pitch variation is currently set to zero because Whisper does not provide robust pitch contours.
- Voice analysis depends on OpenAI key and audio quality.
- Frontend session metrics can be manipulated by a malicious user because they are computed client-side.
- No frontend automated tests are present.
- Rate limiting is IP-based and in-memory by default, which is limited for multi-instance scaling.

### Future Scope

- Clinically validated thresholds.
- Doctor dashboard.
- Caregiver invitation system.
- Wearable integration.
- Server-side metric verification where possible.
- Redis-backed distributed rate limiting.
- Background jobs for heavy audio/AI processing.
- Multi-language voice passages.
- More robust observability, audit logs, and alerts.

### Technologies Used And Why

- Vite: fast frontend dev server and optimized build.
- React: component-based UI and SPA routing.
- TypeScript: static typing for frontend contracts.
- React Router: browser routing and protected route nesting.
- Tailwind CSS: utility-first styling.
- Radix/shadcn primitives: accessible reusable UI building blocks.
- FastAPI: fast Python API framework with automatic validation and OpenAPI docs.
- Pydantic: request and response validation.
- SQLAlchemy: ORM layer for database models and queries.
- Alembic: schema migrations.
- SQLite: zero-friction local development DB.
- Postgres/Neon: production-ready relational DB.
- python-jose: JWT encode/decode.
- scrypt plus HMAC compare: password hashing and safe verification.
- SlowAPI: request rate limiting.
- OpenAI SDK: optional narrative analysis and Whisper transcription.
- ReportLab: PDF generation.
- pytest: backend test framework.
- nginx: reverse proxy, static serving, gzip, security headers.
- systemd: process supervision.
- Gunicorn plus Uvicorn: production ASGI serving.
- GitHub Actions: CI/CD.

### Full Execution Flow

1. Browser requests `/`.
2. In development, Vite serves `index.html`; in production, nginx serves `frontend/dist/index.html`.
3. `frontend/src/main.tsx` mounts React into `<div id="root">`.
4. `AuthProvider` runs and calls `getMe()`.
5. `getMe()` calls `GET /api/v1/auth/me` with `credentials: "include"`.
6. Backend `AuthMiddleware` reads cookie `neurowatch_auth`.
7. If cookie contains a valid JWT, middleware sets `request.state.auth_user`.
8. `get_current_user()` loads the user from DB.
9. If no user exists, frontend treats visitor as logged out.
10. User signs up or logs in.
11. Backend validates email/password using Pydantic.
12. Password is hashed with scrypt during signup.
13. Login verifies password using scrypt and constant-time comparison.
14. Backend creates JWT with `sub`, `email`, `display_name`, `iat`, `exp`, and `iss`.
15. Backend sets httpOnly cookie.
16. Frontend stores only user object in React state, not token.
17. User starts session.
18. React collects typing, reaction, memory, and optional voice audio.
19. Frontend sends `FormData` to `POST /api/v1/sessions`.
20. Backend validates JSON metrics using `SessionMetrics`.
21. If audio exists and OpenAI key exists, backend uses Whisper transcription.
22. Backend builds baseline from last seven sessions.
23. Backend calls analysis service.
24. If OpenAI key exists, it asks OpenAI for strict JSON; if not or if validation fails, local deterministic analysis is used.
25. Backend stores raw metrics and analysis JSON in the `sessions` table.
26. Backend returns session id and analysis.
27. Frontend navigates to `/session/:id`.
28. Result page fetches stored session detail.
29. Dashboard fetches sessions and trends.
30. Report buttons call report APIs and PDF download endpoint.

### Why This Architecture Is Production-Oriented

- Backend and frontend are split cleanly.
- Settings are environment-driven.
- Secrets are not committed.
- Database migrations exist.
- Tests run in CI before deploy.
- nginx serves static assets efficiently.
- Gunicorn manages worker processes.
- systemd restarts failed services.
- Deployment script includes migrations, restart, nginx validation, and smoke test.
- Security headers, cookie flags, rate limits, and `.env` permissions are considered.

### Why Monorepo Structure Is Used

A monorepo keeps frontend, backend, deployment, docs, and tests in one repository. This makes student/demo deployment simpler because one commit contains the whole system.

Advantages:

- One place for code.
- Shared version history.
- Easier CI/CD.
- API and UI changes can be committed together.

Disadvantages:

- Repository can become large.
- Frontend and backend dependency workflows differ.
- Larger teams may need more ownership rules.

### Advantages Of Splitting Frontend And Backend

- Frontend focuses on UI and browser interactions.
- Backend focuses on data, auth, validation, and business logic.
- Each tier can scale separately in future.
- API boundary is clear.
- Security-sensitive code stays on backend.

### 25 Project Overview Viva Questions

1. What is NeuroWatch?
   Answer: It is a full-stack behavioral monitoring web app that collects typing, reaction, memory, and voice signals and converts them into longitudinal awareness reports.
   Follow-up: Is it a medical diagnosis tool? No, it is awareness and decision support only.

2. What problem does it solve?
   Answer: It helps users track subtle behavioral changes over time in a structured way.
   Follow-up: Why not one-time testing? Because baseline and trends are more meaningful than isolated values.

3. Who are the users?
   Answer: Patients, caregivers, doctors, and health-tech reviewers.
   Follow-up: Which user is primary? The patient using the guided session.

4. Why collect typing metrics?
   Answer: Typing rhythm can reflect motor coordination, speed, and consistency.
   Follow-up: Can it diagnose motor disease? No, it can only show behavioral change.

5. Why collect reaction time?
   Answer: Reaction time reflects processing speed and attention consistency.
   Follow-up: What affects reaction time? Sleep, stress, device latency, age, fatigue.

6. Why collect memory metrics?
   Answer: Memory recall and recognition provide cognitive signal patterns.
   Follow-up: Why false positives? They show incorrect recognition behavior.

7. Why collect voice?
   Answer: Speech rate, pauses, and articulation may reveal pacing or clarity changes.
   Follow-up: Why is pitch limited? The current Whisper pipeline does not provide robust pitch contours.

8. What is the main architecture?
   Answer: React SPA frontend plus FastAPI backend plus relational database.
   Follow-up: Why not server-rendered pages? SPA gives smoother interactive task flows.

9. What is the API prefix?
   Answer: `/api/v1`.
   Follow-up: Why version APIs? It allows future `/api/v2` without breaking old clients.

10. How does frontend talk to backend?
    Answer: With `fetch` calls to REST endpoints and cookies included.
    Follow-up: Why include credentials? Because JWT is stored in an httpOnly cookie.

11. What data is stored?
    Answer: Users, raw session metrics, generated analysis, session timestamps.
    Follow-up: Why store raw and analysis? Raw data allows re-analysis; analysis supports fast display.

12. What is a baseline?
    Answer: Average of recent personal session metrics after enough sessions.
    Follow-up: Why personal baseline? Users naturally differ, so self-comparison is better.

13. Why seven sessions?
    Answer: It gives a minimum history before treating averages as stable.
    Follow-up: Is seven clinically proven? No, it is an engineering threshold in this project.

14. Why deterministic fallback?
    Answer: The app should work even without OpenAI or if AI output fails validation.
    Follow-up: Is fallback safer? It is predictable and testable.

15. Why use OpenAI optionally?
    Answer: To enhance narrative analysis, but not as a hard dependency.
    Follow-up: What risk does AI add? Cost, latency, invalid JSON, hallucination, and availability risk.

16. Why FastAPI?
    Answer: It gives Python productivity, strong validation, async support, and OpenAPI docs.
    Follow-up: Alternative? Django REST Framework, Flask, Express, Spring Boot.

17. Why React?
    Answer: The frontend needs interactive stateful tasks, and React is good for component-based UI.
    Follow-up: Alternative? Vue, Angular, Svelte.

18. Why SQLAlchemy?
    Answer: It maps Python classes to tables and avoids repeated raw SQL.
    Follow-up: Does ORM remove SQL knowledge? No, developers still need DB understanding.

19. Why Alembic?
    Answer: It versions database schema changes.
    Follow-up: Why not `create_all` in production? Migrations are controlled and reversible.

20. Why cookie JWT instead of localStorage?
    Answer: httpOnly cookies reduce token theft by JavaScript XSS.
    Follow-up: What risk remains? CSRF must be considered for cookie auth.

21. Why nginx?
    Answer: It serves static files, proxies API, adds headers, handles gzip/TLS.
    Follow-up: Why not expose Uvicorn directly? Uvicorn should not directly handle all production edge concerns.

22. Why Gunicorn plus Uvicorn?
    Answer: Gunicorn manages multiple worker processes; Uvicorn workers run the ASGI app.
    Follow-up: What is ASGI? Python async server gateway interface.

23. Why GitHub Actions?
    Answer: It automates tests, build, and deployment on pushes.
    Follow-up: Why CI/CD matters? It reduces manual deployment mistakes.

24. What is the biggest limitation?
    Answer: The analysis thresholds are heuristic and not clinically validated.
    Follow-up: How to improve? Collect validated datasets and clinician-reviewed thresholds.

25. How would you explain the project in one minute?
    Answer: NeuroWatch is a secure full-stack app that lets a user perform short digital tasks, calculates behavioral metrics, compares them with personal baseline, optionally uses AI for structured analysis, stores results, and provides dashboard trends and doctor reports.
    Follow-up: What makes it production-ready? Auth, validation, migrations, tests, deployment, and security hardening.

## Part 2 - Complete Folder And File Structure Analysis

### Top-Level Structure

- `.cursor/`: editor/tool settings.
- `.github/workflows/`: CI/CD pipeline.
- `backend/`: FastAPI application, models, schemas, services, middleware, migrations, tests.
- `deploy/lightsail/`: production VM scripts and config templates.
- `docs/`: deployment docs and this viva guide.
- `frontend/`: Vite React SPA.
- `guidelines/`: Figma Make guideline placeholder.
- `.env.example`: root environment template.
- `.gitignore`: prevents secrets/build artifacts from being committed.
- `package.json`: monorepo shell scripts.
- `pytest.ini`: pytest configuration.
- `README.md`: project setup and deployment overview.

### Why Folders Are Separated

Separation keeps responsibilities clear:

- `api/` handles HTTP routes.
- `schemas/` handles input/output validation.
- `models/` handles database tables.
- `services/` handles business logic.
- `middleware/` handles cross-cutting request behavior.
- `core/` handles shared settings, logging, security.
- `db/` handles engine and sessions.
- `frontend/src/app/pages/` handles route-level screens.
- `frontend/src/app/components/` handles reusable UI pieces.
- `deploy/` keeps server automation separate from app code.

### Every File Inventory

Use this section for "what is this file?" questions. For line-by-line detail, see Part 11 for the most important files.

| File | Purpose and viva explanation |
|---|---|
| `.cursor/settings.json` | Local editor/plugin settings. Enables tools like superpowers and Firebase in the editor environment. Not used by runtime. |
| `.env.example` | Root environment template. Shows database, JWT, OpenAI, CORS, local/Firebase auth, and Vite Firebase variables. Exists so secrets are configured locally without committing `.env`. |
| `.github/workflows/deploy-lightsail.yml` | GitHub Actions pipeline. On push to `main`, checks out code, sets up Python/Node, installs deps, runs integration tests, builds frontend, SSHes to Lightsail, pulls code, and runs deploy script. |
| `.gitignore` | Excludes `node_modules`, `dist`, `.env`, local DB files, Python caches, backend venv, coverage, and deploy metadata. Security reason: prevents secrets and generated artifacts from entering Git. |
| `README.md` | Main project overview, setup, local run, tests, production build, Lightsail deployment, and security note. First file to explain in viva. |
| `package.json` | Monorepo command wrapper. Runs frontend dev, backend dev, frontend build, backend tests, and combined dev server. |
| `pytest.ini` | Tells pytest to use `backend/tests` and auto async mode. |
| `guidelines/Guidelines.md` | Placeholder for design generation rules. Useful if using Figma Make, not part of runtime. |
| `docs/lightsail-deployment.md` | Detailed deployment manual for Lightsail, nginx, systemd, Gunicorn, Neon, TLS, CI/CD, hardening, and troubleshooting. |
| `docs/VIVA_GUIDE.md` | This viva guide. |
| `backend/.env.example` | Backend-only environment template. Smaller than root template; includes DB, JWT, OpenAI, CORS, and auth provider. |
| `backend/README.md` | Backend stack, run commands, migration command, API prefix, and auth provider seam. |
| `backend/requirements.txt` | Python dependencies: FastAPI, Uvicorn, Gunicorn, Pydantic, SQLAlchemy, Alembic, auth libs, OpenAI, Firebase Admin, httpx, pytest, ReportLab. |
| `backend/Procfile` | Platform-as-a-service start command: `uvicorn app.main:app --host 0.0.0.0 --port ${PORT}`. Useful for Heroku-style deployments. |
| `backend/alembic.ini` | Alembic configuration: migration script location, Python path, default SQLAlchemy URL, logging. |
| `backend/alembic/env.py` | Alembic runtime environment. Imports app settings, Base metadata, and models, then runs migrations online/offline. Connects migrations to real models. |
| `backend/alembic/versions/0001_init.py` | Initial migration. Creates `users` and `sessions` tables plus indexes and foreign key. Downgrade drops them. |
| `backend/app/__init__.py` | Marks `app` as a Python package. Empty but important for imports. |
| `backend/app/main.py` | FastAPI entry point. Configures logging, middleware, exception handlers, startup DB init, API routes, and static SPA serving. |
| `backend/app/api/__init__.py` | Marks API package. Empty. |
| `backend/app/api/deps.py` | FastAPI dependencies for current user and required user. Bridges middleware auth state to database user. |
| `backend/app/api/v1/__init__.py` | Creates `api_router` and includes health, auth, and sessions routers. |
| `backend/app/api/v1/auth.py` | Auth endpoints: signup, login, Firebase login, logout, me. Sets/deletes httpOnly JWT cookie. Rate-limited. |
| `backend/app/api/v1/health.py` | Health endpoint returning `{"status":"ok"}`. Used by smoke tests and deployment checks. |
| `backend/app/api/v1/sessions.py` | Session endpoints: create, list, trends, detail, doctor report, PDF report. Handles multipart metrics/audio flow. |
| `backend/app/core/__init__.py` | Marks core package. Empty. |
| `backend/app/core/config.py` | Pydantic settings. Reads `.env`, normalizes Postgres URLs, configures secure cookies, Firebase key normalization, CORS list parsing. |
| `backend/app/core/logging.py` | Basic logging configuration. Gives timestamp, level, logger, message. |
| `backend/app/core/security.py` | Password hashing/verification and JWT create/decode helpers. Uses scrypt, HMAC compare, HS256 JWT. |
| `backend/app/db/__init__.py` | Marks DB package. Empty. |
| `backend/app/db/base.py` | SQLAlchemy declarative base class used by models and migrations. |
| `backend/app/db/init_db.py` | Creates tables automatically only for SQLite local development. Production should use Alembic. |
| `backend/app/db/session.py` | Creates database engine and `SessionLocal`; provides `get_db()` dependency. Handles SQLite special settings and in-memory test pooling. |
| `backend/app/middleware/__init__.py` | Marks middleware package. Empty. |
| `backend/app/middleware/auth.py` | Reads `neurowatch_auth` cookie, verifies token, stores auth user on request state. |
| `backend/app/middleware/errors.py` | Standard JSON error responses for HTTP, validation, and unexpected exceptions. |
| `backend/app/middleware/rate_limit.py` | SlowAPI limiter and 429 response structure. |
| `backend/app/middleware/request_id.py` | Adds or propagates request id for traceability. |
| `backend/app/models/__init__.py` | Imports model classes so Alembic/Base can see metadata. |
| `backend/app/models/user.py` | `User` table model: id, email, display name, password hash, timestamps, sessions relationship. |
| `backend/app/models/session.py` | `Session` table model: id, user id, number, raw metrics JSON, analysis JSON, created at, relationship. |
| `backend/app/schemas/__init__.py` | Re-exports schema classes for convenient imports. |
| `backend/app/schemas/auth.py` | Auth Pydantic schemas. Validates email, password min length, display name, Firebase id token. |
| `backend/app/schemas/metrics.py` | Metric schemas for typing, reaction, memory, voice, and full session. Enforces numeric ranges. |
| `backend/app/schemas/analysis.py` | Analysis response schema. Enforces risk score, risk level, domain scores, recommendations, disclaimer. |
| `backend/app/schemas/session.py` | Session list/detail/trends/report response schemas. Defines doctor report shape. |
| `backend/app/services/__init__.py` | Marks services package. Empty. |
| `backend/app/services/auth_provider.py` | Auth abstraction, local auth provider, Firebase token verification, Firebase user upsert, provider seam. |
| `backend/app/services/metrics.py` | Pure functions to derive typing, reaction, memory, and merged voice metrics. Mirrors frontend logic for tests/server use. |
| `backend/app/services/baseline.py` | Builds personal baseline from last seven sessions by averaging domain metrics. |
| `backend/app/services/analysis.py` | Core analysis engine. Computes domain risk scores locally, optionally calls OpenAI JSON analysis, validates output, falls back deterministically. |
| `backend/app/services/prompts.py` | System prompt for OpenAI behavioral analysis. Contains rules, thresholds, scoring framework, JSON-only requirement. |
| `backend/app/services/voice.py` | Whisper transcription and voice metric derivation: speech rate, pause frequency, pause duration, articulation score, data quality notes. |
| `backend/app/services/reporting.py` | Doctor report and PDF generation. Sanitizes technical notes and formats sections for clinician review. |
| `backend/scripts/run_dev.ps1` | Windows helper to run backend dev server. |
| `backend/scripts/run_dev.sh` | Unix helper to run backend dev server. |
| `backend/tests/conftest.py` | Test fixtures. Sets in-memory SQLite, test JWT secret, no OpenAI key, overrides DB dependency. |
| `backend/tests/integration/test_auth_api.py` | Tests signup, me, logout, login, and Firebase token route with monkeypatch. |
| `backend/tests/integration/test_sessions_api.py` | Tests protected sessions, create/list/trends/detail/report/PDF, and voice failure fallback. |
| `backend/tests/unit/test_security.py` | Tests password hashing and verification. |
| `backend/tests/unit/test_metrics.py` | Tests typing/reaction/memory metric derivation. |
| `backend/tests/unit/test_analysis.py` | Tests deterministic analysis returns valid schema and disclaimer. |
| `backend/tests/unit/test_baseline.py` | Tests baseline is unavailable before seven sessions and averaged after seven. |
| `backend/tests/unit/test_reporting.py` | Tests doctor report sanitizes technical errors and PDF bytes are produced. |
| `backend/tests/unit/test_voice.py` | Tests speech rate, pause metrics, articulation, and timestamp extraction without confidence. |
| `deploy/lightsail/bootstrap.sh` | One-time Ubuntu server setup. Installs system packages, Node, nginx, certbot, ufw, fail2ban, unattended upgrades. |
| `deploy/lightsail/deploy.sh` | Main release script. Installs deps, builds frontend, runs migrations, installs systemd/nginx, restarts app, smoke-tests health. |
| `deploy/lightsail/deploy-production.sh` | Pulls latest branch and runs deploy; records successful commit. |
| `deploy/lightsail/rollback.sh` | Rolls back to previous successful commit or specified ref, rebuilds/restarts, records rollback. |
| `deploy/lightsail/enable-ssl.sh` | Uses certbot nginx plugin to issue Let's Encrypt TLS certificate, redirects HTTP to HTTPS, tests renewal. |
| `deploy/lightsail/neurowatch.service` | systemd unit template. Runs Gunicorn with Uvicorn workers, sets env, restart policy, sandboxing, logs. |
| `deploy/lightsail/nginx.neurowatch.conf` | nginx site template. Serves SPA, proxies `/api`, sets gzip, security headers, caching, upload size, blocks dotfiles. |
| `frontend/ATTRIBUTIONS.md` | License attribution for shadcn/ui and Unsplash. |
| `frontend/default_shadcn_theme.css` | Default shadcn theme variables. Reference styling file. |
| `frontend/index.html` | HTML shell with `<div id="root">` and module script loading React entry. |
| `frontend/package.json` | Frontend dependencies and scripts. Includes Vite, React, Firebase, Radix UI, Recharts, lucide, Tailwind. |
| `frontend/package-lock.json` | Exact npm dependency lockfile. Ensures reproducible installs. Do not explain line by line; it is generated. |
| `frontend/postcss.config.mjs` | Empty PostCSS config because Tailwind v4 Vite plugin handles setup. |
| `frontend/tailwind.config.ts` | Tailwind theme extension. Some content paths are legacy-ish, but Tailwind v4 source import also scans files. |
| `frontend/tsconfig.json` | TypeScript compiler settings. Strict mode, JSX transform, bundler module resolution. Includes some leftover Next plugin/include entries but Vite still builds TS through bundler. |
| `frontend/vite.config.ts` | Vite config. React plugin, Tailwind plugin, env dir at repo root, alias `@`, dev proxy for `/api`, Figma asset resolver. |
| `frontend/types/vendor.d.ts` | Declares modules for `fluent-ffmpeg` and `node-wav`; currently not used in frontend runtime. |
| `frontend/src/main.tsx` | React entry. Mounts `AuthProvider`, `BrowserRouter`, and `App`. Imports global CSS. |
| `frontend/src/styles/index.css` | Imports fonts, Tailwind, theme, globals in one place. |
| `frontend/src/styles/fonts.css` | Empty placeholder for font imports. |
| `frontend/src/styles/tailwind.css` | Tailwind v4 import and source scanning. |
| `frontend/src/styles/theme.css` | CSS variables and base layer for shadcn theme, light/dark tokens, typography defaults. |
| `frontend/src/styles/globals.css` | App background, root height, breathing animation. |
| `frontend/src/app/App.tsx` | Main route table. Public routes, protected routes inside `RequireAuth`, 404, and `/home` redirect. |
| `frontend/src/app/routes/RequireAuth.tsx` | Protected route wrapper. Shows loading while restoring session, redirects unauthenticated users to `/login`. |
| `frontend/src/app/lib/api.ts` | API client. Handles JSON requests, cookie credentials, session multipart upload, report/PDF download. |
| `frontend/src/app/lib/auth-context.tsx` | React auth context. Holds user state, restores session, local login/signup, optional Firebase email/Google auth, logout. |
| `frontend/src/app/lib/firebase.ts` | Initializes Firebase Web SDK only if all Vite Firebase config variables exist. |
| `frontend/src/app/lib/types.ts` | TypeScript mirror of backend response/data schemas. Keeps frontend type-safe. |
| `frontend/src/app/lib/metrics.ts` | Frontend metric derivation for typing, reaction, memory, and voice merge. |
| `frontend/src/app/data/session-content.ts` | Static prompts: typing passage, memory words, sequence pattern, voice reading passage. |
| `frontend/src/app/data/mockData.ts` | Dashboard preview data used when user has no real sessions. Also provides sample metric rows. |
| `frontend/src/app/pages/Landing.tsx` | Home page. Shows patient-first pitch and routes based on auth state. |
| `frontend/src/app/pages/Login.tsx` | Login page wrapper around `AuthCard`. |
| `frontend/src/app/pages/Signup.tsx` | Signup page wrapper around `AuthCard`. |
| `frontend/src/app/pages/Dashboard.tsx` | Dashboard page. Fetches sessions/trends, shows risk, domain cards, charts, history, insights, detail views. |
| `frontend/src/app/pages/Session.tsx` | Session page shell. Explains four-stage flow and renders `SessionFlow`. |
| `frontend/src/app/pages/SessionResult.tsx` | Session result page. Fetches session by id, shows analysis, generates/copies/emails/downloads doctor report. |
| `frontend/src/app/pages/NotFound.tsx` | 404 page with link home. |
| `frontend/src/app/components/AppShell.tsx` | Shared app layout/header/nav. Uses auth state to show login/signup or dashboard/session/sign out. |
| `frontend/src/app/components/AuthCard.tsx` | Login/signup form. Handles email/password and optional Google sign-in. Displays auth errors. |
| `frontend/src/app/components/DomainCard.tsx` | Domain summary card for typing/reaction/memory/voice. Shows score, trend, flags, observation. |
| `frontend/src/app/components/RiskIndicator.tsx` | Large overall risk display with level colors and trend icon. |
| `frontend/src/app/components/InsightsPanel.tsx` | Shows summary, positives, watch areas, recommendations, next focus, caregiver alert message. |
| `frontend/src/app/components/MetricDetail.tsx` | Detailed metric rows for a selected domain, with status icons and flags. |
| `frontend/src/app/components/SessionHistory.tsx` | Scrollable session history list with score, risk level, date, trend. |
| `frontend/src/app/components/TrendChart.tsx` | Recharts area/line chart for overall and domain risk trends. |
| `frontend/src/app/components/figma/ImageWithFallback.tsx` | Image component that swaps to embedded fallback image on load error. |
| `frontend/src/app/components/session/SessionFlow.tsx` | State machine for the full guided session. Collects all domain metrics and submits to backend. |
| `frontend/src/app/components/session/SessionShell.tsx` | Shared card wrapper for each session step. |
| `frontend/src/app/components/session/PreSessionCalm.tsx` | Intro breathing/calming screen before tests. |
| `frontend/src/app/components/session/InterTestRest.tsx` | Timed rest between tests. Uses interval and auto-continue. |
| `frontend/src/app/components/session/TypingTest.tsx` | Captures typed text, keydown/up timings, duration, and derives typing metrics. |
| `frontend/src/app/components/session/ReactionTest.tsx` | Randomized stimulus task. Measures hits, misses, anticipation, reaction time. |
| `frontend/src/app/components/session/MemoryTest.tsx` | Multi-stage memory test: study, recall, recognition, sequence study, sequence recall. |
| `frontend/src/app/components/session/VoiceTest.tsx` | Uses browser `MediaRecorder` to capture audio blob or skip voice. |

### UI Component Library Files

The `frontend/src/app/components/ui/` folder contains reusable shadcn/Radix-style primitives. Most are not domain-specific; they wrap accessible components and styling conventions. Viva answer: these files exist to avoid rewriting UI behavior like dialogs, menus, tabs, sliders, forms, sidebars, and tooltips from scratch.

| UI file | Purpose |
|---|---|
| `accordion.tsx` | Expand/collapse vertical content sections. |
| `alert.tsx` | Styled alert message with title/description. |
| `alert-dialog.tsx` | Accessible confirmation modal for destructive or important actions. |
| `aspect-ratio.tsx` | Maintains fixed media aspect ratios. |
| `avatar.tsx` | User/image avatar with fallback. |
| `badge.tsx` | Small status labels. |
| `breadcrumb.tsx` | Hierarchical navigation trail. |
| `button.tsx` | Variant-based button component using class variance authority. |
| `calendar.tsx` | Date picker calendar wrapper. |
| `card.tsx` | Card layout primitives. |
| `carousel.tsx` | Carousel/slider UI, based on embla. |
| `chart.tsx` | Chart helpers and theme config for Recharts. |
| `checkbox.tsx` | Accessible checkbox primitive. |
| `collapsible.tsx` | Collapsible content primitive. |
| `command.tsx` | Command palette/search UI. |
| `context-menu.tsx` | Right-click/context menu. |
| `dialog.tsx` | Generic accessible modal dialog. |
| `drawer.tsx` | Mobile bottom/side drawer, likely via Vaul. |
| `dropdown-menu.tsx` | Dropdown menu primitive. |
| `form.tsx` | React Hook Form helpers for labels/messages/control. |
| `hover-card.tsx` | Hover-triggered information panel. |
| `input.tsx` | Styled text input. |
| `input-otp.tsx` | One-time password input layout. |
| `label.tsx` | Accessible label component. |
| `menubar.tsx` | Horizontal menu bar. |
| `navigation-menu.tsx` | Navigation menu primitive. |
| `pagination.tsx` | Pagination controls. |
| `popover.tsx` | Floating popover content. |
| `progress.tsx` | Progress bar. |
| `radio-group.tsx` | Radio option group. |
| `resizable.tsx` | Resizable panel group. |
| `scroll-area.tsx` | Styled scrollable area. |
| `select.tsx` | Select/dropdown input. |
| `separator.tsx` | Visual divider. |
| `sheet.tsx` | Side sheet modal. |
| `sidebar.tsx` | Full sidebar system with provider, trigger, rail, menu, groups, and responsive behavior. |
| `skeleton.tsx` | Loading placeholder. |
| `slider.tsx` | Range slider input. |
| `sonner.tsx` | Toast notification provider. |
| `switch.tsx` | Toggle switch. |
| `table.tsx` | Table primitives. |
| `tabs.tsx` | Tabbed UI. |
| `textarea.tsx` | Styled multiline input. |
| `toggle.tsx` | Toggle button. |
| `toggle-group.tsx` | Group of toggle buttons. |
| `tooltip.tsx` | Hover/focus tooltip. |
| `use-mobile.ts` | Hook for mobile breakpoint detection. |
| `utils.ts` | `cn()` utility combines `clsx` and `tailwind-merge` for class names. |

Common mistake: do not claim every UI primitive is used by NeuroWatch screens. Many are available reusable primitives from the design system, but the domain-specific screens mostly use custom Tailwind markup and a few libraries like Recharts and lucide icons.

## Part 3 - Frontend Complete Explanation

### Vite

What it is: Vite is a modern frontend build tool and dev server.

Why used: It starts quickly, supports hot module replacement, handles TypeScript/React, and builds optimized static assets.

How it works internally: In development, Vite serves source files as native ES modules and transforms only what the browser requests. In production, it bundles and optimizes with Rollup.

Why chosen: This project has an SPA with React and TypeScript. Vite is lighter and faster than older Create React App.

Alternatives: Create React App, Next.js, Parcel, Webpack.

Advantages:

- Fast startup.
- Fast hot reload.
- Simple config.
- Good TypeScript and React support.

Disadvantages:

- SPA needs backend/nginx fallback for deep links.
- Some older ecosystem assumptions may target Webpack.

### React SPA Architecture

What it is: A single-page app loads one HTML page and changes views in the browser without full page reloads.

How it works here:

- `index.html` loads `/src/main.tsx`.
- `main.tsx` renders React into `#root`.
- `BrowserRouter` enables URL-based routing.
- `App.tsx` defines pages.
- `RequireAuth` protects private routes.

Why it fits: Session tasks need smooth state transitions, timers, media recording, and no page reload.

### Component-Based Architecture

The UI is divided into components:

- Pages: route-level screens such as Dashboard, Session, Login.
- Domain components: RiskIndicator, DomainCard, TrendChart.
- Session components: TypingTest, ReactionTest, MemoryTest, VoiceTest.
- UI primitives: buttons, dialogs, tabs, tooltips, etc.

Why it matters: Components isolate responsibility, make UI easier to reason about, and allow reuse.

### Routing

Routes in `App.tsx`:

- `/`: Landing.
- `/login`: Login.
- `/signup`: Signup.
- `/dashboard`: protected.
- `/session`: protected.
- `/session/:id`: protected result page.
- `*`: Not found.
- `/home`: redirect to `/`.

Protected route flow:

1. `RequireAuth` reads `user` and `loading` from auth context.
2. If loading, shows restoration message.
3. If no user, redirects to `/login` and stores original path.
4. If user exists, renders nested route with `<Outlet />`.

### State Management

This project uses React local state and context, not Redux.

- `AuthProvider` holds global auth state.
- Page components hold their own loading/data/error state.
- SessionFlow holds step and collected metrics state.

Why no Redux: The state is not complex enough to justify global store overhead.

### API Calling

`api.ts` centralizes all backend calls.

Important points:

- Uses `fetch`.
- Uses `credentials: "include"` so browser sends auth cookie.
- JSON APIs use `Content-Type: application/json`.
- Session creation uses `FormData`, so no manual JSON content type.
- Error envelope is parsed and converted to thrown `Error`.

### Form Handling

Auth forms use controlled inputs:

- `displayName`, `email`, `password` use `useState`.
- `onSubmit` prevents default form reload.
- `submitting` disables actions.
- `error` displays user-friendly failures.

### Authentication Handling

Frontend does not store JWT manually. Backend stores token in httpOnly cookie. Frontend only stores the returned user object in React state.

Local auth:

1. User submits email/password.
2. `AuthCard` calls `useAuth().signIn`.
3. Auth context calls API `/auth/login`.
4. Backend sets cookie.
5. Auth context sets `user`.
6. UI redirects to dashboard or original protected page.

Firebase auth:

1. Firebase Web SDK signs in user.
2. Frontend obtains Firebase ID token.
3. Frontend sends ID token to backend `/auth/firebase`.
4. Backend verifies ID token and creates local JWT cookie.
5. From that point, protected backend APIs still use the same cookie flow.

### Session Handling

On app startup:

1. `AuthProvider` `useEffect` runs.
2. If Firebase redirect result exists, it exchanges Firebase token.
3. Otherwise it calls `refreshUser()`.
4. `refreshUser()` calls `/auth/me`.
5. On success, user is restored; on failure, user is null.
6. Loading becomes false.

### UI Rendering Flow

Dashboard rendering:

1. `DashboardPage` mounts.
2. `useEffect` fetches sessions and trends in parallel.
3. If live data exists, it uses backend data.
4. If no live data exists, it uses `mockData` preview.
5. `currentView` controls whether dashboard, trends, history, insights, or domain detail is shown.

Session rendering:

1. `SessionPage` renders intro and `SessionFlow`.
2. `SessionFlow` shows one step at a time.
3. Completed metrics are stored in state.
4. Final step submits to backend.
5. Browser navigates to result page.

### Build Process

Development:

- `npm run dev` in frontend runs Vite.
- Root `npm run dev` runs frontend and backend through frontend script `dev:all`.
- Vite proxies `/api` to backend.

Production:

- `npm --prefix frontend run build`.
- Vite generates `frontend/dist`.
- nginx serves `frontend/dist`.
- FastAPI can also mount `frontend/dist` if present.

### Frontend Optimization

- Vite bundles static assets.
- nginx caches hashed `/assets/*` immutably.
- React state avoids unnecessary full page reloads.
- Dashboard fetches sessions and trends concurrently with `Promise.all`.
- Recharts uses responsive container for chart sizing.

### React Hooks Used

`useState`: stores component state like forms, loading, step, metrics, timers, errors.

`useEffect`: runs side effects like fetching data, restoring session, timers, and reaction scheduling.

`useMemo`: memoizes derived values like session metrics object, latest raw metrics, expected words, recognition questions.

`useCallback`: keeps stable auth functions so context value and effects do not change unnecessarily.

`useContext`: reads auth context through `useAuth`.

`useRef`: stores mutable values that should not trigger re-render, such as key events, timers, MediaRecorder, audio chunks, stimulus timestamps.

Custom hook: `useAuth` wraps `useContext(AuthContext)` and ensures usage inside provider.

### Component Details

`AppShell`: receives `children`; reads `user` and `signOut`; renders shared header/nav; navigates to login after sign out.

`AuthCard`: props `mode`; state for fields/submitting/error; calls local or Firebase auth; maps Firebase errors to readable messages.

`LandingPage`: reads auth state; shows different CTA for logged-in or logged-out user.

`DashboardPage`: state for current view, menu, loading, sessions, trends, latest. Fetches data once. Uses mock fallback if no sessions. Converts backend objects into UI chart/history rows.

`SessionFlow`: central session state machine. Holds metrics for each domain and current step. Submits full metrics plus audio blob.

`TypingTest`: uses timer, keydown/up refs, typed text, and derives typing metrics.

`ReactionTest`: uses randomized delays, active stimulus state, trial results, and derives reaction metrics.

`MemoryTest`: staged memory task; computes recall, recognition, and sequence scores.

`VoiceTest`: uses `navigator.mediaDevices.getUserMedia` and `MediaRecorder`; returns audio blob or null.

`SessionResultPage`: uses URL param id; fetches session; can generate doctor report, copy report text, download PDF, or open email link.

`TrendChart`: renders Recharts AreaChart/Line components.

`DomainCard`, `RiskIndicator`, `MetricDetail`, `InsightsPanel`, `SessionHistory`: presentational components receiving props and rendering styled UI.

### Frontend Viva Questions

1. Why Vite instead of CRA?
   Answer: Vite is faster, simpler, and modern; CRA is older and heavier.

2. What is SPA routing?
   Answer: Browser URL changes while React swaps components without full reload.

3. How are protected routes implemented?
   Answer: `RequireAuth` checks auth context and redirects unauthenticated users.

4. Where is JWT stored in frontend?
   Answer: It is not accessible to JS; it is stored as an httpOnly cookie by backend.

5. Why use `credentials: "include"`?
   Answer: To send cookies with fetch requests.

6. Why use `FormData` for session creation?
   Answer: It sends JSON metrics plus optional audio file in one multipart request.

7. Why use `useRef` in typing/reaction/voice?
   Answer: To store mutable values like key timing, timers, recorder without re-rendering.

8. Why no Redux?
   Answer: App state is simple and localized; context plus local state is enough.

9. What happens if user reloads `/session/abc` in production?
   Answer: nginx falls back to `index.html`; React Router renders the route.

10. How does frontend know user is logged in after refresh?
    Answer: AuthProvider calls `/auth/me` using the cookie.

## Part 4 - Backend Complete Explanation

### FastAPI Architecture

FastAPI maps Python functions to HTTP endpoints. It uses function signatures, type hints, Pydantic schemas, and dependency injection to parse requests, validate data, and produce responses.

Why chosen:

- Python fits AI/data workflows.
- FastAPI is concise and fast.
- Built-in OpenAPI docs help viva/demo.
- Pydantic validation reduces manual checks.

Alternatives: Flask, Django REST Framework, Express, Spring Boot.

### ASGI vs WSGI

WSGI is the older Python web server interface for synchronous apps. ASGI supports async requests, WebSockets, and long-lived connections. FastAPI is ASGI. Uvicorn is an ASGI server. Gunicorn manages worker processes and uses Uvicorn workers.

### Pydantic

Pydantic validates and serializes data using Python type hints.

Example: `EmailStr` ensures valid email; `Field(ge=0, le=100)` ensures scores are valid.

Why important: It prevents invalid request shapes from reaching business logic.

### SQLAlchemy

SQLAlchemy maps Python classes to DB tables. `User` and `Session` are ORM models. It provides query APIs like `select(User).where(...)`.

Why used: Safer and more maintainable than hand-written SQL everywhere.

### Alembic

Alembic versions schema changes. `0001_init.py` creates initial tables. In production, deploy runs `alembic upgrade head`.

### Middleware Flow

Middleware wraps requests before they reach routes:

- CORS handles browser cross-origin rules.
- RequestIdMiddleware attaches request id.
- AuthMiddleware verifies cookie and stores `auth_user`.
- SlowAPIMiddleware enforces rate limits.
- Exception handlers shape errors.

FastAPI/Starlette middleware execution order can be subtle because each middleware wraps the previous stack. Viva-safe answer: the app registers CORS, request id, auth, and rate limiting so requests pass through cross-cutting layers before/around endpoint execution; the exact wrap order is less important than each layer's responsibility.

### Request Lifecycle

Browser -> nginx/Vite proxy -> FastAPI app -> middleware -> route matching -> dependency injection -> Pydantic validation -> route function -> service layer -> database/session -> schema response -> exception handler if needed -> JSON response -> frontend.

### Dependency Injection

FastAPI dependencies are functions declared with `Depends`.

- `get_db`: provides DB session.
- `get_current_user`: reads request auth state and loads DB user.
- `require_user`: raises 401 if no user.

Why used: It avoids manually creating DB sessions and auth checks in every route.

### Exception Handling

`errors.py` standardizes error JSON:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed.",
    "details": [...]
  }
}
```

This gives frontend consistent errors.

### APIs

| Endpoint | Method | Auth | Request | Response |
|---|---|---|---|---|
| `/api/v1/health` | GET | No | none | `{"status":"ok"}` |
| `/api/v1/auth/signup` | POST | No | display_name, email, password | user; sets cookie |
| `/api/v1/auth/login` | POST | No | email, password | user; sets cookie |
| `/api/v1/auth/firebase` | POST | No | Firebase id_token | user; sets cookie |
| `/api/v1/auth/logout` | POST | No | none | `{"ok": true}`; clears cookie |
| `/api/v1/auth/me` | GET | Cookie optional but required logically | none | current user or 401 |
| `/api/v1/sessions` | POST | Yes | multipart: metrics JSON, session_id, optional audio | session id and analysis |
| `/api/v1/sessions` | GET | Yes | none | user's sessions |
| `/api/v1/sessions/trends` | GET | Yes | none | trend points and latest analysis |
| `/api/v1/sessions/{id}` | GET | Yes | path id | session detail or 404 |
| `/api/v1/sessions/{id}/report` | POST | Yes | path id | doctor report |
| `/api/v1/sessions/{id}/report/pdf` | GET | Yes | path id | PDF binary |

Status codes:

- 200: success.
- 400: invalid metrics JSON in session creation.
- 401: unauthenticated or wrong login.
- 404: session not found.
- 409: duplicate signup email.
- 422: Pydantic validation failure.
- 429: rate limit exceeded.
- 500: unexpected server error.

### Why Services Layer Is Used

Routes should handle HTTP only. Services handle business logic:

- Auth provider logic.
- Metrics derivation.
- Baseline calculation.
- Analysis.
- Voice transcription.
- Report generation.

Advantages:

- Easier testing.
- Cleaner routes.
- Logic reusable outside HTTP.
- Better separation of concerns.

### Why Schemas Are Separate From Models

Models describe database tables. Schemas describe API input/output shape. Keeping them separate prevents accidental exposure of internal fields like password hashes and allows API shape to evolve independently.

### Backend Viva Questions

1. Why FastAPI?
2. What is ASGI?
3. What is Pydantic validation?
4. What happens if request body has extra fields?
5. Why use service layer?
6. Why separate schemas and models?
7. What is dependency injection?
8. How does `get_db` manage sessions?
9. What is `expire_on_commit=False`?
10. Why SQLite needs `check_same_thread=False`?
11. Why use Alembic?
12. Why not `create_all` in production?
13. How does JWT verification happen?
14. What is stored in `request.state.auth_user`?
15. Why load the user from DB after token verification?
16. What does rate limiting protect?
17. What is the rate limit key?
18. How are errors standardized?
19. Why use JSON columns for metrics?
20. What is the disadvantage of JSON columns?
21. How does session creation parse multipart?
22. Why does metrics arrive as string in multipart?
23. How is audio handled?
24. What if voice transcription fails?
25. How is baseline calculated?
26. Why last seven sessions?
27. How is risk score weighted?
28. What if OpenAI returns invalid JSON?
29. Why validate OpenAI output with Pydantic?
30. What is deterministic fallback?
31. How does report generation work?
32. How does PDF endpoint differ from JSON endpoint?
33. How is ownership enforced for sessions?
34. Why query by both `session_id` and `user_id`?
35. What is CORS?
36. Why `allow_credentials=True`?
37. What security risk comes with cookie auth?
38. How are duplicate emails prevented?
39. What is an index?
40. Why index `user_id, created_at`?
41. What is `ondelete="CASCADE"`?
42. Why set cookie `httponly=True`?
43. Why `secure` only in production?
44. What is SameSite lax?
45. How are environment variables loaded?
46. What is normalized database URL?
47. What does `lru_cache` do for settings?
48. Why use `Response` to set cookies?
49. Why include health check?
50. What is the biggest backend risk? Client-side metrics can be manipulated; medical scoring is heuristic.

## Part 5 - Database Complete Explanation

### Database Used

Local default: SQLite file at `backend/data/neurowatch.db`.

Production-ready: Postgres/Neon through `DATABASE_URL`.

Why SQLite locally:

- No server setup.
- Fast demo.
- Easy for students.

Why Postgres in production:

- Better concurrency.
- Managed backups/options.
- Strong relational DB features.
- More reliable for multi-user production.

### Tables

`users`:

- `id`: string UUID primary key.
- `email`: unique, indexed.
- `display_name`: optional.
- `password_hash`: optional because Firebase users may not have local password.
- `created_at`: timezone-aware creation time.
- `last_login_at`: optional last login timestamp.

`sessions`:

- `id`: string UUID primary key.
- `user_id`: foreign key to users.
- `number`: user's session number.
- `raw_metrics`: JSON copy of submitted/derived metrics.
- `analysis`: JSON generated analysis.
- `created_at`: timestamp.

Relationship:

- One user has many sessions.
- One session belongs to one user.
- Deleting user cascades to sessions.

### ORM

SQLAlchemy models are Python classes:

- `User(Base)` maps to `users`.
- `Session(Base)` maps to `sessions`.

Advantages of ORM:

- Reduces SQL repetition.
- Maps rows to objects.
- Easier relationship handling.
- Safer query construction.

Disadvantages:

- Can hide inefficient queries.
- Developers still need SQL knowledge.
- JSON fields can reduce queryability.

### Migration Flow

1. Developer changes models.
2. Alembic migration is created.
3. Migration file describes upgrade/downgrade.
4. Deploy runs `alembic upgrade head`.
5. DB schema changes before app restarts.

In this project, initial migration creates both tables and indexes.

### Session Management

`get_db()` creates a DB session per request and closes it in `finally`. Routes commit when they create/update data. This avoids global shared sessions and prevents connection leaks.

### Transactions

When signup/session creation happens:

- `db.add(...)` stages object.
- `db.commit()` writes changes transactionally.
- `db.refresh(...)` reloads generated/default values.

If commit fails, DB rolls back at the connection/session level in SQLAlchemy behavior, but explicit rollback handling could be added for more robustness.

### Database Viva Questions

1. Why use relational DB?
2. Why SQLite for local?
3. Why Postgres for production?
4. What are the tables?
5. What is primary key?
6. Why UUID string id?
7. Why unique email?
8. Why index email?
9. Why foreign key?
10. What is cascade delete?
11. Why JSON columns?
12. What is disadvantage of JSON columns?
13. What is ORM?
14. Why ORM instead of raw SQL?
15. What is Alembic?
16. What is migration?
17. What is upgrade?
18. What is downgrade?
19. What is a transaction?
20. Why one DB session per request?
21. What is connection pooling?
22. What is `StaticPool` used for in tests?
23. Why use in-memory SQLite in tests?
24. What query lists sessions?
25. How is ownership enforced in DB queries?

## Part 6 - Authentication And Security

### Complete Login Flow

1. User enters email/password.
2. Frontend sends `POST /api/v1/auth/login`.
3. Pydantic validates email format and password length.
4. Backend finds user by normalized email.
5. Backend verifies password using scrypt and constant-time compare.
6. Backend updates `last_login_at`.
7. Backend creates JWT.
8. Backend sets cookie `neurowatch_auth`.
9. Cookie options: httpOnly, SameSite lax, secure in production, max age, path `/`.
10. Response returns public user data.
11. Frontend stores user object.
12. Future requests include cookie automatically.

### JWT

JWT is a signed token with claims.

This project stores:

- `sub`: user id.
- `email`: email.
- `display_name`: display name.
- `iat`: issued-at timestamp.
- `exp`: expiry timestamp.
- `iss`: issuer.

It uses HS256, meaning same secret signs and verifies token.

### Cookies

httpOnly: JavaScript cannot read token, reducing XSS token theft.

SameSite lax: Cookie is sent on normal navigation and same-site requests, but not most cross-site subrequests.

Secure in production: Cookie only sent over HTTPS.

### Protected APIs

Protected routes depend on `require_user`. If middleware did not set valid auth user or DB user no longer exists, API returns 401.

### Firebase Auth Integration

Frontend can sign in with Firebase. Backend verifies Firebase ID token, creates/updates local user, then sets the same project JWT cookie. This is called an auth provider seam because the route contract remains stable even if provider changes.

### Security Threats And Prevention

XSS:

- httpOnly cookie prevents JS from reading token.
- React escapes text by default.
- Avoid rendering untrusted HTML.

CSRF:

- SameSite lax reduces CSRF risk.
- Current project does not implement explicit CSRF token. Examiner trap: acknowledge this honestly and say adding CSRF token or double-submit token would strengthen cookie-based auth, especially for state-changing endpoints.

SQL Injection:

- SQLAlchemy query APIs bind parameters.
- Pydantic validates shape before queries.

Cookie theft:

- httpOnly and secure cookies reduce theft.
- HTTPS via certbot protects network transport.

Brute force:

- SlowAPI rate limits auth endpoints.
- fail2ban protects SSH on server.

Secrets leakage:

- `.gitignore` excludes `.env`.
- Deploy script sets `.env` to mode 600.

CORS:

- Backend only allows configured origins.
- `allow_credentials=True` is needed for cookie auth.

Secure headers:

- nginx sets `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`.

File exposure:

- nginx blocks hidden dotfiles like `.env` and `.git`.

### Tough Security Viva Questions

1. Why not store JWT in localStorage?
   Answer: XSS could read localStorage; httpOnly cookie is safer.

2. Does httpOnly prevent all XSS damage?
   Answer: No. It prevents token reading, but attacker JS could still perform actions in the user's browser.

3. Is CSRF fully solved?
   Answer: SameSite lax reduces it, but explicit CSRF protection would be stronger.

4. Can users fake metrics?
   Answer: Yes, since metrics are computed client-side. For medical-grade systems, server-side verification or trusted instrumentation is needed.

5. Why constant-time password comparison?
   Answer: To reduce timing attacks comparing hashes.

6. Why scrypt?
   Answer: It is slow and memory-hard, making brute-force password cracking harder.

7. What happens if JWT secret leaks?
   Answer: Attackers can forge tokens. Rotate secret and force logout.

8. Why query `session_id` with `user_id`?
   Answer: Prevents one user from reading another user's session.

9. Why rate limit?
   Answer: Reduces brute-force and abuse.

10. What production security improvement would you add?
    Answer: CSRF token, Redis rate limiter, audit logs, stronger CSP, MFA, secret manager.

## Part 7 - AI / OpenAI / Analysis Pipeline

### Why AI Is Used

AI is optional and used to enhance the narrative interpretation of metrics. It can produce more natural summaries, watch areas, and recommendations. However, the project does not depend on AI for core scoring because deterministic fallback exists.

### OpenAI Analysis Flow

1. Backend receives validated metrics.
2. Baseline service provides baseline information.
3. `analyze_session()` checks if `OPENAI_API_KEY` exists.
4. If missing, local analysis is used.
5. If present, OpenAI chat completion is called with:
   - system prompt from `prompts.py`.
   - user JSON containing current metrics, baseline metrics, and instructions.
   - temperature 0.2.
   - JSON object response format.
6. OpenAI response content is validated with `Analysis.model_validate_json`.
7. If validation fails, local fallback is used.

### Deterministic Fallback

Local analysis computes domain scores with threshold rules:

- Typing score: WPM decline, timing variance rise, error rate rise, backspace frequency, key hold.
- Reaction score: reaction time rise, variance, miss rate, anticipation.
- Memory score: recall decline, latency, pattern score, sequence score, false positives.
- Voice score: speech rate decline, pause duration rise, pitch variation decline, articulation.

Overall score:

```text
typing * 0.25 + reaction * 0.25 + memory * 0.30 + voice * 0.20
```

Risk levels:

- 0-25: low.
- 26-50: moderate.
- 51-70: elevated.
- 71-100: high.

### Voice Processing

1. Browser records audio with `MediaRecorder`.
2. Frontend sends audio file in multipart request.
3. Backend reads bytes.
4. If OpenAI key missing, voice analysis is skipped with note.
5. If key exists, backend sends file to Whisper model `whisper-1`.
6. Requests verbose JSON with word timestamps.
7. Extracts word start/end times.
8. Counts words.
9. Calculates speech rate.
10. Detects pauses longer than 0.25 seconds.
11. Calculates pause frequency and mean pause duration.
12. Estimates articulation from word confidence minus filler penalty.
13. Sets pitch variation to 0 with explicit data quality note.

### AI Limitations

- AI output can be invalid or unavailable.
- AI can hallucinate if not constrained.
- AI calls cost money and add latency.
- Whisper accuracy depends on accent, noise, microphone, language.
- This is not clinically validated.

### AI Viva Questions

1. Why is AI optional?
2. What happens if OpenAI fails?
3. Why validate AI output with Pydantic?
4. What model is used for analysis?
5. What model is used for transcription?
6. Why temperature 0.2?
7. What is prompt injection risk?
8. How does project reduce hallucination?
9. Why not diagnose disease?
10. What ethical concerns exist?

## Part 8 - Testing Complete Explanation

### Why Testing Matters

Tests protect core behavior from regressions. In viva, say: because this project handles auth, health-style metrics, and reports, we need confidence that validation, analysis, and API flows keep working after changes.

### Test Setup

`conftest.py`:

- Inserts backend path into `sys.path`.
- Sets `DATABASE_URL` to in-memory SQLite.
- Sets test JWT secret.
- Clears OpenAI key for deterministic fallback.
- Creates in-memory DB using `StaticPool`.
- Overrides FastAPI `get_db`.
- Provides `TestClient`.

### Unit Tests

- `test_security.py`: password hash/verify.
- `test_metrics.py`: typing/reaction/memory formulas.
- `test_analysis.py`: local analysis shape and disclaimer.
- `test_baseline.py`: baseline threshold and averaging.
- `test_reporting.py`: technical note sanitization and PDF generation.
- `test_voice.py`: speech/pause/articulation helpers.

### Integration Tests

- `test_auth_api.py`: complete signup -> me -> logout -> me 401 -> login flow; Firebase route monkeypatch.
- `test_sessions_api.py`: auth requirement; create/list/trends/detail/report/pdf; voice transcription failure fallback.

### Verification Note

I attempted to run `python -m pytest` locally, but the current machine's default Python is 3.13 and `pytest` is not installed. `py -3.11` is also not installed. So the tests were inspected from source but not executed in this environment. To run them, install Python 3.11 and dependencies:

```bash
python -m pip install -r backend/requirements.txt
python -m pytest
```

## Part 9 - DevOps And Deployment

### Deployment Architecture

Internet -> nginx on Lightsail -> static SPA or `/api` proxy -> Gunicorn -> Uvicorn workers -> FastAPI -> Postgres/Neon.

### Why AWS Lightsail

Lightsail is simpler than full EC2/VPC setup. It gives a VM, static IP, firewall, and predictable pricing.

### Why nginx

- Serves built frontend efficiently.
- Proxies API to localhost backend.
- Handles gzip.
- Adds security headers.
- Handles TLS with certbot.
- Protects hidden files.

### Why Gunicorn + Uvicorn

FastAPI is ASGI, so Uvicorn runs it. Gunicorn manages multiple worker processes, restarts, timeouts, max requests, and process lifecycle.

### Why systemd

systemd starts app on boot, restarts on failure, captures logs, and applies hardening settings.

### CI/CD Flow

1. Push to `main`.
2. GitHub Actions starts.
3. Checkout repo.
4. Setup Python 3.11.
5. Setup Node 20.
6. Install backend deps.
7. Run integration tests.
8. Install frontend deps and build.
9. SSH into Lightsail.
10. Clone or update repo.
11. Reset to latest branch.
12. Run `deploy.sh`.
13. Server installs deps, builds frontend, runs migrations.
14. systemd service and nginx config are installed.
15. App restarts.
16. Health smoke test checks `/api/v1/health`.

### Rollback Strategy

`deploy-production.sh` records successful commit. `rollback.sh` resets repo to previous successful commit or specified ref, runs deploy again, and records rollback.

### Production Security

- UFW firewall.
- fail2ban.
- unattended upgrades.
- `.env` mode 600.
- systemd sandboxing.
- nginx security headers.
- dotfile blocking.
- TLS with HSTS.
- SSH private key through GitHub secrets.

### Deployment Viva Questions

1. Why use nginx?
2. Why not expose FastAPI directly?
3. What does systemd do?
4. What does Gunicorn do?
5. Why Uvicorn workers?
6. What is reverse proxy?
7. What is SSL/TLS?
8. What does certbot do?
9. Why run migrations during deploy?
10. What is smoke testing?
11. How would you debug 502?
12. How would you rollback?
13. Why cache `/assets` forever?
14. Why not cache `index.html` forever?
15. Why `.env chmod 600`?

## Part 10 - Advanced Computer Science Concepts

Operating Systems:

- Processes: Gunicorn master and workers.
- Services: systemd supervises app.
- File permissions: `.env` 600.
- Memory: worker recycling with max requests.

DBMS:

- Tables, primary keys, foreign keys, indexes.
- Transactions and commits.
- ORM mapping.
- Migrations.

Computer Networks:

- HTTP request/response.
- Cookies.
- CORS.
- Reverse proxy.
- TLS.
- Status codes.

Software Engineering:

- Layered architecture.
- Separation of concerns.
- Testing.
- CI/CD.
- Environment configuration.
- Documentation.

Distributed Systems:

- Client/server boundary.
- Stateless API auth through signed token.
- Scaling concerns with in-memory rate limit.
- External managed services: OpenAI, Firebase, Neon.

Cyber Security:

- Password hashing.
- JWT signing.
- Secure cookies.
- Rate limiting.
- Security headers.
- Least privilege systemd settings.

Cloud Computing:

- Lightsail VM.
- Managed Postgres.
- GitHub Actions runner.
- Infrastructure scripts.

OOP:

- SQLAlchemy model classes.
- Abstract `AuthProvider`.
- Dataclasses for service inputs/results.

Design Patterns:

- Provider pattern: `AuthProvider`.
- Dependency injection: FastAPI `Depends`.
- Repository-like access through SQLAlchemy sessions.
- Service layer pattern.
- Adapter/fallback pattern for Firebase and OpenAI.

REST Concepts:

- Resource endpoints under `/sessions`.
- HTTP methods: GET, POST.
- Status codes.
- Stateless backend with signed cookie token.

Concurrency:

- React timers and event handlers.
- FastAPI ASGI support.
- Gunicorn multiple worker processes.
- Database sessions per request.

Scalability:

- Add more Gunicorn workers.
- Move rate limiting to Redis.
- Move audio/AI processing to background queue.
- Split frontend CDN and backend API.
- Add DB indexes and query monitoring.

## Part 11 - Code Walkthrough Of Most Important Files

### `backend/app/main.py`

- Imports `Path`: used to locate frontend build directory.
- Imports `FastAPI`: creates app instance.
- Imports `CORSMiddleware`: allows browser frontend origin and cookies.
- Imports `StaticFiles`: serves built React app when `frontend/dist` exists.
- Imports `api_router`: all API routes.
- Imports settings/logging/db/middleware/handlers.
- `_resolve_frontend_dist()`: checks several likely paths for `frontend/dist`. If removed, FastAPI would not serve SPA directly.
- `create_app()`: app factory. Useful for tests because tests can create a fresh app.
- `settings = get_settings()`: loads env config.
- `configure_logging()`: initializes logs.
- `app = FastAPI(title=settings.app_name)`: creates FastAPI app.
- `app.state.limiter = limiter`: attaches SlowAPI limiter.
- Exception handlers: standardize 429, HTTP errors, validation errors, and 500 errors.
- CORS middleware: required for browser calls from frontend origin with cookies.
- RequestIdMiddleware: adds trace id header.
- AuthMiddleware: reads JWT cookie and sets request state.
- SlowAPIMiddleware: enforces rate limits.
- `on_startup`: calls `init_db()` for SQLite local bootstrap.
- `include_router`: mounts `/api/v1` routes.
- Static mount: serves React dist if available.
- `app = create_app()`: ASGI server imports this variable.

If removed: backend would not start as ASGI app.

### `backend/app/api/v1/auth.py`

- Defines router prefix `/auth`.
- `_cookie_kwargs()` centralizes cookie security flags.
- `_to_user_response()` ensures only safe public user fields are returned.
- `signup()`: validates payload, calls provider, handles duplicate email as 409, sets cookie.
- `login()`: validates credentials, calls provider, handles wrong credentials as 401, sets cookie.
- `login_with_firebase()`: exchanges Firebase ID token for local JWT cookie.
- `logout()`: overwrites cookie with empty value and max_age 0.
- `me()`: returns current user or 401.

Why important: This file is the bridge between browser auth actions and secure backend session cookie.

### `backend/app/middleware/auth.py`

- `AUTH_COOKIE_NAME = "neurowatch_auth"` centralizes cookie name.
- `AuthMiddleware` inherits Starlette base middleware.
- `dispatch()` runs for each request.
- Reads cookie from request.
- Initializes `request.state.auth_user = None`.
- If token exists, verifies through auth provider.
- Stores authenticated user info on request state.
- Calls next handler.

If removed: protected endpoints would always see no current user.

### `backend/app/core/security.py`

- `hash_password()`: generates random salt, derives scrypt hash, stores `salt:hash`.
- `verify_password()`: splits stored value, re-derives hash, compares with `hmac.compare_digest`.
- `create_access_token()`: builds JWT claims and signs with secret.
- `decode_access_token()`: verifies signature, algorithm, issuer, and expiry.
- `try_decode_access_token()`: returns `None` on JWT error instead of throwing.

Why scrypt: It is intentionally expensive and memory-hard, making offline cracking harder.

### `backend/app/db/session.py`

- `create_engine_for_url()`: builds SQLAlchemy engine.
- SQLite uses `check_same_thread=False` because FastAPI test/client threads may differ.
- In-memory SQLite uses `StaticPool` so all sessions share same DB.
- `SessionLocal`: session factory.
- `get_db()`: yields a DB session and closes it after request.

If DB session were global: concurrency bugs and stale connections could occur.

### `backend/app/api/v1/sessions.py`

- `create_session()`: main endpoint.
- `metrics: str = Form(...)`: metrics are string because multipart form fields are text.
- `audio: UploadFile | None`: optional audio file.
- Parses metrics JSON and validates `SessionMetrics`.
- If audio exists, calls voice service.
- If voice fails, records data quality note but does not fail whole session.
- Gets user baseline.
- Creates session id.
- Combines missing-data notes.
- Calls analysis service.
- Creates DB record with raw metrics and analysis JSON.
- Commits and returns response.
- `list_sessions()`: returns sessions owned by current user, newest first.
- `session_trends()`: returns chart-ready scores over time.
- `get_session()`: fetches one session by id and user id.
- `generate_session_report()`: builds report JSON.
- `download_session_report_pdf()`: builds PDF bytes and returns binary response.

Security point: every session query includes `current_user.id`.

### `backend/app/services/analysis.py`

- `AnalyzeSessionInput`: dataclass packaging all analysis inputs.
- `_round()`: clamps score to 0-100.
- `_compare_trend()`: compares current to baseline percentage change.
- `_determine_risk_level()`: maps score to label/color/level.
- `_typing_domain()`, `_reaction_domain()`, `_memory_domain()`, `_voice_domain()`: domain-specific scoring.
- `create_local_analysis()`: combines domain scores, weighted score, strongest signal, summary, recommendations, alerts, disclaimer.
- `analyze_session()`: chooses OpenAI path if key exists, validates JSON output, otherwise fallback.

If removed: app could collect metrics but not produce meaningful result.

### `frontend/src/app/lib/auth-context.tsx`

- Creates `AuthContext`.
- Reads `VITE_AUTH_PROVIDER`.
- Holds `user` and `loading` in state.
- `exchangeFirebaseToken`: sends Firebase token to backend and sets user.
- `refreshUser`: calls `/auth/me`.
- Startup `useEffect`: handles Firebase redirect result or cookie restore.
- `signIn`: local or Firebase email/password flow.
- `signInWithGoogle`: popup first, redirect fallback if popup blocked.
- `signUp`: local or Firebase signup.
- `signOut`: signs out Firebase if needed, clears backend cookie, clears state.
- `useMemo`: creates stable context value.
- `useAuth`: safe consumer hook.

Why important: It is the frontend source of truth for auth.

### `frontend/src/app/lib/api.ts`

- `API_PREFIX = "/api/v1"` centralizes API version.
- `requestJson<T>()`: generic JSON fetch helper.
- All auth functions call backend auth routes.
- `createSession()`: uses `FormData`, sends metrics JSON and optional audio blob.
- `downloadSessionReportPdf()`: fetches binary blob and extracts filename from content disposition.

If `credentials: "include"` is removed: protected requests will fail because cookie will not be sent.

### `frontend/src/app/components/session/SessionFlow.tsx`

- Defines step union.
- Holds current step and each metric state.
- `metrics` is memoized.
- `submit()` calls `createSession()` with `crypto.randomUUID()` and audio blob.
- Conditional rendering shows exactly one step at a time.
- Each child calls `onComplete`, stores result, and advances step.

This is the frontend workflow controller.

## Part 12 - Mock Viva Session

Use this interactively. Answer one question at a time, then compare with the ideal answer.

Easy:

1. What is NeuroWatch?
2. What are the four behavioral domains?
3. What is the difference between frontend and backend?
4. What is an API?
5. What is the database used locally?

Medium:

6. Explain the login flow.
7. Why are cookies httpOnly?
8. How is a session submitted?
9. Why is baseline needed?
10. How does the dashboard get trends?

Difficult:

11. What happens if OpenAI returns invalid JSON?
12. Why is CSRF still a concern with cookie auth?
13. Why must session query include user id?
14. Why is `StaticPool` used in tests?
15. How would you scale this beyond one VM?

Trap:

16. Is this medically validated?
    Ideal answer: No. It is a software prototype/awareness tool using heuristic thresholds, not a clinically validated diagnostic system.

17. Can a user fake metrics?
    Ideal answer: Yes, because frontend computes them. For production-grade health systems, server-side verification or trusted clients are needed.

18. Does Firebase replace JWT here?
    Ideal answer: No. Firebase proves identity to backend; backend still sets its own httpOnly JWT cookie for app sessions.

19. Is SQLite good for production?
    Ideal answer: Not for this multi-user deployment; project supports Postgres/Neon for production.

20. Does OpenAI make the system reliable?
    Ideal answer: OpenAI can enhance narrative, but reliability comes from validation and deterministic fallback.

## Part 13 - Most Important Rapid Revision

### Top 100 Viva Questions With One-Line Answers

1. What is NeuroWatch? A behavioral monitoring full-stack web app.
2. Main stack? React/Vite frontend and FastAPI backend.
3. Database? SQLite locally, Postgres-ready in production.
4. Auth method? httpOnly JWT cookie, local or Firebase exchange.
5. API prefix? `/api/v1`.
6. Four domains? Typing, reaction, memory, voice.
7. Baseline threshold? Seven sessions.
8. Why baseline? Personal comparison is more meaningful.
9. Is it diagnostic? No.
10. Why FastAPI? Validation, speed, OpenAPI, Python ecosystem.
11. Why React? Interactive component-based SPA.
12. Why Vite? Fast dev/build.
13. Why SQLAlchemy? ORM and safer query construction.
14. Why Alembic? Schema versioning.
15. Why Pydantic? Validation and serialization.
16. Why cookie? Safer than localStorage for tokens.
17. Why httpOnly? JS cannot read token.
18. Why SameSite? Reduces CSRF.
19. Missing CSRF token? Acknowledge and propose adding one.
20. Password hashing? scrypt with random salt.
21. Constant-time compare? Reduces timing leakage.
22. JWT algorithm? HS256.
23. JWT subject? User id.
24. Login endpoint? `POST /auth/login`.
25. Signup endpoint? `POST /auth/signup`.
26. Me endpoint? `GET /auth/me`.
27. Session create? `POST /sessions` multipart.
28. Metrics format? JSON string in form field.
29. Audio field? Optional upload file.
30. Voice transcription model? `whisper-1`.
31. Analysis model? Chat completion model `gpt-4o` in code.
32. AI fallback? Deterministic local analysis.
33. Why validate AI output? Prevent invalid/hallucinated shape.
34. Overall scoring weights? Typing .25, reaction .25, memory .30, voice .20.
35. Risk levels? Low, moderate, elevated, high.
36. Health endpoint? `/api/v1/health`.
37. Report JSON endpoint? `POST /sessions/{id}/report`.
38. PDF endpoint? `GET /sessions/{id}/report/pdf`.
39. PDF library? ReportLab.
40. Rate limit library? SlowAPI.
41. CORS reason? Allow browser frontend origin.
42. `credentials: include` reason? Send auth cookie.
43. Route protection frontend? `RequireAuth`.
44. Route protection backend? `require_user`.
45. Why user id in session queries? Ownership check.
46. Main app entry backend? `app.main:app`.
47. Main app entry frontend? `src/main.tsx`.
48. Router file frontend? `App.tsx`.
49. Auth context file? `auth-context.tsx`.
50. API client file? `api.ts`.
51. Metric formulas frontend? `lib/metrics.ts`.
52. Metric formulas backend? `services/metrics.py`.
53. Baseline file? `services/baseline.py`.
54. Analysis file? `services/analysis.py`.
55. Voice file? `services/voice.py`.
56. Report file? `services/reporting.py`.
57. Models? `models/user.py`, `models/session.py`.
58. Schemas? `schemas/*.py`.
59. Middleware? Auth, errors, rate limit, request id.
60. nginx role? Static serving, proxy, headers, TLS.
61. systemd role? Supervise process.
62. Gunicorn role? Manage workers.
63. Uvicorn role? Run ASGI app.
64. CI file? `.github/workflows/deploy-lightsail.yml`.
65. Deploy script? `deploy/lightsail/deploy.sh`.
66. Rollback script? `rollback.sh`.
67. SSL script? `enable-ssl.sh`.
68. Bootstrap script? `bootstrap.sh`.
69. Production DB example? Neon Postgres.
70. Secret file? `.env`.
71. Secret protection? `.gitignore` and chmod 600.
72. XSS mitigation? React escaping, httpOnly cookie.
73. SQL injection mitigation? SQLAlchemy parameter binding.
74. Brute force mitigation? Rate limiting.
75. Server brute force? fail2ban.
76. Static asset cache? Immutable hashed assets.
77. Index cache? No-cache.
78. Dotfile protection? nginx deny dotfiles.
79. Session number? Previous sessions count + 1.
80. Baseline data source? Last seven sessions.
81. Trend labels? `S1`, `S2`, etc.
82. OpenAI absent? Local analysis note is added.
83. Voice absent? Missing data note.
84. Audio failure? Session still succeeds.
85. Frontend tests? Not present.
86. Backend tests? pytest unit/integration.
87. Test DB? In-memory SQLite.
88. Why `StaticPool`? Share in-memory DB across connections.
89. Why `expire_on_commit=False`? Keep object data usable after commit.
90. Why `lru_cache` settings? Avoid repeated env parsing.
91. Why `extra="forbid"` schemas? Reject unexpected fields.
92. Why `EmailStr`? Validate email format.
93. Why `Field(ge/le)`? Enforce numeric ranges.
94. Major limitation? Heuristic thresholds not clinically validated.
95. Major security improvement? CSRF tokens and CSP.
96. Major scaling improvement? Redis rate limit, background jobs, separate frontend CDN.
97. Major data improvement? Clinically validated dataset.
98. Major AI improvement? Safer structured outputs and audit trail.
99. One-line defense? Secure full-stack monitoring prototype with deterministic fallback and production deployment.
100. Best honest answer if stuck? State what you know, identify the file/layer, and explain how you would verify.

### Flow Summaries

Auth: Login form -> `/auth/login` -> verify password -> create JWT -> set httpOnly cookie -> frontend stores user -> protected calls include cookie.

API: React `fetch` -> Vite/nginx proxy -> FastAPI middleware -> dependency validation -> route -> service -> DB -> schema response.

AI: Metrics + baseline -> OpenAI if key exists -> Pydantic validation -> fallback if missing/failing -> analysis stored.

Deployment: Push main -> GitHub Actions tests/build -> SSH -> git reset -> deploy.sh -> install/build/migrate/restart -> health check.

Security: httpOnly cookie, scrypt password hashing, rate limit, SQLAlchemy, CORS config, nginx headers, TLS, dotfile blocking, `.env` permission.

## Part 14 - Professor Trap Questions

1. If JWT is stateless, how do you logout?
   Answer: This app clears browser cookie. Existing token remains valid until expiry if stolen; server-side denylist would be needed for immediate global invalidation.

2. Why is CSRF relevant if token is httpOnly?
   Answer: Browser automatically sends cookies, so cross-site form/request attacks may use the victim's cookie unless SameSite/CSRF defenses stop it.

3. Does OpenAI guarantee correct medical advice?
   Answer: No. The project validates structure and uses disclaimers/fallback, but it is not medical diagnosis.

4. Why store analysis JSON instead of recomputing always?
   Answer: Fast display and historical consistency; raw metrics are also stored for future re-analysis.

5. What breaks if CORS origin is wrong?
   Answer: Browser blocks credentialed API calls; login/session calls fail.

6. What breaks if `credentials: include` is removed?
   Answer: Cookie is not sent; protected APIs return 401.

7. What if user deletes account?
   Answer: Sessions are configured to cascade delete through foreign key relationship.

8. Why not run migrations with app startup?
   Answer: Production migrations should be explicit during deploy to avoid race conditions and uncontrolled schema changes.

9. What if two sessions submit simultaneously?
   Answer: Session number calculation can race because it uses count + 1 without transaction lock. For production, add DB constraint or transactional sequence.

10. Is rate limiting reliable across multiple servers?
    Answer: In-memory/IP-based rate limiting is limited; use Redis for distributed rate limiting.

11. Why is pitch variation zero?
    Answer: Current Whisper pipeline does not provide robust pitch data; project explicitly notes this rather than guessing.

12. Why is frontend metric calculation a trust issue?
    Answer: Browser code is controlled by user, so malicious clients can send fake metrics.

13. Why is `password_hash` nullable?
    Answer: Firebase-created users may not have a local password.

14. Why can Firebase and local users conflict by email?
    Answer: Backend normalizes email and upserts same user, but provider linking policies should be designed carefully in production.

15. Why use JSON response envelope for errors?
    Answer: Frontend can consistently display messages and inspect error codes.

## Part 15 - Project Defense

### How To Introduce The Project

"My project is NeuroWatch, a full-stack behavioral awareness platform. It lets users complete short typing, reaction, memory, and voice tasks, derives structured metrics, compares them with personal baselines after enough sessions, and generates dashboard insights and doctor-ready reports. It uses React/Vite on the frontend, FastAPI with SQLAlchemy and Alembic on the backend, secure httpOnly JWT cookie authentication, optional Firebase/OpenAI integrations, and a production deployment setup using nginx, systemd, Gunicorn/Uvicorn, and GitHub Actions on AWS Lightsail."

### How To Explain Architecture On Whiteboard

Draw:

Browser React SPA -> `/api/v1` -> nginx/Vite proxy -> FastAPI -> middleware -> routes -> services -> SQLAlchemy -> DB.

Side boxes:

- OpenAI/Whisper optional.
- Firebase optional.
- GitHub Actions -> Lightsail deploy.

### How To Answer If You Do Not Know

Use this structure:

"I do not want to guess. From the architecture, this belongs to [frontend/backend/database/deployment]. The relevant file is [file]. My understanding is [what you know]. I would verify by checking [test/log/API/docs]."

That sounds honest and engineering-minded.

### How To Defend Technology Choices

- React/Vite: best for interactive browser session flow.
- FastAPI: Python-friendly, validation-heavy, AI-compatible backend.
- SQLAlchemy/Alembic: maintainable DB access and migration control.
- JWT cookie: stateless session with reduced XSS token exposure.
- OpenAI optional: enhancement without availability dependency.
- nginx/systemd/Gunicorn: standard single-VM production architecture.

### How To Explain Limitations Honestly

Say:

"This is not a certified clinical diagnostic system. It is a production-oriented software prototype for behavioral awareness. The strongest engineering parts are secure auth, structured validation, baseline comparison, deterministic fallback, report generation, tests, and deployment. The main future work is clinical validation, stronger CSRF protection, server-side metric integrity, and scalable background processing."

### Final Defense Sentence

"The project is valuable because it combines a meaningful health-tech workflow with real software engineering practices: authentication, validation, database design, AI fallback, testing, deployment automation, and security-aware production setup."

