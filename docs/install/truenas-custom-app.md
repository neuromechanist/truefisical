# TrueNAS SCALE Custom App install

Install truefisical on TrueNAS SCALE 24.10+ by pasting `compose/docker-compose.yml` into the **Install Custom App** UI. This is the simplest way to get truefisical running on a TrueNAS box without waiting for the catalog packaging (Branch 4).

> Last smoke-tested: **PENDING** — author has not yet run an end-to-end install on TrueNAS hardware. First successful install should update this stamp with TrueNAS version, host, and the date.

## Before you start

1. **Confirm SCALE version is 24.10+ ("Electric Eel") or newer.** Older SCALE used k3s/Helm for apps and these instructions do not apply. Check via SSH:
   ```bash
   ssh skynas 'midclt call system.version'
   # Expected output starts with TrueNAS-25.04 or TrueNAS-24.10
   ```
2. **Confirm the Apps service is on and Docker is healthy.**
   ```bash
   ssh skynas 'systemctl is-active docker && docker --version && docker compose version'
   # Expected: active / Docker version 25+ / Compose v2.20+
   ```
3. **Pick a host port** for truefisical's web UI. The default is `8080`. Verify it's free:
   ```bash
   ssh skynas 'sudo ss -tlnp | grep :8080 || echo "8080 is free"'
   ```
   Avoid 80 and 443 — those are owned by the TrueNAS web UI. If 8080 is taken, pick another (e.g., 8081, 8443) and use it for `TRUEFISICAL_HOST_PORT` and `SITE_URL` below.
4. **Identify a pool to host app data.** Note the pool name (e.g., `p1`); volumes will live under TrueNAS's managed `/mnt/.ix-apps/` path by default.

## 1. Generate secrets locally

