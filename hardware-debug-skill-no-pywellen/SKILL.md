---
name: hardware-debug-waveform
description: Use when analyzing a hardware failure from a waveform dump (`.vcd` or `.fst`), XiangShan simulator logs, or when the user says "xs ci run wave" and provides an XS CI result path. Handles difftest/assert triage, waveform-to-RTL ownership lookup, and CI failed-checkpoint reruns.
---

# Hardware Debug Waveform

## Overview

Use this skill to debug hardware failures from large waveform dumps (`.vcd` or `.fst`) with a Scala/Chisel source tree and, when available, emitted RTL.

Core approach:

- use the waveform to identify the failure pattern
- use emitted RTL to recover exact ownership and hierarchy
- use Scala/Chisel source as the primary material for root-cause analysis
- use generated SystemVerilog only as a fallback

**REQUIRED SUB-SKILL:** Use `xiangshan-wave-signal-mapper` when the task needs exact Chisel/Scala-to-FST/GTKWave signal hierarchy mapping, facname extraction from `.fst`, or a waveform search checklist from generated signal names.

**REQUIRED SUB-SKILL:** Use `xiangshan-debug-report-writer` when the user asks for a written debug report, verification document, Chinese analysis report, or a reusable record of root-cause backtrace, waveform evidence, expected vs observed values, signal meaning, propagation relation, and confidence.

**REQUIRED SUB-SKILL:** Use `xiangshan-gtkwave-savefile-generator` when the user asks for a GTKWave `.gtkw` view, grouped waveform signals, color-highlighted proof signals, marker placement, or repair of a savefile that opens with markers but no signals.

## Workflow

### Fast Path - `xs ci run wave`

If the user says `xs ci run wave`, `xs ci result`, asks to rerun failed XS CI checkpoints, or asks to create wave/debug runs from an XS CI result directory, use this flow before the waveform-analysis flow.

Required input:

- XS CI result directory path. It must contain `score.txt`, `emu`, `riscv64-nemu-interpreter-so`, and per-checkpoint result directories.

If the path is missing, ask only for:

```text
Please provide the XS CI result path, for example:
/nfs/home/share/.../timing-fix-report/<run-name>
```

#### 1. Inspect the result directory

From the result directory:

```bash
cd /path/to/xs-ci-result
sed -n '1,90p' score.txt
ls -lh emu riscv64-nemu-interpreter-so
```

Parse `score.txt`:

- `Unfinished / Aborted Tests` gives the failed benchmark names and point ids.
- `Checkpoint Version` gives the checkpoint profile name.
- The checkpoint root is usually:

```bash
/nfs/home/share/checkpoints_profiles/<Checkpoint Version>/checkpoint-0-0-0/
```

For each failed item, locate the GEM image with `find` instead of reconstructing the full decimal suffix exactly:

```bash
find "$GCPT" -path "*/<benchmark>/<point>/_<point>_*.zstd" -print -quit
```

Example:

```bash
GCPT=/nfs/home/share/checkpoints_profiles/spec06_gcc15_rv64gcb_base_260122/checkpoint-0-0-0/
find "$GCPT" -path "*/h264ref_sss/159971/_159971_*.zstd" -print -quit
```

This avoids mismatches such as score coverage `0.0798411` while the checkpoint file is `_159971_0.079841_.zstd`.

#### 2. Choose a server

Use the user's `zzqxstop` helper to inspect server load:

```bash
EXECUTE_CURRENT_NODE=n /nfs/home/zhengzhongqiang/Work/script/zzqxstop
```

Choose a reachable node with `cores > 256`, preferring the lowest load. If SSH to the best node is not permitted, use the best already-approved reachable node and state that choice.

#### 3. Choose the rerun mode and create one directory per failed checkpoint

First choose exactly one rerun mode. Default to `perf-rerun`, so reruns produce `[PERF]` counters and remain score/report compatible. Switch away from `perf-rerun` only when the user explicitly asks for waveform capture or LightSSS fork debugging.

- `perf-rerun`: default mode for score/report compatibility, IPC comparison, and `[PERF]` counters. Always include `--force-dump-result`.
- `wave-rerun`: use this only when the user wants a waveform around a failing window.
- `fork-debug-rerun`: use this only when the user wants LightSSS fork debugging for an actual abort/assert window.

Do not mix the modes in one command. In particular:

