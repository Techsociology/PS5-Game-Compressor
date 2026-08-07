# PS5 Game Compressor

A standalone PS5 payload for compressing, unpacking, validating, repairing,
and moving [ShadowMountPlus](https://github.com/drakmor/ShadowMountPlus)-mounted
games from a simple web UI.

Pick a title, choose an action, and let the PS5 do the work. Long operations
keep running on the console even if you close the browser window.

```text
http://<PS5_IP>:5910/
```

---

## Table of contents

- [Features](#features)
- [Requirements](#requirements)
- [Build](#build)
- [Deploy](#deploy)
- [Usage](#usage)
- [Compression settings](#compression-settings)
- [APR Emu support](#apr-emu-support)
- [Game discovery](#game-discovery)
- [Shutting down](#shutting-down)
- [License](#license)
- [Credits](#credits)
- [Disclaimer](#disclaimer)

---

## Features

- Compress mounted game folders or images into `.ffpfsc` (compressed PFS)
  output, with a choice of nested `PFS` or `exFAT` image format.
- Automatically build the APR Emu `ampr_emu.index` before compressing APR
  titles, or run `Build AMPR Index` manually to refresh it on its own.
- Validate compressed games and repair detected PFSC block issues where
  possible.
- Uncompress compressed games back to folder/app form.
- Move supported titles between internal storage and USB storage.
- Track progress, speed, ETA, and full operation history.
- Keep jobs running on the PS5 even if the browser tab or window closes.
- Install or refresh a home-screen launcher tile:

  ```text
  Game Compressor / PSGC50001 -> http://127.0.0.1:5910/
  ```

  The tile just opens the web UI — it isn't the worker itself. Compression,
  validation, repair, move, and uncompress jobs are owned by the PS5 side
  and survive the browser tab closing or the tile being reopened later.
- Light and dark UI themes, remembered across sessions.

## Requirements

- A PS5 homebrew environment capable of running payload ELFs.
- [ShadowMountPlus](https://github.com/drakmor/ShadowMountPlus) (latest
  version), installed and managing mounted titles.
- [KStuff Lite](https://github.com/EchoStretch/kstuff-lite/releases/tag/v1.07)
  1.07 Beta or later.
- [Payload Manager](https://github.com/itsPLK/ps5-payload-manager) or another
  way to launch `game-compressor.elf`.

This is homebrew software. You're expected to understand the risks of
running PS5 homebrew payloads. Keep backups of important data and test with
non-critical titles first.

## Build

```sh
export PS5_PAYLOAD_SDK=/path/to/ps5-payload-sdk
make
```

This produces `game-compressor.elf`. Generated build output (`build/`,
`gen/`, the ELF itself) is git-ignored.

The build needs zlib **cross-compiled for the PS5 target** — a host-installed
zlib (e.g. from `apt`) won't link. `ZLIB_INCLUDE`/`ZLIB_LIB` in the `Makefile`
default to `$(PS5_PAYLOAD_SDK)/include` and `$(PS5_PAYLOAD_SDK)/lib/libz.a`;
see [`.github/workflows/build.yml`](.github/workflows/build.yml) for a
working example of cross-compiling zlib with the SDK's own
`prospero-clang`/`prospero-ar`/`prospero-ranlib` and installing it there.
Override either variable if your zlib lives somewhere else.

### CI

- **`_build.yml`** is a reusable workflow holding the actual build logic
  (deps, downloading the SDK, cross-compiling zlib, `make`). The other
  three call it instead of duplicating it.
- **`build.yml`** builds automatically on every push/PR to `main`.
- **`release.yml`** is triggered manually (Actions tab → Release → Run
  workflow, or `gh workflow run release.yml -f tag_name=v1.1.0`). It builds,
  creates the `v1.1.0`-style tag, and publishes a GitHub Release with the
  ELF and its sha256 checksum attached.
- **`prerelease.yml`** works the same way but is triggered separately and
  marks the release as a pre-release — use a tag like `v1.1.0-rc1`.

CI builds against the **latest** `ps5-payload-sdk` release by default. The
`latest` alias is resolved to a real tag at the start of the build (logged
in the job summary), and that resolved tag — not the literal word
`latest` — is what the build cache is keyed on, so a new SDK release
naturally invalidates the old cache instead of silently reusing a stale
build. If you ever want to pin to a specific SDK version instead, set a
`PS5_SDK_VERSION` repository variable (Settings → Secrets and variables →
Actions → Variables) — all three workflows read it.

Release notes are auto-generated from commits/PRs on each tagged release —
check the [Releases](../../releases) page for version history.

## Deploy

Copy the built ELF to your PS5 payload folder. A typical Payload Manager
path:

```text
/data/pldmgr/payloads/game-compressor/game-compressor.elf
```

## Usage

1. Make sure ShadowMountPlus has mounted one or more games.
2. Launch `game-compressor.elf` from Payload Manager or your payload loader.
3. Open `http://<PS5_IP>:5910/`.
4. Pick a game from the sidebar.
5. Use the primary action — `Compress` for folder/image titles,
   `Validate and Repair` for compressed ones.
6. When compressing, choose `PFS` or `exFAT`.
7. Use the secondary action menu for `Build AMPR Index`, `Uncompress`,
   `Move to USB`, or `Move to Internal SSD`.
8. Check the History button to review past operations.

The UI remembers the last game you viewed via a browser cookie, and picks
back up on the current job if you close and reopen the tab mid-operation.

## Compression settings

`Compress` always produces a `.ffpfsc` file — a compressed PFS container.
You're asked for three things:

**Format** — the nested image stored inside that container:
- `exFAT` (default, recommended) — preferred for most games, especially APR
  Emu workflows.
- `PFS Experimental` — only if you specifically want to test the PFS
  nested-image path.

**Destination**:
- `Compress in place` — writes next to the currently selected game.
- `Internal SSD` — writes under `/data/homebrew` (shown only if the game
  isn't already on internal storage).
- `External Storage` — writes to a selected USB/external target; if the
  game is already external, you may be offered a `Compress to...` picker.

**Original handling**:
- `Keep original` — safest; needs enough free space for the compressed copy.
- `Delete after verified` — writes and validates first, then removes the
  source. Default for in-place compression. Still needs full-size temporary
  free space, since the original is kept until verification passes.
- `Destructive` — deletes source data while writing. Needs at least 1 GB
  free, can't be cancelled once the unsafe phase starts, and only applies to
  same-storage folder compression (not cross-drive or `Make Image`).

Compressing to internal SSD or external storage keeps the original by
default; choosing to remove it uses the safer `Delete after verified` path.

## APR Emu support

APR Emu titles need an `ampr_emu.index` file and the correct ShadowMountPlus
read-only/sector-size settings to run from internal SSD. Game Compressor
handles the common cases:

- **Compress an APR Emu folder-format game from USB, run it from internal
  SSD**: select the title on USB, choose `Compress`. Game Compressor builds
  `ampr_emu.index` (same approach as `build_ampr_index.py`), sets the
  ShadowMountPlus read-only/sector-size options, creates the `.ffpfsc`
  image, mounts it, and validates it byte-for-byte against the original.
- **Compressed game stutters, want it on internal SSD anyway**: secondary
  menu → `Uncompress` → `exFAT`. Existing ShadowMountPlus settings and
  `ampr_emu.index` are left as-is; an uncompressed exFAT image is created.
- **Already know it should stay uncompressed**: secondary menu →
  `Make Image` → `exFAT` + `Internal SSD`. Game Compressor detects the APR
  Emu title, builds/refreshes `ampr_emu.index`, applies read-only settings,
  and creates the uncompressed image.
- **Already have an exFAT image and an `ampr_emu.index`**: use
  `Set Read Only` from the secondary menu to apply ShadowMountPlus's
  read-only settings.

The one unsupported automatic case: an existing exFAT image with no
`ampr_emu.index`. Run the game once from external USB without read-only
settings so APR Emu can create its index, confirm it starts, then copy it to
internal SSD and use `Set Read Only`.

Non-APR titles use the normal compression path. When APR indexing runs, the
game screen and operation history show `APR indexed`.

The in-app `APR-EMU Version` picker reads Pippo's public APR-EMU manifest
and binary mirror at build/run time — there's no version pinned in this
repo:

```text
https://pippo26442999.github.io/.exFAT/ampr-emu-drakmor/manifest.json
```

Manifest entries are downloaded by the browser, uploaded to Game Compressor,
cached under `/data/GameCompressor/ampr-emu`, then applied to the selected
title or image. Custom `.sprx`/`.prx` files can also be uploaded manually
from a desktop browser. Upstream source: [drakmor/ampr_emu](https://github.com/drakmor/ampr_emu).

## Game discovery

Mounted games are shown automatically. Game Compressor also scans:

```text
/data/homebrew
/data/etaHEN/games
/mnt/ext0, /mnt/ext0/homebrew, /mnt/ext0/etaHEN/games
/mnt/ext1, /mnt/ext1/homebrew, /mnt/ext1/etaHEN/games
/mnt/usb0 .. /mnt/usb7 (and their /homebrew, /etaHEN/games subpaths)
```

## Shutting down

Game Compressor should only run while you're actively compressing,
validating, repairing, moving, or unpacking games. Once you're done, use the
terminate button in the top bar — it removes the home-screen tile, stops the
payload, and shows a final screen telling you to close the browser window.

Notes:
- Launcher installation is nonfatal — if it fails, the UI still works
  directly at `http://<PS5_IP>:5910/`.
- Payload operations are owned by the PS5-side worker, not the browser tab.
- Compression and repair on large titles can take a while.

## License

Licensed under the [GNU General Public License v3.0](LICENSE) or (at your
option) any later version.

This project links against [`ps5-payload-sdk`](https://github.com/ps5-payload-dev/sdk)
(GPLv3) and is built on top of [ShadowMountPlus](https://github.com/drakmor/ShadowMountPlus)
(GPLv3) for mounted-title support and PS5-side `ampr_emu.index` generation —
GPLv3 keeps this compatible with both.

## Credits

Created by Juma Sayeh. Tested by Osama Abualia.

Thanks to Pippo (`pippo26442999`) for maintaining the public APR-EMU
manifest and binary mirror used by the in-app APR-EMU version picker.

Built on and inspired by:

- [PSBrew/MkPFS](https://github.com/PSBrew/MkPFS)
- Drakmor's [ShadowMountPlus](https://github.com/drakmor/ShadowMountPlus),
  [APR Emu](https://github.com/drakmor/ampr_emu), and `build_ampr_index.py`
  work, which Game Compressor builds on for mounted-title support and
  PS5-side `ampr_emu.index` generation.

Made with love in Palestine.

## Disclaimer

Experimental PS5 homebrew software. Use at your own risk. Not affiliated
with Sony, PlayStation, or any game publisher.
