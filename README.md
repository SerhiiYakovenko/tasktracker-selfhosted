# TaskTracker (Self-Hosted AI Review)

A production-like task management SPA reviewed in real-time by a **self-hosted AI code reviewer** — the same TaskTracker app from Part A, now paired with your own fork of PR-Agent running as a GitHub App.

This repo is part of the **TechLeadConf 2026 workshop** *"AI-Powered Code Review"* and serves as Part B of a hands-on two-part demonstration: after seeing the out-of-the-box marketplace action in Part A, you'll understand how to self-host the same tool, extend it with custom rules, and run it on your own infrastructure. The `demo/add-search` branch contains intentional code-review teaching artifacts — realistic issues for the AI reviewer to surface.

## What this is

A **clean, full-stack app** (FastAPI backend + React/TypeScript frontend, tests, CI) paired with a **GitHub App webhook server** running a fork of PR-Agent with custom code-standard enforcement. The bot auto-reviews pull requests and responds to slash commands like `/check_standards`, which validates code against your team's conventions without leaving the PR.

**For your team:** a template for shipping your own AI code reviewer with fine-grained control — set your model, tune the prompts, deploy once, and let it scale across your repos.

**For this workshop:** a live demo of self-hosting, custom tooling, and incremental review as code changes.

## Features

- **Kanban board** — organize work in `todo`, `in_progress`, `done` columns with one-click status moves.
- **Projects & rich tasks** — title, description, priority, assignee, due date per task.
- **JWT auth** — secure backend, password hashed with bcrypt.
- **Typed end-to-end** — Pydantic v2 models mirrored by TypeScript; full OpenAPI/Swagger docs.
- **Production-shaped stack** — service layer, dependency injection, Docker, GitHub Actions CI.

## Tech stack (the app)

| Layer    | Technologies |
|----------|--------------|
| Frontend | React 18, TypeScript, Vite, React Router, CSS Modules, Vitest + Testing Library |
| Backend  | Python 3.12, FastAPI, SQLAlchemy 2.x, Pydantic v2, JWT (python-jose), bcrypt, pytest |
| Data     | SQLite (default); Postgres-ready via `DATABASE_URL` |
| Ops      | Docker, docker-compose, GitHub Actions |

### How the AI reviewer is wired

This repo's self-hosted setup runs two key components:

1. **GitHub App** — registers your fork of `qodo-ai/pr-agent` as a GitHub App (permissions: read contents, write pull requests / issues). The app receives webhooks for PR events and slash commands via the webhook server.

2. **Webhook server** — a Render-hosted (or self-hosted) service running PR-Agent's `github_app` target, listening on `POST /api/v1/github_webhooks`. On PR open/reopen, it auto-runs `/describe` and `/review`; on PR comments, it dispatches slash commands.

3. **Custom `/check_standards` command** — a hand-written tool and tuned prompt in the fork that catches team conventions (hardcoded secrets, camelCase naming, bare exceptions, missing tests). Triggers on-demand via `@bot /check_standards` or automatically on every new PR, depending on your `configuration.toml` settings.

**The flow:** You push code → PR opens → webhook fires → bot clones the repo, runs your configured tools (e.g., `/describe`, `/review`, `/check_standards`), and posts a single GitHub comment with findings. Each push triggers an incremental review. No polling, no external services scanning your code — it all runs on your server.

**Trade-off vs. marketplace:** you own the infrastructure, control the model and cost, and can add custom rules. You also run a server (even free-tier Render works for a workshop, ~USD 7/month for production). The marketplace action is instant but less flexible.

## Quick start (Docker)

```bash
docker-compose up --build
```

- **Frontend** — http://localhost:5173
- **Backend API** — http://localhost:8000
- **Swagger UI** — http://localhost:8000/docs
- **Health check** — http://localhost:8000/health

Demo account (auto-seeded):
- **Email:** `demo@tasktracker.dev`
- **Password:** `change-me`

