---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=33"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=33"
source_title: "BF16 DenseGemm Memory Bound 优化：把 tile 自由度花在「凑整轮」上"
source_id: "blog-33"
source_kind: "technical-blog"
source_date: "2026-09-28"
source_version: null
source_format: "HTML"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "641f79355da1d6f38b4c982e641512fa699b2c50a1ddab8076a4e46f3461d09a"
tags:
  - "BF16"
  - "DenseGemm"
  - "memory-bound"
  - "tile-selection"
  - "wave-quantization"
  - "persistent-kernel"
  - "CU-balance"
  - "EpilogueNoTsm"
  - "WarpOnK"
  - "GEMV"
---
# BF16 DenseGemm Memory-Bound 优化：把 tile 自由度花在「凑整轮」上

PPU GEMM kernel 优化实践 · 技术博客 · 2026-09
硬件：真武 M890 · 39 CU · 256 KB SMEM/CU · 8 WE/CU（512 regs/WE）· LLC 峰值 2.664 T/s（实测 rd 2.25 T/s @ 84.5%）
数据：K=1024 全量 2300 UT（base · opt）· K=7168 深\_K 654 UT（独占模式重采）· small-N LLC 利用率扫描

0.912

K=1024 全量 2300 UT
opt / base 时间几何平均
胜 1539 / 2300

0.920

memory-bound 核心区
DG\_M 2–160（1000 UT）
胜 665 / 1000

1.004

K=7168 深\_K · WarpOnK 适用区
DG\_M 2–160 vs base(57% splitK)
整体 0.945（654 UT）

4 条

泛化规则：整轮优先（消拖尾）
NoTsm 解锁细粒度 BN / 奇数 warp
WarpOnK 补 Tensor Cell 利用率 · m=1 走 gemv

一句话：memory-bound UT 的主要矛盾是**波数量化（拖尾）**而不是 tile 形状本身 —— memory-bound 下 LLC 利用率理论上都能达到极限（DG\_M 方向不切 Block，确保 Weight 方向不重复 load，尽早把数据 load 指令发出去；occ=1 单条 AIU 指令即可尽早下发全部数据 request），整轮下四组 occ 实测极差 ≤ 3.3pp 只是印证。 因此把选择器的自由度全部花在「把 wave 凑整」上：wave 均衡（`bn_wave_even`）+ CU 均衡（persistent 轮询调度）消拖尾； `EpilogueNoTsm` 解锁 BN%16 粒度与奇数 WarpOnN，使凑整成为可能；深\_K 区域叠加 `WarpOnK` 补齐 Tensor Cell 利用率； m=1 极端区交还 gemv 专用路径。

## 1\. 背景与口径

**问题定义。**本文的 memory-bound 区域指 `DG_M ≤ 160`（token/M 维很小、DG\_N 即 model 维很大）的 BF16 DenseGemm： 激活矩阵 A（M×K） footprint 可忽略，权重矩阵 B（N×K）流式读取主导全部访存，性能上限由 LLC 带宽决定（真武 M890 实测 2.25 T/s）。 该区域对应 decode / 小 batch 推理的主力 UT 形态，也是本次泛化工作的目标区间。

**坐标口径。**本文全部使用 DeepGemm 坐标：`DG_M`\=token 维（小），`DG_N`\=model 维（大）， tile 记法 `BM×BN` 与 DG 坐标对齐（BM 覆盖 DG\_M，BN 覆盖 DG\_N）。 base kernel 的 M/N 与 DG 坐标相反（ACBLAS M = DG\_N，ACBLAS N = DG\_M），文中所有 base tile/grid 数据均已换算到 DG 坐标。

### 1.1 数据集

| 数据集 | 对象 | 说明 | 规模 |
| :--- | :--- | :--- | :--- |
| K=1024 全量 | base | 优化前基线库（62/2300 UT 使用 splitK） | 2300 UT DG\_M 1–2048 DG\_N 512–40960 |
| opt | tile 泛化后版本（本文主数据） |
| K=7168 深\_K | opt vs base | opt（含 WarpOnK）/ base（独占模式全量重采） | 654 UT DG\_M 1–2048 × DG\_N 512–10240 |
| small-N 扫描 | base tile | 固定 tileN=64/128/256/512（occ=8/4/2/1），扫 N 构造 wave 0.25–2.75 | 真武 M890 LLC 利用率 |

