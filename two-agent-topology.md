# Two-Agent Topology

A reusable pattern: one principal, two agents. Different roles, different containers, different ops repos, **shared spec repo (this one)**.

This document describes when and why to split into two agents, how to draw the line between them, and what to share vs. what to keep separate.

---

## When to use this pattern

The default setup is one agent: a single OpenClaw container, one ops repo, one runtime-config repo, one channel. That is enough for most cases.

You probably want a second agent when:

1. **Operational load conflicts with public voice.** The operator-side agent — the one handling email, projects, follow-ups, contacts — accumulates a lot of context that should not appear in public output. Mixing public posting into the same session leads to leaks ("oh I'll just check this real quick" → publishes draft).
2. **Different audiences require different personas.** The operator speaks to vendors, collaborators, the principal — formal-ish, business-shaped. The public-voice agent writes blog posts, podcasts, social content — looser, voicey, performative.
3. **You want clean separation of authority.** The operator can send email on the principal's behalf inside an authority boundary. The public-voice agent should never reply to email at all. Two boxes, two sets of rules, two channels.
4. **You want one of them to fail without taking the other down.** A bad LLM session in the public-voice agent (e.g. a publish-then-rollback loop) should not interfere with email handling. Different containers = different blast radii.

If none of those apply, you do not need this pattern. One agent is simpler and cheaper.

---

## Roles

The two agents play distinct roles.

### Agent A — the operator

- **Identity:** an autonomous project operator. The default persona this whole spec is written for.
- **Owns:** projects, contacts, follow-ups, email, escalation to the principal, learning loop.
- **Channels:** chat with the principal, IMAP+SMTP for email, optionally a Slack/Telegram bot for notifications.
- **Authority:** as defined in `01_constitution.md` Section 3 — can reply to email, update files, propose new work; needs approval for new projects, public output, persistent infrastructure.
- **Output style:** concise, factual, business-shaped. Reports, summaries, escalations.

### Agent B — the public voice

- **Identity:** a writer / performer. May or may not also be an autonomous operator.
- **Owns:** blog posts, podcast scripts, social content, public-facing site copy. *None* of the operator's data.
- **Channels:** chat with the principal (for drafting), Git for publishing, possibly a CMS/RSS pipeline.
- **Authority:** can draft public content; **never** publishes without principal approval; **never** sends email; **never** has access to the operator's contacts or project data.
- **Output style:** voicey, narrative, longer-form. Allowed to be weirder.

The principal sits between them and routes work to whichever role fits the request.

---

## Topology

### Two containers

Each agent runs in its own OpenClaw container. They do not share a workspace. They do not have file-system access to each other.

```
container-A (operator)              container-B (public voice)
  ~/.openclaw/                        ~/.openclaw/
    workspace/                          workspace/
      <agent-A>-ops/   ← private          <agent-B>-ops/   ← private
      ...                                 ...
```

This is the strongest form of separation. Compromising one does not compromise the other.

### Two ops repos

Each agent has its own private ops repo. Their structures are identical (both follow `elias-ops-config/ops-repo-guide.md`); their contents are completely separate.

| Aspect | Agent A (`<agent-A>-ops`) | Agent B (`<agent-B>-ops`) |
|---|---|---|
| Projects | Operational projects | Content projects (episodes, series, drafts) |
| Contacts | Vendors, collaborators, prospects | Public correspondents only (subscribers, mentions, fan mail if relevant) |
| State | `pending_approvals.md` for ops decisions | `pending_approvals.md` for content decisions |
| Reviews | Weekly ops review, learning review | Weekly content review |
| Memory | Daily logs of operator work | Daily logs of writing work |

Hard rule: **agent B does not read agent A's ops repo at runtime.** No git-clone-each-other, no shared volume, no cross-channel scraping. If they need to share a fact, the principal carries it across.

### Two runtime-config repos (or one merged one)

You can keep two separate runtime-config repos for symmetry, or share a single one if both containers run on the same VPS and share Caddy / docker-compose. The latter is fine — secrets are container-scoped anyway, and host config isn't sensitive in the same way as ops data.

### One shared spec repo

Both agents reference **this repo** (the public spec) as the canonical pattern. Both have working copies of the four governance files in their respective ops repos. Both follow the same constitutional rules — the difference is in *role*, not in *governance*.

This keeps the bar high: even the public-voice agent has the same authority boundaries, escalation rules, and learning loop as the operator. No "creative-mode bypass" of the constitution.

---

## What to share, what to keep separate

