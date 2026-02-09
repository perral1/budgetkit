# AGENTS.md

Repository instructions for people and coding agents working in this project.

## Repository Structure

- `database_spec.md`: Authoritative PocketBase schema, data semantics, and budgeting rules.
- `module_best_practices.md`: Auth/session/API/querying/UX conventions for standalone modules.

## Source of Truth

- Database design decisions belong in `database_spec.md`.
- Module implementation conventions belong in `module_best_practices.md`.
- Avoid duplicating normative rules across files.

## How to Add a New Module

1. Create one standalone HTML file in repo root unless a different location is agreed.
2. Keep CSS and JavaScript inline.
3. Implement auth exactly as documented in `module_best_practices.md`.
4. Use only schema fields guaranteed by `database_spec.md`.
5. Guard list endpoints against schema/filter mismatch (HTTP 400 fallback strategy).
6. Ensure module works from `file://`, `http://`, and `https://`.
7. Confirm refresh/reload preserves session behavior.

## Module Checklist

- Login, refresh, and logout flows work with PocketBase `users` auth.
- Authorization header uses `Bearer <token>`.
- Passwords are never stored.
- UI cleanly switches between login and authenticated module state.
- Errors differentiate auth vs network vs query/schema failures.

## Editing Guidelines

- Keep docs concise and normative.
- If behavior changes, update specs first, then update module code.
- Prefer additive migrations for schema changes; document new fields/indexes in `database_spec.md`.
