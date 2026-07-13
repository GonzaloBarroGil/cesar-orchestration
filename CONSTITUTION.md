# CONSTITUTION — CESAR Ecosystem

> **CESAR Ecosystem**: Three-project orchestration layer for Argentine condominium management
>
> Ecosystem-level governance. Coordinates cesar (backend), cesar-web (frontend), and cesar-orchestration/ (this repo).
>
> **Status: DRAFT — awaiting HITL validation**
> **Date: 2026-07-13**

---

## 1. Ecosystem Identity

The CESAR ecosystem consists of three independent projects connected by a shared API contract:

| Project | Role | Tech | Status |
|---|---|---|---|
| **cesar** | REST API backend | Rust/Axum, PostgreSQL 16, Redis, NATS | Phase 1 complete, 6 modules, 1,083 tests |
| **cesar-web** | Web SPA frontend | React/Vite, TypeScript, TanStack Query | Wave 2 in progress, 3/9 features built |
| **cesar-orchestration** | Ecosystem governance + orchestration | Documentation, scripts, tracking | This document |

### Repository Layout

```
~/Repos/
├── cesar/                    # Backend — owns api/openapi.yaml
│   ├── docs/CONSTITUTION.md  # Project-level governance (HOW cesar is built)
│   ├── api/openapi.yaml      # THE ecosystem contract (61 endpoints)
│   └── .opencode/agents/     # 8 backend subagents
├── cesar-web/                # Frontend — consumes the contract
│   ├── docs/CONSTITUTION.md  # Project-level governance (HOW cesar-web is built)
│   └── .opencode/agents/     # 4 frontend subagents
└── cesar-orchestration/              # Ecosystem governance (THIS repo)
    ├── CONSTITUTION.md       # THIS DOCUMENT — ecosystem authority
    ├── cesar-web-feasibility.md       # Frozen: feasibility analysis
    ├── cesar-web-sdd-methodology.md   # Frozen: CFFD methodology
    ├── cesar-openapi-spec-issues.md   # Cross-project issue: OAS bugs
    ├── cesar-backend-concerns.md      # Cross-project issue: backend blockers
    └── .opencode/agents/     # Orchestrator agent
```

---

## 2. Ecosystem Principles

### EC-P1 — Contract-Centric

The OpenAPI 3.1 specification (`cesar/api/openapi.yaml`) is the ecosystem-level contract. It is the single source of truth for all cross-project integration.

