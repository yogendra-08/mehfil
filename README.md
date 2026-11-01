# Mehfil

**Every Guest Matters. Every Celebration Counts.**

A smart and minimal wedding guest management platform for digital invitations, RSVP tracking, guest management, meal preferences, and attendance planning.

> **Project status: Phase 0 (planning).** The documentation is written first, and no application code exists yet. Everything under "Features" describes the planned scope. This README is updated as each phase is implemented.

## The problem

Indian weddings often invite hundreds of guests, but actual attendance is uncertain. Hosts end up over-ordering catering, wasting food, and chasing relatives over phone calls and WhatsApp messages to learn who is really coming.

## The solution

Mehfil gives each family one personal invitation link. Guests open it on their phone, with no account and no app, and answer in under a minute. Hosts see live responses and get meal estimates they can hand to the caterer.

## Planned features

**For hosts**
- Events with multiple functions (Mehendi, Sangeet, Wedding, Reception)
- Guest management grouped by family, with search, filters, and CSV import/export
- Dashboard: invited, confirmed, declined, pending, expected attendance, veg/non-veg counts, recent activity
- Secure invitation links, QR codes, and WhatsApp sharing
- "Nudge pending" list with prefilled WhatsApp reminders
- Catering estimates with configurable buffer percentages, and exportable reports

**For guests (no account needed)**
- Mobile-friendly invitation page
- Accept or decline per function
- Choose who attends, food preference, dietary notes, and arrival time
- Edit the response until the RSVP deadline
- Add the event to a calendar

## Technology stack

| Layer | Technology |
|---|---|
| Frontend | HTML5, CSS3, vanilla JavaScript, Jinja2 templates, Lucide icons (self-hosted) |
| Backend | Python, Flask |
| Database | SQLite (development), PostgreSQL (production) |
| ORM and migrations | SQLAlchemy 2.0, Flask-Migrate (Alembic) |
| Security | Flask-Login, Flask-WTF (CSRF), Flask-Limiter |
| QR codes | segno |
| Testing | pytest, Playwright |
| Tooling | ruff |
| Production server | Gunicorn (Linux only) |

The reasoning behind each choice is documented in `docs/DECISIONS.md` (added in this phase).

## Installation

Not available yet. Setup instructions will be added in Phase 1, when the application skeleton exists.

Prerequisites for development:
- Git
- Python 3.10 or newer (3.12+ recommended)
- Node.js (used later for Playwright browser tests)

## Documentation

| Document | Contents |
|---|---|
| [PLAN.md](PLAN.md) | Vision, scope, roadmap, milestones, risks |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Branching, commit conventions, guidelines |
| `ARCHITECTURE.md` | System, security, and deployment architecture (coming in this phase) |
| `docs/DATABASE_SCHEMA.md` | Tables, relationships, indexes (coming in this phase) |
| `docs/DECISIONS.md` | Architecture decisions and alternatives (coming in this phase) |
| `docs/APP_FLOW.svg` | Application flow diagram (coming in this phase) |

## Screenshots

Screenshots will be added once the user interface exists (Phase 2 onward).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). In short: all work goes to `develop`, `main` receives only tested milestone merges, and commits follow Conventional Commits.

## License

See [LICENSE](LICENSE).