# BudgetKit Module Best Practices

This file defines implementation guidance for standalone BudgetKit modules.

## 1. Module Format

Required:
- Single static HTML file.
- Inline CSS and inline JavaScript.
- No frameworks (React, Vue, etc.).
- Must run from `file://`, `http://`, and `https://`.
- Must survive refresh/reload without losing usable session state.

## 2. Authentication

Canonical auth model:
- Auth collection is always `users`.
- Normal modules do not offer admin/superuser login.

Endpoints:
- Login: `POST /api/collections/users/auth-with-password`
- Token refresh: `POST /api/collections/users/auth-refresh`

Storage (localStorage only):
- `pbUrl`: PocketBase base URL (no trailing slash)
- `pbEmail`: last-used email (prefill only)
- `pbToken`: JWT

Rules:
- Never persist passwords.
- Keep login UI and app UI mutually exclusive.
- Disable module actions while unauthenticated.

## 3. Required Login/Session Flow

On initial load:
1. Read `pbUrl` and `pbToken`.
2. If both exist, call `auth-refresh`.
3. If refresh succeeds:
   - update token if returned
   - hide login UI
   - show logout link
   - load module data
4. If refresh fails:
   - clear token
   - show login UI

On successful login:
- save `pbUrl`, `pbEmail`, `pbToken`
- hide login UI
- load module data

On logout:
- clear `pbUrl`, `pbEmail`, `pbToken`
- reset in-memory auth state
- show login UI
- clear module data

## 4. API and Error Handling

Authorization header (always):
- `Authorization: Bearer <token>`

Network failures:
- Catch `fetch()` errors and rethrow clear messages.
- Mention likely causes when relevant: CORS, mixed content (`https` -> `http`), unreachable host/port.

Example error:
- `Network error (fetch failed): check URL, protocol, and CORS`

## 5. Safe Querying Patterns

When listing records:
1. Do not sort by fields unless they are guaranteed to exist.
2. Filter only on known schema fields.
3. If list request returns HTTP 400:
   - retry without optional filters
   - then retry with no filters as last resort

Why:
- PocketBase may reject invalid filters/sorts with HTTP 400.

## 6. UX and Diagnostics

- Show exact HTTP status when available.
- Distinguish auth failures from schema/filter failures and network failures.
- Avoid generic catch-all errors when a specific class is known.

## 7. Archived Record Handling

- For collections with `is_archived`, do not offer archived records in search/autocomplete options used to create or edit transactions/entries.
- If an existing record already references an archived account/category/payee, still load and display that linked value in read and edit views.
- Archived-linked transactions/entries must remain included in all totals and calculations.
- Where a list module manages archived-capable collections, provide filters for `unarchived`, `archived`, and `both`.
- For budget views that hide archived categories, hide only line items; keep archived categories in group totals and overall totals.
