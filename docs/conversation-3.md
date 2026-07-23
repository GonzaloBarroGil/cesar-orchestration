# Conversation 3 — Remote Deployment (Railway → Render + Vercel)

> **Date:** July 22, 2026
> **Participants:** Gonzalo (user), opencode (assistant)
> **Goal:** Deploy cesar (backend) and cesar-web (frontend) to production cloud services with GitHub auto-deploy.

---

## Transcript

### Step 1: Recovering Previous Session

**User:** Hi, again. Can you recover our last conversation? We were preparing remote deploy, I was creating projects in Vercel and Railway and then this laptop shutted down.

**Assistant:** Resumed session `ses_0752e3151ffex71QfJn2qazLye` — "Resuming Vercel and Railway deployment". Found deployment configs already committed:
- `cesar/Dockerfile` + `railway.json` — Railway backend
- `cesar-web/vercel.json` — Vercel frontend

Both committed July 21.

---

### Step 2: Railway Build Failure

**User:** Shares Railway log — build fails on `cargo build --release`.

**Error:** Multiple crates require rustc ≥ 1.86 or 1.88, but Dockerfile uses `rust:1.85-slim-bookworm`:

| Crate | Requires |
|-------|----------|
| darling, image, time, serde_with, home | rustc 1.88 |
| icu_collections, icu_properties, idna_adapter | rustc 1.86 |

**Fix:** Bumped Dockerfile builder image from `rust:1.85-slim-bookworm` → `rust:1.88-slim-bookworm`.

Commit: `fix(deploy): bump Rust builder image to 1.88 for dependency compat`

---

### Step 3: Switching from Railway to Render

**User:** Railway OAuth login failed. "Unable to connect to authorize. I think this server is not safe. Could we switch to another service?"

**Assistant:** Evaluated alternatives (Render, Fly.io, DO, Koyeb). Recommended Render — built-in PostgreSQL, Docker support, GitHub auto-deploy, free tier.

**User:** Switched to Render. Removed Railway from GitHub (Settings → Installations → Revoke).

---

### Step 4: Preparing cesar for Render

Four issues identified:

| Issue | Fix |
|-------|-----|
| Port hardcoded to `127.0.0.1:3000` | Read `PORT` env var, bind to `0.0.0.0` |
| Redis/NATS mandatory at startup | Made `Option<ConnectionManager>` and `Option<Client>` in AppState |
| Middleware panics without Redis | Skip idempotency when Redis is None |
| Health check reports degraded | "not configured" treated as healthy for Redis/NATS |

**Files changed:**
- `src/main.rs` — port from `$PORT` env var
- `src/shared/state.rs` — Redis/NATS optional connections
- `src/shared/middleware.rs` — skip idempotency w/o Redis
- `src/shared/health.rs` — accept "not configured" as healthy
- `render.yaml` — blueprint: web service (Dockerfile) + PostgreSQL 16
- `Dockerfile` — builds `seed_admin` binary, uses entrypoint.sh
- `entrypoint.sh` — seeds admin user on startup, then starts cesar

---

### Step 5: Deploying to Render

**User:** Created Render project, applied blueprint.

**Render created:**
- `cesar-db` — PostgreSQL 16 database (success)
- `cesar-api` — web service from Dockerfile (success)

**Result:** Server live at `https://cesar-api-ijf1.onrender.com`. Health check passes:

```json
{"status":"ok","dependencies":{"postgres":"ok","redis":"not configured","nats":"not configured"}}
```

---

### Step 6: Preparing cesar-web for Vercel

**Problem:** `client.ts` used dead `process.env.VITE_API_URL` path (never resolved in browser Vite builds). Hardcoded fallback `http://localhost:3000/api/v1`.

**Fix:**
- `src/api/client.ts` — changed `getBaseUrl()` to use `import.meta.env.VITE_API_URL ?? "http://localhost:3000/api/v1"`
- `.env.production` — `VITE_API_URL=https://cesar-api-ijf1.onrender.com/api/v1`

Build passes: `tsc --noEmit` clean, `npm run build` successful (727 KB JS bundle).

**User:** Deployed to Vercel. Committed changes.

---

### Step 7: Admin User Seeding

**User:** Reminded how to use seed_admin credentials.

**Assistant:** Explained seed_admin binary creates user via argon2 hash. But on Render the Dockerfile only built the main `cesar` binary, not `seed_admin`.

**Fix:**
- `Dockerfile` — builds both binaries, copies them to runtime stage
- `entrypoint.sh` — runs `seed_admin` before `cesar` if `ADMIN_PASSWORD` is set
- `render.yaml` — added `ADMIN_USERNAME=admin` and auto-generated `ADMIN_PASSWORD`

On next Render deploy, admin user will be created automatically. Password visible in Render dashboard.

---

## Current State (End of Conversation)

### Deployed Services

| Service | Platform | URL |
|---------|----------|-----|
| cesar API | Render | `https://cesar-api-ijf1.onrender.com` |
| cesar DB | Render | PostgreSQL 16 (internal) |
| cesar Web | Vercel | TBD (deploying) |

### Pending

- [ ] Verify cesar-web Vercel deploy succeeds
- [ ] Login with admin credentials (`ADMIN_PASSWORD` from Render dashboard)
- [ ] Test full stack end-to-end

### Key Technical Decisions

- **EAD-002 (implicit): Render over Railway** — Railway OAuth issues + CLI limitations led to Render which provides built-in PostgreSQL, simpler Docker deployment, and free tier
- **EAD-003 (implicit): Optional infrastructure** — Redis and NATS made optional so deployment works without external services; health check treats "not configured" as healthy

---

*End of conversation.*
