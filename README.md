# Conduit Container

**Goal:** run the **Conduit (RealWorld)** app end-to-end locally or on a VM with a reproducible Docker setup and a single CI/CD workflow that builds, publishes, and deploys images.

**backend/** — Django REST API (production)
- RealWorld API: auth (JWT), profiles, follow/unfollow, articles CRUD, comments, favorites, tags.
- Stack: Django 1.10, DRF, PyJWT, SQLite, Gunicorn, WhiteNoise.
- Startup: **migrate**, **collectstatic**, **create superuser only if missing** (reads `.env`).
- Data: SQLite lives in a **named volume** (`dbdata`, mounted at `/data/db.sqlite3`).
- Port: **8000**, endpoints under `/api/*`.

**frontend/** — Angular SPA (production)
- RealWorld UI: feeds, tags, editor, auth, profile, favorites, settings.
- Built with Angular CLI, served by **Nginx**.
- Nginx **proxies `/api/*` to backend** to avoid CORS.
- Port: container **80** (Compose maps to **8282**).

---

## Table of Contents
- [Conduit Container](#conduit-container)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Quickstart](#quickstart)
  - [Usage](#usage)
    - [Features](#features)
    - [Environment Variables](#environment-variables)
    - [Common Compose commands](#common-compose-commands)
  - [Services](#services)
    - [backend](#backend)
    - [frontend](#frontend)
  - [Networking](#networking)
  - [CI/CD: deployment workflow](#cicd-deployment-workflow)
    - [Purpose](#purpose)
    - [What runs](#what-runs)
    - [Triggers](#triggers)
    - [Image names used by Compose](#image-names-used-by-compose)
    - [Required secrets (repo → Settings → Actions → Secrets)](#required-secrets-repo--settings--actions--secrets)
  - [License](#license)

---

## Prerequisites
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

3- Start (uses public GHCR images, no build)
```bash
docker compose pull
docker compose up -d
```

4- Verify
```bash
docker compose ps
curl -sSf http://127.0.0.1:8000/api/tags/ >/dev/null && echo "API ok"
curl -sSfI http://127.0.0.1:8282/        >/dev/null && echo "Web ok"
```

Open locally:
- **Web app:** http://127.0.0.1:8282
- **API:** http://127.0.0.1:8000/api/tags/

---

## Usage

### Features
- **Production images** from GHCR; no local builds required.
- **Idempotent backend start:** runs migrations, collects static, creates admin only if missing.
- **SQLite persistence** via named volume (`dbdata` → `/data/db.sqlite3`).
- **Nginx-served SPA** with `/api/*` proxy to `backend:8000` (no CORS hassle).
- **Single compose file** for local and VM runs.
- **Deterministic source** via Git submodules; CI builds from pinned SHAs.

### Environment Variables

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

### Common Compose commands

- Pull published images and start
```bash
docker compose pull
docker compose up -d
```
- Restart one service
```bash
docker compose up -d backend
docker compose up -d frontend
```
- Logs
```bash
docker compose logs -f backend
docker compose logs -f frontend
```
- Shell into backend
```bash
docker compose exec backend python manage.py shell
```

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

## CI/CD: deployment workflow

A single **deployment** workflow builds images, publishes them to GHCR, then deploys them to the VM with Docker Compose.

### Purpose
- Build once in CI. Deploy many times on the VM.
- Make releases traceable by tag and commit SHA.

### What runs
1) **Build & push**
   - Checks out this repo **with submodules**.
   - Builds `./backend` and `./frontend` using their `Dockerfile`s.
   - Pushes two tags per image to GHCR:
     - Moving tag: `feature-automation-workflow`
     - Immutable tag: `feature-automation-workflow-<git-sha>`
   - No `.env` data is baked into images.

2) **Deploy**
   - SSH to the VM.
   - copies the docker-compose file to the VM.
   - Clean up old containers on the VM.
   - Runs `docker compose pull` then `docker compose up -d --remove-orphans`.
   - Health checks:
     - `http://127.0.0.1:8000/api/tags/` (API)
     - `http://127.0.0.1:8282/` (Web)

### Triggers
- **Automatic on push** to branch `main` **when files under these 
- **Manual**: Actions → **deployment** → **Run workflow** on branch `main`.

### Image names used by Compose
- `ghcr.io/e1pmis/conduit-backend:latest`
- `ghcr.io/e1pmis/conduit-frontend:latest`

### Required secrets (repo → Settings → Actions → Secrets)
- `GHCR_USERNAME` = `e1pmis`
- `GHCR_TOKEN` = PAT with `read:packages` if images are private (not needed for public)
- `VM_HOST` = VM IP (e.g., `128.1.1.2`)
- `VM_USER` = SSH user (e.g., `root`)
- `VM_PORT` = `22`
- `VM_SSH_KEY` = OpenSSH private key for the VM
- `VM_DEPLOY_DIR` = repo path on the VM (e.g., `/home/root/conduit-container`)

---

## License

MIT — see [LICENSE](LICENSE) file for details.
