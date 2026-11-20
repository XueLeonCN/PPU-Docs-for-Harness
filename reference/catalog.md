# 真武 PPU 文档目录与详细摘要

本目录面向 Agent 检索。每篇文档先给出关键事实摘要，再列出原文中的具体机制、参数、结论和适用问题。链接指向本地来源正文；实现和事实仍以来源原文为准。

## 1. 硬件与编程基础

M890 芯片架构、HGGC 和 TIX 指令语义相关内容。

### [平头哥真武 M890 架构白皮书](architecture/m890-architecture-whitepaper.md)

**官方来源**：官方架构白皮书；2026-09-16；版本 T-Head SAIL SDK v2.2.0；平头哥开发者网站。

**文档总结**

这是 M890 的硬件与软件栈总览。芯片共 10 个 CE、39 个 CU，支持标量、向量和张量流水，以及 FP8、FP4（含 microscale）、INT8/16/32、FP16、BF16、FP32 等数据类型；缓存为 CU 内 L1、CE 内共享 L2、全芯片共享 LLC，TSM 总计 9.75MiB。AIU 可自动完成多维张量的地址计算和异步搬运，文中给出 GEMM 中 TFU 利用率可达 95% 的测试结果。

**关键内容**

- M890 采用 10 个 CE、39 个 CU 的计算组织；每个 CU 具有 L1、TSM、标量/向量寄存器和 VALU、TFU、SALU、SFU、LSU 等功能单元。
- TSM 每 CU 为 256KiB，全芯片共 9.75MiB；L1、CE 内共享 L2 与全芯片共享 LLC 共同形成三级缓存，用于过滤远端内存访问。
- AIU 通过一条指令读取多维张量并完成地址计算、布局转换和 global-to-TSM 搬运，可与 TFU 计算组成软件流水。
- 芯片通过 PCIe 5.0 x16 连接主机；提供 4 个视频解码器、2 个编码器和 4 个独立 JPEG 解码器，并列出 H.264、H.265、AV1、VP9 等格式及吞吐。
- ICN 有 8 个端口，每端口 100GB/s 双向带宽，整芯片合计 800GB/s；支持 OAM8 直连环网、ICN64 单级 Switch 和两级 Switch 扩展到 1024 卡。
- T-HEAD SAIL 栈包含固件、KMD、用户驱动/运行时、hgcc、计算与通信库、视频 SDK、SMI、Asight、PPU-GDB 和推理引擎。

**适用问题**

- 建立 M890 的 CE/CU、缓存、TSM、AIU、TFU、视频和 ICN 硬件心智模型。
- 判断某个算子优化是否依赖 M890 特有资源，例如 39 CU、9.75MiB TSM、FP8/FP4 Tensor Cell 或 ICN 拓扑。
- 为 Harness 建立硬件能力查询、资源预算和架构 feature gate。
- 查找 M890 软件栈组件、库和专业工具的职责边界。
- 区分“架构白皮书明确事实”与后续博客中的特定实现经验。

**检索标签**：`M890`、`architecture`、`GPGPU`、`CE`、`CU`、`TFU`、`AIU`、`TSM`、`cache`、`PCIe`、`video`、`ICN`、`SAIL`。

### [T-Head SAIL HGGC 编程指南](programming-guide/01-hggc-programming-guide.md)

**官方来源**：官方编程指南；版本 T-Head SAIL SDK v2.2.1；平头哥开发者网站。

**文档总结**

该指南给出 HGGC 从异构计算模型到高级 PPU 特性的完整编程方法。核心模型是 CPU 上的 host 代码通过异步 API 管理 PPU device 内存并启动 kernel；线程按 Grid、Thread Block、Warp、Thread 组织，warp 固定围绕 32 线程以 SIMT 执行，线程块调度顺序不可依赖。文档进一步覆盖 runtime/driver API、stream/event、graph、多设备 P2P、统一内存、协作组、流有序分配器、延迟加载和异步屏障/流水线。

**关键内容**

- HGGC 程序从 host 开始执行，host 与 device 可并发；数据依赖通过流、事件、设备同步或流水线原语表达。
- 线程块被整体调度到一个 CU，块内可通过 TSM/shared memory 和同步协作；不同线程块之间没有执行顺序保证，不能依赖未同步结果。
- SAIL 使用 `ppu_10`、`ppu_15` 等 ARCH，`hgbin` 面向单一架构，`hgfatbin` 可携带多个架构的代码以支持混合编译。
- 运行时 API 建立在驱动 API 之上，提供隐式初始化、context/module 管理；驱动 API 暴露更低层控制，二者可互操作。
- 异步章节覆盖阻塞、轮询、回调三类同步，以及 stream/event、从 stream 到 graph、graph 生命周期/内存节点/条件执行等用法。
- PPU 专项能力包括多设备 P2P、流有序分配器、cooperative groups、延迟加载、`hggc::barrier` 异步屏障和 pipeline 生命周期管理。

**适用问题**

- 从零编写 HGGC/HGGC C++ kernel、host 启动代码以及内存/流/事件管理逻辑。
- 确认 Grid/Block/Warp 的语义、同步边界、共享存储作用域和 occupancy 约束。
- 将多 kernel 执行流改造成 HGGC Graph，或分析 graph 节点、条件执行和内存节点。
- 实现 producer-consumer 流水、异步屏障、协作组归约或多设备通信。
- 为自动代码生成器提供 API、线程模型和生命周期约束。

**检索标签**：`HGGC`、`host-device`、`grid`、`thread-block`、`warp`、`SIMT`、`memory`、`stream`、`event`、`graph`、`multi-device`、`cluster`、`barrier`、`pipeline`。

### [T-Head SAIL TIX 编程指南](programming-guide/02-tix-programming-guide.md)

**官方来源**：官方编程指南；版本 T-Head SAIL SDK v2.2.1；平头哥开发者网站。

**文档总结**

