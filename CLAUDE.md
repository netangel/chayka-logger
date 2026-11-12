# SY Chayka — Boat Data Logger

Raw NMEA 2000 data from the boat is captured on board, pulled to a VPS, stored in InfluxDB and visualised in Grafana.
Principle: **dumb boat, smart server.** The boat only captures raw data; all parsing, storage and rendering happen on the server.

## Architecture

```
GPS160/AIS ─0183─► YDNR-02 ─N2K─► instruments, C90W
                     ├─ srv1 TCP  N2K RAW ─► Pi Zero 2 WH (logger) ◄── rsync/SSH every 1 min ── VPS
                     ├─ srv3 UDP  NMEA0183 ─► gpsd (time) + phones/tablets        (Tailscale subnet route)
                     └─ srv2 free (future TCP clients)
VPS: pull job ─► canboat ─► InfluxDB 2 ─► Grafana (public)      [stage 2: BYOS ─► e-ink]
```

## Stage 1 — Logger + VPS + Grafana

### Boat hardware
- Raspberry Pi Zero 2 WH, official Zero case (GPIO lid, plastic for Wi-Fi), small heatsink
- SanDisk High Endurance 64 GB microSD (verify with H2testw/F3 first)
- Power: existing boat USB socket, short micro-USB cable into PWR IN, strain-relieved
- RTC: DS1307 Pi module (3.3 V I²C, no 5 V pull-ups, plain CR2032)
- Wi-Fi 2.4 GHz to boat router, power saving off; DHCP reservations for Pi and YDNR

### Pi OS
- Raspberry Pi OS Lite 64-bit; timezone **UTC**; SSH key-only; password auth off
- NetworkManager profiles: home + boat Wi-Fi
- Read-only root (overlay), **enabled last**; data on separate partition (`noatime`)
- journald volatile; fake-hwclock disabled

### Logger (Rust, systemd service, auto-restart)
- Static binary, target `aarch64-unknown-linux-musl`, cross-built off-boat (cargo-zigbuild); deployed as a single file
- Synchronous, no async runtime; minimal pinned crates: `socket2` (keepalive), `time` (UTC formatting), `signal-hook` (clean flush on SIGTERM)
- TCP client to YDNR server 1 (N2K RAW); read timeout + keepalive; reconnect with backoff
- Buffer partial lines across reads; write only complete lines; drop any fragment left at a disconnect
- Prefix each line with full UTC timestamp (RAW has time-of-day only)
- Monotonic clock for all timers (fsync, status, backoff); wall-clock UTC only for line prefixes and file rotation
- Hourly UTC files, **append mode**; rotation decided by the timestamp being written; files are never rewritten, renamed or compressed on the Pi
- Flush regularly, fsync ~every 30 s
- Status line every minute: connection state, frame counts, clock-sync state; boot ID and connect/disconnect markers
- Cleanup: daily timer, keep 60 days, free-space guard ~85 %, never touch the current file
- Deploy: versioned binaries in `/opt/chayka-logger/`, systemd unit points to a `current` symlink; keep the previous version for rollback

### Time
- RTC at boot → gpsd (YDNR server 3, UDP) → chrony (+ router NTP)
- chrony may step at any time; drift file on data partition; keeps RTC updated (rtcsync)

### Network
- Boat router's Tailscale advertises boat LAN; Pi has no Tailscale
- Pull model: VPS initiates all transfers
- Aboard: unlimited Starlink. Winter: Starlink Standby (500 kbps), 2–4 h windows
- SSH key on Pi restricted to read-only rsync of the data directory

### VPS (1 vCPU / 4 GB / 50 GB)
- Tailscale with subnet routes accepted
- Pull: systemd timer every 1 min (no overlap), rsync append mode + compression, persistent SSH connection, short connect timeout
- Ingestion: per-file byte offsets, complete lines only → canboat → InfluxDB 2 with original timestamps, downsampled
- Realign segments recorded before clock sync using GNSS time PGNs in the stream
- Grafana: public read-only dashboards via Caddy (HTTPS); admin login protected, sign-ups off; InfluxDB never public
- Monitoring: "last data received" → Uptime Kuma push monitor (Kuma runs on a separate host)
- Storage: raw archive ~10–60 GB/season; keep recent raw on VPS, move older to object storage/home; second copy of raw data from day one

## Stage 2 — E-ink (BYOS)
- BYOS renderer on VPS, reading InfluxDB; only the image endpoint public (Caddy, HTTPS, device key)
- TRMNL-style ESP32 device, fetches from anywhere (boat or home in winter)
- Content: battery SOC/current, tanks, temperatures, 24–48 h barograph, prominent "data as of …" timestamp

