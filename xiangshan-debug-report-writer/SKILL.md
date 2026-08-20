---
name: xiangshan-debug-report-writer
description: Use when XiangShan hardware debug analysis needs to be written as a Chinese Markdown verification report after waveform/source investigation, especially reports that must document assert_error, difftest mismatch, abort, root-cause backtrace, waveform evidence time points, expected vs observed values, GTKWave signal hierarchy, and confidence.
---

# XiangShan Debug Report Writer

## Overview

Use this skill after `hardware-debug-waveform` has gathered bug type, source evidence, waveform observations, and signal hierarchy. The output is a Chinese Markdown report that lets the user verify the debug reasoning step by step.

正文、推理、结论使用中文。硬件术语、bug 类型、模块名、信号名、文件名、路径、命令保持英文，例如 `assert_error`、`difftest mismatch`、`s1_specInstrCount`、`TOP.SimTop...inner_ifu`、`Ifu.scala`。

## Inputs

Before writing the report, collect or state missing:

- bug type: `assert_error`、`difftest mismatch`、`abort`、`hang`、`protocol violation`、`data corruption` 或 `unknown`
- failure site: assert site、mismatch PC、abort point、violated invariant
- waveform artifact: `.fst`/`.vcd` path and exact time points used
- source files: Scala/Chisel files and emitted RTL files used as evidence
- signal map: Chisel/Scala signal to exact FST/GTKWave hierarchy, usually from `xiangshan-wave-signal-mapper`
- observed values and expected correct values
- root-cause confidence and unresolved assumptions

Do not invent missing waveform values. If evidence is absent, put it in `待验证项`.

## Report Structure

Use this section order for every debug verification report:

1. `问题概述`
2. `Bug 类型`
3. `错误点与回溯起点`
4. `错误点反向回溯`
5. `Bug 源头证明`
6. `错误值与正确期望值`
7. `波形证据时间点`
8. `信号含义与传导关系`
9. `信号三层对齐表`
10. `GTKWave 信号清单`
11. `结论与置信度`
12. `修复建议`（仅当结论已定位 root cause 时必填；未定位时省略，把修复方向写入 `待验证项`）
13. `待验证项`

## Required Content

### 问题概述

用 3-5 句话说明：

- 哪个 testcase/checkpoint 失败
- 最终错误现象是什么
- 使用了哪些主要证据：`simulator_out.txt`、`simulator_err.txt`、`.fst`、Scala/Chisel、RTL
- 结论是否已闭合，还是仍有待验证项

### Bug 类型

说明分类依据：

- `assert_error`: 给出 assert message、RTL file/line、触发条件
- `difftest mismatch`: 给出 mismatch PC、instruction、architectural state difference
- `abort`: 给出 abort log、触发模块、退出原因
- `hang/deadlock`: 给出无前进条件、阻塞链、最后活跃时间

### 错误点与回溯起点

从最终失败点开始，不要直接跳到源码结论。必须写清楚：

- 最终失败信号或状态
- exact waveform time
- observed value
- expected behavior
- 该点为什么适合作为反向回溯起点

### 错误点反向回溯

按因果链逐级写，每一级使用相同格式：

```text
Step N:
- 当前异常点:
- 主信号名(GTKWave):
- Chisel/Scala 对应:
- emitted RTL 对应:
- exact waveform time:
- observed:
- expected:
- 上游来源:
- branch classification: meets expectation / violates expectation / unknown/not observable
- propagation relation:
- 判断: 下游症状 / source candidate
```

停止回溯时说明为什么停止：

- 更上游输入正常
- 更上游信号不存在于波形但有替代证据
- 当前 source candidate 的错误值足以传导到最终失败
- 排除了下游独立出错的可能

### Bug 源头证明

必须包含：

| 项目 | 内容 |
| --- | --- |
| Bug Type | `assert_error` / `difftest mismatch` / `abort` / ... |
| Source Signal | exact FST/GTKWave hierarchy as primary key, plus Chisel/Scala and emitted RTL mapping |
| Source Time | exact waveform time |
| Observed | actual wrong value |
| Expected | expected correct value/behavior |
| Source Logic | Scala/Chisel source and RTL when needed |
| Propagation | how the wrong value reaches the final failure |
| Exclusion | why downstream stages are symptoms |

### 错误值与正确期望值

单独列出最关键的值对比：

| Signal | exact waveform time | Observed | Expected | Why expected |
| --- | --- | --- | --- | --- |

`Why expected` 必须来自 protocol invariant、Scala/Chisel logic、RTL assignment、queue capacity、valid/ready contract、ISA architectural rule 或 test oracle。

### 波形证据时间点

列出可复查时间点：

| Time | Evidence | Meaning |
| --- | --- | --- |

时间点必须足够具体，不要只写“错误前后”。