- Do not combine `--enable-fork` with `-b/-e` and `--dump-wave`.
- Do not add `--wave-path`; for this workflow it is not effective. Wave dumps are created under `build/` in the directory selected by `export NOOP_HOME="$PWD"`.
- Do not use `--dump-wave-full` unless the user explicitly asks for a full waveform and accepts the very large output size.
- Keep `[PERF]` counters enabled by default with `--force-dump-result`; prefer `perf-rerun` without wave/fork flags unless the user asks for another mode.

For each failed checkpoint create:

```text
<case-name>_debug/
  run_debug.sh
  launch_local.sh
  simulator_out.txt
  simulator_err.txt
  launch.info
  emu.pid
```

Case-name convention:

```text
<benchmark>_<point>_<coverage>_debug
```

Use the score coverage string for the directory name, but use `find` to locate the actual `.zstd`.

`perf-rerun` template, suitable for scoring and `[PERF]` counters:

```bash
#!/usr/bin/env bash
set -euo pipefail

GEM=/path/to/checkpoint.zstd
GCPT=/nfs/home/share/checkpoints_profiles/<Checkpoint Version>/checkpoint-0-0-0/

cd "$(dirname "$0")"
export NOOP_HOME="$PWD"

../emu \
  --diff ../riscv64-nemu-interpreter-so \
  -i "$GEM" \
  -W 20000000 \
  -I 40000000 \
  -r "$GCPT" \
  -s 4564 \
  --force-dump-result \
  > simulator_out.txt \
  2> simulator_err.txt
```

`wave-rerun` template, suitable for a bounded waveform window:

```bash
#!/usr/bin/env bash
set -euo pipefail

GEM=/path/to/checkpoint.zstd
GCPT=/nfs/home/share/checkpoints_profiles/<Checkpoint Version>/checkpoint-0-0-0/
LOG_BEGIN=123456
LOG_END=124456

cd "$(dirname "$0")"
export NOOP_HOME="$PWD"

../emu \
  --diff ../riscv64-nemu-interpreter-so \
  -i "$GEM" \
  -W 20000000 \
  -I 40000000 \
  -r "$GCPT" \
  -s 4564 \
  -b "$LOG_BEGIN" \
  -e "$LOG_END" \
  --dump-wave \
  > simulator_out.txt \
  2> simulator_err.txt
```

`fork-debug-rerun` template, suitable for LightSSS abort/assert debugging:

```bash
#!/usr/bin/env bash
set -euo pipefail

GEM=/path/to/checkpoint.zstd
GCPT=/nfs/home/share/checkpoints_profiles/<Checkpoint Version>/checkpoint-0-0-0/

cd "$(dirname "$0")"
export NOOP_HOME="$PWD"

../emu \
  --enable-fork \
  --diff ../riscv64-nemu-interpreter-so \
  -i "$GEM" \
  -W 20000000 \
  -I 40000000 \
  -r "$GCPT" \
  -s 4564 \
  > simulator_out.txt \
  2> simulator_err.txt
```

`launch_local.sh` template:

```bash
#!/usr/bin/env bash
set -euo pipefail

cd "$(dirname "$0")"

{
  date
  hostname
} > launch.info

nohup ./run_debug.sh > launch.log 2>&1 &
echo "$!" > emu.pid
echo "started pid $(cat emu.pid) on $(hostname)"
```

Make both scripts executable:

```bash
chmod +x <case>_debug/run_debug.sh <case>_debug/launch_local.sh
```

#### 4. Launch on the chosen server

Run one launch command per failed checkpoint:

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=no <node> \
  /path/to/xs-ci-result/<case>_debug/launch_local.sh
```

Then verify each wrapper has an emu child:

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=no <node> \
  "ps -p <wrapper-pids> -o pid,ppid,stat,etime,time,cmd"

ssh -o BatchMode=yes -o StrictHostKeyChecking=no <node> \
  "pgrep -P <wrapper-pid> -a"
```

Also verify each `simulator_out.txt` has the right checkpoint and that `--enable-fork` took effect by checking for lines like:

```text
The image is ...
DRAMsim3 memory system initialized.
Overwrite 3584 bytes ...
The reference model is ../riscv64-nemu-interpreter-so
```

If output stops at `DRAMsim3 memory system initialized.` and never reaches `The reference model is ...`, check that `--enable-fork` is present.

#### 5. Report

Report:

- selected node and why
- every failed checkpoint found in `score.txt`
- debug directory for each case
- wrapper PID and emu child PID for each launched case
- where to tail logs

Do not kill existing remote runs unless the user asks. If the user asks to stop a prior run, terminate the wrapper and its emu child together, then verify no matching process remains.

### Step0 - Resolve Input

If the user has not already provided them, first try to discover them reliably from local context. Only ask the user when one or more required inputs cannot be found with high confidence.

Required or useful inputs:

- waveform path (`.vcd` or `.fst`)
- Chisel source root
- optional emitted RTL root (`build/rtl`)
- optional error log path such as `simulator_out.txt`
- optional focus scope (e.g. `TOP.SimTop.core.rob`) or debug hint

Recommended prompt when discovery is insufficient:

> Please fill in this debug template and send it back:
>
> ```text
> debug_type: hardware bug debug
> waveform: /path/to/run.fst_or.vcd
> scala-root: /path/to/XiangShan/src/main/scala/xiangshan
> rtl-root: /path/to/XiangShan/build/rtl
> error-log: /path/to/simulator_out.txt
> focus-scope: TOP.SimTop.core.rob
> suggestion: what you suspect or what looks wrong
> top: SimTop
> window-len: 1000
> ```

When a user asks to debug but the required inputs are missing, prefer asking them to fill in the template above instead of asking a vague free-form question.

### Debug Request Template

```text
debug_type: hardware bug debug
waveform: /path/to/run.fst_or.vcd
scala-root: /path/to/XiangShan/src/main/scala/xiangshan
rtl-root: /path/to/XiangShan/build/rtl
error-log: /path/to/simulator_out.txt
focus-scope: TOP.SimTop.core.rob
suggestion: what you suspect or what looks wrong
top: SimTop
window-len: 1000
```


All commands run from the skill root directory. Use `cd` once at the start:

```bash
cd ~/.codex/skills/hardware-skill/debug/hardware-debug-skill-no-pywellen
```

### Step 1 — Inspect inputs

```bash
python scripts/hw_debug_cli.py inspect-inputs \
  --scala-root /path/to/src/main/scala/xiangshan \
  --waveform /path/to/run.fst \
  [--rtl-root /path/to/build/rtl] \
  [--error-log /path/to/simulator_out.txt] \
  [--focus-scope TOP.SimTop.core.rob] \
  [--suggestion "hang near rob tail"] \
  [--top SimTop]
```

`inspect-inputs` validates paths, reports artifact sizes, checks cache status, and, when an error log is provided, extracts a likely bug type such as `difftest_error` or `assert_error`.

If the bug type is `difftest_error`, `inspect-inputs` also tries to run the configured difftest disassembler helper and writes `disassembly.txt` next to the input `simulator_out.txt`. It also writes `waveform_search_signals.txt` in the same directory, using Markdown-style headings, bullets, and tables to make the saved waveform-search checklist easy to read.

If the bug type is `assert_error`, `inspect-inputs` also tries to write `assert_debug_guide.md` next to the input `simulator_out.txt`. It should also write `waveform_search_signals.txt` next to the same `simulator_out.txt`, and that file must include the asserted Verilog site, matching Scala/Chisel locations, and the waveform signals most directly involved in the trigger condition.

Then **prints the exact commands to run next**. Use those printed commands as the next steps.

If it warns that artifacts are large, tell the user before proceeding.

### Step 2 — Build RTL authority (skip if no `--rtl-root`)

```bash
python scripts/hw_debug_cli.py build-authority \
  --rtl-root /path/to/build/rtl \
  --top SimTop \
  [--out-dir <authority-out>]
```

Reuses cache automatically. Add `--force` to rebuild.

### Step 3 — Build waveform metadata cache

```bash
python scripts/hw_debug_cli.py build-wave-meta \
  --waveform /path/to/run.fst \
  [--out-dir <meta-out>]
```

Reuses cache automatically. Add `--force` to rebuild.

`--vcd` remains available as a compatibility alias for older command lines.

### Step 4 — Query a debug packet

```bash
python scripts/hw_debug_cli.py query-packet \
  --waveform /path/to/run.fst \
  --focus-scope TOP.SimTop.core.rob \
  --t-start 123000 \
  --t-end 124000 \
  --out <packet-out>/packet_t123000_124000.json \
  [--meta-dir <meta-out>] \
  [--authority <authority-out>/rtl_authority.sqlite3] \
```