**命名口径。**base = 优化前基线库（原 acblas）；opt = tile 泛化后的版本。 所有比值均为**时间几何平均**：opt/base（<1 = opt 更快）； K=7168 数据经过 bi-directional 异常审计与独占模式全量重采。

## 2\. 核心洞察：拖尾是主要矛盾，memory-bound 下 LLC 利用率理论上都能达到极限

memory-bound UT 长期以来的调优直觉是「小 tile + 高 occupancy 更抗拖尾」。 small-N 扫描实验（真武 M890；固定 tileN=64/128/256/512 四组 base tile、扫 N 构造 0.25–2.75 波数；LLC 峰值 2.664 T/s）给出了定量答案： memory-bound 场景下，LLC 利用率**理论上都能达到极限**，与 tile 形状 / occupancy 无关 —— 核心是 **DG\_M 方向不切 Block**（确保 Weight 方向不重复 load），并且**尽早把数据 load 指令发出去**； occ=1 相比 occ=2/4/8 的 tile，**单条 AIU 指令就能把所有数据 request 尽早下发**，wave=1.0 时反而最优（81.2%）。 **wave 为整数**的实测印证了这一点（四组利用率极差 ≤ 3.3pp）； 只有 wave 落在小数上时，四组才拉开差距 —— 且越是高 occupancy 越抗拖尾、越是低 occupancy 伤害越大。

① Fractional Wave · tileN=64/128/256/512（occ=8/4/2/1）· LLC 利用率

真武 M890 · LLC 峰值 2.664 T/s · 固定 tileN（64/128/256/512）扫 N 构造 0.25–2.75 波数 · UT shape：DG\_M=16 × DG\_N 4992–54912（步长 4992 = 0.25 波/档，wave=1.0 → DG\_N=19968）× DG\_K=5120，四组 tileN×occ 均为 512，故同一 DG\_N 序列对四组波数一致。整轮（wave=1.0/2.0）四线收敛；非整轮大幅分层。

**读法：**wave=1.0 处四组利用率 77.8–81.2%（极差 **3.3pp**）；wave=2.0 处极差仅 **1.9pp**。 wave=0.25 处 occ=1（tileN=512）只有 **22.6%**，而 occ=8（tileN=64）有 **56.1%**，极差 **33.5pp**。
• memory-bound 下 LLC 利用率**理论上都能达到极限**（DG\_M 方向不切 Block，确保 Weight 方向不重复 load；尽早把数据 load 指令发出去）—— tile 形状 / occupancy 是**次要因素**；occ=1 单条 AIU 指令即可尽早下发全部数据 request，wave=1.0 时反而最优（81.2%），且 AIU load 指令数少 87.5%
• 拖尾（wave 量化到小数）才是**主要矛盾** —— 非整轮 9 个采样点极差平均 ~16pp，最大 33.5pp

② 各 wave 下四组 occ 的利用率极差（max − min）

整轮柱（1.0 / 2.0）以绿色标出：3.3pp / 1.9pp；非整轮平均 ~16pp。

**推论（本文方法论的地基）：**既然整轮下 tile 形状差异 ≤3pp，那么选择器完全可以**牺牲「理想 tile 形状」来换「整轮」** —— 把 BN 当作连续可调的旋钮去凑 `tiles = 39 × 整数`，代价可忽略；收益是消掉最大 33pp 的拖尾损失。 这直接导出 3. 的 wave 均衡设计。

## 3\. Wave 均衡 + CU 均衡：把拖尾清零

### 3.1 选择器六步流水线（`_select_tile_memory_bound`）

memory-bound 路径（BM ≤ 160）将 (BN, WN, BK, stages, warp\_k) 作为一个整体联合决策，核心是第 3 步的「每波均摊 BN」：

