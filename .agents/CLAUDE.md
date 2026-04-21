# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This repo is a thin shell around **two git submodules** ([.gitmodules](.gitmodules)):

- [frontend/](frontend/) → `Horarios-Automaticos-Frontend` (React + Vite + Tauri 2 desktop/Android wrapper)
- [backend/](backend/) → `Horarios-Automaticos-Backend` (FastAPI + async SQLAlchemy + PostgreSQL)

The root owns only [docker-compose.yml](docker-compose.yml), [.env.example](.env.example), [Jenkinsfile](Jenkinsfile), [docs/](docs/), and [infra/](infra/). When changing app code, remember commits need to go into the submodule repo — a commit in the root only updates the submodule pointer.

After pulling the root, always run `git submodule update --init --recursive` before building.

## Commands

### Full stack (root)
```bash
docker compose up --build          # frontend :3000, backend :8080, postgres :5432
docker compose down -v             # wipe DB volume
```

### Backend ([backend/](backend/))
Uses [`uv`](https://docs.astral.sh/uv/) — not pip/poetry. Python 3.12+.
```bash
uv sync --extra dev                           # install deps (creates .venv)
uv run uvicorn src.main:app --reload --port 8080
uv run alembic upgrade head                   # apply migrations
uv run alembic revision --autogenerate -m "msg"
uv run pytest                                 # all tests
uv run pytest tests/test_generation.py -v     # single file
uv run pytest tests/test_generation.py::test_name   # single test
```
Note: [backend/main.py](backend/main.py) is a stub — the real app entrypoint is [src/main.py](backend/src/main.py). Prod container uses [backend/entrypoint.sh](backend/entrypoint.sh) which runs `alembic upgrade head` before starting uvicorn.

### Frontend ([frontend/](frontend/))
```bash
npm install
npm run dev          # vite on :5173 (strict port — Tauri expects it)
npm run build
npm run lint         # eslint
npm test             # vitest run (unit)
npm run test:watch
npm run test:e2e     # playwright against :5173 (auto-starts `npm run dev`)
npm run tauri:dev    # desktop dev build (native window)
npm run tauri:build  # produce installers
```

## Backend architecture

FastAPI app composed of feature routers wired in [src/main.py](backend/src/main.py):
`auth`, `members`, `shift_types`, `preferences`, `schedule_periods`, `assignments`, `generation`, `export`.

Layering inside [backend/src/](backend/src/):
- `routes/` — thin HTTP handlers, validation, auth dep injection
- `services/` — business logic; each route module has a matching service
- `models/` — SQLAlchemy 2.x async ORM (see [core/database.py](backend/src/core/database.py), all inherit `Base`)
- `schemas/` — Pydantic v2 request/response DTOs
- `core/` — `config` (pydantic-settings, reads `.env`), `database`, `security` (JWT + bcrypt), `deps` (FastAPI DI), `rate_limit` (slowapi), `logging`

**Schedule generation** ([src/services/generator/](backend/src/services/generator/)) is the domain core. [engine.py](backend/src/services/generator/engine.py) loads members/shifts/existing assignments/preferences into a `GenerationContext` ([base.py](backend/src/services/generator/base.py)), then dispatches to one of three strategies: `balanced`, `coverage`, `conservative` (registered in `STRATEGIES` dict). Each strategy returns `ProposedAssignment` objects which the engine persists and logs into `GenerationRun`.

**Database URL normalization**: [core/config.py](backend/src/core/config.py) rewrites `postgresql://` → `postgresql+psycopg://` automatically because Railway provides the former. Do not hand-write the async driver prefix in `.env`.

**Migrations**: Alembic lives at [backend/alembic/](backend/alembic/) (not [infra/db/migrations/](infra/db/migrations/) despite the scaffold README). Versions folder: [backend/alembic/versions/](backend/alembic/versions/).

## Frontend architecture

React 19 + React Router 7 SPA wrapped by Tauri 2 for desktop/Android.

[App.tsx](frontend/src/App.tsx) sets the provider stack: `QueryClientProvider` → `TooltipProvider` → `ToastProvider` → `ErrorBoundary` → `UpdateChecker` → `BrowserRouter`. Auth-gated routes sit inside `ProtectedRoute`; only two pages exist: [LoginPage](frontend/src/pages/LoginPage.tsx) and [DashboardPage](frontend/src/pages/DashboardPage.tsx).

State split: **TanStack Query** for server state (hooks per resource in [src/api/](frontend/src/api/)), **Zustand** for local/auth (see [authStore.ts](frontend/src/stores/authStore.ts)). `@` alias maps to `./src` — use it for all internal imports ([vite.config.ts](frontend/vite.config.ts)).

Component organization in [src/components/](frontend/src/components/): top-level files are page-level widgets; subfolders group by feature (`calendar/`, `config/`, `dashboard/`, `drag/`) and `ui/` holds the Radix-wrapped primitives.

## Testing conventions

**Backend tests hit a real PostgreSQL** — [tests/conftest.py](backend/tests/conftest.py) points at `postgresql+psycopg://appuser:changeme@localhost:5432/appdb` (override via `TEST_DATABASE_URL`) and `TRUNCATE`s a fixed list of tables after every test. No SQLite fallback, no mocked DB. Use the `client`/`auth_client`/`seed_user`/`db` fixtures; `pytest-asyncio` is in `auto` mode so tests can be plain `async def`.

When adding a new table, append it to `TABLES_TO_CLEAN` in [conftest.py](backend/tests/conftest.py) (FK order matters — children before parents).

Frontend has two test runners: **Vitest** (`.test.ts(x)` under `__tests__/` folders, jsdom env, setup in [src/test/setup.ts](frontend/src/test/setup.ts)) and **Playwright** ([e2e/](frontend/e2e/)). Playwright is excluded from the Vitest run via [vite.config.ts](frontend/vite.config.ts).

## Deploy

Jenkins ([Jenkinsfile](Jenkinsfile)) builds both apps and deploys each to Railway as separate services (`backend`, `frontend`) on `main`. Backend's Railway entrypoint is [backend/entrypoint.sh](backend/entrypoint.sh) (runs migrations on boot). Per-service `railway.toml` / `docker-compose.yml` also exist inside each submodule for standalone deploys.

## Project rules (from [docs/agents.md](docs/agents.md))

- Never commit secrets — `.env` is gitignored; update `.env.example` when adding new vars.
- Every public API route needs an integration test.
- Docker Compose is the source of truth for local dev.