| Thing | Share? | Notes |
|---|---|---|
| Spec repo | **Yes** | Both agents read from it; both contribute generalised improvements back to it. |
| Identity files (`SOUL.md`, `IDENTITY.md`) | **No** | Different agents, different personas. Each has its own. |
| `USER.md` | **Maybe** | The principal is the same person. A shared `USER.md` keeps facts about the principal aligned. Practically, each agent maintains its own copy. |
| `MEMORY.md` (facts) | **No** | Each agent's facts file describes its own infra and contacts. There is overlap (the principal, their timezone, their preferred channels) but the file is per-agent. |
| Project files | **No** | Strictly per-agent. Operator projects ≠ content projects. |
| Contact files | **No** | Two agents talk to disjoint sets of people. If an operator contact happens to also be a public correspondent, the contact appears in both repos with separate communication logs. |
| Memory (`memory/YYYY-MM-DD.md`) | **No** | Each agent's daily log is its own. |
| Learning candidates | **No** by default; **Yes** for governance changes | Operational learnings stay per-agent. Governance-pattern changes (e.g. a new rule that should apply to both) get proposed in either, then end up in the spec, then sync to both ops repos. |
| Cron manifest | **No** | Per-agent. |
| Runtime scripts | **Mostly per-agent** | The agents may share `restore-tools.sh` *structure* via the spec, but each instance has its own copy with instance-specific paths. |

---

## Memory-sharing options (when you want some bridging)

The strict default is no sharing. But sometimes the principal wants the agents to know each other exists and reference each other's work. Three options, in increasing coupling:

1. **Manual carry.** The principal pastes facts across (e.g. tells agent B "agent A just landed PROJ-014, you might want to write about it"). Cleanest, slowest.
2. **Read-only spec mention.** Each agent's `USER.md` includes a paragraph like *"You also have a sibling agent named B who handles the public voice. You do not share data with B; if B needs something, the principal will route it."* No file-level sharing, just awareness.
3. **Curated pull-only feed.** Agent A maintains a single curated file in its ops repo (e.g. `state/public-summary.md`) with the small subset of project state that's safe for B to know about — at most one paragraph per project, no contact data. The principal periodically copies that file across when relevant. Still no automated sync; the principal is the gate.

Anything tighter than (3) (e.g. agent B reading agent A's repo on a cron) starts to defeat the point of the split. Don't do it unless there's a real reason and explicit principal approval.

---

## Setup checklist

To stand up a second agent next to an existing operator:

1. **Decide the role.** Write down what agent B owns and what it does *not* own. Get the principal to approve the boundary.
2. **Create a second OpenClaw container** on the same VPS or a different one. Different data volume, different web-gateway hostname (e.g. `b.<domain>`), different bot token.
3. **Create a private ops repo** `<agent-B>-ops` following `elias-ops-config/ops-repo-guide.md`. Empty state indexes, empty projects, empty contacts.
4. **Copy the four governance files** from the spec repo into the new ops repo.
5. **Choose a different `IDENTITY.md` and `SOUL.md`.** Same governance, different voice.
6. **Wire the boot hook** to `restore-tools.sh` exactly as for agent A.
7. **Configure a different escalation channel** if you want messages from each agent to be visually distinguishable (e.g. agent A on Telegram, agent B on a different Telegram bot — the principal sees clearly which one is talking).
8. **Test the boundary.** Send agent B an email. It should refuse to handle it (or ignore it entirely if no IMAP daemon is wired up). Ask agent A to publish something publicly. It should refuse.

---

## When this pattern goes wrong

A few failure modes worth flagging:

- **Drift in governance.** Both agents are supposed to follow the same constitution and playbook, but updates land in only one ops repo. Mitigation: when a governance file changes in either ops repo, sync it to the spec immediately, then have the other agent pull the spec change.
- **Cross-channel leakage.** Agent B drafts a post that quotes operator data agent A handled. The principal must be the gate; agent B should never see operator data automatically.
- **Subtle persona collapse.** Over time, agent B starts replying like an operator (because it has read the same spec) and agent A starts reaching for narrative voice (because the principal paid more attention to B's outputs in recent reviews). Mitigation: run separate Monday reviews per agent, evaluate persona explicitly.
- **Two operators by accident.** If both agents end up handling email, you have two operators, not an operator + a writer. Decide which one owns inbound mail and shut the other one's mail handling off explicitly (no IMAP daemon wired up, governance file forbids replies).

---

## See also

- `elias-ops-config/ops-repo-guide.md` — structure each ops repo follows
- `clawbot-config/openclaw-setup-guide.md` — runtime setup for each container
- `01_constitution.md` — governance rules both agents share
- `04_escalation_rules.md` — how each agent talks to the principal