TIX 是位于高级语言和物理 ISA 之间的 PPU 虚拟指令集，可通过 C/C++ inline asm 使用。文档明确 `ppu001` 对应 810、610、810E、610E，`ppu0015` 对应 M890；两代同为 32 线程 warp、每 CU 最多 2048 线程、每 block 最多 1024 线程、256KiB 共享内存和 64K VREG/CU，但 Tensor Cell、共享内存分组、AIU 和预取能力不同。正文逐类给出标量、数学、位操作、访存、同步和矩阵指令的语法、操作数、平台支持与语义。

**关键内容**

- `ppu001` 使用 `tc01` Tensor Cell，支持 FP16、BF16、TF32、F32、INT8；共享内存为 2 组×128B；AIU 为基础 tile。
- `ppu0015` 使用 `tc02`，新增 FP8 E4M3/E5M2 和 MXFP4 E2M1；共享内存为 4 组×1024B；AIU 新增 2D tile、5D im2col 和 128B/256B 预取提示。
- 线程由 `%tid`、Warp `%warpid`、Block `%blkid`、Grid `%gridid` 标识；共享存储可由块内线程同步访问，Grid 级没有内置屏障。
- 存储空间包括寄存器、shared、global、constant 和 local；局部数组或寄存器溢出会进入 local memory。
- 指令章节覆盖算术、比较、超越函数、位操作、load/store、转换、异步拷贝、barrier/warp 通信、原子和 Tensor Cell MMA。
- inline TIX 使用 `asm()` 与输入/输出/读写约束，文档提供 global load、标量 add、global store 的完整 vector-add 示例。

**适用问题**

- 编写、审查或自动生成 inline TIX 指令，确认操作数类型、约束和平台支持。
- 为 810E（ppu001）与 M890（ppu0015）生成不同 Tensor Cell、AIU 或预取路径。
- 查询 special register、线程/块坐标、内存空间限定符和同步/原子语义。
- 从低层指令角度解释 kernel 的访存、冲突、矩阵乘或异步搬运行为。
- 在 HGGC 示例或性能博客中追踪具体 TIX 指令的含义。

**检索标签**：`TIX`、`inline-assembly`、`instruction-set`、`ppu001`、`ppu0015`、`special-register`、`vector`、`tensor-cell`、`memory-instruction`、`synchronization`。

### [T-Head SAIL HGGC 示例程序](programming-guide/03-hggc-samples.md)

**官方来源**：官方编程指南；版本 T-Head SAIL SDK v2.2.1；平头哥开发者网站。

**文档总结**

这是 SAIL 2.2.1 HGGC 示例程序的构建目录和导读。示例使用 CMake（要求 3.20+），默认同时生成 `ppu_10` 与 `ppu_15`，也可通过 `CMAKE_HG_ARCHITECTURES` 只构建单一架构。内容按入门、工具、算法、HGGC 特性、计算库和性能测试分类，包含 graph、atomic、stream、统一内存、多设备、AIU、Tensor Cell GEMM、acBLAS/acDNN/acFFT 等可运行样例；附带的 `run_tests.py` 是快速完整性检查工具，并非完整验证或性能基准套件。

**关键内容**

- 仓库为 `github.com/t-head/hggc-samples`；CMake 默认构建 ppu001（`ppu_10`）和 ppu0015（`ppu_15`）两套架构。
- SAIL 2.2.1 新增 `acdnn_softmax_bf16`，并更新 atomic、graph memory、AIU 和各精度 Tensor Cell GEMM 示例。
- 入门示例覆盖 assert、event timing、atomic、clock、cooperative sync、模板 kernel、多设备、占用率、统一内存和向量加法。
- 算法与特性示例覆盖 convolution、histogram、scan、spmv、sort、warp redux、AIU 异步拷贝、BF16/INT8/TF32 GEMM、graph 和批量 D2D。
- `run_tests.py` 支持 `--dir`、`--config`、`--no-config`、`--output`、`--parallel`、`--arch`；会读取 `test_args.json`、`ppu-smi -q` 并跳过架构或设备数不满足的示例。
- 官方明确示例不覆盖全部边界条件、Runtime/Driver API，也不作为正式性能基准。

**适用问题**

- 为 GEMM、GEMV、图、原子、协作组、AIU 或多设备任务寻找最小可运行代码模板。
- 搭建 ppu001/ppu0015 双架构构建与快速回归。
- 查找某个 HGGC 特性的实际调用方式和 `test_args.json` 配置格式。
- 为 Harness 的代码生成样本库、编译矩阵和 smoke test 选择基线。
- 判断某项能力是否有官方示例证据，同时避免把示例结果误当成完整验证或 benchmark。

**检索标签**：`samples`、`build`、`GEMM`、`GEMV`、`convolution`、`sparse`、`graph`、`cooperative-group`、`AIU`、`tensor-cell`、`benchmark`。

### [T-Head SAIL HGGC 算子性能优化实战指南](programming-guide/04-hggc-kernel-optimization.md)

**官方来源**：官方编程指南；版本 T-Head SAIL SDK v2.2.1；平头哥开发者网站。

**文档总结**

该指南把 kernel 内优化分为资源利用率、内存吞吐、指令吞吐、寄存器/stack 溢出和 Asight 定位五部分。总原则是先以 profiler 判断访存、计算或占用率瓶颈，再按“算法/数据结构 → 访存 → 执行配置 → 指令级”的优先级选择手段。文中的 810E 参考配置为 64 CU、每 CU 64 warp/2048 线程、128K VREG、单 warp 最多 255 VREG、256KiB shared memory；同时强调真实值必须从设备属性查询。

**关键内容**

- block 线程数应为 32 的倍数并优先取 2 的幂；grid 应接近 `CU 数 × 每 CU 常驻 block 数` 的整数倍以减少尾波。
- TLP 由 occupancy 驱动，ILP 让单 warp 有足够独立指令；二者都服务于隐藏访存、依赖和同步延迟。
- 访存优化包括合并访问、int4/float4 宽读写、shared memory 中转、padding 消除 bank conflict、缓存策略与 AIU/async copy 流水。
- M890 之外，文档明确 ppu001 的 shared memory 为 2 个 128B 组，ppu0015 为 4 个 1024B 组；访问应同时避免 bank 与 group conflict。
- 指令部分建议优先 Tensor Cell、float/int、硬件原子和一致控制流，并区分精度/快速数学选项对结果的影响。
- 寄存器溢出会进入片外 local memory；可用 `__launch_bounds__`、`__maxnreg__`、`--maxrregcount`、资源报告与 spill 告警约束和检测。
- 性能采集推荐 `-lineinfo` 而非 `-G`，因为 `-G` 会阻止部分优化，不能代表发布版本。

