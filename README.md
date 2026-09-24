# meta-pass — turn the FoloToy AI Passport into a multi-firmware device

[简体中文](README.zh_CN.md) | English

meta-pass is a **multi-firmware launcher** for the FoloToy AI Passport (ESP32-C3,
8MB flash): flash meta-pass once, then install community firmware into three slots at
any time and boot them from a menu — **no more full reflashes**. Plays no longer
overwrite each other, and the launcher is always one power cycle away.

<p align="center">
  <img src="docs/assets/images/meta-pass-cover.png"
       alt="meta-pass launcher: slot list with Pocket Walkie / Passport Radar / empty Slot 2"
       width="800">
</p>

[\![FoloToy plays #281](https://img.shields.io/badge/play-281-informational)](https://ai-passport.folotoy.cn/plays/281)

A stock Passport runs one firmware at a time; trying community plays means reflashing
the whole flash and back. meta-pass lives in the factory partition as a launcher and
turns the remaining flash into three OTA slots for child firmware:

```text
0x000000   bootloader
0x008000   partition table   nvs / phy_init (identical to stock)
0x010000   factory (1.44MB)  ← the meta-pass launcher itself
0x180000   ota_0 (1.84MB)    ← slot 0 (last 4KB = display-name blob)
0x356000   cardid (16KB)     ← device identity; untouched by every channel
0x360000   ota_1 (2MB)       ← slot 1 (last 4KB = display-name blob)
0x560000   ota_2 (2.61MB)    ← slot 2: bootable child slot OR littlefs recording storage
                               dual-use: boot checks for image, else mounts littlefs
0x7FE000   otadata           ← written by the launcher to pick a slot, then reboot
```

Selecting a slot = write otadata + reboot; the stock 2nd-stage bootloader does the
switch. No custom bootloader changes.

## Features

- **Three-slot switching**: each slot row shows firmware name/version/size/SHA-256;
  pick and boot. The display name is written at install time (community firmware all
  carry the template's default `project_name`, so the real name can only come from
  the install channel).
- **Dual-use slot 2**: when empty, slot 2 can be repurposed as littlefs storage (e.g.
  for a future recording firmware); the launcher detects the absence of a valid image
  and mounts the FS instead.
- **Two install channels**:
  - **USB serial install page** (recommended): open a local page in Chrome, hold UP
    while powering on to enter ROM download mode, write straight into a slot; accepts
    a local `.bin` (Full Flash images are unpacked in-page) or a plays marketplace
    link (auto-downloaded and verified against the store's published SHA-256);
  - **Device hotspot + web import**: the device starts a SoftAP and shows a pairing
    code; upload from a phone or computer browser.
- **Integrity checks**: magic, chip id, size and segment structure, with
  `esp_ota_end()` as the authoritative recheck. A slot that fails verification is listed
  as `(invalid)` and cannot be booted until you upload over it again.
- **One-click boot**: UP/DOWN to pick a slot, OK to start it — signed and unsigned
  firmware alike, with no confirmation step. There is no eFuse-enforced signing, so
  malicious firmware would still get full flash access; only install firmware from
  sources you trust.
- **Never trapped in a child**: un-adapted firmware runs as a trial — any reboot
  (including power loss) returns to the launcher. Adapted firmware can persist and
  wires OK LONG to return to the launcher. A child that sleeps on its own idle
  timeout resumes when you press a key.
- **Identity safety**: the `cardid` partition is avoided by every install/flash path;
  `verify_firmware.py` byte-checks the baseline layout in the gate.

<p align="center">
  <img src="docs/assets/images/meta-pass-launcher.png"
       alt="Launcher main list: SLOT 0/1/2 rows with firmware names plus an IMPORT FIRMWARE row"
       width="800">
  &nbsp;&nbsp;&nbsp;
  <img src="docs/assets/images/meta-pass-usb-installer.png"
       alt="USB serial install page: connect, pick slot, pick source, display name, progress and log"
       width="800">
  &nbsp;&nbsp;&nbsp;
  <img src="docs/assets/images/meta-pass-wifi-import.png"
       alt="Wi-Fi import page: SSID, password, one-time pairing code, countdown"
       width="800">
</p>

## Quick start

### 1. Flash meta-pass (once)

Download `meta-pass_v0.2.2.bin` from Releases, or build it yourself (see
"Development"). Then:

```bash
python -m esptool --chip esp32c3 -p <port> -b 460800 \
    write-flash 0x0 meta-pass_v0.2.2.bin
```

The image ends at `0x780000` and never touches `cardid` (flashing tools only erase/write
the covered region; **never** run `erase-flash` on an identity-written device).

### 2. Install child firmware

**Option A: USB serial install page** (no hotspot needed):

Open **https://meta-pass.pages.dev/** in Chrome (hosted page + API proxy, zero
setup) — or run locally with `node tools/install-slot/server.mjs` →
http://localhost:4191/.

Hold UP while powering on → Connect in the page → pick a slot → choose a local file
or paste a plays link → Install → power-cycle. Full guide:
[install-slot/README.md](install-slot/README.md).

**Option B: device hotspot import** (no Chrome required):

Pick IMPORT FIRMWARE in the main list → the device starts a hotspot and shows a pairing
code → connect from a phone/computer, open `192.168.4.1` → enter the code, pick a
slot, upload a `.bin`.

### 3. Boot

UP/DOWN to pick a slot, then OK to boot it right away. Booting is one press whatever the
firmware's signature status; a slot holding an empty or invalid image does nothing.

## Button map

| Page | UP/DOWN | OK click | OK LONG (1.5 s) |
| --- | --- | --- | --- |
| Main list | select slot | boot the slot / open import page | — |
| Import page | — | — | exit import, back to list |
| Easter egg | scroll text | back to list | — |

Fast `UP UP DOWN DOWN` on the main list opens the selected slot's easter-egg text.
To replace or clear a slot's firmware, re-import over it (the web import page is the only
slot-writing path); there is no on-device delete.

Inside an adapted child firmware: OK LONG = return to launcher (wired by the child,
see below).

## Adapting a child firmware (optional)

Children work unmodified (trial-boot mode). To persist across reboots and get OK LONG
return, include `main/metapass_hook.h` and wire two calls:

1. Call `metapass_mark_valid()` after self-check as an optional signature self-diagnostic
   (return value only — children are single-session: every power-on returns to the launcher
   list page, so OK LONG always works as the escape hatch);
2. Route the OK key's LONG (1.5 s) event to `metapass_return_to_launcher()`.

Signed badge (optional): `tools/signing/sign-firmware.sh <app.bin> [--egg-text "..."]`
appends an ECDSA-P256 badge (+ optional easter-egg text) after the image; meta-pass then
records SIGNED for that slot (logged at boot). Signing does not change the boot path — the
badge is provenance metadata, not a gate. Input may be a bare
app image **or the full merged image (bootloader + partition table + app — the
marketplace flashable format)**; merged inputs keep their header bytes byte-for-byte and
only the pad + 4 KB metadata sector are appended. The private key lives
in the macOS Keychain (created once via `tools/signing/bin/keychain-keygen`, which also
publishes `tools/signing/public.pem`); the signing key is held by the meta-pass
publisher — third-party developers submit binaries for signing rather than self-signing.

## Repository layout

| Path | Content |
| --- | --- |
| `main/` | Launcher UI (`main.c`), storage layer (`meta_store`), Wi-Fi import (`meta_net`), pure-logic modules (`meta_image`/`meta_slots`/`meta_import`/`meta_name`), child-firmware hook (`metapass_hook.h`) |
| `components/bsp/` | Board support package (stock + explicit `BSP_BTN_LONG` 1.5 s threshold) |
| `install-slot/` | USB serial install page, live at https://meta-pass.pages.dev/ (Cloudflare Pages: static assets + `_worker.js` API proxy) |
| `tools/install-slot/` | `server.mjs` localhost server (serves the canonical `install-slot/` page directly — single source, zero dependencies) |
| `tools/validate.sh` | Unified gate: static checks + host tests + firmware build + protected-layout verification |
| `tools/build-firmware.sh` | One-command local firmware build for beginners (finds ESP-IDF, builds, merges, verifies, prints flashing guide) |
| `tests/` | Host tests (C, pure-logic modules, run on PC) |
| `docs/assets/meta-pass-design.md` | Design document (decision log and acceptance checklist) |

## Development

### Build the firmware locally (one command)

```bash
tools/build-firmware.sh
```

That's it. The script finds ESP-IDF v5.5.3 automatically (`~/esp/esp-idf-v5.5.3`,
or pass `--idf-path <dir>`), builds, merges the 8 MB full image, verifies the
protected layout **and upgrade safety** (the NVS / cardid / ota_0-2 / otadata
regions must stay erased in the artifact), and drops artifacts into `build/`.
The **only release artifact** is `meta-pass_v<version>.bin` (~1.1 MB): a hybrid
single file — bootable body (bootloader + partition table + phy + app) plus a
44-byte `MPUPV2` footer — that serves both the market install (flash tools
write it raw at 0x0) and the USB-installer launcher upgrade (the page verifies
the footer and slices the upgrade segments out of the body). If ESP-IDF is
missing it prints step-by-step install commands. First-time install of ESP-IDF:

