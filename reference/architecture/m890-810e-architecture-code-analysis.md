# 真武 M890 与 810E 架构代码对照分析

> 分析日期：2026-11-22。本文依据平头哥官方开源实现和本地官方文档整理。代码结论只代表对应 commit 中实际启用的实现路径，不代表芯片未公开的全部理论能力。

## 一、分析范围与版本

| 来源 | 版本或 commit |
| --- | --- |
| Flash Attention for SAIL | `380f519ee881ce5e27e9a46b85a9495b8ab6cbd5`，分支 `v2.8.2` |
| DeepGEMM for SAIL | `2cc83ab849cf0804ef4a1791e7cbe13736ddbf55`，分支 `main` |
| 上游 Flash Attention | `7321879fde54f09ed94f7f6ce9377e2f4cf1fac0`，版本 `v2.8.2` |
| 上游 DeepGEMM | `5081f8513af57e61696de4f4f04b0e707ffb503e`，分支 `main` |

配套本地官方资料：

- [平头哥真武 M890 架构白皮书](m890-architecture-whitepaper.md)
- [T-Head SAIL TIX 编程指南](../programming-guide/02-tix-programming-guide.md)
- [T-Head SAIL HGGC 算子性能优化实战指南](../programming-guide/04-hggc-kernel-optimization.md)

## 二、核心结论

1. 真武 810E 与 M890 都是 32 线程 warp、SIMT 与 Tensor Cell 混合执行的 GPGPU 架构，但两代 Tensor Cell、AIU 搬运和输出布局不同。
2. 810E 对应 `ppu001`，使用 `tc01`；M890 对应 `ppu0015`，使用 `tc02`。两者的最低层矩阵指令不是同一个 ISA 分支。
3. 真武 Tensor Cell 的基础输出 tile 是 `16×16`，不同于原版 Flash Attention 在 Ampere 路径使用的 `16×8×16`。
4. Tensor accumulator 位于 VREG。CU 内部还被建模为 8 个 WE，warp 的 accumulator、operand 和 scale fragment 必须满足单 WE 的 VREG 预算。
5. 数据搬运以 AIU 和 TSM 为核心，而不是直接复制 NVIDIA TMA/shared-memory 模型。AIU 负责 global→TSM 的批量搬运，TSM 提供带 swizzle 的片上暂存和 fragment 读取。
6. M890 增加 FP8/MXFP4 Tensor Cell、更丰富的 AIU 2D tile、NoTsm 输出路径和大 tile 能力；810E 更突出 INT8、INT8×INT4、窄输出、GEMV 和 SIMT 资源利用。
7. 810E 的硬件/驱动同时暴露“64 个通用 CU”和独立的 Tensor CU 数量属性（当前参考配置为 20），官方 DeepGEMM 在此基础上分别维护 Tensor 与 SIMT 预算；M890 在 DeepGEMM 中直接把常规 CU 数作为 Tensor GEMM 调度预算。

## 三、共同架构特征

### 1. CU 内部存在 8 个 WE

DeepGEMM 的占用率模型把每个 CU 的 VREG 平均划分给 8 个 WE，并要求单个 warp 的 accumulator、双缓冲 A/B 和 scale fragment 不超过单 WE 的可分配 VREG。

最终 occupancy 同时受以下约束：

- TSM 容量；
- CU 最大 warp 数；
- 每 WE 的 VREG；
- CTA 线程/Warp 总数。

代码依据：

