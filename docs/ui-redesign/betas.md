# Beta builds and the A/B variants

## Variants

| | A: palette button with popup | B: palette card |
|---|---|---|
| Branch | `ui-redesign` ([PR #1](https://github.com/TRusselo/WLED/pull/1)) | `ui-redesign-palette-card` |
| Beta | [`beta-ui-redesign`](https://github.com/TRusselo/WLED/releases/tag/beta-ui-redesign) | [`beta-ui-redesign-palette-card`](https://github.com/TRusselo/WLED/releases/tag/beta-ui-redesign-palette-card) |
| With "Palette" selected for a color, and for palette-only effects | the palette is a button; it opens the palette list as a popup, which closes after a palette is picked | the palette list is shown as a card where the color picker was and stays open |
| Main page, gzipped | 40,237 B | 40,376 B |

Everything else is the same. B is A plus one commit (25bf6ed); changes to A are merged into B so both stay current.
B brings back the inline list of e3ce97c, which a WLED developer preferred; A is the popup list the fork owner prefers.

## Publishing a beta

The workflow `.github/workflows/beta.yml` ("Deploy Beta") exists only in this fork.

1. Actions tab > **Deploy Beta** > **Run workflow** > choose the branch.
2. It builds the branch with the shared `build.yml` (the same environments and `npm test` as CI). A failed build or test publishes
   nothing.
3. It replaces the pre-release `beta-<branch>` (a `/` in the branch name becomes `-`) and moves its tag to the built commit. The
   download links stay the same.

From the command line: `gh workflow run beta.yml -R TRusselo/WLED --ref <branch>`.

Notes:

- GitHub reads a manually started workflow from the chosen branch, so `beta.yml` must exist on that branch. Branches made from `main`
  have it; `ui-redesign` got it by merging `main` (d480de1).
- Pre-releases are never GitHub's "latest release", so the update check on devices (the update page, which asks GitHub for the latest
  release of the build's repository) does not offer them.
- Releases on this fork are public and can be downloaded without signing in. The release text says they are unstable test builds.
- The workflow does nothing on wled/WLED or for tags. Leave `beta.yml` out of anything sent to wled/WLED.

## Installing a beta

1. Open the release and download the `.bin` for the board: `WLED_17.0.0-devV5_ESP32.bin` for most ESP32 boards (for example the
   QuinLED Dig-Uno), `..._ESP8266.bin` for ESP8266 boards, and so on.
2. On the device: Config > Security & Updates > "Update WLED" (the page `/update`), upload the file. Settings and presets are kept.
3. After the reboot the device starts with its boot preset or default state, so an effect that was running but not saved in a preset is
   not restored. Reload the UI page (hard refresh if the old page shows).

The same upload works from a computer on the same network:

```sh
curl -F "update=@WLED_17.0.0-devV5_ESP32.bin" http://<device-ip>/update
```

This was done for the Dig-Uno on 2026-10-05 (beta B at cb5fb6ef); see [testing.md](testing.md).
