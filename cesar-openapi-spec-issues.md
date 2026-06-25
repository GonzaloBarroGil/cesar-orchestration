# CESAR OpenAPI Spec — Issues Found During Web Contract Setup

> **Date:** 2026-06-25
> **Phase:** Contract Setup (cesar-web Phase 0)
> **Severity:** Medium — blocks orval generation without workarounds
> **Target:** `cesar/api/openapi.yaml` (5,779 lines, OpenAPI 3.1.0)

---

## Issue 1: Invalid `$ref` to `#/components/headers/IdempotencyKey`

**Severity:** High — causes orval generation to fail with `MissingPointerError`.

### Description

The spec references a non-existent path `#/components/headers/IdempotencyKey` in
7 locations. The `IdempotencyKey` parameter is defined at
`#/components/parameters/IdempotencyKey` (line 2784), not under a `headers`
namespace. The `components` object has three sections: `securitySchemes`,
`parameters`, and `schemas`. There is no `headers` section.

### Affected Lines

```
#/components/headers/IdempotencyKey  →  should be  #/components/parameters/IdempotencyKey
```

The 7 occurrences span POST and PUT/PATCH endpoints across all modules that
accept the `Idempotency-Key` header.

### Reproduction

```bash
# With any OpenAPI tool that resolves $ref pointers:
openapi-generator validate api/openapi.yaml
# Expected: resolution error
```

### Fix

Replace all 7 occurrences of `#/components/headers/IdempotencyKey` with
`#/components/parameters/IdempotencyKey`.

```bash
sed -i 's|#/components/headers/IdempotencyKey|#/components/parameters/IdempotencyKey|g' api/openapi.yaml
```

### Workaround (used by cesar-web)

A sed-fixed copy is used at `/tmp/cesar-openapi-fixed.yaml` as the orval input.
This avoids modifying CESAR's source spec while the fix is pending upstream.

---

## Issue 2: Undefined `#/components/responses/` References

**Severity:** Low — orval generates but with missing type imports.

### Description

Multiple operations reference response objects via `$ref` to
`#/components/responses/400`, `#/components/responses/401`,
`#/components/responses/403`, `#/components/responses/404`,
`#/components/responses/409`, and `#/components/responses/422`. However, the
OpenAPI spec does not define a `responses` section under `components`. These
references resolve to nothing, causing orval to generate TypeScript code that
imports non-existent types.

The following missing response types were observed:

| Referenced `$ref` | Generated TS Type | Actual Resolution |
|---|---|---|
| `#/components/responses/400` | `BadRequestResponse` / `N400Response` | Undefined |
| `#/components/responses/401` | `UnauthorizedResponse` / `N401Response` | Undefined |
| `#/components/responses/403` | `ForbiddenResponse` / `N403Response` | Undefined |
| `#/components/responses/404` | `NotFoundResponse` / `N404Response` | Undefined |
| `#/components/responses/409` | `ConflictResponse` / `N409Response` | Undefined |
| `#/components/responses/422` | `BusinessRuleViolationResponse` / `N422Response` | Undefined |
| `#/components/responses/500` | `N500Response` | Undefined |

### Impact

- orval generates a warning (`MissingPointerError`) but does not fail.
- The generated TypeScript code imports types that don't exist, causing `tsc`
  errors until stub types are manually created.
- MSW handler type-checking against error responses is incomplete.

### Fix

Add a `responses` section under `components` in `api/openapi.yaml` that defines
each referenced response object, reusing the existing `ErrorResponse` schema:

