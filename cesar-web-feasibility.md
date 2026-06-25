# CESAR Back-Office Web App — Feasibility Analysis

> **Date:** 2026-06-25  
> **Status:** Draft — first approach  
> **Target:** React + TypeScript SPA consuming CESAR REST API  
> **Audience:** Consorcio administrators (ADMIN) + owners/tenants (propietarios)

---

## 1. Project Context

**CESAR** (Consorcio Expense & Service Administration Runtime) is a Rust/Axum
backend for Argentine condominium administration, implementing:

- CCC (Código Civil y Comercial) Articles 2037–2072 — horizontal property
- CABA Ley 941 — Buenos Aires city regulation
- ARCA RG 4290/2018 — electronic invoicing

The backend is a modular monolith with DDD bounded contexts, 1,083 tests, and
an append-only SHA-256 audit hash chain for all financial data.

### 1.1 Backend Tech Stack

| Component       | Choice                              |
|-----------------|-------------------------------------|
| Language        | Rust (edition 2021, requires 1.85+) |
| Web framework   | Axum 0.8 (Tokio async)              |
| Database        | PostgreSQL 16 (SQLx 0.8)            |
| Cache           | Redis 7 (idempotency, token blacklist) |
| Messaging       | NATS 2.10 (inter-context events)    |
| Auth            | JWT Bearer (ADMIN / ACCOUNTANT)     |
| API spec        | OpenAPI 3.1 (61 endpoints)          |
| Observability   | OpenTelemetry (tracing, metrics)    |
| Tests           | 1,083 tests (unit, integration, property-based) |
| License         | AGPL-3.0-or-later                   |

---

## 2. API Surface

The CESAR OpenAPI 3.1 specification defines **61 endpoints** across **7 modules**
with fully specified request/response schemas, error codes, and business rules.

### 2.1 Endpoint Inventory

| Module         | Endpoints | Domain Objects                                              | Roles        |
|----------------|-----------|-------------------------------------------------------------|--------------|
| Auth           | 3         | Login, /me, health check                                    | Public + Auth |
| Consorcio      | 10        | Consorcio CRUD, UFs, Propietarios, Fiscal Periods           | ADMIN        |
| Expenses       | 15        | Expense CRUD, approval workflow, concepts, categories, providers | ADMIN   |
| Invoices       | 12        | Factura A/B/C, Notas de Crédito/Débito, Recibos             | ADMIN / ACCOUNTANT |
| Debt           | 9         | Liquidaciones (single + batch), Certificados de Deuda, signing | ADMIN   |
| Payments       | 6         | Payment CRUD, reversal, unit balance, payment history       | ADMIN        |
| Incomes/Outputs| 6         | Cash-flow entries, per-period and cross-period summaries    | ADMIN        |

### 2.2 Auth Model

- **JWT Bearer tokens** — 24-hour TTL, HS256 with secret key.
- **Two roles:** `ADMIN` (full access) and `ACCOUNTANT` (invoices only).
- JWT payload: `sub` (user UUID), `username`, `role`, `consorcio_id` (optional), `exp`, `iat`, `jti`.
- **Idempotency-Key** header (UUID v7) required on all mutation endpoints.
- Standard error envelope: `{ error: { code, message, details?: [] } }`.

### 2.3 Entry Points

```
POST   /api/v1/auth/login          # Login → JWT
GET    /api/v1/auth/me              # Current user
GET    /api/v1/health               # Health (PG + Redis + NATS)
```

#### Consorcio (`/api/v1/consorcios`)
```
POST   /consorcios                  # Create consorcio (ADMIN)
GET    /consorcios/{id}             # Get consorcio
PUT    /consorcios/{id}             # Update consorcio (ADMIN)
PUT    /consorcios/{id}/ufs         # Configure UFs (ADMIN)
POST   /consorcios/{id}/propietarios  # Register propietario (ADMIN)
GET    /consorcios/{id}/propietarios  # List propietarios
GET    /consorcios/{id}/propietarios/{tax_id}  # Find by tax ID
POST   /consorcios/{id}/periodos    # Open period (ADMIN)
PUT    /consorcios/{id}/periodos/{pid}/close   # Close period (ADMIN)
PUT    /consorcios/{id}/periodos/{pid}/archive # Archive period (ADMIN)
```

