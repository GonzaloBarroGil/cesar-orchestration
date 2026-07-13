# Ecosystem Assessment — 2026-07-13

> First ecosystem-level assessment after governance layer establishment.
> Assesses readiness, identifies blockers, and produces a prioritized cross-project backlog.

---

## 1. cesar (Backend)

### 1.1 What's Done

| Layer | Status | Details |
|---|---|---|
| **Domain models** | Complete | All 6 modules (consorcio, expenses, invoices, debt, payments, incomes_outputs). 12,839 lines total. Types, value objects, aggregate roots, repository traits, error enums, invariants. |
| **Services** | Complete | All 6 modules. 7,647 lines total. Business logic for prorrateo, interest calculation, invoice generation, payment reversal, etc. |
| **Handlers** | 5 of 6 | Axum handlers exist for consorcio, expenses, invoices, debt, incomes_outputs. **Payments handler missing** — routes not wired in `main.rs`. |
| **Shared foundation** | Complete | JWT auth, role guards, health probes (PG/Redis/NATS), idempotency middleware, telemetry, error framework. |
| **OpenAPI contract** | Complete | 50 endpoints across 6 modules. Merged into `api/openapi.yaml` (5,779 lines). |
| **BDD scenarios** | Complete | 29 `.feature` files across 7 modules (~614 scenarios). |
| **Tests** | Complete | 1,083 passing (487 domain unit + ~350 integration/property + ~200 service + 43 property). All pass. |
| **CONSTITUTION + ADRs** | Complete | 6 ADRs. ADR-006 (Ecosystem Governance) committed 2026-07-13. |
| **Legal compliance** | Complete | Reports for all 6 modules committed. |

### 1.2 What's Missing

| Gap | Severity | Impact |
|---|---|---|
| **PG repository implementations** | **Blocking** | 16 repository traits defined. Only 1 stub file exists (`PgIncomeOutputRepository`), and all 7 of its methods are `todo!()`. The other 5 modules have NO `repository.rs` file at all. |
| **Database migrations** | **Blocking** | Only 2 of ~22 tables migrated (`users`, `consorcios`). Missing: `unidades_funcionales`, `propietarios`, `periodos`, `exemptions`, `expenses`, `concepts`, `categories`, `providers`, `invoices`, `invoice_lines`, `notas_credito`, `notas_debito`, `recibos`, `liquidaciones`, `expense_allocations`, `interest_tables`, `compensatory_interests`, `certificados_deuda`, `payments`, `payment_history`, `incomes_outputs`. |
| **Payments handler** | **High** | 5 endpoints defined in OAS, service logic exists, but no Axum handler. Routes not in router. |
| **Production wire-up** | **Blocking** | `AppState::new()` sets all service/repo fields to `None`. No PG repository instantiation. All business endpoints return "module not configured". |
| **Auth/health in OpenAPI** | **Low** | `GET /health`, `POST /auth/login`, `GET /auth/me` exist in code but not in `api/openapi.yaml`. |

### 1.3 Test Coverage

All 1,083 tests pass against mock/in-memory repositories. No integration tests against a real PostgreSQL instance because the migrations don't exist.

---

## 2. cesar-web (Frontend)

### 2.1 What's Done

| Layer | Status | Details |
|---|---|---|
| **Foundation (W1)** | Complete | Vite + React + Tailwind + shadcn/ui + Biome + TanStack Query. Shared components (5 components, 42 tests). API plumbing (orval gen, MSW handlers, Zod schemas, axios client). |
| **Auth feature** | Complete | LoginPage, LoginForm, AuthProvider, AuthGuard, RoleGuard, HealthIndicator. 17 tests + a11y audit. |
| **Layout feature** | Complete | AppShell, Sidebar, Header, Breadcrumbs, ConsorcioContext. 24 tests + a11y audit. |
| **Consorcios feature (W2)** | Complete | ConsorcioListPage, ConsorcioForm, ConsorcioDetailPage, PropietarioList, PropietarioNewPage, UfTable, PeriodTimeline. 25 tests + a11y audit. |
| **BDD .feature files** | Specs only | 27 `.feature` files across all 9 modules (~177 scenarios). **Zero step definitions written** — no `.ts` files under `features/`. |
| **Generated API hooks** | Complete | All 7 endpoint modules generated via orval from OpenAPI spec. 156 model types. |
| **CONSTITUTION + ADRs** | Complete | 8 ADRs. ADR-008 (Ecosystem Integration) committed 2026-07-13. |

### 2.2 What's Missing

| Gap | Severity | Impact |
|---|---|---|
| **PLAN.md is out of sync** | **High** | PLAN claims all 69 tasks across all 7 waves are complete. Reality: 20/69 tasks done. Waves 3-7 have zero source code. This breaks orchestrator planning. |
| **Features W3-W6** | **Pending** | Expenses (T2.x), Invoices (T3.x), Debt (T4.x), Payments (T5.x), Incomes-Outputs (T6.x), Dashboard (T7.x) — 49 pending tasks. BDD specs and API hooks exist, but zero component code. |
| **BDD step definitions** | **Pending** | 177 scenarios defined, zero executable step definitions. |
| **CI pipeline** | **Missing** | `.github/workflows/` does not exist. No automated quality gates. |
| **Orval OAS source** | **Fragile** | Points to `/tmp/cesar-openapi-fixed.yaml` (generated, ephemeral). Should point to a committed copy or a sync script. |

### 2.3 PLAN.md Status vs Reality

