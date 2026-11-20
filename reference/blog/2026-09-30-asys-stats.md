---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=39"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=39"
source_title: "用 asys 统计系统高效分析性能报告"
source_id: "blog-39"
source_kind: "technical-blog"
source_date: "2026-09-30"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "77006c78230c1458bba2e35c423b45018a8880484d0b120e6c0c312e734e11fa"
tags:
  - "Asight Systems"
  - "asys"
  - "stats"
  - "compare"
  - "aggregate"
  - "kernel-summary"
  - "latency"
  - "utilization"
  - "interconnect"
---
# 用 asys 统计系统高效分析性能报告

> 从单报告画像到多节点对比汇总，介绍 asys 统计规则的使用方法

## 1. 为什么需要统计系统 {#1-为什么需要统计系统}

用 asys 采集一份大模型推理的跟踪报告，动辄包含数百万条跟踪事件。在 Timeline View 里逐条查看固然直观，但面对海量数据时，手动定位性能瓶颈如同大海捞针。

统计系统解决的就是这个问题。它将原始的逐条跟踪事件按维度聚合——按 kernel 名称、按算子类型、按通信组——输出结构化的统计表格，让你快速获得宏观性能画像。统计引擎可以高效处理大规模跟踪报告，相比导出原始数据后自行编写脚本分析，不仅速度更快，统计结果也更加完整准确。

每条统计规则还提供丰富的选项：按时间范围过滤、按设备过滤、按 HGTX range 白名单/黑名单过滤、指定输出列等，灵活定制统计口径。

## 2. 三类统计规则 {#2-三类统计规则}

asys 的统计规则分为三类，分别对应不同的分析场景：

| 类型 | 命令 | 输入 | 典型场景 |
|:------|:------|:------|:----------|
| 单报告统计 | `asys stats` | 一份报告 | 了解单次采集的性能画像 |
| 两报告对比 | `asys compare` | 两份报告 | 优化前后 / 不同配置对比 |
| 多报告汇总 | `asys aggregate` | 多份报告 | 多节点分布式训练全局分析 |

这三类规则在命令行端各自对应一个子命令，在 GUI 端分别对应三个页面。三者的底层引擎是同一个——`asys compare` 和 `asys aggregate` 本质上是 `asys stats` 的封装，内部都转发到 stats 子命令处理。因此，对比规则和多报告汇总规则也可以通过 `asys stats -r` 直接运行。

## 3. 命令行使用统计系统 {#3-命令行使用统计系统}

### 3.1. asys stats 基本用法 {#31-asys-stats-基本用法}

最简单的用法是不指定任何规则，直接对报告执行默认统计：

```bash
asys stats report.asysrep
```

不指定 `-r` 选项时，asys 会执行一组默认规则，覆盖算子类型汇总、时间利用率、算子性能分解、kernel 汇总、API 汇总等常用分析维度，默认输出到控制台。一条命令就能获得一份完整的性能概览。

### 3.2. 指定规则与多规则联合 {#32-指定规则与多规则联合}

通过 `-r` 选项可以指定需要执行的统计规则。`-r` 可以多次指定，也可以用逗号分隔多个规则：

```bash
asys stats -r hggc_ppu_kern_sum,ppu_op_type_sum report.asysrep
```

规则还可以带选项，选项之间用 `:` 分隔，每个选项为 `key=value` 或 `key`（布尔型）：

```bash
asys stats -r ppu_time_util:threshold=60:chunks=80:range-mode=full report.asysrep
```

通过 `asys stats --help-reports ALL` 可以查看所有规则及其可用选项。

### 3.3. 输出格式与输出目标 {#33-输出格式与输出目标}

统计结果默认输出到控制台，使用 column 格式（人类可读的列表格式）。通过 `--output` 选项可以指定输出到文件，文件默认使用 csv 格式：

```bash
# 输出到控制台（默认）
asys stats -r hggc_ppu_kern_sum report.asysrep

# 输出到报告所在目录的 csv 文件
asys stats -r hggc_ppu_kern_sum --output . report.asysrep

# 输出到指定路径
asys stats -r hggc_ppu_kern_sum --output /home/user/analysis report.asysrep
```

当指定多个规则时，可以分别指定不同的输出目标。

### 3.4. asys compare —— 两报告对比 {#34-asys-compare-两报告对比}

```bash
asys compare report1.asysrep report2.asysrep
```

`asys compare` 是 `asys stats` 的简化封装，默认执行一组对比规则，涵盖算子类型汇总比较、时间利用率比较、kernel 汇总比较等。对比结果精简输出到控制台，详细数据写入 CSV 文件。加 `-f true` 可覆盖已存在的文件。这些对比规则同样可以通过 `asys stats -r` 直接运行。

### 3.5. asys aggregate —— 多报告汇总 {#35-asys-aggregate-多报告汇总}

```bash
asys aggregate -f true report1.asysrep report2.asysrep report3.asysrep report4.asysrep
```

