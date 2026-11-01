# Mehfil: Architecture

> **Status:** Phase 0 (planning). This document describes the intended architecture. It is updated as each phase is implemented, and sections that differ from the code will be corrected.

## 1. Overview

Mehfil is a server-rendered web application. Flask produces HTML with Jinja2 templates, and small vanilla JavaScript modules enhance specific interactions (RSVP submission, live search, dashboard refresh) through JSON endpoints. There is no single-page-application framework and no frontend build step.

**Why server-rendered:** invitation pages must load fast on low-end phones and must carry Open Graph tags in the HTML so WhatsApp can show link previews. See `docs/DECISIONS.md`.

## 2. Layered structure

```
Browser (HTML + CSS + vanilla JS)
        |  HTTPS
        v
Reverse proxy (provided by the host)
        |
        v
Gunicorn --> Flask application
              |-- Blueprints (routes)   thin: parse request, call service, render
              |-- Services              business rules
              |-- Models (SQLAlchemy)   tables as Python classes
              '-- Templates (Jinja2)
                      |
                      v
              SQLite (development) / PostgreSQL (production)
```

**Rule: routes contain no business logic.** A route parses the request, calls a function in `app/services/`, and renders the result. This keeps business rules (RSVP validation, catering formulas) testable without a web server.

## 3. Planned project structure

```
mehfil/
|-- app/
|   |-- __init__.py        create_app() application factory
|   |-- config.py          Development / Testing / Production settings
|   |-- extensions.py      db, migrate, login_manager, csrf, limiter
|   |-- models/            User, Event, EventFunction, Family, Guest,
|   |                      Invitation, Rsvp, AuditLog
|   |-- blueprints/        auth, host, guests, public_rsvp, catering
|   |-- services/          rsvp.py, catering.py, tokens.py, audit.py, csv_io.py
|   |-- templates/
|   '-- static/            css/, js/, icons/
|-- migrations/
|-- tests/                 unit/, integration/, e2e/
|-- docs/
|-- wsgi.py                entry point for Gunicorn
|-- requirements.txt, requirements-dev.txt
|-- .env.example
'-- pyproject.toml         ruff and pytest configuration
```

The **application factory** builds the Flask app on demand instead of at import time. Tests can then create a fresh app with a temporary database for every run.

## 4. Frontend and backend communication

- **Pages** are rendered on the server and work without JavaScript.
- **Enhancements** use `fetch()` against small JSON endpoints: RSVP submit, live guest search, dashboard statistics.
- **CSRF:** each page embeds the token in a `<meta>` tag, and JavaScript sends it in the `X-CSRFToken` header.
- **Guest pages** receive a short-lived session cookie whose only purpose is carrying the CSRF token, so one CSRF mechanism covers the whole app.

## 5. Security architecture

| Concern | Design |
|---|---|
| Invitation tokens | `secrets.token_urlsafe(16)`: 128 bits of randomness, 22 URL-safe characters. Unique index. Revocable and regenerable. |
| Token storage | Plain text, because hosts must re-copy and re-share links. A database leak would expose invitation links (RSVP details), not passwords. Documented trade-off. |
| Token leakage | Guest pages send `Referrer-Policy: no-referrer` and `Cache-Control: no-store`, and load no third-party resources (icons and fonts are self-hosted). |
| Passwords | Werkzeug password hashing. Never stored or logged in plain text. |
| Sessions | Cookies are `HttpOnly`, `SameSite=Lax`, and `Secure` in production. Fixed lifetime. Logout clears the session. |
| Role-based access | Roles `host` and `admin`, enforced by a `@role_required` decorator. |
| Object-level authorization | Every query is scoped to the signed-in host's events, preventing IDOR (accessing another host's data by changing an ID in the URL). Covered by tests. |
| CSRF | Flask-WTF `CSRFProtect` on all state-changing requests. |
| Rate limiting | Flask-Limiter on login and public RSVP endpoints, keyed by client IP (`ProxyFix` behind the host's proxy). |
| Input validation | Server-side validation of every field. Jinja2 auto-escaping for output. No SQL string concatenation. |
| Security headers | Restrictive Content-Security-Policy, `X-Content-Type-Options`, frame protection, HSTS in production. |
| Audit log | Append-only `audit_logs` table recording actor, action, entity, and old and new values. Guest actions are attributed to the invitation's family. |
| Transactions | Each RSVP submission is a single database transaction. A unique constraint on `(family_id, function_id)` plus update-in-place prevents duplicate rows from double submits. |
| RSVP deadline | Enforced on the server, never only in the UI. |
| Secrets | Environment variables (`.env`, git-ignored). `.env.example` documents the keys without values. |

## 6. Key request flows

**Guest RSVP submission**
```
Token valid and active?     no  -> invalid-link page
Deadline passed?            yes -> read-only page
Within rate limit?          no  -> 429 response
Input valid?                no  -> field errors
Attendees <= invited_count? no  -> error
BEGIN -> upsert RSVP rows -> replace attendees -> write audit log -> COMMIT
```

**Catering calculation** (per function, over accepted RSVPs): count attendees by food preference and age group, apply age-group weights, multiply by (1 + buffer), round up. The buffer is the function's override if set, otherwise the event default. Full formula and edge cases are documented in Phase 6.

## 7. Deployment architecture (outline)

Browser -> host's HTTPS proxy -> Gunicorn (Linux) -> Flask -> PostgreSQL. Migrations run on each deploy. Logs go to stdout. Hosting provider and configuration are decided in Phase 9, after checking current free-tier terms.

## 8. Known limitations

- Flask-Limiter's default in-memory storage is per process. This is adequate for a single worker, and multiple workers would need a shared store.
- Free hosts may sleep when idle, slowing the first load of an invitation link.
- WhatsApp sharing opens a `wa.me` link with prefilled text. Mehfil does not send messages automatically, because the official WhatsApp Business API is paid.
- Gunicorn does not run on Windows. Local development uses Flask's development server.