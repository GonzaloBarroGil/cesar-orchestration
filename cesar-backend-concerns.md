# CESAR Backend Concerns — Web Project Impact

> **Context:** Issues discovered during CESAR Web CFFD phases that depend on
> CESAR backend resolution.
> **Date:** 2026-06-25
> **Source Phases:** Feasibility Analysis → Contract Setup → SPEC (auth)

---

## 1. Missing Auth Endpoints

**Discovered:** SPEC phase — auth
**Severity:** High — blocks typed auth hooks

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

## 2. Missing `GET /consorcios` List Endpoint

**Discovered:** Feasibility Analysis §7
**Severity:** Medium — frontend cannot populate consorcio selector

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

## 6. Only 2 DB Migrations Exist

**Discovered:** Feasibility Analysis §3.2
**Severity:** High — blocks backend integration

### Description

CESAR has only 2 SQLx migrations (users, consorcios). The remaining ~18+
tables (expenses, invoices, debt, payments, incomes-outputs and their
supporting tables) have not been migrated.

### Frontend Impact

- When the frontend switches from MSW to the real backend, many endpoints
  will fail because database tables don't exist.
- This is the primary blocker for integration testing.

### What the Backend Needs

Complete all migrations for Phase 1 modules: expenses schema, invoices schema
(JSONB for items and adicionales_rg), debt schema, payments schema,
incomes-outputs schema, plus junction tables and audit event tables.

---

## 7. Many Repository Implementations Are `todo!()` Stubs

**Discovered:** Feasibility Analysis §3.2
**Severity:** High — blocks backend integration

### Description

Multiple repository layer implementations contain `todo!()` macro stubs.
Service layers may reference repositories that return mock data or panic.

### Frontend Impact

Same as §6 — switching from MSW to real backend will fail on stubbed endpoints.

### What the Backend Needs

Replace all `todo!()` stubs with real SQLx implementations.

---

## Summary

| # | Concern | Severity | Blocking v0.1? | Phase |
|---|---|---|---|---|
| 1 | Missing auth endpoints in spec | High | Yes | SPEC |
| 2 | Missing `GET /consorcios` endpoint | Medium | Yes (integration) | Feasibility |
| 3 | PDF liquidacion returns 501 | Low | No (deferred) | Feasibility |
| 4 | No dashboard aggregation | Medium | No (client-side) | Feasibility |
| 5 | No user management endpoints | Low | No (out of scope) | Feasibility |
| 6 | Only 2 DB migrations exist | High | Yes (integration) | Feasibility |
| 7 | Many repository `todo!()` stubs | High | Yes (integration) | Feasibility |

**Verdict:** The frontend can proceed with full development against MSW mocks.
Three concerns block backend integration: auth endpoints in spec (§1), DB
migrations (§6), and `todo!()` stubs (§7). These must be resolved before the
first integration test against the real backend.

---

*End of backend concerns.*
