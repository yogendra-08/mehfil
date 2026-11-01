# Mehfil: Architecture Decisions

> **Status:** Phase 0 (planning). Decisions are recorded when made. If a decision is later reversed, the entry is marked **Superseded** and a new entry explains why. Entries are not deleted.

Format: **Context** (the problem), **Decision**, **Alternatives considered**, **Consequences** (including downsides).

---

## D1. Server-rendered pages with vanilla JavaScript, no SPA
**Status:** Accepted
**Context:** Guests open invitations from WhatsApp, mostly on low-end phones. WhatsApp link previews read Open Graph tags from the HTML.
**Decision:** Flask renders HTML with Jinja2. Small JavaScript modules enhance specific interactions through JSON endpoints. No frontend framework or build step.
**Alternatives:** React or Next.js (rejected: heavier first load, harder to provide reliable link previews, and the developer already has experience with them and wants to explore a different stack); HTMX (viable, but vanilla JS teaches more).
**Consequences:** Pages work without JavaScript and load fast. Navigation feels less "app-like", and interactive parts need hand-written JS.

## D2. Flask over Django
**Status:** Accepted
**Context:** The developer wants to learn how the pieces (auth, CSRF, migrations, ORM) fit together.
**Decision:** Flask with an application factory and blueprints.
**Alternatives:** Django (includes auth, admin, CSRF, and migrations, so less code, but it hides more of the mechanics).
**Consequences:** More assembly work and more security responsibility on us. Better learning value and a smaller footprint.

## D3. SQLAlchemy 2.0 and Flask-Migrate
**Status:** Accepted
**Context:** The schema will change across phases, and the app must run on two databases.
**Decision:** SQLAlchemy models with parameterized queries, and Alembic migrations through Flask-Migrate.
**Alternatives:** Raw SQL with `sqlite3` (rejected: string-built SQL risks injection, and it is not portable); manual schema scripts (rejected: no versioning).
**Consequences:** A learning curve, offset by portability, safety, and a versioned schema history.

## D4. SQLite for development, PostgreSQL for production
**Status:** Accepted
**Context:** Local setup should need nothing installed, while production needs a robust multi-user database.
**Decision:** SQLite locally and PostgreSQL deployed. The test suite is run on PostgreSQL before deployment.
**Alternatives:** PostgreSQL everywhere (more setup friction on Windows); SQLite in production (weak under concurrent writes, and many free hosts have ephemeral disks).
**Consequences:** The two databases differ in places (foreign keys are off by default in SQLite, and types behave differently). We enable `PRAGMA foreign_keys=ON` and avoid database-specific features.

## D5. Family is the invitation unit
**Status:** Accepted
**Context:** Hosts think in households ("Sharma family, 5 people"). They often know only the head's name and a headcount.
**Decision:** One invitation link and one `families` row per household. Individual `guests` belong to a family and can be filled in by the family during RSVP.
**Alternatives:** One invitation per individual (simpler schema, but 5 people from one house would need 5 links and 5 forms).
**Consequences:** Guest-side effort is a single form. The schema has one extra level (family, then guests).

## D6. Multiple functions (ceremonies) in v1
**Status:** Accepted
**Context:** Indian weddings run several ceremonies with different guest lists, and catering is counted per ceremony.
**Decision:** `event_functions` with `family_functions` (who is invited to what) and RSVPs per family per function. A single-ceremony wedding has one function.
**Alternatives:** One RSVP per event, with functions added later (rejected: it would require rewriting RSVP and catering).
**Consequences:** Slightly more UI and schema in v1. Per-ceremony headcounts work from the start.

## D7. Derived values are computed, not stored
**Status:** Accepted
**Context:** Stored counts can disagree with the rows they summarize.
**Decision:** Attendee counts, pending status, and meal totals are computed from rows. "Pending" means invited but no `rsvps` row exists, so empty rows are never pre-created.
**Alternatives:** Counter columns (faster reads, but risk of drift); pre-created pending rows (thousands of empty rows).
**Consequences:** Reads use joins and aggregates, which are fast at wedding scale (hundreds to low thousands of guests). We add caching only if measurements show a need.

## D8. Food preference and age group live on the guest
**Status:** Accepted
**Context:** Catering needs veg and non-veg counts, and children eat less.
**Decision:** `guests.food_pref` and `guests.age_group`. Catering is a grouped count over attendees, with configurable weights per age group.
**Alternatives:** Preference per RSVP (allows a different choice per ceremony, which is rare and complicates the form).
**Consequences:** A guest has one preference across all ceremonies.

## D9. Invitation tokens: 128-bit random, stored in plain text
**Status:** Accepted
**Context:** Links must be unguessable, and hosts must be able to copy and re-share them later.
**Decision:** `secrets.token_urlsafe(16)` with a unique index. Tokens are revocable and regenerable. Guest pages send `Referrer-Policy: no-referrer` and `Cache-Control: no-store`.
**Alternatives:** Hashed tokens (more secure at rest, but a link could be shown only once and never re-copied); signed stateless tokens (cannot be revoked without extra state).
**Consequences:** If the database leaks, invitation links are exposed. That reveals RSVP details but no passwords. Accepted and documented.

