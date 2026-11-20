---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=37"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=37"
source_title: "PPU Triton 编程指南"
source_id: "blog-37"
source_kind: "technical-blog"
source_date: "2026-09-29"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "e23f5f75377ea28fc7ee39ca6499fc6f34da6b41e346f90b7d2e3942f8ec4706"
tags:
  - "Triton"
  - "PPU"
  - "tl.aiu_load"
  - "block-pointer"
  - "tensor-descriptor"
  - "tiling"
  - "GEMM"
  - "Flash-Attention"
  - "FusedMoE"
  - "AIU"
---
# PPU-Triton 编程指南 {#ppu-triton-编程指南}

## 目录 {#目录}

- [1. Triton language](#1-triton-language)
  - [1.1. `tl.aiu_load`](#11-tlaiu_load)
  - [1.2. 传统使用方法](#12-传统使用方法)
  - [1.3. `make_block_ptr`](#13-make_block_ptr)
  - [1.4. `make_tensor_descriptor`](#14-make_tensor_descriptor)
- [2. Tiling](#2-tiling)
  - [2.1. GEMM](#21-gemm)
  - [2.2. FA](#22-fa)
  - [2.3. FusedMoE](#23-fusedmoe)

## 1. Triton language {#1-triton-language}

常规 Triton Language 的使用可以参见 OpenAI 文档：<https://triton-lang.org/main/python-api/triton.language.html>。

PPU Triton 从 3.2.0 版本开始增加了 `aiu_load` API，用来通过 AIU 实现 global memory 到 shared memory 的 data movement，加速数据传输。

### 1.1. `tl.aiu_load`：通过 AIU 加速 global→shared 的 data movement {#11-tlaiu_load}

```python
def aiu_load(
    pointer,
    offsets=(0, 0),
    block_shape=(0, 0),
    shape=(0, 0),
    dtype=void,
    order=(1, 0),
    _builder=None,
):
    """
    Return a tensor of data whose values are loaded from memory at location
    defined by `pointer`:

    (1) If `pointer` is a single element pointer, offsets, block_shape and
        shape should be provided. In this case:
        - `pointer` is the start address of a memory tensor
        - `offsets` is a 2D value indicating the offset of the loading tile
        - `block_shape` is a 2D value indicating the tile size
        - `shape` is a 2D value indicating the memory tensor size
        - `order` is the order of the original data format

    (2) If `pointer` is a block pointer defined by `make_block_ptr`, a tensor
        is loaded.
    """
```

当需要从一个 `M × K` 的 global data tensor 中加载一个 `BLOCK_M × BLOCK_K` tensor 到 shared memory 时，可以使用 `tl.aiu_load`。要求如下：

- tensor 必须 32-byte aligned；
- `BLOCK_M` 是 16 的倍数；
- `BLOCK_K` 方向上的数据是 32 byte 的倍数；
- block tensor 在 M、K 方向上都连续；
- 支持的数据类型：
  - 810/810E：b16；
  - M890：b16、b8。

![aiu_load 从 Data Tensor 选取 Block Tensor 的布局示意](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763948906/23ed3c660696082f79ff2a1245b4331f/aiu_load_description.webp)

*图 1：`src_ptr`、M/K offset 与 `BLOCK_SIZE_M × BLOCK_SIZE_K` 的关系。*

### 1.2. 传统使用方法 {#12-传统使用方法}

下面是直接给 `tl.aiu_load` 传入指针、offset、block shape 和原 tensor shape 的 matmul kernel 示例：

```python
@triton.jit
def matmul_kernel_aiu(a_ptr, b_ptr, c_ptr,
                      stride_am, stride_ak,
                      stride_bk, stride_bn,
                      stride_cm, stride_cn,
                      M, N, K,
                      BLOCK_SIZE_M: tl.constexpr,
                      BLOCK_SIZE_N: tl.constexpr,
                      BLOCK_SIZE_K: tl.constexpr):
    pid = tl.program_id(axis=0)
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    pid_m = pid % num_pid_m
    pid_n = pid // num_pid_m
    offs_am = pid_m * BLOCK_SIZE_M
    offs_bn = pid_n * BLOCK_SIZE_N
    offs_k = 0
    accumulator = tl.zeros(
        (BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32
    )

    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        a = tl.aiu_load(
            a_ptr,
            [offs_am, offs_k],
            [BLOCK_SIZE_M, BLOCK_SIZE_K],
            [M, K],
            tl.float16,
        )
        b = tl.aiu_load(
            b_ptr,
            [offs_k, offs_bn],
            [BLOCK_SIZE_K, BLOCK_SIZE_N],
            [K, N],
            tl.float16,
        )
        accumulator = tl.dot(a, b, acc=accumulator)
        offs_k += BLOCK_SIZE_K

    c = accumulator.to(tl.float16)

    # Write back the block of the output matrix C with masks.
    offs_cm = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_cn = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    c_ptrs = (
        c_ptr
        + stride_cm * offs_cm[:, None]
        + stride_cn * offs_cn[None, :]
    )
    c_mask = (offs_cm[:, None] < M) & (offs_cn[None, :] < N)
    tl.store(c_ptrs, c, mask=c_mask)
```

### 1.3. `make_block_ptr` {#13-make_block_ptr}

使用 `make_block_ptr` 可以让 `aiu_load` 更简单，需要 `make_block_ptr`、`aiu_load` 和 `advance` 配合使用。

```python
def make_block_ptr(base: tensor, shape, strides, offsets,
                   block_shape, order, _builder=None):
    """
    Returns a pointer to a block in a parent tensor

    :param base: The base pointer to the parent tensor
    :param shape: The shape of the parent tensor
    :param strides: The strides of the parent tensor
    :param offsets: The offsets to the block
    :param block_shape: The shape of the block
    :param order: The order of the original data format
    """
```

- `tl.aiu_load`：接受 `make_block_ptr` 的结果作为输入；
- `tl.advance`：推进 block pointer。

```python
def advance(base, offsets, _builder=None):
    """
    Advance a block pointer

    :param base: the block pointer to advance
    :param offsets: the offsets to advance, a tuple by dimension
    """
```

组合示例：

```python
@triton.jit
def matmul_kernel_aiu(a_ptr, b_ptr, c_ptr,
                      stride_am, stride_ak,
                      stride_bk, stride_bn,
                      stride_cm, stride_cn,
                      M, N, K,
                      BLOCK_SIZE_M: tl.constexpr,
                      BLOCK_SIZE_N: tl.constexpr,
                      BLOCK_SIZE_K: tl.constexpr):
    pid = tl.program_id(axis=0)
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    pid_m = pid % num_pid_m
    pid_n = pid // num_pid_m

    offs_am = pid_m * BLOCK_SIZE_M
    offs_bn = pid_n * BLOCK_SIZE_N
    offs_k = 0
    accumulator = tl.zeros(
        (BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32
    )

    a_tensor_ptr = tl.make_block_ptr(
        a_ptr, (M, K), (stride_am, stride_ak),
        (offs_am, offs_k), (BLOCK_SIZE_M, BLOCK_SIZE_K), (1, 0)
    )
    b_tensor_ptr = tl.make_block_ptr(
        b_ptr, (K, N), (stride_bk, stride_bn),
        (offs_k, offs_bn), (BLOCK_SIZE_K, BLOCK_SIZE_N), (1, 0)
    )

    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        a = tl.aiu_load(a_tensor_ptr)
        b = tl.aiu_load(b_tensor_ptr)
        accumulator = tl.dot(a, b, acc=accumulator)
        a_tensor_ptr = tl.advance(a_tensor_ptr, (0, BLOCK_SIZE_K))
        b_tensor_ptr = tl.advance(b_tensor_ptr, (BLOCK_SIZE_K, 0))

    c = accumulator.to(tl.float16)
    offs_cm = pid_m * BLOCK_SIZE_M
    offs_cn = pid_n * BLOCK_SIZE_N
    c_tensor_ptr = tl.make_block_ptr(
        c_ptr, (M, N), (stride_cm, stride_cn),
        (offs_cm, offs_cn), (BLOCK_SIZE_M, BLOCK_SIZE_N), (1, 0)
    )
    tl.store(c_tensor_ptr, c)
```

### 1.4. `make_tensor_descriptor` {#14-make_tensor_descriptor}

Triton 3.4 之后增加了 `tl.make_tensor_descriptor`，用于兼容 Hopper 以下版本中 descriptor 的使用。

```python
def make_tensor_descriptor(
    base: tensor,
    shape: List[tensor],
    strides: List[tensor],
    block_shape: List[constexpr],
    padding_option="zero",
    _semantic=None,
) -> tensor_descriptor:
    """Make a tensor descriptor object

    :param base: the base pointer of the tensor, must be 16-byte aligned
    :param shape: A list of non-negative integers representing the tensor shape
    :param strides: A list of tensor strides. Leading dimensions must be
        multiples of 16-byte strides and the last dimension must be contiguous.
    :param block_shape: The shape of block to be loaded/stored from global memory
    """
```

在真武 810/810E/M890 中使用 `tl.make_tensor_descriptor` 时，条件符合即可自动 promotion 到 AIU load。

## 2. Tiling {#2-tiling}

对于 Triton kernel，除了算法本身，tile 大小对性能也至关重要；不同 tiling 可能带来数倍性能差异。CTA 中 tile shape 决定 per-warp 的计算规模：

- tile 过大可能引起寄存器 spill；
- tile 过小会增加 memory 数据量；
- PPU 上寄存器 spill 后，需要写回 global memory，使用时再从 global memory 加载；
- 寄存器紧张也会加重 data dependency。

PPU assembler 中 spill 的 pattern 较明显，通常能看到很多以 `tid` 为 index 的 `vmem.st`/`vmem.ld`。也可以使用：

```text
hg objdump --dump-resource-usage
```

查看 device binary 中的寄存器和 stack 情况，以判断是否发生 spill。

![PPU assembler 中的 spill 指令（示例一）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954596/c89e6d25213b2789363600777f29ec72/spill-assembly-1.png)

![PPU assembler 中的 spill 指令（示例二）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954765/791b91164d644977dc67b02c9dc89f5b/spill-assembly-2.png)

![Device binary 的寄存器和 stack 资源用量](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954467/d71d41af752c2501f68a2f9e8644e685/resource-usage.png)

*图 2：`vmem.st`/`vmem.ld` spill pattern 与 `dump-resource-usage` 输出。*

Spill 不一定总会带来坏处：部分 LOOP 外 spill 对性能影响较小，并可能减小 LOOP 内寄存器压力；有时大 tile 下的轻微 spill 仍优于小 tile。这体现了 kernel 在 memory 与 computation 之间的平衡。不同算法适合的 tiling 不同：有条件时应 auto-tuning，在合理范围内搜索 best config；条件有限时，也可以根据算法分析估算合理 config。

下面以 GEMM、FA、FusedMoE 为例。

### 2.1. GEMM {#21-gemm}

GEMM kernel 主要加载两个 tensor 并进行 dot 运算。Tile size 不改变计算量，但会影响 global memory 和 shared memory 的总加载量。示例 case：`03-matrix-multiplication.py`。

理论公式：

$$
\mathrm{AsyncCopy}_{TOTAL}
= M N K\left(\frac{1}{\mathrm{BLOCK\_SIZE\_M}}+
\frac{1}{\mathrm{BLOCK\_SIZE\_N}}\right)
$$

$$
\mathrm{SharedMem\_LOAD}_{TOTAL}
= M N K\left(\frac{\mathrm{warpN}}{\mathrm{BLOCK\_SIZE\_N}}+
\frac{\mathrm{warpM}}{\mathrm{BLOCK\_SIZE\_M}}\right)
$$

测试初始配置：

```python
triton.Config(
    {
        'BLOCK_SIZE_M': 128,
        'BLOCK_SIZE_N': 128,
        'BLOCK_SIZE_K': 128,
        'GROUP_SIZE_M': 1,
    },
    num_stages=3,
    num_warps=4,
)
```

在 H20/810/810E/M890 上测试时，PPU 出现寄存器 spill。调节 `BLOCK_SIZE_M/N/K` 和 `num_warps` 可以改变 tiling、减少 per-warp 计算量与寄存器使用，但不同配置的效果不同。

| BLOCK_SIZE_M | BLOCK_SIZE_N | BLOCK_SIZE_K | 810/810E num_warps | 810/810E vreg | 810/810E spill | 810E cycle | 810 cycle | M890 vreg | M890 spill | M890 cycle | H20 num_warps | H20 vreg | H20 spill | H20 cycle |
|---:|---:|---:|---|---:|---:|---:|---:|---:|---:|---:|---|---:|---:|---:|
| 128 | 128 | 128 | 4 (2,2) | 256 | 220 | 3,693,842 | 1,141,503 | 256 | 216 | 1,426,959 | 4 (4,1) | 252 | 0 | 1,919,300 |
| 128 | 128 | 64 | 4 (2,2) | 252 | 0 | 1,956,555 | 606,838 | 256 | 32 | 882,274 | 4 (4,1) | 240 | 0 | 1,866,795 |
| 128 | 128 | 32 | 4 (2,2) | 242 | 0 | 1,913,344 | 615,265 | 240 | 0 | 799,211 | 4 (4,1) | 220 | 0 | 1,864,303 |
| 128 | 128 | 128 | 8 (4,2) | 242 | 0 | 2,190,008 | 657,067 | 256 | 0 | 940,144 | 8 (8,1) | 147 | 0 | 1,924,074 |
| 64 | 128 | 128 | 4 (2,2) | 236 | 0 | 2,937,386 | 875,883 | 256 | 0 | 1,735,209 | 4 (4,1) | 206 | 0 | 1,867,202 |
| 128 | 64 | 128 | 4 (4,1) | 220 | 0 | 2,826,971 | 823,440 | 256 | 0 | 1,198,320 | 4 (4,1) | 152 | 0 | 1,877,455 |

*表 1：GEMM 配置及跨设备测试结果。*

#### 2.1.1. `BLOCK_SIZE_K` {#211-block_size_k}

由公式可知，global memory load（GM_load）和 shared memory load（SHM_load）的数据量与 `BLOCK_SIZE_K` 无关；仅调节 K 不影响 GM_load 与 SHM_load 数据量。

##### 2.1.1.1. 810E：以 K=32、`num_warps=4` 的最佳结果为 baseline {#2111-810e}

对 `M128N128K128`、`M128N128K64` 与 `M128N128K32` 的比较表明：

1. 在 spill 时，减小 `BLOCK_SIZE_K` 可以降低 VREG 使用量，同时不影响 AsyncCopy 和 SHM_load 数据量；LOOP 数增加还可能改善 pipeline。
2. 设置 tiling 时可选择较小的 `BLOCK_SIZE_K`，从而在有限资源下选择更大的 `BLOCK_SIZE_M/N`。

![810E 上 M128N128K128 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763950174/6147748733ce8535619730133a5177ee/gemm-810e-k128.png)

![810E 上 M128N128K64 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763950332/c0e798a72eb5f2e0c6b4723bd19da014/gemm-810e-k64.png)

*图 3：810E 上不同 `BLOCK_SIZE_K` 的性能计数对比。*

##### 2.1.1.2. M890：以 K=32、`num_warps=4` 的最佳结果为 baseline {#2112-m890}

从指令观察，AsyncCopy 和 SHM_load 数据量与上述分析一致，改变 K 时变化不大；K=128 时因 spill 严重，增加了 `vmem.ld` 指令。

![M890 上 M128N128K128 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951755/08b2536a6aeec124d543e4e048bab645/gemm-890-k128-profile.png)

![M890 上 M128N128K128 的指令统计](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951577/f3e5f7256997bd5ff36cbf34b6ae933d/gemm-890-k128-instructions.png)

![M890 上 M128N128K64 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763952114/c66f7f5ee6faf796c57f7f5e04e9dd20/gemm-890-k64-profile.png)

![M890 上 M128N128K64 的指令统计](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951962/7d6ee5f5bd7358dab4e64f9b78615e84/gemm-890-k64-instructions.png)

- 相对 K=32，memory dependency 更严重，是因为 LOOP 内存在少量 spill；
- compute dependency 更严重，主要是寄存器紧张，使地址计算频繁复用同一寄存器，指令难以 schedule 开。

![M890 上不同 BLOCK_SIZE_K 的性能对比](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951428/f831e91b19123894732c3933fd67adc7/gemm-890-k-comparison.png)

![M890 上不同 BLOCK_SIZE_K 的反汇编对比](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951287/cbe571f75f34e1e88144178b759889e8/gemm-890-k-assembly.png)

*图 4：M890 上 K 配置的 memory/compute dependency、指令统计与反汇编分析。*

#### 2.1.2. Warp {#212-warp}

##### 2.1.2.1. 810E {#2121-810e}

将 `numWarps` 增加到 8 时，根据公式，SHM_load 数据增加 50%。按 `(4, 2)` 分配时，operand A load 不变，operand B 增加一倍：

$$
\mathrm{operandA}=MNK\frac{\mathrm{warpN}}{\mathrm{BLOCK\_SIZE\_N}}
$$

$$
\mathrm{operandB}=MNK\frac{\mathrm{warpM}}{\mathrm{BLOCK\_SIZE\_M}}
$$

![810E 上 numWarps=8 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763951064/c95253745fd604ea3e5ee77cc7443106/gemm-810e-warp8.png)

##### 2.1.2.2. M890 {#2122-m890}

Warp 增大一倍后 spill 消失，但 SHM_load 数据增加 50%。表现为 memory 利用率上升、compute 利用率下降。Memory 数据量增加且 throughput 较满时，单条指令 latency 会变长，memory dependency 增加，进而降低 compute 利用率。

![M890 上 numWarps=4 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763952824/fb18654e111fc5f2ecd39a582b921b8e/gemm-890-warp4.png)

![M890 上 numWarps=8 的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953029/10b4a04fde410096d5c309d89cbd0313/gemm-890-warp8.png)

*图 5：Warp 数变化对 810E 与 M890 的影响。*

#### 2.1.3. `BLOCK_SIZE_M/N` {#213-block_size_mn}

比较以下 tiling：

- `M128N128K64W4 (2, 2)`；
- `M64N128K128W4 (2, 2)`；
- `M128N64K128W4 (4, 1)`。

LOOP 内 MMA 指令数相同，共 64 条：

$$
128\times128\times64/(16\times16\times16)/4=64
$$

对应数据量为：

| 配置 | AsyncCopy_TOTAL | TSM_LOAD_TOTAL |
|---|---:|---:|
| M128N128K64W4 (2,2) | $MNK/64$ | $MNK/32$ |
| M64N128K128W4 (2,2) | $3MNK/128$ | $3MNK/64$ |
| M128N64K128W4 (4,1) | $3MNK/128$ | $3MNK/64$ |

`BLOCK_SIZE_M/N` 减半与 `BLOCK_SIZE_K` 减半都能减少 LOOP 内 MMA 指令，从而降低寄存器使用；但减小 `BLOCK_SIZE_M/N` 会同时影响 AsyncCopy 和 SHM_load，在此比较中两类 memory load 都增加 50%。因此，即使 MMA 运算量不变，tiling 引起的 memory load 变化也会改变 kernel 性能。

##### 2.1.3.1. 810E {#2131-810e}

![810E 上 M/N tiling 的性能对比（第一组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763950545/4a28aa631d60a43de50d765e7498b584/gemm-810e-mn-comparison-1.png)

![810E 上 M/N tiling 的性能对比（第二组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763950766/79024f25ac1f3a7d703877694a56c869/gemm-810e-mn-comparison-2.png)

![810E 上 M/N tiling 的性能对比（第三组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763950925/36e473b6b52f895ea69112b25f44f52b/gemm-810e-mn-comparison-3.png)

*图 6：810E 上不同 M/N tiling 的 memory load 与性能对比。*

##### 2.1.3.2. M890 {#2132-m890}

调节 `BLOCK_SIZE_M` 或 `BLOCK_SIZE_N` 在 810/810E 上表现类似，但在 M890 上不同。减小 `BLOCK_SIZE_M` 比减小 `BLOCK_SIZE_N` 产生更多 L2 cache miss，增加 LLC 到 L2 的数据量。810E/810/M890 都有类似趋势，但由于 LLC 带宽不同，性能影响不同，在 M890 上更明显。

![M890 上 M/N tiling 的 memory path（第一组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763952268/22d6bacb3157840b7a327555600f033e/gemm-890-memory-path-1.png)

![M890 上 M/N tiling 的 memory path（第二组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763952425/b5947ac85ca7de989326ef400159f4e2/gemm-890-memory-path-2.png)

![M890 上 M/N tiling 的 memory path（第三组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763952661/98b7a7d4481979c44b4cc1a75578f4cc/gemm-890-memory-path-3.png)

*图 7：M890 的 memory chart 与 LLC/L2 路径变化。*

#### 2.1.4. PPU Devices {#214-ppu-devices}

不同 device 在这些 case 上略有差异，但变化趋势与理论分析基本一致。

- GEMM 主要是 computation-bound；按 tensor cell 算力比，810 是 810E 的 3.6 倍，cycle 观察已达到约 3 倍，memory 影响相对较小。
- 810E 的 memory 与 810 相同，但 tensor cell 更少，因此 memory 相对冗余；相同 case 的 memory 利用率也反映了这一点。
- memory 带宽越满，load latency 越长，stall sync 越多；810 与 810E 的对比可观察到该现象。

![810 与 810E 的性能对比（第一组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953203/13a49d85650d5d8907ee206cf549238a/gemm-ppu-device-comparison-1.png)

![810 与 810E 的性能对比（第二组）](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953387/14167585b36e4513754ee0c16aeb6c15/gemm-ppu-device-comparison-2.png)

*图 8：PPU device 的 throughput 与 stall 对比。*

#### 2.1.5. H20 {#215-h20}

Tiling 变化对 H20 cycle 的影响较小。H20 使用 WGMMA，因此 shared memory 与寄存器使用方式不同。

| 配置 | Shared Store From Global Load | Shared Load 布局 |
|---|---:|---|
| M128N128K32W4 (4,1) | 4,259,480 | ![M128N128K32W4](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953849/fc601fbfe1bda916b2e22b499727ce34/h20-m128-n128-k32-w4.png) |
| M128N128K64W4 (4,1) | 4,325,767 | ![M128N128K64W4](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954011/2b5c27964e2d452b68dd5f7cab369633/h20-m128-n128-k64-w4.png) |
| M128N128K128W4 (4,1) | 4,456,448 | ![M128N128K128W4](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953543/339093c9197032a0b87d6de6d26fea3e/h20-m128-n128-k128-w4.png) |
| M128N128K128W8 (8,1) | 4,456,448 | ![M128N128K128W8](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763953715/b3996d75d71cbfd1d0458eddfc118bdc/h20-m128-n128-k128-w8.png) |
| M64N128K128W4 (4,1) | 6,684,672 | ![M64N128K128W4](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954329/b2816ca2fbd6db5b1025eb23668d5f91/h20-m64-n128-k128-w4.png) |
| M128N64K128W4 (4,1) | 6,684,672 | ![M128N64K128W4](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763954179/3c4f918429d812ba1936c1ae0fdcfa0b/h20-m128-n64-k128-w4.png) |

*表 2：H20 配置及 Shared Load 布局。*

WGMMA 指令示例：

```text
HGMMA.64X128X16.F32 R88, gdesc[UR12].tnspB, RB88
```

WGMMA 规定 matrix B 必须位于 shared memory 中，并可在 warp 之间共享；matrix A 可以在寄存器或 shared memory 中。按 TSM load 公式：

$$
\mathrm{matrixA}=MNK\frac{\mathrm{warpN}}{\mathrm{BLOCK\_SIZE\_N}}
$$

$$
\mathrm{matrixB}=MNK\frac{\mathrm{warpM}}{\mathrm{BLOCK\_SIZE\_M}}
$$

matrix B 从 TSM 中共享，不需要 load。若 `BLOCK_SIZE_N=128`、`warpN=1`，且每条指令加载 `16×16 FP16`：

$$
\mathrm{Inst}=4096^3/128/16/16=2M
$$

与 NCU 数据一致；若 `BLOCK_SIZE_N=64`，则 `Inst=4M`，同样与 NCU 数据一致。

**H20 小结：**

1. `BLOCK_SIZE_M/N` 会影响 AsyncCopy 数据量；
2. `BLOCK_SIZE_N` 会影响 TSM load request 数据量；
3. GEMM 主要 bound 在 computation。H20 通过 MMA 降频限制算力，使 memory 带宽相对 tensor core 算力充足；相同 MMA 计算量下，即使 memory 操作增加 50%，性能下降也不多。

**GEMM 结论与硬件特点：**

- H20：MMA latency 更长，能 hide 更多 memory 操作，因此在一定范围内调节 tiling 对性能影响很小；例如 `BLOCK_SIZE_M/N` 减半后，AsyncCopy 和 shared-memory load 增加，但性能波动不大。
- 810/810E：虽然 kernel bound 在 computation，但 memory 影响仍较大；`BLOCK_SIZE_M/N` 改变后，AsyncCopy 和 shared-memory load 的变化会造成较大性能变化。
- M890：由于硬件架构差异，MMA 能 hide 的 memory 操作减少，因此比 810/810E 更敏感。

文档随后给出了 H20/810/810E/M890 上的 best config，并总结 tiling 建议：

1. 发生 spill 时：
   1. 优先减小 `BLOCK_SIZE_K`；它主要影响 pipeline/staging，对 memory 的影响很小。
   2. 其次增大 `warpNum`；这不影响 global-to-shared 数据量，只影响 shared-memory load。
   3. 最后再减小 `BLOCK_SIZE_M/N`；M/N 会同时影响 AsyncCopy 和 SharedMem_Load。
2. 需要增大 tiling 时，优先增加 `BLOCK_SIZE_M/N`。

### 2.2. FA {#22-fa}

以下以 `causal=False` 分析。

#### 2.2.1. AsyncCopy 的数据 {#221-asynccopy-的数据}

Q 只需加载一次：

$$
Q=\frac{\mathrm{seq\_len}}{\mathrm{BLOCK\_M}}\cdot
\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{BLOCK\_M}\cdot\mathrm{HEAD\_DIMS}
=\mathrm{seq\_len}\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}
$$

因此 Q 的 AsyncCopy 数据量只与 Q 的大小有关。

$$
K=\frac{\mathrm{seq\_len}^2\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}}
{\mathrm{BLOCK\_M}}
$$

$$
V=\frac{\mathrm{seq\_len}^2\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}}
{\mathrm{BLOCK\_M}}
$$

由此可见，`BLOCK_M` 越大，AsyncCopy 数据量越小；其他参数不影响。

#### 2.2.2. Shared Memory Load 的数据 {#222-shared-memory-load-的数据}

仅考虑 chained dot，warp 分配需满足：

```text
(warpM, warpN) = (numWarps, 1)
```

原算法中 Q 在 LOOP 外，但 Triton 为缓解寄存器压力，将 Q lowering 到 LOOP 内：

$$
Q=\frac{\mathrm{seq\_len}^2\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}}
{\mathrm{BLOCK\_N}}
$$

$$
K=\mathrm{seq\_len}^2\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}
\frac{\mathrm{numWarps}}{\mathrm{BLOCK\_M}}
$$

$$
V=\mathrm{seq\_len}^2\cdot\mathrm{batch}\cdot\mathrm{HEADS}\cdot\mathrm{HEAD\_DIMS}
\frac{\mathrm{numWarps}}{\mathrm{BLOCK\_M}}
$$

按照 Triton 默认方式：

1. `BLOCK_N` 影响 Q 的 SharedMem_load；增大 `BLOCK_N` 会减少 Q 的 load 数据量。
2. `numWarps` 与 `BLOCK_M` 影响 K/V 的 SharedMem_load；`numWarps` 越大，load 越多；`BLOCK_M` 越大，load 越少。

#### 2.2.3. 测试数据分析 {#223-测试数据分析}

测试条件：

```text
BATCH, N_HEADS, HEAD_DIM = 4, 32, 64
stage = 3
num_warps = 4
```

| BLOCK_M | BLOCK_N | num_warp | 810E reg | 810E spill | 810E cycle | M890 reg | M890 spill | M890 cycle |
|---:|---:|---|---:|---:|---:|---:|---:|---:|
| 128 | 64 | 4 (4,1) | 256 | 52 | 207,005,534 | 256 | 0 | 72,937,652 |
| 64 | 64 | 4 (4,1) | 160 | 0 | 242,731,919 | 208 | 0 | 81,772,287 |
| 128 | 32 | 4 (4,1) | 226 | 0 | 216,138,610 | 224 | 0 | 81,677,332 |

观察 810E：大 tile `(128,64)` 虽发生 spill，但性能仍较好，主要因为 spill 位于 LOOP 外；较大的 `BLOCK_M` 使 AsyncCopy 与 SHM_load 较少，较大的 `BLOCK_N` 使 SHM_load 与 VALU 数量较少。

固定 `BLOCK_M` 比较 `BLOCK_N`。Triton default 模式下 Q 被 lowering 到 LOOP 内，SHM_load 数量会随 `BLOCK_N` 改变：

$$
\frac{Q_{TSMLoad,\,N=32}}{Q_{TSMLoad,\,N=64}}
=\frac{1/32+4/128}{1/64+4/128}=\frac{4}{3}
$$

因此 `tsm_ld_ncom` 增加 $1/3$，与 ACU 数据相符。

![810E 上 BLOCK_N 变化后的指令统计](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763949217/b00784a427d4bd08fb214e381f78ef7d/fa-810e-block-n-instructions.png)

![810E 上 BLOCK_N 变化后的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763949384/020666374efd5c45e3651166a7771e51/fa-810e-block-n-profile.png)

M890 与 810E 趋势一致；M890 中 `tsm.ld` 合并统计，`BLOCK_N=32` 时增长 20%：

$$
\frac{1/32+4/128+4/128}{1/64+4/128+4/128}=\frac{12}{10}
$$

![M890 上 BLOCK_N 变化后的指令统计](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763949650/8661e4109340473a828799329b4e5297/fa-890-block-n-instructions.png)

![M890 上 BLOCK_N 变化后的性能数据](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763949830/a781bc446beee4bc52dafef694996fd0/fa-890-block-n-profile.png)

*图 9：`BLOCK_N=32` 与 `BLOCK_N=64` 在 810E 和 M890 上的指令与性能差异。*

实验中，810 上性能下降约 5%，M890 上约 12%。M890 对 TSM 更敏感；此外 M890 的 12% drop 也与 VALU 增加有关。`causal=False` 相对更偏 computation-bound。

**结论：**`BLOCK_N` 不影响 global-memory load，对 shared-memory load 的影响相对较小，同时可以减少 LOOP 内 MMA 指令数和寄存器使用。因此发生 spill 时，可以优先调节 `BLOCK_N`。

##### 2.2.3.1. VALU 的影响 {#2231-valu-的影响}

实际测量发现，调整 `BLOCK_SIZE_N` 时部分 VALU 指令也显著变化。例如 `BLOCK_SIZE_N` 减半后，`vmul`/`fma`/`exp` 指令数量改变，尤其 `vmul` 增加一倍。

1. LOOP 中的乘法：

   ```python
   tl.max(qk, 1) * qk_scale
   ```

   `tl.max(qk, 1)` 对 `qk` 的 dim=1 求最大值。原 `qk` 为 `[BLOCK_SIZE_M, BLOCK_SIZE_N]`，降维后为 `[BLOCK_SIZE_M]`。该 `v.mul.f32` 与 LOOP 次数相关，因此 `BLOCK_SIZE_N` 减半会使它增加一倍。

2. LOOP 中还有：

   ```python
   acc = acc * alpha[:, None]
   ```

   `acc` 大小为 `[BLOCK_SIZE_M, HEAD_DIM]`，执行次数同样与 LOOP 次数相关，因此 `BLOCK_SIZE_N` 减半会使这里的 `v.mul.f32` 增加一倍。

3. 其他 `fma`/`exp` 的增长与 `vmul` 类似：部分 tensor 大小与 `BLOCK_SIZE_N` 无关，但执行次数与 LOOP 数相关；减半 `BLOCK_SIZE_N` 会增加相关计算量。

![FA LOOP 中 qk_scale、alpha 与 accumulator 更新的代码](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763949991/db009993ec12b1646cb5042e5d8a1347/fa-valu-code.png)

*图 10：FA 中受 LOOP 次数影响的 VALU 代码位置。*

#### 2.2.4. AIU {#224-aiu}

FA 中 Q/K/V 都是连续 data block，可使用 AIU load。FA 在 PPU 上使用 AIU 可获得 **10+%** 的性能提升。

### 2.3. FusedMoE {#23-fusedmoe}

MoE 的 Triton 实现先重排 token 的 topK，得到每个 expert 的 token；再将每个 expert 的 token 数 padding 到 `BLOCK_SIZE_M` 的整数倍，之后执行 GEMM。

若 `BLOCK_SIZE_M` 过大，尤其在 token 数较小时，一个 block 中可能大部分是无效 padding，造成大量无效 memory 与 computation。若要增大 tiling，应尽量避免增大 `BLOCK_SIZE_M`，可改为增大 `BLOCK_SIZE_N`。

`fused_moe_kernel` 的接口与说明节选如下：

```python
@triton.jit
def fused_moe_kernel(
    # Pointers to matrices
    a_ptr,
    b_ptr,
    c_ptr,
    b_bias_ptr,
    a_scale_ptr,
    b_scale_ptr,
    topk_weights_ptr,
    sorted_token_ids_ptr,
    expert_ids_ptr,
    num_tokens_post_padded_ptr,
    # Matrix dimensions
    N,
    K,
    EM,
    num_valid_tokens,
):
    """
    Implements the fused computation for a Mixture of Experts (MOE) using
    token and expert matrices.

    Key Parameters:
    - A: token tensor with shape (*, K).
    - B: stacked MOE weight tensor with shape (E, N, K).
    - C: output cache tensor with shape (M, topk, N).
    - sorted_token_ids: sorted token indices, repeated topk times and arranged
      by expert index.
    - expert_ids: expert index for each block, selecting the matrix from B.

    The sorting of `sorted_token_ids` by expert index and padding ensures
    divisibility by BLOCK_SIZE_M, maintaining consistent block matrix
    multiplication across blocks processed by the same expert.
    """
```

输入定义：

- A：token tensor `(M, K)`，M 是 token 数量，K 是每个 token 的 input feature dimension；
- B：expert tensor `(E, N, K)`，每个 expert 的 weight 为 `(N, K)`，N 是 output feature dimension。

执行 token tensor 与对应 expert tensor 的 GEMM 时：

- **load A**：加载当前 tile（`BLOCK_SIZE_M × K`）对应的 `token_id`。`token_id` 位于 `sorted_token_ids_ptr`；根据它加载 A 中对应 token。padding 的 `token_id` 用 `token_mask` zero-fill。由于 token 数据在 M 方向不连续，不能使用 AIU load。
- **load B**：从 `expert_ids_ptr` 加载当前 tile 的 `expertID`，计算对应 expert tensor offset 并加载数据。`moe_align_block_size` 已保证一个 tile 对应同一 expert，因此数据在 N/K 方向连续，可以使用 AIU load。部分 kernel 使用 AIU 可获得 **15+%** 的性能提升。

#### 2.3.1. 算法总结 {#231-算法总结}

- 使用较小的 `BLOCK_SIZE_M`，减少 token padding 造成的数据与计算冗余；
- 使用较大的 `BLOCK_SIZE_N` 和较小的 `BLOCK_SIZE_K`，在不 spill 时获得较好的 memory 数据表现；
- 若 kernel spill，先判断是否有较多冗余计算：
  - token 数远大于 expert 数时，较大的 `BLOCK_SIZE_M` 通常不会造成太多冗余；
  - token 数较小时，`BLOCK_SIZE_M` 可选择接近 `topk * num_tokens / num_expert`，不要偏离太多；
- MoE kernel 内部有两个 GEMM。不考虑冗余计算时，`BLOCK_SIZE_M/N/K` 与其他 GEMM kernel 类似；发生 spill 时仍优先减小 `BLOCK_SIZE_K`。