```
# tile 选择器 · memory-bound 主流程
def _select_tile_memory_bound(problem_n, problem_m, problem_k, num_sms, block_m, warp_m):
    # 1. plan_max_bn: 各 WN 候选 {16,32,48,64} 在 SMEM+VREG+warp 预算下的最大可达 BN
    plan_max_bn = max(achievable_bn(wn) for wn in wn_candidates)
    # 2. wave_upper: 用最大 BN 粗算总波数
    m_tiles    = ceil(problem_m / block_m)                    # memory-bound 区 m_tiles 通常 = 1
    wave_upper = ceil(m_tiles * ceil(N / plan_max_bn) / num_sms)
    # 3. bn_wave_even: 把 N 均摊到 (39 CU × wave_upper 轮)，得到每轮每 CU 的理想 BN  ← wave 均衡
    bn_wave_even = ceil(m_tiles * N / (num_sms * wave_upper))
    # 4. warp_n: 取最小可覆盖 bn_wave_even 的 WN（过寄存器模型/经验 cap 校验）
    # 5. block_n = ceil(bn_wave_even / warp_n) * warp_n       ← BN 量化到 WN 的倍数
    # 6. BK 从 64 起倍增直到 stages ≤ 3；stages ∈ {2,3}；warp_k = get_warp_k(...)
```

### 3.2 CU 均衡：persistent kernel + 轮询调度器

kernel 以 `grid = min(num_tiles, 39) × blocks_per_cu` 常驻启动（选择器返回值的第一元素即做了 tile 数 clamp）， 由 `DenseGemmScheduler::fetch_next_work` 以 `next = iter × gridDim.x + blockIdx.x` 轮询领 tile： 波内均分由硬件块号天然保证，**不存在「最后一个 ragged wave」** —— 只要 tile 总数是 39 的整数倍，每个 CU 恰好做整数轮。

```
// persistent kernel · 轮询调度器
__device__ bool fetch_next_work(uint32_t& m_block_idx, uint32_t& n_block_idx) {
    const auto next_block_idx = (current_iter++) * gridDim.x + blockIdx.x;   // 轮询领号
    if (next_block_idx >= num_blocks) return false;
    get_swizzled_block_idx(...);                                             // 2-tile 分组 swizzle 保 L2
    return true;
}
```

两个均衡的关系：**wave 均衡**决定 tile 总数是否贴合 `39 × 整数`（宏观无拖尾）； **CU 均衡**保证波内 39 个 CU 每轮领到的 tile 数严格相等（微观无倾斜）。 配合 2. 的结论（整轮下 tile 形状不敏感），BN 可以被自由量化去凑整。

### 3.3 Worked examples（K=1024，opt vs base）

| DG\_M×DG\_N | base tile (M×N) | base blocks | base 有效 wave† | opt tile BM×BN×BK(S) | opt warps (Wm×Wn×Wk) | opt tiles | opt wave | base μs | opt μs | opt/ base |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 16×5120 | 16×64 | 80 | 0.68 | 16×144×256 (S3) | 9 (1×9×1) | 36 | 0.92 | 10.10 | 9.88 | 0.978 |
| 48×13824 | 64×128 | 108 | 0.92 | 48×368×128 (S2) | 23 (1×23×1) | 38 | 0.97 | 19.42 | 20.13 | 1.037 |
| 64×24576 | 64×128 | 192 | 0.98 | 64×640×64 (S2) | 20 (1×20×1) | 39 | 1.00 | 31.25 | 30.45 | 0.974 |
| 136×40960 | 64×128 | 960 | 4.92 | 160×352×128 (S2) | 22 (2×11×1) | 117 | 3.00 | 78.63 | 68.44 | 0.870 |
| 160×40960 | 64×128 | 960 | 4.92 | 160×352×128 (S2) | 22 (2×11×1) | 117 | 3.00 | 80.55 | 69.04 | 0.857 |

† 有效 wave = blocks / (39 × occ)，occ = 256KB / 每 block SMEM（base 四例 SMEM 80/72/48/48 KB → occ 3/3/5/5）。 opt tile SMEM 176–256 KB → occ=1，名义 wave 即实际 wave。base tile/grid 已换算到 DG 坐标。所有配置均与 acu 报告逐字段核对（block\_size / grid\_size / shmem 一致）。

③ Wave 均衡效果：每 CU 分到的 tile/block 数

opt（绿）贴合整数 1.00 / 3.00；base（蓝，有效 wave）停在小数。opt 因 SMEM≥176KB 全部 occ=1，名义即实际。

