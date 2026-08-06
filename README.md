# ESP32-C6 Zigbee Coordinator on a Raspberry Pi Zero 2 W

A Zigbee2MQTT coordinator built from a generic ESP32-C6 dev board wired to a Pi Zero 2 W
over the GPIO header UART — no USB dongle, no USB cable between the two boards.

Most Zigbee2MQTT setups use a commercial dongle (Sonoff ZBDongle-P/E, SLZB-06). This one
uses the ESP32-C6's built-in 802.15.4 radio instead, running
[tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator) as a ZBOSS NCP.
It's a niche path and the UART-to-host variant is thinly documented, which is most of the
reason this repo exists.

**Status:** working. Coordinator on channel 25, Zigbee2MQTT + Mosquitto in Docker,
management UI reachable over Tailscale only.

---

## Why you might not want to do this

Buy a $20 Sonoff ZBDongle-P if you want a Zigbee network rather than a project. It ships
pre-flashed, plugs into USB, and works immediately. The C6 route is worth it if you
already own the board, want the +20 dBm transmit power and 200-node capacity without
paying for a CC2652P, or simply want to build the thing.

Budget an evening. Most of it will be the two failure modes in
[Troubleshooting](#troubleshooting), neither of which produces an error message that
points at the actual cause.

---

## Hardware

| Part | Notes |
|---|---|
| Raspberry Pi Zero 2 W | 512 MB RAM is enough for Mosquitto + Zigbee2MQTT |
| ESP32-C6 dev board | Any generic one. This build used a SuperMini-style board |
| 4 × jumper wires | Power, ground, and the two UART lines |
| 5 V / 2 A USB supply | Powers the Pi, and the C6 through it |
| microSD card | 8 GB+ |

The Pi Zero 2 W ships without a soldered GPIO header. You need pins (or wires) on at
least physical pins 2/4, 6, 8 and 10.

### Power budget

The C6 runs off the Pi's 5 V rail, so one supply feeds both. Raspberry Pi publishes no
current limit for the 5 V header pins — they're fed straight from the Pi's own power
input, so the real constraint is your PSU.

- Pi Zero 2 W: ~150–350 mA depending on Wi-Fi activity
- ESP32-C6: ~80–500 mA, peaking during radio transmission

Worst case is roughly 850 mA. A 2 A supply has comfortable headroom.

Note this is the **5 V** rail. The 3.3 V header pins are a different story — that rail is
capped around 50 mA total across all GPIO, which is nowhere near enough. Don't power the
C6 from 3.3 V.

---

### References
<img width="4032" height="3024" alt="IMG_2009" src="https://github.com/user-attachments/assets/52d6dd1a-a3ea-48f6-ab67-63d27a9283b3" />
<img width="499" height="513" alt="Screenshot 2026-08-06 at 16 46 17" src="https://github.com/user-attachments/assets/f992d5e3-f7ae-45bb-95e3-3161108d121e" />
<img width="635" height="1142" alt="Screenshot 2026-08-06 at 19 45 24" src="https://github.com/user-attachments/assets/4b8aac27-7ef5-4afb-ba1b-599821c982f4" />


---

## Wiring

**Use the pins labelled GPIO22 and GPIO23, not the ones silkscreened TX and RX.**

On many C6 boards these are different pins. The silkscreened TX/RX are the board
manufacturer's convention; the firmware only listens on GPIO22/23. Wiring to the wrong
pair produces total silence with no error to explain it.

| Pi (physical pin) | Pi function | ESP32-C6 |
|---|---|---|
| 2 or 4 | 5 V | 5 V / VIN |
| 6 | GND | GND |
| 8 | GPIO14 — TX | **GPIO23** (coordinator RX) |
| 10 | GPIO15 — RX | **GPIO22** (coordinator TX) |

TX goes to RX. Getting this backwards is the single most common cause of a dead link, and
it looks identical to every other failure.

Power the Pi down while wiring.

---

## Build

### 1. Flash the SD card

Raspberry Pi OS Lite (64-bit). Any flasher works — Raspberry Pi Imager bakes in the
headless config for you; with Balena Etcher you write the config files yourself
afterwards, which is what the [`cloud-init/`](cloud-init/) directory here is for.

Recent Pi OS images use cloud-init. After flashing, the `bootfs` partition needs:

- `ssh` — an empty file. Its presence enables the SSH server on first boot.
- `user-data` — user account, SSH key, hostname. See [`cloud-init/user-data`](cloud-init/user-data).
- `network-config` — Wi-Fi credentials. See [`cloud-init/network-config`](cloud-init/network-config).

On macOS the partition may need mounting by hand:

```bash
diskutil list                    # find the disk, e.g. disk5
diskutil mount /dev/disk5s1      # bootfs is the FAT32 partition
```

**`lock_passwd: true` will lock you out of `sudo`.** With no password set, `sudo` prompts
for one that cannot exist, and there's no way to fix it without re-flashing. Either set
`lock_passwd: false` with a hashed `passwd:` (what this repo does), or add
`sudo: ALL=(ALL) NOPASSWD:ALL` to the user block.

Generate the hash:

```bash
openssl passwd -6 -stdin
```

This is separate from `ssh_pwauth: false`, which stays false — SSH remains key-only. The
password exists purely for `sudo`.

Eject cleanly (`cd` out of the volume first, or the unmount is refused), boot the Pi, wait
~90 s, then:

```bash
ssh <user>@<hostname>.local
```

### 2. Free up the UART

Two things hold `/dev/ttyAMA0` on a stock image, and both must go.

In `/boot/firmware/config.txt`, under the `[all]` section:

```
enable_uart=1
dtoverlay=disable-bt
```

`disable-bt` matters because Bluetooth otherwise claims the good UART and leaves the GPIO
pins on the slower mini-UART.

In `/boot/firmware/cmdline.txt`, **remove `console=serial0,115200`**. The file must stay a
single line. Then:

```bash
sudo systemctl disable --now serial-getty@ttyAMA0.service
sudo reboot
```

Skipping this step is the failure documented in [Troubleshooting](#zigbee2mqtt-connects-then-times-out-after-30-seconds).

Confirm afterwards:

```bash
ls -l /dev/ttyAMA0        # should exist, root:dialout
```

Your user needs to be in `dialout` — the `user-data` in this repo puts it there.

### 3. Flash the coordinator firmware

Do this over USB from a laptop, before wiring the C6 to the Pi.

A [web flasher](https://install.busware.de/zboss/) exists and is the quickest route
(Chrome/Edge/Opera only — WebSerial). To do it from the CLI instead:

```bash
pip3 install esptool
curl -L -o factory.bin \
  https://raw.githubusercontent.com/tostmann/esp-coordinator/master/binaries/factory.bin
ls /dev/tty.*                    # macOS; find the new usbmodem entry
esptool.py -p /dev/tty.usbmodemXXXX --chip esp32c6 write_flash 0x0 factory.bin
```

If it fails with `Device not configured` or `Failed to get VID/PID`, put the board into
bootloader mode by hand: hold **BOOT**, tap **RESET**, release BOOT, then run the command
again.

Verify before moving on — with a serial monitor open at 115200, reset the board. It emits
a 7-byte boot frame, `DE AD 05 00 06 01 8F`. Binary junk on screen is the correct result.

### 4. Wire it up

Per the table above, Pi powered off.

### 5. Verify the link before touching Docker

This step is worth the two minutes. It isolates wiring problems from software ones.

```bash
sudo timeout 10 cat /dev/ttyAMA0 | xxd
```

Press RESET on the C6 during the capture. You want:

```
00000000: dead 0500 0601 8f...
```

- **`dead 0500 0601 8f`** — correct. The coordinator is running and reaching the Pi.
- **`ESP-ROM:esp32c6-...` then nothing** — the ROM bootloader is getting through but the
  application isn't. Usually means you're on the silkscreened TX/RX pins rather than
  GPIO22/23.
- **Nothing at all** — TX/RX crossed, GND missing, or a loose wire.
- **A stray byte or two** — bad contact. Reseat.

This only proves C6 → Pi. The other direction is proven when Zigbee2MQTT starts cleanly.

### 6. Docker

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $USER
sudo systemctl enable docker
```

Log out and back in, then `docker ps` should work without `sudo`.

### 7. Zigbee2MQTT and Mosquitto

Copy [`docker-compose.yml`](docker-compose.yml),
[`mosquitto/config/mosquitto.conf`](mosquitto/config/mosquitto.conf) and
[`z2m-data/configuration.yaml`](z2m-data/configuration.yaml) into place, then:

```bash
docker compose up -d
docker compose logs -f zigbee2mqtt
```

Two things in the Zigbee2MQTT config that the default gets wrong for this setup:

- `server: mqtt://mosquitto:1883` — not `localhost`. Inside the container, `localhost` is
  the container itself. `mosquitto` is the Compose service name.
- `frontend.enabled: true` — the generated default may be `false`, which leaves you stuck
  on the onboarding page.

Success looks like:

```
z2m: zigbee-herdsman started (reset)
z2m: Coordinator firmware version: '...busware.de ESP32 (ZBOSS)...'
z2m: Connected to MQTT server
z2m: Zigbee2MQTT started!
```

`docker compose` must be run from the directory holding `docker-compose.yml`, or you get
`no configuration file provided: not found`.

---

## Channel selection

Zigbee channels 11–26 overlap 2.4 GHz Wi-Fi channels 1–11. **25 or 26** sit clear of the
common Wi-Fi channels (1, 6, 11) and are a safe default regardless of your router's
settings — which matters here, since the Pi's Wi-Fi antenna and the C6's Zigbee antenna
are inches apart.

Pick this before pairing anything. Changing channel later requires re-pairing every
device, and see the backup-mismatch note in
[Troubleshooting](#channel-change-refuses-to-apply).

---

## Remote access

SSH is key-only (`ssh_pwauth: false`, key in `user-data`), reachable over both LAN and
Tailscale.

The Zigbee2MQTT frontend is a different matter — it has **no authentication by default**
and lets anyone reload it pair devices, remove devices, and see your whole network. Left
on `0.0.0.0:8080` it's open to everyone on your Wi-Fi.

This build binds it to the Tailscale interface only:

```yaml
ports:
  - "100.x.x.x:8080:8080"     # the Pi's own Tailscale IP
```

Confirm with `docker ps` — the PORTS column should show the Tailscale address, not
`0.0.0.0`. A LAN client then gets connection refused.

Alternatives, if you'd rather not: set `frontend.auth_token` in `configuration.yaml`, or
accept the exposure on a trusted network.

SSH is deliberately left reachable on the LAN as a fallback — if Tailscale is down you can
still get into the box. Binding sshd to the Tailscale interface only is tighter but
removes that escape hatch.

---

## Troubleshooting

### Zigbee2MQTT connects, then times out after 30 seconds

```
zh:zboss:uart: Serial port opened
zh:zboss:driv: Driver connected
zh:zboss:driv: Driver startup
zh:zboss:uart: --> Error: Error: 0 after 30000ms
zh:zboss:uart: -!- break waiting (0)
zh:zboss:uart: Can't send DATA frame (0,0,0): dead0c0006c07c7df70000140001
```

The port opens, the driver connects, then nothing. This has three plausible causes and
they're indistinguishable from the log alone:

1. **`console=serial0,115200` still in `/boot/firmware/cmdline.txt`.** The kernel is using
   the UART as a console, so a second process owns the port. Zigbee2MQTT can open it and
   write to it, but the frames go nowhere. Remove it, disable
   `serial-getty@ttyAMA0`, reboot. This one is invisible from the Zigbee2MQTT side and
   raw `cat /dev/ttyAMA0` still shows inbound data, which makes it deeply misleading.
2. **TX/RX not crossed.** Pi TX must reach coordinator RX.
3. **Wired to the silkscreened TX/RX rather than GPIO22/23.**

Work through them with the raw UART test in step 5.

### Channel change refuses to apply

```
Error: ZBOSS: coordinator_backup.json does not match configuration.yaml
(backup pan=0xb040 ch=11 ... vs config pan=0xb040 ch=[25] ...).
Refusing to restore a non-matching backup
```

Zigbee2MQTT won't reconcile a stored backup against a changed channel, because doing so
can destroy a live network. With nothing paired, move the backup aside and let it
commission fresh:

```bash
docker compose stop zigbee2mqtt
mv z2m-data/coordinator_backup.json z2m-data/coordinator_backup.json.old
docker compose up -d
```

You should then see `Leaving current network and forming new network`. Delete the `.old`
file afterwards so it can't confuse a later rebuild.

**With devices paired, this is destructive** — they'll all need re-pairing.

### Host key changed after re-flashing

Expected. The Pi generates new host keys on a fresh image:

```bash
ssh-keygen -R <hostname>.local
```

### `sudo` asks for a password that doesn't exist

`lock_passwd: true` with no `passwd:`. Requires re-flashing. See step 1.

### Files in `z2m-data` are root-owned

Normal — the container writes as root through a bind mount. You'll need `sudo` to edit or
delete them from the host. Only matters for backups (`sudo` when archiving) and restores
(`sudo chown -R root:root z2m-data` if the container can't write).

---

## Moving house

Only three things are tied to location:

1. Wi-Fi SSID and password
2. Static LAN IP and gateway, if you set one
3. Zigbee channel, worth re-checking against the new Wi-Fi environment — though 25/26 is
   usually still fine

Everything else travels: Tailscale identity, SSH keys, the Docker stack, Zigbee2MQTT
config, paired devices, and the wiring. Paired devices are bound to the coordinator's PAN
ID, channel and network key — not to your Wi-Fi — so they survive the move as long as the
coordinator's NVRAM is intact.

Keep `coordinator_backup.json`. With the `ghcr.io/tostmann/zigbee2mqtt-esp32` image it
carries a raw NVRAM blob, so a dead C6 can be swapped for a new one and restored without
re-pairing anything.

---

## Credits

- [tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator) — the ZBOSS NCP
  firmware, and the actively maintained fork of the original
  [andryblack/esp-coordinator](https://github.com/andryblack/esp-coordinator)
- [Zigbee2MQTT](https://www.zigbee2mqtt.io/)
- [Eclipse Mosquitto](https://mosquitto.org/)
