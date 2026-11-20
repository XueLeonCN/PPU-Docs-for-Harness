---
name: zhenwu-ppu-operator-optimizer
description: "面向真武 PPU（M890、810E 等）的通用算子开发与优化工作流。适用于需要优先检索 t-head 官方开源算子、编译器和集合通信库，再结合本地架构与性能文档完成方案设计、实现、正确性验证、Asight 性能分析和可复现优化迭代的任务。"
---

# 真武 PPU 自动化算子开发与优化

## 目标

把算子开发或优化的任务推进到可验证、可复现、可回滚的工程化结果：

```text
任务约束
→ 官方代码与本地文档侦察
→ 基线测量与工作负载分档
→ 核函数方案和资源预算
→ 实现/适配
→ 编译
→ 正确性
→ Asight 工具链性能分析
→ 单假设优化迭代
→ 性能晋升或回滚
```

优先复用平头哥官方开源实现，再补写新核函数。不要在已有官方实现、同类基础算子或相邻算子尚未排查时直接重写。

## 必须遵守的边界

1. 先确定产品、架构、SDK 和工具链版本。`M890` 与 `810E` 不可混用硬件事实；M890 对应 `ppu0015`，810E 对应 `ppu001`，但运行时仍应通过设备属性确认实际配置。
2. 官方开源代码和官方文档优先于本文件的摘要。遇到冲突时，先比较 commit、SDK 版本、架构条件和 workload。
3. 不得把其他平台的指标、tile、计算单元数、缓存容量或工具参数直接当作真武硬件事实。
4. 没有正确性结果时不得宣布优化有效；没有同轮同负载基线和重复测量时不得宣布稳定加速。
5. 每次实验只改变一个主要假设。保留基线、候选源码、编译参数、性能分析记录和测试结果。
6. 不得未经确认执行高成本的全量集群扫描、长时间生产压测或会覆盖用户现有最佳版本的修改。

## 第一步：官方开源代码优先侦察

这是每轮算子工作的必选阶段，不是背景阅读。

### 搜索顺序

