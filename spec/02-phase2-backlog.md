# Phase 2 — Backlog (build only for proven gaps)

Rule from the spec interview: nothing here gets built until Phase 1 has run for a few weeks
and the gap is confirmed by actual usage. Re-evaluate this list then.

## 2.1 Rabot tariff template (most likely real gap)

evcc has no native `rabot` tariff template (checked against this fork, 2026-07-11). The
Phase 1 workaround (`energy-charts-api` + formula) gives correct *relative* prices, which is
all the optimizer needs — but absolute costs in the UI are only as good as the hand-maintained
formula.

**If the workaround annoys:** contribute a Rabot template upstream.
- Check first whether Rabot exposes a customer price API (like Tibber/Octopus do); if yes, a
  Go implementation under `tariff/` + template YAML under `templates/definition/tariff/`.
- If Rabot is strictly spot+markup with no API, a template wrapping the formula (like
  `octopus-de`-style provider templates) may still be accepted.
- Effort: small, well-scoped Go + YAML change; good first upstream contribution. Follow
  `CONTRIBUTING.md`; upstream requires templates + docs PR at evcc-io/docs.

## 2.2 Simplified control page ("easy buttons")

Deferred: the stock UI's mode toggle + plan dialog may already be simple enough. If the
partner-usability test fails in practice:
- **Preferred shape:** tiny static companion page (few big buttons: "Jetzt laden", "Bis 7 Uhr
  voll", "Nur Solar") calling evcc's REST API (`/api/loadpoints/0/mode`,
  `/api/vehicles/<v>/plan/...`). Hosted on the Pi next to evcc; no fork changes.
- Also evaluate first: iOS/Android shortcuts hitting the REST API directly — zero code hosting.

## 2.3 Home Assistant (door open, not before)

Decision: not now. When other home automation arrives:
- Run HA as a second container on the Pi (2 GB is workable but tight — watch memory; a Pi 5 /
  more RAM is the upgrade path).
- evcc integrates via the official HA integration (REST) and/or MQTT (`mqtt:` block in
  evcc.yaml + a broker container).

## 2.4 Cupra Born swap (~6 months)

Not a project, a config edit: replace the `vw` vehicle block with `template: seat-cupra`,
new credentials, `capacity: 58` (Born 58 kWh) — done.

## Explicitly rejected

- Public HTTPS exposure of the UI (WireGuard chosen instead).
- Forking/patching evcc's UI (fork maintenance cost; companion page pattern instead).
- Reading the PPC smart meter gateway (no consumer API).