**wave 贴合度与收益（K=1024）：**
• 48×13824：base 0.92 → opt 0.97，**−3.7%**（base 本身已近整轮，±4% 内视为打平）
• 64×24576：base 0.98 → opt 1.00，**+2.6%**
• 16×5120：base 0.68 → opt 0.92，**+2.2%**
• 136/160×40960：base 4.92 且 M 方向 3 个 tile 重复读 B，opt 3.00 整轮 + B 单次读 → **+13% / +14%** —— 收益集中在拖尾最重的 shape

### 3.4 M-padding 拖尾：用计算浪费换访存量

base 固定 BM=64，对 DG\_M=136/160 切出 **3 个 M-tile**，同一列的 B 条带被逻辑读 3×（部分被 LLC 吸收，但 LLC 带宽同样被消耗）； opt 直接取 BM=160（1 个 M-tile），B 单次读。代价是 M=136 时 24/160 = 15% 的行计算浪费 —— 在 B 流式主导的 memory-bound 区，这笔交易稳赚： 两例分别净赚 **13% / 14%**。BM=144 候选因编译器 stack 问题暂以 160 替代。

### 3.5 CU 内部均衡：WE 对齐与寄存器预算

真武 M890 每 CU 8 个 WE、每 WE 512 个寄存器，`regs/warp = 512 / ceil(warps/8)`（3 warps/WE 时按编译器约定取 168）。 以 worked example #4/5 为例：22 warps → ceil(22/8) = 3 warps/WE → 168 regs/warp，恰好覆盖 `acc(WM×WN) + 双缓冲输入片段` 的需求（VREG 模型 `_estimate_warpOnN` 按 (128, 168, 256) 三档预算校验，经验 cap 表 `_REGISTER_WARPONN_CAP` 覆盖例外）。 warp 总数上限同时受两条硬件约束：64 warps/CU（常驻）与 32 warps/block —— 二者在 `_MAX_WARPS_PER_CU / _MAX_WARPS_PER_BLOCK_HW` 中分开建模。

## 4\. EpilogueNoTsm：细粒度 BN 与奇数 warp 数的使能器

3\. 的 wave 均衡要求 BN 能以 16 为步长任意取值（如 144、368）。 之前 kernel 默认 epilogue 阶段做完 mma 之后先进 share memory 重排，重排之后再读数据并写回 global，其 SMEM 布局原子 `tile_to_shape(SmemLayoutAtomO, Shape<WarpOnM*16, BlockN>)` 中 `ElemInLine = 128 / sizeof(ElementAcc) = 32`（fp32 累加），强制 **BN % 32 == 0**，WarpOnN 只能取偶数。 本次切换到 `DefaultEpilogueNoTsm`：寄存器累加值直接 `bf16x2` 向量化写回 global，绕过 TSM staging。

**WithTsm 的限制。**此前的 epilogue 方案都是 mma 完成后先存入 TSM 重新排序，后续 warp 的线程才能读取**连续的 128 bit（16 Byte）**数据，之后再连续整块地写 global。代价有两层：其一，TSM 的一读一写本身有开销，还引入 sync 开销；其二，对齐约束落在 BN 方向 —— 每个线程需要读取 128 bit，BN 方向会切分成**整数线程数**（如 4/8/16/32 线程），即 BN 需要满足一定的对齐，**BN 的选择范围因此受限**。

**NoTsm 为何可行。**NoTsm 直接跳过 TSM 中转：mma 完成后每个线程保有 **2 个连续数值**（`bf16x2`），连续 4 个线程合计 **8 个连续的数值**，直接 store 写 global。这些写会 cache 在 L1、并不会立马从 L1 flush 到 LLC —— 因此 NoTsm 优先先在 BN 方向写 global，相邻的数据在 L1 里**自然拼接起来**。acu 报告中可以看到**从 L1 到 L2 的 write request 数量并不变化**，碎片化写没有放大到 L2。所以 NoTsm 相比 WithTsm 本身会有一些提升；其次 NoTsm 解除了 BN 的对齐限制 —— **BN 只需要对齐到 WN 的整数倍即可**（16 粒度，见下表）。