**适用问题**

- 建立从 profile 到瓶颈分类、改写、正确性复测的 kernel 优化流程。
- 调 grid/block、occupancy、TLP/ILP、register cap 和 launch bounds。
- 排查 global access、shared bank/group conflict、cache 和 AIU 搬运问题。
- 判断何时应换 Tensor Cell、native math、宽指令、原子或 warp-uniform 控制流。
- 检测 register spill、local memory 和 stack，并生成对应编译诊断选项。
- 为 810E/M890 建立分别可查的设备属性与优化检查表。

**检索标签**：`kernel-optimization`、`occupancy`、`TLP`、`ILP`、`launch-bounds`、`coalesced-access`、`bank-conflict`、`group-conflict`、`cache-policy`、`AIU`、`tensor-core`、`register-spill`、`Asight`。

## 2. GEMM、Attention、MoE 与量化

具体算子的分析与优化相关博客。

### [真武 M890 上 FP8 blockwise GEMM 的实现与优化](blog/2026-11-16-fp8-blockwise-gemm.md)

**官方来源**：官方技术博客；2026-11-16；平头哥开发者网站。

**文档总结**

文章面向 M890（ppu0015）实现 FP8 blockwise GEMM：A 为 M×K 的 E4M3、B 为 K×N 的 E4M3，激活使用 `1×128`、权重使用 `128×128` 的 FP32 scale，输出 BF16。与普通 GEMM 相比，每 K128 都必须把四个 K32 的 raw MMA 结果用对应 scale promotion 到 final accumulator，因此需要两份累加器，寄存器压力把 warp tile 从 `64×64` 压到 `48×64`，算术强度由 256 降到 219。文章的核心优化是 warp interleaving，把 promotion 藏进另一 warp 的 MMA 窗口，并继续压缩 scale 搬运、分支和无用屏障。

**关键内容**

- M890 最终采用 `192×256` CTA、BK128、4-stage，共享存储（SMEM）约 232,704B；增大 CTA 会被寄存器或 SMEM 卡住。
- 单个 warp 内 promotion 与 MMA 对同一 raw accumulator 存在串行依赖，单纯增加 occupy 或缩小 tile 不足以消除 Tensor Cell 空档。
- warp interleaving 让同一 WE 上的两个 warp 相位错开，一组做 MMA 时另一组做 promotion，从而填充 tensor pipe 空档。
- 其他收益来自 MMA 蛇形寄存器复用、linear AIU scale 搬运、把角色判断移出稳态循环、放松 barrier 内存序及调整 global copy 发射位置。
- 基准 shape 的中位耗时从 1,752.356µs 降到 1,330.953µs，TFU 从 53.94% 提升到 71.02%；MMA 数不变，非数学指令从 142.6 降到 122.1 条/(warp·K128)。
- 文章同时给出 roofline、L2 命中率、指令周期和 Asight stall 结构，强调理论带宽条件不等于实测满供。

**适用问题**

- 设计或优化 FP8 blockwise/DeepSeek-V3 风格 GEMM，尤其是 K128 scale 与 K32 MMA 的 promotion 流水。
- 分析双 accumulator、寄存器上限、warp tile 和 CTA 选择的相互约束。
- 用 warp interleaving 掩盖串行 promotion，或评估同类跨 warp 错峰调度。
- 解读 TFU、tensor pipe stall、not_selected、pipe_busy 和 compute dependency 指标。
- 生成 tile/SMEM/寄存器预算模板和针对具体 problem shape 的性能实验计划。

**检索标签**：`FP8`、`blockwise`、`GEMM`、`roofline`、`tile`、`register-pressure`、`warp-interleaving`、`AIU`、`TSM`、`promotion`、`Asight`。

### [扬长避短：W4A8 在真武 M890 上的实现详解](blog/2026-10-22-w4a8-m890.md)

**官方来源**：官方技术博客；2026-10-22；平头哥开发者网站。

**文档总结**

文章提出 M890 上的 W4A8 路线：保留原生 MXFP4 E2M1 权重和每组 32 元素的 E8M0 scale，把激活量化为 per-token E4M3，在寄存器中用 FP4 MMA 与单位阵相乘完成反量化，将权重恢复为 E5M2，再执行 E4M3×E5M2 混合 FP8 MMA。该路线复用 W4A16 的 FP4 反量化，但把主 GEMM 的 MMA 条数减半，并通过列置换、成对转换和打包减少 VALU 指令。三组 prefill MoE workload 的 up+down GEMM 合计加速比为 1.75–1.97×。

**关键内容**

- 输入权重为 MXFP4：E2M1 数据加 E8M0 group scale；激活为 E4M3 per-token；最终主 GEMM 使用 E5M2 权重与 E4M3 激活。
- 用 FP4 MMA 的硬件 scale 元数据完成 group scale 乘入，单位阵列置换让反量化结果落入目标线程/寄存器。
- E5M2 的较大指数范围用于承接反量化结果，减少二级 scale 的使用。
- FP8 MMA 的 `m16n16k32` 相比 BF16 `m16n16k16` 将相同 K 的 GEMM MMA 条数减半。
- 反量化固定成本随 warp M 增大摊薄，因此大 M、计算受限的 prefill 最受益；decode 也测试，但预填场景优势更明确。
- 六个 MoE workload 覆盖 Kimi、DeepSeek、MiMo 的 prefill/decode 和不同 TP/EP 配置。

**适用问题**

