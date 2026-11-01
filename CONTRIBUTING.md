# Contributing to Mehfil

## Branching strategy

| Branch | Purpose |
|---|---|
| `main` | Stable, tested milestone releases only. Never commit to it directly. |
| `develop` | Integration branch. All development commits go here. |

`develop` is merged into `main` with `git merge --no-ff` only after a milestone is complete, tests pass, and the result is manually verified. Milestones are tagged (`v0.1.0`, `v0.2.0`, `v1.0.0`).

Rules:
- Never force push.
- Run `git status` before every Git operation.
- Stage specific files (`git add <path>`), not everything. Never commit `.env`, `.venv`, or database files.
- Pull before pushing if the remote has changed.

## Commit conventions

Format: `type: short imperative summary` (about 72 characters or fewer).

| Type | Use for |
|---|---|
| `feat` | A new capability |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `test` | Adding or changing tests |
| `refactor` | Code change that doesn't alter behavior |
| `chore` | Tooling, configuration, dependencies |
| `style` | Formatting only |

Examples:
- `feat: implement guest management`
- `fix: prevent duplicate RSVP submissions`
- `test: add RSVP integration test coverage`

Commit once per completed, tested phase, or per clearly separate part of a phase. Avoid commits for tiny changes.

## Development guidelines

- **Routes stay thin.** Blueprints parse the request and call a function in `app/services/`. Business rules live in services, not routes.
- **Validate on the server.** Client-side checks are a convenience only.
- **Scope every query to the signed-in host's events**, so one host can never reach another's data.
- **Use SQLAlchemy models or parameterized queries.** Never build SQL with string concatenation.
- **Change the schema only through migrations.**
- **Write tests with the feature.** Business logic gets unit tests, and request flows get integration tests.
- **Accessibility is part of done:** keyboard navigation, visible focus, labelled form fields, sufficient contrast.
- **Keep docs current.** Update the relevant docs in the same phase as the code change.

## Development setup

> Finalized in Phase 1. Until then there is no application code to run.

Requirements: Git, Python 3.10+ (3.12 or newer recommended), Node.js (for Playwright browser tests later).