## Local development

### Backend
```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload
```

Load demo data (optional):
```bash
python -m app.seed
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

Expects the API at `http://localhost:8000` by default. Override with `VITE_API_BASE_URL`.

## The workshop: Part B (self-hosted)

**TechLeadConf 2026 workshop:** *"AI-Powered Code Review"* — June 2, 2026, remote.

### The demo repository

This repo (`tasktracker-selfhosted`) is Part B's demo. The `demo/add-search` branch carries a feature PR with **intentionally planted review-worthy code**:

- **Backend:** hardcoded secret-like token, O(n) in-memory search instead of a DB query, raw-SQL string interpolation, bare `except:` clause, missing type hints, no tests.
- **Frontend:** `any` types, missing React list keys, unbound search keystroke (no debounce), left-in `console.log`, `dangerouslySetInnerHTML` (XSS risk).

These issues are not marked or commented. When the AI reviewer (manually or auto) analyzes the PR, it surfaces these findings in realistic PR comments — showing you exactly what a self-hosted setup can catch. The `/check_standards` command is tuned to spot conventions like hardcoded secrets and bare exceptions.

### Part B workflow (what you'll see live)

1. **Repo tour** — the fork layout: `pr_agent/tools/`, `pr_agent/settings/*.toml` (prompts), `pr_agent/servers/github_app.py` (webhook handler).
2. **Build the image** — `docker build -f docker/Dockerfile --target=github_app …` (shown via GIF, not built live).
3. **Register a GitHub App** — create the app in *Settings → Developer settings → GitHub Apps*, get the App ID and private key.
4. **Deploy to Render** — push the image to GHCR, deploy two Render Starter services (primary + backup), wire the webhook URL.
5. **Add `/check_standards`** — the custom tool + prompt are already in a branch; show the diff, redeploy.
6. **Trigger it** — open the PR on `demo/add-search`, watch the bot auto-review; comment `/check_standards` and see the custom tool reply; push a commit and see incremental review fire.

### Running the bot yourself

To run the self-hosted bot on your own repo:

1. **Fork** `qodo-ai/pr-agent` (the official PR-Agent repo).
2. **Add your custom tools** in `pr_agent/tools/` (copy the `/check_standards` example).
3. **Update the registration** in `pr_agent/agent/pr_agent.py` (add your tool to `command2class`).
4. **Build the `github_app` Docker target** and push to a registry (GHCR, Docker Hub, etc.).
5. **Register a GitHub App** in your org's settings, subscribe to PR and issue-comment events.
6. **Deploy the webhook server** (Render, AWS Lambda + webhook bridge, your own server, etc.).
7. **Set automatic triggers** in `pr_agent/settings/configuration.toml` under `[github_app]` — define which commands run on PR open, push, etc.

The bot will then auto-review every PR in that repo. Customize the prompts in `pr_agent/settings/*.toml` to match your team's style guide.

## Project structure

```
tasktracker-selfhosted/
├── README.md
├── docker-compose.yml
├── .github/workflows/ci.yml
├── backend/
│   ├── requirements.txt
│   ├── .env.example
│   ├── Dockerfile
│   └── app/
│       ├── main.py, config.py, database.py, logging_config.py, seed.py
│       ├── api/routers/  (health, auth, users, projects, tasks)
│       ├── core/security.py
│       ├── models/       (user, project, task ORM models)
│       ├── schemas/      (Pydantic request/response models)
│       ├── services/     (business logic layer)
│       └── tests/        (pytest: auth, projects, tasks, conftest fixtures)
└── frontend/
    ├── package.json, tsconfig.json, vite.config.ts, .eslintrc.cjs
    ├── Dockerfile
    └── src/
        ├── main.tsx, App.tsx, types.ts
        ├── api/client.ts      (typed fetch + JWT bearer auth)
        ├── components/        (Header, Login, TaskBoard, TaskColumn, TaskCard, TaskForm)
        ├── pages/             (LoginPage, BoardPage)
        ├── hooks/             (useAuth, useProjects, useTasks)
        ├── styles/            (CSS modules + global)
        └── __tests__/         (Vitest component tests)
```

