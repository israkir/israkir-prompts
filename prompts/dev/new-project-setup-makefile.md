---
id: new-project-setup-makefile
title: New Project Setup — Makefile Conventions + Production Integration
category: dev
tags: [makefile, deployment, podman, local-dev, onboarding, monorepo, ghcr, ci, pnpm, uv]
version: 1.3.0
---

# Prompt: New Project Setup — Makefile Conventions + Production Integration

> **How to use:** Paste this document as the full task brief for an LLM coding agent onboarding a new app to local dev and a production deployment stack.  
> **Goal:** Implement canonical Makefile targets, repo layout, CI/container-registry image flow, and production integration consistent with the org’s established patterns.

---

## Role

You are a senior platform engineer setting up a new application for:

- **Local dev** — app processes on the host with hot reload; only datastores in containers
- **Production** — CI-built images pushed to a container registry; the production host pulls and runs them (no local image builds on the server)
- **Consistent Makefile vocabulary** — `be-*`, `fe-*`, `storage-*`, `worker-dev` targets agents and humans can rely on across repos

Adapt paths and stack details to the project, but preserve target names and contracts unless you document a deliberate exception.

---

# New project setup — Makefile conventions + production integration

Checklist for onboarding a new app to local development and a shared production deployment stack.

## Principles

1. **Local dev** — app code on the host with hot reload (`be-dev`, `fe-dev`); only datastores in containers (`storage-up`). Do **not** run the API or web app in containers for daily dev.
2. **Production** — CI builds images → container registry → production host pulls and runs them. No local image builds on the server.
3. **Naming** — `be-*` backend, `fe-*` frontend, `storage-*` datastores, `worker-dev` for background jobs.
4. **Container runtime** — **Podman Compose** locally (`podman-compose` preferred; fall back to `podman compose`). Dockerfiles exist for CI/prod only.
5. **Env** — single root `.env` (from `.env.example`) consumed by compose and host dev servers. Do not scatter secrets across multiple env files for local dev unless production requires it.
6. **Aliases** — keep deprecated names (`up`, `down`, `migrate`, `dev-api`, `install-api`, …) as thin forwards to canonical targets; list them in docs.

---

## Required Makefile targets

Implement every target below (or a documented no-op with a clear message).

| Target | Contract |
|--------|----------|
| `env` | Copy `.env.example` → `.env` if missing |
| `storage-up` | Start dev datastores (Postgres/Timescale, Redis, app-specific: MinIO / …). **Does not** migrate or seed. Run post-start hooks (e.g. MinIO CORS) when needed. |
| `storage-down` | Stop compose services without wiping volumes |
| `storage-reset` | Wipe volumes and recreate empty datastores. **Does not** auto-migrate — caller runs `be-migrate` after. |
| `wait-postgres` | Block until Postgres accepts connections (used by `be-migrate`) |
| `be-install` | Backend venv + deps via **uv** (idempotent). Alias: `install-api`. |
| `fe-install` | Frontend workspace deps via **pnpm** (idempotent). Alias: `install-web`. |
| `install` | Full onboarding: Node monorepo + `be-install` + pre-commit hooks |
| `be-migrate` | Alembic `upgrade head` (after `wait-postgres`). Alias: `migrate`. |
| `be-dev-seed` | Demo / fixture data. Prefer explicit names: `be-dev-seed-all`, `be-dev-seed-accounts`, … |
| `be-dev` | API with reload on `:8000`; free port if occupied. Alias: `dev-api`. |
| `fe-dev` | Next dev server on `:3000`; free port if occupied. Alias: `dev-web`. |
| `worker-dev` | Background worker on host (streaq / equivalent). Alias: `dev-worker`. |
| `lint` / `test` | Both stacks (aggregate `lint-ts` + `lint-api`, `test-api` + `test-web`) |
| `ci` | Local pre-push gate (API + web; disk-aware) |
| `help` | Daily targets + colored quick start (default goal) |
| `help-all` | Extended targets: install tiers, OpenAPI, utilities, deprecated aliases |

### Shared quick start (include in `make help`)

```bash
make env                        # once — .env from .env.example
make be-install fe-install      # once
make storage-up
make be-migrate
make be-dev-seed-all            # optional — full demo (or lighter: be-dev-seed-accounts)

make be-dev                     # terminal 1 — http://localhost:8000/docs
make fe-dev                     # terminal 2 — http://localhost:3000
make worker-dev                 # optional — async jobs
```

Point to the org’s shared local-dev doc when one exists (e.g. `DEVELOPMENT.md` in the app repo or a central infra repo).

### Recommended `make help` sections