Choose `t-start` and `t-end` to cover the suspected failure region. Use a narrower range when you want a compact packet for LLM analysis.

### Step 4b — (Optional) Query one signal value at one time

```bash
python scripts/hw_debug_cli.py query-signal-value \
  --waveform /path/to/run.fst \
  --signal TOP.SimTop.core.rob.commit_valid \
  --time 123456 \
  [--meta-dir <meta-out>]
```

Use this when you need the value of one specific signal at one specific simulation time.

### Step 5 — (Optional) Add rough Chisel candidates

```bash
python scripts/hw_debug_cli.py rough-map-chisel \
  --packet <packet-out>/packet_t123000_124000.json \
  --mapping /path/to/rough-mapping.json \
  --out <packet-out>/packet_t123000_124000_rough.json
```

Only run this step if a rough mapping artifact is available. Treat results as guesses, not exact source truth.

### Step 6 — Analyze

1. If an error log is available, use its bug-type hint as a prior, not as proof.
2. For `assert_error`, start from the asserted Verilog line first, recover the actual trigger condition, locate the corresponding Scala/Chisel file and line, and explain why that condition became true.
3. For `difftest_error`, first locate the mismatching instruction from `simulator_out.txt` and print it explicitly.
4. If `disassembly.txt` was generated, use it to recover the mismatching instruction's assembly and nearby instruction stream before diving into waveform details.
5. Search the ROB commit path in Scala first for `difftest_error`. Prioritize:
   - `backend/rob/RobBundles.scala` for `commit_v`, `commit_w`, `debug_pc`, `debug_instr`, `rfWen`, `commitType`
   - `backend/rob/Rob.scala` for `io.commits.commitValid`, `io.commits.isCommit`, `io.commits.robIdx(i)`, `io.commits.info(i).debug_pc`, `io.commits.info(i).debug_instr`
   - `backend/CtrlBlock.scala` for `frontendCommit`, `rob.io.flushOut`, and redirect/flush timing
6. Read waveform evidence with `query-packet` or `query-signal-value`.
7. If `rtl.match_status == "exact"`, use `module_type` and `local_signal_name` to narrow the search to the most relevant Scala/Chisel source candidates.
8. Search the Scala root by module name, signal name, and nearby subsystem names to find the best candidates.
9. Analyze the Scala/Chisel code first.
10. Present rough Chisel candidates from step 5 only as secondary, lower-confidence hints.
11. Only inspect generated SystemVerilog if Scala cannot explain the behavior.

If you need a point lookup instead of a time-range packet, use `query-signal-value`.

### Step 6a - Root-Cause Backtrace Contract

After classifying the bug type, use that type only to choose the starting point and first source files. Do not let the type replace evidence.

Start from the final assert, difftest mismatch, abort, or violated invariant as the failure scene. Treat it as the first observable failure site, not as the root cause by default. Then recursively walk backward one causal hop at a time:

- name the current bad signal or state
- give the exact waveform time where it is bad
- give the actual wrong value observed in the waveform
- give the expected correct value or correct behavior from the protocol, invariant, Scala/Chisel source, or emitted RTL
- list the immediate upstream signals or state that generate it, as explicit branch candidates
- classify each branch candidate as `meets expectation`, `violates expectation`, or `unknown/not observable`
- explain the propagation relation from upstream to current signal
- continue depth-first only on `violates expectation` branches first, then on necessary `unknown/not observable` branches
- record `meets expectation` branches as exclusion evidence and do not continue into them unless all abnormal branches are ruled out
- if a searched branch cannot explain the downstream failure, backtrack to the nearest branch point and search the next violating or unknown branch
- decide whether the current hop is only a downstream symptom or a source candidate

Stop the backtrace only when the earliest abnormal source candidate satisfies both conditions:

- its inputs or earlier prerequisites are normal, unavailable, or explicitly ruled out
- its actual wrong value is sufficient to propagate through the observed downstream chain to the final failure

The final conclusion must name the bug-triggering code, the boundary scenario that activates it, and the proposed fix or next validation patch.

Every strong debug conclusion must close both sides of the evidence chain:

- Code evidence: Scala/Chisel source, emitted RTL, generated expression, instance wiring, or protocol invariant explaining how the signal is produced.
- Waveform evidence: exact FST/GTKWave signal, exact waveform time, actual value, and expected value proving the behavior occurred.
- Code-only reasoning is a hypothesis. Waveform-only observation is a symptom. Only a code-plus-waveform chain can support a root-cause conclusion.

