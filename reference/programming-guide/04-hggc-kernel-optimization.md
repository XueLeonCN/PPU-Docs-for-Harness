---
source_url: "https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=39&chapterId=1044"
source_api_url: "https://developer.t-head.cn/api/developer/document/chapter/content?projectId=39&chapterId=1044"
source_title: "T-Head SAIL HGGC 算子性能优化实战指南"
source_id: "chapter-1044"
source_kind: "programming-guide"
source_date: null
source_version: "T-Head SAIL SDK v2.2.1"
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "434150e6f000e42602190f84c67b44772f494bb354e9ff4826cca55132b13d7a"
tags:
  - "kernel-optimization"
  - "occupancy"
  - "TLP"
  - "ILP"
  - "launch-bounds"
  - "coalesced-access"
  - "bank-conflict"
  - "group-conflict"
  - "cache-policy"
  - "AIU"
  - "tensor-core"
  - "register-spill"
  - "Asight"
---
# T-Head SAIL HGGC 算子性能优化实战指南

本指南面向 T-Head SAIL 真武 PPU 的算子（kernel）开发者，讨论 kernel 内部的性能优化，即在一个已启动的核函数内，如何提高 PPU 算力、内存带宽和执行资源的利用率。

主机-设备数据传输、多卡通信、HGGC 图、启动级延迟等 kernel 外的主题不在本指南范围内，请查阅编程指南对应章节。

---

## 目录

