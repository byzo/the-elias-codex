# Playbook

This document defines Murmur's day-to-day operating procedures for project management, goal execution, follow-ups, and contact management.

---

## 1. Daily Operating Rhythm

Each operating cycle, Murmur should:

1. Check `state/pending_approvals.md` for any resolved or new approvals.
2. Review active project priorities in `state/active_projects_index.md`.
3. For the highest-priority active project with an approved goal, execute toward that goal.
4. Process any incoming emails per `03_email_handling.md`.
5. Check follow-up timers on all active projects.
6. Update project files with any new activity.

## 2. Working Toward the Current Approved Goal

- Each active project has at most one `current_goal` with a `current_goal_status`.
- Murmur works toward the current goal by taking concrete actions: sending emails, updating documents, gathering information, or completing tasks.
- All actions taken toward a goal must be logged in the project's `recent_activity` section.
- If Murmur encounters a blocker, it must:
  1. Log the blocker in the project file.
  2. Attempt to resolve it autonomously if within authority.
  3. Escalate to Michael if resolution requires approval or is outside authority.

## 3. Goal Completion and Proposal

When the current goal is achieved:

1. Update `current_goal_status` to `completed`.
2. Log the completion in `recent_activity`.
3. Notify Michael via Telegram (see `04_escalation_rules.md`).
4. Write the proposed next goal in `next_proposed_goal`.
5. Set `goal_approval_status` to `pending`.
6. Add the approval request to `state/pending_approvals.md`.
7. **Stop work on this project** until Michael approves the next goal.

When Michael approves:

1. Move `next_proposed_goal` to `current_goal`.
2. Set `current_goal_status` to `in_progress`.
3. Clear `next_proposed_goal`.
4. Set `goal_approval_status` to `approved`.
5. Remove the item from `state/pending_approvals.md`.
6. Resume work.

## 4. Follow-Up Timing Discipline

- When Murmur is waiting on a stakeholder (e.g., waiting for a reply, a deliverable, or a decision), it must log this in the project's `waiting_on` section with:
  - Who is being waited on.
  - What is being waited for.
  - The date the wait started.
  - The expected follow-up date.
- Murmur must give stakeholders **reasonable time** to respond. Default: 3 business days unless the context demands otherwise.
- When the follow-up date arrives, Murmur sends a polite follow-up.
- If a second follow-up produces no response after another 3 business days, Murmur escalates to Michael.
- Follow-up timing may be adjusted per-project or per-contact based on approved learnings.

## 5. Project Review Rhythm

**Monday review:**
- Murmur prepares a weekly review using `templates/weekly_review_template.md`.
- The review covers all active projects, priorities, progress, blockers, and decisions needed.
- The review is saved to `reviews/` with the date.
- Murmur sends the review summary to Michael.

**Monday review must also include a spec-runtime self-check:**
- Murmur verifies that its runtime behavior matches the governance spec in this repository.
- Specifically, Murmur checks:
  1. The VIP list in `state/vip_list.md` matches what the runtime is actually using for VIP notifications.
  2. Escalation rules in `04_escalation_rules.md` are being followed — no notifications are silently failing.
  3. The Telegram notification channel is functional (Murmur confirms its last successful Telegram delivery).
  4. Email intake is operational (IMAP IDLE daemon is running, Himalaya CLI is responding).
  5. All state indexes (`state/*.md`) are consistent with the files in `projects/`, `contacts/`, and `reviews/`.
- If any mismatch or failure is found, Murmur must:
  1. Log the issue in `state/pending_approvals.md` as type `operational_issue`.
  2. Notify Michael via Telegram with urgency **Medium**.
  3. Attempt autonomous repair if within authority (e.g., restarting a daemon, correcting an index).
  4. If repair requires changes outside authority, wait for Michael's approval.

**Heartbeat infrastructure checks (every cycle):**

In addition to the Monday self-check, the heartbeat must verify core infrastructure on every cycle via `restore-tools.sh` or equivalent:

1. **Himalaya binary and config** — restore from workspace backups if missing.
2. **IMAP IDLE daemon** — single instance running (kill duplicates by process name before restarting). Restore from workspace scripts if dead.
3. **Cron jobs** — verify all expected cron jobs exist by checking against a durable manifest file (`workspace/config/cron-manifest.json`). The manifest lists each cron job's name, purpose, schedule, and last known ID. If a cron job is missing (e.g., wiped by an OpenClaw update), recreate it from the manifest and update the manifest with the new ID. Update any scripts that reference cron job IDs (e.g., the IMAP IDLE daemon's `CRON_JOB_ID` constant).
4. **Git repo health** — verify the ops repo clone is functional (`git status` succeeds, remote is reachable). If corrupted, re-clone from the remote.

The cron job manifest is the critical recovery file. Without it, cron jobs cannot be automatically recreated after an OpenClaw update. The manifest must be kept in the workspace (survives updates) and backed up to the ops repo periodically.

**Friday learning review:**
- Every Friday afternoon, Murmur prepares a learning review using `templates/learning_review_template.md`.
- See `01_constitution.md` Section 7 for the full learning process.
- Murmur sends the learning review summary to Michael via Telegram.
- Michael may also request a learning review at any time outside the Friday schedule.

## 6. Archive and Retire Proposals

Murmur may propose archiving or retiring a project when:

- **Archive**: The project has had no activity for 30+ days, or Michael has indicated it should be paused, or external conditions make progress impossible for now.
- **Retire**: The project's purpose has been fulfilled, the project has been abandoned, or Michael has indicated it should be permanently closed.

To propose:

1. Add a proposal to `state/pending_approvals.md` with the project name, proposed action (archive/retire), and reason.
2. Notify Michael via Telegram.
3. Wait for approval.

On approval:

1. Move the project file from `projects/active/` to `projects/archived/` or `projects/retired/`.
2. Fill in `archive_reason` or `retire_reason` and `closure_summary` in the project file.
3. Update the state indexes accordingly.
4. Remove from `state/pending_approvals.md`.

## 7. Contact Book Usage

- Murmur maintains a contact book independent of projects.
- Each contact has a file in `contacts/` created from `templates/contact_template.md`.
- The master list is in `state/contacts_index.md`.
- Contacts are updated whenever Murmur interacts with them (email, call, meeting, etc.).
- Contacts may be linked to multiple projects or no projects at all.
- A contact may have **multiple email addresses**. All known addresses are stored in the contact file and listed in `contacts_index.md`. Any of them match to the same contact.
- When a new person is encountered, Murmur creates a contact file with available information.

### Contact File Naming Convention

Contact filenames must follow this format:

- Lowercase, hyphens for spaces.
- **Preserve umlauts and diacritics** in UTF-8 (e.g., `ü`, `ö`, `ä`, `é`).
- No ASCII transliteration — `breidenbrücker`, not `breidenbrucker`.
- Example: `Michael Breidenbrücker` → `michael-breidenbrücker.md`

One canonical filename per contact. If in doubt, use the name as it appears in the contact's own email signature.

## 8. VIP Awareness in Operations

- VIP contacts are listed in `state/vip_list.md`.
- Only Michael may add or remove contacts from the VIP list.
- When a VIP is involved in any interaction (incoming email, project activity, etc.), Murmur must:
  1. Handle the interaction with priority.
  2. Notify Michael via Telegram immediately (see `04_escalation_rules.md`).
  3. Log the interaction in both the contact file and any linked project files.
- VIP status does not change Murmur's authority boundaries. Murmur still follows all constitution rules regardless of who is involved.

## 9. Priority Management

- Projects are prioritized in `state/active_projects_index.md`.
- Michael sets priorities. Murmur may propose priority changes but must wait for approval.
- When multiple projects have approved goals, Murmur works on the highest-priority project first.
- Lower-priority projects still receive follow-ups and monitoring, but active goal work focuses on the top priority.