| Wave | PLAN claims | Reality | Gap |
|---|---|---|---|
| W1 (Foundation) | ✓ Complete | ✓ Complete (12 tasks) | — |
| W2 (Consorcios) | ✓ Complete | ✓ Complete (8 tasks) | — |
| W3 (Expenses + Invoices) | ✓ Complete | **Not started** (0 of 16 tasks) | 16 tasks |
| W4 (Debt) | ✓ Complete | **Not started** (0 of 8 tasks) | 8 tasks |
| W5 (Payments + Incomes/Outputs) | ✓ Complete | **Not started** (0 of 16 tasks) | 16 tasks |
| W6 (Dashboard) | ✓ Complete | **Not started** (0 of 6 tasks) | 6 tasks |
| W7 (Validation) | ✓ Complete | **Not started** (0 of 4 tasks) | 4 tasks |
| **Total** | **69/69 claimed** | **20/69 actual** | **49 pending** |

---

## 3. Cross-Project Blockers

### 3.1 Blocking — Prevents integration

| # | Blocker | cesar status | cesar-web status | Priority |
|---|---|---|---|---|
| B1 | No PG repositories | 0 of 16 implemented | Generated hooks exist, can't integrate | **P0** |
| B2 | Missing DB migrations | 2 of ~22 tables | Can't test against real backend | **P0** |
| B3 | Payments handler missing | No Axum handler | Payments hooks generated but unreachable | **P1** |

### 3.2 Blocker Dependency Chain

```
B1 (PG repos) depends on → B2 (Migrations)
    → B2 must be done first (repos need tables to query)

B3 (Payments handler) depends on → B1 (PG repos)
    → Handler needs repository to inject

cesar-web T2 (Expenses) integration depends on → B1 + B2
cesar-web T3 (Invoices) integration depends on → B1 + B2
cesar-web T4 (Debt) integration depends on → B1 + B2
cesar-web T5 (Payments) integration depends on → B1 + B2 + B3
cesar-web T6 (Incomes/Outputs) integration depends on → B1 + B2
cesar-web T7 (Dashboard) integration depends on → B1 + B2
```

### 3.3 Not blocking — Can proceed now

- cesar-web W3-W6: MSW-based development (independent of backend)
- cesar-web CI setup (independent)
- cesar-web PLAN.md correction (independent)
- cesar OAS spec fixes (independent)
- cesar-web BDD step definitions (independent, uses MSW)

---

## 4. Known OAS Issues (from `cesar-openapi-spec-issues.md`)

| # | Issue | Severity | Fixed? |
|---|---|---|---|
| 1 | Invalid `$ref` to `#/components/headers/IdempotencyKey` (7 occurrences) | High | No — sed workaround |
| 2 | Missing `#/components/responses/` error definitions | Medium | No — stub types |
| 3 | Missing `GET /consorcios` list endpoint | Medium | No — MSW workaround |
| 4 | Missing auth/health endpoints in spec | Medium | No — hand-written types |

---

## 5. Prioritized Backlog

### Priority 0 — Blocking (backend readiness)

| Task | Project | Effort | Unblocks |
|---|---|---|---|
| **Write remaining DB migrations** (~20 tables) | cesar | Large (T0.3) | All PG repos, all cesar-web integration |
| **Implement PG repositories** (16 traits) | cesar | Large | All cesar endpoints, cesar-web integration |
| **Wire AppState with real repos** | cesar | Small | App becomes functional |

### Priority 1 — Unblock remaining features

| Task | Project | Effort | Unblocks |
|---|---|---|---|
| **Implement payments handler** | cesar | Medium | Payments routes |
| **Fix PLAN.md** (reset gates for W3-W7) | cesar-web | Small | Accurate orchestrator planning |
| **Add auth/health to OpenAPI spec** | cesar | Small | cesar-web OAS completeness |

### Priority 2 — Can proceed in parallel

| Task | Project | Effort | Notes |
|---|---|---|---|
| **cesar-web W3 (Expenses)** — MSW-based | cesar-web | 8 tasks | Independent — hooks generated |
| **cesar-web W3 (Invoices)** — MSW-based | cesar-web | 8 tasks | Independent — hooks generated |
| **cesar-web CI pipeline** | cesar-web | Small | GitHub Actions |
| **Fix orval OAS source** | cesar-web | Small | Replace /tmp/ workaround |
| **Write BDD step definitions** | cesar-web | Ongoing | For all 27 .feature files |

### Priority 3 — Quality and polish

| Task | Project | Effort |
|---|---|---|
| **Fix OAS spec issues** (#1-#4) | cesar | Small |
| **W4-W6 cesar-web features** | cesar-web | 30 tasks |
| **Cross-project BDD traceability matrix** | cesar-orchestration | Small |
| **Release log** | cesar-orchestration | Small |

---

## 6. Recommended Next Actions

### Immediate (this session)

1. **Fix cesar-web PLAN.md** — Reset W3-W7 gates from "complete" to "pending". Blocking accurate orchestration.
2. **Begin cesar DB migrations** — Highest-priority blocking work. Unlocks everything downstream.

### Short-term (next 1-2 sessions)

3. **Complete cesar migrations** — All 6 modules
4. **Implement cesar PG repositories** — Start with consorcio + expenses (most needed by cesar-web)
5. **cesar-web CI pipeline** — Basic check: lint, typecheck, test

### Medium-term

6. **cesar-web W3 (Expenses + Invoices)** — MSW-based implementation
7. **cesar-web BDD step definitions** — Executable tests for existing .feature files
8. **cesar integration testing** — Once PG repos exist, verify real endpoints