- 设计 M890 上的 MXFP4 权重、W4A8 混合精度 GEMM。
- 比较 W4A8 与 W4A16、INT4 在 prefill 和 decode 下的指令与内存成本。
- 实现 FP4 单位阵反量化、E4M3×E5M2 MMA 及寄存器布局转换。
- 为 MoE gate/up/down GEMM 选择量化格式和 shape 分流。
- 生成 W4A8 kernel 的数值范围、scale 和精度验证策略。

**检索标签**：`W4A8`、`FP4`、`E4M3`、`E5M2`、`mixed-precision`、`MMA`、`dequantization`、`scale`、`MoE`、`M890`。

### [FlashMLA2 的 Warp Interleave：从双 Warpgroup 到 WI + Cross cut](blog/2026-09-29-flashmla2-warp-interleave.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章拆解 FlashMLA2 在 PPU 上使用两个 8-warp warpgroup 的一般性设计。两个 WG 通过 Warp-Interleave 把 QK、softmax、PV 的阶段错开，共享在线 softmax 的行最大值与分母；同时用 cross-cut 在组内重新切分 QK/PV 的 M/N warp 分工。对 BF16、d512、M64 路径，PV 布局从 `(4,2)` 改为 `(2,4)`，每个 WG、每 K16 的逻辑读取估算由 P+V 36KiB 降到 24KiB，以减少大 V 的重复读取。

**关键内容**

- WI 不消除 QK→softmax→PV 依赖，而是让另一个已就绪的 WG 在等待窗口继续工作。
- 两个 WG 各 8 warp、共 16 warp；共享 softmax 通过四个跨组交接点维持正确性。
- 交叉三缓冲需分别管理“数据已到”和“buffer 可覆盖”两类语义；`awbar`/wait 不能把二者混为一谈。
- Cross-cut 的 36→24KiB 是逻辑读取请求估算，不是实测 TSM 流量或加速倍数。
- 整体 Tensor Cell 利用率从 30% 多提升到 50% 多；WI 与 cross-cut 的收益未单独拆分。

**适用问题**

- 设计 MLA/Attention kernel 的双 WG 错峰执行和共同 softmax。
- 分析 TSM 三缓冲、barrier、异步搬运与计算的重叠关系。
- 优化 QK/PV 的 warp 二维分工与 V 操作数复用。
- 用 Asight timeline 区分“通道空闲”“WG 等待”“数据未到”和资源竞争。
- 为 FlashAttention/MLA Harness 提取同步状态机和 buffer 生命周期模型。

**检索标签**：`FlashMLA2`、`MLA`、`warp-group`、`warp-interleaving`、`cross-cut`、`softmax`、`pipeline`、`triple-buffer`、`TSM`、`awbar`。

### [FP8 Flash Attention 性能优化简介](blog/2026-09-29-fp8-flash-attention.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章聚焦 FP8 Flash Attention 前向 PV GEMM 中 V 操作数必须转置的问题，因为 `ldmatrix` 的 b8 转置加载不直接满足 MMA B-fragment。方案从 TSM 显式转置，演进到 warp 内 `shfl` 寄存器转置，最终采用 V 直读：用 `ldmatrix.trans.b16` 读取原始 V，利用归约轴天然正确、只有输出轴偶奇错位的特点，每 n-tile 仅用 4 条 `byte_perm` 装配 B fragment，并在主循环外用寄存器交换和 epilogue CLayout 修正输出布局。

**关键内容**

- TSM 显式转置需要 V 读 2 次、写 1 次，并增加两次 `__syncthreads`。
- 寄存器版避免 TSM 往返，但需要大量 shfl/sel/byte_perm，且仍受寄存器压力。
- V 直读实现 V 只读一次、零 shfl、零 sel，主循环内每 n-tile 只有 4 次 byte_perm。
- 输出 πn 错位拆成主循环外每 kernel 一次 `permute_output_fp8` 和零数据搬运的 epilogue CLayout 重解释。
- 相对显式转置，V 直读的耗时比在 prefill 为 0.653–0.891、decode 为 0.851–0.892，均小于 1。

**适用问题**

- 实现 FP8 Flash Attention 的 PV GEMM 与 V 操作数布局。
- 优化主循环中的跨 lane 转置、TSM 往返和寄存器压力。
- 分析 b16.trans 与 b8 MMA fragment 的布局差异。
- 将转置问题改写成 epilogue/layout 重解释问题。
- 对不同 head dimension 的 Attention kernel 设计性能 A/B。

**检索标签**：`FP8`、`Flash-Attention`、`FlashAttention-3`、`PV-GEMM`、`transpose`、`ldmatrix`、`byte-permute`、`MMA`、`TSM`。

### [DeepGemm (PPU) W4A16 MXFP4 GEMM 实现详解—— 寄存器内反量化的两条路线](blog/2026-09-29-deepgemm-w4a16-mxfp4.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章比较 DeepGemm 的两种 MXFP4 权重量化 GEMM 路线：VALU 版在寄存器中用位操作把 FP4+E8M0 展开为 BF16，两代 PPU 通用；MMA 版只在 ppu0015 上，把权重作为 FP4 MMA 的 A 操作数、与常驻寄存器单位阵相乘，使数值展开与 scale 乘入在同一条带块缩放 MMA 中完成。按每个 int32 的 8 个 FP4 计算，VALU 反量化约 15 条 IALU + 8 条 FALU、约 46 cycles；MMA 版为 1 次 FP4 MMA + 4 条 pcnvt，保守约 16 cycles。

**关键内容**

- MMA 版权重保持 uint8 自然序、零权重重排；scale 预排后 `ldmatrix.x1` 直接形成 meta，加载即就绪。
- `W × I` 同时完成 FP4→FP32 数值展开、E8M0 scale 乘入和格式转换；4 条 `pcnvt` 转 BF16 时兼做 fragment 转置。
- 代价是把压力移到 Tensor Cell、常驻 identity 增加 VREG 压力，并要求 BlockK≥128。
- 实测 MMA 版相对基线在 decode 快 1.43×、prefill 快 1.26×；VALU FP4 在同测试中比 INT4 慢约 1.25–1.34×。
- MMA 版要求 N/K 为 64 的倍数，仅适用于 `__HGGC_ARCH__ >= 150`；VALU 版是全平台基线。

