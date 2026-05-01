# AUDIT-NOTES.md — refresh pass 2026-05-01

Notes from the audit branch `audit/2026-05-01-refresh`. Things to confirm or punt before merging.

## Decisions made (please confirm)

1. **Persona-rename convention.** Throughout the spec I used:
   - `Murmur` / `elias` / `mibb` → `the agent`
   - `Michael` → `the principal`
   - `elias-ops` / `elias-management` / `murmur-ops` → `<agent>-ops` (with "the ops repo" in prose)
   - `clawbot-config` → `<agent>-runtime-config` in prose, with the directory-name `clawbot-config/` preserved in the spec repo for historical continuity
   - "Telegram" baked in as the channel → "the configured channel" (Telegram is now an example, not a requirement)

   If you'd rather keep `elias` / `Michael` as worked-example names rather than fully generic, this is the most reversible decision — happy to do either. The full anonymisation feels right for a public spec but is more work to read for someone already used to the `elias`/`Michael` convention.

2. **Removed `05_murmur_protocol.md` entirely.** It was an instance-specific protocol (the elias / murmur network) with hardcoded references to `quietweb-org/elias`, `elias@mur-mur.at`, etc. Per the strict "no murmur-specific content" rule it had to go. If you want a generalised "agent discovery protocol" doc later, it would be a fresh write — not a rescue of this file.

3. **`about.md` left alone.** It's the Jekyll blog's about page and contains the blog's actual identity statement: "Working with byzo is written by mibb…". This is blog content, not spec content. Per the task instructions ("do NOT modify… about.md unless instance-specific leakage… minimal-touch fix"), I read it as blog voice and left it. If you'd prefer it anonymised too, it's a 4-line edit.

4. **`_includes/`, `_layouts/`, `index.md`, `_config.yml` left alone.** Same reasoning — blog plumbing. The `_includes/footer.html` and friends include real `mailto:murmur@mur-mur.at` subscribe links, but those go to the actual production mailbox. Anonymising them would break the blog.

## Structural questions that came up

5. **Naming inconsistency between AGENTS.md and the rest.** The pre-audit AGENTS.md used `elias-management` as the ops-repo name; the rest of the spec used `elias-ops`. I unified by removing both — AGENTS.md now says `<ops-repo>` and points at `elias-ops-config/ops-repo-guide.md` for the canonical name. If you'd prefer a single concrete name throughout, my recommendation is `elias-ops` (it matches the existing directory `elias-ops-config/` in this repo).

6. **`02_playbook.md` Section 5 (Heartbeat infrastructure checks).** The original playbook contained a multi-paragraph spec of how `restore-tools.sh` and the cron manifest should work, including a "governance preamble rule" about cron job message prefixes. That detail is now in `clawbot-config/openclaw-setup-guide.md` where it actually belongs. The playbook now points at the setup guide. The "governance preamble" rule was *not* carried forward because it's tied to the older isolated-session cron model — happy to add it back as a generalised rule if you still want it.

7. **`REPOS.md` vs the README's three-repo table.** I aligned both. The README is now the short overview, REPOS.md is the long form. If the README's table feels redundant, we can prune it to a one-liner pointer.

8. **`two-agent-topology.md` placement.** Put it at the root (linked from README and from setup guide). Could also live under a new `patterns/` directory if more pattern docs land later. Not urgent.

## Things I deliberately did not change

- `_posts/` — blog content, untouched per task constraints
- `_config.yml`, `_includes/`, `_layouts/`, `assets/`, `index.md`, `about.md` — blog plumbing, untouched per task constraints
- The historical directory names `clawbot-config/` and `elias-ops-config/` (these are kept; only the *content* inside was rewritten)

## Anonymisation issues found

- `about.md` (blog) leaks Michael/byzo/mur-mur.at/mibb — left alone as discussed
- `_includes/subscribe_cta.html`, `_layouts/post.html`, `_layouts/home.html`, `_includes/footer.html` all contain `mailto:murmur@mur-mur.at` subscribe links — left alone as discussed
- The pre-audit `02_playbook.md` Section 5 contained a hardcoded reference to `workspace/elias-management/` in a "governance preamble" example — removed (see point 6 above)
- The pre-audit README "Best Practices: Building the System with Two AIs" section talked about "elias" running on OpenClaw and "the public `the-elias-codex` repo". Cleaned up references to specifics; the rest of the section reads as generic build-process advice now.

## Suggested next step

Open the branch, skim the README + the two new/rewritten guides, and either:
(a) merge to main as-is, or
(b) push back any of the persona-rename / `about.md` decisions above and I'll re-run.

If (a) and you want a v2 pass later: the obvious next bit is to add a small `patterns/` or `examples/` directory with a worked example of `cron-manifest.json`, an example `restore-tools.sh` skeleton, and an example `HEARTBEAT.md`. They're referenced everywhere now but nowhere demonstrated in the spec repo.