#### Expenses (`/api/v1`)
```
POST   /expenses                    # Create expense (ADMIN)
GET    /expenses                    # List expenses (filters: consorcio, period, type, status)
GET    /expenses/{id}/versions      # Version history (hash chain audit)
PUT    /expenses/{id}/tombstone     # Soft-delete (ADMIN)
PUT    /expenses/{id}/approve       # Approve EXTRAORDINARIA (ADMIN)
PUT    /expenses/{id}/reject        # Reject EXTRAORDINARIA (ADMIN)
POST   /expenses/concepts           # Create concept
GET    /expenses/concepts           # List concepts
GET    /expenses/concepts/{group}   # List by group
POST   /expenses/categories         # Create category
GET    /expenses/categories         # List categories
POST   /expenses/providers          # Register provider
GET    /expenses/providers          # List providers
PUT    /expenses/providers/{id}     # Update provider
PUT    /expenses/providers/{id}/tombstone  # Tombstone provider (ADMIN)
```

#### Invoices (`/api/v1`)
```
POST   /invoices/factura-a          # Create Factura A (ADMIN/ACCOUNTANT)
POST   /invoices/factura-b          # Create Factura B
POST   /invoices/factura-c          # Create Factura C
GET    /invoices/{id}               # Get invoice
GET    /invoices/cae/{cae}          # Get by CAE
PUT    /invoices/{id}/anular        # Annul invoice (ADMIN)
POST   /invoices/{id}/nota-credito  # Create NC
PUT    /invoices/nota-credito/{id}/anular  # Annul NC
POST   /invoices/{id}/nota-debito   # Create ND
PUT    /invoices/nota-debito/{id}/anular   # Annul ND
POST   /recibos                     # Create Recibo
GET    /recibos/{id}                # Get Recibo
PUT    /recibos/{id}/anular         # Annul Recibo
```

#### Debt (`/api/v1/debt`)
```
POST   /liquidaciones               # Generate (single unit)
POST   /liquidaciones/batch         # Generate batch (all units)
GET    /liquidaciones               # List (filters: consorcio, period, unit, status)
GET    /liquidaciones/{id}          # Get detail
PUT    /liquidaciones/{id}/issue    # Issue (DRAFT → ISSUED)
GET    /liquidaciones/{id}/pdf      # Download PDF (**returns 501**)
POST   /certificados               # Generate certificado
GET    /certificados               # List (by unit_id)
GET    /certificados/{id}          # Get detail
PUT    /certificados/{id}/sign     # Sign (makes executive title)
```

#### Payments (`/api/v1`)
```
POST   /payments                    # Create payment (+ auto Recibo B/C)
GET    /payments                    # List (filters + pagination)
GET    /payments/{id}               # Get detail
POST   /payments/{id}/reverse       # Reverse (creates compensating entry)
GET    /payments/unit/{unit_id}/balance    # Unit balance summary
GET    /payments/unit/{unit_id}/history    # Payment history (descending)
```

#### Incomes/Outputs (`/api/v1`)
```
POST   /incomes-outputs             # Create entry
GET    /incomes-outputs             # List (filters + date range)
GET    /incomes-outputs/{id}        # Get detail
PUT    /incomes-outputs/{id}        # Update (append-only versioning)
PUT    /incomes-outputs/{id}/cancel # Cancel (tombstone)
GET    /incomes-outputs/summary/{period_code}   # Single-period summary
GET    /incomes-outputs/summary                  # Cross-period summary
```

---

## 3. Backend Readiness Assessment

### 3.1 What Is Implemented

| Layer              | Status                                                    |
|--------------------|-----------------------------------------------------------|
| Domain models      | Fully defined — value objects, aggregates, invariants     |
| Service layer      | Partially implemented; some modules have stubs            |
| HTTP handlers      | Routers defined; most handlers exist                      |
| API spec           | Complete for Phase 1 (61 endpoints, full schemas)         |
| Tests              | 1,083 passing (487 domain + ~200 service + ~350 integration + 43 proptest) |
| CI/CD              | GitHub Actions (check, test, clippy, release build)       |
| Infrastructure     | docker-compose (PG + Redis + NATS), CORS enabled          |

