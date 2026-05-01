# REPOS.md — Repository Map

The pattern uses three repositories. This document defines what goes in each, once and for all.

---

## Overview

**The ops repo** and **the runtime-config repo** are where real work happens. **The Elias Codex** (this repo) is the public, generalised documentation of the pattern — no personal data, no credentials, no operational data.

**The core rule:** whenever something changes structurally in the ops repo or the runtime-config repo, the corresponding generalised version in this spec repo must be updated too.

---

## The Three Repos

### 1. `<agent>-ops` — Private operational data + runtime guts
**Visibility:** Private
**Purpose:** The agent's working memory and runtime body. All operational activity lives here.

**What lives here:**
- Live working copies of the governance files (`01_constitution.md`, `02_playbook.md`, `03_email_handling.md`, `04_escalation_rules.md`)
- `AGENTS.md` — live session startup and operating procedures
- `BOOT.md`, `HEARTBEAT.md`, `RUNBOOK.md` — canonical copies of files OpenClaw loads from the workspace, so they survive container wipes
- Project files (`projects/active/`, `projects/archived/`, `projects/retired/`)
- Contact records (`contacts/`)
- State indexes (`state/`)
- Learning candidates and reviews (`candidates/`, `reviews/`)
- Memory files (`memory/YYYY-MM-DD.md`)
- In-container runtime scripts (`bin/`, `imap-idle/`)
- Cron manifest (`config/cron-manifest.json`) and openclaw config overrides (`config/openclaw-overrides.json`)
- Backup of the email CLI config (`config/<email-cli>/config.toml`)
- The static site source if the agent serves one

**What does NOT go here:**
- LLM API keys, channel tokens, gateway tokens — those stay in `~/.openclaw/openclaw.json` (not in any repo)
- Host-side infrastructure config — Caddyfile, docker-compose.yml, hostnames (those go in the runtime-config repo)

See `elias-ops-config/ops-repo-guide.md` for the full structure and rationale.

---

### 2. `<agent>-runtime-config` — Private host-side runtime config
**Visibility:** Private
**Purpose:** Host-side infrastructure for this OpenClaw deployment.

**What lives here:**
- VPS hostname, SSH user, IP details
- Caddy config (`Caddyfile`)
- `docker-compose.yml`
- Secrets inventory — *where* each secret is stored, never the values
- Incident log — root causes and fixes for past production incidents
- Runbook for host-side operations (start, stop, restart, update OpenClaw, clear stuck sessions)
- Cron job IDs as currently deployed (the canonical manifest is in the ops repo; this is a snapshot)

**What does NOT go here:**
- Operational data — projects, contacts, state, memory (that's the ops repo)
- Governance spec (that's this repo)
- Live secret values

See `clawbot-config/openclaw-setup-guide.md` for the full structure and rationale.

---

### 3. `the-elias-codex` — Public spec / pattern
**Visibility:** Public
**Purpose:** Anonymised, public-safe documentation of the whole pattern. Useful as a template for anyone setting up a similar system.

**What lives here:**
- Root governance files (`01_constitution.md` … `04_escalation_rules.md`, `AGENTS.md`)
- `templates/` — canonical file templates
- `elias-ops-config/ops-repo-guide.md` — anonymised guide to the ops repo
- `clawbot-config/openclaw-setup-guide.md` — anonymised guide to the runtime-config repo and the OpenClaw setup
- `_posts/` — blog posts (the public-voice content lives in this repo's Jekyll site)

**What does NOT go here:**
- Real names, email addresses, channel IDs
- Project details, contact records, communication logs
- Credentials, tokens, hostnames
- Anything instance-specific

---

## The sync rule

The spec repo is a **documentation mirror** — not the source of truth for any specific instance. The source of truth is always the ops repo (for operational files and runtime scripts) or the runtime-config repo (for host-side infrastructure).

**When you change something structural in the ops repo or the runtime-config repo:**
1. Make the change in the private repo first.
2. Update the corresponding generalised version in the spec repo.
3. Open a PR (main is protected on the spec repo).

**When in doubt about where something goes:**

| It's… | Repo |
|---|---|
| Operational — projects, contacts, state, memory, runtime scripts | `<agent>-ops` (private) |
| Host-side — hostnames, Caddy, docker-compose, secrets inventory | `<agent>-runtime-config` (private) |
| Anonymised pattern documentation | `the-elias-codex` (public) |
