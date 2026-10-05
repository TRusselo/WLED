# Testing log

What was tested for the redesign, how, and what was not. "Verified" means it was run and checked; anything else is listed under
"Not tested".

## Builds and automated checks

- **CI** (`.github/workflows/wled-ci.yml`): firmware for all 27 default environments (ESP32, ESP32-S2/S3/C3/C5/C6/P4, ESP8266 incl.
  compat builds, usermods) and `npm test`. Green on every commit, most recently 268b5c4e (A) and cb5fb6ef (B).
- `tools/cdata-test.js` has a timing-sensitive check that failed once on a slow runner (d480de1, PR run) and passed on re-run; it is not
  related to this work (see [README.md](README.md#known-limits)).
- **Local compile checks** (ESP32 toolchain, arduino-esp32 v3 / IDF 5.5, `-fsyntax-only`): FX.cpp, FX_fcn.cpp, led.cpp, json.cpp,
  FXparticleSystem.cpp, user_fx.cpp, TetrisAI_v2.cpp. No warnings on the changed lines.
- **`sizeof(Segment)`** stays 72 bytes on ESP32: checked with a `static_assert` compiled with the ESP32 toolchain (and a negative control
  that fails with 76).
- **actionlint** 1.7.12 passes for `.github/workflows/beta.yml`; its publish script was run locally with a stand-in `gh` (branch names
  with `/`, `.bin` and `.bin.gz` files, no files → fails). The workflow then published both betas on GitHub.

## Web UI in a browser (mock device)

Headless Chromium (Playwright) against a mock WLED server that serves `wled00/data` and answers the JSON API with the firmware's effect
and palette data. The mock reports `pcol` from a static scan of FX.cpp, not from running firmware. No JS errors in any run.

- Layout: phone (390 px), tablet (1100 px), desktop (1280 px), Simplified UI on and off.
- My color / Palette switch: Blink, Chase, Bouncing Balls, Glitter, Fire 2012, "\* Colors 1&2"; first "Palette" with and without a
  previous palette; Background / Main; My color / Palette again.
- Railway, Slow Transition, Colorful (128 / 200), Flow, Black Hole (Solid off / on / off).
- "Default" naming: Fire 2012, Juggle, Blink, Railway; Default seen and not yet seen.
- Effect and palette dialogs, segment chip.
- Code review fixes (3d55f70): white and white balance sliders on an RGBW+CCT strip (Rainbow; Blink with Ocean and with Default);
  Presets toolbar not sticky; dialog clicks on its padding and on the backdrop (also real mouse clicks); Random palette preview stable;
  layout recalculated only when Simplified UI changes; state read again over HTTP 1.5 s after a change; 2D live view style; filter chips
  without `:has()`; obsolete `pcmbot` setting dropped.
- Variant B (palette card): first "Palette" (Party, card shown, picker hidden), picking palettes in the card (card stays), Background /
  Main, My color / Palette again, "\* Colors 1&2" with the popup, Fire 2012 with Sunset and Default, palette changed from outside (card
  scrolls to it), click on a visible palette (no jump).
- Not run in a browser: 268b5c4e (the 2D chip is shown without a matrix; the change only removes the code that hid it). It was flashed
  to the Dig-Uno, see below.

## Firmware learning logic (host simulation)

The learning functions of 3d55f70 (`addPaletteColors()`, `endPaletteFrame()`, `relearnPaletteColors()`) were copied into a host program
with a simulated clock: Bouncing Balls (color 3 set, then black), Black Hole (Solid off / on, with Default and with a palette), an effect
that reports nothing, an effect that reports only every 800 ms, a color added between changes. All as expected.

## Hardware: QuinLED Dig-Uno v3 (ESP32), RGB strip, no matrix

- **6e2f497** (CI build): the My color / Palette switch appears, so the firmware reports `pcol`.
- **25bf6ed** (beta B, 2026-10-05): `pcol` learning checked through the JSON API with a script that saved the device state, ran the
  steps, polled `pcol` every 0.1 s and restored the state:

  | Step | `pcol` in the reply | then | Expected |
  |---|---|---|---|
  | Bouncing Balls, palette Party, color 3 red | 0 (new effect) | 7 after 0.28 s | 7 |
  | color 3 → black | 7 | 1 after 0.49 s | 1 (Background drops) |
  | color 3 → red | 1 | 7 after 0.57 s | 7 |
  | Bouncing Balls, Default, color 3 black | 7 | 23 (has run with Default added), then 17 after 0.51 s | 17 |
  | Colorful, Default, slider 200 | 0 | 16, then 23 after 0.31 s | 23 |
  | slider → 100 | 23 | 16 after 0.55 s | 16 (colors drop) |
  | slider → 200 | 16 | 23 after 0.25 s | 23 |
  | Fire 2012, Default | 16 | 24 after 0.44 s | 24 |
  | Fire 2012, speed change only | 24 | 24 throughout | no change, no drop |

  Times are measured from the reply to the request, so the 0.5 s learning window can appear shorter. The value never dropped while
  relearning.
- **cb5fb6ef** (beta B with the 2D chip): flashed over the network (`/update`), "Update successful", back after about 10 s; the served page
  is variant B and no longer hides the 2D chip; presets file unchanged; `pcol` learned again after the reboot.

## Not tested

- The UI of 3d55f70 and later on a phone or computer against real hardware (the owner is testing variant B).
- ESP8266 hardware, 2D matrices, RGBW / CCT strips on real hardware, audio reactive effects.
- Browsers other than Chromium (Firefox, Safari, older Android WebViews).
- Effects not listed above: the `pcol` behaviour of every effect was not checked one by one.
