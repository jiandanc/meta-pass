# meta-pass — Bug Report (current branch)

English | [简体中文](BUGS.zh_CN.md)

Code review of the current branch against `docs/assets/meta-pass-design.md`.
Each entry: symptom → root cause → fix. Severity: **High** = data loss / feature
broken, **Medium** = wrong behavior or latent defect in real flows, **Low** =
robustness / hygiene. Suspects that were investigated and found correct are
listed at the end, with evidence.

| ID | Severity | Component | One-liner | Status (fix applied on this branch) |
|----|----------|-----------|-----------|------------------------------------|
| BUG-01 | High | `main/main.c` | Battery label built from an uninitialized stack buffer; SOC never rendered | **FIXED** — guard + `snprintf` at `main.c:69-71` |
| BUG-02 | High | `main/meta_sign.c` | Inverted egg-magic check in the `HOST_TEST` `meta_egg_parse` variant | **FIXED** (upstream commit) — regression tests `m1`–`m4` added to `tests/test_meta_net_upload.c` |
| BUG-03 | High | `tools/install-slot/` (dev copy) | Diverged from shipped installer: out-of-partition name-blob write, double-write erases signatures, header-flag drift | **FIXED + structural fix done** — dev page files deleted; `server.mjs` now serves the canonical `install-slot/` directly |
| BUG-04 | Low | `main/meta_net.c` | `%d` used for `size_t` in the upload-success log | **FIXED** — `%zu` at `meta_net.c:407` |
| BUG-05 | High | `components/bsp/src/bsp_display.c`, `bootloader_components/meta_boot_hooks/hooks.c` | Child's idle deep sleep + key wake lands in the launcher (panel never re-initialized → backlight on, screen black; once lit, still the launcher list instead of the child) | **FIXED** — wake-recovery ported from the child BSP + the bootloader hook renews the child's otadata copy to `VALID` on a deep-sleep wake |

---

## BUG-05 — child idle-sleep wake: backlight on, screen black

**Symptom (on-device).** Boot slot 1 (a child firmware that dims at 30 s and deep-sleeps
at 60 s idle). Let it sleep, then press a key. The backlight lights up but the display
stays black. Flashing the child directly (no launcher) does not reproduce it.

**Root cause — two independent defects, both launcher-side.**

A wake from deep sleep is a full bootloader start. The child's BSP leaves two things
behind that the launcher never handled:

1. **The launcher is what boots, not the child.** The child never writes `VALID`, so its
   otadata copy sits at `PENDING_VERIFY`. With `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`
   the IDF bootloader marks that copy `ABORTED` before selection
   (`bootloader_utility.c:399-401`), `bootloader_common_ota_select_invalid()` rejects it
   (`bootloader_common_loader.c:76`), neither copy is a candidate, and it logs "Defaulting
   to factory image" (`bootloader_utility.c:412-415`) — i.e. the launcher. The hook's
   deep-sleep early return (`hooks.c:98`) skips the *policy* but cannot change this
   standard path. So the child exits on wake rather than resuming.
2. **The launcher's display init could not revive a slept panel.** `bsp_display_init()`
   sent SWRESET first (`bsp_display.c:118`) and SLPOUT only later via
   `esp_lcd_panel_init()`. The panel is still in Sleep In (oscillator stopped), where
   SWRESET is ineffective and can deadlock the command decoder; the child's own BSP
   documents this and pre-sends SLPOUT (`tianshang .../bsp_display.c:173-177`). The
   launcher's copy also never released the child's `gpio_hold_en()` /
   `gpio_deep_sleep_hold_en()` pins, which survive a reset and keep the pads latched at
   sleep levels — so every SPI command is swallowed while the backlight (a separate LEDC
   channel) comes up normally. Net effect: backlight on, screen black.

The child's BSP is simply newer: commit `8501cb2` hardened it upstream; the launcher's
copy predates it (`git log -S "prepare_deep_sleep"` is empty here). Confirmed at the
artifact level — the recovery strings occur once in the child image and zero times in the
shipped launcher image.