1. [性能优化总览](#性能优化总览)
2. [PPU 硬件模型速览](#ppu-硬件模型速览)
3. [第一部分：最大化资源利用率](#第一部分最大化资源利用率)
4. [第二部分：最大化内存吞吐量](#第二部分最大化内存吞吐量)
5. [第三部分：最大化指令吞吐量](#第三部分最大化指令吞吐量)
6. [第四部分：消除 Stack 与寄存器溢出](#第四部分消除-stack-与寄存器溢出)
7. [第五部分：用 Asight 定位优化空间](#第五部分用-asight-定位优化空间)
8. [Kernel 优化检查清单](#kernel-优化检查清单)
9. [附录](#附录)

---

## 性能优化总览

kernel 的性能取决于能否充分利用 PPU 的几类资源。kernel 内部的优化手段可以归结为下面几个目标：

| 优化目标 | 主要手段 | 对应章节 |
| :--- | :--- | :--- |
| 最大化资源利用率 | block/grid 配置、TLP / ILP、Occupancy、寄存器分配 | [第一部分](#第一部分最大化资源利用率) |
| 最大化内存吞吐量 | 合并访问、共享内存、缓存 / AIU、`__restrict__`、消除 spill | [第二部分](#第二部分最大化内存吞吐量)、[第四部分](#第四部分消除-stack-与寄存器溢出) |
| 最大化指令吞吐量 | 高效计算单元、精度 / fast math、原子 / 控制流 | [第三部分](#第三部分最大化指令吞吐量) |

### 这些目标都围绕隐藏延迟

PPU 与 CPU 的设计思路不同：CPU 依靠大缓存和复杂控制逻辑降低单线程延迟，PPU 依靠大量线程隐藏延迟。因此 kernel 优化的核心思路是让计算单元持续有任务执行，避免 warp 因等待访存、依赖或同步而停顿。

隐藏延迟有两种互补途径：

- TLP（Thread-Level Parallelism，线程级并行）：同时驻留足够多的 warp，一个 warp 停顿时切换到另一个。TLP 由 Occupancy（占用率）决定。
- ILP（Instruction-Level Parallelism，指令级并行）：让单个 warp 内部有足够多互相独立的指令，可以流水执行。

两者的目的都是隐藏延迟。高性能 kernel 通常需要兼顾 TLP 与 ILP，而不是只提高占用率。

### 优化分析的检查顺序

优化一个 kernel 时，建议按下面的顺序进行：

1. 用 profiler（Asight）定位瓶颈方向：这个 kernel 卡在访存、计算还是占用率？
2. 判断优化空间与大致方向：profiler 能反映出“哪里慢”，但不能反映出“具体怎么改”。
3. 对照上述目标选择手段：内存吞吐、指令吞吐，还是资源利用率？
4. 改写，重新 profile，验证正确性与性能。

Asight 等工具可以定位 kernel 的优化空间和大致方向（例如占用率只有 25%、访存效率低、存在大量分支发散），但不会给出具体的优化方法。具体手段来自后续章节总结的编程模式与经验。

### 优化手段与预期收益

不同层次的优化，投入产出比差异很大。建议先做上层，再做下层：

| 优化层级 | 预期收益 | 本文对应章节 |
| :--- | :--- | :--- |
| 算法 / 数据结构级（改变计算方式、减少工作量） | 高 | 贯穿全文 |
| 内存访问优化（合并访问、共享内存、避免 spill 到 local） | 较高 | [第二部分](#第二部分最大化内存吞吐量)、[第四部分](#第四部分消除-stack-与寄存器溢出) |
| 执行配置 / 占用率（block/grid、寄存器、`launch_bounds`） | 中 | [第一部分](#第一部分最大化资源利用率) |
| 指令级优化（计算单元选择、精度、控制流、快速数学） | 低 | [第三部分](#第三部分最大化指令吞吐量) |

---

## PPU 硬件模型速览

在介绍具体优化手段前，需要先建立 PPU 的概念模型。后续的优化都基于这些硬件事实。

### 执行层次：Grid、Block、Warp 与 Thread

- Grid（网格）
  - Block（线程块）：整体调度到某一个 CU 上执行
    - Warp（线程束，32 线程）：SIMT 执行的基本单位
      - Thread（线程）：拥有独立 VREG 与程序计数器

- CU（Compute Unit，计算单元）：PPU 的基本计算单元，相当于其他架构中的流式多处理器，内部有本地寄存器文件、L1 数据缓存、共享内存和若干功能单元。线程块整体调度到一个 CU 上执行，块内能使用共享内存和 `__syncthreads()` 协作。
- CE（Compute Engine）：CU 之上的一层硬件结构。一个 CE 包含多个 CU，它们共享同一块 L2 缓存。
- Warp（线程束）：32 个线程一组，以 SIMT（单指令多线程）方式执行，即 warp 内线程同时执行同一条指令；走不同分支的线程会被屏蔽，这就是 warp 分歧（divergence）。
- Warp Lane（通道）：warp 内的线程编号 0~31。线程块中的线程按固定规则分配到各 warp，合并访问和 bank conflict 的分析都以此为基础。

### 计算能力（Compute Capability）与架构

每款 PPU 有一个计算能力（CC）编号，指明其支持的特性与硬件参数。当前支持的架构可通过 `hgcc -arch=<value>` 指定：

| `hgcc -arch` | `__HGGC_ARCH__` 宏 | 代号 | 说明 |
| :----------- | :----------------- | :--- | :--- |
| `ppu_10`     | `100`              | ppu001  | 第一代真武 PPU 架构 (如真武 810E) |
| `ppu_15`     | `150`              | ppu0015 | 第二代真武 PPU 架构 (如真武 M890) |

在设备代码中可用 `__HGGC_ARCH__` 宏做条件编译，为不同架构选择不同实现（例如不同的 `__launch_bounds__`、是否启用 FP8 路径等）：

```cpp
#if __HGGC_ARCH__ >= 150
    // ppu0015 专属优化路径（如 FP8 / 预取提示）
#else
    // ppu001 通用路径
#endif
```

### 代表性硬件参数（真武 810E）

下表给出一款代表性 PPU 配置的关键资源参数，用于建立数量级概念。具体数值随架构和产品而异，运行时应通过 `hggcDeviceGetAttribute` / `hggcGetDeviceProperties` 查询真实设备：

| 资源 | 数值 | 优化含义 |
| :--- | :--- | :--- |
| CU 数量 | 64 | grid 大小的参考基准（见拖尾效应） |
| 每 CU 最大 Warp 数 | 64 | 占用率的分母（= 2048 线程/CU） |
| 每 CU VREG 总量 | 128K | 寄存器预算，决定能同时驻留多少 warp |
| 单 Warp 最大 VREG 使用量 | 255 | 超过即溢出到 local memory |
| 每 CU 共享内存 | 256 KB | 独立于 L1，块内协作的高速暂存 |
| Warp Size | 32 | block 大小必须是它的倍数 |

!!! note "VREG 与 SREG"

    PPU 有两类寄存器：向量寄存器 VREG（Vector Register），warp 内每个 thread 各有一份；标量寄存器 SREG（Scalar Register），整个 warp 内所有 thread 共享。SREG 适合存放 warp 内统一的值，例如循环上界、统一分支条件、公共地址偏移。合理搭配 VREG 和 SREG 能提高 kernel 效率，这也是后文强调使用 uniform condition 的硬件原因。

### 内存层次结构

从快到慢，PPU 的内存层次依次为：

1. VREG / SREG 寄存器：片上，CU 内，作用域为单线程 / warp。
2. Shared Memory 共享内存：片上，CU 内，256KB/CU，作用域为线程块。
3. L1 Cache：片上，每 CU 独有。
4. L2 Cache：片上，CE 内所有 CU 共享。
5. LLC（Last Level Cache）：所有 CE 共享，HBM 前最后一级。
6. Global Memory 全局内存：片外 HBM。

越靠上速度越快、容量越小、作用域越窄；越靠下越慢，作用域扩展到整个设备。

各内存空间的作用域与生命周期：

| 内存类型 | 作用域 | 生命周期 | 物理位置 |
| :------- | :----- | :------- | :------- |
| Register 寄存器 | Thread 线程 | 核函数 | 片上 CU |
| Shared 共享内存 | Block 块 | 核函数 | 片上 CU |
| Local 局部内存 | Thread 线程 | 核函数 | 片外设备内存（寄存器溢出的去处）|
| Global 全局内存 | Grid 网格 | 应用程序 | 片外设备内存 |

!!! note "Local Memory"

    局部内存虽然名为局部，但物理上驻留在片外全局内存中，访问延迟和带宽与全局内存相当。当寄存器不够用（溢出），或使用了编译期无法确定索引的数组时，变量会落到 local memory。因此减少 local memory 访问本身就是一个重要的优化目标（详见[第四部分](#第四部分消除-stack-与寄存器溢出)）。

---

## 第一部分：最大化资源利用率

资源利用率部分检查 kernel 是否充分利用了 PPU 提供的 CU、寄存器、共享内存和 warp 槽位。

### 1.1 选择合适的 block 与 grid

执行配置 `kernel<<<grid, block, smem, stream>>>(...)` 是 kernel 性能的第一步。下面两条要同时满足。

#### 原则 1：block 大小是 32 的倍数，最好是 2 的整数次幂

```cpp
// 推荐：256 = 32 × 8 = 2^8
kernel<<<gridDim, 256>>>(...);

// 不推荐：100 不是 32 的倍数
//    最后一个 warp 只有 4 个活跃通道，其余 28 个通道全程闲置
kernel<<<gridDim, 100>>>(...);
```

warp 大小固定为 32。如果 block 线程数不是 32 的倍数，最后一个 warp 会有部分通道（lane）在整个执行过程中闲置，该 warp 的功能单元利用率和访存效率都会下降。同时取 2 的幂也便于地址计算（用移位代替乘除）和索引推导。

#### 原则 2：让 grid 大小接近 `CU 数量 × 2 的整数倍`，避免拖尾效应

```
假设 PPU 有 64 个 CU，每个 CU 可同时驻留 4 个 block（一波 = 256 个 block）：

grid = 256 的整数倍  →  ████████████████  所有 CU 满载，无拖尾
grid = 256 + 32      →  ████████████████░  最后一波只有 32 个 block，
                        64 个 CU 里 32 个在执行、32 个闲置，即拖尾  
```

上例中最后一波只有 32 个 block，64 个 CU 有一半闲置，这就是拖尾效应（tail effect）。让 grid 接近 `CU 数 × 每 CU 常驻 block 数` 的整数倍，可以让所有 CU 尽量同时结束。**block 数远多于 CU 数时拖尾影响较小；对 block 数刚好略多于一波的 kernel，拖尾影响最大，需要重点调整。**

### 1.2 平衡 TLP 与 ILP

如前所述，隐藏延迟依靠 TLP 和 ILP 两方面，只追求其中一个通常达不到最优。

#### TLP：由占用率驱动

TLP 受 Occupancy 影响，Occupancy 又受下列因素制约：

- block 大小
- 每线程寄存器（VREG）数量
- 每 block 共享内存用量

任一资源成为瓶颈，都会限制 CU 上能同时驻留的 warp 数（详见 [1.3 占用率](#13-占用率occupancy)）。

#### ILP：避免单 warp 工作量过少

占用率高不代表性能够高。如果一个 warp 每次只 `load`/`store` 一个 float、只做一个 float 加法，即使占用率 100%，单 warp 内部的流水线也无法填满，计算单元会大量闲置。

```cpp
// ILP 差：每个线程只处理 1 个元素，warp 内几乎没有独立指令可流水
__global__ void lowILP(const float* in, float* out, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < N) out[i] = in[i] * 2.0f;
}

// ILP 好：每个线程处理 多个元素，
//    多次 load/乘法 互相独立，可在 warp 内流水重叠
__global__ void highILP(const float* in, float* out, int N) {
    int stride = gridDim.x * blockDim.x;
#pragma unroll 4
    for (int i = blockIdx.x * blockDim.x + threadIdx.x; i < N; i += stride) {
        out[i] = in[i] * 2.0f;   // 配合向量化进一步提高 ILP
    }
}
```

!!! note "ILP"

    warp 内可以流水重叠的独立指令，来源通常是多次不相关的访存、多条不依赖的算术运算，以及循环展开（`#pragma unroll`）后暴露的并行迭代；打断 ILP 的停顿则主要来源于寄存器依赖、访存延迟和内存栅栏或同步（memory fence / sync）。

#### 权衡：占用率与寄存器

有时主动降低占用率、换取每线程更多寄存器反而更快。更多寄存器可以缓存更多中间结果、支持更深的 ILP，减少重复访存。这类权衡没有通用公式，需要靠 profile 实测（见[第五部分](#第五部分用-asight-定位优化空间)）。

### 1.3 占用率（Occupancy）

占用率 = CU 上活跃 warp 数 / CU 支持的最大 warp 数。高占用率有助于掩盖访存延迟，但并非越高越好，过高的占用率可能带来更多寄存器压力和共享内存争用。

#### 估算占用率

以一个代表性 CU（最大 2048 线程 = 64 warp、128K 个 VREG、256KB 共享内存）为例：

```
kernel 配置：block = 256 线程，每线程 80 个寄存器，每 block 用 8KB 共享内存

1. 线程数限制：   2048 / 256           = 8   个 block
2. 寄存器限制：   131072 / (256 × 80)  = 6.4，取 6 个 block
3. 共享内存限制： 256KB / 8KB          = 32  个 block
                                          （受每 CU 最大 block 数上限约束）
4. 实际常驻 block = min(8, 6, 32)      = 6   个 block

占用率 = (6 × 256) / 2048 = 75%
瓶颈资源 = 寄存器；想提高占用率就要降低每线程寄存器数
```

占用率由最紧张的那一项资源决定。上例中寄存器是瓶颈，共享内存还有余量。

#### 用 Asight 采集占用率

Asight 会直接给出理论占用率、实际占用率及各资源的 block 限制：

| 指标 | 示例值 | 资源限制项 | 示例值 |
| :--- | :--- | :--- | :--- |
| Theoretical Occupancy [%] | 100 | Block Limit Registers [block] | 32 |
| Theoretical Active Warps per CU | 64 | Block Limit Shared Mem [block] | 64 |
| Achieved Occupancy [%] | 0.09 | Block Limit Warps [block] | 16 |
| Achieved Active Warps per CU | 0.06 | Block Limit CU [block] | 64 |

!!! note "理论与实际占用率"

    上例中理论占用率 100%，但实际占用率仅 0.09%，这通常意味着 grid/block 太小、kernel 太短，或存在严重的不均衡调度。理论与实际差距明显时，应优先检查执行配置和负载均衡，而不是继续优化指令。

#### 占用率与性能的关系

- 占用率提升到某个拐点后，继续提高占用率的收益明显递减；拐点位置因 kernel 和 PPU 架构而异，没有精确阈值；
- 100% 占用率不一定最优；
- 对访存密集型 kernel，较高占用率有利于隐藏延迟；对计算密集型 kernel，适当降低占用率换取更多寄存器和更强 ILP 常常更好。

### 1.4 用 launch_bounds 与寄存器上限引导编译器

编译器会用启发式方法在寄存器用量、寄存器溢出、指令数量之间求平衡。可以通过启动界限向编译器传递额外的约束。

#### `__launch_bounds__`：声明最大块大小

```cpp
// 单参数：保证至少一个 block 能驻留，避免请求资源过多导致启动失败
__global__ void __launch_bounds__(256) kernel(float* p) { /* ... */ }

// 双参数：额外指定期望的每 CU 最小常驻 block 数，
//         编译器据此收紧寄存器预算（可能在某些情况下提升性能）
__global__ void __launch_bounds__(256, 4) kernel(float* p) { /* ... */ }
```

- `maxThreadsPerBlock`：kernel 启动时使用的每块最大线程数（编译为 `.maxntid` TIX 指令）。
- `minBlocksPerMultiprocessor`（可选）：期望的每 CU 最小常驻 block 数（编译为 `.minnctapersm` TIX 指令）。

如果实际启动的每块线程数超过 `maxThreadsPerBlock`，kernel 将启动失败。

!!! note

    为保证与未来硬件和工具包的前向兼容，并确保至少有一个 block 能在 CU 上运行，建议每个 kernel 都带上单参数 `__launch_bounds__(maxThreadsPerBlock)`。双参数版本的最佳 `minBlocksPerMultiprocessor` 值应通过 profile 实测确定。

#### 按架构定制启动界限

```cpp
#if __HGGC_ARCH__ >= 100
  #define MY_KERNEL_MAX_THREADS 512
  #define MY_KERNEL_MIN_BLOCKS  3
#else
  #define MY_KERNEL_MAX_THREADS 256
  #define MY_KERNEL_MIN_BLOCKS  2
#endif

__global__ void __launch_bounds__(MY_KERNEL_MAX_THREADS, MY_KERNEL_MIN_BLOCKS)
kernel() { /* ... */ }
```

!!! note

    在主机代码里 `__HGGC_ARCH__` 是未定义的，所以不要在启动配置里直接写 `kernel<<<blocks, MY_KERNEL_MAX_THREADS>>>()` 来复用上面依赖架构宏的常量，它会按 `#else` 分支取值。启动用的每块线程数应当用一个与 `__HGGC_ARCH__` 无关的常量，或在主机端根据 `hggcGetDeviceProperties` 的结果在运行时决定。

#### `__maxnreg__` 与 `--maxrregcount`

```cpp
// 逐 kernel 精确控制：指定单线程可分配的最大寄存器数（编译为 .maxnreg）
__global__ void __maxnreg__(32) kernel(float* p) { /* ... */ }
```

```bash
# 整个文件粒度：限制所有 __global__ 函数的每线程寄存器数
hgcc --maxrregcount=32 kernel.hg -o kernel.out
# 带 __maxnreg__ 限定符的 kernel 会忽略此选项
```

- `__launch_bounds__()` 与 `__maxnreg__()` 不能用于同一个 kernel。
- 限制寄存器可能诱发溢出到 local memory，但也可能让更多 block 得以调度、提高占用率，从而带来净性能提升，需要实测权衡。
- 用户可以用 `-res-usage`（`--resource-usage`）查看每个 kernel 的寄存器、共享内存、局部内存用量。

---


## 第二部分：最大化内存吞吐量

多数 kernel 的瓶颈在访存而非计算。内存吞吐量优化关注两件事：
- 提高每笔访存事务中有效数据的比例；
- 让数据尽量靠近计算单元。

本部分的优化要点概览：

- 2.1 全局内存合并访问（Coalesced Access，合并单位 128B）
- 2.2 避免低效全局访问模式（尽量用宽读写，避免写回再读、部分写）
- 2.3 共享内存：避免 Bank Conflict 与 Group Conflict
- 2.4 缓存策略、`__restrict__` 与只读加载
    - L1 / L2 / LLC 多级缓存：改善局部性
    - `__restrict__`：给编译器更多别名信息，换取更多优化空间
    - `__ldg()`：只读数据走只读缓存路径
    - 精细 Cache policy：through
    - Prefetch：实验性提升 cache 命中率（不一定有效）
- 2.5 异步拷贝与 AIU：让数据搬运不阻塞计算
    - async_cp / memcpy_async：元素级异步拷贝（global 到 shared）
    - AIU（1/2/3D）：张量数据直搬共享内存

### 2.1 全局内存合并访问（Coalesced Access）

设计算法和数据结构，使同一 warp 内的线程访问连续的内存地址。这样硬件可以把多个访问合并成少数几个内存访问，提高全局内存带宽利用率。

PPU 的合并单位是 128 字节。一个 warp 的单次访问，理想情况是恰好落在一条 128 字节的 cache line 内。

#### 好的合并与差的合并

```
较好的合并：warp 内单次访问连续地址，命中较少 cacheline
   thread:  0    1    2    3   ...  31
   addr:   [0]  [4]  [8]  [12] ... [124]   →  1 条 128B cache line 全覆盖

较差的合并：整体读到的数据可能相同，但每笔 request 都跨多条 cache line
   thread:  0        1        2        3    ...
   addr:   [0]    [1024]   [2048]   [3072]  ...  →  每个线程各占一条 line，
                                                    一笔 request 触及 32条 line
```

较差的合并访问读取的数据总量可能和好的访问完全相同，但由于地址分散，每笔 request 都要访问多条 cache line，实际搬运的字节数成倍增加，有效带宽随之下降。合并访问优化就是减少一次 warp 访存所触及的 cache line 数量。

#### 代码示例

```cpp
// 合并访问：连续线程访问连续地址
__global__ void coalesced(const float* __restrict__ in, float* __restrict__ out) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    out[idx] = in[idx];      // thread 0→[0], thread 1→[1] ... 完全合并
}

// 跨步访问：连续线程访问间隔地址，跨步越大，合并越差
__global__ void strided(const float* __restrict__ in, float* __restrict__ out, int stride) {
    int idx = (blockIdx.x * blockDim.x + threadIdx.x) * stride;
    out[idx] = in[idx];      // thread 0→[0], thread 1→[stride] ... 有效带宽 ≈ 1/stride
}
```

#### 二维数据：行优先与列优先

HGGC/C++ 数组是行优先存储的。应让变化最快的线程维度对应内存上连续的维度：

```cpp
// 行访问：threadIdx.x 对应列（连续），warp 内地址连续
float v = matrix[threadIdx.y * width + threadIdx.x];

// 列访问：threadIdx.x 对应行，warp 内地址间隔 width，严重不合并
float v = matrix[threadIdx.x * width + threadIdx.y];
//    需要转置或经由共享内存中转（见 2.4）
```

#### 小粒度访问（sector）

与合并访问相关：最好让单条读写指令访问的地址集中，减少所触及的缓存块数量。

- ppu001：cache line 为 128 字节。若单条读写指令的地址分散在多条 128B line 上，性能下降。目标是减少触及的 128B line 数。
- ppu0015：建议同样集中访问，但优化目标变为减少所触及的 64 字节块数量。

### 2.2 避免低效的全局访问模式

除合并外，PPU 上还有这些经验。

#### 1）使用更宽的读写指令（int4 / float4）

一次搬运更多字节，可以减少指令数、提高每事务的有效载荷：

```cpp
// 向量化：一条指令搬 16 字节（4 × float）
__global__ void vecCopy(const float4* __restrict__ in, float4* __restrict__ out) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    float4 v = in[i];
    v.x *= 2.0f; v.y *= 2.0f; v.z *= 2.0f; v.w *= 2.0f;
    out[i] = v;
}
```

!!! note "ppu001 的 generic address 性能问题"

    在 ppu001 上，当编译器无法从地址推导出指针属于 global 还是 shared（即作为 generic address 访问）时，更宽的读写指令会被拆散到 32bit，向量化收益消失（ppu0015 已优化此问题）。规避方法：
    - 指针来源明确（直接用 `__global__` 传入的指针，而不是在多种地址空间间来回转换）；
    - 避免把 global 指针和 shared 指针混在同一个变量里做运行时分支选择；
    - 必要时使用 tix "ppu.ld.global/.shared" 来直接声明指针地址；
    - 需要时用 `-res-usage` 或反汇编（[Binary Utilities](/docs_center/doc_detail/index.html?projectId=39&chapterId=1061)）确认生成了宽访存指令。

#### 2）避免写回全局再读回，用共享内存做中转

```cpp
// 用片上 shared memory 中转中间结果
__shared__ float tmp[BLOCK_SIZE];
tmp[threadIdx.x] = partial;    // 写 shared
__syncthreads();
result = tmp[threadIdx.x ^ 1]; // 读 shared
```

如果把中间结果写到 global 再读回来使用，一来一回是两次片外访存，还往往伴随部分写；

- 减少先写数据到 global memory、再读回来使用的行为，改用速度更快的 shared memory 完成这部分算法。
- 如果 shared memory 不够大、必须使用 global memory，最好保证 warp 内的访问是 128 字节对齐且完整覆盖 128 字节（full cover）。

#### 3）出现部分写（partial write），用 shared memory 代替 temp

部分写指一个 128B 内存事务里只写入了其中一部分字节，会造成读-改-写式的带宽浪费。

出现部分写时，可用 shared memory 代替 temp：先在共享内存里把数据拼成完整、对齐的块，再一次性整块写回 global，避免零散的部分写。

### 2.3 共享内存：Bank Conflict 与 Group Conflict

共享内存是块内线程协作的高速暂存（片上，256KB/CU 量级），访问延迟远低于 global，常用于归约、转置、tiling 等协作算法。在ppu上使用共享内存要避开两类冲突：Bank Conflict 和 Group Conflict。

共享内存有两种分配方式：

#### 静态与动态分配

```cpp
// 静态分配：编译期确定大小
__global__ void k1() {
    __shared__ float s[1024];
    // ...
}

// 动态分配：启动时用第三个参数指定字节数，kernel 内用 extern __shared__
__global__ void k2() {
    extern __shared__ float s[];
    // ...
}
k2<<<grid, block, smemBytes>>>();   // smemBytes 决定动态共享内存大小
```

#### Bank Conflict

- PPU 的共享内存 bank 宽度为 4 字节，共 32 个 bank。

当一个 warp 内的多个线程同时访问落在同一个 bank 的不同地址时，这些访问会被串行化，即 bank conflict。访问同一地址是例外，硬件可以广播，不算冲突。

```cpp
// 32 路 bank conflict：列访问 32×32 数组，stride=32，全部落在同一 bank
__shared__ float tile[32][32];
float v = tile[threadIdx.x][col];   // thread 0..31 命中 bank (0*32..31*32)%32 = 同一 bank

// 加 1 列填充：把 stride 从 32 改成 33，错开 bank
__shared__ float tile[32][33];
float v = tile[threadIdx.x][col];   // 现在 32 个线程落在 32 个不同 bank，无 bank conflict
```

消除 bank conflict 最常用、代价最小的方法是填充（padding）：给二维共享数组多加一列（`[N][N]` 改为 `[N][N+1]`），即可让原本对齐到同一 bank 的列访问错开。

#### Group Conflict

- 除避免 bank 冲突外，还宜在分组（groups）之间实现负载均衡，降低分组冲突。
- ppu001：共享内存被划分为 2 个分组，每个分组每拍可提供 128 字节吞吐带宽；每组内连续地址跨度为128B，因此若多个线程访问的地址间隔为 256 字节的整数倍，这些地址会映射到同一个分组，多线程同时访问同一分组时引发分组冲突，请求被序列化处理，延迟增加。
- ppu0015：分组数从 2 增加到 4，每组内连续地址跨度从 128B 扩展到 1024B，冲突概率随之降低。

以 ppu001（2 个分组）为例：三个请求分别访问地址 0、256B、512B。 256B 的整数倍间隔让它们都映射到分组 0，请求被串行化，产生冲突。规避方法是让请求地址呈线性连续分布，自然分散到不同分组，实现组间负载均衡。

### 2.4 缓存策略、`__restrict__` 与只读加载

PPU 有 L1 / L2 / LLC 多级缓存。提高内存吞吐除了用好缓存，还要给编译器足够的别名信息。

#### `__restrict__`：便于编译器优化

指针别名（两个指针指向重叠内存）会阻碍代码重排和公共子表达式消除。给指针加 `__restrict__`，是向编译器承诺该指针指向的内存在其生命周期内只通过它访问：

```cpp
// 编译器无法假设 a/b/c 不重叠，a[0]+b[1] 可能被重复计算、无法重排
__device__ void f(const int* a, const int* b, int* c);

// 加了 __restrict__，编译器可自由重排和公共子表达式消除，减少指令数
__device__ void f(const int* __restrict__ a,
                  const int* __restrict__ b,
                  int* __restrict__ c);
```

!!! note "__restrict__"

    `__restrict__` 是对单个指针的非别名承诺，加了它，编译器就知道经由该指针的访问不与别的指针重叠。收益是渐进的：把彼此可能别名的指针加得越全，编译器能做的重排、公共子表达式消除、向量化就越多；漏掉某个，涉及它的那部分优化就不触发。上面 `f(a,b,c)` 这种读 a、b 写 c 的例子，要让编译器复用 `a[0]+b[1]`、跨 `c` 的写入重排，就得先确认写 c 不会改到 a/b。最简单的做法：kernel 涉及的所有读写指针都加上，或用编译选项 `-restrict` 一次性声明全部指针参数为 restrict。

#### 只读数据与 `__ldg()`

对于 kernel 内只读的全局数据，可用 `__ldg()` 走只读缓存路径加载：

```cpp
float v = __ldg(&readonly_array[idx]);   // 只读 L2 缓存加载
```

`__ldg()` 支持所有 C++ 基本类型、HGGC 向量类型（3 元素除外）以及扩展浮点类型（如 `__half`、`__half2`、`__ppu_bfloat16`、`__ppu_bfloat162`）。将只读数据限定为 `const __restrict__` 也有助于编译器自动选用只读路径。

#### 精细 Cache policy：through

PPU 的读写指令支持多种缓存策略，各级缓存的绕过（through）行为可以独立控制：

当数据缺乏时间和空间局部性、预期不会命中缓存时，宜选择绕过（through）缓存，避免这些只用一次的数据占用缓存空间、把有用数据提前替换出去。

缓存策略既可在指令层通过 [TIX](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042) 指定，也可在编译期用透传选项设置默认策略：

```bash
# 为 load / store 设置默认缓存策略；-flcm/-fscm 优先级更高，强制覆盖
hgcc a.hg -Xllvm --ppu-dlcm=1,--ppu-dscm=1 -o a.out
```

| LLVM 后端选项 | 分类 | 说明 |
| :--- | :--- | :--- |
| `--ppu-dlcm=<int>` | 缓存策略 | 读取（load）指令的默认缓存策略 |
| `--ppu-dscm=<int>` | 缓存策略 | 存储（store）指令的默认缓存策略 |
| `--ppu-flcm=<int>` | 缓存策略 | 强制覆盖所有 load 的缓存策略（优先级更高）|
| `--ppu-fscm=<int>` | 缓存策略 | 强制覆盖所有 store 的缓存策略（优先级更高）|

#### Prefetch（预取）：实验性

可以实验性地使用 Prefetch 指令提高 cache 命中率，但不一定有效果，需要实测验证。

- ppu0015 的 `load` / `aiu` 指令支持预取提示，可指定预取粒度 `{128B, 256B}`：
  - 请求读 64 字节时，实际会把 128 字节预取到 LLC；
  - 请求读 128 字节时，实际会把 256 字节预取到 LLC。
- 预取只对有规律、可预测的后续访问有意义；对随机访问模式，预取反而浪费带宽。

### 2.5 异步拷贝与 AIU：让数据搬运不阻塞计算

同步的 `load→store` 搬运会占用寄存器，且线程要等数据到位。PPU 提供了两级异步搬运机制，把搬数据和做计算重叠起来。

#### `async_cp` / `memcpy_async`：元素级异步拷贝（global 到 shared）

`ppu.cp.async` 是一条非阻塞指令，直接把数据从全局内存异步拷贝到共享内存，不经过寄存器，发起线程可继续计算：

```asm
ppu.cp.async.ca.shared.global [shrd], [gbl + 4], 4;   // 拷贝 4 字节 global→shared
```

- 仅支持 global → shared 方向。
- 指针需按拷贝大小对齐到 1 / 2 / 4 / 8 / 16 字节；shared 和 global 的对齐都是 128 字节时性能最佳。

在 C++ 层，推荐用 `hggc::memcpy_async` 或 `cooperative_groups::memcpy_async` 协作发起拷贝，并配合 `hggc::barrier` / `hggc::pipeline` 等待完成：

```cpp
#include <cooperative_groups.h>
#include <cooperative_groups/memcpy_async.h>
namespace cg = cooperative_groups;

__global__ void stencil(const float* left, const float* center, const float* right) {
    cg::thread_block block = cg::this_thread_block();
    __shared__ float buffer[8 + 32 + 8];   // 左halo - 中心 - 右halo

    // 跨所有线程协作发起异步拷贝（不占用寄存器、可与计算重叠）
    cg::memcpy_async(block, buffer,      left,   8  * sizeof(float));
    cg::memcpy_async(block, buffer + 8,  center, 32 * sizeof(float));
    cg::memcpy_async(block, buffer + 40, right,  8  * sizeof(float));

    cg::wait(block);        // 等待全部拷贝完成
    __syncthreads();
    // ... 在 shared 上计算 stencil ...
}
```

#### 多级流水预取

在拷贝加计算的迭代模式下，可用 `hggc::pipeline` 搭建多级流水线，用当前迭代的计算隐藏下一批数据的传输延迟：

```cpp
#include <hggc/pipeline>

template <size_t num_stages = 2>
__global__ void prefetch_kernel(int* out, const int* in, size_t batch_size) {
    extern __shared__ int shared[];        // num_stages × block.size() × sizeof(int)
    hggc::pipeline<hggc::thread_scope_thread> pipe = hggc::make_pipeline();

    // 用前 num_stages 批数据填满流水线，随后计算一批、预取一批交替推进
    // ... producer_commit() / producer_acquire() ...
}
```

#### AIU：批量多维张量异步拷贝

AIU（AI accelerator in Compute Unit，计算单元内 AI 加速器）是 PPU 专门负责数据搬运的硬件机制，用于在 global 与 shared 之间高效传输多维张量数据。

AIU 的特点：

- 编程模型是异步、批量（bulk）风格的 warp 级指令，支持 1D / 2D / 3D 多维数据的异步搬移；
- AIU 写共享内存时采用 swizzle（交错）内存布局自动规避 bank conflict，开发者无需感知具体布局；
- 从共享内存往寄存器读取 AIU 写入的数据时，要用 PPU 带 Swizzle 功能的读指令按 Tile 为单位解析加载。

AIU 指令能直接把数据从设备内存搬到共享内存，减少了指令数量，地址计算也由专用硬件完成，省下本用于地址生成的通用算术指令，提高整体执行效率。做张量或 GEMM 类 kernel 时优先考虑 AIU 路径。指令细节参见 [TIX 编程指南的 aiu-copy 章节](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042)。

---


## 第三部分：最大化指令吞吐量

当 kernel 已经做到访存最优、占用率合理，进一步的收益来自指令吞吐量：用更少、更高吞吐的指令完成同样的计算。优先使用吞吐更高的硬件单元和运算，避免低效的替代方案。

### 3.1 使用更高效的计算单元

具体原则：

- 能用 tensor cell 尽量用 tensor cell，少用 fma 拼 mma；
- 能用 float / int，尽量不用 double / i64；
- 能用 fast math，尽量不用 full-prec math；
- ppu001 上尽量不用 `tsm.atomic.cas`；
- 尽量不用非 native 支持的 `tsm.atomic` 操作；
- 选择满足精度要求、且能被 native SFU 支持的计算指令；
- 编译时选择适合的精度选项：`-ftz`、`-prec-div`、`-prec-sqrt`。

#### 1）张量核（Tensor Cell）优先

PPU 的张量计算核（tensor cell）为矩阵乘加（MMA）提供了比普通 FMA 单元高得多的吞吐。GEMM、卷积、attention 等矩阵类计算应优先走 tensor cell 路径：用 MMA 指令单指令完成一个小 tile 的乘加，而不是用大量标量 fma 手工累加实现矩阵乘、占用大量通用计算单元与寄存器。

低精度格式能进一步放大 tensor cell 的吞吐优势：

| 架构 | 张量核支持的低精度格式 | 相对算力 |
| :--- | :--- | :--- |
| ppu001 | FP16、BF16 | FP16 = 基准 |
| ppu0015 | 新增 FP8（E4M3 / E5M2）、MXFP4（E2M1，缩放因子 E8M0）| FP8 ≈ 2× FP16；FP4 ≈ 4× FP16 |

- FP8 适用于训练和推理；FP4 / MXFP4 主要面向推理。

低精度可以把算力提高数倍，但会改变数值范围和精度。切换前应评估对结果精度的影响，必要时用混合精度（关键累加用高精度、输入用低精度）。

#### 2）优先 float / int，避免 double / i64

双精度（`double`）和 64 位整数（`i64`）在 PPU 上的吞吐通常远低于单精度和 32 位整数。除非算法确实需要 double 的动态范围或精度，多数情况选 `float` / `int32`。

```bash
# 编译期找出设备代码里的 double 运算
hgcc a.hg -Xllvm --ppu-warn-double-usage -o a.out
```

!!! note

    C/C++ 中 `double` 常量（如 `1.0` 而非 `1.0f`）、某些数学库重载会把计算意外提升到双精度。写浮点字面量时应带 `f` 后缀（`2.0f`、`0.5f`），并优先使用单精度函数（`sinf` 而非 `sin`）。

#### 3）整数除法 / 取模改用移位

整数除法和取模是低吞吐操作。当除数是 2 的幂时，用移位或位与替代：

```cpp
int q = value / 8;        // 编译器可自动优化为 value >> 3 (非const情况下编译器无法自动优化)
int q2 = value >> 3;
int r  = value % 8;       // 编译器可自动优化为value & 7 (非const情况下编译器无法自动优化)
int r2 = value & 7;
```

#### 4）半精度向量化计算

对精度不敏感的逐元素运算，可用 `__half2` 一次算两个：

```cpp
__half2 a = __halves2half2(a0, a1);
__half2 b = __halves2half2(b0, b1);
__half2 c = __hmul2(a, b);   // 单指令完成两次乘法
```

### 3.2 数学函数选择：精度与速度

HGGC 为标准数学函数提供了以 `__` 为前缀的内建（intrinsic）快速版本，通过减少原生指令数量换取更高速度，代价是精度下降。

| 标准函数 | 内建（快速）版本 | 说明 |
| :--- | :--- | :--- |
| `sinf(x)` / `cosf(x)` / `tanf(x)` | `__sinf` / `__cosf` / `__tanf` | 快速三角函数 |
| `sincosf(x,&s,&c)` | `__sincosf` | 同时算 sin/cos |
| `expf(x)` / `exp10f(x)` | `__expf` / `__exp10f` | 快速指数 |
| `logf` / `log2f` / `log10f` | `__logf` / `__log2f` / `__log10f` | 快速对数 |
| `powf(x,y)` | `__powf` | 实现为 `exp2f(y*__log2f(x))` |
| `x / y` | `__fdividef(x,y)` | 快速除法 |
| `tanhf(x)` | `__tanhf(x)` | 快速双曲正切 |

```cpp
// 精度要求不高时，用内建函数
float y = __sinf(x);        // 比 sinf 快，精度降低
float e = __expf(x);

// 用专用函数替代通用组合
float r = rsqrtf(x);        // 1/sqrt(x) 的专用指令，比 1.0f/sqrtf(x) 快
float c = cbrtf(x);         // 立方根，比 powf(x, 1.0f/3.0f) 快
```

#### 小分数指数：用专用函数

| 目标计算 | 推荐写法 | 备注 |
| :--- | :--- | :--- |
| x^(1/2) | `sqrtf(x)` | |
| x^(-1/2) | `rsqrtf(x)` | 倒数平方根 |
| x^(1/3) | `cbrtf(x)` | 立方根 |
| x^(-1/3) | `rcbrtf(x)` | |
| x^(2/3) | `float r = cbrtf(x); r*r;` | |
| x^(3/2) | `x * sqrtf(x)` | |

#### 取整：`rintf` 而非 `roundf`

将单精度浮点舍入为整数时，推荐 `rintf()` 而非 `roundf()`：`rintf()` 采用舍入到最近偶数（IEEE-754 默认舍入模式），语义更通用，通常也比 `roundf()`（半数远离零）开销更低。双精度同理，优先 `rint()` 而非 `round()`。

#### 利用 native SFU

SFU（Special Function Unit，特殊函数单元）原生支持一批超越函数。应选择满足精度要求、且能被 native SFU 直接支持的计算指令，避免用一长串通用指令逼近一个 SFU 本可直接完成的操作。

### 3.3 原子操作优化

原子操作（read-modify-write）会强制线程同步，使用不当会明显降低性能。

#### 优先使用 native 支持的原子类型

PPU 对原子操作提供 native 硬件支持，非native支持的原子操作性能会显著下降。global memory 与 shared memory、ppu001 与 ppu0015 各自 native 支持的类型并不相同，因此按内存空间和架构分开列出：

**Global memory 原子（ppu001）**

| 原子操作 | Native 支持类型 |
| :--- | :--- |
| `atomicAdd` | U32 / I32 / U64 / I64 / F32 / F64 |
| `atomicMin` / `atomicMax` | U32 / I32 / U64 / I64 / F32 / F64 |
| `atomicAnd` / `atomicOr` / `atomicXor` | B32 / B64 |
| `atomicInc` / `atomicDec` | U32 / U64 |
| `atomicSwap`（`atomicExch`）| B32 / B64 |
| `atomicCAS`| B32 / B64 |

**Global memory 原子（ppu0015）**

| 原子操作 | Native 支持类型 |
| :--- | :--- |
| `atomicAdd` | U32 / I32 / U64 / I64 / F32 / F64 / **F16x2** / **BF16x2** |
| `atomicMin` / `atomicMax` | U32 / I32 / U64 / I64 / F32 / F64 / **F16x2** / **BF16x2** |
| `atomicAnd` / `atomicOr` / `atomicXor` | B32 / B64 |
| `atomicInc` / `atomicDec` | U32 / U64 |
| `atomicSwap`（`atomicExch`）| B32 / B64 |
| `atomicCAS`| B32 / B64 |

**Shared memory 原子（ppu001）**

| 原子操作 | Native 支持类型 |
| :--- | :--- |
| `atomicAdd` | U32 / I32 |
| `atomicMin` / `atomicMax` | U32 / I32 |
| `atomicAnd` / `atomicOr` / `atomicXor` | B32 |
| `atomicInc` / `atomicDec` | U32 |
| `atomicSwap`（`atomicExch`）| B32 / B64 |
| `atomicCAS`| B32 / B64 （注意：ppu001 上性能很差） |

**Shared memory 原子（ppu0015）**

| 原子操作 | Native 支持类型 |
| :--- | :--- |
| `atomicAdd` | U32 / I32 |
| `atomicMin` / `atomicMax` | U32 / I32 |
| `atomicAnd` / `atomicOr` / `atomicXor` | B32 |
| `atomicInc` / `atomicDec` | U32 |
| `atomicSwap`（`atomicExch`）| B32 / B64 |
| `atomicCAS`| B32 / B64 |

#### 硬性经验

以下要点均针对 ppu001（ppu0015 上大多已优化或行为不同），除非另有标注：

```
避免：尽量不用 tsm.atomic.cas
避免：尽量不用非 native 支持的 tsm.atomic 操作（ppu001 / ppu0015 均适用）
限制：不支持用 generic atomic cas 访问 shared memory
性能不好：少用 same address（同地址）的 global.atomic 操作，会竞争串行化（ppu001 / ppu0015 均适用）
推荐：全局原子同 warp 的访问地址尽量连续且按 128 字节对齐
```

!!! note "ppu001的原子性能问题"

    在 ppu001 上，同 warp 的 32 个线程做全局内存原子操作时，若访问地址不连续或未按 128 字节对齐，64 字节访问模式的性能只有 128 字节模式的一半，32 字节模式降至四分之一。ppu0015 已优化，使 64B/32B 模式与 128B 模式相当。

#### 用作用域明确的原子接口

HGGC 提供了 `hggc::atomic` / `hggc::atomic_ref`，可显式指定原子的线程作用域，比传统 `atomicXxx()` 更清晰地表达意图：

```cpp
__global__ void sumReduce(int n, float* array, float* result) {
    int tid = threadIdx.x + blockIdx.x * blockDim.x;
    hggc::atomic_ref<float, hggc::thread_scope_device> ref(result);
    ref.fetch_add(array[tid]);   // 设备范围原子加
}
```

减少原子操作总量往往比优化单次原子操作更有效。归约类需求优先考虑：warp 内用 `__shfl` 系列做 warp 级归约，block 内用共享内存归约，最后每个 block 只做一次全局原子写，把同地址竞争降到最低。

### 3.4 控制流优化

硬件速览已经介绍过 warp 分歧：分支的两条路径被串行执行，硬件利用率下降。控制流优化要尽量减少以分歧方式执行的 warp 指令。

#### 1）算法层面降低控制流复杂度

- 减少以分歧方式执行的 warp 指令；
- 尽量避免细碎的小分支代码块（大量小分支会打断流水、增加调度开销）。

#### 2）条件与 warp 对齐

分支条件如果逐线程判断（如按 `threadIdx.x` 的奇偶分叉），warp 内会一半走 if、一半走 else，两条路径串行执行；把条件改成按 warp 判断，整个 warp 走同一路径，就不会分歧：

```cpp
// 按 warp 分叉，整个 warp 走同一路径，无分歧
//（warp_idx 需 warp-uniform；threadIdx.x/32 不天然 uniform，用广播保证，见 5））
int warp_idx = __shfl_sync(0xffffffff, threadIdx.x / 32, 0);
if (warp_idx == 0) { /* 第一个 warp 走 A */ } else { /* 其余走 B */ }
```

#### 3）用 template condition 消除死代码

如果热点代码（hot code）中有一个很大的代码段根本不会执行，最好用 template condition（模板条件），让编译器在编译期就把它优化掉。

```cpp
template <bool ENABLE_EXTRA>
__global__ void kernel(float* p) {
    // 常规计算 ...
    if constexpr (ENABLE_EXTRA) {   // 编译期常量条件
        // 一大段可选逻辑：ENABLE_EXTRA=false 时整段被编译器消除，
        // 不占用寄存器、不产生指令
    }
}
```

相比运行时的 `if (flag)`（flag 是运行期变量，代码必须保留、还可能引发分歧），`if constexpr` 配合模板参数能让无用分支完全从二进制中消除。

#### 4）warp 重聚：多数自动，必要时才用 `__syncwarp()`

分歧结束后，warp 会在汇合点重新汇聚。这通常由编译器自动完成：只要分歧区域内没有 yield 相关行为（如带 `volatile` load 或 atomic 的等待循环），编译器便能在汇合点让整 warp 自动重聚，不必手写同步。

需要显式 `__syncwarp()` 的，主要是无法依赖自动重聚的两种情况。一是分歧区域内含 yield 相关行为，例如自旋等待 `volatile` 值、循环内做 atomic 等，线程可能各自独立推进，无法保证在汇合点对齐；二是在调用 warp 级协作原语（`__shfl` 系列、异步屏障的 arrive / commit）之前，需要显式确保整 warp 已汇聚。

```cpp
// 分歧内只是普通计算，汇合点自动重聚，通常无需 __syncwarp()
if (threadIdx.x % 2 == 0) { /* A：纯计算 */ } else { /* B：纯计算 */ }

// 分歧内有自旋等待（volatile / atomic 循环），自动重聚不可依赖，需显式兜底
if (threadIdx.x % 2 == 0) {
    while (*(volatile int*)&flag == 0) { }   // yield 行为
}
__syncwarp();   // 确保整 warp 汇聚后，再安全使用 __shfl 等协作原语
```

在使用异步屏障等高级同步原语时同理：当分歧可能无法自动重聚时，先 `__syncwarp` 汇聚 warp，再调用 arrive / commit，避免过度等待或线程未对齐。

#### 5）sync 逻辑尽量用 uniform condition

在需要使用 sync 逻辑的 kernel 中，宜使用 uniform（warp 内一致）的 condition，让编译器能判断是否会产生分叉，从而优化 warp sync。

uniform condition 指整个 warp 求值结果相同的条件，例如基于 `blockIdx` / `blockDim` / `gridDim`，或 warp 内共享的 SREG 值。条件 uniform 时，编译器可以确定这里不会分歧，省掉不必要的同步和重聚开销，也能更好地利用 SREG。

!!! note "`threadIdx.x / 32` 并不天然 uniform"

    warp 的归属由线性线程 id（`threadIdx.x + threadIdx.y * blockDim.x + …`）决定，而不是只看 `threadIdx.x`。在 2D/3D block，或 `blockDim.x` 不是 32 整数倍时，同一 warp 内 `threadIdx.x / 32` 可能取到不同值，不能直接当作 uniform condition。要得到 warp 一致的 warp id，需要显式建立 uniform 性：

    ```cpp
    // 方式一：把 lane 0 算出的值广播给整 warp，确保依赖代码按 warp-uniform 编译
    int warp_idx = __shfl_sync(0xffffffff, threadIdx.x / 32, 0);
    // 方式二：用 PPU 内建向编译器声明该值 warp 内一致（若实际不一致则行为未定义）
    int warp_idx2 = __builtin_ppu_to_uniform_b32(threadIdx.x / 32);
    ```

### 3.5 精度与浮点行为的编译选项

编译期即可在精度与速度之间做全局或细粒度的取舍：

```bash
# 全局启用快速数学：等价于 --ftz=true --prec-div=false --prec-sqrt=false --fmad=true
hgcc -use_fast_math kernel.hg -o kernel.out

# 细粒度控制
hgcc --ftz=true \        # 非正规数（denormals）刷新为零
     --prec-div=false \  # 单精度除法不严格遵循 IEEE 舍入（更快）
     --prec-sqrt=false \ # 单精度平方根不严格遵循 IEEE 舍入（更快）
     --fmad=true \       # 将浮点乘加合并为融合乘加 FMA（更快、精度更高）
     kernel.hg -o kernel.out
```

| 选项 | 作用 | 权衡 |
| :--- | :--- | :--- |
| `-use_fast_math` | 全局启用快速数学，并把标准函数替换为 `__` 内建版本 | 精度下降，且可能改变 NaN/无穷大等边界行为 |
| `--ftz=true` | 非正规数刷零 | 极小值当作 0，多数场景可接受 |
| `--prec-div=false` | 快速除法 | `x/y` 误差从 0 ulp 变为最多 4 ulp |
| `--prec-sqrt=false` | 快速平方根 | `sqrtf` 误差从 0 变为 1 ulp |
| `--fmad=true` | 融合乘加（默认开启）| FMA 中间不舍入，通常更快也更精确 |

---


## 第四部分：消除 Stack 与寄存器溢出

Stack 和寄存器溢出（reg spill）对 PPU 性能影响很大。

当 kernel 需要的寄存器超过可用量时，多出来的变量会溢出（spill）到 local memory（物理位置见[硬件模型速览](#ppu-硬件模型速览)）。单次溢出就可能在热点循环里引入大量片外访存，使性能明显下降。

### 4.1 溢出的常见原因与对策

#### 1）寄存器预算不足，用 launch_bounds / maxrregcount 调控

选取合适的 `__launch_bounds__` 或 `--maxrregcount` / `__maxnreg__` 来控制每线程寄存器预算，避免溢出（详见 [1.4](#14-用-launch_bounds-与寄存器上限引导编译器)）：

```cpp
__global__ void __launch_bounds__(256, 4) kernel(float* p) { /* ... */ }
```

收紧寄存器上限可能减少溢出（如果原本就超了），也可能诱发溢出（如果上限压得过低）。寄存器不是用得越少越好，要找到占用率提升的收益大于溢出损失的平衡点，建议用 `-res-usage` 和 spill 警告option实测。

#### 2）动态 index 访问 temp 数组导致溢出

编译器无法确定索引是否为常量的数组，很可能被分配到 local memory。下例中 `sel` 是运行期变量，`arr` 大概率进 local；如果索引是编译期可确定的常量，数组可以留在寄存器：

```cpp
__global__ void bad(int sel) {
    float arr[8];
    // ... 填充 arr ...
    float v = arr[sel];      // 动态索引，arr 大概率进 local memory
}

__global__ void good() {
    float arr[8];
    // ... 填充 arr ...
    float v = arr[3];        // 常量索引，寄存器
}
```

#### 3）循环里的动态 index，用 `#pragma unroll` 化解

循环里的动态 index 可以通过 `#pragma unroll` 解决。展开后，原本运行期的循环变量索引变成了编译期可确定的常量索引，数组得以留在寄存器：

```cpp
// 两个循环都加 #pragma unroll，i 在编译期展开为 0,1,2,...，tmp 留在寄存器
__global__ void goodLoop(float* out) {
    float tmp[8];
    #pragma unroll
    for (int i = 0; i < 8; i++) {
        tmp[i] = compute(i);
    }
    #pragma unroll
    for (int i = 0; i < 8; i++) {
        out[i] = tmp[i] * 2.0f;
    }
}
```

如果不展开，第二个循环里的 `i` 是运行期变量，`tmp` 就可能被放进 local memory，每次读写都变成片外访存。

!!! note "unroll 的收益"

    循环展开不仅消除 spill，还能暴露 ILP（多个迭代互相独立、可流水）、减少循环控制指令。对小的、迭代次数固定的循环，`#pragma unroll` 几乎总是有益的。对大循环可用 `#pragma unroll N` 部分展开，在代码膨胀与收益之间折中。

### 4.2 检测溢出与 local memory

编译期就能发现溢出问题：

```bash
# 打印每个 kernel 的资源使用报告（寄存器 / 共享内存 / 局部内存）
hgcc -res-usage kernel.hg -o kernel.out

# 寄存器溢出到 local memory 时告警
hgcc a.hg -Xllvm --ppu-warn-spills -o a.out

# 设备代码使用 local memory（stack）时告警
hgcc a.hg -Xllvm --ppu-warn-lmem-usage -o a.out
```

一般 kernel 程序中少出现或不出现 local memory，是程序优化的一个重要目标。但要注意：局部变量的多少本身并不直接决定程序效率，关键是不要在热点路径上频繁访问 local memory。

---

## 第五部分：用 Asight 定位优化空间

前几部分介绍的是优化手段，本部分介绍如何用 profiler 找到 kernel 的瓶颈，确定该用哪种手段。

### 5.1 分析流程

1. 采集：用 Asight 运行 kernel，拿到占用率 / 访存 / 指令指标。
2. 分析瓶颈：分析 kernel 是访存受限、计算受限，还是占用率受限：
    - 访存受限 → [第二部分](#第二部分最大化内存吞吐量)（合并、共享内存、缓存、AIU）
    - 计算受限 → [第三部分](#第三部分最大化指令吞吐量)（计算单元、精度、控制流、原子）
    - 占用率受限 → [第一部分](#第一部分最大化资源利用率)（block/grid、寄存器、`launch_bounds`）
3. 查 spill：`-res-usage` / `--ppu-warn-spills`，有问题转[第四部分](#第四部分消除-stack-与寄存器溢出)。
4. 修改，重新采集对比，并验证数值正确性。

### 5.2 关键观测点

| 维度 | 关注点 | 对应章节 |
| :--- | :--- | :--- |
| Occupancy | 理论与实际占用率的差距；哪项资源是 Block Limit 瓶颈 | [第一部分](#第一部分最大化资源利用率) |
| 访存效率 | 每笔 warp 访存触及多少 cache line；是否合并 | [2.1](#21-全局内存合并访问coalesced-access) |
| 共享内存 | 是否有 bank conflict / group conflict | [2.3](#23-共享内存bank-conflict-与-group-conflict) |
| 指令构成 | 是否有 double、低吞吐指令、大量分歧分支 | [第三部分](#第三部分最大化指令吞吐量) |
| 寄存器 / spill | 每线程寄存器数、是否溢出到 local | [第四部分](#第四部分消除-stack-与寄存器溢出) |

### 5.3 让 profile 结果可追溯到源码

```bash
# 为设备代码生成行号信息（不影响执行性能），
# 使 Asight 能把热点关联回具体源码行
hgcc -lineinfo kernel.hg -o kernel.out
```

- 无需为使用 Asight 添加额外编译标志，但 `-lineinfo` 能让分析工具查看到对应的源代码（要求原始源码在编译位置可用）。
- 注意：`-G`（设备调试信息）会阻止某些编译器优化，用 `-G` 编出来的性能不代表发布版本；性能分析请用 `-lineinfo` 而非 `-G`。

---

## Kernel 优化检查清单

清单按优化的实际顺序排列：先用 profiler 找到瓶颈，再按瓶颈类型改写，改完复测。

### 定位瓶颈

流程和工具见[第五部分](#第五部分用-asight-定位优化空间)，寄存器与溢出问题见[第四部分](#第四部分消除-stack-与寄存器溢出)。

- 用 Asight 采集占用率、访存效率和指令构成，判断 kernel 属于访存受限、计算受限还是占用率受限
- 用 `-res-usage` 或 `--ppu-warn-spills` 查每线程寄存器用量，确认有无溢出

### 访存

这一组对应[第二部分](#第二部分最大化内存吞吐量)和[第四部分](#第四部分消除-stack-与寄存器溢出)。收益表里内存访问优化仅次于算法层，多数 kernel 的瓶颈也在这里。

- warp 内访问连续地址，让单次访存尽量落在一条 128B cache line 内
- 中间结果用 shared memory 中转，不要写回 global 再读回
- 用 int4 / float4 宽读写指令。ppu001 上还要防止 generic address 把宽指令拆散
- 二维共享数组多加一列，`[N][N]` 写成 `[N][N+1]`，用 padding 消除 bank conflict
- 张量和 GEMM 类 kernel 优先走 AIU，其余场景可用异步拷贝让搬运与计算重叠
- 迭代次数固定的小循环加 `#pragma unroll`，让多个迭代的 load 并行发出、隐藏访存延迟，同时化解动态索引导致的 spill
- 指针加 `__restrict__`，或整体用 `-restrict`。收益是渐进的，加得越全编译器能做的优化越多
- 只用一次、没有局部性的数据走 through，不占用缓存；prefetch 是实验性手段，要实测验证

### 执行配置与占用率

见[第一部分](#第一部分最大化资源利用率)。

- block 大小是 32 的倍数，最好取 2 的整数次幂
- grid 大小接近 `CU × 2 的整数倍`，减少拖尾
- 占用率提到收益拐点即可，不必追求更高；拐点位置因 kernel 和架构而异，靠实测确定
- 同时让每个 warp 有足够的独立指令，TLP 与 ILP 都要兼顾
- 加 `__launch_bounds__` 引导寄存器分配，只用单参数版本也有帮助

### 指令与精度

见[第三部分](#第三部分最大化指令吞吐量)。

- 矩阵乘加走 tensor cell，数值允许时配合低精度格式
- 优先 float / int32，避免 double 和 i64；浮点字面量带 `f` 后缀
- 数学函数按精度需求选内建版本，取整用 `rintf` 而不是 `roundf`
- 原子操作用 native 支持的类型。ppu001 上全局原子的地址尽量连续且 128B 对齐
- 分支条件与 warp 对齐，或改用 uniform condition，减少以分歧方式执行的指令
- 除数是 2 的幂时，整数除法和取模改用移位和位与
- `-use_fast_math`、`-ftz`、`-prec-div`、`-prec-sqrt` 会改变精度和边界行为，评估后开启

### 改完复测

- 数值结果与参考实现一致
- 占用率、访存效率、指令构成有改善
- 性能提升可复现，多跑几次取稳定值
- 没有引入新的 spill、bank conflict 或分歧

---


## 附录

### 附录 A：PPU 关键硬件参数速查

代表性配置（真武 810E），具体数值请以 `hggcGetDeviceProperties` / `hggcDeviceGetAttribute` 的运行时查询为准：

| 资源 | 数值 |
| :--- | :--- |
| CU 数量 | 64 |
| 每 CU 最大 Warp 数 | 64（= 2048 线程/CU）|
| 每 CU VREG 总量 | 128K |
| 单 Warp 最大 VREG 使用量 | 255 |
| 每 CU 共享内存 | 256 KB |
| Warp Size | 32 |
| 共享内存 bank | 4 B / bank，共 32 bank |
| 全局内存合并单位 | 128 B |
| 共享内存分组 | 2 组，每组 128 B/拍 |
| cache line | 128 B |

内存层次（由近及远）：`Register(VREG/SREG) → Shared Memory → L1(每 CU) → L2(CE 内共享) → LLC(所有 CE 共享) → Global Memory(HBM)`

架构差异备忘（ppu001 → ppu0015）：

| 维度 | ppu001 | ppu0015 |
| :--- | :--- | :--- |
| 低精度格式 | FP16 / BF16 | 增加 FP8（E4M3/E5M2）、MXFP4（E2M1 + E8M0 缩放）|
| 共享内存分组 | 2 组 × 128B | 4 组 × 1024B 跨度（group conflict 更少）|
| 全局原子对齐敏感度 | 64B/32B 模式性能减半/减至 1/4 | 已优化，64B/32B ≈ 128B |
| global native原子type | 见[3.3](#33-原子操作优化) | native支持 F16x2 / BF16x2|
| 预取提示 | — | load/aiu 支持 {128B, 256B} 预取到 LLC |
| 小粒度访问目标 | 减少 128B cache line 数 | 减少到 64B 数 |

### 附录 B：Kernel 优化常用编译选项汇总

```bash
# --- 性能/精度 ---
hgcc -use_fast_math kernel.hg -o out          # 快速数学（= --ftz=true --prec-div=false --prec-sqrt=false --fmad=true）
hgcc --fmad=true kernel.hg -o out             # 融合乘加（默认开启）
hgcc -restrict kernel.hg -o out               # 所有 kernel 指针参数视为 restrict

# --- 寄存器 / 占用率 ---
hgcc --maxrregcount=32 kernel.hg -o out       # 限制每线程寄存器数（文件级）

# --- 诊断 / 报告 ---
hgcc -res-usage kernel.hg -o out              # 打印寄存器/共享内存/局部内存用量
hgcc -lineinfo kernel.hg -o out               # 生成行号信息，供 Asight 关联源码
hgcc a.hg -Xllvm --ppu-warn-spills            # 寄存器溢出告警
hgcc a.hg -Xllvm --ppu-warn-lmem-usage        # 使用 local memory 告警
hgcc a.hg -Xllvm --ppu-warn-double-usage      # 使用 double 运算告警

# --- 缓存策略 ---
hgcc a.hg -Xllvm --ppu-dlcm=1,--ppu-dscm=1    # 设置 load/store 默认缓存策略

# --- 架构 ---
hgcc -arch=ppu_10 kernel.hg -o out            # 真实架构（直接可执行 hgbin）
hgcc -arch=vm_10  kernel.hg -o out            # 虚拟架构（LLVM bitcode，运行时 JIT）
```

!!! note

    `-G`（`--device-debug`）为设备代码插入调试信息，会阻止某些编译器优化，其性能不代表发布版本。性能分析用 `-lineinfo`，调试才用 `-G`。

### 附录 C：设备端关键字与内建函数一览

| 类别 | 名称 | 用途 |
| :--- | :--- | :--- |
| 执行空间 | `__global__` / `__device__` / `__host__` | 标注函数执行位置 |
| 内存 | `__shared__` / `extern __shared__` | 静态 / 动态共享内存 |
| 同步 | `__syncthreads()` / `__syncwarp()` | 块内 / warp 内同步（重聚）|
| 启动界限 | `__launch_bounds__(maxTpb[, minBlocks])` | 引导寄存器分配 |
| 寄存器 | `__maxnreg__(N)` | 逐 kernel 指定最大寄存器数 |
| 指针 | `__restrict__` | 声明无别名，便于编译器优化 |
| 只读加载 | `__ldg(ptr)` | 走只读缓存路径 |
| 快速数学 | `__sinf` / `__cosf` / `__expf` / `__logf` / `__powf` / `__fdividef` / `__tanhf` | 精度换速度 |
| 专用数学 | `rsqrtf` / `cbrtf` / `rcbrtf` / `rintf` / `sincospif` | 单指令专用函数 |
| 半精度 | `__half2` / `__hmul2` / `__hadd2` | 2 路 SIMD 半精度 |
| 原子（传统）| `atomicAdd` / `atomicMin` / `atomicMax` / `atomicCAS` / `atomicExch` … | 读-改-写 |
| 原子（作用域）| `hggc::atomic_ref<T, Scope>` / `hggc::atomic` | 显式线程作用域原子 |
| 原子（内建）| `__hg_atomic_add` / `__hg_atomic_fetch_add` / `__hg_atomic_exchange` … | 底层内建原子 |
| 异步拷贝 | `hggc::memcpy_async` / `cg::memcpy_async` / `ppu.cp.async` | global→shared 异步搬运 |
| 流水线 | `hggc::pipeline` / `hggc::barrier` | 多级流水、异步屏障 |

### 附录 D：参考资料

- [T-Head SAIL HGGC 编程指南](/docs_center/doc_detail/index.html?projectId=39&chapterId=1041)：完整编程模型、内存空间、异步拷贝、原子、数学函数精度表
- [T-Head SAIL HGCC](/docs_center/doc_detail/index.html?projectId=39&chapterId=1056)：编译器全部选项与透传参数
- [T-Head SAIL TIX 编程指南](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042)：虚拟 ISA、缓存策略、`aiu-copy` / `ppu.cp.async` 指令细节
- [T-Head SAIL HGGC Math API](/docs_center/doc_detail/index.html?projectId=39&chapterId=1048)：设备端数学函数与精度界限
- [T-Head SAIL HGGC Runtime API](/docs_center/doc_detail/index.html?projectId=39&chapterId=1046&chapterType=API) / [Driver API](/docs_center/doc_detail/index.html?projectId=39&chapterId=1047&chapterType=API)：运行时/驱动接口
- [T-Head SAIL Binary Utilities](/docs_center/doc_detail/index.html?projectId=39&chapterId=1061)：反汇编、查看生成指令
- [T-Head SAIL HGGC 示例程序](/docs_center/doc_detail/index.html?projectId=39&chapterId=1043)：含性能优化分类的可运行示例

---