**适用问题**

- 在 ppu001 与 ppu0015 间选择软件反量化或 FP4 MMA 反量化。
- 设计 MXFP4 权重、E8M0 scale 和 BF16 fragment 的预重排格式。
- 核算 IALU/FALU/Tensor Cell 指令成本与寄存器预算。
- 为 decode 小 M、TC 空闲场景迁移反量化工作。
- 生成双路线 feature gate、正确性测试和性能基准。

**检索标签**：`DeepGemm`、`W4A16`、`MXFP4`、`BF16`、`GEMM`、`VALU`、`MMA`、`dequantization`、`scale-layout`、`register-double-buffer`。

### [从 Nopad 到 Fused：TP 场景下的 PPU FusedMoE 设计与实践](blog/2026-09-29-fused-moe-hybrid.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章针对 TP 场景 MoE 的 token 专家重排，提出用索引替代物理搬数据。最终 Hybrid 方案让 GEMM1 按 `sorted_token_ids` gather 读取，GEMM2 继续走连续 Nopad，`ep_gather` 通过 `inv_perm` 写回原位；`moe_align_block_size` 一次生成调度描述符、排序索引、每专家行数、m_indices 和逆置换。索引构建按规模选路：numel≤16384 用单 block WarpOrdered，大 batch 用无全局原子的 4-Kernel，两条路径输出逐 bit 一致。

**关键内容**

- Hybrid 相对 Nopad 的端到端收益为 M=1 −8.0%、M=1024 −5.9%、M=4096 −13.0%、M=32768 −16.5%。
- M=4096 时辅助开销从 661.70µs 降到 425.36µs（−35.7%），是主要收益来源；大 M 的 GEMM1 gather 回退约 9%–12%，但远小于省掉的物理重排。
- WarpOrdered 使用 `__match_any_sync`、warp 前缀和、CUB BlockScan，一次 launch 生成全部索引；M=1 为 7.0µs。
- 4-Kernel 大 batch 路径 M=32768 为 92.9µs，相对原型 278.9µs 明显降低，且位级确定。
- peer_rank 数组从 64 缩到 16 后完全驻留寄存器，M=256 从 32µs 降到 9.5µs。
- 正确性覆盖 BF16/INT8/FP8、尾块和空专家等 14 个 case，14/14 通过。

**适用问题**

- 设计 MoE dispatch/align、专家路由此类索引密集 kernel。
- 在物理 token 重排、Nopad 和 Fused/Hybrid GEMM 之间做端到端取舍。
- 构建位级确定的并行前缀和、scatter 和多 kernel 路径。
- 针对小 batch / 大 batch 自动选择单 kernel 或多 kernel 算法。
- 排查寄存器数组溢出、全局原子竞争和 CUDA/Triton 风格 API 差异。

**检索标签**：`MoE`、`FusedMoE`、`Nopad`、`TP`、`HBM`、`TSM`、`indexed-GEMM`、`scheduler`、`hybrid`、`small-batch`。

### [BF16 DenseGemm Memory Bound 优化：把 tile 自由度花在「凑整轮」上](blog/2026-09-28-bf16-densegemm-memory-bound.md)

**官方来源**：官方技术博客；2026-09-28；平头哥开发者网站。

**文档总结**

文章研究 M890 上 BF16 DenseGemm 的 memory-bound 小 M 区（`DG_M≤160`）。在这一区域权重 B 的流式读取主导性能，M890 LLC 峰值约 2.664T/s；实验显示整波时不同 tile/occupancy 的 LLC 利用率差不超过 3.3pp，而分数波可拉开至 33.5pp，因此主要矛盾是 wave quantization，而非理想 tile 形状。方案用 `bn_wave_even` 把 BN 调成任意 16 的倍数以凑 `39×整数` tile，再用 persistent 轮询调度和 NoTsm epilogue 保持 CU/波内均衡。

**关键内容**

- `DefaultEpilogueNoTsm` 让 `bf16x2` 从 VREG 直接写 global，解除 BN%32 和偶数 WarpOnN 限制，允许 144、272、368 等 tile 与 9/11/17/23 个 warp。
- 深 K、小 BN 的 latency-bound 区叠加块内 WarpOnK：warp 沿 K 算部分和，在 SMEM 规约，不需要 splitK workspace 或第二个 kernel。
- WarpOnK 有四道门控：仅 memory-bound、K/BK≥6、fat tile 限制、SMEM 规约预算；并修正每 CU 最大 warp 预算为 64。
- K=1024 的 2300 UT 全量几何平均 opt/base=0.912，整体约 9% 提速；核心 memory-bound 区 0.920，过渡带 0.844。
- K=7168 的 654 UT 为 0.945；WarpOnK 适用区与大量使用 splitK 的基线约 1.004。微小 M=1 应路由专用 GEMV。
- 优化只适用于 A、B 都为 K-Major；AIU 连续方向受 64/128B 对齐约束，非 K-major 时细粒度 BN 不成立。

**适用问题**

- 优化 M890 BF16 decode/小 batch GEMM 的 wave/CU 拖尾。
- 生成 BN 连续量化、persistent scheduler、NoTsm epilogue 和 WarpOnK 的选择器。
- 决定何时使用 splitK，何时改为块内 WarpOnK。
- 为 m=1 选择 GEMV 专用路径，避免 GEMM 泛化。
- 识别 memory-bound 与 latency-bound 叠加区，并建立对应观测指标。
- 根据 layout 判断细粒度 tile 优化是否适用。

**检索标签**：`BF16`、`DenseGemm`、`memory-bound`、`tile-selection`、`wave-quantization`、`persistent-kernel`、`CU-balance`、`EpilogueNoTsm`、`WarpOnK`、`GEMV`。

### [group conflict 在 FlashMLA 上的优化](blog/2026-09-20-flashmla-group-conflict.md)

**官方来源**：官方技术博客；2026-09-20；平头哥开发者网站。

**文档总结**