- [8 WE 与 VREG 占用模型](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/deep_gemm/include/deep_gemm/common/gemm_occ_model.cuh#L132-L172)

这不是普通“每 SM 只有统一 warp 池”的资源模型。即使 CU 总 VREG 足够，只要单个 WE 放不下一个 warp 的状态，也会限制并行度。

### 2. Tensor Cell 采用固定 16×16 输出面

Flash Attention for SAIL 用 `PPU0010_16x16x16` 或 `PPU0015_16x16x16` 替换上游 Ampere 的 `SM80_16x8x16`。PPU 的 accumulator C fragment 每 lane 含 8 个位置，上游路径为 4 个，因此需要专门的 accumulator layout 和 epilogue。

代码依据：

- [PPU0010 与 PPU0015 MMA 分支](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/csrc/flash_attn/src/kernel_traits.h#L39-L58)
- [PPU accumulator layout](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/csrc/flash_attn/src/utils.h#L189-L248)

DeepGEMM 进一步显示，K 方向随 dtype 位宽增加：

| 输入 dtype | 典型 Tensor 原子 |
| --- | --- |
| FP16/BF16 | `16×16×16` |
| INT8/FP8 | `16×16×32` |
| FP4 | `16×16×64` |

因此，低精度既减少数据宽度，也提高单条 Tensor 指令覆盖的 MAC 数。

### 3. Accumulator 位于 VREG

DeepGEMM 明确将以下资源计入每 warp 预算：

- FP32 accumulator VREG；
- 双缓冲 A/B operand VREG；
- blockwise scale fragment；
- cp.async/AIU 地址；
- epilogue 临时值。

Blockwise GEMM 保留 raw accumulator 和最终 accumulator，因此 VREG 数量直接翻倍。

代码依据：

- [ACC、A/B 与 scale 的 VREG 公式](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/deep_gemm/include/deep_gemm/common/gemm_occ_model.cuh#L112-L163)

这就是为什么 tile、double buffer、WarpOnK 和 epilogue 选择会直接影响 occupancy。

### 4. AIU 负责批量数据搬运

Flash 和 DeepGEMM 均把主要 global→TSM 搬运交给 AIU：

- Flash 使用 `PPU_AIU_LOAD` 读取 Q/K/V；
- 通常由一个指定 warp 发起 AIU；
- DeepGEMM 使用 `ppu.cp.async.aiu.bulk.tensor...`；
- 其他 warp 继续计算或等待阶段数据。

代码依据：

- [Flash AIU 与 TSM load](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/csrc/flash_attn/src/kernel_traits.h#L60-L75)
- [由指定 warp 发起 AIU copy](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/csrc/flash_attn/src/utils.h#L346-L359)

AIU 将多维地址计算、布局处理和批量搬运从普通 ALU/LSU 指令中剥离，使计算 warp 能继续执行 Tensor Cell 工作。

### 5. TSM 带有硬件 swizzle 和 fragment 读取能力

Flash 使用 `PPU_TSM_LD_SWZL` 从 TSM 直接读取符合 MMA fragment 布局的数据。M890 的 `ldmatrix.swzl` 还能携带 bulk tensor、x4 fragment、LBO/SBO 和 swizzle mode。

代码依据：

- [810E tc01 AIU/TSM 基址模型](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/deep_gemm/include/deep_gemm/cute_free/arch/ppu_tc01_copy.cuh)
- [M890 tc02 AIU/TSM 指令](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/deep_gemm/include/deep_gemm/cute_free/arch/ppu_tc02_copy.cuh)

Flash 还明确处理了 PPU shared memory 不自动清零的问题，非对齐 tile 必须显式清越界区域。

- [PPU 越界 shared memory 清零](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/csrc/flash_attn/src/utils.h#L333-L359)

## 四、M890 架构特点

### 1. tc02 扩展了低精度 Tensor Cell

M890 的 `tc02` 支持：

- FP16/BF16 `16×16×16`；
- FP8 E4M3 `16×16×32`；
- MXFP4 `16×16×64`。

DeepGEMM 的 FP8 和 FP4 主循环固定使用 `PPU0015`。代码检查范围也明确标注 FP4、FP8 和 W4A16 MXFP4 MMA 需要 M890。

代码依据：

- [M890 FP8 GEMM](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/impls/fp8_gemm.hpp#L279-L364)
- [M890 FP4 GEMM](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/impls/fp4_gemm.hpp#L280-L335)
- [810E 支持范围](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/tests/checkin/checkin_810e.list#L1-L7)
- [M890 支持范围](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/tests/checkin/checkin_890p.list#L1-L8)

根据 TIX 文档，M890 的 FP8 理论算力可达到 FP16 的 2 倍，FP4 可达到 4 倍。

### 2. AIU 支持更丰富的二维 tile

M890 AIU 指令携带以下信息：

- 2D tile 尺寸；
- LLC 128B 提示；
- 零填充；
- 硬件 swizzle；
- tile 坐标与 tensor stride；
- TSM 地址。

810E 的 tc01 copy 仍需分别维护 shared base、row 和 K cube。M890 则用 16B 单位的 TSM 地址表达片段位置。

### 3. 默认使用 NoTsm Epilogue

DeepGEMM 在 M890 上对 BF16、FP8 和 INT8 优先使用 `DefaultEpilogueNoTsm`：

- VREG 结果直接转换为 BF16；
- 直接向量化写全局内存；
- 避免写入 TSM 再读取重排；
- 释放 TSM 供主循环流水使用；
- 支持奇数 N 和更灵活的 BN。

代码依据：

- [M890 NoTsm Epilogue 分支](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/impls/bf16_gemm.hpp#L151-L305)
- [M890 FP8 Epilogue](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/impls/fp8_gemm.hpp#L350-L364)

### 4. 支持更大的 CTA 和更深流水

Flash Attention for SAIL 中，M890 可以使用：

- hdim192 的 `128×64`、8 warp 路径；
- D>192 的 `64×48` decode tile；
- D256 BF16 的 `M256/N64/16 warp/3 stage` decoupled 路径。

代码依据：

- [PPU tile 选择](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/hopper/tile_size.h#L96-L176)
- [M890 D256 decoupled 配置](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/hopper/mainloop_fwd_sm80.hpp#L114-L120)

### 5. 通过降低 occupancy 控制 group conflict

Flash backward 对 M890 的 hdim128/256 路径把每 block 的动态 shared memory 请求补到 128KiB，以限制每 SM 同时驻留的 block 数，降低 block 间 TSM 资源争用。代码注释明确称之为 group conflict。

代码依据：

- [M890 128KiB shared-memory 预算](https://github.com/t-head/flash-attention-for-sail/blob/380f519ee881ce5e27e9a46b85a9495b8ab6cbd5/hopper/flash_bwd_kernel_sm80.h#L21-L46)

这说明在 M890 上，更高 occupancy 不一定更快；当 TSM 分组竞争严重时，限制常驻 block 数可能提高吞吐。

### 6. 编译器栈具备更多低层调度能力

DeepGEMM 的 M890 编译路径增加了：

- `-dopt 3`；
- fence patch；
- `ppu-cg-to-kp1`；
- 未初始化值修复；
- SIMT branch 控制；
- TSM/VALU WAR 调整；
- FMA/MMA reuse；
- subregister reassign；
- warp-interleaving 专用边界调度。

810E 只启用较小的编译优化集合。

代码依据：

- [M890 与 810E 编译参数](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit/compiler.hpp#L257-L313)

## 五、810E 架构特点

### 1. 芯片同时暴露通用 CU 与 Tensor CU 两种资源口径

810E 的设备组织不能只用一个 CU 数量描述：

- 官方编程指南给出的 810E 参考配置是 64 个通用 CU；
- 对 `ZW810E` 和 `ZW610E`，官方 DeepGEMM 通过 `hgDeviceGetAttributeAD` 单独读取 `HGAD_DEVICE_ATTRIBUTE_TENSOR_MULTIPROCESSOR_COUNT`；
- 当前参考配置返回 20 个 Tensor CU；
- 因此 64 是通用 CU 总数口径，20 是该参考配置上报的 Tensor 能力资源数，而不是 DeepGEMM 自造的常数。

这项区分属于 810E 的设备/驱动资源模型。它并不只服务于 DeepGEMM；任何需要给 Tensor 和 SIMT 路径分别设资源上限的 Harness、调度器或性能模型，都应先读取这两个属性。

设：

- `n = set_num_sms(...)` 的设置值；
- `T = TENSOR_MULTIPROCESSOR_COUNT`，当前 810E 参考配置为 20；
- `C = multiProcessorCount`，当前 810E 参考配置为 64。

此时 DeepGEMM 的预算映射是：

| 设置值 `n` | Tensor 预算 | SIMT 预算 | 参考配置下的例子 |
| --- | --- | --- | --- |
| `n = 0`（自动） | `T` | `C` | Tensor 20 / SIMT 64 |
| `1 ≤ n ≤ T` | `n` | `C - T + n` | `n=1` 为 Tensor 1 / SIMT 45；`n=10` 为 Tensor 10 / SIMT 54；`n=20` 为 Tensor 20 / SIMT 64 |
| `T < n ≤ C` | `T` | `n` | `n=21` 为 Tensor 20 / SIMT 21；`n=64` 为 Tensor 20 / SIMT 64 |

对应源码公式为：

```cpp
tensor_budget = min(n, T);
simt_budget = n <= T ? C - T + n : n;
```

`get_num_simt_sms()` 的原始返回语句是：

```cpp
return sms <= tensorCUs ? prop->multiProcessorCount - tensorCUs + sms : sms;
```

这里最容易误读的是 `1–20` 区间。把 `n` 从 20 调小到 10，Tensor 预算从 20 降到 10，SIMT 预算也从 64 降到 54；它不会把少用的 10 个 Tensor CU“转换”成额外 SIMT CU。到 `21–64` 区间，Tensor 仍封顶为 20，而 SIMT 直接跟随 `n` 扩展。因此：

- 自动档给出当前配置下最大的 Tensor 预算 `20` 和完整 SIMT 预算 `64`；
- `1–20` 是同时收缩 Tensor 与 SIMT 的受限预算，不是释放更多 SIMT；
- `21–64` 只扩大 SIMT，Tensor 不再增长。

代码还保留了 `kTensorCUs810E = 20` 作为离线 wave 成本模型，但实际运行时 Tensor 调度和配置校验使用设备查询值。二者不能混为一谈。

代码依据：

- [810E Tensor CU 查询与 SIMT 预算公式](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit/device_runtime.hpp#L48-L183)
- [Tensor 预算取设备 Tensor CU 上限](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/heuristics/common_bf16.hpp#L716-L735)
- [预算映射测试](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/tests/test_gemv_dense.py#L359-L410)
- [810E Tensor/SIMT 预算规则](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/README.md#L826-L829)

当前材料没有给出 `20` 与 `64` 在硅片上的完整物理映射，也没有证明二者是 `20 + 44` 的固定互斥分区。因此可以确认“设备提供了两种可查询的资源口径，官方库维护两套预算”，但不能进一步断言某个 CU 是否专门只执行 Tensor 或 SIMT。

### 2. tc01 以经典精度和整数计算为主

810E 的 Tensor Cell 支持：

- FP16；
- BF16；
- TF32；
- FP32；
- INT8。

DeepGEMM 在 810E 上覆盖：

- BF16 GEMM；
- INT8 GEMM；
- INT8×INT4 A8W4；
- W4A16 INT4；
- true INT8 MQA/indexer。

代码依据：

- [810E A8W4 INT8](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/csrc/jit_kernels/impls/int8_gemm.hpp#L539-L575)
- [810E/M890 indexer dtype 差异](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/README.md#L461-L467)

### 3. 更依赖 TSM Epilogue 和严格对齐

当前 DeepGEMM 的 810E BF16/INT8 路径默认使用 TSM epilogue。与 M890 的 NoTsm 路径相比，它需要先经过 TSM 做输出重排和转换。

由此带来更明显的约束：

- tiled BF16 要求 `K>=64` 且 `K%32==0`；
- 输入和输出要求 16 字节对齐；
- 部分路径要求 N 为偶数或满足特定粒度；
- 窄 N 和小 M 走 GEMV/Direct 专用路径。

代码依据：

- [810E BF16 接口约束](https://github.com/t-head/DeepGEMM-for-sail/blob/2cc83ab849cf0804ef4a1791e7cbe13736ddbf55/README.md#L130-L149)

### 4. 更适合小 M、整数推理和窄输出

810E 的验证集包含大量：

- `m=1/2` GEMV；
- 窄 N 与部分输出；
- INT8 channel scale；
- A8W4 INT8；
- SIMT split-K；
- INT8 indexer。

M890 的验证集更强调：

- FP8；
- MXFP4/FP4；
- W4A8/NVFP4；
- BF16 FP32 输出；
- 大 tile 与奇数 N；
- 稀疏 MXFP4 MQA。

## 六、两款芯片的关键差异

| 维度 | 810E | M890 |
| --- | --- | --- |
| 软件架构 | `ppu001`、`PPU0010` | `ppu0015`、`PPU0015` |
| CU 与 Tensor 资源 | 参考配置为 64 个通用 CU；官方驱动另报 Tensor CU 数（参考值 20）。DeepGEMM 采用独立 Tensor/SIMT 预算，自动档为 20 / 64 | 白皮书为 39 CU；当前 DeepGEMM 将常规 CU 数直接作为 Tensor GEMM 调度预算 |
| Tensor Cell | `tc01`，FP16/BF16/TF32/F32/INT8 | `tc02`，新增 FP8、MXFP4 |
| FP8/FP4 | 当前 DeepGEMM/Flash 验证路径不使用 | FP8、MXFP4、NVFP4、W4A8 的主要平台 |
| AIU | 基础 bulk tensor | 2D tile、swizzle、LLC 提示、更丰富的 TSM 地址 |
| 输出路径 | 默认 TSM Epilogue | 默认 NoTsm，VREG 直接写全局内存 |
| 典型约束 | 对齐、K/N 粒度、TSM staging | 更强的 VREG、TSM 与 group conflict 控制 |
| 典型强项 | INT8、INT8×INT4、GEMV、窄输出、SIMT | FP8/FP4、大 tile、低精度 Attention/MoE、深度异步流水 |

## 七、与上游 GPU 代码的差异

### Flash Attention

上游 Flash Attention 2.8.2：

- Ampere/SM80 路径使用 `16×8×16` MMA；
- Hopper/SM90 路径使用 TMA、GMMA、producer warpgroup、cluster 和 mbarrier。

真武移植：

- 用 PPU 的 `16×16×16` MMA 替换 SM80；
- 用 AIU 替换主要 TMA/cp.async 数据搬运；
- 用 TSM swizzle load 替换普通 shared memory + ldmatrix；
- PPU 构建默认禁用 SM90 和 cluster；
- 使用更扁平的计算 warp 自调度，并选择性提供 decoupled 流水；
- 增加 PPU/OOB 清零和 accumulator layout 转换。

### DeepGEMM

上游 DeepGEMM：

- SM90 路径使用 TMA、cluster multicast、GMMA；
- SM100 路径增加 UMMA、Tensor Memory、CTA pair 和 2-SM TMA。

真武移植：

- TMA descriptor 映射为 AIU descriptor；
- shared memory 映射为 TSM；
- GMMA/UMMA/TMEM 改为 PPU Tensor Cell 与 VREG accumulator；
- 不沿用 SM90/SM100 cluster 和 TMEM 模型；
- 使用 PPU 自己的 multistage AIU mainloop、NoTsm epilogue 和编译器调度。

因此，这两份真武实现适合做**流水和 tile 结构参考**，但不能直接复用 NVIDIA 的 MMA shape、TMA、warpgroup、TMEM、cluster 或同步语义。

## 八、对 Harness 的直接启示

1. 架构识别必须先于算子生成：`__HGGC_ARCH__ == 100` 与 `150` 需要生成不同指令和布局。
2. 优化 M890 时优先评估 FP8/FP4、AIU 2D copy、NoTsm epilogue、D256 大 tile 和 warp-interleaving。
3. 优化 810E 时优先评估 INT8、A8W4、GEMV/Direct、SIMT split-K 和 TSM epilogue。
4. 两款芯片都必须把 accumulator、operand 和 scale fragment 的 VREG 预算纳入 tile 选择。
5. 不能只按照一个 CU 数量估算 810E 性能：先查询通用 CU 总数和 Tensor CU 数，再按目标库的预算接口解释。参考配置下 `n=10` 表示 Tensor 10 / SIMT 54，而不是“Tensor 10 + 额外 SIMT 奖励”。
6. 非对齐 shape 必须考虑 TSM 非自动清零、OOB 清零、输出对齐和 TSM 重排成本。
7. M890 出现 group conflict 时，适当降低 occupancy 可能是正确优化。

## 九、证据等级与限制

- **强证据**：`tc01/tc02`、FP8/FP4 dtype、16×16 Tensor 原子、AIU/TSM 指令、NoTsm/TSM epilogue、8 WE 占用率模型。
- **硬件/驱动资源事实**：810E 官方编程指南给出 64 个通用 CU 的参考值；官方驱动提供独立的 Tensor CU 属性，当前参考配置为 20。
- **软件预算规则**：自动档为 20 Tensor / 64 SIMT；`n=1–20` 时预算为 `n` Tensor / `44+n` SIMT，`n=21–64` 时为 20 Tensor / `n` SIMT。这是 DeepGEMM 当前接口对硬件属性的映射，不能反推未公开的硅片物理分区。
- **证据限制**：现有本地资料没有匹配的 810E 架构白皮书，也没有说明 64 个通用 CU 与 20 个 Tensor CU 在物理上如何重叠或复用；文档只采用官方指南、官方驱动查询和官方仓库代码能够直接支持的范围。
- **受 workload 约束**：大 tile、小 tile、GEMV、INT8 indexer 等选择来自当前官方实现和验证集，不代表所有业务都应按相同阈值分流。
- **文档交叉验证**：M890 的 39 CU、9.75MiB TSM、FP8/FP4 算力关系、ICN 800GB/s 和 TSM group 结构同时得到架构白皮书与 TIX 文档支持。
