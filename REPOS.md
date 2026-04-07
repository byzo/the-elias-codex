# REPOS.md — Repository Map

There are three repositories. This document defines what goes in each, once and for all.

---

## Overview

**murmur-ops** and **clawbot-config** are where real work happens. **murmur-management** is the public documentation mirror of both — everything generalised, no personal data, no credentials.

**The core rule:** whenever something changes in murmur-ops or clawbot-config, the corresponding generalised version in murmur-management must be updated too.

---

## The Three Repos

### 1. `murmur-ops` — Private operational data
**Visibility:** Private  
**Local path:** `workspace/murmur-management/`  
**Purpose:** Murmur's working memory. All operational activity lives here.

**What lives here:**
- Live governance files (`01_constitution.md`, `02_playbook.md`, `03_email_handling.md`, `04_escalation_rules.md`)
- `AGENTS.md` — live session startup and operating procedures
- Project files (`projects/active/`, `projects/archived/`, `projects/retired/`)
- Contact records (`contacts/`)
- State indexes (`state/`)
- Learning candidates and reviews (`candidates/`, `reviews/`)
- Memory files (`memory/YYYY-MM-DD.md`)

**What does NOT go here:**
- Credentials, tokens, API keys
- Infrastructure config — hostnames, cron IDs, deployment details (those go in clawbot-config)

---

### 2. `clawbot-config` — Private deployment config
**Visibility:** Private  
**Purpose:** Infrastructure and deployment specifics for this OpenClaw instance.

**What lives here:**
- Cron job IDs, schedules, and recreate commands
- Hostnames, server details
- Credential file locations and references
- Deployment runbooks and setup guides

**What does NOT go here:**
- Operational data — projects, contacts, state (that's murmur-ops)
- Governance spec (that's murmur-management)

---

### 3. `murmur-management` — Public documentation mirror
**Visibility:** Public  
**Purpose:** A generalised, public-safe documentation of everything in murmur-ops and clawbot-config. No real names, no credentials, no operational data. Useful as a template for anyone setting up a similar system.

**Structure mirrors the private repos:**
- **Root files** (`01_constitution.md`, `02_playbook.md`, `AGENTS.md`, etc.) — generalised versions of the live governance files in murmur-ops
- **`murmur-ops-config/`** — generalised documentation of the murmur-ops repo structure
- **`clawbot-config/`** — generalised documentation of the clawbot-config repo structure
- **`templates/`** — canonical file templates
- **`_posts/`** — blog posts (working-with-byzo and other published writing)

**What does NOT go here:**
- Real names, email addresses, Telegram IDs
- Project details, contact records, communication logs
- Credentials, tokens, hostnames
- Anything instance-specific

---

## The Sync Rule

murmur-management is a documentation mirror — not the source of truth. The source of truth is always murmur-ops (for operational files) or clawbot-config (for infrastructure).

**When you change something in murmur-ops or clawbot-config:**
1. Make the change in the private repo first
2. Update the corresponding generalised version in murmur-management
3. Open a PR (main is protected on murmur-management)

**When in doubt about where something goes:**

| It's... | Repo |
|---|---|
| Operational — projects, contacts, state, memory | `murmur-ops` (private) |
| Infrastructure — cron IDs, hostnames, credentials | `clawbot-config` (private) |
| A generalised/documented version of either | `murmur-management` (public) |
