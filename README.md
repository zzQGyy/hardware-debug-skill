# XiangShan Debug Skills

这个目录收纳 XiangShan 硬件功能 bug 分析相关 skill。它们从 assert、difftest mismatch、abort、hang 或协议行为异常出发，利用源码、RTL 和波形证据定位功能根因。文档说明使用中文；硬件术语、bug 类型、模块名、信号名、文件名、路径和命令保持英文。

## Skill 关系

```text
hardware-debug-waveform
  主流程 skill
  负责: 解析错误、分类 bug type、跑/读 waveform、结合 Scala/Chisel 和 RTL 回溯 root cause

xiangshan-wave-signal-mapper
  子工具 skill
  负责: Chisel/Scala signal -> exact FST/GTKWave hierarchy

xiangshan-debug-report-writer
  文档生成 skill
  负责: 把 debug 过程整理成中文 Markdown 验证报告

xiangshan-gtkwave-savefile-generator
  波形视图 skill
  负责: 生成/修复 GTKWave .gtkw，按 debug 回溯阶段分组信号、添加 marker、标色关键信号
```

推荐调用顺序：

```text
1. hardware-debug-waveform
   从 assert_error / difftest mismatch / abort / hang 等错误点开始分析，给出 bug type。

2. hardware-debug-waveform + source/waveform evidence
   结合 Scala/Chisel、emitted RTL、simulator log 和 waveform，从错误点反向回溯。

3. xiangshan-wave-signal-mapper
   当需要给用户 GTKWave 信号清单时，把 Chisel/Scala signal 映射成 exact FST/GTKWave hierarchy。

4. xiangshan-gtkwave-savefile-generator
   当用户要看波形、需要分组/marker/颜色，或 .gtkw 打开后只有 marker 没有 signal 时，生成可直接打开的 GTKWave savefile。

5. hardware-debug-waveform
   从 assert/difftest/abort 第一现场开始，递归追问每个异常信号为什么出现，直到证明 root cause。

6. xiangshan-debug-report-writer
   生成中文 debug 验证报告，方便人工逐项核查。
```

## Debug 方法论

assert / difftest / abort 是第一现场，也就是最先观察到错误被触发的位置。debug 不能停在第一现场，也不能把第一现场直接当成 root cause；应从第一现场开始，递归向上游回溯：

1. 证明第一现场的触发条件：列出 actual wrong value、expected correct value、exact waveform time。
2. 找到生成当前错误值的直接上游信号或状态。
3. 把直接上游信号/状态作为 branch candidates，逐个用代码期望和波形实际值分类为 `meets expectation`、`violates expectation` 或 `unknown/not observable`。
4. 对 `violates expectation` 的分支优先 depth-first search；必要时再查 `unknown/not observable` 分支。
5. 对 `meets expectation` 的分支记录为 exclusion evidence，不继续深搜，除非其他异常分支都被排除。
6. 如果当前深搜分支无法解释下游失败，回到最近的分叉点，换另一个异常或未知分支继续搜索。
7. 每一跳都说明 propagation relation，判断它是 downstream symptom 还是 source candidate。
8. 收敛时必须给出 bug-triggering code、触发边界场景、修复方法或下一步验证补丁。

所有强结论都必须同时有代码依据和波形依据：

- 代码依据：Scala/Chisel 源码、emitted RTL、生成表达式、实例连接或协议不变量，说明这个信号应该如何产生。
- 波形依据：exact FST/GTKWave signal、exact waveform time、actual value、expected value，证明这个行为确实发生。
- 只有“代码 + 波形”闭合的证据链才能作为 root cause 结论依据。只有代码推理只能作为 hypothesis，只有波形现象只能作为 symptom。

报告里每个 proof signal 必须先完成三层对齐：

- Chisel/Scala 层：设计源码里的信号或表达式。
- emitted RTL 层：生成后的 Verilog/SystemVerilog 信号、assign、concat/mux 或实例端口连接。
- FST/GTKWave 层：波形中 exact hierarchy signal。

正文推理统一使用 FST/GTKWave 层次信号名作为证据主键；Chisel/Scala 和 emitted RTL 名字只在映射表、source logic 或括号说明里出现。不要在同一段推理中无说明地交叉使用三套名字。

常见易错点：

- simulator report/abort cycle 是第一现场，但不一定是第一个坏组合出现的 cycle。
- ready/valid queue 要区分“clock edge 前 valid=1”和“dequeue 后 empty=1”。
- `_GEN_3[auto_out_r_bits_id]` 这类动态索引必须从 Chisel `VecInit(...)(rid)`、emitted RTL concat/mux、实例端口连接共同证明，不能只靠 id 名字猜 queue。
- FST/GTKWave signal name 必须精确，尤其 `foo [6:0]` 和 `foo[6:0]` 可能不是同一个名字。

## Skill 的功能边界

| Skill | 主要输入 | 主要输出 | 不负责 |
| --- | --- | --- | --- |
| `hardware-debug-waveform` | `.fst`/`.vcd`、`simulator_out.txt`、`simulator_err.txt`、Scala/Chisel root、RTL root | bug type、root-cause backtrace、source proof、confidence | 生成固定中文验证报告模板 |
| `xiangshan-wave-signal-mapper` | `.fst` 或 `fstminer -n` facnames、Chisel/Scala signal list、scope | exact FST/GTKWave hierarchy table | 判断 bug root cause |
| `xiangshan-debug-report-writer` | 已完成或阶段性的 debug 分析、signal map、waveform evidence | 中文 Markdown debug report | 重新跑仿真、重新查波形、猜缺失证据 |
| `xiangshan-gtkwave-savefile-generator` | `.fst`、已有/目标 `.gtkw`、debug 时间点、proof signal list | grouped GTKWave savefile、marker、colored evidence signals | 判断 root cause、替代波形证据分析 |

## Debug 报告必须覆盖

`xiangshan-debug-report-writer` 生成的报告应包含：

- 问题概述
- Bug 类型
- 错误点与回溯起点
- 错误点反向回溯
- Bug 源头证明
- 错误值与正确期望值
- 波形证据时间点
- 信号含义与传导关系
- 信号三层对齐表
- GTKWave 信号清单
- bug-triggering code
- 触发边界场景
- 修复方法或下一步验证补丁
- 结论与置信度
- 待验证项

## 语言规则

- 中文: 推理过程、结论、证据解释、报告正文。
- 英文: `assert_error`、`difftest mismatch`、`valid/ready`、`flush`、`redirect`、`source candidate`、signal/module/file/path/command。

示例：

```text
在 time=3546986，`inner_ifu.s1_specInstrCount[5:0] = 1`，但根据 `s1_invalidTaken_0 = 1` 时的 `s1_instrCount` 计算，正确期望值应为 10。该错误通过 `io_in_bits_prevInstrCount[5:0]` 传导到 `inner_ibuffer.io_in_ready` 的容量判断。
```
