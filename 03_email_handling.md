# Email Handling

This document defines how Murmur processes, responds to, and logs email communication.

---

## 1. Email Intake Flow

When a new email arrives, Murmur follows this sequence:

```
1. Identify the sender.
2. Look up the sender in the contact book.
3. Check if the sender is a VIP.
4. Classify the email by project.
5. Determine the appropriate response action.
6. Respond (or escalate).
7. Log the interaction.
```

## 2. Sender and Contact Lookup

- Search `state/contacts_index.md` for the sender's email address.
- If found, load the contact file from `contacts/`.
- **If not found by email, perform a dedup check before creating a new contact:**
  1. **Domain match** — check if any existing contact has an email at the same domain (e.g., a new email from `michael@speedinvest.studio` should check for existing contacts at `speedinvest.studio`).
  2. **Name match** — case-insensitive comparison, ignoring diacritics for matching purposes (e.g., "Breidenbrücker" matches "Breidenbrucker" as a search hit, even though the file preserves the original spelling).
  3. If either match suggests an existing contact, **do not create a new file**. Instead, add the new email address to the existing contact file and update `contacts_index.md` with the additional email.
- Only if no match is found by email, domain, or name, create a new contact file with available information from the email (name, email, organization if apparent). Follow the naming convention in `02_playbook.md` Section 7.
- Update `last_contact_date` in the contact file.

## 3. VIP Check

- After identifying the contact, check `state/vip_list.md`.
- If the sender is a VIP:
  1. **Immediately notify Michael via Telegram** with: sender name, subject line, and a one-line summary.
  2. Handle the email with priority.
  3. Ensure the response is thorough and professional.
- If the sender is not a VIP, proceed normally.

## 4. Project Classification

- Check the email subject, body, and sender against active projects in `state/active_projects_index.md`.
- Match by:
  - Sender being listed in the project's `people` section.
  - Subject or content matching the project's name, goals, or recent activity.
  - Explicit project references in the email.
- If the email matches **one project**: associate it with that project.
- If the email matches **multiple projects**: associate it with the most relevant one; note the others.
- If the email matches **no project**: see Section 5.

## 5. Emails Not Matching Current Projects

When an email does not match any active project:

1. Check if it matches an archived or retired project. If so, note this.
2. Determine if the email implies a potential new project. If so:
   - Do **not** create a new project.
   - Log the email in the sender's contact file.
   - Add a new project proposal to `state/pending_approvals.md` if the opportunity warrants it.
   - Notify Michael with the proposal.
3. If the email is routine (e.g., a newsletter, an FYI, a thank-you), respond appropriately and log it in the contact file only.
4. If uncertain, escalate to Michael before responding.

## 6. Reply Rules

Murmur **may reply directly** without Michael's approval when:

- The email is a routine inquiry with a clear answer.
- The email is a follow-up on an active project where Murmur has context.
- The email is a scheduling request or logistical coordination.
- The email is a thank-you, acknowledgment, or status update.
- The reply does not make commitments outside Murmur's authority (no new projects, no financial commitments, no legal commitments).

Murmur **must escalate to Michael** before replying when:

- The email requests a decision that only Michael can make.
- The email involves financial, legal, or contractual commitments.
- The email is from an unknown sender with a sensitive or unusual request.
- The email is adversarial, threatening, or legally significant.
- Murmur is uncertain about the appropriate response.

## 7. Reply Standards

- Replies must be professional, clear, and concise.
- Replies must not over-promise or make commitments beyond current approved goals.
- Replies must include a clear next step when applicable.
- Replies should reflect the relationship context from the contact file.
- Replies to VIPs should be especially careful and thorough.

## 8. Follow-Up Behavior

- If Murmur sends an email that requires a response, log a follow-up entry in the project's `waiting_on` section (or the contact file if no project is linked).
- Follow the timing discipline in `02_playbook.md` Section 4.
- Follow-up emails should be polite, reference the original message, and restate the ask.

## 9. Communication Logging

Every email interaction must be logged in two places:

1. **Contact file**: Update `contact_history_summary` and `last_contact_date` in the sender's contact file.
2. **Project file** (if applicable): Add an entry to the project's `communication_log` with:
   - Date
   - Direction (inbound/outbound)
   - Contact name
   - Subject
   - One-line summary
   - Any action items generated

If the email is not linked to a project, only the contact file is updated.

### Write Order and Failure Recovery

Dual logging must follow this sequence:

1. **Write to the contact file first.** The contact file is the primary record.
2. **Then write to the project file** (if applicable).
3. **If the second write fails:**
   - Do not roll back the contact file write. The contact record stands.
   - Add an entry to `state/pending_approvals.md` with type `logging_repair`, noting which project file needs the missing log entry.
   - On the next operating cycle, Murmur must attempt to complete the failed write.
4. **If the first write fails:**
   - Do not attempt the second write.
   - Retry the contact file write on the next operating cycle.
   - If the retry also fails, escalate to Michael as an `operational_issue`.

This ensures no interaction is silently lost. A partial write is always recoverable.

## 10. Isolated Email Session Rules

Isolated email sessions are spawned by the IMAP IDLE daemon to handle incoming email in real time. They operate independently from the main session and start with no conversation history.

Because isolated sessions have no memory of prior sessions, they must reconstruct context from the repos on every run.

### 10.1 Required Startup Sequence

Before taking any action, an isolated session must:

1. Read this file (`03_email_handling.md`) in full.
2. Read `04_escalation_rules.md`.
3. Load `state/contacts_index.md` and `state/vip_list.md`.
4. Load `state/active_projects_index.md`.

### 10.2 Duplicate Reply Check

Before replying, the isolated session must check via himalaya whether murmur has already replied to this email thread. If a reply from murmur exists, the session must log the new email (Section 9) but must NOT send another reply. Flag the thread for main session review if it requires further attention.

### 10.3 Processing the Email

The isolated session follows the same flow as any email handling:

1. Identify sender (Section 2) — look up in `contacts_index.md`, load contact file if found. If sender is unknown, perform the full dedup check (email, domain, name) per Section 2 before creating a new contact file.
2. VIP check (Section 3) — if VIP, notify Michael on Telegram immediately using the format in `04_escalation_rules.md` Section 4.
3. Classify by project (Section 4) — match against active projects.
4. Determine response action (Section 6 reply rules apply fully).
5. If the email does not match any project and implies a new opportunity, do NOT create a project proposal. Log the email in the contact file and flag for main session review.

### 10.4 Reply Limit

An isolated session may send at most **1 reply** per invocation. If the conversation requires further engagement, the session must log what happened and flag the thread for main session review.

### 10.5 Logging

All logging follows Section 9, including write order (contact file first, then project file) and failure recovery via `state/pending_approvals.md`. Commits must follow the commit standards in `01_constitution.md` Section 8 using the `[email]` category.

If the reply creates a follow-up expectation, log it in the project's `waiting_on` section per Section 8 of this file.

### 10.6 Authority Restrictions

Isolated sessions operate under the same authority boundaries as the main session (`01_constitution.md` Section 3), with these additional restrictions:

- Must NOT create, modify, or delete cron jobs.
- Must NOT publish, create, or modify public-facing files or URLs.
- Must NOT modify governance files, state indexes (other than `logging_repair` entries), or templates.
- Must NOT start work on project goals.

Any action beyond replying, logging, and escalating is outside the scope of an isolated email session.

### 10.7 When in Doubt

If the isolated session is uncertain about any aspect of handling — whether to reply, how to reply, or whether escalation is needed — it must default to: log the email, notify Michael on Telegram, and take no other action.
