# Web UI redesign: notes

Notes for this fork (TRusselo/WLED) on the web UI redesign. They are not meant for a pull request to wled/WLED.
Written with AI assistance (Claude Code) and reviewed by the fork owner.

There are currently 2 varients:
one that has the palette picker as a popup (ui-redesign), and one that keeps it shown as a card when viable requiring 1 less click. (ui-redesign-palette-picker)

## Purpose
- Purposeful inconsistency between effects, and UI presenting options that cannot be changed with other current settings makes the UI feel confusing at times and like "you just click shit until its good enough".
- Claude's diagnosis found 4 main reasons:

1. **Choosing an effect on page 2 can replace your selected palette on page 1.** `setFX()` sends `fxdef: true` by default (`wled00/data/index.js:35`, `:2451`). When an effect has a palette default, the firmware then calls `setPalette()` (`wled00/FX_fcn.cpp:633-634`). 59 of the 216 effects in `FX.cpp` have one. So you pick a palette on page 1, pick an effect on page 2, and your palette is gone.
2. **Your color choice is often ignored without any warning.** Palette colors are applied to different layers for each effect, could be Bg, Fg, or Fx ). Effects that read colors through `color_from_palette()` only use your color slots when the palette is "Default" (`FX_fcn.cpp:1188-1191`). With any other palette, those effects use the palette instead, but the color picker still looks active. 
3. **"Default" palette means something different for each effect.** It switches to whatever palette that effect declares, or Party colors if it declares none (`FX_fcn.cpp:232`, `:635-636`). Default palette, could be an existing palette, individual colours, gradient, ect, or not set at all.
4. **The colors page depends on a choice made on the next page.** The effect decides which color slots appear and what they're called (`index.js:1684-1714`). The generic labels are "Fx", "Bg", "Cs" (`index.js:1698-1700`), and effects that define their own names get cut to 2 letters with a legend underneath (`:1691-1695`).

## Goal

Choosing a look (effect, colors, palette) should be clear and truthful:

- effect, colors and palette on one tab ("Look"), effect first;
- only the controls the selected effect really uses;
- say which colors come from the palette, and what "Default" means for the current effect;
- keep the page and the firmware about as small as before.

## Where things are