|  | TSM Epilogue（默认） | NoTsm Epilogue（本工作） |
| :--- | :--- | :--- |
| SMEM 占用 | 输出 staging：原子 (8, min(BN,32)) 铺满 (WarpOnM×16, BN) | SharedStorage 为空，SMEM 全部留给 mainloop 流水 |
| BN 粒度 | BN % 32 == 0（fp32 行 32 元素对齐） | BN % 16 == 0（WN 候选粒度） |
| WarpOnN | 偶数（BN/WN 被 32 整除约束连带限制） | 奇数可用：9 / 11 / 17 / 23（见 worked examples） |
| 写回路径 | VREG → TSM → GMEM（经 SMEM 中转） | VREG → GMEM 直存（bf16x2，kOutputAlignment=2） |
| 副产物 | epilogue 占用 SMEM 与时间 | epilogue 不碰 SMEM → 可与下一 tile 的 prologue 重叠（overlap-prologue） |

```
// BF16 DenseGemm kernel · epilogue 选型
using CollectiveEpilogue = DefaultEpilogueNoTsm<
    Shape<int,int,int,int>, CollectiveMainloop, EpilogueDefault, ElementC, ElementD, void,
    LinearCombination<ElementC, 2, float, float, ScaleType::Nothing>,
    EpilogueDefault, IsAlignedN>;                          // ^ 每 thread 2 元素 bf16x2

// DefaultEpilogueNoTsm 定义（SharedStorage 为空）
template <class...> class DefaultEpilogueNoTsm {
  struct SharedStorage { };                                 // 空 —— 不与 mainloop 抢 SMEM
  static constexpr int kOutputAlignment = 2;                // bf16x2 直存 global
};
```

**证据：本文所有 worked examples 的 BN 均为 NoTsm 独有配置。** BN=144（=16×9）、BN=368（=16×23）、BN=272（=16×17）都不是 32 的倍数 —— TSM epilogue 下这些配置**根本不存在**； 对应 WarpOnN = 9 / 23 / 17 / 11 全部为奇数。 没有 NoTsm，3. 的「BN 当连续旋钮凑整轮」就没有物理基础。
**附带收益：**epilogue 脱离 SMEM 后，overlap-prologue 得以开启（下一 tile 的 AIU 预取与当前 epilogue 重叠）， 门控为 wave>3 且 K≤2048（A/B：K=1024 +3.8%、K=2048 +2.8%、K≥8192 ≤0.2%）。

## 5\. WarpOnK：memory × latency-bound 区域补齐 Tensor Cell 利用率

**问题。**深\_K（如 K=7168，MLA / 大 FFN 权重形态）叠加小 BN 时，wave 均衡后的 warp 网格往往只有 12–17 个 warp —— MN 方向 warp 数不足以喂满 Tensor Cell，mainloop 退化成 latency-bound（等 AIU 返回、等 TSM\_LD），此时凑整轮已无增量：LLC 利用率不低，MAC 占用不足。 这是 memory-bound（带宽）与 latency-bound（流水线深度）的叠加区域。

**对手的答案。**base 在深\_K 区大量启用 grid.z splitK：K=7168 全量 654 UT 中 **298 个（46%）**，DG\_M≤160 小\_M 区 **226/400（57%）** （例：`tile16x64x16x16x64x3 · splitk=2 · kslice=112`，grid.z 2–56）。作为对照，K=1024 时该比例仅 62/2300（~3%）—— splitK 随 K 深度大幅升级，说明该策略确实是深\_K 刚需。

**opt 的答案：块内 WarpOnK。**TiledMMA 的 warp 布局扩展为 `Layout<Shape<WarpOnM, WarpOnN, WarpOnK>>`（`WarpOnK = BLOCK_K / WARP_K`）， 各 warp 沿 K 累加各自部分和；mainloop 结束后在 SMEM 内做 `warp_on_k_reduce` 规约；epilogue 仅由 `warp_k_idx == 0` 的 warp 执行。 与 splitK 相比：无 workspace、无第二个规约 kernel、与 persistent 调度器天然组合。

### 5.1 四道门控（全部带 A/B 证据）

