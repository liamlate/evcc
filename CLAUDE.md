# CLAUDE.md — this fork

This is **liamlate's fork of evcc**, used as the baseline for a home EV-charging automation
project. It is intentionally kept as a **clean mirror of upstream** — do not modify evcc
source, UI, or templates unless a task explicitly says so.

## Start here

All project context lives in [`spec/`](spec/):

| Doc | Read when |
|---|---|
| [`spec/README.md`](spec/README.md) | Always first — goals, hardware, decisions, phases |
| [`spec/01-phase1-deployment.md`](spec/01-phase1-deployment.md) | Deploying/configuring evcc on the Pi, WireGuard, acceptance tests |
| [`spec/02-phase2-backlog.md`](spec/02-phase2-backlog.md) | Feature work (Rabot tariff, companion UI) — check it's actually unblocked |
| [`spec/03-claude-model-guide.md`](spec/03-claude-model-guide.md) | Which Claude model to use per task |
| [`spec/RUNBOOK.de.md`](spec/RUNBOOK.de.md) | German end-user how-to (keep updated when behavior changes) |

## Rules for working in this repo

1. **Fork stays a mirror.** Config lives on the Pi, docs live in `spec/` — the only in-repo
   code change ever anticipated is an upstream-quality Rabot tariff template (see backlog).
2. **Never remove or bypass the sponsorship check** (`util/sponsor/`). That module is not
   MIT-licensed (all rights reserved) — patching it out violates the license. The project
   pays for a sponsor token.
3. For upstream evcc development conventions, see [`AGENTS.md`](AGENTS.md) and
   [`CONTRIBUTING.md`](CONTRIBUTING.md) — relevant only for Phase 2 upstream contributions.
4. Deployment targets a Raspberry Pi 4 (2 GB) on the home LAN — commands in specs are meant
   to be run there by the user, not in this sandbox.