| What | Where |
|---|---|
| Redesign, variant A (palette button with popup list) | branch `ui-redesign`, [PR #1](https://github.com/TRusselo/WLED/pull/1) into `main` |
| Variant B (palette list as a card in place of the color picker) | branch `ui-redesign-palette-card` (A plus one commit) |
| Beta builds | pre-releases [`beta-ui-redesign`](https://github.com/TRusselo/WLED/releases/tag/beta-ui-redesign) and [`beta-ui-redesign-palette-card`](https://github.com/TRusselo/WLED/releases/tag/beta-ui-redesign-palette-card) |
| Beta workflow | `.github/workflows/beta.yml` (fork only), see [betas.md](betas.md) |

## Notes in this folder

- [user-guide.md](user-guide.md): how the new UI works, for users.
- [effect-authors.md](effect-authors.md): what effect code and metadata need so the UI shows the right controls.
- [json-api.md](json-api.md): the read-only segment field `pcol`.
- [betas.md](betas.md): beta pre-releases, the A/B variants, installing a beta.
- [testing.md](testing.md): what was tested, how, and what was not; sizes.

## Status (2026-10-05)

- PR #1 is open, CI green. Variant A and B betas are published.
- Variant B runs on the owner's QuinLED Dig-Uno v3 (ESP32). The firmware part (`pcol` learning) was checked on it through the JSON API, see [testing.md](testing.md).
- Open: choose A or B; decide whether to win back the remaining 270 bytes of page size (see below); user documentation in WLED-Docs format if ever needed.

## Size compared with upstream

Upstream here means `main` of this fork at 6720ce4 (wled/WLED `main` at the fork point). Numbers from the CI build logs.

| | Upstream | Variant A | Variant B |
|---|---|---|---|
| Main page, gzipped (`PAGE_index`) | 39,967 B | 40,237 B (+270) | 40,376 B (+409) |
| ESP32 (esp32dev) flash | 1,335,963 B | 1,337,095 B (+1,132) | 1,337,239 B (+1,276) |
| ESP32 RAM | 86,952 B | 86,976 B (+24) | 86,976 B (+24) |
| ESP8266 (nodemcuv2) flash | 930,435 B | 931,475 B (+1,040) | 931,603 B (+1,168) |
| ESP8266 RAM | 46,860 B | 46,868 B (+8) | 46,868 B (+8) |

The flash numbers include the page. The rest (about 850 bytes on ESP32, an estimate: flash difference minus page difference) is the
firmware code for `pcol`. `sizeof(Segment)` is unchanged (72 bytes on ESP32): the two new bytes use existing padding.

Ways to get the page back under the original size, not done: remove the "Changing: segment" chip, or the words on the filter chips.

## Known limits

- `pcol` is learned while the effect runs. After a change, the My color / Palette switch and the "Default" name can take up to about
  1.5 seconds (0.5 s learning plus the websocket update cooldown). Nothing is learned while the segment is off or frozen.
- What "Default" shows can only be learned while Default is selected. If an effect's settings change while another palette is selected,
  the Default name keeps the last learned value.
- A color an effect uses only now and then (at random) can be missing for a moment after a change; it comes back when the effect uses it.
- `tools/cdata-test.js` has a timing-sensitive check ("a inlined file changes", 850 ms) that can fail on slow CI runners. It is not part
  of this work; a fix is proposed in a [PR #1 comment](https://github.com/TRusselo/WLED/pull/1#issuecomment-5986843892).


<p align="center">
  <img src="/images/wled_logo_akemi.png">
  <a href="https://github.com/wled-dev/WLED/releases"><img src="https://img.shields.io/github/release/wled-dev/WLED.svg?style=flat-square"></a>
  <a href="https://raw.githubusercontent.com/wled-dev/WLED/main/LICENSE"><img src="https://img.shields.io/github/license/wled-dev/wled?color=blue&style=flat-square"></a>
  <a href="https://wled.discourse.group"><img src="https://img.shields.io/discourse/topics?colorB=blue&label=forum&server=https%3A%2F%2Fwled.discourse.group%2F&style=flat-square"></a>
  <a href="https://discord.gg/QAh7wJHrRM"><img src="https://img.shields.io/discord/473448917040758787.svg?colorB=blue&label=discord&style=flat-square"></a>
  <a href="https://kno.wled.ge"><img src="https://img.shields.io/badge/quick_start-wiki-blue.svg?style=flat-square"></a>
  <a href="https://github.com/Aircoookie/WLED-App"><img src="https://img.shields.io/badge/app-wled-blue.svg?style=flat-square"></a>
  <a href="https://gitpod.io/#https://github.com/wled-dev/WLED"><img src="https://img.shields.io/badge/Gitpod-ready--to--code-blue?style=flat-square&logo=gitpod"></a>
</p>

# Welcome to WLED! ✨

A fast and feature-rich firmware for ESP32 microcontrollers to control addressable LEDs — from simple strips to large 2D matrices and HUB75 panels.

Originally created by [Aircoookie](https://github.com/Aircoookie), now maintained by a community of contributors.

## 🤝 Contributing

Want to help improve WLED? Awesome! Please skim [CONTRIBUTING.md](CONTRIBUTING.md) first - it covers how we like PRs and issues to look, including our take on AI-assisted contributions.
If you're an AI coding agent, [AGENTS.md](AGENTS.md) is for you - please read it before modifying any files. 😊

## ⚙️ Features

### Effects & Visuals
- [**200+ built-in effects**](https://kno.wled.ge/features/effects/) including classic animations, audio-reactive, and 2D/matrix effects
- [50+ color palettes](https://kno.wled.ge/features/palettes/) plus a built-in **custom palette editor** (PixelForge)
- [**2D LED matrix support**](https://kno.wled.ge/advanced/mapping/) with dedicated 2D effects and flexible panel mapping
- [**HUB75 RGB matrix panel support**](https://kno.wled.ge/advanced/HUB75/) (ESP32)
- [**AudioReactive**](https://kno.wled.ge/advanced/audio-reactive/) effects — included by default, responding to sound via microphone, line-in, or network audio source
- Effect blending for smooth transitions between animations
- Antialiased drawing functions for smooth graphics

### Segments & Control
- [**Segments**](https://kno.wled.ge/features/segments/) — apply different effects, colors and palettes to independent parts of your LED setup simultaneously
- Up to **250 presets** to save and recall colors, effects and segment configurations — supports [playlists](https://kno.wled.ge/features/presets/) for automated cycling
- Nightlight function with configurable dimming curve
- Configurable **Auto Brightness Limiter** (per output) for safe operation

### Hardware Support
- **ESP32** (all variants: original, S2, S3, C3)
- [**Up to 17 LED outputs**](https://kno.wled.ge/features/multi-strip/) on ESP32 using parallel I2S + RMT
- [Addressable LED support](https://kno.wled.ge/basics/compatible-led-strips/): WS2812B, WS2811, WS2815, SK6812, WS2805, TM1914, APA102, WS2801, LPD8806, and many more
- RGBW, [RGB+CCT](https://kno.wled.ge/features/cct/) and white-only strips
- PWM outputs for analog LEDs and dimmers
- [**Ethernet** support](https://kno.wled.ge/features/ethernet-lan/) for a wide range of boards (QuinLED, LILYGO, Olimex, and more)
- Filesystem-based config for easy backup and restore of presets and settings
- Full OTA firmware updates (HTTP + ArduinoOTA), password-protectable

### Connectivity & Integrations
- **WLED app** for [Android](https://play.google.com/store/apps/details?id=ca.cgagnier.wlednativeandroid) and [iOS](https://apps.apple.com/gb/app/wled-native/id6446207239)
- [JSON](https://kno.wled.ge/interfaces/json-api/) and [HTTP request](https://kno.wled.ge/interfaces/http-api/) APIs
- **Multi-WiFi** — connect to up to 3 networks with automatic AP fallback
- **ESP-NOW** wireless sync between devices (no WiFi router required)
- [**MQTT**](https://kno.wled.ge/interfaces/mqtt/) with Home Assistant discovery
- [**E1.31, Art-Net**](https://kno.wled.ge/interfaces/e1.31-dmx/), [DDP](https://kno.wled.ge/interfaces/ddp/) and [TPM2.net](https://kno.wled.ge/interfaces/udp-realtime/) for DMX/professional lighting control
- [UDP realtime sync](https://kno.wled.ge/interfaces/udp-notifier/) across multiple WLED devices
- Alexa voice control (on/off, brightness, color)
- [Philips Hue sync](https://kno.wled.ge/interfaces/philips-hue/)
- [diyHue](https://github.com/diyhue/diyHue) and [Hyperion](https://github.com/hyperion-project/hyperion.ng) integration
- [Adalight / TPM2](https://kno.wled.ge/interfaces/serial/) (PC ambilight via serial)
- [Infrared remote control](https://kno.wled.ge/interfaces/infrared/) (24-key RGB, receiver required)
- Timers and schedules (NTP time sync, full timezone and DST support)

### Developer-Friendly
- **Usermod system** — extend WLED with community or custom modules without modifying core code
- Large and active [usermod library](https://kno.wled.ge/advanced/community-usermods/) including AudioReactive, temperature sensors, rotary encoders, displays, and much more
- Well-documented [JSON API](https://kno.wled.ge/interfaces/json-api/)
- Licensed under the **EUPL v1.2**

## 📲 Quick start guide and documentation

See the [documentation at kno.wled.ge](https://kno.wled.ge)!

[Tutorials and getting-started guides](https://kno.wled.ge/basics/tutorials/) to help you get your project running quickly.

## 🖼️ User interface

<img src="/images/macbook-pro-space-gray-on-the-wooden-table.jpg" width="50%"><img src="/images/walking-with-iphone-x.jpg" width="50%">

## 💾 Compatible hardware

See the [compatible hardware list](https://kno.wled.ge/basics/compatible-hardware) on the wiki.

## ✌️ Other

Licensed under the [EUPL v1.2](https://raw.githubusercontent.com/wled-dev/WLED/main/LICENSE).  
Credits to all [contributors](https://kno.wled.ge/about/contributors/)!  
CORS proxy by [Corsfix](https://corsfix.com/).

![CodeRabbit Pull Request Reviews](https://img.shields.io/coderabbit/prs/github/wled/WLED?utm_source=oss&utm_medium=github&utm_campaign=wled%2FWLED&labelColor=171717&color=FF570A&link=https%3A%2F%2Fcoderabbit.ai&label=PR+Reviews+supported+by+CodeRabbit)

Join the Discord server to discuss everything about WLED!

<a href="https://discord.gg/QAh7wJHrRM"><img src="https://discordapp.com/api/guilds/473448917040758787/widget.png?style=banner2" width="25%"></a>

Check out the WLED [Discourse forum](https://wled.discourse.group)!

If you'd like to reach the original creator privately: [dev.aircoookie@gmail.com](mailto:dev.aircoookie@gmail.com).

If WLED brightens up your day, you can [send a gift to Aircoookie via PayPal](https://paypal.me/aircoookie).

---

*Disclaimer:*

If you are prone to photosensitive epilepsy, we recommend you do **not** use this software.  
If you still want to try, avoid strobe, lightning or noise modes and high effect speed settings.

As per the EUPL license, no liability is assumed for any damage to you or any other person or equipment.
