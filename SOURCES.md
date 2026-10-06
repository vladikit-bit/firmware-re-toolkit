# SOURCES — every third-party source used, with what for

Verified 2026-10-06 (URLs as found; link rot is the reader's problem).

## MStar firmware pack/unpack

- **oso2dev/NPNLab-bin-tool** — https://github.com/oso2dev/NPNLab-bin-tool
  `unpack.py` / `pack.py` for MStar `.bin` firmware; `extract_keys.py`
  (AES + RSA public keys out of MBOOT); `secure_partition.py`
  (encrypt + sign). Reference toolchain for MStar containers (MULTIBIN,
  MBOOT, secure boot). Found via the JMGO/4PDA line of investigation.
- **XDA "[TOOLS] Mstar Android TV firmware tools"** —
  https://xdaforums.com/t/tools-mstar-android-tv-firmware-tools.3559227/
  `unpack.py`/`pack.py`/`extract_keys.py`/`secure_partition.py`,
  `MstarUpgrade` workflow, Project-ID notes. Same family as above;
  historical reference for MStar TV packs.

## Philips (reference + worked negative)

- **Toengel Philips blog** — https://toengel.net/philipsblog/
  Firmware download table + archive (`firmware-download/`,
  `firmware-archiv/`), TPM211EA posts with changelogs (incl. the
  "Fix for audio loss in HDMI (SPDIF)" line), downgrade/recovery guides
  (`upgrade_loader.pkg` vs `autorun.upg`), per-version feedback forum.
  Used for: TPM211EA_101.001.005.115 reference evaluation.
- **frederic/pflupg-tool** — https://github.com/frederic/pflupg-tool
  Unpacker for old Philips Fusion/MIPS `autorun.upg` (needs RSA public
  key in `pflupg.h`). NOT applicable to TPM211EA (OpenSSL-encrypted) —
  recorded so nobody retries it blindly.
- **XDA "Philips TV Autorun.upg how to decrypt"** —
  https://xdaforums.com/t/closed-philips-tv-autorun-upg-how-to-decrypt.4644043/
  Confirms TV-held key (`recovery-resource.dat` from a live TV).
- **mikrocontroller.net topic 312831** (German) — service-manual
  evidence (keys in NAND), CI+ licensing rationale, `upgApp` RE notes,
  `TPVision_MTK…_UPG_AES.key` path. Corroboration for "not crackable".
- **Philips support docs** (tv-sound-monitors.philips.com) — USB update
  procedure (`.zip` → single `autorun.upg` on stick).

## TCL / MediaTek platform

- **TCL Discussion Telegram + docs** — `faserf.github.io/TCL-Discussion-Telegram/`
  (`chipsets/`, `firmwares/`, `guides/`): platform↔SoC table
  (**T615T01 = MT9615/MT5889**; T653T01 = Pentonic — caught our
  near-miss of pulling the wrong SoC branch), naming
  `V8-<PLATFORM>-LF1V<NNN>`, `@tclupdates` / `@FirmwareTCLbot`
  distribution, file sizes (e.g. `V8-T615T01-LF1V082` Full OTA 1.92 GB).
- **XDA TCL 2023 Google TVs** (`.../t/tcl-2023-google-tvs-...4576005/`)
  and **avforums C845 thread** — V/M/R version semantics, rename rules,
  optical-audio bug history (V062→M999), file mirrors.
- **filewale.com** — mirror index for TCL TV firmware zips.
- **4PDA** — `showtopic=1106581` (JMGO O2 Ultra / Bonfire OS Mods /
  ADB guides / "Звук 5.1" ARC discussion — methodology reference +
  live counter-example of multichannel fragility); TCL threads for
  V8-T615T01 hunting.
- **MediaTek / spec aggregators** — mediatek.com (MT9679 page),
  gadgetversus.com (MT9615/MT9638/Pentonic specs),
  displayspecifications.com (model↔SoC mapping: TCL 55C845, Philips
  OLED707/807 + leaflets with DTS-HD(M6)/MS12 lines, Xiaomi specs).
- **JMGO / yuque** — `yuque.com/jmgo/fw/o2uxt` (official O2 Ultra
  firmware page, referenced from the 4PDA thread; unpackability
  untested at time of writing).

## Disassembly / RE tooling

- **Ghidra** (12.1.2 used here) + **public AEON slaspec**
  (`aeon_ORBIS32.sinc`, `i32_opcode = 0x3B`) — DSP disassembly; the
  3-fix headless recipe lives in `projector-diagnostic-tools`.
- **Reko** (`reko.exe disassemble --arch aeon --data <HEX>`) — the
  `0x28–0x2F` MAC group blank in the public slaspec.
- **Capstone + pyelftools** (`pip install capstone pyelftools`) —
  ARM32 `.ko`/AEON programmatic sweeps behind every table in this
  project's reports.
- **squashfs-tools** (LZO-enabled `unsquashfs`), **7-Zip**
  (https://www.7-zip.org), **simg2img** (AOSP),
  **`strings`**, `avbtool` — standard FS/image layer (see TOOLS.md §2).

## Reviews / behavior references (for the DTS question, not tooling)

- laurentwillen.com TD98 (measured, critical), promotop/schardtech
  TD98 Pro affiliates, thundeal.net official (SPDIF listed as a port),
  FlatpanelsHD Philips OLED807 (measurements), displayspecifications
  user review(s), OSMC/Nuvio/mpv/Fedora SPDIF-passthrough threads
  (app/OS-layer counterpoints — not our FW case).

## What was deliberately NOT used

- Blind `binwalk -e` on the Philips UPG (encrypted input — run the
  entropy scan first; §6 template).
- Same-SoC assumption without the chipset table (burned once on
  T653 vs T615 — now a checklist item).