1. 用户当前项目中的现有算子、模板、测试、benchmark 和构建配置。
2. 当前项目已经 vendored 的 t-head 官方代码及其 commit。
3. [平头哥 GitHub 官方组织](https://github.com/t-head)中的对应仓库。
4. 本目录中的官方架构、编程指南和性能博客。
5. 只有上述路径都无法提供可复用方案时，才从零设计核函数。

### 官方仓库优先级

| 仓库 | 优先用途 |
| --- | --- |
| [`t-head/actlize`](https://github.com/t-head/actlize) | 通用算子库、kernel 结构、模板和已有实现 |
| [`t-head/DeepGEMM-for-sail`](https://github.com/t-head/DeepGEMM-for-sail) | GEMM、量化 GEMM、tile、流水和寄存器组织 |
| [`t-head/flash-attention-for-sail`](https://github.com/t-head/flash-attention-for-sail) | FlashAttention 及 Attention 类算子 |
| [`t-head/FlashMLA-for-sail`](https://github.com/t-head/FlashMLA-for-sail) | MLA、KV 分块、warpgroup 和流水 |
| [`t-head/flashinfer-for-sail`](https://github.com/t-head/flashinfer-for-sail) | 推理算子、融合算子及通用 backend |
| [`t-head/flash-linear-attention-for-sail`](https://github.com/t-head/flash-linear-attention-for-sail) | 线性 Attention 和相邻序列原语 |
| [`t-head/triton-for-sail`](https://github.com/t-head/triton-for-sail) | Triton 前端、编译器、AIU load 及 lowering |
| [`t-head/tilelang-for-sail`](https://github.com/t-head/tilelang-for-sail) | TileLang operator DSL 和调度 |
| [`t-head/hggc-samples`](https://github.com/t-head/hggc-samples) | 最小可运行 HGGC/TIX、AIU、Graph、GEMM 示例 |
| [`t-head/pccl`](https://github.com/t-head/pccl) | 集合通信 API、协议和实现 |
| [`t-head/pccl-tests`](https://github.com/t-head/pccl-tests) | 集合通信正确性、拓扑和性能测试 |
| [`t-head/gloo-for-sail`](https://github.com/t-head/gloo-for-sail) | 主机/分布式通信参考实现 |
| [`t-head/sailbandwidth`](https://github.com/t-head/sailbandwidth) | 真武 PPU 带宽基础测量 |

### 侦察要求

- 读取仓库顶层 `README`、构建脚本、代码目录和测试目录。
- 同时搜索精确算子名和相邻 primitive，例如 `gemm`、`matmul`、`attention`、`mla`、`moe`、`gather`、`scatter`、`paged`、`decode`、`prefill`、`topk`、`quant`、`dequant`、`allreduce`、`allgather`、`reduce-scatter`。
- 检查目标架构条件，例如 `ppu001`、`ppu0015`、`ppu_10`、`ppu_15`、`__HGGC_ARCH__`。
- 记录仓库 URL、分支、commit、文件路径、可复用 primitive、适配方式和许可证边界。
- 如果已有实现接近目标，先适配；若只是算法思想可借用，必须写清不能直接复制的部分。
- 如果网络不可用，优先检查项目内 vendored 源码和本目录文档，并明确记录“未验证最新上游版本”。

## 第二步：建立任务卡和证据表

编码前先形成任务卡：

```yaml
operator:
  name: ""
  semantic_reference: ""
  input_layouts: []
  input_shapes: []
  output_semantics: ""
target:
  product: "M890 | 810E | other"
  architecture: "ppu0015 | ppu001 | unknown"
  sdk_version: ""
  driver_version: ""
build:
  frontend: "HGGC | TIX | Triton | TileLang | other"
  compiler: ""
  arch_flags: []
  rdc: false
  register_limit: null
quality:
  tolerances: {}
  deterministic: null
  boundary_cases: []
performance:
  baseline: ""
  primary_metric: ""
  secondary_metrics: []
  representative_workloads: []
```

证据表至少记录：

```text
结论 | 类型（源码/官方文档/实测/推断） | 仓库 commit 或文档路径 | 适用硬件和版本
```

没有证据的硬件结论只能标记为待验证，不能写入最终交付。

## 第三步：工作负载分档与基线测量

不要只针对一个 shape 优化。从真实业务输入中提取 2 至 4 类代表负载：

- 小 batch、单 token 或 decode 型；
- 中等规模、过渡型；
- 大 batch、prefill 或 compute-bound 型；
- 尾块、长 K、稀疏、不规则或最耗时的 outlier。

通信任务还要覆盖：

- 单机与多机；
- 小消息与大数据量；
- 单 group 与多 group；
- 不同 Rank 数和拓扑。

固定以下口径：

- 输入 dtype、layout、shape 和分布；
- 设备数、频率、SDK/驱动版本；
- 预热次数、重复次数、同步点和计时范围；
- 正确性容差、NaN/Inf 行为和确定性要求；
- 编译选项。

Baseline 至少记录：

- 首次运行与预热后运行；
- 中位耗时、P95 耗时和波动；
- 编译耗时（如属于交付指标）；
- kernel 耗时与端到端耗时；
- 通信任务的带宽、时延、不同步和节点差异。

单次最快值只作为线索。晋升判断优先使用重复运行的平均值或中位数，并检查 P95 和方差。

## 第四步：选择实现路线

按以下优先级选择：

1. 官方库已有精确实现：直接复用。
2. 官方库有相邻实现或可复用 primitive：适配、组合或移植。
3. 官方示例覆盖所需硬件能力：以示例为骨架进行实现。
4. HGGC/TIX 可表达：编写定制 kernel。
5. Triton/TileLang 可表达且性能足够：优先保持可维护性。
6. 只有控制面或性能上限确实受阻时，再下沉到更低层 TIX。

### 算子类型检索路由

- GEMM 与线性层算子：先看 `actlize`、`DeepGEMM-for-sail`、`hggc-samples`；再读 FP8 blockwise、BF16 DenseGemm、W4A8、W4A16 文档。
- Attention 与 MLA 算子：先看 `flash-attention-for-sail`、`FlashMLA-for-sail`、`flash-linear-attention-for-sail`；再读对应性能博客。
- MoE 与路由算子：先看 `flashinfer-for-sail`、`actlize`、`DeepGEMM-for-sail`；再读 FusedMoE、Triton 和 MoE 量化文档。
- 集合通信算子：先看 `pccl`、`pccl-tests`、`gloo-for-sail`；不得自行假设集合通信语义。
- Triton 与 TileLang 算子：先看对应官方编译器仓库和能力文档。
- 基础编译与调试问题：先读 HGGC、TIX、RDC、预定义宏和示例程序。

### 核函数方案必须包含

- Grid、Block、Warp 映射；
- 数据从 global 到 TSM/shared 到寄存器的路径；
- AIU/async copy、stage 数和 buffer 生命周期；
- Tensor Cell/MMA 的 shape、dtype 和 accumulator 预算；
- 寄存器、TSM/shared memory 和占用率预算；
- barrier、wait、arrive 和 producer-consumer 拓扑；
- 架构条件、shape 分流和 fallback；
- 可能的 bank conflict、group conflict 和尾波。

## 第五步：实现与编译

- 优先沿用官方仓库的目录、构建方式、命名和测试约定。
- 架构相关实现必须通过 `__HGGC_ARCH__` 等条件编译隔离。
- `ppu001` 使用 `tc01` 能力；`ppu0015` 使用 `tc02` 能力。FP8/FP4 Tensor Cell 路径不得默认在 `ppu001` 上启用。
- 多翻译单元设备代码需要 RDC 时，统一使用 `-rdc=true`/`-dc`；非单文件工程通常评估 `-dlto`。
- 不在主机和设备之间共享由 `__HGGC_ARCH__` 推导出的类型、`sizeof`、函数签名或启动配置。
- 保存完整编译命令、编译器版本、架构参数、寄存器限制和告警。

## 第六步：正确性与确定性

至少覆盖：

- tiny、typical、tail shape；
- 对齐和非对齐 layout；
- 空输入、边界值、NaN/Inf 和饱和行为；
- 目标 dtype 与混合精度容差；
- 编译选项变化前后的数值差异；
- 重复运行确定性；
- 多设备场景的 Rank/group 组合；
- 目标架构上的实际执行。

对索引、MoE、Atomic、collective 或并发任务，把“结果正确”和“输出确定性”分成两个 gate。

## 第七步：Asight 工具链性能分析

### 系统级性能分析

如果需要进行系统级性能分析，使用 Asight Systems（asys）工具；如果需要进行算子级性能分析，直接使用 Asight Compute（acu）工具。

使用 asys 工具采集时优先限制范围：

```bash
asys profile -t hggc --delay <预热秒数> --duration <采集秒数> \
  --output <report_name> <command>
```

需要按阶段采集时，使用 Profiler API、HGTX/NVTX range、进程/Rank/PPU 过滤、报告轮转或大小上限。已有大报告可用 `Open Advanced` 只加载目标区间和事件类型。

统计与比较：

```bash
asys stats -r hggc_ppu_kern_sum,ppu_op_type_sum <report>
asys stats -r ppu_time_util:threshold=60:chunks=80 <report>
asys compare <baseline_report> <candidate_report>
asys aggregate <report1> <report2> <report3>
```

重点观察：

- 最长 kernel、算子类型占比和端到端时延；
- TFU、VALU、带宽和 occupancy 等资源使用；
- stall 分类、依赖、同步和 not-selected；
- TSM bank/group conflict；
- VREG、stack/local memory 和 spill；
- 通信不同步、最晚启动 Rank 和 group 差异。

### 核函数（kernel）级性能分析

- 优先使用官方的 acu 工具采集并分析具体 kernel 性能。
- 优先使用 acu-report-skill 这个 skill 进行 kernel 性能分析（如果已安装这个 skill）。
- 将指标映射到瓶颈类别：compute-bound、memory-bound、latency-bound、register/occupancy-bound、communication/synchronization-bound。
- 如果当前工具无法证明瓶颈，先补采集和基线，不要凭经验连续改 tile。

## 第八步：优化循环

每次只提出一个可证伪假设：

1. 固定基线和代表工作负载。
2. 用性能分析结果指出限制资源。
3. 选择一个优化手段。
4. 保存候选源码后再修改。
5. 重新编译、运行正确性测试、复测同一工作负载。
6. 查看瓶颈是否移动，而不只看耗时。
7. 记录收益、代价、适用 shape 和回滚点。

### 优化优先级

1. 算法和数据结构：减少无效计算、无效 padding、无效重排或通信。
2. 数据移动：提高 reuse，减少 global/local 往返，选择 AIU/async copy，优化 TSM staging。
3. 执行配置：block/grid、warp 映射、occupancy、TLP/ILP、register cap。
4. 指令：Tensor Cell、native math、宽读写、原子和控制流。
5. 通信：collective 选择、拓扑、group、同步和计算通信重叠。

### 真武架构专用

- AIU 与异步张量搬运：适合连续多维块和 global 到 TSM 的搬运；gather 或非连续 M/N 方向需确认对齐限制。
- warp interleaving：当一个 warp 在 promotion、同步或依赖中停顿时，让另一 warp 填满 Tensor Cell 空档。
- TSM 冲突：区分 bank conflict 与 group conflict；M890 为 4 个 group，ppu001 为 2 个 group，具体布局以 TIX 与 HGGC 文档为准。
- Tensor Cell：优先矩阵运算；`tc02` 可用于 FP8 和 MXFP4，但必须确认数据类型、形状和架构支持。
- 快慢路径分流：根据真实工作负载分档，而不是为单个异常热点污染默认路径。
- 集合通信：先复用 PCCL；用 `pccl-tests` 验证，再分析通信组对齐、Rank 差异和计算通信重叠。

## 第九步：晋升方案与交付

候选的算子开发或优化的晋升方案必须满足：

- 目标架构执行通过；
- 正确性和边界测试通过；
- 规定重复次数下结果稳定；
- 代表 workload 不出现不可接受回退；
- 主指标相较同轮 baseline 改善；
- 性能收益不是由更宽松的精度、错误计时或缓存热状态造成；
- 源码、测试、profile、命令和结果已归档。

建议目录：

```text
optimization/<operator>/
├── evidence.md
├── plan.md
├── baseline/
├── candidates/
├── profiles/
├── reports/
└── best/
```

每轮报告记录：

| 项目 | 内容 |
| --- | --- |
| 硬件/SDK | 产品、架构、SDK、驱动 |
| Workload | shape、dtype、layout、通信规模 |
| Baseline | 中位数、P95、方差 |
| Candidate | 改动、指标、正确性 |
| 瓶颈证据 | 主要指标和章节 |
| 决策 | promote / archive-only / rollback |
| 来源 | 官方 repo commit、文档路径 |

如果候选只在单个 shape 上获胜，可保留为 shape-specialized 路径，但必须提供廉价 dispatch 和稳定 fallback；在全量回归前不能替换默认路径。

## 本地文档快速索引

完整摘要见[官方资料目录](reference/catalog.md)，机器索引见[manifest.json](reference/manifest.json)。下列路径均相对本 SKILL 所在目录。

### 硬件与编程基础

| 主题 | 关键内容 | 必读文档 |
| --- | --- | --- |
| 平头哥真武 M890 架构白皮书 | 10 个计算引擎、39 个计算单元、TSM、L1/L2/LLC、AIU、TFU、PCIe 5.0、ICN、SAIL 软件栈 | [阅读原文](reference/architecture/m890-architecture-whitepaper.md) |
| T-Head SAIL HGGC 编程指南 | Host 与 Device、Grid/Block/Warp、SIMT、内存、流与事件、图、协作组、异步屏障 | [阅读原文](reference/programming-guide/01-hggc-programming-guide.md) |
| T-Head SAIL TIX 编程指南 | ppu001 与 ppu0015、`tc01` 与 `tc02`、特殊寄存器、访存、同步、矩阵指令 | [阅读原文](reference/programming-guide/02-tix-programming-guide.md) |
| T-Head SAIL HGGC 示例程序 | CMake、ppu_10 与 ppu_15、AIU、Tensor Cell GEMM、图、`test_args.json` | [阅读原文](reference/programming-guide/03-hggc-samples.md) |
| T-Head SAIL HGGC 算子性能优化实战指南 | block 与 grid、occupancy、访存、group conflict、指令、寄存器溢出、Asight | [阅读原文](reference/programming-guide/04-hggc-kernel-optimization.md) |

### 算子实现与优化实践参考

| 原文主题 | 关键内容 | 必读文档 |
| --- | --- | --- |
| 真武 M890 上 FP8 blockwise GEMM 的实现与优化 | K128 与 K32 promotion、双累加器、warp interleaving、scale 搬运 | [阅读原文](reference/blog/2026-11-16-fp8-blockwise-gemm.md) |
| 扬长避短：W4A8 在真武 M890 上的实现详解 | MXFP4 权重、E4M3 激活、FP4 单位阵反量化、E4M3×E5M2 | [阅读原文](reference/blog/2026-10-22-w4a8-m890.md) |
| DeepGemm (PPU) W4A16 MXFP4 GEMM 实现详解—— 寄存器内反量化的两条路线 | VALU 与 FP4 MMA 两条反量化路线、指令成本与边界 | [阅读原文](reference/blog/2026-09-29-deepgemm-w4a16-mxfp4.md) |
| BF16 DenseGemm Memory Bound 优化：把 tile 自由度花在「凑整轮」上 | memory-bound、wave 均衡、persistent、NoTsm、WarpOnK | [阅读原文](reference/blog/2026-09-28-bf16-densegemm-memory-bound.md) |
| FP8 Flash Attention 性能优化简介 | V 转置困境、寄存器转置、V 直读、epilogue layout | [阅读原文](reference/blog/2026-09-29-fp8-flash-attention.md) |
| FlashMLA2 的 Warp Interleave：从双 Warpgroup 到 WI + Cross cut | 双 Warpgroup、warp interleave、共同 softmax、cross-cut、三缓冲 | [阅读原文](reference/blog/2026-09-29-flashmla2-warp-interleave.md) |
| group conflict 在 FlashMLA 上的优化 | TSM 编址、group hash、PU 仲裁、swizzle 和冲突消除 | [阅读原文](reference/blog/2026-09-20-flashmla-group-conflict.md) |
| 从 Nopad 到 Fused：TP 场景下的 PPU FusedMoE 设计与实践 | Nopad、Hybrid、索引映射、WarpOrdered、4-Kernel 确定性 | [阅读原文](reference/blog/2026-09-29-fused-moe-hybrid.md) |
| PPU Triton 编程指南 | `tl.aiu_load`、block pointer、descriptor、GEMM、FA 与 MoE tiling | [阅读原文](reference/blog/2026-09-29-ppu-triton-guide.md) |

### 编译与构建实践参考

| 主题 | 关键内容 | 必读文档 |
| --- | --- | --- |
| hgcc 预定义宏使用介绍 | `__HGGCCC__`、`__HGGC_ARCH__`、版本、RDC、Debug 宏和静默风险 | [阅读原文](reference/blog/2026-09-21-hgcc-predefined-macros.md) |
| hgcc 分离编译（RDC 模式）使用介绍 | `-rdc`、`-dlto`、跨翻译单元、设备静态库、链接错误 | [阅读原文](reference/blog/2026-09-21-hgcc-rdc.md) |
| 性能持平，编译提速 7.7×：从 CuTe 到 CuTe Free 的一次编译时间优化 | 模板实例化与中间表示成本、逐层降级、产物等价验证 | [阅读原文](reference/blog/2026-09-29-cute-to-cute-free.md) |

### 性能采集、统计与智能体分析

| 主题 | 关键内容 | 必读文档 |
| --- | --- | --- |
| 用 Asight Systems 分析大型 asys 报告：按需加载、过滤时间线与区间统计 | Open Advanced、时间线过滤、Group View、区间统计 | [阅读原文](reference/blog/2026-11-05-asight-large-report.md) |
| 用 asys 统计系统高效分析性能报告 | `stats`、`compare`、`aggregate`、通信对齐和多报告汇总 | [阅读原文](reference/blog/2026-09-30-asys-stats.md) |
| 如何控制 asys 跟踪采集的报告大小 | Profiler API、HGTX 与 NVTX、Rank 过滤、轮转、Attach | [阅读原文](reference/blog/2026-09-29-asys-report-size.md) |
| Asight Systems MCP 使用实践：通过 AI agent 分析 asys 报告 | Asight MCP、规则选择、跨平台对比、Insight、ACU 命令 | [阅读原文](reference/blog/2026-09-29-asight-mcp-agent.md) |

## 完成标准

在结束任务前确认：

- [ ] 任务卡和代表工作负载已明确。
- [ ] t-head 官方仓库与本地文档已检索，并记录仓库版本和文件路径。
- [ ] 当前最佳基线可复现。
- [ ] 核函数方案包含数据流、同步、资源和架构分支。
- [ ] 编译与正确性检查门槛已通过。
- [ ] Asight 性能分析支持当前瓶颈判断。
- [ ] 优化结果经过同轮同负载基线和重复测量。
- [ ] 未支持或退化场景已明确回退路径。
- [ ] 交付包含源码、测试、命令、性能分析记录、结果、来源和回滚点。
