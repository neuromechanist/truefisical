# Workstation development install

Run the full truefisical stack on your laptop with Docker Compose. Use this for hacking on the compose file, trying upgrades, or previewing changes before deploying to TrueNAS.

> Last smoke-tested: 2026-04-22 against Infisical v0.159.19, Postgres 16-alpine, Redis 7-alpine on macOS Darwin 25.4.0 with Docker 29.4.0 / Compose v5.1.2. Backend healthy, `/api/status` returned HTTP 200. Re-verified after the env-interpolation refactor that drops `env_file` in favor of `${VAR}` substitution.

## Prerequisites

- Docker Engine 20.10+ and Docker Compose v2 (`docker compose version` should report v2.x).
- `openssl` on `PATH` for generating secrets.
- At least 4 GB free RAM (Infisical + Postgres + Redis together).

## 1. Generate the env file

From the repo root:

```bash
cp env/infisical.env.example .env
chmod 600 .env
```

Fill in the **REQUIRED** block in `.env`. For local dev you can generate fresh secrets with:

```bash
# ENCRYPTION_KEY — 32-byte hex
openssl rand -hex 16

# AUTH_SECRET — 32-byte base64 (used as opaque JWT signing secret, not in URLs)
openssl rand -base64 32

# POSTGRES_PASSWORD — must be URL-safe (no /, +, @, ?) since it's embedded
# into DB_CONNECTION_URI without encoding. Hex is the simplest safe choice:
openssl rand -hex 24
```

Paste each generated value into the matching line in `.env`. Leave `SITE_URL=http://localhost:8080` for local dev.

> **Save the `ENCRYPTION_KEY` outside this directory too** (password manager, printed copy). If you lose it, every secret you store becomes unrecoverable. For a workstation-only sandbox that won't hold real data this is less critical — but make the habit now, not after you start putting real secrets in.

## 2. Start the stack

```bash
docker compose -f compose/docker-compose.yml --env-file .env up -d
```

This pulls the pinned Infisical, Postgres, and Redis images and brings up three containers under the compose project name `compose` (taken from the directory name): `compose-backend-1`, `compose-db-1`, `compose-redis-1`. To use a friendlier project name, add `-p truefisical` to every compose command.

## 3. Wait for health, then verify

```bash
# Backend has a healthcheck on /api/status — wait for it to report healthy.
docker compose -f compose/docker-compose.yml --env-file .env ps

# Hit the status endpoint directly once the backend is healthy.
curl -s http://localhost:8080/api/status
```

Expect a JSON response (no error body). Then open <http://localhost:8080> in your browser and complete the first-run admin sign-up flow — the first user created becomes the instance admin.

## 4. Tear down

```bash
# Stop containers, keep volumes (data persists across restarts).
docker compose -f compose/docker-compose.yml --env-file .env down

# Stop containers AND delete volumes (wipes the Postgres database — only do
# this in a scratch workstation install, never against a real one).
docker compose -f compose/docker-compose.yml --env-file .env down -v
```

## Common tweaks

**Run on a different host port:** set `TRUEFISICAL_HOST_PORT=9000` in `.env` and update `SITE_URL=http://localhost:9000` to match. Restart with `docker compose … up -d`.

**View logs:**
```bash
docker compose -f compose/docker-compose.yml --env-file .env logs -f backend
docker compose -f compose/docker-compose.yml --env-file .env logs -f db
```

**Reset the whole thing:** `docker compose … down -v` then start fresh from step 2. You'll need to redo the admin sign-up.

## Troubleshooting

- **Backend container keeps restarting.** Almost always a missing or malformed env var. Check `docker compose … logs backend` for the message; re-check the REQUIRED block in `.env`.
- **`/api/status` returns nothing / connection refused.** The backend healthcheck waits up to ~2 minutes for Postgres migrations. Run `docker compose … ps` and look at the `HEALTH` column. Only try `curl` once backend reports `healthy`.
- **Postgres healthcheck failing.** Confirm `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` in `.env` are set and that `POSTGRES_PASSWORD` is URL-safe (no `/`, `+`, `@`, `?`). The backend's `DB_CONNECTION_URI` is composed from these three values via compose interpolation, so a bad password breaks both Postgres and the backend.
- **Port 8080 already taken.** Set `TRUEFISICAL_HOST_PORT` to something else (see "Common tweaks").

For anything Infisical-specific (login errors, secret UI behavior), see the upstream docs at <https://infisical.com/docs> rather than this file.
