# JSON API: segment field `pcol`

The redesign adds one **read-only** field to each segment in the state (`/json/state`, `/json/si`, websocket updates):

```json
{"seg": [{"id": 0, "fx": 91, "pal": 6, "pcol": 7, "...": "..."}]}
```

It is not saved in presets and is ignored when sent to the device.

## Bits

| Bit | Value | Name in the firmware | Meaning |
|---|---|---|---|
| 0 | 1 | | the effect draws color 1 through the palette |
| 1 | 2 | | the effect draws color 2 through the palette |
| 2 | 4 | | the effect draws color 3 through the palette |
| 3 | 8 | `PALCOL_DEFAULT_PALETTE` | with "Default", the effect draws its own palette (`pal=` in its metadata, else Party) |
| 4 | 16 | `PALCOL_DEFAULT_SEEN` | the effect has run with "Default", so bit 3 is known |

Bits 0-2 are the colors that any palette except "Default" replaces. Defined in `wled00/FX.h`, serialized in `serializeSegment()`
(`wled00/json.cpp`).

## How the UI uses it

- Bits 0-2 put the My color / Palette switch on those colors, and show colors that the metadata hides while the effect uses them.
- Bits 3-4 name "Default": own palette (bit 3), your colors (bits 0-2), both, or "built-in colors" (neither), once bit 4 is set.

## When it changes

- Cleared when the effect changes, then learned while the effect runs.
- After every state change it is learned again: for 0.5 s the colors the effect reports are collected, then they replace the previous
  value. Until then the previous value stays, so a client never sees it drop to 0 in between (unless the effect changed).
- Changes are sent to websocket clients by a websocket-only update, at most once per second (the interface update cooldown).
- Clients without websockets should read the state again about 1.5 s after a change; the web UI does this.
- Nothing is learned while the segment is off or frozen.

## Examples (measured on an ESP32, see [testing.md](testing.md))

| Situation | `pcol` |
|---|---|
| Bouncing Balls, palette Party, color 3 set | 7 |
| same, color 3 black | 1 |
| Bouncing Balls, Default, color 3 black | 17 (color 1, has run with Default) |
| Colorful, Default, "Pastel / Classic / My colors" 200 | 23 (three colors, has run with Default) |
| same, slider at 100 | 16 (no colors, has run with Default) |
| Fire 2012, Default | 24 (own palette, has run with Default) |