**Fix — first attempt (kept for the record, it failed on device).** Enable
`CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y` so the wake takes the fast-boot path
(boots the partition in RTC-retained memory, no otadata read, no ABORT). On device the
wake still landed in the launcher. Root cause, from the IDF sources: that option stores the
resume target in `rtc_retain_mem_t` at the top of RTC fast memory
(`SOC_RTC_DRAM_HIGH - 16`), which the link script reserves only when
`CONFIG_BOOTLOADER_RESERVE_RTC_MEM` is set. A child firmware's build does not set it — its
`RTC_TIMER_RESERVE_RTC` region covers exactly those bytes — so the child overwrites the
record the moment it runs, the CRC fails, `bootloader_load_image_no_verify` returns an
error, and `bootloader_utility_load_boot_image_from_deep_sleep` logs "Fast booting is not
successful" and falls through to the normal path. The option would need every child
firmware to opt in, which is the per-child cooperation the boot-policy rule forbids.

**Fix — final.** (1) The bootloader hook (`bootloader_after_init`, which runs at
`bootloader_start.c:39` — after flash is up, before `select_partition_number` marks
PENDING as ABORTED) branches on `esp_rom_get_reset_reason(0)`. On a deep-sleep wake it
renews the running child's `PENDING_VERIFY` otadata copy to `VALID`, so the bootloader
resumes that slot with no child-side configuration. The rewrite is safe because the CRC
covers only `ota_seq` (`bootloader_common_loader.c:69-72`), and it erases before writing
because flash programming is one-way and `0x1 -> 0x2` sets a bit. (2) Port the child's wake
recovery into the launcher BSP: release the global and per-pin holds before SPI takes the
pins, then send `0x11` SLPOUT + 120 ms before the panel reset. (2) covers the fallback path
and helps any deep-sleeping child.

