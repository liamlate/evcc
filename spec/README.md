# Home EV Charging Automation — Project Spec

Owner: Liam · Baseline: this evcc fork (kept as a clean mirror of upstream) · Last updated: 2026-07-11

## Goal

Charge two cars and the home battery as cheaply as possible: **solar surplus first, then the
cheapest grid hours (Rabot dynamic tariff)**, with a one-tap manual "charge now" override.
Everything controllable from both phones, from anywhere, via VPN.

## Hardware & accounts

| Component | Details | evcc support |
|---|---|---|
| Wallbox | cFos (single charge point, both cars) | native template `cfos` (needs evcc sponsor token) |
| Inverter + battery | SolarEdge hybrid; **storage installed but NOT yet connected** | native templates `solaredge-hybrid` (Modbus TCP); battery joins in Phase 1b |
| Grid meter | SolarEdge SE-MTR 3-phase inline meter at fuse box — **installed but NOT yet connected** | native template, read via inverter; joins in Phase 1b |
| Interim grid meter | Tasmota Hichi IR head on utility meter | native template `tasmota-sml` — **primary grid meter in Phase 1a** (needs signed live power, see discovery) |
| EV 1 | VW ID.3 Pro (11 kW AC, 3-phase) — **replaced by Cupra Born in ~6 months** | `vw` template now, `seat-cupra` later (same VW-group API — config swap only) |
| EV 2 | BMW 520e PHEV (3.7 kW AC, 1-phase) | `bmw` template (ConnectedDrive) |
| Tariff | Rabot (EPEX spot + markup) | **no native template** — workaround via `energy-charts-api` + price formula; candidate upstream contribution |
| Host | Raspberry Pi 4, 2 GB RAM | plenty (evcc uses < 200 MB) |
| Router | FritzBox | built-in WireGuard for remote access |
| Users | Liam + partner (non-technical) | stock evcc UI, German runbook |

## Key decisions (from spec interview, 2026-07-11)

1. **Phased approach.** Deploy vanilla evcc first; only build custom features for gaps that
   survive real usage. The stock UI already has Fast mode ("charge now"), charging plans
   ("full by 7:00"), and price-limit charging — likely covering most of the wishlist.
2. **Fork stays a mirror.** No changes to evcc source unless a real gap appears. If one does,
   prefer a companion page against evcc's REST API, or an upstream contribution — never a
   long-lived UI fork.
3. **Remote access — split approach** (updated 2026-07-11). evcc stays LAN-only, no port
   forwarding. Liam: FritzBox WireGuard (full LAN access for admin/maintenance). Partner:
   evcc's built-in Remote Access (included in sponsorship) — zero-friction dashboard URL,
   no VPN app. Sponsorship gets paid after the demo-token trial, so this is covered anyway.
4. **Vehicle handling.** One wallbox, two cars, auto-detection via the cars' cloud APIs.
   Fallback logic: only two cars exist, so "not the detected one" = the other one.
5. **Optimization target.** Solar surplus first; remaining need from cheapest Rabot hours;
   home battery coordinated so it doesn't dump into the car; manual override always wins.
6. **Two-stage Phase 1** (updated 2026-07-11): the SolarEdge meter and the battery are
   installed but not yet electrically connected/commissioned. **Phase 1a** runs evcc now
   with the hichi as grid meter and the inverter as PV meter — PV surplus charging without
   battery coordination. **Phase 1b** (after the installer visit) switches the grid meter to
   the SolarEdge SE-MTR and adds the battery. Battery-vs-car priority: decide with real
   data in 1b (open question).

## Phases

| Phase | Content | Doc |
|---|---|---|
| 0 | Discovery: hichi capabilities (critical path!), inverter Modbus TCP, car accounts, cFos network access; schedule installer for meter + battery | [01-phase1-deployment.md](01-phase1-deployment.md) §1 |
| 1a | evcc on the Pi (Docker Compose), hichi as grid meter, PV surplus charging working (= milestone 1), remote access | [01-phase1-deployment.md](01-phase1-deployment.md) |
| 1b | After installer: switch grid meter to SolarEdge SE-MTR, add battery + priority config | [01-phase1-deployment.md](01-phase1-deployment.md) §6 |
| 2 | Only if needed: Rabot tariff template upstream, UI presets, Home Assistant | [02-phase2-backlog.md](02-phase2-backlog.md) |

Model usage strategy for working with Claude on this project: [03-claude-model-guide.md](03-claude-model-guide.md)
Household how-to (German): [RUNBOOK.de.md](RUNBOOK.de.md)
