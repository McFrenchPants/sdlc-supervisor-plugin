# sdlc-supervisor

A Claude Code plugin that drives a project through **backlog → analysis →
design → task-packet implementation → verification → supervised release**,
behind one command: `/continue-development`.

Extracted from the `sdlc-supervisor` framework originally built inside
`spotify-jukebox` (see that repo's `docs/sdlc/` for the original design
history), and generalized so it can be installed into any project instead
of being copy-pasted and re-diverging per repo.

## What's in here

- `commands/continue-development.md` — the one command. Detects whether a
  project has adopted this framework yet (and initializes it if not),
  resumes in-progress work, or picks/scaffolds new work from the project's
  backlog, then drives it through implementation via subagents.
- `commands/sdlc-doctor.md` — a read-only diagnostic for the framework's own
  state (lifecycle-state sanity, hook wiring, owned-files existence).
- `agents/implementer.md`, `agents/verifier.md`, `agents/supervisor.md` —
  the three subagent roles. Implementer does scoped work against one task
  packet; verifier independently checks it; supervisor (only relevant for
  projects in `full` release mode) is the sole role that merges to
  production or reaches a live system.
- `hooks/hooks.json` + `hooks/sdlc-path-check.mjs` — a `PreToolUse` hook
  that denies an implementer's `Edit`/`Write` outside its active task
  packet's declared paths, before the write happens.
- `schemas/*.json` — JSON Schemas for `project.yaml`, `state.json`, task
  packets, completion reports, and approval records.
- `scripts/generate-task-packet.mjs`, `scripts/validate-state.mjs` — the
  packet generator (a text heuristic, not a code-aware tool — see its own
  header comment) and the lifecycle-transition validator.
- `templates/` — what `/continue-development`'s init step scaffolds into a
  newly-adopting project: `project.yaml.template`, a `CLAUDE.md` snippet,
  `APPROVAL_RECORDS.md`, and a `SUPERVISOR_RUNBOOK.template.md` skeleton.

## Installing

This repo is itself a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`
points at the plugin in `./`). From any project:

```
/plugin marketplace add <path-or-git-url-to-this-repo>
/plugin install sdlc-supervisor
```

Install at **user scope** if you want every project you work in to pick up
updates from this repo automatically; install at **project scope** if you
want a specific project to pin its own copy.

## Updating

Bump `.claude-plugin/plugin.json`'s `version` on every meaningful change to
this repo. Projects that installed with auto-update enabled for this
marketplace pick up a new version automatically; without an explicit
version bump, installs track the latest commit on every push instead (fine
for solo use, riskier if you want deliberate rollout). Third-party/private
marketplaces default to auto-update **off** — enable it explicitly per
install if you want zero-touch propagation.

Commands and agents are picked up live from the installed plugin, so an
update to `commands/continue-development.md` or the agent files reaches
every installed project the next time it runs, no per-project copy to
maintain. **The hook script, JSON schemas, and generator scripts are
different** — the init step *copies* those into each project (see "Known
gaps" below), so a change to `hooks/sdlc-path-check.mjs` or `schemas/*.json`
here does **not** automatically reach a project that already initialized.
Re-copy those files by hand (or via a future `/continue-development
--update-framework-files` step, not yet built) when you want an existing
project to pick up a framework-level fix to them.

## Known gaps / things to verify before trusting this in production

- **Hook auto-wiring is unconfirmed.** It isn't clearly documented whether
  a plugin-bundled `hooks/hooks.json` gets wired into a project
  automatically on install, or needs a manual step per project. Until
  verified, `continue-development.md`'s init step falls back to copying
  `hooks/sdlc-path-check.mjs` into the target project's `.claude/hooks/`
  and writing a project-local `.claude/settings.json` entry — which works
  regardless of the answer, at the cost of that file no longer being
  centrally updated (see "Updating" above). Test this for real (install
  into a scratch repo, try an out-of-scope implementer write, see whether
  it's denied without the fallback) and simplify the init step once known.
- **`CLAUDE_PLUGIN_ROOT` availability in a command's own execution context**
  (as opposed to a hook's) is assumed but not independently verified here.
  The init step's instructions account for this being uncertain by telling
  the running session to ask the user if it can't resolve the plugin's
  installed path.
- **`KNOWN_DIRS` in `scripts/generate-task-packet.mjs` ships with generic
  placeholder defaults.** The single highest-value post-init customization
  is editing that list to match a project's real source layout — the init
  step in `continue-development.md` says to do this, but it's easy to skip
  and the packet generator degrades quietly (not incorrectly) if you do.
- This has only been dogfooded inside the single project it was extracted
  from. Test it end-to-end in a second, unrelated project before relying on
  it for anything that matters.