### 3.2 What Is Incomplete

| Gap                        | Impact on Frontend                        |
|----------------------------|-------------------------------------------|
| Only 2 DB migrations exist (users, consorcios) | Remaining ~18+ tables not migrated     |
| Many repository impls are `todo!()` stubs      | Backend returns mock/stub data          |
| PDF liquidacion returns 501                   | "Download PDF" not functional           |
| No `GET /consorcios` list endpoint            | Cannot show consorcio selector           |
| No dashboard aggregation endpoints            | KPIs must be computed client-side        |
| No user management endpoints                  | Cannot manage users from UI              |
| Phase 2 modules deferred (salary, investment, legal) | Future scope                  |

### 3.3 Development Strategy

**Build against the OpenAPI spec with a mock server.** This allows full-stack
development to proceed in parallel:

1. Frontend consumes typed API client generated from OpenAPI.
2. MSW (Mock Service Worker) intercepts requests and returns realistic data.
3. When CESAR implements a module, switch from MSW to real API by changing the
   base URL.
4. The OpenAPI spec acts as the single source of truth for both teams.

---

## 4. Recommended Frontend Architecture

### 4.1 Project Structure

```
cesar-web/
├── src/
│   ├── api/
│   │   ├── client.ts              # Axios/fetch instance with JWT interceptor
│   │   ├── idempotency.ts         # Idempotency-Key header generator
│   │   └── generated/             # Auto-generated from OpenAPI (orval)
│   │       ├── cesar.ts           # All typed endpoints + TanStack Query hooks
│   │       └── models/            # TypeScript types for all schemas
│   ├── components/
│   │   ├── ui/                    # shadcn/ui primitives
│   │   ├── layout/                # AppShell, Sidebar, Header, ConsorcioSelector
│   │   └── shared/                # MoneyDisplay, StatusBadge, CUITInput, etc.
│   ├── features/                  # One directory per bounded context
│   │   ├── auth/                  # Login page, AuthProvider, role guards
│   │   ├── consorcios/            # List, detail, UFs, propietarios, periods
│   │   ├── expenses/              # CRUD, approval queue, providers, concepts
│   │   ├── invoices/              # Factura A/B/C forms, annul flow
│   │   ├── debt/                  # Liquidacion generation, certificados
│   │   ├── payments/              # Registration, history, reversals
│   │   ├── incomes/               # Cash-flow entries, summaries
│   │   └── dashboard/             # Key metrics per consorcio
│   ├── hooks/                     # Shared hooks (useConsorcio, usePeriod)
│   ├── lib/                       # Utilities
│   │   ├── format.ts              # formatMoney, formatPercentage, formatDate
│   │   ├── validators.ts          # CUIT checksum, YYYYMM, email validation
│   │   └── constants.ts           # Enums, concept groups, status mappings
│   ├── mocks/                     # MSW handlers
│   │   ├── handlers/              # Per-module mock handlers
│   │   ├── fixtures/              # Realistic test data
│   │   └── server.ts              # MSW server setup
│   ├── router/                    # React Router config
│   │   ├── index.tsx              # Route tree
│   │   └── guards.tsx             # Role-based route guards
│   ├── App.tsx
│   └── main.tsx
├── public/
├── package.json
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── orval.config.ts                # OpenAPI code generation config
```

### 4.2 Technology Choices

| Concern              | Choice                    | Rationale                                             |
|----------------------|---------------------------|-------------------------------------------------------|
| Build tool           | Vite 6                    | Fast HMR, standard for React SPAs                     |
| Language             | TypeScript 5.5+           | Matches Rust's type safety philosophy                 |
| UI framework         | React 19                  | As chosen                                              |
| API client gen       | **orval**                 | Generates typed TanStack Query hooks from OpenAPI     |
| Server state         | TanStack Query 5          | Caching, refetch, optimistic updates                  |
| Routing              | React Router 7            | Layout routes, loaders, guards                        |
| UI components        | shadcn/ui + Tailwind CSS  | Accessible, themeable, copy-paste primitives          |
| Forms                | React Hook Form + Zod     | Matches CESAR's strict validation                     |
| Mock server          | **MSW 2**                 | Intercepts at network level, typed handlers           |
| Tables               | TanStack Table 8          | Sorting, filtering, pagination (matches limit/offset) |
| I18n                 | react-i18next             | Spanish primary language for Argentine users          |
| Testing              | Vitest + Testing Library  | Consistent with CESAR's testing culture               |
| Linting              | Biome                     | Fast, all-in-one (ESLint + Prettier)                  |