## API summary

All endpoints versioned at `/api/v1` (except `/health`). Endpoints marked **Auth** require `Authorization: Bearer <token>`.

| Method | Path | Auth | Description |
|--------|------|:----:|-------------|
| `GET`    | `/health`                  |     | Liveness check. |
| `POST`   | `/api/v1/auth/register`    |     | Register user. |
| `POST`   | `/api/v1/auth/login`       |     | Log in, get JWT. |
| `GET`    | `/api/v1/users/me`         |  ✓  | Current user. |
| `GET`    | `/api/v1/projects`         |  ✓  | List your projects. |
| `POST`   | `/api/v1/projects`         |  ✓  | Create project. |
| `GET`    | `/api/v1/projects/{id}`    |  ✓  | Get project. |
| `PATCH`  | `/api/v1/projects/{id}`    |  ✓  | Update project. |
| `DELETE` | `/api/v1/projects/{id}`    |  ✓  | Delete project. |
| `GET`    | `/api/v1/tasks`            |  ✓  | List tasks; filter by project, status, assignee; paginate. |
| `POST`   | `/api/v1/tasks`            |  ✓  | Create task. |
| `GET`    | `/api/v1/tasks/{id}`       |  ✓  | Get task. |
| `PATCH`  | `/api/v1/tasks/{id}`       |  ✓  | Update task. |
| `DELETE` | `/api/v1/tasks/{id}`       |  ✓  | Delete task. |
| `POST`   | `/api/v1/tasks/{id}/move`  |  ✓  | Move task to new status. |

Full interactive specification at `/docs` (Swagger UI) when the backend runs.

## Testing & linting

### Backend
```bash
cd backend
ruff check .       # lint
pytest             # tests (40+ test functions)
```

### Frontend
```bash
cd frontend
npm run lint       # eslint
npm run build      # tsc + vite (type-check + bundle)
npm run test       # vitest (component tests)
```

All suites run on every push and PR via GitHub Actions (`.github/workflows/ci.yml`).

## Configuration

Backend config is environment-driven via pydantic-settings. Copy and edit:

```bash
cp backend/.env.example backend/.env
```

Key variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `APP_NAME` | `TaskTracker` | App name in OpenAPI. |
| `ENVIRONMENT` | `development` | One of development / staging / production. |
| `LOG_LEVEL` | `INFO` | Logging level. |
| `DATABASE_URL` | `sqlite:///./data/app.db` | SQLAlchemy URL; use a Postgres URL in production. |
| `SECRET_KEY` | `change-me` | JWT signing key — **set a strong value outside local dev**. |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `60` | Token lifetime. |
| `BACKEND_CORS_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | Comma-separated allowed origins. |

Frontend reads `VITE_API_BASE_URL` at build time (default `http://localhost:8000`). Vite inlines `VITE_*` into the bundle, so override it at build time:

```bash
VITE_API_BASE_URL=https://api.example.com npm run build
```

Secrets: `.env.example` files hold placeholders only. Never commit real secrets; always override `SECRET_KEY` and seed credentials outside development.

## Resources & links

- **Workshop page** — https://techleadconf.com/#workshop-ai-powered-code-review
- **PR-Agent official** — https://github.com/qodo-ai/pr-agent
- **PR-Agent docs** — https://docs.pr-agent.ai
- **TechLeadConf 2026** — https://techleadconf.com (main conference, June 11–12)
- **Workshop companion talk** — *"LLM Integration Patterns for Engineering Infrastructure"* (same author, same day, main conference track)

## License

Released under the MIT License.

---

**Recorded at TechLeadConf 2026.** This workshop and repo are public portfolio and conference evidence. Read the code, fork it, self-host the bot, and extend it for your team.

