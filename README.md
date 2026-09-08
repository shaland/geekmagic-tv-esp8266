# GeekMagic SmallTV — Custom Firmware (shaland fork)

[![License](https://img.shields.io/github/license/shaland/geekmagic-tv-esp8266)](/LICENSE)
[![Release](https://img.shields.io/github/v/release/shaland/geekmagic-tv-esp8266)](https://github.com/shaland/geekmagic-tv-esp8266/releases)

ESP8266 firmware for GeekMagic SmallTV devices, compatible with the GeekMagic HTTP API.

> [!NOTE]
> **Lineage:** [bvweerd/geekmagic-tv-esp8266](https://github.com/bvweerd/geekmagic-tv-esp8266)
> → [aydarik/geekmagic-tv-esp8266](https://github.com/aydarik/geekmagic-tv-esp8266)
> → this fork. Huge thanks to [@bvweerd](https://github.com/bvweerd) and
> [@aydarik](https://github.com/aydarik) for the original work ❤️
>
> This is a **personal fork** with a couple of additions (below). It is not intended to stay
> in sync with upstream and is not a place to file issues about the base firmware — use the
> upstream repos for that.

![Clock](/assets/photo_clock.jpg)

> [!WARNING]
> **SmallTV** / **SmallTV-Ultra** use an ESP8266. The **SmallTV-Pro** uses an ESP32 and is
> **not supported**. Flashing custom firmware is at your own risk.

## What this fork adds

| Change | Details |
|---|---|
| **Auto-rotation** | Timer-driven cycling through the clock and every JPEG in `/image` (name order), so the device works as a rotating dashboard with no external driver. Interval set via `/set?rotateSec=N` (seconds, `0` = off) or the Web UI. The physical button steps the same rotation. While rotation is on, `/set?img=` (with no `timeout`) updates the file without yanking the display — periodic image pushes don't interrupt the cycle. |
| **Configurable clock date format** | `/set?dateFmt=<urlencoded strftime>` or the Web UI. Default `%Y/%m/%d %a` (e.g. `2026/09/08 Mon`). Was previously hard-coded to `%d-%m-%Y`. |
| **Fixes** | `handleOTAForm()` served the OTA page with the wrong `Content-Length`; `settingsValidate()` wrote `brightness` where it meant `defaultTheme`. |

Both new settings appear in `app.json` (`rotateSec`, `dateFmt`) and in the Web UI's *Settings* card.
The settings struct changed, so **the device resets its configuration when upgrading** from an
older version.

### Turning the device into a Home-Assistant dashboard

This firmware has no Home Assistant client of its own. The pattern (also used by
[aydarik/hass-geekmagic](https://github.com/aydarik/hass-geekmagic)) is: HA renders an HTML
page to a 240×240 image with a rendering service and pushes it to `/doUpload?dir=/image/`.
Combine that with `rotateSec` above and each `/image/*.jpg` becomes a rotating page. The clock
page stays native.

## 🛠️ Installation

### First flash (UART required)

Factory devices ship with the stock firmware, so the **first** flash must be over serial.

1. Connect the device via USB/serial.
2. Flash `firmware.bin` from the [latest release](https://github.com/shaland/geekmagic-tv-esp8266/releases)
   with [web.esphome.io](https://web.esphome.io/) (or esptool).
3. Wiring / pinout: see upstream
   [FLASHING.md](https://github.com/bvweerd/geekmagic-tv-esp8266/blob/dev/FLASHING.md).

<details>
<summary>Looks messy, but works 🫢</summary>

![Flashing 1](/assets/photo_flash_1.jpg) ![Flashing 2](/assets/photo_flash_2.jpg)

</details>

### First boot

1. Device starts in AP mode — the display shows the SSID and password.
2. Connect and open `http://192.168.4.1`, enter your Wi-Fi credentials.
3. Device restarts; its IP is shown at boot and at the top of the clock screen.

### OTA updates (after the first flash)

Open the device IP → **Firmware Update (OTA)** at the bottom → upload `firmware.bin`.
On failure the device keeps the current firmware.

## Building

CI (`.github/workflows/release.yml`) builds `firmware.bin` on any `X.Y.Z` tag push and
attaches it to a **draft** release — publish with `gh release edit <tag> --draft=false --latest`.

Local build (PlatformIO):

```bash
mkdir -p src/generated
gzip -9 -c res/index.html > src/generated/index.html.gz && xxd -i src/generated/index.html.gz > src/generated/index_html.h
gzip -9 -c res/ota.html   > src/generated/ota.html.gz   && xxd -i src/generated/ota.html.gz   > src/generated/ota_html.h
pio run -e nodemcuv2
```

## 💬 Supported characters

![Charset](assets/charset.png)

## 📡 HTTP API

```bash
# Auto-rotation interval (seconds, 0 = off)   [fork]
curl "http://DEVICE_IP/set?rotateSec=8"

# Clock date format (urlencoded strftime)     [fork]
curl "http://DEVICE_IP/set?dateFmt=%25Y%2F%25m%2F%25d%20%25a"

# Brightness / theme / seconds
curl "http://DEVICE_IP/set?brt=50"
curl "http://DEVICE_IP/set?theme=1"
curl "http://DEVICE_IP/set?sec=true"

# Timezone (POSIX TZ)
curl "http://DEVICE_IP/set?tz=JST-9"

# Custom message / gauge / sticky note
curl "http://DEVICE_IP/set?msg=Hello%20world!&sbj=Notification&style=center&timeout=10"
curl 'http://DEVICE_IP/set?msg=21.4%2F40%20%E2%84%83&sbj=Living%20room&style=big_num&timeout=60'
curl "http://DEVICE_IP/set?note=+8%E2%84%83%20cloudy&rpm=6&timeout=3600"

# Countdown
curl "http://DEVICE_IP/set?cnt=2026-02-19T09%3A30&sbj=Next%20call&timeout=5"

# Images
curl -F "file=@photo.jpg" "http://DEVICE_IP/doUpload?dir=/image/"
curl "http://DEVICE_IP/set?img=/image/photo.jpg&timeout=30"
curl "http://DEVICE_IP/filelist?dir=/image/"

# Status
curl "http://DEVICE_IP/v.json"      # firmware version
curl "http://DEVICE_IP/app.json"    # device state (incl. rotateSec, dateFmt)
curl "http://DEVICE_IP/space.json"  # filesystem
```

![Custom Message](/assets/photo_message.jpg) ![Gauge](/assets/photo_gauge.jpg) ![Sticky Note](/assets/photo_note.jpg)

## 📜 License

MIT — see [LICENSE](/LICENSE). Same as upstream.
