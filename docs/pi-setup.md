# Pi Zero 2 WH — deployment and configuration

Checklist for taking the Pi from the box to a running logger on the boat.
Architecture and rationale live in `CLAUDE.md`; this file is the step list.

The logger binary is not needed until Phase 5. Phases 1 to 4 are plain OS setup.

## Phase 1: before the Pi arrives (~1 h)

1. Verify the SD card (SanDisk High Endurance 64 GB). Takes 40 to 90 min, mostly waiting.
   - macOS / Linux: `brew install f3`, then `f3write /Volumes/<card> && f3read /Volumes/<card>`
   - Windows: H2testw 1.4 (single .exe, from heise.de). Format the card FAT32 or exFAT, "Select target", leave "all available space", click "Write + Verify".
   - Pass: no errors reported. Fail: any data-lost or mismatch message. The card is fake or bad, so return it.
2. On the VPS, create the pull key: `ssh-keygen -t ed25519 -f ~/.ssh/chayka_pull`. The public half goes on the Pi in Phase 3.
3. Write down the boat Wi-Fi SSID and password.
4. On the build machine: `rustup target add aarch64-unknown-linux-musl`, install `zig`, then `cargo install cargo-zigbuild`.

## Phase 2: OS on the bench at home (~1.5 h)

1. Flash Raspberry Pi OS Lite 64-bit with Raspberry Pi Imager. In its settings: hostname, user, SSH key only, home Wi-Fi, timezone UTC.
2. Boot, `ssh` in, then `sudo apt update && sudo apt full-upgrade`.
3. Add the boat Wi-Fi as a second NetworkManager profile. Turn Wi-Fi power saving off.
4. Create the separate data partition, mounted with `noatime`.
5. Set journald to volatile (`Storage=volatile`). Disable fake-hwclock.

**Open problem:** Pi OS expands the root partition to fill the card on first boot, so a separate data partition needs a trick to avoid that. Exact commands are still to be written.

## Phase 3: RTC, time, SSH lockdown (~1 h)

1. Fit the DS1307 module. Enable I²C and add `dtoverlay=i2c-rtc,ds1307` to `/boot/firmware/config.txt`.
2. Check `sudo hwclock -r`. Set it once from NTP with `sudo hwclock -w`.
3. Install `chrony` and `gpsd`. Set the chrony drift file on the data partition and enable `rtcsync`.
4. Put the VPS public key in `authorized_keys`, restricted to read-only rsync of the data directory.
5. Confirm password auth is off: `sudo sshd -T | grep passwordauthentication`.

## Phase 4: on the boat (~1 h)

1. Add DHCP reservations on the router for the Pi and the YDNR.
2. Power the Pi from the boat USB socket. Confirm it is reachable over Tailscale.
3. Check the YDNR server 1 stream from the Pi: `nc <ydnr-ip> <srv1-port>` should print RAW lines.
4. Point gpsd at the YDNR server 3 UDP stream. Check `chronyc sources`.

## Phase 5: logger deploy and lock-down

1. Cross-build off-boat: `cargo zigbuild --release --target aarch64-unknown-linux-musl`.
2. Copy the binary to `/opt/chayka-logger/<version>/`. Point the `current` symlink at it. Keep the previous version for rollback.
3. Install the systemd unit (it runs `current`, with auto-restart). Enable and start it.
4. Run for a day. Confirm hourly files appear, status lines are written, and the VPS pull works.
5. Enable the read-only root overlay. **Do this last.**

## Done when

- Hourly UTC files grow on the data partition.
- The VPS pulls them every minute.
- `chronyc tracking` shows a synced clock.
- The Pi survives a power cut and comes back logging.
