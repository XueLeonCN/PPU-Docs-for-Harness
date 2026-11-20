---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=51"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=51"
source_title: "用 Asight Systems 分析大型 asys 报告：按需加载、过滤时间线与区间统计"
source_id: "blog-51"
source_kind: "technical-blog"
source_date: "2026-11-05"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "d709f691d242a6ba5472c2ef1f727d384e9b294aea9709e6e6c01cc66119ca54"
tags:
  - "Asight Systems"
  - "asys"
  - "large-report"
  - "partial-load"
  - "timeline-filter"
  - "group-view"
  - "interval-stats"
---
# 用 Asight Systems 分析大型 asys 报告：按需加载、过滤时间线与区间统计

本文介绍如何使用 Asight Systems 的高级打开、时间线显示选项和 Group View，以及时间区间过滤，完成大型 asys 报告从按需加载到区间统计的分析流程

## 开篇

分析大模型训练或推理的性能时，一份 asys 跟踪报告可能达到几个 GB。直接打开这样的报告，往往需要等待几分钟，内存占用也较高。即使报告已经加载，几百行时间线和整个采集周期的统计结果，也未必能直接回答眼前的问题。

例如，要分析某一次训练迭代，通常不需要把初始化、预热和其他迭代都算进去。要定位 PPU 侧的热点，也不一定需要同时加载 CPU 调度、内存跟踪和所有采样数据。

Asight Systems 提供了几种缩小分析范围的方法，但它们解决的问题不同：

| 遇到的问题 | 使用的功能 | 影响范围 |
| --- | --- | --- |
| 打开报告慢、内存占用高 | 高级打开（Open Advanced） | 加载的数据类型和报告区间 |
| 时间线太多，目标行不好找 | 时间线显示选项、Group View | 当前显示的时间线 |
| 只想统计某一次迭代或某个阶段 | Filter and Zoom in | 事件搜索和统计的时间范围 |

下面按打开报告、查看时间线、执行统计的顺序介绍具体操作。

## 1. 打开之前，只加载需要的数据

从菜单栏选择 `File` -> `Open Advanced`，可以在加载前指定事件类型和报告区间。

![高级打开](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793869100948/e5543413a99b616845d989e53aca67e8/open_advanced.png)

对话框中的报告路径列表支持添加多份报告，这些报告会显示在同一个 Timeline View 中。多报告分析时，可以选择是否启用智能时间线对齐；需要解析调用栈时，也可以导入符号文件。

对于大报告，最值得先调整的是百分比区间和事件类型。

### 1.1. 按百分比选择加载区间

百分比控件指定的是一个**起止区间**，例如选择 30% 到 50%，GUI 只加载报告中间这一段。

### 1.2. 按事件类型选择加载内容

减少加载的事件类型是降低内存占用最直接有效的手段。加载事件类型与跟踪数据的对应关系如下：

| 加载事件类型 | 跟踪数据类型 |
| --- | --- |
| Metrics sampling | PPU Metrics 跟踪<br/>CPU Metrics 跟踪 |
| CPU schedule | CPU 线程调度情况 |
| Memory usage | PPU 内存使用跟踪<br/>Pinned 内存使用跟踪<br/>Host 内存使用采样<br/>Heap 内存使用跟踪 |
| OS runtime | OS Runtime（OSRT）跟踪和调用栈 |
| HGTX | HGTX 跟踪 |
| HGGC | HGGC API 跟踪和调用栈<br/>HGGC PPU Activity 跟踪 |
| acDNN | acDNN API 跟踪和调用栈 |
| acBlas | acBlas API 跟踪和调用栈 |
| Video | Video API 跟踪<br/>Video PPU Activity 跟踪 |
| PCCL | PCCL 活动跟踪 |
| Log | 应用 stdout 和 stderr 日志 |
| NIC metrics | RDMA 网卡运行指标跟踪 |
| Backtrace | HGGC / acDNN / acBlas / OSRT API 调用栈<br/>CPU backtrace 采样跟踪<br/>Python backtrace 采样跟踪<br/>PPU / Pinned / Heap 内存使用调用栈 |

按分析目的，可以参考下面几套组合：

| 分析目的 | 建议勾选 | 可以不勾 |
| --- | --- | --- |
| PPU 侧热点定位 | HGGC、HGTX | CPU schedule、Memory usage、Metrics sampling |
| 通信与多机瓶颈 | PCCL、HGTX、NIC metrics、HGGC | Memory usage、Backtrace |
| 主机侧阻塞排查 | CPU schedule、OS runtime、Backtrace、HGGC | Metrics sampling、Memory usage |
| 内存问题排查 | Memory usage、Backtrace、HGGC | CPU schedule、NIC metrics |