Do this on your workstation (do not type secrets into a TrueNAS UI textbox you'll paste over later — generate once, store in your password manager, paste into the UI).

```bash
echo "ENCRYPTION_KEY=$(openssl rand -hex 16)"
echo "AUTH_SECRET=$(openssl rand -base64 32)"
echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)   # MUST be URL-safe — see env/infisical.env.example"
```

Save all three to your password manager **now**, before going further. `ENCRYPTION_KEY` in particular cannot be recovered if lost; you would lose every secret stored in this instance.

## 2. Open the Custom App form

In the TrueNAS web UI:

1. **Apps → Discover Apps**
2. Click the three-dot menu (top right) → **Install via YAML** *(in some SCALE versions this is "Install Custom App")*
3. Form fields you'll fill (exact labels may vary slightly by SCALE version):
   - **Application Name:** `truefisical` (lowercase, no spaces — this becomes the compose project name `ix-truefisical`)
   - **Custom Compose Config (YAML):** paste the full contents of [`compose/docker-compose.yml`](../../compose/docker-compose.yml)
   - **Container Images Update Policy:** "Always pull image" (matches `pull_policy: always` in the backend)

## 3. Set environment variables in the UI

The compose file uses `${VAR}` interpolation for all secrets and config — TrueNAS feeds these in via its env-vars field. Set these (and only these — leave optional ones empty unless you need them):

**Required:**

| Key | Value |
|---|---|
| `ENCRYPTION_KEY` | the hex string from step 1 |
| `AUTH_SECRET` | the base64 string from step 1 |
| `POSTGRES_USER` | `infisical` |
| `POSTGRES_PASSWORD` | the hex string from step 1 |
| `POSTGRES_DB` | `infisical` |
| `SITE_URL` | `http://<truenas-host>:<port>` (e.g., `http://skynas.local:8080` or `http://192.168.x.y:8080`) |
| `TRUEFISICAL_HOST_PORT` | the port from pre-flight step 3 (e.g., `8080`) |

**Optional (skip unless you have a reason):** `LICENSE_KEY` (Infisical EE), `SMTP_*` (password reset emails), `CLIENT_*_LOGIN` (OAuth providers), `OTEL_*` / `SENTRY_DSN` (observability). See [`env/infisical.env.example`](../../env/infisical.env.example) for full notes on each.

> The compose file does not reference `DB_CONNECTION_URI` or `REDIS_URL` directly — both are derived from the values above. You can override them by setting them explicitly in the UI; otherwise they default to in-network container references that work as-is.

## 4. Install and verify

Click **Install** (or **Save**, depending on UI version). TrueNAS will:
- Pull the three pinned images (Infisical v0.159.19, Postgres 16-alpine, Redis 7-alpine).
- Bring up the stack as docker compose project `ix-truefisical`.
- Containers will be named `ix-truefisical-backend-1`, `ix-truefisical-db-1`, `ix-truefisical-redis-1`.

Watch the install progress in the Apps UI. The backend has a 30s start-period plus up to 2 minutes for Postgres migrations on first boot, so allow a few minutes before declaring failure.

Verify from your workstation:

```bash
# from the workstation
curl -s http://<truenas-host>:<port>/api/status
# expect a JSON body with "message":"Ok"
```

Then open `http://<truenas-host>:<port>` in a browser and complete the **first-run admin sign-up**. The first user becomes the instance administrator.

## 5. Where things live on TrueNAS

After install, files are under TrueNAS's app-managed paths:

```
/mnt/.ix-apps/custom_apps/truefisical/      # compose + env (managed by TrueNAS)
/mnt/.ix-apps/app_mounts/truefisical/       # named-volume backing dirs (pg_data, redis_data)
```

Containers, project name, and naming via the docker CLI:

```bash
ssh skynas 'sudo docker ps --filter "name=ix-truefisical-"'
ssh skynas 'sudo docker logs -f --tail=200 ix-truefisical-backend-1'
ssh skynas 'sudo docker exec -t ix-truefisical-db-1 pg_isready -U infisical'
```

> Backups: this is the simple Custom-App install, not a full backup setup. The runbook covering `pg_dump` automation, ZFS snapshot schedules, and restore drills lands in a follow-up branch (Branch 3 of the implementation plan). Do not put real production secrets in this install before backups are configured.

## Updating

To bump the Infisical / Postgres / Redis version:

1. **Back up Postgres first.** Without a backup, an upgrade going wrong loses all secrets. Branch 3's runbook covers this.
2. Apps → truefisical → **Edit**.
3. In the compose YAML, bump the `image:` tag (e.g., `infisical/infisical:v0.159.19` → `v0.160.0`). Watch the Infisical changelog for breaking migrations.
4. **Save.** TrueNAS pulls the new image and restarts the stack.
5. Verify `/api/status` again, and check `docker logs ix-truefisical-backend-1` for migration messages.

If anything breaks, restore Postgres from the backup taken in step 1 and revert the image tag.

## Troubleshooting

- **App stays "Deploying" or never reaches Healthy.** SSH in and tail backend logs: `ssh skynas 'sudo docker logs -f ix-truefisical-backend-1'`. The most common first-install failure is a missing or malformed env var — re-check every entry in step 3, especially that `POSTGRES_PASSWORD` contains no `/`, `+`, `@`, or `?` (it must be URL-safe, see env example).
- **"Port already allocated".** Something else on the box is using the host port. Pick a different one for `TRUEFISICAL_HOST_PORT` and update `SITE_URL` to match, then re-deploy.
- **Browser loads but shows blank or 502.** `SITE_URL` doesn't match the URL you're visiting. Update it in the UI's env vars (e.g., from `http://localhost:8080` to `http://skynas.local:8080`) and restart.
- **Anything Infisical-specific** (login flows, secret UI, API behavior) — see <https://infisical.com/docs>, not this file.

## What this install does NOT give you

- Automated backups (Branch 3 runbook).
- HTTPS / reverse proxy (you'd front this with TrueNAS's reverse proxy app or your own ingress; LAN-only via Tailscale is also a fine choice).
- One-click upgrades with rollback (Branch 4's catalog packaging).
- Pro/Enterprise Infisical features (those need a paid `LICENSE_KEY` set in step 3 — see the README for what's gated).
