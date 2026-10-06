# TOOLS — full pipeline (verified facts + reference toolchain)

Legend: **[verified]** = checked byte-exact during this project.
**[reconstructed]** = standard practice, exact historical command not
recoverable from traces. Default working facts first, theory second.

## 0. Acquisition

- TCL T615T images (V083/V098/V474): eMMC dumps + OTA zips from the
  TCL community pipeline — Telegram `@tclupdates` / `@FirmwareTCLbot`
  (query platform string, e.g. `T615T01`), XDA 2023 Google-TV thread,
  file mirrors (Google Drive / Yandex). Firmware naming
  `V8-<PLATFORM>-LF1V<NNN>`; platform must match SoC
  (**T615T01 = MT9615/MT5889**; T653T01 = Pentonic 700 — different SoC,
  verified via the TCL chipset table after nearly pulling the wrong
  branch). **[verified]**
- Philips TPM211EA: Toengel blog posts / Philips support pages per model
  (e.g. `55OLED807/12` → Software updates). Ships as `.zip` → single
  `autorun.upg` (`upgrade_loader` family). **[verified]**

## 1. eMMC layout (TCL V083 — verified magic of all 22 partitions)

`F:\v083\uni_out\` (sizes + first bytes verified 2026-10-06):

| File | Size | Magic | Format |
|---|---|---|---|
| `boot_a.bin`, `recovery_a.bin` | 48 MB | `ANDROID!` | Android bootimg |
| `linux_rootfs_a_a.bin` | 89 MB | `hsqs` | SquashFS 4.0 (LZO per repo README) |
| `3rd_a_a.bin` | 120 MB | `hsqs` | SquashFS |
| `tvservice_a.bin` | 160 MB | zeros + `53 EF`@0x438 | ext4 |
| `tclconfig_a.bin` | 120 MB | zeros + `53 EF`@0x438 | ext4 (panel/config ini, 3840×2160) |
| `tvconfig.bin` | 32 MB | zeros + `53 EF`@0x438 | ext4 |
| `cache.bin`, `metadata.bin`, `oem_a.bin` | — | zeros + `53 EF`@0x438 | ext4 |
| `MBOOT.bin`, `MBOOT_b.bin`, `MBOOTBAK.bin` | 3.2 MB | `00 24 00 00 …` | MStar bootloader (custom) |
| `armfw.bin`, `optee.bin` | ~0.1/5 MB | `7F 'MULTIBIN'` | MStar MULTIBIN container |
| `dtbo_a.bin` | 8 MB | `D7 B7 AB 1E` | Android DTBO |
| `vbmeta_a.bin`, `vbmeta_system_a.bin` | 12/4 KB | `AVB0` | Android Verified Boot metadata |
| `frc_a.bin`, `RTPM_a.bin`, `_mmc_boot.bin` | small | proprietary | MStar-specific, opaque |
| `upgrade.bin` | 0 B | — | placeholder |

(`53 EF` little-endian at offset `0x438` = ext4 superblock magic —
decisive test for "is this ext4".)

## 2. Filesystem extraction (reference toolchain)

| Format | Tool | Source |
|---|---|---|
| SquashFS (`hsqs`, incl. LZO) | `unsquashfs` (squashfs-tools ≥ 4.4, LZO enabled) or 7-Zip (open archive) | distro packages / https://github.com/plougher/squashfs-tools ; 7-Zip https://www.7-zip.org |
| ext4 (plain images) | 7-Zip (open), Linux `mount -o loop,ro`, `ext4extract`/`ext2read` on Windows | same as above |
| sparse ext4 (`simg`) | `simg2img` first, then as ext4 | AOSP `system/core` (`simg2img`), or the XDA MStar firmware tools thread |
| Android bootimg (`ANDROID!`) | `unmkbootimg` / AOSP `mkbootimg` tools, or `payload` inspection via `file` + `strings` | AOSP `system/tools/mkbootimg`; https://xdaforums.com/t/tools-mstar-android-tv-firmware-tools.3559227/ |
| MStar MULTIBIN (`MULTIBIN`) | MStar-specific splitter (see NPNLab below); else carve by inner magics | https://github.com/oso2dev/NPNLab-bin-tool |
| MStar MBOOT / secure boot | `extract_keys.py` (AES + RSA public keys out of MBOOT) | NPNLab-bin-tool (same repo) |
| MStar `.bin` firmware packs | `unpack.py` / `pack.py` | NPNLab-bin-tool; XDA Mstar tools thread (same) |
| Philips `autorun.upg` (old Fusion/MIPS only) | `pflupg` (+ RSA public key in `pflupg.h`) | https://github.com/frederic/pflupg-tool |
| Philips `autorun.upg` (TPM211EA, 2022+) | **not unpackable** — OpenSSL `Salted__` full-file encryption, key held in TV NAND (CI+ requirement). Proven: 1.3 GB uniform entropy (78×16 MB chunks), ~340 password×KDF/cipher trials, 0 hits | this project (see §6); corroborated by XDA `autorun.upg` decrypt thread + German µC.net analysis |

Platform ID shortcut (used on the TCL images): `strings` for
`androidboot.boot_devices=mstar_mci.0`, `buildvariant=user`,
WiFi/BT `MT7961`+`MT7663`/`MT7668` blobs. **[verified]**

## 3. DSP firmware (MStar AEON R2: DEC + SND)

- Images: `dec_full.bin` (**file offset == fw address**, md5 `4b7e9509…`),
  `snd_full.bin` (**file offset = fw address − `0x16F00`**). **[verified]**
- ISA: MStar AEON (DSP-flavoured). Disassemble with **Ghidra 12.1.2 +
  public AEON slaspec** (`aeon_ORBIS32.sinc`) + the 3-fix headless recipe
  (see `projector-diagnostic-tools`, `aeon-tooling/` — not duplicated
  here). The `0x28–0x2F` MAC group is blank in the public slaspec —
  decode slices with `reko.exe disassemble --arch aeon --data <HEX>`
  (`--entry`/`--base` rejected on raw images; feed hex slices).
  **[verified]**
- Listing pipeline that produced `dec33_realcode.txt` (504 216 insns) /
  `snd33_realcode.txt` (253 507): sweep every offset, keep only
  reachable code (reachability filter removes ~14% data-as-code).
  Line format: `<6-hex-addr> <len> <mnemonic…>`; `jal` targets 8 hex
  digits lowercase; some mnemonics carry suffix + trailing comma
  (`bg.sw_0 0xa0(r1),r30,`); DEC offsets lowercase (`0xc`).
  **[verified]**
- AEON assembler was unavailable, so the encoder was re-derived and
  machine-checked: `enc(rD,rA,imm,sub) =
  (7<<29)|(0x3B<<26)|(rD<<21)|(rA<<16)|(imm & ~3)|sub`, `sub` 0 = `sw_0`,
  2 = `lwz`, from `aeon_ORBIS32.sinc` `i32_opcode = 0x3B`; 0 mismatches
  over 24 859 DEC instructions. **[verified]**

## 4. Kernel modules (ARM32, relocatable)

- `utpa2k.ko` (25 813 336 B): ARM32 relocatable, `.text` file-off
  `0xd654`, vaddr 0. Disassemble `.text` with Capstone
  (`CS_ARCH_ARM`, `CS_MODE_ARM`); resolve `bl` via `.rel.text`
  (`r_offset`/`r_info`, symtab section indices); read jump tables
  (`ldr pc,[rX,rY,lsl#2]`) + handler bodies for SHM offsets; full-`.text`
  operand scans for offset-reachability proofs. **[verified]**
- Python stack: `capstone` + `pyelftools` (both `pip install`).
  **[verified]**

## 5. Sweep discipline (the method that produced the findings)

- Never trust one regex: the listings needed 7 format traps
  (8-digit `jal`, suffix+comma mnemonics, lowercase offsets, `jr r9`
  fragment ends, `bn.sw` offset-first vs `bg.lwz` reg-first,
  `sbz`/`sh` sub-word stores, `movhi` shift semantics).
- Caller lists: exact-match `jal 0x…` greps; branch-into verified per
  entry (`jr`-fragment discipline — `0x190F0` false-prologue lesson).
- Writer attribution only with pointer provenance fixed (base-register
  flow), else 600 false hits on common offsets (`+0x10`, `+0x28`).
- Claim vocabulary: `VERIFIED | STRONGLY SUPPORTED | INFERRED |
  UNPROVEN | DISPROVEN | residual | clean negative`.

## 6. Worked crypto negative (Philips TPM211EA — template for "prove it is shut")

File: `65oled706_12_fus_aen.zip` (1.21 GB) → one stored (uncompressed)
`autorun.upg` (1 297 636 698 B): 16-byte `Salted__` header + salt, then
ciphertext. Proof of full encryption: 78×16 MB chunk entropy scan
(ent = 8.00 everywhere, zero ASCII runs). Password sweep harness
(Python `cryptography`: AES-256/128-CBC × EVP-MD5/`PBKDF2-SHA256`,
structure scoring on 64 KB): ~22 obvious + ~279 extended + 38
FWID-derived candidates — 0 hits. Conclusion: TV-held key (service
manual: keys in NAND; CI+ licensing); XDA confirms
`recovery-resource.dat`-from-TV requirement. Do not attempt further
static recovery; pivot to unpackable same-SoC donors (TCL T615T).

## 7. Windows environment specifics (all bitten, all verified)

- Shell is PowerShell 5.1: chain with `cmd1; if ($?) { cmd2 }`
  (`&&` unsupported); quote spaced paths; `& "exe path"` for binaries
  with spaces.
- Python heredocs FAIL — always write `.py` files first (used
  `C:\Users\k0994\AppData\Local\Temp\opencode\`), run with
  `$env:PYTHONIOENCODING='utf-8'`.
- Full-decode sweeps time out at 300 s — use block regions, not whole
  637 K-line files per query.
- Free space: C: is small (~2 GB); stage multi-GB work on F:
  (12.5 GB free at time of writing). Raw images stay local (GitHub
  100 MB/file limit) — see the `tcl-t615t-firmware-analysis` README.
