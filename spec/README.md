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
| Inverter + battery | SolarEdge hybrid + storage | native templates `solaredge-hybrid` (Modbus TCP) |
| Grid meter | SolarEdge SE-MTR 3-phase inline meter at fuse box (commissioning status unknown) | native template, read via inverter |
| Backup meter reading | Tasmota Hichi IR head on utility meter | native template `tasmota-sml` (validation only, not wired into evcc initially) |
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
3. **Remote access via FritzBox WireGuard.** evcc stays LAN-only; both phones get VPN
   profiles. No public exposure, no auth/TLS hardening project.
4. **Vehicle handling.** One wallbox, two cars, auto-detection via the cars' cloud APIs.
   Fallback logic: only two cars exist, so "not the detected one" = the other one.
5. **Optimization target.** Solar surplus first; remaining need from cheapest Rabot hours;
   home battery coordinated so it doesn't dump into the car; manual override always wins.

## Phases

| Phase | Content | Doc |
|---|---|---|
| 0 | Discovery: verify SolarEdge meter/storage commissioning, Modbus TCP, car accounts, cFos network access | [01-phase1-deployment.md](01-phase1-deployment.md) §1 |
| 1 | evcc on the Pi (Docker Compose), all devices configured, PV surplus charging working (= milestone 1), WireGuard remote access | [01-phase1-deployment.md](01-phase1-deployment.md) |
| 2 | Only if needed: Rabot tariff template upstream, UI presets, Home Assistant | [02-phase2-backlog.md](02-phase2-backlog.md) |

Model usage strategy for working with Claude on this project: [03-claude-model-guide.md](03-claude-model-guide.md)
Household how-to (German): [RUNBOOK.de.md](RUNBOOK.de.md)
