# Effects and the redesigned UI: notes for effect authors

The Look tab shows only the controls an effect really uses, and says which colors come from the palette. It learns this from two
places: the effect's **metadata string** (static) and what the effect **does while it runs** (reported by the firmware as
[`pcol`](json-api.md)). This page lists what effect code should do so the UI is right.

## 1. Metadata: hide what the effect does not read

Format (see the [upstream description](https://kno.wled.ge/interfaces/json-api/#effect-metadata)):

```text
"Name@sliders;colors;palette;flags;defaults"
```

- **Sliders** (speed, intensity, custom1-3, then checkboxes 1-3): an empty entry hides that control; `!` uses the default label. Name the
  controls after what they do. The UI shows the names as visible labels, so put each name on the control the code really reads
  (Waverly had "Blur" on the wrong checkbox).
- **Colors** (3 entries): an empty entry hides that color; `!` shows it with its default name ("Main", "Background", "Accent").
  Leave a color empty if the effect reaches it only through "\* Color" palettes or only with some settings. The UI shows hidden colors
  again for "\* Color" palettes and while the firmware reports that the effect uses them (`pcol`), so hiding them is safe.
- **Palette**: empty hides the palette list. Leave it empty if the effect never calls `color_from_palette()`, `color_wheel()` with a
  palette, or `SEGPALETTE`.
- **Defaults**: `pal=<id>` is the palette the effect shows with "Default" (`Segment::setMode()` stores it as `_default_palette`;
  0 or missing means Party). The UI uses it to name "Default" and to keep the color picker for Default when `pal=` is a "\* Color"
  palette (Railway `pal=3`, Slow Transition `pal=2`).

## 2. Drawing: tell the firmware which colors a palette replaces

The firmware notes what each effect reports while it runs. The Look tab uses this to put the My color / Palette switch on the right
colors and to name "Default".

### Use `color_from_palette()` with the right slot

```cpp
uint32_t c = SEGMENT.color_from_palette(index, mapping, moving, slot);
```

- `slot` 0-2: with "Default" this returns color `slot`; with any other palette it returns the palette color. Pass the color the palette
  replaces, so the switch appears on that color.
- `slot` 255 (any value of 3 or more): palette-only drawing. It never returns a color slot; with Default it draws the effect's own
  palette. Use it when the effect never shows a user color there.
- `color_wheel()` reports nothing about color slots (it uses slot 255 internally). With Default it draws a rainbow, which the UI calls
  "built-in colors".
- `SEGPALETTE` (the current palette) reports "the effect draws its own palette with Default". It is a function call, so in loops over
  all pixels or particles keep a reference: `const CRGBPalette16 &pal = SEGPALETTE;`.

### Effects that pick between palette and colors themselves: `addPaletteColors()`

If the effect reads `SEGCOLOR(n)` directly in one case and the palette in another (for example "palette unless color 3 is set"),
the firmware cannot see it. Declare the colors the palette replaces:

```cpp
SEGMENT.addPaletteColors(hasCol2 ? 0b111 : 0b001); // bit n = color slot n is replaced by a palette
```

- Call it **on every frame** the effect draws, with the colors used **right now** (0 if none). After each state change the firmware
  collects what the effect reports for 0.5 s and then replaces the learned set, so colors the effect stops using drop out. A call
  made only once (for example only on the first frame) gets lost at the next change.
- Examples in the code: Fire Flicker, Bouncing Balls, Rolling Balls, Popcorn, Scrolling Text (gradient), Colorful, Black Hole,
  Ants (`usermods/user_fx`).

### What the firmware does with it

- `Segment::color_from_palette()`, `effectPalette()` (`SEGPALETTE`) and `color_wheel()` call `addPaletteColors()` themselves.
- `WS2812FX::service()` calls `Segment::endPaletteFrame()` after each effect call. It marks that the effect ran with Default and, 0.5 s
  (`PALCOL_LEARN_TIME`) after a state change, replaces the learned colors with what the effect reported since then.
- `stateUpdated()` (`led.cpp`) starts that relearning on every segment when the state changed.
- The result is the segment's `pcol` value (see [json-api.md](json-api.md)). When it changes, a websocket-only update is sent
  (`CALL_MODE_WS_SEND`: no MQTT, no sync, current preset kept).
- An effect that reports nothing (for example Solid, which uses `SEGCOLOR(0)` directly) keeps what was learned.

## 3. Checklist for a new or changed effect

- [ ] Metadata hides every slider, checkbox, color and the palette the code does not read; labels sit on the controls the code reads.
- [ ] Colors the palette replaces are drawn with `color_from_palette(…, slot)`; palette-only drawing uses slot 255.
- [ ] If the effect itself chooses between `SEGCOLOR()` and the palette: `addPaletteColors()` on every drawing frame, with the
      current colors.
- [ ] `pal=` is set if "Default" should show a specific palette.
- [ ] `SEGPALETTE` is read once before per-pixel loops.
