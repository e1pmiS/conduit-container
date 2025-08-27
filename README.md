# Conduit Container — README

Monorepo with two Git submodules:
- **backend/**: Django REST API (Django 1.10, SQLite)
- **frontend/**: Angular client (Angular CLI dev server)

Orchestration via **Docker Compose**.

---

## Table of Contents
- [Conduit Container — README](#conduit-container--readme)
  - [Table of Contents](#table-of-contents)
  - [Description](#description)
  - [Features](#features)
  - [Prerequisits](#prerequisits)
  - [Environment Variables](#environment-variables)
  - [Quickstart](#quickstart)
  - [Usage](#usage)
    - [Common Compose commands](#common-compose-commands)
    - [Backend behavior](#backend-behavior)
    - [Frontend behavior](#frontend-behavior)
  - [Services](#services)
    - [backend](#backend)
    - [frontend](#frontend)
  - [Networking](#networking)
  - [License](#license)

---

## Description

**Goal:** provide a self-contained, reproducible environment to run the **Conduit (RealWorld)** app end‑to‑end on locally or in cloud. The repo standardizes setup, hides legacy toolchains, and lets you start building or testing immediately.

**backend/** — Django REST API
- Implements the RealWorld API spec: user registration and login (JWT), profiles and follow/unfollow, articles CRUD, comments, favorites, and tag discovery.
- Tech stack: Django 1.10, Django REST Framework, PyJWT, SQLite.
- Startup behavior: on container start, it **applies migrations** and **creates/updates a superuser** from `.env`.
- Persistence: SQLite file bind-mounted from the host for simple, zero‑ops storage.
- External interface: exposes HTTP on port **8000**; all endpoints are under `/api/*` (e.g., `/api/users/login`, `/api/articles`, `/api/tags`).

**frontend/** — Angular Single‑Page App
- Implements the RealWorld UI: home feed, global feed, tag filtering, article page, editor, auth pages, profile, favorites, and settings.
- Tech stack: Angular CLI dev server for development; the app calls the API through **same‑origin `/api/*`** paths.
- Proxy: the dev server proxies `/api` to the backend service, removing CORS complexity.
- External interface: serves the SPA on container port **4200** (mapped to **8282** by Compose).

---

## Features
- One‑command startup with Docker Compose.
- Automatic DB migrations on backend start.
- Automatic bootstrap of a Django superuser from `.env`.
- Angular dev server with live rebuild and API proxy.
- SQLite persistence via bind mount (`backend/db.sqlite3`).

---

## Prerequisits
- Docker Engine 24+
- Docker Compose v2

---

## Environment Variables

Place **`.env`** in the **repo root** (same folder as `docker-compose.yml`). Keep **`example.env`** there as a committed template and **do not commit** your real `.env`.

**example.env**
```env
# --- Django settings ---
DJANGO_SETTINGS_MODULE=conduit.settings
DJANGO_SECRET_KEY=REPLACE_WITH_A_RANDOM_50_CHAR_STRING

# Hosts and CSRF (include your VM IP if used)
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

# Admin bootstrap (created/updated by backend entrypoint)
DJANGO_ADMIN_EMAIL=admin@example.com
DJANGO_ADMIN_USERNAME=admin
DJANGO_ADMIN_PASSWORD=ChangeMe123!
```

Create your real `.env`:
```bash
cp example.env .env
```
then edit .env and set real secrets (SECRET_KEY, admin password, etc.)

Notes:
- `DJANGO_SECRET_KEY` is required beyond localhost.
- `DJANGO_ALLOWED_HOSTS` must include your VM IP if you want to host it.

---

## Quickstart

1- Clone and pull submodules

```bash
git clone <your-main-repo>
cd <your-main-repo>
git submodule update --init --recursive
```

2- Prepare env
edit .env and set DJANGO_SECRET_KEY and other values

```bash
cp example.env .env
```

 3- Prepare SQLite file (avoid a directory bind)

```bash
touch backend/db.sqlite3
chmod 666 backend/db.sqlite3
```

4- Build and start
```bash
docker compose up -d --build
```

5- Verify
```bash
docker compose ps
docker compose logs --tail=50 backend
docker compose logs --tail=50 frontend
```

Open locally:
- **Web app:** http://127.0.0.1:8282
- **API:** http://127.0.0.1:8000/api/tags/

---

## Usage

### Common Compose commands
```bash
# start or rebuild a single service
docker compose up -d --build backend
docker compose up -d --build frontend

# follow logs
docker compose logs -f backend
docker compose logs -f frontend

# exec into backend
docker compose exec backend python manage.py shell
```

### Backend behavior
- `backend/entrypoint.sh` runs `migrate` and ensures a superuser from `.env` every start.
- Data persists via bind mount: `./backend/db.sqlite3:/app/db.sqlite3`.

### Frontend behavior
- `frontend/Dockerfile.dev` serves the Angular app on container port **4200**.
- `frontend/proxy.compose.json` proxies `/api` to `http://backend:8000` so the browser uses same‑origin `/api/*`.

---

## Services

### backend
- **Build context:** `./backend`
- **Exposes:** `8000/tcp` (mapped to host `8000` by Compose)
- **Responsibilities:** Run Django, apply migrations, create/update admin user from `.env`.
- **Persistence:** `./backend/db.sqlite3` bind‑mounted to `/app/db.sqlite3`.

### frontend
- **Build context:** `./frontend`
- **Exposes:** `4200/tcp` (mapped to host `8282` by Compose)
- **Responsibilities:** Serve Angular SPA with live reload and API proxy to backend.

---

## Networking

- Compose network provides service discovery by name.
- Frontend calls the API via **`/api/*`**, which the dev server proxies to `http://backend:8000`.
- External access:
  - API → `http://<host-or-vm-ip>:8000`
  - Web → `http://<host-or-vm-ip>:8282`

---

## License

MIT — see [LICENSE](LICENSE) file for details.
