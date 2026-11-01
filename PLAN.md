# Mehfil: Project Plan

> **Status:** Phase 0 (planning). Nothing below is implemented yet. This document is updated as each phase completes.

## 1. Vision

**Mehfil: Every Guest Matters. Every Celebration Counts.**

Indian weddings invite hundreds of guests, but real attendance is uncertain. This causes over-ordering of catering, food wastage, and poor coordination. Mehfil is a guest management platform where hosts create a wedding, send digital invitations, collect RSVPs, and get reliable meal estimates. Guests open a personal link, with no account needed, and respond in under a minute.

## 2. Scope

### In scope (v1)
- Host accounts with role-based access (host, admin)
- Events with multiple **functions** (ceremonies such as Mehendi, Sangeet, Wedding, Reception)
- Guest management by **family** (the invitation unit) with individual members
- Secure invitation links, QR codes, and WhatsApp sharing
- Guest RSVP (accept/decline per function, attendee selection, food preference, dietary notes, arrival time, edit until deadline)
- Dashboard with response statistics
- Catering estimation with configurable buffers, and report export
- CSV import and export of guests
- Audit log of guest and RSVP changes
- Calendar (`.ics`) download and email notifications

### Out of scope (v1)
- Paid services (WhatsApp Business API, SMS gateways)
- Automatic WhatsApp sending (links open WhatsApp with prefilled text only)
- Seating and room allocation
- Native mobile apps
- Payments and gift registries

## 3. Technology summary

Flask, SQLAlchemy 2.0, Flask-Migrate, Flask-Login, Flask-WTF (CSRF), Flask-Limiter, Jinja2 templates, vanilla JavaScript, self-hosted Lucide icons, segno (QR), SQLite (development), PostgreSQL (production), pytest, Playwright, ruff, Gunicorn (Linux production only).
Details and trade-offs: `ARCHITECTURE.md` and `docs/DECISIONS.md`.

## 4. Roadmap

| Phase | Objective | Definition of done |
|---|---|---|
| **0**: Planning | Docs and architecture | All planning docs exist and match the design; plan approved |
| **1**: Initialization | Flask app factory, config, virtualenv, tooling | `flask run` serves a health page; `pytest` and `ruff` pass; `.gitignore`, `.gitattributes`, `.env.example` present |
| **2**: Design system and dashboard UI | Tokens, components, static dashboard | Layouts verified at 375, 768 and 1280 px; visible keyboard focus; WCAG AA contrast |
| **3**: Database and auth | Models, migrations, login, RBAC | Migrations apply on a clean DB; passwords hashed; cross-host access blocked (tested); SQLite foreign keys enabled |
| **4**: Guests and families | CRUD, search, filters, CSV import/export | Validation works; import reports row-level errors; export round-trips |
| **5**: Invitations and RSVP | Tokens, public page, RSVP, WhatsApp, audit | Invalid, revoked and expired links handled; double submit creates one RSVP; deadline enforced server-side; rate limit verified |
| **6**: Catering and analytics | Formulas, dashboard stats, report export | Formula edge cases unit-tested; dashboard numbers match database counts |
| **7**: Notifications and QR | QR codes, `.ics`, email | QR opens the correct link; `.ics` imports into a calendar; email works with a free SMTP provider |
| **8**: Testing and security | Coverage, E2E, hardening | E2E covers host and guest journeys; security checklist passes; headers verified |
| **9**: Deployment and docs | Live deployment, final docs | Runs on PostgreSQL with migrations; screenshots added; limitations documented |

Phases follow dependency order: authentication before guests, guests before invitations, RSVPs before catering.

## 5. Milestones

| Milestone | After | Tag | Meaning |
|---|---|---|---|
| M1 | Phase 3 | `v0.1.0` | Foundation: app, database, authentication |
| M2 | Phase 6 | `v0.2.0` | Complete working product |
| M3 | Phase 9 | `v1.0.0` | Tested, secured, deployed release |

`develop` merges into `main` only at milestones, after tests pass and manual verification.

## 6. Assumptions

- Hosts are comfortable with a web dashboard. Guests may be non-technical and on low-end phones.
- Most traffic to invitation pages arrives from WhatsApp links on mobile.
- One host account manages an event in v1. Co-host access is v2.
- Free hosting tiers are sufficient for demo-scale traffic.

## 7. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Free hosts sleep when idle or limit databases | Slow first load of invitation links; possible data limits | Verify current terms in Phase 9; document limits; keep backups via CSV export |
| Free SMTP limits and spam filtering | Email notifications unreliable | Treat email as optional; WhatsApp and QR are the primary channels |
| In-memory rate limiting is per process | Weaker limits with multiple workers | Run one worker initially; move to a shared store if scaled |
| SQLite and PostgreSQL behave differently | Bugs appear only in production | Enable SQLite foreign keys; run the test suite on PostgreSQL before deploying |
| Tokens stored in plain text | Database leak exposes invitation links | Tokens are 128-bit random; links reveal RSVP data only, never passwords; documented in `docs/DECISIONS.md` |
| Scope growth | Project never reaches release | v2 items stay out of v1 until M3 |

## 8. Post-v1 ideas (not committed)

Arrival-time grouping, co-host access, duplicate guest detection, Hindi and Marathi invitation pages, seating and room assignment.