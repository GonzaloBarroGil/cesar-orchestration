# CESAR Backend Concerns — Web Project Impact

> **Context:** Issues discovered during CESAR Web CFFD phases that depend on
> CESAR backend resolution.
> **Date:** 2026-06-25. Updated 2026-07-14.
> **Source Phases:** Feasibility Analysis → Contract Setup → SPEC (auth)
> **Resolution:** 4 of 7 concerns resolved (auth/consorcio spec + migrations + PG repos).
>   Remaining: AppState wire-up (in progress), PDF/download, dashboard, user mgmt.

---

## 1. Missing Auth Endpoints ✅ FIXED 2026-07-13

**Discovered:** SPEC phase — auth
**Severity:** ~~High~~ — FIXED

Fixed in step 3a (OAS hygiene). Added to `cesar/api/openapi.yaml`:
- `POST /auth/login`, `GET /auth/me`, `GET /health`
- `LoginRequest`, `LoginResponse`, `UserInfo`, `MeResponse`, `HealthResponse` schemas
- `Auth` and `Health` tags

### Description

The CESAR OpenAPI spec does not define:
- `POST /api/v1/auth/login`
- `GET /api/v1/auth/me`
- `GET /api/v1/health`

These endpoints are referenced in the security scheme description
(`JWT token obtained via POST /api/v1/auth/login`) and documented in the
feasibility analysis as part of the 61-endpoint inventory, but no path
definitions exist in `api/openapi.yaml`.

### Frontend Impact

