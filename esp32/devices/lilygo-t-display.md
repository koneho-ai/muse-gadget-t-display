<!--
Copyright (c) Meta Platforms, Inc. and affiliates.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# LILYGO T-Display

A Muse gadget on the original LILYGO (TTGO) T-Display — the classic ESP32
board, **not** the T-Display-S3, which is a different chip (ESP32-S3) and is
not covered here.

| | |
|---|---|
| Chip | ESP32 (Xtensa dual-core LX6), chip revision 1.0 or later (both profiles declare v1.0–v3.99) |
| Display | 1.14 inch ST7789V IPS LCD, 135×240, SPI |
| Flash | 4 MB or 16 MB, depending on the variant — two profiles below |
| PSRAM | none |
| USB | CH9102 USB-UART bridge (some older units use a CP2104 or CH340); shows up as a `usbserial` / `wchusbserial` / `ttyUSB` port |
| Buttons | GPIO0 (right, also BOOT) and GPIO35 (left, not used by this profile) |

Display pins, from LILYGO's own
[TTGO-T-Display repo](https://github.com/Xinyuan-LilyGO/TTGO-T-Display)
(README pin table and `TFT_eSPI/User_Setups/Setup25_TTGO_T_Display.h`):
MOSI 19, SCLK 18, CS 5, DC 16, RST 23, backlight 4 (active high). The
135×240 panel sits inside the ST7789's 240×320 CGRAM at column offset 52,
row offset 40 (TFT_eSPI's portrait offsets for a 135-wide panel).

## What it does

A status-screen gadget, like the ideaspark:

- The screen shows Muse's status: colour bars along the top and bottom
  edges, the pixel animation and Muse's name in between, and a green dot in
  the top-right corner once it is connected.
- Muse can draw images on the 135×240 screen (`display.draw_url`) and
  return to the animation (`display.show_animation`).
- It pairs with the Muse app over BLE as `MuseGadget-Disp-XXXXXX`, joins
  2.4 GHz Wi-Fi, and holds the encrypted session to Muse, which can also
  read `device.health`.

What it does not do, because the hardware isn't there: no microphone or
speaker (so no push-to-talk), no touch, no camera or sensors, and no PSRAM,
so no home-network tunnel and no full avatar UI. The left-hand button
(GPIO35) and the battery-voltage sense on GPIO34 are wired on the board but
not used by this profile; `device.health` reports its battery fields as
null.

A note on chip revisions: T-Displays ship with both revision 1.0 and
revision 3.0 ESP32 silicon (`esptool` prints it as it connects, e.g.
`ESP32-D0WDQ6 (revision v1.0)`). Revision 1.0 predates the Secure Boot V2
RSA app-signing scheme the other classic-ESP32 profiles use, so these
profiles sign with the V1 ECDSA scheme and the shared development key in
`esp32/dev_signing_key_ecdsa.pem` instead, and declare a minimum chip
revision of 1.0. If a flash fails with `requires chip revision in range
v3.0 - v3.99`, you have an older build of these profiles — pull the
current ones.

## Which profile: 4 MB or 16 MB?

LILYGO sells the board with 4 MB or 16 MB of flash. If you know which you
have, build that profile. If not, ask the chip — this reads the board
without writing anything to it:

```sh
python -m esptool -p PORT flash-id   # "Detected flash size: 4MB" (or 16MB)
```

Flashing the 16 MB build onto a 4 MB board writes past the end of the
flash, so check first.

## Build and flash

You need ESP-IDF v6.0.1 and an SDK token from
[gadgets.muse.ai](https://gadgets.muse.ai/settings/sdk-tokens)
(Account > SDK tokens); see the top-level `AGENTS.md`. Put the token in the
build directory's `sdkconfig` (`CONFIG_GADGET_SDK_TOKEN="mgst_…"`), never
in git.

From the `esp32/` directory:

```sh
# 16 MB variant
tools/board.sh lilygo-t-display build
tools/board.sh lilygo-t-display flash-monitor /dev/ttyUSB0

# 4 MB variant (single app slot, no OTA)
tools/board.sh lilygo-t-display-4mb build
tools/board.sh lilygo-t-display-4mb flash-monitor /dev/ttyUSB0
```

Replace the port with yours; without a port the helper uses the only
matching one it finds. Each variant builds in its own directory
(`build-lilygo-t-display/`, `build-lilygo-t-display-4mb/`).

The 4 MB profile uses `partitions_lilygo_tdisplay_4mb.csv` (one factory app
slot taking the rest of the flash), because the SDK's standard partition
table ends past 4 MB — the same approach as the 4 MB Waveshare
ESP32-C6-LCD-1.47 profile. The 16 MB profile uses the standard table
unchanged.

If flashing over the CH9102 bridge fails partway or the port drops, retry
at a lower baud — hold the GPIO0 button, tap reset, and run the build
directory's flash step with `idf.py -b 230400 … flash`, as for the M5Stack
StickC Plus2, whose CH9102 bridge drops out above 230400 baud.

## Pair it

Once flashed, the status bars breathe orange: the gadget is advertising.
In the Muse app, turn on **Settings > Devices > Developer mode**, then
**Settings > Devices > Add Device** and pick `MuseGadget-Disp-XXXXXX`.
When the bars breathe blue, press the GPIO0 (right-hand) button to confirm
the pairing. The bars turn solid blue while it joins Wi-Fi and connects,
then give way to the green dot.

- **Short press** of the GPIO0 button: confirm a pending pairing, or reopen
  the setup window if setup isn't complete.
- **Hold for 5 s**: reset setup — unpair and forget Wi-Fi.

## Validation

This port was built for both flash variants and checked with the SDK's
host tests and the overlay-applied config check from `devices/AGENTS.md`.
The first hardware flash attempt (2026-10-16, a 4 MB revision-1.0 unit)
exposed the chip-revision signing issue above, fixed the same day; the
rebuilt images declare chip revision v1.0–v3.99 for both the bootloader
and the app. Still to confirm on hardware: the panel orientation and
offsets (portrait, image centred with no clipped edges), the backlight,
and a BLE scan showing `MuseGadget-Disp-XXXXXX`.
