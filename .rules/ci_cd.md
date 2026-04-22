# CI/CD Workflow Standards — truefisical

## Purpose
This repo is deployment configuration plus a TrueNAS SCALE catalog app. CI's job is to make sure the compose files render, the catalog template stays in sync with them, the catalog metadata is valid, and no secret has leaked into a commit.

## Minimum Pipeline (GitHub Actions)
Target file: `.github/workflows/validate.yml`

### Jobs
1. **Lint compose** — `docker compose -f compose/docker-compose.yml config` with a dummy `.env` must exit 0.
2. **Render catalog template** — render `app/templates/docker-compose.yaml` with `app/values.yaml` defaults and run `docker compose config` on the output.
3. **Drift check** — diff the rendered catalog compose against `compose/docker-compose.yml` (canonical). A non-trivial diff fails the build.
4. **Catalog lint** — run iX's catalog validator against `app/` once we confirm which tool is canonical (see `.context/research.md`). Until then, skip with a TODO.
5. **Env template diff** — informational diff between `env/infisical.env.example` and `../infisical/.env.example` when the pinned Infisical version is bumped.
6. **Secret scan** — `gitleaks detect --no-git -v` against the working tree.
7. **Shell lint** — `shellcheck scripts/*.sh` if any shell scripts exist.

### Skeleton
```yaml
name: validate
on: [push, pull_request]

jobs:
  lint-compose:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Render compose with placeholder env
        run: |
          cp env/infisical.env.example .env
          docker compose -f compose/docker-compose.yml config > /dev/null

  catalog-render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Render catalog template
        run: |
          # Tool TBD — likely `helm template` or iX's catalog renderer.
          # Until confirmed, leave as a placeholder that always fails loudly.
          echo "TODO: wire up the TrueNAS catalog renderer (see .context/research.md)" && exit 1

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
```

### Pre-commit hook (recommended)
Run `gitleaks` and `docker compose config` locally via pre-commit so CI rarely catches surprises. Keep hooks fast; if a hook takes >5s, it will get disabled.

## What CI Should NOT Do
- Do not deploy to TrueNAS from CI. Installs are performed by the user through the TrueNAS UI (or a signed, deliberate release tag consumed by TrueNAS's catalog refresh).
- Do not bring up the full stack on GitHub runners unless we explicitly add a Postgres service; `compose config` and the render-diff are the cheap signals we want on every PR.
- Do not store `ENCRYPTION_KEY` or `AUTH_SECRET` in GitHub Actions secrets. If CI ever needs to bring up the stack, generate fresh throwaway keys per run.

## Release Flow
1. Bump the app version in `app/app.yaml` and update `CHANGELOG.md`.
2. Merge to `main`; CI validates.
3. Tag `v<major>.<minor>.<patch>` on `main`.
4. A release workflow (future) publishes the tagged tree as the catalog version TrueNAS users pull.

## Key Practices
- **Pin versions** for every action (`@v4`, not `@main`).
- **Fail fast:** lint and secret-scan before any heavier step.
- **Clear failure messages:** if secret scan fires, surface the file + line in the job summary.
- **Drift is a bug:** the catalog install path and the compose-paste path must stay in lockstep; CI's render-diff is what enforces it.

## Reference (if tooling is added later)
If JS/TS tooling lands in this repo (e.g., a small CLI helper), use Bun:
```yaml
- uses: oven-sh/setup-bun@v2
- run: bun install
- run: bun run biome check .
```
If Python tooling lands, use UV:
```yaml
- uses: astral-sh/setup-uv@v4
- run: uv sync --dev
- run: uv run ruff check .
```

---
*CI here is a leak detector, YAML validator, and drift check — not a deployer.*