```yaml
components:
  responses:
    '400':
      description: Malformed request body
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '401':
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '403':
      description: Insufficient role
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '404':
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '409':
      description: Conflict (duplicate resource or idempotency key)
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '422':
      description: Business rule violation
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    '500':
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

### Workaround (used by cesar-web)

Stub TypeScript types were manually created at
`src/api/generated/models/{badRequestResponse,unauthorizedResponse,...}.ts`,
each aliasing `ErrorResponse`. These must be regenerated if orval is re-run
with `clean: true`.

---

## Issue 3: Missing `GET /consorcios` List Endpoint

**Severity:** Medium — frontend cannot show consorcio selector (already documented
in `cesar-web-feasibility.md` §7).

### Description

The OpenAPI spec defines `POST /consorcios` (create) and `GET /consorcios/{id}`
(get by ID), but no `GET /consorcios` endpoint to list all consorcios. The
frontend's ConsorcioSelector component needs this to populate the dropdown on
the app shell.

### Fix

Add `GET /consorcios` returning an array of `ConsorcioResponse`:

```yaml
/consorcios:
  get:
    tags: [Consorcio]
    summary: List all consorcios
    operationId: listConsorcios
    responses:
      '200':
        description: List of consorcios
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/ConsorcioResponse'
```

### Workaround (used by cesar-web)

MSW handler manually returns a hardcoded array. When the real endpoint is added,
remove the manual handler.

---

## Issue 4: Missing Auth Endpoints in OpenAPI Spec

**Severity:** High — frontend cannot generate typed hooks for login/session/health.

### Description

The OpenAPI spec does not define any authentication or health endpoints:

- `POST /api/v1/auth/login` — not in the spec
- `GET /api/v1/auth/me` — not in the spec
- `GET /api/v1/health` — not in the spec

The tags section also has no `Auth` tag. These endpoints are referenced in the
security scheme description (`BearerAuth: JWT token obtained via POST /api/v1/auth/login`)
and in the feasibility analysis, but no path definitions exist.

### Impact

- orval cannot generate typed TanStack Query hooks for auth.
- MSW handlers must be hand-written without type support.
- TypeScript cannot verify request/response contract for auth.
- Health endpoint is used by the frontend's `HealthIndicator` component.

### Fix

Add three endpoints to the OpenAPI spec under a new `Auth` tag:

```yaml
tags:
  - name: Auth
    description: Authentication and health endpoints

paths:
  /auth/login:
    post:
      tags: [Auth]
      summary: Login
      operationId: login
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [username, password]
              properties:
                username:
                  type: string
                password:
                  type: string
      responses:
        '200':
          description: JWT token and user info
          content:
            application/json:
              schema:
                type: object
                required: [token, user]
                properties:
                  token:
                    type: string
                  user:
                    $ref: '#/components/schemas/UserResponse'
        '401':
          $ref: '#/components/responses/401'

  /auth/me:
    get:
      tags: [Auth]
      summary: Get current user
      operationId: getMe
      security:
        - BearerAuth: []
      responses:
        '200':
          description: Current user info
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '401':
          $ref: '#/components/responses/401'

  /health:
    get:
      tags: [Auth]
      summary: Health check
      operationId: healthCheck
      responses:
        '200':
          description: Service health status
          content:
            application/json:
              schema:
                type: object
                required: [status]
                properties:
                  status:
                    type: string
                    enum: [ok, degraded]
                  postgres:
                    type: string
                  redis:
                    type: string
                  nats:
                    type: string
```

### Workaround (used by cesar-web)

MSW handlers manually written in `src/mocks/handlers/auth.ts` with hand-typed
request/response shapes. Zod schemas written manually in `src/api/schemas.ts`.
No generated type support.

---

## 5. Priority & Recommendations

| Issue | Priority | Fix Complexity | Blocking? |
|---|---|---|---|
| Issue 1 — Invalid `$ref` headers/IdempotencyKey | **High** | Trivial (7x sed) | Yes — orval fails |
| Issue 2 — Missing `responses` component | **Medium** | Low (add 7 response objects) | Partially — stub types needed |
| Issue 3 — Missing `GET /consorcios` | Medium | Low (1 endpoint + handler) | No — MSW workaround in place |
| Issue 4 — Missing auth endpoints | **High** | Low (add 3 endpoints + `UserResponse` schema) | Yes — no typed auth hooks |

### Recommended Action

1. **Fix Issue 1 immediately** — it's a trivial spec bug (7 string replacements)
   that blocks any OpenAPI tooling.
2. **Fix Issue 4** — auth is a foundation feature needed by every other module.
   Without typed hooks, the contract boundary is broken.
3. **Fix Issue 2** — defines proper response objects, letting orval generate
   correct types without stubs. Low effort, improves spec completeness.
4. **Fix Issue 3** — needed before v0.1 integration with the real backend.
   MSW covers development phase.

---

*End of spec issues.*
