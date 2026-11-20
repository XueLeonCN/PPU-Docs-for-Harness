---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=35"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=35"
source_title: "DeepGemm (PPU) W4A16 MXFP4 GEMM 实现详解—— 寄存器内反量化的两条路线"
source_id: "blog-35"
source_kind: "technical-blog"
source_date: "2026-09-29"
source_version: null
source_format: "HTML"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "fdb47794f1b1340ffdbd474e19c9dc635075259c283b87c968d39e4d203f6692"
tags:
  - "DeepGemm"
  - "W4A16"
  - "MXFP4"
  - "BF16"
  - "GEMM"
  - "VALU"
  - "MMA"
  - "dequantization"
  - "scale-layout"
  - "register-double-buffer"
---
# DeepGemm (PPU) W4A16 MXFP4 GEMM 实现详解
—— 寄存器内反量化的两条路线

目录

1.  [前言](#ch1)
2.  [内存排布预处理：weight 与 scale 的两套约定](#ch2)
3.  [VALU 版 kernel：寄存器内软件反量化](#ch3)
4.  [MMA 版 kernel：FP4 MMA 硬件反量化](#ch4)
5.  [指令成本核算：两条路线的对比](#ch5)
6.  [局限与参考阅读](#ch6)

## §1前言

LLM decoding 阶段是典型的 memory bound 场景：batch 小、算术强度低，权重读取占了大头。 **weight-only 量化（W4A16）**因此成为标配：权重压到 4 bit，激活仍用 BF16 计算， 矩阵乘的主计算路径不变，只多出一个"反量化"环节——把 4 bit 权重还原成 BF16 再进 tensor cell。 Marlin（[MARLIN: Mixed-Precision Auto-Regressive Parallel Inference](https://arxiv.org/abs/2408.11743)），它证明了 W4A16 的性能上限不取决于"算得多快"， 而取决于**反量化开销能否被访存与计算的空隙所掩盖**。

对 MoE 模型这个矛盾更尖锐。以 Kimi K3 为例：2.8T 参数的 MoE，专家权重在发布时就是 **原生 MXFP4**（E2M1 数据 + E8M0 scale，group=32），不存在"从 BF16 离线压"的选项—— 推理引擎要么原生支持 MXFP4 权重，要么先反量化成 BF16 再计算（权重体积 ×4，MoE decoding 的访存量成倍增加）。因此需要一个 MXFP4 版本的 W4A16 GEMM：权重以 4 bit 形式进入显存， kernel 在计算前将其展开回 BF16。MoE 的 grouped GEMM 形态（每层几百个专家、每个专家一次几十~几百 token）又放大了 decode 侧 memory bound 的特征，反量化路径的开销几乎决定了整个 MoE 层的耗时。

本文分析 DeepGemm 的 PPU 适配中 MXFP4 weight-only GEMM 的两条 kernel 路线（仓库内叫 `w4fa16` 系路径：FP4 权重 + BF16 激活）。两条路线做的是同一件事—— **在寄存器中将 FP4 权重反量化为 BF16 fragment，直接作为 BF16 MMA 的操作数**——但手段有较大差异：

|  | VALU 版（`W4A16GEMM`） | MMA 版（`W4A16GEMM_MMA`） |
| :--- | :--- | :--- |
| **反量化执行单元** | IALU + FALU，纯指令位操作 | Tensor Cell，一次 FP4 MMA |
| **E8M0 scale** | 位拼装成 BF16 后逐元素乘 | MMA 指令的 meta 操作数，硬件乘入 |
| **权重内存布局** | int32 8-nibble 交错 + Marlin 式 perm 重排 | uint8 自然序，零重排 |
| **适用平台** | PPU001 / PPU0015 | 仅 PPU0015（FP4 MMA 指令） |

VALU 版 —— 软件反量化（IALU + FALU） MMA 版 —— 硬件反量化（Tensor Cell） int32 权重 8-nibble 交错 + perm 重排 shl · and · shr · lop3 15 条 IALU 位域重排 mul ×2^126 4 条 FALU 指数补偿 × E8M0 scale ≈4 条 FALU mul.bf16x2 BF16 MMA 主 GEMM B 操作数 = tCrB（BF16 fragment，两版共用） uint8 权重 自然序 · 零重排 fp4 MMA × 单位阵 W × I = dequant(W) E8M0 经 meta 寄存器 TC 硬件乘入 · 0 指令 pcnvt f32→bf16 4 条 · 寄存器级转换 数值展开 + scale 乘入 → 同一条 MMA 指令内完成

图 1-1　两条反量化路线总览。上路（VALU 版）：约 23 条 ALU 指令把一个 int32 的 8 个 FP4 逐位重排成 BF16 并乘上 scale； 下路（MMA 版）：用 PPU0015 的 FP4 MMA 把权重与寄存器常驻的单位阵相乘，数值展开与 scale 乘法在同一条 MMA 指令内完成，末段仅需 fp32→bf16 转换；两路终点相同——BF16 MMA 主 GEMM。

两条路线的分界线是 PPU0015（`__HGGC_ARCH__ >= 150`）引入的 **带硬件块缩放的 FP4 MMA 指令**：

```
ppu.tc02.mma.mx.sync.aligned.m16n16k64.row.col.f32.f4.f4.f32
    {d0..d7}, {a0..a3}, {b0..b3}, {c0..c7}, {meta_a, meta_b}, a-psel, b-psel, 0;
```

这条指令的语义是 `D = (A · scale_A) × (B · scale_B) + C`， scale 是 E8M0（8 bit 纯指数）metadata，粒度恰好是**每行 × 每 32 个 FP4 一个字节**—— 与 MXFP4 的 group\_size=32 逐元素对应。它本来就是给"FP4 激活 × FP4 权重"（W4A4）准备的； MMA 版的核心思路在于**将其用于反量化**：让 FP4 权重乘一个单位阵，一次矩阵乘同时完成数值展开和 scale 乘入（§4）。在这条指令出现之前，VALU 版的软件位操作是唯一选择（§3）， 它在 PPU001 上至今仍是主力路径。

### §1.1 术语与命名约定

仓库里 "W4A16" 家族命名与业界叫法对应关系如下，本文统一使用左列：

| 本文用语 | 仓库内名称 | 含义 |
| :--- | :--- | :--- |
| **MXFP4 权重** | `w4fa16` / `mxfp4_e8m0` | E2M1 数据 + E8M0 scale（group=32），BF16 激活，走 VALU 版 kernel |
| **MXFP4-MMA 权重** | `w4fa16_mma` / `mxfp4_e8m0_mma` | 同上，但权重 uint8 自然序，走 MMA 版 kernel |
| INT4 权重 | `w4a16` / `int4` | 对称 INT4 + BF16 scale，VALU 版 kernel（本文不展开） |
| **E2M1** | `float4_t` | 4 bit 浮点，位序 \[sign\]\[exp\]\[exp\]\[man\]：1 符号 + 2 指数 + 1 尾数，取值 {0, 0.5, 1, 1.5, 2, 3, 4, 6} |
| **E8M0** | scale / SF / meta | 8 bit 纯指数：数值 = 2^(byte)，即 IEEE 浮点的指数字段直接当字节 |
| **IALU / FALU** | — | PPU 标量执行单元：IALU 执行整数/逻辑（shl/and/lop3），FALU 执行浮点（mul.bf16x2、cvt）；二者合称 VALU（与 Tensor Cell 相对的向量算术层） |
| fragment | `tCrB` 等 | MMA 指令每线程持有的操作数寄存器块（CuTe 术语） |

## §2内存排布预处理：weight 与 scale 的两套约定

两条路线的反量化都发生在**寄存器里**，这决定了一个硬约束：数据从 smem 进寄存器时的 nibble / 字节顺序，必须与 MMA 指令的 fragment 布局**逐元素对齐**。解包指令 （移位、掩码、lop3）是固定模式，它不会"去找数据"——只能让数据提前排好。 因此每条路线都定义了一整套**离线重排约定**（量化时一次性完成， `tests/math_utils.py::quant_w4a16` 是参考实现），kernel 内部零重排指令。

|  | VALU 版 | MMA 版 |
| :--- | :--- | :--- |
| weight 布局 | int32 `(G, K/16, N·2)`，8-nibble 交错 + tile perm | uint8 `(G, N, K/2)`，自然序，**零重排** |
| scale 布局 | uint8 `(G, K/32, N)`，64B 块 8×8 转置 + 组内交换 | uint8 `(G, N/64, K·2)`，128B 块重排对齐 ldmatrix/meta |
| 重排对象 | weight + scale 都要重排 | 只重排 scale |

### §2.1 目标布局：BF16 MMA 的 B fragment

两版 kernel 的主 GEMM 是同一条指令：AIU 的 bf16 `m16n16k16` MMA （`GetAiuMmaInst`，PPU001/0015 同布局）。W 是它的 B 操作数（K×N）， 一个 atom 的 B fragment 每 thread 8 个 bf16 = 4 个 bf16x2 寄存器。 从 MMA 的 BLayout（[PPU ISA 规范](https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=39&chapterId=197) §9.5 的 matrix fragment 定义） 可以完整解出线程↔元素映射：

thread (c₀, c₁) = (lane%4, lane/4)，寄存器 m ∈ \[0,4)，半槽 v₀ ∈ {0,1}（bf16x2 的 low/high）：
element (k, n) = ( 2·c₀ + v₀ + 8·(m mod 2) ,  c₁ + 8·⌊m/2⌋ )

即每个 thread 拿 4 个不相邻的 (k-pair, n) 组合：k ∈ {2c₀, 2c₀+1} ∪ {2c₀+8, 2c₀+9}， n ∈ {c₁, c₁+8}——一个"梳子"形。图 2-1（交互）给出 16×16 atom 的全貌——点击左侧线程切换选中（默认 T5）：

Thread（G = lane/4 分色，TID = lane%4）

tc01 / tc02 的 B fragment 分布一致；
A / C fragment 的寄存器编号两代不同

B fragment：16(k) × 16(n)，单元格 = T# / b#

图 2-1　BF16 m16n16k16 的 B fragment（BLayout，交互图）。列颜色 = 线程组 G（= n%8），线程 T = G×4 + (k%8)/2；每 thread 8 元素 = 4 个 bf16x2 寄存器 R0–R3：R0/R1 落在 n = G 列（k 偏移 +0/+8），R2/R3 落在 n = G+8 列——"梳子"形分布。点击左侧线程查看寄存器配对（默认选中 T5，与 §2.2 示例同一线程）。

这个映射是后文所有布局约定的**目标**：反量化产物的寄存器顺序必须与它完全一致。 VALU 版用 nibble 交错使移位解包的输出序与之自然一致（§2.2）；MMA 版通过 pcnvt 的源寄存器配对顺序实现同样目标（§4.4）。warp 视角下（WarpN=64），一个 warp 持有 64 个 n × 16 个 k 的 B 数据 = 4 个 atom（n\_idx ∈ \[0,4)），每 thread 每 atom 8 个 bf16——这是 §3 指令计数的基准量。

### §2.2 VALU 版权重：int32 8-nibble 交错 + tile perm

量化后的权重按 (k, n) 组织，打包流程（`tests/math_utils.py:337-345`）：

```
# tests/math_utils.py :: quant_w4a16
w = w.reshape((wk // tile, tile, wn // tile, tile))   # (k/16, 16, n/16, 16)
w = w.permute((0, 2, 1, 3))                           # (k/16, n/16, 16k, 16n)
w = w.reshape((wk // tile, wn * tile))                # (k/16, n*16)
w = w.reshape((-1, _perm.numel()))[:, _perm]          # 16k×64n 块内 perm（1024 元素）
w = w.to(torch.int32)
q = torch.zeros((w.shape[0], w.shape[1] // 8), dtype=torch.int32)
for j in range(8):
    q |= w[:, j::8] << (4 * j)                        # 每 8 个连续元素装进一个 int32
```

两个重排各司其职：**\_perm 决定"哪些元素进哪个 int32"**， **nibble 交错决定"元素在 int32 里的位序"**。

#### §2.2.1 \_perm：int32 编号 ↔ 线程/atom 的对齐

\_perm（`math_utils.py:184-201`）在一个 16k × 64n 的块内重排，使得： warp 的 64-n 切片里，**int32 #(4·lane + p)** 恰好装着 thread lane 的 atom p（第 p 个 16-n 块）所需的 8 个元素：

int32 #(4·ℓ + p) 的 8 个元素：(k, n) = (2r + {0,1,8,9}, 16·p + c + {0,8})，其中 r = ℓ%4，c = ℓ/4

kernel 侧因此只需一条 `UniversalCopy<uint128_t>`（`w4a16_gemm_cutlass3.cuh:113-121`）： thread 一次 16 字节读出 4 个连续 int32，即该线程 4 个 atom 的 fragment 输入，直接就位 `tCrB_i32(n_idx, _, k_block)`（`w4a16_gemm_cutlass3.cuh:299`）。

#### §2.2.2 nibble 交错：让移位解包的输出序 = fragment 寄存器序

int32 内 8 个 nibble 的排布（交错 `[0,2,4,6,1,3,5,7]`， `math_utils.py:200-201`）与 kernel 侧解包的注释一一对应 （`dequant_w4a16.cuh:61-62`）：

```
// dequant_w4a16.cuh —— int32 内的 nibble 交错布局
bits [ 3: 0] = e0   bits [ 7: 4] = e2   bits [11: 8] = e4   bits [15:12] = e6
bits [19:16] = e1   bits [23:20] = e3   bits [27:24] = e5   bits [31:28] = e7
```

以 thread lane ℓ=5（r=1, c=1）、atom p=0 为例，8 个元素的坐标与位序如图 2-2：

int32 #20 的 nibble 交错布局（bit31 → bit0，左高位） thread ℓ=5（r=1, c=1）· atom p=0 bit31（MSB） bit0（LSB） \[31:28\] e7 (k11, n9) \[27:24\] e5 (k3, n9) \[23:20\] e3 (k11, n1) \[19:16\] e1 (k3, n1) \[15:12\] e6 (k10, n9) \[11:8\] e4 (k2, n9) \[7:4\] e2 (k10, n1) \[3:0\] e0 (k2, n1) 提取窗口 low bit\[15:12\] 提取窗口 high bit\[31:28\] 高 16 bit：k 为奇数的 4 个元素 低 16 bit：k 为偶数的 4 个元素 fp4s\[0\] = src << 0　→　h\[3\] （窗口内容 = e6, e7） (k10, n9) (k11, n9) low high fp4s\[1\] = src << 4　→　h\[2\] （窗口内容 = e4, e5） (k2, n9) (k3, n9) low high fp4s\[2\] = src << 8　→　h\[1\] （窗口内容 = e2, e3） (k10, n1) (k11, n1) low high fp4s\[3\] = src << 12　→　h\[0\] （窗口内容 = e0, e1） (k2, n1) (k3, n1) low high \= B fragment 寄存器序 h\[0\]=R0: (k2,n1) (k3,n1) h\[1\]=R1: (k10,n1) (k11,n1) h\[2\]=R2: (k2,n9) (k3,n9) h\[3\]=R3: (k10,n9) (k11,n9) 与图 2-1 的 T5 寄存器 逐槽一致，直接写 tCrB

图 2-2　int32 内的 nibble 交错布局与 4 路移位解包（示例：thread ℓ=5 的 atom 0，bit31→bit0 左高位排布）。每格标注元素序号 e# 及其在权重矩阵 W 中的逻辑坐标 (k, n)；bit\[15:12\] 与 bit\[31:28\] 是两处固定的提取窗口（e6/e7 天然在窗口内，其余元素经移位送入）：4 路 `src << 0/4/8/12` 分别把 `(e6,e7)、(e4,e5)、(e2,e3)、(e0,e1)` 送进窗口，输出 h\[m\] 的 low/high 恰为 W 中相邻两行、同一列的元素——与 B fragment 的寄存器序（图 2-1 的 T5）逐槽一致，直接写入 tCrB。

图中可见两个设计点：

-   **低 16 bit 全是偶数 k（e0/e2/e4/e6），高 16 bit 全是奇数 k**（e1/e3/e5/e7）。 解包时 4 路平行移位 `<<0/4/8/12` 每路把一对 (偶 k, 奇 k) 的 nibble 送进固定的提取窗口 \[15:12\] / \[31:28\]——4 路之间零依赖，可完全并行发射。
-   输出 h\[m\] 的 low/high 恰为**相邻 k、同一 n**的一对元素——正是 bf16x2 寄存器内 fragment 要求的元素顺序（图 2-1 寄存器面板的 R m）。若按自然序打包（不做交错），h\[m\] 的 low/high 会变成"同 k、相差 8 的两个 n"，与 fragment 顺序错位，就需要额外的寄存器重排指令补救。

### §2.3 VALU 版 scale：8×8 转置 + 组内交换

scale 张量为 `(G, K/32, N)` uint8。注意存的是 **biased 指数字节** （fp32 的指数字段直接截出来，127 ↔ 1.0）：

```
# tests/math_utils.py:331-332 —— E8M0 字节 = fp32 指数字段
s = ((s.to(torch.float32).view(torch.int32) >> 23) & 0xFF).to(torch.uint8)
```

加载侧的约束来自 §2.1 的 fragment 映射：thread (c₀, c₁) 需要 n = 16·n\_idx + c₁ + 8·b （n\_idx ∈ \[0,4)，b ∈ {0,1}）共 8 个 n 的 scale（同一 k-group）——即 **n ≡ c₁ (mod 8)** 的 8 个字节。而 smem→寄存器拷贝是 `UniversalCopy<uint64_t>` 一次搬 8 个连续字节， 且 **4 个线程共享一组**（`get_thread_slice(threadidx_nk / 4)`， `w4a16_gemm_cutlass3.cuh:302`）。于是量化侧把 64 字节块做 8×8 转置 （`scale_perm`）+ 组内 \[0,2,1,3\] 交换（`scale_perm_e8m0`）：

```
# tests/math_utils.py:203-209
scale_perm = []
for i in range(8):
    scale_perm.extend([i + 8 * j for j in range(8)])      # 8×8 转置
scale_perm_e8m0 = []
for i in range(16):
    for sw in (0, 2, 1, 3):                            # 组内交换，补偿 DequantE8M0 的配对
        scale_perm_e8m0.append(scale_perm[4 * i + sw])
```

VALU 版 scale：64 字节块（一个 k-group × 64 n）的重排 重排前：n 行主序 n0 n1 n2 n3 n4 n5 n6 n7 n8 n9 n10 n11 n12 n13 n14 n15 n16 n17 n18 n19 n20 n21 n22 n23 n24 n25 n26 n27 n28 n29 n30 n31 n32 n33 n34 n35 n36 n37 n38 n39 n40 n41 n42 n43 n44 n45 n46 n47 n48 n49 n50 n51 n52 n53 n54 n55 n56 n57 n58 n59 n60 n61 n62 n63 0 8 16 24 32 40 48 56 字节位置 = 8·row + col，存放 n = 同值 new\[8i+j\] \= old\[i+8j\] 8×8 转置 重排后：行 i = 线程 slice n0 n8 n16 n24 n32 n40 n48 n56 n1 n9 n17 n25 n33 n41 n49 n57 n2 n10 n18 n26 n34 n42 n50 n58 n3 n11 n19 n27 n35 n43 n51 n59 n4 n12 n20 n28 n36 n44 n52 n60 n5 n13 n21 n29 n37 n45 n53 n61 n6 n14 n22 n30 n38 n46 n54 n62 n7 n15 n23 n31 n39 n47 n55 n63 i=0 i=1 i=2 i=3 i=4 i=5 i=6 i=7 每行 = n ≡ i (mod 8) 的 8 个 scale（同 k-group），一次 uint64 读取 E8M0 版追加：每 4 字节组内 \[0,2,1,3\] 交换（补偿 DequantE8M0 的 bf16x2 配对） 交换前 n0 n8 n16 n24 n32 n40 n48 n56 交换后 n0 n16 n8 n24 n32 n48 n40 n56 DequantE8M0 输出 {b0,b2},{b1,b3}， 交换后 bf16 线性序 = 字节序（§3.2）

图 2-3　VALU 版 scale 的 64 字节块重排。8×8 转置（scale\_perm）使每线程一次 uint64 读取恰好得到 n ≡ c₁ (mod 8) 的 8 个 scale；E8M0 路径再追加组内 \[0,2,1,3\] 交换（scale\_perm\_e8m0），补偿 DequantE8M0 把 4 字节输出为 {b0,b2}/{b1,b3} 两对 bf16x2 的配对顺序。

转置让"每线程一次 uint64 读的字节"恰好是 n ≡ c₁ (mod 8) 的 8 个 scale； \[0,2,1,3\] 交换补偿 `DequantE8M0` 的 bf16x2 输出配对 （它把 4 个字节 b0..b3 输出为 {b0,b2}、{b1,b3} 两对——§3.2 详述）， 使 scale 的 bf16 线性顺序与 B fragment 元素顺序一致。乘 scale 时经 `tCrS_dequant_view`（stride 0 广播，`w4a16_gemm_cutlass3.cuh:315-321`） 每个 scale 服务 fragment 里 4 个共享同一 (n, k-group) 的元素。

### §2.4 MMA 版权重：零重排

MMA 版的权重保持逻辑 \[N, K\] 顺序，uint8 打包，偶数 K 在低 nibble：

```
# tests/math_utils.py:136-137 :: per_token_cast_to_fp4
codes2 = codes.view(m, padded_n // 2, 2)
packed = (codes2[:, :, 0] & 0x0F) | ((codes2[:, :, 1] & 0x0F) << 4)  # 偶 K 低 nibble，奇 K 高 nibble
```

它能零重排的原因在于 FP4 MMA 的 A fragment（W 作为 A 操作数）粒度是**字节**而非 nibble： 每 thread 4 个 u32 = 16 字节，字节 (行, k8) 直接对应 W 的 (n, 2·k8 + {0,1})—— 低 nibble 偶 K、高 nibble 奇 K 恰好就是自然行主序。gmem→smem（AIU copy）→ smem→寄存器（CuTe `make_tiled_copy_A` + operand config 的 swizzle）全由标准机制完成映射， Python 侧无需任何 perm。这是 mma 路径的附带收益：**量化侧省掉整个 Marlin 式重排**。 代价是 AIU 对 smem 目标布局的 64B swizzle 粒度约束推导出 BlockK ≥ 128（§5 详述）。

### §2.5 MMA 版 scale：ldmatrix.x1 序 + meta part/section 对齐

MMA 版的 scale 是唯一重排对象。目标布局由硬件 meta 语义决定（PPU ISA 规范 §9.5.10 块缩放）：

> 一个 metadata 寄存器可以分为 4 个 part，part0 包含 T0，T4…T28 的数据，part1 包含 T1，T5…T29 的数据…… 每个 part 又可以进一步分为 4 个 section（bit\[7:0\] / \[15:8\] / \[23:16\] / \[31:24\]）。 psel 用于选择 part，ssel 选择 section。……矩阵 A 的每一行以 K=32 为单位划分为两块， 每一块与 scale\_A 中对应行的对应 section 相乘。

解读：meta-a 是**warp 集体操作数**——32 个 thread 各出 1 个 b32，共 128 字节； a-psel = p 时硬件只取 **thread ≡ p (mod 4)** 那 8 个 thread 的寄存器（8×4B = 32B）， 恰好是 16 行 × 2 个 k-group 的全部 scale。kernel 把 a-psel 设为 n\_block （`w4fa16_gemm.cuh:412`），于是 **"warp 64 行中的第 p 个 16 行块" ↔ "第 p 个 part"**， 4 次 n\_block 迭代分别用 4 个 part，零搬运。

量化侧的变换链（`math_utils.py:281-287`）：

```
# tests/math_utils.py :: quant_w4a16(d='w4fa16_mma')
scales = scales.reshape(n, k // 32)
scales = ((scales.to(torch.float32).view(torch.int32) >> 23) & 0xFF).to(torch.uint8)
# [N,K/32] -> [N/64,64,K/64,2] -> [N/64,K/64,64,2] -> [N/64, K*2]
scales = scales.reshape(n // 64, 64, k // 64, 2).permute(0, 2, 1, 3).reshape(n // 64, k * 2)
scales = scales.reshape((-1, 128))[:, _scale_perm_fp4_mma].reshape(scales.shape)

# _scale_perm_fp4_mma: (i, j) -> [2i+16j, 2i+16j+1]，i = n 低 3 位，j = n 高 3 位
for i in range(8):
    for j in range(8):
        scale_perm_fp4_mma.extend([2 * i + j * 16, 2 * i + j * 16 + 1])
```

本质与 §2.3 的转置同构：把 128 字节 tile（64 n × 2 k-group）按 (n 低 3 位, n 高 3 位) 快慢维交换，使字节序与 `ldmatrix.x1.m8n8` 的线程映射 （thread t ← 字节 \[4t, 4t+4)，行 ⌊t/4⌋ × 列对 t%4，PPU ISA 规范 §9.5.12）逐字节吻合。 kernel 侧每个 64-K 块一条 ldmatrix 直达 meta 寄存器：

```
// w4fa16_gemm.cuh :: s2r_copy_B_and_scale —— 每 64-K 块一条
uint32_t addr = cute::cast_smem_ptr_to_uint(cute::raw_pointer_cast(tCsS(_,_,fp4_k_block,stage_idx).data()));
asm volatile("ppu.tc02.ldmatrix.sync.aligned.x1.m8n8.shared.b16 {%0}, [%1];" : "=r"(d) : "r"(addr));
tCrS(0,0,fp4_k_block) = d;   // 每线程 1 个 b32 = 4 个 e8m0 part，直接作为 meta-a
```

MMA 版 scale：128 字节 tile（64 n × 2 k-group）重排 → ldmatrix → meta part/section new\[16i+2j+δ\] = old\[2i+16j+δ\] （n′ 低 3 位 ↔ 行，高 3 位 ↔ 列）转置 重排前：\[n′(64), kg(2)\] 行主序 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 0 1 2 3 4 5 6 7 0 0 1 1 2 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9 10 10 11 11 12 12 13 13 14 14 15 15 16 16 17 17 18 18 19 19 20 20 21 21 22 22 23 23 24 24 25 25 26 26 27 27 28 28 29 29 30 30 31 31 32 32 33 33 34 34 35 35 36 36 37 37 38 38 39 39 40 40 41 41 42 42 43 43 44 44 45 45 46 46 47 47 48 48 49 49 50 50 51 51 52 52 53 53 54 54 55 55 56 56 57 57 58 58 59 59 60 60 61 61 62 62 63 63 cell = 字节，标号 = n′；蓝色填充 = kg1 同一 n′ 的 kg0/kg1 两字节相邻 重排后：列组 = part b，行 = a 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 a=0 a=1 a=2 a=3 a=4 a=5 a=6 a=7 0 0 8 8 16 16 24 24 32 32 40 40 48 48 56 56 1 1 9 9 17 17 25 25 33 33 41 41 49 49 57 57 2 2 10 10 18 18 26 26 34 34 42 42 50 50 58 58 3 3 11 11 19 19 27 27 35 35 43 43 51 51 59 59 4 4 12 12 20 20 28 28 36 36 44 44 52 52 60 60 5 5 13 13 21 21 29 29 37 37 45 45 53 53 61 61 6 6 14 14 22 22 30 30 38 38 46 46 54 54 62 62 7 7 15 15 23 23 31 31 39 39 47 47 55 55 63 63 part 0 part 1 part 2 part 3 ldmatrix.x1：thread t ← 字节 \[4t, 4t+4) 行 a × 列组 b 的 4 字节 = thread (b+4a) 的 meta-a 粗黑框 = T5（b=1, a=1）一次 ldmatrix.x1 取走的 4 字节（SECT0-3） T5（b=1, a=1）的 4 字节（SECT0-3） \= s(17, kg0), s(17, kg1), s(25, kg0), s(25, kg1) 即 atom 1 的第 17、25 行（每 thread 2 行 × 2 k-group） a-psel = b ⇒ 硬件读 threads ≡ b (mod 4) 8 thread × 4B = 32B = atom b 的 16 行 × 2 k-group 全部 scale kernel 令 a-psel = n\_block：4 次 n\_block 迭代恰好用满 4 个 part

图 2-4　MMA 版 scale 的 128 字节 tile 重排与 meta 对齐。重排（\_scale\_perm\_fp4\_mma）把 \[n′(64), kg(2)\] 行主序变为 (n′ 低 3 位 ↔ 行, n′ 高 3 位 ↔ 列) 的转置布局；ldmatrix.x1 按线性序给 thread t 分发字节 \[4t, 4t+4)，于是行 a × 列组 b 的 4 字节恰好落在 thread (b+4a) 的 b32 里（bit \[7:0\]/\[15:8\]/\[23:16\]/\[31:24\] 即 SECT0-3）。a-psel = b 时硬件读取 threads ≡ b (mod 4) 的 32 字节 = atom b（16 行 × 2 k-group）的全部 scale。

图 2-4 的右侧即对齐结果：重排后 thread (b, a)（t = b + 4a）的 4 个字节恰为 s(16b+a, kg0)、s(16b+a, kg1)、s(16b+8+a, kg0)、s(16b+8+a, kg1)—— a-psel = b 时硬件读取 threads ≡ b 的 32 字节，正好覆盖 atom b 的 16 行 × 2 个 k-group。 **加载即就绪，没有任何转换指令**；对比 VALU 版 scale 路径的 "uint64 拷贝 + DequantE8M0 位拼装 + 广播乘法"（§3），这是把对齐工作全部前移到量化期的典型案例。

## §3VALU 版 kernel：寄存器内软件反量化

VALU 版的反量化全部发生在 `dequant_w4a16.cuh` 的两个函数里： `DequantW4A16<bfloat16_t, float4_t, 8>::convert`（FP4→BF16 位域重排）和 `DequantE8M0<8>::convert`（scale 字节→BF16）。它们被 `mix_gemm`（`w4a16_gemm_cutlass3.cuh:427`）在 mainloop 内每个 (k\_block, n\_idx) 上调用。本章逐条指令分析这两段位操作，再看它们如何嵌入流水线。

### §3.1 FP4 → BF16：位域重排的四步流水

一个 int32 装着 8 个 FP4（§2.2 的交错布局），目标是产出 4 个 bf16x2 寄存器 h\[0..3\]。 E2M1（1S + 2E + 1M）与 BF16（1S + 8E + 7M）的位域关系是整个变换的骨架：

E2M1 的 2-bit 指数与 1-bit 尾数经移位后，恰分别落入 BF16 指数域低 2 位与尾数域最高位 ⇒  valueBF16 = valueE2M1 × 2126

E2M1 数值 = 1.m × 2exp-1（exp=0 为次正规数 0 / 0.5），位序为 \[sign\]\[exp1\]\[exp0\]\[man\]。 其 2-bit 指数与 1-bit 尾数经 shr 6 后恰好分别落在 BF16 指数域低 2 位与尾数域最高位——两者只差指数偏置：BF16 偏置 127， E2M1 等效偏置 1，差 126。所以"移位对齐 + 拼符号"之后只需再乘 2126 就完成数值转换。四步流程（`dequant_w4a16.cuh:69-94`）：

```
// dequant_w4a16.cuh :: DequantW4A16<bfloat16_t, float4_t, 8>::convert
uint32_t fp4s[4];
fp4s[0] = source_fp4s;           // (e6,e7) 已在提取窗口
fp4s[1] = source_fp4s << 4;      // (e4,e5) → h[2]
fp4s[2] = source_fp4s << 8;      // (e2,e3) → h[1]
fp4s[3] = source_fp4s << 12;     // (e0,e1) → h[0]
for (int ii = 0; ii < 4; ++ii) {
    uint32_t tmp;
    asm("ppu.and.b32 %0, %1, %2;"  : "=r"(tmp) : "r"(fp4s[ii]), "n"(DATA_MASK));        // ① and 0x70007000
    asm("ppu.shr.b32 %0, %1, %2;"  : "=r"(tmp) : "r"(tmp),     "n"(DATA_SHIFT));       // ② shr 6
    asm("ppu.lop3.b32 %0, %1, %2, %3, %4;" : "=r"(h[3-ii]) : "r"(fp4s[ii]), "n"(SIGN_MASK), "r"(tmp), "n"(immLut)); // ③ 拼符号
}
for (int ii = 0; ii < 4; ++ii)
    asm("ppu.mul.rtte.bf16x2 %0, %1, %2;" : "=r"(h[ii]) : "r"(h[ii]), "n"(EXP_OFFSET_126));  // ④ ×2^126
```

常量表：

| 常量 | 值 | 作用 |
| :--- | :--- | :--- |
| `DATA_MASK` | `0x70007000` | 保留两个提取窗口 nibble 的 2-bit 指数 + 1-bit 尾数共 3 个数据位（bit 14-12 / 30-28） |
| `SIGN_MASK` | `0x80008000` | 两个 nibble 的符号位（bit 15 / 31），移位后恰在 bf16 符号位上 |
| `DATA_SHIFT` | `6` | 右移把指数 2 bit 送到 bf16 指数域低 2 位（bit 8-7）、尾数 1 bit 送到尾数域最高位（bit 6） |
| `immLut` | `(0xf0 & 0xcc) \| 0xaa` | LOP3 真值表 `(a&b)\|c`：一条指令拼出 sign \| data 完整 bf16 位域 |
| `EXP_OFFSET_126` | `0x7E807E80` | bf16 编码的 2126（exp=253, man=0），两通道共享同一乘数 |

以 e6（nibble \[15:12\]，`0b0101` = +3.0）为例的逐位演变如图 3-1， 每步标注 bf16 位域中各字段的落位：

e6 = 0b0101（nibble \[15:12\]，值 +3.0）的四步位域演变 E2M1 位序 = \[sign\]\[exp\]\[exp\]\[man\]（e 前 m 后）；以 32-bit 寄存器低半区（bf16 low 通道）为舞台 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0 bit 0 1 0 1 ① 输入 fp4s\[0\] nibble e6 = 0101 1 0 1 ② and 0x7000 sign 被滤掉，保留 e e m 1 0 1 ③ shr 6 e e m → 指数域低 2 位 + 尾数最高位 0 1 0 1 ④ lop3 拼 sign sign 取自原始寄存器， 一条指令拼出完整位域 0 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 ⑤ mul 2^126 \= bf16(+3.0)，指数域对齐 完成：0 10000000 1000000 S E（8 bit） M ×7 指令构成（每 int32 = 8 个 fp4，4 路并行）：3 shl + 4×(and + shr + lop3) + 4 mul = 19 条 IALU 15 条（3 shl + 4 and + 4 shr + 4 lop3）· FALU 4 条（mul.rtte.bf16x2）—— 详见 §5 成本核算

图 3-1　FP4→BF16 位域重排的逐步演变（以 e6 = 0b0101 = +3.0 为例（E2M1 位序 = \[sign\]\[exp\]\[exp\]\[man\]），展示 32-bit 寄存器低半区的 bf16 low 通道）。② and 0x70007000 滤出数据位（e,e,m）→ ③ shr 6 送入指数域低 2 位与尾数域最高位 → ④ lop3 用 (a&b)|c 真值表把符号位与数据位一次拼齐（符号直接取自移位前的原始寄存器，虚线箭头）→ ⑤ 乘 2^126 补偿指数偏置差。高半区由 e7 以相同流程同步处理。

指令账：4 路合计 **3 shl + 4×(and + shr + lop3) + 4 mul = 19 条**。 四个要点：

-   **4 路完全独立**：fp4s\[0..3\] 只依赖 source，四个 h 互不依赖，可乱序发射填满 ALU 流水；
-   **and / shr / lop3 走 IALU pipe**（12 条），mul.rtte.bf16x2 走 FALU pipe（4 条）——IALU 是这条路径的瓶颈资源（§5）；
-   **次正规数天然正确**：E2M1 的 0 全零位模式经流程仍得 0；0.5（`0b0001`）经位域映射后 ×2126 精确还原——位域关系对正规/次正规数统一成立，无需分支；
-   **乘数是立即数**：两个 bf16 通道共享一个编译期常量，无额外寄存器占用。

### §3.2 E8M0 scale → BF16：两条移位的几何

E8M0 字节 b 的数值是 2(b-127)（biased 指数）。BF16 表示 2e 的位模式是 exp = e+127、mantissa 全 0——把字节 b 塞进 bf16 的指数域（bit 14-7）就是答案：

```
// dequant_w4a16.cuh :: DequantE8M0<8>::convert —— q = {b0,b1,b2,b3}
h[2g]   = (q << 7) & 0x7F807F80u;   // b0 → low bf16，b2 → high bf16
h[2g+1] = (q & 0xFF00FF00u) >> 1;   // b1 → low bf16，b3 → high bf16
```

两条移位各自把**一对**字节同时送进 bf16 指数域：

| 操作 | 字节 → bf16 槽位 | 位几何 |
| :--- | :--- | :--- |
| `(q<<7) & 0x7F807F80` | b0 → low，b2 → high | b0 从 bit\[7:0\] 左移 7 位到 bit\[14:7\]，8 bit 指数域一次对齐；掩码清掉越界残位 |
| `(q&0xFF00FF00)>>1` | b1 → low，b3 → high | b1 从 bit\[15:8\] 右移 1 位到 bit\[14:7\]；先掩码锁住两个 16-bit 半区再移，避免相邻半区串扰 |

合计约 4 条指令（每条 = 移位 + 掩码，向量化双通道）产出 4 个 scale 的 BF16 位模式， **零算术指令**。输出配对 {b0,b2}/{b1,b3} 的"低/高字节交替"顺序，与 §2.3 量化侧 \[0,2,1,3\] 交换相互配合，使 scale 的 bf16 线性序与 B fragment 元素序对齐 （两次交换自相抵消，§2.3 图 2-3 底部示例）。

DequantE8M0：两条移位把 4 个字节送进 bf16 指数域 E8M0 字节 b 的数值为 2^(b−127)，其位模式即 bf16 指数域（bit 14–7）的输入；展示 q = {b0,b1,b2,b3} 的两个 16-bit 半区 A：q << 7，再 & 0x7F807F80　 b0/b2 进入指数域 b0 b1 b0 b1 左移 7 位 \= q << 7（残位尚未掩掉） b0 b1 \= & 0x7F807F80（清 bit15 与 bit6:0 残位） bf16 指数域 bit\[14:7\] b0 → low 通道指数域（bit7 起）；b2 同步进入 high 通道 B：q & 0xFF00FF00，再 >> 1 b1 b3 b1 b3 右移 1 位 b0/b2 与 b1/b3 分别经 A、B 路进入两个 bf16 通道的指数域 bit\[14:7\]

图 3-2　DequantE8M0 的两条移位几何（图中展示 16-bit 半区，b0/b1 占 bit\[7:0\] 原位、b2/b3 在 high 半区同步）。A 路：q<<7 把 b0 从 bit\[7:0\] 送到 bit\[14:7\]——8 bit 指数域一次对齐，掩码清残位；B 路：先 &0xFF00FF00 锁住各半区高字节再 >>1 到达同一落位。两条指令产出 4 个 scale 的 BF16 位模式，零算术指令。

### §3.3 反量化嵌入 mainloop：mix\_gemm 与 scale 的时机

mainloop 计算核心 `mix_gemm(k_block)`（`w4a16_gemm_cutlass3.cuh:427-451`）：

```
auto mix_gemm = [&](int%20k_block) {
  auto dequant = [&](int%20n_idx%2C%20int%20k_block) {
    const int* src = tCrB_i32(n_idx, _, k_block).data();      // 1 个 int32 = 8 个 fp4
    Dequant::convert(src, dst);                                          // §3.1：19 条位操作
    cute::transform(tCrB_dequant_view(_,_,n_idx,k_block),               // §3.2 的 scale bf16
                    tCrS_dequant_view(_,_,n_idx,k_block/MmaPerGroup),   //   与 8 个元素相乘
                    tCrB_dequant_view(_,_,n_idx,k_block),
                    cute::multiplies{});
  };
  for (int n_idx = 0; n_idx < 4; n_idx++) {          // warp 的 4 个 16-n atom
    dequant(n_idx, k_block);
    for (int m_idx = 0; m_idx < size<1>(accum); m_idx++)
      cute::gemm(tiled_mma, tCrA(_,m_idx,k_block), tCrB(_,n_idx,k_block), accum(_,m_idx,n_idx));  // bf16 MMA
  }
};
```

每个 k\_block（16 个 K）内的数据流与依赖关系：

mix\_gemm 单个 k\_block（16 K）内的数据流与流水重叠 4 个 n\_idx（atom）依次执行 dequant → ×scale → MMA；相邻 atom 的 dequant 与 MMA 依赖不同执行单元而重叠 atom 0 atom 1 atom 2 atom 3 dequant 19 条 ALU ×s 4 条 bf16 MMA 16 K × 16 n dequant 19 条 ALU ×s 4 条 bf16 MMA 16 K × 16 n dequant 19 条 ALU ×s 4 条 bf16 MMA 16 K × 16 n dequant 19 条 ALU ×s 4 条 bf16 MMA 16 K × 16 n t →（单个 k\_block 内的时间推进） 关键依赖与重叠 1 tCrB\_i32(n) ——dequant——▶ tCrB(n)：结果写回操作数寄存器本身（WAR：需等 atom n-1 的 MMA 读走） 2 dequant(n+1) 与 mma(n) 分属 ALU / TC，两条 pipe 并行——atom 串行循环即软件流水 3 tCrS\_dequant\_view 以 stride-0 广播：1 个 scale bf16 服务同 (n, k-group) 的 4 个元素

图 3-3　mix\_gemm 单 k\_block 的时间线。4 个 atom 依次 dequant（19 条 ALU）→ 乘 scale（4 条 FALU，stride-0 广播）→ bf16 MMA；相邻 atom 之间 dequant（标量 ALU pipe）与 mma（TC pipe）分属不同执行单元而相互重叠，atom 串行循环即构成软件流水。dequant 的输出直接写回 tCrB 操作数寄存器，零中间缓冲。

4 个 atom 依次"dequant（19 条 ALU）→ 乘 scale（4 条 FALU）→ bf16 MMA"。注意 dequant 的目标寄存器就是 `tCrB` 本身——**结果与操作数是同一块寄存器**， 零搬运；代价是引入对上一次 gemm 读 tCrB 的 WAR 依赖（编译器按序排即可，无运行时开销）。

#### §3.3.1 scale 加载与转换的时机分离

scale 每 32 个 K 才换一次（MmaPerGroup = 32/16 = 2 个 k\_block 一组），所以 smem→reg 拷贝与 e8m0→bf16 转换只在**组边界**发生 （`w4a16_gemm_cutlass3.cuh:417-425`）：

```
auto s2r_next = [&](int%20k_block) {
  auto k_block_next = (k_block + 1) % K_BLOCK_MAX;
  copy(smem_tiled_copy_B, ...);                                // B：每个 k_block 都预取
  if (k_block_next % MmaPerGroup == 0) {                       // scale：仅组边界
    copy(smem_tiled_copy_scale, ...);
    dequant_scale_e8m0(k_block_next / MmaPerGroup);             // §3.2 的位拼装，提前一组完成
  }
};
```

scale 的（拷贝 + 转换）与 B 的解包在时间上**完全分离**：组边界处的 scale 指令插入下一个 k\_block 的 B 解包窗口发射。8 个 scale 服务 2 个 k\_block × 4 atom × 8 元素 = 64 个权重元素，摊销后每 int32 不足 0.1 条指令——这就是 scale 开销在 §5 成本核算中可忽略的原因。

### §3.4 寄存器级 double buffer 与三层流水

VALU 版在 smem 多级流水（kNumStages 环形缓冲 + cp\_async\_fence/wait）之上又加了一层 **寄存器预取**：mix\_gemm 算当前 k\_block 的同时，`s2r_next` 把下一个 k\_block 的 A/B（及组边界的 scale）从 smem 拷进寄存器。任意时刻每线程持有两份 tCrA / tCrB\_i32：正在计算的一份 + 预取的一份。这层 buffer 换来： smem→reg 拷贝延迟（含 bank conflict）与 19 条解包指令完全重叠，mix\_gemm 入口处操作数永远就绪。

```
auto process_kblock_iterations = [&](int%20k_block) {
  if (k_block == K_BLOCK_MAX - 1) {          // 尾块：切换 smem stage 读指针
    update_s2r_source(smem_pipe_read);
    cp_async_wait<kNumStages-2>(); __syncthreads();
  }
  s2r_next(k_block);                        // 预取 k_block+1 的 A/B/scale → 寄存器
  if (k_block == 0) {                      // 头块：发起下一 tile 的 g2s
    g2s(k_tile_iter, smem_pipe_write); cp_async_fence();
    smem_pipe_write = smem_pipe_read;
    smem_pipe_read  = (smem_pipe_read + 1) % kNumStages;
  }
  mix_gemm(k_block);                        // 计算当前 k_block
};
```

整体是一张三层重叠的流水：gmem→smem（AIU copy / cp.async，多 stage 环形）、 smem→reg（s2r\_next 寄存器预取）、reg→MMA（dequant + gemm）。warp 分工上 A 的搬运由 warp 0 负责，`SplitAIU` 时 B+scale 交给 warp 1；反量化路径每 warp 独立，不引入任何跨 warp 同步——`__syncthreads` 只在 smem stage 边界出现， 与反量化无关。

warps\_on\_k>1 的 CTA reduce 与 N\_EXPAND 多 tile 复用属于通用 GEMM 机制，与反量化无交互，不展开；但它们决定了 warp 数与每 warp 的 atom 数（4 个 n\_idx），即 §5 指令账的基准规模。

## §4MMA 版 kernel：FP4 MMA 硬件反量化

MMA 版的核心洞察：**PPU0015 的 FP4 MMA 指令本身就是一台反量化机器**。 这条指令为 W4A4（激活也量化）设计，语义是带硬件块缩放的矩阵乘；但换个角度看， "FP4 输入 → FP32 输出 + E8M0 scale 硬件乘入"恰好就是反量化要做的全部工作。 本章先解剖指令，再看"乘单位阵"如何把这条指令变成 dequant，最后过 mainloop 编排。

### §4.1 指令解剖：ppu.tc02.mma.mx

```
ppu.tc02.mma.mx.sync.aligned.m16n16k64.row.col.f32.f4.f4.f32
    {d0..d7}, {a0..a3}, {b0..b3}, {c0..c7}, {meta_a, meta_b}, a-psel, b-psel, 0;
```

| 操作数 | 每 thread 规模 | 语义 |
| :--- | :--- | :--- |
| `{a0..a3}` | 4 × u32 = 32 个 FP4 | A 矩阵 fragment（16 M × 64 K），本 kernel 中 A = **W**（16 n 行 × 64 k） |
| `{b0..b3}` | 4 × u32 = 32 个 FP4 | B 矩阵 fragment（64 K × 16 N），本 kernel 中 B = **单位阵的一个 16 列块** |
| `{c0..c7}` / `{d0..d7}` | 8 × f32 | 累加器 C / 结果 D（16 × 16） |
| `meta_a`, `meta_b` | 各 1 × b32 | **warp 集体**操作数：32 thread 各出 1 个 b32；存 E8M0 scale，粒度 = 每行 × 32 K 一个字节（§2.5） |
| `a-psel` / `b-psel` | 立即数 0–3 | 选择 meta 的 part：a-psel = p 时硬件只读 threads ≡ p (mod 4) 的 8 个 b32（32 字节） |

计算语义（PPU ISA 规范 §9.5.10）：`D = (A · scale_A) × (B · scale_B) + C`， 其中 scale 按**每行 × 每 32 个 K** 施加——A 的每行切成两个 32-K 块， 每块乘 meta\_a 中对应行的 section（字节）。这与 MXFP4 的 group=32 逐元素对应。 对 kernel 来说，唯一需要正确设置的是：A = W、meta\_a = W 的 scale、a-psel = 当前 n\_block。

### §4.2 核心技巧：W × I = dequant(W)

令 B 为 FP4 编码的 64×64 单位阵，scale\_B ≡ 1.0，则对任一 16 列块（列 j ∈ \[16s, 16s+16)）：

C\[i, j\] = Σk=0..63 (W\[i,k\] · scale\_A\[i, k/32\]) · I\[k, j\]  =  W\[i, 16s+j\] · scale\_A\[i, (16s+j)/32\]

一次矩阵乘同时完成三件事：**数值展开**（FP4→FP32，单位阵每列仅 1 个非零元， 数学上就是"按索引取出"）、**scale 乘入**（硬件按 k 所在 group 取 meta 字节）、 **格式转换**（输出天然是 FP32）。对照 §3：三件事对应 VALU 版的 19 条位操作 + 4 条 scale 乘法——这里全部折叠进**一条** TC 指令。

#### §4.2.1 单位阵的寄存器构造（init\_identity）

64×64 的 FP4 单位阵不需要访存——kernel 开头用**编译期常量表达式**在每 thread 的 16 个 b32 寄存器里就地构造（`w4fa16_gemm.cuh:181-209`），全程常驻：

```
// w4fa16_gemm.cuh :: init_identity(lane_idx)
int group = tid >> 2;                     // 0..7：b32 内的 nibble 位置
int tid_in_group = tid & 3;
const int bit = (group >> 1) * 8 + ((group & 1) ? 5 : 1);
const uint32_t fp4_one = 1u << bit;          // E2M1 编码 1.0 = 0b0010，放到第 group 个 nibble
uint32_t b_identity[4][4] = {};              // [4 个 B 寄存器][4 个 identity 列块]
switch (tid_in_group) {                     // 每 thread 恰好放 2 个 1.0
  case 0: b_identity[0][0] = fp4_one; b_identity[2][1] = fp4_one; break;
  case 1: b_identity[0][2] = fp4_one; b_identity[2][3] = fp4_one; break;
  case 2: b_identity[1][0] = fp4_one; b_identity[3][1] = fp4_one; break;
  case 3: b_identity[1][2] = fp4_one; b_identity[3][3] = fp4_one; break;
}
```

init\_identity：每 thread 用常量构造 64×64 FP4 单位阵 fp4\_one = 1u << ((group>>1)\*8 + (group&1 ? 5 : 1))，E2M1 编码 1.0 = 0b0010；switch(tid\_in\_group) 决定放入哪个 \[寄存器\]\[列块\] fp4\_one 的 bit 位置（group → bit） group 奇偶 bit nibble fp4\_one 0 偶 1 0 0x00000002 1 奇 5 1 0x00000020 2 偶 9 2 0x00000200 3 奇 13 3 0x00002000 4 偶 17 4 0x00020000 5 奇 21 5 0x00200000 6 偶 25 6 0x02000000 7 奇 29 7 0x20000000 b\_identity\[4 寄存器\]\[4 列块\] —— lane 5（group=1, tid\_in\_group=1） 列块 0 列块 1 列块 2 列块 3 B 寄存器 0 0 0 1.0 0 B 寄存器 1 0 0 0 0 B 寄存器 2 0 0 0 1.0 B 寄存器 3 0 0 0 0 每 thread 恰好 2 个 1.0：8 group × 2 × 4 tid\_in\_group = 64 个对角元 其余 nibble 全 0（FP4 零 = 0x0，零初始化免费） W × I = dequant(W)：一次矩阵乘的三重含义 C\[i,j\] = Σ\_k ( W\[i,k\] · scale\_A\[i, k/32\] ) · I\[k,j\] = W\[i, 16s+j\] · scale\_A\[i, (16s+j)/32\] 数值展开 FP4 → FP32 单位阵每列仅 1 个非零元 \= 按索引取出 scale 乘入 硬件按 k 所在 group 取 meta\_a 字节乘入 0 条 ALU 指令 格式转换 输出天然 FP32 随后 4 条 pcnvt 转 bf16（§4.4）

图 4-1　单位阵的寄存器构造与 W×I 的数学。左：group（=lane/4）决定 fp4\_one 在 u32 内的 bit 位置——偶 group 用 bit 1、奇 group 用 bit 5，恰为 E2M1 编码 1.0（0b0010）落在对应 nibble；右：lane 5 按 switch(tid\_in\_group=1) 在 b\_identity\[0\]\[2\] 与 \[2\]\[3\] 放入两个 1.0，32 线程合计铺满 64×64 对角线。底部：乘单位阵同时完成数值展开、scale 乘入、格式转换三件事。

规模核算：8 个 group × 每 thread 2 个 1.0 × 4 线程组 = 对角线 64 个 1.0，其余全 0 （FP4 的 0.0 = 全零 nibble，零初始化免费）。16 个 b32/thread 的代价在 §5 量化。 mainloop 中按 `k_block % 4` 选取第 s 个列块：

```
// w4fa16_gemm.cuh :: fp4_mma_dequant(n_block, k_block)
Tensor s = make_tensor<uint32_t>(Int<4>{});
s(0) = tCrS(0,0,k_block / 4);  // meta-a：W 的 scale（ldmatrix 加载的 1 b32 = 4 个 e8m0 part）
s(1) = 0x7f7f7f7fu;            // meta-b：identity 的 scale，E8M0(2^0)=1.0，编译期常量
s(2) = n_block;                // a-psel：选 meta-a 的第 n_block 个 part
s(3) = 0;                      // b-psel
Tensor b = tCrID(_, k_block % 4);   // identity 第 (k_block%4) 个 16 列块
cute::mma_unpack(MMA_Atom<FP4MmaInst>{}, d, a, b, c, s);
```

### §4.3 scale：一条 ldmatrix 直达 meta 寄存器

§2.5 已给出重排推导，这里只看 kernel 侧的**加载**。scale 的 smem 布局由量化侧按 ldmatrix.x1 序预排好（图 2-4），于是每个 64-K 块只需一条指令：

```
// w4fa16_gemm.cuh :: s2r_copy_B_and_scale —— 每 64-K 块一条
uint32_t addr = cute::cast_smem_ptr_to_uint(cute::raw_pointer_cast(tCsS(_,_,fp4_k_block,stage_idx).data()));
asm volatile("ppu.tc02.ldmatrix.sync.aligned.x1.m8n8.shared.b16 {%0}, [%1];" : "=r"(d) : "r"(addr));
tCrS(0,0,fp4_k_block) = d;   // 每线程 1 个 b32 = 4 个 e8m0 part，即 meta-a 操作数
```

加载即就绪：**0 条转换指令**。对比 VALU 版 scale 路径（uint64 拷贝 + DequantE8M0 位拼装 + 广播乘法，§3.2/§3.3），meta 语义把"scale 是什么、乘在哪" 全部交给硬件。每条 ldmatrix 服务一个 warp 的 64-K 块（1024 个 fp4），均摊 0.008 条/int32。

### §4.4 末段转换：pcnvt 兼布局转置

fp4 mma 输出 fp32 的 C fragment（N-major 行交错布局），而主 GEMM 的 bf16 mma 要求 B fragment 为 K-major 行连续布局（图 2-1）。两者不同——但转换在 `quant_fp32C_to_bf16B`（`w4fa16_gemm.cuh:422-435`）里被 **免费**完成：

```
// ppu.cvt.rtte.bf16x2.f32 d, hi, lo  →  d.lo = cvt(lo), d.hi = cvt(hi)
fp32_to_bf16_helper(0, 0);   // dst[0..1] ← src[0], src[1]
fp32_to_bf16_helper(2, 4);   // dst[2..3] ← src[4], src[5]
fp32_to_bf16_helper(4, 2);   // dst[4..5] ← src[2], src[3]   ← 交换的两对
fp32_to_bf16_helper(6, 6);   // dst[6..7] ← src[6], src[7]
```

fp32→bf16 本来就必须做（4 条 pcnvt），转置只是**改变源寄存器的配对顺序** ——把 C fragment 的 (0,1),(2,3),(4,5),(6,7) 重排为 (0,1),(4,5),(2,3),(6,7)， bf16 结果直接以主 GEMM 的 B fragment 布局落位。**零额外指令、零 smem 往返、 零 barrier**（布局验证见 `tests/test_w4a16.py::debug_f4mma` 的寄存器打印路径）。

### §4.5 mainloop 编排与数据流总图

```
for_each(make_int_sequence<K_BLOCK_MAX>{}, [&] (auto k_block_ic) {
  constexpr int k_block = decltype(k_block_ic)::value;
  s2r_copy_A(stage_idx, k_block);              // A(bf16) 标准 s2r
  clear(fp4_accum);                            // C 初值 = 0：dequant 用不到累加
  for (int n_block = 0; n_block < 4; n_block++) {     // BlockN 方向 4 个 16 行块
    fp4_mma_dequant(n_block, k_block);         // 1 次 16×16×64 fp4 mma（W × I[列块]）
    quant_fp32C_to_bf16B(n_block, k_block);    // 4 条 pcnvt（兼布局转置）
    for (int m_block = 0; m_block < size<1>(accum); m_block++)
      cute::gemm(tiled_mma, tCrA(_,m_block,k_block), tCrB(_,n_block,k_block), accum(_,m_block,n_block));
  }
});
```

MMA 版 mainloop：单个 k\_block（16 K）数据流 —— 全程寄存器，零 smem 往返 sB / sScale (smem) AIU copy 装载的 stage s2r 装载 ldmatrix fp4\_tCrB（vreg） W 的 FP4 fragment tCrS（vreg） scale → meta-a tCrID（vreg 常驻） 64×64 单位阵（B 操作数） fp4 MMA × I pcnvt f32→bf16 bf16 MMA 主 GEMM n\_block 0 1 次 16×16×64 / n\_block n\_block 0 4 条 · 兼布局转置 n\_block 0 与 tCrA 乘累加 n\_block 1 1 次 16×16×64 / n\_block n\_block 1 4 条 · 兼布局转置 n\_block 1 与 tCrA 乘累加 n\_block 2 1 次 16×16×64 / n\_block n\_block 2 4 条 · 兼布局转置 n\_block 2 与 tCrA 乘累加 n\_block 3 1 次 16×16×64 / n\_block n\_block 3 4 条 · 兼布局转置 n\_block 3 与 tCrA 乘累加 accum（fp32） 主累加器 跨 k\_block 累加 → epilogue 依赖与重叠（对照 §3.3 的 VALU 版时间线） fp4 mma → pcnvt 严格串行：mma 输出即 pcnvt 输入（同 n\_block 寄存器组） 跨 n\_block 可交错：不同数据块分属不同寄存器组，编译器软件流水 fp4 mma（TC）与 bf16 mma、pcnvt（FALU）分属不同执行单元，可并行 fp4\_accum 每 k\_block clear：dequant 是纯函数，用不到累加语义

图 4-2　MMA 版 mainloop 数据流（单 warp、单 k\_block）。左列自上而下：smem 中的 W/scale 经 s2r / ldmatrix 装载为寄存器 fp4\_tCrB 与 tCrS（meta-a），tCrID 为寄存器常驻的 64×64 单位阵——三者即 fp4 mma 的全部输入；中段每 n\_block 一次“fp4 MMA × I 列块 → 4 条 pcnvt（兼布局转置）→ bf16 MMA”链，反量化结果全程在寄存器流转、零 smem 往返；右侧 accum 跨 k\_block 累加后进 epilogue。

关键性质：反量化结果（fp4\_accum → tCrB）**全程在寄存器中流转**，不经过 smem、不需要跨线程交换；唯一的同步来自原有的 smem stage 流水 （cp\_async\_fence / cp\_async\_wait / \_\_syncthreads），与 VALU 版相同。 依赖链上，fp4 mma → pcnvt 严格串行（mma 输出即 pcnvt 输入），但跨 n\_block 的不同数据块分属不同寄存器组，编译器可交错调度（软件流水）；fp4 mma 与主 GEMM 的 bf16 mma 分属不同发射路径，§5 按"保守串行"与"理想交错"给出区间。

warp 分块上，`FP4TiledMma` 以 WarpN = 64 切分 BlockN（WarpsOnN = BlockN/64）， warp tile 16×64×64：A\_act 16×64 bf16 = 2 KB，W 64×64 fp4 = 2 KB，两侧负载均衡； 每 warp 独立反量化自己负责的 W 块，无 warp 间依赖。整条路径与 VALU 版共享 DeepGemmScheduler / smem stage / epilogue 框架（§3.4），差异只集中在"反量化如何发生"。

## §5指令成本核算：两条路线的对比

前四章分析了两条路线的每个环节，本章逐项核算成本。基准单位沿用全文： **1 个 int32（8 个 FP4）→ 8 个 BF16（含 E8M0 scale 乘法）**—— 即 VALU 版一次 `Dequant::convert` + 对应的 scale 摊销，或 MMA 版的 1 次 fp4 mma + 4 条 pcnvt——一次 fp4 mma 输出 16×16 的 FP32，分到每线程为 8 个 FP32，恰由 4 条 pcnvt 转成 8 个 BF16。

### §5.1 VALU 版：逐条计数

| 环节 | 指令 | 条数 | 执行 pipe |
| :--- | :--- | :--- | :--- |
| 位域重排（§3.1） | `ppu.shl.b32`（fp4s\[1..3\] 平行移位） | 3 | IALU |
| `ppu.and.b32`（DATA\_MASK） | 4 | IALU |
| `ppu.shr.b32`（DATA\_SHIFT） | 4 | IALU |
| `ppu.lop3.b32`（拼符号） | 4 | IALU |
| 指数补偿（§3.1） | `ppu.mul.rtte.bf16x2`（×2^126） | 4 | FALU |
| scale 乘法（§3.3） | `mul.bf16x2`（transform，向量化） | ≈4 | FALU |
| scale 拼装（§3.2） | 移位+掩码（DequantE8M0，摊销） | ≈0.1 | IALU |
| **合计** |  | **≈23** | **IALU 15 + FALU 8** |

本文按 PPU 的两类标量执行单元精确计数：`mul.rtte.bf16x2`、`cvt.rtte.bf16x2.f32` 等浮点指令走 **FALU**（float ALU），整数/逻辑指令走 **IALU**；广义 VALU 包含二者。 按 IALU/FALU pipe 各 2 cycles/条折算，单 int32 的反量化占用约 **46 cycles 的 ALU pipe 时间**；且 IALU 是单发射瓶颈 pipe—— 15 条 IALU 必须串行通过同一个执行单元，无论编译器怎么乱序。

### §5.2 MMA 版：逐条计数

一次 `fp4 mma`（m16n16k64）输出 16×16 的 FP32 结果（C fragment）， 分到每线程为 8 个 FP32——恰好是"8 个 FP4 → 8 个 BF16"的一份反量化， 后续由 4 条 pcnvt 转成 8 个 BF16。完整反量化每线程 4 个 u32（8 个 int32 份） 需要 4 次 mma（identity 的 4 个列块，`k_block % 4` 选取）：

每份"8 fp4 → 8 bf16" = **1 次 fp4 mma（TC pipe）+ 4 条 pcnvt（FALU pipe）**

| 环节 | 指令 | 条数 / int32 | 执行 pipe |
| :--- | :--- | :--- | :--- |
| 数值展开 + scale | `ppu.tc02.mma.mx`（含 meta 硬件乘入） | 1 | TC |
| fp32→bf16 | `ppu.cvt.rtte.bf16x2.f32`（兼转置） | 4 | FALU |
| scale 加载 | `ppu.tc02.ldmatrix.x1`（1 条 / 1024 fp4） | ≈0.008 | — |
| **合计** |  | **1 次 mma + 4 条** | **TC 1 + FALU 4** |

### §5.3 汇总对比

指令成本核算：每 int32（8 fp4 → 8 bf16 含 scale）的反量化开销 VALU 版 · IALU 15 条 15 条 · 30 cy VALU 版 · FALU 8 条 8 条 · 16 cy MMA 版 · TC 1 次 1 次 mma · 8 cy MMA 版 · FALU 4 条 4 条 pcnvt · 8 cy VALU 版合计 ≈46 cy MMA 版保守 16 cy 0 10 20 30 40 50 ALU/TC pipe 占用（cycles，按 2 cy/条折算；fp4 mma 吞吐为 bf16 的 4 倍） IALU（瓶颈 pipe，单发射）：15 条 → 0 　·　ALU 总占用：≈46 cy → 保守 16 cy （理想 8 cy） 代价：TC 占用 +≈100%（1 次 fp4 mma / int32 ≈ 同块主 GEMM 的 bf16 mma 占用）· vreg 逼近 256 上限 · BlockK ≥ 128

图 5-1　两条路线的指令成本对比（每 int32 = 8 fp4 → 8 bf16，含 E8M0 scale）。VALU 版：IALU 15 条 + FALU 8 条 ≈ 46 cycles 全部压在标量 ALU pipe（IALU 单发射、串行通过）；MMA 版：1 次 fp4 mma（TC pipe）+ 4 条 pcnvt（FALU pipe），保守串行 16 cycles、依赖交错时向 8 cycles 逼近；代价是 TC 总占用约翻倍、vreg 常驻 identity 等反量化专用寄存器、BlockK ≥ 128 的 smem 放大。

| 项目（每 int32：8 fp4 → 8 bf16） | VALU 版 | MMA 版 |
| :--- | :--- | :--- |
| 移位/位操作（shl/and/shr/lop3） | 15 条 IALU | **0** |
| 指数补偿（×2^126） | 4 条 FALU | **0**（TC 完成） |
| E8M0 scale 乘法 | ≈4 条 FALU | **0**（meta 硬件乘入） |
| fp32→bf16（pcnvt） | 0（直接出 bf16） | 4 条 |
| TC 指令 | 0 | 1 次 fp4 mma |
| **ALU 合计** | **≈23 条 ≈ 46 cycles（IALU+FALU）** | **4 条 ≈ 8 cycles** |
| **TC 合计** | 0 | 1 次 ≈ 8 cycles |

三组数字值得展开：

-   **IALU 清零**：VALU 版的 15 条 IALU 是瓶颈 pipe（单发射、串行通过）； MMA 版 IALU 完全退出反量化——瓶颈资源被释放给地址计算与循环控制。
-   **保守 ~2.9×，理想至 ~5.75×**：同一段数据上 fp4 mma → pcnvt 存在寄存器依赖， 保守按串行累加 = TC 8 + FALU 8 ≈ 16 cycles（对比 46）；依赖允许时跨 n\_block 交错， 关键路径向 max(4, 8) = 8 cycles 逼近——实际收益介于两者之间。注意这里只算 "反量化本体"的占用，两版共用的主 GEMM bf16 mma 不在其中。
-   **代价转移到 TC**：fp4 mma 吞吐是 bf16 mma 的 4 倍（2048 vs 512 MAC/cycle）， 反量化 1 次 mma/int32（吞吐为 bf16 mma 的 4 倍）折算下来约等于同块数据主 GEMM 的 bf16 mma 占用—— 即 TC 总量约翻倍。划算与否取决于两条 pipe 的相对余量，这正是引入此方案的前提条件 （详见 5.5 适用边界）。

### §5.4 代价清单

| 代价 | 量化 | 出处 |
| :--- | :--- | :--- |
| **vreg 常驻开销** | identity 16 b32/thread 全程常驻；fp4\_tCrB 4 b32 × 4 n\_block；fp4\_accum 32 个 f32；tCrB（bf16 fragment）随 BlockN/BlockK 增长 | 逼近 256 vreg 上限，`warp_m==64` 的 GroupedFused 场景 `n_expand > 1` 即溢出，Python 配置强制 `n_expand = 1` |
| **BlockK ≥ 128 约束** | FP4 权重 uint8 进 smem，AIU 对 smem 目标布局要求 64B swizzle 粒度 → K 方向每行 ≥ 64 B → BlockK/2 ≥ 64 | sA（BF16）smem 随 BlockK 翻倍：BlockM=64, BlockK=128, stages=2 时 sA = 32 KB（原路径 BlockK=64 可减半） |
| **适用面收窄** | group\_size 固定 32；仅 PPU0015 | §6 详述 |

vreg 是最实际的约束：VALU 版的 double buffer（§3.4）在 MMA 版里放不下了—— 每 thread 的寄存器被 identity、fp4 fragment、fp4\_accum 三组"反量化专用"占用， smem→reg 只能退回单 buffer、按 stage 即时读取。这是两条路线在设计上的真实取舍： **VALU 版以寄存器换取流水深度，MMA 版以 TC 换取 ALU 但寄存器占用接近上限**。

### §5.5 适用边界（成本视角）

成本优势只在**pipe 余量错配**时最大化：反量化在 ALU 是瓶颈、TC 大量空闲的场景， 将工作从 ALU 迁移到 TC 的净收益最高。**MoE decoding 是最大受益场景** （小 M、memory bound、TC 低占用）——这也是两条路线并存的原因： VALU 版是全平台基线，MMA 版在 PPU0015 上针对 decode 形态优化。 prefill 等 TC 占比更高的场景同样适用此方案（IALU 压力同样得到释放）， 只是反量化 TC 占用与主 GEMM 的资源竞争变多，收益随之降低。

### §5.6 实测数据

**decode 形态**（GroupedMasked，28 组、m=8192、n=6144、k=4096、 expected\_m=16、group\_size=32，uniform 分布），以 INT4 版本（w4a16）为 baseline：

| 路径 | tile 配置（BlockM, BlockN, BlockK / Warp / stages） | cycles | 相对 baseline |
| :--- | :--- | :--- | :--- |
| **w4fa16\_mma**（MMA 版） | (32, 256, 128) / (32, 64, 64) / 3 | **3,167,463** | **0.70×**（快 1.43×） |
| w4a16（INT4，baseline） | (32, 256, 128) / (32, 64, 32) / 3 | 4,541,381 | 1.00× |
| w4fa16（FP4 + E8M0，VALU 版） | (32, 256, 128) / (32, 64, 32) / 3 | 6,078,111 | 1.34×（慢 1.34×） |

**prefill 形态**（GroupedFused，896 专家、num\_token=8192、topk=16、 n=768、k=3584、group\_size=32；下表为主 GEMM kernel 的 cycles，排序等辅助 kernel 三条路径开销相同、未计入）：

| 路径 | tile 配置（BlockM, BlockN, BlockK / Warp / stages） | cycles | 相对 baseline |
| :--- | :--- | :--- | :--- |
| **w4fa16\_mma**（MMA 版） | (128, 256, 128) / (64, 64, 128) / 2 | **6,168,370** | **0.79×**（快 1.26×） |
| w4a16（INT4，baseline） | (128, 256, 64) / (64, 64, 64) / 3 | 7,783,958 | 1.00× |
| w4fa16（FP4 + E8M0，VALU 版） | (128, 256, 64) / (64, 64, 64) / 3 | 9,715,992 | 1.25× |

三组观察，与 §5.1–5.5 的分析吻合：

-   **decode：MMA 版比 baseline 快 1.43×**：反量化从 "15 条 IALU + 8 条 FALU / int32"变为"1 次 fp4 mma + 4 条 pcnvt"， IALU 瓶颈让位于 TC 富余——正是 §5.3 预测的 pipe 余量再分配；
-   **prefill：MMA 版仍有 1.26× 收益，但较 decode 收窄**—— 印证 §5.5 的判断：prefill 的 TC 占比更高，反量化的 TC 占用与主 GEMM 的资源竞争变多，收益随之降低；
-   **VALU 版 FP4 在两种形态下都比 INT4 慢**（decode 1.34×、 prefill 1.25×）：FP4 → BF16 的位域重排（15 条 IALU + 4 条 FALU）比 INT4 的 magic-number 转换路径开销更大，dequant 指令数约为 INT4 路径的 1.5 倍（§5.1） ——这是 VALU 软件反量化路线的固有代价，也是 MMA 版存在的动机。

## §6局限与参考阅读

### §6.1 适用边界汇总

| 边界 | 原因 |
| :--- | :--- |
| 仅 PPU0015（`__HGGC_ARCH__ >= 150`） | `ppu.tc02.mma.mx` / `ldmatrix` 是第二代 Tensor Cell 指令，PPU001 无此能力；VALU 版两代通用 |
| group\_size 固定 32 | 硬件块缩放粒度 = 每行 × 32 K 一个 e8m0 字节，与 MX 标准绑定 |
| N、K 需为 64 的倍数（MMA 版） | scale tile 以 64 行 × 64 K 为重排单位；权重自然序布局的对齐要求 |
| BlockK ≥ 128 且 warp\_k ∈ {64, 128} | AIU 64B swizzle 约束（§5.4） |

### §6.2 参考阅读

-   PPU ISA 规范：[T-Head 开发者文档 · PPU ISA](https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=39&chapterId=197)——§9.5.10（块缩放 / meta / psel）、§9.5.11（tc02.mma）、§9.5.12（tc02.ldmatrix），指令语义的权威出处
-   MARLIN 论文：[MARLIN: Mixed-Precision Auto-Regressive Parallel Inference](https://arxiv.org/abs/2408.11743)——W4A16 的路线起点
-   DefTruth：[\[LLM 推理优化\] WINT8/4-(03): LOP3 指令详解及 INT4 转 FP16/BF16 分析](https://zhuanlan.zhihu.com/p/657073857)——位操作反量化的经典分析（通用 GPU 侧对照）
-   CalebDu：[Nvidia CuTe 实战 — Marlin W4A16 Gemm](https://zhuanlan.zhihu.com/p/18984902584)——CuTe 实现 W4A16 GEMM 的实践参考
