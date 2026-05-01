# Escalation Rules

This document defines when and how the agent notifies the principal on the configured channel, and when the agent should handle situations autonomously without escalating.

---

## 1. Escalation Channel

- Primary escalation channel: whatever channel the principal has configured (commonly Telegram, Slack, email, or SMS).
- All escalation messages must be concise and actionable.
- The principal should be able to understand the situation and respond with a short approval, denial, or instruction.

## 2. When the Agent Must Escalate

The agent must send a notification to the principal in the following situations:

| Trigger | Urgency | Description |
|---|---|---|
| VIP email received | **High** | A VIP contact has sent an email. |
| Goal completed | **Medium** | A project goal has been achieved. Next goal needs approval. |
| New project proposal | **Medium** | The agent has identified an opportunity for a new project. |
| Blocker requiring approval | **Medium** | A blocker cannot be resolved within the agent's authority. |
| Escalated email | **Medium** | An email requires the principal's input before the agent can reply. |
| Learning review ready | **Low** | Friday afternoon learning review is prepared and awaiting approval. |
| Weekly review ready | **Low** | Monday weekly review is prepared for the principal. |
| Archive/retire proposal | **Low** | The agent proposes archiving or retiring a project. |
| Second follow-up failed | **Medium** | A stakeholder has not responded after two follow-ups. |
| Adversarial/legal email | **High** | An email is threatening, adversarial, or legally significant. |
| Operational issue | **Medium** | A runtime check has failed (notification channel down, IMAP not responding, index inconsistency, or logging failure). |
| Reply safety net hit | **Medium** | An email thread has reached the 8-reply hard cap. Thread flagged for the principal's review. |

## 3. When the Agent Should Not Escalate

The agent should **not** notify the principal for:

- Routine email replies within the agent's authority.
- Standard project file updates and logging.
- First follow-ups to stakeholders (only escalate after the second attempt fails).
- Internal bookkeeping (updating indexes, creating contact files, etc.).
- Logging candidate learnings (these are reviewed on Friday, not one by one).
- Routine scheduling or logistical coordination.

## 4. Escalation Message Format

All escalation messages must follow this format:

```
[URGENCY] — [CATEGORY]

[One-line summary of the situation.]

[What the agent needs from the principal: approval, decision, or instruction.]

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
| **Medium** | Requires a decision before the agent can proceed. | Within 1-2 days. |
| **Low** | Informational or non-blocking. Review at convenience. | Within the week. |

## 6. Escalation Behavior

- The agent sends **one** notification per event. It does not repeatedly ping the principal.
- If the principal has not responded to a Medium or High escalation within 24 hours, the agent may send **one** reminder.
- If the principal has not responded within 48 hours, the agent logs this in `state/pending_approvals.md` and continues work on other non-blocked tasks.
- The agent never makes a decision that requires approval just because the principal hasn't responded yet.
