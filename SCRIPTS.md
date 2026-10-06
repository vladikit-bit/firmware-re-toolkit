# SCRIPTS — session probe inventory (what, where, status)

~100+ single-purpose Python probes in
`C:\Users\k0994\AppData\Local\Temp\opencode\` (prefix = task family).
They are **raw work artifacts, not a library**: many overlap, early ones
were superseded. The curated subset lives in
[`projector-diagnostic-tools`](https://github.com/vladikit-bit/projector-diagnostic-tools)
— consult that repo first; what follows is the map of the raw pool so a
future analyst can find (not re-derive) any sweep.

Conventions shared by all probes: parse the 6-hex-address listing format
with `^([0-9a-f]{6}) ([0-9a-f]) (.*)$`; exact-match `jal 0x…` (8-digit,
lowercase); dual mnemonic patterns (offset-first `bn.sw` vs reg-first
`bg.lwz`/`bg.sw_0`); suffix-tolerant store regex
`s[wb](_\d+)?` + separate `sbz`/`sh` pass; branch-into checks per entry.

## Families

| Prefix | Task | Representative members |
|---|---|---|
| `q_*` | DEC dispatch / entry gate / bitmap / type-tag / ring-remaining (`F_643DC`, fork, size gate, `F_64149` copy, jump tables) | `q_fork.py` (0x64280 window + callers), `q_final2.py` (F_410A3/F_45FD1 windows), `q_last.py` (F_64149 body), `q_jtab*.py` (remaining jump-table LE decode), `q_callers.py`, `q_entry_gate*.py`, `q_bm*.py` (bitmap), `q_38e5e.py`, `q_0c.py`, `q_38e5e.py` |
| `u_*` | `utpa2k.ko` host side (`Set/Get_SHM_PARAM` tables+handlers, reg-bank, DSP-SRAM backends, mdb strings, dump_es tap hunt, reloc parsing) | `u_shm*.py` (Get handlers), `u_q3*.py`, `u_regbank.py`, `u_mdb.py`, `u_dump*.py`, `u_snd*.py` / `u_dec*.py` (SND/DEC Set tables), `u_scan.py` (full-`.text` operand scan) |
| `s_*` | SND image (`snd33_realcode` struct, zeroer, readers, base hunt, SND callers) | `s_recon.py`, `s_q1..s_q6.py` |
| `d_*` | DEC image (`dec33_realcode` clusters, formula writers, latch chain, arms, burst) | `d_q2..d_q7.py`, `d_entry_arms.py`, `d_sweep.py`, `d_setup.py`, `d_heads.py`, `d_arm*.py`, `d_gaps.py`, `d_final.py`, `d_micro*.py` |
| `e_*` | ES→SPDIF dead-end (packer consts, `0x0B/0x77` discriminators, classifier, handoff, TX-arm, SND whitelist) | `e_recon.py`, `e_pack.py`, `e_disc.py`, `e_handoff.py`, `e_chain.py`, `e_final.py`, `e_410a3.py` |
| `b_*` | B-latch object (pump head, return map, `-0x5EF8` writers, `F_37D18`) | `b_q.py`, `b_q2.py`, `b_q3.py` |
| `t1_*` | TV/TPM + IP-auth line (enums, tables, SPDIF calls, relocs, syscalls) | `t1_enum*.py`, `t1_base.py`, `t1_arm.py`, `t1_spdif*.py` |
| `t3_*`, `q3_*` | Section/table dumps and disasm helpers | `t3_plt*.py`, `q3_dis.py`, `q3_sec.py` |
| `philips_*` | Philips `autorun.upg` forensics (entropy scan, EVP/PBKDF2 password sweeps) | `philips_scan.py`, `philips_dec.py`, `philips_dec2.py`, `philips_dec3.py` |
| `v083_*` | V083 full-walk inventories | `v083_vfull.py`, `v083_fullwalk.py`, `v083_final.py` |

## Reusable one-pattern appendix (copy-paste)

```python
import re
P = r"C:\firmware_temp\aeon_validate\dec_work\dec33_realcode.txt"
rows = []
with open(P, encoding="utf-8", errors="replace") as f:
    for line in f:
        m = re.match(r"^([0-9a-f]{6}) ([0-9a-f]) (.*)$", line.rstrip("\n"))
        if m:
            rows.append((int(m.group(1), 16), m.group(3)))
# exact caller list (8-digit jal target):
def callers(tgt, txt):
    return re.findall(r"^([0-9a-f]{6}) [0-9a-f] .*jal 0x%s" % tgt, txt, re.M)
# suffix-tolerant word stores at offset OFF (any base reg):
def writers(rows, off):
    return [(a, t) for a, t in rows
            if re.search(r"\bsw(_\d+)?\s+%s\(r\d+\)" % off, t)]
# sub-word stores need their own pass: \bsbz?\b, \bsh(_\d+)?\b (reg-first
# bg-forms need r\d+,OFF tweezer: (lwz|sw_0)\s+r\d+,(-?)OFF\(r\d+\))
```

## Curation status

- Raw pool: local only (`Temp\opencode`), includes superseded iterations.
- Curated: `projector-diagnostic-tools` (recon, SRAM sweeps, Ghidra
  runners, static analyzers, patch builders, AEON tooling, SKILL.md).
- This folder documents; it does not re-bundle. Promote a probe to the
  curated repo deliberately, not by copy-all.
