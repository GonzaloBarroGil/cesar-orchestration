# CESAR Tech Debt

> **Date:** 2026-07-14. Updated 2026-07-14.
> **Discovered:** P0 AppState wire-up analysis + P1 payments handler implementation
> **Severity:** Low — all items are code quality issues. No behavior is affected.
> **Deferral rationale:** TD-1, TD-2, and TD-4 are pure internal consistency issues with no user-facing impact. Deferred to post-v0.1 when the internal architecture can be refactored without blocking cesar-web integration.

---

## Resolved

### TD-5: PaymentService missing `get_payment` and `list_payments` methods ✅ FIXED 2026-07-14

**Was:** `PaymentService` exposed `create_payment`, `reverse_payment`, `get_unit_balance`, `get_payment_history` but NOT `get_payment(id)` or `list_payments(filters)`. The OpenAPI spec defined `GET /api/v1/payments/{id}` and `GET /api/v1/payments` but the handler couldn't implement them.

**Fix:** Added `get_payment` and `list_payments` to `PaymentService` (delegating to existing `PaymentRepository` trait methods). Added `GET /api/v1/payments/{id}` and `GET /api/v1/payments` routes to the handler.

**Files:** `src/payments/service.rs`, `src/payments/handler.rs`

**Note:** This was a genuine feature gap, not code quality debt. The OAS contract defined endpoints that didn't exist in the service layer.

### TD-3: Payments service took repos by value (not Arc) ✅ FIXED 2026-07-14

**Was:** `PaymentService<P: PaymentRepository, R: ReciboRepository>` took repos by value. Every other service uses `Arc<dyn Trait>`. This meant the invoices module and payments module each had their own `PgReciboRepository` instance.

**Fix:** Changed `PaymentService` to non-generic struct taking `Arc<dyn PaymentRepository>` + `Arc<dyn ReciboRepository>`. The recibo repo is now shared between `ReciboService` and `PaymentService`. Removed generic params from handler's `payment_svc()` helper.

**Files:** `src/payments/service.rs`, `src/payments/handler.rs`, `src/shared/state.rs`

---

## Deferred to post-v0.1

### TD-1: Expenses handler constructs services per-request

**File:** `src/expenses/handler.rs`
**Pattern:** Each handler function clones the repo from AppState and calls `XService::new(repo)` on every HTTP request. Services are short-lived.

**Fix:** Pre-build services at AppState construction time. Requires handler refactoring in 4 handler functions.

**Risk:** Medium — touching handlers that currently work. No user-facing impact.

### TD-2: Incomes/outputs AppState field uses concrete generic type

**File:** `src/shared/state.rs`
**Pattern:** `income_output_service: Option<Arc<IncomeOutputService<PgIncomeOutputRepository>>>` hardwires the PG implementation. Every other module uses `dyn Trait`.

**Fix:** Change to `Arc<IncomeOutputService<dyn IncomeOutputRepository>>`. Requires handler signature changes.

**Risk:** Small — handler type adjustments. No user-facing impact.

### TD-4: Debt handler accesses repos directly from AppState

**File:** `src/debt/handler.rs`
**Pattern:** Debt handler has `liquidacion_repo` and `certificado_repo` helpers, bypassing the service layer for some operations.

**Fix:** Investigate whether handlers genuinely need direct repo access. If services can provide equivalent query methods, remove the repo fields from AppState.

**Risk:** Investigation — unclear if this is intentional design. No user-facing impact.

---

## Summary

| # | Concern | Status |
|---|---|---|
| TD-1 | Per-request service construction (expenses) | Deferred post-v0.1 |
| TD-2 | Concrete generic type on AppState (incomes/outputs) | Deferred post-v0.1 |
| TD-3 | Repos owned by value (payments) | ✅ FIXED 2026-07-14 |
| TD-4 | Handler bypasses service layer (debt) | Deferred post-v0.1 |
| TD-5 | Missing get/list payments service methods | ✅ FIXED 2026-07-14 |
