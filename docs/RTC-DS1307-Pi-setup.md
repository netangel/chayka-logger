Next action: run `sudo nano /boot/firmware/config.txt` and add one line at the end. 
It takes about 10 minutes in total.

- Turn on I2C (if it isn’t already). Run `sudo raspi-config` and enable I2C under 
  the interface options. The tutorials say to enable it in raspi-config before 
  anything else. 
- Add the overlay. On modern Pi OS, add `dtoverlay=i2c-rtc,ds1307` to the end of 
  `/boot/firmware/config.txt`. On an older OS the file is `/boot/config.txt`. 
- Reboot. The overlay loads the RTC at startup, so the old manual 
  `echo ds1307 0x68 …` trick and the rc.local hack aren’t needed. Check with 
  `i2cdetect -y 1` (from i2c-tools). The address 0x68 should show UU, meaning 
  the kernel driver has claimed it. 
- Set the RTC from the system time. First make sure the system time is correct 
  (via network or by setting it by hand). Then run `sudo hwclock -w` to write it, 
  and sudo `hwclock -r` to read it back. Those are the write and read commands. 
  If you get “command not found”, install `util-linux-extra`, because Debian moved 
  `hwclock` into that package. 

Test it offline. Power off, wait a minute, then boot with no network and run date. 
If the time is right, you’re done.

Two optional items, only if the test in step 5 fails:

`fake-hwclock`. Current Raspberry Pi OS images usually don’t include it, 
and apt will tell you if it isn’t installed. If it is installed, remove it 
with `sudo apt remove fake-hwclock`, so it doesn’t override the RTC. 

`udev` rule. Some guides, including AB Electronics’ (updated 2025), 
also comment out three lines in `/lib/udev/hwclock-set` and add `rtc-ds1307` 
to `/etc/modules`. The Adafruit and raspberry.tips pages don’t mention this, 
so I’d skip it unless the time is wrong after a reboot even though `hwclock -r works`. 

