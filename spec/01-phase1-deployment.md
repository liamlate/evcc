# Phase 1 — Deploy & Configure evcc

Milestone: **PV surplus charging works** — a plugged-in car charges automatically from excess
solar, visible in the evcc UI from both phones via VPN.

## 1. Discovery checklist (do this first, ~1 evening)

Several unknowns block configuration. Verify each and note the result here.

- [ ] **SolarEdge meter commissioned?** Check the SolarEdge monitoring app/portal: does it show
      grid import/export and self-consumption? If yes, the SE-MTR meter is active. If not, the
      installer must commission it (SetApp).
- [ ] **Battery visible?** Same portal: does the storage show charge/discharge and SoC?
- [ ] **Modbus TCP enabled on the inverter?** evcc talks to SolarEdge via Modbus TCP,
      port 1502. Enabling it is done in SetApp (installer access) or via the inverter's local
      web interface on newer firmware. This is the single most likely blocker — schedule the
      installer if needed.
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

## 3. evcc.yaml skeleton

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
  - name: grid
    type: template
    template: solaredge-hybrid
    usage: grid
    host: <inverter-ip>       # Modbus TCP, port 1502
    port: 1502
  - name: pv
    type: template
    template: solaredge-hybrid
    usage: pv
    host: <inverter-ip>
    port: 1502
  - name: battery
    type: template
    template: solaredge-hybrid
    usage: battery
    host: <inverter-ip>
    port: 1502

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
    battery: [battery]
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
- **Battery vs car:** evcc's battery settings (UI → battery) control priority, e.g. "fill home
  battery to X % before the car gets surplus" and lock the battery during fast charging so it
  doesn't discharge into the car.
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

## 5. Acceptance tests

- [ ] evcc UI shows live grid / PV / battery / charge power that matches the SolarEdge app.
- [ ] Hichi reading ≈ evcc grid reading (sanity check, ±ripple).
- [ ] Plug in a car on a sunny day in PV mode → charging starts/ramps with the sun.
- [ ] Fast mode button → charging at full power immediately.
- [ ] Charging plan "80 % by 07:00" → evcc schedules cheap/solar hours and hits the target.
- [ ] Both cars are auto-identified when plugged in.
- [ ] Liam's phone reaches the UI (and the Pi via SSH) from mobile data via WireGuard.
- [ ] Partner's phone reaches the dashboard from mobile data via the evcc Remote Access URL
      (after the paid token is active).
- [ ] Pi reboot → evcc comes back up on its own (`restart: unless-stopped`).
