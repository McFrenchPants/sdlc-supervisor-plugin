# Supervisor runbook: merge, push, and live verification

This document is for the **supervisor role only** — see
`agents/supervisor.md` and this project's `CLAUDE.md` for why that boundary
exists. An agent implementing a feature/fix should never need anything in
this file; it should stop at this project's own testing doc and hand off.

Everything here reaches outside this repo/machine: the git remote, and
whatever live systems this project actually depends on
(`.sdlc/project.yaml`'s `release.live_systems`: {{LIVE_SYSTEMS}}). Treat
every command in this file as something to run deliberately and sparingly,
not to loop or poll with.

**This file was scaffolded with placeholders during `/continue-development`'s
init step and still needs a human (or a deliberate follow-up task) to fill
in the real commands for this project's actual live systems below — the
scaffolding cannot know those. Don't let an agent invent plausible-looking
commands for a system it doesn't actually have verified access to.**

## Guardrails (read before running anything below)

1. **Local tests pass first, always.** Never merge/push/verify-live on a
   change that hasn't cleanly passed this project's own test/build checks.
2. **Verify, don't iterate, against the live system.** Live checks are for
   confirming an already-tested change actually deployed correctly — not a
   debugging loop. If something's broken live, the fix happens locally
   (edit, retest, redeploy), not by repeatedly poking the live instance.
3. **Minimize call count.** One status check to confirm a deploy landed is
   normal. Polling every few seconds, or re-running the same check "just to
   be sure" more than once or twice, is not — if you need to wait for
   something, wait a sensible real amount of time once, not in a tight loop.
4. **Read-only by default.** Prefer a status/health check over any command
   that changes state. Only take a state-changing action when the task
   actually requires it, and say what you're about to do and why before
   doing it.
5. **Never run a destructive or host-/administration-level action** against
   any live system this project merely depends on, without explicit,
   per-instance user confirmation — administering the underlying platform
   is out of scope for this project's supervisor role entirely; if one
   seems necessary, stop and ask instead.
6. **Log what you actually did.** When a supervisor session performs a live
   merge/push/deploy-verify, note the real commands run in the backlog entry
   or commit/PR description for that change, matching whatever
   incident-documentation convention this project already has.

## Merge & push workflow

Two-stage, per `.sdlc/project.yaml`'s `release` block: feature branches fork
from and merge into `{{INTEGRATION_BRANCH}}` (routine, automatic); that
branch only promotes to `{{PRODUCTION_BRANCH}}` on an explicit live
instruction. Both stages still require this project's own tests fully green
first.

### Stage 1 — feature branch → `{{INTEGRATION_BRANCH}}` (routine, no approval record needed)

```bash
# From a feature branch, tests already green:
git add <specific files>              # never `git add -A`/`.` blindly
git commit -m "..."

git checkout {{INTEGRATION_BRANCH}}
git merge --no-ff <feature-branch> -m "Merge branch '<feature-branch>'"
git push origin {{INTEGRATION_BRANCH}}
git branch -d <feature-branch>
```

Do this as the normal way a finished task/proposal wraps up — no need to
wait for the user to separately ask for this merge, and no approval record
to write for it.

### Stage 2 — `{{INTEGRATION_BRANCH}}` → `{{PRODUCTION_BRANCH}}` (production; requires a live instruction + approval record)

```bash
git checkout {{PRODUCTION_BRANCH}}
git merge --no-ff {{INTEGRATION_BRANCH}} -m "Merge {{INTEGRATION_BRANCH}} into {{PRODUCTION_BRANCH}}: <summary>"

# TODO (fill in during a deliberate follow-up, not invented by an agent):
# if this project's integration and production branches carry any
# environment-specific identity that differs between them (a config file's
# name/slug/port, a version string, feature flags), restore the production
# side of those here before pushing -- a plain merge silently carries the
# integration branch's side of any changed line into production.

git push origin {{PRODUCTION_BRANCH}}
```

Never push a production promotion without the approval-record check in
`docs/sdlc/APPROVAL_RECORDS.md` passing first — this is the step that
reaches real users/guests/customers.

## Live systems: read-only checks (preferred)

TODO: fill in the actual health-check / status commands for this project's
real live systems ({{LIVE_SYSTEMS}}) here, once they're known and verified
working. Until this section is filled in, the supervisor role should ask
the user for the right command rather than guessing one.

## Live systems: state-changing actions (use deliberately, say why first)

TODO: fill in the actual restart/redeploy/install commands for this
project's real live systems here, along with any system-specific guardrails
(credentials that are single-purpose, access that must be requested fresh
each time, host-level actions that are permanently out of scope, etc.).
