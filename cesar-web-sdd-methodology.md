# CESAR Web — SDD Methodology & Multi-Agent Framework

> **Date:** 2026-06-25
> **Status:** Draft — first approach
> **Context:** Frontend methodology deriving from CESAR backend governance
> **Audience:** Developers adopting CESAR's SDD/BDD/TDD approach for the web app

---

## 1. Why This Document

CESAR's backend was built under a strict Spec-Driven Development (SDD) model with
8 subagents, HITL gates, BDD Gherkin scenarios, and TDD at all layers — codified
in `docs/CONSTITUTION.md`. The frontend inherits the OpenAPI contract from the
backend but must define its own governance model adapted to:

- **External contract ownership** — the OpenAPI spec is authored by CESAR backend, not the frontend team.
- **Different test layers** — UI component tests, E2E flows, and mock-based integration vs domain/property-based tests.
- **Smaller team** — 1–2 developers vs CESAR's potential multi-agent parallel execution.
- **JavaScript/TypeScript ecosystem** — different tooling, different licensing landscape, different quality enforcement mechanisms.

This document defines the **Contract-First Frontend Development (CFFD)**
variant of SDD, establishes the agent framework, estimates BDD scenario coverage,
and codifies governance rules that will be formalized later by the
`constitution-architect` agent in a dedicated constitution phase.

---

## 2. CFFD — Contract-First Frontend Development

### 2.1 What It Is

CFFD is a derivative of SDD where the API contract is **externally owned but
internally enforced**. Unlike backend SDD where the team authors the spec first,
CFFD treats the OpenAPI 3.1 specification as an immutable external constraint:

| Dimension | Backend SDD (CESAR) | Frontend CFFD (cesar-web) |
|---|---|---|
| Spec ownership | Writes & owns OpenAPI | Consumes OpenAPI |
| Spec changes | Internal; any module can trigger a spec update | External; spec changes arrive from backend team |
| Contract enforcement | `schemars` validates responses at runtime against OpenAPI | TypeScript + orval catches mismatches at **compile time** |
| BDD layer | Gherkin scenarios test API endpoints directly (subcutaneous) | Gherkin scenarios test UI flows against MSW-mocked API |
| TDD layer | Domain → service → handler | Component → page → integration flow |
| Primary test types | Unit, integration, property-based, BDD | Component, integration, E2E, BDD |
| Spec drift protection | Schema validation at integration test time | Compilation error: TypeScript refuses to build |

### 2.2 The Compile-Time Contract Boundary

The key architectural invariant:

```
CESAR OpenAPI (api/openapi.yaml)
        │
        ├─→ orval ──→ TypeScript types (src/api/generated/)
        │                  │
        │                  ├─→ typed API hooks (TanStack Query)
        │                  ├─→ Zod schemas (request/response)
        │                  └─→ if the spec removes a field → **build fails**
        │
        └─→ MSW ──→ mock handlers (src/mocks/handlers/)
                       │
                       └─→ if the spec adds an endpoint → handler absent → test fails
```

This makes **specification drift** a compile-time error, not a runtime surprise.
When CESAR's backend team changes the OpenAPI spec, `orval` regenerates, and
TypeScript immediately enumerates every call site that needs updating.

### 2.3 Relationship to CESAR Backend SDD

| CESAR Principle | Frontend CFFD Adaptation |
|---|---|
| **P1 — Spec First** | Contract First: OpenAPI is the spec; no frontend code without a corresponding API endpoint defined in the OpenAPI |
| **P2 — BDD/TDD Woven** | BDD scenarios from CESAR are **mirrored** as E2E test flows; TDD at component level (render → interact → assert) |
| **P3 — Immutable Audit** | Not applicable to frontend (audit is a backend concern) |
| **P4 — Idempotency Everywhere** | Idempotency-Key header injected by API client; MSW mimics 24h TTL dedup |
| **P5 — Strict Typing** | TypeScript strict mode; no `any`; Zod validates all API responses and form inputs |
| **P6 — Open Telemetry** | Frontend telemetry via `@opentelemetry/web` for page loads, API latency, user interactions |
| **P7 — Open Source First** | Adopted verbatim for npm ecosystem (see §5.3) |
| **P8 — Subagent-Ready** | Adapted: 4 agents instead of 8 (see §3) |

---

## 3. Agent Framework