### 4.3 Key Architectural Decisions

**ADR-1: OpenAPI as single source of truth.** All types, API client, and mock
handlers are generated from `cesar/api/openapi.yaml`. When the spec changes,
run `orval` to regenerate.

**ADR-2: Feature-based directory structure.** Each CESAR bounded context maps
to a `features/` directory containing its own pages, components, and hooks.
Cross-cutting concerns live in `components/shared/` and `lib/`.

**ADR-3: JWT stored in memory + HttpOnly cookie fallback.** Token is held in a
React context during the session. On page refresh, the `/api/v1/auth/me` endpoint
validates the persisted token.

**ADR-4: Idempotency keys generated client-side.** Every mutation request includes
a UUID v7 `Idempotency-Key` header. MSW mimics CESAR's 24h TTL deduplication.

**ADR-5: Spanish-first UI with i18n support.** All labels, messages, and error
messages are in Spanish. i18n keys are organized by module.

### 4.4 Component Tree

```
<App>
  <QueryClientProvider>
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          {/* Public */}
          <Route path="/login" element={<LoginPage />} />

          {/* Protected — requires auth */}
          <Route element={<AuthGuard />}>
            <Route element={<AppShell />}>        {/* Sidebar + Header */}
              {/* ADMIN routes */}
              <Route element={<RoleGuard role="ADMIN" />}>
                <Route path="/" element={<Dashboard />} />
                <Route path="/consorcios/*" element={<ConsorciosRoutes />} />
                <Route path="/expenses/*" element={<ExpensesRoutes />} />
                <Route path="/debt/*" element={<DebtRoutes />} />
                <Route path="/payments/*" element={<PaymentsRoutes />} />
                <Route path="/incomes/*" element={<IncomesRoutes />} />
              </Route>
              {/* ADMIN + ACCOUNTANT routes */}
              <Route element={<RoleGuard role={["ADMIN", "ACCOUNTANT"]} />}>
                <Route path="/invoices/*" element={<InvoicesRoutes />} />
              </Route>
              {/* Owner routes (read-only, scoped to their unit) */}
              <Route path="/my-unit/*" element={<OwnerRoutes />} />
            </Route>
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  </QueryClientProvider>
</App>
```

---

## 5. Development Phases

### Phase 0 — Scaffolding (1–2 days)

- **Vite + React + TypeScript project** with Tailwind CSS and shadcn/ui
- **orval configuration** pointing at `cesar/api/openapi.yaml` for type
  generation
- **MSW mock server** with handlers for all 61 endpoints returning realistic
  data
- **Auth context**: login flow, JWT storage, token refresh, role extraction
- **Axios client** with JWT interceptor, idempotency key injection, error
  normalization
- **Layout shell**: sidebar navigation, header with consorcio selector, user
  menu
- **Route structure** with role-based guards

### Phase 1 — Core Admin Flows (2–3 weeks)

- **Login page**: username/password form, JWT exchange, redirect
- **Consorcios**: create, view, update; UF configuration with percentage sum
  validation; propietario registry (CRUD); fiscal period state machine
  (OPEN → CLOSED → ARCHIVED)
- **Expenses**: CRUD with expense type toggle (ORDINARIA → auto-approved,
  EXTRAORDINARIA → pending); approval/rejection workflow; provider, concept,
  and category management
- **Dashboard**: key metrics per consorcio (total debt, collection rate,
  pending approvals)

### Phase 2 — Financial Operations (2–3 weeks)

