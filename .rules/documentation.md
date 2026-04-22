# Documentation Standards — truefisical

## Core Philosophy: Thin Wrapper, Honest Comparison
- Our docs should only cover what is **truefisical-specific** or **TrueNAS-specific**. Everything about Infisical as a product is already documented upstream — link to it, don't duplicate it.
- If a section could live unchanged in the upstream Infisical docs, it shouldn't live here.
- Our most valuable doc is not an install guide — it's an **honest comparison** of raw `.env` files vs. Infisical Cloud (free / paid tiers) vs. self-hosted (truefisical), including what you give up by self-hosting.

## What Belongs in This Repo's Docs
**In scope (we own it):**
- Installing `truefisical` on TrueNAS SCALE (catalog path + Custom App path).
- TrueNAS-specific operations: dataset layout, snapshots, port conflicts with DSM/TrueNAS UI, backup via `pg_dump` + ZFS.
- Comparison between hosted and self-hosted, including feature-parity gaps.
- Per-project onboarding workflow (one page template + one file per migrated project).
- Troubleshooting specifically for Infisical-on-TrueNAS interactions.
- Changelog for this app and its pinned upstream version.

**Out of scope (link upstream):**
- How Infisical's permission model works.
- CLI / SDK reference.
- Machine-identity concepts.
- Infisical's own self-hosting env-var reference (we link to `https://infisical.com/docs/self-hosting/configuration/envars`).

## The README Is the Front Door
Every reader lands on `README.md` first. It must contain:
1. One sentence explaining what truefisical is.
2. A comparison table: **Local `.env`** vs. **Infisical Cloud (Free)** vs. **Infisical Cloud (Paid)** vs. **Self-hosted (truefisical)**.
3. A short "what you give up by self-hosting" paragraph (honest, not defensive).
4. Install link → `docs/install/`.
5. A "who this is for" note: if a reader should use Infisical Cloud instead, we say so.

The comparison table is the headline artifact. Keep it current: every Infisical release that moves a feature between tiers is a README update.

## Doc Structure
```
README.md                  # front door + comparison table
docs/
├── install/
│   ├── truenas-catalog.md      # primary path
│   ├── truenas-custom-app.md   # paste-compose fallback
│   └── workstation-dev.md      # compose-only for hacking
├── operate/
│   ├── runbook.md              # upgrade, backup, restore, rotate
│   ├── monitoring.md
│   └── troubleshooting.md      # TrueNAS + Infisical interactions only
├── onboard/
│   ├── project-onboarding.md   # generic workflow template
│   └── <project>.md            # one per migrated project
├── compare/
│   ├── vs-cloud.md             # expanded rationale behind the README table
│   └── vs-local-env.md         # why move off raw .env at all
└── reference/
    └── upstream-links.md       # every Infisical concept we defer to upstream on
```

## Writing Rules
- **Link, don't copy.** If you're tempted to explain something Infisical already documents, paste the upstream URL instead.
- **Date every how-to.** At the top of every operational doc: `Last smoke-tested: YYYY-MM-DD against Infisical vX.Y.Z on TrueNAS SCALE A.B.C.` A doc without a date is assumed stale.
- **Show the exact command.** If a reader has to translate, we failed. Include the real commands with realistic placeholder values.
- **Call out what won't work.** If a feature is paid-only in Infisical Cloud and is also missing self-hosted, say so plainly.

## Comparison-Table Accuracy
The comparison table is the most load-bearing part of this repo's docs. Rules:
- Every cell must cite a source: either upstream docs, the pricing page, or a direct test we ran.
- Cells we haven't verified are marked `TBD — verify against <URL>`, not guessed.
- When Infisical ships a release, the table is re-checked before we tag a truefisical release.
- If a feature is "available self-hosted but requires extra setup", say which setup — don't just write "Yes".

## Tooling
- Start with plain Markdown. If docs grow enough to need navigation, add MkDocs Material (`bun`-installed or `uv`-installed per global rules).
- Until then, GitHub's Markdown rendering is the site.

## Writing Mindset
**Ask yourself:**
- Am I writing something that belongs upstream? (If yes, link instead.)
- Would a stranger with a TrueNAS box and a weekend succeed with this doc?
- Does this doc tell them what they're **giving up** by using truefisical, not just what they're getting?

---
*Be honest, be specific, be linkable.*