### 3.1 CESAR Baseline

CESAR's CONSTITUTION defines 8 subagents. The PLAN activates 6 of them across
80 tasks:

| Agent | Tasks | Scope |
|---|---|---|
| domain-modeler | 19 | Rust value objects, entities, invariants, repository traits |
| rust-developer | 33 | Services, handlers, migrations, Docker, middleware |
| test-engineer | 13 | Unit, integration, BDD, property-based tests |
| api-contract-designer | 7 | Per-module OpenAPI schemas + merge |
| bdd-scenario-writer | 6 | Gherkin `.feature` files per module |
| legal-compliance-checker | 2 | CCC, CABA, ARCA audit of all artifacts |

### 3.2 Frontend Reduction — Why 4, Not 8

The frontend eliminates four backend-only concerns and merges two others:

**Eliminated:**

| Agent | Why Not Needed |
|---|---|
| **domain-modeler** | OpenAPI defines domain types. orval generates TypeScript types from the spec. No manual domain modeling needed — the contract *is* the domain model. |
| **api-contract-designer** | The contract is externally owned by CESAR backend. Frontend consumes it. The `contract-guardian` agent manages the pipeline, but does not author the spec. |
| **spec-author** | Feature specs for the frontend are simpler (page flows, form validation rules, navigation trees) and are handled inline by `feature-developer` or folded into the BDD scenarios by `quality-engineer`. No standalone spec-author role needed. |
| **legal-compliance-checker** | Legal compliance (CCC, ARCA, CABA) is the backend's responsibility. Frontend enforces presentation-layer rules (CUIT formatting, IVA display, invoice type selection) but does not perform legal-domain validation of the kind that requires a dedicated checker. |

**Merged:**

| CESAR Agent | Maps To | Rationale |
|---|---|---|
| bdd-scenario-writer + test-engineer | **quality-engineer** | Frontend BDD scenarios are UI flows (shorter, fewer, simpler). Writing and implementing them is a single workflow for a single agent. |
| rust-developer | **feature-developer** | All implementation (components, hooks, forms, routing, state) collapses into one agent. No system/infrastructure coding needed (Vite handles that). |

**Retained:**

| CESAR Agent | Maps To | Rationale |
|---|---|---|
| constitution-architect | **constitution-architect** | Unchanged role. Sets governance, ADRs, SDD workflow for the frontend. |

### 3.3 The 4-Agent Roster

| # | Agent | Role | Input | Output |
|---|---|---|---|---|
| 1 | **constitution-architect** | Defines governance + ADRs | Methodology doc + CESAR CONSTITUTION + project requirements | `docs/CONSTITUTION.md`, `.opencode/agents/*.md`, `.opencode/opencode.json` rules |
| 2 | **contract-guardian** | Manages OpenAPI → orval → MSW pipeline | `cesar/api/openapi.yaml` | `src/api/generated/` types, `src/mocks/handlers/` MSW handlers, Zod schemas |
| 3 | **feature-developer** | Implements pages, components, hooks, forms | OpenAPI types + BDD scenarios + UI specs | `src/features/<module>/`, `src/components/`, `src/hooks/` |
| 4 | **quality-engineer** | All test layers + BDD scenario coverage + accessibility | Feature code + OpenAPI contract + BDD scenarios | Vitest tests, Playwright E2E, BDD scenarios, a11y reports |

### 3.4 Task Distribution Estimate

Based on the feasibility analysis's four development phases and the backend's
task count (80), a rough frontend task estimate:

| Agent | Estimated Tasks | Phase Coverage |
|---|---|---|
| constitution-architect | ~6 | Phase 0 (governance, ADRs, agent definitions, rule formalization) |
| contract-guardian | ~8 | Phase 0 (setup) + all phases (maintain spec pipeline, update MSW on spec changes) |
| feature-developer | ~35 | Phase 1 (core admin), Phase 2 (financial ops), Phase 3 (reports), Phase 4 (polish) |
| quality-engineer | ~15 | Across all phases (BDD, E2E, component tests, a11y audits) |
| **Total** | **~64** | |

This compares to CESAR's 80 tasks across 6 agents. The frontend has fewer tasks
per agent because there is no domain modeling, no 6× separate API contract
authoring, and no per-module DB migration work.

### 3.5 Accessibility — Sub-Role, Not Separate Agent

