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

### Windows10 for handling partitions:

We can create the data partition from Windows between the flash and the first boot, which means we don’t have to shrink anything.

1. Flash Pi OS Lite with Imager, with your user, SSH and Wi-Fi settings. Don’t boot the Pi yet.
2. In an admin PowerShell run diskpart, then:
```
list disk
select disk 1
create partition primary offset=8388608
exit
```
Check list disk first and make sure disk 1 is the ~60 GB card. The offset is in KB, so this starts the new partition at 8 GiB and leaves a gap after root.
3. If Windows offers to format the new partition, click Cancel. If diskpart refuses on removable media, use the loopback option instead.
4. Boot the Pi, SSH in, and run lsblk. You should see a small root (p2) and p3 starting after the gap. If anything looks wrong, reflash. Nothing is lost.
5. If root is still around 3 GB, grow it into the gap, which can be done while running:
```
sudo parted /dev/mmcblk0 resizepart 2 8000MiB
sudo resize2fs /dev/mmcblk0p2
```
6. Format p3 with `sudo mkfs.ext4 -L data /dev/mmcblk0p3`, and make the mount point with `sudo mkdir /data`.
7. Add this line to /etc/fstab:
```
LABEL=data  /data  ext4  defaults,noatime,nofail  0  2
```
8. Test it with `sudo mount -a`, then run `findmnt /data` (the options column should show noatime). Finish with `sudo chown youruser: /data`.

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
