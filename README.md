# Conduit Container

**Goal:** provide a self-contained, reproducible environment to run the **Conduit (RealWorld)** app end-to-end locally or on a VM. The repo standardizes setup, hides legacy toolchains, and lets you start building or testing immediately.

**backend/** — Django REST API (production)
- Implements the RealWorld API spec: user registration and login (JWT), profiles and follow/unfollow, articles CRUD, comments, favorites, and tag discovery.
- Tech stack: Django 1.10, Django REST Framework, PyJWT, SQLite, Gunicorn, WhiteNoise.
- Startup behavior: on container start it **applies migrations**, **collects static**, and **creates a superuser only if missing** from `.env` (no password overwrite).
- Persistence: SQLite stored in a **named volume** (Compose volume `dbdata`, mounted at `/data/db.sqlite3`).
- External interface: HTTP on **8000**; endpoints under `/api/*` (e.g., `/api/users/login`, `/api/articles`, `/api/tags`).

**frontend/** — Angular Single-Page App (production)
- Implements the RealWorld UI: home feed, global feed, tags, article page, editor, auth pages, profile, favorites, settings.
- Built with Angular CLI, **served by Nginx**. Nginx **proxies `/api/*` to the backend** service to avoid CORS.
- External interface: container listens on **80**, mapped to host **8282** by Compose.


---

## Table of Content
- [Conduit Container](conduit-container)
  - [Table of Contents](#table-of-contents)
  - [Prerequisits](#prerequisits)
  - [Quickstart](#quickstart)
  - [Environment Variables](#environment-variables)
  - [Features](#features)
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

## Prerequisits
- Docker Engine 24+
- Docker Compose v2
  
---

## Quickstart

1- Clone and pull submodules

```bash
git clone https://github.com/e1pmiS/conduit-container.git
cd conduit-container
git submodule update --init --recursive
```

2- Prepare env
edit .env and set DJANGO_SECRET_KEY and other values

```bash
cp example.env .env
```

3- Build and start
```bash
docker compose up -d --build
```

4- Verify
```bash
docker compose ps
docker compose logs --tail=50 backend
docker compose logs --tail=50 frontend
```

Open locally:
- **Web app:** http://127.0.0.1:8282
- **API:** http://127.0.0.1:8000/api/tags/

---

## Features
- One-command startup with Docker Compose.
- Production backend via Gunicorn and WhiteNoise.
- Automatic DB migrations and static collection on start.
- Named volume for SQLite persistence.
- Production Angular build served by Nginx, /api/* proxied to backend.

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
- Database path comes from env/compose (/data/db.sqlite3) and lives in the dbdata named volume.

### Frontend behavior
- `frontend/Dockerfile.dev` builds the Angular app for production.
- `frontend/nginx.conf` serves the SPA and proxies /api/* to backend:8000.

---

## Services

### backend
- **Build context:** `./backend`
- **Exposes:** `8000/tcp` (mapped to host `8000` by Compose)
- **Responsibilities:** Run Django via Gunicorn, apply migrations, collect static, bootstrap admin if needed.
- **volumes:**  Named volume dbdata mounted at /data (SQLite at /data/db.sqlite3).

### frontend
- **Build context:** `./frontend`
- **Exposes:** 80/tcp (mapped to host 8282)
- **Responsibilities:** Serve Angular SPA via Nginx and proxy /api/* to backend.
- **volumes:** frontend_cache for /var/cache/nginx.

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