1. **Top row** — `install`, `build`, `lint`, `test`, `ci`, `clean`
2. **Frontend** — `fe-install`, `fe-dev`, `lint-ts`, `format`, `test-web`
3. **Mobile** (if applicable) — `dev-mobile`, `dev-emulator`, `test-mobile`
4. **Backend** — `be-install`, `be-dev`, `be-migrate`, `be-dev-seed-*`, `lint-api`, `test-api`, `worker-dev`
5. **Storage** — `storage-up`, `storage-down`, `storage-reset`, `ps`
6. **Quick start** — block above
7. Footer — `make help-all` for extended targets

Use section colors (FE yellow, BE magenta, storage green), dim descriptions, and `NO_COLOR=1` support. Use `printf '%b\n'` with double-quoted strings so ANSI codes render.

### Tiered install targets (monorepo)

| Target | Scope | Disk guard (~free) |
|--------|--------|-------------------|
| `be-install` | Python venv only | minimal |
| `install-api-client` | OpenAPI codegen package only | ≥ 1 GB |
| `fe-install` | Web + shared packages (no mobile) | ≥ 4 GB |
| `install-node` | Full monorepo incl. mobile | ≥ 8 GB |
| `install` | `install-node` + `be-install` + pre-commit | ≥ 8 GB |

Wire disk checks through `scripts/preflight.sh` (`make doctor`).

---

## Repo layout (monorepo example)

```
myapp/
  apps/api/              # FastAPI backend (or apps/backend/)
  apps/web/              # Next.js frontend (or apps/frontend/)
  apps/mobile/           # optional — Expo
  packages/              # shared TS: api-client, domain, ui-*, …
  infra/
    compose.yml          # storage + optional --profile app services
  scripts/
    preflight.sh         # disk + node_modules health
    seed_demo.py         # demo data
    export_openapi.py    # contract export
  .env.example           # single local env template (compose + host dev)
  apps/api/Dockerfile    # CI / prod API (+ worker reuses image)
  apps/web/Dockerfile    # CI / prod web (build context = repo root for workspaces)
  Makefile               # canonical dev vocabulary
```

- **Do not** run the app in containers for daily dev (`be-dev` / `fe-dev` on the host).
- **Do** keep Dockerfiles for CI and production image pulls.
- **Optional** containerized stack: `up-app` starts infra + API + worker via compose `--profile app` (smoke / integration only).

### Compose conventions

```makefile
COMPOSE_CMD  := $(strip $(shell \
  command -v podman-compose >/dev/null 2>&1 && echo "podman-compose" || \
  (podman compose version >/dev/null 2>&1 && echo "podman compose") || \
  echo ""))
COMPOSE_FILE   := infra/compose.yml
COMPOSE_PROJECT := <app>-infra
COMPOSE        := $(COMPOSE_CMD) -p $(COMPOSE_PROJECT) -f $(COMPOSE_FILE) --env-file .env
COMPOSE_APP    := $(COMPOSE) --profile app
```

- Bind datastore ports to localhost (`5432`, `6379`, `9000`, …).
- Healthchecks on postgres, redis, minio.
- App services (`api`, `worker`) behind `profiles: ["app"]` — not started by `storage-up`.
- `storage-up` may chain one-shot setup (e.g. `minio-cors` for browser presigned PUTs).

---

## Makefile implementation patterns

### Shell and defaults

```makefile
SHELL := /bin/bash
.DEFAULT_GOAL := help
```

### Path variables

```makefile
API_DIR   := apps/api
API_VENV  := $(API_DIR)/.venv
API_BIN   := $(CURDIR)/$(API_VENV)/bin
INFRA_DIR := infra
PNPM      := pnpm
```

### Backend install (uv, not raw pip)

```makefile
bootstrap-uv:  ## install uv if missing (Homebrew or astral.sh)
be-install: bootstrap-uv
  cd $(API_DIR) && (test -d .venv || uv venv --python 3.12 .venv) && uv pip install -e ".[dev]"
```

### Frontend install (pnpm workspace filters)

```makefile
PNPM_INSTALL_WEB := CI=true $(PNPM) install --frozen-lockfile --filter @<org>/web...
fe-install: env
  $(PNPM_INSTALL_WEB)
```

### Run banners and guards

- `say` / `run` macros for colored `▶` output (respect `NO_COLOR`, TTY, `FORCE_COLOR=1`).
- `require_compose` — fail with install hints if Podman Compose missing.
- `require_podman` — fail if `podman machine` not running (macOS).
- `be-dev` / `fe-dev` — `lsof -ti tcp:PORT | xargs kill -9` before starting.

### Storage targets

```makefile
storage-up: env
  $(COMPOSE) up -d
  @$(MAKE) --no-print-directory minio-cors   # if MinIO + browser uploads

storage-down:
  $(COMPOSE) down

storage-reset:
  $(COMPOSE) down -v --remove-orphans
  # optional: podman volume rm for $(COMPOSE_PROJECT) orphans

be-migrate: wait-postgres install-api
  cd $(API_DIR) && $(API_BIN)/alembic upgrade head
```