`asys aggregate` 同样是 `asys stats` 的封装，默认执行多报告版本的互联不同步统计等规则。多报告统计的核心机制是**报告对齐**：asys 根据各报告中通信算子的结束时间，通过最长公共子序列匹配算法自动计算报告间的时间偏移，将多份报告对齐到统一时间域后再做跨报告统计。这种对齐方式支持各节点采集起始时间不同步的场景，且通过通信组过滤避免不同通信组的错误对齐。

## 4. GUI 使用统计系统 {#4-gui-使用统计系统}

在 GUI 底部 tab 中切换到 "Stats System View" 即可进入统计系统页面：

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763957261/9d5e3696d0f06d3d19b3acf4aecfd109/stats_gui.png)

页面左侧是规则列表，选择规则后右侧显示统计结果表格和饼图。饼图支持点击交互——点击某个扇形，表格只会显示对应分类的行。表格支持选中行自动求列和、列的显示/隐藏和冻结。在 Timeline View 中按住鼠标拖动选择一段时间区间，右键 "Filter and Zoom in"，统计系统会自动只统计该区间内的事件，适合分析特定迭代的性能。

GUI 也支持两报告对比（Comparison View）和多报告汇总（Aggregate View），在底部 tab 切换即可。对比视图提供热点柱状图和 Perf Ratio 性能比值，汇总视图勾选多份报告后操作方式与单报告一致。

## 5. 统计规则介绍 {#5-统计规则介绍}

asys 统计系统支持多种规则，覆盖 kernel 级、算子级、通信、利用率、端到端时延等分析维度。下面介绍几条常用规则的机制和用法。

### 5.1. HGGC PPU kernel 汇总（hggc_ppu_kern_sum） {#51-hggc-ppu-kernel-汇总hggc_ppu_kern_sum}

这是最基础的 PPU 侧统计规则。它将报告中所有 HGGC kernel 按名称分组，累加每类 kernel 的执行时间，按总耗时降序输出。

```bash
asys stats -r hggc_ppu_kern_sum report.asysrep
```

| Time (%) | Total Time (ns) | Instances | Avg (ns) | Name |
|:---------|---------------:|----------:|---------:|:------|
| 81.6 | 826473830 | 28 | 29516922 | pcclKernel_AllReduce_RING_LL_Sum_int8_t... |
| 13.9 | 141273640 | 4 | 35318410 | deltaKernIaLi256... |
| 2.3 | 23263744 | 4 | 5815936 | InitDataKernelIa... |

Time 列是该 kernel 占所有列出 kernel 执行时间的百分比。通过这个表格可以快速定位耗时最长的 kernel。规则支持 `base`（短名称）、`mangled`（原始符号名）、`device`（设备过滤）等选项，还可以通过 `hgtx-name` 选项在 kernel 名前拼接最近的 HGTX range 名称，便于关联到上层算子。

### 5.2. PPU 算子类型汇总（ppu_op_type_sum） {#52-ppu-算子类型汇总ppu_op_type_sum}

这个规则将 PPU 侧活动识别为更高维度的算子，按算子类型（MoE、GEMM、PCCL、FlashAttention 等）聚合，输出每种类型的宏观耗时占比和热点算子名称。算子识别通过 HGTX range 标注和 kernel 名称完成，参数取值相同的算子合并为一组，对性能影响较小的参数差异会被自动忽略。

```bash
asys stats -r ppu_op_type_sum report.asysrep
```

| OP Type | Percent (%) | Total Time (ns) | Instances | Top OP Name |
|:--------|------------:|----------------:|----------:|:------------|
| Summary | 100.0 | 3354637726 | 67247 | MoE:M_72_E8_H6144_In8192_topk2 |
| MoE | 63.6 | 2133601622 | 7266 | MoE:M_72_E8_H6144_In8192_topk2 |
| PCCL | 8.2 | 273847048 | 7395 | AllReduce, p:0, c:442368, d:9... |

Summary 行汇总所有算子类型。这是快速了解应用算子组成的最佳入口。

### 5.3. 互联不同步统计（interconnect_desync_trace / interconnect_desync_summary） {#53-互联不同步统计interconnect_desync_trace-interconnect_desync_summary}

分布式训练中，多个节点的通信算子（如 AllReduce）如果不能同步执行，就会产生等待——一个节点先完成通信，另一个节点还在执行，先完成的节点空等。这就是"互联不同步"（desync）。

**不同步比例**的计算方式为：一个通信组内，所有互联算子的重叠时间与持续时间（从第一个算子开始到最后一个算子结束）的差值占比。不同步比例越高，说明通信等待越严重。

`interconnect_desync_trace` 输出每个通信组的跟踪数据，包含不同步比例和瓶颈算子（最晚开始的算子）的进程和设备信息：

```bash
asys stats -r interconnect_desync_trace report.asysrep
```

| Desync Rate (%) | Duration (ns) | Overlap Duration (ns) | Last Process | Last Device | Instances |
|:-----------------|---------------:|-----------------------:|:-------------|:------------|----------:|
| 92.1 | 551426 | 43302 | 37788 | 7 | 8 |
| 26.5 | 642477 | 472183 | 37785 | 4 | 8 |