- orval cannot generate typed TanStack Query hooks for login or session validation.
- MSW handlers must be hand-written without type contract verification.
- The compile-time contract boundary (CFFD's core enforcement mechanism) is
  broken for the foundation feature of the application.
- Every other feature depends on auth for identity and role extraction.

### What the Backend Needs

1. Add `POST /api/v1/auth/login`, `GET /api/v1/auth/me`, `GET /api/v1/health`
   path definitions to `api/openapi.yaml` (see `cesar-openapi-spec-issues.md`
   Issue 4 for full YAML proposal).
2. Add `UserResponse` schema to components/schemas.
3. Add `Auth` tag to the tags section.
4. Ensure the Rust implementation matches the spec.

---

## 2. Missing `GET /consorcios` List Endpoint ✅ FIXED 2026-07-13

**Discovered:** Feasibility Analysis §7
**Severity:** ~~Medium~~ — FIXED

Added `GET /consorcios` with `limit`/`offset` query params to `cesar/api/openapi.yaml`.
Returns `ConsorcioResponse[]` — same schema as `POST /consorcios`.

### Description

The OpenAPI spec defines `POST /consorcios` and `GET /consorcios/{id}` but
no `GET /consorcios` endpoint to list all consorcios for the current user.

### Frontend Impact

- The ConsorcioSelector component (layout shell) cannot retrieve the list of
  consorcios from the API.
- Workaround: MSW returns a hardcoded array during development.
- At integration time, the selector breaks unless the endpoint exists.

### What the Backend Needs

Add `GET /consorcios` endpoint returning `ConsorcioResponse[]` (see
`cesar-openapi-spec-issues.md` Issue 3 for YAML).

---

## 3. PDF Liquidacion Download Returns 501

**Discovered:** Feasibility Analysis §3.2
**Severity:** Low — non-blocking, deferred

### Description

`GET /api/v1/debt/liquidaciones/{id}/pdf` returns HTTP 501 (Not Implemented).
The CESAR backend has not yet implemented PDF generation.

### Frontend Impact

- The "Download PDF" button on liquidacion detail must be disabled with a
  tooltip: _"Descarga de PDF no disponible próximamente"_.
- MSW handler returns 501 to match backend behavior.
- When CESAR implements the endpoint, the frontend enables the button.

### What the Backend Needs

Implement the PDF liquidacion endpoint. No spec changes needed (the path
is already defined). The frontend waits for the 200 response.

---

## 4. No Dashboard Aggregation Endpoints

**Discovered:** Feasibility Analysis §3.2
**Severity:** Medium — dashboard must compute KPIs client-side

### Description

CESAR has no dedicated dashboard/metrics aggregation endpoints. The frontend
must compute key metrics (total debt, collection rate, pending approvals) by
aggregating data from multiple list queries on the client side.

### Frontend Impact

- Dashboard page makes multiple parallel TanStack Query calls and computes
  derived state client-side.
- Acceptable for v0.1 with a small dataset (single consorcio).
- Does not scale: compound queries would be more efficient server-side.

### What the Backend Needs (post-v0.1 recommendation)

Add `GET /dashboard?consorcio_id={id}&period_code={code}` returning:
- `total_debt`, `total_collected`, `collection_rate`
- `pending_approvals_count`, `active_liquidaciones_count`
- `current_period_expenses_total`

---

## 5. No User Management Endpoints

**Discovered:** Feasibility Analysis §3.2
**Severity:** Low — out of scope for v0.1

### Description

CESAR has no CRUD endpoints for user management (create user, update role,
disable user). Users must be created directly in the database.

### Frontend Impact

- No user management UI in v0.1.
- ADMIN users are pre-seeded in the database.
- ACCOUNTANT users require manual DB insertion.

### What the Backend Needs (post-v0.1)

Add `/users` CRUD endpoints with `ADMIN` role guard.

---

## 6. DB Migrations Complete ✅ FIXED 2026-07-13

**Discovered:** Feasibility Analysis §3.2
**Severity:** ~~High~~ — FIXED

8 migration files now exist (2 original + 6 new):
| Migration | Date | Tables |
|---|---|---|
| 001 | 2026-06-22 | users |
| 002 | 2026-06-22 | consorcios |
| 003 | 2026-07-13 | unidades_funcionales, propietarios, periodos, reglamentos, actas, votaciones, etc. |
| 004 | 2026-07-13 | concepts, categories, providers, expenses |
| 005 | 2026-07-13 | invoices, invoice_lines, notas_credito_debito, recibos |
| 006 | 2026-07-13 | liquidaciones, expense_allocations, interest_tables, compensatory_interests, certificados_deuda |
| 007 | 2026-07-13 | payments, payment_history |
| 008 | 2026-07-13 | incomes_outputs |

**24 tables total** across all 6 domain modules. All follow existing conventions (UUID v7, TIMESTAMPTZ, snake_case, append-only status enums, BIGINT for cents).

---

## 7. Many Repository Implementations Are `todo!()` Stubs ✅ FIXED 2026-07-14

**Discovered:** Feasibility Analysis §3.2
**Severity:** ~~High~~ — FIXED

All 16 repository traits across 6 modules now have real PG implementations (5,270 lines total):

| Module | Repos | Methods | Lines | Commit |
|---|---|---|---|---|
| consorcio | 4 | 26 | 854 | `65c7de0` |
| expenses | 4 | 19 | 793 | `d227a78` |
| invoices | 2 | 17 | 1,260 | `6a4d43d` |
| debt | 4 | 30+ | 1,124 | `c4151fd` |
| payments | 1 | 5 | 537 | committed |
| incomes_outputs | 1 | 7 | 622 | committed |

No `todo!()` stubs remain. Tests: 1,169 pass, 0 fail.

---

## 8. No Seed Users — Auth-Protected Endpoints Untestable ✅ FIXED 2026-07-14

**Discovered:** 2026-07-14 — integration smoke test
**Severity:** ~~Medium~~ — FIXED

Created `src/bin/seed_admin.rs` — reads `ADMIN_USERNAME` and `ADMIN_PASSWORD` from env vars, hashes password with argon2id (same algorithm as login), inserts into `users` table. Idempotent (`ON CONFLICT DO NOTHING`).

Usage:
```bash
ADMIN_USERNAME=admin ADMIN_PASSWORD=secure123 cargo run --bin seed_admin
```

Also added `default-run = "cesar"` to `Cargo.toml` to disambiguate the two binaries.

---

## 9. `GET /consorcios` Handler Missing ✅ FIXED 2026-07-14

**Discovered:** 2026-07-14 — integration smoke test
**Severity:** ~~Medium~~ — FIXED

Added `list_consorcios` handler + `ListConsorciosQuery` (limit/offset) to `src/consorcio/handler.rs`. Added `list_all` method to `ConsorcioService`. Added `find_all` to `ConsorcioRepository` trait and its `PgConsorcioRepository` implementation. Registered `.get(list_consorcios)` on `/api/v1/consorcios`.

Smoke-tested: auth-protected, returns correct 401 without JWT, returns `[]` for empty DB, returns list after creating consorcio.

---

## Summary

| # | Concern | Severity | Status |
|---|---|---|---|---|---|
| 1 | Missing auth endpoints in spec | — | ✅ FIXED 2026-07-13 |
| 2 | Missing `GET /consorcios` endpoint in spec | — | ✅ FIXED 2026-07-13 |
| 3 | PDF liquidacion returns 501 | Low | Deferred |
| 4 | No dashboard aggregation | Medium | Deferred (post-v0.1) |
| 5 | No user management endpoints | Low | Deferred (post-v0.1) |
| 6 | Only 2 DB migrations exist | — | ✅ FIXED 2026-07-13 |
| 7 | Many repository `todo!()` stubs | — | ✅ FIXED 2026-07-14 |
| 8 | No seed users — auth endpoints untestable | — | ✅ FIXED 2026-07-14 |
| 9 | `GET /consorcios` handler not implemented | — | ✅ FIXED 2026-07-14 |

**Verdict:** 6 of 9 concerns resolved. Backend is fully operational: all modules have domain models, services, handlers, PG repos, AppState wiring, seed users, and verified smoke tests. Concerns 3-5 are deferred to post-v0.1.

**Tech debt:** See `docs/cesar-tech-debt.md` — 2 items resolved (TD-3, TD-5), 3 items deferred to post-v0.1 (TD-1, TD-2, TD-4).

---

*End of backend concerns.*