### Deprecated aliases (document in README)

| Alias | Canonical |
|-------|-----------|
| `up` | `storage-up` |
| `down` | `storage-down` |
| `clean-storage` | `storage-reset` |
| `migrate` | `be-migrate` |
| `seed` / `be-dev-seed` | `be-dev-seed-all` |
| `dev-api` | `be-dev` |
| `dev-web` | `fe-dev` |
| `dev-worker` | `worker-dev` |
| `install-api` | `be-install` |
| `install-web` | `fe-install` |

---

## Production integration checklist

Production often lives in a **separate infrastructure / deploy repo**, not the app repo. Wire the new app there following existing stacks in that repo.

1. Add a compose stack — registry images, external edge network, healthchecks, resource limits (copy patterns from sibling apps).
2. Add reverse-proxy routes (hostnames, TLS, upstreams).
3. Add DB user/password and shared secrets to the deploy repo’s `.env.example`.
4. Add app-specific runtime env template under the deploy project (e.g. `<app>.env.example`).
5. Extend the deploy repo Makefile: `<app>-up`, `<app>-down`, `deploy-<app>`.
6. Extend the deploy script with a migrate command for the new stack.
7. Document hostnames and CI image names in deployment docs (e.g. `DEPLOYMENT.md` § CI/images).
8. Keep app-specific datastores (MinIO, Mongo, …) in the app’s compose project when they are not shared infra.

**Example production hostnames** (replace with your app):

| Service | Hostname |
|---------|----------|
| Web + API edge | `app.example.com` |
| Media / object storage origin | `media.app.example.com` |

---

## CI / container registry

### Image names

- `ghcr.io/<org>/<app>-api` — API + worker (worker = same image, different `command`)
- `ghcr.io/<org>/<app>-web` — Next.js frontend

### Release workflow pattern

`.github/workflows/release.yml`:

- **Gate:** publish only after successful `CI` workflow on `main` (`workflow_run`) or manual `workflow_dispatch`.
- **Tags:** git SHA (short) always; `latest` on `main`; optional `v*` tag on version branches.
- **Web build context:** repo root (monorepo workspace packages).
- **API build context:** `apps/api/` (or `apps/backend/`).
- Bake `NEXT_PUBLIC_*` at **image build time** for production hostnames:

```yaml
build-args: |
  NEXT_PUBLIC_API_URL=https://app.example.com
  NEXT_PUBLIC_S3_ORIGIN=https://media.app.example.com
```

### Local prod-like smoke build

```makefile
PROD_HOST ?= app.example.com
PROD_MEDIA_HOST ?= media.app.example.com
PROD_IMAGE_TAG ?= latest
RUNTIME ?= $(shell command -v podman >/dev/null && echo podman || echo docker)

prod-images:
  $(RUNTIME) build -f apps/web/Dockerfile \
    --build-arg NEXT_PUBLIC_API_URL=https://$(PROD_HOST) \
    --build-arg NEXT_PUBLIC_S3_ORIGIN=https://$(PROD_MEDIA_HOST) \
    -t <app>-web:$(PROD_IMAGE_TAG) .
  $(RUNTIME) build -f apps/api/Dockerfile -t <app>-api:$(PROD_IMAGE_TAG) apps/api
```

Deploy from the infrastructure repo (`deploy-<app>`), not from the app repo.

---

## Quality / OpenAPI (API + generated client monorepos)

When the frontend consumes a generated OpenAPI client:

| Target | Purpose |
|--------|---------|
| `openapi-export` | Regenerate `openapi.json` + TS client |
| `openapi-export-stage` | Export + `git add` contract artifacts |
| `check-api-contract` | Fail CI if contract is stale |
| `ci-api` | API lint + tests + contract (no full pnpm install) |
| `lint-api` | Ruff + format check + vulture + mypy |
| `lint-ts` | Typecheck + ESLint + knip (web filter) |
| `pre-commit` | All git hooks manually |

Register hooks via `install-hooks` (`pre-commit install` in API venv).

---

## Starter Makefile snippet

Minimal template — adapt paths; for full UX add colored `help`, `help-all`, and `preflight.sh` integration.