- **Invoices**: Factura A (RI → RI, IVA discriminated), Factura B (RI → CF/Exento),
  Factura C; annul flow with confirmation; Notas de Crédito/Débito; Recibos
- **Liquidaciones**: generate for single unit or batch (all units); list with
  period/unit/status filters; view detailed expense allocation breakdown;
  issue workflow (DRAFT → ISSUED); PDF placeholder with "Coming soon" message
- **Payments**: register payment with payment method selector (CASH, TRANSFER,
  DEPOSIT, MERCADOPAGO, OTHER); reversal flow with reason dialog; unit balance
  summary; payment history with sort/filter

### Phase 3 — Reports & Owner Portal (1–2 weeks)

- **Certificados de Deuda**: generate from issued liquidación; list by unit;
  sign to make executive title
- **Financial summaries**: per-period income/output/net flow; cross-period
  with cascading balances
- **Owner-facing views**: unit liquidaciones, payment history, current balance,
  active debt certificates (read-only, scoped to their unit)
- **PDF download** (integrate once CESAR implements the endpoint)

### Phase 4 — Polish & Production Readiness (1–2 weeks)

- **Responsive design** for tablet and mobile
- **Error handling**: consistent toast notifications, field-level validation
  errors, retry logic for network failures
- **Loading states**: skeleton screens, progress indicators for long operations
- **Empty states**: helpful messages and CTAs for first-time users
- **Pagination**: limit/offset controls matching CESAR's API
- **E2E tests** with Playwright against MSW mocks

---

## 6. Risks & Mitigations

| Risk                                             | Impact | Mitigation                                                                        |
|--------------------------------------------------|--------|-----------------------------------------------------------------------------------|
| CESAR backend incomplete (only 2 DB migrations, many `todo!()` repos) | Cannot integrate until backend catches up | MSW mocks are fully typed from OpenAPI; switch to real API by changing base URL |
| OpenAPI spec may diverge from Rust implementation | Type mismatches at integration | Keep OpenAPI as source of truth; orval re-generates on every spec change |
| Complex domain rules (prorrateo, ARCA invoices, interest) may differ between frontend and backend | Validation inconsistencies | Don't implement business logic client-side; pass through API errors; use Zod only for field-level validation |
| Multi-role UI (ADMIN vs ACCOUNTANT vs owner) | Significant UI variants | Role guards in routing; MSW simulates different JWT payloads |
| Argentine-specific UX (CUIT, YYYYMM, ARS, IVA) | Non-standard UI patterns | Build shared components (CUITInput, MoneyDisplay, PeriodPicker, IvaConditionSelect) early |
| Missing `GET /consorcios` list endpoint | Cannot show consorcio selector | Propose adding to OpenAPI spec; fallback: store last selection in localStorage |
| No dashboard aggregation endpoints | KPIs must be client-side | Compute from filtered list queries; recommend backend aggregation endpoints later |
| CORS misconfiguration in production | Frontend blocked from API | CESAR already has `CorsLayer::permissive()`; tighten for production with allowed origins |

---

## 7. Gaps in CESAR That Impact the Frontend

| Missing Piece                                  | Frontend Impact               | Proposed Resolution                               |
|------------------------------------------------|-------------------------------|---------------------------------------------------|
| `GET /consorcios` (list all)                   | No consorcio selector on login| Add endpoint to OpenAPI spec; endorse with backend team |
| PDF liquidacion download returns 501           | "Download" button non-functional | Show disabled state with tooltip; mock in MSW for demo |
| No user management endpoints                   | Cannot create/edit users      | Not in scope for v0.1; handled by backend directly |
| No dashboard/metrics aggregation               | Stats computed client-side    | Acceptable for v0.1; propose `GET /dashboard` later |
| Phase 2 modules (salary, investment, legal)    | Not in CESAR either           | Deferred to future phases                           |

---

## 8. Summary

**The project is feasible and well-scoped.** The CESAR OpenAPI 3.1 specification
provides a complete, typed contract suitable for generating:

1. A fully typed TypeScript API client (via orval)
2. A comprehensive MSW mock server for independent development
3. Zod validation schemas matching CESAR's business rules

