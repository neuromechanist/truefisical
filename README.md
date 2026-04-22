# truefisical

Self-hosted [Infisical](https://infisical.com) packaged as a **TrueNAS SCALE app**, so your projects can pull secrets from one authoritative store that lives on hardware you control.

> **Status:** work-in-progress. See [`.context/plan.md`](.context/plan.md) for the current phase.

---

## TL;DR — what you miss by not paying

Self-hosting Infisical **does not unlock paid features**. Infisical is **open core**, not fully open source:

- The **Free-tier code** is MIT-licensed open source (see `LICENSE` in upstream).
- The **Pro and Enterprise code** lives under `backend/src/ee/` in the upstream repo and is under the **Infisical Enterprise License** — *source-available* (you can read and modify it), but only usable in production with a paid Infisical Enterprise subscription. It's also runtime-gated by a `LICENSE_KEY` env var that validates against Infisical's license server.

So to use any Pro or Enterprise feature (RBAC, secret versioning, SAML SSO, dynamic secrets, SCIM, audit log retention, approval workflows, etc.), you need to buy a `LICENSE_KEY` from Infisical and set it as an environment variable — whether you self-host or use Cloud.

So the real trade is:

- **Cloud Free vs. truefisical (no license):** same feature set. truefisical wins on data-sovereignty; Cloud wins on ops.
- **Cloud Pro ($18/mo/identity) vs. truefisical + Pro license:** same features. Self-hosting becomes a cost question.
- **Cloud Enterprise vs. truefisical + Enterprise license:** Cloud adds the SLA, SOC 2 posture, and dedicated infra; self-hosted gets the same code features.

Sources: <https://infisical.com/pricing> and <https://infisical.com/docs/self-hosting/ee>.

---

## Is this for you?

Use **truefisical** if:
- You already run (or are happy to run) a TrueNAS SCALE box (24.10+, Docker-native apps).
- You want your secrets on your own hardware, behind your LAN, backed up by your own ZFS snapshots.
- You're OK living on Free-tier features, OR you're willing to buy an Infisical EE license and apply it to your own instance.

Use **Infisical Cloud** instead if:
- You don't want to be the SRE for your secrets store.
- You need Infisical's SLA or SOC 2 attestation covering the hosted service.
- You'd rather pay per identity than maintain a NAS.

Stay on **local `.env` files** if:
- You have one developer, one machine, no CI, and no cross-project secrets.

---

## Feature matrix (what's in each tier)

Every feature is available in **both** Infisical Cloud **and** self-hosted Infisical — the question is just which tier gates it. On self-hosted, anything above Free requires a `LICENSE_KEY`.

| Capability | Local `.env` | Free (Cloud or self-hosted) | Pro (Cloud $18/mo/identity or self-hosted + Pro license) | Enterprise (Cloud custom or self-hosted + Enterprise license) |
|---|---|---|---|---|
| **Dashboard UI, API, CLI, SDKs** | — | Yes | Yes | Yes |
| **Kubernetes Operator** | — | Yes | Yes | Yes |
| **Infisical Agent** | — | Yes | Yes | Yes |
| **Webhooks** | — | Yes | Yes | Yes |
| **2FA** | — | Yes | Yes | Yes |
| **Self-hosting allowed** | n/a | Yes | Yes (requires license) | Yes (requires license) |
| **All integrations** (AWS, Vercel, GitHub Actions, GitLab CI/CD, Jenkins, Ansible, …) | Manual copy-paste | Yes | Yes | Yes |
| **Secret Referencing & Overrides** | — | Yes | Yes | Yes |
| **Secret Scanning & Leak Prevention** | — | Yes | Yes | Yes |
| **Secret Sharing** | — | Yes | Yes | Yes |
| **Community Support (Slack)** | — | Yes | Yes | Yes |
| **Secret Versioning** | Git, if you commit them (don't) | — | Yes | Yes |
| **Point-in-Time Recovery** | — | — | Yes | Yes |
| **Role-Based Access Controls (RBAC)** | File perms | — | Yes | Yes |
| **Secret Rotation** | Manual | — | Yes | Yes |
| **Temporary Access Provisioning** | — | — | Yes | Yes |
| **SAML SSO** | — | — | Yes | Yes |
| **IP Allowlisting** | — | — | Yes | Yes |
| **90-day Audit Log Retention** | — | — | Yes | Yes (customisable on EE) |
| **Higher Rate Limits** | n/a | — | Yes | Yes (customisable on EE) |
| **Priority Customer Support** | — | — | Yes | Yes (dedicated engineer on EE) |
| **Dynamic Secrets** (DB creds, AWS STS, …) | — | — | — | Yes |
| **Approval Workflows / Access Requests** | — | — | — | Yes |
| **LDAP Authentication** | — | — | — | Yes |
| **Enterprise SCIM** | — | — | — | Yes |
| **User Groups / Custom Roles** | — | — | — | Yes |
| **Sub-organizations** | — | — | — | Yes |
| **KMIP / KMS & HSM Support** | — | — | — | Yes |
| **Gateways** | — | — | — | Yes |
| **Audit Log Streaming** | — | — | — | Yes |
| **AI Security Advisor** | — | — | — | Yes |

*Last verified: 2026-04-22 against <https://infisical.com/pricing>. If the pricing page moves features between tiers, re-verify before tagging a truefisical release.*

---

## Operational differences at the same tier

These are the differences that **don't** go away by buying a license — they're inherent to hosted vs. self-hosted.

| Concern | Infisical Cloud | Self-hosted truefisical |
|---|---|---|
| **Where secrets live** | Infisical's infrastructure (AWS) | Your TrueNAS pool |
| **Who runs the SRE function** | Infisical | You |
| **Backups** | Infisical's job | Your job (`pg_dump` + ZFS snapshots; see runbook) |
| **Disaster recovery** | Infisical's job | Your job (replicate dataset, keep `ENCRYPTION_KEY` offline) |
| **Upgrades** | Automatic | You trigger them via the catalog when ready |
| **Outage blast radius** | Affects all Infisical customers | Affects only you |
| **Works offline / on LAN** | No | Yes |
| **SLA on the service** | Yes on Pro/Enterprise | Whatever your NAS delivers |
| **SOC 2 attestation of the service** | Yes (covers Infisical's Cloud) | **Not applicable** — attestation is of Infisical's Cloud, not your install |
| **PenTest reports of the service** | Yes on Enterprise | — |
| **Telemetry to Infisical** | Default on | Default on; opt-out via env var |
| **Data sovereignty** | Shared infra | Your hardware |
| **Per-seat cost growth** | Yes ($18/mo/identity on Pro) | No (flat; license is org-level) |

---

## What you give up by self-hosting

Writing this honestly so future-you doesn't feel misled.

- **Reliability is now yours.** Infisical Cloud has an SRE team. Your NAS has you. If the pool goes down at 2am, nothing rotates to fix it.
- **Paid features still cost money, just differently.** Self-hosting is not a way around the paid tiers. RBAC, secret versioning, SSO, dynamic secrets, etc. still require a `LICENSE_KEY` from Infisical. The trade is **per-seat Cloud subscription** vs. **one enterprise license + your hardware/time**. Do the math for your team size before assuming self-hosting is cheaper.
- **You won't get their roadmap for free.** New features land on Cloud first. You'll be on whatever Infisical version you pinned last, until you deliberately bump.
- **No compliance attestation by association.** Running truefisical does not give your org Infisical's SOC 2. Your secrets store is only as compliant as your TrueNAS is.
- **Integrations still take maintenance.** GitHub, Vercel, k8s integrations are "free" to use, but if upstream APIs change, you apply patches on your own cadence.

## What self-hosted gets you that Cloud can't

- Secrets physically on your hardware, behind your LAN.
- Works when the internet is down.
- No per-identity cost as your fleet of machine identities grows.
- You control retention, backup strategy, and blast radius.
- You can audit the entire supply chain (image, DB, network).

---

## Install

See [`docs/install/`](docs/install/) (WIP):

- [`truenas-catalog.md`](docs/install/truenas-catalog.md) — primary path: add our catalog, install with a form.
- [`truenas-custom-app.md`](docs/install/truenas-custom-app.md) — fallback: paste the compose file into TrueNAS's Custom App UI.
- [`workstation-dev.md`](docs/install/workstation-dev.md) — run it locally with `docker compose` for development.

## Operating truefisical

See [`docs/operate/`](docs/operate/) (WIP):

- [`runbook.md`](docs/operate/runbook.md) — upgrade, backup, restore, key rotation, applying an EE `LICENSE_KEY`.
- [`troubleshooting.md`](docs/operate/troubleshooting.md) — TrueNAS + Infisical gotchas.

## Using Infisical itself

We deliberately do not re-document Infisical. Point yourself at the upstream:

- Docs: <https://infisical.com/docs>
- CLI: <https://infisical.com/docs/cli/overview>
- SDKs: <https://infisical.com/docs/sdks/overview>
- Self-hosting configuration (env vars): <https://infisical.com/docs/self-hosting/configuration/envars>
- Enterprise licensing: <https://infisical.com/docs/self-hosting/ee>

## Relationship to upstream Infisical

`truefisical` packages Infisical — it does not fork it. We pin a specific Infisical image tag, wrap it in a TrueNAS catalog app, and ship TrueNAS-specific operational docs. All credit for Infisical itself goes to <https://github.com/Infisical/infisical>. An Infisical `LICENSE_KEY` you purchase works with this package exactly as it does with any other self-hosted install.

**What this project won't do:** we will not reimplement Infisical's Enterprise-tier features as OSS, ship patches that bypass the license check, or host a rebuilt Infisical image. Open core is the business model that keeps Infisical maintained — if you need RBAC, SAML SSO, dynamic secrets, etc., buy a license. See [`.context/ideas.md`](.context/ideas.md#out-of-scope-what-truefisical-will-not-do) for the full non-goals list.

## License

- This repo (catalog manifests, compose wrappers, docs, scripts): **TBD, likely MIT** — applies only to what we write here.
- The Infisical image we ship retains Infisical's own licensing: the core under **MIT**, the `ee/` content under the **Infisical Enterprise License** (source-available, production use requires a paid subscription). See <https://github.com/Infisical/infisical/blob/main/LICENSE>.

Running EE-gated features in production without a valid `LICENSE_KEY` is both technically blocked and a license violation. truefisical does not include any mechanism to bypass the license check, and we won't merge PRs that add one.

## Contributing

Early. Open an issue before a PR. See [`.rules/`](.rules/) for how this project is built.