For every proof signal, align the three naming layers before citing it:

- Chisel/Scala layer: design source signal or expression.
- emitted RTL layer: generated Verilog/SystemVerilog signal, assign, concat/mux, or instance port wiring.
- FST/GTKWave layer: exact waveform hierarchy signal.

Use the exact FST/GTKWave signal as the primary evidence key in the analysis. Use Chisel/Scala and emitted RTL names only in mapping tables, source-logic explanations, or parenthetical aliases after the mapping is established. Do not mix Chisel, RTL, and GTKWave names interchangeably in the same reasoning step.

If the waveform lacks an internal Chisel `val`, use exact generated FST signals, nearby handshakes, registered outputs, or probe signals. Use `xiangshan-wave-signal-mapper` to turn Chisel/Scala names into exact GTKWave hierarchy names before citing them.

### Step 6a.1 - Waveform Boundary and Dynamic-Index Discipline

When debugging a final assert generated from a Chisel dynamic index, do not stop at "selected queue is empty" and do not infer the selected queue from the id name alone.

Apply these checks before writing the conclusion:

- Treat the assert/difftest report as the failure scene and prove its trigger condition first. Then check whether an earlier bad combination exists in the waveform and use that as the next backtrace hop.
- Query exact point values around the boundary with `query-signal-value`, especially the last legal handshake, the first bad trigger-condition cycle, and the report/abort cycle.
- For ready/valid queues, distinguish "valid before the clock edge" from "empty after a dequeue." A legal `last=1` beat with `deq.valid=1` and `do_deq=1` can empty the queue; a later same-id non-last beat is a new protocol violation, not proof the earlier queue was wrong.
- For generated expressions such as `_GEN_3[auto_out_r_bits_id]`, map the Chisel source and emitted RTL together:
  - Chisel dynamic select, e.g. `VecInit(rqueues.map(_.deq.valid))(rid)`
  - emitted RTL concat or mux that builds `_GEN_3`
  - instance wiring that proves which queue signal corresponds to the selected bit
- If the selected signal name contains a generated suffix or vector width, use the exact FST path. Do not cite `foo[6:0]` when the FST signal is `foo [6:0]`.

For AXI read-response asserts, the minimal boundary table should include:

| Role | Signals |
| --- | --- |
| final assert inputs | `out.r.valid`, `out.r.bits.id`, selected `r_valid`, `out.r.bits.last` |
| queue state | selected queue `io_deq_valid`, `io_deq_ready`, `do_deq`, `empty`, pointers when useful |
| protocol lifecycle | previous AR/fire or read-beat count, final legal `last=1` beat, first illegal extra beat |
| propagation | upstream R-channel `valid/ready/id/last` at each boundary |

### Step 6b - Bug-Source Proof Contract

A claimed bug source must include all of these fields:

- `Bug Type`: assert, difftest mismatch, abort, hang, deadlock, protocol violation, data corruption, or unknown
- `Source Signal`: Chisel/Scala name and exact GTKWave/FST hierarchy
- `Source Time`: exact waveform time
- `Observed`: actual wrong value
- `Expected`: expected correct value or behavior
- `Source Logic`: Scala/Chisel source, and emitted RTL when needed, that defines the source signal
- `Propagation`: how the wrong value reaches the final assert, mismatch, abort, or invariant failure
- `Exclusion`: why later stages are symptoms rather than independent root causes

Do not claim root cause from source logic alone. A root-cause claim needs waveform value evidence at a specific time plus source logic explaining why the value is wrong and how it propagates.

### Step 6c - Evidence Signal Table

For every debug conclusion, include a compact signal table. Use exact FST/GTKWave names as the primary key, not guessed Chisel names.

| Role | Chisel/Scala signal | emitted RTL signal/expression | FST/GTKWave signal | signal meaning | expected correct value/behavior | actual wrong value/observation | exact waveform time | mapping proof | propagation relation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

Keep this table focused on proof signals: final failure signal, each backtrace hop, the source candidate, and the key normal/ruling-out signals. Avoid dumping every nearby waveform signal.


## Output

Write the answer in two parts:

**Summary** (2-4 sentences)