The 61 endpoints cover the full condominium management lifecycle: legal entity
setup, expense tracking, ARCA-compliant invoicing, debt calculation and
collection, payment processing, and financial reporting. The multi-role
architecture (ADMIN for full access, ACCOUNTANT for invoicing only, owner views
for self-service) maps cleanly to React Router guard patterns.

**Estimated timeline: 6–10 weeks** for a full-featured v0.1, assuming 1–2
developers working against MSW mocks while the CESAR backend matures in
parallel.

### Next Steps

1. Review and approve this feasibility document
2. Add missing `GET /consorcios` endpoint to CESAR OpenAPI spec
3. Scaffold Phase 0 (project setup with Vite + orval + MSW)
4. Begin Phase 1 development (core admin flows)

---

## 9. Architectural Alternatives Considered

Three architectural alternatives were evaluated against the project's specific
constraints: microfrontends, Web Components, and a BFF (Backend for Frontend)
layer.

### 9.1 Microfrontends

**What it means:** Each bounded context (consorcios, expenses, invoices, etc.)
is deployed as an independent frontend application, composed at runtime by a
shell app via Module Federation or a similar mechanism.

| Factor                     | Microfrontends                    | Monolithic SPA (current plan)        |
|----------------------------|-----------------------------------|--------------------------------------|
| Team size                  | Suited for 3+ independent teams   | Optimal for 1–2 developers           |
| Module independence        | Works best when modules are standalone apps, never cross-linked | CESAR's bounded contexts are **sequential in workflow**: create expenses → generate liquidaciones → register payments. Cross-navigation is frequent |
| Shared domain objects      | Requires shared library package; drift risk across MFs | Single import; no drift              |
| Routing                    | Cross-MF navigation complex (e.g., expense detail linking to its invoice) | React Router handles naturally       |
| Bundle size                | Each MF ships its own React, inflating total payload | Single tree-shaken bundle            |
| Build tooling              | Module Federation requires Webpack 5 / Rspack, adding config complexity | Vite alone; minimal configuration    |
| CI/CD                      | Each MF has its own pipeline and versioning | Single pipeline; simpler             |

**Verdict: Not adopted.** Microfrontends introduce orchestration complexity
(shell app, shared dependency negotiation, cross-MF communication, design
system consistency) with no payoff at this scale. The bounded contexts are not
independent "apps" — they are steps in a single administrative workflow driven
by one team.

### 9.2 Web Components

**What it means:** Shared Argentine-specific UI elements (CUITInput,
MoneyDisplay, PeriodPicker, IvaConditionSelect) are built as
framework-agnostic Web Components using the Custom Elements API.

