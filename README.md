# Firmware RE Toolkit — MStar MT5889 / TCL T615T (and TD98 Pro)

English-language reference for every tool, script and source used while
reverse-engineering the **TCL T615T firmwares (V083 / V098 / V474, SoC
MediaTek MT5889)** and the **Thundeal TD98 Pro / C50A projector** (same SoC
family, Android 11). Written 2026-10-06 from local evidence; every format
claim below was verified byte-exact unless marked `[reconstructed]`.

## Contents

| File | What |
|---|---|
| `TOOLS.md` | Full pipeline: acquisition → eMMC layout → filesystem extraction → DSP firmware → kernel modules → static analysis → crypto/forensics, incl. Windows specifics |
| `SCRIPTS.md` | Inventory of the 100+ session probe scripts (what each family does, where it lives, curation status) |
| `SOURCES.md` | Every third-party source (GitHub, XDA, 4PDA, Toengel, docs, blogs) with URL and what it was used for |

## Relationship to existing repos (no duplication)

This folder documents the **toolchain**. Content already lives elsewhere:

- Inventories + extract trees of V083/V098/V474 →
  [`tcl-t615t-firmware-analysis`](https://github.com/vladikit-bit/tcl-t615t-firmware-analysis)
  (Ukrainian; raw ~2.2 GB images stay local on `F:\` — GitHub 100 MB/file limit).
- Curated RE scripts, AEON tooling (patched `aeon_ghidra_public`, Reko
  projects), Ghidra runners, patch builders →
  [`projector-diagnostic-tools`](https://github.com/vladikit-bit/projector-diagnostic-tools)
  (`TOOLS.md`, `aeon-tooling/`). Referenced here, not duplicated.
- `Mstar-Analyzer` (different focus: STB/OTT, Lua, ECoS) — out of scope here.

## Scope note

"TCL firmware parsing" in practice covered two coupled targets: the TCL
T615T donor images (V083/V098/V474) and the TD98 Pro's own firmware
(DEC/SND DSP images, `utpa2k.ko`, `libmi3.so`). Both are MT5889, so one
pipeline serves both. Philips TPM211EA (`autorun.upg`) appears only as a
**worked negative example** (encrypted, §6 of TOOLS.md).

## One-line method

> Identify by magic → split by layout → extract each filesystem with the
> standard tool for its format → disassemble each ISA with its own
> toolchain → sweep with small scripts, never by eye → cite address or
> offset for every claim.