A 5th agent (`accessibility-auditor`) was considered but rejected at this
scale. The reasoning follows CESAR's own pattern for `legal-compliance-checker`:

| Trait | `legal-compliance-checker` (CESAR) | `accessibility-auditor` (hypothetical) |
|---|---|---|
| When active | End of phase (audit gate) | End of phase / per-feature gate |
| Continuous work? | No — audit, report, hand off | No — audit, report, hand off |
| Owns implementation? | No — `rust-developer` fixes | No — `feature-developer` fixes |
| Task count | 2 of 80 (2.5%) | ~3 of 64 (4.7%) |
| Justifies dedicated agent? | Yes (domain expertise in Argentine law) | No — a11y expertise lives in `quality-engineer` |

A dedicated `accessibility-auditor` would introduce coordination overhead for
minimal task volume. Instead:

- **`quality-engineer`'s scope expands** to include axe-core audits, keyboard
  navigation tests, screen reader label verification, and WCAG 2.1 AA compliance
  reports as primary deliverables — not a footnote in Quality Gate G6.
- **Agent definition** lists a11y checklist alongside BDD and E2E in its output
  contract.
- **If a11y workload grows** (e.g., 3+ developers, government certification),
  the role can be split out later — same as CESAR's ADR-001 (modular monolith →
  microservices when scaling demands it).

This keeps the roster at 4 while giving accessibility equal prominence to BDD
and E2E in the `quality-engineer`'s responsibilities.

---

## 4. BDD Scenario Framework

### 4.1 CESAR Baseline

CESAR backend has **29 `.feature` files** across 7 modules with approximately
**614 Gherkin scenarios**:

| Module | Feature Files | Scenarios | Type |
|---|---|---|---|
| Consorcio | 4 | ~130 | Domain invariants (percentage sums, CUIT validation, period lifecycle, UF exemptions) |
| Expenses | 8 | ~129 | Expense types, approval, audit hash chain, multi-term, providers |
| Invoices | 7 | ~119 | Factura A/B/C, NC/ND, Recibo, CAE, QR, IVA, invariants |
| Debt | 3 | ~74 | Prorrateo math, interest calculation, liquidacion cascade, certificado |
| Payments | 3 | ~57 | Payment creation, reversal, query, Recibo auto-gen |
| Incomes/Outputs | 2 | ~95 | CRUD, period validation, summaries |
| Shared | 2 | ~10 | Auth, health check |

These scenarios are **subcutaneous** (test API endpoints directly, no browser)
and encode CCC/CABA/ARCA legal requirements as behavioral specifications.

### 4.2 Frontend BDD Difference

Frontend BDD scenarios are fundamentally different:

| Dimension | Backend BDD | Frontend BDD |
|---|---|---|
| What it tests | Domain invariants, financial correctness, legal compliance | User flows, UI state, form validation, navigation, role-based access |
| Test surface | API endpoint (HTTP request → HTTP response) | UI (user interaction → rendered DOM) |
| Scenario length | Average 8–12 steps with data tables | Average 5–8 steps with form fills and assertions |
| Edge case density | High (prorrateo edge cases, rounding, cascade) | Medium (form errors, loading/empty/error states, role guards) |
| Tooling | `cucumber` crate or `rstest` in Rust | Playwright + Cucumber.js or Playwright component testing |
| Data | Inline data tables in Gherkin | MSW fixture data shared across scenarios |

### 4.3 Frontend BDD Scenario Estimate

Frontend BDD scenarios focus on **user workflows**, not domain math. The
estimate below maps each module to the key user flows:

| Module | Estimated Scenarios | Covers |
|---|---|---|
| Auth | 10 | Login happy/sad, logout, expired token, role extraction, token refresh, unauthorized redirect |
| Consorcios | 25 | CRUD, UF configuration with percentage validation, propietario management (register, list, find by tax ID), period lifecycle (open/close/archive) |
| Expenses | 30 | CRUD, type toggle (ORDINARIA/EXTRAORDINARIA), approval/rejection workflow, provider CRUD, concept/category management, soft-delete, version history view |
| Invoices | 28 | Factura A form (RI → RI), Factura B form (RI → CF/Exento), Factura C form (CF), NC/ND creation linked to invoice, Recibo creation, annul flows |
| Debt | 22 | Liquidacion generation (single unit), batch generation, list with filters, detail view with expense allocation, issue (DRAFT → ISSUED), certificado generation and signing |
| Payments | 18 | Register payment with method selector, reversal flow, unit balance summary, payment history with sort/filter |
| Incomes/Outputs | 18 | CRUD, filters with date range, single-period summary, cross-period summary, cancel/tombstone |
| Dashboard | 12 | Key metrics display, empty state (new consorcio), loading skeletons, error state, period selector interaction |
| Layout/Navigation | 12 | Sidebar navigation, consorcio selector, role-based menu visibility, breadcrumbs, responsive behavior |
| Cross-cutting (API) | 12 | MSW failure modes (network error, 422, 409), idempotency retry, JWT expiry mid-session, concurrent tab detection |
| Optimistic updates | 5 | Approve/reject shows new status immediately, rolls back on 422; payment creation shows pending then confirms; annul action renders strikethrough then confirms |
| Confirmation dialogs | 6 | Annul invoice → confirm dialog → proceed / cancel → no side effect; delete draft → confirm → removed from list; irreversible actions gated behind second click |
| Form draft persistence | 5 | Partially filled form survives tab switch, browser back navigation, and accidental close; restored draft shows stale-data warning after 5 minutes |
| Keyboard & a11y | 5 | Tab order through form fields matches visual layout; screen reader announces validation error on submit; modal focus trap prevents background interaction; dropdown selection via arrow keys |
| Print & export | 4 | Liquidacion print layout matches issued PDF styling; certificado download triggers correct filename; payment receipt print shows consorcio header |
| Offline resilience | 4 | Network fails during payment submit → form state preserved, retry restores in-progress data; 503 response → "service unavailable" toast with retry timer |
| Deep linking | 3 | Bookmarked `/expenses/123` resolves correctly post-login redirect; `/liquidaciones/456/pdf` triggers download after auth; invalid ID in URL → 404 page with navigation suggestion |
| Theme & locale | 3 | Dark mode toggle persists across sessions; date format switches on locale change (es-AR: 25/06/2026); currency display respects locale (ARS: $ 1.500,00) |
| **Total** | **~220** | |

This is approximately **36% of the backend's 614 scenarios** (220 vs 614). The
reduction is driven by:

1. **No domain math** — prorrateo, interest compounding, cascade, and rounding
   are all backend-tested. Frontend only asserts correct display of pre-computed values.
2. **No legal edge cases** — ARCA invoice validation (CAE format, sequential
   numbering, IVA rate constraints) is the backend's domain.
3. **Shorter scenarios** — UI scenarios test fewer steps per flow than API
   domain-invariant scenarios.
4. **Offset by frontend-specific concerns** — optimistic UI, a11y, offline
   resilience, form persistence, and confirmation dialogs add scenarios the
   backend never tests.

### 4.4 Scenario-to-Test Mapping

```
BDD Scenario (.feature)
    │
    ├─→ Playwright E2E test (validates the full user flow)
    │       Given a logged-in admin on the expenses page
    │       When they create an EXTRAORDINARIA expense
    │       Then the expense appears with "PENDING" status
    │
    ├─→ Integration test (validates API client + MSW interaction)
    │       useCreateExpense().mutateAsync(payload) → onSuccess → cache invalidation
    │
    └─→ Component test (validates isolated component behavior)
            <ExpenseForm /> with type=EXTRAORDINARIA → renders approval notice
```

The `quality-engineer` owns all three layers, writing scenarios first (in
Gherkin), then implementing the step definitions across the appropriate test
layer.

---

## 5. Governance Rules

The following rules are established in this methodology document and will be
**formalized** by the `constitution-architect` agent in Phase 0 with concrete
enforcement mechanisms (`.opencode/agents/*.md` instructions, Biome config,
CI/CD checks, and HITL gate scripts).

### 5.1 Versioning

CESAR's CONSTITUTION §8 already defines a HITL-only model for git mutations.
The frontend adopts the same model with one refinement — a third tier for
branch operations:

#### Tier 1 — AI-Allowed (informative, no side effects)

Agents may freely execute these without HITL approval:

```
git status, git log, git diff, git show, git branch, git branch -a,
git blame, git stash list, git tag -l, git remote -v
```

#### Tier 2 — Ask-Before (low-risk, reversible)

Agents must **ask the HITL before** executing. Once approved, the agent may
proceed:

```
git branch {name}, git branch -m {new}, git branch -d {name},
git stash, git stash pop, git stash drop, git checkout {file}
```

#### Tier 3 — HITL-Only (history-modifying or remote-affecting)

Only a human may execute. Agents must never invoke these:

```
git commit, git reset, git revert, git rebase, git merge, git cherry-pick,
git push, git fetch, git pull, git remote add/remove, git tag -a,
git checkout {branch} (context switch)
```

**Enforcement (to be formalized by constitution-architect):**
Agent `.md` files declare blocked commands in their skill definitions.
`.opencode/opencode.json` permission rules enforce Tier 3 at the tool level.

### 5.2 Quality, Clean Code & Declarative Programming

Every artifact (code, test, spec) must conform to quality gates. The
`constitution-architect` shall formalize these into enforceable rules. This
section establishes the expectations.

#### TypeScript Quality

- **Strict mode** enabled in `tsconfig.json` — `strict: true`, `noUncheckedIndexedAccess: true`.
- **No `any`** — eslint rule `@typescript-eslint/no-explicit-any: error`.
- **No `as` casts** — prefer type guards and Zod validation over type assertions.
- **Discriminated unions** for component state (e.g., `type State = { status: 'loading' } | { status: 'ready'; data: T } | { status: 'error'; error: E }`).
- **`Readonly<>`** on all props, hook returns, and API response types.
- **All API data validated at the boundary** — `Zod` schema on every API response before it enters component state.

#### React Declarative Patterns

- **Hooks over classes** — no class components.
- **Derived state, not duplicated state** — prefer `useMemo` over `useState` + `useEffect`.
- **TanStack Query is the single source of server state** — no manual `useState`/`useEffect` for API data fetching or caching.
- **Composition over inheritance** — no component inheritance; prefer render props, slots, and compound components.
- **Colocate styles with components** — Tailwind CSS inline; no separate CSS files per component.
- **Immutability** — `useState` updaters always return new objects; no mutation of props or context.

#### Code Organization

- **Feature-based directory structure** — one directory per bounded context
  (`src/features/consorcios/`, `src/features/expenses/`, etc.).
- **No cross-feature imports except through `shared/` and `lib/`** — prevents
  sinkhole components and accidental coupling.
- **Barrel exports** — each feature directory exports a public API via
  `index.ts`; internal files are not imported from outside the feature.
- **Linting** — Biome with recommended rules; no ESLint or Prettier (Biome replaces both).

#### Architectural Patterns

- **Feature-Sliced Design (simplified)** — each feature follows:
  `api/` (query hooks) → `components/` (UI) → `pages/` (routing entry).
- **Custom hooks for cross-cutting concerns** — `useConsorcio()`, `usePeriod()`,
  `useAuth()`, `useIdempotency()` encapsulate shared logic.
- **Error boundaries** — one per route segment for graceful degradation.

### 5.3 Open Source First

Adopt CESAR's Principle 7 (`P7 — Open Source First`) for the npm ecosystem:

- **Every dependency must have an OSS-compatible license** — MIT, Apache 2.0,
  BSD, ISC are pre-approved. GPL/AGPL copyleft licenses require explicit review.
- **Proprietary gateways** (payment processors, ARCA web services) must be
  behind swappable adapters (same pattern as CESAR backend).
- **Tooling** — `license-report` or `license-checker` in CI pipeline to enforce.
- **The cesar-web project itself** — same dual licensing as CESAR:
  AGPL-3.0-or-later for the application, MIT for shared libraries/schemas.

**Enforcement (to be formalized by constitution-architect):**
- Pre-commit or CI check: `npx license-checker --onlyAllow "MIT;Apache-2.0;BSD;ISC;CC0" --excludePackages "..."`.
- Agent instructions: `contract-guardian` validates license of any new
  dependency proposed by `feature-developer`.

### 5.4 Anti-Pattern Prevention

The five anti-patterns below are inherent risks of Contract-First development
and must be prevented by design, not by vigilance.

#### AP-1: Specification Drift

**Definition:** Frontend code that diverges from the OpenAPI contract — using
fields or types not defined in the spec, or assuming undocumented behavior.

**Prevention:** Compile-time enforcement by the orval pipeline.

```
cesar/api/openapi.yaml
    ↓ orval generate
src/api/generated/cesar.ts (typed hooks)
    ↓ imports
src/features/**/*.tsx
    ↓ TypeScript compilation
BUILD FAILS if any code references removed/renamed/mistyped fields
```