## D10. Authentication and authorization
**Status:** Accepted
**Context:** Two roles are needed, and the main risk is one host reaching another host's data.
**Decision:** Flask-Login sessions, Werkzeug password hashing, a `role` column with a `@role_required` decorator, and every query scoped to the signed-in host's events.
**Alternatives:** A full RBAC library (overengineering for two roles); JWTs (no benefit for a server-rendered app, and harder to revoke).
**Consequences:** Cookie-based sessions require secure cookie settings and CSRF protection (D11). Cross-host access is covered by tests.

## D11. CSRF protection and rate limiting
**Status:** Accepted
**Context:** State-changing requests come from forms and `fetch()`, and public RSVP endpoints can be abused.
**Decision:** Flask-WTF `CSRFProtect` globally, with the token sent in an `X-CSRFToken` header from JavaScript. Flask-Limiter on login and public RSVP routes, using `ProxyFix` to see real client IPs.
**Alternatives:** Hand-written CSRF (error-prone); no rate limiting (rejected for public endpoints).
**Consequences:** Default in-memory limiter storage is per process, so we run one worker initially and move to a shared store if we scale.

## D12. Audit log is append-only and survives deletions
**Status:** Accepted
**Context:** Hosts need to know who changed what, and guests' changes must be attributable without accounts.
**Decision:** `audit_logs` with actor type, JSON old and new values, and no cascading deletes (`SET NULL` for event and user, and plain integers for family and entity IDs).
**Alternatives:** Logging to files only (hard to query); full row versioning (heavier than needed).
**Consequences:** The table grows over time, so retention rules are a later concern. The app never updates or deletes its rows.

## D13. Self-hosted assets and hand-written charts
**Status:** Accepted
**Context:** Guest pages should make no third-party requests (token privacy, speed, strict CSP).
**Decision:** Lucide icons as a self-hosted SVG sprite, self-hosted fonts, and small inline SVG and CSS charts. Chart.js is reconsidered only if charts become complex.
**Alternatives:** CDN-hosted icons, fonts, and Chart.js (simpler, but they leak invitation URLs through Referer headers unless suppressed and add external dependencies).
**Consequences:** More setup and more of our own code, with full control of the visual design.

## D14. Small features written by hand, with one focused dependency for QR
**Status:** Accepted
**Context:** We avoid dependencies unless they pay for themselves.
**Decision:** `.ics` calendar files and CSV handling use the standard library. QR codes use **segno** (pure Python, SVG output, no Pillow).
**Alternatives:** `qrcode` with Pillow (heavier); a JavaScript QR library (adds client code to a page that doesn't need it).
**Consequences:** We must get `.ics` line folding and escaping right, and tests will cover it.

## D15. WhatsApp sharing through `wa.me` links
**Status:** Accepted
**Context:** The official WhatsApp Business API is paid, and the project avoids paid services.
**Decision:** Buttons open `https://wa.me/...` with prefilled text, and the host presses send.
**Alternatives:** WhatsApp Business API or an SMS gateway (paid); unofficial automation (against terms of service and fragile).
**Consequences:** Sending is manual, one family at a time. The "nudge pending" list makes this practical.

## D16. Testing: pytest with Python Playwright
**Status:** Accepted
**Context:** We need unit, integration, and browser tests, and one toolchain is easier to learn.
**Decision:** pytest for unit and integration tests, and `pytest-playwright` (Python bindings) for end-to-end tests.
**Alternatives:** Playwright's Node test runner (Node is installed, but two toolchains); Selenium (slower and more brittle).
**Consequences:** Browser binaries are a large download, so E2E tests are run separately from the fast suite.

## D17. No background-job infrastructure in v1
**Status:** Accepted
**Context:** Email is optional, and no workload needs a queue.
**Decision:** No Celery, Redis, or scheduler. Emails are sent inline with `smtplib`, and failures are logged without blocking the RSVP.
**Alternatives:** Celery and Redis (operational weight with no current need).
**Consequences:** Large email batches are not supported in v1. We revisit this if the need appears.

## D18. Git workflow: `main` and `develop` only
**Status:** Accepted
**Context:** Solo project with a requirement that `main` stays stable.
**Decision:** All work is committed to `develop`. `main` receives `--no-ff` merges at milestones M1, M2, and M3, each tagged. No force pushes.
**Alternatives:** Feature branches with pull requests (better for teams, extra ceremony for one developer).
**Consequences:** History on `develop` is linear and phase-based. Feature branches can be adopted later if collaborators join.

## D19. Hosting is deferred to Phase 9
**Status:** Accepted
**Context:** Free-tier terms change often, and some hosts sleep when idle or expire databases.
**Decision:** Choose the host in Phase 9 after checking current terms. Phase 0 commits only to Linux and Gunicorn plus PostgreSQL.
**Alternatives:** Choosing now from memory (rejected: likely outdated).
**Consequences:** The deployment section of `ARCHITECTURE.md` stays an outline until then.