`interconnect_desync_summary` 按算子类型分组汇总，输出 P90 不同步率和瓶颈节点信息：

```bash
asys stats -r interconnect_desync_summary report.asysrep
```

| Desync P90 (%) | Desync Avg (%) | Bottleneck Process | Bottleneck Device | Bottleneck Percent (%) | OP Name |
|:---------------|:---------------|:-------------------|:-----------------|:----------------------:|:--------|
| 60.1 | 42.9 | 37787 | 6 | 89.2 | DeepEP:t:64 k:8 e:256... |
| 69.7 | 44.1 | 37781 | 0 | 17.1 | DeepEP:t:56 k:8 e:256... |

Bottleneck 列直接指出哪个进程和设备最频繁地成为通信瓶颈——在这个例子中，进程 37787、设备 6 在 89.2% 的通信组中都是最晚完成的节点，是明显的通信短板。

互联不同步统计支持多报告版本，可以对多节点分布式训练中各节点的报告进行跨报告统计。统计过程会自动对齐多份报告，即使各节点采集起始时间不同步也能准确计算。多报告版本的统计结果能从多机的报告中定位到导致 desync 的具体进程和 PPU 设备，帮助快速锁定通信瓶颈节点。

### 5.4. PPU 端到端时延汇总（ppu_end_to_end_latency_summary） {#54-ppu-端到端时延汇总ppu_end_to_end_latency_summary}

传统的算子汇总是 duration 累加，没有考虑算子之间的遮蔽关系。一个占比高的算子如果总是被另一个算子遮蔽（重叠执行），它对端到端时延的实际贡献为零，但在传统统计中仍然显示为高占比。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763957079/9a38c2a437299d9e144a8817e57de53f/ppu_end_to_end_latency.png)

端到端时延统计通过**遮蔽关系**解决这个问题。遮蔽优先级为：计算类型 > 通信类型 > 内存类型，同优先级下先开始的遮蔽后开始的。只有顶层未被遮蔽的算子才对端到端时延产生贡献。

统计时间从每进程、每 PPU 设备的第一个活动开始到最后一个活动结束，包含期间的 PPU 空闲时间。对于互联通信算子，还能计算由于不同步问题导致的等待时间对端到端时延的贡献：

```bash
asys stats -r ppu_end_to_end_latency_summary report.asysrep
```

| OP Type | Effective Latency Percent (%) | Overlap Percent (%) | Avg Duration (ns) | Instances |
|:--------|------------------------------:|--------------------:|------------------:|----------:|
| MoE | 28.6 | 0.0 | 105994 | 5970 |
| Idle | 10.3 | 0.0 | 82223 | 2764 |
| FlashAttention | 7.2 | 0.0 | 26654 | 5968 |

Effective Latency Percent 列是关键——它表示该算子对端到端时延的实际贡献百分比。Idle 行显示 PPU 空闲时间占了 10.3%，意味着有优化空间。对于通信算子，Desync Latency Percent 列量化了不同步等待的时延贡献。

### 5.5. PPU 时间利用率分析（ppu_time_util） {#55-ppu-时间利用率分析ppu_time_util}

这个规则将 PPU 的活跃时间范围等分为若干时间段（chunk），计算每个时间段的时间利用率——即 PPU 有 kernel、memcpy 等 activity 的时间占该时间段长度的比例。时间上重叠的 activity 不会重复计算，利用率不会超过 100%。

利用率低于阈值的时间段会被输出，相邻的低利用率时间段会自动合并：

```bash
asys stats -r ppu_time_util:threshold=60:chunks=80:range-mode=full report.asysrep
```

| In-Use (%) | Duration (ns) | Start (ns) | PID | Device ID |
|:-----------|---------------:|-----------:|:----|:----------|
| 0.0 | 12411 | 124667294244 | 1142418 | 5 |
| 0.0 | 7648 | 124667369358 | 1142419 | 6 |
| 2.4 | 7092 | 124667285970 | 1142418 | 5 |

In-Use 列为 0.0 表示该时间段 PPU 没有任何活动。配合 `range-mode` 选项可以切换时间范围：`active` 模式从第一个 PPU 活动到最后一个活动，`full` 模式使用指定的时间范围或报告整体时间范围。

## 6. 总结 {#6-总结}

asys 统计系统提供了一套完整的性能分析工具链：从单报告的多维度画像，到两报告的优化效果对比，再到多报告的分布式训练全局汇总。统计规则覆盖了从 kernel 级到算子级、从计算到通信、从利用率到端到端时延的各个分析维度。

命令行的灵活组合（多规则联合、多种输出目标）适合自动化分析流程和 CI/CD 集成；GUI 的交互式表格、饼图和火焰图适合探索式分析。统计规则的丰富选项（时间范围、设备过滤、HGTX 白名单/黑名单）让分析聚焦在真正关心的数据上。

配合事件触发采集和报告大小控制，统计系统可以让性能分析聚焦在关键迭代区间，大幅提升报告分析效率。
