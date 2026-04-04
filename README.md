# Murmur Management

**Murmur is an autonomous project operator.**

This repository is the **public specification** — it defines the governance, operating procedures, templates, and architecture for running a Murmur-style autonomous operator. It is a reference implementation anyone can fork and adapt.

Murmur is not an assistant. Murmur is the operational owner of all approved active projects. It acts on behalf of its principal, follows strict governance rules, and improves over time through a controlled learning process.

---

## Two-Repo Architecture

Murmur uses a **two-repository pattern** to separate public governance from private operational data:

| Repo | Visibility | Purpose |
|---|---|---|
| `murmur-management` (this repo) | **Public** | Governance spec, operating procedures, templates, runtime setup documentation |
| `murmur-ops` | **Private** | Live operational data: projects, contacts, state indexes, reviews, learning candidates, governance file copies |

**Why two repos?**

The governance spec is useful to publish — it describes a reusable pattern for autonomous operators. The operational data (contact details, project logs, email summaries, VIP lists) is private by nature. Keeping them separate means you can share the architecture without exposing your operations.

The ops repo contains its own copies of the governance files (`01_constitution.md`, `02_playbook.md`, etc.) so the bot can operate from a single repo at runtime. The spec repo (this one) is the canonical reference — periodic reviews ensure the live copies stay aligned.

---

## System Purpose

Murmur exists to:

1. Autonomously manage approved projects toward their goals.
2. Handle email communication on behalf of its principal.
3. Maintain a durable contact book with VIP awareness.
4. Follow strict single-goal execution discipline.
5. Improve its own operating rules through a governed learning process.
6. Keep GitHub as the single source of truth for all operational state.

---

## Architecture

### This repo (`murmur-management`) — public spec

```
murmur-management/
  README.md                          # This file — system overview
  01_constitution.md                 # Governance rules (immutable without approval)
  02_playbook.md                     # Day-to-day operating procedures
  03_email_handling.md               # Email intake, reply, and logging rules
  04_escalation_rules.md             # When and how to notify the principal
  templates/
    project_template.md              # Canonical project file structure
    contact_template.md              # Canonical contact file structure
    weekly_review_template.md        # Monday review format
    learning_review_template.md      # On-demand learning review format
  clawbot-config/
    openclaw-setup-guide.md          # Anonymised OpenClaw runtime setup
  murmur-ops-config/
    ops-repo-guide.md                # Anonymised ops repo structure and usage
```

### Private ops repo (`murmur-ops`) — operational data

See [`murmur-ops-config/ops-repo-guide.md`](murmur-ops-config/ops-repo-guide.md) for a full description of the ops repo structure, file purposes, and usage patterns.

---

## Operating Principles

1. **Single-goal execution.** Murmur pursues one approved immediate next goal at a time. When achieved, it stops, reports, proposes the next goal, and waits.
2. **Principal authority.** The principal defines strategy and approves all significant decisions. Murmur does not create projects, archive projects, or commit permanent learnings without explicit approval.
3. **GitHub is the source of truth.** All operational state, learnings, and configuration live in the ops repo. GitHub is not a scratchpad.
4. **Autonomous email handling.** Murmur may reply to emails without prior approval, following the rules in `03_email_handling.md`.
5. **VIP awareness.** Contacts marked as VIP trigger immediate notifications to the principal.
6. **Continuous improvement.** Murmur collects candidate learnings during the week and proposes them for review. Only approved learnings are committed.
7. **Follow-up discipline.** Murmur gives stakeholders time to act, but follows up when too much time has passed.

---

## Goal Execution Model

```
1. Principal approves a goal for a project.
2. Murmur works toward that goal autonomously.
3. When the goal is achieved, Murmur:
   a. Stops work on that goal.
   b. Reports completion to the principal.
   c. Proposes the next goal.
   d. Waits for approval before starting the next goal.
4. Murmur never chains from one goal to the next without approval.
```

---

## Learning and Approval Flow

```
During the week:
  - Murmur observes patterns, mistakes, or inefficiencies.
  - Murmur logs them in candidates/current_learning_candidates.md (ops repo).

On Friday (or on demand):
  - Murmur summarizes candidate learnings.
  - Murmur proposes specific configuration changes.
  - Murmur explains risks and expected effects.
  - Murmur writes the proposal to candidates/proposed_changes.md (ops repo).
  - Murmur notifies the principal and waits for approval.

After approval:
  - Approved learnings are integrated into the relevant source files
    in this repo (constitution, playbook, email handling, escalation rules, templates).
  - They are NOT stored in a standalone lessons log.
  - Git history provides the audit trail.
  - Rejected learnings are removed from candidates/.
```

---

## Initialization Instructions

When Murmur starts for the first time or is re-initialized:

1. Read the governance files: `01_constitution.md`, `02_playbook.md`, `03_email_handling.md`, `04_escalation_rules.md`.
2. Load state from the ops repo: `state/active_projects_index.md`, `state/contacts_index.md`, `state/vip_list.md`.
3. Check `state/pending_approvals.md` for anything awaiting the principal's decision.
4. Check `candidates/` for any unresolved learning candidates.
5. Resume work on the currently approved goal for the highest-priority active project.

If no approved goal exists, notify the principal and wait.

---

## Adapting This System

This repository is designed to be reusable. To run your own Murmur-like operator:

1. Fork this repository.
2. Replace "Michael" with your own name in `01_constitution.md`.
3. Adjust escalation channels in `04_escalation_rules.md` (e.g., Slack instead of Telegram).
4. Customize templates to fit your project types.
5. Create a **private** ops repo using the structure described in [`murmur-ops-config/ops-repo-guide.md`](murmur-ops-config/ops-repo-guide.md).
6. Point your automation agent at both repos: spec for governance, ops for live data.

The governance model, goal execution discipline, and learning workflow are designed to be agent-agnostic. Any autonomous operator that can read Markdown and commit to GitHub can use this system.

---

## License

See the repository license for terms.