文章从 M890 的 256KiB/CU TSM 结构出发：TSM 分为 4 个 group，每组 `64×1KB`，组内 32 bank；地址以 1KB interleave 映射到 group，再以 4B interleave 映射到 bank。除了传统 bank conflict，同一 warp/PU 访问分布还会产生 group conflict。文中解释 `tsm.ld.swzl`、warp 与 PU 映射、4PU-to-4TSM group 仲裁，并在 FlashMLA 的 cross-cut 和 warp-iterative 路径上通过统一 ld pattern、padding 和相邻 tile MMA 交错降低冲突。

**关键内容**

- M890 TSM 每个 1KB group block 包含 8 条 128B line；满带宽需同时利用 4 个 group 的 4×32 bank。
- bank conflict 是组内 bank 竞争，group conflict 是请求集中落到少数可并行服务端口；两者分析维度不同。
- `tsm.ld.swzl` 可改变 line/bank swizzle，但可能加重 group conflict，必须与 warp/PU 映射联合分析。
- FlashMLA 优化前后，KBlockM=64 的 TFU 由 33% 升至 36%，group conflict ratio 由 0.44 降至 0.10。
- warp-iterative KBlockM=128 的 TFU 由 42% 升至 49.5%，group conflict ratio 由 0.42 降至 0.12。
- 文章强调先看 ACU 的 group conflict ratio 和实际场景是否 memory/compute-bound，再决定是否做复杂布局改造。

**适用问题**

- 分析 M890 TSM 地址、bank 和 group 映射。
- 区分 bank conflict 与 group conflict 并选择不同修复方式。
- 优化 FlashMLA/GEMM 中 `tsm.ld.swzl` 与 MMA tile/warp 调度。
- 设计 padding、ld pattern 统一、相邻 tile 交错等实验。
- 建立 ACU conflict ratio 与 TFU/耗时联合判断规则。

**检索标签**：`FlashMLA`、`TSM`、`bank-conflict`、`group-conflict`、`group-hash`、`swizzle`、`PU-mapping`、`GEMM`、`ldmatrix`。

### [PPU Triton 编程指南](blog/2026-09-29-ppu-triton-guide.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章说明 PPU Triton 从 3.2.0 起可用 `tl.aiu_load` 把 global 数据通过 AIU 搬到 shared memory，并展示直接传指针、`make_block_ptr`、`make_tensor_descriptor` 三种调用方式。文章用 GEMM、Flash Attention 和 FusedMoE 的 tiling 实验说明：BLOCK_K 主要影响 staging/pipeline，BLOCK_M/N 还会改变 global/shared load 请求；M890 上减小 BLOCK_M 比减小 BLOCK_N 更易增加 L2 miss。文章还给出 spill 时的调参顺序和 AIU 的适用边界。

**关键内容**

- `tl.aiu_load` 支持 tensor 指针/offset/block shape 或 block pointer，tensor descriptor 在满足条件时可自动 promotion 为 AIU load。
- 发生 spill 时优先减小 `BLOCK_SIZE_K`，再考虑 `BLOCK_SIZE_N`，最后才减小 `BLOCK_SIZE_M/N`；增大 tile 时优先增大 M/N。
- Flash Attention 的 Q/K/V 连续 block 可用 AIU，文中报告 10%+ 性能提升。
- MoE 的 token gather 在 M 方向不连续，A 不能直接用 AIU；expert B 在 N/K 连续，可用 AIU，部分 kernel 报告 15%+ 提升。
- MoE 的 BLOCK_M 过大会增加 padding 的无效访存与计算，小 token 数时应接近平均每专家 token 数。

**适用问题**

- 把 PPU Triton kernel 的 global-to-shared 搬运改为 AIU。
- 为 GEMM/FA/FusedMoE 选择 block pointer、descriptor 和 tiling。
- 在 810/810E/M890 上排查 register spill 并调整 BLOCK_M/N/K。
- 识别哪些张量访问连续、哪些 gather 只能走普通异步拷贝。
- 生成 Triton 到 TIX/HGGC 的对照实现和性能实验。

**检索标签**：`Triton`、`PPU`、`tl.aiu_load`、`block-pointer`、`tensor-descriptor`、`tiling`、`GEMM`、`Flash-Attention`、`FusedMoE`、`AIU`。

## 3. 编译与构建

编译优化相关技术博客。

### [性能持平，编译提速 7.7×：从 CuTe 到 CuTe Free 的一次编译时间优化](blog/2026-09-29-cute-to-cute-free.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章记录 BF16 Dense GEMM 从 CuTe 到 CuTe-Free 的等价降级。CuTe 版把 Layout 作为编译期代数对象，产生 7,131 次模板实例化、19,596KB 前端 IR 和 797MB 编译峰值；CuTe-Free 版用 constexpr 查表、普通循环和直接 atom 调用，将冷编译从 15.5s 降到 2.02s（7.7×），峰值内存降到 147MB（5.4×），实例化降到 67 次。最终汇编指令量为 1,183 vs 1,179，2200 个 shape 扫描中主力区性能持平或略快。

**关键内容**

- 四条降级判据是把可硬件表达的布局交给指令语义、把固定映射写成 constexpr、把组合切分改成普通循环、删除不必要的 Layout 代数层。
- 性能扫描为 M 1–2048、N 512–40960、K=1024；清理抖动后剩 1763 个 clean case。
- `M≥192` 主力区间 CuTe-Free 稳定快约 2%–3%，小 M 互有胜负；每臂精度 24/24 通过。
- 收益不是靠固定 shape：模板参数化能力保留，验证需同时比较编译指标、汇编产物和精度。
- 代价是 atom 需要人工组装、用 `static_assert` 维护编译期契约，失去部分自动泛化。
- 已知正确性风险：`STAGES≥3` 且 `BLOCK_M≤32` 时可能出现非确定误差，应强制两 stage。

**适用问题**

- 定位 hgcc/CuTe kernel 的模板实例化与 IR 膨胀。
- 在保持运行时产物和性能的前提下降低 JIT 冷编译与 autotune 成本。
- 设计“先证明产物等价，再声明编译优化”的验证流程。
- 决定何时保留高层模板，何时为已收敛 kernel 写专用 atom。
- 生成编译时间、峰值内存、IR、实例化、SASS 和精度的联合回归指标。

