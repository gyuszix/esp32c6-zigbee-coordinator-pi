# ESP32-C6 Zigbee Coordinator on a Raspberry Pi Zero 2 W

**v0.4** — Zigbee2MQTT coordinator using an ESP32-C6 DevKit wired to a Pi Zero 2 W over
the GPIO UART. No USB dongle. Runs [tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator)
as a ZBOSS NCP.

**Status:** working. Channel 25, Zigbee2MQTT + Mosquitto in Docker, Tailscale-only access.

> Want a Zigbee network rather than a project? Buy a Sonoff ZBDongle-P.

## Version history

- **v0.4** — Host-specific values moved to gitignored `.env`, `secret.yaml`, `devices.yaml`.
  Repo clones straight onto the Pi. `log_directories_to_keep: 50`.
- **v0.3** — SuperMini → DevKit (SuperMini doesn't break out GPIO22/23 properly).
- **v0.2** — Working build on a SuperMini.
- **v0.1** — Initial build.

## Hardware

| Part | Notes |
| --- | --- |
| Raspberry Pi Zero 2 W | With GPIO header |
| ESP32-C6 DevKit | Any board that breaks out GPIO22/23 |
| 4 jumper wires | 5V, GND, TX, RX |
| 5V / 2A supply | Powers both. Worst case ~850 mA |
| microSD | 8 GB+ |

Power the C6 from **5V**, not 3.3V.

<img width="864" height="627" alt="Screenshot 2026-08-13 at 23 06 07" src="https://github.com/user-attachments/assets/9af95c97-60c6-4c43-83a9-75e58a482532" />
<img width="4032" height="3024" alt="IMG_2009" src="https://github.com/user-attachments/assets/52d6dd1a-a3ea-48f6-ab67-63d27a9283b3" />
<img width="499" height="513" alt="Screenshot 2026-08-06 at 16 46 17" src="https://github.com/user-attachments/assets/f992d5e3-f7ae-45bb-95e3-3161108d121e" />
<img width="635" height="1142" alt="Screenshot 2026-08-06 at 19 45 24" src="https://github.com/user-attachments/assets/4b8aac27-7ef5-4afb-ba1b-599821c982f4" />

## Wiring

| Pi pin | Pi function | ESP32-C6 |
| --- | --- | --- |
| 2 or 4 | 5V | 5V |
| 6 | GND | GND |
| 8 | GPIO14 TX | **GPIO23** (RX) |
| 10 | GPIO15 RX | **GPIO22** (TX) |

Use the pins labelled **GPIO22/23**, not the silkscreened TX/RX. TX goes to RX.

## Build

### 1. SD card

Raspberry Pi OS Lite 64-bit. Put `ssh` (empty file), [`user-data`](cloud-init/user-data)
and [`network-config`](cloud-init/network-config) on `bootfs`.

Use `lock_passwd: false` with a hashed `passwd:` (`openssl passwd -6 -stdin`), or you
can't `sudo`. SSH stays key-only.

### 2. Free the UART

`/boot/firmware/config.txt`, under `[all]`:

```
enable_uart=1
dtoverlay=disable-bt
```

Remove `console=serial0,115200` from `/boot/firmware/cmdline.txt` (keep it one line), then:

```bash
sudo systemctl disable --now serial-getty@ttyAMA0.service
sudo reboot
```

### 3. Flash the C6

Over USB, before wiring. [Web flasher](https://install.busware.de/zboss/), or:

```bash
curl -L -o factory.bin https://raw.githubusercontent.com/tostmann/esp-coordinator/master/binaries/factory.bin
esptool.py -p /dev/tty.usbmodemXXXX --chip esp32c6 write_flash 0x0 factory.bin
```

If it won't connect: hold BOOT, tap RESET, release BOOT, retry.

### 4. Wire and test the link

Wire with the Pi off. Then press RST on the C6 during:

```bash
sudo timeout 10 cat /dev/ttyAMA0 | xxd
```

| Output | Meaning |
| --- | --- |
| `dead 0500 0601 8f` | Working |
| `ESP-ROM:...` then nothing | On silkscreened TX/RX, not GPIO22/23 |
| Nothing | TX/RX not crossed, no GND, or loose wire |

### 5. Docker

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $USER    # log out and back in
```

### 6. Deploy

```bash
git clone https://github.com/gyuszix/esp32c6-zigbee-coordinator-pi.git ~/zigbee
cd ~/zigbee
echo "TS_IP=$(tailscale ip -4)" > .env
python3 -c "import secrets as s; print(f'network_key: {list(s.token_bytes(16))}\npan_id: {s.randbelow(0xFFFE)+1}\next_pan_id: {list(s.token_bytes(8))}')" > z2m-data/secret.yaml
echo '{}' > z2m-data/devices.yaml
docker compose up -d && docker compose logs -f zigbee2mqtt
```

Look for `Zigbee2MQTT started!`. Update later with `git pull && docker compose up -d`.

| Gitignored file | Holds |
| --- | --- |
| `.env` | `TS_IP`. Compose refuses to start without it |
| `z2m-data/secret.yaml` | Network key, PAN IDs. **Back this up.** Lose it and you re-pair everything |
| `z2m-data/devices.yaml` | Paired devices |

## Config notes

- `server: mqtt://mosquitto:1883` — not `localhost` (that's the container itself).
- `adapter: zboss` — mandatory.
- `channel: 25` — clear of Wi-Fi 1/6/11. Changing it later means re-pairing everything.
- `frontend` and broker are bound to `TS_IP` only. The frontend has no auth.

## Troubleshooting

### Times out 30s after `Driver connected` (`break waiting`)

| Cause | Fix |
| --- | --- |
| `console=serial0` still in `cmdline.txt` | Remove it, disable `serial-getty`, reboot |
| TX/RX not crossed, or on silkscreened pins | Rewire, re-run the link test |
| **C6 wedged** (was working, Pi checks clean) | Press RST on the C6, then `docker compose restart zigbee2mqtt`. A Pi reboot won't help: the 5V rail stays up |

### `coordinator_backup.json does not match configuration.yaml`

Happens after a channel change. **With nothing paired**, move the backup aside:

```bash
docker compose stop zigbee2mqtt
mv z2m-data/coordinator_backup.json z2m-data/coordinator_backup.json.old
docker compose up -d
```

With devices paired, this means re-pairing all of them.

### Misc

- Host key changed after re-flash: `ssh-keygen -R <hostname>.local`
- `z2m-data` files root-owned: normal, the container writes as root. Use `sudo`.

## Moving house

Only the Wi-Fi credentials, static IP and (maybe) the channel change. Keep
`coordinator_backup.json` and `secret.yaml` backed up privately, and a dead C6 can be
swapped without re-pairing.

## Credits

[tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator) ·
[Zigbee2MQTT](https://www.zigbee2mqtt.io/) · [Eclipse Mosquitto](https://mosquitto.org/)