**No runtime check can beat this.** The `contract-guardian` agent owns the
pipeline. When the backend delivers a spec update, `contract-guardian` runs
`orval`, and TypeScript enumerates every call site affected. The
`feature-developer` fixes them before the build passes.

#### AP-2: Semantic Drift

**Definition:** Fields are used correctly by type (the code compiles) but
interpreted incorrectly in the UI. Examples:
- `amount` is in integer cents; UI divides by 100 but on the wrong field.
- `status: "ISSUED"` in API is displayed as "Aprobado" instead of "Emitido".
- `iva_condition` enum value "RI" triggers the wrong form field.

**Prevention:** BDD scenarios + Zod validation + MSW tests.

- **BDD scenarios** author explicit expectations: "Given the API returns `amount:
  150000`, then the UI displays `$1,500.00`."
- **Zod schemas** with `.describe()` annotations document the semantic meaning
  of each field.
- **MSW handlers** return data with specific decimal/status combinations that
  BDD scenarios assert against.

#### AP-3: Moving the Goalposts

**Definition:** Requirements change mid-implementation without updating the
OpenAPI spec first. The UI team builds against Feature X, then is told Feature X
needs fields A, B, C that don't exist in the API.

**Prevention:** Spec-update-first rule.

```
Requirement change request
        │
        ▼
Is it a new API field/endpoint?
        │
   YES ─┼─→ Backend team updates OpenAPI spec first
        │   → Frontend team re-runs orval
        │   → TypeScript surfaces affected components
        │   → Feature-developer adapts
        │
   NO  ─┼─→ UI-only change
        │   → Proceed with component updates
        │