```
// tile 选择器 · WarpOnK 门控
inline int get_warp_k(int block_m, int block_n, int block_k, int problem_k,
                      int warp_m, int warp_n, int num_stages, bool allow_warp_on_k = false) {
    if (!allow_warp_on_k) return block_k;                    // 门 0 区域门：仅 memory-bound 路径传 true
    if (ceil_div(problem_k, block_k) < 6) return block_k;    // 门 1 K 深度：<6 个 K 迭代不切
    int base_warps = (block_m/warp_m) * (block_n/warp_n);
    int warp_on_k  = std::min(block_k/128, MAX_WARPS_PER_BLOCK/base_warps);
    if (base_warps >= 3) warp_on_k = std::min(warp_on_k, 2); // 门 2 fat-tile 上限
    if ((warp_on_k-1)*block_m*block_n*4 > smem_bytes)        // 门 3 SMEM 规约预算
        return block_k;
    if ((block_k==256 || block_k==512) && warp_on_k <= MAX_WARPS_PER_CU/(base_warps*blocks_per_cu))
        return block_k / warp_on_k;                          // BK=256 → WarpOnK=2；BK=512 封顶 2
    return block_k;
}
```

| 门控 | 规则 | A/B 证据（回归 = 开启后变慢） |
| :--- | :--- | :--- |
| 门 0 · 区域门 | 仅 memory-bound 路径 `allow_warp_on_k=true` | compute-bound / legacy 路径一律 `warp_k = block_k`（无切分） |
| 门 1 · K 深度 | `ceil(K/BK) ≥ 6` 才启用 | (130,3072,512) K/BK=2 → **+12.67%** 回归 (2,7168,768) K/BK=3 → −0.5%（临界，不启用） |
| 门 2 · fat-tile 上限 | `base_warps ≥ 3` 时 WarpOnK ≤ 2 | (2,1536,7168) base\_warps=3、BK=512、WK=4 → **+6.86%** 回归 |
| 门 3 · SMEM 规约预算 | `(WK−1)×BM×BN×4 ≤ block SMEM` | 部分和规约缓冲必须装进 block SMEM 预算（与 mainloop 流水共存） |
| 预算修正 | `MAX_WARPS_PER_CU` 32 → **64** | 原实现把 per-block 32 上限误当 per-CU 上限，系统性压低可用 WarpOnK |

### 5.2 Worked examples（K=7168，opt vs base）

| DG\_M×DG\_N | base tile · splitk | opt BM×BN×BK(S) | opt warps (Wm×Wn×Wk) | WarpOnK | base μs | opt μs | opt/base |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 16×5120 | 16×64 · splitk=2 | 16×144×256 (S3) | 18 (1×9×2) | 2 | 47.40 | 41.12 | 0.868 |
| 16×7168 | 16×64 · splitk=0 | 16×192×256 (S2) | 24 (1×12×2) | 2 | 55.65 | 55.21 | 0.992 |
| 16×10240 | 16×64 · splitk=0 | 16×272×128 (S3) | 17 (1×17×1) | 1 | 91.22 | 76.84 | 0.842 |
| 64×7168 | 64×128 · splitk=2 | 64×192×256 (S2) | 24 (1×12×2) | 2 | 62.84 | 67.02 | 1.067 |
| 112×7168 | 128×128 · splitk=2 | 112×192×128 (S3) | 12 (1×12×1) | 1 | 75.25 | 72.34 | 0.961 |

注：WarpOnK 只在 BK≥256 时激活（门控的隐含粒度：WARP\_K 最小 128）；BN 已较大（如 272 → 17 warps）时无需切 K。opt 配置均与 acu 报告核对一致（block\_size/grid\_size/shmem）。

**K=7168 结果：**全量 654 UT opt/base = **0.945**（胜 436/654）。 WarpOnK 适用区（DG\_M 2–160，400 UT）= **1.004**（胜 231/400）—— 与一个在小\_M 区 **57% 使用 splitK** 的 base 打平：块内 WarpOnK 以更低的调度代价 （无 workspace / 无二次规约 kernel）达成同等效果；个别 shape（如 64×7168）仍输 splitK 约 6%。

## 6\. m=1 极端区域：交还 gemv 专用路径

m=1（单 token decode 的极端）同时踩两个坑：**小 N → latency-bound**（kernel 启动与流水线延迟主导，GEMM tile 结构无用武之地）、 **大 N → memory-bound**（贴 LLC roofline，GEMM 与 gemv 差距收敛）。 base 对 m=1 直接路由专用 kernel `gemvt_op<bfloat16_t, float>`。

