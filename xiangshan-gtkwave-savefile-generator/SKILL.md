---
name: xiangshan-gtkwave-savefile-generator
description: Use when creating or repairing GTKWave savefiles for XiangShan hardware debug waveforms, especially .gtkw files with grouped signals, markers, colors, exact FST signal names, AXI/TileLink/Chisel backtrace evidence, or broken GTKWave trace syntax.
---

# XiangShan GTKWave Savefile Generator

## Overview

Generate `.gtkw` files as debug artifacts, not decorative views. A good savefile must load every listed FST signal, preserve the dumpfile/time markers, group signals by causal backtrace stage, and make key evidence signals visually obvious without using invalid GTKWave syntax.

**REQUIRED SUB-SKILL:** Use `xiangshan-wave-signal-mapper` before adding signals whose exact FST names are uncertain.

**REQUIRED SUB-SKILL:** Use `hardware-debug-waveform` when the savefile is part of root-cause debugging from an assert, difftest mismatch, abort, or protocol violation.

## Workflow

1. Identify the waveform and existing savefile:
   - Prefer an existing `.gtkw` next to the `.fst`; preserve its `[dumpfile]`, `[timestart]`, `[size]`, marker line, and `[pattern_trace]` footer.
   - If no savefile exists, create a minimal one using absolute FST path and the target time window.
2. Build the signal list from the debug chain:
   - Group by causal stage: source candidate, pipeline/queue handoff, boundary propagation, final assert/mismatch side.
   - Include only proof signals: valid/ready/fire, id/tag, last/count/len, queue empty/full/pointers, source state, and final assert inputs.
3. Verify exact FST names before writing them:
   - Use `wave_db/signals.json`, `wave_meta/signals.json`, `fstminer -n`, or `xiangshan-wave-signal-mapper`.
   - Use the full GTKWave path exactly as emitted. Width suffixes can be part of the name, e.g. `auto_out_r_bits_id [6:0]`.
   - Do not invent Chisel-style names such as `auto_out_r_bits_id[6:0]` if the FST contains `auto_out_r_bits_id [6:0]`.
4. Write groups and markers:
   - Add markers for expected-good event, first bad combination, propagation point, and abort/report point.
   - Use standard GTKWave divider syntax for group headers and blank rows.
5. Validate by text inspection:
   - No unknown high-bit trace flags such as `@800022` or `@1000028`.
   - Every `[color] N` is directly before the intended signal.
   - `[pattern_trace] 1` and `[pattern_trace] 0` remain at the end.

## GTKWave Savefile Syntax

Use only stable GTKWave savefile constructs.

Group header:

```text
@200
-01 Source candidate
```

Visible blank divider row:

```text
@200
-
```

Signal format and color:

```text
@22
[color] 1
TOP.SimTop.cpu.l_soc.nocMisc.axi4yank.auto_out_r_bits_id [6:0]
```

Recommended formats:

| Signal kind | Format |
| --- | --- |
| scalar, 0/1 | `@28` or existing local scalar format |
| vector hex | `@22` |
| vector binary / existing binary view | `@23` |
| scalar with existing alternate view | `@29` |

Colors are standalone directives. Do **not** encode colors by modifying the trace flag:

```text
# Wrong: can make GTKWave load no signals
@800022
TOP.bad.signal

# Right
@22
[color] 1
TOP.good.signal
```

Use colors sparingly:

| Color | Suggested meaning |
| --- | --- |
| `1` | root/source candidate |
| `3` | queue or pipeline transfer |
| `4` | boundary/interface propagation |
| `5` | downstream transformed value |
| `7` | final assert or mismatch side |

## Signal Selection Pattern

For each debug hop, include the exact inputs that prove or disprove the invariant.

Example AXI4UserYanker assert:

```text
01 AXI4Memory source
  read_flap
  read_flap_tracker_id
  read_resp_id
  rPipe.io_enq_valid/id/last

02 Memory boundary propagation
  memory_rvalid
  memory_rready
  memory_rid
  memory_rlast

03 UserYanker assert side
  auto_out_r_valid
  auto_out_r_ready
  auto_out_r_bits_id
  auto_out_r_bits_last
  Queue64_BundleMap_<rid>.io_deq_valid
  Queue64_BundleMap_<rid>.empty
  Queue64_BundleMap_<rid>.do_deq
```

When a generated RTL expression uses a dynamic index, map it back before adding signals:

```scala
val rid = out.r.bits.id
val r_valid = VecInit(rqueues.map(_.deq.valid))(rid)
assert(!out.r.valid || r_valid)
```

This commonly emits RTL like:

```text
_GEN_3[auto_out_r_bits_id]
```

For `auto_out_r_bits_id = 4`, prove the selected signal from RTL concat/instance wiring before claiming it is `Queue64_BundleMap_4.io_deq_valid`.

## Marker Guidance

Markers should preserve both the failure scene and the causal backtrace:

| Marker | Meaning |
| --- | --- |
| expected-good | last known legal handshake or correct state update |
| first-bad | first waveform cycle proven to contain the bad trigger condition |
| propagated | cycle where the wrong value crosses an interface or queue |
| report/abort | simulator assertion, abort, or difftest failure scene |

Do not treat `first-bad` as root cause. Use it as the next backtrace hop and continue tracing upstream until the bug-triggering code, boundary scenario, and fix are identified.

## Common Mistakes

- Treating physical blank lines in `.gtkw` as visible rows. Use `@200` followed by `-`.
- Adding color by changing `@22` into `@800022`; use `[color] N` instead.
- Dropping spaces in FST vector names. GTKWave may distinguish `foo [6:0]` from `foo[6:0]`.
- Assuming `Queue64_BundleMap_4` from `id=4` without checking generated RTL when queues are emitted in multiple banks or duplicated modules.
- Mixing source Chisel names and generated FST names in the same savefile.
