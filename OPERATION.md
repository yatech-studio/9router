# 9Router — Docker Compose Operations

Run 9Router with Docker Compose from the repository root.

- **Dashboard:** http://localhost:20128
- **Default port:** `20128`
- **Compose files:** `docker-compose.yml`, `docker-compose.headroom.yml` (optional)

For plain `docker run` and published images, see [DOCKER.md](./DOCKER.md).

---

## Prerequisites

- Docker Engine 20.10+ with Compose v2 (`docker compose`)
- Git clone of this repository

---

## Quick start

```bash
# Optional: copy and edit secrets before first run
cp .env.example .env

# Build image and start in the background
docker compose up -d --build

# Or use the helper script
./start.sh
```

Open http://localhost:20128 and log in. If `INITIAL_PASSWORD` is not set in `.env`, the default password is `123456` until you change it in the dashboard.

---

## What the stack runs

| Item | Value |
|------|-------|
| Service name | `app` |
| Container name | `9router` |
| Image (local build) | `9router:local` |
| Published port | `20128` → container `20128` |
| Data volume | `9router-data` → `/app/data` |
| Health check | `GET /api/health` |

Compose sets these container variables automatically:

```text
NODE_ENV=production
PORT=20128
HOSTNAME=0.0.0.0
DATA_DIR=/app/data
```

Additional variables are loaded from `.env` when that file exists (`required: false`).

---

## Environment variables

Copy `.env.example` to `.env` and adjust for production:

```bash
cp .env.example .env
```

Important values for Docker:

```bash
JWT_SECRET=change-me-to-a-long-random-secret
INITIAL_PASSWORD=change-me
DATA_DIR=/app/data          # must match the container mount target
PORT=20128
BASE_URL=http://localhost:20128
NEXT_PUBLIC_BASE_URL=http://localhost:20128
```

Notes:

- If `JWT_SECRET` is omitted, the app generates one on first run and stores it under `$DATA_DIR/jwt-secret`.
- Set `DATA_DIR=/app/data` in `.env` when using the default Compose volume (not `/var/lib/9router` from the example file).
- Outbound proxy vars (`HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`) are supported; see `.env.example`.

---

## Data persistence

Compose uses a named Docker volume:

```yaml
volumes:
  - 9router-data:/app/data
```

Data layout inside the volume:

```text
/app/data/
├── db/
│   ├── data.sqlite       # main SQLite database
│   └── backups/          # auto backups
├── jwt-secret            # auto-generated if JWT_SECRET not set
└── ...                   # certs, logs, runtime configs
```

Inspect the volume:

```bash
docker volume inspect 9router_9router-data
```

### Bind mount instead of a named volume

Replace the volume entry in `docker-compose.yml`:

```yaml
volumes:
  - ./data:/app/data
# or
  - "$HOME/.9router:/app/data"
```

Keep `DATA_DIR=/app/data` in the service environment.

---

## Day-to-day commands

```bash
# Start (reuse existing image)
docker compose up -d

# Rebuild after code changes
docker compose up -d --build

# Follow logs
docker compose logs -f app

# Stop containers (keep data volume)
docker compose down

# Stop and delete the data volume (destructive)
docker compose down -v

# Restart
docker compose restart app

# Shell inside the container
docker compose exec app sh
```

---

## Optional Headroom sidecar

Headroom is not bundled in the 9Router image. Enable it with the overlay compose file:

```bash
docker compose -f docker-compose.yml -f docker-compose.headroom.yml up -d --build
```

This starts:

| Service | Container | Port |
|---------|-----------|------|
| `app` | `9router` | `20128` |
| `headroom` | `9router-headroom` | `8787` |

In the dashboard: **Endpoint → Token Saver → Headroom** → set URL to `http://headroom:8787`, recheck status, then enable.

If Headroom runs on the Docker host instead of as a sidecar:

- **macOS / Windows:** `http://host.docker.internal:8787`
- **Linux:** add to the `app` service in compose:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

---

## Updating

### Local build (this repo)

```bash
git pull
docker compose up -d --build
```

### Published image (no local build)

Replace the `build:` block in `docker-compose.yml` with:

```yaml
image: decolua/9router:latest
```

Then:

```bash
docker compose pull
docker compose up -d
```

---

## Troubleshooting

**Container exits immediately**

```bash
docker compose logs app
```

Check `.env` for invalid values and ensure port `20128` is free on the host.

**Health check failing**

Wait for the 60s start period, then:

```bash
curl http://localhost:20128/api/health
docker compose ps
```

**Reset everything (fresh install)**

```bash
docker compose down -v
docker compose up -d --build
```

**Permission errors on volume (Linux)**

The image entrypoint runs the app as the `node` user and fixes ownership on `/app/data` at startup. If bind-mounting a host directory, ensure it is writable or pre-create it with appropriate ownership.

---

## Related docs

- [DOCKER.md](./DOCKER.md) — `docker run`, published images, CI publish flow
- [.env.example](./.env.example) — full environment variable reference
