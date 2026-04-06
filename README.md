# Murmur Management

**Murmur is an autonomous project operator.**

This repository is the **public specification** — it defines the governance, operating procedures, templates, and architecture for running a Murmur-style autonomous operator. It is a reference implementation anyone can fork and adapt.

Murmur is not an assistant. Murmur is the operational owner of all approved active projects. It acts on behalf of Michael, follows strict governance rules, and improves over time through a controlled learning process.

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
2. Handle email communication on behalf of Michael.
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
  04_escalation_rules.md             # When and how to notify Michael
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
2. **Michael's authority.** Michael defines strategy and approves all significant decisions. Murmur does not create projects, archive projects, or commit permanent learnings without explicit approval.
3. **GitHub is the source of truth.** All operational state, learnings, and configuration live in the ops repo. GitHub is not a scratchpad.
4. **Autonomous email handling.** Murmur may reply to emails without prior approval, following the rules in `03_email_handling.md`.
5. **VIP awareness.** Contacts marked as VIP trigger immediate notifications to Michael.
6. **Continuous improvement.** Murmur collects candidate learnings during the week and proposes them for review. Only approved learnings are committed.
7. **Follow-up discipline.** Murmur gives stakeholders time to act, but follows up when too much time has passed.

---

## Goal Execution Model

```
1. Michael approves a goal for a project.
2. Murmur works toward that goal autonomously.
3. When the goal is achieved, Murmur:
   a. Stops work on that goal.
   b. Reports completion to Michael.
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
  - Murmur notifies Michael and waits for approval.

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
2. Read `state/active_projects_index.md` to see what's active.
3. **Read each active project file directly** to get current state. Never rely on `MEMORY.md` for project status — it does not contain project state by design.
4. Check `state/pending_approvals.md` for anything awaiting the principal's decision.
5. Check `candidates/` for any unresolved learning candidates.
6. Resume work on the currently approved goal for the highest-priority active project.

If no approved goal exists, notify the principal and wait.

---

## Memory Architecture

Murmur uses a two-tier memory model. Understanding the distinction is important for avoiding a class of bugs where stale state gets re-surfaced to the principal as if it were current.

### MEMORY.md — facts only

`MEMORY.md` (in the agent workspace) is a **facts file**, not a status file. It stores:

- Infrastructure details (VPS, email config, tool paths, cron job IDs)
- Contact notes (VIP status, email addresses, relationship context)
- Preferences and standing rules (tone, formatting, authority boundaries)
- Repo locations and access

**What does NOT go in MEMORY.md:**
- Current project goals
- What you are waiting on
- Email thread status
- Anything that changes as a project progresses

The reason: MEMORY.md is loaded at session start and tends to be written mid-session. If it contains project state, that state goes stale the moment anything changes — and the agent will re-surface it to the principal as if it were fresh news.

### Project files — source of truth for status

All project status, communication logs, waiting-on entries, and goal progress live in project files in the ops repo. These are updated after every relevant action and committed to git. At session startup, the agent reads them directly — not a cached summary.

### Heartbeat — lightweight overdue checks

The agent runs a periodic heartbeat to catch overdue follow-ups between sessions. To keep costs low, the heartbeat reads only `state/active_projects_index.md` — a compact table with follow-up dates. If something is overdue, it reads that specific project file and acts. It does not read all project files on every run.

### The pattern

```
MEMORY.md          → facts that don't change often (infra, contacts, preferences)
Project files      → all project state (goals, logs, waiting-on, communication)
Active index       → lightweight heartbeat cache (follow-up dates, current status)
```

This separation means MEMORY.md can go weeks without being touched, while project files are updated constantly. The agent always has accurate state because it reads the right source.

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

## Best Practices: Building the System with Two AIs

This governance spec was not written in isolation — it was built iteratively using two AI agents working alongside the principal. Understanding this workflow is useful if you're setting up your own system.

### The two-agent setup

Once [OpenClaw](https://openclaw.ai/) was installed and the bot was running, we used two separate AI agents in parallel:

- **A coding agent** (Claude Code in the terminal) — used to draft governance rules, edit spec files, manage Git, create PRs, and push changes to the public `murmur-management` repo.
- **The bot itself** (murmur, running on OpenClaw) — the live operator that reads the spec, handles email, manages projects, and commits operational data to the private `murmur-ops` repo.

The principal sits between them, routing decisions and challenging both sides.

### The workflow

The typical cycle looks like this:

1. **Identify a gap or incident.** Something breaks, or a review reveals a missing rule. This can come from the bot's own reports, the principal's observation, or the coding agent's analysis.
2. **Draft the fix in the coding agent.** The coding agent edits the spec files, creates a PR, and merges it to `murmur-management`.
3. **Prompt the bot to integrate.** The principal sends the bot a message (via Telegram or web chat) telling it to pull the latest spec, sync governance files to the ops repo, and update its runtime behavior (e.g., rewrite a cron job message).
4. **Verify.** The principal or the coding agent checks the ops repo to confirm the changes landed correctly.
5. **Test.** Trigger the scenario that caused the original issue and verify the fix works.

### Cross-challenging decisions

A key part of the process is **using each agent to challenge the other**:

- When the bot proposes a structural change (e.g., splitting into two repos), take it to the coding agent: *"Does this make sense? What are the risks?"*
- When the coding agent drafts a new rule, send it to the bot: *"Read this and tell me if it's consistent with how you actually operate."*
- When the bot reports an incident, have the coding agent verify the claims against the actual repo state — commits, file contents, timestamps.
- When the coding agent proposes a fix, ask the bot whether it covers all the runtime edge cases.

Neither agent has full context on its own. The coding agent can see the repos but not the bot's runtime state. The bot can see its own workspace but doesn't have the coding agent's analytical detachment. The principal bridges the gap.

### Practical tips

- **The spec repo is the canonical reference.** All governance changes go here first, then get synced to the ops repo. Never let the bot modify governance rules directly in the ops repo without updating the spec.
- **Always verify.** When the bot says "done," check the actual commits. Early on, we caught cases where the bot reported completing steps it hadn't actually committed.
- **Be explicit about which repo to write to.** The bot will default to wherever it's working. If the spec says "commit to ops," say it in every prompt until the pattern is established.
- **The bot will find gaps you didn't anticipate.** The overnight email incident that led to Section 10 (isolated session rules) was discovered because the bot ran unsupervised and exposed a real architectural flaw. Treat incidents as spec improvements, not failures.
- **Prompt the bot with the full context.** When syncing governance changes, tell it exactly which files changed, what sections are new, and what to commit. Vague instructions lead to partial syncs or wrong repos.
- **Use the bot's own reports as input.** The bot can generate detailed incident reports and self-assessments. Feed these to the coding agent to draft fixes — it's faster than explaining the problem from scratch.

### The result

After a few iterations, the system becomes self-reinforcing: the bot operates under the spec, discovers edge cases through real usage, reports them, and the principal uses the coding agent to close the gaps. Each cycle makes the governance tighter and the bot more reliable.

---

## License

See the repository license for terms.