- **Owned** by cesar (it defines the backend's public API)
- **Consumed** by cesar-web (via orval code generation)
- **Validated** by cesar-orchestration scripts

Changes to the contract follow this sequence:

```
cesar modifies api/openapi.yaml
    → compatibility validation
    → if compatible: cesar-web adapts (orval regeneration)
    → if breaking: ecosystem HITL gate required before either project proceeds
```

### EC-P2 — Backend-First Sequencing

cesar-web depends on cesar's API. Therefore:

- **Spec work** can proceed in parallel across both projects (both can write SPECs independently)
- **Integration** requires backend readiness: DB migrations exist, repository implementations are not stubs, endpoints respond correctly
- Before a cesar-web feature reaches implementation tasks that hit real API endpoints, the corresponding cesar endpoints must be integration-tested and deployed
- MSW mocks are acceptable for development; real API integration is gated by backend readiness

### EC-P3 — Independent Deployability

Each project deploys independently:

- cesar runs its own docker-compose (PostgreSQL 16, Redis, NATS)
- cesar-web is a static SPA served independently
- Neither project requires the other at build time (orval works from spec file, not running API)
- cesar-orchestration provides convenience tooling (combined docker-compose) but neither project depends on it

### EC-P4 — Cross-Project Traceability

Every ecosystem-level task is traceable end-to-end:

- Which cesar change triggered which spec update
- Which spec update triggered which cesar-web tasks
- Which cesar-web BDD scenario maps to which cesar BDD scenario

### EC-P5 — Orchestrator Delegation

The ecosystem orchestrator plans and sequences cross-project work. It NEVER writes implementation code. It delegates all implementation to project-level subagents:

- cesar implementation → `rust-developer` and `test-engineer` subagents
- cesar-web implementation → `feature-developer` and `quality-engineer` subagents
- Exploration/research → `explore` subagent

---

## 3. The API Contract

### Ownership and Consumption

| Role | Project | What |
|---|---|---|
| **Owner** | cesar | Defines the backend's public API in `api/openapi.yaml` |
| **Consumer** | cesar-web | Generates TypeScript types + TanStack Query hooks via orval |
| **Validator** | cesar-orchestration | Cross-project compatibility checks and issue tracking |

### Contract Change Rules

| Change Type | Process |
|---|---|
| **Additive** (new endpoint, new optional field, new response code) | cesar adds → cesar-web may use when ready. No gate required. |
| **Breaking** (field removed, type changed, endpoint renamed, required field added) | Ecosystem HITL gate required. Both projects update in coordinated fashion. Contract version bumped. |
| **Spec fix** (invalid $ref, missing schema, incorrect response shape) | Tracked in `cesar-openapi-spec-issues.md` → fixed in cesar → verified in cesar-web |

---

## 4. Sequencing Rules

### Dependency Chain

```
cesar DB migrations complete
    → cesar repository implementations complete
        → cesar integration tests pass
            → cesar-web can integrate against real API
```

### What CAN proceed in parallel

- cesar spec work + cesar-web spec work (independent)
- cesar-web MSW-based development + cesar backend development (independent until integration)
- Both projects' CI/CD setup (independent)

### What REQUIRES sequencing

- cesar-web real-API integration tasks (depend on cesar endpoint readiness)
- Contract changes (must flow cesar → validation → cesar-web)
- Cross-project BDD acceptance (both sides must be green for the same bounded context)

---

## 5. Cross-Project Workflow

```
┌─────────────────────────────────────────────────────┐
│              ECOSYSTEM ORCHESTRATOR                  │
│                                                       │
│  1. Read state from both projects                     │
│     - cesar: git log, CONSTITUTION, PLAN, test results│
│     - cesar-web: git log, CONSTITUTION, PLAN, tests   │
│                                                       │
│  2. Identify blockers and dependencies                │
│     - Check cesar-backend-concerns.md                  │
│     - Identify which cesar endpoints block cesar-web  │
│                                                       │
│  3. Plan next task batch                              │
│     - Prioritize cesar completeness (migrations, repos│
│     - Schedule cesar-web features that are unblocked  │
│                                                       │
│  4. Delegate to project subagents                     │
│     - Task(rust-developer, "implement X migration")   │
│     - Task(feature-developer, "build Y feature")      │
│                                                       │
│  5. Verify cross-project gates                        │
│     - Contract compatibility verified?                │
│     - BDD traceability updated?                       │
│     - Integration tests green on both sides?          │
│                                                       │
│  6. HITL validation → advance                         │
└─────────────────────────────────────────────────────┘
```

---

## 6. Ecosystem Quality Gates

| Gate | Criteria |
|---|---|
| **EG1 — Contract Compatibility** | orval generation succeeds against current `cesar/api/openapi.yaml`. `tsc --noEmit` passes in cesar-web with generated types. |
| **EG2 — Backend Readiness** | For each endpoint cesar-web needs: DB migration exists, repository implemented (not stub), service tests pass, integration tested against real database. |
| **EG3 — Cross-Project BDD** | Corresponding cesar .feature scenarios pass AND cesar-web .feature scenarios pass for the same bounded context. |
| **EG4 — Independent Deployability** | Each project can be deployed independently. Combined docker-compose is optional convenience, not a dependency. |
| **EG5 — Traceability** | Every ecosystem task is documented: what changed in cesar → what changed in cesar-web → compatibility verified. |

---

## 7. Ecosystem Architecture Decision Log

### EAD-001: Three-Project Ecosystem with Orchestration Layer

- **Decision**: Establish `cesar-orchestration/` as the ecosystem governance hub with a dedicated orchestrator agent. Project constitutions (cesar, cesar-web) are amended to acknowledge ecosystem authority for cross-project matters (sequencing, contract coordination, cross-project quality gates) while retaining full authority for internal matters (architecture, technology, internal quality gates).
- **Rationale**: Independent projects connected by a shared API contract require explicit coordination. Without an ecosystem layer, contract changes in cesar silently break cesar-web, backend readiness gaps block frontend integration without visibility, and sequencing decisions lack a neutral authority. The orchestrator agent reads state from both projects, plans cross-project task sequences, and delegates implementation to project-level subagents.
- **Date**: 2026-07-13

---

## 8. Amendment Process

1. Proposed amendment documented as an Ecosystem Architecture Decision (EAD)
2. Reviewed by HITL
3. If approved, orchestrator updates this CONSTITUTION
4. Project constitutions updated for any cascading changes
5. Affected cross-project artifacts updated

### Authority Scope

This document supersedes both project constitutions on matters of:

- **Sequencing**: which project does what when
- **Contract coordination**: how the OAS spec is shared, validated, and kept compatible
- **Cross-project quality gates**: integration readiness, BDD traceability, contract compatibility

This document does NOT supersede project constitutions on matters of:

- **Internal architecture**: how Rust code is structured, how React components are organized
- **Internal quality gates**: test coverage, type safety within a single project
- **Technology choices**: framework selection, library versions, tooling within a project

---

## 9. Current Ecosystem State (2026-07-13)

### cesar — Backend

- **Constitution**: APPROVED (2026-06-21)
- **Phase 1**: COMPLETE — all 6 modules (consorcio, expenses, invoices, debt, payments, incomes-outputs)
- **Tests**: 1,083 passing (487 domain + 200 service + 350 integration + 43 property-based)
- **Last commit**: `fix(cesar): [T0.9,W7] make debt services optional, fix route syntax, wire app for startup`
- **Working tree**: clean
- **Known gaps** (from `cesar-backend-concerns.md`):
  - 2/20 DB migrations exist
  - Many repository implementations are `todo!()` stubs
  - Missing auth endpoints in OpenAPI spec
  - Missing `GET /consorcios` list endpoint
  - No dashboard aggregation endpoints

### cesar-web — Frontend

- **Constitution**: APPROVED (2026-06-25)
- **Features**: 3/9 implemented (auth, consorcios, layout)
- **PLAN**: 70 tasks, 7 waves — currently at Wave 2 (Consorcios)
- **Last commit**: `test(consorcio): [TEST,T1.5,RED] add passing tests for consorcio components`
- **Working tree**: unstaged changes in consorcio components
- **Next wave**: Wave 3 — Expenses ∥ Invoices (16 tasks)

### Blockers (from `cesar-backend-concerns.md`)

| # | Blocker | Blocks |
|---|---|---|
| 1 | Missing auth endpoints in OAS | cesar-web W1 auth integration against real API |
| 2 | Missing `GET /consorcios` endpoint | cesar-web W2 ConsorcioSelector (MSW workaround in place) |
| 3 | Only 2 DB migrations | Real backend cannot run full schema |
| 4 | Repository stubs (todo!()) | Endpoints return 501 / unimplemented |

---

## 10. Subagents

The ecosystem has one subagent:

| Agent | Role | Input | Output |
|---|---|---|---|
| **orchestrator** | Cross-project planning and delegation | State from both projects + this CONSTITUTION | Task plans delegated to project subagents |

The orchestrator is defined in `.opencode/agents/orchestrator.md`. It uses project-level subagents for all implementation work:

| Project | Available Subagents |
|---|---|
| cesar | constitution-architect, spec-author, domain-modeler, bdd-scenario-writer, api-contract-designer, rust-developer, test-engineer, legal-compliance-checker |
| cesar-web | constitution-architect, contract-guardian, feature-developer, quality-engineer |

The orchestrator never writes implementation code. It delegates via the Task tool.

---

*End of CONSTITUTION.*
