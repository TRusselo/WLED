# Web UI redesign: notes

Notes for this fork (TRusselo/WLED) on the web UI redesign. They are not meant for a pull request to wled/WLED.
Written with AI assistance (Claude Code) and reviewed by the fork owner.

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