```makefile
SHELL := /bin/bash
.DEFAULT_GOAL := help

API_DIR      := apps/api
API_VENV     := $(API_DIR)/.venv
API_BIN      := $(CURDIR)/$(API_VENV)/bin
INFRA_DIR    := infra
PNPM         := pnpm
PNPM_INSTALL_WEB := CI=true $(PNPM) install --frozen-lockfile --filter @myorg/web...
COMPOSE_CMD  := $(strip $(shell \
  command -v podman-compose >/dev/null 2>&1 && echo "podman-compose" || \
  (podman compose version >/dev/null 2>&1 && echo "podman compose") || echo ""))
COMPOSE_FILE   := $(INFRA_DIR)/compose.yml
COMPOSE_PROJECT := myapp-infra
COMPOSE        := $(COMPOSE_CMD) -p $(COMPOSE_PROJECT) -f $(COMPOSE_FILE) --env-file .env

.PHONY: help env storage-up storage-down storage-reset wait-postgres \
        be-install fe-install install be-migrate be-dev-seed-all be-dev fe-dev worker-dev lint test

help:
	@echo "Quick start:"
	@echo "  make env && make be-install fe-install"
	@echo "  make storage-up && make be-migrate"
	@echo "  make be-dev-seed-all    # optional"
	@echo "  make be-dev             # :8000"
	@echo "  make fe-dev             # :3000"
	@echo "  make worker-dev         # optional"
	@echo ""
	@echo "See make help-all for install tiers, OpenAPI, and utilities."

env:
	@test -f .env || cp .env.example .env

storage-up: env
	@test -n "$(COMPOSE_CMD)" || (echo "Install podman-compose"; exit 1)
	$(COMPOSE) up -d

storage-down:
	$(COMPOSE) down

storage-reset:
	$(COMPOSE) down -v --remove-orphans

wait-postgres:
	@for i in $$(seq 1 90); do \
	  pg_isready -h 127.0.0.1 -p 5432 -U myapp >/dev/null 2>&1 && exit 0; \
	  sleep 2; \
	done; echo "Postgres not ready"; exit 1

be-install:
	@command -v uv >/dev/null || (echo "Install uv: brew install uv"; exit 1)
	cd $(API_DIR) && (test -d .venv || uv venv --python 3.12 .venv) && uv pip install -e ".[dev]"

fe-install: env
	$(PNPM_INSTALL_WEB)

install: fe-install be-install

be-migrate: wait-postgres be-install
	cd $(API_DIR) && $(API_BIN)/alembic upgrade head

be-dev-seed-all: be-migrate
	@echo "Implement scripts/seed_demo.py or no-op."

be-dev: be-install
	cd $(API_DIR) && $(API_BIN)/uvicorn myapp.main:app --reload --host 0.0.0.0 --port 8000

fe-dev: fe-install
	$(PNPM) --filter @myorg/web dev

worker-dev: be-install
	@echo "No worker configured."  # e.g. cd $(API_DIR) && $(API_BIN)/python -m myapp.workers

lint:
	@$(MAKE) --no-print-directory lint-ts lint-api

test:
	@$(MAKE) --no-print-directory test-api test-web
```

---

## Documentation

| Document | Contents |
|----------|----------|
| `README.md` | Prerequisites, quick start, deprecated aliases, troubleshooting |
| `DEVELOPMENT.md` | Local workflow, env setup, common issues |
| `DEPLOYMENT.md` (app or infra repo) | Production hostnames, image names, deploy/migrate |
| `apps/api/README.md` | API-specific dev notes |
| `make help` / `make help-all` | Authoritative target list |

### Troubleshooting table (include in README)

| Symptom | Fix |
|---------|-----|
| Podman not running | `podman machine start` |
| Postgres not ready | `make wait-postgres` or `make ps` / `make logs` |
| `ENOSPC` / broken `node_modules` | `make doctor` → `make clean-node` → `pnpm store prune` |
| Port 8000 / 3000 in use | `be-dev` / `fe-dev` auto-kill; or `lsof -ti tcp:PORT \| xargs kill` |
| Stale OpenAPI client | `make openapi-export-stage` |
| Web upload CORS to MinIO | `make minio-cors` |
| Nuclear reset | `make clean` then `make install` |

---

## Agent checklist (implementation order)

1. Create `infra/compose.yml` (datastores only; optional `app` profile).
2. Create `.env.example` covering compose + host dev (`DATABASE_URL` → `localhost`, not service names).
3. Implement root `Makefile` with canonical targets, `help` / `help-all`, and deprecated aliases.
4. Add `scripts/preflight.sh` (disk + `node_modules` corruption guard).
5. Wire `be-install` (uv) and `fe-install` (pnpm filters).
6. Add seed script(s) and `be-dev-seed-*` targets.
7. Add `worker-dev` if the app has async jobs.
8. Add Dockerfiles + `.github/workflows/ci.yml` + `release.yml` (GHCR, gated publish).
9. Add `prod-images` for local smoke builds.
10. Wire the app into the infrastructure deploy repo (compose, proxy, secrets, `deploy-<app>`).
11. Update `README.md` and `DEVELOPMENT.md` quick start; document production in `DEPLOYMENT.md`.
