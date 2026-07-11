# Phase 1 — Deploy & Configure evcc

Milestone: **PV surplus charging works** — a plugged-in car charges automatically from excess
solar, visible in the evcc UI from both phones.

**Status 2026-07-11:** the SolarEdge SE-MTR grid meter and the battery are installed but
not yet electrically connected — **installer appointment within ~2 weeks**. Plan:
- **Now:** prep work only (§1 remaining items, §2 Pi setup, §4a WireGuard, car accounts,
  demo token) — none of it depends on the meter.
- **Phase 1a (optional fast-track / fallback):** hichi as grid meter + inverter as PV meter
  gives PV surplus charging before the installer visit. Verified 2026-07-11: the hichi
  outputs signed power (negative on export), so this works. Do it if prep finishes early or
  the appointment slips; otherwise skip straight to 1b.
- **Phase 1b (§6, after the installer):** full config with SE meter + battery.

⚠️ **Installer-visit agenda — hand this to the electrician:** (1) connect + commission the
SE-MTR meter, (2) connect + commission the battery, (3) **enable Modbus TCP on the inverter
(port 1502, via SetApp)**. Item 3 is easily forgotten and blocks everything evcc does —
without it a second visit is needed.

## 1. Discovery checklist (do this first, ~1 evening)

- [x] **Hichi delivers signed live power?** ✅ Verified 2026-07-11: outputs negative power
      on export. Qualifies as Phase 1a grid meter (evcc `tasmota-sml` template needs the
      `SML` group with `Total_in`, `Total_out`, `Power_curr` — confirm those exact JSON tag
      names in the Tasmota UI when configuring; script reference in
      `templates/definition/meter/tasmota-sml.yaml`).
- [x] **Installer appointment for SE-MTR meter + battery.** ✅ Scheduled within ~2 weeks of
      2026-07-11. **Give the installer the 3-point agenda from the top of this doc** —
      especially Modbus TCP.
- [ ] **Modbus TCP enabled on the inverter?** Unknown whether it can be enabled without
      installer access (newer firmware exposes it in the inverter's local web UI; otherwise
      SetApp). Try locally once; if not accessible, it's covered by the installer agenda.
      Without it there is no PV reading, so optional Phase 1a would be tariff-only.
- [ ] **PPC LTE smart meter gateway:** assume *not* accessible (German SMGWs expose no local
      consumer API). The SolarEdge meter is evcc's grid meter; the Tasmota Hichi stays as an
      independent sanity check. No action needed.
- [ ] **cFos wallbox reachable?** Find its IP in the FritzBox, open its web UI, enable
      **Modbus TCP** in the charging manager settings. Note IP + port.
- [ ] **Car accounts:** log into the Volkswagen app (ID.3) and My BMW app (520e) on the phone.
      If either login works, evcc can use the same credentials. Check whether the VW "We
      Connect" data contract is still active (SoC readout needs it); renew if expired — the
      basic tier is free.
- [ ] **Rabot pricing formula:** from the contract/bill, note the per-kWh composition:
      EPEX spot + procurement markup + grid fees + taxes/levies. Needed for the tariff config.
- [ ] **evcc sponsor token:** the cFos template requires sponsorship. Get a trial or regular
      token at https://sponsor.evcc.io (a few €/month, funds the project).
- [ ] **FritzBox:** give the Pi, inverter, and wallbox **static DHCP leases** (Home Network →
      Network → device → "always assign the same IPv4 address").

## 2. Pi setup (Docker Compose)

Docker keeps the install reproducible and leaves the door open for Home Assistant later.
Raspberry Pi OS Lite 64-bit, then:

```bash
curl -fsSL https://get.docker.com | sh
mkdir -p ~/evcc && cd ~/evcc
```

`docker-compose.yml`:

```yaml
services:
  evcc:
    image: evcc/evcc:latest
    restart: unless-stopped
    ports:
      - "7070:7070"
    volumes:
      - ./evcc.yaml:/etc/evcc.yaml
      - evcc-data:/root/.evcc
    environment:
      - TZ=Europe/Berlin
volumes:
  evcc-data:
```

