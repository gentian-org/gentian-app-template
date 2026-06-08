# AGENTS.md — Gentian app development conventions

This file helps AI coding agents and humans extend Gentian first-party apps.

## Directory map

| Path | Purpose |
|------|---------|
| `backend/app/main.py` | FastAPI entrypoint |
| `backend/app/core/config.py` | Settings from environment (ESO-injected in cluster) |
| `backend/app/core/auth.py` | OIDC JWT validation |
| `backend/app/api/routes/` | HTTP routers |
| `frontend/src/` | React UI |
| `chart/` | Helm chart (Pattern A `existingSecret`) |
| `profile/appprofile.yaml.tmpl` | AppProfile skeleton for `gentian-apps/profiles/` |

## Add an API endpoint

1. Create `backend/app/api/routes/<feature>.py` with an `APIRouter`.
2. Register it in `backend/app/main.py`.
3. Protect routes with `Depends(get_current_user)` when tenant-scoped.

## Add a React page

1. Add component under `frontend/src/pages/`.
2. Wire routing in `frontend/src/App.tsx` (or add a router).
3. Call backend via `/api/v1/...` (proxied by nginx in production).

## Kernel secrets (cluster)

Never commit secrets. The orchestrator injects via ExternalSecret:

- `DATABASE_URL`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`

Map keys in `profile/appprofile.yaml.tmpl` `valueMapping` must match Helm `values.yaml`.

## Publish a new app version

1. Bump `chart/Chart.yaml` version and image tags in `chart/values.yaml`.
2. CI builds and pushes images + OCI chart.
3. Update `gentian-apps/profiles/<app>.yaml` `spec.chart.version`.
4. AppProfile update reconciler rolls out to tenants.

## Local dev

```bash
docker compose -f docker-compose.dev.yaml up --build
```

`AUTH_DISABLED=true` skips OIDC locally.

## Cursor Cloud specific instructions

### Dependency refresh (automatic on VM startup)

- Frontend: `cd frontend && npm install`
- Backend: `cd backend && ([ -x .venv/bin/python ] || uv venv .venv) && uv pip install --python .venv/bin/python -e ".[dev]"` (requires `uv` on `PATH`, typically `~/.local/bin`)

### Running services (native dev — recommended in Cloud Agent VMs)

Docker Compose is documented in the README, but the web container’s `nginx.conf` proxies to `127.0.0.1:8000`, which does not reach the separate `api` container. Use native dev instead:

```bash
# Terminal 1 — API (from backend/, with venv activated)
AUTH_DISABLED=true BACKEND_CORS_ORIGINS=http://localhost:5173 \
  uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 2 — UI (from frontend/)
npm run dev
```

- API: http://localhost:8000/docs
- UI: http://localhost:5173

Vite proxies `/api` to port 8000 but not `/healthz`, so the template UI’s health badge may show “Backend: error” even when the API is healthy. Verify the API directly at http://localhost:8000/healthz or via Swagger.

### Lint and test

```bash
# Backend (from backend/, venv active)
ruff check app
pytest   # no tests in template yet; exit code 5 = nothing collected

# Frontend
npm run build   # tsc + vite build
```

### Docker (optional)

If you need Compose (db + api + web), start the daemon first (`sudo dockerd` in background), then `sudo docker compose -f docker-compose.dev.yaml up --build`. API is reachable on :8000; the web container’s nginx proxy to the API will not work until `nginx.conf` uses the compose service hostname (e.g. `http://api:8000`).