```bash
mkdir -p ~/esp
git clone -b v5.5.3 --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf-v5.5.3
~/esp/esp-idf-v5.5.3/install.sh esp32c3
```

### Upgrading the launcher without losing data

The partition layout is stable, so launcher upgrades never need to touch user
data. Two paths:

- **USB installer page (recommended)** — section "7. Upgrade launcher": pick
  the **same single file** `build/meta-pass_<version>.bin` that goes to the
  market. The page verifies the `MPUPV2` footer (integrity SHA-256), reads
  back the device partition table and byte-compares it (a layout mismatch
  refuses the upgrade), then writes bootloader / partition table / app from
  the file body and resets otadata to its erased state. NVS (stored data:
  Wi-Fi config, per-app state), `cardid`, and all three child-firmware slots
  stay untouched. Legacy `MPUPV1` upgrade containers (already distributed)
  remain accepted.
- **Command line** — equivalent esptool invocation (upgrade = four regions
  only; never flash the full merged image over an existing installation):

```bash
python -m esptool --port PORT write_flash 0x0 bootloader.bin 0x8000 partition-table.bin \
  0x10000 FoloToy-AI-Passport.bin 0x7fe000 ota_data_initial.bin
```

Never flash a release file raw at 0x0 over an existing installation to
"upgrade": esptool erases every sector it writes, and the file's ~1.1 MB
coverage includes the NVS region (carried as erased 0xFF), so Wi-Fi settings
and per-app stored data would be wiped. Child-firmware slots start at 0x180000,
beyond the file's coverage, and would survive — but upgrade through the
installer page above or the four-region command form instead. The 8 MB merged
image is a build/verification intermediate kept in `build/`, never distributed;
`tools/verify_firmware.py` enforces that it carries no content in any
user-data region.