## 2. 报告打开后，隐藏无关时间线

报告加载进来之后，第二个常见问题是 Timeline View 中的行数太多。一份多卡报告中，进程、线程、CPU Core、PPU Stream、各类 metrics 行加起来可能有几百行，目标行被埋在中间。

此时可以先隐藏不关心的行，或者把需要一起看的行放进一个 Group。

### 2.1. 用显示选项隐藏无关行

点击 Timeline View 搜索框左侧的 `Options` 按钮，可以调整当前报告的时间线显示选项。

![时间线显示选项](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793869101100/1d8c7afec618bcf284e1a52ebc9f2636/timeline_options.png)

支持的选项：

| 选项 | 作用 |
| --- | --- |
| Show correlation arrows | 是否显示选中时间线的关联箭头 |
| Show CPU sampling rows | 是否显示 CPU sampling 相关行 |
| Show PPU metrics rows | 是否显示 PPU metrics 相关行 |
| Show NIC metrics rows | 是否显示 NIC metrics 相关行 |
| Show non-HGGC process and thread rows | 是否显示没有 HGGC 信息的线程行和进程行 |
| Show graph node based HGTX timelines | 是否显示基于 graph node 映射的 HGTX 时间线 |

这些选项调整后立即生效，只作用于当前报告；与全局选项冲突时，以当前报告的设置为准。设置不会持久保存，重启 Asight 后会复位。

如果当前主要关注 PPU 执行，可以尝试关闭 `Show non-HGGC process and thread rows`，隐藏没有 HGGC 信息的进程和线程。暂时不看采样指标时，也可以关闭对应的 `metrics` 行。

需要注意，隐藏时间线并不意味着这些数据没有加载。如果目的是减少加载的数据量，仍应在 Open Advanced 中取消相应的事件类型。

### 2.2. 用 Group View 集中查看目标行

如果需要在大量时间线中反复查看同一组行，可以在目标行上右键，将它添加到一个 Group。

![group_view](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793869100599/52fce47a5f3bd8f3099f7c4ffcf10866/group_view.png)

添加后 Timeline View 上方会出现切换 Group 的按钮，目标行右侧出现同色的 Group 标志。切换到该 Group View 后，**只显示目标行、它的所有祖先节点和它的子节点**，其余行全部隐藏。

Group 按钮上右键可以重命名；也可以向 Group 中继续追加其他行；同一行可以加入多个 Group，比如一个 Group 看某张卡的 Stream，另一个 Group 看通信算子。

![group_view_1](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793869100781/2e3a5286a4d413fab0504ee23d856176/group_view_1.png)

## 3. 只统计目标时间段

如果要分析某一次迭代，可以拖拽鼠标左键框选一段时间，选好范围后，右键 -> `Filter and Zoom in`。

![filter](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793869100336/c4ca06b42a35d310c725f51e9b22b49f/filter.png)

这个操作不只是放大时间线。它还会把 Events View、事件搜索和统计的时间范围限制到选定区间：

| 受影响的功能 | 变化 |
| --- | --- |
| Timeline View | 缩放到该区域进行显示，其他区域置灰 |
| Events View | 仅显示该范围内的事件 |
| Event 搜索 | 搜索范围收窄到该区间 |
| 统计系统 | 只对该区间内的事件做统计 |
| 对比视图 | 两份报告可以分别设置统计区间 |
| 汇总视图 | 参与汇总的报告可以各自设置区间 |
| Heap 内存统计 | 从选定的起始时间算到当前点击位置 |

设置过滤区间后，再执行统计规则、查看饼图或进行对比，结果对应的就是这一段时间，而不是整份报告。

例如，比较两份训练报告中一次迭代的耗时时，可以分别在两份报告里选择目标迭代，再查看统计结果。这样可以避免初始化和预热阶段的数据混入当前迭代的统计。

## 4. 总结

大报告的分析效率问题，很多时候不是"报告太大"，而是"一次加载了太多不需要的数据、一次统计了太长的时间范围"。

Asight Systems 在三个层次上提供了不同的聚焦手段：

1. **高级打开：**支持按百分比和事件类型控制加载内容。
2. **时间线显示选项和 Group View：**用于控制显示多少时间线。
3. **Filter and Zoom in：**用于控制统计的时间范围。

配合采集阶段的报告大小控制，从"采得小"到"加载少"再到"统计准"，完整覆盖了大报告的分析链路。