| Component           | Feasible as a Web Component? | Practical concern                                                                                                                                          |
|---------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CUITInput`         | Yes (input + validation)     | Must integrate with React Hook Form, which does not natively bind to custom elements. Requires `useRef` + `useEffect` wiring, losing type safety          |
| `MoneyDisplay`      | Trivial (format + render)    | Overkill — it is a single formatting function; a React component with props is simpler                                                                     |
| `PeriodPicker`      | Possible (date range picker) | Already a complex component; wrapping in a Web Component adds an abstraction layer. React integration requires `@lit/react` wrappers or manual binding    |
| `IvaConditionSelect`| Yes (dropdown)               | shadcn/ui's `Select` primitive already solves this with full keyboard navigation, ARIA attributes, and consistent styling                                 |

Adding Web Components to a React project carries specific costs:

- React's synthetic event system does not propagate events from custom elements
  natively; every event must be manually forwarded.
- Form libraries (React Hook Form + Zod) lose type safety across the custom
  element boundary — `register()` cannot type-check the emitted value.
- The entire shadcn/ui ecosystem is React-only. Mixing Web Components means
  maintaining two parallel sets of design tokens and interaction patterns.
- Shadow DOM encapsulation blocks Tailwind CSS utility classes unless the
  stylesheet is injected into each shadow root.

**Verdict: Not adopted.** Web Components solve cross-framework sharing, a
problem that does not exist in this project (React is the only framework).
Shared components as plain React components with typed props are simpler,
type-safe, and work seamlessly with React Hook Form, shadcn/ui, and
Tailwind CSS.

### 9.3 BFF (Backend for Frontend)

**What it means:** A lightweight server-side layer (e.g., Next.js API routes)
sits between the React SPA and CESAR's Rust API. It handles endpoint
aggregation, response shaping, SSR, and auth proxying.

**Arguments for adding a BFF:**

1. **Endpoint aggregation** — CESAR lacks `GET /consorcios` (list) and
   `GET /dashboard` (metrics) endpoints. A BFF could fan out to multiple CESAR
   calls and return a single aggregated response, reducing client-side
   chattiness.
2. **Response shaping** — CESAR returns raw domain objects. A BFF could flatten
   nested structures, strip fields based on role, or add computed values (e.g.,
   collection rate, overdue count) without modifying the Rust backend.
3. **SSR potential** — If SEO or faster initial loads matter later, a BFF
   (Next.js with SSR) enables server-rendered pages. Public-facing owner
   portals could benefit.
4. **Auth hardening** — Instead of storing the JWT in the browser (localStorage
   or memory), the BFF validates the token server-side and uses an HttpOnly
   session cookie with the browser. More resilient to XSS.

**Arguments against adding a BFF now:**

1. **Additional service** — CESAR + web app = 2 services. Adding a BFF makes it
   3. More infrastructure, deployment, and monitoring overhead.
2. **MSW complexity** — MSW currently mocks the CESAR API directly. With a BFF,
   you must mock either the BFF itself or the CESAR calls behind it. Either
   adds indirection during development.
3. **TanStack Query already handles client-side aggregation** — you can compose
   multiple queries, cache results, and compute derived state without a server
   intermediary.
4. **CESAR already has permissive CORS** — no CORS bypass is needed.
5. **The missing endpoints should ideally be fixed at the source** — adding
   `GET /consorcios` and dashboard aggregation to CESAR itself is the correct
   long-term solution, not papering over gaps with a BFF.

| Scenario                         | Direct API (current plan)                         | With BFF                                           |
|----------------------------------|---------------------------------------------------|----------------------------------------------------|
| Dashboard loading                | 6 parallel GET calls; TanStack Query handles race | 1 BFF call that fans out and aggregates all 6      |
| Missing `GET /consorcios`        | Client-side workaround (localStorage fallback)    | BFF could proxy a filtered list, but still needs the backend endpoint to exist |
| PDF download                     | Direct `GET /liquidaciones/{id}/pdf` (binary)     | BFF proxies the binary stream; adds latency        |
| Auth                             | JWT in browser memory, Bearer header on every request | JWT validated server-side; HttpOnly session cookie in browser |
| Added complexity                 | None                                              | Another repo, CI/CD pipeline, deployment target    |

**Verdict: Deferred, not rejected.** A BFF adds real value for endpoint
aggregation and auth hardening, but it is premature for v0.1. The recommended
approach:

- **v0.1:** Direct API consumption (current plan). Prove the UI works
  against the OpenAPI contract. Identify concrete pain points (e.g., how many
  parallel calls does a dashboard really need? Is chattiness a real problem?).
- **Post Phase 2 evaluation:** If aggregation chattiness or auth concerns
  become bottlenecks, introduce a thin BFF layer. The most natural fit would be
  migrating the Vite SPA to Next.js, keeping the React components and adding
  API routes incrementally.

When/if a BFF is introduced, the technology choice would be **Next.js API
routes** (or tRPC if type-safety across the BFF boundary is desired), keeping
the React frontend within the same project for colocation benefits.

### 9.4 Summary of Architectural Decisions

| Concern                           | Evaluated              | Adopted / Deferred    | Rationale                                                                 |
|-----------------------------------|------------------------|------------------------|---------------------------------------------------------------------------|
| Microfrontends                    | No                     | Monolithic React SPA   | Team of 1–2; sequential workflows; no independent release benefit         |
| Web Components for shared widgets | No                     | React components       | Single-framework project; Web Components break React Hook Form + shadcn/ui |
| BFF (Backend for Frontend)        | Deferred (post-Phase 2) | Direct API consumption | Premature for v0.1; evaluate after proving UI against OpenAPI contract    |
