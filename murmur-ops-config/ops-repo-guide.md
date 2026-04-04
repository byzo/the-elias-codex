# Murmur Ops — Repository Guide

**Repo:** `murmur-ops` (private)

This document describes the structure, purpose, and usage patterns of the private operational repository. All personal data, project details, and contact information have been removed — this is a reference for anyone setting up their own Murmur ops repo.

---

## Purpose

The ops repo is where Murmur does its actual work. It contains:

- **Live project files** — one Markdown file per project, tracking goals, activity logs, and status
- **Contact records** — one file per contact with communication history
- **State indexes** — master lists that Murmur loads on startup to understand what's active
- **Learning candidates** — observations collected during the week, awaiting review
- **Review snapshots** — completed weekly and learning reviews
- **Governance file copies** — the bot's working copies of the spec files from `murmur-management`

The ops repo is the bot's durable memory. If it's not committed here, it didn't happen.

---

## Repository Structure

```
murmur-ops/
  README.md                              # Repo overview (can mirror murmur-management or be ops-specific)
  01_constitution.md                     # Working copy of governance rules
  02_playbook.md                         # Working copy of operating procedures
  03_email_handling.md                   # Working copy of email handling rules
  04_escalation_rules.md                 # Working copy of escalation rules
  state/
    active_projects_index.md             # Index of all active projects
    archived_projects_index.md           # Index of all archived projects
    retired_projects_index.md            # Index of all retired projects
    contacts_index.md                    # Master contact list
    vip_list.md                          # VIP contacts requiring special handling
    pending_approvals.md                 # Items awaiting principal's approval
  projects/
    active/                              # One .md file per active project (e.g., PROJ-001-name.md)
    archived/                            # Completed or paused projects
    retired/                             # Permanently closed projects
  contacts/                              # One .md file per contact (e.g., jane-doe.md)
  candidates/
    current_learning_candidates.md       # Working observations collected during the week
    proposed_changes.md                  # Changes proposed for principal approval
  reviews/                               # Completed weekly and learning review snapshots
  templates/
    project_template.md                  # Canonical project file structure
    contact_template.md                  # Canonical contact file structure
    learning_review_template.md          # Learning review format
```

---

## Key Files

### State Indexes

State indexes are the first thing Murmur loads on startup. They provide a quick overview without needing to scan individual files.

| File | Purpose |
|---|---|
| `active_projects_index.md` | Lists all active projects with priority, current goal, and status |
| `archived_projects_index.md` | Lists completed/paused projects |
| `retired_projects_index.md` | Lists permanently closed projects |
| `contacts_index.md` | Master list of all contacts with key details |
| `vip_list.md` | Contacts that trigger immediate principal notifications |
| `pending_approvals.md` | Queue of items awaiting principal decision (new goals, learnings, operational issues) |

### Project Files

Each project gets a single Markdown file following the template in `templates/project_template.md`. A project file contains:

- **Metadata** — project ID, name, status, priority, creation date
- **Goal** — the currently approved immediate next goal
- **Activity log** — chronological record of actions taken, emails sent/received, decisions made
- **Waiting on** — what the project is blocked on, with follow-up dates
- **Next goal proposal** — what Murmur recommends doing next (awaiting approval)

Project files move between `active/`, `archived/`, and `retired/` as their status changes.

### Contact Files

Each contact gets a file following `templates/contact_template.md`. Contains:

- **Contact details** — name, email, role, organization
- **VIP status** — whether this contact triggers escalation
- **Communication log** — chronological record of all emails and interactions
- **Relationship to projects** — which projects this contact is involved in

### Learning Candidates

During normal operation, Murmur observes patterns, mistakes, or inefficiencies and logs them in `candidates/current_learning_candidates.md`. On review (weekly or on-demand), these are summarized into `candidates/proposed_changes.md` with specific recommendations for the principal to approve or reject.

Approved changes are integrated into the governance files (constitution, playbook, etc.) — not stored as standalone lessons.

---

## Governance File Copies

The ops repo contains working copies of the four governance files from `murmur-management`:

- `01_constitution.md`
- `02_playbook.md`
- `03_email_handling.md`
- `04_escalation_rules.md`

This allows the bot to operate from a single repo at runtime. The public spec repo (`murmur-management`) is the canonical reference. Periodic reviews should check that the ops copies haven't drifted from the spec.

---

## Commit Standards

All commits to the ops repo follow the same format defined in the constitution:

```
[category] Short description of what changed and why
```

Categories: `[project]`, `[contact]`, `[state]`, `[email]`, `[review]`, `[learning]`, `[governance]`, `[ops]`

---

## Setting Up Your Own Ops Repo

1. Create a **private** GitHub repository.
2. Create the folder structure shown above.
3. Add `.gitkeep` files to empty directories (`projects/active/`, `projects/archived/`, etc.).
4. Copy the governance files from `murmur-management` into the repo root.
5. Copy the templates from `murmur-management/templates/`.
6. Initialize `state/` files with empty indexes (use the templates as reference for format).
7. Point your bot at this repo as its working repository.

---

## Privacy

This repo is private by design. It contains:

- Real contact names and email addresses
- Project details and business context
- Email communication logs
- VIP designations

Never make this repo public. The public spec in `murmur-management` documents the architecture without exposing operational data.
