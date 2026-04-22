# Testing Standards — truefisical

## Core Philosophy: Test Reality, Not Fiction
This repo packages a self-hosted Infisical stack as a TrueNAS SCALE app. "Tests" here mean **real compose-up smoke checks** on a workstation, then **real install-and-boot checks on TrueNAS**. A passing smoke test proves the stack actually boots and serves traffic — a mocked test would prove nothing useful.

## [STRICT] NO MOCKS
- No mock containers, fake Postgres, or simulated Infisical responses.
- If you cannot run the real stack, do not write a test — open an issue about the missing environment instead.
- HTTP recordings / fixtures are acceptable only for testing client code that calls a remote HTTP API, not for testing the stack itself.

## Smoke-Test Levels

### Level 1 — workstation compose smoke (required for every PR touching `compose/`, `env/`, `proxy/`, or `app/templates/`)
```bash
# 1. Validate compose syntax + env substitution
docker compose -f compose/docker-compose.yml config > /tmp/rendered.yml

# 2. Bring the stack up in an isolated project name
docker compose -p truefisical-smoke -f compose/docker-compose.yml up -d

# 3. Wait for backend health
until curl -fsS http://localhost:<port>/api/status; do sleep 2; done

# 4. Tear down, including volumes (smoke tests must be ephemeral)
docker compose -p truefisical-smoke -f compose/docker-compose.yml down -v
```

### Level 2 — TrueNAS Custom App smoke (required before tagging a release)
1. Rsync the repo to `skynas`.
2. TrueNAS UI → Apps → Discover Apps → **Install Custom App**.
3. Paste `compose/docker-compose.yml`; set volumes to a scratch dataset, e.g., `/mnt/<pool>/apps/truefisical-smoke/`.
4. Verify `/api/status` from LAN; log in as first-run admin.
5. Delete the scratch app + dataset.

### Level 3 — Catalog install smoke (required when `app/` changes)
1. Push the branch; on `skynas`, add the repo+branch as an additional catalog.
2. Install `truefisical` from the catalog with default answers.
3. Verify the rendered compose matches the canonical one (CI should already enforce this, but re-confirm on real hardware).
4. Verify login + `/api/status`.
5. Uninstall.

Record the exact command / UI steps used in the PR description under "Test plan".

## What NOT to Smoke-Test Against the Primary TrueNAS App
- Anything that involves `down -v` or dropping volumes — only in a scratch app, never the real one.
- Image bumps without a verified Postgres backup in the last hour.
- Changes to `ENCRYPTION_KEY` or `AUTH_SECRET` — these require the dedicated rotation procedure, not a smoke test.

## Before Merging
Ask yourself:
- Did I actually bring the stack up, or just run `compose config`?
- If this change hit prod as-is, what would break? Did the smoke test exercise that path?
- Will the PR description tell future-me (or a stranger) which exact command to rerun to reproduce?

## When Testing Really Is Impossible
If a change cannot be smoke-tested (e.g., you're only editing docs), say so explicitly in the PR. Silence is not evidence.

---
*Real compose-up beats any amount of YAML review. Real TrueNAS install beats any amount of rendered compose.*