Update path: `docker compose pull && docker compose up -d` (evcc releases ~monthly).

## 3. evcc.yaml skeleton (Phase 1a — hichi as grid meter, no battery)

Fill in IPs/credentials from discovery. Validate with
`docker compose run --rm evcc -c /etc/evcc.yaml configure` errors or start and check the logs.
evcc also has a UI-based configuration wizard for most of this — using it instead of hand-written
YAML is fine; the file below documents the target state either way.

```yaml
network:
  port: 7070

sponsortoken: # from sponsor.evcc.io

interval: 30s

meters:
  - name: grid                 # Phase 1a: hichi on the utility meter
    type: template
    template: tasmota-sml
    usage: grid
    host: <hichi-ip>
    user: admin
    password: <tasmota-password>
  - name: pv
    type: template
    template: solaredge-hybrid
    usage: pv
    host: <inverter-ip>        # Modbus TCP, port 1502
    port: 1502
  # Phase 1b (after installer) — replace the grid meter above with this and add battery:
  # - name: grid
  #   type: template
  #   template: solaredge-hybrid
  #   usage: grid
  #   host: <inverter-ip>
  #   port: 1502
  # - name: battery
  #   type: template
  #   template: solaredge-hybrid
  #   usage: battery
  #   host: <inverter-ip>
  #   port: 1502

chargers:
  - name: wallbox
    type: template
    template: cfos
    host: <wallbox-ip>

vehicles:
  - name: id3                  # swap to template `seat-cupra` when the Born arrives
    type: template
    template: vw
    title: ID.3 Pro
    user: <vw-account-email>
    password: <vw-password>
    capacity: 58
  - name: bmw
    type: template
    template: bmw
    title: BMW 520e
    user: <connecteddrive-email>
    password: <bmw-password>
    capacity: 12               # usable PHEV battery
    phases: 1                  # 520e charges single-phase, max 16 A (3.7 kW)

loadpoints:
  - title: Garage
    charger: wallbox
    mode: pv                   # default: solar surplus charging
    phases: 0                  # automatic 1p/3p if the cFos model supports switching; else set 3

site:
  title: Home
  meters:
    grid: grid
    pv: [pv]
    # battery: [battery]       # Phase 1b
  residualPower: 100           # keep small grid draw margin

tariffs:
  currency: EUR
  grid:
    type: template
    template: energy-charts-api   # EPEX day-ahead DE-LU, no registration
    bzn: DE-LU
    # Rabot ≈ spot + markup. API returns EUR/MWh; replicate the Rabot formula here, e.g.:
    # formula: (price / 1000 + 0.17) * 1.0   # TODO: exact values from Rabot contract
  feedin:
    type: fixed
    price: 0.08                # TODO: actual feed-in rate
```

Notes:

- **PHEV surplus threshold:** the BMW at 6 A single-phase needs only ~1.4 kW surplus to start;
  the ID.3 at 6 A three-phase needs ~4.1 kW. If the cFos supports automatic phase switching,
  `phases: 0` lets evcc pick; verify the specific Power Brain model during discovery.
- **Hichi as grid meter:** the utility meter's reading *includes* the wallbox and (later) the
  battery — that's exactly what a grid meter should measure, so no correction needed. If the
  hichi turns out to deliver only unsigned/consumption values (see discovery), PV surplus
  mode won't regulate correctly — run tariff-only (mode Off/Fast + cheap-hour plans) until
  1b rather than charging on bad data.
- **Battery vs car (Phase 1b):** evcc's battery settings (UI → battery) control priority,
  e.g. "fill home battery to X % before the car gets surplus" and lock the battery during
  fast charging so it doesn't discharge into the car. Priority choice is an **open
  decision** — start with evcc defaults, tune after a few weeks of real data.
- **Smart charging on price:** with the dynamic tariff configured, the UI offers a price limit
  in Min+PV/PV mode and cost-optimized charging plans ("cheapest hours until departure").