| 数据集 | 子区间 | UT 数 | opt/base | 胜/负 | 判读 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| K=1024 （base / opt） | N ≤ 4096 | 8 | 1.263 | 0/8 | latency-bound：opt GEMM 落后 gemvt\_op 最多 ~60%（N=1024：6.45 vs 4.02 μs） |
| N > 4096 | 42 | 1.004 | 23/42 | memory-bound：贴 roofline，与 gemvt\_op 打平 |
| 合计 | 50 | 1.041 | 23/50 | — |
| K=7168 （opt / base） | N=512 / 1024 | 2 | 1.47 / 1.28 | 0/2 | 12.0 vs 8.2 μs / 13.9 vs 10.9 μs —— latency 差距放大 |
| N=3072 / 10240 | 2 | 0.91 / 0.97 | 2/2 | 大 N 反超；20 UT 合计 1.002（15/20） |

**结论：**m=1 不值得用 GEMM 泛化强吃 —— 小 N 输给专用 gemv（延迟主导），大 N 仅打平 roofline（无增益）。 **行动项：**m=1 路由 gemv 专用路径。DeepGemm 已有 gemv 配置框架（`get_gemv_best_configs`，按 K 深度选 ThreadPerN/NUM\_UNROLL/SWZL）, 需打通 bf16 m=1 的路由与验证；GEMM 自适应路径服务 m ≥ 2。

## 7\. 最终性能数据（K=1024 · 2300 UT）

| 区域（按 DG\_M） | UT 数 | opt/ base | opt 胜 | 注 |
| :--- | :--- | :--- | :--- | :--- |
| \= 1 | 50 | 1.041 | 23/50 | 极端区 → 6. gemv |
| 2–160（memory-bound 核心，本文） | 1000 | 0.920 | 665/1000 | 3.+4.+5. 机制主场 |
| 192–576（过渡带） | 650 | 0.844 | 596/650 | compute-bound 选择器（另文） |
| 640–2048（大 M） | 600 | 0.968 | 255/600 | wave-fit 成本模型（m>512 线） |
| 全量 | 2300 | 0.912 | 1539/2300 | — |

全量 2300 UT 中 opt 胜出 1539 个（67%），核心区 DG\_M 2–160 胜率最高（665/1000 = 66.5%），几何平均 opt/base = 0.912，整体提速 ~9%。

④ K=1024 · 每 DG\_M 档位 geomean（opt/base）· 2300 UT · DG\_N 512–40960

46 个 DG\_M 档位 × DG\_N 512–40960 全扫。opt/base：核心区（2–160）0.84–1.01，136–160 最深（0.84，3. M-padding 案例），640+ 回 ~0.97 平手带。

**全量结果：**opt 把整体时间几何平均从 base 降至 **0.912**（提速 ~9%）； 核心区 2–160 达到 **0.920**，过渡带 192–576 达到 **0.844**（收益最大，主要归功于 wave 均衡 + M-padding）。 全量 2300 UT 中胜出 1539 个（67%）。

## 8\. 适用范围：A / B 均为 K-Major

本文的优化只适用于 **A 和 B 都是 K-Major** 的场景（3. 的细粒度 BN / 奇数 warp、M-padding、WarpOnK 均建立在这一前提上）。 原因在 AIU load 的对齐约束：AIU 搬运（GMEM → TSM）对**连续方向**有 **64/128 Byte 对齐限制** —— 只有 K-Major 时，**BN 是非连续方向**，不受该限制，**BN 才可以是任意 16 的倍数**（144 / 192 / 272 / 368…）。 若 A / B 不是 K-Major（连续方向落在 M / N 上），BN / BM 进入连续方向、被 64/128 Byte 对齐卡住，细粒度 tile 不再成立 —— 超出本文范围。

文中所有 opt tile 配置由 C++ 选择器复算，并与 acu 报告的 block\_size / grid\_size / shmem 逐字段核对一致；数据口径与审计说明见 1.。
BF16 DenseGemm Memory-Bound 优化 · PPU kernel 技术博客 · 2026-09