**检索标签**：`CuTe`、`CuTe-Free`、`BF16`、`GEMM`、`compile-time`、`template-metaprogramming`、`Layout`、`IR`、`hgcc`、`trade-off`。

### [hgcc 预定义宏使用介绍](blog/2026-09-21-hgcc-predefined-macros.md)

**官方来源**：官方技术博客；2026-09-21；平头哥开发者网站。

**文档总结**

本文是 hgcc 2.2.0 预定义宏的完整使用说明。`__HGGC__` 在 HGGC/C++ 源文件编译时定义，`__HGGCCC__` 仅在 `.hg` HGGC 源文件定义；版本宏为 2/2/0；`__HGGC_ARCH__` 只在设备端编译中存在，且 `ppu_10→100`、`ppu_15→150`，虚拟架构与真实架构共用编号。文章重点说明架构宏只能出现在预处理指令中、不能把由它推导的常量用于 host/device 共享启动配置，并列出不得改变外部链接实体、头文件共享行为、类型布局和模板实例化的四条静默风险。

**关键内容**

- `__HGGCCC_RDC__` 在 `-rdc=true`、`-dc` 或单独 `-dlto` 时两端定义为 1；`__HGGCCC_DEBUG__` 由设备编译 `-G` 定义，主机调试信息另由 `-g` 控制。
- `__HGGC_ARCH__` 在 host 编译中不存在，`#if __HGGC_ARCH__ >= 150` 在 host 端会静默走 `#else`。
- 由架构宏推导的 `LAUNCH_BLOCK_SIZE` 不可用于主机 launch 配置；host/device 必须看到同一常量。
- `__launch_bounds__` 与 `__maxnreg__` 不能用于同一个 kernel。
- 架构宏不得改变跨 host/device 边界的实体存在性、`sizeof`、函数签名、模板实参或 ODR 共享行为。
- 扩展 lambda/relaxed constexpr 分别由 `--expt-extended-lambda`、`--expt-relaxed-constexpr` 开启。

**适用问题**

- 为 ppu001/ppu0015 双架构源码生成正确的条件编译。
- 检查 hgcc 版本、RDC/Debug/扩展特性与编译端。
- 排查 host/device 宏取值不一致、launch 配置越界和静默 ODR 错误。
- 从源码确认一项编译路径是否真的由目标架构启用。
- 为自动代码生成器建立禁止用 `__HGGC_ARCH__` 改变跨边界类型的规则。

**检索标签**：`hgcc`、`predefined-macros`、`__HGGC__`、`__HGGC_ARCH__`、`feature-macro`、`version-macro`、`RDC`、`DEBUG`、`conditional-compilation`。

### [hgcc 分离编译（RDC 模式）使用介绍](blog/2026-09-21-hgcc-rdc.md)

**官方来源**：官方技术博客；2026-09-21；平头哥开发者网站。

**文档总结**

文章完整说明 hgcc RDC：`-rdc=true`/`-dc` 生成可重定位设备代码，使设备函数、变量和静态库可以跨翻译单元解析；同时保留独立 device 函数，可能损失跨文件内联。`-dlto` 把优化推迟到设备链接，通常应与非单文件 RDC 一起使用。文中还给出 `.hggc_fatbin`/`.hggc_relfatbin`、EXEC/REL、符号可见性、分步 `-nodlink`/`-dlink` 链接和按错误前缀排错的方法。

**关键内容**

- 参与设备链接的目标文件必须统一启用或统一不启用 RDC；混合会导致 `lld`/`hglld` undefined symbol。
- 跨文件非 const `extern __device__` 变量在非 RDC 下只给警告并静默生成各自副本；`extern __constant__` 和函数会编译报错。
- 设备代码跨库共享应打包为 `.a`；RDC 对象需要设备链接阶段解析。
- 最坏构造场景中裸 RDC 比全程序编译慢 4.90×，加 `-dlto` 后与全程序仅差 0.04%；该数字代表内联损失上界，不是普遍开销。
- `-dlto` 隐含 RDC，可只作用于部分文件，但 LTO IR 只在同主版本编译器间兼容。
- 推荐用 `__HGGCCC_RDC__` 为依赖跨文件设备符号的文件加编译期保护。

**适用问题**

- 多文件 hgcc 工程、设备静态库和分层构建。
- 配置 `-rdc`/`-dlto` 并评估链接时间和性能代价。
- 排查 undefined symbol、架构不一致和 `__hggcRegisterLinkedBinary` 错误。
- 设计自有 CMake/Make 的分步设备链接。
- 把可能静默出错的跨 TU 代码提前转成编译失败。

**检索标签**：`hgcc`、`RDC`、`separate-compilation`、`device-link`、`dlto`、`ELF`、`cross-TU`、`device-global`、`static-library`、`troubleshooting`。

## 4. 性能采集、统计与 Agent 分析

性能分析相关技术博客。

### [用 Asight Systems 分析大型 asys 报告：按需加载、过滤时间线与区间统计](blog/2026-11-05-asight-large-report.md)

**官方来源**：官方技术博客；2026-11-05；平头哥开发者网站。

**文档总结**

本文给出大型 asys 报告从加载到统计的 GUI 操作路径。`File → Open Advanced` 可按百分比区间以及事件类型只加载部分数据；打开后可用时间线显示选项隐藏 CPU、非 HGGC 或 metrics 行，或用 Group View 只显示目标节点及其祖先和子节点。框选时间区间后，Events 搜索、统计、报告对比和多报告汇总都会收窄到该区间。

**关键内容**

- 高级打开支持按报告起止百分比加载，例如只加载 30% 至 50%。
- 取消事件类型可从源头减少加载和内存占用；隐藏 timeline 行仅影响显示，不会减少已加载数据。
- Group View 会显示目标节点、祖先和子节点，把相关调用关系集中到同一视图。
- 时间过滤同时作用于 Events、搜索、统计、compare 和 aggregate。

**适用问题**