- **Vehicle auto-detection:** evcc polls both car APIs and assigns the plugged-in car
  automatically; with exactly two vehicles, misdetection is a one-tap manual fix in the UI.

## 4. Remote access — split approach

Two paths with different jobs; neither opens a port on the FritzBox.

### 4a. Liam (admin): FritzBox WireGuard

Full LAN access — evcc UI, cFos web interface, SolarEdge inverter, FritzBox admin, SSH to
the Pi. This is the maintenance path when something misbehaves while away.
Requires FritzOS ≥ 7.50 and a MyFRITZ account (free, provides DynDNS).

1. FritzBox UI → Internet → Permit Access → **VPN (WireGuard)** → Add connection →
   "Simplified setup" for a single device.
2. Scan the generated QR code with the WireGuard app (iOS/Android).
3. Toggle the tunnel (or use "on-demand"), then open `http://<pi-ip>:7070` — bookmark it.
4. Optional: mDNS name `http://raspberrypi.local:7070` works on LAN; over VPN use the IP.

### 4b. Partner (daily use): evcc Remote Access

Included in the sponsorship (requires a real token — enable after switching from the demo
token to the paid one). Exposes *only* the evcc dashboard via evcc's cloud; the instance
connects outbound, nothing listens on the home network.

1. evcc UI → Settings → Remote Access → enable, follow the pairing flow.
2. Put the resulting URL on the partner's phone home screen. No VPN app, no tunnel toggle.
3. Record the URL in `RUNBOOK.de.md` (placeholder there until this step).

Note: Remote Access is a newer evcc feature — check its current settings/docs state when
enabling, and fall back to a second WireGuard profile if it doesn't fit.

Trade-off, for the record: WireGuard keeps everything first-party; Remote Access routes the
dashboard through evcc's cloud (TLS, account-gated) — accepted for day-to-day convenience,
same trust level as the SolarEdge/VW/BMW cloud apps already in use.

## 5. Acceptance tests (Phase 1a)

- [ ] evcc UI shows live grid power (hichi) and PV power (inverter); grid goes negative when
      the sun exports.
- [ ] PV reading matches the SolarEdge monitoring app.
- [ ] Plug in a car on a sunny day in PV mode → charging starts/ramps with the sun.
- [ ] Fast mode button → charging at full power immediately.
- [ ] Charging plan "80 % by 07:00" → evcc schedules cheap/solar hours and hits the target.
- [ ] Both cars are auto-identified when plugged in.
- [ ] Liam's phone reaches the UI (and the Pi via SSH) from mobile data via WireGuard.
- [ ] Partner's phone reaches the dashboard from mobile data via the evcc Remote Access URL
      (after the paid token is active).
- [ ] Pi reboot → evcc comes back up on its own (`restart: unless-stopped`).

## 6. Phase 1b — after the installer visit (SE meter + battery connected)

Prerequisite: installer has connected and commissioned the SE-MTR meter and the battery
(SetApp), and the SolarEdge monitoring app shows grid import/export and battery SoC.

1. Swap the grid meter in `evcc.yaml`: comment out the `tasmota-sml` block, enable the
   `solaredge-hybrid` `usage: grid` block (§3). Add the battery meter and uncomment
   `battery: [battery]` under `site.meters`.
2. Restart evcc, then cross-check for a day: evcc grid reading ≈ hichi reading ≈ SolarEdge
   app. The hichi is retired to sanity-check duty after that (leave it running — free
   validation).
3. Battery coordination: verify evcc shows battery SoC/power; set battery priority in the
   UI. **Open decision** — options: battery-first (max evening self-sufficiency), car-first,
   or threshold ("battery to X %, then car"). Start with evcc defaults, revisit with a few
   weeks of data.
4. Re-run the §5 sun-ramp test plus: during Fast charging, battery must NOT discharge into
   the car (evcc battery lock).

Acceptance (1b):
- [ ] Grid/PV/battery/charge power in evcc match the SolarEdge app.
- [ ] Battery does not discharge into the car during fast charging.
- [ ] Surplus split between battery and car follows the chosen priority.