### 信号含义与传导关系

说明每个关键 proof signal 的语义和方向：

- `valid/ready` 是握手关系还是 backpressure
- counter/count 信号表示当前 packet、next packet、queue occupancy 还是 speculative allocation
- flush/redirect/cancel 是否会改变当前证据链
- source signal 如何一级一级传到 final failure signal

### 信号三层对齐表

Chisel 设计存在三层命名空间：Chisel/Scala 源码层、emitted RTL 层、FST/GTKWave 波形层。报告必须先把关键 proof signal 的三层名字对齐，再进入正文推理。

统一规则：

- `FST/GTKWave signal` 是报告正文中的主信号名和证据主键。
- `Chisel/Scala signal` 用来说明设计意图、source logic 和 expected behavior。
- `emitted RTL signal/expression` 用来证明 Chisel 动态索引、生成表达式、实例连接和 FST 层次之间的映射。
- 同一个信号在正文中不要混用三套名字。首次出现时可写 `GTKWave signal`，括号中给出 Chisel/RTL 对应；后续统一使用 GTKWave 名。
- 若任一层映射缺失，标为 `missing/not emitted/not mapped`，并把相关结论降级为 hypothesis 或待验证项。

对齐表格式：

| Role | Chisel/Scala signal | emitted RTL signal/expression | FST/GTKWave signal | Mapping proof |
| --- | --- | --- | --- | --- |
| final assert input |  |  |  |  |
| source candidate |  |  |  |  |
| propagation boundary |  |  |  |  |

`Mapping proof` 应说明映射依据，例如 source line、RTL assign/concat、instance port wiring、`wave_db/signals.json` 或 `fstminer -n` 命中。

### GTKWave 信号清单

使用 exact FST/GTKWave hierarchy：

| Chisel/Scala signal | FST/GTKWave signal | 信号含义 | 期望行为 | 实际观察 | 关键时间 |
| --- | --- | --- | --- | --- | --- |

如果某个 Chisel `val` 不在波形中，写明 replacement signal 或 `missing in FST`，不要猜层级名。

### 结论与置信度

结论必须包含：

- root cause 一句话
- 置信度：high / medium / low
- 置信度依据
- 如果是 low/medium，缺什么证据能提升置信度

### 修复建议

**适用条件**：仅当 `结论与置信度` 已定位 root cause（high/medium confidence）时必填；若结论是 hypothesis、证据不足或未定位，不写本小节，只把修复方向写进 `待验证项`。

必须写清：

- 修复对象分层：明确是 DUT RTL、仿真/参考模型（如 difftest/NEMU/checker）、脚本还是配置；若 root cause 在参考模型侧，必须明确说明“不是 DUT bug”，并给出参考模型侧的修复方式
- 修复方式：修改的文件与函数、关键代码位置或伪代码、修复后的行为
- 修复原理：为什么该修改能消除错误传导（结合报告中的 source signal 与 propagation chain）
- 验证方法：重跑原 testcase 的预期结果（如不再 abort 的 cycle 与信号）、需要回归的同类场景、性能/开销对比
- 风险与影响范围：是否影响综合、单核运行（如 `NUM_CORES == 1`）、现有 check 行为，以及退化/冗余说明

示例（参考模型侧修复）：在 `RefillChecker::check` 的 goldenmem 一致分支追加 `proxy->ref_memcpy(paddr, probe.data, 64, DUT_TO_REF)`，或在 `AtomicChecker` 更新 goldenmem 后同步对应地址到 ref 私有 pmem，使 NEMU 每 hart 私有内存能反映跨核 cache-to-cache 传导的数据。

### 待验证项

列出用户可以继续在 GTKWave 或源码里核查的点：

- 需要看的 signal
- 需要看的 exact waveform time
- 需要确认的 source line 或 invariant
- 若第 12 节 `修复建议` 未填写，此处必须给出修复方向

## Rules

- 用中文表述推理过程；保留英文术语和信号名。
- 不要把源码逻辑当作 root cause 证明的全部；必须有 waveform value 和 exact time。
- 报告正文必须统一使用 FST/GTKWave 层次信号名作为证据主键；Chisel/Scala 和 emitted RTL 名字必须通过 `信号三层对齐表` 绑定后再引用。
- 不要在同一段推理里无说明地混用 Chisel 名、Verilog 名和 GTKWave 名；这会破坏可复查性。
- 不要只给结论；必须给错误点反向回溯过程。
- 不要输出无法复查的笼统描述，例如“某个时候”“大概是 ready 问题”。
- 报告可以指出证据不足，但必须明确写入 `待验证项`。
- 结论已定位 root cause 时，报告必须包含 `修复建议` 一节；未定位时不得编造修复方式，只能在 `待验证项` 中给修复方向。