- 分析 GB 级 asysrep，先减少加载数据再选择时间与事件类型。
- 在大量 timeline 中围绕单个 kernel、通信组或调用链做局部检查。
- 让 Agent 生成“仅统计某个迭代/区间”的可复现分析步骤。
- 区分加载优化与显示优化，避免误以为隐藏行减少了内存占用。

**检索标签**：`Asight Systems`、`asys`、`large-report`、`partial-load`、`timeline-filter`、`group-view`、`interval-stats`。

### [用 asys 统计系统高效分析性能报告](blog/2026-09-30-asys-stats.md)

**官方来源**：官方技术博客；2026-09-30；平头哥开发者网站。

**文档总结**

asys 统计系统把大规模 trace 事件按 kernel 名、算子类型或通信组聚合，输出可读表格或 CSV。命令行分为单报告 `asys stats`、双报告 `asys compare` 和多报告 `asys aggregate`，底层共用 stats 引擎。规则支持时间、设备、HGTX range 和输出列过滤；聚合多报告时，系统按通信算子结束时间用最长公共子序列匹配自动对齐节点时间。

**关键内容**

- 常用规则包括 `hggc_ppu_kern_sum`、`ppu_op_type_sum`、`interconnect_desync_trace/summary`、`ppu_end_to_end_latency_summary` 和 `ppu_time_util`。
- `ppu_time_util` 可设置 `threshold`、`chunks`、`range-mode` 等选项，合并连续低利用率区间。
- 可通过多个 `-r` 或逗号组合规则，并用 `--output` 输出 CSV；不同规则可指定不同目标。
- `asys compare` 默认生成算子、利用率和 kernel 汇总对比，详细数据写 CSV。
- 多报告 aggregate 依赖报告对齐，适合跨节点训练性能分析。

**适用问题**

- 建立单报告性能画像、优化前后 A/B 对比或多节点汇总。
- 让 Agent 选择统计规则并把结果转成结构化 CSV。
- 定位耗时最长 kernel、算子类型占比、通信不同步、端到端延迟和低利用率区间。
- 设计 Harness 的 profile 自动汇总与回归报告。
- 理解多节点报告为何需要基于通信事件对齐时间。

**检索标签**：`Asight Systems`、`asys`、`stats`、`compare`、`aggregate`、`kernel-summary`、`latency`、`utilization`、`interconnect`。

### [如何控制 asys 跟踪采集的报告大小](blog/2026-09-29-asys-report-size.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章系统整理 asys 报告体积控制。最高收益的方法是在应用内用 Profiler API 或 HGTX/NVTX range 只采集目标阶段，模型加载和初始化常占全部跟踪事件的 50%–80%；还可通过 delay/duration、交互式启停、持久应用 attach、报告轮转、最大文件尺寸、PPU/Rank/进程过滤，以及 Open Advanced 部分加载控制体积。

**关键内容**

- 事件触发可只记录指定代码段，`repeat-shutdown` 可限制重复迭代采集次数。
- `--delay` 与 `--duration` 跳过预热/初始化并限定采集时间。
- `--trace-rotation` 持续生成小报告并只保留最近若干份；`--max-report-size` 设置文件硬上限。
- 可按 PPU、Rank ID 或进程过滤，多卡场景只采集目标节点。
- `asys attach` 可附着到已运行、不可重启的服务，跳过启动阶段。
- Open Advanced 不改变磁盘文件，但可减少 GUI 实际解析和渲染的数据。

**适用问题**

- 为长时推理/训练服务设计低开销 trace 采集。
- 控制初始化阶段造成的报告膨胀，或只采集指定 decode/prefill 迭代。
- 在不可重启的线上进程上执行短时 attach。
- 为 CI 和多卡回归指定进程过滤、文件尺寸和轮转策略。
- 诊断报告过大、加载慢或内存占用高的问题。

**检索标签**：`Asight Systems`、`asys`、`trace`、`Profiler-API`、`HGTX`、`NVTX`、`report-size`、`rank-filter`、`attach`。

### [Asight Systems MCP 使用实践：通过 AI agent 分析 asys 报告](blog/2026-09-29-asight-mcp-agent.md)

**官方来源**：官方技术博客；2026-09-29；平头哥开发者网站。

**文档总结**

文章演示把 Asight Systems MCP 接入 Qoder，让 Agent 查询报告、选择统计规则、做跨报告比较、把结论写回 Timeline，并根据 kernel 记录生成 Asight Compute 命令。案例比较 M890 64 卡与 810E 128 卡的 Qwen2.5-VL VLA 预训练：M890 慢 8.8%，聚合报告显示 M890 Native 占 38.0%、810E 仅 12.0%，而 810E PCCL 占 49.8%；同参数匹配后，M890 的 elementwise kernel 平均 69.1µs，是 810E 19.0µs 的 3.6 倍，成为后续 ACU 下钻目标。

**关键内容**

- MCP 暴露报告查询、50+ 统计规则、Insight 标注和事件级下钻能力。
- 配置流程为启动 Asight GUI 内置 MCP Server，在 Qoder 添加 JSON 连接器，并用“列出分析能力”验证。
- Agent 能自动选择规则、解释统计表，并把 High/Medium/Low Insight 锚定到 Timeline 事件或全局位置。
- 跨平台算子按完整参数匹配；810E 128 卡 AllReduce `w:128` 与 M890 64 卡 `w:64` 不能直接对应。
- 案例定位到 Native/elementwise 差异，并能继续生成 Asight Compute 的 kernel 级采集命令。

**适用问题**

- 把 MCP 性能工具接入 Agent，自动完成多报告汇总和定位。
- 建立 M890 与 810E 的端到端、算子级和 kernel 级分层分析流程。
- 处理跨平台并行度/API 参数不同导致的算子匹配失败。
- 将自然语言问题转换为统计规则、时间区间、Insight 和 ACU 命令。
- 设计可持久化、可点击、可跨会话复用的性能分析结论。

**检索标签**：`Asight Systems`、`MCP`、`AI-agent`、`Qoder`、`M890`、`810E`、`training-performance`、`Asight Compute`、`trace-analysis`。