### Full validation gates

```bash
./tools/validate.sh --static        # repo checks + host tests
./tools/validate.sh --firmware      # firmware build + protected-layout verification
                                    # (isolated /tmp build, artifact copied back to
                                    #  build/meta-pass_v<version>.bin)
node tools/install-slot/test-extract.mjs   # installer unpacking / name-blob tests
```

Firmware changes are host-test-first (TDD); image parsing, slot metadata and checksums
are pure-logic modules with no ESP-IDF dependency.

## Verification record

| Category | Result (2026-09-13; build, host-test and device rows updated 2026-09-24) |
| --- | --- |
| Build | Full `validate.sh` gate PASS; app 1,026,288 / 1,507,328 B (32% free); merged image 8 MB; `cardid` untouched; release single file `meta-pass_v1.0.0-5-g0abb320.bin` (1,091,868 B, MPUPV2 footer self-check passes) |
| Host tests | `meta_image`/`meta_slots`/`meta_import`/`meta_name` suites all pass; installer node tests 7/7; `test_meta_net_contract.py` pins JS↔C HTTP route consistency; `test_meta_net_upload.c` (21 cases) drives the full pair→upload→flash→verify→blob flow with real SHA-256, stubbing ESP-IDF — zero hardware required; **new** `test_display_wake_contract.py` (6 cases) pins the deep-sleep wake-recovery ordering and the otadata-resume wiring in the bootloader hook; `test_meta_boot_policy.c` now covers `must_resume` across every state |
| Device tests | Child idle deep-sleep + key wake returns to the child with a lit screen, and a repeat cycle still resumes (2026-09-24, on the board that reported BUG-05). Not covered: cold-reset rollback re-run, other children, other board revisions |
| Simulator (passport-sim) | 3-slot list with real names via dynamic blob offsets (ota_0→0x355000, ota_1→0x55f000); navigation; empty-slot OK no-op; boot ota_0; hard-reset rollback to launcher; ota_1 Passport Radar boot + rollback; IMPORT page (credentials/pair code/countdown) — *recorded 2026-09-13 and updated 2026-09-23; the row still names the pre-2026-09-23 interactions (detail metadata view, unsigned warning page, BOOT/CANCEL menu, on-device DELETE) that the one-click boot flow removed, and those need a re-run* |
| GitHub Actions | Static checks (Linux/GCC), firmware gate (ESPIDF Docker) — both green |
| CI artifact SHA-256 | `b86ca4fe…1b28e773` (canonical reference for marketplace publishing; local builds differ in embedded compile timestamp) |

## Relationship to the official firmware

This repository is based on
[FoloToy/ai-passport](https://github.com/FoloToy/ai-passport) (`f75873f`, MIT): the
`factory`/`cardid` layout, `verify_firmware.py` and other baseline contracts remain
byte-compatible; the stock demo pages were removed to make room for the launcher UI.
This is an unofficial project, not affiliated with FoloToy.

## FAQ

| Symptom | Fix |
| --- | --- |
| Serial picker is empty | Device is not in download mode (hold UP while powering on), or the USB cable is charge-only |
| Child firmware ignores buttons / no OK LONG return | Un-adapted firmware has no return hook; power-cycle to return (rollback). By design |
| Rebooting a child lands back in the launcher | By design (single-session model): every power-on returns to the launcher list page; crash recovery uses the same rollback |
| Child's screen stays black after its own idle-sleep wake, or the wake lands in the launcher | The child slept (e.g. 60 s idle), and the key press is a full bootloader start. Fixed in the current launcher: display init releases the child's pin holds and wakes the panel (no more black screen), and the bootloader hook renews the slot's otadata copy to VALID so the bootloader boots the child directly (no more landing on the list page). Update the launcher if you still see this |
| Slot shows "AI-Passport" instead of the play name | The firmware was installed without a display-name blob (merged image / old channel); reinstall via the USB page with a Display name |
| Image rejected | Over the slot's limit (`partition_size − 4KB`, the last 4KB sector is reserved for the name blob), or not an ESP32-C3 image |
