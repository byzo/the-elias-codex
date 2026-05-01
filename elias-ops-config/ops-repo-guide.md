# Ops Repo Guide

**Pattern name:** the ops repo (commonly named `<agent>-ops`).

This document describes the structure, purpose, and usage of the **private operational repository** that an agent built on this spec uses as its working memory.

The ops repo is the single most important asset in the system. It holds:

- The agent's working data — projects, contacts, state, candidates, reviews
- The runtime scripts and daemons (under version control, not stranded in the container's writable layer)
- The canonical copies of the workspace governance files (so they survive container wipes)
- The cron manifest and overrides config
- The static site source (so a wiped Caddy directory can be rebuilt)
- Daily memory files

If it lives only in `/home/node/.openclaw/workspace/` and not in the ops repo, it is **at risk of being wiped** by the next container update.

---

## Why this exists separately from the spec repo

| Repo | Visibility | What it holds |
|---|---|---|
| **Spec repo** (this repo) | Public | Anonymised governance + setup pattern. Reusable for anyone. |
| **Ops repo** (`<agent>-ops`) | **Private** | Live ops data + the agent's runtime guts |
| **Runtime-config repo** | Private | Host-side setup (Caddy, docker-compose, secrets inventory, incident log) |

The ops repo is private by nature — it contains contact details, project context, email logs, VIP designations, learning candidates, and a daily memory log. None of that should ever be public.

The spec repo (this one) is the canonical reference for the *pattern*. Periodic reviews ensure the live governance copies in the ops repo haven't drifted from the spec.

---

## Repository structure

```
<agent>-ops/
  README.md                              # repo overview
  AGENTS.md                              # working copy of the spec's AGENTS.md
  HEARTBEAT.md                           # canonical heartbeat config (workspace copy is restored from this)
  BOOT.md                                # canonical boot-hook script (workspace copy is restored from this)
  RUNBOOK.md                             # in-container runtime architecture (this instance's runbook)
  TOOLS.md                               # local tooling notes (canonical copy)
  MEMORY.md                              # facts file (infra, contacts, preferences) — never project state
  SOUL.md                                # persona (canonical copy if you keep one)

  01_constitution.md                     # working copy of governance rules
  02_playbook.md                         # working copy of operating procedures
  03_email_handling.md                   # working copy of email handling rules
  04_escalation_rules.md                 # working copy of escalation rules

  state/
    active_projects_index.md             # ranked list of active projects
    archived_projects_index.md
    retired_projects_index.md
    contacts_index.md                    # master contact list
    vip_list.md                          # VIPs that trigger escalation
    pending_approvals.md                 # outstanding decisions for the principal
    reminders.md                         # principal-set reminders surfaced by heartbeat

  projects/
    active/                              # one .md file per active project (e.g. PROJ-001-name.md)
    archived/                            # paused projects (may be reactivated)
    retired/                             # permanently closed

  contacts/                              # one .md file per contact

  candidates/
    current_learning_candidates.md       # observations collected during the week
    proposed_changes.md                  # changes proposed for principal approval

  reviews/                               # weekly + learning review snapshots
                                         # plus other long-form reports the agent generates

  templates/                             # canonical file templates (mirror the spec's templates/)

  bin/                                   # in-container runtime scripts
    restore-tools.sh
    start-uptime-watchdog.sh
    start-model-fallback-watcher.sh
    uptime-watchdog.sh
    uptime-check.sh                      # plus per-service uptime check scripts
    model-fallback-watcher.py
    <email-cli-binary>                   # large binary kept in git so it survives container wipes
  imap-idle/
    <agent>-mail.py                      # the IMAP IDLE daemon
    start.sh                             # singleton wrapper
    *.pid, *.log                         # runtime state, gitignored

  config/
    cron-manifest.json                   # single source of truth for cron jobs
    openclaw-overrides.json              # non-secret config overrides deep-merged into openclaw.json
    <email-cli>/config.toml              # backup of the email CLI config
    github-token.txt                     # gitignored
  scripts/                               # higher-level recipe documents (e.g. agent-identification.md)

  memory/
    YYYY-MM-DD.md                        # daily memory log
    ...

  <static-site>/                         # canonical source of the static site Caddy serves
                                         # (workspace copy is restored from this by restore-tools.sh)
```

The cleanest mental model: **`bin/` and `imap-idle/` are the agent's body; `state/`, `projects/`, `contacts/`, `candidates/`, `reviews/` are the agent's mind; `01_*.md`–`04_*.md` are the agent's rules; `memory/` is the agent's diary.** All four belong in version control.

---

## Key files and patterns

### The four governance files

`01_constitution.md`, `02_playbook.md`, `03_email_handling.md`, `04_escalation_rules.md` are working copies of the spec versions. They are what the agent actually reads at runtime.

The spec repo (this one) is the canonical reference. Periodic review checks that the live copies haven't drifted. If a learning is approved that changes governance, update both copies (or update the spec first, then sync).

### State indexes

State indexes are loaded on startup as a fast overview without scanning every project file.

| File | Purpose |
|---|---|
| `active_projects_index.md` | Ranked list of active projects with priority + current goal stub |
| `archived_projects_index.md` | Paused projects |
| `retired_projects_index.md` | Permanently closed projects |
| `contacts_index.md` | Master contact list (name, email, role, VIP flag, project links) |
| `vip_list.md` | VIPs that trigger escalation |
| `pending_approvals.md` | Outstanding decisions blocking the agent |
| `reminders.md` | Principal-set reminders surfaced by the heartbeat |

The active projects index is the heartbeat-friendly cache: a compact table the heartbeat can read in one shot to find anything overdue, without scanning every project file every 2 hours.

### Project files

One Markdown file per project, following `templates/project_template.md`. A project file holds:

- **Metadata** — id, name, status, priority, creation date
- **Current goal** — the single approved next goal
- **Recent activity** — chronological log
- **Communication log** — emails sent/received, decisions made
- **Waiting on** — what the project is blocked on, with follow-up dates
- **Next proposed goal** — awaiting approval

Project files move between `projects/active/`, `projects/archived/`, and `projects/retired/` as their status changes (the active index is updated to match).

### Contact files

One Markdown file per contact. Contains:

- Contact details — name, email(s), role, organisation
- VIP flag
- Communication log — chronological record of all interactions
- Project links — which projects this contact is involved in

### Candidates and reviews

The learning loop:

1. During the week, the agent observes patterns/mistakes/inefficiencies and appends to `candidates/current_learning_candidates.md`.
2. On Friday (or on-demand), it summarises into `candidates/proposed_changes.md` with concrete file edits and risk notes.
3. The principal approves or rejects.
4. Approved changes are integrated into the relevant governance file. Rejected changes are deleted.
5. A review snapshot lands in `reviews/`.

There is **no standalone lessons log**. Git history is the audit trail.

### Memory

`memory/YYYY-MM-DD.md` is a daily scratch log — what happened today, what was decided, what was sent. Cron / isolated sessions also write here so silent action leaves a trace.

`MEMORY.md` (in the workspace, restored to canonical from the ops repo) is a **facts file**, not a status file. Infra, contacts, preferences, standing rules. It is loaded at session start. Putting project state into MEMORY.md is the bug pattern that produces "stale-state-resurfaced-as-fresh-news" — see `RUNBOOK.md` lessons.

### Scripts directory

`scripts/` holds longer-form recipe documents that don't belong in the four governance files but that the agent should read when entering a specific flow (e.g. a podcast/blog observer pass, a network identification probe). `AGENTS.md` lists flow-words that map to these scripts.

### Reviews directory

`reviews/` holds completed weekly reviews, learning reviews, incident reports, architecture reviews, and other long-form artefacts the agent generates. **All generated reports land here, never in `workspace/reports/`** — that path is wipeable.

---

## In-container scripts: why they live here

A subtle but important detail: this repo is the canonical home for the agent's runtime scripts (`bin/`, `imap-idle/`).

The reason is the workspace-wipe model. OpenClaw container updates wipe most of the writable layer. Anything that lives only at `workspace/bin/` or `workspace/imap-idle/` will be gone after the next update. The fix is to put the scripts under version control here, and have `bin/restore-tools.sh` restore them on container start (via the `boot-md` hook running `BOOT.md`).

This produces a self-bootstrapping system: even after a fresh container, the boot hook clones the ops repo, runs `restore-tools.sh`, and the daemons come back up.

The full runtime model is documented in `clawbot-config/openclaw-setup-guide.md` (in the spec repo).

---

## Governance file copies — keeping the workspace bootable

Files like `BOOT.md`, `HEARTBEAT.md`, and `AGENTS.md` are sometimes loaded by OpenClaw from the workspace root (not the ops repo). They get wiped on container update along with everything else in the writable workspace layer.

The pattern: keep the **canonical copy in the ops repo** and have `restore-tools.sh` copy it into place at `workspace/<file>` if the workspace copy is missing. That way you can edit the canonical copy in git, restart, and the workspace gets the new version.

---

## Commit standards

All commits follow the format defined in `01_constitution.md` Section 8:

```
[category] Short description of what changed and why
```

Categories: `[project]`, `[contact]`, `[state]`, `[email]`, `[review]`, `[learning]`, `[governance]`, `[ops]`

Each commit = one logical change. **Commit immediately after every file change** — don't batch, don't defer to session close. A change that isn't committed is at risk.

---

## Setting up your own ops repo

1. Create a **private** GitHub repository.
2. Create the directory structure shown above. Add `.gitkeep` to empty dirs.
3. Copy the four governance files from the spec repo into the root.
4. Copy `templates/` from the spec repo.
5. Initialise empty state indexes (use the templates as format reference).
6. Drop the runtime scripts into `bin/` and `imap-idle/` (see the spec's setup guide for what each script does).
7. Add a `cron-manifest.json` listing the cron jobs you want — initially `<service> uptime check`, `<service> recovery`, and `Friday Learning Review` are usually enough.
8. Add `openclaw-overrides.json` for any non-secret config you want to pin (e.g. `tools.exec.notifyOnExit: false`).
9. Add `.gitignore` covering: PID/log files, the github-token, the email CLI password file (or commit a stub config and keep the password in `~/.config` only).
10. Point the agent at this repo as its working repository.
11. Configure the `boot-md` hook to read `workspace/BOOT.md` so `restore-tools.sh` runs on every container start.

---

## Privacy

This repo is private by design. It contains:

- Real contact names and email addresses
- Project details and business context
- Email communication logs
- VIP designations
- Daily memory of what happened
- (Optionally) backups of email-CLI configs containing IMAP passwords — gitignore those

Never make this repo public. The public spec is the only thing that should be visible from the outside.