- For a debug request, include:
  - `Bug Type Hint`: if an error log was provided, report `difftest_error`, `assert_error`, or `unknown`
  - `Assert Site`: for `assert_error`, report the asserted Verilog file and line
  - `Mismatching Instruction`: for `difftest_error`, report the mismatching PC and instruction
  - `Phenomenon`: one sentence describing the anomaly seen in the waveform
  - `Root Cause Category`: a standard hardware bug class such as state machine deadlock, data hazard, backpressure stall, or flush-handling miss
  - `Root Cause Source`: source signal, exact waveform time, actual wrong value, and expected correct value
  - `Confidence`: state whether this is high confidence or low confidence
- For an exploration request, include:
  - `Function`: what the module does
  - `Structure`: its main internal buffers, state, and submodules
  - `Interconnect`: how it connects to other modules

**Detailed Analysis**

- For a debug request:
  1. Expand the `Bug Type Hint`, `Assert Site` / `Mismatching Instruction`, and `Root Cause Category` from the summary.
  2. For `assert_error`, explain the trigger condition from emitted Verilog first, then show the matching Scala/Chisel logic.
  3. For `difftest_error`, explain the ROB commit-path chain you traced from Scala/Chisel to waveform.
  4. Perform the Step 6a root-cause backtrace from final failure to earliest abnormal source candidate.
  5. Prove the source with the Step 6b fields: exact waveform time, actual wrong value, expected correct value, source logic, propagation relation, and exclusion evidence.
  6. Include the Step 6c evidence signal table with exact FST/GTKWave hierarchy names and signal meaning.
  7. Cite the most relevant error-log clues, waveform evidence, and Scala/Chisel logic that support the hypothesis.
  8. Give a fix recommendation if confidence is high.
  9. Otherwise give the next best debugging steps.
- For an exploration request:
  1. Support `Function` by using the Scala/Chisel source to explain what the module does, and by using waveform evidence to analyze its key pipeline signals and timing behavior when sufficient evidence is available.
  2. Support `Structure` with the main state, buffers, queues, or submodules.
  3. Support `Interconnect` with the other modules, or interfaces that matter most.

Use precise terms in the detailed analysis:

- signals/timing: `rising edge`, `falling edge`, `valid`, `ready`, `handshake`, `backpressure`, `stall`, `flush`, `state transition`
- architecture/control: `pipeline stage`, `hazard detection`, `forwarding`, `cache hierarchy`, `fetch/decode/execute`, `instruction set architecture`, `bus arbitration`, `memory consistency`, `reorder buffer`, `issue queue`, `commit/retire`

Avoid: raw per-cycle value dumps, long exact-signal lists, large artifact path inventories, large SystemVerilog excerpts, and preprocessing detail.
Include only the few source files or artifact paths that materially support the analysis.

## Rules

- Let `inspect-inputs` choose default artifact paths; only override when the user asks.
- If `--error-log` is provided, use it to infer a likely bug type, but do not treat it as sufficient proof.
- Do not present a root cause unless it is supported by both code evidence and waveform evidence. If either side is missing, label the result as a hypothesis or next debugging step.
- Do not mix Chisel, emitted RTL, and FST/GTKWave names without a mapping table. Establish the three-layer mapping first, then use the FST/GTKWave name consistently as the evidence key.
- If the error log indicates `assert_error`, treat the asserted RTL file/line as the first narrowing clue.
- If the error log indicates `assert_error`, try to generate `assert_debug_guide.md` and `waveform_search_signals.txt` in the same directory as `simulator_out.txt`.
- If the error log indicates `difftest_error`, use mismatch or abort PC as a starting hint, but confirm the root cause from waveform and source.
- If the error log indicates `difftest_error`, try to generate `disassembly.txt` and `waveform_search_signals.txt` in the same directory as `simulator_out.txt`.
- If writing helper files fails because the target directory is not writable, tell the user the exact target path and ask for permission. Do not silently write fallback copies to hidden temporary locations.
- Reuse cached artifacts; rebuild only when needed or explicitly requested.
- Treat `rtl_authority.sqlite3` matches as exact RTL ownership.
- If no `build/rtl` is provided, label the result `waveform-only analysis`.
- Treat rough Chisel joins as guesses, never as proven ownership.
- Avoid reading large SystemVerilog files unless Scala-first analysis is blocked.

## Reference

For command flags, artifact layout, and schema details:

- `README_en.md` (English reference)
- `README.md` (Chinese reference)

For wavedrom language:

- `wavedrom.md`
