---
name: xiangshan-wave-signal-mapper
description: Use when debugging XiangShan waveforms and Codex needs to map Chisel/Scala signal names, bundle fields, probes, or module instances to exact Verilog/FST/GTKWave hierarchy names from .fst dumps, fstminer -n output, or cached facname lists.
---

# XiangShan Wave Signal Mapper

## Overview

Use this skill before waveform inspection when the user gives Chisel/Scala signal names but GTKWave needs generated Verilog/FST facility names. Prefer extracting real names from the `.fst` over guessing from source.

## Workflow

1. Identify the waveform input:
   - Use `scripts/fst_signal_filter.py --fst <wave.fst>` when an FST is available and `fstminer` exists.
   - Use `scripts/fst_signal_filter.py --facnames <facnames.txt>` when a cached `fstminer -n` output already exists.
2. Scope the search with generated instance fragments such as `frontend`, `inner_ifu`, `inner_ibuffer`, `core_with_l2`, or a full `TOP.SimTop...` prefix.
3. Pass Chisel names through `--signals`, or one name per line with `--signal-file`.
4. Report exact GTKWave signal names from the tool output. Mark absent Chisel internals as missing instead of inventing names.

## Command

```bash
python3 scripts/fst_signal_filter.py \
  --fst /path/to/wave.fst \
  --scope frontend \
  --signals inner_ifu.s1_specInstrCount,numValid,allowEnq
```

For faster repeated work:

```bash
fstminer -n /path/to/wave.fst > /tmp/facnames.txt
python3 scripts/fst_signal_filter.py --facnames /tmp/facnames.txt --scope inner_ibuffer --signals numValid,allowEnq
```

## Matching Rules

| Match | Meaning |
| --- | --- |
| `exact` | Query equals a full FST facility path without width suffix. |
| `exact-suffix` | Query matches the final hierarchy suffix, such as `inner_ifu.s1_valid`. |
| `probe-suffix` | Query maps to generated debug/probe name, such as `numValid -> numValid_probe`. |
| `alias` | Query maps through a known XiangShan/Chisel alias, such as `allowEnq -> io_in_ready`. |
| `keyword` | Query tokens appear in a facility name but need human confirmation. |
| `missing` | No FST facility matched; the Chisel value may be optimized away or renamed. |

## Common XiangShan Name Patterns

- Bundle fields flatten with underscores: `io.in.bits.prevInstrCount` often becomes `io_in_bits_prevInstrCount`.
- Vec elements flatten with numeric suffixes: `s1_invalidTaken(0)` can become `s1_invalidTaken_0`.
- Debug probes can add `_probe`: `numValid` may appear as `numValid_probe`.
- Internal Chisel `val`s may not exist in the FST. Use nearby handshakes or registered outputs as evidence.
- Module type names and instance names differ; GTKWave normally uses generated instance paths like `TOP.SimTop.cpu...frontend.inner_ifu`.

## Reporting

When answering the user, include:

- The FST or facnames source used.
- The scope filters used.
- A table of requested Chisel names to exact GTKWave names.
- A separate missing/alias note for optimized-away or equivalent signals.
