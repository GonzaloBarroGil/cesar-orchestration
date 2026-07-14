# CESAR Tech Debt — AppState Wire-Up Inconsistencies

> **Date:** 2026-07-14
> **Discovered:** P0 AppState wire-up analysis
> **Severity:** Medium — application works correctly, but AppState and handler patterns are inconsistent across modules. Fixing these before adding new modules prevents compounding the inconsistency.

---

## TD-1: Expenses handler constructs services per-request

**File:** `src/expenses/handler.rs`
**Pattern:** Each handler function clones the repo from AppState and calls `XService::new(repo)` on every HTTP request. Services are short-lived (dropped at response end). This is wasteful compared to pre-building services once at startup.

```rust
// Current (every request)
let repo = state.expense_repo.clone().ok_or(...)?;
let svc = ExpenseService::new(repo);
svc.find_by_id(id).await?;
```

**Fix:** Pre-build `ExpenseService`, `ConceptService`, `CategoryService`, `ProviderService` at AppState construction time, store them on AppState, and access them directly in handlers. Requires handler refactoring.

---

## TD-2: Incomes/outputs AppState field uses concrete generic type

**File:** `src/shared/state.rs:49-50`
**Pattern:** `income_output_service: Option<Arc<IncomeOutputService<PgIncomeOutputRepository>>>` — hardwires the PG implementation into AppState. Every other module uses `dyn Trait`.

```rust
// Current
pub income_output_service: Option<Arc<IncomeOutputService<PgIncomeOutputRepository>>>,

// Should be
pub income_output_service: Option<Arc<IncomeOutputService<dyn IncomeOutputRepository>>>,
```

**Fix:** Change AppState field to use `dyn IncomeOutputRepository`. Requires `IncomeOutputRepository` to be `?Sized`-compatible (already true via `async_trait`). Requires handler signature changes in `src/incomes_outputs/handler.rs`.

---

## TD-3: Payments service takes repos by value (not Arc)

**File:** `src/payments/service.rs`
**Pattern:** `PaymentService<P: PaymentRepository, R: ReciboRepository>` takes repos by value `(payment_repo: P, recibo_repo: R)`. Every other service takes `Arc<dyn Trait>`. This means the service OWNS its repos and they can't be shared.

```rust
pub fn new(
    payment_repo: P,        // owned by value — cannot share
    recibo_repo: R,         // owned by value — cannot share
    ...
) -> Self
```

**Fix:** Change to `Arc<dyn PaymentRepository>` and `Arc<dyn ReciboRepository>` like every other service. The invoices module already creates a `PgReciboRepository` — they can share the same `Arc<dyn ReciboRepository>`.

---

## TD-4: Debt handler accesses repos directly from AppState

**File:** `src/debt/handler.rs`
**Pattern:** Debt handler has 4 helper functions: `liquidacion_svc`, `certificado_svc`, `liquidacion_repo`, `certificado_repo`. The handler bypasses the service layer for some repo operations. Unclear if this is intentional design or leftover scaffolding.

```rust
fn liquidacion_repo(state: &AppState) -> Result<&(dyn LiquidacionRepository + 'static), AppError> {
    state.liquidacion_repo.as_deref()
        .ok_or_else(|| AppError::Internal("Debt module not yet configured".into()))
}
```

**Investigation needed:** Does the handler need direct repo access, or should all access go through the service? If services provide sufficient query methods, the `liquidacion_repo` and `certificado_repo` fields can be removed from AppState (they'd be private to the services).

---

## Summary

| # | Concern | Module | Effort | Risk |
|---|---|---|---|---|
| TD-1 | Per-request service construction | expenses | Medium | Handler refactor |
| TD-2 | Concrete generic type on AppState | incomes/outputs | Small | Handler signature changes |
| TD-3 | Repos owned by value | payments | Small | Service + AppState changes |
| TD-4 | Handler bypasses service layer | debt | Investigation | Remove/redundant fields |

**Priority:** Low. All 4 are code quality issues. The application compiles and passes all tests. Address before adding new modules (e.g., payments handler) to avoid compounding the inconsistency.