```

The `constitution-architect` shall encode this as a HITL gate: no frontend task
shall start for a feature that requires API changes unless the OpenAPI spec
already reflects those changes.

#### AP-4: Shadow State

**Definition:** Duplicating server state in local `useState`/`useEffect` when
TanStack Query already manages it. Leads to stale UI, double-fetching, and
inconsistent views.

**Prevention:** Architectural rule enforced by code review and linting.

- TanStack Query is the **single source of truth** for all server data.
- Components read from `useQuery` and write via `useMutation`.
- No `useState` initialized from `useQuery` data without an explicit derived
  state justification (e.g., form draft editing).
- No `useEffect` that calls `setState` with API response data — that's what
  `useQuery` returns directly.

**Enforcement:** ESLint/Biome rule that flags `useEffect` containing `useQuery`
calls or `setState` from fetched data (to be configured by `constitution-architect`).

#### AP-5: Sinkhole Components

**Definition:** Components that import from sibling features, creating hidden
coupling that defeats the feature-based architecture.

**Prevention:** Import boundary enforcement.

```
src/features/
├── consorcios/    ← may import from components/shared/ and lib/
├── expenses/      ← may import from components/shared/ and lib/
│   └── may NOT import from consorcios/ or invoices/
├── shared/        ← no feature imports
└── lib/           ← no feature imports; no UI imports
```

- Shared UI primitives live in `components/shared/` (MoneyDisplay, StatusBadge,
  CUITInput, PeriodPicker).
- Business logic free of UI lives in `lib/` (validation functions, formatters,
  constants).
- Cross-feature composition happens at the **router level** (not via direct
  component import).

**Enforcement:** ESLint import/no-restricted-paths rule (to be configured by
`constitution-architect`).

### 5.5 Summary: Methodology vs Constitution

| Rule Area | In Methodology (this doc) | To Be Formalized By constitution-architect |
|---|---|---|
| **Versioning** | Three-tier model defined | Agent `.md` instruction blocks, `.opencode/opencode.json` permission rules |
| **Quality & Clean Code** | Expectations listed per paradigm | Biome config, eslint rules (no-any, no-as, no-shadow-state), tsconfig strictness |
| **Open Source First** | Principle stated | Specific allowed/denied license list, CI check script, agent instruction |
| **Anti-patterns** | AP-1 through AP-5 defined with prevention mechanisms | Enforcement tooling (lint rules, CI gates, HITL gate scripts) |
| **BDD framework** | Estimate and layer mapping | Actual `.feature` files, step definitions, Playwright config |
| **Agent definitions** | 4-agent roster with roles | Individual agent `.md` files in `.opencode/agents/`, skill definitions |

The methodology document is the **analytical pre-constitution phase**. It says
*what* the rules are and *why*. The `constitution-architect` in Phase 0 says
*how* they are enforced and produces the concrete configuration artifacts.

---

## 6. Quality Gates

These gates apply to all development phases. Each gate must pass before a task
or phase is considered complete. The `constitution-architect` shall formalize
them into CI/CD checks.

| Gate | Criteria | Owner |
|---|---|---|
| **G1 — Type Safety** | `tsc --noEmit` passes with `strict: true`. Zero `as` casts | contract-guardian + feature-developer |
| **G2 — Contract Compliance** | All MSW handlers match OpenAPI schema. All API hooks are generated by orval (no hand-written fetch calls). | contract-guardian |
| **G3 — BDD Coverage** | Every module has `.feature` files covering: CRUD happy path, validation sad paths, empty/loading/error states, role-based access. | quality-engineer |
| **G4 — Component Tests** | Every shared component has a Vitest test. Every form has validation tests. | quality-engineer |
| **G5 — E2E Coverage** | Critical user flows (login → create consorcio → create expense → generate liquidacion → register payment) pass end-to-end. | quality-engineer |
| **G6 — Accessibility** | All interactive elements pass axe-core checks (WCAG 2.1 AA). Keyboard navigation works for all forms. | quality-engineer |
| **G7 — Open Source** | Dependency tree contains zero proprietary or copyleft licenses not explicitly approved. | contract-guardian (automated CI check) |
| **G8 — No Anti-Patterns** | Code review confirms: no shadow state, no cross-feature imports, no hand-written API calls outside generated hooks. | feature-developer + code review |

---

## 7. Phase 0 Roadmap — What the Constitution-Architect Produces

The methodology document ends here. Phase 0 (executed by the
`constitution-architect` agent) will produce:

1. **`docs/CONSTITUTION.md`** (frontend) — Formal governance document with:
   - Adapted principles from this methodology's §2.3
   - Formalized rules from §5.1–5.4
   - Architecture Decision Log (ADRs) for frontend-specific decisions
   - Quality gates from §6 with enforcement scripts
   - Git rules with tool-level permissions

2. **`.opencode/agents/*.md`** — 4 agent definition files with:
   - Explicit allowed/blocked commands per §5.1 tiers
   - Input/output contracts
   - Quality checklist per agent

3. **`.opencode/opencode.json`** — Project-level permission rules:
   - Block Tier 3 git commands for all agents
   - Allow Tier 1 commands unconditionally
   - Require confirmation for Tier 2 commands

4. **Linting & CI configuration:**
   - Biome config with anti-pattern rules (§5.4)
   - `tsconfig.json` strictness baseline
   - License checker CI step (§5.3)
   - Pre-commit hook setup

5. **ADR catalog** — Initial architecture decisions:
   - ADR-1: OpenAPI as compile-time contract boundary
   - ADR-2: Feature-based directory structure with import restrictions
   - ADR-3: JWT storage strategy (memory + HttpOnly cookie fallback)
   - ADR-4: Idempotency key generation client-side
   - ADR-5: Spanish-first UI with i18n support
   - ADR-6: TanStack Query as single source of server state

---

## 8. Summary

| Dimension | CESAR Backend | CESAR Web Frontend |
|---|---|---|
| SDD variant | Spec-Driven Development (author spec first) | Contract-First Frontend Development (validate against external contract) |
| Agents | 8 defined / 6 active | 4 defined / 4 active |
| Tasks (estimated) | 80 | ~64 |
| BDD scenarios | 29 feature files / ~614 scenarios | ~220 scenarios across ~18–21 feature files |
| Contract enforcement | schemars (runtime) + integration tests | TypeScript compile + orval regeneration |
| License oversight | Cargo.toml review | CI license-checker on npm dependencies |
| Git mutations | HITL-only | HITL-only (Tier 3) + ask-before (Tier 2) |
| Anti-pattern prevention | Constitution P1-P8 | AP-1 to AP-5 with compile-time and lint enforcement |

**Next step:** Review and approve this methodology document, then proceed to
Phase 0 where the `constitution-architect` agent formalizes it into
`docs/CONSTITUTION.md` and associated configuration artifacts.

---

*End of SDD Methodology Document.*