**Verification.** `tests/test_display_wake_contract.py` (6 cases) pins the ordering and the
resume wiring; `tests/test_meta_boot_policy.c` covers `must_resume` across all states and
asserts the two rules are mutually exclusive. Built artifact checked: `bsp_display_init`
disassembles with `gpio_deep_sleep_hold_dis` → `gpio_config`/`gpio_hold_dis` →
`spi_bus_initialize` → `esp_lcd_new_panel_io_spi` → `esp_lcd_panel_io_tx_param(0x11)` →
`vTaskDelay` → `esp_lcd_panel_reset`; `bootloader_after_init` in the bootloader ELF calls
`esp_rom_get_reset_reason`, `bootloader_common_ota_select_crc`, `bootloader_flash_read`,
`bootloader_flash_erase_sector` and `bootloader_flash_write`, and the bootloader image
carries both policy strings ("resuming ota_%u … PENDING -> VALID" and "in VALID state ->
erasing"). The fast-boot string is gone. On device (2026-09-24): the child's idle deep
sleep followed by a key press returns to the child with a lit screen, and a second
sleep/wake cycle still resumes. Not re-run: cold-reset rollback to the launcher, other
children, other board revisions.

---

## PASS-RADAR "still unsigned" — root cause and resolution

Symptom reported on-device: the pass-radar firmware signed with the *fixed*
signing tool still shows the "Unsigned" state at boot (the launcher logged
`signature: unsigned`; at the time this also showed an on-screen "Unsigned
firmware!" warning, a page removed on 2026-09-23).

Investigation (all checks reproducible on this host):

1. **The signing chain itself is correct.**
   `tools/signing/run-verify-tests.sh` compiles the *real* firmware verifier
   (`main/meta_sign.c`, no `HOST_TEST`) against the embedded
   `meta_sign_pubkey.h` and reports `META_SIG_OK` on the newest signed image
   (`../pass-radar/build/pass-radar_v0.1-2-g8fcce59-signed.bin`, 19:09).
   The pre-fix image (`pass-radar_signed_v2.bin`, 13:53) correctly fails with
   `META_SIG_VERIFY_FAIL` — it was signed with the broken digest and must be
   re-signed. Launcher builds in `build/` (12:07 and 19:06) both embed the
   current public key, so launcher and signer agree.
2. **The real bug was the install path: BUG-03.** The README's primary local
   install workflow is `node tools/install-slot/server.mjs` → the dev copy of
   the installer. That copy (a) wrote the display-name blob with a second
   `writeFlash` into the same 4 KB tail sector that already held the MSIG
   signature, erasing it, and (b) computed the blob address from the
   partition size, flashing outside the slot. Installing through it destroys
   the signature sector → the launcher sees an erased (0xFF) tail → the slot is
   reported unsigned — even though the .bin file itself was signed correctly.

**Conclusion:** re-sign with the current `sign-firmware.sh`, then install via
the (now fixed) dev page or the Cloudflare Pages page — do **not** re-install
from an artifact previously written by the old dev page without re-writing
the tail sector. BUG-03's fix closes the on-device reproduction path.

**Re-sign verified (follow-up):** `pass-radar.bin` (962416 B) was confirmed
to be byte-identical prefix of the latest signed image, then re-signed fresh
with the current `tools/signing/sign-firmware.sh` (image_len 962416,
sig_offset 962560, payload_len 70): the firmware verifier reports
`META_SIG_OK` on both the fresh artifact and the existing
`pass-radar_v0.1-2-g8fcce59-signed.bin`. The old `pass-radar_signed_v2.bin`
remains invalid by design and must not be installed.

**Local flasher verified end-to-end (follow-up 2):** the local install
service (`node tools/install-slot/server.mjs`, canonical `install-slot/` page)
was smoke-tested (page, ES modules, vendor bundles all 200; SSRF guard and
404 behave), and the full install byte-path was replayed against the real
signature: `extractAppImage()` on the signed image yields image_len 962416 /
tail offset 0xEB000 / full 4 KB tail sector; after applying the page's MNAM
display-name patch (right-aligned at 4056), the assembled slot bytes pass
the *firmware* verifier (`main/meta_sign.c` + real mbedtls): `META_SIG_OK`,
`meta_sign_detect_sector() == true`, egg text intact. The installer's
`tailSectorOffset` (4K-aligned after image_len) matches the device-side
`meta_sign_sector_offset()` and the design doc §7 layout exactly
(MSIG [0..127] / MAEG [128..4055] / MNAM [4056..4095], single-write sector).

Verification commands:

```bash
tools/signing/run-verify-tests.sh ../pass-radar/build/pass-radar_v0.1-2-g8fcce59-signed.bin
node tools/install-slot/test-extract.mjs   # installer unit tests (canonical modules)
bash tools/validate.sh --static            # static checks + host tests, all PASS
PORT=4191 node tools/install-slot/server.mjs  # local flasher; open http://localhost:4191/
```

---

## BUG-01 (High) — `add_battery()` renders a garbage battery label

**File:** `main/main.c:64-71`
**Status: FIXED** — the exact fix below is applied (`soc < 0` guard at
`main.c:69`); code block kept as the historical defect.

```c
static void add_battery(lv_obj_t *parent)
{
    const int soc = bsp_battery_soc();
    char text[12];
    lv_obj_t *lbl = ui_pixel_label(parent, text, &lv_font_montserrat_14, UI_PAPER);
    lv_obj_set_pos(lbl, 204, 30);
}
```

**Symptom.** A label is created from `text`, a **completely uninitialized stack
buffer**. `ui_pixel_label()` immediately calls `lv_label_set_text(label, text)`
(`ui_pixel.c:18-26`), which `strlen()`s the buffer: it draws whatever bytes
happen to be on the stack, and can even read past the 12-byte array when no
NUL is present. The comment promises "when reading −1 (unavailable), don't
draw, to avoid showing a fake number" — and `bsp_battery_soc()` does return −1
on failure (`bsp_battery.h:13`) — but the value is never used. `soc` is dead,
which also means the firmware build carries an unused-variable warning nobody
acted on (the host test suite in `tools/validate.sh` never compiles `main.c`,
so `-Werror` never sees it).

**Root cause.** Half-finished feature: the SOC → text formatting and the
"don't draw when unavailable" guard were never written. Affected on both call
sites — the list screen (`main.c:143`) and the detail screen (`main.c:308`,
removed 2026-09-23) —
i.e. the two most-used pages show garbage in the top-right corner.

**Fix.**

```c
static void add_battery(lv_obj_t *parent)
{
    const int soc = bsp_battery_soc();
    if (soc < 0) return;                       // no gauge → don't draw
    char text[12];
    snprintf(text, sizeof(text), "%d%%", soc);
    lv_obj_t *lbl = ui_pixel_label(parent, text, &lv_font_montserrat_14, UI_PAPER);
    lv_obj_set_pos(lbl, 204, 30);
}
```

(Ensure `<stdio.h>` is included for `snprintf`.)

---

## BUG-02 (High) — Inverted egg-magic check in the `HOST_TEST` parser

**File:** `main/meta_sign.c:47` (`#ifdef HOST_TEST` variant of `meta_egg_parse`)
**Status: FIXED** — the stray `!` was removed upstream; the line now reads
`if (memcmp(egg, egg_magic, 4) != 0) return META_EGG_ABSENT;` and matches the
device variant and host stub. Code block kept as the historical defect.
**Regression guard added:** `tests/test_meta_net_upload.c` now exercises the
egg path — `m1_egg_parse_valid` (valid MAEG → `META_EGG_OK`),
`m2_egg_parse_absent` (erased sector → `META_EGG_ABSENT`),
`m3_upload_signed_tail_preserved` (signed tail with MAEG survives an upload,
MSIG + egg intact, dispname rejected) and `m4_upload_unsigned_tail_rebuilt`
(unsigned upload rebuilds MNAM, no MAEG residue). The old inverted variant
would fail m1/m3 immediately.

```c
static const unsigned char egg_magic[4] = META_EGG_MAGIC_BYTES;
if (!memcmp(egg, egg_magic, 4)) return META_EGG_ABSENT;
```

**Symptom.** The semantics are inverted relative to every other
implementation of the same function:

- device variant, `main/meta_sign.c:97`: `if (!check_egg_magic(egg)) return META_EGG_ABSENT;` (absent = magic *missing*)
- host stub, `tests/esp_stubs/meta_sign_stub.c:58`: `if (memcmp(egg, egg_magic, 4) != 0) return META_EGG_ABSENT;`
- format tests, `tests/test_meta_sign.c:158`: "erased sector (no MAEG) → `META_EGG_ABSENT`"

The HOST_TEST variant returns `META_EGG_ABSENT` when the magic **matches**,
i.e. a valid egg is reported absent, and a blank (0xFF) sector is treated as
"egg present" and pushed into the parse path.

**Root cause.** A stray `!`. Note this is not dead code:
`tools/validate.sh` compiles `tests/test_meta_net_upload.c` from the *real*
`main/meta_sign.c` with `-DHOST_TEST` (validate.sh:73-78), so the variant is
built on every CI run — the upload test just doesn't exercise the egg path
yet, which is why nothing caught it.

**Fix.** Make it match the other two implementations:

```c
if (memcmp(egg, egg_magic, 4) != 0) return META_EGG_ABSENT;
```

Better: delete the HOST_TEST block from `main/meta_sign.c` and have
`test_meta_net_upload` link `tests/esp_stubs/meta_sign_stub.c` for
`meta_egg_parse`/`meta_sign_verify` (as `test_meta_sign` already does), so
there is only one implementation left to keep correct.

---

## BUG-03 (High) — Dev installer copy has drifted into correctness bugs

**Files:** `tools/install-slot/install-slot.html`, `tools/install-slot/extract-app-image.js`
vs. the shipped `install-slot/` copies.
**Status: FIXED** — the shipped single-write tail-sector flow and the
`& 1` bit-test were ported into the dev copy; `diff` reported the copies
identical (modulo the one-line path annotation in `name-blob.js`).
**Structural fix also done (dedupe):** the duplicated dev page files were
deleted; `tools/install-slot/` now contains only `server.mjs` and
`test-extract.mjs`, and `server.mjs` serves the canonical `install-slot/`
directory directly (`PAGE_DIR = ../../install-slot`), i.e. the exact bytes
deployed to Cloudflare Pages. `test-extract.mjs` imports the canonical
modules and reads the canonical HTML for its i18n test. `README.md`,
`README.zh_CN.md` and `install-slot/README(.zh_CN).md` layout sections were
updated accordingly. Verified: `node tools/install-slot/test-extract.mjs`
→ 8/8 PASS; live smoke of `server.mjs` (`/`, `/name-blob.js`,
`/extract-app-image.js`, `/vendor/esptool-js.js` → 200).

The repo keeps two installer copies (README:152-153): `install-slot/` is the
deployed Cloudflare Pages page; `tools/install-slot/` is the local dev copy
served by `server.mjs:91-94`. Nothing checks that they agree, and the dev copy
has fallen behind in three behavior-affecting ways.

### (a) Name blob written outside the slot partition (corrupts the next partition)

Dev copy, `tools/install-slot/install-slot.html:133,561-563`:

```js
import { packNameBlob, sanitizeDisplayName, blobOffset, maxAppImageSize } from "./name-blob.js";
...
const blob = packNameBlob(dispName);
const blobAddr = address + blobOffset(s.size);
```

`blobOffset(len)` = `ceil(len/4096)*4096 + 4056` (`name-blob.js:23-29`) — it
expects the **image length**, but the dev page passes the **partition size**
`s.size`. Since `s.size` is 4K-aligned, this lands the blob at

```
slot_offset + part_size + 4056
```

which is 4056 bytes *past the end of the slot*. Concretely for slot 0
(`ota_0`, 0x180000 + 0x1D6000 = 0x356000 end): the blob is flashed at
**0x356FD8 — inside the `cardid` NVS partition** (0x356000, 0x4000), and for
the other slots it lands at the start of the next app partition. Even under
the old pre-tail-sector layout this is off by a full sector (an "end of
partition" blob would be `slot_offset + part_size − 40`). Result: the display
name silently doesn't work *and the installer corrupts an unrelated partition*
(the device's card ID, or the neighboring slot's first sectors).

The shipped page has the correct, image-length-based single-write flow
(`install-slot/install-slot.html:564-584`, `packNameBlobTail` merged into the
4 KB `image.tailSector` written at `image.tailSectorOffset`).

### (b) Double-write erases the signature/egg sector

The dev page writes the app image first, then issues a **second**
`writeFlash` for the blob (dev page:552-569). The esptool flash write erases
each destination sector before programming, so the second call — targeting
the same 4 KB tail sector the first call already touched — wipes whatever the
first write put there. For a signed image the MNAM window overlaps the MSIG
sector, so **installing via the dev page erases the signature**; the launcher
then shows "Unsigned firmware!" for a trusted build. This is exactly what the
shipped page's comment calls out ("split writes will repeatedly erase the same
sector, wiping the signature/easter egg written first",
`install-slot/install-slot.html:562-563`) — the fix never made it back into
the dev copy.

### (c) `hashAppended` flag detection drift

`tools/install-slot/extract-app-image.js:29`:

```js
const hashAppended = buf[start + 23] === 1;              // dev copy
const hashAppended = (buf[start + 23] & 1) === 1;        // shipped copy, line 29
```

Byte 23 of the ESP image header is a flags byte (bit0 = hash_appended). Today
ESP-IDF writes exactly 0 or 1 so both behave identically, but the dev copy's
equality test miscomputes the image length (32 B short) for any image with
another bit set in that byte, truncating the app image written to flash.
The dev copy should adopt the shipped bit-test.

**Root cause.** Copy duplication without a sync mechanism: `tools/check_repo.py`
doesn't compare the copies, `tools/install-slot/test-extract.mjs` tests the dev
`extract-app-image.js` but not against the shipped one, and the dev HTML simply
wasn't updated when the tail-sector write landed.

**Fix.**
1. Short term: port the shipped page's write flow into
   `tools/install-slot/install-slot.html` and the bit-test into its
   `extract-app-image.js`.
2. Structural: make `tools/install-slot/server.mjs` serve the canonical
   `install-slot/` directory (single source of truth), or add a drift check to
   `tools/check_repo.py` / CI asserting the copies are identical modulo the
   one-line header comments.

---

## BUG-04 (Low) — `%d` for a `size_t` value in the upload log

**File:** `main/meta_net.c:407`
**Status: FIXED** — now `%zu` with the `size_t` argument directly.

```c
ESP_LOGI(TAG, "slot %d written: %s %s (%d B)", slot, name, ver, req->content_len);
```

`req->content_len` is `size_t` (ESP-IDF `httpd_req_t`; the project's own stub
types it so at `tests/esp_stubs/esp_http_server.h:32`). Harmless on the
32-bit ESP target, but a latent `-Wformat` warning on other hosts and
inconsistent with the `%u`-style handling elsewhere. Use
`(unsigned)req->content_len` with `%u`. Cosmetic.

---

## Investigated and cleared (evidence)

- **LVGL 9.5 timer self-delete in UI teardown paths** — `lv_timer.c` guards
  callbacks with `act_timer_deleted`; deleting one's own timer from its
  callback is supported. Not an issue.
- **Programmatic scrolling of the egg panel** — `block()` strips
  `LV_OBJ_FLAG_SCROLLABLE` from its own children, but the egg panel
  (`main.c:189-190`) re-enables scrolling via `lv_obj_set_scroll_dir()`;
  `lv_obj_scroll_by_raw()` doesn't check the flag at all. Scroll-by direction
  semantics match the LVGL 9.5 header contract (`dy > 0` moves toward
  top/beginning), so `btn == BSP_BTN_UP ? +step : −step` (`main.c:440`) is the
  correct mapping.
- **Size-limit arithmetic** — for the 4K-aligned partition sizes in
  `partitions.csv` (0x1D6000 / 0x200000 / 0x29E000) the bound
  `part_size − 4096` used by `meta_sign_app_limit()` (`meta_sign.h:56`),
  `meta_name_max_app_size()` (`meta_name.h:42`) and JS `maxAppImageSize()`
  (`name-blob.js:33`) is *exactly* tight: for any accepted `image_len`,
  `ceil(image_len/4096)*4096 ≤ part_size − 4096`, so the tail metadata sector
  always fits. No off-by-one.
- **Button long-press wiring** — `bsp_button.c:72-76` registers
  `BUTTON_LONG_PRESS_START` with `press_time = BSP_BTN_LONG_MS (1500)` in a
  stack-local `button_event_args_t`; `iot_button_register_cb` copies
  `press_time` into its internal cb_info array before returning
  (`iot_button.c:378,402-411`), so the stack lifetime is fine. The zeroed
  `button_config_t` makes `TIME_TO_TICKS(0, LONG_TICKS)` fall back to
  `CONFIG_BUTTON_LONG_PRESS_TIME_MS` (Kconfig default 1500 ms), consistent
  with the explicit args.
- **`meta_seq` matcher** — traced all paths (gap timeout at index > 0, key
  mismatch with restart, repeated first key, wraparound via unsigned
  subtraction): behavior matches the design doc.
- **`meta_name_pack_tail`/`unpack_tail` ↔ JS `packNameBlobTail`** — right-
  aligned 40 B window, xor last byte; C and JS agree byte-for-byte.
- **`meta_egg_parse` window bounds** — compile-time checked
  (`META_EGG_WINDOW_OFF + META_EGG_TOTAL_LEN ≤ META_NAME_BLOB_OFF` =
  128 + 3928 ≤ 4056) and match `sign-firmware.sh`'s layout (MSIG@0, MAEG@128,
  MNAM@4056).
- **`image_len` sourcing** — upload path uses `esp_image_verify` metadata,
  correctly avoiding the trap of reading the header's non-length field at
  +20 (`meta_net.c:314-318`).
- **Shipped installer tail-sector extraction** — extracts MSIG/MAEG/MNAM into
  one `image.tailSector` and writes it in a single `writeFlash`; correct.

---

*Report generated from static review + host-test source verification. Line
numbers refer to the current branch. Companion file:
[BUGS.zh_CN.md](BUGS.zh_CN.md).*
