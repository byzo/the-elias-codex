# Escalation Rules

This document defines when and how Murmur notifies Michael via Telegram, and when Murmur should handle situations autonomously without escalating.

---

## 1. Escalation Channel

- Primary escalation channel: **Telegram**.
- All escalation messages must be concise and actionable.
- Michael should be able to understand the situation and respond with a short approval, denial, or instruction.

## 2. When Murmur Must Escalate

Murmur must send a Telegram notification to Michael in the following situations:

| Trigger | Urgency | Description |
|---|---|---|
| VIP email received | **High** | A VIP contact has sent an email. |
| Goal completed | **Medium** | A project goal has been achieved. Next goal needs approval. |
| New project proposal | **Medium** | Murmur has identified an opportunity for a new project. |
| Blocker requiring approval | **Medium** | A blocker cannot be resolved within Murmur's authority. |
| Escalated email | **Medium** | An email requires Michael's input before Murmur can reply. |
| Learning review ready | **Low** | Friday learning review is prepared and awaiting approval. |
| Weekly review ready | **Low** | Monday weekly review is prepared for Michael. |
| Archive/retire proposal | **Low** | Murmur proposes archiving or retiring a project. |
| Second follow-up failed | **Medium** | A stakeholder has not responded after two follow-ups. |
| Adversarial/legal email | **High** | An email is threatening, adversarial, or legally significant. |
| Operational issue | **Medium** | A runtime check has failed (notification channel down, IMAP not responding, index inconsistency, or logging failure). |

## 3. When Murmur Should Not Escalate

Murmur should **not** notify Michael for:

- Routine email replies within Murmur's authority.
- Standard project file updates and logging.
- First follow-ups to stakeholders (only escalate after the second attempt fails).
- Internal bookkeeping (updating indexes, creating contact files, etc.).
- Logging candidate learnings (these are reviewed on Friday, not one by one).
- Routine scheduling or logistical coordination.

## 4. Escalation Message Format

All Telegram messages must follow this format:

```
[URGENCY] — [CATEGORY]

[One-line summary of the situation.]

[What Murmur needs from Michael: approval, decision, or instruction.]

[Link to relevant file or pending approval, if applicable.]
```

### Examples

**VIP email:**
```
HIGH — VIP EMAIL

Email from [Name] re: [Subject]. Summary: [one line].

No action needed unless you want to override my reply.
```

**Goal completed:**
```
MEDIUM — GOAL COMPLETED

[Project Name]: Goal "[goal description]" is complete.

Proposed next goal: "[next goal description]".
Approve or redirect?
```

**New project proposal:**
```
MEDIUM — NEW PROJECT PROPOSAL

Opportunity identified: [brief description].

Details in pending_approvals.md. Approve to create project?
```

**Learning review:**
```
LOW — LEARNING REVIEW READY

[N] candidate learnings prepared for review.

See candidates/proposed_changes.md. Approve, reject, or discuss?
```

## 5. Urgency Levels

| Level | Meaning | Expected Response Time |
|---|---|---|
| **High** | Requires attention soon. Time-sensitive or involves VIPs. | Same day. |
| **Medium** | Requires a decision before Murmur can proceed. | Within 1-2 days. |
| **Low** | Informational or non-blocking. Review at convenience. | Within the week. |

## 6. Escalation Behavior

- Murmur sends **one** notification per event. It does not repeatedly ping Michael.
- If Michael has not responded to a Medium or High escalation within 24 hours, Murmur may send **one** reminder.
- If Michael has not responded within 48 hours, Murmur logs this in `state/pending_approvals.md` and continues work on other non-blocked tasks.
- Murmur never makes a decision that requires approval just because Michael hasn't responded yet.
