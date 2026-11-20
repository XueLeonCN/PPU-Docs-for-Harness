---
source_url: "https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=39&chapterId=1041"
source_api_url: "https://developer.t-head.cn/api/developer/document/chapter/content?projectId=39&chapterId=1041"
source_title: "T-Head SAIL HGGC 编程指南"
source_id: "chapter-1041"
source_kind: "programming-guide"
source_date: null
source_version: "T-Head SAIL SDK v2.2.1"
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "8ad906da61210c6ae8948ac908367237fe2e889a0f6ae9c26b23d1060b519c44"
tags:
  - "HGGC"
  - "host-device"
  - "grid"
  - "thread-block"
  - "warp"
  - "SIMT"
  - "memory"
  - "stream"
  - "event"
  - "graph"
  - "multi-device"
  - "cluster"
  - "barrier"
  - "pipeline"
---
# T-Head SAIL HGGC 编程指南

## 1. PPU 与 HGGC 入门 {#1-ppu-与-hggc-入门}
本章分为五个部分，第一部分是真武 PPU（以下简称 PPU）并行计算与 HGGC（HeteroGeneous General-purpose Computing，以下简称 HGGC）的介绍，第二部分是并行编程模型介绍，第三部分是 PPU 硬件架构介绍，第四部分是整个 HGGC 软硬件生态介绍，最后是 HGGC 特性概览。

### 1.1. PPU 并行计算与 HGGC {#11-ppu-并行计算与-hggc}

在相近的成本和功耗范围内， PPU 能够提供远高于 CPU 的指令吞吐量和内存带宽。许多应用程序正是利用了这些特性，从而在 PPU 上的运行速度显著快于在 CPU 上。其他计算设备（例如 FPGA）虽然同样具备极高的能效，但在编程灵活性方面却远不及 PPU。

PPU 与 CPU 在设计之初便有着截然不同的目标。CPU 的设计旨在尽可能快速地执行一系列串行操作（即所谓的"线程"），仅能够并行执行数十个此类线程；而 PPU 的设计则旨在并行执行数千个线程，通过牺牲单线程性能来换取极高的整体吞吐量。

PPU 专为高度并行计算而优化，将更多的晶体管资源分配给了数据处理单元；相比之下，CPU 则将更多的晶体管资源用于数据缓存与流控制。

利用 PPU 提供的计算能力有多种途径。本指南主要涵盖如何使用 C++ 等高级语言在 PPU 平台上进行编程。不过，在许多应用程序中，人们也可以通过无需直接编写 PPU 代码的方式来利用 PPU。

目前，通过各类专用库，人们可以获取来自不同领域的、日益丰富的算法与例程集合。如果某个库（尤其是 T-Head 官方提供的库）已实现了所需功能，那么直接调用该库往往比从零开始重新实现算法更具开发效率，且能获得更优的性能。[acBLAS](/docs_center/doc_detail/index.html?projectId=39&chapterId=1050)、[acFFT](/docs_center/doc_detail/index.html?projectId=39&chapterId=1051)、[acDNN](/docs_center/doc_detail/index.html?projectId=39&chapterId=1049) 和 CUTLASS 等库便是其中的典型代表，它们能帮助开发者避免重复实现那些已成熟定型的算法。此外，这些库还具备一项额外优势：它们针对不同的 PPU 架构进行了专门优化，从而在开发效率、运行性能与代码可移植性之间实现了理想的平衡。

此外，还有一些软件框架，尤其是那些应用于人工智能领域的框架，提供了基于 PPU 加速的功能构建模块。其中许多框架正是通过调用上文提及的那些 PPU 加速库，从而实现了其加速效果。

### 1.2. HGGC 并行编程模型 {#12-hggc-并行编程模型}


本章从高层次介绍 HGGC 编程模型，并独立于任何特定语言。这里引入的术语和概念适用于所有基于 HGGC 的编程语言。后面的章节将使用 C++ 来举例说明这些概念。

#### 1.2.1. 异构系统 {#121-异构系统}

HGGC 编程模型假定一个异构计算系统，一个包含 PPU 和 CPU 的系统。

CPU 和与 CPU 直接连接的内存被称为宿主机 (host) 和宿主内存 (host memory)； PPU 和与 PPU 直接连接的内存则被称为设备 (device) 和设备内存 (device memory)。

HGGC 应用程序执行其部分代码在 PPU 上，但应用程序始终在 CPU 上开始执行。运行在 CPU 上的代码称为宿主代码 (host code)，它可以使用 HGGC API 在宿主内存和设备内存之间复制数据，在 PPU 上启动代码执行，并等待数据复制或 PPU 代码完成。CPU 和 PPU 可以同时执行代码，通常通过最大化利用 CPU 和 PPU 来获得最佳性能。

应用程序在 PPU 上执行的代码被称为设备代码 (device code)，出于历史原因，在 PPU 上调用执行的函数被称为核函数 (kernel)。启动核函数运行的行为称为启动 (launching) 核函数。核函数启动可以看作是在 PPU 上并行执行许多线程运行核函数代码。 PPU 线程的操作类似于 CPU 上的线程，尽管存在一些对正确性和性能都很重要的差异，稍后的章节会对此进行讨论。

### 1.3. PPU 硬件模型 {#13-ppu硬件模型}

像任何编程模型一样，HGGC 依赖于底层硬件的概念模型。对于 HGGC 编程的目的而言， PPU 可以被认为是一系列计算单元（Compute Unit）的集合。每个 CU 包含本地寄存器文件、数据缓存（Cache）、共享内存（shared memory）以及若干用于执行计算的功能单元。

共享内存的分配可以在运行时配置。不同类型的内存大小和 CU 内的功能单元数量因 PPU 架构而异。

!!! note

    PPU 的实际硬件布局或其物理执行编程模型的方式可能有所不同。这些差异不会影响使用 HGGC 编程模型编写的软件的正确性。

#### 1.3.1. 线程块和网格 {#131-线程块和网格}

当应用程序启动核函数时，它会同时启动很多线程，通常是数百万个线程。这些线程被组织成线程块 (thread block)。线程块被组织成网格 (grid)。网格中的所有线程块具有相同的大小和维度。

线程块和网格可以是 1 维、2 维或 3 维的。这些维度可以简化将各个线程映射到工作单元或数据项的过程。

当启动核函数时，它是使用特定的执行配置启动的，该配置指定网格和线程块的维度。执行配置还可以包括可选参数，如集群大小、流 (stream) 和 CU 配置设置，这些将在后续章节中介绍。

使用内置变量，执行核函数的每个线程都可以确定自己在其所属块中的位置以及其块在父级网格中的位置。线程也可以使用这些内置变量来确定启动核函数的线程块和网格的维度。这使得每个线程在运行核函数的所有线程中都有唯一的标识。这种标识经常用来确定线程负责的数据或操作。

所有线程块的线程都在单个 CU 中执行。这允许线程块内的线程高效地相互通信和同步。线程块内的线程都访问片上共享内存，可用于在线程块的线程之间交换信息。

网格可能由数百万个线程块组成，而执行网格的 PPU 可能只有几十个或几百个 CU。线程块的所有线程都由单个 CU 执行，并且在大多数情况下都会在此 CU 上运行至完成。线程块之间的调度没有保证，因此线程块不能依赖其他线程块的结果，因为直到当前线程块完成后才有可能调度其他线程块。
HGGC 编程模型使任意大的网格能够在任何尺寸的 PPU 上运行，无论是只有一个 CU 还是数十个 CU。为了实现这一点，HGGC 编程模型（有一些例外）要求不同线程块中的线程之间不存在数据依赖关系。也就是说，一个线程不应依赖同一网格中另一个线程块中的线程的结果或与其同步。同一个线程块中的所有线程在同一时间在相同 CU 上运行。网格中的不同线程块在可用的 CU 之间进行调度，可以按任何顺序执行。简而言之，HGGC 编程模型要求能够以任何顺序并行或串行执行线程块。

#### 1.3.2. 线程束 (Warps) 和 SIMT {#132-线程束warps和simt}

在一个线程块内，线程被组织成 32 个线程的组，称为 warps。warp 以单指令多线程（SIMT，Single-Instruction Multiple-Threads）范式执行核函数代码。在 SIMT 中，warp 中的所有线程正在执行相同的核函数代码，但是每个线程可能会沿着不同的分支路径执行代码。也就是说，虽然程序的所有线程执行相同的代码，但线程不需要遵循相同的执行路径。

这意味着 CU 的状态需要被存储到系统管理的 PPU 内存区域中，然后被释放去执行其他线程块。这类似于 CPU 上的上下文切换。

当 warp 执行线程时，它们会被分配到 warp 通道 (warp lane)。warp 通道编号从 0 到 31，线程块中的线程按照可预测的方式分配到 warps 中。

warp 中的所有线程同时执行同一条指令。如果 warp 中的某些线程跟随控制流程分支执行，而其他线程不跟随，则不跟随分支的线程将被屏蔽掉，而跟随分支的线程将继续执行。例如，如果条件判断只对 warp 中一半的线程成立，则另一半的 warp 将被屏蔽，而活动线程执行那些指令。当 warp 中的不同线程遵循不同的代码路径时，有时称为 warp 分歧 (warp divergence)。由此可知，当 warp 内的线程遵循相同的控制流路径时， PPU 利用率达到最大值。

在 SIMT 模型中，warp 中的所有线程一起逐步执行核函数。硬件执行可能有所不同。不建议利用对 warp 执行实际映射到真实硬件的知识。HGGC 编程模型和 SIMT 表明 warp 中的所有线程一起通过代码前进。只要遵循编程模型，硬件就可以优化掩码通道，这对程序透明。如果程序违反此模型，可能导致未定义行为，在不同 PPU 硬件上表现不同。

编写 HGGC 代码时不一定需要考虑 warps，理解 warp 执行模型有助于理解全局内存合并 (global memory coalescing) 和共享内存库访问模式 (shared memory bank access patterns) 等概念。一些高级编程技术使用线程块内 warps 的专业化来限制线程发散并最大化利用率。这种方法和其他优化利用了线程在执行时被分组为 warps 的知识。

warp 执行的一个含义是将线程块指定为 32 倍数的线程总数。使用任意数量的线程都是合法的，但如果总数不是 32 的倍数，则线程块的最后一个 warp 在整个执行过程中将有一些未使用的通道。这可能导致该 warp 的功能单元利用率和内存访问不是最优的。

SIMT 常与单指令多数据（Single Instruction Multiple Data, SIMD）并行性进行比较，但两者有重要区别。在 SIMD 中，执行遵循单一控制流路径；而在 SIMT 中，每条线程被允许遵循独立的控制流路径，因此 SIMT 不像 SIMD 那样受限于固定的数据宽度。有关 SIMT 的更详细讨论，请参见 [线程束 (Warps) 和 SIMT](#132-线程束warps和simt)。


### 1.4. HGGC 软硬件生态 {#14-hggc-软硬件生态}

T-Head SAIL 平台由众多软硬件组件及关键技术构成，旨在实现异构系统上的计算能力。本章旨在介绍 SAIL 平台的一些基本概念与核心组件，这些内容对于应用程序开发人员而言至关重要。与"并行编程基础"一章类似，本章的内容不局限于特定的编程语言，而是适用于所有基于 SAIL 平台进行开发的场景。

#### 1.4.1. 计算能力（Compute Capability） {#141-计算能力compute-capability}

每一款 PPU 都拥有一个"计算能力"（Compute Capability，简称 CC）编号。该编号不仅指明了该 PPU 所支持的各项功能特性，还详细规定了其相关的硬件参数。
目前 PPU 支持的 ARCH 是 ppu_10 和 ppu_15，在实际编译使用时，可以通过 `-arch=<value>` 的形式来指定，也支持多代架构的混合编译，具体配置说明可以查询 [T-Head SAIL HGCC](/docs_center/doc_detail/index.html?projectId=39&chapterId=1056) 的编译文档。

#### 1.4.2. SAIL 工具包与 HGGC 驱动程序 {#142-sail-工具包与-hggc-驱动程序}

HGGC 驱动程序可被视为 PPU 的操作系统。HGGC 驱动程序是一个软件组件，必须安装在宿主系统的操作系统上；对于 PPU 的所有用途，该驱动程序都是必不可少的。HGGC 驱动程序是 SAIL 平台的基础。
SAIL 工具包是一套包含库、头文件和工具的集合，用于编写、构建和分析利用 PPU 计算能力的软件。SAIL 工具包是一个独立于 HGGC 驱动程序的软件产品。

HGGC 运行时（HGGC runtime）是 SAIL 工具包所提供的一系列库中的一个特例。HGGC 运行时既提供了一套 API，也提供了一些语言扩展，用于处理诸如分配内存、在 PPU 与其他 PPU 或 CPU 之间复制数据、以及启动核函数（kernels）等常见任务。HGGC 运行时中的 API 组件被称为 HGGC 运行时 API。

##### 1.4.2.1. HGGC 运行时 API 与 HGGC 驱动 API {#1421-hggc-运行时api与hggc驱动api}
HGGC 运行时 API 通过提供隐式初始化、context 管理和 module 管理来简化设备代码管理。对于大多数操作使用 HGGC 运行时 API 就足够了。HGGC 驱动 API 是更底层的 API，直接由 HGGC 驱动程序提供。HGGC 运行时 API 是在 HGGC 驱动 API 之上实现的。

一般情况下，使用运行时 API 即可满足需求；驱动 API 同样可以实现所有相同的功能，此外某些特定功能仅可通过驱动 API 来实现。应用程序既可以单独使用其中一套，也可以将两套 API 结合起来交替使用——详见 [第 3.1 节 HGGC 底层驱动接口](#31-hggc-底层驱动接口)。

#### 1.4.3. T-Head Instruction eXtension (TIX) {#143-t-head-instruction-extension-tix}

SAIL 平台中有一个被隐藏的层级，即（T-Head Instruction eXtension，简称 TIX）虚拟指令集架构（ISA）。[TIX](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042) 是一种专为 PPU 设计的高级汇编语言。TIX 在实际 PPU 硬件的物理指令集架构之上，提供了一个抽象层。

用户能通过 [Inline TIX](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#13-与CC混合编程inline-tix) 内联汇编的方式在 C++ 代码中使用。当前并不支持直接使用 TIX 汇编语言独立编写程序，后续 SDK 版本会逐步开放。

特定领域语言（DSL）以及高级语言编译器主要通过 LLVM IR（ 未来也可以通过 TIX 代码 ）作为中间表示，随后利用 T-Head 提供的离线编译工具或即时（JIT）编译工具，将其转化为可执行的 PPU 二进制代码。这一机制使得 SAIL 平台不仅局限于 T-Head 官方工具（例如编译器 hgcc）所支持的那些语言，还能通过其他语言进行编程。

#### 1.4.4. HGbins and HGFatbins {#144-hgbins-and-hgfatbins}

HGGC 应用程序和库通常使用 C++ 等高级语言编写。这些高级语言代码会被编译成适用于物理 PPU 的实际二进制代码，这种二进制代码被称为 *HGGC 二进制*（HGGC binary），简称 *hgbin*。每个 hgbin 都针对特定的 ARCH 版本（例如 ppu001/ppu0015）采用特定的二进制格式。

凡是涉及 PPU 计算的可执行文件和库二进制文件，其中都同时包含了 CPU 代码和 PPU 代码。其中的 PPU 代码被存储在一个名为 [hgfatbin](/docs_center/doc_detail/index.html?projectId=39&chapterId=1062)（HG Fat Binary）的容器中。一个 hgfatbin 容器内可以包含针对多个不同目标平台（即不同 PPU 架构）的 hgbin 代码。例如，一个应用程序在构建时，可以同时包含适用于多种不同 PPU 架构（即不同 Arch 版本）的二进制代码。我们将这种编程应用能力称为"混合编译"，即一次编译，在多个 PPU 支持的 ARCH 上都可以执行。当该应用程序运行时，其 PPU 代码会被加载到特定的 PPU 上，系统随后会从 hgfatbin 容器中选取最适合当前 PPU 的那份二进制代码来执行。


### 1.5. HGGC 特性概览 {#15-hggc-特性概览}

本编程指南的第 1-3 章将介绍 HGGC 与 PPU 编程，涵盖概念层面的基础主题，并配以简单的代码示例。本指南第 4 部分中描述具体 HGGC 特性的各节，默认读者已掌握本指南第 1-3 章所覆盖的概念。

HGGC 具有许多适用于不同问题的特性。并非所有特性都适用于每一种使用场景。本章旨在介绍这些特性，并说明它们的预期用途以及可能帮助解决的问题。各特性会按其旨在解决的问题类型进行粗略分类排序。一些特性（例如 HGGC 图）可能同时适用于多个类别。

#### 1.5.1. 提升算子性能指引 {#151-提升算子性能指引}

本节概述的特性都旨在帮助算子开发者最大化其算子（kernels）的性能。

##### 1.5.1.1. 避免线程共享内存冲突 {#1511-避免线程共享内存冲突}

为提升性能，线程共享内存的数据应该尽量对齐到 16 字节。对共享内存的访问除了需避免 bank 冲突外，还应在分组之间尽量实现负载均衡，降低分组冲突。

在 ppu001 架构中，线程共享内存被划分为两个分组（groups），每个分组每拍可提供 128 字节的吞吐带宽。当多个线程同时访问同一分组时，将引发分组冲突，可能导致性能下降。当多个请求映射到同一分组时，这些请求将被序列化处理，导致访问延迟增加。

需要注意的是，如果共享内存读写指令访问的地址间隔为 256 字节（即地址相差 256B 的整数倍），这些地址会映射到同一个分组，从而容易造成分组冲突。

> 例如：请求 1 访问地址 0，请求 2 访问地址 256B，请求 3 访问地址 512B。这些地址虽然物理上相隔较远，但会映射到同一个分组，从而导致冲突。

因此，为最大化共享内存带宽利用率，应避免所有 warp 集中访问同一个分组，应在两个分组之间尽量实现负载均衡。实践中，若请求地址呈线性连续分布，通常能自然地将访问分散到不同分组，从而有效避免分组冲突。

分组冲突示意图：4 个 warp 的请求都进了分组 0。分组 0 的每个颜色框代表 128 字节。
![warp-bank-group](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793675663929/d144abe24d2d344f910b73e0d0a24c30/tsm_conflict_v2.png)

与之相比，ppu0015 将共享内存的分组数从 2 组增加到了 4 组，同时把每组内的连续地址跨度从 128 字节扩展到了 1024 字节。

##### 1.5.1.2. 原子操作 {#1512-原子操作}

在 ppu001 上，同 warp 的 32 个线程执行全局内存上的原子操作时，应尽量使访问地址连续且按 128 字节对齐。否则性能会显著下降：64 字节访问模式的性能仅为 128 字节模式的一半，32 字节模式的性能则降至 128 字节模式的四分之一。

ppu0015 对此进行了优化，使得 64 字节和 32 字节访问模式的原子性能与 128 字节模式相当。

##### 1.5.1.3. 小粒度访问 {#1513-小粒度访问}

在 ppu001 上，若全局内存上单条读写指令所访问的地址过于分散，例如分布在多个不同的缓存行（cache line，每行为 128 字节）上，会导致性能下降。因此，应尽量使全局读写的访问地址集中，以减少所涉及的 128 字节缓存行数量。

在 ppu0015 上，同样建议将全局内存上单条读写指令的访问地址尽量集中，但优化目标变为减少所涉及的 64 字节块的数量。

##### 1.5.1.4. 预取 {#1514-预取}

ppu0015 的 load/aiu 指令支持预取提示，可指定预取粒度为 {128B, 256B}。
具体而言：
> 当请求读取 64 字节时，实际会将 128 字节数据预取到最后一级缓存（LLC）；
> 当请求读取 128 字节时，实际会将 256 字节数据预取到 LLC。

##### 1.5.1.5. 异步指令 {#1515-异步指令}

建议尽可能使用 AIU（AI accelerator in compute Unit，计算单元内 AI 加速器）相关的异步指令替代同步读写指令。AIU 指令可直接将数据从设备内存搬运至线程共享内存，不仅减少了指令数量，还能利用专用硬件模块完成地址计算，从而避免使用通用算术指令进行地址生成，提升整体执行效率。

##### 1.5.1.6. 缓存策略 {#1516-缓存策略}

PPU 的读写指令支持多种缓存策略选项。当数据缺乏时间或空间局部性、预期不会产生缓存命中时，建议选择绕过缓存，以避免占用宝贵的缓存空间，防止有用数据被提前替换出去。
PPU 硬件配备三级缓存，各级缓存的绕过行为可独立控制，具体配置请参考 [T-Head SAIL TIX 编程指南](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042) 中相关指令的说明文档。

##### 1.5.1.7. 低精度运算 {#1517-低精度运算}

ppu001 的张量计算核支持 FP16 和 BF16 数据格式。
ppu0015 在此基础上新增对 FP8 的支持，并率先兼容开放计算项目（OCP）定义的 MXFP4（Microscaling FP4，微缩放 4 位浮点格式）数据格式。

FP8 支持两种元素格式：E4M3 和 E5M2。
MXFP4 采用 E2M1 元素格式，每 32 个元素共享一个缩放因子；该缩放因子采用 E8M0 格式。ppu0015 的 MMA FP4 指令可同时接收两个输入矩阵及其对应的缩放因子。
借助低精度计算，可显著提升计算吞吐：

> FP8 的理论算力可达 FP16 的 2 倍；
> FP4 的理论算力可达 FP16 的 4 倍。

FP8 适用于模型训练和推理场景，而 FP4（包括 MXFP4）则主要面向推理应用。

#### 1.5.2. 改善延迟 {#152-改善延迟}

本节所述的各项特性具有一个共同目标，即降低某种类型的延迟，尽管不同特性所针对的延迟类型各不相同。总体而言，这些特性主要关注核函数启动层级或更高层级的延迟，而不涉及核函数内部的 PPU 内存访问延迟。

##### 1.5.2.1. 流有序内存分配器 {#1521-流有序内存分配器}

[流有序内存分配器](#42-流有序内存分配器) 允许程序将 PPU 内存的分配与释放按顺序编排到一个 [HGGC stream](#23-异步编程模式) 中。不同于会立即执行的 `hggcMalloc` 和 `hggcFree`，`hggcMallocAsync` 和 `hggcFreeAsync` 会将一次内存分配或释放操作插入到一个 HGGC stream 中。

##### 1.5.2.2. HGGC 图 {#1522-hggc图}

[HGGC 图](#41-hggc-图) 使应用能够指定一系列 HGGC 操作（例如 kernel 启动或内存拷贝）以及这些操作之间的依赖关系，从而可以在 PPU 上高效执行。使用 [HGGC 流](#23-异步编程模式) 也可以获得类似行为；事实上，用于创建图的机制之一称为"stream capture"，它允许将一个 stream 上的操作记录到一个 HGGC 图中。Graphs 也可以使用 HGGC graphs API 创建。

一旦图被创建，它就可以被实例化并执行多次。这对于指定将被重复执行的工作负载很有用。图在减少调用 HGGC 操作相关的 CPU 启动开销方面提供了一些性能收益，同时也能启用只有在提前指定整个工作负载时才可用的优化。

[第 4.1 节](#41-hggc-图) 描述并演示了如何使用 HGGC 图。

#### 1.5.3. 功能特性 {#153-功能特性}

这里描述的特性有一个共同点：它们旨在启用额外的能力或功能。

##### 1.5.3.1. 扩展 PPU 内存 {#1531-扩展-ppu-内存}

扩展 PPU 内存是一个在 ICN 连接的系统上可用的特性，它使得在 PPU 内能够高效访问系统中的所有内存。

##### 1.5.3.2. 进程间通信 {#1532-进程间通信}

对于非常大的计算，常见做法是将多个 PPU 一起使用，以便利用更多内存与更多计算资源，共同在一个问题上协同工作。在单个系统内（或以集群计算术语来说的一个节点内），可以在单个 host 进程中使用多个 PPU。

也常见的做法是使用独立的 host 进程，跨越单台计算机或多台计算机。当多个进程一起工作时，它们之间的通信称为进程间通信。HGGC 进程间通信（HGGC IPC）提供了在不同进程之间共享 PPU 缓冲区的机制。[第 4.6 节](#46-进程间通信与虚拟内存管理) 解释并演示了如何使用 HGGC IPC 来协调与通信不同的 host 进程。


#### 1.5.4. 细粒度控制 {#154-细粒度控制}

##### 1.5.4.1. 虚拟内存管理 {#1541-虚拟内存管理}

如 [第 2.4 节](#24-统一和系统内存) 所提到的，系统中的所有 PPU 以及 CPU 内存共享一个统一的虚拟地址空间。大多数应用可以使用 HGGC 提供的默认内存管理，而无需改变其行为。不过，[HGGC 驱动 API](#31-hggc-底层驱动接口) 为有需要的场景提供了对该虚拟内存空间布局的高级且细粒度的控制。这主要适用于控制在 PPU 之间共享缓冲区的行为（既包括单个系统内，也包括跨多个系统的共享）。

[虚拟内存管理](#463-虚拟内存管理) 覆盖了 HGGC 驱动 API 提供的控制项、它们如何工作，以及开发者在何时可能会发现它们具有优势。

## 2. 核心编程 {#2-核心编程}

### 2.1. 用 C++ 编写 HGGC 程序 {#21-用-c-编写-hggc-程序}

本章通过介绍如何在 C++ 中使用 HGGC 的核心特性来展示 HGGC 并行编程的实践方法。
本编程指南重点关注 HGGC 运行时 API。HGGC 运行时 API 是使用 C++ 编写 HGGC 代码最常用的方式，并且构建于底层的 HGGC 驱动程序 API 之上。1.4.2.1 HGGC 运行时 API 与 HGGC 驱动 API 这一节讨论了这两个 API 之间的区别，而 3.1.4 运行时 API 与驱动程序 API 之间的互操作性则讨论了混合使用这两种 API 的编码方法。
本指南假定已安装 SAIL 工具包和 HGGC 驱动程序，并且存在支持的 PPU。

#### 2.1.1. 使用 hgcc 编译 {#211-使用ppu-hgcc编译}

用 C++ 编写的 PPU 代码通过特有的 HGGC 编译器 `hgcc` 进行编译。`hgcc` 是一个编译器驱动程序，它提供简洁且风格熟悉的命令行选项，简化了 C++ 代码的编译过程，并通过调用实现不同编译阶段的一系列工具来执行它们。
第二章的 hgcc 部分涵盖了 hgcc 的常见用例，而完整的文档由 [T-Head SAIL HGCC 用户手册](/docs_center/doc_detail/index.html?projectId=39&chapterId=1056) 提供。

#### 2.1.2. 核函数程序 {#212-核函数程序}

如并行编程基础一章中所述，在 PPU 上执行并且可以从主机 (host) 调用的函数称为核函数 (kernel)。核函数程序被设计成可以同时由许多并行线程运行。

##### 2.1.2.1. 核函数指定方式 {#2121-核函数指定方式}

在 HGGC C++ 中，通过 `__global__` 限定符标记一个函数为核函数入口。编译器据此将函数编译为可在 PPU 上并行执行的设备代码。核函数是返回类型为 `void` 的函数。核函数启动是一种使核函数开始运行的操作，通常从 CPU 发起。

```cpp
// 核函数定义：逐元素 ReLU 激活
__global__ void relu_activate(const float* input, float* output, int len)
{
    int tid = threadIdx.x + blockDim.x * blockIdx.x;
    if (tid < len) {
        output[tid] = (input[tid] > 0.0f) ? input[tid] : 0.0f;
    }
}
```

##### 2.1.2.2. 核函数的启动 {#2122-核函数的启动}

核函数启动期间决定指定执行在该核函数程序的线程数量，这也被称为执行配置。同一个核函数的不同调用可能使用不同的执行配置，如使用不同的线程数或线程块 (thread block) 数执行。
目前有两种从 CPU host 代码启动核函数的方法：三重尖括号 (triple chevron) 符号法和 hggcLaunchKernelEx 函数法。三重尖括号是最常用的启动核函数的方法，这里会做介绍。使用 hggcLaunchKernelEx 启动核函数的示例会在 2.3.4.1 节中详细展示和讨论。

###### 2.1.2.2.1. 三重尖括号表示法 {#21221-三重尖括号表示法}

三重尖括号是 HGGC C++ 语言扩展，用于启动核函数。之所以叫这个名字是因为它使用三个尖括号字符封装核函数启动的执行配置，即 `<<<>>>`。执行配置参数以逗号分隔列表的形式在尖括号内指定，类似于函数调用的参数。

下面给出一个完整的可运行示例，展示如何使用三重尖括号启动核函数：

```cpp
__global__ void relu_activate(const float* input, float* output, int len)
{
    int tid = threadIdx.x + blockDim.x * blockIdx.x;
    if (tid < len) {
        output[tid] = (input[tid] > 0.0f) ? input[tid] : 0.0f;
    }
}

int main()
{
    int len = 1024;
    // ... 分配并初始化 d_input, d_output ...
    int threadsPerBlock = 256;
    int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
    relu_activate<<<blocksPerGrid, threadsPerBlock>>>(d_input, d_output, len);
    // ...
}
```

三重尖括号是 HGGC C++ 语言扩展，用于指定核函数的执行配置。`<<<>>>` 中的参数以逗号分隔，前两个参数分别是网格（grid）维度和线程块维度。当使用一维线程块或网格时，可以使用整数指定维度。

上述代码启动 4 个线程块，每块 256 个线程，共 1024 个线程并行执行 ReLU 操作。每个线程根据其全局索引 `tid` 处理数组中的一个元素。

线程块的大小受 CU 物理资源约束：同一块的全部线程在同一 CU 上执行，并必须共享该 CU 的资源。在当前 PPU 上，一个线程块最多可包含 1024 个线程。如果资源允许，可以在一个 CU 上同时调度多个线程块。

核函数启动相对于 host 主机线程来说是异步的。也就是说，host 下发核函数 Kernel 后在 PPU 上启动核函数执行，但 host 主机代码不会等待核函数完成（下发甚至无需感知核函数 kernel 开始执行）在 PPU 上的执行就继续前进。这种异步的执行方式，需要某种形式的 PPU 与 CPU 之间的同步。
当使用 2 维或 3 维网格或线程块时，HGGC 上网格和线程块维度都使用 dim3（三维）类型表示。以下代码片段展示了使用 16 乘 16 线程块网格启动二维矩阵逐元素求和核函数的情况，每个线程块是 8 乘 8。
```cpp
int main()
{
    ...
    dim3 grid(16,16);
    dim3 block(8,8);
    elementwise_add_2d<<<grid, block>>>(A, B, C);
    ...
}
```

##### 2.1.2.3. 线程和网格索引内置函数 {#2123-线程和网格索引内置函数}

在核函数代码内部，HGGC 提供了访问执行配置参数和线程或块索引的内置函数。

具体包括：
- `threadIdx` 提供线程在其线程块内的索引。线程块中的每个线程都会有不同的索引。
- `blockDim` 提供线程块的尺寸，这是在核函数启动的执行配置中指定的。
- `blockIdx` 提供线程块在网格中的索引。每个线程块都会有不同的索引。
- `gridDim` 提供网格的尺寸，这是在核函数启动时执行配置中指定的。

这些内置函数都具有.x、.y 和.z 成员的三维向量。未经启动配置指定的维度默认为 1。threadIdx 和 blockIdx 是从零开始索引的。也就是说，threadIdx.x 的值范围从 0 到包括 blockDim.x-1。.y 和.z 分别在其各自的维度中相同地工作。
类似地，blockIdx.x 的值范围从 0 到包括 gridDim.x-1，对于.y 和.z 维度也是一样。
这些允许单个线程确定应执行的工作。回到 `relu_activate` 核函数程序，该核函数接受输入数组、输出数组和长度三个参数。核函数对输入数组执行逐元素的 ReLU 激活并将结果写入输出数组。核函数进行了并行化，使得每个线程处理一个元素。每个线程计算自己的全局索引 `tid`，从而确定其负责处理的数据位置。
```cpp
__global__ void relu_activate(const float* input, float* output, int len)
{
    // 计算此线程的全局索引
    int tid = threadIdx.x + blockDim.x * blockIdx.x;
    if (tid < len) {
        output[tid] = (input[tid] > 0.0f) ? input[tid] : 0.0f;
    }
}

int main()
{
    // input 和 output 是长度为 len 的向量
    int len = 1024;
    int threadsPerBlock = 256;
    int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
    relu_activate<<<blocksPerGrid, threadsPerBlock>>>(d_input, d_output, len);
}
```
在这个例子中，使用 4 个线程块（`blocksPerGrid = 4`），每块 256 个线程来处理长度为 1024 的向量。在第一个线程块中，`blockIdx.x` 为 0，因此每个线程的 `tid` 就是它的 `threadIdx.x`（范围 0–255）。在第二个线程块中，`blockIdx.x` 为 1，所以 `tid = threadIdx.x + 256`（范围 256–511）。以此类推，第四个线程块中 `tid` 的范围为 768–1023。

###### 2.1.2.3.1. 边界检查 {#21231-边界检查}

在上面的 `relu_activate` 示例中，边界检查已经内嵌在核函数逻辑中（`if (tid < len)`）。这是因为启动的线程总数（`blocksPerGrid * threadsPerBlock`）通常会向上取整到线程块大小的倍数，当数组长度不是线程块大小的整数倍时，最后一个线程块中会有部分线程的 `tid` 超出有效范围。

边界检查确保这些"超额"线程不会执行越界内存访问。虽然启动少量不做工作的线程并不会产生显著的性能开销，但应当避免启动整块都无工作可做的线程块。

通常，所需线程块数量可以通过将数据长度除以线程块大小并向上取整得到。常见写法为：`blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock`。

#### 2.1.3. PPU 计算中的内存 {#213-ppu-计算中的内存}

为了使用上面展示的 `relu_activate` 核函数，输入和输出数组必须位于 PPU 可访问的内存中。有几种不同的方法可以做到这一点，这里将演示其中两种。 PPU 上代码可用的内存空间已在 HGGC 并行编程基础一节中介绍，并在 [编写 HGGC SIMT 核函数程序](#22-编写hggc-simt-核函数程序) 中更详细地覆盖。

##### 2.1.3.1. Unified Memory {#2131-unified-memory}

Unified memory 是 HGGC runtime 的一个特性，它让 HGGC Driver 管理 host 与 device(s) 之间的数据移动。内存通过 `hggcMallocManaged` API 分配，或通过 `__managed__` 标识符声明变量。HGGC Driver 会确保当 PPU 或 CPU 任一方尝试访问时，这段内存对其可访问。

下面的代码展示了一个完整函数，用于启动 `relu_activate` 核函数，并对将在 PPU 上使用的输入和输出数组使用 unified memory。`hggcMallocManaged` 分配的 buffer 可被 CPU 或 PPU 访问。它们使用 `hggcFree` 释放。
```cpp
void unified_memory_example(int len)
{
    float* input  = nullptr;
    float* output = nullptr;
    float* expected = (float*)malloc(len * sizeof(float));

    hggcMallocManaged(&input,  len * sizeof(float));
    hggcMallocManaged(&output, len * sizeof(float));

    // 在主机端填充测试数据
    fill_random(input, len);

    // 启动核函数
    int threadsPerBlock = 256;
    int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
    relu_activate<<<blocksPerGrid, threadsPerBlock>>>(input, output, len);
    hggcDeviceSynchronize();

    // 主机端参考计算
    cpu_relu(input, expected, len);

    // 比较结果
    if (arrays_match(output, expected, len)) {
        printf("Unified Memory: 结果一致\n");
    } else {
        printf("Unified Memory: 结果不一致\n");
    }

    hggcFree(input);
    hggcFree(output);
    free(expected);
}
```

Unified memory 在 HGGC 支持的所有操作系统和 PPU 上都受支持，但具体支持程度取决于硬件能力和驱动版本（参见 [统一内存模式](#2421-统一内存模式)）。[统一内存](#242-统一内存) 提供了更多细节。在一些 Linux 系统上（例如具备 统一和系统内存的系统），所有系统内存会自动成为 unified memory，无需使用 `hggcMallocManaged` 或 `__managed__` 标识符。

##### 2.1.3.2. 显式内存管理 {#2132-显式内存管理}

显式管理内存分配以及在不同内存空间之间的数据迁移可以帮助提升应用性能，但也会使代码更冗长。下面的代码使用 `hggcMalloc` 在 PPU 上显式分配内存。 PPU 上的内存使用与前一个 unified memory 示例相同的 `hggcFree` API 释放。


```cpp
void explicit_memory_example(int len)
{
    size_t bytes = len * sizeof(float);

    // 主机端缓冲区
    float* h_input  = nullptr;
    float* h_output = nullptr;
    float* expected = (float*)malloc(bytes);

    // 设备端缓冲区
    float* d_input  = nullptr;
    float* d_output = nullptr;

    // 集中完成所有主机端分配
    hggcMallocHost(&h_input,  bytes);
    hggcMallocHost(&h_output, bytes);

    // 初始化输入数据
    fill_random(h_input, len);

    // 集中完成所有设备端分配
    hggcMalloc(&d_input,  bytes);
    hggcMalloc(&d_output, bytes);

    // 集中完成所有 H2D 拷贝
    hggcMemcpy(d_input, h_input, bytes, hggcMemcpyDefault);
    hggcMemset(d_output, 0, bytes);

    // 启动核函数
    int threadsPerBlock = 256;
    int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
    relu_activate<<<blocksPerGrid, threadsPerBlock>>>(d_input, d_output, len);
    hggcDeviceSynchronize();

    // 将结果拷回主机
    hggcMemcpy(h_output, d_output, bytes, hggcMemcpyDefault);

    // 主机端参考计算
    cpu_relu(h_input, expected, len);

    // 验证结果
    if (arrays_match(h_output, expected, len)) {
        printf("Explicit Memory: 结果一致\n");
    } else {
        printf("Explicit Memory: 结果不一致\n");
    }

    // 释放资源
    hggcFree(d_input);
    hggcFree(d_output);
    hggcFreeHost(h_input);
    hggcFreeHost(h_output);
    free(expected);
}
```

HGGC API `hggcMemcpy` 用于将数据从位于 CPU 的 buffer 复制到位于 PPU 的 buffer。除了 destination pointer、source pointer 和以字节为单位的大小之外，`hggcMemcpy` 的最后一个参数是 `hggcMemcpyKind_t`。这可以取值如 `hggcMemcpyHostToDevice`（从 CPU 到 PPU 的复制）、`hggcMemcpyDeviceToHost`（从 PPU 到 CPU 的复制）、或 `hggcMemcpyDeviceToDevice`（在 PPU 内或 PPU 间的复制）。

在这个示例中，`hggcMemcpy` 的最后一个参数传入 `hggcMemcpyDefault`。这会导致 HGGC 使用 source 和 destination pointer 的值来确定要执行的 copy 类型。

`hggcMemcpy` API 是同步的。也就是说，它直到 copy 完成才返回。异步 copy 在 [在 HGGC 流中启动内存传输](#23-异步编程模式) 中介绍。

代码使用 `hggcMallocHost` 在 CPU 上分配内存。这会在 host 上分配 [page-locked memory](#24-统一和系统内存)，可以提升 copy 性能，并且对 [asynchronous](#23-异步编程模式) memory transfers 是必要的。对于将用于在 PPU 与 CPU 内存间进行数据传输的 CPU buffer，使用 page-locked memory 是良好实践。如果 page-locked 的 host 内存过多，某些系统上性能可能会下降。最佳实践是只对将用于向 PPU 发送或从 PPU 接收数据的 buffer 进行 page-lock。

##### 2.1.3.3. 内存管理与应用性能 {#2133-内存管理与应用性能}

从上面的示例可以看出，显式内存管理更冗长，需要程序员显式指定 host 与 device 之间的 copy。这既是显式内存管理的优势也是劣势：它提供了更强的控制能力，可以控制数据何时在 host 与 device 之间复制、内存驻留在何处、以及到底分配了哪些内存在何处。显式内存管理可以通过控制 memory transfers 并将其与其他计算重叠，从而带来性能机会。

使用 unified memory 时，也有 HGGC APIs（将在 [Memory Advise and Prefetch](#24-统一和系统内存) 中覆盖），用于向管理内存的 HGGC driver 提供提示，从而在使用 unified memory 时也能获得一些类似显式内存管理的性能收益。

#### 2.1.4. 同步 CPU 与 PPU {#214-同步-cpu-与-ppu}

如在 Launching Kernels 中所提到的，kernel launches 相对于调用它们的 CPU thread 是异步的。这意味着 CPU thread 的控制流会在 kernel 完成之前继续执行，甚至可能在 kernel 还未 launch 之前就继续执行。为了保证在 host code 继续执行之前 kernel 已完成执行，需要某种同步机制。

同步 PPU 与 host thread 的最简单方式是使用 `hggcDeviceSynchronize`，它会阻塞 host thread，直到 PPU 上此前提交的所有工作都完成。在本章示例中这就足够了，因为 PPU 上只执行单个操作。在更大的应用中，可能有多个 [streams](#23-异步编程模式) 在 PPU 上执行工作，而 `hggcDeviceSynchronize` 会等待所有 stream 中的工作完成。在这些应用中，推荐使用 [Stream Synchronization](#23-异步编程模式) APIs 仅与特定 stream 同步，或使用 [HGGC Event](#23-异步编程模式)。这些将在 异步执行 章节中详细介绍。

#### 2.1.5. 组合使用 {#215-组合使用}

下面的列表展示了本章引入的 ReLU 激活核函数的全部代码，以及所有 host code 与用于验证结果正确性的 utility functions。这些示例默认使用 1024 的数组长度，但也接受通过可执行文件命令行参数传入不同的长度。

Unified Memory

```cpp
#include <hggc_runtime_api.h>
#include <memory.h>
#include <cstdlib>
#include <ctime>
#include <stdio.h>

__global__ void relu_activate(const float* input, float* output, int len)
{
  int tid = threadIdx.x + blockIdx.x * blockDim.x;
  if (tid < len)
  {
    output[tid] = (input[tid] > 0.0f) ? input[tid] : 0.0f;
  }
}

void fill_random(float* arr, int length)
{
  std::srand(std::time({}));
  for (int i = 0; i < length; i++)
  {
    arr[i] = (rand() / (float)RAND_MAX) * 2.0f - 1.0f;
  }
}

void cpu_relu(const float* input, float* output, int length)
{
  for (int i = 0; i < length; i++)
  {
    output[i] = (input[i] > 0.0f) ? input[i] : 0.0f;
  }
}

bool arrays_match(const float* a, const float* b, int length, float epsilon=0.00001f)
{
  for (int i = 0; i < length; i++)
  {
    if (fabs(a[i] - b[i]) > epsilon)
    {
      printf("Index %d mismatch: %f != %f", i, a[i], b[i]);
      return false;
    }
  }
  return true;
}

void unified_memory_example(int len)
{
  float* input  = nullptr;
  float* output = nullptr;
  float* expected = (float*)malloc(len * sizeof(float));

  hggcMallocManaged(&input,  len * sizeof(float));
  hggcMallocManaged(&output, len * sizeof(float));

  fill_random(input, len);

  int threadsPerBlock = 256;
  int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
  relu_activate<<<blocksPerGrid, threadsPerBlock>>>(input, output, len);
  hggcDeviceSynchronize();

  cpu_relu(input, expected, len);

  if (arrays_match(output, expected, len))
  {
    printf("Unified Memory: 结果一致\n");
  }
  else
  {
    printf("Unified Memory: 结果不一致\n");
  }

  hggcFree(input);
  hggcFree(output);
  free(expected);
}


int main(int argc, char** argv)
{
  int len = 1024;
  if (argc >= 2)
  {
    len = std::atoi(argv[1]);
  }
  unified_memory_example(len);
  return 0;
}
```

Explicit Memory Management

```cpp
#include <hggc_runtime_api.h>
#include <memory.h>
#include <cstdlib>
#include <ctime>
#include <stdio.h>

__global__ void relu_activate(const float* input, float* output, int len)
{
  int tid = threadIdx.x + blockIdx.x * blockDim.x;
  if (tid < len)
  {
    output[tid] = (input[tid] > 0.0f) ? input[tid] : 0.0f;
  }
}

void fill_random(float* arr, int length)
{
  std::srand(std::time({}));
  for (int i = 0; i < length; i++)
  {
    arr[i] = (rand() / (float)RAND_MAX) * 2.0f - 1.0f;
  }
}

void cpu_relu(const float* input, float* output, int length)
{
  for (int i = 0; i < length; i++)
  {
    output[i] = (input[i] > 0.0f) ? input[i] : 0.0f;
  }
}

bool arrays_match(const float* a, const float* b, int length, float epsilon=0.00001f)
{
  for (int i = 0; i < length; i++)
  {
    if (fabs(a[i] - b[i]) > epsilon)
    {
      printf("Index %d mismatch: %f != %f", i, a[i], b[i]);
      return false;
    }
  }
  return true;
}

void explicit_memory_example(int len)
{
  size_t bytes = len * sizeof(float);

  float* h_input  = nullptr;
  float* h_output = nullptr;
  float* expected = (float*)malloc(bytes);

  float* d_input  = nullptr;
  float* d_output = nullptr;

  hggcMallocHost(&h_input,  bytes);
  hggcMallocHost(&h_output, bytes);

  fill_random(h_input, len);

  hggcMalloc(&d_input,  bytes);
  hggcMalloc(&d_output, bytes);

  hggcMemcpy(d_input, h_input, bytes, hggcMemcpyDefault);
  hggcMemset(d_output, 0, bytes);

  int threadsPerBlock = 256;
  int blocksPerGrid = (len + threadsPerBlock - 1) / threadsPerBlock;
  relu_activate<<<blocksPerGrid, threadsPerBlock>>>(d_input, d_output, len);
  hggcDeviceSynchronize();

  hggcMemcpy(h_output, d_output, bytes, hggcMemcpyDefault);

  cpu_relu(h_input, expected, len);

  if (arrays_match(h_output, expected, len))
  {
    printf("Explicit Memory: 结果一致\n");
  }
  else
  {
    printf("Explicit Memory: 结果不一致\n");
  }

  hggcFree(d_input);
  hggcFree(d_output);
  hggcFreeHost(h_input);
  hggcFreeHost(h_output);
  free(expected);
}


int main(int argc, char** argv)
{
  int len = 1024;
  if (argc >= 2)
  {
    len = std::atoi(argv[1]);
  }
  explicit_memory_example(len);
  return 0;
}
```

在这些示例中，所有 threads 都在做彼此独立的工作，不需要彼此协调或同步。很多情况下，threads 需要协作并与其他 threads 通信以完成工作。一个 block 内的 threads 可以通过 [共享内存](#22-编写hggc-simt-核函数程序) 共享数据并同步，以协调内存访问。

在 block 层面最基本的同步机制是 `__syncthreads()` intrinsic，它充当一个 barrier：block 中的所有 threads 都必须在此等待，直到所有 threads 都到达之后才允许继续执行。[共享内存](#22-编写hggc-simt-核函数程序) 给出了使用 shared memory 的示例。

为了高效协作，shared memory 预计是靠近每个处理器核心的低延迟内存（很像 L1 cache），而 `__syncthreads()` 预计是轻量的。`__syncthreads()` 只同步单个 thread block 内的 threads。HGGC programming model 不支持 block 间同步。[协作组](#43-协作组) 提供了将同步域设置为非单个 thread block 的机制。

最佳性能通常在同步限制在一个 thread block 内时获得。thread blocks 仍然可以使用 atomic function 来共同处理共享结果，这将在后续章节覆盖。


#### 2.1.6. Runtime 初始化 {#216-runtime-初始化}

HGGC runtime 采用**按需初始化**策略：每个 PPU 设备对应一个 primary [context](#311-上下文-context)，该 context 在首次被任何 runtime API 实际使用时完成创建，并在进程内所有 host 线程间共享。

`hggcInitDevice` 和 `hggcSetDevice` 调用会初始化 runtime 以及与指定 device 关联的 primary context。如果这些调用发生之前就出现 runtime API 请求，runtime 将隐式使用 device 0 并按需自初始化以处理这些请求。这在对 runtime function 调用进行计时时，以及解释第一次进入 runtime 的调用返回的错误码时很重要。

!!! note

    `hggcSetDevice` 在切换当前设备后会触发目标设备的 runtime 初始化（如果尚未完成）。之前的版本会延迟新设备上的 runtime 初始化，直到 `hggcSetDevice` 之后的第一次 runtime 调用。因此务必检查其返回值以捕获初始化错误。

`hggcDeviceReset` 会销毁当前 device 的 primary context。如果在 primary context 被销毁之后仍调用 HGGC 运行时 APIs，则会为该 device 创建一个新的 primary context。

!!! note

    HGGC 运行时依赖进程级全局状态，该状态在 `main` 函数执行前的 C++ 静态初始化阶段尚未就绪，在 `main` 返回后的析构阶段也可能已被销毁。在 `main` 之前或 `main` 之后的程序启动/终止阶段（隐式或显式）调用 HGGC API 会导致未定义行为。

参考手册中错误处理与版本管理相关的函数属于例外——它们不会触发 runtime 初始化。

#### 2.1.7. HGGC 中的错误检查 {#217-hggc-中的错误检查}

每个 HGGC API 都返回一个枚举类型 `hggcError_t` 的值。在示例代码中这些错误通常不被检查。在生产应用中，最佳实践是始终检查并处理每个 HGGC API 调用的返回值。当没有错误时，返回值为 `hggcSuccess`。许多应用选择实现如下所示的 utility macro：

```cpp
#define HGGC_CHECK(expr_to_check) do {            \
    hggcError_t result  = expr_to_check;          \
    if(result != hggcSuccess)                     \
    {                                             \
        fprintf(stderr,                           \
                "HGGC Runtime Error: %s:%i:%d = %s\n", \
                __FILE__,                         \
                __LINE__,                         \
                result,\
                hggcGetErrorString(result));      \
    }                                             \
} while(0)
```

该 macro 使用 `hggcGetErrorString` API，它返回描述特定 `hggcError_t` 值含义的人类可读字符串。使用上述 macro，应用会将 HGGC 运行时 API 调用放在 `HGGC_CHECK(expression)` macro 内，如下所示：

```cpp
HGGC_CHECK(hggcMalloc(&devA, vectorLength*sizeof(float)));
HGGC_CHECK(hggcMalloc(&devB, vectorLength*sizeof(float)));
HGGC_CHECK(hggcMalloc(&devC, vectorLength*sizeof(float)));
```

如果这些调用中任意一个检测到错误，该 macro 会把错误打印到 `stderr`。该 macro 在小型项目中很常见，但在大型应用中也可以适配到日志系统或其他错误处理机制中。

!!! note

    需要注意的是，任何 HGGC API 调用返回的错误状态也可能表示先前发出的异步操作的错误。[异步错误处理](#23-异步编程模式) 更详细地覆盖了这一点。

##### 2.1.7.1. 错误状态 {#2171-错误状态}

HGGC runtime 为每个 host thread 维护一个 `hggcError_t` 状态。其默认值为 `hggcSuccess`，并在发生错误时被覆盖。`hggcGetLastError` 返回当前错误状态，然后将其重置为 `hggcSuccess`。或者，`hggcPeekAtLastError` 返回错误状态但不重置。

使用三重尖括号的 kernel launches 不返回 `hggcError_t`。最佳实践是在 kernel launch 之后立即检查错误状态，以检测 kernel launch 的即时错误或 kernel launch 之前的异步错误。在 kernel launch 后立即检查错误状态得到 `hggcSuccess` 并不意味着 kernel 已成功执行，甚至不意味着它已开始执行。这只验证传递给 runtime 的 kernel launch 参数和执行配置没有触发任何错误，并且错误状态不是 kernel 开始前的历史或异步错误。

推荐的错误检查模式如下：

```cpp
relu_activate<<<blocksPerGrid, threadsPerBlock>>>(d_input, d_output, len);
// check error state after kernel launch
HGGC_CHECK(hggcGetLastError());
// wait for kernel execution to complete
// The HGGC_CHECK will report errors that occurred during execution of the kernel
HGGC_CHECK(hggcDeviceSynchronize());
```

##### 2.1.7.2. 异步错误 {#2172-异步错误}

HGGC kernel launches 和许多运行时 APIs 都是异步的。异步 HGGC 运行时 APIs 将在 [异步执行](#23-异步编程模式) 中详细讨论。HGGC error state 在发生错误时会设置并覆盖。这意味着在异步操作执行期间发生的错误，只有在下次检查错误状态时才会被报告。如前所述，这可能是调用 `hggcGetLastError`、`hggcPeekAtLastError`，或任何返回 `hggcError_t` 的 HGGC API。

当 HGGC 运行时 API functions 返回错误时，错误状态不会被清除。这意味着来自异步错误（例如 kernel 的非法内存访问）的错误码，会被每个 HGGC 运行时 API 返回，直到通过调用 `hggcGetLastError` 清除了错误状态。

!!! note

    `hggcError_t` 值 `hggcErrorNotReady`（可能由 `hggcStreamQuery` 和 `hggcEventQuery` 返回）不被视为错误，且不会被 `hggcPeekAtLastError` 或 `hggcGetLastError` 报告。

#### 2.1.8. Device 与 Host 函数 {#218-device-与-host函数}

`__global__` 标识符用于指示 kernel 的入口点。也就是说，一个会在 PPU 上被并行执行调用的函数。

`__device__` 表示一个函数应为 PPU 编译，并可从其他 `__device__` 或 `__global__` functions 调用。一个函数（包括 class member functions、functors 和 lambdas）可以同时指定为 `__device__` 与 `__host__`，如下例所示。

#### 2.1.9. 变量限定符 {#219-变量限定符}

HGGC C++中提供了如下变量限定符：

  * `__device__` 声明全局内存空间中的变量
  * `__constant__` 声明常量内存空间中的变量
  * `__managed__` 声明统一内存空间中的变量
  * `__shared__` 声明共享内存空间中的变量
如果一个变量没有使用变量限定符，而被声明在 `__device__` 或 `__global__` 内的，一般被认为是局部变量；而声明在 `__device__` 或 `__global__` 外的，被认为是存于系统内存中。

##### 2.1.9.1. 检测设备编译 {#2191-检测设备编译}

当函数被指定为 `__host__ __device__` 时，编译器会收到指令，为该函数同时生成 PPU 和 CPU 版本的代码。在此类函数中，往往需要利用预处理器来指定仅适用于 PPU device 或 CPU host 版本函数的代码。


### 2.2. 编写 HGGC SIMT 核函数程序 {#22-编写hggc-simt-核函数程序}

编写 HGGC C++核函数程序可以像为特定问题编写传统 CPU 代码的方式一样进行。然而， PPU 有一些独特的特性可用于提高性能。了解 PPU 上的线程是如何调度的、它们如何访问内存以及其执行过程如何推进，可以帮助开发人员编写能最大化利用可用计算资源的核函数程序。

---
#### 2.2.1. SIMT 基础 {#221-simt基础}

从开发者的角度来看，HGGC Thread 线程是并行性的基本单位。SIMT 模型允许每个线程维护自己的状态和控制流。从功能的角度来看，每个线程都可以执行一个独立的代码路径。
**性能优化要点**：通过注意让核函数代码在执行同一线程束 (warp) 中的线程时，尽量不发生分叉执行或者分叉执行情况尽可能少，可以获得显著的性能提升。

#### 2.2.2. 线程层次结构 {#222-线程层次结构}

**组织结构**
线程被组织成以下层级：
- **线程块**(Thread Blocks) → **网格**(Grid)

网格可以是一维、二维或三维的，线程块也可以是一维、二维或三维的。

**关键内置变量**

| 变量                  | 描述                                                              |
| --------------------- | ----------------------------------------------------------------- |
| `gridDim.[x\|y\|z]`   | 分别表示 x、y 和 z 维度上网格的大小（核函数启动时设置）           |
| `blockDim.[x\|y\|z]`  | 分别表示 x、y 和 z 维度上线程块的大小（核函数启动时设置）         |
| `blockIdx.[x\|y\|z]`  | 分别表示 x、y 和 z 维度上线程块的索引（根据正在执行的线程块变化） |
| `threadIdx.[x\|y\|z]` | 分别表示 x、y 和 z 维度上线程的索引（根据正在执行的线程变化）     |

**多维线程布局说明**
多维线程块和网格仅是为了方便起见，不会影响性能。线程块中的线程以可预测的方式线性化：

- 第一个索引 x 移动最快，其次是 y，最后是 z
- 连续的 `threadIdx.x` 值表示连续的线程
- `threadIdx.y` 有 `blockDim.x` 的步长
- `threadIdx.z` 有 `blockDim.x * blockDim.y` 的步长

这会影响线程分配给 warp 的方式。


#### 2.2.3. PPU 设备内存空间 {#223-ppu设备内存空间}

HGGC 设备提供多种内存空间供线程在核函数中访问：

**内存类型概览**

| 内存类型        | 范围        | 生命周期 | 物理位置       |
| --------------- | ----------- | -------- | -------------- |
| Global 全局     | Grid 网格   | 应用程序 | device 设备    |
| shared 共享     | Block 块    | 核函数   | 计算单元（CU） |
| local 局部      | Thread 线程 | 核函数   | device 设备    |
| Register 寄存器 | Thread 线程 | 核函数   | 计算单元（CU） |

##### 2.2.3.1. 全局内存 {#2231-全局内存}

全局内存（也称为设备内存）是主要的内存空间，存储核函数中所有线程可访问的数据，类似于 CPU 系统的 RAM。

**特点**：
- 持久化存储：分配及其中数据会一直存在直到释放或应用程序终止

全局内存使用 HGGC API 调用分配，例如 hggcMalloc 和 hggcMallocManaged。可以使用 HGGC 运行时 API 调用（如 hggcMemcpy）将数据从 CPU 内存复制到全局内存。使用 HGGC API 进行的全局内存分配使用 hggcFree 释放。
在核函数启动之前，全局内存由 HGGC API 调用分配和初始化。在核函数执行期间，HGGC 线程可以从全局内存读取数据，并且 HGGC 线程执行操作的结果可以写回全局内存。一旦核函数完成执行，它写入全局内存的结果可以复制回主机或由 PPU 上的其他核函数使用。
由于全局内存可供网格中的所有线程访问，因此必须注意避免线程之间的数据竞争。由于从主机启动的 HGGC 核函数返回类型为 void，核函数计算的数值结果返回给主机的唯一方法是将这些结果写入全局内存。
虽然运算单元可以直接使用全局内存，但是并不能直接访问到全局内容，需要经过缓存，其过程如下图所示：
```text
graph LR
    A[global memory] --> B
    B[LLC] --> C
    C[L2 cache] --> D
    D[L1 cache] --> E[Register]
```

##### 2.2.3.2. 共享内存 {#2232-共享内存}
共享内存是一种高速缓存内存，由同一线程块内的所有线程共享。由于共享内存比全局内存快得多，因此经常用于实现协作算法，如归约 (reduction) 操作。与全局内存相比，共享内存还支持原子操作，并且对相同地址的并发读取能够广播到请求的线程。

下面是一个使用共享内存的例子：

```cpp
#define BLOCK_SIZE 512 // 必须满足2的幂

__global__ void sumReduction(float* d_in, float* d_out) {
    // 1. block内声明共享内存
    __shared__ float sdata[BLOCK_SIZE];

    unsigned int tid = threadIdx.x;
    unsigned int i = blockIdx.x * (blockDim.x * 2) + threadIdx.x;

    // 2. 将数据从全局内存加载到共享内存中
    sdata[tid] = d_in[i] + d_in[i + blockDim.x];
    __syncthreads(); // Ensure all loads are complete

    // 3. 在共享内存中执行规约操作
    for (unsigned int s = blockDim.x / 2; s > 0; s >>= 1) {
        if (tid < s) {
            sdata[tid] += sdata[tid + s];
        }
        // 4. 每次执行后同步所有线程
        __syncthreads();
    }

    // 5. 数据从共享内存写回到全局内存中
    if (tid == 0) d_out[blockIdx.x] = sdata[0];
}
```

共享内存的大小因使用的 PPU 架构而异。每一代 PPU 都有对应的共享内存大小配置。HGGC 运行时 API 提供函数来查询基于 CU 和基于线程块的共享内存大小，使用 hggcDeviceGetAttribute 函数查询 hggcDevAttrSharedMemPerMultiprocessor 和 hggcDevAttrSharedMemPerBlock 设备属性。

共享内存既可以静态分配，也可以动态分配。

###### 2.2.3.2.1. 共享内存的静态分配 {#22321-共享内存的静态分配}

要静态分配共享内存，程序员必须在核函数内部使用 __shared__ 说明符声明一个变量。该变量将在共享内存中分配，并在整个核函数执行期间持续存在。以这种方式声明的共享内存的大小必须在编译时指定。例如，以下代码片段位于核函数主体中，声明了一个 float 类型的包含 1024 个元素的共享内存数组。

```cpp
__shared__ float sharedArray[1024];
```

在此声明后，线程块中的所有线程都将有权访问这个共享内存数组。需要注意避免同一线程块中线程之间的数据竞争，通常使用__syncthreads()。

###### 2.2.3.2.2. 共享内存的动态分配 {#22322-共享内存的动态分配}

要动态分配共享内存，程序员可以在三重尖括号符号中作为第三个（也是可选的）参数指定每个线程块所需的共享内存量（以字节为单位），如下所示：functionName<<<grid, block, sharedMemoryBytes>>>()。

然后，在核函数内部，程序员可以使用 extern __shared__说明符来声明一个将在核函数启动时动态分配的变量。

```cpp
extern __shared__ float sharedArray[];
```

有一个注意事项是，如果想要多个动态分配的共享内存数组，则单个 extern __shared__声明只能处理一种数组类型，需要其他方法来处理多种类型的动态共享内存。

##### 2.2.3.3. 寄存器 {#2233-寄存器}

寄存器是最快速度的内存形式，直接集成在 CU 中。寄存器是本地线程范围的，意味着只有创建它的线程才能访问它。寄存器的数量有限，当核函数使用超过可用数量时，可能会发生寄存器溢出到局部内存 (local memory)。在 PPU 中，物理存在两种寄存器：向量寄存器（VREG，Vector Register）和标量寄存器（SREG，Scalar Register），warp 内每个 thread 都有自己存储空间的是 VREG，而整个 warp 内所有 thread 共享的寄存器被称为 SREG。寄存器被 hgcc 编译器分配和使用，合理的搭配使用 VREG 和 SREG 数量，可以使 Kernel 程序执行更加高效。

##### 2.2.3.4. 局部内存 {#2234-局部内存}

局部内存是每个线程专用的内存区域。虽然名为"局部"，但它实际上驻留在全局内存中，只是作用域限制在单个线程内。局部内存主要用于存储不适合放入寄存器的大型或复杂变量，如大数组或未展开的循环变量。访问局部内存的速度与访问全局内存相似，因此应尽量减少对局部内存的访问。 编译器很可能将其分配至局部存储器（Local Memory）的自动变量包括：
*   编译器无法确定其索引是否为常量的数组；
*   体积过大、会占用过多寄存器空间的大型结构体或数组；
*   当核函数所需的寄存器数量超过可用数量时，任何受此影响的变量（即发生"寄存器溢出"现象时的变量）。

由于局部存储空间位于全局设备存储器中，因此对局部存储器的访问具有与全局存储器访问相同的延迟和带宽。一般核函数程序中少出现或者不出现局部内存，会是程序优化的一个手段。但是局部变量静态的多或者少，并不能决定程序执行效率的高低。

##### 2.2.3.5. 缓存 {#2235-缓存}

PPU 设备有多级缓存结构，包括 L2 和 L1 缓存、LLC 缓存。
LLC（Last Level Cache）位于设备上，由所有 CE 共享，提供上层访问到 HBM 的最后一级缓存。
L2 缓存位于设备上，由 CE 内的所有 CU 共享。L2 缓存的大小可以通过 hggcDeviceGetAttribute 函数查询 hggcDevAttrL2CacheSize 设备属性。
如上面共享内存部分所述，L1 缓存物理上位于每个 CU 上。
L2 和 L1 缓存可通过函数进行控制，这些函数允许开发者指定各种缓存行为。这些函数的详细信息见配置 hgcc 指令配置表。如果不使用这些函数/提示，编译器和运行时将尽力高效地利用缓存。

#### 2.2.4. 原子操作 {#224-原子操作}

原子操作是这样的一组函数：它们在一个内存位置上执行读-修改-写 (read-modify-write) 操作，而不受其他线程可能也在访问同一内存位置的影响。当多个线程试图同时更新相同的数据位置时，原子操作非常重要。
hggc 提供了几种原子操作函数，如 atomicAdd、atomicSub、atomicExch、atomicMin、atomicMax、atomicInc、atomicDec、atomicCAS 和 atomicAnd、atomicOr、atomicXor 等。也引入了新的原子接口 hggc::atomic 和 hggc::atomic_ref，允许开发者指定原子操作的线程范围。原子函数的详细信息在原子函数部分中有涵盖。

以下是一个使用 hggc::atomic_ref 执行设备范围内原子加法的示例，其中 array 是一个浮点数组，result 是指向全局内存中某个位置的浮点指针，该位置是要存储数组之和的地方。

```cpp
__global__ void sumReduction(int n, float *array, float *result) {
   ...
   tid = threadIdx.x + blockIdx.x * blockDim.x;

      hggc::atomic_ref<float, hggc::thread_scope_device> result_ref(result);
      result_ref.fetch_add(array[tid]);
      ...
}
```

应谨慎使用原子函数，因为它们强制执行线程同步，可能会影响性能。

#### 2.2.5. 协作组 {#225-协作组}

协作组（Cooperative Groups）是 HGGC C++中可用的一种软件工具，允许应用程序定义一组可以相互同步的线程，即使该组线程跨越多个线程块、单个 PPU 上的多个网格，甚至跨多个 PPU 也是如此。一般而言，HGGC 编程模型允许线程块内的线程有效同步，但没有提供指定小于线程块的线程组的机制。同样，HGGC 编程模型没有提供跨线程块同步的机制或保证。

协作组通过软件提供这两种能力。协作组允许应用程序创建跨越线程块和簇边界的线程组，尽管这样做会带来一些语义限制和性能影响，这些都在覆盖协作组特性的功能部分中进行了详细介绍。

#### 2.2.6. 核函数启动和占用率 {#226-核函数启动和占用率}

当启动 HGGC 核函数时，HGGC 线程根据核函数启动时指定的执行配置被分组为线程块和网格。一旦核函数启动，调度程序将线程块分配给 CU。哪些线程块被安排在哪些 CU 上执行的细节无法由应用程序控制或查询，调度程序也不做任何排序保证，因此程序不能依赖于特定的调度顺序或方案来进行正确执行。

可以安排在 CU 上的块数取决于给定线程块所需的硬件资源以及 CU 上可用的硬件资源。当核函数首次启动时，调度程序开始将线程块分配给 CU。只要 CU 有空闲的硬件资源不受其他线程块占用，调度程序将继续将线程块分配给 CU。如果在某个时刻没有任何 CU 有能力接受另一个线程块，调度程序将等待 CUs 完成先前分配的线程块。一旦发生这种情况，CU 就可以接受更多工作，调度程序将线程块分配给它们。此过程将持续到所有线程块都被调度和执行完毕。

占用率 (Occupancy) 是衡量活跃 warp 数占 CU 最大潜在 warp 数比例的指标。高占用率有助于掩盖内存延迟，但并非总是最佳选择，因为它可能导致更多的寄存器压力和共享内存争用。占用率计算器 (occupancy calculator) 可帮助确定理论占用率，并建议调整块大小 (block dimensions) 或共享内存使用量以提高占用率。

寄存器使用量直接影响占用率。每个 CU 上有固定数量的寄存器，如果一个核函数每个线程使用大量寄存器，那么在任何时间点可以活动的线程数就会减少。编译器通常会自动优化寄存器使用量，因为编译器和运行时将尝试优化寄存器使用。然而，程序员可以通过 hgcc 的 `--maxrregcount` 选项指定每个线程的最大寄存器数。如果核函数需要的寄存器数量超过了指定的数量，核函数很可能会溢出到局部内存，这将改变核函数的性能特征。在某些情况下，即使发生了溢出，限制寄存器也能允许多个线程块被调度，从而增加占用率，可能导致净性能提升。

通常 kernel 核函数程序占用率可以通过 HGGC 提供的 API 和 PPU 提供的软件工具 Asight 工具采集得到。下表格是一个真实用例中 Asight 工具采集到的一些信息，日常开发可以通过相关字段和提示信息进行分析：

| 指标类别                                 | 数值 | 资源限制                       | 值  |
| ---------------------------------------- | ---- | ------------------------------ | --- |
| Theoretical Occupancy[%]                 | 100  | Block Limit Registers [block]  | 32  |
| TheoreticalActive WarpsperCU[warp/cycle] | 64   | Block Limit Shared Mem [block] | 64  |
| Achieved Occupancy [%]                   | 0.09 | Block Limit Warps[block]       | 16  |
| Achieved Active WarpsPer CU[warp/cycle]  | 0.06 | Block Limit CU [block]         | 64  |

#### 2.2.7. PPU 内存 {#227-ppu内存}

在现代计算系统中，有效利用内存与最大化执行计算的功能单元的使用同样重要。异构系统有多重内存空间， PPU 还包含各种类型的可编程片上内存以及缓存。以下各节将更详细地介绍这些内存空间。

##### 2.2.7.1. 异构系统中的 DRAM 内存 {#2271-异构系统中的dram内存}

PPU 和 CPU 都配有直连的 DRAM 芯片。在拥有多个 PPU 的系统中，每个 PPU 都有自己的内存。从设备代码的角度来看，连接到 PPU 的 DRAM 被称为全局内存 (global memory)，因为它可以被 PPU 中的所有 CU 访问。这个术语并不意味着它一定能在整个系统中都能访问。连接到 CPU 的 DRAM 被称为系统内存 (system memory) 或宿主内存 (host memory)。

与 CPU 类似， PPU 使用虚拟内存寻址。在所有当前支持的系统中，CPU 和 PPU 使用统一的虚拟内存空间（virtual memory space）。这意味着系统中每个 PPU 的虚拟内存地址范围都是唯一且与其他 CPU 及其他 PPU 不同的。对于给定的虚拟内存地址，可以确定该地址位于 PPU 内存还是系统内存中，以及在多 PPU 系统中哪个 PPU 内存包含了该地址。

HGGC API 可以用于分配 PPU 内存、CPU 内存，以及在 CPU 和 PPU 之间、 PPU 内部或者多 PPU 系统的 PPU 之间复制内存。可以根据需要显式控制数据的位置。下面讨论的统一内存 (Unified Memory) 允许自动处理内存放置，由 HGGC 运行时或系统硬件自动管理。

##### 2.2.7.2. PPU 中的片上内存 {#2272-ppu中的片上内存}

除了全局内存之外，每个 PPU 还有一些片上内存。每个 CU 都有自己的寄存器文件 (register file) 和共享内存 (shared memory)。这些内存是 CU 的一部分，可以从在 CU 内执行的线程非常快速地访问，但无法被在其他 CU 中运行的线程访问。

寄存器文件存储线程局部变量，这些变量通常由编译器分配。共享内存可供线程块的所有线程访问。共享内存可用于在线程块的线程之间交换数据。

CU 中的寄存器文件和数据缓存大小有限。CU 的寄存器文件大小、统一数据缓存以及共享内存大小可在 [计算能力](#55-计算能力) 一节中找到。寄存器文件、共享内存空间和 L1 缓存由线程块中的所有线程共享。

要将线程块调度到 CU，每个线程所需的寄存器总数乘以线程块中的线程数必须小于或等于 CU 中的可用寄存器数。如果线程块所需的寄存器数超过了寄存器文件的大小，则核函数无法启动，必须减少线程块中的线程数才能使其能够启动。

共享内存分配是以线程块级别进行的。即与按线程分配的寄存器不同，共享内存的分配是整个线程块共有的。

###### 2.2.7.2.1. 缓存 {#22721-缓存}

除了可编程内存， PPU 还有 L1 和 L2、LLC 缓存。 PPU 上的数据缓存分为多层，离寄存器文件最近的是 CU 独有的 L1 缓存。较大的 L2 缓存由 CE(Compute Engine, CU 的上一层硬件结构) 内的所有 CU 共享。L2 之下还有一层 LLC 缓存，负责 DDR 通道的 global memory 和 L2 之间进行缓存数据，进一步缓解各层带宽的速度不匹配问题。

##### 2.2.7.3. 统一内存 {#2273-统一内存}

当应用程序明确在 PPU 或 CPU 上分配内存时，该内存只能被运行在该设备上的代码访问。也就是说，CPU 内存只能从 CPU 代码访问， PPU 内存只能从在 PPU 上运行的核函数访问。在 CPU 和 PPU 之间复制内存的 HGGC API 用于在适当的时间显式地将数据复制到正确的内存中。

一种名为统一内存 (unified memory) 的 HGGC 特性允许应用程序进行内存分配，以便可以从 CPU 或 PPU 访问。HGGC 运行时或底层硬件启用访问或在需要时重新定位数据。即使有了统一内存，最优性能仍然是通过最小化内存迁移并尽可能多地从直接连接到内存所在的处理器访问数据来实现的。

系统的硬件特性决定了内存空间之间数据访问和交换的实现方式。统一内存部分介绍了统一内存系统的不同类别。统一内存部分包含了更多关于在各种情况下的统一内存使用和行为的细节。


### 2.3. 异步编程模式 {#23-异步编程模式}

#### 2.3.1. 异步执行基础 {#231-异步执行基础}


HGGC 允许并发（或重叠）执行多个任务，具体包括：

- 主机上的计算
- 设备上的计算
- 从主机到设备的内存传输
- 从设备到主机的内存传输
- 在给定设备内存内部的内存传输
- 设备之间的内存传输

并发通过异步接口表达，其中调度函数调用或核函数启动会立即返回。异步调用通常在所调度的操作完成之前就返回了，甚至可能在异步操作开始之前就返回了。然后应用程序可以自由地执行其他任务，同时执行最初调度的操作。当需要初始调度操作的最终结果时，应用程序必须执行某种形式的同步，以确保相关操作已完成。一个典型的并发执行模式是将主机和设备内存传输与计算重叠，从而减少或消除其开销。

异步接口提供三种主要的同步方式：

- **阻塞方式**：应用程序调用一个阻塞（或等待）直到操作完成的函数
- **非阻塞方式**: 也称为轮询方式：应用程序调用一个立即返回并提供操作状态信息的函数
- **回调方式**：当操作完成时执行预注册的函数

虽然编程接口是异步的，但实际执行各种并发操作的能力将取决于 HGGC 的版本和硬件的计算能力，这些细节将在本指南的后续章节中介绍（参见 [计算能力](#55-计算能力)）。

在 [同步 CPU 和 PPU ](#214-同步-cpu-与-ppu) 中，我们介绍了 HGGC 运行时函数 `hggcDeviceSynchronize()`，这是一个阻塞调用，会等待所有先前发出的工作完成。需要 `hggcDeviceSynchronize()` 调用的原因是因为核函数启动是异步的，会立即返回。HGGC 为同步提供了阻塞和非阻塞两种方式的 API，甚至支持使用主机端回调函数。

HGGC 中异步执行的核心 API 组件是 **HGGC 流**和 **HGGC 事件**。
在本节的剩余部分，我们将解释如何使用这些元素来表达 HGGC 中的异步执行。

一个相关的主题是 **HGGC 图**，它允许预先定义异步操作图，然后可以以最小的开销重复执行。我们将在 [使用流捕获介绍 HGGC 图](#41-hggc-图) 中以入门级别介绍 HGGC 图，并在 [HGGC 图](#41-hggc-图) 更全面的讨论。

#### 2.3.2. 流与事件 {#232-流与事件}

##### 2.3.2.1. 流的创建与基本使用 {#2321-流的创建与基本使用}


在最基本层面上，HGGC 流是一种允许程序员表达操作序列的抽象。流就像一个工作队列，程序可以向其中添加操作（如内存复制或核函数启动），并按顺序执行。给定流的队列前端的操作执行后出队，允许下一个排队的操作来到前端并被考虑执行。流中操作的执行顺序是顺序的，操作按其入队的顺序执行。

应用程序可以同时使用多个流。在这种情况下，运行时将根据 PPU 资源的状态从有可用工作的流中选择要执行的任务。流可以被分配优先级，作为提示来影响运行时调度，但不能保证特定的执行顺序。

在流中操作的 API 函数调用和核函数启动相对于主机线程是异步的。应用程序可以等待流中的任务全部完成来与流同步，也可以在设备级别进行同步。

HGGC 有一个默认流，没有指定特定流的操作和核函数启动会被加入这个默认流。不指定流的代码示例隐式地使用这个默认流。默认流有一些特定的语义，将在"阻塞和非阻塞流与默认流"小节中讨论。


HGGC 流可以使用 `hggcStreamCreate()` 函数创建。该函数调用初始化流句柄，后续函数调用可以使用该句柄来标识流。

```cpp
// 创建流、使用流、销毁流的基本流程
hggcStream_t stream;
hggcStreamCreate(&stream);

// 在流上提交操作（核函数启动、内存传输等）
// ...

// 流使用完毕后销毁
hggcStreamDestroy(stream);
```


如果在应用程序调用 `hggcStreamDestroy()` 时设备仍在流 `stream` 中执行工作，流将在被销毁之前完成流中的所有工作。


启动核函数通常使用的三尖括号语法也可以用于将核函数启动到特定流中。流作为核函数启动的额外参数指定。在下面的示例中，名为 `kernel` 的核函数被启动到句柄为 `stream`（类型为 `hggcStream_t`，假设已预先创建）的流中：

```cpp
kernel<<<grid, block, shared_mem_size, stream>>>(...);
```


核函数启动是异步的，函数调用立即返回。假设核函数启动成功，核函数将在流 `stream` 中执行，应用程序可以自由地在核函数执行期间在 CPU 上或其他流中执行其他任务。


要将内存传输启动到流中，我们可以使用 `hggcMemcpyAsync()` 函数。该函数类似于 `hggcMemcpy()` 函数，但需要一个额外参数来指定用于内存传输的流。以下代码块中的函数调用在流 `stream` 中将 `size` 字节从 `src` 指向的主机内存复制到 `dst` 指向的设备内存。

```cpp
// PPU 异步数据搬运：使用 pinned memory 配合 AIU 实现传输与计算重叠
float *src;
hggcMallocHost(&src, size);  // 分配 pinned memory（页锁定主机内存）

float *dst;
hggcMalloc(&dst, size);       // 分配设备内存

// AIU 异步传输：在流 stream 中执行，与后续核函数可并行
hggcMemcpyAsync(dst, src, size, hggcMemcpyHostToDevice, stream);
```


与其他异步函数调用一样，此函数调用立即返回，而 `hggcMemcpy()` 函数会阻塞直到内存传输完成。为了安全访问传输的结果，应用程序必须使用某种形式的同步来确定操作已完成。

其他 HGGC 内存传输函数（如 `hggcMemcpy2D()`）也有异步变体。

!!! note

    为了使涉及 CPU 内存的内存复制能够异步执行，主机缓冲区必须是固定的且被页锁定的。如果使用未固定和未页锁定的主机内存，`hggcMemcpyAsync()` 将正常工作，但它会恢复为同步行为，不会与其他工作重叠。这会抑制使用异步内存传输的性能优势。建议程序使用 `hggcMallocHost()` 分配将用于向 PPU 发送或从 PPU 接收数据的缓冲区。

##### 2.3.2.2. 事件与依赖跟踪 {#2322-事件与依赖跟踪}


HGGC 事件是一种向 HGGC 流中插入标记的机制。它们本质上是示踪粒子，可用于跟踪流中任务的进度。想象一下，向一个流中启动两个核函数。如果没有这样的跟踪事件，我们只能确定流是否为空。如果我们有一个依赖于第一个核函数输出的操作，在我们知道流已为空之前，我们无法安全地启动该操作，到那时两个核函数都已完成。

使用 HGGC 事件我们可以做得更好。通过在第一个核函数之后、第二个核函数之前将一个事件入队到流中，我们可以等待这个事件到达流的前端。然后，我们可以安全地启动依赖操作，因为我们知道第一个核函数已完成，但第二个核函数尚未开始。以这种方式使用 HGGC 事件可以构建操作和流之间的依赖图。这个图类比直接转化到后面关于 [HGGC 图](#235-从流到图hggc-图简介) 的讨论中。

HGGC 流还保留时间信息，可用于计时核函数启动和内存传输。


HGGC 事件可以使用 `hggcEventCreate()` 和 `hggcEventDestroy()` 函数创建和销毁。

```cpp
hggcEvent_t inferenceEvent;

// 创建事件
hggcEventCreate(&inferenceEvent);

// 使用事件跟踪推理流水线进度

// 事件不再需要时销毁
hggcEventDestroy(inferenceEvent);
```

应用程序负责在不再需要事件时销毁它们。


可以使用 `hggcEventRecord(hggcEvent_t event, hggcStream_t stream)` 函数将事件记录到指定流中，其中第一个参数为要记录的事件对象，第二个参数为目标流。

```cpp
hggcEvent_t stageComplete;
hggcStream_t stream;

// 创建事件
hggcEventCreate(&stageComplete);

// 将事件插入流中，标记当前阶段完成
hggcEventRecord(stageComplete, stream);
```



HGGC 事件可用于对包括核函数在内的各种流操作进行计时。当事件到达流的前端时，它会记录一个时间戳。通过在流中用两个事件包围目标操作，可以获得精确的执行耗时，如下面的代码片段所示：

```cpp
// 使用事件计时测量 PPU 核函数与数据传输的执行耗时
// 假设 kernel 已定义为执行目标计算的 __global__ 函数
hggcStream_t stream;
hggcStreamCreate(&stream);

hggcEvent_t kernelStart, kernelEnd;
hggcEvent_t memcpyStart, memcpyEnd;

// 创建计时事件
hggcEventCreate(&kernelStart);
hggcEventCreate(&kernelEnd);
hggcEventCreate(&memcpyStart);
hggcEventCreate(&memcpyEnd);

// 阶段1：测量数据传输耗时
hggcEventRecord(memcpyStart, stream);
hggcMemcpyAsync(d_input, h_input, dataSize, hggcMemcpyHostToDevice, stream);
hggcEventRecord(memcpyEnd, stream);

// 阶段2：测量核函数执行耗时
hggcEventRecord(kernelStart, stream);
kernel<<<grid, block, 0, stream>>>(d_input, d_output, numElements);
hggcEventRecord(kernelEnd, stream);

// 同步并获取计时结果
hggcStreamSynchronize(stream);

float memcpyTime, kernelTime;
hggcEventElapsedTime(&memcpyTime, memcpyStart, memcpyEnd);
hggcEventElapsedTime(&kernelTime, kernelStart, kernelEnd);

// 清理
hggcEventDestroy(kernelStart);
hggcEventDestroy(kernelEnd);
hggcEventDestroy(memcpyStart);
hggcEventDestroy(memcpyEnd);
hggcStreamDestroy(stream);
```



与检查流状态的情况一样，我们可以以阻塞或非阻塞方式检查事件的状态。

`hggcEventSynchronize()` 函数将阻塞直到事件完成。在下面的代码片段中，我们向一个流中启动一个核函数，然后是一个事件，然后是第二个核函数。我们可以使用 `hggcEventSynchronize()` 函数等待第一个核函数之后的事件完成，原则上可以立即启动依赖任务，可能在 `kernel2` 完成之前。

```cpp
// PPU 推理流水线中使用事件同步
hggcEvent_t inferEvent;
hggcStream_t pipeline;

// 创建流和事件
hggcStreamCreate(&pipeline);
hggcEventCreate(&inferEvent);

// 向流中启动推理前处理核函数
preprocess_kernel<<<grid, block, 0, pipeline>>>(d_input, d_prepared);

// 在第一个核函数之后插入事件
hggcEventRecord(inferEvent, pipeline);

// 向流中启动推理核函数
inference_kernel<<<grid, block, 0, pipeline>>>(d_prepared, d_output);

// 等待事件完成——此时第一个核函数已完成
// 可以安全地启动依赖于预处理结果的 CPU 任务
hggcEventSynchronize(inferEvent);
dependentCPUtask();

// 等待流中所有工作完成
hggcStreamSynchronize(pipeline);

// 清理
hggcEventDestroy(inferEvent);
hggcStreamDestroy(pipeline);
```

HGGC 事件可以使用 `hggcEventQuery()` 函数以非阻塞方式检查完成状态。在下面的示例中，我们向一个流中启动 2 个核函数。第一个核函数 `kernel1` 生成一些数据，我们希望将其复制到主机，但我们也有一些 CPU 端的工作要做。在下面的代码中，我们在流 `stream1` 中入队 `kernel1`，然后是一个事件（`event`），然后是 `kernel2`。然后我们进入 CPU 工作循环，但偶尔查看事件是否已完成，表示 `kernel1` 已完成。如果是这样，我们在流 `stream2` 中启动主机到设备的复制。这种方法允许 CPU 工作与 PPU 核函数执行以及设备到主机复制重叠。

```cpp
// PPU 训练场景：CPU 端数据预处理与 PPU 计算重叠
hggcEvent_t computeDone;
hggcStream_t computeStream, transferStream;

hggcStreamCreate(&computeStream);
hggcStreamCreate(&transferStream);
hggcEventCreate(&computeDone);

float *d_batchA, *d_batchB, *d_grad;
float *h_nextBatch;
hggcMalloc(&d_batchA, batchSize);
hggcMalloc(&d_batchB, batchSize);
hggcMalloc(&d_grad, gradSize);
hggcMallocHost(&h_nextBatch, batchSize);

bool transferStarted = false;

// 在 computeStream 中启动当前批次的前向+反向传播
forward_kernel<<<grid, block, 0, computeStream>>>(d_batchA, d_grad);
backward_kernel<<<grid, block, 0, computeStream>>>(d_grad, d_batchA);
hggcEventRecord(computeDone, computeStream);

// CPU 端：准备下一批数据（数据增强、归一化等）
// 同时轮询 PPU 计算是否完成
while (!allPreprocessingDone()) {
    preprocessNextSample(h_nextBatch);

    // 非阻塞检查 PPU 计算是否完成
    if (!transferStarted && hggcEventQuery(computeDone) == hggcSuccess) {
        // PPU 计算完成，立即启动下一批数据传输
        hggcMemcpyAsync(d_batchB, h_nextBatch, batchSize,
                        hggcMemcpyHostToDevice, transferStream);
        transferStarted = true;
    }
}

// 确保传输完成
if (!transferStarted) {
    hggcEventSynchronize(computeDone);
    hggcMemcpyAsync(d_batchB, h_nextBatch, batchSize,
                    hggcMemcpyHostToDevice, transferStream);
}
hggcStreamSynchronize(transferStream);
hggcStreamSynchronize(computeStream);

hggcEventDestroy(computeDone);
hggcStreamDestroy(computeStream);
hggcStreamDestroy(transferStream);
hggcFree(d_batchA);
hggcFree(d_batchB);
hggcFree(d_grad);
hggcFreeHost(h_nextBatch);
```


##### 2.3.2.3. 流/事件的同步策略 {#2323-流事件的同步策略}


与流同步的最简单方法是等待流中的任务全部完成。这可以通过两种方式完成：使用 `hggcStreamSynchronize()` 函数或 `hggcStreamQuery()` 函数。

`hggcStreamSynchronize()` 函数将阻塞直到流中的所有工作完成。

```cpp
// 等待流中的所有操作完成
hggcStreamSynchronize(stream);

// 此时流已完成
// 可以安全地访问流操作的结果
```


如果我们不想阻塞，而只是想快速检查流是否为空，可以使用 `hggcStreamQuery()` 函数。

```cpp
// 非阻塞查询流状态
// 如果流为空则返回 hggcSuccess
// 如果流不为空则返回 hggcErrorNotReady
hggcError_t status = hggcStreamQuery(stream);

switch (status) {
    case hggcSuccess:
        // 流中所有操作已完成
        break;
    case hggcErrorNotReady:
        // 仍有操作在执行
        break;
    default:
        // 发生错误——应进行错误处理
        break;
};
```

**显式同步**{#显式同步}

HGGC 提供了多个粒度不同的 API 用于显式同步，开发者可根据需要选择合适的同步范围：

| API                       | 同步范围             | 阻塞行为                   | 典型用途                                                                                               |
| ------------------------- | -------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------ |
| `hggcDeviceSynchronize()` | 所有主机线程的所有流 | 阻塞至全部完成             | 全局同步，通常用于调试或程序结束前                                                                     |
| `hggcStreamSynchronize()` | 指定的单个流         | 阻塞至该流完成             | 仅等待特定流完成，不影响其他流继续执行                                                                 |
| `hggcStreamWaitEvent()`   | 跨流依赖             | 非阻塞（向流中插入等待点） | 建立流间执行顺序——使目标流后续命令等待指定事件完成（事件相关说明见 [HGGC 事件](#2322-事件与依赖跟踪)） |
| `hggcStreamQuery()`       | 指定的单个流         | 非阻塞（立即返回状态）     | 轮询检查流中所有先前命令是否已完成                                                                     |

**隐式同步**{#隐式同步}

除显式同步外，NULL 流（默认流）上的操作还会引入隐式的序列化约束。具体而言：当两个分属不同流的操作之间插入了一次 NULL 流上的提交，这两个操作将无法并发执行。此约束不适用于以 `hggcStreamNonBlocking` 标志创建的非阻塞流——非阻塞流不受 NULL 流的序列化影响。

为最大化核函数的并发执行潜力，建议在实践中注意以下两点：

- 优先提交不存在数据依赖的操作，将有依赖关系的操作排在后面
- 将同步点尽量推迟到真正需要结果的时刻

**阻塞和非阻塞流与默认流**{#阻塞和非阻塞流与默认流}

HGGC 中的流按其与默认流（NULL 流）的同步关系分为两类：

| 流类型   | 创建方式                                               | 与默认流的关系                                                 |
| -------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| 阻塞流   | `hggcStreamCreate()`                                   | 与默认流互相等待——默认流上的操作会等待所有阻塞流排空，反之亦然 |
| 非阻塞流 | `hggcStreamCreateWithFlags(&s, hggcStreamNonBlocking)` | 与默认流互不干扰，各自独立调度                                 |

"阻塞"一词容易引起歧义——它并不意味着流本身是同步的，而仅描述该流是否参与默认流的全局屏障行为。两种流的销毁方式相同，均使用 `hggcStreamDestroy()`。

**传统默认流**{#传统默认流}

当未显式指定流时（如 `kernel<<<grid, block>>>(...)` 或阻塞式 `hggcMemcpy()`），HGGC 会将操作提交到**传统默认流**（NULL 流，流 ID 为 0）。该流在进程内所有主机线程间共享，并具有全局屏障语义：任何提交到默认流的操作都会隐式等待所有阻塞流中先前的操作完成，同时阻塞流中后续提交的操作也会等待默认流上的操作完成。

下面的示例展示了这一串行化效应： {#legacy-default-stream-example}

```cpp
hggcStream_t work_stream_a, work_stream_b;
hggcStreamCreate(&work_stream_a);  // 默认为阻塞流
hggcStreamCreate(&work_stream_b);

kernel1<<<grid, block, 0, work_stream_a>>>(...);
kernel2<<<grid, block>>>(...);  // 提交到默认流
kernel3<<<grid, block, 0, work_stream_b>>>(...);

hggcDeviceSynchronize();
```

由于 `kernel2` 使用默认流，它会等待 `kernel1`（阻塞流上的操作）完成；而 `kernel3` 又必须等待默认流上的 `kernel2` 完成。最终三个核函数被迫串行执行，尽管它们之间并无数据依赖。

将 `work_stream_a` 和 `work_stream_b` 改为非阻塞流即可消除这种不必要的序列化——三个核函数可自由并发。代价是开发者需要通过事件或显式同步 API 来保证正确的执行顺序。 {#non-blocking-stream-example}

```cpp
hggcStream_t work_stream_a, work_stream_b;
hggcStreamCreateWithFlags(&work_stream_a, hggcStreamNonBlocking);
hggcStreamCreateWithFlags(&work_stream_b, hggcStreamNonBlocking);

kernel1<<<grid, block, 0, work_stream_a>>>(...);
kernel2<<<grid, block>>>(...);
kernel3<<<grid, block, 0, work_stream_b>>>(...);

hggcDeviceSynchronize();
```

**每线程默认流**{#每线程默认流}

传统默认流是全局共享的，当多线程程序的不同主机线程同时向默认流提交工作时，会因序列化而降低并发度。HGGC 提供了"每线程默认流"模式来解决这一问题：在此模式下，每个主机线程拥有各自独立的默认流，线程之间不再因共享默认流而相互阻塞。

启用方式（二选一）：

- 编译选项：`hgcc --default-stream per-thread`
- 预处理器宏：在包含 HGGC 头文件之前定义 `HGGC_API_PER_THREAD_DEFAULT_STREAM`

启用后，[传统默认流示例](#传统默认流) 将表现出与 [非阻塞流示例](#non-blocking-stream-example) 相同的同步行为。

**HGGC 流顺序**{#hggc-流顺序}

现在我们已经讨论了流、事件和回调函数的基本机制，考虑异步操作在流中的顺序语义很重要。这些语义是为了让应用程序程序员能够安全地考虑流中操作的顺序。在某些特殊情况下，这些语义可能会放宽以实现性能优化，例如在编程式依赖核函数启动场景中，该场景允许通过特殊属性和核函数启动机制重叠两个核函数。我们稍后会在需要时讨论这些优化。

最重要的是，HGGC 流是所谓的**有序流**。这意味着流中操作的执行顺序与这些操作入队的顺序相同。流中的操作不能跳过其他操作。内存操作（如复制）由运行时跟踪，并且始终会在顺序上下一个操作之前完成，以允许依赖的核函数安全访问正在传输的数据。

#### 2.3.3. 来自流的回调函数 {#233-来自流的回调函数}

HGGC 提供了一种机制，可以从流内启动主机上的函数。目前有两个函数可用于此目的：`hggcLaunchHostFunc()` 和 `hggcAddCallback()`。但是，`hggcAddCallback()` 计划弃用，因此应用程序应该使用 `hggcLaunchHostFunc()`。

##### 2.3.3.1. 使用 hggcLaunchHostFunc() {#2331-使用-hggclaunchhostfunc}

`hggcLaunchHostFunc()` 函数的签名如下：

```cpp
hggcError_t hggcLaunchHostFunc(hggcStream_t stream, void (*func)(void *), void *data);
```

其中：

- `stream`：要将回调函数启动到的流
- `func`：要启动的回调函数
- `data`：要传递给回调函数的数据指针

主机函数本身是一个简单的 C 函数，签名为：

```cpp
void hostFunction(void *data);
```

其中 `data` 参数指向用户定义的数据结构，函数可以解释它。使用这样的回调函数有一些注意事项。特别是，主机函数不能调用任何 HGGC API。

在统一内存场景下，主机函数的执行遵循以下语义保证：

| 保证项                                               | 含义                                                                                               | 实际影响                             |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------ |
| 函数执行期间流视为空闲                               | 运行时不会在函数执行时向同一流提交新的设备工作                                                     | 函数内可安全访问绑定到该流的托管内存 |
| 函数启动等效于一次事件记录                           | 函数开始执行时，所有在此之前"加入"该流的其他流均已完成                                             | 可依赖函数启动点作为跨流同步屏障     |
| 设备工作需等待所有先前主机函数和流回调完成后才激活流 | 即使新的设备工作被排入其他流，只要通过事件排序在主机函数和流回调之后，流仍不会进入活动状态         | 函数执行期间全局附加内存仍可安全使用 |
| 函数完成本身不激活流                                 | 如果后续没有设备工作跟随，流将保持空闲；如果在连续的主机函数或流回调之间没有设备工作，流将保持空闲 | 可在流末尾通过主机函数发出同步信号   |

##### 2.3.3.2. 使用 hggcStreamAddCallback() {#2332-使用-hggcstreamaddcallback}

!!! note

    `hggcStreamAddCallback()` 函数计划弃用和移除，在此讨论是为了完整起见，因为它可能仍出现在现有代码中。应用程序应该使用或切换到使用 `hggcLaunchHostFunc()`。

`hggcStreamAddCallback()` 函数的签名如下：

```cpp
hggcError_t hggcStreamAddCallback(hggcStream_t stream, hggcStreamCallback_t callback,
                                   void* userData, unsigned int flags);
```

其中：

- `stream`：要将回调函数启动到的流
- `callback`：要启动的回调函数
- `userData`：要传递给回调函数的数据指针
- `flags`：目前，此参数必须为 0 以保持未来兼容性

回调函数的签名与我们使用 `hggcLaunchHostFunc()` 函数的情况略有不同。在这种情况下，回调函数是一个 C 函数，签名为：

```cpp
void callbackFunction(hggcStream_t stream, hggcError_t status, void *userData);
```

函数现在接收：

- `stream`：从中启动回调函数的流句柄
- `status`：触发回调的流操作的状态
- `userData`：传递给回调函数的数据指针

特别地，`status` 参数将包含流的当前错误状态，可能是由先前的操作设置的。
与 `hggcLaunchHostFunc()` 的情况类似，在主机函数完成之前，流不会处于活动状态并推进到任务，并且不能从回调函数内调用任何 HGGC 函数。

##### 2.3.3.3. 异步错误处理 {#2333-异步错误处理}

在 HGGC 流中，错误可能来自流中的任何操作，包括核函数启动和内存传输。这些错误可能在运行时不会传播回开发者，直到流被同步（例如，通过等待事件或调用 `hggcStreamSynchronize()`）。有两种方法可以发现流中可能发生的错误。

- 使用函数 `hggcGetLastError()`—— 此函数返回并清除当前上下文中任何流中遇到的最后一个错误。如果在两次调用之间没有发生其他错误，立即第二次调用 `hggcGetLastError()` 将返回 `hggcSuccess`。
- 使用函数 `hggcPeekAtLastError()`—— 此函数返回当前上下文中的最后一个错误，但不会清除它。

这两个函数都返回一个类型为 `hggcError_t` 的错误值。可以使用函数 `hggcGetErrorName()` 和 `hggcGetErrorString()` 生成错误的人类可读名称。

使用这些函数的示例如下：

**清单 1. 使用 hggcGetLastError() 和 hggcPeekAtLastError() 的示例**{#清单1-hggcgetlasterror示例}

```cpp
// 一些工作在流中发生
hggcStreamSynchronize(stream);

// 查看最后一个错误但不清除它
hggcError_t err = hggcPeekAtLastError();
if (err != hggcSuccess) {
    printf("Error with name: %s\n", hggcGetErrorName(err));
    printf("Error description: %s\n", hggcGetErrorString(err));
}

// 查看最后一个错误并清除它
hggcError_t err2 = hggcGetLastError();
if (err2 != hggcSuccess) {
    printf("Error with name: %s\n", hggcGetErrorName(err2));
    printf("Error description: %s\n", hggcGetErrorString(err2));
}

if (err2 != err) {
    printf("As expected, hggcPeekAtLastError() did not clear the error\n");
}

// 再次检查
hggcError_t err3 = hggcGetLastError();
if (err3 == hggcSuccess) {
    printf("As expected, hggcGetLastError() cleared the error\n");
}
```

!!! note

    当在同步时出现错误时，特别是在有许多操作的流中，通常很难精确定位错误可能发生在流中的哪个位置。调试这种情况时，一个有用的技巧是设置环境变量 `HGGC_LAUNCH_BLOCKING=1`，然后运行应用程序。此环境变量的作用是在每个单个核函数启动后同步。这可以帮助追踪是哪个核函数或传输导致了错误。
    同步可能很昂贵；设置此环境变量后，应用程序可能会运行得慢很多。

#### 2.3.4. 高级流管理与核函数配置 {#234-高级流管理与核函数配置}


本节将介绍更高级的 HGGC API 和功能。这些主题涉及的技术或功能通常不需要修改 HGGC 核函数（Kernel），但仍能从宿主（Host）端影响应用程序层面的行为，包括 PPU 任务执行、性能以及 CPU 端的性能。

##### 2.3.4.1. hggcLaunchKernelEx {#2341-hggclaunchkernelex}

当 [三尖括号语法](#21221-三重尖括号表示法) 在 HGGC 的早期版本中引入时，核函数的 [核函数配置](#52-hggc-语言扩展) 仅有四个可编程参数：
- 线程块维度（Thread block dimensions）
- 网格维度（Grid dimensions）
- 动态共享内存（可选，未指定则为 0）
- 流（Stream，未指定则使用默认流）

某些 HGGC 功能可以受益于核函数启动时提供的额外属性和提示。`hggcLaunchKernelEx` 允许程序通过 `hggcLaunchConfig_t` 结构体设置上述执行配置参数。此外，`hggcLaunchConfig_t` 结构体还允许程序传入零个或多个 `hggcLaunchAttributes`，以控制或建议核函数启动的其他参数。

##### 2.3.4.2. 流（Streams）与事件（Events）进阶 {#2342-流streams与事件events进阶}

[HGGC 流](#23-异步编程模式) 介绍了 HGGC 流的基础知识。默认情况下，提交给特定 HGGC 流的操作是序列化的：前一个操作未完成前，下一个操作无法开始执行。拥有多个 HGGC 流是实现并发执行的一种方式；另一种方式是使用 [HGGC 图（Graphs）](#41-hggc-图)。这两种方法也可以结合使用。

在特定情况下，提交到不同 HGGC 流的任务可以并发执行，例如：不存在事件依赖关系、不存在隐式同步、拥有充足的资源等。

除非是"非阻塞（non-blocking）"HGGC 流，否则如果两个独立 HGGC 流的操作之间插入了对 NULL 流的 HGGC 操作，它们将无法并发运行。非阻塞流是通过运行时 API `hggcStreamCreateWithFlags()` 并设置 `hggcStreamNonBlocking` 标志创建的。为了提高 PPU 任务并发执行的可能性，建议用户创建非阻塞 HGGC 流。

同时建议用户为解决问题选择粒度最细的同步选项。例如，如果需求是让 CPU 等待（阻塞）特定 HGGC 流上的所有任务完成，使用 `hggcStreamSynchronize()` 优于 `hggcDeviceSynchronize()`，因为后者会不必要地等待设备上所有 HGGC 流的 PPU 任务完成。如果需求是让 CPU 在不阻塞的情况下等待，那么在轮询循环中使用 `hggcStreamQuery()` 并检查其返回值可能更合适。

类似的同步效果也可以通过 HGGC 事件（[HGGC Events](#23-异步编程模式)）实现，例如：在流上记录一个事件，并调用 `hggcEventSynchronize()` 以阻塞方式等待该事件所捕获的任务完成。这同样比使用 `hggcDeviceSynchronize()` 更可取且目标更明确。调用 `hggcEventQuery()` 并检查返回值（如在轮询循环中）则是一种非阻塞的替代方案。

如果该操作处于应用程序的关键路径中，选择明确的同步方法尤为重要。表 1 总结了各种宿主同步选项：

*表 1：宿主显式同步选项总结*

|                             | 等待特定流                | 等待特定事件             | 等待设备上所有任务        |
| :-------------------------- | :------------------------ | :----------------------- | :------------------------ |
| **非阻塞** (需配合轮询循环) | `hggcStreamQuery()`       | `hggcEventQuery()`       | 不适用                    |
| **阻塞**                    | `hggcStreamSynchronize()` | `hggcEventSynchronize()` | `hggcDeviceSynchronize()` |

对于 HGGC 流之间的同步（即表达依赖关系），建议使用非计时类（non-timing）的 HGGC 事件，如 [HGGC Events](#23-异步编程模式) 所述。开发者可以调用 `hggcStreamWaitEvent()` 来强制特定流上后续提交的操作等待先前记录的事件（例如在另一个流上的事件）完成。注意，对于任何等待或查询事件的 HGGC API，开发者有责任确保已调用 `hggcEventRecord` API，因为未记录的事件总是会返回成功。

HGGC 事件默认携带计时信息，可用于 `hggcEventElapsedTime()` API 调用。然而，仅用于跨流表达依赖关系的 HGGC 事件不需要计时信息。对于此类情况，建议在创建事件时禁用计时功能以提升性能，这可以通过使用 `hggcEventCreateWithFlags()` API 并设置 `hggcEventDisableTiming` 标志来实现。

###### 2.3.4.2.1. 流优先级 {#23421-流优先级}

PPU 采用基于队列的派发模型：每条 HGGC 流维护一条独立的任务队列，调度器在执行单元出现空闲时，从所有就绪流中挑选下一个待派发的任务。流优先级正是作用于这一**任务挑选**环节——它向调度器提供一个排序提示，用于在多条流的就绪任务之间打破平局，但并不改变 PPU 的派发与执行机制本身。

基于这一调度模型，流优先级具有以下行为约束：

*   **仅在派发时刻生效**。当调度器需要选取下一个待派发任务时，高优先级流中处于就绪状态的任务优先于低优先级流中的任务进入执行单元。
*   **不具备抢占语义**。任务一旦被派发到执行单元上运行，将持续执行至完成；即便此后有更高优先级的任务进入就绪队列，也不会中断当前正在执行的工作。
*   **运行期不重估**。 PPU 在任务执行过程中不会重新审视各队列的状态。运行中调用 API 提升某条流的优先级，对已在执行的任务没有任何影响，仅会改变后续派发的选择顺序。
* **属于调度提示而非时序契约**。优先级表达的是"倾向"而非"承诺"，不应作为构造任务依赖的手段；如需严格的执行顺序，应通过 HGGC 事件、`hggcStreamWaitEvent()` 等机制显式建立依赖。

在 API 层面，流的优先级在创建时通过 `hggcStreamCreateWithPriority()` 传入一个整数值指定，其有效区间需先通过 `hggcDeviceGetStreamPriorityRange()` 查询。该函数以 `[最低优先级, 最高优先级]` 的顺序返回端点，整数值越小代表优先级越高；不同设备所支持的优先级层级数可能不同，超出范围的取值会被钳制到端点。

下例先在当前设备上获取允许的优先级区间，再据此创建一对非阻塞流——其一承载延迟敏感的关键路径任务，其二承载可被延后的吞吐型后台任务：

```cpp
// 查询当前设备支持的流优先级区间，约定：数值越小，优先级越高
int lowestPrio = 0;
int highestPrio = 0;
hggcDeviceGetStreamPriorityRange(&lowestPrio, &highestPrio);

// 关键路径流：使用最高优先级，承载对延迟敏感的任务
hggcStream_t criticalStream;
hggcStreamCreateWithPriority(&criticalStream,
                             hggcStreamNonBlocking,
                             highestPrio);

// 后台流：使用最低优先级，承载可被延后的吞吐型任务
hggcStream_t backgroundStream;
hggcStreamCreateWithPriority(&backgroundStream,
                             hggcStreamNonBlocking,
                             lowestPrio);
```

###### 2.3.4.2.2. 显式同步 {#23422-显式同步}

显式同步 API 的详细说明（含同步范围、阻塞行为和典型用途对照表）参见 [第 2.3.2.3 节 流/事件的同步策略](#2323-流事件的同步策略)。

###### 2.3.4.2.3. 隐式同步 {#23423-隐式同步}

如果主机线程在两个不同流的命令之间执行了以下任何操作，则它们无法并发运行：

*   页锁定（page-locked）主机内存分配
*   设备内存分配
*   设备内存设置（memory set）
*   同一设备内存中两个地址之间的内存拷贝
*   任何发送到 NULL 流的 HGGC 命令
*   L1/共享内存配置的切换

需要进行依赖检查的操作包括：正在检查的启动命令所在流中的任何其他命令，以及对该流的任何 `hggcStreamQuery()` 调用。因此，应用程序应遵循以下准则以提高并发核函数执行的可能性：

*   所有独立操作应在依赖操作之前提交；
*   任何形式的同步都应尽可能推迟。

##### 2.3.4.3. 环境变量 {#2343-环境变量}

HGGC 提供了各种环境变量（请参阅 [环境变量](#53-hggc-环境变量)），这些变量会影响执行和性能。如果未明确设置，HGGC 会使用合理的默认值，但在某些情况下，例如为了调试目的或为了获得更好的性能，可能需要进行特殊处理。

例如，增加 `HGGC_DEVICE_MAX_CONNECTIONS` 环境变量的值对于减少来自不同 HGGC 流的独立任务因虚假依赖而被序列化的可能性可能是必要的。当使用相同的底层资源时，可能会引入此类虚假依赖。建议先使用默认值，仅在遇到性能问题时（例如，无法归因于其他因素（如可用 CU 资源不足）的跨 HGGC 流独立任务的意外序列化）才探讨该环境变量的影响。值得注意的是，在使用 MPS 的情况下，该环境变量具有不同（更低）的默认值。

类似地，对于延迟敏感的应用程序，将 `HGGC_MODULE_LOADING` 环境变量设置为 `EAGER` 可能更可取，以便将模块加载引起的所有开销转移到应用程序的初始化阶段，从而避开其关键执行阶段。当前的默认模式是延迟模块加载。在这种默认模式下，通过在应用程序的初始化阶段添加各种核函数的"预热"调用，可以达到与立即模块加载类似的效果，从而强制模块更早地加载。

请参阅 [HGGC 环境变量](#53-hggc-环境变量) 以获取有关各种 HGGC 环境变量的更多详细信息。建议你在启动应用程序**之前**将环境变量设置为新值；在应用程序内部设置它们可能不会产生任何效果。

#### 2.3.5. 从流到图——HGGC 图简介 {#235-从流到图hggc-图简介}


HGGC 流允许程序指定操作、核函数或内存复制的序列操作。使用多个流和通过 `hggcStreamWaitEvent` 的跨流依赖，应用程序可以指定一个完整的操作有向无环图（DAG）。某些应用程序可能有一系列或 DAG 操作，需要在执行过程中多次运行。

对于这种情况，HGGC 提供了一个称为 **HGGC 图**的功能。本节介绍 HGGC 图和一种称为**流捕获**的创建它们的机制。更详细的 HGGC 图讨论在 [HGGC 图](#41-hggc-图) 中提供。捕获或创建图可以帮助减少从主机线程重复调用相同 API 链的延迟和 CPU 开销。相反，指定图操作的 API 可以调用一次，然后生成的图可以执行多次。

HGGC 图的工作方式如下：

1. **图被应用程序捕获**。此步骤在第一次执行图时完成一次。也可以使用 HGGC 图 API 手动组合图。
2. **图被实例化**。此步骤在图被捕获后完成一次。此步骤可以设置执行图所需的所有各种运行时结构，以尽可能快地启动其组件。
3. **在剩余步骤中**，预实例化的图根据需要执行多次。由于执行图操作所需的所有运行时结构已就位，图执行的 CPU 开销被最小化。

**清单 2. 使用 HGGC 图捕获、实例化和执行简单线性图的阶段**

```cpp
// PPU 推理引擎：将推理流水线捕获为图以减少启动开销
#define INPUT_SIZE  (224 * 224 * 3)

__global__ void normalize_kernel(float *dst, const float *src, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) dst[idx] = src[idx] / 255.0f;
}

bool graphCaptured = false;
hggcGraph_t inferenceGraph;
hggcGraphExec_t inferenceExec;

for (int req = 0; req < totalRequests; req++) {
    if (!graphCaptured) {
        // 首次请求：捕获推理流水线为图
        hggcStreamBeginCapture(inferStream, hggcStreamCaptureModeGlobal);

        // 阶段1：输入归一化
        normalize_kernel<<<(INPUT_SIZE+255)/256, 256, 0, inferStream>>>(
            d_normalized, d_rawInput, INPUT_SIZE);

        // 阶段2：推理计算（各层核函数已在别处定义）
        layer1_kernel<<<grid1, block1, 0, inferStream>>>(d_normalized, d_hidden1);
        layer2_kernel<<<grid2, block2, 0, inferStream>>>(d_hidden1, d_hidden2);
        output_kernel<<<grid3, block3, 0, inferStream>>>(d_hidden2, d_output);

        hggcStreamEndCapture(inferStream, &inferenceGraph);
        hggcGraphInstantiate(&inferenceExec, inferenceGraph, NULL, NULL, 0);
        graphCaptured = true;
    }

    // 后续请求：直接启动预实例化的图
    hggcGraphLaunch(inferenceExec, inferStream);
    hggcStreamSynchronize(inferStream);
}
```


关于 HGGC 图的更多细节在 [HGGC 图](#41-hggc-图) 中提供。

#### 2.3.6. 异步编程小结 {#236-异步编程小结}


本节的关键点：

- 异步 API 允许我们表达任务的并发执行，提供表达各种操作重叠的方式。实际达到的并发度取决于可用的硬件资源和计算能力。
- HGGC 中异步执行的关键抽象是流、事件和回调函数。
- 同步可以在事件、流和设备级别进行。
- 默认流是阻塞流，与所有其他阻塞流同步，但不与非阻塞流同步。
- 可以通过使用 `--default-stream per-thread` 编译器选项或 `HGGC_API_PER_THREAD_DEFAULT_STREAM` 预处理器宏的每线程默认流来避免默认流行为。
- 流可以创建时带有不同的优先级，这是对运行时的提示，对于内存传输可能不会遵循。
- HGGC 提供 API 函数来减少或重叠核函数启动和内存传输的开销，如 HGGC 图、批量内存传输和编程式依赖核函数启动。


### 2.4. 统一和系统内存 {#24-统一和系统内存}

PPU 异构系统中存在多层物理内存——CPU 侧的主机 DRAM 和每个 PPU 设备自带的显存。数据在哪一侧的内存中，直接决定了访问延迟和带宽。前文介绍的 [显式内存管理](#2132-显式内存管理) 给予了开发者完整的控制权，但也带来了分配、拷贝、同步等大量样板代码。为此 HGGC 提供了统一内存和系统内存等自动化数据放置与迁移机制，力求在编程便利性与性能之间取得平衡。

本章将介绍 HGGC 在内存管理方面提供的各项功能及其对应用性能的影响。统一内存的具体行为取决于操作系统版本、HGGC 驱动版本以及 PPU 硬件能力，下文将说明如何判断当前环境所属的统一内存模式，以及各模式下的行为特征。

我们目前只支持有限统一内存。

#### 2.4.1. 统一虚拟地址空间 {#241-统一虚拟地址空间}

在单个操作系统进程中，单个虚拟地址空间用于所有主机内存和系统中所有 PPU 上的所有全局内存。主机和所有设备上的所有内存分配都位于这个虚拟地址空间中。无论分配是通过 HGGC API（如 hggcMalloc、hggcMallocHost）还是通过系统分配 API（如 new、malloc、mmap）进行的，都是如此。CPU 和每个 PPU 在统一虚拟地址空间中有唯一的范围。

统一虚拟地址空间为开发者带来两个关键便利：

1. **指针归属查询**：对于任意指针，调用 `hggcPointerGetAttributes()` 即可判断其背后的物理内存归属于 CPU 还是某个 PPU，无需开发者手动跟踪分配来源。
2. **自动推断拷贝方向**：在调用 `hggcMemcpy*()` 系列函数时，将 `hggcMemcpyKind` 设为 `hggcMemcpyDefault` 即可让运行时根据源和目标指针所在的地址范围自动选择正确的传输路径，省去显式指定方向的步骤。

#### 2.4.2. 统一内存 {#242-统一内存}

统一内存是一种 HGGC 内存功能，允许从在 CPU 或 PPU 上运行的代码访问称为托管内存的内存分配。统一内存在 [C++ 中的 HGGC 简介](#21-用-c-编写-hggc-程序) 中已介绍。统一内存在 HGGC 支持的所有系统上都可用。

在某些系统上，托管内存必须显式分配。在 HGGC 中可以通过几种不同的方式显式分配托管内存：

- HGGC API `hggcMallocManaged`
- HGGC API `hggcMallocFromPoolAsync`，其中池的 `allocType` 设置为 `hggcMemAllocationTypeManaged`
- 具有 `__managed__` 说明符的全局变量（参见 [内存空间说明符](#52-hggc-语言扩展)）


##### 2.4.2.1. 统一内存模式 {#2421-统一内存模式}

统一内存的功能和行为因操作系统、Linux 上的内核版本、 PPU 硬件和 PPU-CPU 互连而异。可以使用 `hggcDeviceGetAttribute` 查询几个属性来确定可用的统一内存形式：

- `hggcDevAttrConcurrentManagedAccess` - 1 表示完整统一内存支持，0 表示有限支持
- `hggcDevAttrPageableMemoryAccess` - 1 表示所有系统内存都是完全支持的统一内存，0 表示只有显式分配为托管内存的才是完全支持的统一内存
- `hggcDevAttrPageableMemoryAccessUsesHostPageTables` - 指示 CPU/PPU 一致性的机制：1 是硬件，0 是软件

##### 2.4.2.2. 完整统一内存功能支持 {#2422-完整统一内存功能支持}

大多数 Linux 系统具有完整的统一内存支持。如果设备属性 `hggcDevAttrPageableMemoryAccess` 为 1，则所有系统内存（无论是通过 HGGC API 还是系统 API 分配）都作为具有完全功能支持的统一内存运行。这包括使用 `mmap` 创建的文件支持的内存分配。

如果 `hggcDevAttrPageableMemoryAccess` 为 0，则只有通过 HGGC 分配为托管内存的内存才作为统一内存运行。通过系统 API 分配的内存不是托管的，不一定可以从 PPU 核函数访问。

对于具有完全支持的统一分配：

| 特征         | 说明                                                                 |
| ------------ | -------------------------------------------------------------------- |
| 首次访问放置 | 托管内存通常分配在首次访问它的处理器的内存空间中                     |
| 按需迁移     | 当托管内存被当前驻留所在处理器以外的处理器使用时，通常会发生迁移     |
| 迁移粒度     | 托管内存以内存页（软件一致性）或缓存行（硬件一致性）的粒度迁移或访问 |
| 超额订阅     | 允许超额订阅：应用程序可以分配比 PPU 物理可用更多的托管内存          |

分配和迁移行为可能与上述有所不同。这可以通过程序员使用"提示和预取"来影响。

##### 2.4.2.3. 有限统一内存支持 {#2423-有限统一内存支持}

有限模式下，统一内存的行为受以下约束：

| 约束       | 说明                                                      |
| ---------- | --------------------------------------------------------- |
| 初始放置   | 托管内存始终首先分配在 CPU 物理内存中                     |
| 迁移粒度   | 以大于虚拟内存页的块为单位批量迁移                        |
| 迁移时机   | PPU 核函数启动时整体迁入 PPU， PPU 同步完成后整体迁回 CPU |
| 独占访问   | PPU 执行期间 CPU 不得访问托管内存                         |
| 无超额订阅 | 托管内存总量不得超过 PPU 物理显存                         |
| 显式分配   | 仅通过 HGGC API 显式分配为托管内存的区域才具有统一语义    |

##### 2.4.2.4. 内存建议和预取 {#2424-内存建议和预取}

程序员可以向管理统一内存的驱动程序提供提示，以帮助其最大化应用程序性能。HGGC API `hggcMemAdvise` 允许程序员指定影响分配放置位置以及从另一个设备访问时内存是否迁移的属性。

`hggcMemPrefetchAsync` 允许程序员建议开始将特定分配异步迁移到不同位置。常见用法是在核函数启动之前启动核函数将使用的数据传输。这允许在其他 PPU 核函数执行时进行数据复制。

#### 2.4.3. 页锁定主机内存 {#243-页锁定主机内存}

在 [介绍性代码示例](#215-组合使用) 中，`hggcMallocHost` 用于在 CPU 上分配内存。这会在主机上分配页锁定内存（也称为固定内存）。通过传统分配机制（如 malloc、new 或 mmap）进行的主机分配不是页锁定的，这意味着它们可能被操作系统交换到磁盘或物理重新定位。

页锁定主机内存是 [CPU 和 PPU 之间的异步复制](#23-异步编程模式) 所必需的。页锁定主机内存也提高了同步复制的性能。页锁定内存可以映射到 PPU 以便从 PPU 核函数直接访问。

HGGC 运行时提供 API 来分配页锁定主机内存或对现有分配进行页锁定：

- `hggcMallocHost` 分配页锁定主机内存
- `hggcHostAlloc` 默认为与 `hggcMallocHost` 相同的行为，但也接受标志来指定其他内存参数
- `hggcFreeHost` 释放使用 `hggcMallocHost` 或 `hggcHostAlloc` 分配的内存
- `hggcHostRegister` 对通过 HGGC API 外部分配的范围进行页锁定，如使用 malloc 或 mmap

`hggcHostRegister` 使由第三方库或其他开发人员控制之外的代码分配的主机内存能够被页锁定，以便用于异步复制或映射。

!!! note

    页锁定主机内存可用于系统中所有 PPU 的异步复制和映射内存。


##### 2.4.3.1. 映射内存 {#2431-映射内存}

以下代码示例将说明以下数组复制核函数直接操作映射的主机内存。

```cpp
__global__ void copyKernel(float* a, float* b) {
    int idx = threadIdx.x + blockDim.x * blockIdx.x;
    a[idx] = b[idx];
}
```

虽然映射内存在某些情况下可能很有用，其中某些不会被复制到 PPU 的数据需要从核函数访问，但在核函数中访问映射内存需要通过 CPU-PPU 互连（PCIe 或 Icnlink C2C）进行事务处理。与访问设备内存相比，这些操作具有更高的延迟和更低的带宽。对于大多数核函数的内存需求，映射内存不应被视为统一内存或显式内存管理的高性能替代方案。

###### 2.4.3.1.1. hggcMallocHost 和 hggcHostAlloc {#24311-hggcmallochost-和-hggchostalloc}

使用 `hggcMallocHost` 或 `hggcHostAlloc` 分配的主机内存会自动映射。这些 API 返回的指针可以直接在核函数代码中使用以访问主机上的内存。主机内存通过 CPU-PPU 互连访问。

```cpp
// hggcMallocHost
void usingMallocHost() {
    float* a = nullptr;
    float* b = nullptr;

    HGGC_CHECK(hggcMallocHost(&a, vLen * sizeof(float)));
    HGGC_CHECK(hggcMallocHost(&b, vLen * sizeof(float)));

    initVector(b, vLen);
    memset(a, 0, vLen * sizeof(float));

    int threads = 256;
    int blocks = vLen / threads;
    copyKernel<<<blocks, threads>>>(a, b);
    HGGC_CHECK(hggcGetLastError());
    HGGC_CHECK(hggcDeviceSynchronize());

    printf("Using hggcMallocHost: ");
    checkAnswer(a, b);
}
```

```cpp
// hggcHostAlloc
void usinghggcHostAlloc() {
    float* a = nullptr;
    float* b = nullptr;

    HGGC_CHECK(hggcHostAlloc(&a, vLen * sizeof(float), hggcHostAllocMapped));
    HGGC_CHECK(hggcHostAlloc(&b, vLen * sizeof(float), hggcHostAllocMapped));

    initVector(b, vLen);
    memset(a, 0, vLen * sizeof(float));

    int threads = 256;
    int blocks = vLen / threads;
    copyKernel<<<blocks, threads>>>(a, b);
    HGGC_CHECK(hggcGetLastError());
    HGGC_CHECK(hggcDeviceSynchronize());

    printf("Using hggcHostAlloc: ");
    checkAnswer(a, b);
}
```

###### 2.4.3.1.2. hggcHostRegister {#24312-hggchostregister}

系统分配器进行的分配可以使用 `hggcHostRegister` 映射以便直接从 PPU 核函数访问。然而，与使用 HGGC API 创建的内存不同，内存不能使用主机指针从核函数访问。必须使用 `hggcHostGetDevicePointer()` 获得设备内存区域中的指针，并且必须使用该指针进行核函数代码中的访问。

```cpp
void usingRegister() {
    float* a = nullptr;
    float* b = nullptr;
    float* devA = nullptr;
    float* devB = nullptr;

    a = (float*)malloc(vLen * sizeof(float));
    b = (float*)malloc(vLen * sizeof(float));
    HGGC_CHECK(hggcHostRegister(a, vLen * sizeof(float), 0));
    HGGC_CHECK(hggcHostRegister(b, vLen * sizeof(float), 0));

    HGGC_CHECK(hggcHostGetDevicePointer((void**)&devA, (void*)a, 0));
    HGGC_CHECK(hggcHostGetDevicePointer((void**)&devB, (void*)b, 0));

    initVector(b, vLen);
    memset(a, 0, vLen * sizeof(float));

    int threads = 256;
    int blocks = vLen / threads;
    copyKernel<<<blocks, threads>>>(devA, devB);
    HGGC_CHECK(hggcGetLastError());
    HGGC_CHECK(hggcDeviceSynchronize());

    printf("Using hggcHostRegister: ");
    checkAnswer(a, b);
}
```

###### 2.4.3.1.3. 比较统一内存和映射内存 {#24313-比较统一内存和映射内存}

映射内存使 CPU 内存可从 PPU 访问，但不保证所有类型的访问（例如原子操作）在所有系统上都受支持。统一内存保证所有访问类型都受支持。

映射内存保留在 CPU 内存中，这意味着所有 PPU 访问都必须通过 CPU 和 PPU 之间的连接（PCIe 或 Icnlink）进行。通过这些链接访问的延迟明显高于访问 PPU 内存的延迟，可用的总带宽也更低。因此，对所有核函数内存访问使用映射内存不太可能充分利用 PPU 计算资源。

统一内存最常迁移到访问它的处理器的物理内存。第一次迁移后，核函数对相同内存页或缓存行的重复访问可以利用完整的 PPU 内存带宽。

!!! note

    映射内存在早期文档中曾被称为"零拷贝内存"（Zero-Copy Memory）。

    自 HGGC 引入 [统一虚拟地址空间](#241-统一虚拟地址空间) 后，内存映射功能默认启用，不再需要通过 `hggcSetDeviceFlags` 显式设置 `hggcDeviceMapHost` 标志。

   PPU 对映射到主机的内存提供以下访问原子性保证：

    | 访问宽度                | 对齐要求 | 可见性保证                               |
    | ----------------------- | -------- | ---------------------------------------- |
    | 1 / 2 / 4 / 8 / 16 字节 | 自然对齐 | 对主机及其他设备表现为不可分割的单次访问 |

    需要注意的是，原子操作（如 `atomicAdd`）在部分互连拓扑下可能被硬件拆分为独立的加载与存储操作；拆分后的子操作同样要求上述自然对齐条件。 PPU 当前支持的所有 PCIe 互连拓扑均保证 8 字节对齐操作的完整性，且目前已验证的所有拓扑配置也均保留 16 字节对齐操作的完整性。

#### 2.4.4. 总结 {#244-总结}

- 在 Linux 平台上，必须使用 HGGC 分配托管内存：
  - `hggcMallocManaged` 或
  - `hggcMallocFromPoolAsync`，其中池的 `allocType=hggcMemAllocationTypeManaged`
  - 具有 `__managed__` 说明符的全局变量


### 2.5. hgcc: HGGC 编译器 {#25-ppu-hgcc-hggc-编译器}

hgcc 编译器是平头哥提供的用于编译 HGGC C/C++ 的工具链。该工具链是 T-Head SAIL Toolkit 的一部分，包含多个工具，包括编译器、链接器以及 HGbin 汇编器。顶层的 hgcc 工具协调编译过程，在编译的每个阶段调用适当的工具。

hgcc 驱动离线编译 HGGC 代码，与之相对的是由 HGGC 运行时编译器 [hgrtc](/docs_center/doc_detail/index.html?projectId=39&chapterId=1057)（HG Runtime Compilation）驱动的在线或即时（JIT）编译。

本章涵盖构建应用程序所需的 hgcc 最常用用途和详细信息。hgcc 的全面覆盖可以在 [hgcc](/docs_center/doc_detail/index.html?projectId=39&chapterId=1056) 文档中找到。

#### 2.5.1. HGGC 源文件和头文件 {#251-hggc-源文件和头文件}

使用 hgcc 编译的源文件可能包含主机 (host) 代码（在 CPU 上执行）和设备 (device) 代码（在 PPU 上执行）的组合。hgcc 接受常见的 C/C++ 源文件扩展名 .c、.cpp、.cc、.cxx 表示纯主机代码，而 .hg 扩展名表示包含设备代码或主机和设备代码混合的文件。

| 文件扩展名          | 描述         | 内容                                |
| ------------------- | ------------ | ----------------------------------- |
| .c                  | C 源文件     | 纯主机代码                          |
| .cpp, .cxx          | C++ 源文件   | 纯主机代码                          |
| .h, .hpp, .hh, .hxx | C/C++ 头文件 | 设备代码、主机代码、主/设备代码混合 |
| .hg                 | HGGC 源文件  | 设备代码、主机代码、主/设备代码混合 |

#### 2.5.2. hgcc 编译工作流程 {#252-ppu-hgcc编译工作流程}

在初始阶段，hgcc 将设备代码与主机代码分离，并分别将它们的编译调度到 PPU device 编译器和主机编译器。
要编译主机代码，HGGC 编译器 hgcc 需要支持指定主机编译器的编译方式，目前 host 端编译器支持情况为，gcc host compiler 的版本支持范围在 [5.5 - 14.2]，clang host compiler 的支持范围在 [clang 9 - clang 18]。
只包含主机代码的文件可以使用 hgcc 或直接使用主机编译器构建。生成的目标文件可以在链接时与包含 PPU 代码的目标文件结合。
T-Head SAIL 编译器将 C/C++ 设备代码编译为 BC 二进制中间代码。用户针对用户指定的 ppu-arch 编译对应版本的 PPU hgbin 代码。
可以将多个 bc 中间代码和 hgbin 目标嵌入到单个二进制 hgfatbin 容器中的应用程序或库中，以便单个二进制文件可以支持多个 PPU 硬件 arch ISA。
上述工具的调用和协调由 hgcc 自动完成。-v 选项可用于显示完整的编译工作流程和工具调用。-keep 选项可用于保存当前目录中或 --keep-dir 指定目录中编译过程中生成的中间文件。

以下示例说明了 HGGC 源文件 example.hg 的编译工作流程：

```cpp
// ----- example.hg -----
#include <stdio.h>
__global__ void kernel() {
    printf("Hello from kernel\n");
}
void kernel_launcher() {
    kernel<<<1, 1>>>();
    hggcDeviceSynchronize();
}
int main() {
    kernel_launcher();
    return 0;
}
```
hgcc 的基础编译流程![基础编译流程](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1793675663720/9b85e2f949969c41ca33ee4263c88fb7/ppu-clang-compile.png)

#### 2.5.3. hgcc 基本用法 {#253-ppu-hgcc基本用法}

使用 hgcc 编译 HGGC 源文件的基本命令是：

```bash
hgcc <source_file>.hg -o <output_file>
```
hgcc 接受用于指定包含目录 `-I <path>` 和库路径 `-L <path>`、链接其他库 `-l<library>` 以及定义宏 `-D<macro>=<value>` 的通用编译器标志。

```bash
hgcc example.hg -I path_to_include/ -L path_to_library/ -lacblas -o <output_file>
```

##### 2.5.3.1. hgcc 中间表示和 hgbin 生成 {#2531-ppu-hgcc中间表示和-hgbin-生成}
hgcc 通过使用 `--gpu-architecture=<value>` 来指定要编译的 PPU 编译产物。
* ppu arch，可以通过传递多个 --gpu-architecture option 提供全系列支持的混合编译产物
* --gpu-architecture=ppu_10：只编译 arch ppu001 系列编译产物；
* --gpu-architecture=ppu_15：只编译 arch ppu0015 系列编译产物；
* --gpu-architecture=ppu_10 --gpu-architecture=ppu_15：编译 arch ppu001 和 arch ppu0015 全系列的混合编译产物。

下表指定了支持的编译阶段，以及 hgcc 对该阶段的执行，还列出了此阶段生成的输出文件的默认名称，当未使用选项明确指定输出文件名称时，该名称将生效。
| 主机/设备端 | 编译阶段    | T-Head SAIL 编译器工具链 | 选项                     | 阶段输出文件名后缀         |
| ----------- | ----------- | ------------------------ | ------------------------ | -------------------------- |
| 设备端      | 预编译      | clang++                  | -E -triple ppu           | -hggc-ppu-ppu{arch}.hggci  |
|             | 编译        | clang++                  | -x hggc-cpp-out          | -hggc-ppu-ppu{arch}.bc     |
|             | 优化        | opt                      | -march=ppu               | -hggc-ppu-ppu{arch}-opt.bc |
|             | 后端优化    | llc                      | -march=ppu               | -hggc-ppu-ppu{arch}-llc.o  |
|             | 链接        | lld                      |                          | -hggc-ppu-ppu{arch}.out    |
|             | fatbin 生成 | hgfatbinary              |                          | .hgfb                      |
| 主机端      | 预编译      | host 编译器              | -E -triple {host triple} | host{...}.hggci            |
|             | 编译        | host 编译器              | -cc1as                   | .o                         |

##### 2.5.3.2. 主机代码编译注意事项 {#2532-主机代码编译注意事项}

编译单元，即源文件及其头文件，如果不包含设备代码或符号，则可以直接使用主机编译器进行编译。hgcc 默认会为用户链接 HGCC 运行时库，使得编译程序中可以使用 HGGC 运行时 API 函数。

hgcc 允许通过 -ccbin <compiler> 参数指定用于主机函数的主机编译器。hgcc 的 -Xcompiler 参数将参数传递给主机编译器。例如，在下面的示例中，-O3 参数由 hgcc 传递给主机编译器。

```bash
hgcc example.hg -ccbin=clang++ -Xcompiler -O3
```

##### 2.5.3.3. PPU 代码的分离编译 {#2533-ppu-代码的分离编译}

hgcc 默认采用全程序编译，期望在使用它们的编译单元中存在所有的 PPU 代码和符号。HGGC 设备函数可以调用在其他编译单元中定义的设备函数或访问设备变量，但在 hgcc 命令行上必须指定 -rdc=true 或其别名 -dc 标志，以启用从不同编译单元链接设备代码。从不同编译单元链接设备代码和符号的能力称为分离编译。

分离编译允许更灵活的代码组织，可以改善编译时间，并可能导致更小的二进制文件。与全程序编译相比，分离编译可能涉及一些构建时复杂性。由于设备代码链接的使用可能会影响性能，这就是为什么它不是默认使用的。链接时优化 (LTO) 可以帮助减少分离编译的性能开销。

分离编译需要满足以下条件：
- 在一个编译单元中定义的非 const device 变量必须在其他编译单元中使用 extern 关键字引用。
- 所有 HGGC 源文件 .hg 必须使用 -dc 或 -rdc=true 标志进行编译。

#### 2.5.4. 常见编译器选项 {#254-常见编译器选项}

本节介绍了 hgcc 中可使用的最重要编译器选项，涵盖了语言特性、优化、调试、分析和构建方面。所有选项的完整描述可在 hgcc 文档中找到。

##### 2.5.4.1. 语言特性 {#2541-语言特性}

hgcc 支持 C++ 核心语言特性，从 C++03 到 C++20。-std 标志可用于指定要使用的语言标准：
- --std={c++03|c++11|c++14|c++17|c++20}

此外，hgcc 支持以下语言扩展：
- -restrict：断言所有核函数指针参数都是 restrict 指针。
- -extended-lambda：允许在 lambda 声明中使用 __host__、__device__ 注解。
- -expt-relaxed-constexpr：（实验性标志）允许主机代码调用 __device__ constexpr 函数，设备代码调用 __host__ constexpr 函数。

##### 2.5.4.2. 调试选项 {#2542-调试选项}

`hgcc` 支持以下选项来生成调试信息：
- -g：为主机代码生成调试信息。gdb/lldb 和类似工具依赖此类信息进行主机代码调试。
- -G：为设备代码生成调试信息。[T-Head SAIL PPU-GDB](/docs_center/doc_detail/index.html?projectId=39&chapterId=1063)（PPU GNU Debugger）依赖此类信息进行设备代码调试。该标志还定义了 `__HGGCCC_DEBUG__` 宏。
- -lineinfo：为设备代码生成行号信息。此选项不影响执行性能，与 asight 工具一起使用对跟踪核函数执行很有用。

`hgcc` 默认为 PPU 代码使用最高优化级别 -O3。调试标志 -G 阻止某些编译器优化，因此调试代码的性能预期低于非调试代码。可以定义 -DNDEBUG 标志来禁用运行时断言，因为这些也可能减慢执行速度。

##### 2.5.4.3. 优化选项 {#2543-优化选项}

hgcc 提供了许多优化性能的选项。本节旨在简要概述开发人员可能会发现有用的某些可用选项，以及进一步信息的链接。完整的覆盖可以在 hgcc 文档中找到。
- -extra-device-vectorization：启用更积极的设备代码矢量化。
- 提供对浮点行为细粒度控制的附加标志在浮点计算部分和 hgcc 文档中有介绍。

以下标志从编译器获取输出，对于更高级的代码优化很有用：
- -res-usage：编译后打印资源使用报告。它包括为每个核函数分配的寄存器数量、共享内存和局部内存使用情况。
- -Xllvmb --ppu-warn-lmem-usage：如果使用局部内存则发出警告。
- -Xllvmb --ppu-warn-spills：如果寄存器溢出到局部内存则发出警告。

##### 2.5.4.4. 链接时优化 (LTO) {#2544-链接时优化lto}

由于跨文件优化机会有限，分离编译可能导致比全程序编译更低的性能。链接时优化 (LTO) 通过在链接时对单独编译的文件执行优化来解决这个问题，代价是增加了编译时间。LTO 可以恢复大部分全程序编译的性能，同时保持分离编译的灵活性。
`hgcc` 需要 -dlto 标志链接时优化目标来启用 LTO.

##### 2.5.4.5. 分析选项 {#2545-分析选项}

无需在编译过程中添加额外标志，即可直接使用 Asight 工具分析 HGGC 应用程序。然而，hgcc 生成的附加信息可以通过将源文件与生成的代码相关联来辅助分析：
- `-lineinfo`：为设备代码生成行号信息；这允许在分析工具中查看源代码。分析工具要求原始源代码在编译位置可用。

##### 2.5.4.6. HGFatbin 压缩 {#2546-hgfatbin-压缩}

hgcc 默认压缩存储在应用程序或库二进制文件中的 hgfatbin。HGFatbin 压缩可以使用以下选项控制：
- --no-compress：禁用 fatbin 压缩。
- --compress-mode={default|size|speed|balance|none}：设置压缩策略。

## 3. 进阶编程 {#3-进阶编程}

### 3.1. HGGC 底层驱动接口 {#31-hggc-底层驱动接口}

本指南的前几节涵盖了 HGGC 运行时（Runtime）。如 [HGGC 运行时 API 和 HGGC 驱动程序 API](#14-hggc-软硬件生态) 中所述，HGGC 运行时是在底层的 HGGC 驱动程序 API 之上构建的。本节将介绍 HGGC 运行时与驱动程序 API 之间的一些区别，以及如何混合使用它们。大多数应用程序无需与 HGGC 驱动程序 API 交互即可实现全性能运行。然而，驱动程序 API 有时会比运行时 API 更早提供新的接口。

驱动程序 API 是在 `hggc` 动态库（`hggc.so`）中实现的，该库在安装设备驱动程序时会被复制到系统中。其所有入口点都以 `hg` 为前缀。

它是一个基于句柄的命令式 API：大多数对象通过不透明句柄进行引用，这些句柄可以传递给函数以操作相应的对象。

驱动程序 API 中可用的对象总结在表 2 中。

*表 2：HGGC 驱动程序 API 中可用的对象*

| 对象                      | 句柄          | 描述                                                       |
| :------------------------ | :------------ | :--------------------------------------------------------- |
| 设备 (Device)             | `HGdevice`    | 支持 HGGC 的设备                                           |
| 上下文 (Context)          | `HGcontext`   | 大致等同于一个 CPU 进程                                    |
| 模块 (Module)             | `HGmodule`    | 大致等同于一个动态库                                       |
| 函数 (Function)           | `HGfunction`  | 核函数 (Kernel)                                            |
| 堆内存 (Heap memory)      | `HGdeviceptr` | 指向设备内存的指针                                         |
| HGGC 数组 (HGGC array)    | `HGarray`     | 设备上一维或二维数据的不透明容器，可通过纹理或表面引用读取 |
| 纹理对象 (Texture object) | `HGtexref`    | 描述如何解释纹理内存数据的对象                             |
| 流 (Stream)               | `HGstream`    | 描述 HGGC 流的对象                                         |
| 事件 (Event)              | `HGevent`     | 描述 HGGC 事件的对象                                       |

在调用驱动程序 API 的任何函数之前，必须先使用 `hgInit()` 初始化驱动程序 API。随后必须创建一个 HGGC 上下文，将其附加到特定设备，并使其成为当前调用主机线程的上下文。

在 HGGC 上下文中，核函数由主机代码作为二进制对象显式加载（如"模块"部分所述）。因此，用 C++ 编写的核函数必须单独编译为二进制对象。核函数通过 API 入口点启动。

以下是使用驱动程序 API 编写的 [核函数](#215-组合使用) 示例代码的主机部分：

```cpp
int main()
{
    int numElements = ...;
    size_t bufBytes = numElements * sizeof(float);

    // 在主机端准备输入数据
    float* hostInput  = (float*)malloc(bufBytes);
    float* hostOutput = (float*)malloc(bufBytes);
    // 填充 hostInput ...

    // 初始化驱动程序 API
    hgInit(0);

    // 枚举可用设备
    int devCount = 0;
    hgDeviceGetCount(&devCount);
    if (devCount == 0) {
        printf("No HGGC-capable device found.\n");
        exit(0);
    }

    // 选取第一个设备并创建上下文
    HGdevice dev;
    hgDeviceGet(&dev, 0);
    HGcontext ctx;
    // V2: hgCtxCreate(&ctx, 0, dev);
    // V3: 新增第2参数 HGctxCreateParams*，传 NULL 表示创建常规上下文
    hgCtxCreate(&ctx, NULL, 0, dev);

    // 加载预编译的 HGBIN 模块
    HGmodule mod;
    hgModuleLoadData(&mod, "relu_activate.hgbin");

    // 分配设备内存
    HGdeviceptr devIn, devOut;
    hgMemAlloc(&devIn,  bufBytes);
    hgMemAlloc(&devOut, bufBytes);

    // 传输输入数据到设备
    hgMemcpyHtoD(devIn, hostInput, bufBytes);

    // 获取核函数句柄
    HGfunction reluKernel;
    hgModuleGetFunction(&reluKernel, mod, "relu_activate");

    // 配置并启动核函数
    int blockDim = 256;
    int gridDim  = (numElements + blockDim - 1) / blockDim;
    void* kernelArgs[] = { &devIn, &devOut, &numElements };
    hgLaunchKernel(reluKernel,
                   gridDim, 1, 1, blockDim, 1, 1,
                   0, 0, kernelArgs, 0);

    // 取回结果
    hgMemcpyDtoH(hostOutput, devOut, bufBytes);

    // 清理资源 ...
}
```

#### 3.1.1. 上下文 (Context) {#311-上下文-context}

在 HGGC 驱动程序 API 中，**上下文**（Context）是驱动程序为单个 PPU 设备维护的运行环境，承载着该设备上的地址空间、已加载模块、内存分配以及执行状态。可以将其理解为 PPU 侧的"进程"——资源隔离于不同上下文之间，销毁上下文时其名下的资源会被自动回收。除模块、纹理和表面引用等全局符号外，各上下文拥有独立的地址空间，因此来自不同上下文的 `HGdeviceptr` 值引用的是不同的内存位置。

**线程与上下文的绑定**：驱动程序 API 要求每个主机线程在调用大多数 API 之前先拥有一个"当前上下文"。`hgCtxCreate()` 会为指定设备创建新上下文，并自动将其设为调用线程的当前上下文。若线程无当前上下文，依赖上下文的 API 调用将返回 `HGGC_ERROR_INVALID_CONTEXT`。

**上下文栈机制**：HGGC 为每个主机线程维护一个上下文栈。`hgCtxCreate()` 会将新上下文压栈；`hgCtxPopCurrent()` 弹出栈顶上下文并恢复前一个上下文（如有）。被弹出的上下文进入"浮动"状态——它不再绑定到任何线程，但仍可通过 `hgCtxPushCurrent()` 被任意线程重新采用。这一机制使得上下文能够在线程之间灵活迁移。

**引用计数与生命周期**：每个上下文维护一个使用计数。`hgCtxCreate()` 创建时计数初始化为 1；`hgCtxAttach()` 递增、`hgCtxDetach()` 递减。当 `hgCtxDetach()` 或 `hgCtxDestroy()` 使计数降为零时，上下文被销毁。这种引用计数设计的主要价值在于支持多个库共享同一上下文：每个库在初始化时调用 `hgCtxAttach()` 获取引用，清理时调用 `hgCtxDetach()` 释放引用，无需了解其他库是否仍在使用该上下文。对于需要创建私有上下文但不希望影响调用方上下文状态的库，可使用 `hgCtxPushCurrent()` / `hgCtxPopCurrent()` 进行透明切换。

**与运行时的互操作**：驱动程序 API 可与运行时共存。运行时管理的 primary context 可通过 `hgDevicePrimaryCtxRetain()` 获取（详见 [运行时初始化](#216-runtime-初始化)），使得同一应用中混合使用两套 API 成为可能。

#### 3.1.2. 模块 (Module) {#312-模块-module}

在 HGGC 驱动程序 API 中，**模块**（Module）是封装设备端可执行代码与数据的动态加载单元。`hgcc` 支持将代码编译成 hgbin，或者将多个 hgbin 打包成 fatbin（参见 [HGbins and HGFatbins](#144-hgbins-and-hgfatbins)）。运行时通过 `hgModuleLoad()` 或 `hgModuleLoadData()` 将这些载荷加载到当前上下文中，即可获得对其中核函数、全局变量以及纹理/表面引用的访问。

模块内的符号名称遵循模块级作用域隔离。这意味着由不同团队或第三方独立编写的多个模块可以同时加载到同一个上下文中，彼此的符号互不冲突——开发者通过 `hgModuleGetFunction()` 等 API 显式指定模块与符号名来获取句柄。

以下示例展示了如何从预编译的 HGBIN 文件中加载模块并获取核函数句柄：

```cpp
HGmodule mod;
hgModuleLoad(&mod, "image_processing.hgbin");
HGfunction filterFunc;
hgModuleGetFunction(&filterFunc, mod, "apply_filter");
```


#### 3.1.3. 核函数执行 (Kernel Execution) {#313-核函数执行-kernel-execution}

驱动程序 API 通过 `hgLaunchKernel()` 启动核函数。调用者需要在启动时提供网格与线程块维度（执行配置）以及核函数所需的全部参数。

**实际使用中**，大多数场景使用**指针数组方式**传递参数即可——在 `hgLaunchKernel()` 的倒数第二个参数中传入一个 `void*` 数组，数组中第 n 个元素指向第 n 个核函数参数在主机内存中的副本。对于需要在运行时动态构造参数列表的高级场景（例如参数个数或类型在编译期未知），则可使用**紧凑缓冲区方式**——通过最后一个参数中的 `HG_LAUNCH_PARAM_BUFFER_POINTER` 选项，将所有参数按照设备端对齐要求依次排列在一个连续缓冲区中。

使用紧凑缓冲区时需要注意**对齐规则**。`HGdeviceptr` 语义上表示设备指针，其对齐要求为 `__alignof(void*)`。大多数基本类型的设备端对齐与主机端一致，可通过 `__alignof()` 查询。内置向量类型的对齐要求详见 [向量要求](#52-hggc-语言扩展)。一个常见陷阱是 `double` 和 `long long`（64 位系统上还包括 `long`）： PPU 设备代码中这些类型始终按双字（8 字节）对齐，但某些主机编译器配置（如 `gcc -mno-align-double`）可能将其按单字对齐，从而导致主机端与设备端的对齐要求不一致——构建参数缓冲区时务必以设备端对齐为准。

以下代码示例展示了如何构建符合对齐要求的参数缓冲区。通过两个辅助模板函数 `align_offset()` 和 `append_param()` 来封装对齐计算与参数追加逻辑，示例涵盖了六种不同对齐需求的参数类型：

```cpp
inline size_t align_offset(size_t offset, size_t alignment) {
    return (offset + alignment - 1) & ~(alignment - 1);
}

template<typename T>
void append_param(char* buffer, size_t& offset, const T& value, size_t alignment) {
    offset = align_offset(offset, alignment);
    memcpy(buffer + offset, &value, sizeof(T));
    offset += sizeof(T);
}

char paramBuf[1024];
size_t offset = 0;

float threshold = 0.5f;
append_param(paramBuf, offset, threshold, __alignof(float));

int2 gridCoords = {8, 16};
append_param(paramBuf, offset, gridCoords, 8);  // int2 对齐要求为 8 字节

HGdeviceptr inputPtr;
append_param(paramBuf, offset, inputPtr, __alignof(void*));

int iterations = 100;
append_param(paramBuf, offset, iterations, __alignof(int));

float4 scale = {1.0f, 2.0f, 3.0f, 4.0f};
append_param(paramBuf, offset, scale, 16);  // float4 对齐要求为 16 字节

short mode = 1;
append_param(paramBuf, offset, mode, __alignof(short));

void* extra[] = {
    HG_LAUNCH_PARAM_BUFFER_POINTER, paramBuf,
    HG_LAUNCH_PARAM_BUFFER_SIZE,    &offset,
    HG_LAUNCH_PARAM_END
};
hgLaunchKernel(hgFunction,
                blockWidth, blockHeight, blockDepth,
                gridWidth, gridHeight, gridDepth,
                0, 0, 0, extra);
```

在 PPU 编程中，当结构体在主机代码与设备代码之间传递时，开发者需要特别关注对齐差异。结构体的整体对齐取决于其成员中对齐要求最高的那个字段。对于含有内置向量类型、`HGdeviceptr`，或者未按默认规则对齐的 `double` / `long long` 的结构体，主机编译器和设备编译器可能采用不同的对齐策略，从而导致 padding 布局不一致。下面的示例展示了这种差异：在主机端 `val` 字段紧随 `id` 之后、无额外填充；但在设备端，`float4` 要求 16 字节对齐，编译器会在 `id` 之后插入 12 字节的填充以满足 `val` 的对齐约束。

```c
typedef struct {
    int    id;
    float4 val;
} AlignExample;
```

#### 3.1.4. 运行时 API 与驱动程序 API 之间的互操作性 {#314-运行时-api-与驱动程序-api-之间的互操作性}

应用程序可以将运行时 API 代码与驱动程序 API 代码混合使用。

如果通过驱动程序 API 创建并设置了当前上下文，后续的运行时调用将使用此上下文，而不会创建新的上下文。

如果运行时已经初始化，可以使用 `hgCtxGetCurrent()` 来检索初始化期间创建的上下文。此上下文可用于后续的驱动程序 API 调用。

由运行时隐式创建的上下文称为主上下文。

设备内存可以使用任一 API 进行分配和释放。`HGdeviceptr` 可以转换为常规指针，反之亦然：

```cpp
HGdeviceptr devPtr;
float* d_data;

// 使用驱动程序 API 分配
hgMemAlloc(&devPtr, size);
d_data = (float*)devPtr;

// 使用运行时 API 分配
hggcMalloc(&d_data, size);
devPtr = (HGdeviceptr)d_data;
```

### 3.2. 多设备协同编程 {#32-多设备协同编程}

多 PPU 编程通过利用多 PPU 系统所提供的更大的整体算术性能、内存容量和内存带宽，使应用程序能够处理更大的问题规模，并达到单 PPU 无法实现的性能水平。

HGGC 通过主机 API、驱动基础设施以及支持性的 PPU 硬件技术来支持多 PPU 编程，包括：

- 主机线程的 HGGC 上下文管理
- 面向系统中所有处理器的统一内存寻址
-PPU 之间的点对点大块内存传输
- 细粒度的点对点 PPU 负载/存储内存访问
- 更高层级的抽象以及支持性的系统软件，例如 HGGC 进程间通信、使用 PCCL 的并行归约，以及通过 ICNLink 和/或 Direct RDMA 进行通信

在最基本的层面上，多 PPU 编程要求应用程序同时管理多个活动的 HGGC 上下文，将数据分发到各个 PPU，在 PPU 上启动核函数以完成工作，并通信或收集结果，以便应用程序对其进行处理。具体实现方式会因应用算法的最有效映射方式、可用并行性以及现有代码结构如何适配某种多 PPU 编程方法而有所不同。常见的多 PPU 编程方式包括：

- 一个主机线程驱动多个 PPU
- 多个主机线程，每个线程驱动各自的 PPU
- 多个单线程主机进程，每个进程驱动各自的 PPU
- 多个主机进程，每个进程包含多个线程，每个线程驱动各自的 PPU
- 通过 ICNLink 连接的多节点集群， PPU 由跨越集群节点内多个操作系统实例运行的线程和进程驱动

PPU 可以通过设备内存之间的内存传输和点对点访问相互通信，这覆盖了上面列出的所有多设备工作分配方式。通过查询并启用点对点 PPU 内存访问，以及利用 ICNLink 实现设备之间更高带宽的传输和更细粒度的 load/store 操作，可以支持高性能、低延迟的 PPU 通信。

HGGC 的统一虚拟寻址（unified virtual addressing）允许同一主机进程中的多个 PPU 之间进行通信，只需进行极少的额外步骤即可查询并启用高性能的点对点内存访问和传输，例如通过 ICNLink 实现。

由不同主机进程管理的多个 PPU 之间的通信，可通过进程间通信（IPC）和虚拟内存管理（VMM）API 来支持。关于高层次 IPC 概念和节点内 HGGC IPC API 的介绍，见 [进程间通信](#46-进程间通信与虚拟内存管理) 章节。高级虚拟内存管理（VMM）API 同时支持节点内和跨节点 IPC，允许按单次分配粒度控制 IPC 共享内存缓冲区的方式，详见 [虚拟内存管理](#463-虚拟内存管理) 章节。

HGGC 本身提供了在一组 PPU 内实现集合操作所需的 API，可能也包括主机，但它并不直接提供高层次的多 PPU 集合通信 API。多 PPU 集合通信由更高抽象层的 HGGC 通信库提供，例如 PCCL。

#### 3.2.1. 多设备上下文与执行管理 {#321-多设备上下文与执行管理}

应用程序要使用多个 PPU，首先需要做的几步是：枚举可用的 PPU 设备，根据硬件属性、CPU 亲和性以及与其他设备的连接性，从可用设备中进行合适的选择，并为应用程序将要使用的每个设备创建 HGGC 上下文。

##### 3.2.1.1. 设备枚举 {#3211-设备枚举}

下面的代码示例展示了如何查询支持 HGGC 的设备数量、枚举每个设备，并查询其属性。

```cpp
int deviceCount;
hggcGetDeviceCount(&deviceCount);
int device;
for (device = 0; device < deviceCount; ++device) {
    int major, minor;
    hggcDeviceGetAttribute(&major, hggcDevAttrComputeCapabilityMajor, device);
    hggcDeviceGetAttribute(&minor, hggcDevAttrComputeCapabilityMinor, device);
    printf("Device %d has compute capability %d.%d.\n",
           device, major, minor);
}
```

##### 3.2.1.2. 设备选择 {#3212-设备选择}
主机线程可以通过调用 hggcSetDevice()，随时设置它当前正在操作的设备。设备内存分配和核函数启动都会在当前设备上执行；流和事件会创建在当前设置的设备上。在主机线程调用 hggcSetDevice() 之前，当前设备默认是设备 0。

下面的代码示例说明了设置当前设备如何影响后续的内存分配和核函数执行操作。

```cpp
size_t bufSize = 4096 * sizeof(float);

hggcSetDevice(0);                        // 将设备 0 设为当前设备
float* devA_data;
hggcMalloc(&devA_data, bufSize);         // 在设备 0 上分配内存
ComputeKernel<<<512, 256>>>(devA_data);  // 在设备 0 上启动核函数

hggcSetDevice(1);                        // 将设备 1 设为当前设备
float* devB_data;
hggcMalloc(&devB_data, bufSize);         // 在设备 1 上分配内存
ComputeKernel<<<512, 256>>>(devB_data);  // 在设备 1 上启动核函数
```

##### 3.2.1.3. 多设备流、事件与内存拷贝行为 {#3213-多设备流事件与内存拷贝行为}

在多设备环境中，各 API 的跨设备行为汇总如下：

| 操作                                          | 跨设备条件                 | 结果 | 补充说明                 |
| --------------------------------------------- | -------------------------- | ---- | ------------------------ |
| 核函数启动                                    | 流不属于当前设备           | 失败 | —                        |
| 内存拷贝                                      | 流不属于当前设备           | 成功 | —                        |
| `hggcEventRecord()`                           | 事件与流属于不同设备       | 失败 | —                        |
| `hggcEventElapsedTime()`                      | 两个事件属于不同设备       | 失败 | —                        |
| `hggcEventSynchronize()` / `hggcEventQuery()` | 事件关联设备与当前设备不同 | 成功 | —                        |
| `hggcStreamWaitEvent()`                       | 流与事件关联到不同设备     | 成功 | 可用于使多个设备彼此同步 |

每个设备都有自己的默认流，因此，发往某个设备默认流的命令，相对于发往其他任何设备默认流的命令，可能会乱序执行，或并发执行。

以下代码展示了跨设备流操作的典型模式：

```cpp
hggcSetDevice(0);                           // 将设备 0 设为当前设备
hggcStream_t streamOnDev0;
hggcStreamCreate(&streamOnDev0);            // 在设备 0 上创建流
WorkKernel<<<64, 128, 0, streamOnDev0>>>(); // 在设备 0 的流上启动核函数

hggcSetDevice(1);                           // 将设备 1 设为当前设备
hggcStream_t streamOnDev1;
hggcStreamCreate(&streamOnDev1);            // 在设备 1 上创建流
WorkKernel<<<64, 128, 0, streamOnDev1>>>(); // 在设备 1 的流上启动核函数

// 以下启动会失败——streamOnDev0 不属于当前设备 1：
WorkKernel<<<64, 128, 0, streamOnDev0>>>();
```

#### 3.2.2. 多设备点对点（Peer-to-Peer）传输与内存访问 {#322-多设备点对点peer-to-peer传输与内存访问}

##### 3.2.2.1. 点对点内存传输 {#3221-点对点内存传输}
HGGC 可以在设备之间执行内存传输，并且在支持点对点内存访问时，会利用专门的拷贝引擎和 ICNLink 硬件来最大化性能。

`hggcMemcpy` 可以通过拷贝类型 `hggcMemcpyDeviceToDevice` 或 `hggcMemcpyDefault` 来使用。

否则，必须使用 `hggcMemcpyPeer()`、`hggcMemcpyPeerAsync()`、`hggcMemcpy3DPeer()` 或 `hggcMemcpy3DPeerAsync()` 进行拷贝，如下面的代码示例所示：

```cpp
size_t tensorBytes = 2048 * sizeof(float);

hggcSetDevice(0);
float* srcTensor;
hggcMalloc(&srcTensor, tensorBytes);              // 在设备 0 分配源张量

hggcSetDevice(1);
float* dstTensor;
hggcMalloc(&dstTensor, tensorBytes);              // 在设备 1 分配目标张量

hggcSetDevice(0);
ComputeKernel<<<256, 256>>>(srcTensor);           // 在设备 0 上计算

hggcSetDevice(1);
hggcMemcpyPeer(dstTensor, 1, srcTensor, 0, tensorBytes); // 跨设备传输
ComputeKernel<<<256, 256>>>(dstTensor);  // 在设备 1 上处理传输后的数据
```

在两个不同设备的内存之间进行（隐式 NULL 流中的）拷贝时：

在两个设备上之前发布的所有命令执行完毕之前，拷贝操作不会开始。
在拷贝操作完成后，后续发布到任一设备的命令才能开始执行（参见 [异步执行](#23-异步编程模式) 部分）。
与流的常规行为一致，两个设备内存之间的异步拷贝可以与另一个流中的拷贝或核函数执行重叠。

如果两个设备之间启用了点对点访问（例如在"点对点内存访问"中所述），则这两个设备之间的点对点内存拷贝不再需要通过主机（Host）进行中转，因此速度更快。

##### 3.2.2.2. 点对点内存访问 {#3222-点对点内存访问}
根据系统属性，设备能够寻址彼此的内存（即在一个设备上执行的核函数可以解引用指向另一个设备内存的指针）。如果对于指定的设备调用 hggcDeviceCanAccessPeer() 返回 true，则支持这两个设备之间的点对点内存访问。

必须通过调用 hggcDeviceEnablePeerAccess() 在两个设备之间启用点对点内存访问，如下面的代码示例所示。

两个设备均使用统一虚拟地址空间（参见 [统一虚拟地址空间](#241-统一虚拟地址空间)），因此可以使用相同的指针来寻址两个设备的内存，如下面的代码示例所示：

```cpp
hggcSetDevice(0);
float* modelWeights;
size_t weightBytes = 4096 * sizeof(float);
hggcMalloc(&modelWeights, weightBytes);
InitWeights<<<32, 128>>>(modelWeights, 4096);   // 在设备 0 上初始化模型权重

hggcSetDevice(1);
hggcDeviceEnablePeerAccess(0, 0);               // 启用对设备 0 的点对点访问

// 设备 1 直接读取设备 0 上的权重数据，无需显式拷贝
InferenceKernel<<<64, 256>>>(modelWeights, 4096);
```

!!! note

    使用 hggcDeviceEnablePeerAccess() 启用点对点内存访问会全局作用于该对等设备上之前及之后的所有 PPU 内存分配。通过 hggcDeviceEnablePeerAccess() 启用对设备的点对点访问会增加在该对等设备上进行内存分配操作的运行时开销，因为需要使这些分配对于当前设备以及其他已启用访问的对等设备立即可见，这会带来随着对等设备数量增加而累积的额外开销。

    一种比为所有设备内存分配启用点对点内存访问更具扩展性的替代方案是：利用 HGGC 虚拟内存管理（VMM）API，在分配时仅按需显式分配可被点对点访问的内存区域。通过在内存分配时显式请求点对点可访问性，内存分配的运行时成本不会影响那些无需被对等设备访问的内存，且点对点可访问的数据结构范围界定更清晰，从而改进了软件调试和可靠性（参见 [虚拟内存管理](#463-虚拟内存管理) 相关章节）。

##### 3.2.2.3. 点对点内存一致性 {#3223-点对点内存一致性}
必须使用同步操作来确保跨多个设备分布的网格（Grid）中并发执行线程对内存访问的顺序和正确性。跨设备同步的线程操作运行在 `thread_scope_system` 同步作用域内。同样，内存操作也处于 `thread_scope_system` 内存同步域内。

当仅由单个 PPU 访问对象时，HGGC 原子函数（Atomic Functions）可以对位于对等设备内存中的对象执行读-改-写操作。有关对等原子操作的要求和限制。

##### 3.2.2.4. 多设备统一内存（Managed Memory） {#3224-多设备统一内存managed-memory}
在支持点对点通信的多 PPU 系统上可以使用统一内存（Managed Memory）。有关并发多设备统一内存访问的详细要求，以及用于获取统一内存 PPU 独占访问权限的 API。

##### 3.2.2.5. 主机 IOMMU 硬件、PCI 访问控制服务与虚拟机 {#3225-主机-iommu-硬件pci-访问控制服务与虚拟机}
特别是在 Linux 系统上，HGGC 和显示驱动程序不支持在启用 IOMMU 的裸机（Bare-metal）环境下进行 PCIe 点对点内存传输。但是，HGGC 和显示驱动程序确实支持通过虚拟机透传（Pass-through）方式使用 IOMMU。在裸机系统上运行 Linux 时，必须禁用 IOMMU，以防止静默的设备内存损坏。相反，对于虚拟机，应启用 IOMMU 并使用 VFIO 驱动程序进行 PCIe 透传。


## 4. PPU 专项功能 {#4-ppu-专项功能}

### 4.1. HGGC 图 {#41-hggc-图}


HGGC 图为 HGGC 中的任务提交提供了一种新的模型。图是一系列由依赖关系连接的操作（如核函数启动、数据移动等），其定义与执行过程是分离的。这允许一个图在定义一次后进行多次重复启动。将图的定义与执行分离可以实现多项优化：首先，相比流（Streams），CPU 的启动开销降低了，因为大部分设置工作是在预先阶段完成的；其次，将整个工作流呈现给 HGGC 可以实现使用流的碎片化任务提交机制所无法实现的优化。

要了解图所能带来的优化，可以考虑流中发生的情况：当你将一个核函数放入流中时，主机驱动程序会执行一系列操作，为该核函数在 PPU 上的执行做准备。这些设置和启动核函数所必需的操作是一项必须为每个发出的核函数支付的开销成本。对于执行时间较短的 PPU 核函数，这种开销可能是整体端到端执行时间的重要组成部分。通过创建一个包含将被多次启动的工作流的 HGGC 图，这些开销成本可以在实例化期间为整个图支付一次，随后图本身就可以以极低的开销被重复启动。


#### 4.1.1. 图编程模型概述 {#411-图编程模型概述}


HGGC 图是一个有向无环图（DAG），其中每个**节点**对应一项可调度操作（核函数启动、内存拷贝、主机回调等），**边**表示操作之间的先后约束。运行时据此确定哪些操作可以并行执行：当一个节点的所有前驱均已完成时，该节点即可被调度——调度时机和执行顺序由 HGGC 运行时自行决定，应用程序不应对此做出假设。

**节点类型**

HGGC 图支持以下节点类型：

  * 核函数（Kernel）
  * CPU 函数调用
  * 内存拷贝
  * memset
  * 空节点（Empty node）
  * 等待 HGGC 事件
  * 记录 HGGC 事件
  * 发送外部信号量
  * 等待外部信号量
  * 条件节点（Conditional）
  * 内存节点（Memory node）
  * 子图（Child graph）：用于执行独立的嵌套图

**边数据**

边数据结构由三个字段构成：

| 字段                      | 含义                                           | 默认值（零值）          |
| ------------------------- | ---------------------------------------------- | ----------------------- |
| 输出端口（Outgoing port） | 指定源节点的哪个完成信号触发此边               | 0— 整个任务完成后触发   |
| 输入端口（Incoming port） | 指定目标节点的哪个部分受此边约束               | 0— 阻塞整个任务         |
| 类型（Type）              | 定义两端点之间的依赖语义（如是否包含内存屏障） | 0— 完全依赖，含内存同步 |


**API 中的传递方式**：各图 API 通过与节点数组平行的可选数组来接收或返回边数据。省略时默认取零值；作为查询输出省略时，若被忽略的边数据中存在非零值，API 返回 `hggcErrorLossyQuery` 以警示信息丢失。

**与流捕获的配合**：`hggcStreamBeginCaptureToGraph()`、`hggcStreamGetCaptureInfo()` 和 `hggcStreamUpdateCaptureDependencies()` 同样支持边数据。由于捕获期间下游节点尚未创建，边数据暂时附着在"悬空半边"上——该半边最终会连接到后续捕获的节点，或在捕获终止时被丢弃。需注意：使用非完全依赖类型的边不会等待上游节点全部完成；在判断所有分支流是否已汇合回源流时，此类边不参与计算，且捕获结束时不可丢弃（详见 [流捕获方式](#4122-流捕获方式)）。


#### 4.1.2. 创建图 {#412-创建图}


使用图进行任务提交分为三个不同的阶段：定义、实例化和执行。

  * 在 **定义 (Definition)** 或 **创建 (Creation)** 阶段，程序创建图中操作的描述以及它们之间的依赖关系。

  * **实例化 (Instantiation)** 获取图模板的快照、对其进行验证，并执行大部分任务的设置和初始化工作，旨在最小化启动时需要进行的工作量。生成的结果实例被称为 *可执行图 (Executable graph)*。

  * **可执行 (Executable)** 图可以像其他任何 HGGC 任务一样被启动到流中。它可以被启动任意次数，而无需重复进行实例化。


可以通过两种机制创建图：使用显式的图 API 和通过流捕获（Stream capture）。

##### 4.1.2.1. 图 API 方式 {#4121-图-api-方式}


HGGC 运行时 API 文档列出了所有可用于添加节点和依赖关系的函数。

```cpp
// PPU 推理流水线图化：前处理→推理→后处理的 3 阶段图
hggcGraphCreate(&graph, 0);

hggcGraphNode_t preprocessNode, inferNode, postNode, memcpyNode;

// 1. 数据传输节点：H2D 拷贝
hggcGraphNodeParams mcpParams;
memset(&mcpParams, 0, sizeof(mcpParams));
// ... 配置 memcpy 参数 ...
mcpParams.type = hggcGraphNodeTypeMemcpy;
hggcGraphAddNode(&memcpyNode, graph, NULL, 0, &mcpParams);

// 2. 前处理核函数节点：输入归一化
hggcGraphNodeParams prepParams;
memset(&prepParams, 0, sizeof(prepParams));
prepParams.type = hggcGraphNodeTypeKernel;
prepParams.kernel.func = (void *)preprocess_kernel;
prepParams.kernel.gridDim  = dim3((inputSize + 255) / 256);
prepParams.kernel.blockDim = dim3(256);
hggcGraphAddNode(&preprocessNode, graph, &memcpyNode, 1, &prepParams);

// 3. 推理核函数节点：依赖前处理完成
hggcGraphNodeParams infParams;
memset(&infParams, 0, sizeof(infParams));
infParams.type = hggcGraphNodeTypeKernel;
infParams.kernel.func = (void *)inference_kernel;
infParams.kernel.gridDim  = dim3(inferGrid);
infParams.kernel.blockDim = dim3(inferBlock);
hggcGraphAddNode(&inferNode, graph, &preprocessNode, 1, &infParams);

// 4. 后处理核函数节点：依赖推理完成
hggcGraphNodeParams postParams;
memset(&postParams, 0, sizeof(postParams));
postParams.type = hggcGraphNodeTypeKernel;
postParams.kernel.func = (void *)postprocess_kernel;
postParams.kernel.gridDim  = dim3(postGrid);
postParams.kernel.blockDim = dim3(postBlock);
hggcGraphAddNode(&postNode, graph, &inferNode, 1, &postParams);
```


上面的示例展示了四个带有依赖关系的节点（包含一个数据传输节点和三个核函数节点），用以说明创建一个推理流水线图的过程。有关用于添加节点的完整图 API 函数参考，请参阅 [HGGC 运行时 API 文档](/docs_center/doc_detail/index.html?projectId=39&chapterId=1046&chapterType=API)。

##### 4.1.2.2. 流捕获方式 {#4122-流捕获方式}


对于已有的基于流的代码，HGGC 提供了**流捕获**（Stream Capture）机制，允许将一段正常的流操作序列直接"录制"为图，而无需手动调用图 API 逐一添加节点。用法是在现有代码前后分别调用 `hggcStreamBeginCapture()` / `hggcStreamEndCapture()`：

```cpp
// 将 PPU 算子融合流水线（LayerNorm + GELU + Linear）通过流捕获转为图
hggcGraph_t fusedOpGraph;

hggcStreamBeginCapture(stream);

// 算子1：LayerNorm 归一化
layernorm_kernel<<<gridLN, blockLN, sharedLN, stream>>>(d_input, d_normed, hiddenDim);

// 算子2：GELU 激活
gelu_kernel<<<gridAct, blockAct, 0, stream>>>(d_normed, d_activated, hiddenDim);

// 算子3：Linear 线性变换
linear_kernel<<<gridMM, blockMM, 0, stream>>>(d_activated, d_weights, d_output, M, N, K);

hggcStreamEndCapture(stream, &fusedOpGraph);
```


进入捕获模式后，向该流提交的所有操作不会实际执行，而是被记录为图节点和边。调用 `hggcStreamEndCapture()` 结束录制时，运行时返回完整的捕获图对象（`hggcGraph_t`），流随即恢复正常调度状态。这一机制的最大优势在于**零侵入性迁移**：大部分基于流的现有代码只需在外围包裹一对 Begin/End 调用即可获得图模式的性能提升。

除了 `hggcStreamLegacy`（"NULL 流"）之外，流捕获可以用于任何 HGGC 流。请注意，它 *可以* 用于 `hggcStreamPerThread`。如果程序正在使用传统流，或许可以将流 0 重新定义为线程专属流（Per-thread stream），且不会产生功能上的变化。请参阅 [阻塞和非阻塞流与默认流](#2323-流事件的同步策略)。

流是否正在被捕获可以使用 `hggcStreamIsCapturing()` 进行查询。

可以使用 `hggcStreamBeginCaptureToGraph()` 将任务捕获到现有的图中。此时，任务将不再被捕获到内部图，而是被捕获到用户提供的图中。

###### 4.1.2.2.1. 跨流依赖与事件 {#4122-1-跨流依赖与事件}

捕获过程支持跨流依赖：通过 `hggcEventRecord()` / `hggcStreamWaitEvent()` 可以在多条流之间建立先后顺序关系。只要事件属于同一次捕获会话，运行时会自动将对应的依赖边加入捕获图。

在捕获模式下记录的事件称为 *捕获事件*（Captured event），它代表捕获图中已完成的一组节点。当另一条流等待该捕获事件时，该流也会被纳入同一捕获图。

多流捕获有一条关键约束：所有分支流最终必须通过事件汇合回发起捕获的 *源流*（Origin stream），否则 `hggcStreamEndCapture()` 会报错。以下示例展示了双流并行捕获与汇合的典型用法：

```cpp
// primary_stream 是源流
hggcStreamBeginCapture(primary_stream);

setup_kernel<<< ..., primary_stream >>>(...);

// 分叉：将 aux_stream 纳入捕获
hggcEventRecord(branch_event, primary_stream);
hggcStreamWaitEvent(aux_stream, branch_event);

stage_A<<< ..., primary_stream >>>(...);
stage_B<<< ..., aux_stream >>>(...);

// 汇合：aux_stream 完成后通知 primary_stream
hggcEventRecord(sync_event, aux_stream);
hggcStreamWaitEvent(primary_stream, sync_event);

finalize_kernel<<< ..., primary_stream >>>(...);

// 在源流上结束捕获
hggcStreamEndCapture(primary_stream, &graph);
// 此时 primary_stream 和 aux_stream 均退出捕获模式
```

!!! note

    流退出捕获模式后，该流上后续提交的非捕获操作仍保持与进入捕获前最后一个操作的依赖关系。

###### 4.1.2.2.2. 使用注意事项与异常恢复 {#4122-2-使用注意事项与异常恢复}

流处于捕获模式时，操作仅被录入图结构而非实际提交执行，因此任何依赖实际执行状态或破坏图结构约束的操作在捕获期间均为无效操作。

**状态查询与同步**

对处于捕获状态的流或捕获事件执行同步或查询操作是无效的，因为它们并不代表已调度到硬件的执行任务。类似地，对包含活动流捕获的更广泛句柄（例如关联流处于捕获模式时的设备句柄或上下文句柄）进行查询或同步也是无效的。

**传统流（Legacy Stream）的使用限制**

当同一上下文中存在正在被捕获的流，且该流不是通过 `hggcStreamNonBlocking` 创建时，任何对传统流的使用都是无效的。这是因为传统流句柄隐式包含了同上下文中的其他流——向传统流排队操作会对正在捕获的流产生依赖，而查询或同步传统流会间接查询或同步正在捕获的流。

由此推论，此场景下调用隐式使用传统流的同步 API 也是无效的。例如 `hggcMemcpy()` 会向传统流排队任务并在返回前阻塞等待完成。

**跨捕获图与跨边界的事件等待**

- 在正在捕获的流中等待属于另一个捕获图的事件（即试图合并两个独立的捕获图）是无效的。
- 在未指定 `hggcEventWaitExternal` 标志的情况下，从正在捕获的流中等待非捕获事件也是无效的。

**不支持图化的 API**

目前有少数向流排队异步操作的 API 在图中不受支持（例如 `hggcStreamAttachMemAsync()`），在捕获期间调用它们将返回错误。

!!! note

    当依赖关系试图将已捕获的操作与未捕获的实时执行操作相连接时，HGGC 优先返回错误而非静默忽略该依赖。流进入或退出捕获模式是唯一的例外——模式转换会自动切断转换前后操作之间的依赖链。

**捕获失效与恢复**

若在捕获期间触发了上述任何非法操作，整个捕获图将被标记为 *失效*（Invalidated）。失效后，对该捕获图关联的流或事件的任何后续操作都会返回错误。唯一的恢复路径是调用 `hggcStreamEndCapture()`——它会将所有流移出捕获模式，但返回错误码和 NULL 图指针。应用程序可在此之后重新开始捕获。

###### 4.1.2.2.3. 捕获自省 {#4122-3-捕获自省}

可以使用 `hggcStreamGetCaptureInfo()` 检查活动的流捕获操作。这允许开发者获取捕获的状态、捕获的唯一（进程内）ID、底层的图对象，以及流中下一个即将被捕获节点的依赖关系/边数据。此依赖信息可用于获取流中最后被捕获节点（或多个节点）的句柄。

#### 4.1.3. 图的生命周期管理 {#413-图的生命周期管理}

##### 4.1.3.1. 实例化与执行 {#4131-实例化与执行}


一旦通过图 API 或流捕获创建了图，必须对图进行实例化以创建可执行图，然后才能启动它。假设 `hggcGraph_t graph` 已成功创建，以下代码将实例化该图并创建可执行图 `hggcGraphExec_t graphExec`：

```cpp
hggcGraphExec_t graphExec;
hggcGraphInstantiate(&graphExec, graph, NULL, NULL, 0);
```


在创建图并实例化为可执行图后，即可启动它。假设 `hggcGraphExec_t graphExec` 已成功创建，以下代码片段将把该图启动到指定的流中：

```cpp
hggcGraphLaunch(graphExec, stream);
```

综合以上内容，并使用 4.1.2.2 节中的流捕获示例，以下代码片段将完成图的创建、实例化和启动：


```cpp
hggcGraph_t graph;

hggcStreamBeginCapture(stream);

init_kernel<<< ..., stream >>>(...);
compute_kernel<<< ..., stream >>>(...);
libraryProcess(stream);
reduce_kernel<<< ..., stream >>>(...);

hggcStreamEndCapture(stream, &graph);

hggcGraphExec_t graphExec;
hggcGraphInstantiate(&graphExec, graph, NULL, NULL, 0);
hggcGraphLaunch(graphExec, stream);
```

##### 4.1.3.2. 参数刷新 {#4132-参数刷新}


**问题**：许多实际工作负载中，图的拓扑在多次迭代间保持稳定，但节点携带的参数（核函数入参、缓冲区地址、传输大小等）逐次变化。如果每次参数变化都销毁旧图并重新实例化，实例化本身的开销会抵消图执行带来的收益。

**解决方案——图更新机制**：HGGC 允许对已实例化的图进行原地参数刷新，而无需重建拓扑或重新执行优化。只要拓扑结构未发生变化，此路径比完整重新实例化代价低得多。当拓扑确实发生了变化（新增/删除节点、改变依赖关系），则必须重新实例化。

更新在图的下一次启动时生效，不影响正在执行的先前启动。图可在同一个流中反复更新并重新启动。

HGGC 为参数刷新提供了两条路径：

| 路径         | 适用场景                                   | 说明                                                                         |
| ------------ | ------------------------------------------ | ---------------------------------------------------------------------------- |
| 全图更新     | 需要刷新大量节点，或调用方不持有各节点句柄 | 提交一个拓扑相同但参数不同的 `hggcGraph_t`，运行时自动逐节点匹配并写入新参数 |
| 单个节点更新 | 变化节点数量少，调用方持有目标节点句柄     | 通过专用 API 直接设置各节点参数，跳过全图比对，效率更高                      |

此外，HGGC 还支持在不修改参数的情况下启用或禁用单个节点。

以下章节详细解释了每种方法。

**全图更新**

`hggcGraphExecUpdate()` 提供了一种就地刷新已实例化图参数的方式：传入一个拓扑完全相同的新 `hggcGraph_t`（称为"更新图"），运行时会将更新图中各节点的参数（核函数指针、内存地址等）写入对应的已实例化节点。要求：更新图的拓扑结构与原始图完全一致，且依赖关系的指定顺序、汇节点（Sink nodes）的排列顺序均必须匹配。

`hggcGraphExecUpdate()` 通过确定性的节点匹配算法将更新图中的每个节点与原始图中的对应节点配对。为保证匹配成功，构建更新图时必须遵守以下约束：

| 约束维度       | 要求                                                                                                                                                                       | 违反后果                                       |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| 捕获流操作顺序 | 同一捕获流上的所有 API 调用（含事件等待等非节点操作）必须与原始图构建时的顺序一致                                                                                          | 节点匹配失败，`hggcGraphExecUpdate()` 返回错误 |
| 入边指定顺序   | 操作节点入边的 API（捕获流 API、节点/边添加移除）调用顺序和依赖数组内的元素顺序必须一致                                                                                    | 节点匹配失败                                   |
| 汇节点顺序     | 无输出边的节点（汇节点）集合及其相对顺序必须一致；影响汇节点顺序的操作包括：节点添加、边移除、`hggcStreamUpdateCaptureDependencies()` 移除汇节点、`hggcStreamEndCapture()` | 节点匹配失败                                   |

以下示例展示了如何使用此 API 更新已实例化的图：


```cpp
hggcGraphExec_t runnable = NULL;
hggcStream_t pipeline;
hggcStreamCreate(&pipeline);

for (int step = 0; step < totalSteps; step++) {
    // 捕获本轮迭代的计算图（参数可能随 step 变化）
    hggcGraph_t snapshot;
    hggcStreamBeginCapture(pipeline, hggcStreamCaptureModeGlobal);
    assemble_pipeline_stage(pipeline, step);
    hggcStreamEndCapture(pipeline, &snapshot);

    // 尝试就地刷新参数；首次运行或拓扑变更时回退到完整实例化
    bool needsRebuild = (runnable == NULL);
    if (!needsRebuild) {
        hggcGraphExecUpdateResult status;
        hggcGraphNode_t errNode;
        hggcGraphExecUpdate(runnable, snapshot, &errNode, &status);
        needsRebuild = (status != hggcGraphExecUpdateSuccess);
    }
    if (needsRebuild) {
        if (runnable != NULL) hggcGraphExecDestroy(runnable);
        hggcGraphInstantiate(&runnable, snapshot, NULL, NULL, 0);
    }

    hggcGraphDestroy(snapshot);
    hggcGraphLaunch(runnable, pipeline);
    hggcStreamSynchronize(pipeline);
}
```

上述代码示例展示了推荐的使用模式：首次迭代完成完整实例化，后续迭代优先尝试 `hggcGraphExecUpdate()` 进行原地参数刷新（通过 `updateResult` 判断是否成功），仅在更新失败时回退到销毁并重新实例化。

除全图更新外，还可以先通过 `hggcGraphKernelNodeSetParams()` 等 API 逐个修改 `hggcGraph_t` 中的节点，再调用 `hggcGraphExecUpdate()` 同步到 `hggcGraphExec_t`。不过，对于少量节点的参数变更，下一节介绍的单节点更新 API 效率更高。

图更新操作同时会刷新条件句柄的标志位和默认值。

有关用法和当前限制的更多信息，请参阅 [使用图 API](#417-使用图-api) 文档。

**单个节点更新**

当仅需修改少量节点的参数时，逐节点更新比全图更新更高效——既省去了重新创建 `hggcGraph_t` 的开销，也跳过了实例化步骤。以下 API 可直接修改 `hggcGraphExec_t` 中对应节点的参数：

*表 3：单个节点更新 API*

| API                                      | 节点类型     |
| :--------------------------------------- | :----------- |
| `hggcGraphExecKernelNodeSetParams()`     | 核函数节点   |
| `hggcGraphExecMemcpyNodeSetParams()`     | 内存拷贝节点 |
| `hggcGraphExecMemsetNodeSetParams()`     | 内存设置节点 |
| `hggcGraphExecHostNodeSetParams()`       | 主机节点     |
| `hggcGraphExecChildGraphNodeSetParams()` | 子图节点     |
| `hggcGraphExecEventRecordNodeSetEvent()` | 事件记录节点 |
| `hggcGraphExecEventWaitNodeSetEvent()`   | 事件等待节点 |

有关用法和当前限制的更多信息，请参阅 [使用图 API](#417-使用图-api)。

**单个节点启用**

`hggcGraphNodeSetEnabled()` 允许对已实例化图中的核函数节点、内存设置节点和内存拷贝节点进行动态启用/禁用切换，从而实现一种"超集图"设计模式——构建一个涵盖所有可能操作的图，每次启动前按需激活或关闭特定节点。节点的启用状态可以使用 `hggcGraphNodeGetEnabled()` API 查询。

在重新启用之前，禁用的节点在功能上等同于空节点。节点的参数不会因节点的启用/禁用而受到影响。启用状态不受单个节点更新或使用 `hggcGraphExecUpdate()` 进行的全图更新的影响。在节点禁用期间进行的参数更新将在节点重新启用时生效。

有关用法和当前限制的更多信息，请参阅 [使用图 API](#417-使用图-api)。

##### 4.1.3.3. 参数刷新限制 {#4133-参数刷新限制}


下表汇总了各节点类型在全图更新和单节点更新时的可修改属性与限制：

| 节点类型                               | 可更改属性                 | 不可更改/限制                                                                       |
| -------------------------------------- | -------------------------- | ----------------------------------------------------------------------------------- |
| 核函数节点                             | 核函数参数、启动配置       | 上下文不可变；不能从非动态并行切换为动态并行                                        |
| Memset/Memcpy 节点                     | 1D 操作的源/目标地址和大小 | 设备不可变；源/目标必须来自同一上下文；不支持 2D/3D 节点更新                        |
| Memcpy 节点（额外）                    | —                          | 不支持更改内存类型（`hggcPitchedPtr`、`hggcArray_t`）或传输方向（`hggcMemcpyKind`） |
| 条件节点                               | 默认值、标志               | 句柄创建和赋值顺序必须匹配；不支持更改图数量或上下文；条件体内节点受本表规则约束    |
| 内存节点                               | —                          | 若 `hggcGraph_t` 已实例化为其他 `hggcGraphExec_t`，不能用于更新                     |
| 主机节点 / 事件记录节点 / 事件等待节点 | 所有参数                   | 无限制                                                                              |


#### 4.1.4. 图内存节点 {#414-图内存节点}

在许多典型工作负载中，图的每次执行都需要临时分配一块设备内存用于中间计算，执行完毕后即可释放。如果每次启动前都由主机侧调用分配/释放 API，不仅引入额外的主机-设备同步开销，还无法利用跨次启动之间的物理内存复用机会。图内存节点解决了这一问题——它将内存分配和释放操作直接编码为图中的节点，由运行时按照 PPU 有序生命周期语义自动管理分配时机和物理内存复用。

图分配的虚拟地址在分配节点的整个生命周期（包括重复的实例化和启动）内保持不变，图内其他节点可直接引用该地址而无需图更新，即使运行时更改了底层的物理内存映射也是如此。在同一个图中，生命周期不重叠的图分配可以共享同一块物理内存，运行时通过虚拟别名（Virtual aliasing）透明地实现这一优化。当多个图被启动到同一流中时，运行时同样可以对不同图中生命周期不重叠的分配进行物理内存共享。图内存节点的语义与 `hggcMallocAsync` / `hggcFreeAsync` 一致，捕获这些流有序 API 调用可自动生成对应的图内存节点。

##### 4.1.4.1. API 使用方式 {#4141-api-使用方式}

图内存节点分为两类：

| 节点类型     | 作用                                                                 | 创建时机   |
| ------------ | -------------------------------------------------------------------- | ---------- |
| **分配节点** | 创建一块图分配（Graph allocation），运行时在节点创建时即分配虚拟地址 | 图构建阶段 |
| **释放节点** | 释放先前由分配节点创建的图分配                                       | 图构建阶段 |

虚拟地址在分配节点存续期间保持不变，但内存内容在释放后不保证持久，底层物理内存可能被运行时重新映射给其他分配。

**图分配的生命周期**遵循 PPU 有序语义——它在 PPU 实际执行到分配节点时开始（而非 API 调用时），在以下任一事件发生时结束：

  *PPU 执行到达图中的释放节点；
  *PPU 执行到达流中的 `hggcFreeAsync()` 调用；
  * 主机侧调用 `hggcFree()` 立即释放。

!!! note

    图的销毁不会自动释放任何存活的图分配内存，即使它结束了分配节点的生命周期。该分配必须随后在另一个图中被释放，或者使用 `hggcFreeAsync()` / `hggcFree()` 进行释放。

程序必须通过依赖边保证所有访问图分配的操作都位于分配节点之后、释放操作之前。由于生命周期的起止由 PPU 实际执行顺序决定（而非工作入队顺序），图分配被称为"PPU 有序的"。

###### 4.1.4.1.1. 图节点 API {#41411-图节点-api}

通过 `hggcGraphAddNode` 可以显式创建图内存节点。添加 `hggcGraphNodeTypeMemAlloc` 类型的节点时，运行时分配的虚拟地址会写入 `hggcGraphNodeParams` 结构的 `alloc::dptr` 字段，供后续节点引用。释放节点使用 `hggcGraphNodeTypeMemFree` 类型创建，且必须依赖于所有使用该分配的节点。

以下代码片段展示了如何在图中创建分配节点、多个使用该分配的核函数节点，以及最终的释放节点：

```cpp
hggcGraphCreate(&graph, 0);

// 配置分配参数：在设备 0 上分配锁页内存
hggcGraphNodeParams allocParams = { hggcGraphNodeTypeMemAlloc };
allocParams.alloc.poolProps.allocType = hggcMemAllocationTypePinned;
allocParams.alloc.poolProps.location.type = hggcMemLocationTypeDevice;
allocParams.alloc.poolProps.location.id = 0;
allocParams.alloc.bytesize = bufferSize;

hggcGraphAddNode(&allocNode, graph, NULL, 0, &allocParams);

// 创建使用该分配的核函数节点
hggcGraphNodeParams kParams = { hggcGraphNodeTypeKernel };
kParams.kernel.kernelParams[0] = allocParams.alloc.dptr;
// ...设置其他核函数参数...

// 节点 procA 依赖分配节点，procB 和 procC 分别依赖 procA（形成扇出）
hggcGraphAddNode(&procA, graph, &allocNode, 1, &kParams);
hggcGraphAddNode(&procB, graph, &procA, 1, &kParams);
hggcGraphAddNode(&procC, graph, &procA, 1, &kParams);

// 释放节点必须依赖所有访问该分配的节点（procB 和 procC）
// procB 对 procA 的依赖已间接建立，因此释放节点无需显式依赖 procA
hggcGraphNode_t freeDeps[2] = { procB, procC };
hggcGraphNodeParams freeParams = { hggcGraphNodeTypeMemFree };
freeParams.free.dptr = allocParams.alloc.dptr;
hggcGraphAddNode(&freeNode, graph, freeDeps, 2, &freeParams);

// procD 不依赖释放节点，因此不得访问已释放的图分配
hggcGraphAddNode(&procD, graph, &procC, 1, &kParams);

// procE 不依赖分配节点，因此不得访问该图分配（即使释放节点依赖 procE 也是如此）
hggcGraphAddNode(&procE, graph, NULL, 0, &kParams);
```

###### 4.1.4.1.2. 流捕获 {#41412-流捕获}

另一种创建图内存节点的方式是通过流捕获：在捕获模式下调用 `hggcMallocAsync` 和 `hggcFreeAsync`，运行时会自动将它们转换为图中的分配/释放节点。被捕获的分配返回的虚拟地址可直接在后续捕获操作中使用。由于流有序依赖关系会被一并捕获，只要原始流代码的排序正确，生成的图内存节点也会具有正确的依赖关系。

以下代码片段展示了通过流捕获方式构建与上述等价的图结构（省略了 procD 和 procE）：

```cpp
hggcMallocAsync(&buf, bufferSize, captureStream);
process_A<<< ..., captureStream >>>(buf, ...);

// 分叉到辅助流
hggcEventRecord(splitEvent, captureStream);
hggcStreamWaitEvent(auxStream, splitEvent);

process_B<<< ..., captureStream >>>(buf, ...);
// 事件依赖自动转换为图依赖，process_C 对分配节点的依赖由此建立
process_C<<< ..., auxStream >>>(buf, ...);

// 辅助流汇合回源流
hggcEventRecord(mergeEvent, auxStream);
hggcStreamWaitEvent(captureStream, mergeEvent);

// 释放操作依赖于所有访问该内存的工作
hggcFreeAsync(buf, captureStream);

// 在源流上结束捕获
hggcStreamEndCapture(captureStream, &graph);
```

###### 4.1.4.1.3. 在分配图之外访问和释放图内存 {#41413-在分配图之外访问和释放图内存}

图分配不一定要由分配它的那个图来释放。当一个图没有释放某个分配时，该分配在图执行结束后依然存续，并可被后续的 HGGC 操作访问。只要访问操作通过 HGGC 事件或其他流排序机制被置于分配之后，这些分配就可以在另一个图中被访问，或直接通过流操作访问。分配随后可以通过常规的 `hggcFree` 或 `hggcFreeAsync` 调用来释放，或者通过启动另一个包含相应释放节点的图来释放，也可以通过该分配图的后续启动来释放（如果实例化时设置了 `hggcGraphInstantiateFlagAutoFreeOnLaunch` 标志）。在内存被释放后访问它是非法的，释放操作必须通过图依赖、HGGC 事件和其他流排序机制，被置于所有访问该内存的操作之后。

!!! note

    由于图分配可能会共享底层物理内存，因此释放操作必须在所有设备操作完成后执行。带外同步（例如计算核函数内的基于内存的同步）对于内存写入和释放操作之间的排序是不够的。有关更多信息，请参阅与一致性和连贯性相关的 [虚拟别名支持](#46342-虚拟别名支持) 规则。

以下三个代码片段演示了在分配图之外访问图分配，并通过以下方式建立正确的排序：使用单个流、使用跨流事件，以及使用嵌入在分配图和释放图中的事件。

**方式一：单流排序。** 同一流上的操作天然有序，无需额外同步。

```cpp
// 分配图：创建图分配节点
void *workspace;
hggcGraphNodeParams ap = { hggcGraphNodeTypeMemAlloc };
ap.alloc.poolProps.allocType = hggcMemAllocationTypePinned;
ap.alloc.poolProps.location.type = hggcMemLocationTypeDevice;
ap.alloc.bytesize = workspaceSize;
hggcGraphAddNode(&allocNode, allocGraph, NULL, 0, &ap);
workspace = ap.alloc.dptr;

hggcGraphInstantiate(&allocExec, allocGraph, NULL, NULL, 0);

// 在同一流上依次启动分配图、使用内存的核函数、释放操作
hggcGraphLaunch(allocExec, taskStream);
compute_kernel<<< ..., taskStream >>>(workspace, ...);
hggcFreeAsync(workspace, taskStream);
```

**方式二：跨流事件排序。** 分配图和消费者位于不同流时，通过事件建立依赖。

```cpp
// 分配图
void *workspace;
hggcGraphAddNode(&allocNode, allocGraph, NULL, 0, &allocParams);
workspace = allocParams.alloc.dptr;

// 消费/释放图
// 注意：此处释放节点在图内无显式依赖（NULL），其执行顺序通过下方的跨流事件同步保证。
// 在实际使用中，释放节点应依赖所有使用该内存分配的节点，确保释放操作在所有使用完成后执行。
consumeParams.kernel.kernelParams[0] = allocParams.alloc.dptr;
hggcGraphAddNode(&freeNode, freeGraph, NULL, 1, &workspace);

hggcGraphInstantiate(&allocExec, allocGraph, NULL, NULL, 0);
hggcGraphInstantiate(&freeExec, freeGraph, NULL, NULL, 0);

hggcGraphLaunch(allocExec, producerStream);

// 通过事件让消费者流等待分配完成
hggcEventRecord(readyEvent, producerStream);
hggcStreamWaitEvent(consumerStream, readyEvent);

compute_kernel<<< ..., consumerStream >>>(workspace, ...);

// 通过事件让释放图等待消费者完成
hggcEventRecord(doneEvent, consumerStream);
hggcStreamWaitEvent(cleanupStream, doneEvent);

// 消费者完成后启动释放图
hggcGraphLaunch(freeExec, cleanupStream);
```

第三，通过使用图外部事件节点来建立排序：


```cpp
// ── 分配图：创建内存并记录就绪事件 ──
void *workspace;
hggcEvent_t allocReadyEvt;    // 分配完成后触发，通知消费方可安全访问
hggcEvent_t computeDoneEvt;   // 外部流计算结束后触发，通知释放图可回收内存

// 向分配图添加分配节点，并获取运行时分配的虚拟地址
hggcGraphAddNode(&allocNode, allocGraph, NULL, 0, &allocNodeParams);
workspace = allocNodeParams.alloc.dptr;
// 事件记录节点依赖分配节点，确保分配完成后才发出就绪信号

hggcGraphNodeParams allocEvtNodeParams = { hggcGraphNodeTypeEventRecord };
allocEvtNodeParams.eventRecord.event = allocReadyEvt;
hggcGraphAddNode(&recordNode, allocGraph, &allocNode, 1, &allocEvtNodeParams);
hggcGraphInstantiate(&allocGraphExec, allocGraph, NULL, NULL, 0);

// ── 释放图：等待就绪事件和计算完成事件后释放内存 ──
hggcGraphNodeParams computeDoneWaitParams = { hggcGraphNodeTypeEventWait };
computeDoneWaitParams.eventWait.event = computeDoneEvt;
hggcGraphAddNode(&computeDoneEventNode, cleanupGraph, NULL, 0, &computeDoneWaitParams);

hggcGraphNodeParams allocReadyWaitParams = { hggcGraphNodeTypeEventWait };
allocReadyWaitParams.eventWait.event = allocReadyEvt;
hggcGraphAddNode(&allocReadyEventNode, cleanupGraph, NULL, 0, &allocReadyWaitParams);

kernelNodeParams->kernelParams[0] = allocNodeParams.alloc.dptr;

// 核函数节点依赖就绪事件节点，保证分配已可用
hggcGraphAddNode(&kernelNode, cleanupGraph, &allocReadyEventNode, 1, &kernelNodeParams);

// 释放节点须同时依赖核函数节点和外部计算完成事件节点，
// 确保所有使用者（图内和图外）都已结束对该分配的访问
dependencies[0] = kernelNode;
dependencies[1] = computeDoneEventNode;

hggcGraphNodeParams freeNodeParams = { hggcGraphNodeTypeMemFree };
freeNodeParams.free.dptr = workspace;
hggcGraphAddNode(&freeNode, cleanupGraph, &dependencies, 2, &freeNodeParams);
hggcGraphInstantiate(&cleanupGraphExec, cleanupGraph, NULL, NULL, 0);

hggcGraphLaunch(allocGraphExec, allocStream);

// 外部流通过等待就绪事件建立对分配图的依赖
hggcStreamWaitEvent(computeStream, allocReadyEvt);
kernel<<< ..., computeStream >>> (workspace, ...);
hggcEventRecord(computeDoneEvt, computeStream);

// 释放图中的事件等待节点确保回收操作不会与外部流的计算并发执行
hggcGraphLaunch(cleanupGraphExec, releaseStream);
```

###### 4.1.4.1.4. 子图中的内存节点 {#41414-子图中的内存节点}

HGGC 12.9 引入了将子图所有权转移到父图的功能。被转移到父图的子图允许包含内存分配和释放节点。这使得包含分配或释放节点的子图可以在添加到父图之前被独立构建。

子图在被转移后适用以下限制：

  * 不能被独立实例化或销毁。
  * 不能作为另一个独立父图的子图添加。
  * 不能用作 `hggcGraphExecUpdate` 的参数。
  * 不能添加额外的内存分配或释放节点。





```cpp
// 创建子图
hggcGraphCreate(&child, 0);

// 配置分配参数：在设备 0 上分配锁页内存
hggcGraphNodeParams allocNodeParams = { hggcGraphNodeTypeMemAlloc };
allocNodeParams.alloc.poolProps.allocType = hggcMemAllocationTypePinned;
allocNodeParams.alloc.poolProps.location.type = hggcMemLocationTypeDevice;
// 指定驻留设备为设备 0
allocNodeParams.alloc.poolProps.location.id = 0;
allocNodeParams.alloc.bytesize = size;

hggcGraphAddNode(&allocNode, graph, NULL, 0, &allocNodeParams);
// 此处可插入使用该分配的其他节点
hggcGraphNodeParams freeNodeParams = { hggcGraphNodeTypeMemFree };
freeNodeParams.free.dptr = allocNodeParams.alloc.dptr;
hggcGraphAddNode(&freeNode, graph, &allocNode, 1, &freeNodeParams);

// 创建父图
hggcGraphCreate(&parent, 0);

// 将子图所有权转移给父图
hggcGraphNodeParams childNodeParams = { hggcGraphNodeTypeGraph };
childNodeParams.graph.graph = child;
childNodeParams.graph.ownership = hggcGraphChildGraphOwnershipMove;
hggcGraphAddNode(&parentNode, parent, NULL, 0, &childNodeParams);
```

##### 4.1.4.2. 资源管理与性能优化 {#4142-资源管理与性能优化}

###### 4.1.4.2.1. 内存重用机制 {#41421-内存重用机制}

HGGC 通过两种方式重用内存：

  * 图内的虚拟和物理内存重用基于虚拟地址分配，类似于流有序分配器。
  * 图之间的物理内存重用通过虚拟别名（Virtual aliasing）实现：不同的图可以将相同的物理内存映射到它们各自唯一的虚拟地址。

**图内的地址重用**

HGGC 可能会通过将相同的虚拟地址范围分配给生命周期不重叠的不同分配来重用图内的内存。由于虚拟地址可能被重用，因此不同生命周期且不相交的分配指针不能保证是唯一的。

**物理内存管理与共享**

在 PPU 执行顺序中，HGGC 负责在分配节点到达之前将物理内存映射到虚拟地址。作为对内存占用和映射开销的优化，如果多个图不会同时运行，它们可以使用相同的物理内存进行不同的分配；但是，如果物理页面同时绑定到超过一个正在执行的图，或者绑定到尚未释放的图分配，则不能重用这些页面。

HGGC 可能在图实例化、启动或执行期间的任何时间更新物理内存映射。HGGC 还可能在未来的图启动之间引入同步，以防止存活的图分配引用相同的物理内存。对于任何"分配-释放-分配"模式，如果程序在分配的生命周期之外访问指针，该错误访问可能会静默地读取或写入属于另一个分配的存活数据（即使该分配的虚拟地址是唯一的）。使用 [计算消毒工具（Compute sanitizer tools）](/docs_center/doc_detail/index.html?projectId=39&chapterId=1065) 可以捕获此错误。

###### 4.1.4.2.2. 性能调优 {#41422-性能调优}

当多个图被启动到同一个流中时，HGGC 会尝试为它们分配相同的物理内存，因为这些图的执行无法重叠。图的物理映射会在启动之间保留，这是一种避免重映射成本的优化。如果后续其中一个图的启动方式导致其执行可能与其他图重叠（例如启动到不同的流中），则 HGGC 必须执行一些重映射，因为并发图需要不同的内存以避免数据损坏。

通常，HGGC 中的图内存重映射很可能是由以下操作引起的：

  * 更改图启动时所在的流；
  * 对图内存池执行修剪（Trim）操作，该操作会显式释放未使用的内存（参见 [4.2.6.5 节](#4265-cpu-虚拟内存)）；
  * 在来自另一个图的未释放分配映射到相同内存时重新启动某个图，会导致在重新启动前进行内存重映射。

重映射必须按执行顺序进行，但必须在该图的任何先前执行完成后执行（否则仍在使用中的内存可能会被取消映射）。由于这种排序依赖性，以及映射操作是操作系统调用，因此映射操作可能相对昂贵。应用程序可以通过始终将包含分配内存节点的图启动到同一个流中来避免此成本。

**首次启动 / `hggcGraphUpload`**

物理内存不能在图实例化期间分配或映射，因为此时图将在哪个流中执行是未知的。映射是在图启动期间完成的。调用 `hggcGraphUpload` 可以通过立即执行该图的所有映射并将图与上传流相关联，将分配成本从启动阶段中分离出来。如果随后将该图启动到相同的流中，它将无需任何额外的重映射即可启动。

使用不同的流进行图上传和图启动，其表现与切换流类似，很可能导致重映射操作。此外，不相关的内存池管理允许从空闲流中提取内存，这可能会抵消上传带来的影响。

###### 4.1.4.2.3. 物理内存占用管理 {#41423-物理内存占用管理}

异步分配的池管理行为意味着销毁包含内存节点的图（即使它们的分配已释放）不会立即将物理内存返回给操作系统供其他进程使用。为了显式将内存释放回给操作系统，应用程序应使用 `hggcDeviceGraphMemTrim` API。

`hggcDeviceGraphMemTrim` 将取消映射并释放图内存节点保留的任何未被主动使用的物理内存。未释放的分配以及处于调度或运行状态的图被视为正在主动使用物理内存，因此不会受到影响。使用修剪 API 将使物理内存可供其他分配 API 和其他应用程序或进程使用，但会导致 HGGC 在修剪后的图下次启动时重新分配和重映射内存。注意 `hggcDeviceGraphMemTrim` 操作的池与 `hggcMemPoolTrimTo()` 不同。图内存池不会暴露给流有序内存分配器。HGGC 允许应用程序通过 `hggcDeviceGetGraphMemAttribute` API 查询其图内存占用情况。查询 `hggcGraphMemAttrReservedMemCurrent` 属性会返回驱动程序为当前进程中的图分配保留的物理内存量。查询 `hggcGraphMemAttrUsedMemCurrent` 会返回当前至少被一个图映射的物理内存量。这两个属性中的任何一个都可以用于追踪 HGGC 何时为了分配图而获取了新的物理内存。这两个属性对于检查共享机制节省了多少内存非常有用。

##### 4.1.4.3. 多设备内存访问 {#4143-多设备内存访问}

图分配可以配置为允许从多个 PPU 访问，在这种情况下，HGGC 将根据需要将分配映射到对等 PPU 上。HGGC 允许需要不同映射的图分配重用相同的虚拟地址。当发生这种情况时，地址范围会被映射到不同分配所需的所有 PPU 上。这意味着分配有时可能会允许比其创建时请求的更多的对等访问；然而，依赖这些额外的映射仍然是错误的。

###### 4.1.4.3.1. 使用图节点 API 的对等访问 {#41431-使用图节点-api-的对等访问}

`hggcGraphAddNode` API 接受分配节点参数结构中 `accessDescs` 数组字段里的映射请求。嵌入式结构 `poolProps.location` 指定了分配的驻留设备。假定需要来自驻留 PPU 的访问，因此应用程序不需要在 `accessDescs` 数组中为驻留设备指定条目。


```cpp
hggcGraphNodeParams allocNodeParams = { hggcGraphNodeTypeMemAlloc };
allocNodeParams.alloc.poolProps.allocType = hggcMemAllocationTypePinned;
allocNodeParams.alloc.poolProps.location.type = hggcMemLocationTypeDevice;
// 指定设备 1 为分配的驻留设备
allocNodeParams.alloc.poolProps.location.id = 1;
allocNodeParams.alloc.bytesize = size;

// 创建驻留在设备 1 上、仅设备 1 可访问的分配
hggcGraphAddNode(&allocNode, graph, NULL, 0, &allocNodeParams);

accessDescs[2];
// 初始化访问描述符（图节点 API 仅支持 ReadWrite 与 Device 类型）
accessDescs[0].flags = hggcMemAccessFlagsProtReadWrite;
accessDescs[0].location.type = hggcMemLocationTypeDevice;
accessDescs[1].flags = hggcMemAccessFlagsProtReadWrite;
accessDescs[1].location.type = hggcMemLocationTypeDevice;

// 请求设备 0 和设备 2 的访问权限；设备 1 作为驻留设备隐式具备访问权限
accessDescs[0].location.id = 0;
accessDescs[1].location.id = 2;

// 访问描述符数组包含 2 个条目
allocNodeParams.accessDescCount = 2;
allocNodeParams.accessDescs = accessDescs;

// 创建驻留在设备 1、设备 0/1/2 均可访问的分配（0 和 2 来自描述符，1 为驻留设备）
hggcGraphAddNode(&allocNode, graph, NULL, 0, &allocNodeParams);
```

###### 4.1.4.3.2. 使用流捕获的对等访问 {#41432-使用流捕获的对等访问}

对于流捕获，分配节点会在捕获时记录分配池的对等可访问性。在捕获 `hggcMallocFromPoolAsync` 调用之后更改分配池的对等可访问性，不会影响图为该分配所做的映射。


```cpp
// 初始化访问描述符（图节点 API 仅支持 ReadWrite 与 Device 类型）
accessDesc.flags = hggcMemAccessFlagsProtReadWrite;
accessDesc.location.type = hggcMemLocationTypeDevice;
accessDesc.location.id = 1;

// 假设 memPool 驻留在设备 0 且初始仅设备 0 可访问

hggcStreamBeginCapture(stream);
hggcMallocAsync(&allocPtr1, size, memPool, stream);
hggcStreamEndCapture(stream, &graph1);

hggcMemPoolSetAccess(memPool, &accessDesc, 1);

hggcStreamBeginCapture(stream);
hggcMallocAsync(&allocPtr2, size, memPool, stream);
hggcStreamEndCapture(stream, &graph2);

// allocPtr1 对应的图节点仅具有设备 0 的可访问性，即使 memPool 此时已具有设备 1 可访问性。
// allocPtr2 对应的图节点将具有设备 0 和设备 1 的可访问性，因为捕获时池已具备该权限。
```

#### 4.1.5. 条件执行 {#415-条件执行}

!!! note

    真武 810 / 真武 610 / 真武 810E / 真武 610E 及 真武 M890 产品在硬件上暂不支持条件执行的能力，因此在图执行的主机端 API 中需采用软件方式进行模拟：即对当前流执行同步操作，等待所有前序依赖任务执行完毕后，再继续后续流程。

在许多实际场景中，任务编排需要根据运行时状态进行分支选择或循环迭代——例如迭代直到收敛、根据中间结果选择不同的处理路径等。传统方式需要在每个判断点返回主机侧做决策，再提交后续工作，这会引入主机-设备之间的往返延迟。条件图节点将这类控制流逻辑直接内嵌到图结构中，使分支和循环完全在设备侧完成评估与调度，主机 CPU 无需参与中间决策。

HGGC 提供三种条件节点类型，覆盖常见的控制流模式：

| 类型       | 触发条件         | 执行行为                                                              | 主体图数量     |
| ---------- | ---------------- | --------------------------------------------------------------------- | -------------- |
| **IF**     | 条件值非零时触发 | 执行主体图一次；可选提供第二个主体图（else 分支），在条件值为零时执行 | 1 或 2         |
| **WHILE**  | 条件值非零时触发 | 反复执行主体图，每轮结束后重新评估条件值，直到条件值变为零            | 1              |
| **SWITCH** | 条件值等于索引 n | 执行索引为 n 的主体图一次；若无匹配索引则不执行任何主体图             | 用户指定（≥1） |

条件值的读写通过条件句柄（Conditional handle）进行，设备代码中使用 `hggcGraphSetConditional()` 写入条件值。创建条件节点时，运行时会为每个主体图槽位生成一个空图并返回句柄，开发者可使用图 API 或 `hggcStreamBeginCaptureToGraph()` 向其中填充节点。条件节点支持嵌套。

##### 4.1.5.1. 条件句柄 {#4151-条件句柄}

条件句柄的类型为 `hggcGraphConditionalHandle`，通过 `hggcGraphConditionalHandleCreate()` 创建。该句柄必须与单个条件节点关联。句柄无法销毁，因此无需跟踪它们。

关于条件值的初始化，有两种策略：

| 策略           | 做法                                                                                                  | 适用场景                   |
| -------------- | ----------------------------------------------------------------------------------------------------- | -------------------------- |
| 指定默认值     | 创建时传入 `hggcGraphCondAssignDefault` 标志和默认值，运行时在每次图启动时自动初始化                  | 条件值在图启动前即可确定   |
| 上游核函数赋值 | 不指定默认值，条件值在每次执行开始时处于未定义状态，由上游核函数通过 `hggcGraphSetConditional()` 写入 | 条件值需要在设备侧动态计算 |

句柄关联的默认值和标志会随全图更新一同刷新。

##### 4.1.5.2. 条件节点主体图要求 {#4152-条件节点主体图要求}

主体图的内容受到以下约束，违反约束将导致实例化失败：

| 约束类别           | 具体要求                                                                                                                                                                        |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 设备一致性         | 所有节点必须驻留在同一设备上                                                                                                                                                    |
| 允许的节点类型     | 仅限核函数节点、空节点、Memcpy 节点、Memset 节点、子图节点和条件节点                                                                                                            |
| 核函数节点限制     | 不允许使用 HGGC 动态并行（Dynamic Parallelism）或设备图启动（Device Graph Launch）；在非 MPS 环境下允许协作启动（Cooperative launches）                                         |
| Memcpy/Memset 限制 | 仅允许涉及设备内存和/或锁页（pinned）设备映射主机内存的操作；不允许涉及 HGGC 数组；实例化时两个操作数都必须能从当前设备访问（即使目标内存位于其他设备，拷贝仍由图所在设备执行） |

##### 4.1.5.3. 条件节点类型及示例 {#4153-条件节点类型及示例}

**条件 IF 节点**

IF 节点在条件值为非零时执行主体图一次。典型用途是根据某个运行时判据决定是否执行一组额外的计算步骤。

下面的示例构建了一个包含 IF 条件节点的图。条件值未指定默认值，而是由上游核函数在设备侧动态写入。主体图通过图 API 填充。

```cpp
// PPU 推理场景：根据 batch 大小动态选择执行路径
__global__ void checkBatchSize(hggcGraphConditionalHandle cond,
                                int *batchSize, int threshold)
{
    // 当 batch 大于阈值时启用大 batch 优化路径
    hggcGraphSetConditional(cond, (*batchSize > threshold) ? 1 : 0);
}

void buildBatchAdaptiveGraph(int threshold) {
    hggcGraph_t topGraph;
    hggcGraphExec_t executable;
    hggcGraphNode_t prevNode;

    hggcGraphCreate(&topGraph, 0);

    // 条件句柄：由检查核函数动态决定
    hggcGraphConditionalHandle cond;
    hggcGraphConditionalHandleCreate(&cond, topGraph);

    // 检查 batch 大小的核函数
    hggcGraphNodeParams kp = { hggcGraphNodeTypeKernel };
    kp.kernel.func = (void *)checkBatchSize;
    kp.kernel.gridDim  = dim3(1);
    kp.kernel.blockDim = dim3(1);
    void *args[] = { &cond, &d_batchSize, &threshold };
    kp.kernel.kernelParams = args;
    hggcGraphAddNode(&prevNode, topGraph, NULL, 0, &kp);

    // IF 条件节点：满足条件时执行大 batch 优化核函数
    hggcGraphNodeParams cp = { hggcGraphNodeTypeConditional };
    cp.conditional.handle = cond;
    cp.conditional.type   = hggcGraphCondTypeIf;
    cp.conditional.size   = 1;
    hggcGraphAddNode(&prevNode, topGraph, &prevNode, 1, &cp);

    // 大 batch 优化路径：使用 tiling 策略提升吞吐
    hggcGraph_t optimizedPath = cp.conditional.phGraph_out[0];
    hggcGraphNodeParams optKp = { hggcGraphNodeTypeKernel };
    optKp.kernel.func = (void *)largeBatchInferenceKernel;
    optKp.kernel.gridDim  = dim3(largeGrid);
    optKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, optimizedPath, NULL, 0, &optKp);

    hggcGraphInstantiate(&executable, topGraph, NULL, NULL, 0);
    hggcGraphLaunch(executable, 0);
    hggcDeviceSynchronize();

    hggcGraphExecDestroy(executable);
    hggcGraphDestroy(topGraph);
}
```


IF 节点还支持 else 分支：将 `size` 设为 2 即可获得第二个主体图，条件值为零时执行该分支。

```cpp
// 根据输入规模选择不同精度（FP16/FP32）计算路径
void buildPrecisionSelectGraph() {
    hggcGraph_t topGraph;
    hggcGraphExec_t executable;
    hggcGraphNode_t prevNode;

    hggcGraphCreate(&topGraph, 0);

    hggcGraphConditionalHandle cond;
    hggcGraphConditionalHandleCreate(&cond, topGraph);

    // 上游核函数：根据输入规模决定精度模式
    // 大规模输入使用 FP16 加速，小规模使用 FP32 保精度
    hggcGraphNodeParams kp = { hggcGraphNodeTypeKernel };
    kp.kernel.func = (void *)selectPrecisionKernel;
    kp.kernel.gridDim  = dim3(1);
    kp.kernel.blockDim = dim3(1);
    void *args[] = { &cond, &d_inputSize, &precisionThreshold };
    kp.kernel.kernelParams = args;
    hggcGraphAddNode(&prevNode, topGraph, NULL, 0, &kp);

    // IF-ELSE 条件节点：size=2 表示包含 if 和 else 分支
    hggcGraphNodeParams cp = { hggcGraphNodeTypeConditional };
    cp.conditional.handle = cond;
    cp.conditional.type   = hggcGraphCondTypeIf;
    cp.conditional.size   = 2;
    hggcGraphAddNode(&prevNode, topGraph, &prevNode, 1, &cp);

    // IF 分支（条件为真）：FP16 快速推理
    hggcGraph_t fp16Branch = cp.conditional.phGraph_out[0];
    hggcGraphNodeParams fp16Kp = { hggcGraphNodeTypeKernel };
    fp16Kp.kernel.func = (void *)fp16InferenceKernel;
    fp16Kp.kernel.gridDim  = dim3(fp16Grid);
    fp16Kp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, fp16Branch, NULL, 0, &fp16Kp);

    // ELSE 分支（条件为假）：FP32 高精度推理
    hggcGraph_t fp32Branch = cp.conditional.phGraph_out[1];
    hggcGraphNodeParams fp32Kp = { hggcGraphNodeTypeKernel };
    fp32Kp.kernel.func = (void *)fp32InferenceKernel;
    fp32Kp.kernel.gridDim  = dim3(fp32Grid);
    fp32Kp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, fp32Branch, NULL, 0, &fp32Kp);

    hggcGraphInstantiate(&executable, topGraph, NULL, NULL, 0);
    hggcGraphLaunch(executable, 0);
    hggcDeviceSynchronize();

    hggcGraphExecDestroy(executable);
    hggcGraphDestroy(topGraph);
}
```


**条件 WHILE 节点**

WHILE 节点在条件值为非零的情况下反复执行主体图。运行时在进入循环前和每轮主体图执行完毕后都会重新评估条件值。

以下示例使用 `hggcGraphCondAssignDefault` 将条件默认值设为 1（即默认进入循环），循环体内的核函数在每轮迭代中执行共轭梯度迭代步，当残差范数小于收敛阈值时将条件置为 0 以退出循环。

```cpp
// PPU 迭代收敛算法（共轭梯度法）：条件判断收敛阈值
__global__ void conjugateGradientStep(hggcGraphConditionalHandle cond,
                                       float *residual, float *x,
                                       float *p, float *Ap,
                                       float *threshold, int n)
{
    // 执行一步共轭梯度迭代
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        // ... 共轭梯度迭代计算 ...
        x[idx] += alpha * p[idx];
        residual[idx] -= alpha * Ap[idx];
    }
    // 线程 0 检查收敛条件
    if (idx == 0) {
        float normR = computeNorm(residual, n);
        if (normR < *threshold) {
            hggcGraphSetConditional(cond, 0);  // 已收敛，退出循环
        }
    }
}

void buildConvergenceGraph(float convergenceThreshold) {
    hggcGraph_t topGraph;
    hggcGraphExec_t executable;
    hggcGraphNode_t loopNode;

    hggcGraphCreate(&topGraph, 0);

    // 条件句柄：默认值 1（进入迭代循环）
    hggcGraphConditionalHandle cond;
    hggcGraphConditionalHandleCreate(&cond, topGraph, 1, hggcGraphCondAssignDefault);

    // WHILE 条件节点
    hggcGraphNodeParams cp = { hggcGraphNodeTypeConditional };
    cp.conditional.handle = cond;
    cp.conditional.type   = hggcGraphCondTypeWhile;
    cp.conditional.size   = 1;
    hggcGraphAddNode(&loopNode, topGraph, NULL, 0, &cp);

    // 循环体：共轭梯度迭代步
    hggcGraph_t loopBody = cp.conditional.phGraph_out[0];
    hggcGraphNodeParams kp = { hggcGraphNodeTypeKernel };
    kp.kernel.func = (void *)conjugateGradientStep;
    kp.kernel.gridDim  = dim3((n + 255) / 256);
    kp.kernel.blockDim = dim3(256);
    void *args[] = { &cond, &d_residual, &d_x, &d_p, &d_Ap, &d_threshold, &n };
    kp.kernel.kernelParams = args;
    hggcGraphAddNode(&loopNode, loopBody, NULL, 0, &kp);

    // 设置收敛阈值
    hggcMemcpy(d_threshold, &convergenceThreshold, sizeof(float), hggcMemcpyHostToDevice);

    hggcGraphInstantiate(&executable, topGraph, NULL, NULL, 0);
    hggcGraphLaunch(executable, 0);
    hggcDeviceSynchronize();

    hggcGraphExecDestroy(executable);
    hggcGraphDestroy(topGraph);
}
```


**条件 SWITCH 节点**

SWITCH 节点根据条件值的大小选择对应索引的主体图执行一次。索引从零开始，若条件值超出已定义的主体图范围，则不执行任何分支。

下面的示例创建了一个包含 5 个分支的 SWITCH 条件节点，条件值由上游核函数动态写入。

```cpp
// PPU 多算子调度器：根据算子类型分派到不同核函数
__global__ void dispatchOperator(hggcGraphConditionalHandle cond, int *opType)
{
    // opType: 0=卷积, 1=矩阵乘, 2=归约, 3=元素级, 4=归一化
    hggcGraphSetConditional(cond, *opType);
}

void buildOperatorDispatchGraph() {
    hggcGraph_t topGraph;
    hggcGraphExec_t executable;
    hggcGraphNode_t prevNode;

    hggcGraphCreate(&topGraph, 0);

    hggcGraphConditionalHandle cond;
    hggcGraphConditionalHandleCreate(&cond, topGraph);

    // 调度核函数：读取算子类型并设置分支索引
    hggcGraphNodeParams kp = { hggcGraphNodeTypeKernel };
    kp.kernel.func = (void *)dispatchOperator;
    kp.kernel.gridDim  = dim3(1);
    kp.kernel.blockDim = dim3(1);
    void *args[] = { &cond, &d_opType };
    kp.kernel.kernelParams = args;
    hggcGraphAddNode(&prevNode, topGraph, NULL, 0, &kp);

    // SWITCH 条件节点：5 种算子类型
    hggcGraphNodeParams cp = { hggcGraphNodeTypeConditional };
    cp.conditional.handle = cond;
    cp.conditional.type   = hggcGraphCondTypeSwitch;
    cp.conditional.size   = 5;
    hggcGraphAddNode(&prevNode, topGraph, &prevNode, 1, &cp);

    // 各分支填充对应的算子核函数
    hggcGraph_t *branches = cp.conditional.phGraph_out;

    // 分支0：卷积算子
    hggcGraphNodeParams convKp = { hggcGraphNodeTypeKernel };
    convKp.kernel.func = (void *)conv2d_kernel;
    convKp.kernel.gridDim  = dim3(convGrid);
    convKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, branches[0], NULL, 0, &convKp);

    // 分支1：矩阵乘算子
    hggcGraphNodeParams matmulKp = { hggcGraphNodeTypeKernel };
    matmulKp.kernel.func = (void *)matmul_kernel;
    matmulKp.kernel.gridDim  = dim3(matmulGrid);
    matmulKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, branches[1], NULL, 0, &matmulKp);

    // 分支2：归约算子
    hggcGraphNodeParams reduceKp = { hggcGraphNodeTypeKernel };
    reduceKp.kernel.func = (void *)reduce_kernel;
    reduceKp.kernel.gridDim  = dim3(reduceGrid);
    reduceKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, branches[2], NULL, 0, &reduceKp);

    // 分支3：元素级算子
    hggcGraphNodeParams elemKp = { hggcGraphNodeTypeKernel };
    elemKp.kernel.func = (void *)elementwise_kernel;
    elemKp.kernel.gridDim  = dim3(elemGrid);
    elemKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, branches[3], NULL, 0, &elemKp);

    // 分支4：归一化算子
    hggcGraphNodeParams normKp = { hggcGraphNodeTypeKernel };
    normKp.kernel.func = (void *)normalize_kernel;
    normKp.kernel.gridDim  = dim3(normGrid);
    normKp.kernel.blockDim = dim3(256);
    hggcGraphAddNode(&prevNode, branches[4], NULL, 0, &normKp);

    hggcGraphInstantiate(&executable, topGraph, NULL, NULL, 0);
    hggcGraphLaunch(executable, 0);
    hggcDeviceSynchronize();

    hggcGraphExecDestroy(executable);
    hggcGraphDestroy(topGraph);
}
```


#### 4.1.6. HGGC 用户对象 {#416-hggc-用户对象}

当异步工作涉及动态分配的资源（缓冲区、临时对象等）时，一个核心难题是：何时才能安全释放这些资源？在基于流的模型中，可以通过事件或主机回调来追踪完成时机；但在图模式下，操作可能被反复启动，且图的生命周期与单次执行脱钩，传统方案往往不再适用。**HGGC 用户对象**（User Object）正是为解决这一问题而设计的轻量引用计数机制，特别适用于 HGGC 图和流捕获场景。

下面的代码展示了两种典型的资源管理模式，以及它们在图环境中面临的困难：


```cpp
// 模式 A：池化管理——每次启动前需检查可用性并在完成后归还
void submitAsyncTask(hggcStream_t taskStream) {
    auto &slot = resourcePool.acquire();
    slot.ensureReady(taskStream);
    runComputation(taskStream, slot);
    slot.markInUse(taskStream);
}

// 模式 B：回调驱动的释放——通过主机函数在操作完成后清理
void submitAsyncTask(hggcStream_t taskStream) {
    Workspace *ws = new Workspace(...);
    runComputation(taskStream, ws);
    hggcLaunchHostFunc(
        taskStream,
        [](void%20*arg) {
            delete static_cast<Workspace *>(arg);
        },
        ws,
        0);
}
```

上述方案在图模式下的困难在于：基于池的方式需要在每次启动时更新资源指针（需图更新或间接寻址），而回调方式涉及同步 API 调用且在捕获中被禁止。如果这些考虑因素对库的调用者隐藏，或者因为在捕获过程中使用了不允许的 API，它们也无法与流捕获配合使用。目前存在多种解决方案，例如将资源暴露给调用者。HGGC 用户对象（User Objects）提供了另一种实现途径。

**工作机制**：用户对象内部维护一个引用计数和一个用户指定的析构回调。引用可以由以下两方持有：

- **开发者代码**（CPU 端）：通过 `hggcUserObjectCreate` 创建时获得初始引用。注意：与 C++ 智能指针不同，不存在表示引用的对象；开发者必须手动追踪这些用户拥有的引用。
- **HGGC 图**：通过 `hggcGraphRetainUserObject` 将引用附着到图上。

一个典型的使用场景是，在创建用户对象后，立即将唯一的"用户拥有引用"转移给 HGGC 图（通过 `hggcGraphRetainUserObject` 配合 `hggcGraphUserObjectMove` 标志），此后资源生命周期完全由图托管。

**图操作对引用的影响**：

| 图操作                             | 引用行为                                   |
| ---------------------------------- | ------------------------------------------ |
| `hggcGraphClone`                   | 新图继承源图持有的全部引用副本（含多重性） |
| `hggcGraphInstantiate`             | 可执行图获得图模板引用的独立副本           |
| 销毁 `hggcGraphExec_t`（未同步时） | 引用保留直到所有在途执行完成后才释放       |

当引用计数降为零时，运行时自动调用创建时注册的析构回调，完成资源清理。

以下是一个使用示例：


```cpp
hggcGraph_t taskGraph;  // 已构建好的图模板

// 创建一个带有非平凡析构逻辑的 C++ 对象
ScratchBuffer *buf = new ScratchBuffer(1024 * 1024);
hggcUserObject_t managedObj;
hggcUserObjectCreate(
    &managedObj,
    buf,          // 使用 HGGC 提供的模板包装器，
                  // 该包装器生成删除 C++ 对象指针的回调
    1,            // 初始引用计数
    hggcUserObjectNoDestructorSync  // 表明回调不能通过 HGGC 等待
);

// 将所有权从调用线程转移给图
hggcGraphRetainUserObject(
    taskGraph,
    managedObj,
    1,              // 引用数量
    hggcGraphUserObjectMove  // 转移调用方拥有的引用
                             // （不修改总引用计数）
);

// 实例化——可执行图获得独立的引用副本
hggcGraphExec_t runnableGraph;
hggcGraphInstantiate(&runnableGraph, taskGraph, nullptr, nullptr, 0);
hggcGraphDestroy(taskGraph);

// 异步提交并同步
hggcGraphLaunch(runnableGraph, 0);
hggcGraphExecDestroy(runnableGraph);
hggcStreamSynchronize(0);
// 同步完成后所有引用已释放，ScratchBuffer 的析构函数被自动调用
```

由子图节点中的图所拥有的引用与子图关联，而非与父图关联。如果子图被更新或删除，引用也会相应地发生变化。如果可执行图或子图通过 `hggcGraphExecUpdate` 或 `hggcGraphExecChildGraphNodeSetParams` 进行更新，则新源图中的引用会被克隆并替换目标图中的引用。无论哪种情况，如果之前的启动操作未同步，任何会被释放的引用都将保持直至这些启动操作执行完成。

目前尚无机制可以通过 HGGC API 等待用户对象析构函数。开发者可以在析构函数代码中手动向同步对象发送信号。此外，从析构函数中调用 HGGC API 是非法的，这与 `hggcLaunchHostFunc` 的限制类似。这样做的目的是为了避免阻塞 HGGC 内部的共享线程并防止阻碍执行进度。如果依赖关系是单向的，且执行 API 调用的线程不会阻塞 HGGC 工作的进度，那么通知另一个线程去执行 API 调用是合法的。

用户对象使用 `hggcUserObjectCreate` 创建，这是浏览相关 API 的一个很好的起点。

#### 4.1.7. 使用图 API {#417-使用图-api}

`hggcGraph_t` 对象不是线程安全的。开发者有责任确保多个线程不会同时访问同一个 `hggcGraph_t`。

`hggcGraphExec_t` 不能与其自身并发运行。`hggcGraphExec_t` 的启动操作将按顺序排在同一可执行图的先前启动操作之后。

图的执行是在流中完成的，以便与其他异步工作进行排序。然而，流仅用于排序；它不会限制图的内部并行性，也不会影响图节点执行的位置。

请参阅 [使用图 API](#417-使用图-api)。

### 4.2. 流有序内存分配器 {#42-流有序内存分配器}

#### 4.2.1. 概述与查询支持 {#421-概述与查询支持}


在许多 PPU 应用中，内存分配与释放并非一次性操作，而是在迭代训练循环、流水线推理等场景下高频发生。传统的 `hggcMalloc` / `hggcFree` 每次调用都会引发跨所有流的隐式同步，在多流重叠执行的场景下会成为严重的性能瓶颈。

流有序内存分配器正是为解决这一问题而引入的。其核心思想是将内存分配与释放操作绑定到 HGGC 流上——分配在流时间线中的某个点变为可用，释放则在流时间线的另一个点生效——从而将内存操作融入已有的流有序执行模型，无需额外同步。分配器内部维护内存池，在池缓存策略允许的范围内自动复用已释放的内存块，减少向操作系统申请和归还内存的开销。应用程序可以通过释放阈值配置来权衡内存占用与分配性能。此外，该分配器还原生支持进程间内存共享。

该机制为应用带来三方面收益：

>   * 简化高频分配场景的内存管理，降低自行实现内存池的复杂度。
>
>   * 不同库和模块可共享同一个由驱动管理的内存池，避免各自独立管理带来的内存浪费。
>
>   * 驱动程序能够利用流有序语义做出更优的分配与调度决策。
>

**查询是否支持**

应用可以通过调用 `hggcDeviceGetAttribute()` 并查询设备属性 `hggcDevAttrMemoryPoolsSupported` 来判断设备是否支持流有序内存分配器。

IPC 内存池支持可以通过设备属性 `hggcDevAttrMemoryPoolSupportedHandleTypes` 查询。


```cpp
int driverVersion = 0;
int deviceSupportsMemoryPools = 0;
int poolSupportedHandleTypes = 0;
hggcDriverGetVersion(&driverVersion);
if (driverVersion >= 11020) {
    hggcDeviceGetAttribute(&deviceSupportsMemoryPools,
                           hggcDevAttrMemoryPoolsSupported, device);
}
if (deviceSupportsMemoryPools != 0) {
    // 该设备支持流有序内存分配器
}

if (driverVersion >= 11030) {
    hggcDeviceGetAttribute(&poolSupportedHandleTypes,
              hggcDevAttrMemoryPoolSupportedHandleTypes, device);
}
if (poolSupportedHandleTypes & hggcMemHandleTypePosixFileDescriptor) {
   // 该设备上的内存池可使用基于 POSIX 文件描述符的 IPC 机制
}
```


在查询之前先检查驱动版本，可以避免在属性尚未定义的驱动上触发 `hggcErrorInvalidValue`。也可以选择使用 `hggcGetLastError` 清除错误，而不是避免触发它。

#### 4.2.2. 内存管理 {#422-内存管理}

`hggcMallocAsync` 与 `hggcFreeAsync` 是实现流有序内存管理的 API。`hggcMallocAsync` 返回一段分配，`hggcFreeAsync` 释放一段分配。两个 API 都接收 stream 参数，用于定义该分配何时开始可用以及何时停止可用。这些函数允许将内存操作绑定到特定 HGGC 流，使其在不阻塞主机线程或其他流的情况下发生。通过避免 `hggcMalloc` 与 `hggcFree` 可能带来的高代价同步，可以提升应用性能。

这些 API 还可以结合内存池进行进一步的性能优化。内存池会管理并复用大块内存，以更高效地进行分配与释放。内存池有助于降低开销并防止碎片化，在频繁进行内存分配操作的场景下可提升性能。

##### 4.2.2.1. 分配内存 {#4221-分配内存}

`hggcMallocAsync` 函数会在 PPU 上触发异步内存分配，并与特定 HGGC 流关联。`hggcMallocAsync` 允许内存分配在不妨碍主机或其他流的情况下发生，从而消除高代价同步的需要。

!!! note

    `hggcMallocAsync` 在决定分配驻留位置时会忽略当前 device/context。相反，`hggcMallocAsync` 会基于所指定的内存池或所提供的流来确定合适的设备。

下面的清单展示了一个基本的使用模式：分配内存、使用它，然后在同一个流中释放回去。


```cpp
void *devBuf;
size_t nbytes = 512;
hggcMallocAsync(&devBuf, nbytes, hggcStreamPerThread);
// 在已分配的缓冲区上执行核函数计算
kernel<<<..., hggcStreamPerThread>>>(devBuf, ...);
// 流有序释放——无需显式 CPU/PPU 同步
hggcFreeAsync(devBuf, hggcStreamPerThread);
```


!!! note

    当从非分配该内存的流访问这段分配时，开发者必须保证访问发生在分配操作之后，否则行为未定义。

##### 4.2.2.2. 释放内存 {#4222-释放内存}

`hggcFreeAsync()` 以流有序的方式异步释放设备内存，这意味着内存释放操作被分配到某个特定 HGGC 流，并且不会阻塞主机或其他流。

开发者必须保证释放操作发生在分配操作以及该分配的所有使用之后。在释放操作开始之后对该分配的任何使用都会导致未定义行为。

应使用事件和/或流同步操作来保证来自其他流的任何访问在释放操作开始之前已经完成，如下例所示。


```cpp
hggcMallocAsync(&devBuf, nbytes, allocStream);
hggcEventRecord(allocReady, allocStream);
// 计算流需等待分配完成后才可访问该缓冲区
hggcStreamWaitEvent(computeStream, allocReady);
kernel<<<..., computeStream>>>(devBuf, ...);
hggcEventRecord(computeDone, computeStream);
// 释放流需等待计算流完成后才可回收该缓冲区
hggcStreamWaitEvent(releaseStream, computeDone);
hggcFreeAsync(devBuf, releaseStream);
```


使用 `hggcMalloc()` 分配的内存可以用 `hggcFreeAsync()` 释放。与上面相同，在释放操作开始前，必须完成对该内存的所有访问。


```cpp
hggcMalloc(&devBuf, nbytes);
kernel<<<..., workStream>>>(devBuf, ...);
hggcFreeAsync(devBuf, workStream);
```


同样地，使用 `hggcMallocAsync` 分配的内存也可以用 `hggcFree()` 释放。当通过 `hggcFree()` API 释放此类分配时，驱动会假定对该分配的所有访问都已完成，并且不会进行额外同步。开发者可以使用 `hggcStreamQuery` / `hggcStreamSynchronize` / `hggcEventQuery` / `hggcEventSynchronize` / `hggcDeviceSynchronize` 来保证相应异步工作已完成，并且 PPU 不会再尝试访问该分配。


```cpp
hggcMallocAsync(&devBuf, nbytes, workStream);
kernel<<<..., workStream>>>(devBuf, ...);
// 使用同步 API 释放时须确保所有异步操作已完成
hggcStreamSynchronize(workStream);
hggcFree(devBuf);
```


#### 4.2.3. 内存池管理 {#423-内存池管理}

##### 4.2.3.1. 默认池与显式池 {#4231-默认池与显式池}


内存池封装了虚拟地址与物理内存资源，并根据内存池的属性与特性进行分配与管理。内存池最主要的方面是它所管理的内存类型与位置。

所有对 `hggcMallocAsync` 的调用都会使用内存池资源。如果未指定内存池，`hggcMallocAsync` 会使用所提供 stream 的设备的当前内存池。设备的当前内存池可通过 `hggcDeviceSetMempool` 设置，并可通过 `hggcDeviceGetMempool` 查询。每个设备都有一个默认内存池；如果未调用 `hggcDeviceSetMempool`，则默认内存池处于活动状态。

API `hggcMallocFromPoolAsync` 以及 "hggcMallocAsync 的 c++ 重载" 允许用户在不将其设为当前内存池的情况下，为某次分配指定要使用的内存池。API `hggcDeviceGetDefaultMempool` 与 `hggcMemPoolCreate` 会返回内存池句柄。`hggcMemPoolSetAttribute` 与 `hggcMemPoolGetAttribute` 用于控制内存池的属性。

!!! note

    设备的当前 mempool 将是该设备的本地资源。因此，在不指定内存池的情况下进行分配，总会得到位于该流设备上的本地分配。


可以通过调用 `hggcDeviceGetDefaultMempool` 获取某设备的默认内存池。从设备默认内存池进行的分配，是位于该设备上的不可迁移（non-migratable）的设备分配。这些分配始终可从该设备访问。默认内存池的可访问性可通过 `hggcMemPoolSetAccess` 修改，并可通过 `hggcMemPoolGetAccess` 查询。由于默认池不需要显式创建，因此有时也称为隐式池。设备的默认内存池不支持 IPC。


`hggcMemPoolCreate` 会创建一个显式内存池。这允许应用为其分配请求默认/隐式池之外的属性。这些属性包括：IPC 能力、最大池大小、在受支持平台上将分配驻留在特定 CPU NUMA 节点等。


以下示例展示了两种典型的显式池创建场景——设备本地池与 NUMA 感知 IPC 池：

```cpp
// 场景 1：为指定 PPU 创建显式设备本地池
int ppuIdx = 0;
hggcMemPoolProps devPoolCfg = { };
devPoolCfg.allocType = hggcMemAllocationTypePinned;
devPoolCfg.location.id = ppuIdx;
devPoolCfg.location.type = hggcMemLocationTypeDevice;
hggcMemPool_t devicePool;
hggcMemPoolCreate(&devicePool, &devPoolCfg);

// 场景 2：在指定 CPU NUMA 节点上创建支持 IPC 共享的池
int numaNode = 0;
hggcMemPoolProps ipcPoolCfg = { };
ipcPoolCfg.allocType = hggcMemAllocationTypePinned;
ipcPoolCfg.location.id = numaNode;
ipcPoolCfg.location.type = hggcMemLocationTypeHostNuma;
ipcPoolCfg.handleType = hggcMemHandleTypePosixFileDescriptor;
hggcMemPool_t sharedPool;
hggcMemPoolCreate(&sharedPool, &ipcPoolCfg);
```


##### 4.2.3.2. 缓存行为与复用策略 {#4232-缓存行为与复用策略}


默认情况下，分配器会尽量最小化某个池所拥有的物理内存。为了尽量减少向操作系统申请与释放物理内存的调用，应用必须为每个池配置一个内存占用上限（footprint）。应用可以通过释放阈值属性（`hggcMemPoolAttrReleaseThreshold`）来实现。

释放阈值是指某个池在尝试将内存释放回操作系统之前应保留的内存字节数。当内存池持有的内存超过释放阈值字节时，分配器会在下一次进行流、事件或设备同步调用时尝试将内存释放回操作系统。将释放阈值设置为 UINT64_MAX 将阻止驱动在每次同步后都尝试收缩内存池。


```cpp
Hguint64_t setVal = UINT64_MAX;
hggcMemPoolSetAttribute(memPool, hggcMemPoolAttrReleaseThreshold, &setVal);
```


将 `hggcMemPoolAttrReleaseThreshold` 设得足够高、从而有效禁用内存池收缩的应用，可能仍希望显式收缩某个内存池的内存占用。`hggcMemPoolTrimTo` 允许应用这样做。收缩内存池占用时，`minBytesToKeep` 参数允许应用保留指定数量的内存，例如其预期在后续执行阶段仍需要的内存量。


```cpp
Hguint64_t setVal = UINT64_MAX;
hggcMemPoolSetAttribute(memPool, hggcMemPoolAttrReleaseThreshold, &setVal);

// 需要大量流有序内存的应用阶段
for (i=0; i<10; i++) {
    for (j=0; j<10; j++) {
        hggcMallocAsync(&ptrs[j],size[j], stream);
    }
    kernel<<<...,stream>>>(ptrs,...);
    for (j=0; j<10; j++) {
        hggcFreeAsync(ptrs[j], stream);
    }
}

// 下一阶段不再需要这么多内存
// 同步以确保收缩操作知道分配已不再使用
hggcStreamSynchronize(stream);
hggcMemPoolTrimTo(mempool, 0);

// 收缩操作释放的物理内存现在可供其他进程或分配机制使用
```


**内存复用策略**

为了满足一次分配请求，驱动会在尝试从操作系统分配更多内存之前，优先尝试复用之前通过 `hggcFreeAsync()` 释放的内存。例如，在某个流中释放的内存可以在同一流后续的分配请求中立即复用。当某个流与 CPU 同步后，该流中先前释放的内存就会对任何流中的分配变得可复用。复用策略可应用于默认内存池与显式内存池。

流有序分配器提供若干可控的分配策略。内存池属性 `hggcMemPoolReuseFollowEventDependencies`、`hggcMemPoolReuseAllowOpportunistic` 与 `hggcMemPoolReuseAllowInternalDependencies` 控制这些策略，详见 [4.2.3.2.1 节 复用策略详解](#42321-复用策略详解)。可以通过 `hggcMemPoolSetAttribute` 调用来启用或禁用这些策略。升级到更新的 HGGC 驱动可能会改变、增强、扩展并/或重新排列复用策略枚举的次序。

###### 4.2.3.2.1. 复用策略详解 {#42321-复用策略详解}

以下分别说明各复用策略的具体行为：

| 策略                                        | 触发条件                       | 效果                                     |
| ------------------------------------------- | ------------------------------ | ---------------------------------------- |
| `hggcMemPoolReuseFollowEventDependencies`   | 存在由 HGGC 事件建立的跨流依赖 | 利用事件依赖关系复用其他流中已释放的内存 |
| `hggcMemPoolReuseAllowOpportunistic`        | 释放操作的流有序语义已满足     | 根据流实际执行进度进行机会式复用         |
| `hggcMemPoolReuseAllowInternalDependencies` | 无法从操作系统获取更多物理内存 | 驱动自动插入流间依赖以强制复用           |

**hggcMemPoolReuseFollowEventDependencies**

在分配更多物理 PPU 内存之前，分配器会检查由 HGGC 事件建立的依赖信息，并尝试从另一个流中已释放的内存进行分配。


```cpp
hggcMallocAsync(&ptr, size, originalStream);
kernel<<<..., originalStream>>>(ptr, ...);
hggcFreeAsync(ptr, originalStream);
hggcEventRecord(event,originalStream);

// 等待捕获了另一流中释放操作的事件，使得分配器在
// hggcMemPoolReuseFollowEventDependencies 启用时可以复用该内存
// 来满足其他流中的新分配请求
hggcStreamWaitEvent(otherStream, event);
hggcMallocAsync(&ptr2, size, otherStream);
```


**hggcMemPoolReuseAllowOpportunistic**

启用 `hggcMemPoolReuseAllowOpportunistic` 策略时，分配器会检查已释放的分配，判断释放操作的流有序语义是否已满足，例如该流是否已经执行过释放操作所指示的执行点。当该策略被禁用时，分配器仍会复用在流与 CPU 同步后变得可用的内存。禁用该策略不会阻止 `hggcMemPoolReuseFollowEventDependencies` 的生效。


```cpp
hggcMallocAsync(&ptr, size, originalStream);
kernel<<<..., originalStream>>>(ptr, ...);
hggcFreeAsync(ptr, originalStream);


// 经过一段时间后，核函数执行完成
wait(10);

// 当 hggcMemPoolReuseAllowOpportunistic 启用时，分配器可根据
// originalStream 的实际执行进度，使用先前释放的内存满足此分配请求
hggcMallocAsync(&ptr2, size, otherStream);
```


**hggcMemPoolReuseAllowInternalDependencies**

当无法从操作系统分配并映射更多物理内存时，驱动会寻找那些可用性依赖于另一个流的待完成进度的内存。如果找到这样的内存，驱动会在发起分配的流中插入所需依赖，并复用该内存。


```cpp
hggcMallocAsync(&ptr, size, originalStream);
kernel<<<..., originalStream>>>(ptr, ...);
hggcFreeAsync(ptr, originalStream);

// 当 hggcMemPoolReuseAllowInternalDependencies 启用且驱动无法分配更多物理内存时，
// 驱动可能在发起分配的流中隐式插入 hggcStreamWaitEvent，
// 以确保 otherStream 中的后续工作发生在原始流中对该分配的访问完成之后
hggcMallocAsync(&ptr2, size, otherStream);
```


###### 4.2.3.2.2. 禁用复用策略 {#42322-禁用复用策略}

尽管这些可控复用策略能提升内存复用效果，开发者也可能希望禁用它们。允许机会式复用（例如 `hggcMemPoolReuseAllowOpportunistic`）会根据 CPU 与 PPU 执行的交错情况引入运行间差异，导致每次运行的分配模式不同。内部依赖插入（例如 `hggcMemPoolReuseAllowInternalDependencies`）可能会以出乎意料且潜在非确定性的方式串行化工作，而开发者可能更希望在分配失败时显式同步某个事件或流。

##### 4.2.3.3. 资源使用统计 {#4233-资源使用统计}

查询某个池的 `hggcMemPoolAttrReservedMemCurrent` 属性会报告该池当前消耗的物理 PPU 内存总量。查询 `hggcMemPoolAttrUsedMemCurrent` 会返回从该池分配且尚不可复用的内存总大小。

`hggcMemPoolAttr*MemHigh` 属性是水位线，记录自上次重置以来相应 `hggcMemPoolAttr*MemCurrent` 属性达到的最大值。可以通过 `hggcMemPoolSetAttribute` API 将其重置为当前值。


```cpp
// 批量获取使用统计的辅助函数示例
struct usageStatistics {
    hguint64_t reserved;
    hguint64_t reservedHigh;
    hguint64_t used;
    hguint64_t usedHigh;
};

void getUsageStatistics(hggcMemoryPool_t memPool, struct usageStatistics *statistics)
{
    hggcMemPoolGetAttribute(memPool, hggcMemPoolAttrReservedMemCurrent, statistics->reserved);
    hggcMemPoolGetAttribute(memPool, hggcMemPoolAttrReservedMemHigh, statistics->reservedHigh);
    hggcMemPoolGetAttribute(memPool, hggcMemPoolAttrUsedMemCurrent, statistics->used);
    hggcMemPoolGetAttribute(memPool, hggcMemPoolAttrUsedMemHigh, statistics->usedHigh);
}


// 重置水位线后，它们将取当前值作为新的起点
void resetStatistics(hggcMemoryPool_t memPool)
{
    hguint64_t value = 0;
    hggcMemPoolSetAttribute(memPool, hggcMemPoolAttrReservedMemHigh, &value);
    hggcMemPoolSetAttribute(memPool, hggcMemPoolAttrUsedMemHigh, &value);
}
```


#### 4.2.4. 多设备与 IPC 支持 {#424-多设备与-ipc-支持}

##### 4.2.4.1. 面向多 PPU 支持的设备可访问性 {#4241-面向多-ppu-支持的设备可访问性}

与通过虚拟内存管理 API 控制分配可访问性类似，内存池分配的可访问性不遵循 `hggcDeviceEnablePeerAccess` 或 `hgCtxEnablePeerAccess`。对于内存池，API `hggcMemPoolSetAccess` 用于修改哪些设备可以访问某个池中的分配。默认情况下，分配仅能从分配驻留的设备访问，并且这种访问无法被撤销。要允许其他设备访问，访问设备必须与该内存池所在设备具备 peer 能力。可以通过 `hggcDeviceCanAccessPeer` 进行验证。如果未检查 peer 能力，设置访问权限可能会以 `hggcErrorInvalidDevice` 失败。不过，如果该池尚未产生任何分配，即使设备不具备 peer 能力，`hggcMemPoolSetAccess` 调用也可能成功；在这种情况下，该池的下一次分配将会失败。

需要注意的是，`hggcMemPoolSetAccess` 会影响内存池中的所有分配，而不仅是未来的分配。同样，`hggcMemPoolGetAccess` 报告的可访问性也适用于池中的所有分配，而不仅是未来的分配。不建议频繁更改某个池针对某个 PPU 的可访问性设置。也就是说，一旦让某个 PPU 可以访问某个池，那么在该池的生命周期内应保持该 PPU 的访问权限。


```cpp
// 为目标 PPU 开启对指定内存池的读写访问权限
hggcError_t enableCrossDeviceAccess(hggcMemPool_t pool, int ownerPpu,
              int peerPpu) {
    // 先验证两个 PPU 之间是否具备 peer 能力
    int peerCapable = 0;
    hggcError_t status = hggcDeviceCanAccessPeer(&peerCapable, peerPpu,
              ownerPpu);
    if (status != hggcSuccess) {
        return status;
    }
    if (peerCapable == 0) {
        return hggcErrorPeerAccessUnsupported;
    }

    // 构造访问描述符并设置权限
    hggcMemAccessDesc peerDesc = {};
    peerDesc.location.type = hggcMemLocationTypeDevice;
    peerDesc.location.id = peerPpu;
    peerDesc.flags = hggcMemAccessFlagsProtReadWrite;
    return hggcMemPoolSetAccess(pool, &peerDesc, 1);
}
```


##### 4.2.4.2. 为 IPC 启用内存池 {#4242-为-ipc-启用内存池}

内存池可以启用进程间通信（IPC），以便在进程之间轻松、高效且安全地共享 PPU 内存。HGGC 的 IPC 内存池提供与 HGGC 的"虚拟内存管理 API"相同的安全收益。

使用内存池在进程间共享内存分两步：进程首先需要共享对该池的访问权限，然后共享该池中的特定分配。第一步建立并强制执行安全策略。第二步协调每个进程中使用哪些虚拟地址，以及在导入进程中何时需要映射有效。

###### 4.2.4.2.1. 创建并共享 IPC 内存池 {#42421-创建并共享-ipc-内存池}

共享对某个池的访问权限包括：通过 `hggcMemPoolExportToShareableHandle()` 获取该池的 OS 原生句柄；使用 OS 原生 IPC 机制将该句柄传输到导入进程；然后使用 `hggcMemPoolImportFromShareableHandle()` API 创建导入的内存池。要使 `hggcMemPoolExportToShareableHandle` 成功，该内存池必须在创建时于池属性结构中指定所请求的句柄类型。

其流程可见以下代码片段。


```cpp
// ── 导出进程 ──
// 在设备 0 上创建支持 IPC 导出的内存池
hggcMemPoolProps exportPoolCfg = { };
exportPoolCfg.handleTypes = HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR;
exportPoolCfg.allocType = hggcMemAllocationTypePinned;
exportPoolCfg.location.type = hggcMemLocationTypeDevice;
exportPoolCfg.location.id = 0;

hggcMemPoolCreate(&ipcPool, &exportPoolCfg));

// 基于文件描述符的句柄为整数类型
int shareFd = 0;


// 获取内存池的操作系统原生句柄
// 注意：此处传入的是句柄变量的指针
hggcMemPoolExportToShareableHandle(&shareFd,
             ipcPool,
             HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR,
             0);

// 需要通过操作系统特定的 IPC 机制将句柄发送给导入进程



// ── 导入进程 ──
 int shareFd;
// 需要通过操作系统特定的 IPC 机制从导出进程获取句柄
// 使用共享句柄创建导入的内存池
// 注意：此处句柄以值传递
hggcMemPoolImportFromShareableHandle(&importedMemPool,
          (void*)shareFd,
          HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR,
          0);
```


###### 4.2.4.2.2. 在导入进程中设置访问权限 {#42422-在导入进程中设置访问权限}

导入的内存池起初仅能从其驻留设备访问。导入的内存池不会继承导出进程所设置的任何可访问性。导入进程需要为计划访问该内存的任意 PPU 调用 `hggcMemPoolSetAccess` 来启用访问权限。

如果导入的内存池属于导入进程不可见的设备，开发者必须使用 `hggcMemPoolSetAccess` API 来启用将要使用这些分配的 PPU 的访问权限。（参见 [4.2.4.1 节 面向多 PPU 支持的设备可访问性](#4241-面向多-ppu-支持的设备可访问性)）

###### 4.2.4.2.3. 创建并共享来自已导出池的分配 {#42423-创建并共享来自已导出池的分配}

一旦共享了内存池，导出进程中使用该池通过 `hggcMallocAsync()` 创建的分配，就可以与导入了该池的进程共享。由于内存池级别已经建立并验证了安全策略，操作系统无需为特定的池分配进行额外安全记账。换言之，用于导入池分配所需的不透明数据 `hggcMemPoolPtrExportData` 可以通过任何机制发送给导入进程。

虽然分配可以在不以任何方式与分配流同步的情况下导出与导入，但导入进程在访问该分配时必须遵循与导出进程相同的规则。具体而言，对该分配的访问必须发生在分配流中分配操作执行之后。下面两个代码片段展示了 `hggcMemPoolExportPointer()` 与 `hggcMemPoolImportPointer()` 如何共享该分配，并使用 IPC 事件来保证导入进程不会在分配就绪之前访问该分配。


```cpp
// ── 导出进程：准备分配 ──
hggcMemPoolPtrExportData ptrShareData;
hggcEvent_t allocReadyEvent;
hggcIpcEventHandle_t allocReadyEventHandle;

// 创建用于跨进程协调的 IPC 事件
// hggcEventInterprocess 标志使事件支持跨进程传递
// hggcEventDisableTiming 用于提升性能

hggcEventCreate(&allocReadyEvent, hggcEventDisableTiming | hggcEventInterprocess)

// 从导出内存池中分配
hggcMallocAsync(&ptr, size, exportMemPool, stream);

// 记录事件以通知导入进程分配已就绪
hggcEventRecord(allocReadyEvent, stream);
hggcMemPoolExportPointer(&ptrShareData, ptr);
hggcIpcGetEventHandle(&allocReadyEventHandle, allocReadyEvent);

// 通过任意 IPC 机制将事件句柄和指针导出数据发送给导入进程
// 此处以共享内存为例
shmem->ptrData = ptrShareData;
shmem->allocReadyEventHandle = allocReadyEventHandle;
// 通知消费者数据已就绪



// ── 导入进程：导入分配 ──
hggcMemPoolPtrExportData *importData = &shmem->prtData;
hggcEvent_t allocReadyEvent;
hggcIpcEventHandle_t *allocReadyEventHandle = &shmem->allocReadyEventHandle;

// 通过任意 IPC 机制获取事件句柄和导出数据
// 此处使用共享内存，需确保共享内存已被导出进程填充

hggcIpcOpenEventHandle(&allocReadyEvent, allocReadyEventHandle);

// 导入分配；该操作不会阻塞等待分配就绪
hggcMemPoolImportPointer(&ptr, importedMemPool, importData);

// 在导入进程中使用该分配前，必须等待导出进程中的分配操作完成
hggcStreamWaitEvent(stream, allocReadyEvent);
kernel<<<..., stream>>>(ptr, ...);
```


在释放该分配时，必须先在导入进程中释放该分配，然后才能在导出进程中释放。下面的代码片段演示了如何使用 HGGC IPC 事件，在两个进程中的 `hggcFreeAsync` 操作之间提供所需的同步。导入进程对该分配的访问显然会被导入侧的释放操作所限制。需要注意的是，可以在两个进程中都使用 `hggcFree` 来释放该分配，并且也可以使用其他流同步 API 来替代 HGGC IPC 事件。


```cpp
// 必须先在导入进程中释放，再在导出进程中释放
kernel<<<..., stream>>>(ptr, ...);

// 导入进程中的最后一次访问
hggcFreeAsync(ptr, stream);

// 释放后导入进程不可再访问该内存
hggcEventRecord(finishedIpcEvent, stream);



// ── 导出进程 ──
// 导出进程的释放操作需要与导入进程的释放保持流有序协调
hggcStreamWaitEvent(stream, finishedIpcEvent);
kernel<<<..., stream>>>(ptrInExportingProcess, ...);

// 导入进程的释放不影响导出进程继续使用该分配
hggcFreeAsync(ptrInExportingProcess,stream);
```


###### 4.2.4.2.4. IPC 导出池的限制 {#42424-ipc-导出池的限制}

IPC 池目前不支持将物理块释放回操作系统。因此，`hggcMemPoolTrimTo` API 不会产生效果，`hggcMemPoolAttrReleaseThreshold` 也会被事实上忽略。该行为由驱动控制而非 runtime，并可能在未来的驱动更新中发生变化。

###### 4.2.4.2.5. IPC 导入池的限制 {#42425-ipc-导入池的限制}

不允许从导入池进行分配；具体来说，导入池不能被设为当前池，也不能用于 `hggcMallocFromPoolAsync` API。因此，分配复用策略相关属性对这些池没有意义。

IPC 导入池与 IPC 导出池类似，目前也不支持将物理块释放回操作系统。

资源使用统计属性查询只会反映导入到该进程中的分配以及相关的物理内存。

#### 4.2.5. 同步 API 的动作 {#425-同步-api-的动作}

分配器作为 HGGC 驱动一部分所带来的优化之一，是与同步 API 的集成。当用户请求 HGGC 驱动进行同步时，驱动会等待异步工作完成。在返回之前，驱动会确定哪些释放操作已由该同步保证完成。这些分配将变得可用于后续分配，而不受指定流或已禁用分配策略的影响。驱动也会在此处检查 `hggcMemPoolAttrReleaseThreshold`，并尽可能释放多余的物理内存。

#### 4.2.6. 附录 {#426-附录}

##### 4.2.6.1. hggcMemcpyAsync 的当前上下文/设备敏感性 {#4261-hggcmemcpyasync-的当前上下文设备敏感性}

在当前 HGGC 驱动中，任何涉及 `hggcMallocAsync` 内存的异步 `memcpy` 都应在指定流的上下文作为调用线程的当前上下文时执行。对于 `hggcMemcpyPeerAsync` 则不需要这样做，因为该 API 会引用其参数中指定的设备主上下文，而不是当前上下文。

##### 4.2.6.2. hggcPointerGetAttributes 查询 {#4262-hggcpointergetattributes-查询}

在对某个分配调用 `hggcFreeAsync` 之后再对其调用 `hggcPointerGetAttributes` 会导致未定义行为。具体来说，即使某段分配仍可从某个流访问，其行为仍然是未定义的。

##### 4.2.6.3. hggcGraphAddMemsetNode {#4263-hggcgraphaddmemsetnode}

`hggcGraphAddMemsetNode` 无法与通过流有序分配器分配的内存一起工作。不过，可以对这些分配进行 stream capture 的 memset。

##### 4.2.6.4. 指针属性 {#4264-指针属性}

`hggcPointerGetAttributes` 查询适用于流有序分配。由于流有序分配不与上下文相关联，查询 `HG_POINTER_ATTRIBUTE_CONTEXT` 会成功，但在 `*data` 中返回 NULL。属性 `HG_POINTER_ATTRIBUTE_DEVICE_ORDINAL` 可用于确定分配所在位置：这在选择用于使用 `hggcMemcpyPeerAsync` 进行 p2h2p 拷贝的上下文时可能很有用。属性 `HG_POINTER_ATTRIBUTE_MEMPOOL_HANDLE` 可用于调试，并用于在进行 IPC 之前确认某个分配来自哪个池。

##### 4.2.6.5. CPU 虚拟内存 {#4265-cpu-虚拟内存}

在使用 HGGC 流有序内存分配器 API 时，应避免使用 "ulimit -v" 设置 VRAM 限制，因为这不受支持。

### 4.3. 协作组 {#43-协作组}

#### 4.3.1. 简介 {#431-简介}

协作组（Cooperative Groups）是 HGGC 编程模型的一个扩展，用于组织协同工作的线程组。协作组允许开发人员控制线程协作的粒度，帮助他们表达更丰富、更高效的并行分解。协作组还提供了常见并行原语的实现，如扫描 (scan) 和平行归约 (reduce)。

HGGC 编程模型最初只提供了一种简单的同步协作线程组的构造：一个跨线程块所有线程的屏障 (barrier)，由 `__syncthreads()` 内联函数实现。为了表达更广泛的并行交互模式，协作组为编写高性能代码提供了一个安全和面向未来的机制。

#### 4.3.2. 协作组句柄与成员函数 {#432-协作组句柄与成员函数}
协作组通过协作组句柄管理。协作组句柄允许参与线程了解它们在组中的位置、组大小和其他组信息。下表显示了一些选择的组成员函数。

| 访问接口       | 返回值                          |
| -------------- | ------------------------------- |
| thread_rank()  | 调用线程的 rank                 |
| num_threads()  | 组中线程总数                    |
| thread_index() | 在启动块中线程的三维索引        |
| dim_threads()  | 启动块的三维尺寸 (以线程为单位) |

#### 4.3.3. 默认行为/协作组执行 {#433-默认行为协作组执行}

协作组的 group 是表示 grid 网格和 block 线程块，其被核函数启动配置隐式创建。这些"隐式"组为开发人员提供了一个起点，可以将其显式分解为更细粒度的组。可以通过以下方法访问隐式组：

| 访问器               | 组作用域                               |
| -------------------- | -------------------------------------- |
| this_thread_block()  | 返回包含当前线程块中所有线程的组的句柄 |
| this_grid()          | 返回包含网格中所有线程的组的句柄       |
| coalesced_threads()¹ | 返回 warp 中当前活动线程组的句柄       |

##### 4.3.3.1. 尽早创建隐式组句柄 {#4331-尽早创建隐式组句柄}

为了获得最佳性能，建议你尽早创建隐式组的句柄（尽可能早，在任何分支发生之前），并在整个核函数中使用该句柄。

##### 4.3.3.2. 传引用传递协作组句柄 {#4332-传引用传递协作组句柄}

句柄必须在声明时初始化，因为没有默认构造函数。不鼓励复制构造组句柄。

#### 4.3.4. 创建协作组 {#434-创建协作组}

协作组通过将父组分区为子组来创建。当一个组被分区时，会创建一个组句柄来管理生成的子组。开发者可使用以下分区操作：

| 分区类型          | 描述                                                     |
| ----------------- | -------------------------------------------------------- |
| tiled_partition   | 将父组划分为一系列固定大小的子组，排列成一维行优先格式   |
| stride_partition  | 将父组划分为等大小的子组，其中线程以循环方式分配给子组   |
| labeled_partition | 基于条件标签将父组划分为一维子组，标签可以是任意整数类型 |
| binary_partition  | 标记分区的专门形式，其中标签只能是"0"或"1"               |

以下示例展示如何创建平铺分区：
```cpp
namespace cg = cooperative_groups;
// 获取当前线程的协作组
cg::thread_block my_group = cg::this_thread_block();

// 将协作组分割为大小为8的瓦片
cg::thread_block_tile<8> my_subgroup = cg::tiled_partition<8>(cta);

// 作为my_subgroup工作
```

##### 4.3.4.1. 避免组创建危险 {#4341-避免组创建危险}
对组进行分区是一个集体操作，组中的所有线程都必须参与。如果组是在不是所有线程都能到达的条件分支中创建的，则可能导致死锁或数据损坏。

#### 4.3.5. 同步 {#435-同步}
在引入协作组之前，HGGC 编程模型只允许在线程块级别在核函数完成边界处进行同步。协作组允许开发人员在不同粒度级别同步合作线程组。

##### 4.3.5.1. cg.Sync {#4351-cgsync}

你可以通过调用集体 sync() 函数来同步组。像 `__syncthreads()` 一样，sync() 函数做出以下保证：
- 所有线程在同步点之前的内存访问（例如读写）对于组中的所有线程在同步点之后都是可见的。
- 组中的所有线程都在任何一个线程被允许继续前进之前达到同步点。

以下示例显示了相当于 `__syncthreads()` 的 cooperative_groups::sync()。

```cpp
namespace cg = cooperative_groups;

cg::thread_block my_group = cg::this_thread_block();

// 同步块中的线程
cg::sync(my_group);
```


##### 4.3.5.2. Barriers 屏障 {#4352-barriers屏障}

协作组提供类似于 `ppu_barrier_sync_cnt` 的屏障 API，可用于更高级别的同步。协作组屏障 API 在几个关键方面不同于 `ppu_barrier_sync_cnt`：
- 协作组屏障自动初始化
- 组中的所有线程必须在一个阶段内一次全部到达并等待在屏障处。
- `barrier_arrive` 返回一个 `arrival_token` 对象，必须将其传递到相应的 `barrier_wait` 中，在那里它会被消耗并且不能再使用。

程序员在使用协作组屏障时必须注意避免危险：
- 在调用 `barrier_arrive` 之后但在调用 `barrier_wait` 之前，不能在组间使用任何集体操作。
- `barrier_wait` 仅保证组中的所有线程都已调用 `barrier_arrive`。`barrier_wait` 并不保证所有线程都已调用 `barrier_wait`。

```cpp
namespace cg = cooperative_groups;

cg::thread_block my_group = cg::this_thread_block();

auto token = block.barrier_arrive();

// 可选：做一些本地处理以隐藏同步延迟
local_processing();

// 确保同步后再访问 shared memory
block.barrier_wait(std::move(token));
```

#### 4.3.6. 协作操作 {#436-协作操作}

协作组包括一组可由线程组执行的协作操作。这些操作要求指定组中的所有线程都参与才能完成操作。
除非在协作组 API 中明确允许不同的值，否则组中的所有线程必须为每个协作调用传递相应参数的相同值。否则调用的行为未定义。

##### 4.3.6.1. 归约操作 {#4361-归约操作}

reduce 函数用于对指定组中每个线程提供的数据执行并行归约。必须通过提供下表所示的操作符之一来指定归约类型。

| 操作符  | 返回值           |
| ------- | ---------------- |
| plus    | 组中所有值的总和 |
| less    | 最小值           |
| greater | 最大值           |
| bit_and | 按位 AND 归约    |
| bit_or  | 按位 OR 归约     |
| bit_xor | 按位 XOR 归约    |
PPU 现存的所有硬件都对归约操作有加速效果。

以下示例展示了如何使用 `__shfl_down_sync` 在 warp 级别的 tile 上执行求和归约。
```cpp
__global__ void tileReduceKernel(float* d_in, float* d_out) {
    namespace cg = cooperative_groups;
    // 1. 获取线程块组，并创建 warp 级别的 tile
    cg::thread_block block = cg::this_thread_block();
    cg::thread_block_tile<32> tile = cg::tiled_partition<32>(block);

    // 2. 将数据加载到寄存器中（例如每个线程处理一个元素）
    float val = d_in[blockIdx.x * blockDim.x + threadIdx.x];

    // 3. 使用 __shfl_down_sync 进行 warp 级归约
    float tile_sum = val;
    for (int offset = tile.size() / 2; offset > 0; offset /= 2) {
        tile_sum += __shfl_down_sync(0xFFFFFFFF, tile_sum, offset);
    }

    // 4. 每个 warp 的线程0将结果写入共享内存，再由线程块的线程0汇总
    __shared__ float warp_sums[32];
    if (tile.thread_rank() == 0) {
        warp_sums[tile.meta_group_rank()] = tile_sum;
    }
    block.sync();

    // 线程块的线程0汇总所有 warp 的结果
    if (threadIdx.x == 0) {
        float block_sum = 0.0f;
        int num_warps = (blockDim.x + 31) / 32;
        for (int i = 0; i < num_warps; i++) {
            block_sum += warp_sums[i];
        }
        d_out[blockIdx.x] = block_sum;
    }
}
```

##### 4.3.6.2. 扫描 {#4362-扫描}

协作组包含了 inclusive_scan 和 exclusive_scan 的实现，可以在任意组大小上使用。这些函数对指定组中命名的每个线程提供的数据执行扫描操作。
程序员可以选择性地指定一个归约运算符，如上面归约运算符表所列。

```cpp
namespace cg = cooperative_groups;
cg::thread_block block = cg::this_thread_block();
cg::thread_block_tile<32> tile = cg::tiled_partition<32>(block);
int val = data[block.thread_rank()];
// 使用 __shfl_up_sync 实现 exclusive scan
int scan_val = val;
for (int offset = 1; offset < tile.size(); offset <<= 1) {
    int n = __shfl_up_sync(0xFFFFFFFF, scan_val, offset);
    if (tile.thread_rank() >= offset) scan_val += n;
}
int exclusive_sum = __shfl_up_sync(0xFFFFFFFF, scan_val, 1);
if (tile.thread_rank() == 0) exclusive_sum = 0;
result[block.thread_rank()] = exclusive_sum;
```

##### 4.3.6.3. 从组中挑选一个 {#4363-从组中挑选一个}

协作组提供 invoke_one 函数，用于当单一线程必须代表组执行串行部分工作时。
- invoke_one 从调用组中选择单一任意线程，并使用该线程调用具有所提供的参数的供应函数。
- invoke_one_broadcast 与 invoke_one 相同，只是调用的结果也会广播到组中的所有线程。

注意：`invoke_one_broadcast` 仅支持 tile 类型（`thread_block_tile` 或 `coalesced_group`），不支持直接传入 `thread_block`，因为广播需要 warp 级别的 shuffle 操作。

线程选择机制不保证是确定性的。
以下示例展示了基本的 invoke_one 用法。

```cpp
__global__ void invokeOneBroadcastExample(int *d_out) {
    namespace cg = cooperative_groups;
    // 获取当前线程块，并创建 warp 级别的 tile
    cg::thread_block block = cg::this_thread_block();
    cg::thread_block_tile<32> tile = cg::tiled_partition<32>(block);

    // lambda 表达式：代表单线程要执行的计算逻辑
    // 假设我们让选定的线程计算 42 * 2 = 84
    auto compute_func = []() -> int {
        int result = 42 * 2;
        return result;
    };
    // 调用 invoke_one_broadcast
    // tile 内所有线程都会等待，直到 1 个被选中的线程执行完 compute_func，
    // 并将结果（84）广播回给 tile 内所有线程。
    int broadcasted_val = cg::invoke_one_broadcast(tile, compute_func);

    // 将结果写入全局内存
    if (block.thread_rank() == 0) {
        *d_out = broadcasted_val;
    }
}
```

#### 4.3.7. 异步数据移动 {#437-异步数据移动}

HGGC 中的协作组 memcpy_async 功能提供了一种在全局内存和共享内存之间执行异步内存拷贝的方法。memcpy_async 特别适用于优化内存传输并将计算与数据传输重叠以提高性能。

memcpy_async 函数用于启动从全局内存到共享内存的异步加载。memcpy_async 旨在用作"预取"，即在需要数据之前加载数据。

wait 函数强制组中的所有线程等待，直到异步内存传输完成。在数据可以在共享内存中访问之前，组中的所有线程都必须调用 wait。
以下示例演示了如何使用 memcpy_async 和 wait 来预取数据。
```cpp
__global__ void kernel(int* global_data) {
    cg::thread_block tb = cg::this_thread_block();
    const size_t elementsPerThreadBlock = 16 * 1024;
    const size_t elementsInShared = 128;
    __shared__ int local_smem[elementsInShared];

    size_t copy_count;
    size_t index = 0;
    while (index < elementsPerThreadBlock) {
        cg::memcpy_async(tb, local_smem, elementsInShared, global_data + index, elementsPerThreadBlock - index);
        copy_count = min(elementsInShared, elementsPerThreadBlock - index);
        cg::wait(tb);
        index += copy_count;
    }
}

```

##### 4.3.7.1. Memcpy Async 对齐要求 {#4371-memcpy-async-对齐要求}

memcpy_async 仅在源地址位于全局内存且目标地址位于共享内存且两者至少为 4 字节对齐的情况下才是异步的。为了获得最佳性能：建议共享内存和全局内存都有 16 字节对齐。

#### 4.3.8. 大规模组 {#438-大规模组}
协作组允许跨越整个网格的大组。前面描述的所有协作组功能都可用于这些大型组，但有一个显著例外：同步整个网格需要使用 hggcLaunchCooperativeKernel 运行时启动 API。


##### 4.3.8.1. 何时使用 hggcLaunchCooperativeKernel {#4381-何时使用hggclaunchcooperativekernel}
hggcLaunchCooperativeKernel 是一个 HGGC 运行时 API 函数，用于启动采用协作组的单设备核函数，专门为执行需要块间同步的核函数设计。此函数确保核函数中的所有线程都可以在整个网格范围内同步和协作，这是传统 HGGC 核函数不可能做到的，后者只允许在单独的线程块内同步。hggcLaunchCooperativeKernel 确保核函数启动是原子的，也就是说，如果 API 调用成功，则提供的线程块数量将在指定设备上启动。

良好的做法是首先通过查询设备属性 hggcDevAttrCooperativeLaunch 来确认设备是否支持协作启动：

```cpp
int dev = 0;
int supportsCoopLaunch = 0;
hggcDeviceGetAttribute(&supportsCoopLaunch, hggcDevAttrCooperativeLaunch, dev);
```

### 4.4. 延迟加载 {#44-延迟加载}

#### 4.4.1. 简介 {#441-简介}

延迟加载通过等待到需要时才加载 HGGC 模块来减少程序初始化时间。延迟加载对于只使用其所包含的核函数的一小部分的程序特别有效，这在使用库时很常见。延迟加载设计为在遵循 HGGC 编程模型时对开发者透明。潜在风险一节详细解释了这一点。延迟加载可通过 `HGGC_MODULE_LOADING` 环境变量控制。

#### 4.4.2. 延迟加载的要求 {#442-延迟加载的要求}

##### 4.4.2.1. 核函数要求 {#4421-核函数要求}

延迟加载不影响包含管理变量的模块，这些模块仍将 eagerly 加载。

#### 4.4.3. 使用方法 {#443-使用方法}

##### 4.4.3.1. 启用与禁用 {#4431-启用与禁用}

延迟加载通过将 `HGGC_MODULE_LOADING` 环境变量设置为 `2` 来启用。延迟加载可通过将 `HGGC_MODULE_LOADING` 环境变量设置为 `1` 来禁用。

##### 4.4.3.2. 在运行时检查延迟加载是否启用 {#4432-在运行时检查延迟加载是否启用}

HGGC 驱动 API 中的 `hgModuleGetLoadingMode` API 可用于确定延迟加载是否已启用。请注意，在运行此函数之前必须先初始化 HGGC。下面的代码片段展示了示例用法。

```cpp
#include <hggc.h>
#include <assert.h>
#include <iostream>

int main() {
 HGmoduleLoadingMode mode;

 assert(HGGC_SUCCESS == hgInit(0));
 assert(HGGC_SUCCESS == hgModuleGetLoadingMode(&mode));

 std::cout << "HGGC Module Loading Mode is " << ((mode == HG_MODULE_LAZY_LOADING) ? "lazy" : "eager") << std::endl;

 return 0;
}
```

##### 4.4.3.3. 在运行时强制模块 eager 加载 {#4433-在运行时强制模块-eager-加载}

核函数和变量的加载是自动发生的，无需任何显式加载。即使不执行核函数，也可以通过以下方式显式加载核函数：

- `hgModuleGetFunction()` 函数会导致模块被加载到设备内存中
- `hggcFuncGetAttributes()` 函数会导致核函数被加载到设备内存中

!!! note

    `hgModuleLoad()` 不能保证模块会被立即加载。

#### 4.4.4. 潜在风险 {#444-潜在风险}

延迟加载的设计使其在使用时不需要对应用程序进行任何修改。尽管如此，仍有一些注意事项，特别是当应用程序不完全符合 HGGC 编程模型时，如下所述。

##### 4.4.4.1. 对并发核函数执行的影响 {#4441-对并发核函数执行的影响}

一些程序错误地假设并发核函数执行是有保证的。如果需要跨核函数同步，但核函数执行已被序列化，则可能发生死锁。为最大程度减少延迟加载对并发核函数执行的影响，请执行以下操作：

- 在启动之前预加载所有希望并发执行的核函数，或
- 使用 `HGGC_MODULE_LOADING = 1` 运行应用程序，以强制急迫加载数据，而不强制每个函数急迫加载

##### 4.4.4.2. 大内存分配 {#4442-大内存分配}

延迟加载将 HGGC 模块的内存分配从程序初始化延迟到接近执行时间。如果应用程序在启动时分配整个 VRAM，HGGC 可能会在运行时无法为模块分配内存。可能的解决方案：

- 使用 `hggcMallocAsync()` 代替在启动时分配整个 VRAM 的分配器
- 添加一些缓冲区以补偿核函数的延迟加载
- 在尝试初始化分配器之前预加载程序中将使用的所有核函数

##### 4.4.4.3. 对性能测量的影响 {#4443-对性能测量的影响}

延迟加载可能会通过将 HGGC 模块初始化移入测量的执行窗口来扭曲性能测量。为避免此问题：

- 在测量之前至少执行一次预热迭代
- 在启动被测试的核函数之前先预加载它


### 4.5. 异步屏障与流水线 {#45-异步屏障与流水线}


异步屏障在高级同步原语中引入，扩展了 HGGC 同步功能，超越了 `__syncthreads()` 和 `__syncwarp()`，实现了细粒度、非阻塞的协调以及更好的通信和计算重叠。
本节主要介绍如何通过 hggc::barrier API 使用异步屏障。

#### 4.5.1. 初始化 {#451-初始化}

初始化必须在任何线程开始参与屏障之前发生。

**HGGC C++ hggc::barrier**

```cpp
#include <hggc/barrier>
#include <cooperative_groups.h>

__global__ void barrier_setup_demo()
{
  __shared__ hggc::barrier<hggc::thread_scope_block> blk_bar;
  auto grp = cooperative_groups::this_thread_block();

    if (grp.thread_rank() == 0)
    {
      init(&blk_bar, grp.size());
    }
    grp.sync();
}
```


#### 4.5.2. 屏障的阶段：到达 (Arrival)、倒计时 (Countdown)、完成 (Completion) 和重置 (Reset) {#452-屏障的阶段到达arrival倒计时countdown完成completion和重置reset}

当多个线程调用 arrive() 操作时，屏障对象会跟踪这些操作。每次 arrive() 都会使内部计数器递增，直到达到预设的期望值。一旦达到该值，屏障就会进入完成状态，等待中的线程可以继续执行。

如果一个线程调用了 `token=bar.arrive()` 并且在调用 `bar.wait(std::move(token))` 之前的阶段没有翻转（仍为零），则线程不会阻塞；如果线程在 `bar.wait(std::move(token))` 中被阻塞时阶段提前，则线程会被解除阻塞。

了解何时可能发生或不可能发生重置至关重要，特别是在复杂的 arrive/wait 同步模式中。

- 线程对 `token=bar.arrive()` 和 `bar.wait(std::move(token))` 的调用必须按顺序进行，使得 `token=bar.arrive()` 发生在屏障的当前阶段期间，而 `bar.wait(std::move(token))` 发生在同一或下一阶段。
- 线程对 `bar.arrive()` 的调用必须发生在屏障计数器非零时。屏障初始化后，如果线程对 `bar.arrive()` 的调用导致倒计时达到零，则必须在屏障可用于后续 `bar.arrive()` 调用之前发生对 `bar.wait(std::move(token))` 的调用。
- 只能使用当前阶段或前一阶段的令牌对象调用 `bar.wait()`。对于令牌对象的任何其他值，行为都是未定义的。

对于简单的 arrive/wait 同步模式，遵守这些使用规则是直截了当的。

##### 4.5.2.1. Warp 缠绕 (Warp Entanglement) {#4521-warp-缠绕warp-entanglement}

Warp 发生分歧（Divergence）会影响 arrive-on 操作更新屏障的次数。如果调用的 warp 完全收敛，则屏障仅更新一次。如果调用的 warp 完全发散，则将 32 个单独的更新应用于屏障。

注意：建议由收敛线程使用 `arrive-on(bar)` 调用来最小化对屏障对象的更新。当前述代码使线程分歧时，应通过 `__syncwarp` 重新汇聚 warp，然后再调用 arrive-on 操作。

#### 4.5.3. 早期退出 {#453-早期退出}

有时需要从循环中提前返回，但仍然保持同步不变量。为此，HGGC 提供了 `arrive_and_drop()` 方法，它减少屏障的预期到达计数，然后从屏障中删除调用线程。

**HGGC C++ hggc::barrier**

```cpp
#include <hggc/barrier>
#include <cooperative_groups.h>

__device__ bool should_terminate();

__global__ void adaptive_worker(int iterations)
{
  __shared__ hggc::barrier<hggc::thread_scope_block> sync_bar;
  auto grp = cooperative_groups::this_thread_block();

  if (grp.thread_rank() == 0)
  {
    init(&sync_bar, grp.size());
  }
  grp.sync();

  for (int step = 0; step < iterations; ++step)
  {
    if (should_terminate())
    {
      sync_bar.arrive_and_drop();
      return;
    }
    auto phase = sync_bar.arrive();

    /* 在 arrive 与 wait 之间可执行不依赖其他线程的独立计算 */

    sync_bar.wait(std::move(phase));

    /* 所有存活线程均已到达，可安全读取共享数据 */
  }
}
```


#### 4.5.4. 完成函数（Completion Function） {#454-完成函数completion-function}

HGGC 支持在屏障完成后立即执行自定义代码的能力。可以通过向屏障构造函数传递可调用对象来实现这一点。

**HGGC C++ hggc::barrier**

```cpp
#include <hggc/barrier>
#include <cooperative_groups.h>
#include <functional>
namespace cg = cooperative_groups;

__device__ int branch_compute(int *, int);
__device__ int solo_work(int *, int);

__global__ void block_reduce(int *input, int totalElems, int *result)
{
  auto grp = cg::this_thread_block();

  constexpr int THREADS_PER_BLK = 128;
  __shared__ int tile[THREADS_PER_BLK];
  assert(THREADS_PER_BLK == grp.size());
  assert(totalElems % THREADS_PER_BLK == 0);

  auto on_phase_complete = [&]
  {
     int partial = 0;
     for (int k = 0; k < THREADS_PER_BLK; ++k)
     {
       partial += tile[k];
     }
     *result += partial;
  };

  using callback_t = decltype(on_phase_complete);
  using bar_type = hggc::barrier<hggc::thread_scope_block,
                                 callback_t>;
  __shared__ std::aligned_storage<sizeof(bar_type),
                                  alignof(bar_type)>
      bar_mem;

  bar_type *sync_bar = reinterpret_cast<bar_type*>(&bar_mem);

  if (grp.thread_rank() == 0)
  {
    new (sync_bar) bar_type(grp.size(), on_phase_complete);
  }
  grp.sync();

  for (int i = 0; i < totalElems; i += THREADS_PER_BLK)
  {
    tile[grp.thread_rank()] = input[i] + *result;
    auto phase = sync_bar->arrive();
    // 此处可执行独立计算。
    sync_bar->wait(std::move(phase));
  }

  grp.sync();

  if (grp.thread_rank() == 0)
  {
    sync_bar->~bar_type();
  }
  grp.sync();
}
```

#### 4.5.5. 使用屏障的生产者-消费者模式 {#455-使用屏障的生产者-消费者模式}

生产者-消费者模式是一种常见的多线程编程范式，其中一组线程（生产者）生成数据，另一组线程（消费者）处理该数据。异步屏障非常适合协调这种模式，特别是当涉及缓冲区管理时。

在典型的生产者-消费者场景中，生产者线程填充缓冲区，发出信号表示缓冲区已满；然而，生产者线程不等待此信号。为了实现完全的生产者/消费者并发，这种模式至少有双缓冲，每个缓冲区都需要两个屏障。

**HGGC C++ hggc::barrier**

```cpp
using barrier_t = hggc::barrier<hggc::thread_scope_block>;

__device__ void fill_buffer(barrier_t buf_available[], barrier_t buf_ready[], float *shared_data, int chunk_size, float *src, int total)
{
   for (int step = 0; step < total / chunk_size; ++step)
   {
     buf_available[step % 2].arrive_and_wait(); /* 等待 slot (step%2) 可写入 */

     /* 将源数据写入 shared_data 的对应 slot */
     barrier_t::arrival_token tok = buf_ready[step % 2].arrive(); /* 通知 slot (step%2) 数据就绪 */

   }
}

__device__ void process_buffer(barrier_t buf_available[], barrier_t buf_ready[], float *shared_data, int chunk_size, float *dst, int total)
{
   barrier_t::arrival_token tok_a = buf_available[0].arrive(); /* slot 0 初始可写 */

   barrier_t::arrival_token tok_b = buf_available[1].arrive(); /* slot 1 初始可写 */

   for (int step = 0; step < total / chunk_size; ++step)
   {
     buf_ready[step % 2].arrive_and_wait(); /* 等待 slot (step%2) 数据就绪 */

     /* 处理 shared_data 的 slot (step%2) */
     buf_available[step % 2].arrive(); /* 释放 slot (step%2) 以供下次写入 */
   }
}

__global__ void dual_buffer_pipeline(int total, float *src, float *dst, int chunk_size)
{
   constexpr int warpSize = 32;

      /* 共享内存分为两个 slot，每个大小为 chunk_size，
         交替用于写入和处理 */
      extern __shared__ float shared_data[];

   /* sync[0]/sync[1] 标记 slot 0/1 是否可写入，
      sync[2]/sync[3] 标记 slot 0/1 的数据是否就绪 */

   #pragma hggc_diag_suppress static_var_with_dynamic_init
   __shared__ hggc::barrier<hggc::thread_scope_block> sync[4];

    if (threadIdx.x < 4)
    {
      init(&sync[threadIdx.x], blockDim.x);
    }
    __syncthreads();
}
```

在这个例子中，第一个 warp 专门负责填充（写入）数据，其余 warps 负责处理（读取）数据。所有线程都参与每个屏障的 arrive 操作（调用 `sync[].arrive()` 或 `sync[].arrive_and_wait()`），因此每个屏障的预期到达计数等于 `blockDim.x`。

写入线程等待处理线程通过 `buf_available` 屏障发出 slot 可写的信号。等待屏障需要先调用 `arrive()` 获取令牌，再用令牌调用 `wait(token)`。`arrive_and_wait()` 将两步合并为一次调用。

```cpp
sync[k].arrive_and_wait();
/* 相当于 */
sync[k].wait(sync[k].arrive());
```

写入线程完成数据填充后，通过到达 `buf_ready` 屏障通知数据就绪（`buf_ready[step%2].arrive()`）。写入线程此时不阻塞，而是继续等待下一个 slot 变为可写——这正是双缓冲的核心：一个 slot 正在被处理的同时，另一个 slot 可以被写入。

处理线程启动时先将两个 slot 都标记为可写。随后在每次迭代中，处理线程等待当前 slot 的数据就绪（`buf_ready[step%2].arrive_and_wait()`），完成处理后释放该 slot（`buf_available[step%2].arrive()`），然后继续等待下一个 slot。


#### 4.5.6. 流水线 (Pipelines) {#456-流水线-pipelines}

流水线是高级同步原语中介绍的一种机制，用于分阶段处理工作并协调多缓冲生产者-消费者模式，常用于重叠计算与异步数据复制。

本节主要介绍如何通过 hggc::pipeline API 使用流水线。

#### 4.5.6.1. 初始化 {#4561-初始化}
`hggc::pipeline` 可以在不同的线程作用域中创建。对于除 `hggc::thread_scope_thread` 之外的作用域，需要一个 `hggc::pipeline_shared_state<scope, count>` 对象来协调参与的线程。此状态封装了有限资源，使流水线能够处理多达 `count` 个并发阶段。

```cpp
// 在线程作用域创建流水线
constexpr auto scope = hggc::thread_scope_thread;
hggc::pipeline<scope> pipeline = hggc::make_pipeline();

// 在块作用域创建流水线
constexpr auto scope = hggc::thread_scope_block;
constexpr auto stages_count = 2;
__shared__ hggc::pipeline_shared_state<scope, stages_count> shared_state;
auto pipeline = hggc::make_pipeline(group, &shared_state);
```

流水线可以是统一的 (unified) 或分区的 (partitioned)。在统一的流水线中，所有参与线程既是生产者又是消费者。在分区的流水线中，每个参与线程要么是生产者要么是消费者，并且在其生命周期内其角色不能改变。线程本地流水线不能被分区。要创建分区流水线，我们需要向 hggc::make_pipeline() 提供生产者的数量或线程的角色。

```cpp
// 创建一个分区的块级流水线，其中只有线程0是生产者
constexpr auto scope = hggc::thread_scope_block;
constexpr auto stages_count = 2;
__shared__ hggc::pipeline_shared_state<scope, stages_count> shared_state;
auto thread_role = (group.thread_rank() == 0) ? hggc::pipeline_role::producer : hggc::pipeline_role::consumer;

auto pipeline = hggc::make_pipeline(group, &shared_state, thread_role);
```

为了支持分区，共享的 hggc::pipeline 会产生额外开销，包括每阶段使用一组共享内存屏障进行同步。即使流水线是统一的并且可以改用__syncthreads() 也如此。因此，在可能的情况下，使用线程本地流水线可避免这些开销。

#### 4.5.6.2. 提交工作 {#4562-提交工作}
将工作提交到 pipline 流水线阶段涉及:
- 使用 pipeline.producer_acquire() 从一组生产者线程集体获取流水线头部。
- 向流水线头提交异步操作，例如 memcpy_async。
- 使用 pipeline.producer_commit() 集体提交 (推进) 流水线头。

如果所有资源都在使用中，pipeline.producer_acquire() 会阻塞生产者线程，直到消费者线程释放下一阶段的资源。

#### 4.5.6.3. 消费工作 {#4563-消费工作}
从前一个提交的阶段消费工作涉及:
- 从一组消费者线程集体等待阶段完成，例如使用 pipeline.consumer_wait() 等待尾部 (最老的) 阶段。
- 集体使用 pipeline.consumer_release() 释放阶段。

对于 `hggc::pipeline<hggc:thread_scope_thread>` 还可以使用 hggc::pipeline_consumer_wait_prior<N>() 友元函数等待除最后 N 个阶段外的所有阶段完成，类似于 primitives API 中的__pipeline_wait_prior(N)。

#### 4.5.6.4. Warp 纠缠 {#4564-warp-纠缠}
流水线机制在同一流水线内的 HGGC 线程间共享。这种共享导致提交的操作序列在 warp 内纠缠，这在某些情况下会影响性能。

__Commit__：提交操作被合并，这样流水线的序列对所有收敛线程递增一次，它们提交的操作被批量组合在一起。如果 warp 完全收敛，则序列递增一，所有提交操作将在流水线的同一阶段批处理；如果 warp 完全发散，则序列递增至 32，所有提交操作将分散到不同阶段。

- 设 PB 为 warp 共享流水线的实际操作序列。
  PB = {BP0, BP1, BP2,…, BPL}

- 设 TB 为线程感知的操作序列，就像序列仅由该线程调用 commit 操作递增一样。
  TB = {BT0, BT1, BT2,…, BTL}
  pipeline::producer_commit() 返回值来自线程感知批次序列。

- 线程感知序列中的索引总是与实际 warp 共享序列中相等或更大的索引对齐。仅当所有提交操作从完全收敛线程调用时，序列才相等。
  BTn ≤ BPm 其中 n <= m

例如，当 warp 完全发散时：
- warp 共享流水线的实际序列为：PB = {0, 1, 2, 3, ..., 31} (PL=31)。
- 此 warp 各线程的感知序列为：
  - 线程 0：TB = {0} (TL=0)
  - 线程 1：TB = {0} (TL=0)
  - ...
  - 线程 31：TB = {0} (TL=0)

__Wait__：HGGC 线程调用 pipeline::consumer_wait() 或 pipeline_consumer_wait_prior<N>() 等待感知序列 TB 中的批次完成。注意 pipeline::consumer_wait() 等价于 pipeline_consumer_wait_prior<N>(), 其中 N = PL。

等待 prior 变体等待实际序列中至少到包括 PL-N 的批次。由于 TL <= PL，等待最多到包括 PL-N 的批次包含等待 TL-N 批次。因此，当 TL < PL 时，线程将无意中等待额外的、较新的批次。在上述极端完全发散 warp 示例中，每个线程可能等待全部 32 个批次。

!!! note

    建议通过汇聚线程执行 commit 调用，以免过度等待，保持线程感知批次序列与实际序列对齐。当这些操作前的代码使线程发散时，应在调用提交操作前通过 `__syncwarp` 重新汇聚 warp。

#### 4.5.6.5. 提前退出 {#4565-提前退出}
当参与流水线的线程必须提前退出时，该线程必须在退出前显式放弃参与，使用 hggc::pipeline::quit()。剩余参与线程可以正常进行后续操作。

#### 4.5.6.6. 跟踪异步内存操作 {#4566-跟踪异步内存操作}
以下示例演示如何使用流水线独立提交内存副本并等待它们完成和消费数据，从而集体地将数据从全局内存复制到共享内存。更多关于异步数据副本的信息，请参见 [4.5.7 节 异步数据拷贝](#457-异步数据拷贝-asynchronous-data-copies)。

**HGGC C++ hggc::pipeline 示例**
```cpp
#include <hggc/pipeline>

__global__ void example_kernel(const float *in)
{
    constexpr int block_size = 128;
    __shared__ __align__(sizeof(float)) float buffer[4 * block_size];

    // 为每个线程创建统一流水线
    hggc::pipeline<hggc::thread_scope_thread> pipeline = hggc::make_pipeline();

    // 内存复制第一阶段
    pipeline.producer_acquire();
    // 每个线程获取第一块的一个元素
    hggc::memcpy_async(buffer, in, sizeof(float), pipeline);
    pipeline.producer_commit();

     // 内存复制第二阶段
     pipeline.producer_acquire();
     // 每个线程获取第二和第三块的一个元素
     hggc::memcpy_async(buffer + block_size, in + block_size, sizeof(float), pipeline);

     hggc::memcpy_async(buffer + 2 * block_size, in + 2 * block_size, sizeof(float), pipeline);

     pipeline.producer_commit();

     // 内存复制第三阶段
     pipeline.producer_acquire();
     // 每个线程获取最后一块的一个元素
     hggc::memcpy_async(buffer + 3 * block_size, in + 3 * block_size, sizeof(float), pipeline);

     pipeline.producer_commit();

    // 等待最旧阶段(等待第一阶段)
    pipeline.consumer_wait();
    pipeline.consumer_release();

    // __syncthreads();
    // 使用第一阶段的数据

    // 等待最旧阶段(等待第二阶段)
    pipeline.consumer_wait();
    pipeline.consumer_release();

    // __syncthreads();
    // 使用第二阶段的数据

    // 等待最旧阶段(等待第三阶段)
    pipeline.consumer_wait();
    pipeline.consumer_release();

    // __syncthreads();
    // 使用第三阶段的数据
}
```

#### 4.5.6.7. 使用流水线的生产者-消费者模式 {#4567-使用流水线的生产者-消费者模式}
在第 4.5.5 节中，我们展示了如何空间划分线程块使用异步屏障实现生产者-消费者模式。有了 hggc::pipeline，可以通过单一分区流水线配合每个数据缓冲区一个阶段来简化实现，而不是每个缓冲区使用两个异步屏障。

#### 4.5.7. 异步数据拷贝 (Asynchronous Data Copies) {#457-异步数据拷贝-asynchronous-data-copies}

本节提供了在 PPU 内存层次结构内进行异步数据移动的详细指导和示例。它涵盖了用于元素级拷贝的 async_cp、用于批量（一维和多维）传输的张量内存拷贝器 AIU，并展示了这些机制如何与异步屏障和流水线集成。

#### 4.5.7.1. 使用 async_cp {#4571-使用async_cp}

许多 HGGC 应用程序需要在全局内存和共享内存之间频繁进行数据移动。通常，这涉及复制较小的数据元素或执行不规则的内存访问模式。使用 async_cp 的主要目标是提供一种从全局 (global) 内存到共享 (shared) 内存的高效异步数据传输机制，用于较小的逐元素数据传输，同时通过重叠执行更好地利用计算资源。

##### 4.5.7.1.1. ppu.async.cp {#45711-ppuasynccp}
这部分详细介绍可以参考 [async_cp 文档](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#762-非批量拷贝指令ppucpasync)。
```asm
 ppu.cp.async.ca.shared.global  [shrd],    [gbl + 4], 4;
```
`ppu.cp.async` 是一条非阻塞指令，它发起一个异步拷贝操作，将数据从源地址操作数 `src` 指定的位置拷贝到目标地址操作数 `dst` 指定的位置。操作数 `src` 指定全局状态空间（全局内存）中的位置，`dst` 指定共享状态空间（共享内存）中的位置。
操作数 `cp-size` 是一个整数常量，指定要拷贝到 `dst` 的数据大小（以字节为单位）。
指令 `ppu.cp.async` 允许可选地指定一个 32 位整数操作数 `src-size`。操作数 `src-size` 表示从 `src` 拷贝到 `dst` 的数据大小（以字节为单位），且必须小于 `cp-size`。在这种情况下，目标 `dst` 中的剩余字节将用零填充。
异步拷贝操作仅支持从全局内存到共享内存的方向。指针需要根据要复制的数据大小对齐到 4、8 或 16 字节。当共享内存和全局内存的对齐都是 128 字节时，可以获得最佳性能。使用 LDGSTS 的数据传输是异步的，并建模为异步线程操作。这允许发起线程在硬件异步复制数据的同时继续计算。

##### 4.5.7.1.2. 条件代码中的批处理加载 {#45712-条件代码中的批处理加载}

条件代码经常会导致分支发散，从而降低性能。例如，如果在一个循环中执行多个小负载，那么只有满足条件的那些线程才会执行加载，导致其他线程闲置。在这种情况下，我们可以使用异步拷贝直接从全局内存加载数据到共享内存。这不仅通过直接将数据复制到共享内存减少了寄存器使用，还确保所有从全局内存的加载都在飞行中。

 HGGC C++ hggc::memcpy_async

```cpp
#include <cooperative_groups.h>
#include <hggc/barrier>

__global__ void block_relu_kernel(const float *left, const float *center, const float *right)
{
     auto block = cooperative_groups::this_thread_block();
     auto thread = cooperative_groups::this_thread();
     using barrier_t = hggc::barrier<hggc::thread_scope_block>;
     __shared__ barrier_t barrier;
     __shared__ float buffer[8 + 32 + 8];

     // 初始化同步对象。
     if (block.thread_rank() == 0) {
         init(&barrier, block.size());
     }
     __syncthreads();

     // 版本1: 在各个线程中发出拷贝。
     if (tid < 8) {
         hggc::memcpy_async(buffer + tid, left + tid, hggc::aligned_size_t<4>(sizeof(float)), barrier); // 左halo

         // 或 hggc::memcpy_async(thread, buffer + tid, left + tid, hggc::aligned_size_t<4>(sizeof(float)), barrier);

     } else if (tid >= 32 - 8) {
         hggc::memcpy_async(buffer + tid + 16, right + tid, hggc::aligned_size_t<4>(sizeof(float)), barrier); // 右halo

         // 或 hggc::memcpy_async(thread, buffer + tid + 16, right + tid, hggc::aligned_size_t<4>(sizeof(float)), barrier);

     }

     // 等待拷贝完成。
     auto token = barrier.arrive();
     barrier.wait(std::move(token));
     __syncthreads();
}
```

 HGGC C++ cooperative_groups::memcpy_async

```cpp
#include <cooperative_groups.h>
#include <cooperative_groups/memcpy_async.h>

namespace cg = cooperative_groups;

__global__ void block_relu_kernel(const float *left, const float *center, const float *right)
{
     cg::thread_block block = cg::this_thread_block();
     // 左halo (8个元素) - 中心 (32个元素) - 右halo (8个元素).

     __shared__ float buffer[8 + 32 + 8];

     // 跨所有线程协作发出拷贝。
     cg::memcpy_async(block, buffer, left, 8 * sizeof(float)); // 左halo
     cg::memcpy_async(block, buffer + 8, center, 32 * sizeof(float)); // 中心
     cg::memcpy_async(block, buffer + 40, right, 8 * sizeof(float)); // 右halo

     cg::wait(block); // 等待所有拷贝完成。
     __syncthreads();

      // 计算stencil。
}
```

##### 4.5.7.1.3. 预取数据 {#45713-预取数据}

在这个例子中，我们将演示如何使用异步数据拷贝从全局内存预取数据到共享内存。在迭代拷贝和计算模式中，这允许隐藏未来迭代的数据传输延迟与当前迭代上的计算，可能增加正在传输的字节数。

 HGGC C++ hggc::memcpy_async

```cpp
#include <cooperative_groups.h>
#include <hggc/pipeline>

template <size_t num_stages = 2 /* 具有num_stages阶段的流水线 */>
__global__ void prefetch_kernel(int* global_out, int const* global_in, size_t size, size_t batch_size) {

     auto grid = cooperative_groups::this_grid();
     auto block = cooperative_groups::this_thread_block();
     auto thread = cooperative_groups::this_thread();
     assert(size == batch_size * grid.size()); // 假设输入大小适合batch_size * grid_size

     extern __shared__ int shared[]; // num_stages * block.size() * sizeof(int) 字节
     size_t shared_offset[num_stages];
     for (int s = 0; s < num_stages; ++s) shared_offset[s] = s * block.size();

     hggc::pipeline<hggc::thread_scope_thread> pipeline = hggc::make_pipeline();

     auto block_batch = [&](size_t%20batch) -> int {
            return block.group_index().x * block.size() + grid.size() * batch;
     };

     // 用前"num_stages"批次填充流水线。
     for (int s = 0; s < num_stages; ++s) {
         size_t block_batch_idx = block_batch(s);

         hggc::memcpy_async(
             shared + shared_offset[s],
             global_in + block_batch_idx,
             hggc::aligned_size_t<4>(block.size() * sizeof(int)),
             pipeline);

         pipeline.producer_commit();
     }

     // 加载未来阶段"num_stages"超出当前计算批次。
     pipeline.producer_acquire();
}
```

##### 4.5.7.1.4. 批量拷贝指令（AIU copy） {#45714-批量拷贝指令aiu-copy}

AIU 是 PPU 中引入的一种专门负责数据搬运的硬件机制，支持更高效的异步数据加载，主要用于在全局内存和共享内存之间高效传输多维张量数据。AIU 具体特点：
  * 指令编程模型为异步、bulk 风格，AIU 是 warp-level 的指令。
  * 支持多维数据的异步搬移。
  * AIU 写共享内存时采用 swizzle memory layout 来解决 bank conflicts，开发者无需感知具体 swizzle layout。
  * 当开发者从共享内存往寄存器加载 AIU 写入的数据时，要用 PPU 的共享内存上带 Swizzle 功能的读指令解析 swizzle layout，并按 Tile 为单位完成数据加载。

有关 AIU 的详细指令介绍，可以参考 [aiu-copy 文档](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#763-批量拷贝指令aiu-copy)。

### 4.6. 进程间通信与虚拟内存管理 {#46-进程间通信与虚拟内存管理}


由不同宿主进程管理的多块 PPU 之间的通信，可通过使用进程间通信（IPC）API 与可进行 IPC 共享的内存缓冲区来实现：通过创建可跨进程移植的句柄，随后用这些句柄获取指向对端 PPU 设备内存的、在本进程内有效的设备指针（process-local device pointers）。

同一进程内，由某个宿主线程创建的任意设备内存指针或事件句柄，都可以被该进程内的其他线程直接引用。然而，设备指针或事件句柄在创建它们的进程之外无效，因此不能被属于不同进程的线程直接引用。要跨进程访问设备内存与 HGGC 事件，应用必须使用 HGGC 进程间通信（IPC）或虚拟内存管理（VMM）API 来创建可跨进程移植的句柄，并使用标准宿主操作系统 IPC 机制（例如进程间共享内存或文件）将其共享给其他进程。一旦进程间交换了这些可移植句柄，就必须使用 HGGC IPC 或 VMM API 从句柄获取在本进程内有效的设备指针。本地设备指针随后就可以像在单进程内那样使用。

用于单节点、单操作系统实例内 IPC 的这种可移植句柄方法，也同样用于多节点互连集群中 PPU 之间的点对点通信。在多节点场景中，参与通信的 PPU 由运行在各个集群节点上、彼此独立的操作系统实例中的进程所管理，因此需要在操作系统实例之上提供额外的抽象层。多节点对等通信通过在多节点 PPU 对等体之间创建并交换所谓的 "fabric" 句柄来实现；随后在参与的各进程与各操作系统实例（对应多节点 rank）内获取本地设备指针。

有关用于建立与交换可跨进程、以及可跨节点与操作系统实例的可移植句柄（并据此获得本地设备指针以进行 PPU 通信）的具体 API，请参见 [4.6.1 节 使用传统进程间通信 API 的 IPC](#461-使用传统进程间通信-api-的-ipc)。

!!! note

    使用 HGGC IPC API 与虚拟内存管理（VMM）API 进行 IPC 时，各自具有不同的优势与限制。
    HGGC IPC API 目前只在 Linux 平台上受支持。
    HGGC 虚拟内存管理 API 允许在内存分配时按分配粒度控制对等可访问性与共享，但需要使用 HGGC 驱动 API。

#### 4.6.1. 使用传统进程间通信 API 的 IPC {#461-使用传统进程间通信-api-的-ipc}

要在进程间共享设备内存指针与事件，应用必须使用 HGGC 进程间通信 API（IPC），其详细说明见参考手册。IPC API 允许应用通过 `hggcIpcGetMemHandle()` 获取给定设备内存指针的 IPC 句柄。HGGC IPC 句柄可通过标准宿主操作系统 IPC 机制（例如进程间共享内存或文件）传递给另一个进程。`hggcIpcOpenMemHandle()` 使用该 IPC 句柄来取回一个在另一个进程内可用的有效设备指针。事件句柄也可以通过类似的入口点进行共享。

IPC API 的一个使用示例是：由单个主进程生成一批输入数据，并在无需重新生成或复制的情况下，将这些数据提供给多个从进程使用。

!!! note

    IPC API 只在 Linux 上受支持。
    请注意，IPC API 不支持用于 `hggcMallocManaged` 分配得到的内存。
    使用 HGGC IPC 彼此通信的应用，应使用相同版本的 HGGC driver 与 runtime 进行编译、链接与运行。
    出于性能原因，通过 `hggcMalloc()` 进行的分配可能会从更大的内存块中进行子分配（sub-allocate）。在这种情况下，HGGC IPC API 将共享整个底层内存块，这可能导致其他子分配也被共享，从而可能引发进程间的信息泄露。为避免此行为，建议仅共享大小按 8MiB 对齐的分配。


#### 4.6.2. 使用虚拟内存管理 API 的 IPC {#462-使用虚拟内存管理-api-的-ipc}

HGGC 虚拟内存管理 API 允许创建可进行 IPC 共享的内存分配，并且通过操作系统特定的 IPC 句柄数据结构来支持多种操作系统。

#### 4.6.3. 虚拟内存管理 {#463-虚拟内存管理}

**传统分配模型的局限**

`hggcMalloc()` 返回的设备内存地址可直接用于 HGGC API 与设备端 kernel。若需跨设备访问同一数据，开发者可通过 `hggcEnablePeerAccess` 启用对等访问。然而该方式存在两个问题：

- **粒度过粗**：启用对等访问后，当前及未来的所有分配都会被映射到目标设备，即使应用仅需共享少量分配，也不得不承担全量映射的运行时开销。
- **多节点扩展困难**：对等访问模型本质上难以扩展到多节点场景。

**虚拟内存管理（VMM）API**

HGGC 提供虚拟内存管理 API，将操作系统中「地址保留」与「物理提交」两阶段分离的思想引入 PPU 内存管理：开发者先保留一段连续的虚拟地址范围（不分配物理内存），再按需将物理 PPU 内存映射到该范围。这使得应用可以精确选择哪些分配对哪些设备可见，而不再依赖全局对等映射。

VMM API 的核心能力包括：

- **地址连续化**：将非连续的物理内存块映射到连续的虚拟地址空间，减少内存碎片、提升利用率，尤其适用于深度神经网络训练等大规模负载。
- **按需提交**：虚拟地址保留与物理内存分配相互独立，开发者可先保留大块地址区间再逐步映射物理内存，避免高成本的拷贝或重新分配。
- **动态增长**：在不拷贝已有数据的前提下扩展分配，效果类似 CPU 侧的 `realloc` 或 `std::vector` 扩容。
- **自定义分配器**：为构建复杂内存管理系统（如大语言模型中的 KV-cache 动态管理）提供底层原语，可改善吞吐与延迟。
- **多设备/多节点共享**：通过解耦虚拟地址与物理存储，允许创建统一虚拟地址空间并将数据动态映射到不同 PPU，优化跨设备通信并减少数据传输开销。

!!! note

    本节介绍的 API 需要系统支持 UVA。请参阅 [虚拟内存管理](#463-虚拟内存管理)。

#### 4.6.3.1. 预备知识 {#4631-预备知识}

##### 4.6.3.1.1. 定义 {#46311-定义}

**Fabric 内存：** Fabric 内存是指可通过高速互连 fabric 访问的内存。该 fabric 为多个 PPU 或节点之间提供内存一致性与高带宽通信层，使它们能够像访问连接在统一 fabric 上的内存一样高效共享内存，而不是各自孤立在单个设备上。

HGGC 更高版本提供了一种 VMM 分配句柄类型 `HG_MEM_HANDLE_TYPE_FABRIC`。在受支持的平台上，并且 IMEX 守护进程正在运行时，该句柄类型不仅允许在节点内通过任意通信机制（例如 MPI）共享分配，还允许跨节点共享。这使得多节点 Icnlink 系统中的 PPU 即使位于不同节点，也能映射同一 fabric 内所有其他 PPU 的内存。

**内存句柄：** 在 VMM 中，句柄是表示物理内存分配的不透明标识符。句柄是低层 HGGC VMM API 中管理内存的核心，它使得可被映射进虚拟地址空间的物理内存对象能够被灵活控制。一个句柄唯一标识一个物理内存分配。句柄作为对内存资源的抽象引用而不暴露直接指针。句柄支持跨进程或跨设备导出与导入等操作，从而促进内存共享与虚拟化。

**IMEX 通道：** IMEX 的名称来自 _internode memory exchange_，是跨节点 PPU 到 PPU 通信解决方案的一部分。IMEX 通道是 PPU 驱动的一项特性，在 IMEX 域内为多用户或多节点环境提供基于用户的内存隔离。IMEX 通道用作安全与隔离机制。

IMEX 通道与 fabric 句柄直接相关，并且在多节点 PPU 通信中必须启用。当 PPU 分配内存并希望让不同节点上的 PPU 可访问时，首先需要为该内存导出一个句柄。IMEX 通道用于导出过程，以生成一个安全的 fabric 句柄；该句柄只能被具有正确通道访问权限的远端进程导入。

**单播内存访问：** 在 VMM API 语境下，单播内存访问指由特定设备或进程对某段物理内存进行受控、直接的映射与访问，将其映射到一段唯一的虚拟地址范围。与向多个设备广播访问不同，单播表示某个特定 PPU 被显式授予对所保留虚拟地址范围（映射到某个物理内存分配）的读写权限。

**多播内存访问：** 在 VMM API 语境下，多播内存访问指通过多播机制，将单个物理内存分配或区域同时映射到多个设备的虚拟地址空间。这使得数据能够以"一对多"的方式在多块 PPU 之间高效共享，减少冗余的数据传输并提高通信效率。HGGC VMM API 支持创建一个多播对象，将来自多个设备的物理内存分配绑定在一起。

##### 4.6.3.1.2. 查询支持情况 {#46312-查询支持情况}

应用在尝试使用这些特性之前，应先查询其支持情况，因为可用性可能随 PPU 架构、驱动版本与所使用的软件库而变化。后续小节说明了如何以编程方式检查必要的支持。

**VMM 支持** 在尝试使用 VMM API 之前，应用必须确保要使用的设备支持 HGGC 虚拟内存管理。下面的代码示例展示了如何查询 VMM 支持：


```cpp
int deviceSupportsVmm;
HGresult result = hgDeviceGetAttribute(&deviceSupportsVmm, HG_DEVICE_ATTRIBUTE_VIRTUAL_MEMORY_MANAGEMENT_SUPPORTED, device);
if (deviceSupportsVmm != 0) {
    // `device` supports Virtual Memory Management
}
```


**Fabric 内存支持：** 在尝试使用 fabric 内存之前，应用必须确保要使用的设备支持 fabric 内存。下面的代码示例展示了如何查询 fabric 内存支持：


```cpp
int deviceSupportsFabricMem;
HGresult result = hgDeviceGetAttribute(&deviceSupportsFabricMem, HG_DEVICE_ATTRIBUTE_HANDLE_TYPE_FABRIC_SUPPORTED, device);
if (deviceSupportsFabricMem != 0) {
    // `device` supports Fabric Memory
}
```


除了使用 `HG_MEM_HANDLE_TYPE_FABRIC` 作为句柄类型以及在交换可共享句柄时不需要 OS 原生的进程间通信机制之外，fabric 内存的使用方式与其他分配句柄类型没有区别。

**IMEX 通道支持** 在 IMEX 域内，IMEX 驱动通过创建字符设备 `alixpu-caps-imex-channels` 来实现这一点。要使用基于 fabric 句柄的共享，开发者应验证两点：

  * 首先，应用必须验证该设备是否存在于 /proc/dev 下：

```bash
cat /proc/dev | grep alixpu

alixpu

alixpu_ctl

alixpu-caps-imex-channels
```


  * 其次，要让两个 HGGC 进程（导出者与导入者）共享内存，它们必须都能访问同一个 IMEX 通道文件。这些文件（例如 /dev/alixpu-caps-imex-channels/channel0）是表示单个 IMEX 通道的节点。系统管理员必须创建这些文件，例如使用 mknod() 命令。

```bash
mknod /dev/alixpu-caps-imex-channels/channelN c <major_number> 0

该命令使用从 /proc/dev 获取的主设备号来创建 channelN。
```

#### 4.6.3.2. API 概览 {#4632-api-概览}

VMM API 为开发者提供了对虚拟内存管理的细粒度控制。由于 VMM 是非常底层的 API，因此需要直接使用 [HGGC 驱动 API](#31-hggc-底层驱动接口)。该通用 API 可用于单节点与多节点环境。

要有效使用 VMM，开发者必须牢固掌握内存管理中的一些关键概念：
- 了解操作系统虚拟内存基础，包括如何处理页面与地址空间
- 需要理解内存层级与硬件特性
- 熟悉进程间通信（IPC）方法，例如 socket 或消息传递
- 具备内存访问权限安全性的基本知识

VMM API 工作流涉及一系列内存管理步骤，并重点关注在不同设备或进程之间共享内存。首先，开发者必须在源设备上分配物理内存。为了便于共享，VMM API 使用句柄来向目标设备或进程传递必要信息。开发者必须导出一个用于共享的句柄，它可以是 OS 特定句柄或 fabric 特定句柄。OS 特定句柄仅限于单节点上的进程间通信；fabric 特定句柄更通用，既可用于单节点也可用于多节点环境。需要注意的是，使用 fabric 特定句柄需要启用 IMEX 通道。

句柄导出后，必须通过某种进程间通信协议与接收进程共享，具体方法由开发者自行选择。接收进程随后使用 VMM API 导入该句柄。在句柄成功导出、共享并导入后，源进程与目标进程都必须保留一段将要映射该物理内存的虚拟地址空间。最后一步是为每个设备设置内存访问权限，确保建立正确的权限。上述全过程（包含两种句柄路径）在配图中有更详细的说明。

#### 4.6.3.3. 单播内存共享 {#4633-单播内存共享}

PPU 内存共享既可以发生在一台机器的多块 PPU 之间，也可以跨越多台机器的网络。该过程遵循以下步骤：

  * 分配并导出：某个 PPU 上的 HGGC 程序分配内存并为其获取可共享句柄。

  * 共享并导入：随后通过 IPC、MPI 或 PCCL 等将句柄发送给节点上的其他程序。在接收端 PPU 上，HGGC 驱动导入句柄并创建所需的内存对象。

  * 保留并映射：驱动在程序的虚拟地址（VA）与 PPU 物理地址（PA）及其网络 fabric 地址（FA）之间创建映射。

  * 访问权限：为该分配设置访问权限。

  * 释放内存：在程序结束执行时释放所有分配。


##### 4.6.3.3.1. 分配并导出 {#46331-分配并导出}

**分配物理内存** 使用虚拟内存管理 API 进行内存分配的第一步，是创建一块物理内存作为该分配的支撑（backing）。要分配物理内存，应用必须使用 `hgMemCreate` API。该函数创建的分配不具有任何设备或主机映射。参数 `HGmemGenericAllocationHandle` 用于描述要分配内存的属性，例如分配位置、是否要共享到其他进程（或图形 API）、或待分配内存的物理属性。开发者必须确保请求分配的大小按适当粒度对齐。关于分配粒度要求的信息可通过 `hgMemGetAllocationGranularity` 查询。

OS 特定句柄（Linux）


```cpp
HGmemGenericAllocationHandle allocatePhysicalMemory(int device, size_t size) {
    HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR;
    HGmemAllocationProp prop = {};
    prop.type = HG_MEM_ALLOCATION_TYPE_PINNED;
    prop.location.type = HG_MEM_LOCATION_TYPE_DEVICE;
    prop.location.id = device;
    prop.requestedHandleTypes = handleType;

    size_t granularity = 0;
    hgMemGetAllocationGranularity(&granularity, &prop, HG_MEM_ALLOC_GRANULARITY_MINIMUM);

    // Ensure size matches granularity requirements for the allocation
    size_t padded_size = ROUND_UP(size, granularity);

    // Allocate physical memory
    HGmemGenericAllocationHandle allocHandle;
    hgMemCreate(&allocHandle, padded_size, &prop, 0);

    return allocHandle;
}
```


Fabric 句柄


    HGmemGenericAllocationHandle allocatePhysicalMemory(int device, size_t size) {
        HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_FABRIC;
        HGmemAllocationProp prop = {};
        prop.type = HG_MEM_ALLOCATION_TYPE_PINNED;
        prop.location.type = HG_MEM_LOCATION_TYPE_DEVICE;
        prop.location.id = device;
        prop.requestedHandleTypes = handleType;

        size_t granularity = 0;
        hgMemGetAllocationGranularity(&granularity, &prop, HG_MEM_ALLOC_GRANULARITY_MINIMUM);

        // Ensure size matches granularity requirements for the allocation
        size_t padded_size = ROUND_UP(size, granularity);

        // Allocate physical memory
        HGmemGenericAllocationHandle allocHandle;
        hgMemCreate(&allocHandle, padded_size, &prop, 0);

        return allocHandle;
    }


!!! note

    `hgMemCreate` 分配的内存由其返回的 `HGmemGenericAllocationHandle` 引用。请注意，这个引用不是指针，并且该内存此时尚不可访问。

!!! note

     可以使用 `hgMemGetAllocationPropertiesFromHandle` 查询分配句柄的属性。

**导出内存句柄** HGGC 虚拟内存管理 API 通过句柄提供了一种新的进程间通信机制，用于交换关于分配与物理地址空间的必要信息。可以导出 OS 特定 IPC 句柄或 fabric 特定 IPC 句柄。OS 特定 IPC 句柄只能用于单节点设置。fabric 特定句柄可用于单节点或多节点设置。

OS 特定句柄（Linux）


    HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR;
    HGmemGenericAllocationHandle handle = allocatePhysicalMemory(0, 1<<21);
    int fd;
    hgMemExportToShareableHandle(&fd, handle, handleType, 0);


Fabric 句柄


    HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_FABRIC;
    HGmemGenericAllocationHandle handle = allocatePhysicalMemory(0, 1<<21);
    HGmemFabricHandle fh;
    hgMemExportToShareableHandle(&fh, handle, handleType, 0);


!!! note

    OS 特定句柄要求所有进程属于同一个 OS。

!!! note

    fabric 特定句柄要求系统管理员启用 IMEX 通道。

##### 4.6.3.3.2. 共享并导入 {#46332-共享并导入}

**共享内存句柄** 一旦句柄被导出，就必须使用某种进程间通信协议将其共享给接收进程。开发者可以自由选择任何方法共享句柄。具体 IPC 方法取决于应用的设计与环境。常见方法包括 OS 特定的进程间 socket，以及分布式消息传递。OS 特定 IPC 提供高性能传输，但仅限于同一台机器上的进程且不具备可移植性。fabric 特定 IPC 更简单且更具可移植性，但需要系统级支持。所选方法必须安全、可靠地将句柄数据传输到目标进程，使其可用于导入内存并建立有效映射。IPC 方法选择的灵活性使得 VMM API 能够集成到广泛的系统架构中，从单节点应用到分布式多节点设置。下面的代码片段将给出使用 socket 编程与 MPI 共享与接收句柄的示例。

发送：OS 特定 IPC（Linux）


    int ipcSendShareableHandle(int socket, int fd, pid_t process) {
        struct msghdr msg;
        struct iovec iov[1];

        union {
            struct cmsghdr cm;
            char* control;
        } control_un;

        size_t sizeof_control = CMSG_SPACE(sizeof(int)) * sizeof(char);
        control_un.control = (char*) malloc(sizeof_control);

        struct cmsghdr *cmptr;
        ssize_t readResult;
        struct sockaddr_un cliaddr;
        socklen_t len = sizeof(cliaddr);

        // 构造客户端地址，以便将可共享句柄发送给目标进程
        memset(&cliaddr, 0, sizeof(cliaddr));
        cliaddr.sun_family = AF_UNIX;
        char temp[20];
        sprintf(temp, "%s%u", "/tmp/", process);
        strcpy(cliaddr.sun_path, temp);
        len = sizeof(cliaddr);

        // 将对应的可共享句柄发送给客户端
        int sendfd = fd;

        msg.msg_control = control_un.control;
        msg.msg_controllen = sizeof_control;

        cmptr = CMSG_FIRSTHDR(&msg);
        cmptr->cmsg_len = CMSG_LEN(sizeof(int));
        cmptr->cmsg_level = SOL_SOCKET;
        cmptr->cmsg_type = SCM_RIGHTS;

        memmove(CMSG_DATA(cmptr), &sendfd, sizeof(sendfd));

        msg.msg_name = (void *)&cliaddr;
        msg.msg_namelen = sizeof(struct sockaddr_un);

        iov[0].iov_base = (void *)"";
        iov[0].iov_len = 1;
        msg.msg_iov = iov;
        msg.msg_iovlen = 1;

        ssize_t sendResult = sendmsg(socket, &msg, 0);
        if (sendResult <= 0) {
            perror("IPC failure: Sending data over socket failed");
            free(control_un.control);
            return -1;
        }

        free(control_un.control);
        return 0;
    }


发送：Fabric IPC


    MPI_Send(&fh, sizeof(HGmemFabricHandle), MPI_BYTE, 1, 0, MPI_COMM_WORLD);


接收：OS 特定 IPC（Linux）


    int ipcRecvShareableHandle(int socket, int* fd) {
        struct msghdr msg = {0};
        struct iovec iov[1];
        struct cmsghdr cm;

        // 使用联合体以满足控制数组的对齐要求
        union {
            struct cmsghdr cm;
            // 此写法在 QNX 上不可用，因为 QNX 的 CMSG_SPACE 调用了 __cmsg_alignbytes，
            // 而 __cmsg_alignbytes 是运行时函数而非编译期宏
            // char control[CMSG_SPACE(sizeof(int))]
            char* control;
        } control_un;

        size_t sizeof_control = CMSG_SPACE(sizeof(int)) * sizeof(char);
        control_un.control = (char*) malloc(sizeof_control);
        struct cmsghdr *cmptr;
        ssize_t n;
        int receivedfd;
        char dummy_buffer[1];
        ssize_t sendResult;
        msg.msg_control = control_un.control;
        msg.msg_controllen = sizeof_control;

        iov[0].iov_base = (void *)dummy_buffer;
        iov[0].iov_len = sizeof(dummy_buffer);

        msg.msg_iov = iov;
        msg.msg_iovlen = 1;
        if ((n = recvmsg(socket, &msg, 0)) <= 0) {
            perror("IPC failure: Receiving data over socket failed");
            free(control_un.control);
            return -1;
        }

        if (((cmptr = CMSG_FIRSTHDR(&msg)) != NULL) &&
            (cmptr->cmsg_len == CMSG_LEN(sizeof(int)))) {
            if ((cmptr->cmsg_level != SOL_SOCKET) || (cmptr->cmsg_type != SCM_RIGHTS)) {
            free(control_un.control);
            return -1;
            }

            memmove(&receivedfd, CMSG_DATA(cmptr), sizeof(receivedfd));
            *fd = receivedfd;
        } else {
            free(control_un.control);
            return -1;
        }

        free(control_un.control);
        return 0;
    }


接收：Fabric IPC


    MPI_Recv(&fh, sizeof(HGmemFabricHandle), MPI_BYTE, 1, 0, MPI_COMM_WORLD);


**导入内存句柄** 同样，开发者可以导入 OS 特定 IPC 或 fabric 特定 IPC 的句柄。OS 特定 IPC 句柄只能用于单节点；fabric 特定句柄可用于单节点或多节点。

OS 特定句柄（Linux）


    HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR;
    hgMemImportFromShareableHandle(handle, (void*) &fd, handleType);


Fabric 句柄


    HGmemAllocationHandleType handleType = HG_MEM_HANDLE_TYPE_FABRIC;
    hgMemImportFromShareableHandle(handle, (void*) &fh, handleType);


##### 4.6.3.3.3. 保留并映射 {#46333-保留并映射}

**保留虚拟地址范围**

由于在 VMM 中地址与内存的概念是分离的，应用必须划出一段能够容纳 `hgMemCreate` 所创建内存分配的地址范围。所保留的地址范围至少应当不小于用户计划放入其中的所有物理内存分配大小之和。

应用可通过向 `hgMemAddressReserve` 传递适当参数来保留虚拟地址范围。获得的地址范围不会关联任何设备或主机物理内存。该保留的虚拟地址范围可以映射到系统中任意设备所属的内存块，因此应用可以获得一段连续的 VA 范围，而其背后由不同设备的内存支撑并映射。应用应当使用 `hgMemAddressFree` 将虚拟地址范围归还给 HGGC。开发者必须确保在调用 `hgMemAddressFree` 前，整个 VA 范围已被解除映射（unmap）。这些函数在概念上类似于 Linux 的 `mmap` 与 `munmap`，或 Windows 的 `VirtualAlloc` 与 `VirtualFree`。下面代码片段展示了该函数的用法：


    HGdeviceptr ptr;
    // `ptr` holds the returned start of virtual address range reserved.
    HGresult result = hgMemAddressReserve(&ptr, size, 0, 0, 0); // alignment = 0 for default alignment


**映射内存**

已分配的物理内存与上一小节中划出的虚拟地址空间，体现了 VMM API 引入的内存与地址分离。要让已分配内存可用，开发者必须将内存映射到地址空间。`hgMemAddressReserve` 得到的地址范围与 `hgMemCreate` 或 `hgMemImportFromShareableHandle` 得到的物理分配，需要通过 `hgMemMap` 关联起来。

只要预留的地址空间足够，用户就可以将来自多个设备的分配关联到连续的虚拟地址范围中。若要解除物理分配与地址范围的关联，开发者必须使用 `hgMemUnmap` 对映射的地址进行解除映射。只要确保不在已映射的 VA 预留范围上再次创建映射，用户就可以对同一地址范围执行任意次数的映射与解除映射。下面代码片段展示了该函数的用法：


    HGdeviceptr ptr;
    // `ptr`：先前通过 hgMemAddressReserve 预留的地址范围中的地址。
    // `allocHandle`：先前通过 hgMemCreate 获取的 HGmemGenericAllocationHandle。
    HGresult result = hgMemMap(ptr, size, 0, allocHandle, 0);


##### 4.6.3.3.4. 访问权限 {#46334-访问权限}

HGGC 的虚拟内存管理 API 允许应用通过访问控制机制显式保护其 VA 范围。使用 `hgMemMap` 将分配映射到地址范围中的某段区域，并不会使该地址可访问；如果 HGGC kernel 访问该地址，将导致程序崩溃。开发者必须在源设备与访问设备上使用 `hgMemSetAccess` 函数显式设置访问控制。这将允许或限制特定设备对某段已映射地址范围的访问。下面代码片段展示了该函数的用法：


    void setAccessOnDevice(int device, HGdeviceptr ptr, size_t size) {
        HGmemAccessDesc accessDesc = {};
        accessDesc.location.type = HG_MEM_LOCATION_TYPE_DEVICE;
        accessDesc.location.id = device;
        accessDesc.flags = HG_MEM_ACCESS_FLAGS_PROT_READWRITE;

        // 设置地址的访问权限
        hgMemSetAccess(ptr, size, &accessDesc, 1);
    }


VMM 暴露的访问控制机制允许用户明确指定希望与系统中其他对等设备共享哪些分配。如前所述，`hggcEnablePeerAccess` 会强制将过去与未来所有通过 `hggcMalloc` 创建的分配映射到目标对等设备。这在许多情况下很方便，因为用户无需跟踪系统中每个分配到每个设备的映射状态。但这种方法会带来性能影响。通过在分配粒度上进行访问控制，VMM 允许以最小开销实现对等映射。

##### 4.6.3.3.5. 释放内存 {#46335-释放内存}

为释放已分配的内存与地址空间，源进程与目标进程都应依次使用 hgMemUnmap、hgMemRelease 与 hgMemAddressFree。hgMemUnmap 会将先前映射的内存区域从地址范围中解除映射，从而将物理内存与保留的虚拟地址空间分离。接着，hgMemRelease 会释放先前创建的物理内存，将其归还系统。最后，hgMemAddressFree 会释放先前保留的虚拟地址范围，使其可供将来使用。按此特定顺序可确保物理内存与虚拟地址空间被干净且完整地释放。


    hgMemUnmap(ptr, size);
    hgMemRelease(handle);
    hgMemAddressFree(ptr, size);


!!! note

    在 OS 特定场景下，导出的句柄必须使用 fclose 关闭。该步骤不适用于基于 fabric 的场景。


#### 4.6.3.4. 高级配置 {#4634-高级配置}

##### 4.6.3.4.1. 内存类型 {#46341-内存类型}

VMM 还提供一种机制，使应用能够分配某些设备可能支持的特殊类型内存。通过 `hgMemCreate`，应用可以使用 `HGmemAllocationProp::allocFlags` 来指定内存类型需求，以选择性启用特定内存特性。应用必须确保所请求的内存类型被设备支持。

##### 4.6.3.4.2. 虚拟别名支持 {#46342-虚拟别名支持}

虚拟内存管理 API 提供了一种方式，可以通过对同一分配使用多个不同虚拟地址分别调用 `hgMemMap`，从而创建多个虚拟内存映射或"代理（proxy）"。这称为虚拟别名（virtual aliasing）。除非 TIX 另有说明，对某个代理的写入在写入设备操作（网格启动、memcpy、memset 等）完成之前，与该内存的其他代理被认为是不一致且不相干（incoherent）的。在写入设备操作之前已在 PPU 上存在、但在写入设备操作完成之后才读取的网格，也被认为具有不一致且不相干的代理。

例如，下列代码片段在假设设备指针 A 与 B 是同一内存分配的虚拟别名时，其行为未定义：


    __global__ void write_and_read(char *A, char *B) {
      *A = 0x1;
      printf("%d\n", *B);    // 未定义行为！*B 可能取到先前的值，
    // 也可能取到某个中间状态的值。
    }


下面的行为是已定义的，假设这两个 kernel 以单调顺序（通过 streams 或 events）被排序：


    __global__ void write_kernel(char *A) {
      *A = 0x1;
    }

    __global__ void read_kernel(char *B) {
      printf("%d\n", *B);    // 假设 read_kernel 在 write_kernel 完成后才启动，
    // 则 *B == *A == 0x1
    }

    hggcMemcpyAsync(B, input, size, stream1);    // 别名访问允许出现在
    // 操作边界处
    write_kernel<<<1,1,0,stream1>>>(A);          // 允许 write_kernel 访问 A
    hggcEventRecord(event, stream1);
    hggcStreamWaitEvent(stream2, event);
    read_kernel<<<1,1,0,stream2>>>(B);
    hggcStreamWaitEvent(stream3, event);
    hggcMemcpyAsync(output, B, size, stream3);  // read_kernel 和 hggcMemcpy
                                                //（均为读操作）都会等待
                                                // write_kernel（写操作）
                                                // 完成后再执行



## 5. 技术参考 {#5-技术参考}

### 5.1. C++ 语言支持 {#51-c-语言支持}


`hggc` 根据以下规范处理 HGGC host 和 device 代码：

* **C++03** (ISO/IEC 14882:2003), `--std=c++03` 标志。

* **C++11** (ISO/IEC 14882:2011), `--std=c++11` 标志。

* **C++14** (ISO/IEC 14882:2014), `--std=c++14` 标志。

* **C++17** (ISO/IEC 14882:2017), `--std=c++17` 标志。

* **C++20** (ISO/IEC 14882:2020), `--std=c++20` 标志。

传递 `hggc` `-std=c++<version>` 标志会启用与指定版本相关的所有 C++ 特性，并在调用预处理器、编译器和链接器时使用相应的 C++ 选项。

编译器支持所有 C++ 标准支持的语言特性，但需遵守以下章节中所述的限制。

#### 5.1.1. C++11 语言支持 {#511-c11-语言支持}

**表 1：HGGC 设备代码支持的 C++11 核心语言特性**

| 功能域            | 语言特性                                                                        | 提案                                                                                                         | 支持情况 |
| ----------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------- |
| **类型系统**      | auto                                                                            | [N1984](https://wg21.link/N1984)                                                                             | ✅        |
|                   | decltype  (FTM)*                                                                | [N2343](https://wg21.link/N2343)<br>[N3276](https://wg21.link/N3276)                                         | ✅        |
|                   | char16_t and char32_t  (FTM)*                                                   | [N2249](https://wg21.link/N2249)                                                                             | ✅        |
|                   | long long                                                                       | [N1811](https://wg21.link/N1811)                                                                             | ✅        |
|                   | nullptr                                                                         | [N2431](https://wg21.link/N2431)                                                                             | ✅        |
|                   | Strongly-typed enum                                                             | [N2347](https://wg21.link/N2347)                                                                             | ✅        |
|                   | Forward (opaque) enum declarations                                              | [N2764](https://wg21.link/N2764)                                                                             | ✅        |
|                   | Compiler support for type traits. N2255*                                        | [N2255](https://wg21.link/N2255)<br>[N2984](https://wg21.link/N2984)<br>[N3142](https://wg21.link/N3142)     | ✅        |
| **初始化与构造**  | Initializer lists  (FTM)*                                                       | [N2672](https://wg21.link/N2672)                                                                             | ✅        |
|                   | Non-static data member initializers  (FTM)*                                     | [N2756](https://wg21.link/N2756)                                                                             | ✅        |
|                   | Delegating constructors  (FTM)*                                                 | [N1986](https://wg21.link/N1986)                                                                             | ✅        |
|                   | Inheriting constructors  (FTM)*                                                 | [N2540](https://wg21.link/N2540)                                                                             | ✅        |
| **模板与泛型**    | Variadic templates  (FTM)*                                                      | [N2242](https://wg21.link/N2242)<br>[N2555](https://wg21.link/N2555)                                         | ✅        |
|                   | Template aliases  (FTM)*                                                        | [N2258](https://wg21.link/N2258)                                                                             | ✅        |
|                   | Right angle brackets                                                            | [N1757](https://wg21.link/N1757)                                                                             | ✅        |
|                   | extern template                                                                 | [N1987](https://wg21.link/N1987)                                                                             | ✅        |
|                   | Expression SFINAE                                                               | [N2634](https://wg21.link/N2634)                                                                             | ✅        |
|                   | Local and unnamed types as template parameters                                  | [N2657](https://wg21.link/N2657)                                                                             | ✅        |
| **函数与 Lambda** | Lambda expressions  (FTM)*                                                      | [N2550](https://wg21.link/N2550)<br>[N2658](https://wg21.link/N2658)<br>[N2927](https://wg21.link/N2927)     | ✅        |
|                   | Trailing function return types                                                  | [N2541](https://wg21.link/N2541)                                                                             | ✅        |
|                   | Explicit conversion operators                                                   | [N2437](https://wg21.link/N2437)                                                                             | ✅        |
|                   | Rvalue references  (FTM)*                                                       | [N2118](https://wg21.link/N2118)<br>[N2844](https://wg21.link/N2844)<br>[CWG1138](https://wg21.link/CWG1138) | ✅        |
|                   | ref-qualifiers  (FTM)*                                                          | [N2439](https://wg21.link/N2439)                                                                             | ✅        |
| **控制流与异常**  | Range-for loop  (FTM)*                                                          | [N2930](https://wg21.link/N2930)<br>[N3271](https://wg21.link/N3271)                                         | ✅        |
|                   | noexcept                                                                        | [N3050](https://wg21.link/N3050)                                                                             | ✅        |
|                   | static_assert  (FTM)*                                                           | [N1720](https://wg21.link/N1720)                                                                             | ✅        |
|                   | constexpr  (FTM)*                                                               | [N2235](https://wg21.link/N2235)                                                                             | ✅        |
| **类与对象**      | Defaulted and deleted functions                                                 | [N2346](https://wg21.link/N2346)                                                                             | ✅        |
|                   | Defaulted move special member functions                                         | [N3053](https://wg21.link/N3053)                                                                             | ✅        |
|                   | Extended friend declarations                                                    | [N1791](https://wg21.link/N1791)                                                                             | ✅        |
|                   | Unrestricted unions                                                             | [N2544](https://wg21.link/N2544)                                                                             | ✅        |
|                   | override and final                                                              | [N2928](https://wg21.link/N2928)<br>[N3206](https://wg21.link/N3206)<br>[N3272](https://wg21.link/N3272)     | ✅        |
| **字面量与属性**  | Unicode string literals  (FTM)*                                                 | [N2442](https://wg21.link/N2442)                                                                             | ✅        |
|                   | Raw string literals  (FTM)*                                                     | [N2442](https://wg21.link/N2442)                                                                             | ✅        |
|                   | User-defined literals  (FTM)*                                                   | [N2765](https://wg21.link/N2765)                                                                             | ✅        |
|                   | Attributes, [[noreturn]]  (FTM)*                                                | [N2761](https://wg21.link/N2761)                                                                             | ✅        |
|                   | [[carries_dependency]]                                                          | [N2556](https://wg21.link/N2556)<br>[N2643](https://wg21.link/N2643)                                         | ✅        |
| **对齐与预处理**  | alignas                                                                         | [N2341](https://wg21.link/N2341)                                                                             | ✅        |
|                   | alignof                                                                         | [N2341](https://wg21.link/N2341)                                                                             | ✅        |
|                   | C99 preprocessor                                                                | [N1653](https://wg21.link/N1653)                                                                             | ✅        |
|                   | Inline namespaces                                                               | [N2535](https://wg21.link/N2535)                                                                             | ✅        |
| **并发（受限）**  | Atomic operations                                                               | [N2427](https://wg21.link/N2427)                                                                             | ❌        |
|                   | Thread-local storage                                                            | [N2659](https://wg21.link/N2659)                                                                             | ❌        |
|                   | Dynamic initialization and destruction with concurrency (magic statics)  (FTM)* | [N2660](https://wg21.link/N2660)                                                                             | ❌        |
|                   | Garbage Collection and Reachability-Based Leak Detection                        | [N2670](https://wg21.link/N2670)                                                                             | ❌        |

#### 5.1.2. C++14 语言支持 {#512-c14-语言支持}

**表 2：HGGC 设备代码支持的 C++14 核心语言特性**

| 功能域           | 语言特性                                                                   | 提案                             | 支持情况 |
| ---------------- | -------------------------------------------------------------------------- | -------------------------------- | -------- |
| **类型推导**     | decltype(auto), Return type deduction for normal functions  (FTM)*  (FTM)* | [N3638](https://wg21.link/N3638) | ✅        |
| **Lambda 增强**  | Initialized/Generalized lambda captures (init-capture)  (FTM)*             | [N3648](https://wg21.link/N3648) | ✅        |
|                  | Generic lambda expressions  (FTM)*                                         | [N3649](https://wg21.link/N3649) | ✅        |
| **模板与常量**   | Variable templates  (FTM)*                                                 | [N3651](https://wg21.link/N3651) | ✅        |
|                  | Extended constexpr  (FTM)*                                                 | [N3652](https://wg21.link/N3652) | ✅        |
| **初始化**       | Aggregates with default member initializers  (FTM)*                        | [N3653](https://wg21.link/N3653) | ✅        |
| **转换与字面量** | Tweaked wording for contextual conversions                                 | [N3323](https://wg21.link/N3323) | ✅        |
|                  | Binary literals  (FTM)*                                                    | [N3472](https://wg21.link/N3472) | ✅        |
|                  | Single quote as digit separator                                            | [N3781](https://wg21.link/N3781) | ✅        |
| **内存与属性**   | Omitting/extending memory allocations                                      | [N3664](https://wg21.link/N3664) | ✅        |
|                  | [[deprecated]] attribute                                                   | [N3760](https://wg21.link/N3760) | ✅        |
|                  | Sized deallocation  (FTM)*                                                 | [N3778](https://wg21.link/N3778) | ✅        |

#### 5.1.3. C++17 语言支持 {#513-c17-语言支持}

**表 3：HGGC 设备代码支持的 C++17 核心语言特性**

| 功能域             | 语言特性                                                                            | 提案                                 | 支持情况 |
| ------------------ | ----------------------------------------------------------------------------------- | ------------------------------------ | -------- |
| **类型与初始化**   | DR11: New auto rules for direct-list-initialization                                 | [N3922](https://wg21.link/N3922)     | ✅        |
|                    | Direct-list-initialization of enumerations                                          | [P0138R2](https://wg21.link/P0138R2) | ✅        |
|                    | Structured Bindings  (FTM)*                                                         | [P0217R3](https://wg21.link/P0217R3) | ✅        |
|                    | Aggregate classes with base classes  (FTM)*                                         | [P0017R1](https://wg21.link/P0017R1) | ✅        |
|                    | Guaranteed copy elision  (FTM)*                                                     | [P0135R1](https://wg21.link/P0135R1) | ✅        |
|                    | Replacement of class objects containing reference members                           | [P0137R1](https://wg21.link/P0137R1) | ✅        |
|                    | DR11: New specification for inheriting constructors (DR1941 et al)  (FTM)*          | [P0136R1](https://wg21.link/P0136R1) | ✅        |
| **模板与泛型**     | typename in a template template parameter                                           | [N4051](https://wg21.link/N4051)     | ✅        |
|                    | Allow constant evaluation for all constant template arguments  (FTM)*               | [N4268](https://wg21.link/N4268)     | ✅        |
|                    | Constant template parameters with auto type  (FTM)*                                 | [P0127R2](https://wg21.link/P0127R2) | ✅        |
|                    | Fold Expressions  (FTM)*                                                            | [N4295](https://wg21.link/N4295)     | ✅        |
|                    | Unary fold expressions and empty parameter packs                                    | [P0036R0](https://wg21.link/P0036R0) | ✅        |
|                    | Class template argument deduction  (FTM)*                                           | [P0091R3](https://wg21.link/P0091R3) | ✅        |
|                    | DR98: Matching of template template-arguments excludes compatible templates  (FTM)* | [P0522R0](https://wg21.link/P0522R0) | ✅        |
|                    | Pack expansions in using-declarations  (FTM)*                                       | [P0195R2](https://wg21.link/P0195R2) | ✅        |
| **Lambda**         | Lambda capture of `*this` (FTM)*                                                    | [P0018R3](https://wg21.link/P0018R3) | ✅        |
|                    | constexpr lambda expressions  (FTM)*                                                | [P0170R1](https://wg21.link/P0170R1) | ✅        |
| **控制流**         | constexpr if statements  (FTM)*                                                     | [P0292R2](https://wg21.link/P0292R2) | ✅        |
|                    | Init-statements for if and switch                                                   | [P0305R1](https://wg21.link/P0305R1) | ✅        |
|                    | Differing begin and end types in range-based for  (FTM)*                            | [P0184R0](https://wg21.link/P0184R0) | ✅        |
|                    | Stricter expression evaluation order                                                | [P0145R3](https://wg21.link/P0145R3) | ✅        |
| **属性与诊断**     | [[fallthrough]] attribute                                                           | [P0188R1](https://wg21.link/P0188R1) | ✅        |
|                    | [[nodiscard]] attribute                                                             | [P0189R1](https://wg21.link/P0189R1) | ✅        |
|                    | [[maybe_unused]] attribute                                                          | [P0212R1](https://wg21.link/P0212R1) | ✅        |
|                    | Attributes for namespaces and enumerators  (FTM)*  (FTM)*                           | [N4266](https://wg21.link/N4266)     | ✅        |
|                    | Using attribute namespaces without repetition                                       | [P0028R4](https://wg21.link/P0028R4) | ✅        |
|                    | Ignore unknown attributes                                                           | [P0283R2](https://wg21.link/P0283R2) | ✅        |
|                    | static_assert with no message  (FTM)*                                               | [N3928](https://wg21.link/N3928)     | ✅        |
| **声明与变量**     | Inline variables  (FTM)*                                                            | [P0386R2](https://wg21.link/P0386R2) | ✅        |
|                    | Nested namespace definition                                                         | [N4230](https://wg21.link/N4230)     | ✅        |
| **异常与内存**     | Make exception specifications part of the type system  (FTM)*                       | [P0012R1](https://wg21.link/P0012R1) | ✅        |
|                    | Removing dynamic exception specifications                                           | [P0003R5](https://wg21.link/P0003R5) | ✅        |
|                    | Dynamic memory allocation for over-aligned data  (FTM)*                             | [P0035R4](https://wg21.link/P0035R4) | ✅        |
| **字面量与预处理** | u8 character literals                                                               | [N4267](https://wg21.link/N4267)     | ✅        |
|                    | Hexadecimal floating-point literals  (FTM)*                                         | [P0245R1](https://wg21.link/P0245R1) | ✅        |
|                    | __has_include in preprocessor conditionals                                          | [P0061R1](https://wg21.link/P0061R1) | ✅        |
| **废弃与清理**     | Remove deprecated use of the register keyword                                       | [P0001R1](https://wg21.link/P0001R1) | ✅        |
|                    | Remove deprecated operator++(bool)                                                  | [P0002R1](https://wg21.link/P0002R1) | ✅        |
|                    | Removing trigraphs                                                                  | [N4086](https://wg21.link/N4086)     | ✅        |

#### 5.1.4. C++20 语言支持 {#514-c20-语言支持}

依赖：GCC 版本 ≥ 10.0，Clang 版本 ≥ 10.0

**表 4：HGGC 设备代码支持的 C++20 核心语言特性**

| 功能域                 | 语言特性                                                                                     | 提案                                                                         | 支持情况 |
| ---------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | -------- |
| **概念与约束**         | Concepts  (FTM)*                                                                             | [P0734R0](https://wg21.link/P0734R0)                                         | ✅        |
|                        | Yet another approach for constrained declarations                                            | [P1141R2](https://wg21.link/P1141R2)                                         | ✅        |
| **常量求值**           | constexpr virtual function  (FTM)*                                                           | [P1064R0](https://wg21.link/P1064R0)                                         | ✅        |
|                        | constexpr try-catch blocks                                                                   | [P1002R1](https://wg21.link/P1002R1)                                         | ✅        |
|                        | Immediate functions (consteval)  (FTM)*                                                      | [P1073R3](https://wg21.link/P1073R3)                                         | ✅        |
|                        | constinit  (FTM)*                                                                            | [P1143R2](https://wg21.link/P1143R2)                                         | ✅        |
|                        | std::is_constant_evaluated()  (FTM)*                                                         | [P0595R2](https://wg21.link/P0595R2)                                         | ✅        |
|                        | constexpr container operations  (FTM)*                                                       | [P0784R7](https://wg21.link/P0784R7)                                         | ✅        |
|                        | Trivial default initialization in constexpr functions                                        | [P1331R2](https://wg21.link/P1331R2)                                         | ✅        |
|                        | Unevaluated asm-declaration in constexpr functions                                           | [P1668R1](https://wg21.link/P1668R1)                                         | ✅        |
|                        | Changing the active member of a union inside constexpr  (FTM)*                               | [P1330R0](https://wg21.link/P1330R0)                                         | ✅        |
|                        | DR11: Specify when constexpr function definitions are needed for constant evaluation  (FTM)* | [P0859R0](https://wg21.link/P0859R0)                                         | ✅        |
|                        | dynamic_cast and polymorphic typeid in constant expressions                                  | [P1327R1](https://wg21.link/P1327R1)                                         | ✅        |
| **比较运算**           | Three-way comparison operator  (FTM)*                                                        | [P0515R3](https://wg21.link/P0515R3)                                         | ✅        |
|                        | Consistency improvements for comparisons                                                     | [P1120R0](https://wg21.link/P1120R0)                                         | ✅        |
|                        | <=> != ==                                                                                    | [P1185R2](https://wg21.link/P1185R2)                                         | ✅        |
|                        | Synthesizing Three-way comparison for specified comparison category                          | [P1186R3](https://wg21.link/P1186R3)                                         | ✅        |
|                        | Allow defaulting comparisons by value                                                        | [P1946R0](https://wg21.link/P1946R0)                                         | ✅        |
|                        | Remove std::weak_equality and std::strong_equality                                           | [P1959R0](https://wg21.link/P1959R0)                                         | ✅        |
| **Lambda**             | Allow Lambda capture [=, this]                                                               | [P0409R2](https://wg21.link/P0409R2)                                         | ✅        |
|                        | template-parameter-list for generic lambdas  (FTM)*                                          | [P0428R2](https://wg21.link/P0428R2)                                         | ✅        |
|                        | Lambdas in unevaluated contexts                                                              | [P0315R4](https://wg21.link/P0315R4)                                         | ✅        |
|                        | Default constructible and assignable stateless lambdas                                       | [P0624R2](https://wg21.link/P0624R2)                                         | ✅        |
|                        | Pack-expansions in lambda init-captures  (FTM)*                                              | [P0780R2](https://wg21.link/P0780R2)                                         | ✅        |
|                        | DR11: Simplifying implicit lambda capture                                                    | [P0588R1](https://wg21.link/P0588R1)                                         | ✅        |
|                        | Deprecate implicit capture of this via [=]                                                   | [P0806R2](https://wg21.link/P0806R2)                                         | ✅        |
|                        | Lambda capture and storage class specifiers of structured bindings                           | [P1091R3](https://wg21.link/P1091R3)<br>[P1381R1](https://wg21.link/P1381R1) |          |
| **类与初始化**         | Designated initializers  (FTM)*                                                              | [P0329R4](https://wg21.link/P0329R4)                                         | ✅        |
|                        | Default member initializers for bit-fields                                                   | [P0683R1](https://wg21.link/P0683R1)                                         | ✅        |
|                        | Initializer list constructors in class template argument deduction                           | [P0702R1](https://wg21.link/P0702R1)                                         | ✅        |
|                        | Prohibit aggregates with user-declared constructors                                          | [P1008R1](https://wg21.link/P1008R1)                                         | ✅        |
|                        | Parenthesized initialization of aggregates  (FTM)*                                           | [P0960R3](https://wg21.link/P0960R3)                                         | ✅        |
|                        | Type mismatch of defaulted special member functions                                          | [P0641R2](https://wg21.link/P0641R2)                                         | ✅        |
|                        | Conditionally trivial special member functions  (FTM)*                                       | [P0848R3](https://wg21.link/P0848R3)                                         | ✅        |
|                        | DR11: Implicit move for more local objects and rvalue references                             | [P1825R0](https://wg21.link/P1825R0)                                         | ✅        |
| **模板**               | class template argument deduction for alias templates  (FTM)*                                | [P1814R0](https://wg21.link/P1814R0)                                         | ✅        |
|                        | class template argument deduction for aggregates  (FTM)*                                     | [P1816R0](https://wg21.link/P1816R0)<br>[P2082R1](https://wg21.link/P2082R1) | ✅        |
|                        | Class types in Constant template parameters                                                  | [P0732R2](https://wg21.link/P0732R2)                                         | ✅        |
|                        | Inconsistencies with constant template parameters  (FTM)*                                    | [P1907R1](https://wg21.link/P1907R1)                                         | ✅        |
|                        | Access checking on specializations                                                           | [P0692R1](https://wg21.link/P0692R1)                                         | ✅        |
| **结构化绑定**         | DR17: Relaxing the structured bindings customization point finding rules                     | [P0961R1](https://wg21.link/P0961R1)                                         | ✅        |
|                        | DR17: Allow structured bindings to accessible members                                        | [P0969R0](https://wg21.link/P0969R0)                                         | ✅        |
| **控制流**             | init-statements for range-based for                                                          | [P0614R1](https://wg21.link/P0614R1)                                         | ✅        |
|                        | DR11: Relaxing the range-for loop customization point finding rules                          | [P0962R1](https://wg21.link/P0962R1)                                         | ✅        |
| **属性**               | Attributes [[likely]] and [[unlikely]]                                                       | [P0479R5](https://wg21.link/P0479R5)                                         | ✅        |
|                        | Attribute [[no_unique_address]]                                                              | [P0840R2](https://wg21.link/P0840R2)                                         | ✅        |
|                        | [[nodiscard]] with message                                                                   | [P1301R4](https://wg21.link/P1301R4)                                         | ✅        |
|                        | DR17: [[nodiscard]] for constructors                                                         | [P1771R1](https://wg21.link/P1771R1)                                         | ✅        |
| **类型与转换**         | char8_t  (FTM)*                                                                              | [P0482R6](https://wg21.link/P0482R6)                                         | ✅        |
|                        | explicit(bool)  (FTM)*                                                                       | [P0892R2](https://wg21.link/P0892R2)                                         | ✅        |
|                        | Signed integers are two's complement                                                         | [P1236R1](https://wg21.link/P1236R1)                                         | ✅        |
|                        | const&-qualified pointers to members                                                         | [P0704R1](https://wg21.link/P0704R1)                                         | ✅        |
|                        | Make typename more optional                                                                  | [P0634R3](https://wg21.link/P0634R3)                                         | ✅        |
|                        | Permit conversions to arrays of unknown bound                                                | [P0388R4](https://wg21.link/P0388R4)                                         | ✅        |
|                        | DR11: Converting from T* to bool should be considered narrowing                              | [P1957R2](https://wg21.link/P1957R2)                                         | ✅        |
| **预处理与字面量**     | __VA_OPT__                                                                                   | [P0306R4](https://wg21.link/P0306R4)<br>[P1042R1](https://wg21.link/P1042R1) | ✅        |
|                        | Integrating feature-test macros                                                              | [P0941R2](https://wg21.link/P0941R2)                                         | ✅        |
|                        | Stronger Unicode requirements                                                                | [P1041R4](https://wg21.link/P1041R4)<br>[P1139R2](https://wg21.link/P1139R2) | ✅        |
| **声明与命名空间**     | Nested inline namespaces                                                                     | [P1094R2](https://wg21.link/P1094R2)                                         | ✅        |
|                        | using enum  (FTM)*                                                                           | [P1099R5](https://wg21.link/P1099R5)                                         | ✅        |
|                        | ADL and function templates that are not visible                                              | [P0846R0](https://wg21.link/P0846R0)                                         | ✅        |
| **内存与对象**         | Destroying operator delete  (FTM)*                                                           | [P0722R3](https://wg21.link/P0722R3)                                         | ✅        |
|                        | Deprecating some uses of volatile                                                            | [P1152R4](https://wg21.link/P1152R4)                                         | ✅        |
|                        | DR98: Pseudo-destructors end object lifetimes                                                | [P0593R6](https://wg21.link/P0593R6)                                         | ✅        |
|                        | DR11: Array size deduction in new-expressions                                                | [P1009R2](https://wg21.link/P1009R2)                                         | ✅        |
|                        | Deprecate comma operator in subscripts                                                       | [P1161R3](https://wg21.link/P1161R3)                                         | ✅        |
| **并发与模块（受限）** | Coroutines  (FTM)*  (FTM)*                                                                   | [P0912R5](https://wg21.link/P0912R5)<br>[LWG3393](https://wg21.link/LWG3393) | ❌        |
|                        | Modules  (FTM)*                                                                              | [P1103R3](https://wg21.link/P1103R3)                                         | ❌        |
|                        | DR11: Explicitly defaulted functions with different exception specifications                 | [P1286R2](https://wg21.link/P1286R2)                                         | ❌        |

#### 5.1.5. HGGC C++ 标准库 {#515-hggc-c-标准库}

HGGC 提供了一个 C++ 标准库（STL）实现。该库具有以下优点：

* 在主机和设备端均可用。

* 与 SAIL 工具包支持的所有 Linux 平台兼容。

* 与 SAIL 工具包支持的所有 PPU 架构兼容。

* 与前面主版本的所有 SAIL 工具包兼容。

* 提供了 C++17 的反向移植特性，并包括来自 C++20、C++23 和 C++26 的标准库特性。

* 支持扩展数据类型，例如 128 位整数（__int128）、半精度浮点（__half）和 Bfloat16（__ppu_bfloat16）。

* 针对设备代码高度优化。

此外，还提供了标准库中没有的扩展功能，以提高生产力和应用性能。这些扩展包括：数学函数、内存操作、同步原语、容器扩展、HGGC 内置的高级抽象、C++ TIX 封装接口等。

该标准库随 HGGC Toolkit 一并提供。

#### 5.1.6. C 标准库函数 {#516-c-标准库函数}

##### 5.1.6.1. `clock()` and `clock64()` {#5161-clock-and-clock64}

```cpp
__host__ __device__ clock_t clock();
__device__ long long clock64();
```

当在设备代码中执行时，该函数会返回每个计算单元（CU）计数器的当前值，该计数器以每时钟周期加一的速率持续递增。通过在核函数起始和终止位置分别记录计数器数值，计算两者差值并将结果对应存储到每个线程，可以估算出设备调度线程期间消耗的时钟周期总数。但需注意，这个数值并不能准确反映设备实际执行线程指令所需的时钟周期量，主要原因包含三个层面：
 * 线程调度机制
 该值显著高于实际执行周期，关键原因在于线程采用时间片轮转调度策略。
 * 系统空闲计数
 计数器在系统空闲状态下仍保持持续递增，导致统计值包含无效时段。
 * 资源等待延迟
 线程因等待内存访问及其他资源而被挂起期间，计数器仍在持续累积计数值。

这个特性使得该测量值更适合用于分析系统级调度行为，而非精确的指令执行周期统计。

!!! note

    对应的 HGGC C++ 函数 `hggc::std::clock()` 提供在 `<hggc/std/ctime>` 头文件中。

    可移植的 [C++](https://en.cppreference.com/w/cpp/header/chrono) `<chrono>` 实现也提供在 `<hggc/std/chrono>` 头文件中用于类似目的。

##### 5.1.6.2. `printf()` {#5162-printf}

```cpp
int printf(const char *format[, arg, ...]);
```

HGGC 在设备端提供了 `printf()` 函数，允许核函数将格式化文本发送到主机端的标准输出流。其用法与 C 标准库中的 `printf()` 基本一致，具体行为细节可参考主机系统的 C 库手册。

由于设备端的执行模型是多线程并行的，每个线程都会独立调用 `printf()` 并使用自身的局部数据进行格式化。这意味着当核函数中所有线程都执行到 `printf()` 时，主机端输出流中会出现与活跃线程数量相同的输出行。

!!! note "返回值语义"

    HGGC 版 `printf()` 的返回值与 C 标准略有不同：成功→已解析参数个数（无参数时为 0）；`NULL` 格式字符串→ -1；内部错误→ -2。

需注意的一个实现细节是，`printf()` 内部依赖共享的数据结构来管理输出缓冲区，这可能导致调用了 `printf()` 的线程比未调用的线程花费更多时钟周期，且额外开销的大小取决于 `printf()` 的参数（如格式字符串复杂度和参数数量）。不过，HGGC 本身就不保证线程间的执行顺序（除非使用 `__syncthreads()` 等显式同步屏障），因此这种行为差异不应被视为可依赖的排序保证。

* * *

**格式说明符**

对于标准 `printf()`，格式说明符的形式为: `%[flags][width][.precision][size]type`

支持以下字段。

* Flags: `#`, `' '`, `0`, `+`, `-`

* Width: `*`, `0-9`

* Precision: `0-9`

* Size: `h`, `l`, `ll`

* Type: `%cdiouxXpeEfgGaAs`

* * *

**限制**

由于格式字符串最终在主机侧被解析和渲染，其行为受主机 C 库实现的影响。HGGC 支持的格式说明符覆盖了主流编译器的共同子集，但特定平台上的边界行为可能存在差异。

设备端的 `printf()` 不会对标志与类型的组合做合法性校验——不合法的组合将原样传递给主机 C 库，输出结果为未定义的。

单次 `printf()` 调用最多支持 32 个格式化参数（不含格式字符串本身），超出部分会被静默忽略，对应的格式占位符将以字面文本形式出现在输出中。

!!! warning

    `long` 类型在 32 位和 64 位平台上的宽度不同，因此使用 `%ld` 格式说明符时，必须确保编译和运行平台的字长一致，否则输出结果可能不正确。

* * *

**主机端缓冲区**

HGGC 为 `printf()` 预分配了一块固定大小的环形缓冲区。当核函数产生的输出超过缓冲区容量时，新内容将覆盖最早的数据。该缓冲区不会实时传输到主机，而是在特定事件发生时被刷新。按场景分类如下：

| 场景分类     | 触发时机                                                      | 相关 API                                                                                                                                              |
| ------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 核函数执行   | 核函数启动前刷新；若设置了 `LAUNCH_BLOCKING=1` 则启动后亦刷新 | `<<< >>>`、`hgLaunchKernel()`                                                                                                                         |
| 数据传输     | 阻塞式主机-设备内存拷贝完成时                                 | `hggcMemcpy*()`/`hgMemcpy*()` 的阻塞变体                                                                                                              |
| 显式同步     | 调用同步 API 使主机等待设备完成时                             | `hggcDeviceSynchronize()`、`hgCtxSynchronize()`、`hggcStreamSynchronize()`、`hgStreamSynchronize()`、`hggcEventSynchronize()`、`hgEventSynchronize()` |
| 资源生命周期 | 模块加载/卸载或上下文销毁时                                   | `hgModuleLoad()`/`hgModuleUnload()`、`hggcDeviceReset()`/`hgCtxDestroy()`                                                                             |
| 主机回调     | 执行注册的主机端回调之前                                      | `hggcLaunchHostFunc()`/`hgLaunchHostFunc()`                                                                                                           |

!!! warning

    进程正常退出时缓冲区**不会**被自动刷新。如需确保所有输出可见，应在退出前显式调用同步 API。

以下 API 函数设置和检索用于将 `printf()` 参数和内部元数据传输到主机的缓冲区大小。默认大小为 18M 字节。

* `hggcDeviceGetLimit(size_t* size, hggcLimitPrintfFifoSize)`

* `hggcDeviceSetLimit(hggcLimitPrintfFifoSize, size_t size)`

* * *

**示例**

下面的示例展示了二维线程索引与多种数据类型的格式化输出：

```cpp
#include <stdio.h>
__global__ void reportPosition(int width) {
  int col = threadIdx.x;
  int row = threadIdx.y;
  int gid = row * width + col;
  printf("[%d,%d] gid=%d area=%d\n", row, col, gid, width * width);
}
int main() {
  dim3 block(3, 2);
  reportPosition<<<1, block>>>(3);
  hggcDeviceSynchronize();
  return 0;
}
```

运行后将产生 6 行输出（3×2 个线程各输出一行），顺序取决于硬件调度，例如：

```text
[1,2] gid=5 area=9
[0,0] gid=0 area=9
[0,1] gid=1 area=9
[1,0] gid=3 area=9
[0,2] gid=2 area=9
[1,1] gid=4 area=9
```

* * *

通过条件判断可以选择性地控制哪些线程执行输出。例如，仅让每行的首线程打印摘要信息：

```cpp
#include <stdio.h>
__global__ void rowSummary(int width) {
  if (threadIdx.x == 0) {
    printf("Row %d: threads [0..%d]\n", threadIdx.y, width - 1);
  }
}
int main() {
  dim3 block(4, 3);
  rowSummary<<<1, block>>>(4);
  hggcDeviceSynchronize();
  return 0;
}
```

输出仅包含 3 行（每行对应一个 `threadIdx.y` 值），因为每行中只有 `threadIdx.x == 0` 的线程满足条件。

##### 5.1.6.3. `memcpy()` 和 `memset()` {#5163-memcpy-和-memset}

```cpp
__host__ __device__ void *memcpy(void *dest, const void *src, size_t size);
```

该函数将 `size` 字节从 `src` 指向的内存位置复制到 `dest` 指向的内存位置。

```cpp
__host__ __device__ void *memset(void *ptr, int value, size_t size);
```

该函数将 `ptr` 指向的内存块的 `size` 字节设置为 `value`，将其当做 `unsigned char`。

!!! note

    建议使用 `<hggc/std/cstring>` 头文件中提供的 `hggc::std::memcpy()` 和 `hggc::std::memset()` 函数作为 `memcpy` 和 `memset` 的更安全版本。

##### 5.1.6.4. `malloc()` 和 `free()` {#5164-malloc-和-free}

HGGC 在设备端提供了一组动态内存管理接口，统一通过设备堆（device heap）进行分配和释放。下表汇总了所有可用的分配与释放函数：

| 功能             | C 风格 API                                | HGGC C++ API（`<hggc/std/cstdlib>`） | 执行空间              | 说明                                     |
| ---------------- | ----------------------------------------- | ------------------------------------ | --------------------- | ---------------------------------------- |
| 分配（默认对齐） | `malloc(size)`                            | `hggc::std::malloc(size)`            | `__host__ __device__` | 返回至少 `size` 字节的 16 字节对齐指针   |
| 零初始化分配     | —                                         | `hggc::std::calloc(count, size)`     | `__host__ __device__` | 分配并零初始化                           |
| 指定对齐分配     | `__hg_aligned_device_malloc(size, align)` | `hggc::std::aligned_alloc()`         | `__device__`          | 对齐值须为非零的二的次幂                 |
| 释放             | `free(ptr)`                               | `hggc::std::free(ptr)`               | `__host__ __device__` | 传 `NULL` 安全忽略；重复释放为未定义行为 |

所有分配函数在内存不足时返回 `NULL`。

**生命周期与跨线程共享**

通过上述任一接口分配的内存，在整个 HGGC 上下文的生命周期内有效——不仅限于分配时所在的核函数调用，后续核函数启动的线程同样可以访问和释放这些内存。任何线程都可以释放另一个线程分配的内存，但调用方须确保同一指针不被释放两次。

* * *

**堆内存配置**

设备堆的容量必须在首次调用任何设备端分配函数（包括 `malloc()`、`hggc::std::malloc()`、`new` 等）之前完成配置。未显式设置时，运行时将使用 8 MB 作为默认堆大小。相关的查询和配置接口如下：

| 操作           | API 层级    | API                                                         |
| -------------- | ----------- | ----------------------------------------------------------- |
| 查询当前堆大小 | Runtime API | `hggcDeviceGetLimit(size_t* size, hggcLimitMallocHeapSize)` |
| 查询当前堆大小 | Driver API  | `hgCtxGetLimit()`                                           |
| 设置堆大小     | Runtime API | `hggcDeviceSetLimit(hggcLimitMallocHeapSize, size_t size)`  |

`hggcDeviceSetLimit` 设置的堆大小为下限——实际分配的堆空间可能大于请求的 `size` 字节。

堆的物理内存在模块加载阶段一次性分配（无论是驱动 API 的显式加载还是运行时 API 的隐式加载）。若分配失败，模块加载将返回 `HGGC_ERROR_SHARED_OBJECT_INIT_FAILED`。需注意以下两点约束：

1. 模块加载完成后堆大小不可更改，运行时也不会自动扩容。
2. 设备堆占用的显存独立于主机端 `hggcMalloc` 等 API 的分配池，两者不共享配额。

* * *

**设备端与主机端内存的隔离性**

设备端分配的内存（通过 `malloc()`、`hggc::std::malloc()`、`hggc::std::calloc()`、`__hg_aligned_device_malloc()`、`hggc::std::aligned_alloc()` 或 `new`）与主机端分配的内存（通过 `hggcMalloc` 等运行时/驱动 API）属于不同的内存管理域。两者之间**不能交叉操作**——即设备端分配的内存不能用 `hggcMemcpy` 等主机 API 访问或释放，反之主机端分配的内存也不能用设备端 `free()`、`hggc::std::free()` 或 `delete` 释放。

* * *

下面的示例展示了每个线程独立分配内存、写入线程编号、读回并验证后释放的完整流程：

```cpp
#include <stdio.h>
#include <stdlib.h>
__global__ void perThreadAlloc() {
  int tid = threadIdx.x;
  int *buf = (int *)malloc(sizeof(int));
  if (buf == NULL) {
    printf("Thread %d: allocation failed\n", tid);
    return;
  }
  *buf = tid * tid;
  printf("Thread %d: wrote %d at %p\n", tid, *buf, buf);
  free(buf);
}
int main() {
  hggcDeviceSetLimit(hggcLimitMallocHeapSize, 64 * 1024 * 1024);
  perThreadAlloc<<<1, 4>>>();
  hggcDeviceSynchronize();
  return 0;
}
```

每个线程独立完成"分配→写入→输出→释放"的生命周期，输出顺序取决于硬件调度：

```text
Thread 2: wrote 4 at 0xd1c31830
Thread 0: wrote 0 at 0xd1c318b0
Thread 3: wrote 9 at 0xd1c31930
Thread 1: wrote 1 at 0xd1c319b0
```

* * *

逐线程块分配示例：

```cpp
#include <stdlib.h>
__global__ void kernel() {
  __shared__ int *data;
  // 块中的第一个线程执行分配并通过共享内存与其他所有线程共享指针，以便访问可以合并。
  if (threadIdx.x == 0) {
    size_t size = blockDim.x * 8; // 为每个线程分配 8 字节。
    data = (int *)malloc(size);
  }
  __syncthreads();
  // 检查失败。
  if (data == nullptr)
    return;
  // 线程索引到内存中，确保合并。
  for (int i = 0; i < 8; ++i)
    data[i * blockDim.x + threadIdx.x] = threadIdx.x;
  // 确保所有线程在释放前完成。
  __syncthreads();
  // 只有一个线程可以释放内存！
  if (threadIdx.x == 0)
    free(data);
}
int main() {
  // 设置 16 兆字节的堆大小。
  hggcDeviceSetLimit(hggcLimitMallocHeapSize, 16 * 1024 * 1024);
  kernel<<<10, 128>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

* * *

跨核函数启动持续存在的分配示例：

```cpp
#include <stdio.h>
#include <stdlib.h>
const int NUM_BLOCKS = 2;
__device__ int *data_ptrs[NUM_BLOCKS]; // 每块指针。
__global__ void kernel() {
  // 只有块中的第一个线程执行分配，因为我们只需要对每个块进行一次分配。
  if (threadIdx.x == 0)
    data_ptrs[blockIdx.x] = (int *)malloc(blockDim.x * 4);
  __syncthreads();
  // 检查失败。
  if (data_ptrs[blockIdx.x] == nullptr)
    return;
  // 用所有线程并行清零数据。
  data_ptrs[blockIdx.x][threadIdx.x] = 0;
}
// 简单示例：将线程 ID 存储到每个元素中。
__global__ void memoryUse() {
  int *memoryPtr = data_ptrs[blockIdx.x];
  if (memoryPtr != nullptr)
    memoryPtr[threadIdx.x] += threadIdx.x;
}
// 打印缓冲区内容然后再释放它。
__global__ void memoryFree() {
  int *memoryPtr = data_ptrs[blockIdx.x];
  if (memoryPtr != nullptr)
    printf("Block %d, Thread %d: final input = %d\n", blockIdx.x, threadIdx.x,
           memoryPtr[threadIdx.x]);
  // 只在一个线程中释放！
  if (threadIdx.x == 0)
    free(memoryPtr);
}
int main() {
  hggcDeviceSetLimit(hggcLimitMallocHeapSize, 128 * 1024 * 1024);
  // 分配内存。
  kernel<<<NUM_BLOCKS, 3>>>();
  // 使用内存。
  memoryUse<<<NUM_BLOCKS, 3>>>();
  // 释放内存。
  memoryFree<<<NUM_BLOCKS, 3>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

将输出：

```cpp
Block 0, Thread 2: final input = 6
Block 0, Thread 1: final input = 3
Block 0, Thread 0: final input = 0
Block 1, Thread 2: final input = 6
Block 1, Thread 1: final input = 3
Block 1, Thread 0: final input = 0
```

##### 5.1.6.5. `alloca()` {#5165-alloca}

```cpp
__host__ __device__ void *alloca(size_t size);
```

`alloca()` 函数在调用者的栈内分配 `size` 字节的内存。返回值是指向分配内存的指针。当从设备代码调用函数时，内存的起始位置是 8 字节对齐的。当调用者从 `alloca()` 返回时，内存会自动释放。

示例：

```cpp
__device__ void deviceFunction(int num_items) {
  float2 *ptr = (float2 *)alloca(num_items * sizeof(float2));
  // 使用 ptr
}
```

#### 5.1.7. Lambda 表达式 {#517-lambda表达式}

编译器通过将 lambda 表达式或闭包类型 (C++11) 与最内层封闭函数作用域的执行空间相关联来确定其执行空间。如果没有封闭函数作用域，则执行空间被指定为 `__host__`。

也可以使用扩展 lambda 语法显式指定执行空间。

示例：

```cpp
// 全局作用域：无封闭函数 → __host__
auto preprocess = [](float%20x) { return x / 255.0f; }; // __host__

void buildPipeline() {
  // 封闭函数为 __host__ → lambda 继承 __host__
  auto normalize = [](float%20x) { return (x - 0.5f) * 2.0f; }; // __host__
  [](int%20n) { return n * n; }; // __host__, 闭包类型
}

__device__ void cuActivation(float *buf, int idx) {
  // 封闭函数为 __device__ → lambda 继承 __device__
  auto relu = [](float%20v) { return v > 0.f ? v : 0.f; }; // __device__
  buf[idx] = relu(buf[idx]);
}

__global__ void inferKernel(float *data, int n) {
  // __global__ 函数体视为 __device__ 空间
  auto sigmoid = [](float%20x) { return 1.0f / (1.0f + expf(-x)); }; // __device__
  int i = blockIdx.x * blockDim.x + threadIdx.x;
  if (i < n) data[i] = sigmoid(data[i]);
}

__host__ __device__ void quantize(float *src, int8_t *dst, int idx) {
  // 封闭函数为 __host__ __device__ → lambda 继承 __host__ __device__
  auto clampAndRound = [](float%20v) { return (int8_t)fminf(fmaxf(v, -128.f), 127.f); };
  dst[idx] = clampAndRound(src[idx]); // __host__ __device__
}

using ReduceFn = float (*)(float, float);
// 默认参数表达式位于全局作用域 → __host__
__device__ void treeReduce(float *buf, int n,
    ReduceFn combine = [](float%20a%2C%20float%20b) { return a + b; } /* __host__ */) {}
```

##### 5.1.7.1. Lambda 表达式和 `__global__` 函数参数 {#5171-lambda表达式和-global-函数参数}

只有当 lambda 表达式或闭包类型的执行空间是 `__device__` 或 `__host__ __device__` 时，才能将其用作 `__global__` 函数的参数。全局或命名空间范围的 lambda 表达式不能用作 `__global__` 函数中的参数。

示例（需要编译选项 `--extended-lambda`）：

```cpp
template <typename Op>
__global__ void applyElementwise(Op op, float *data, int n) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx < n) data[idx] = op(data[idx]);
}

__device__ void deviceLaunch(float *buf, int n) {
  // 设备端动态并行启动（需要分离编译 -rdc=true）
  applyElementwise<<<1, n>>>(
      [] __host__ __device__(float x) { return x * 0.1f; }, buf, n); // 正确, 扩展lambda
}

auto globalScaler = [] __host__ __device__(float x) { return x * 2.0f; };

void hostLaunch(float *d_buf, int n) {
  int blocks = (n + 255) / 256;
  // 正确：__device__ 扩展lambda
  applyElementwise<<<blocks, 256>>>(
      [] __device__(float x) { return fmaxf(x, 0.f); }, d_buf, n);
  // 正确：__host__ __device__ 扩展lambda
  applyElementwise<<<blocks, 256>>>(
      [] __host__ __device__(float x) { return tanhf(x); }, d_buf, n);
  // 错误：普通lambda具有 __host__ 执行空间，不能作为 __global__ 参数
  // applyElementwise<<<blocks, 256>>>([](float%20x) { return x; }, d_buf, n);
}
```

##### 5.1.7.2. 扩展 Lambda {#5172-扩展lambda}

`hggc` 标志 `--extended-lambda` 允许在 lambda 表达式中显式标注执行空间。这些标注应出现在 lambda 引入符之后和可选的 lambda 声明符之前。

当指定了 `--extended-lambda` 标志时，`hggc` 定义宏 `__HGGCCC_EXTENDED_LAMBDA__`。

* 扩展 lambda 是 `__host__` 或 `__host__ __device__` 函数的直接或嵌套块的作用域内定义。

* 扩展设备 lambda 是用 `__device__` 关键字注释的 lambda 表达式。

* 扩展主机-设备 lambda 是用 `__host__ __device__` 关键字注释的 lambda 表达式。

与标准 lambda 表达式不同，扩展 lambda 可以用作 `__global__` 函数中的类型参数。

示例：

```cpp
void hostFunction() {
  auto lambda1 = [] {}; // 不是扩展lambda: 没有显式执行空间注释
  auto lambda2 = [] __device__ {};          // 扩展lambda
  auto lambda3 = [] __host__ __device__ {}; // 扩展lambda
  auto lambda4 = [] __host__ {};            // 不是扩展lambda
}
__host__ __device__ void hostDeviceFunction() {
  auto lambda1 = [] {}; // 不是扩展lambda: 没有显式执行空间注释
  auto lambda2 = [] __device__ {};          // 扩展lambda
  auto lambda3 = [] __host__ __device__ {}; // 扩展lambda
  auto lambda4 = [] __host__ {};            // 不是扩展lambda
}
__device__ void deviceFunction() {
  // 此函数内的所有lambda都不是扩展lambda，因为外层函数不是 __host__ 或 __host__
  // __device__ 函数。
  auto lambda1 = [] {};
  auto lambda2 = [] __device__ {};
  auto lambda3 = [] __host__ __device__ {};
  auto lambda4 = [] __host__ {};
}
auto lambda = [] __host__ __device__ {
}; // 不是扩展lambda，因为它没有定义在 __host__ 或 __host__ __device__ 函数内
```

##### 5.1.7.3. 扩展 Lambda 的类型特征 {#5173-扩展lambda的类型特征}

编译器提供类型特征以在编译时检测扩展 lambda 的闭包类型。

| 返回值类型 | 函数名                                                    | 参数 | 说明                                                                                                                   |
| ---------- | --------------------------------------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------- |
| bool       | __hg_is_extended_device_lambda_closure_type               | type | 如果 `type` 是为扩展 `__device__` lambda 创建的闭包类，返回 `true`，否则返回 `false`                                   |
| bool       | __hg_is_extended_device_lambda_with_preserved_return_type | type | 如果 `type` 是为扩展 `__device__` lambda 创建的闭包类并且 lambda 是以尾随返回类型定义的，返回 `true`，否则返回 `false` |
| bool       | __hg_is_extended_host_device_lambda_closure_type          | type | 如果 `type` 是为扩展 `__host__ __device__` lambda 创建的闭包类，返回 `true`，否则返回 `false`                          |

* * *

lambda 类型特征可以在所有编译模式中使用，无论是否启用了 lambda 还是扩展 lambda。如果扩展 lambda 模式处于不可用状态，特征将始终返回 `false`。

示例——利用类型特征实现 PPU 核函数分派器的编译期校验：

```cpp
#include <type_traits>

template <typename Fn>
__global__ void dispatchKernel(Fn op, float *data, int n) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx < n) data[idx] = op(data[idx]);
}

template <typename Fn>
void launchOnPPU(Fn op, float *d_buf, int n) {
  // 编译期校验：仅接受设备端可执行的扩展 lambda
  static_assert(
      __hg_is_extended_device_lambda_closure_type(Fn) ||
      __hg_is_extended_host_device_lambda_closure_type(Fn),
      "launchOnPPU 仅接受 __device__ 或 __host__ __device__ 扩展 lambda");

  int threads = 256;
  int blocks = (n + threads - 1) / threads;
  dispatchKernel<<<blocks, threads>>>(op, d_buf, n);
}

void runInference(float *d_activations, int count) {
  // 扩展设备lambda：通过类型特征校验
  auto relu = [] __device__(float x) -> float { return x > 0.f ? x : 0.f; };
  static_assert(__hg_is_extended_device_lambda_closure_type(decltype(relu)));
  static_assert(!__hg_is_extended_host_device_lambda_closure_type(decltype(relu)));

  // 带保留返回类型的扩展设备lambda
  auto scale = [] __device__(float x) -> float { return x * 0.01f; };
  static_assert(
      __hg_is_extended_device_lambda_with_preserved_return_type(decltype(scale)));

  // 扩展主机-设备lambda：主机端也可调用
  auto clamp = [] __host__ __device__(float x) -> float {
    return x < -6.f ? -6.f : (x > 6.f ? 6.f : x);
  };
  static_assert(!__hg_is_extended_device_lambda_closure_type(decltype(clamp)));
  static_assert(__hg_is_extended_host_device_lambda_closure_type(decltype(clamp)));

  // 普通lambda（无执行空间注释）：不满足校验，无法传递给 launchOnPPU
  auto hostOnly = [](float%20x) { return x; };
  static_assert(!__hg_is_extended_device_lambda_closure_type(decltype(hostOnly)));
  static_assert(!__hg_is_extended_host_device_lambda_closure_type(decltype(hostOnly)));

  launchOnPPU(relu, d_activations, count);
  launchOnPPU(clamp, d_activations, count);
  // launchOnPPU(hostOnly, ...); // 编译错误：static_assert 失败
}
```

##### 5.1.7.4. 扩展 Lambda 限制 {#5174-扩展lambda限制}

HGGC 编译器在将代码传递给主机编译器之前，会把扩展 lambda 表达式替换为命名空间作用域的占位符类型。该占位符类型的模板参数中包含了**封闭函数**（enclosing function）的地址——这是正确实例化以扩展 lambda 闭包类型为模板参数的 `__global__` 函数模板所必需的。

封闭函数的确定规则可以归纳为：从扩展 lambda 所在的块作用域开始向外查找，直到找到第一个不是 lambda `operator()` 的函数，该函数即为封闭函数。具体而言：

1. 扩展 lambda 按定义总是位于某个 `__host__` 或 `__host__ __device__` 函数的块作用域中（可以是直接的或嵌套的）。
2. 若包含该扩展 lambda 的最内层函数不是另一个 lambda 的 `operator()`，则该函数即为封闭函数。
3. 若包含它的是嵌套 lambda 的 `operator()`，则继续向外层展开，找到最外层 lambda 所处的具名函数 `F`——`F` 即为封闭函数。
4. 若最外层 lambda 本身就定义在全局/命名空间作用域（而非任何函数体内），则不存在封闭函数。

```cpp
void hostFunction() {
  // 情况 2：直接位于具名函数中 → 封闭函数 = hostFunction
  auto lambda1 = [] __device__ {};

  auto lambda2 = [] {
    // 情况 3：嵌套在另一个 lambda 内 → 向外找到 hostFunction
    auto lambda3 = [] __host__ __device__ {};
  };
}

// 情况 4：最外层 lambda 在全局作用域 → 无封闭函数
auto global_lambda = [] {
  auto lambda4 = [] __host__ __device__ {};
};
```

##### 5.1.7.5. 主机-设备 Lambda 优化注意事项 {#5175-主机-设备lambda优化注意事项}

与设备专用 lambda 不同，主机-设备 lambda 可以从主机代码调用。如前所述，HGGC 编译器将主机代码中定义的扩展 lambda 表达式替换为命名占位符类型的实例。扩展主机-设备 lambda 的占位符类型通过间接函数调用调用原始 lambda 的 `operator()`。如果扩展 lambda 模式未激活，特征将始终返回 false。

间接函数调用的存在可能导致主机编译器对扩展主机-设备 lambda 的优化少于隐式或显式 `__host__` 的 lambda。在后一种情况下，主机编译器可以轻松地将 lambda 主体内联到调用上下文中。但是，当它遇到扩展主机-设备 lambda 时，主机编译器可能无法轻松地将原始 lambda 主体内联。

##### 5.1.7.6. `*this` 按值捕获 {#5176-this-按值捕获}

C++17 通过引入新的 `*this` 捕获模式解决了 this 的捕获问题。在这种模式下，编译器复制由 `*this` 表示的对象，而不是按值捕获 `this` 指针。`*this` 捕获模式在 [P0018R3](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0018r3.html) 中有更详细的描述。

HGGC 编译器支持在 `__device__` 和 `__global__` 函数内定义的 lambda 以及在主机代码中定义的扩展设备专用 lambda 的 `*this` 捕获模式，前提是使用了 `--extended-lambda` 标志。

示例：
```cpp
#include <cstdio>
template <typename T> __global__ void kernelFunction(T lambda) {
  printf("\n input = %d", lambda());
}
struct S {
  int var;
  __host__ __device__ S() : var(0) {};
  void run() {
    // 注意"*this"捕获规范
    auto lambda = [=, *this] __device__ {
      // 引用"var"导致'*this'表示的对象按值捕获，并且 PPU代码将访问'copy_of_star_this->var'。
      return var + 1;
    };
    // 核函数启动成功
    kernelFunction<<<1, 1>>>(lambda);
    hggcDeviceSynchronize(); // 等待 kernel 执行结束
  }
};
int main() {
  S s;
  s.run();
}
```

`*this` 捕获模式不允许用于主机代码中定义的非标注 lambda 或扩展主机-设备 lambda，除非所选语言方言启用 `*this` 捕获。以下是支持和不支持使用的示例：

```cpp
struct S {
  int var;
  __host__ __device__ S() : var(10) {};
  void hostFunction() {
    // 正确，在扩展设备专用lambda中使用
    auto lambda1 = [=, *this] __device__ { return var; };
    // 在扩展主机-设备lambda中使用，如果语言方言未启用*this捕获，则出错
    auto lambda2 = [=, *this] __host__ __device__ { return var; };
    // 在主机函数中的非标注lambda中使用，如果语言方言未启用*this捕获，则出错
    auto lambda3 = [=, *this] { return var; };
  }
  __device__ void deviceFunction() {
    // 正确，在设备专用函数中定义的lambda中使用
    auto lambda1 = [=, *this] __device__ { return var; };
    // 正确，在设备专用函数中定义的lambda中使用
    auto lambda2 = [=, *this] __host__ __device__ { return var; };
    // 正确，在设备专用函数中定义的lambda中使用
    auto lambda3 = [=, *this] { return var; };
  }
  __host__ __device__ void hostDeviceFunction() {
    // 正确，在扩展设备专用lambda中使用
    auto lambda1 = [=, *this] __device__ { return var; };
    // 在扩展主机-设备lambda中使用，如果语言方言未启用*this捕获，则出错
    auto lambda2 = [=, *this] __host__ __device__ { return var; };
    // 在主-设备函数中的未标注lambda中使用，如果语言方言未启用*this捕获，则出错
    auto lambda3 = [=, *this] { return var; };
  }
};
```

##### 5.1.7.7. 参数依赖查找 (ADL) {#5177-参数依赖查找-adl}

如前所述，HGGC 编译器在调用主机编译器之前，将扩展 lambda 表达式替换为占位符类型。占位符类型的其中一个模板参数使用封装原始 lambda 表达式的函数的地址。这可能导致附加的命名空间参与 [参数依赖查找 (ADL)](https://en.cppreference.com/w/cpp/language/adl.html)，从而导致主机编译器选择错误的函数。

示例：

```cpp
namespace N1 {
struct S {};
template <typename T> void f(T){};
}; // namespace N1
namespace N2 {
template <typename T> int f(T) { return 0; };
template <typename T> void run(T lambda) { f(lambda); }
} // namespace N2
void g(N1::S s) {
  // 对于扩展设备专用lambda，发送到主机编译器的代码被替换为占位符类型实例化表达式，
  // 结果，命名空间'N1'参与了N2::run主体中"f()"调用的 ADL 查找（Argument-Dependent Lookup），导致歧义。
  auto lambda = [=] __device__ {};
  N2::run(lambda);
}
```

在上面的例子中，HGGC 编译器用涉及 `N1` 命名空间的占位符类型替换了扩展 lambda。因此，`N1` 命名空间参与了 `N2::run()` 主体中 `f(in)` 的 ADL 查找，由于发现了多个重载候选项：`N1::f` 和 `N2::f`，导致主机编译失败。

#### 5.1.8. C/C++ 语言限制 {#518-cc-语言限制}


##### 5.1.8.1. 预留命名空间 {#5181-预留命名空间}

`hggc::`、`hg::` 和 `cooperative_groups::` 是 HGGC 的保留顶层命名空间，开发者代码不得在其中（包括其嵌套命名空间中）添加任何声明或定义，否则属于未定义行为。但允许在用户自定义的命名空间内部创建同名的嵌套命名空间。

以下示例汇总了合法与非法的用法（注意编译器不一定会报错，但违反规则的用法仍属未定义行为）：

```cpp
// ✗ 直接向保留命名空间添加符号
namespace hggc {
  void myHelper();  // 未定义行为
}

// ✓ 在用户命名空间内使用同名嵌套命名空间
namespace mylib {
  namespace hggc {
    void myHelper();  // 合法
  }
}

// ✗ using namespace 会将 mylib::hggc 的符号引入全局 hggc，效果等同于直接添加
using namespace mylib;
```

##### 5.1.8.2. 不支持的功能 {#5182-不支持的功能}

* 运行时类型信息 (RTTI) 和异常在设备代码中不受支持：

  * `typeid` 关键字

  * `dynamic_cast` 关键字

  * `try/catch/throw` 关键字

* `long double` 在设备代码中不受支持。

* 三字符组 (Trigraphs) 在任何平台上都不受支持。

* 用户定义的 `operator new`、`operator new[]`、`operator delete` 或 `operator delete[]` 不能用来替换编译器提供的内置函数，并且在主机和设备上都被认为是未定义行为。

##### 5.1.8.3. 指针和内存地址 {#5183-指针和内存地址}

指针解引用 (`*pointer`, `pointer->member`, `pointer[0]`) 只允许在同一执行空间中进行，其中相关内存位于该空间。以下情况会导致未定义行为，通常是段错误和应用程序终止。

* 在主机代码上解引用指向 [全局内存](#2231-全局内存)、[共享内存](#2232-共享内存) 或常量内存的指针。

* 在设备代码中解引用指向主机内存的指针。

以下限制适用于函数：

* 不允许在主机代码中获取 `__device__` 函数的地址。

* 在主机代码中获取的 `__global__` 函数的地址不能在设备代码中使用。同样，在设备代码中获取的 `__global__` 函数的地址不能在主机代码中使用。

通过 `hggcGetSymbolAddress()` 获取的 `__device__` 或 `__constant__` 变量的地址如 [内存空间说明符](#5222-内存空间说明符) 部分所述，只能在主机代码中使用。

##### 5.1.8.4. 变量 {#5184-变量}

###### 5.1.8.4.1. 局部变量 {#51841-局部变量}

在主机上执行的函数内的非 `extern` 变量声明不允许使用 `__device__`、`__shared__`、`__managed__` 和 `__constant__` 内存空间说明符。

示例：

```cpp
__host__ void hostFunction() {
  __device__ int d;   // 错误，主机函数内的 __device__ 变量声明
  __shared__ int s;   // 错误，主机函数内的 __shared__ 变量声明
  __managed__ int m;  // 错误，主机函数内的 __managed__ 变量声明
  __constant__ int c; // 错误，主机函数内的 __constant__ 变量声明
  extern __device__ int e; // 正确，extern __device__ 变量
}
```

在设备上执行的函数内的既不是 `extern` 也不是 `static` 的变量声明不允许使用 `__device__`、`__constant__` 和 `__managed__` 内存空间说明符。

```cpp
__device__ void deviceFunction() {
  int i;              // 正确，__device__ 变量
  __constant__ int c; // 错误，设备函数内的 __constant__ 变量声明
  __managed__ int m;  // 错误，设备函数内的 __managed__ 变量声明
  extern __device__ int e; // 正确，extern __device__ 变量
}
```

另请参见 [静态变量](#51844-static-变量) 部分。

###### 5.1.8.4.2. `const` 限定变量 {#51842-const-限定变量}

在全局、命名空间或类作用域中声明的没有内存空间注释（`__device__` 或 `__constant__`）的 `const` 限定变量被视为主机变量。设备代码不能包含对变量的引用或获取变量的地址。

如果满足以下条件，则可以直接在设备代码中使用该变量：

* 它在使用点之前已用常量表达式初始化，

* 类型不是 `volatile` 限定的，

* 它具有以下类型之一：

  * 内置整数类型，

  * 内置浮点类型。

从 C++14 开始，建议使用 `constexpr` 或 `inline constexpr`（C++17）变量代替 `const` 限定变量。`constexpr` 变量不受类型相同限制，可以直接在设备代码中使用。

`__managed__` 变量不支持 `const` 限定类型。

示例：

```cpp
const int ConstInt = 1;
const float ConstFloat = 1.0f;
inline constexpr float Constexpr = 1.0f; // C++17
struct S {
  static const int ConstInt = 1;
  static const float ConstFloat = 1.0f; // 错误，静态const变量不能是float
  static inline constexpr float Constexpr = 1.0f; // 正确
};
extern const int ExternVar;
__device__ void foo() {
  int array1[ConstInt];             // 正确
  int array2[S::ConstInt];          // 正确
  const float var1 = ConstFloat;    // 正确
  constexpr float var2 = Constexpr; // 正确
  int var3 = ExternVar;  // 错误，"ExternVar"未用常量表达式初始化
  int &var4 = ConstInt;  // 错误，对主机变量的引用
  int *var5 = &ConstInt; // 错误，主机变量的地址
}
```

###### 5.1.8.4.3. `volatile` 限定变量 {#51843-volatile-限定变量}

HGGC 保留对 `volatile` 关键字的支持以维持 ISO C++ 兼容性，但在 PPU 架构上其实际用途极为有限（参见 [P1152R0 提案](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1152r0.html#prop) 中关于弃用方向的讨论）。

对 `volatile` 限定对象的读写操作在 HGGC 中**不具有原子性**。编译器会将其生成为一条或多条带 `volatile` 语义的 TIX 指令，但这些指令既不保证全局可见的内存操作顺序，也不保证硬件执行的内存事务数量与 TIX 指令数量一一对应。

因此，`volatile` **不适用于线程间同步**。原子内存操作不仅提供线程间同步保证，且在 PPU 架构上比 `volatile` 操作具有更好的性能表现。需要线程间可见性和顺序保证时，应使用 `hggc::atomic_ref`、`hggc::atomic` 或 [传统原子函数](#5254-原子函数)。以下三个示例分别演示了如何通过不同的原子机制实现一个"生产者写入数据 → 消费者等待并读取"的协作模式。

**方式一：`hggc::atomic_ref`（适合已有普通变量的场景）**

```cpp
#include <hggc/atomic>

__global__ void producerConsumer(int *ready, int *payload) {
  // 使用devices作用域的原子引用
  hggc::atomic_ref<int, hggc::thread_scope_device> status_flag{*ready};

  switch(threadIdx.x) {
  case 0: // 生产者线程
    *payload = 2048;                    // 写入有效负载
    status_flag.store(1, hggc::memory_order_release);  // 信号发布
    break;

  case 1: // 消费者线程
    // 等待信号就绪
    while (status_flag.load(hggc::memory_order_acquire) != 1) {
        __nanosleep(100);  // 添加停顿减少忙等待开销
    }
    if (*payload != 2048) {
        __trap();  // 提供更明确的内联陷阱指令
    }
    break;
  }
}
```

---

**方式二：`hggc::atomic`（适合变量从一开始就定义为原子类型的场景）**

```cpp
#include <hggc/atomic>
__global__ void atomicHandoff(
    hggc::atomic<int, hggc::thread_scope_device> *status_flag,
    int *payload) {
  switch (threadIdx.x) {
    case 0:
      *payload = 2048;
      status_flag->store(1, hggc::memory_order_release);
      break;
    case 1:
      while (status_flag->load(hggc::memory_order_acquire) != 1) {
        __nanosleep(100);  // 添加停顿减少忙等待开销
      };
      if (*payload != 2048)
        __trap();
      break;
  }
}
```

---

**方式三：传统原子函数 + `__threadfence()`（适合不使用 C++ 原子类型的代码）**

```cpp
__global__ void legacyAtomicHandoff(int *status_flag, int *payload) {
  switch (threadIdx.x) {
    case 0:
      *payload = 2048;
      __threadfence();
      atomicExch(status_flag, 1);
      break;
    case 1:
      while (atomicAdd(status_flag, 0) != 1) {
        __nanosleep(100);  // 添加停顿减少忙等待开销
      };
      __threadfence();
      if (*payload != 2048)
        __trap();
      break;
  }
}
```


###### 5.1.8.4.4. `static` 变量 {#51844-static-变量}

在以下情况下，设备代码中允许使用 `static` 变量：

* 在 `__global__` 或仅 `__device__` 函数内。

* 在 `__host__ __device__` 函数内：

  * 没有显式内存空间的 `static` 变量（自动推断）。

  * 具有显式内存空间的 `static` 变量，例如 `static __device__/__constant__/__shared__/__managed__`，仅当定义了 `__HGGC_ARCH__` 时才允许。

在 `__host__ __device__` 函数内的 `static` 变量根据执行空间的不同持有不同的值。

下面给出了函数作用域 `static` 变量的合法和非法使用的示例。

```cpp
struct TrivialStruct {
  int x;
};
struct NonTrivialStruct {
  __device__ NonTrivialStruct(int x) {}
};
__device__ void deviceFunction(int x) {
  static int var1;             // 正确，隐式 __device__ 内存空间说明符
  static int var2 = 11;        // 正确，隐式 __device__ 内存空间说明符
  static int var3 = x;         // 错误，不允许动态初始化
  static __managed__ int var4; // 正确，显式
  static __device__ int var5;  // 正确，显式
  static __constant__ int var6; // 正确，显式
  static __shared__ int var7;   // 正确，显式
  static TrivialStruct s1; // 正确，隐式 __device__ 内存空间说明符
  static TrivialStruct s2{22}; // 正确，隐式 __device__ 内存空间说明符
  static TrivialStruct s3{x};    // 错误，不允许动态初始化
  static NonTrivialStruct s4{3}; // 错误，不允许动态初始化
}
```

* * *

```cpp
__host__ __device__ void hostDeviceFunction() {
  static int var1; // 正确，隐式 __device__ 内存空间说明符
#ifdef __HGGC_ARCH__
  static __device__ int var2; // 正确，声明仅在设备编译期间可见
#else
  static int var3; // 正确，声明仅在主机编译期间可见
#endif
}
```

* * *

```cpp
#include <cassert>
__host__ __device__ int hostDeviceFunction() {
  static int var = 0;
  var++;
  return var;
}
__global__ void kernel() {
  int ret = hostDeviceFunction(); // var = 1
  assert(ret == 4);               // 失败
}
int main() {
  hostDeviceFunction();           // var = 1
  hostDeviceFunction();           // var = 2
  int ret = hostDeviceFunction(); // var = 3
  assert(ret == 3);               // OK
  kernel<<<1, 1>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
}
```

###### 5.1.8.4.5. `extern` 变量 {#51845-extern-变量}

在非 RDC 模式下编译时，不能使用 `extern` 关键字定义具有外部链接的 `__device__`、`__shared__`、`__managed__` 和 `__constant__` 变量。

唯一的例外是如 [共享内存的动态分配](#22322-共享内存的动态分配) 部分所述的动态分配的 `__shared__` 变量。

```cpp
__device__ int var1;        // OK
extern __device__ int var2; // 非 RDC 模式（Relocatable Device Code）下的报警
extern __shared__ int var3; // OK
```

##### 5.1.8.5. 函数 {#5185-函数}

###### 5.1.8.5.1. 递归 {#51851-递归}

`__global__` 函数不支持递归，而 `__device__` 和 `__host__ __device__` 函数没有这样的限制。

###### 5.1.8.5.2. 外部链接 {#51852-外部链接}

具有外部链接的设备变量或函数需要跨多个翻译单元的分离编译模式。

###### 5.1.8.5.3. 形式参数 {#51853-形式参数}

形式参数不允许使用 `__device__`、`__shared__`、`__managed__` 和 `__constant__` 内存空间说明符（但编译不会报错）。

```cpp
void deviceFunction1(__device__ int x) { } // 错误，__device__ 参数
void deviceFunction2(__shared__ int x) { } // 错误，__shared__ 参数
```

###### 5.1.8.5.4. `__global__` 函数参数 {#51854-global-函数参数}

`__global__` 函数有以下限制：

* 它不能有可变数量的参数，即 C 的省略号语法 `...` 和 `va_list` 类型。允许使用 C++11 可变参数模板，但受限于 [`__global__` 可变参数模板](#5195-global-变参模板) 部分描述的限制。

* 函数参数通过常量内存传递到设备，其总大小限制为 32764 字节。

* 函数参数不能按引用传递或按右值引用传递。

* 函数参数不能是 `std::initializer_list` 类型。

* 多态类参数（`virtual`）被认为是未定义行为。

* Lambda 表达式和闭包类型是允许的，但受限于 [Lambda 表达式和 `__global__` 函数参数](#5171-lambda表达式和-global-函数参数) 部分描述的限制。

###### 5.1.8.5.5. `__global__` 函数参数传递 {#51855-global-函数参数传递}

从设备端 [启动](#2122-核函数的启动) `__global__` 函数时，所有参数必须同时满足**平凡可复制**（trivially copyable）和**平凡可析构**（trivially destructible）。

从主机端启动时则允许非平凡类型的参数，但 HGGC Runtime 的传参机制与标准 C++ 的值传递语义存在两处关键差异，开发者需确保程序逻辑不依赖于被跳过的行为：

1. **参数通过按位拷贝传递，拷贝构造函数不会被调用**

HGGC Runtime 使用 `memcpy` 将参数的原始字节从主机内存搬运到设备端，因此用户自定义拷贝构造函数中的初始化逻辑和副作用均不会执行。这对于在拷贝构造中建立自引用指针的类型尤其危险：

```cpp
#include <cassert>
struct Widget {
  int id = 1;
  int *self_ref;
  Widget() = default;
  __host__ __device__ Widget(const Widget &) { self_ref = &id; }
};
__global__ void checkWidget(Widget w) {
  // 断言失败：self_ref 仍指向主机端旧对象的 id 地址
  assert(w.self_ref == &w.id);
}
void hostCheck(Widget w) {
  assert(w.self_ref == &w.id); // 主机端正常：拷贝构造函数被调用
}
int main() {
  Widget w;
  hostCheck(w);
  checkWidget<<<1, 1>>>(w);
  hggcDeviceSynchronize();
}
```

2. **参数的析构函数可能在核函数完成前执行**

由于核函数启动是异步的，主机端会在 `<<<>>>` 返回后立即继续执行。当参数离开作用域时，主机端析构函数会正常触发——此时设备端的核函数可能尚未开始或仍在运行。对于析构函数有共享状态副作用的类型，这可能引发数据竞争：

```cpp
#include <cassert>
__managed__ int counter = 0;
struct Guard {
  __host__ __device__ ~Guard() { counter = 99; }
};
__global__ void useGuard(Guard g) {
  // 可能失败：主机端析构函数已将 counter 设为 99
  assert(counter == 0);
}
int main() {
  Guard g;
  useGuard<<<1, 1>>>(g);
  // g 离开 main 作用域时析构 → counter 被主机端修改
  // 而此时 useGuard 可能仍在设备上执行
  hggcDeviceSynchronize();
}
```

##### 5.1.8.6. 类 {#5186-类}

###### 5.1.8.6.1. 类类型变量 {#51861-类类型变量}

使用 `__device__`、`__constant__`、`__managed__` 或 `__shared__` 内存空间说明符定义的变量，其类类型必须拥有**空构造函数**和**空析构函数**。所谓"空"是指平凡的（trivial），或者在当前翻译单元中满足下表所列的全部条件：

| 判定对象       | 需同时满足的条件                                                                 |
| -------------- | -------------------------------------------------------------------------------- |
| **空构造函数** | ① 构造函数有显式定义                                                             |
|                | ② 无参数、无成员初始值列表、函数体为空 `{}`                                      |
|                | ③ 类自身无 `virtual` 函数、无 `virtual` 基类、无非 `static` 数据成员的就地初始值 |
|                | ④ 所有基类的默认构造函数均满足"空"条件                                           |
|                | ⑤ 所有非 `static` 数据成员（含数组元素）的默认构造函数均满足"空"条件             |
| **空析构函数** | ① 析构函数有显式定义                                                             |
|                | ② 函数体为空 `{}`                                                                |
|                | ③ 类自身无 `virtual` 函数、无 `virtual` 基类                                     |
|                | ④ 所有基类的析构函数均满足"空"条件                                               |
|                | ⑤ 所有非 `static` 数据成员（含数组元素）的析构函数均满足"空"条件                 |

上述条件具有递归性——基类和成员的构造/析构函数同样需要满足对应的"空"规则。

!!! note "与 C++ 标准的关系"

    上述判定条件与 C++ 标准中 **trivially constructible** 和 **trivially destructible** 的概念密切相关（参见 [*[class.ctor]*](https://eel.is/c++draft/class.ctor) 和 [*[class.dtor]*](https://eel.is/c++draft/class.dtor)）。具体的判定条件以上表为准。

###### 5.1.8.6.2. 数据成员 {#51862-数据成员}

`class`、`struct` 和 `union` 数据成员不允许使用 `__device__`、`__shared__`、`__managed__` 和 `__constant__` 内存空间说明符。

仅支持在编译时计算的 `static` 数据成员，如 const 限定和 `constexpr` 变量。

```cpp
struct S {
  static const int var1 = 1;
  static constexpr int var2 = 1;        // C++11
  static inline constexpr int var3 = 1; // C++17
};
```

###### 5.1.8.6.3. 成员函数 {#51863-成员函数}

`__global__` 函数不能是 `struct`、`class` 或 `union` 的成员。

`__global__` 函数可以在 `friend` 声明中使用，但不能定义。

示例：

```cpp
struct S {
  friend __global__ void f();   // 正确，仅友元声明
  friend __global__ void g() {} // 错误，友元定义
};
```

###### 5.1.8.6.4. 隐式声明和非虚显式默认函数 {#51864-隐式声明和非虚显式默认函数}

隐式声明的特殊成员函数是指用户未声明但由编译器为类声明的那些；显式默认函数是用户声明但标记为 `= default` 的函数。隐式声明或显式默认的特殊成员函数包括默认构造函数、拷贝构造函数、移动构造函数、拷贝赋值运算符、移动赋值运算符和析构函数。

假设 `F` 表示在其首次声明时要么隐式声明要么显式默认的非 `virtual` 函数。对于 `F` 的执行空间说明符是调用它的所有函数的执行空间说明符的并集。请注意，对此分析时，`__global__` 调用方被视为 `__device__` 调用方。例如：

```cpp
class TensorMeta {
  int rank;
  int dims[4];

public:
  __host__ __device__ TensorMeta() : rank(0), dims{} {}
};

class ActivationDesc : public TensorMeta {
  float alpha;
};

class ConvConfig : public TensorMeta {
  int padH, padW;
};

__device__ void initOnDevice(float *workspace) {
  ActivationDesc act;   // 仅从 __device__ 调用
  ConvConfig conv;      // 从 __device__ 和 __host__ 都调用
}

__host__ void prepareOnHost() {
  ConvConfig conv;      // 从 __host__ 调用
}
```

在这种情况下，隐式声明的构造函数 `ActivationDesc::ActivationDesc()` 将被视为 `__device__` 函数，因为它仅从 `__device__` 函数 `initOnDevice()` 调用。隐式声明的构造函数 `ConvConfig::ConvConfig()` 将被视为 `__host__ __device__` 函数，因为它既从 `__device__` 函数 `initOnDevice()` 又从 `__host__` 函数 `prepareOnHost()` 调用。

此外，如果 `F` 是隐式声明的 `virtual` 函数（例如，`virtual` 析构函数），如果被 `F` 重写的虚函数 `D` 没有被隐式声明，则 `D` 的执行空间将被添加到 `F` 的执行空间集合中。

例如：

```cpp
struct LayerBase {
  virtual __host__ __device__ ~LayerBase() {}
};
struct ReluLayer : LayerBase {
}; // 隐式声明的虚析构函数
   // ~ReluLayer() 具有 __host__ __device__ 执行空间说明符

struct DeviceAllocator {
  virtual __device__ ~DeviceAllocator() = default;
};
struct PoolAllocator : DeviceAllocator {};
// 隐式声明的虚析构函数
// ~PoolAllocator() 具有 __device__ 执行空间说明符
```

###### 5.1.8.6.5. 多态类 {#51865-多态类}

多态类，即具有 `virtual` 函数、派生自其他多态类或具有多态数据成员的类，受到以下限制：

* 从设备到主机或从主机到设备复制多态对象，包括 `__global__` 函数参数是未定义行为。

* 重写的 `virtual` 函数的执行空间必须与基类中函数的执行空间匹配。

示例：

```cpp
struct C {
  virtual __host__ __device__ void f() {}
};
__global__ void kernel(C c) {
  c.f(); // 未定义行为
}
int main() {
  C c;
  kernel<<<1, 1>>>(c);
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
}
```

* * *

```cpp
struct Base {
  virtual __host__ __device__ void f() {}
};
struct Derived : Base {
  __device__ void f() override {} // 错误
};
```

##### 5.1.8.7. 模板 {#5187-模板}

如果以下任一条件成立，则类型不能用作 `__global__` 函数或 `__device__/__constant__` 变量（C++14）的模板参数：

* 该类型在 `__host__` 或 `__host__ __device__` 函数作用域内定义。

* 该类型未命名，如匿名结构体或 lambda 表达式，该类型在 `__device__` 或 `__global__` 函数内部除外。

* 该类型是具有 `private` 或 `protected` 的类成员，该类在 `__device__` 或 `__global__` 函数内部除外。

* 该类型由上述任何类型的组合构成。

示例：

```cpp
template <typename T> __global__ void kernel() {}
template <typename T> __device__ int var; // C++14
struct {
  int var;
} S;
void hostFunction() {
  struct LocalStruct {};
  kernel<LocalStruct><<<1, 1>>>(); // 错误，LocalStruct 在主机函数内定义
  int data = 1;
  hggcMemcpyToSymbol(var<LocalStruct>, &data, sizeof(data)); // 错误，同上
  kernel<decltype(S)><<<1, 1>>>(); // 错误，未命名类型
}
class C {
private:
  struct PrivateStruct {};

public:
  static void launch() {
    kernel<PrivateStruct><<<1, 1>>>(); // 错误，私有类型
  }
};
```

#### 5.1.9. C++11 限制 {#519-c11-限制}

##### 5.1.9.1. `inline` 命名空间 {#5191-inline-命名空间}

当在封闭命名空间中定义了具有相同名称和类型签名的另一个实体时，不允许在 `inline` 命名空间内定义以下实体：

* `__global__` 函数。

* `__device__`, `__constant__`, `__managed__`, `__shared__` 变量。

* 具有表面或纹理类型的变量，例如 `hggcSurfaceObject_t` 或 `hggcTextureObject_t`。

示例：

```cpp
__device__ int var; // 全局作用域
inline namespace NS {
__device__ int var; // 命名空间作用域，编译报错
} // namespace NS
```

##### 5.1.9.2. `inline` 匿名命名空间 {#5192-inline-匿名命名空间}

以下实体不能在 `inline` 匿名命名空间内的命名空间作用域中声明：

* `__global__` 函数。

* `__device__`, `__constant__`, `__managed__`, `__shared__` 变量。

* 具有表面或纹理类型的变量，例如 `hggcSurfaceObject_t` 或 `hggcTextureObject_t`。

##### 5.1.9.3. `constexpr` 函数 {#5193-constexpr-函数}

默认情况下，`constexpr` 函数不能从具有不兼容执行空间的函数中调用，这与标准函数的方式相同。

* 在主机代码生成阶段（即 `__HGGC_ARCH__` 宏未定义时）从主机函数调用仅设备的 `constexpr` 函数。示例：

```cpp
constexpr __device__ int deviceFunction() { return 0; }

int main() {
  int x = deviceFunction(); // 错误，在主机代码中调用仅设备的constexpr函数
}
```

请注意，即使相应的模板函数标记了关键字 `constexpr`，函数模板特化也可能不是 `constexpr` 函数。

**宽松的 constexpr 函数支持**

实验性的 `hggc` 标志 `--expt-relaxed-constexpr` 可用于放宽对 `__host__` 和 `__device__` 函数的这种约束。但是，`__global__` 函数不能声明为 `constexpr`。`hggc` 还将定义宏 `__HGGCCC_RELAXED_CONSTEXPR__`。

当指定此标志时，编译器将支持上述跨执行空间调用，如下所示：

1. 如果在需要常量求值的上下文中发生跨执行空间的 `constexpr` 函数调用，则支持该调用，例如 `constexpr` 变量的初始化器。示例：

```cpp
constexpr __host__ int hostFunction(int x) { return x; };
__global__ void kernelFunction() {
  constexpr int val =
      hostFunction(1); // 正确，调用发生在需要常量求值的上下文中。
}
constexpr __device__ int deviceFunction(int x) { return x; }
int main() {
  constexpr int val =
      deviceFunction(1); // 正确，调用发生在需要常量求值的上下文中。
}
```

2. 在设备代码生成期间为仅主机 constexpr 函数的函数体生成设备代码，它未使用或仅在 `constexpr` 上下文中调用除外。示例：

```cpp
// 注意："hostFunction"在生成的设备代码中发出，因为它在非constexpr上下文中从设备代码调用
constexpr int hostFunction(int x) { return x; }

__device__ int deviceFunction(int in) {
  return hostFunction(in); // 正确，即使参数不是常量表达式
}
```

3. 适用于设备函数的所有代码限制也适用于从设备代码调用的仅主机 `constexpr` 函数。然而，编译器可能不会为与编译过程相关的限制发出任何构建时诊断。

例如，以下代码模式在主机函数的函数体中不受支持。这类似于任何设备函数；但是，可能不会生成编译器诊断。

* One-Definition Rule (ODR)-使用主机变量或仅主机非 `constexpr` 函数。示例：

```cpp
int var;
constexpr int *hostFunction() { return &var; };
__device__ int deviceFunction() {
  // 错误，hostFunction()试图引用主机变量 'var'。
  return *hostFunction();
}
```

* 使用异常 `throw/catch` 和运行时类型信息 `typeid/dynamic_cast`。示例（不会有编译时错误）：

```cpp
struct Base {};
struct Derived : public Base {};
// 注意："hostFunction"在生成的设备代码中发出
constexpr int hostFunction(Base *memoryPtr) {
  // 错误，在 PPU 上执行的代码中使用typeid
  if (typeid(memoryPtr) == typeid(Derived))
    return 1;
  else
    throw int{1}; // 错误，在 PPU 上执行的代码中使用throw
}
__device__ void deviceFunction() {
  Derived d;
  // 错误，hostFunction()尝试使用typeid和throw()，这些在 PPU 上执行的代码中是不允许的
  int val = hostFunction(&d);
}
```

4. 在主机代码生成期间，仅设备 `constexpr` 函数的函数体保留在发送到主机编译器的代码中。然而，如果设备函数的主体尝试 ODR-使用命名空间作用域设备变量或非 `constexpr` 设备函数，则从主机代码调用设备函数不受支持。虽然代码可能在没有编译器诊断的情况下构建，但它可能在运行时表现不正确。示例：

```cpp
__device__ int var;
constexpr __device__ int *deviceFunction() { return &var; };
int hostFunction() {
  // 错误，deviceFunction()试图引用设备变量 'var'，代码将编译，但不会正确执行。
  return *deviceFunction();
}
```

警告

由于上述限制以及缺少对错误使用的编译器诊断，建议避免从设备代码调用标准 C++头文件中的函数 `std::`。此类函数的实现因主机平台而异。相反，强烈建议调用 HGGC C++ 标准库中的等效功能，在 `hggc::std::` 命名空间中。

##### 5.1.9.4. `constexpr` 变量 {#5194-constexpr-变量}

默认情况下，`constexpr` 变量不能在具有不兼容执行空间的函数中使用，这与标准变量的方式相同。

在以下情况下，可以直接在设备代码中使用 `constexpr` 变量：

* C++标量类型，不包括指针和指向成员的指针类型：

  * `nullptr_t`。

  * `bool`。

  * 整数类型：`char`、`signed char`、`unsigned`、`long long` 等。

  * 浮点类型：`float`、`double`。

  * 枚举器：`enum` 和 `enum class`。

* 类类型：具有 `constexpr` 构造函数的 `class`、`struct` 和 `union`。

* 上述类型的原始数组，例如 `int[]`，仅当它们在 `constexpr` `__device__` 或 `__host__ __device__` 函数内部使用时。

不允许使用 `constexpr __managed__` 和 `constexpr __shared__` 变量。

示例（不会有编译时错误）：

```cpp
constexpr int ConstexprVar = 1; // 标量类型
struct S {
  static constexpr int ConstexprVar = 2;
};
constexpr S s = S{}; // 类类型
constexpr int array[] = {1, 2, 3};
__device__ constexpr int get(int idx) {
  return array[idx]; // 正确
}
__device__ void f(int idx) {
  int var1 = ConstexprVar;         // 正确
  int var2 = S::ConstexprVar;      // 正确
  const int &var3 = ConstexprVar;  // 错误，引用主机constexpr变量
  const int *var4 = &ConstexprVar; // 错误，主机constexpr变量的地址
  int var5 = get(2);               // 正确，'get(2)'是常量表达式。
  int var6 = get(idx);             // 错误，'get(idx)'不是常量表达式
  int var7 = array[2];             // 错误，'array'不是标量类型。
  S var8 = s;                      // 正确
}
```

##### 5.1.9.5. `__global__` 变参模板 {#5195-global-变参模板}

变参 `__global__` 函数模板有以下限制：

* 只允许单个包参数。

* 包参数必须列在模板参数列表的最后。

示例（单独的以下例子中的声明不会有编译时错误）：

```cpp
template <typename... Pack> __global__ void kernel1(); // 正确
template <typename... Pack, template T>
__global__ void kernel2(); // 错误，参数包不是最后一个参数
template <typename... Pack1, typename... Pack2>
__global__ void kernel3(); // 错误，超过一个参数包
```

##### 5.1.9.6. 默认函数 `= default` {#5196-默认函数--default}

HGGC 编译器推断显式默认成员函数的执行空间，如 [隐式声明和非虚显式默认函数](#51864-隐式声明和非虚显式默认函数) 中所述。

除了函数在类外定义或为 `virtual` 函数的情况外，编译器会忽略显式默认函数上的执行空间说明符。

示例（不会有编译时错误）：

```cpp
struct S1 {
  S1() = default;
};
void hostFunction() {
  S1 s; // __host__ __device__ 构造函数
}
__device__ void deviceFunction1() {
  S1 s; // __host__ __device__ 构造函数
}
struct S2 {
  __device__ S2() = default; // 错误，__device__ 注解被忽略
};
struct S3 {
  __host__ S3();
};
S3::S3() = default; // 类外定义，不被忽略
__device__ void deviceFunction2() {
  S3 s; // 错误，__host__ 构造函数
}
struct S4 {
  //  S4::~S4 具有主机执行空间，不被忽略，因为是虚函数
  virtual __host__ ~S4() = default;
};
__device__ void deviceFunction3() {
  S4 s;
  // 对's'的隐式析构函数调用：
  // 错误：从 __device__ 函数 'deviceFunction3' 到 __host__ 函数
  // 'S4::~S4' 的调用
}
```

##### 5.1.9.7. `[hggc::]std::initializer_list` {#5197-hggcstdinitializer_list}

默认情况下，HGGC 编译器隐式认为 `[hggc::]std::initializer_list` 的成员函数具有 `__host__ __device__` 执行空间说明符，因此它们可以直接从设备代码调用。

`hggc` 标志 `--no-host-device-initializer-list` 禁用此行为；然后将 `[hggc::]std::initializer_list` 的成员函数视为 `__host__` 函数，并且不能直接从设备代码调用。

`__global__` 函数不能具有 `[hggc::]std::initializer_list` 类型的参数。

示例：

```cpp
#include <initializer_list>
__device__ void f(std::initializer_list<int> in) {}
__device__ void deviceFunction() {
  f({4, 5, 6}); // (a) 仅包含常量表达式的初始化列表。
  int i = 1;
  f({i, 5, 6}); // (b) 至少有一个非常量元素的初始化列表。
                  // 此形式可能比(a)有更好的性能。
}
```

##### 5.1.9.8. `[hggc::]std::move`, `[hggc::]std::forward` {#5198-hggcstdmove-hggcstdforward}

默认情况下，HGGC 编译器隐式认为 `std::move` 和 `std::forward` 函数模板具有 `__host__ __device__` 执行空间说明符，因此它们可以直接从设备代码调用。

!!! note

    `hggc::std::move` 和 `hggc::std::forward` 始终具有 `__host__ __device__` 执行空间。

#### 5.1.10. C++14 限制 {#5110-c14-限制}

##### 5.1.10.1. 具有自动推导返回类型的函数 {#51101-具有自动推导返回类型的函数}

`__global__` 函数不能具有推导返回类型 `auto`。

!!! note

    HGGC 前端编译器在调用主机编译器之前将函数声明更改为具有 `void` 返回类型。这可能会破坏在主机代码中对 `__device__` 函数的推导返回类型的推断。因此，HGGC 编译器将在设备函数体之外引用此类推导返回类型时可能发出编译时错误。

示例：

```cpp
// 推导返回类型 decltype(auto) 具有相同的行为
__device__ auto deviceFunction(int x) { return x; }
__global__ void kernelFunction() {
  int x = sizeof(deviceFunction(1)); // 正确
}
void hostFunction() {
  struct S {
    // 错误，在设备函数体外引用
    decltype(deviceFunction(1)) var;
    S() : var(1) {}
  };
}
```

#### 5.1.11. C++17 限制 {#5111-c17-限制}

##### 5.1.11.1. `inline` 变量 {#51111-inline-变量}

在单个翻译单元中，使用 `inline` 变量与常规变量相比没有提供任何额外功能，也没有提供任何实际优势。

!!! note

    当使用 `gcc/g++` 主机编译器时，使用 `__managed__` 内存空间说明符声明的 `inline` 变量可能对调试器不可见。

示例：

```cpp
static inline __device__ int var1;
namespace {
inline __device__ int var2;
inline __shared__ int var3;
static inline __device__ int var4;
inline __device__ int var5;
} // namespace
```

##### 5.1.11.2. 结构化绑定 {#51112-结构化绑定}

结构化绑定不能使用内存空间说明符声明，如 `__device__`、`__shared__`、`__constant__` 或 `__managed__`。

示例：

```cpp
struct S {
  int x, y;
};
__device__ auto [a, b] = S{1, 2}; // 错误
```

#### 5.1.12. C++20 限制 {#5112-c20-限制}

##### 5.1.12.1. 三路比较运算符 {#51121-三路比较运算符}

三路比较运算符 (`<=>`) 在设备代码中受支持，但某些使用隐式依赖于 C++标准库的功能，该功能由主机实现提供。使用这些运算符可能需要指定标志 `--expt-relaxed-constexpr` 来消除警告，并且该功能要求主机实现在满足设备代码的要求。

示例：

```cpp
#include <compare> // std::strong_ordering 实现
struct S {
  int x, y;
  auto operator<=>(const S &) const = default;                      // (a)
  __host__ __device__ bool operator<=>(int) const { return false; } // (b)
};
__host__ __device__ bool hostDeviceFunction(S a, S b) {
  if (a <=> 1) // 正确，调用用户定义的主机设备重载 (b)
    return true;
  // 正确，调用隐式声明的函数 (a)
  // 注意：它需要设备兼容的std::strong_ordering在头文件<compare>中提供的实现
  return a < b;
}
```

##### 5.1.12.2. `consteval` 函数 {#51122-consteval-函数}

`consteval` 函数可以从主机和设备代码中调用，与其执行空间无关。

示例：

```cpp
consteval int hostFunction() { return 10; }
__device__ consteval int deviceFunction() { return 10; }
__device__ int f() {
  return hostFunction(); // 正确，即使从设备代码调用
}
__host__ __device__ int hostDeviceFunction() {
  return deviceFunction(); // 正确，即使从主机设备代码调用
}
```

### 5.2. HGGC 语言扩展 {#52-hggc-语言扩展}


#### 5.2.1. PPU 特有扩展 {#521-ppu-特有扩展}

本节汇集 PPU 平台特有的语言扩展，包括 HGGC 特定宏与函数，以及编译器优化提示。

##### 5.2.1.1. HGGC 特定宏 {#5211-hggc-特定宏}

###### 5.2.1.1.1. `__HGGC_ARCH__` {#52111-hggc_arch}

宏 `__HGGC_ARCH__` 表示正在编译代码的 PPU 的 [虚拟架构](/docs_center/doc_detail/index.html?projectId=39&chapterId=1056#5-命令行选项)。其值可能与设备的实际计算能力不同。此宏可以编写针对特定 PPU 架构的专门代码路径，这对于获得最佳性能或使用特定架构特性和指令可能是必需的。该宏还可以用于区分主机代码和设备代码。

`__HGGC_ARCH__` 仅在设备代码中定义，即在 `__device__`、`__host__ __device__` 和 `__global__` 函数中定义。宏的值与 `hggc` 选项 `ppu_<version>` 相关，关系为 `__HGGC_ARCH__ = <version> * 10`。

示例：

```bash
hgcc --gpu-architecture ppu_10 test.hg
```

将 `__HGGC_ARCH__` 定义为 `100`。

* * *

`__HGGC_ARCH__` **约束**

**分离编译约束**

**1.** 在单独编译模式下，具有外部链接的函数或变量定义的存在或不存在不应依赖于 `__HGGC_ARCH__` 的定义或其值。

示例：

```cpp
#if !defined(__HGGC_ARCH__)
void hostFunction() { // 错误：hostFunction()的定义只在__HGGC_ARCH__未定义时存在
}
#endif
```

**2.** 在单独编译中，预处理器宏 `__HGGC_ARCH__` 不得在头文件中使用，以防止对象具有不同的行为。或者，所有对象必须为同一虚拟架构编译。如果在头文件中定义了一个弱函数或模板函数且其行为依赖于 `__HGGC_ARCH__`，则当这些对象为不同计算架构编译时，不同对象中的该函数实例可能会发生冲突。

例如，如果头文件 `a.h` 包含：

```cpp
template <typename T> __device__ T *get() {
#if __HGGC_ARCH__ == 100
  return nullptr;
#else
  __shared__ T arr[256];
  return arr;
#endif
}
```

然后如果 `a.hg` 和 `b.hg` 都包含 `a.h` 并对相同类型实例化 `get()`，而 `b.hg` 期望非 `NULL` 地址，并且用以下命令编译：

```bash
hgcc --gpu-architecture ppu_10 -dc a.hg
hgcc --gpu-architecture ppu_15 -dc b.hg
hgcc --gpu-architecture ppu_15 a.o b.o
```

链接时只使用一个版本的 `get()` 函数，因此行为取决于选择了哪个版本。为了避免这个问题，要么 `a.hg` 和 `b.hg` 必须为同一计算架构编译，要么不应该在共享头文件函数中使用 `__HGGC_ARCH__`。

**类型与实例化约束**

**3.** 以下实体的类型签名不应依赖于是否定义了 `__HGGC_ARCH__`，也不应依赖于其值。

* `__global__` 函数和函数模板。

* `__device__` 和 `__constant__` 变量。

* 纹理和表面。

示例：

```cpp
#if !defined(__HGGC_ARCH__)
typedef int type;
#else
typedef double type;
#endif
__device__ type var; // 错误：var的类型取决于__HGGC_ARCH__
// 错误：kernel的类型取决于__HGGC_ARCH__
__global__ void kernel(type x) {
  // ...
}
```

**4.** 如果一个 `__global__` 函数模板从主机实例化并启动，则无论是否定义了 `__HGGC_ARCH__` 或其值如何，都必须使用相同的模板参数进行实例化。

示例：

```cpp
__device__ int result;
template <typename T> __global__ void kernelFunction(T x) { result = x; }
__host__ __device__ void hostDeviceFunction(void) {
#if !defined(__HGGC_ARCH__)
  kernelFunction<<<1, 1>>>(1); // 错误："kernelFunction<int>" 实例化只在__HGGC_ARCH__未定义时！
#endif
}
int main(void) {
  hostDeviceFunction();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

编译器不能保证会为上述不支持的 `__HGGC_ARCH__` 用途生成诊断信息。

###### 5.2.1.1.2. HGGC 特性测试宏 {#52112-hggc-特性测试宏}

`hgcc`（HG C/C++ Compiler）提供以下预处理器宏用于特性测试。当 HGGC 前端编译器支持某个特定特性时，这些宏会被定义。

* `__HGGCCC_EXTENDED_LAMBDA__`: 支持 [扩展 lambda](#5172-扩展lambda)。通过 `--expt-extended-lambda` 或 `--extended-lambda` 标志启用。

* `__HGGCCC_RELAXED_CONSTEXPR__`: 支持 [宽松 constexpr 函数](#5193-constexpr-函数)。通过 `--expt-relaxed-constexpr` 标志启用。

##### 5.2.1.2. HGGC 特定函数 {#5212-hggc-特定函数}

###### 5.2.1.2.1. 地址空间谓词函数 {#52121-地址空间谓词函数}

地址空间谓词函数用于确定指针的地址空间。

```cpp
__device__ unsigned __isGlobal(const void *devPtr);
__device__ unsigned __isShared(const void *devPtr);
__device__ unsigned __isConstant(const void *devPtr);
__device__ unsigned __isGridConstant(const void *devPtr);
__device__ unsigned __isLocal(const void *devPtr);
```

如果 `ptr` 包含指定地址空间中对象的通用地址，则这些函数返回 `1`，否则返回 `0`。如果参数是 `NULL` 指针，则它们的行为未定义。

* `__isGlobal()`: 全局内存空间。

* `__isShared()`: 共享内存空间。

* `__isConstant()`: 常量内存空间。

* `__isGridConstant()`: 使用 `__grid_constant__` 注解的核函数参数。

* `__isLocal()`: 局部内存空间。

###### 5.2.1.2.2. 地址空间转换函数 {#52122-地址空间转换函数}

HGGC 指针 (`T*`) 可以访问对象，无论对象存储在哪里。例如，`int*` 可以访问位于全局或共享内存中的 `int` 对象。

地址空间转换函数用于在通用地址和特定地址空间中的地址之间进行转换。当编译器无法确定指针的地址空间时，这些函数很有用，例如，在跨翻译单元或与 TIX 指令交互时。

```cpp
// TIX: ppu.cvta.to.global
__device__ size_t __cvta_generic_to_global(const void *devPtr);
// TIX: ppu.cvta.to.shared
__device__ size_t __cvta_generic_to_shared(const void *devPtr);
// TIX: ppu.cvta.to.const
__device__ size_t __cvta_generic_to_constant(const void *devPtr);
// TIX: ppu.cvta.to.local
__device__ size_t __cvta_generic_to_local(const void *devPtr);
__device__ void *__cvta_global_to_generic(size_t rawbits); // TIX: ppu.cvta.global
__device__ void *__cvta_shared_to_generic(size_t rawbits); // TIX: ppu.cvta.shared
__device__ void *__cvta_constant_to_generic(size_t rawbits); // TIX: ppu.cvta.const
__device__ void *__cvta_local_to_generic(size_t rawbits);    // TIX: ppu.cvta.local
```

作为一个与 TIX 指令互操作的例子，`ppu.ld.shared.s32 r0, [ptr];` TIX 指令期望 `ptr` 引用共享内存地址空间。一个指向 `__shared__` 内存中对象的 `int*` 指针的 HGGC 程序需要在将此指针传递给 TIX 指令之前通过调用 `__cvta_generic_to_shared` 将其转换为共享地址空间，如下所示：

```cpp
  __shared__ int var;
  var = 121;
  size_t ptr = __cvta_generic_to_shared(&var);
  int res;
  asm volatile("ppu.ld.shared.s32 %0, [%1];"
               : "=r"(res)
               : "l"(ptr)
               : "memory");
  assert(res == 121);
```

利用这些地址表示形式的一种常见优化是减少数据结构大小，利用共享、局部和常量空间的地址范围小于 32 位，存储 32 位地址而不是 64 位指针并节省寄存器。此外，32 位算术比 64 位算术更快。要获得这些地址的 32 位整数表示，需要将 64 位值截断为 32 位，即将无符号 64 位整数强制转换为无符号 32 位整数：

```cpp
  __shared__ int var;
  uint32_t ptr32 = static_cast<uint32_t>(__cvta_generic_to_shared(&var));
```

要从此类 32 位表示恢复通用地址，请将地址零扩展回无符号 64 位整数，然后调用相应的地址空间转换函数：

```cpp
  size_t ptr64 = static_cast<size_t>(ptr32); // zero-extend to 64 bits
  void *ptrGeneric = __cvta_shared_to_generic(ptr64);
  assert(ptrGeneric == &var);
```

* * *

###### 5.2.1.2.3. 低级加载和存储函数 {#52123-低级加载和存储函数}

```cpp
T __ldg(const T *address);
```

函数 `__ldg()` 执行只读 L1/Tex 缓存加载。它支持所有 C++ 基本类型、HGGC 向量类型 (3 元素除外) 和扩展浮点类型，如 `__half`、`__half2`、`__ppu_bfloat16` 和 `__ppu_bfloat162`。

* * *

```cpp
T __ldcg(const T *address);
T __ldca(const T *address);
T __ldcs(const T *address);
T __ldlu(const T *address);
T __ldcv(const T *address);
```

这些函数使用 [TIX ISA](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#491-缓存运算符) 指南中指定的缓存操作符执行加载。它们支持所有 C++ 基本类型、HGGC 向量类型 (3 元素除外) 和扩展浮点类型，如 `__half`、`__half2`、`__ppu_bfloat16` 和 `__ppu_bfloat162`。

* * *

```cpp
void __stwb(T *address, T value);
void __stcg(T *address, T value);
void __stcs(T *address, T value);
void __stwt(T *address, T value);
```

这些函数使用 [TIX ISA](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#491-缓存运算符) 指南中指定的缓存操作符执行存储。它们支持所有 C++ 基本类型、HGGC 向量类型 (3 元素除外) 和扩展浮点类型，如 `__half`、`__half2`、`__ppu_bfloat16` 和 `__ppu_bfloat162`。

###### 5.2.1.2.4. `__trap()` {#52124-trap}

可以从任何设备线程通过调用 `__trap()` 函数启动陷阱操作。

```cpp
void __trap();
```

核函数执行被中止，并在主机程序中引发中断。调用 `__trap()` 会导致 HGGC 上下文损坏，导致后续 HGGC 调用和核函数调用失败。

###### 5.2.1.2.5. `__nanosleep()` {#52125-nanosleep}

```cpp
__device__ void __nanosleep(unsigned ns);
```

函数 `__nanosleep(ns)` 暂停线程大约 `ns` 纳秒的睡眠持续时间。最大睡眠持续时间约为一毫秒。

示例：

以下代码实现了一个具有指数退避的互斥锁。

```cpp
__device__ void mutex_lock(unsigned *mutex) {
  unsigned ns = 8;
  while (atomicCAS(mutex, 0, 1) == 1) {
    __nanosleep(ns);
    if (ns < 256) {
      ns *= 2;
    }
  }
}
__device__ void mutex_unlock(unsigned *mutex) { atomicExch(mutex, 0); }
```

###### 5.2.1.2.6. 动态规划扩展 (DPX) 指令 {#52126-动态规划扩展dpx指令}

DPX 函数集启用了查找最小值和最大值，以及最多三个 16 位或 32 位有符号或无符号整数参数的融合加法和最小/最大运算。有一个可选的 ReLU 功能，即限制到零。

比较函数：

* 三个参数。语义：`max(a, b, c)`，`min(a, b, c)`。

```cpp
int __vimax3_s32(int, int, int);
unsigned __vimax3_s16x2(unsigned, unsigned, unsigned);
unsigned __vimax3_u32(unsigned, unsigned, unsigned);
unsigned __vimax3_u16x2(unsigned, unsigned, unsigned);
int __vimin3_s32(int, int, int);
unsigned __vimin3_s16x2(unsigned, unsigned, unsigned);
unsigned __vimin3_u32(unsigned, unsigned, unsigned);
unsigned __vimin3_u16x2(unsigned, unsigned, unsigned);
```

* 两个参数，带 ReLU。语义：`max(a, b, 0)`，`max(min(a, b), 0)`。

```cpp
int __vimax_s32_relu(int, int);
unsigned __vimax_s16x2_relu(unsigned, unsigned);
int __vimin_s32_relu(int, int);
unsigned __vimin_s16x2_relu(unsigned, unsigned);
```

* 三个参数，带 ReLU。语义：`max(a, b, c, 0)`，`max(min(a, b, c), 0)`。

```cpp
int __vimax3_s32_relu(int, int, int);
unsigned __vimax3_s16x2_relu(unsigned, unsigned, unsigned);
int __vimin3_s32_relu(int, int, int);
unsigned __vimin3_s16x2_relu(unsigned, unsigned, unsigned);
```

* 两个参数，额外返回哪个参数较小/较大：

```cpp
int __vibmax_s32(int, int, bool *const pred);
unsigned __vibmax_u32(unsigned, unsigned, bool *const pred);
unsigned __vibmax_s16x2(unsigned, unsigned, bool *const pred_hi,
                        bool *const pred_lo);
unsigned __vibmax_u16x2(unsigned, unsigned, bool *const pred_hi,
                        bool *const pred_lo);
int __vibmin_s32(int, int, bool *const pred);
unsigned __vibmin_u32(unsigned, unsigned, bool *const pred);
unsigned __vibmin_s16x2(unsigned, unsigned, bool *const pred_hi,
                        bool *const pred_lo);
unsigned __vibmin_u16x2(unsigned, unsigned, bool *const pred_hi,
                        bool *const pred_lo);
```

融合加法和最小/最大：

* 三个参数，将 (第一个+第二个) 与第三个进行比较。语义：`max(a + b, c)`，`min(a + b, c)`

```cpp
int __viaddmax_s32(int, int, int);
unsigned __viaddmax_s16x2(unsigned, unsigned, unsigned);
unsigned __viaddmax_u32(unsigned, unsigned, unsigned);
unsigned __viaddmax_u16x2(unsigned, unsigned, unsigned);
int __viaddmin_s32(int, int, int);
unsigned __viaddmin_s16x2(unsigned, unsigned, unsigned);
unsigned __viaddmin_u32(unsigned, unsigned, unsigned);
unsigned __viaddmin_u16x2(unsigned, unsigned, unsigned);
```

* 三个参数，带 ReLU，将 (第一个+第二个) 与第三个和零进行比较。语义：`max(a + b, c, 0)`，`max(min(a + b, c), 0)`

```cpp
int __viaddmax_s32_relu(int, int, int);
unsigned __viaddmax_s16x2_relu(unsigned, unsigned, unsigned);
int __viaddmin_s32_relu(int, int, int);
unsigned __viaddmin_s16x2_relu(unsigned, unsigned, unsigned);
```

这些指令根据计算能力进行硬件加速或软件模拟。有关计算能力要求，请参见算术指令部分。

完整 API 可以在 [HGGC 数学 API 文档](/docs_center/doc_detail/index.html?projectId=39&chapterId=1048#9-simd-内建函数) 中找到。

* * *

DPX 是实行动态规划算法的非常有用的工具，例如基因组学中的 Smith-Waterman 和 Needleman-Wunsch 算法以及路线优化中的 Floyd-Warshall 算法。

三个有符号 32 位整数的最大值，带 ReLU：

```cpp
  int a = -1;
  int b = 2;
  int c = 3;
  int res0 = __vimax3_s32_relu(a, b, c); // max(-1, 2, 3, 0) = 3
  int d = -2;
  int e = -3;
  int res1 = __vimax3_s32_relu(a, d, e); // max(-1, -2, -3, 0) = 0
```

两个 32 位有符号整数之和、另一个 32 位有符号整数和零 (ReLU) 的最小值：

```cpp
  int a = -5;
  int b = 6;
  int c = -2;
  int res0 = __viaddmax_s32_relu(a, b, c); // max(-5 + 6, -2, 0) = 1
  int d = 4;
  int res1 = __viaddmax_s32_relu(a, d, c); // max(-5 + 4, -2, 0) = 0
```

两个无符号 32 位整数的最小值并确定哪个值较小：

```cpp
  unsigned a = 9;
  unsigned b = 6;
  bool smaller;
  unsigned res = __vibmin_u32(a, b, &smaller); // res = 6, smaller = true
```

三对无符号 16 位整数的最大值：

```cpp
  unsigned a = 0x00050002;
  unsigned b = 0x00030004;
  unsigned c = 0x00010006;
  unsigned res = __vimax3_u16x2(a, b, c); // max(5, 3, 1) 和 max(2, 4, 6) 拼接得到 0x00050006
```

##### 5.2.1.3. 编译器优化提示 {#5213-编译器优化提示}

编译器优化提示用附加信息装饰代码，以帮助编译器优化生成的代码。

* 内建函数在设备代码中始终可用。

* 主机代码支持取决于主机编译器。

###### 5.2.1.3.1. `#pragma unroll` {#52131-pragma-unroll}

编译器默认展开具有已知循环次数的小循环。但是，可以使用 `#pragma unroll` 指令来控制任何给定循环的展开。此指令必须放置在循环之前，并且仅适用于该循环。

可选地跟随一个整数常量表达式。以下是整数常量表达式的情况：

* 如果省略，则如果其循环次数是常量，则循环将完全展开。

* 如果它计算为 `0` 或 `1`，则不会展开循环。

* 如果它是非正整数或大于 `INT_MAX`，则将忽略该 pragma，并发出报错。

示例：

```cpp
struct S {
  static constexpr int value = 4;
};
inline constexpr int count = 4;
__device__ void foo(int *output, int *input) {
// 未指定参数，循环将完全展开
#pragma unroll
  for (int i = 0; i < 12; ++i)
    output[i] += input[i] * 2;
// 展开值 = 5
#pragma unroll(count + 1)
  for (int i = 0; i < 12; ++i)
    output[i] += input[i] * 4;
// 展开值 = 1，循环展开禁用
#pragma unroll 1
  for (int i = 0; i < 12; ++i)
    output[i] += input[i] * 8;
// 展开值 = 4
#pragma unroll(S::value)
  for (int i = 0; i < 12; ++i)
    output[i] += input[i] * 16;
// 负值，报错
// #pragma unroll - 1
  for (int i = 0; i < 12; ++i)
    output[i] += input[i] * 2;
}
```

###### 5.2.1.3.2. `__builtin_assume_aligned()` {#52132-builtin_assume_aligned}

```cpp
void *__builtin_assume_aligned(const void *ptr, size_t align);
void *__builtin_assume_aligned(const void *ptr, size_t align,
                               <integral type> offset);
```

此内建函数使编译器能够假设返回的指针至少 `align` 字节对齐。

* 三参数版本使编译器能够假设 `(char*) ptr - offset` 至少 `align` 字节对齐。

`align` 必须是 2 的幂且为整数字面量。

示例：

```cpp
// 编译器可以假设'res1'至少是32字节对齐的
  void *res1 = __builtin_assume_aligned(ptr, 32);
  // 编译器可以假设'res2 = (char*) ptr - 8'至少是32字节对齐的
  void *res2 = __builtin_assume_aligned(ptr, 32, 8);
```

###### 5.2.1.3.3. `__builtin_assume()` {#52133-builtin_assume}

```cpp
void __builtin_assume(bool predicate);
```

该内建函数使编译器能够假设布尔参数为真。如果参数在运行时为假，则行为未定义。请注意，如果参数有副作用，则行为不确定。

示例：

```cpp
__device__ bool is_greater_than_zero(int input) { return input > 0; }
__device__ bool f(int input) {
  __builtin_assume(input > 0);
  return is_greater_than_zero(input); // 返回true，而不评估条件
}
```

###### 5.2.1.3.4. `__builtin_expect()` {#52134-builtin_expect}

```cpp
long __builtin_expect(long input, long expected);
```

此内建函数告诉编译器 `input` 预期等于 `expected`，并返回 `input` 的值。通常用于向编译器提供分支预测信息。它的行为类似于 C++20 的 `[[likely]]` 和 `[[unlikely]]` [属性](https://en.cppreference.com/w/cpp/language/attributes/likely)。

示例：

```cpp
  // 向编译器指示可能"var == 0"
  if (__builtin_expect(var, 0))
    // ...
```

###### 5.2.1.3.5. `__builtin_unreachable()` {#52135-builtin_unreachable}

```cpp
void __builtin_unreachable(void);
```

此内建函数告诉编译器控制流永远不会到达调用函数的位置。如果控制流确实在此点运行时到达，则程序具有未定义的行为。

此函数对于避免不可达分支的代码生成以及禁用不可达代码的编译器警告很有用。

示例：

```cpp
  // 向编译器指示默认情况标签不可能到达。
  switch (in) {
  case 1:
    return 2;
  case 2:
    return 3;
  default:
    __builtin_unreachable();
  }
```

###### 5.2.1.3.6. `__builtin_ppu_assume_uniform()` {#52136-builtin_ppu_assume_uniform}

```cpp
void __builtin_ppu_assume_uniform(T val);
```

该内建函数使编译器能够假设所有线程到该处时 `val` 的值相同。如果不同，则行为未定义。

示例：

```cpp
__global__ void kernel(int *a) {
  int i = threadIdx.x;
  int val = a[i];
  __builtin_ppu_assume_uniform(val); // a数组中所有访问到的元素相同。
}
```

###### 5.2.1.3.7. `__builtin_ppu_to_uniform_b32()` {#52137-builtin_ppu_to_uniform_b32}

```cpp
int __builtin_ppu_to_uniform_b32(T val);
```

该内建函数使编译器能够假设所有线程到该处时 `val` 的值相同，并返回该值转换成的 `int`。如果不同，则行为未定义。

示例：

```cpp
__global__ void kernel(int *a) {
  int i = threadIdx.x;
  int val = a[i];
  val = __builtin_ppu_to_uniform_b32(val); // a数组中所有访问到的元素相同。
}
```

###### 5.2.1.3.8. `__builtin_ppu_to_uniform_b64()` {#52138-builtin_ppu_to_uniform_b64}

```cpp
long int __builtin_ppu_to_uniform_b64(T val);
```

该内建函数使编译器能够假设所有线程到该处时 `val` 的值相同，并返回该值转换成的 `long int`。如果不同，则行为未定义。

示例：

```cpp
__global__ void kernel(long int *a) {
  int i = threadIdx.x;
  long int val = a[i];
  val = __builtin_ppu_to_uniform_b64(val); // a数组中所有访问到的元素相同。
}
```

#### 5.2.2. 函数和变量注解 {#522-函数和变量注解}

##### 5.2.2.1. 执行空间说明符 {#5221-执行空间说明符}

执行空间说明符 `__host__`、`__device__` 和 `__global__` 指示函数是在主机还是设备上执行。

**表 1：执行空间说明符**

| 执行空间说明符           | 在主机上执行 | 在设备上执行 | 从主机上调用 | 从设备上调用 | 附加约束                                                                                                                                                                              |
| ------------------------ | ------------ | ------------ | ------------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `__host__`（或无说明符） | ✅            | ❌            | ✅            | ❌            | —                                                                                                                                                                                     |
| `__device__`             | ❌            | ✅            | ❌            | ✅            | —                                                                                                                                                                                     |
| `__global__`             | ❌            | ✅            | ✅            | ✅            | 返回类型须为 `void`；须为自由函数；调用时须提供执行配置（见 [核函数配置](#524-核函数配置)）；异步执行；不支持递归；参数有额外限制（见 [__global__ 函数参数](#51854-global-函数参数)） |
| `__host__ __device__`    | ✅            | ✅            | ✅            | ✅            | 分别为主机和设备编译                                                                                                                                                                  |

* * *

使用 `__host__ __device__` 声明的函数会为主机和设备分别编译。可以使用 `__HGGC_ARCH__` 宏来区分主机和设备代码路径：

```cpp
__host__ __device__ void f() {
#if defined(__HGGC_ARCH__)
  // 设备代码路径
#else
  // 主机代码路径
#endif
}
```

##### 5.2.2.2. 内存空间说明符 {#5222-内存空间说明符}

内存空间说明符 `__device__`、`__managed__`、`__constant__` 和 `__shared__` 指示设备上变量的存储位置。

下表总结了内存空间属性：

**表 2：内存空间说明符**

| 内存空间说明符 | 位置                 | 可使用处                  | 生命周期 | 唯一实例 |
| -------------- | -------------------- | ------------------------- | -------- | -------- |
| `__device__`   | 设备全局内存         | 设备线程/ HGGC 运行时 API | 程序     | 每个设备 |
| `__constant__` | 设备常量内存         | 设备线程/ HGGC 运行时 API | 程序     | 每个设备 |
| `__managed__`  | 主机和设备（自动）   | 主机/设备线程             | 程序     | 每个程序 |
| `__shared__`   | 设备（流式多处理器） | 块线程                    | 块       | 块       |
| 无说明符       | 设备（寄存器）       | 单个线程                  | 单个线程 | 单个线程 |

* * *

主机代码无法直接解引用 `__device__` 或 `__constant__` 变量的地址，但 HGGC 运行时提供了一组符号操作 API 来完成主机与这些变量之间的数据交换：

| API                      | 功能                     |
| ------------------------ | ------------------------ |
| `hggcGetSymbolAddress()` | 获取设备符号的设备端地址 |
| `hggcGetSymbolSize()`    | 查询设备符号占用的字节数 |
| `hggcMemcpyToSymbol()`   | 从主机向设备符号写入数据 |
| `hggcMemcpyFromSymbol()` | 从设备符号向主机读取数据 |

其中 `__constant__` 变量在设备代码中仅可读取，写入操作只能通过上述主机端 API 完成。

以下示例演示了符号操作 API 的典型用法：

```cpp
__device__ float d = 1.0f;   // 设备内存中的变量
__constant__ float c = 1.0f; // 常量内存中的变量
// 为了可读性，以下示例重点介绍设备变量。
int main() {
  float *ptr;
  // 获取d的地址
  hggcGetSymbolAddress((void **)&ptr, d);

  size_t size;
  hggcGetSymbolSize(&size, d); // 检索符号的大小（4字节）。

  float h;
  // 从设备复制到主机。
  hggcMemcpyFromSymbol(&h, d, sizeof(h));

  h = 3.0f;
  // 从主机复制到设备。
  hggcMemcpyToSymbol(d, &h, sizeof(h));
}
```

###### 5.2.2.2.1. `__shared__` 内存 {#52221-shared-内存}

`__shared__` 内存变量可以有静态大小（在编译时确定）或动态大小（在核函数启动时确定）。有关在运行时指定共享内存大小的详细信息，请参阅 [核函数配置](#524-核函数配置) 部分。

共享内存约束：

* 动态大小的变量必须声明为外部数组或指针。

* 静态大小的变量不能在其声明中初始化。

以下示例演示如何声明和设置 `__shared__` 变量的大小：

```cpp
extern __shared__ char ptr[];
// extern __shared__ char* ptr; 替代语法

__global__ void kernel() { // 或 __device__ 函数
  __shared__ int var1[3];  // 静态大小
  auto var2 = (int *)ptr;  // 动态大小
}

int main() {
  size_t size = 16;
  kernel<<<1, 1, size>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
}
```

###### 5.2.2.2.2. `__managed__` 内存 {#52222-managed-内存}

`__managed__` 变量具有以下限制：

* `__managed__` 变量的地址不是常量表达式。

* `__managed__` 变量不应具有引用类型 `T&`。

* 当 HGGC 运行时可能处于无效状态时，不得使用 `__managed__` 变量的地址或值，包括以下情况：

  * 具有 `static` 或 `thread_local` 存储持续时间的对象的静态/动态初始化或销毁期间。

  * 在调用 `exit()` 后执行的代码中。例如，标记为 `__attribute__((destructor))` 的函数。

  * 在 HGGC 运行时尚未初始化时执行的代码中。例如，标记为 `__attribute__((constructor))` 的函数。

* `__managed__` 变量的一致性和一致性行为，与动态分配的 managed 内存相同。

* 另请参见 [局部变量](#51841-局部变量) 的限制。

以下是 `__managed__` 变量的合法和非法使用的示例：

```cpp
#include <cassert>
__device__ __managed__ int var = 1; // OK
int *ptr = &var; // 错误：在静态初始化中使用托管变量
struct S1 {
  int field;
  S1() : field(var){};
};
struct S2 {
  ~S2() { var = 1; }
};
S1 s1; // 错误：在动态初始化中使用托管变量
S2 s2; // 错误：在具有静态存储持续时间的对象的析构函数中使用托管变量
__device__ __managed__ const int c = 1; // 错误：const限定类型
__device__ __managed__ int &ref = var;  // 错误：引用类型
template <int *Addr> struct S3 {};
S3<&var> s; // 错误：托管变量的地址不是常量表达式
__global__ void kernel(int *ptr) {
  assert(ptr == &var); // OK
  var = 20;            // OK
}
int main() {
  int *ptr = &var; // OK
  kernel<<<1, 1>>>(ptr);
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  var++; // OK
}
```

##### 5.2.2.3. 内联说明符 {#5223-内联说明符}

以下说明符可用于控制 `__host__` 和 `__device__` 函数的内联：

* `__noinline__`: 指示 `hgcc` 不要内联该函数。

* `__forceinline__`: 强制 `hgcc` 在单个翻译单元内内联该函数。

* `__inline_hint__`: 在使用链接时优化（LTO）时启用跨翻译单元的激进内联。

这些说明符是互斥的。

##### 5.2.2.4. `__restrict__` 指针 {#5224-restrict-指针}

`hgcc` 通过 `__restrict__` 关键字支持受限指针。

指针别名发生在两个或多个指针指向重叠内存区域时。这可能会阻碍诸如代码重排和公共子表达式消除等优化。

受限制的指针是指程序员的承诺，在指针的生命周期内，它所指向的内存只会通过该指针访问。这允许编译器执行更激进的优化（当满足以下条件之一时）。

* 所有访问设备函数的线程都只从该指针中读取数据；

* 或者至多一个线程写入它，并且没有其他线程从中读取数据。

以下示例说明了别名问题，并演示了如何使用受限指针帮助编译器减少指令数量：

```cpp
__device__ void deviceFunction(const int *a, const int *b, int *c) {
  c[0] = a[0] + b[1];
  c[1] = a[0] + b[1] + a[2];
  c[2] = a[1] * a[2];
  c[3] = b[1] - a[1];
  c[4] = a[0] + b[1] + a[1];
  // ...
}
```

由于指针 `a`、`b` 和 `c` 可能被别名化，任何通过 `c` 的写入都可能修改 `a` 或 `b` 的元素。例如，`a[0] + b[1]` 这个子表达式在 `c[0]`、`c[1]`、`c[4]` 三处被重复计算，但编译器无法将其结果缓存并复用——因为如果 `c[0]` 与 `a[0]` 指向同一位置，对 `c[0]` 的写入就会改变后续 `a[0]` 的值。同样，编译器无法将 `c[4]` 的计算重新排列到 `c[0]` 之后立即执行，因为中间对 `c[1]`～`c[3]` 的写入都可能影响 `a` 或 `b` 的读取结果。

通过将 `a`、`b` 和 `c` 声明为受限指针，程序员告知编译器指针不会被别名化。这意味着写入 `c` 永远不会覆盖 `a` 或 `b` 的元素。这改变了函数原型如下：

```cpp
__device__ void deviceFunction(const int *__restrict__ a,
                               const int *__restrict__ b,
                               int *__restrict__ c);
```

请注意，所有指针参数都必须受到限制，编译器优化器才能生效。添加 `__restrict__` 关键字后，编译器可以随意重新排序和执行公共子表达式消除，同时保持与抽象执行模型相同的功能。

```cpp
__device__ void deviceFunction(const int *__restrict__ a,
                               const int *__restrict__ b,
                               int *__restrict__ c) {
  int t0 = a[0];
  int t1 = b[1];
  int t2 = a[1];
  int t3 = a[2];
  int t4 = t0 + t1;
  c[0] = t4;
  c[1] = t4 + t3;
  c[2] = t2 * t3;
  c[3] = t1 - t2;
  c[4] = t4 + t2;
  // ...
}
```

结果是减少了内存访问和计算次数，平衡了缓存加载和寄存器中公共子表达式的压力。

由于寄存器压力是许多 HGGC 代码的关键问题，使用受限指针可能会因降低占用率而对性能产生负面影响。

* * *

`__global__` 函数 `const` 指针中标记为 `__restrict__` 的访问被编译为只读缓存加载，类似于 [TIX](/docs_center/doc_detail/index.html?projectId=39&chapterId=1042#ppu-ld) `ppu.ld.global.nc` 或 `__ldg()` 低级加载和存储函数指令。

```cpp
__global__ void kernel1(const int *in, int *out) {
  *out = *in; // TIX: ppu.ld.global
}
__global__ void kernel2(const int *__restrict__ in, int *out) {
  *out = *in; // TIX: ppu.ld.global.nc
}
```

##### 5.2.2.5. `__grid_constant__` 参数 {#5225-grid_constant-参数}

使用 `__grid_constant__` 注解 `__global__` 函数参数可防止编译器创建参数的每个线程副本。相反，网格中的所有线程将通过单个地址访问参数，这可以提高性能。

`__grid_constant__` 参数具有以下属性：

* 它具有核函数的生命周期。

* 它是单个核函数私有的，也就是说对象不可被来自其他网格（包括子网格）的线程访问。

* 核函数中的所有线程看到相同的地址。

* 它是只读的。修改 `__grid_constant__` 对象或其任何子对象（包括 `mutable` 成员）都是未定义行为。

要求：

* 标记为 `__grid_constant__` 的核函数参数必须是具有 `const` 限定的非引用类型。

* 所有同一函数声明必须对于 `__grid_constant__` 参数保持一致。

* 函数模板特化必须与主模板声明对于 `__grid_constant__` 参数保持一致。

* 函数模板实例化也必须与主模板声明对于 `__grid_constant__` 参数保持一致。

示例：

```cpp
struct S {
  int x;
  mutable int y;
};
__device__ void externalFunction(const S &) {}
__global__ void kernel(const __grid_constant__ S s) {
  s.x++; // 编译错误：试图修改只读内存
  // 编译器将不会创建"s"的每个线程本地副本：
  externalFunction(s);
}
```

##### 5.2.2.6. 注解摘要 {#5226-注解摘要}

下表总结了 HGGC 注解，并给出了每种注解适用于哪个执行空间以及在哪里有效。

**表 3：注解总结**

| 注解                                                 | `__host__` / `__device__` / `__host__ __device__` | `__global__` |
| ---------------------------------------------------- | ------------------------------------------------- | ------------ |
| `__noinline__`, `__forceinline__`, `__inline_hint__` | 函数                                              | ❌            |
| `__restrict__`                                       | 指针参数                                          | 指针参数     |
| `__grid_constant__`                                  | ❌                                                 | 参数         |
| `__launch_bounds__`                                  | ❌                                                 | 函数         |
| `__maxnreg__`                                        | ❌                                                 | 函数         |

#### 5.2.3. 内置类型和变量 {#523-内置类型和变量}

##### 5.2.3.1. 主机编译器类型扩展 {#5231-主机编译器类型扩展}

只要主机编译器支持，HGGC 允许使用非标准算术类型。支持以下类型：

* 128 位整数类型 `__int128`。

  * 当主机编译器定义了 `__SIZEOF_INT128__` 宏时支持。

* [`_Complex` 类型](https://www.gnu.org/software/c-intro-and-ref/manual/html_node/Complex-Data-Types.html) 仅在主机代码中支持。

##### 5.2.3.2. 内置变量 {#5232-内置变量}

用于指定和检索网格和块沿 x、y 和 z 维度的核函数配置的值是 `dim3` 类型。用于获取块和线程索引的变量是 `uint3` 类型。`dim3` 和 `uint3` 都是由名为 `x`、`y` 和 `z` 的三个无符号值组成的简单结构。在 C++11 及以后版本中，`dim3` 的所有元素的默认值为 1。

仅限设备的内置变量：

* `dim3 gridDim`: 包含网格的尺寸，即沿 x、y 和 z 维度的线程块数量。

* `dim3 blockDim`: 包含线程块的尺寸，即沿 x、y 和 z 维度的线程数量。

* `uint3 blockIdx`: 包含网格内的块索引，沿 x、y 和 z 维度。

* `uint3 threadIdx`: 包含块内的线程索引，沿 x、y 和 z 维度。

* `int warpSize` : 运行时定义的值，表示线程束中的线程数，通常为 `32`。请参见 [线程束和 SIMT](#132-线程束warps和simt) 中对线程束的定义。

##### 5.2.3.3. 内置类型 {#5233-内置类型}

HGGC 提供从基本整数和浮点类型派生的向量类型，这些类型同时支持主机和设备。下表显示了可用的向量类型。

**表 4：向量类型**

| C++ 基本类型         | 向量 X1      | 向量 X2      | 向量 X3      | 向量 X4                                    |
| -------------------- | ------------ | ------------ | ------------ | ------------------------------------------ |
| `signed char`        | `char1`      | `char2`      | `char3`      | `char4`                                    |
| `unsigned char`      | `uchar1`     | `uchar2`     | `uchar3`     | `uchar4`                                   |
| `signed short`       | `short1`     | `short2`     | `short3`     | `short4`                                   |
| `unsigned short`     | `ushort1`    | `ushort2`    | `ushort3`    | `ushort4`                                  |
| `signed int`         | `int1`       | `int2`       | `int3`       | `int4`                                     |
| `unsigned`           | `uint1`      | `uint2`      | `uint3`      | `uint4`                                    |
| `signed long`        | `long1`      | `long2`      | `long3`      | `long4/long4_16a/long4_32a`                |
| `unsigned long`      | `ulong1`     | `ulong2`     | `ulong3`     | `ulong4/ulong4_16a/ulong4_32a`             |
| `signed long long`   | `longlong1`  | `longlong2`  | `longlong3`  | `longlong4/longlong4_16a/longlong4_32a`    |
| `unsigned long long` | `ulonglong1` | `ulonglong2` | `ulonglong3` | `ulonglong4/ulonglong4_16a/ulonglong4_32a` |
| `float`              | `float1`     | `float2`     | `float3`     | `float4`                                   |
| `double`             | `double1`    | `double2`    | `double3`    | `double4/double4_16a/double4_32a`          |

* * *

下表详细说明了向量类型的字节大小和对齐要求。

**表 5：对齐要求**

| 类型                              | 大小 | 对齐 |
| --------------------------------- | ---- | ---- |
| `char1`, `uchar1`                 | 1    | 1    |
| `char2`, `uchar2`                 | 2    | 2    |
| `char3`, `uchar3`                 | 3    | 1    |
| `char4`, `uchar4`                 | 4    | 4    |
| `short1`, `ushort1`               | 2    | 2    |
| `short2`, `ushort2`               | 4    | 4    |
| `short3`, `ushort3`               | 6    | 2    |
| `short4`, `ushort4`               | 8    | 8    |
| `int1`, `uint1`                   | 4    | 4    |
| `int2`, `uint2`                   | 8    | 8    |
| `int3`, `uint3`                   | 12   | 4    |
| `int4`, `uint4`                   | 16   | 16   |
| `long1`, `ulong1`                 | 8    | 8    |
| `long2`, `ulong2`                 | 16   | 16   |
| `long3`, `ulong3`                 | 24   | 8    |
| `long4`, `ulong4`                 | 32   | 16   |
| `long4_16a`, `ulong4_16a`         | 32   | 16   |
| `long4_32a`, `ulong4_32a`         | 32   | 32   |
| `longlong1`, `ulonglong1`         | 8    | 8    |
| `longlong2`, `ulonglong2`         | 16   | 16   |
| `longlong3`, `ulonglong3`         | 24   | 8    |
| `longlong4`, `ulonglong4`         | 32   | 16   |
| `longlong4_16a`, `ulonglong4_16a` | 32   | 16   |
| `longlong4_32a`, `ulonglong4_32a` | 32   | 32   |
| `float1`                          | 4    | 4    |
| `float2`                          | 8    | 8    |
| `float3`                          | 12   | 4    |
| `float4`                          | 16   | 16   |
| `double1`                         | 8    | 8    |
| `double2`                         | 16   | 16   |
| `double3`                         | 24   | 8    |
| `double4`                         | 32   | 16   |
| `double4_16a`                     | 32   | 16   |
| `double4_32a`                     | 32   | 32   |

* * *

向量类型是结构体。它们的第一个、第二个、第三个和第四个组件分别可以通过 `x`、`y`、`z` 和 `w` 字段访问。

```cpp
int sum(int4 var) { return var.x + var.y + var.z + var.w; }
```

它们都有一个形式为 `make_<type_name>()` 的工厂函数；例如：

```cpp
int4 inc(int4 var) {
  return make_int4(var.x + 1, var.y + 1, var.z + 1, var.w + 1);
}
```

如果主机代码不是用 `hgcc` 编译的，则可以通过包含 SAIL 工具包中提供的 `hggc_runtime.h` 头文件导入向量类型和相关函数。

#### 5.2.4. 核函数配置 {#524-核函数配置}

任何对 `__global__` 函数的调用都必须为此调用指定一个"执行配置"。此执行配置定义了用于在设备上执行函数的网格和块的维度，以及相关的流。

执行配置通过在函数名和带括号的参数列表之间插入形式为 `<<<grid_dim, block_dim, dynamic_smem_bytes, stream>>>` 的表达式来指定，其中：

* `grid_dim` 是类型为 dim3 的变量，指定网格的维度和大小，`grid_dim.x * grid_dim.y * grid_dim.z` 等于启动的块数；

* `block_dim` 是类型为 dim3 的变量，指定每个块的维度和大小，`block_dim.x * block_dim.y * block_dim.z` 等于每个块中的线程数；

* `dynamic_smem_bytes` 是一个可选的 `size_t` 参数，默认值为零。它指定此次调用中除了静态分配的内存之外，每个块动态分配的共享内存字节数。该内存被 `extern __shared__` 数组使用（参见 [`__shared__` 内存](#52221-shared-内存)）。

* `stream` 是类型为 `hggcStream_t`（指针）的变量，指定关联的流。`stream` 是一个可选参数，默认值为 `NULL`。

以下示例显示了一个核函数定义和调用：

```cpp
__global__ void kernel(float *parame) {}
int main() {
  kernel<<<1, 1, 32>>>(nullptr);
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
}
```

执行配置的参数在实际函数参数之前求值。

如果 `grid_dim` 或 `block_dim` 超过了设备允许的最大尺寸（如计算能力部分所指定），或者如果 `dynamic_smem_bytes` 大于在静态分配内存后可用的共享内存，则函数调用失败。

##### 5.2.4.1. 启动界限 {#5241-启动界限}

正如 [核函数启动和占用率](#226-核函数启动和占用率) 部分所述，使用较少的寄存器可以让更多线程和线程块驻留在多处理器上，这会提高性能。

因此，编译器使用启发式方法来最小化寄存器使用，同时使 [寄存器溢出](#2233-寄存器) 和指令数量保持在最低水平。应用程序可以选择性地通过向编译器提供附加信息来帮助这些启发式方法，这些信息以启动界限的形式出现，这些界限在 `__global__` 函数定义中使用 `__launch_bounds__()` 限定符指定：

```cpp
__global__ void __launch_bounds__(maxThreadsPerBlock,
                                  minBlocksPerMultiprocessor) kernel() {
  //   ...
}
```

* `maxThreadsPerBlock` 指定应用程序启动 `kernel()` 时使用的每块最大线程数；它编译为 `.maxntid` TIX 指令。

* `minBlocksPerMultiprocessor` 是可选的，指定所需的每个多处理器上的最少常驻块数；它编译为 `.minnctapersm` TIX 指令。

如果指定了启动界限，编译器首先推导出核函数应使用的寄存器数量上限 `L`。这确保了 `minBlocksPerMultiprocessor` 块数（或单个块，如果未指定 `minBlocksPerMultiprocessor`）的 `maxThreadsPerBlock` 线程可以驻留在多处理器上。有关核函数使用的寄存器数量与每块分配的寄存器数量之间的关系，请参阅 [占用率](#226-核函数启动和占用率) 部分。然后编译器优化寄存器的使用如下：

* 如果初始寄存器使用量超过 `L`，编译器将其减少直到小于或等于 `L`。这通常会导致局部内存使用增加和/或指令数量增加。

* 如果初始寄存器使用量低于 `L`

  * 如果指定了 `maxThreadsPerBlock` 但没有指定 `minBlocksPerMultiprocessor`，编译器使用 `maxThreadsPerBlock` 来确定 `n` 和 `n + 1` 常驻块之间的转换的寄存器使用阈值。当使用少一个寄存器能腾出空间容纳额外的常驻块时就会发生这种情况。然后，编译器应用类似于未指定启动界限时的启发式方法。

  * 如果同时指定了 `minBlocksPerMultiprocessor` 和 `maxThreadsPerBlock`，编译器可能会增加寄存器使用量至 `L`，以减少指令数量并更好地隐藏单线程指令的延迟。

如果使用以下方式执行核函数，核函数将启动失败：

* 每块线程数超过其启动界限 `maxThreadsPerBlock`。

HGGC 核函数所需的每线程资源可能以不佳方式限制最大块大小。为了与未来硬件和工具包保持前向兼容性，并确保至少有一个线程块可以在流式多处理器上运行，开发人员应该包含单参数 `__launch_bounds__(maxThreadsPerBlock)`，该参数指定核函数将启动的最大块大小。如果不这样做，可能导致"请求的资源太多"的错误。提供两参数版本的 `__launch_bounds__(maxThreadsPerBlock, minBlocksPerMultiprocessor)` 在某些情况下可以提高性能。对于每个核函数的最佳 `minBlocksPerMultiprocessor` 值应通过详细分析确定。

核函数的最佳启动界限通常因主要架构修订而异。以下代码示例说明如何在设备代码中使用 `__HGGC_ARCH__` 宏管理这一点（关于 `__HGGC_ARCH__` 宏值的定义与架构对应关系，参见 [5.2.1.1.1. `__HGGC_ARCH__`](#52111-hggc_arch)）。

```cpp
#if __HGGC_ARCH__ >= 100
#define MY_KERNEL_MAX_THREADS 512
#define MY_KERNEL_MIN_BLOCKS 3
#else
#define MY_KERNEL_MAX_THREADS 256
#define MY_KERNEL_MIN_BLOCKS 2
#endif
__global__ void __launch_bounds__(MY_KERNEL_MAX_THREADS, MY_KERNEL_MIN_BLOCKS)
    kernel() {
  //   ...
}
```

当使用 `__launch_bounds__()` 的第一个参数指定的最大线程数调用 `kernel` 时，很容易倾向于在执行配置中使用 `MY_KERNEL_MAX_THREADS` 作为每块线程数：

```cpp
  // 主机代码
  kernel<<<blocksPerGrid, MY_KERNEL_MAX_THREADS>>>();
```

但是，这是行不通的，因为如执行空间说明符部分所述，在主机代码中 `__HGGC_ARCH__` 是未定义的。因此，`kernel` 将以每块 256 个线程启动。每块线程数应该改为：

* 要么在编译时使用不依赖于 `__HGGC_ARCH__` 的宏或常量，例如

```cpp
  // 主机代码
  kernel<<<blocksPerGrid, THREADS_PER_BLOCK>>>();
```

* 要么根据计算能力在运行时确定

```cpp
  // 主机代码
  hggcGetDeviceProperties(&deviceProp, device);
  int threadsPerBlock =
      (deviceProp.major > 8) ? 2 * THREADS_PER_BLOCK : THREADS_PER_BLOCK;
  kernel<<<blocksPerGrid, threadsPerBlock>>>();
```

`--resource-usage` 编译器选项报告寄存器使用情况。

##### 5.2.4.2. 每线程最大寄存器数 {#5242-每线程最大寄存器数}

为了启用低级性能调优，HGGC C++ 提供了 `__maxnreg__()` 函数限定符，它将性能调整信息传递给后端优化器。`__maxnreg__()` 限定符指定在线程块中的单个线程可以分配的最大寄存器数。在 `__global__` 函数定义中：

```cpp
__global__ void __maxnreg__(maxNumberRegistersPerThread) kernel() {
  // ...
}
```

`maxNumberRegistersPerThread` 变量指定分配给线程块中单个线程的核函数 `kernel()` 的最大寄存器数；它编译为 `.maxnreg` TIX 指令。

不能将 `__launch_bounds__()` 和 `__maxnreg__()` 限定符应用于同一个核函数。

`--maxrregcount <N>` 编译器选项可用于控制文件中所有 `__global__` 函数的寄存器使用。对于带有 `__maxnreg__` 限定符的核函数，忽略此选项。

#### 5.2.5. 同步与原子操作 {#525-同步与原子操作}


##### 5.2.5.1. 线程块同步函数 {#5251-线程块同步函数}

```cpp
void __syncthreads();
int  __syncthreads_count(int predicate);
int  __syncthreads_and(int predicate);
int  __syncthreads_or(int predicate);
```

在同一线程块中，多个线程并行地读写共享内存或全局内存。如果没有显式同步点，不同线程的内存操作可能以任意顺序交叉执行，导致数据竞争（data race）。`__syncthreads*()` 系列内建函数为块内线程提供了一个屏障（barrier），用于消除这类竞争。

语义概述：

* **屏障等待**：调用 `__syncthreads*()` 后，当前线程进入等待状态，直到块内所有活跃线程均到达同一调用点（已退出的线程不参与）。
* **内存排序保证**：屏障同时充当一个隐式内存栅栏——在屏障之前的内存写入对屏障之后的所有参与线程可见。此排序关系符合 C++ 标准中 [intro.races] 的 *happens-before* 语义。

下面的示例利用 `__syncthreads()` 实现块内并行写入共享数组后的安全归约：

```cpp
// 假设 blockDim.x 是 128
__global__ void kernel(int *input, int *output) {
  __shared__ int data[128];
  // 每个线程写入 'data' 的不同元素：
  data[threadIdx.x] = input[threadIdx.x];
  // 所有线程同步，保证 'data' 的所有写入操作有序
  // 在任何线程从 '__syncthreads()' 解除阻塞之前：
  __syncthreads();
  // 单个线程安全地读取 'data'：
  if (threadIdx.x == 0) {
    int sum = 0;
    for (int i = 0; i < blockDim.x; ++i) {
      sum += data[i];
    }
    output[blockIdx.x] = sum;
  }
}
```

在条件分支中放置 `__syncthreads*()` 是合法的，但必须保证块内所有活跃线程对分支条件的求值结果一致（即 uniform condition）。若条件因线程而异，部分线程永远无法到达屏障，导致死锁或不可预测的行为。

以下示例演示了有效行为：

```cpp
// 假设 blockDim.x 是 128
__global__ void kernel(int *input, int *output) {
  __shared__ int data[128];
  data[threadIdx.x] = input[threadIdx.x];
  if (blockIdx.x > 0) { // 正确，整个块线程的统一条件
    __syncthreads();
    output[threadIdx.x] = data[128 - threadIdx.x];
  }
}
```

而以下示例表现出无效行为，例如核函数挂死或未定义行为：

```cpp
// 假设 blockDim.x 是 128
__global__ void kernel1(int *input, int *output) {
  __shared__ int data[256];
  data[threadIdx.x] = input[threadIdx.x];
  if (threadIdx.x > 0) { // 错误，非统一条件
    __syncthreads();     // 未定义行为
    output[threadIdx.x] = data[128 - threadIdx.x];
  }
}
// 假设 blockDim.x 是 128
__global__ void kernel2(int *input, int *output) {
  __shared__ int data[256];
  data[threadIdx.x] = input[threadIdx.x];
  for (int i = 0; i < blockDim.x; ++i) {
    if (i == threadIdx.x) { // 错误，非统一条件
      __syncthreads();      // 未定义行为
    }
  }
  output[threadIdx.x] = data[128 - threadIdx.x];
}
```

* * *

**带谓词的 `__syncthreads()` 变体**：

```cpp
int  __syncthreads_count(int predicate);
```

与 `__syncthreads()` 相同，只是它为块中所有未退出的线程计算谓词，并返回谓词计算结果为非零值的线程数。

```cpp
int  __syncthreads_and(int predicate);
```

与 `__syncthreads()` 相同，只是它为块中所有未退出的线程计算谓词。当且仅当谓词对所有线程都计算为非零值时，它才返回非零值。

```cpp
int  __syncthreads_or(int predicate);
```

与 `__syncthreads()` 相同，只是它为块中所有未退出的线程计算谓词。当且仅当谓词对它们中的一个或多个计算为非零值时，它才返回非零值。

##### 5.2.5.2. 线程束同步函数 {#5252-线程束同步函数}

```cpp
void __syncwarp(unsigned mask = 0xFFFFFFFF);
```

内建函数 `__syncwarp()` 协调同一线程束内的线程之间的通信。当线程束内的某些线程访问共享或全局内存中的相同地址时，可能发生潜在的读后写、写后读或写后写风险。这些数据风险可以通过在这类访问之间同步线程来避免。

调用 `__syncwarp(mask)` 根据 `mask` 来为线程束内的线程提供内存排序：对 `__syncwarp(mask)` 的调用必须发生在 `mask` 选中的线程束中的线程被解除等待状态或退出之前（参见 [C++ 规范 [intro.races]](https://eel.is/c++draft/intro.races)）。

这些函数受 [线程束 __sync 内建约束](#5266-warp-sync-内建约束) 限制。

以下示例演示如何使用 `__syncwarp()` 同步线程束内的线程以安全访问共享内存数组：

```cpp
__global__ void kernel(int *input, int *output) {
  if (threadIdx.x < warpSize) {
    __shared__ int data[warpSize];
    data[threadIdx.x] = input[threadIdx.x];
    __syncwarp(); // 等价于 __syncwarp(0xFFFFFFFF)
    if (threadIdx.x == 0)
      output[0] = data[1];
  }
}
```

##### 5.2.5.3. 内存栅栏函数 {#5253-内存栅栏函数}

HGGC 的内存一致性模型基于弱内存序（relaxed memory ordering）规范。根据该模型的两条核心原则：

1. **操作重排自由度**：编译器和硬件可以自由重排同一线程中无数据依赖的内存操作，以最大化流水线利用率。
2. **跨线程可见性不确定性**：线程 T₁ 执行的写入序列，被线程 T₂ 观察到的顺序可能与 T₁ 的程序顺序不同。

这意味着：若多个线程在无同步保护的情况下并发访问同一地址，其行为未定义。内存栅栏（memory fence）是 HGGC 为解决此类可见性问题而提供的核心同步原语——它在指定范围内建立 [顺序一致性](https://en.cppreference.com/w/cpp/atomic/memory_order) 约束，禁止栅栏前后的内存操作发生重排。栅栏对所有内存空间（共享内存、全局内存、页锁定主机内存）统一生效。

!!! note

    出于安全性和可移植性原因，建议尽可能使用 libhg++提供的 `hggc::atomic_thread_fence`。

下表列出了三个作用域及其对应的 C++ API 与内建函数：

| 作用域 | C++ API (`<hggc/atomic>`)                                                          | 内建函数                 | 排序可见范围                       |
| ------ | ---------------------------------------------------------------------------------- | ------------------------ | ---------------------------------- |
| 块级   | `hggc::atomic_thread_fence(hggc::memory_order_seq_cst, hggc::thread_scope_block)`  | `__threadfence_block()`  | 同一线程块内的所有线程             |
| 设备级 | `hggc::atomic_thread_fence(hggc::memory_order_seq_cst, hggc::thread_scope_device)` | `__threadfence()`        | 同一设备上所有线程块中的线程       |
| 系统级 | `hggc::atomic_thread_fence(hggc::memory_order_seq_cst, hggc::thread_scope_system)` | `__threadfence_system()` | 设备线程 + 主机线程 + 对等设备线程 |

栅栏的语义可以统一描述为：**在作用域内的所有观察者看来，调用线程在栅栏之前发出的内存操作不会与栅栏之后发出的操作发生重排。** 区别仅在于「观察者」的范围：块级限于当前块，设备级扩展到整个 PPU，系统级则覆盖主机和所有可访问的设备。选择栅栏作用域时，应根据参与通信的线程所处层级，选择刚好覆盖它们的最小作用域。

**栅栏效果：发布-订阅真值表**

以典型的发布-订阅（publish-subscribe）场景为例，分析内存栅栏对可观察状态的约束。假设线程 P（发布者）先写入数据 `D=新值`，再设置旗标 `F=1`；线程 S（订阅者）先读取旗标 `F`，再读取数据 `D`：

```cpp
#include <hggc/atomic>
__device__ int F = 0;
__device__ float D = 0.0f;

__device__ void publisher() {   // 线程 P
  D = 3.14f;
  hggc::atomic_thread_fence(hggc::memory_order_seq_cst,
                            hggc::thread_scope_device);
  F = 1;
}
__device__ void subscriber() {  // 线程 S
  int flag = F;
  hggc::atomic_thread_fence(hggc::memory_order_seq_cst,
                            hggc::thread_scope_device);
  float data = D;
}
```

等价的内建函数写法将 `hggc::atomic_thread_fence(...)` 替换为 `__threadfence()` 即可。

下表以真值表形式列出所有可能的观察组合：

| S 读到的 `flag` | S 读到的 `data` | 无栅栏 | 两侧均插入栅栏 | 含义                                             |
| :-------------: | :-------------: | :----: | :------------: | ------------------------------------------------ |
|        0        |       0.0       |   ✓    |       ✓        | S 在 P 开始发布前观察——正常                      |
|        1        |      3.14       |   ✓    |       ✓        | S 在 P 完整发布后观察——正常                      |
|        0        |      3.14       |   ✓    |       ✓        | 时序交错：S 先观察到 D 的更新但 F 尚未可见       |
|        1        |       0.0       |   ✓    |     **✗**      | **违反因果序**：S 看到"已就绪"旗标却读到过期数据 |

最后一行是栅栏排除的唯一危险组合。发布者侧的栅栏保证 `D` 的写入在 `F` 的写入之前提交到内存层次结构；订阅者侧的栅栏保证 `flag` 的读取在 `data` 的读取之前完成。两者配合，消除了"旗标可见、数据不可见"的异常状态。

!!! warning "排序 ≠ 可见性"

    内存栅栏只影响内存操作执行的顺序；它不能保证这些操作对其他线程的可见性。如需确保可见性，可将目标变量声明为 `volatile`（参见 [volatile 限定变量](#51843-volatile-限定变量)）。

**实践示例：Cooperative Groups 分级树形归约**

下面的核函数展示了内存栅栏在 Cooperative Groups 多阶段归约中的应用。该算法采用分级树形归约（hierarchical tree reduction）策略——每轮将活跃块数量减半，经过 ⌈log₂(gridDim.x)⌉ 轮后收敛到最终结果。

设计思路：

1. 所有块并行计算各自区段的部分和，写入全局工作区 `workspace[]`。
2. 进入树形归约循环：步长从 1 开始逐轮翻倍。每轮中，索引对齐的块将相邻位置的值累加到自身位置。
3. 每轮结束时，先插入设备级栅栏确保当前阶段的写入对其他块可见，再通过 `grid.sync()` 等待所有块到达同步点。
4. 循环结束后，最终结果位于 `workspace[0]`。

栅栏在此起到了关键作用：每一轮归约中，块 *k* 需要读取块 *k+stride* 在上一轮写入的值。显式栅栏确保了写入在同步点之前提交到全局内存，使跨块数据交换的正确性得到保证。

!!! note

    此示例需要使用 Cooperative Launch API（`hgLaunchCooperativeKernel`）启动核函数，以保证所有线程块同时驻留在设备上，这是 `grid.sync()` 正确执行的前提。

```cpp
#include <hggc/atomic>
#include <cooperative_groups.h>
namespace cg = cooperative_groups;

__device__ float blockPartialSum(const float *input, int N) {
  extern __shared__ float smem[];
  int tid  = threadIdx.x;
  int base = blockIdx.x * blockDim.x;
  float acc = 0.0f;
  for (int i = base + tid; i < N; i += gridDim.x * blockDim.x)
    acc += input[i];
  smem[tid] = acc;
  __syncthreads();
  for (int s = blockDim.x / 2; s > 0; s >>= 1) {
    if (tid < s) smem[tid] += smem[tid + s];
    __syncthreads();
  }
  return smem[0];
}

__global__ void gridTreeReduce(const float *input, int N,
                               float *workspace, float *output) {
  cg::grid_group grid = cg::this_grid();

  // 阶段 1：每个块独立计算部分和
  float ps = blockPartialSum(input, N);
  if (threadIdx.x == 0)
    workspace[blockIdx.x] = ps;

  // 阶段 2：跨块树形归约，每轮步长翻倍
  for (unsigned stride = 1; stride < gridDim.x; stride <<= 1) {
    hggc::atomic_thread_fence(hggc::memory_order_seq_cst,
                              hggc::thread_scope_device);
    grid.sync();
    unsigned span = stride << 1;
    if (blockIdx.x % span == 0 && blockIdx.x + stride < gridDim.x) {
      if (threadIdx.x == 0)
        workspace[blockIdx.x] += workspace[blockIdx.x + stride];
    }
  }

  // 最终同步确保 workspace[0] 包含全局和
  hggc::atomic_thread_fence(hggc::memory_order_seq_cst,
                            hggc::thread_scope_device);
  grid.sync();
  if (blockIdx.x == 0 && threadIdx.x == 0)
    *output = workspace[0];
}
```


##### 5.2.5.4. 原子函数 {#5254-原子函数}

原子函数执行共享数据上的读-修改-写操作，使它们看起来像在一个步骤中全部执行。原子性确保每个操作要么完全完成，要么根本不执行，为所有参与的线程提供数据一致的视图。

HGGC 提供四种方式的原子函数：

扩展 HGGC C++ 原子函数，hggc::atomic 和 hggc::atomic_ref。

* 它们允许在主机和设备代码中使用。

* 它们遵循 [C++ 标准原子操作](https://en.cppreference.com/w/cpp/atomic/atomic.html) 的语义。

* 它们允许指定原子操作的线程作用域。

标准 C++ 原子函数，[hggc::std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic.html) 和 [hggc::std::atomic_ref](https://en.cppreference.com/w/cpp/atomic/atomic_ref.html)。

* 它们允许在主机和设备代码中使用。

* 它们遵循 [C++ 标准原子操作](https://en.cppreference.com/w/cpp/atomic/atomic.html) 的语义。

* 它们不允许指定原子操作的线程作用域。

编译器内置原子函数，`__hg_atomic_<op>()`。

* 它们只允许在设备代码中使用。

* 它们遵循 [C++ 标准原子内存序](https://en.cppreference.com/w/cpp/atomic/memory_order.html) 的语义。

* 它们允许指定原子操作的线程作用域。

* 它们支持 hggc::std::atomic 和 hggc::std::atomic_ref 允许的数据类型的一个子集，但 128 位数据类型除外。

传统原子函数，`atomic<Op>()`。

* 它们只允许在设备代码中使用。

* 它们仅支持 `memory_order_relaxed` [C++ 原子内存语义](https://en.cppreference.com/w/cpp/atomic/memory_order.html)。

* 它们允许在函数名中指定原子操作的线程作用域。

* 与内置原子函数不同，传统原子函数只保证原子性而不引入同步点（栅栏）。

* 它们支持内置原子函数所允许的数据类型的子集。原子 `add` 操作支持额外的数据类型。

!!! note

    建议出于效率、安全性和可移植性的原因使用扩展 HGGC C++ 原子函数。

##### 5.2.5.4.1. 传统原子函数 {#52541-传统原子函数}

传统原子函数在存储于全局或共享内存中的 32、64 或 128 位字上执行原子读-修改-写操作。例如，`atomicAdd()` 函数读取全局或共享内存中特定地址处的字，向其加一个数，并将结果写回同一地址。

* 原子函数只能在设备函数中使用。

* 对于诸如 `__half2`、`__ppu_bfloat162`、`float2` 和 `float4` 等向量类型，在向量的每个元素上执行读-修改-写操作。不能保证整个向量作为单次访问是原子的。

本节描述的原子函数具有 `hggc::std::memory_order_relaxed` 的 [内存排序](https://en.cppreference.com/w/cpp/atomic/memory_order)，并且仅在特定的线程作用域内是原子的：

* 不带后缀的原子 API，例如 `atomicAdd`，在 `hggc::thread_scope_device` 作用域内是原子的。

* 带 `_block` 后缀的原子 API，例如 `atomicAdd_block`，在 `hggc::thread_scope_block` 作用域内是原子的。

* 带 `_system` 后缀的原子 API，例如 `atomicAdd_system`，如果满足特定条件，则在 `hggc::thread_scope_system` 作用域内是原子的。

以下示例显示 CPU 和 PPU 在地址 `addr` 处原子地更新整数值：

```cpp
#include <hggc_runtime.h>
__global__ void kernel(int *addr) { atomicAdd_system(addr, 10); }
void f(int deviceID) {
  int *addr;
  hggcMallocManaged(&addr, 4);
  *addr = 0;
  hggcDeviceProp deviceProp;
  hggcGetDeviceProperties(&deviceProp, deviceID);
  if (deviceProp.concurrentManagedAccess != 1) {
    return; // 设备无法与CPU同时连贯地访问托管内存
  }
  kernel<<<1, 1>>>(addr);
  __sync_fetch_and_add(addr, 10); // CPU 原子操作
}
```

* * *

请注意，任何原子操作都可以基于 `atomicCAS()`（比较并交换）实现。

###### 5.2.5.4.1.1. `atomicAdd()` {#525411-atomicadd}

```cpp
T atomicAdd(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old + val`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicAdd()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `float`, `double`, `__half2`, `__half`, `__ppu_bfloat16`, `__ppu_bfloat162`。

应用于向量类型（例如 `__half2`）的 `atomicAdd()` 分别保证每个成员的原子性；不能保证整个向量是原子的。

###### 5.2.5.4.1.2. `atomicSub()` {#525412-atomicsub}

```cpp
T atomicSub(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old - val`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicSub()` 支持以下数据类型：

* `int`, `unsigned`。

###### 5.2.5.4.1.3. `atomicInc()` {#525413-atomicinc}

```cpp
unsigned atomicInc(unsigned *address, unsigned val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old >= val ? 0 : (old + 1)`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

###### 5.2.5.4.1.4. `atomicDec()` {#525414-atomicdec}

```cpp
unsigned atomicDec(unsigned *address, unsigned val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `(old == 0 || old > val) ? val : (old - 1)`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

###### 5.2.5.4.1.5. `atomicAnd()` {#525415-atomicand}

```cpp
T atomicAnd(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old & val`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicAnd()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`。

###### 5.2.5.4.1.6. `atomicOr()` {#525416-atomicor}

```cpp
T atomicOr(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old | val`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicOr()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`。

###### 5.2.5.4.1.7. `atomicXor()` {#525417-atomicxor}

```cpp
T atomicXor(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old ^ val`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicXor()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`。

###### 5.2.5.4.1.8. `atomicMin()` {#525418-atomicmin}

```cpp
T atomicMin(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old` 和 `val` 中的最小值。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicMin()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `long long`。

###### 5.2.5.4.1.9. `atomicMax()` {#525419-atomicmax}

```cpp
T atomicMax(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old` 和 `val` 中的最大值。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicMax()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `long long`。

###### 5.2.5.4.1.10. `atomicExch()` {#5254110-atomicexch}

```cpp
T atomicExch(T *address, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 将 `val` 存储到相同的内存地址。

该函数返回 `old` 值。

`atomicExch()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `float`。

###### 5.2.5.4.1.11. `atomicCAS()` {#5254111-atomiccas}

```cpp
T atomicCAS(T *address, T compare, T val);
```

该函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `address` 处的 `old` 值。

2. 计算 `old == compare ? val : old`。

3. 将结果存回相同地址的内存。

该函数返回 `old` 值。

`atomicCAS()` 支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `unsigned short`。

##### 5.2.5.4.2. 内置原子函数 {#52542-内置原子函数}

HGGC 支持用于原子操作的 HGGC 编译器内置函数，遵循与 [C++ 标准原子操作](https://en.cppreference.com/w/cpp/atomic/atomic.html) 和 HGGC 线程作用域相同的内存序语义。这些函数遵循 [GNU 的内置原子函数签名](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)，并带有一个表示线程作用域的额外参数。


以下是 [内存序](https://en.cppreference.com/w/cpp/atomic/atomic.html) 和线程作用域的原始枚举器列表，它们用作内置原子函数的 `order` 和 `scope` 参数：

```cpp
// 原子内存顺序
#  define __ATOMIC_RELAXED 0
#  define __ATOMIC_CONSUME 1
#  define __ATOMIC_ACQUIRE 2
#  define __ATOMIC_RELEASE 3
#  define __ATOMIC_ACQ_REL 4
#  define __ATOMIC_SEQ_CST 5
// 线程作用域
#  define __ATOMIC_SYSTEM 0 // 0 indicates default
#  define __ATOMIC_DEVICE 1
#  define __ATOMIC_BLOCK  2
#  define __ATOMIC_THREAD 10
```

* 内存序对应于 [C++ 标准原子操作的内存顺序](https://en.cppreference.com/w/cpp/atomic/memory_order)。

* 线程作用域遵循 `hggc::thread_scope` 定义。

* `_ATOMIC_CONSUME` 内存序目前使用更强的 `_ATOMIC_ACQUIRE` 内存顺序实现。

* `__ATOMIC_THREAD` 线程作用域目前使用更广泛的 `__ATOMIC_BLOCK` 线程作用域实现。

示例：

```cpp
__device__ T __hg_atomic_load_n(T *src, int order, int scope);
```

原子内置函数有以下限制：

* 它们只能在设备函数中使用。

* 它们不能对局部内存进行操作。

* 这些函数的地址不能被获取。

* `order` 和 `scope` 参数必须是整数字面量；它们不能是变量。

不支持的情况示例：

```cpp
#include <hggc/atomic>
// 主机函数中不允许
__host__ void bar() {
  unsigned u1 = 1, u2 = 2;
  __hg_atomic_load(&u1, &u2, __ATOMIC_RELAXED, __ATOMIC_SYSTEM); // 报错
}
// 不允许应用于局部内存
__device__ void foo() {
  unsigned a = 1, b;
  __hg_atomic_load(&a, &b, __ATOMIC_RELAXED, __ATOMIC_SYSTEM);
}
// 不允许作为模板默认参数。
// 不能获取函数地址。
template <void *F = __hg_atomic_load_n> class X { // 报错
  void *f = F;                                    // 不能获取函数地址。
};
// 不允许在构造函数初始化列表中调用。
class Y {
  int a;

public:
  __device__ Y(unsigned *b)
      : a(__hg_atomic_load_n(b, __ATOMIC_RELAXED, __ATOMIC_SYSTEM)) {}
};
```

###### 5.2.5.4.2.1. `__hg_atomic_fetch_add()`, `__hg_atomic_add()` {#525421-hg_atomic_fetch_add-hg_atomic_add}

```cpp
__device__ T __hg_atomic_fetch_add(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_add(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old + val`。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_add` 返回 `old` 值。

* `__hg_atomic_add` 没有返回值。

这些函数支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `float`, `double`。

###### 5.2.5.4.2.2. `__hg_atomic_fetch_sub()`, `__hg_atomic_sub()` {#525422-hg_atomic_fetch_sub-hg_atomic_sub}

```cpp
__device__ T __hg_atomic_fetch_sub(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_sub(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old - val`。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_sub` 返回 `old` 值。
* `__hg_atomic_sub` 没有返回值。

这些函数支持以下数据类型：

* `int`, `unsigned`, `unsigned long long`, `float`, `double`。

###### 5.2.5.4.2.3. `__hg_atomic_fetch_and()`, `__hg_atomic_and()` {#525423-hg_atomic_fetch_and-hg_atomic_and}

```cpp
__device__ T __hg_atomic_fetch_and(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_and(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old & val`。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_and` 返回 `old` 值。

* `__hg_atomic_and` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.4. `__hg_atomic_fetch_or()`, `__hg_atomic_or()` {#525424-hg_atomic_fetch_or-hg_atomic_or}

```cpp
__device__ T __hg_atomic_fetch_or(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_or(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old | val`。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_or` 返回 `old` 值。

* `__hg_atomic_or` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.5. `__hg_atomic_fetch_xor()`, `__hg_atomic_xor()` {#525425-hg_atomic_fetch_xor-hg_atomic_xor}

```cpp
__device__ T __hg_atomic_fetch_xor(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_xor(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old ^ val`。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_xor` 返回 `old` 值。

* `__hg_atomic_xor` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.6. `__hg_atomic_fetch_min()`, `__hg_atomic_min()` {#525426-hg_atomic_fetch_min-hg_atomic_min}

```cpp
__device__ T __hg_atomic_fetch_min(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_min(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old` 和 `val` 中的最小值。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_min` 返回 `old` 值。

* `__hg_atomic_min` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `int`, `unsigned long long`, `long long`。

###### 5.2.5.4.2.7. `__hg_atomic_fetch_max()`, `__hg_atomic_max()` {#525427-hg_atomic_fetch_max-hg_atomic_max}

```cpp
__device__ T __hg_atomic_fetch_max(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_max(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 计算 `old` 和 `val` 中的最大值。

3. 将结果存回相同地址的内存。

* `__hg_atomic_fetch_max` 返回 `old` 值。

* `__hg_atomic_max` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.8. `__hg_atomic_exchange()`, `__hg_atomic_exchange_n()` {#525428-hg_atomic_exchange-hg_atomic_exchange_n}

```cpp
__device__ T __hg_atomic_exchange_n(T *in1, T in2, int order, int scope);
__device__ void __hg_atomic_exchange(T *in1, T in2, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. `__hg_atomic_exchange_n` 将 `val` 存储到 `in1` 所指向的位置。

`__hg_atomic_exchange` 将 `old` 存储到 `ret` 所指向的位置，并将位于 `val` 地址处的值存储到 `in1` 所指向的位置。

* `__hg_atomic_exchange_n` 返回 `old` 值。

* `__hg_atomic_exchange` 没有返回值。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.9. `__hg_atomic_compare_exchange()`, `__hg_atomic_compare_exchange_n()` {#525429-hg_atomic_compare_exchange-hg_atomic_compare_exchange_n}

```cpp
__device__ bool __hg_atomic_compare_exchange(T *in1, T *in2, T *in3, bool weak,
                                             int success_order, int fail_order,
                                             int scope);
__device__ bool __hg_atomic_compare_exchange_n(T *in1, T *in2, T in3, bool weak,
                                               int success_order,
                                               int fail_order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `in1` 处的 `old` 值。

2. 将 `old` 与 `in2` 所指向的值进行比较。

3. 如果它们相等，则返回值为 `true` 并将 `in3` 存储到 `in1` 所指向的位置。否则，它返回 `false` 并将 `old` 存储到 `in2` 所指向的位置。

参数 `weak` 被忽略，它会选择 `success_order` 和 `fail_order` 中更强的内存序来执行比较交换操作。

这些函数支持以下数据类型：

* `unsigned`, `unsigned long long`。

###### 5.2.5.4.2.10. `__hg_atomic_load()`, `__hg_atomic_load_n()` {#5254210-hg_atomic_load-hg_atomic_load_n}

```cpp
__device__ void __hg_atomic_load(T *src, T *dst, int order, int scope);
__device__ T __hg_atomic_load_n(T *src, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. 读取位于全局或共享内存地址 `src` 处的 `old` 值。

2. `__hg_atomic_load` 将 `old` 存储到 `dst` 所指向的位置。

`__hg_atomic_load_n` 返回 `old`。

这些函数支持以下数据类型：

* 任意大小为 1、2、4、8 或 16 字节的 unsigned 整数数据类型。

`order` 不能是 `__ATOMIC_RELEASE` 或 `__ATOMIC_ACQ_REL`。

###### 5.2.5.4.2.11. `__hg_atomic_store()`, `__hg_atomic_store_n()` {#5254211-hg_atomic_store-hg_atomic_store_n}

```cpp
__device__ void __hg_atomic_store(T *dst, T *src, int order, int scope);
__device__ void __hg_atomic_store_n(T *dst, T res, int order, int scope);
```

这些函数在一个原子事务中执行以下操作：

1. `__hg_atomic_store` 读取位于全局或共享内存地址 `src` 处的 `old` 值。

2. 将 `old` 存储到 `dst` 所指向的位置。

`__hg_atomic_store_n` 直接将 `res` 存储到 `src` 所指向的位置。

`order` 不能是 `__ATOMIC_CONSUME`、`__ATOMIC_ACQUIRE` 或 `__ATOMIC_ACQ_REL`。

###### 5.2.5.4.2.12. `__hg_atomic_thread_fence()` {#5254212-hg_atomic_thread_fence}

```cpp
__device__ void __hg_atomic_thread_fence(int order, int scope);
```

这个原子函数根据指定的内存序建立此线程请求的内存访问之间的排序。线程作用域参数指定了可能观察到此操作排序效果的一组线程。

#### 5.2.6. Warp 函数 {#526-warp函数}

以下部分描述了 Warp 函数，这些函数允许 Warp 内的线程相互通信并执行计算。

!!! note

    出于效率、安全性和可移植性的原因，建议尽可能使用 `CUB`（Warp-Wide 集合原语）来执行 warp 操作。

##### 5.2.6.1. Warp 活动掩码 {#5261-warp活动掩码}

```cpp
unsigned __activemask();
```

该函数返回一个 32 位整数掩码，表示调用 warp 中当前所有活动的线程。当调用 `__activemask()` 时，如果 warp 中的第 N 个通道是活动的，则第 N 位被置 1。非活动线程在返回的掩码中由 0 位表示。退出程序的线程始终标记为非活动。

警告

`__activemask()` 不能用于确定哪些 warp 通道执行给定分支。此函数旨在用于机会主义 warp 级编程，并且仅提供 warp 内活动线程的瞬时快照。

```cpp
  // 检查是否有至少一个线程的谓词评估为真
  if (pred) {
    // 无效：'at_least_one'的值是非确定性的
    // 并且可能在不同执行之间变化。
    at_least_one = __activemask() > 0;
  }
```

请注意，在 `__activemask()` 调用处收敛的线程不能保证在后续指令处保持收敛，除非那些指令是 warp 同步原语 (`__sync`)。

例如，编译器可以重新排序指令，并且活动线程集可能不会保留：

```cpp
  unsigned mask =
      __activemask(); // 假设mask == 0xFFFFFFFF（所有位都设置，所有线程活动）
  int predicate = threadIdx.x % 2 == 0; // 对偶数线程为1，对奇数线程为0
  int result = __any_sync(mask, predicate); // 活动线程可能不会保留
```

##### 5.2.6.2. Warp 表决函数 {#5262-warp表决函数}

```cpp
int __all_sync(unsigned mask, int pred);
int __any_sync(unsigned mask, int pred);
unsigned __ballot_sync(unsigned mask, int pred);
```

warp 表决函数使给定 [warp](#132-线程束warps和simt) 的线程能够执行归约和广播操作。这些函数从 warp 中的每个未退出线程接收一个整数 `pred` 作为输入，并将这些值与零进行比较。然后将比较结果在 warp 的所有活动线程之间以下列方式之一进行合并（归约），并将单个返回值广播给每个参与的线程：

`__all_sync(unsigned mask, int pred)`:

求解 `mask` 中所有未退出线程的 `pred`，如果它们全部为非零则返回非零值。

`__any_sync(unsigned mask, pred)`:

求解 `mask` 中所有未退出线程的 `pred`，如果有其中一个或多个为非零则返回非零值。

`__ballot_sync(unsigned mask, pred)`:

求解 `mask` 中所有未退出线程的 `predicate`，并返回一个整数，其中第 N 位在 warp 的第 N 个线程的 `pred` 求值为非零且第 N 个线程处于活动状态时被置 1，否则为零。

这些函数受 [Warp __sync 内建约束](#5266-warp-sync-内建约束) 限制。

警告

这些内建函数不提供任何内存排序。

##### 5.2.6.3. Warp 匹配函数 {#5263-warp匹配函数}

```cpp
unsigned __match_any_sync(unsigned mask, T value);
unsigned __match_all_sync(unsigned mask, T value, int *pred);
```

warp 匹配函数在 [warp](#132-线程束warps和simt) 内未退出线程之间对变量执行广播和比较操作。

`__match_any_sync`

返回掩码选择出的未退出线程中具有相同 `value` 的线程的掩码。

`__match_all_sync`

如果 `mask` 中所有未退出线程都有相同的 `value`，则返回 `mask`；否则返回 0。如果 `mask` 中所有未退出线程都有相同的 `value`，则谓词 `pred` 设置为 `true`；否则谓词设置为 `false`。

`T` 可以是 `int`、`unsigned`、`long`、`unsigned long`、`long long`、`unsigned long long`、`float` 或 `double`。

这些函数受 [Warp __sync 内建约束](#5266-warp-sync-内建约束) 限制。

警告

这些内建函数不提供任何内存排序。

##### 5.2.6.4. Warp 归约函数 {#5264-warp归约函数}

!!! note

    出于效率、安全性和可移植性的原因，建议尽可能使用 `CUB`（Warp-Wide 集合原语）来执行 warp 归约操作。

```cpp
T __reduce_add_sync(unsigned mask, T value);
T __reduce_min_sync(unsigned mask, T value);
T __reduce_max_sync(unsigned mask, T value);
unsigned __reduce_and_sync(unsigned mask, unsigned value);
unsigned __reduce_or_sync(unsigned mask, unsigned value);
unsigned __reduce_xor_sync(unsigned mask, unsigned value);
```

`__reduce_<op>_sync` 内建函数在同步 `mask` 中选择的所有未退出线程后对 `value` 中提供的数据执行归约操作。

`__reduce_add_sync`, `__reduce_min_sync`, `__reduce_max_sync`

返回对 `mask` 中每个未退出线程提供的 `value` 应用算术加法、最小值或最大值归约操作后的结果。`T` 可以是有符号或无符号整数。

`__reduce_and_sync`, `__reduce_or_sync`, `__reduce_xor_sync`

返回对 `mask` 中每个未退出线程提供的 `value` 应用按位与、或、异或归约操作后的结果。

这些函数受 [Warp __sync 内建约束](#5266-warp-sync-内建约束) 限制。

警告

这些内建函数不提供任何内存排序。

##### 5.2.6.5. Warp 洗牌函数 {#5265-warp洗牌函数}

```cpp
T __shfl_sync(unsigned mask, T var, int srcLane, int width = warpSize);
T __shfl_up_sync(unsigned mask, T var, unsigned delta, int width = warpSize);
T __shfl_down_sync(unsigned mask, T var, unsigned delta, int width = warpSize);
T __shfl_xor_sync(unsigned mask, T var, int laneMask, int width = warpSize);
```

Warp 洗牌函数在 [warp](#132-线程束warps和simt) 内未退出线程之间交换值，而无需使用共享内存。

`__shfl_sync()`：直接从索引通道复制。

内建函数返回由 `srcLane` 给出 ID 的线程持有的 `value` 值。

* 如果 `width` 小于 `warpSize`，那么 warp 的每个子部分表现得像一个独立实体，起始逻辑通道 ID 为 0。

* 如果 `srcLane` 超出范围 `[0, width - 1]`，则结果对应于 `srcLane % width` 持有的值，确保在同一子部分内。

* * *

`__shfl_up_sync()`：从比调用者 ID 更低的通道复制。

内建函数通过把调用者的通道 ID 减去 `delta` 来计算源通道 ID。返回所得通道 ID 持有的 `value` 值：实际上是 `value` 沿 warp 向上移动 `delta` 个通道的效果。

* 如果 `width` 小于 `warpSize`，那么 warp 的每个子部分表现得像一个独立实体，起始逻辑通道 ID 为 0。

* 源通道索引不会回绕 `width` 的值，因此较低的 `delta` 通道将保持不变。

* * *

`__shfl_down_sync()`：从比调用者 ID 更高的通道复制。

内建函数通过把调用者的通道 ID 添上 `delta` 来计算源通道 ID。返回所得通道 ID 持有的 `value` 值：实际上是 `value` 沿 warp 向下移动 `delta` 个通道的效果。

* 如果 `width` 小于 `warpSize`，那么 warp 的每个子部分表现得像一个独立实体，起始逻辑通道 ID 为 0。

* 与 `__shfl_up_sync()` 一样，源通道的 ID 号不会回绕 `width` 的值，所以较高的 `delta` 通道实际上将保持不变。

* * *

`__shfl_xor_sync()`：基于自身通道 ID 的按位 XOR 从通道复制。

内建函数通过对调用者的通道 ID 和 `laneMask` 执行按位 XOR 来计算源通道 ID：返回所得通道 ID 持有的 `value` 值。这种模式实现了蝴蝶寻址模式，用于树形归约和广播。

* 如果 `width` 小于 `warpSize`，那么每组连续的 `width` 个线程都能够访问前面组的元素。然而，如果它们尝试访问来自靠后线程组的元素，它们自己的 `value` 值将被返回。

* * *

`T` 可以是：

* `int`、`unsigned`、`long`、`unsigned long`、`long long`、`unsigned long long`、`float` 或 `double`。

* 包含 `hggc_fp16.h` 头文件时的 `__half` 和 `__half2`。

* 包含 `hggc_bf16.h` 头文件时的 `__ppu_bfloat16` 和 `__ppu_bfloat162`。

线程只能从活动的且参与内建函数的另一个线程读取数据。如果目标线程非活动，获取到的值是未定义的。

`width` 必须是在范围 `[1, warpSize]` 中的 2 的幂，即 1、2、4、8、16 或 32。其他值将产生未定义的结果。

这些函数受 [Warp __sync 内建约束](#5266-warp-sync-内建约束) 限制。

有效 warp 洗牌使用的示例：

```cpp
  int laneId = threadIdx.x % warpSize;
  int data = 0;
  // 所有warp线程从通道0获取数据
  int result = __shfl_sync(0xFFFFFFFF, data, 0);
  if (laneId < 4) {
    // 通道0、1、2、3从通道1获取数据
    result = __shfl_sync(0b1111, data, 1);
  }
  // 通道[0 - 15]从通道0获取数据
  // 通道[16 - 31]从通道16获取数据
  result = __shfl_sync(0xFFFFFFFF, data, warpSize / 2);
  // 每个通道从两个位置之上的通道获取数据
  // 通道30、31获得它们原来的值
  result = __shfl_down_sync(0xFFFFFFFF, data, 2);
```

无效 warp 洗牌使用的示例：

```cpp
  int laneId = threadIdx.x % warpSize;
  int value = 0;
  // 未定义行为：通道0没有参与调用
  int result = (laneId > 0) ? __shfl_sync(0xFFFFFFFF, value, 0) : 0;
  if (laneId <= 4) {
    // 未定义行为：对于通道3、4来说，目标通道5、6不活跃
    result = __shfl_down_sync(0b11111, value, 2);
  }
  // 未定义行为：width不是2的幂
  __shfl_sync(0xFFFFFFFF, value, 0, 31);
```

警告

这些内建函数不隐含内存屏障。它们不保证任何内存序。

示例 1：单个值在整个 warp 中的广播。

```cpp
#include <assert.h>
__global__ void kernel(int input) {
  int laneId = threadIdx.x % 32;
  int value;
  if (laneId == 0) { // 除通道0外所有线程的未使用变量
    value = input;
  }
  // 同步warp中的所有线程，并从通道0获取"value"
  value = __shfl_sync(0xFFFFFFFF, value, 0);
  assert(value == input);
}
int main() {
  kernel<<<1, 32>>>(1234);
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

示例 2：跨越 8 个线程子分区的包含式前缀扫描。

!!! note

    建议使用 cub::WarpScan 函数这种高效和通用的 warp 扫描函数。

HGGC C++方式：

```cpp
#include <cstdio>
#include <cub/cub.cuh>
__global__ void kernel() {
  using WarpScan = cub::WarpScan<int, 8>;
  __shared__ WarpScan::TempStorage storage;
  int laneId = threadIdx.x % 32;
  int value = 31 - laneId; // 起始累加值
  int sum;
  WarpScan(storage).InclusiveSum(value, sum);
}
int main() {
  kernel<<<1, 32>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

内建函数方式：

```cpp
#include <stdio.h>
__global__ void kernel() {
  int laneId = threadIdx.x % 32;
  int value = 31 - laneId; // 起始累加值
  // 循环累积分区内的扫描。
  // 扫描需要log2(8) == 3步才能完成8个线程
  for (int delta = 1; delta <= 4; delta *= 2) {
    // 从laneId - delta读取
    int result = __shfl_up_sync(0xFFFFFFFF, value, delta, 8);
    int lane = laneId % 8 - delta;
    if (lane >= 0) // 'lane < 0'的通道保持其值不变
      value += result;
  }
}
int main() {
  kernel<<<1, 32>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

示例 3：跨 warp 的归约。

!!! note

    建议使用 cub::WarpReduce 函数这种高效和通用的 warp 归约函数。

```cpp
#include <stdio.h>
__global__ void kernel() {
  int laneId = threadIdx.x % 32;
  int value = 31 - laneId; // 起始累加值
  // 使用 XOR 模式执行蝶形规约
  // 完整warp规约需要log2(32) == 5步
  for (int i = 1; i <= 16; i *= 2)
    value += __shfl_xor_sync(0xFFFFFFFF, value, i);
  // "value"现在包含了所有线程的总和
}
int main() {
  kernel<<<1, 32>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

##### 5.2.6.6. Warp `__sync` 内建约束 {#5266-warp-sync-内建约束}

所有的 warp `__sync` 内建函数，如：

* `__shfl_sync`, `__shfl_up_sync`, `__shfl_down_sync`, `__shfl_xor_sync`

* `__match_any_sync`, `__match_all_sync`

* `__reduce_add_sync`, `__reduce_min_sync`, `__reduce_max_sync`, `__reduce_and_sync`, `__reduce_or_sync`, `__reduce_xor_sync`

* `__syncwarp`

使用 `mask` 参数指示哪个 warp 线程参与调用。此参数确保硬件执行内建函数之前正确的收敛。

`mask` 中的每一位对应于线程的通道 ID (`threadIdx.x % warpSize`)。内建函数会等待，直到 `mask` 中指定的所有未退出 warp 线程到达调用点。

以下约束必须满足以正确执行：

* 每个调用线程必须在其对应的 `mask` 位中设置了 1。

* 每个非调用线程必须在 `mask` 中将其对应的位设置为 0。退出的线程会被忽略。

* `mask` 中指定的所有未退出线程必须以相同的 `mask` 值执行内建函数。

* 当 warp 线程并发调用内建函数但带有不同的 `mask` 值时，前提是这些掩码互不重叠。即使在分支控制流中，这种情况也是有效的。

核函数挂起时 warp `__sync` 函数的行为是无效的，或者在以下情况下未定义：

* 调用线程未在 `mask` 中指定。

* `mask` 中指定的未退出线程未能最终退出或在相同程序点以相同的 `mask` 值调用内建函数。

* 在条件代码中，所有条件必须在 `mask` 中指定的所有未退出线程间求值一致。

!!! note

    当所有 warp 线程都参与调用时，即当 `mask` 设置为 `0xFFFFFFFF` 时，内建函数达到最佳效率。

有效 warp 内建函数使用示例：

```cpp
__global__ void kernel(int pred) {
  if (threadIdx.x < 4) {      // 线程0, 1, 2, 3是活动的
    __all_sync(0b1111, pred); // 正确，线程0, 1, 2, 3参与调用
  }
  if (threadIdx.x == 0)
    return; // 退出
  // 正确，所有未退出线程参与调用
  __all_sync(0xFFFFFFFF, pred);
}
```

不相交 `mask` 示例：

```cpp
__global__ void kernel(int *input, int *output) {
  if (threadIdx.x < warpSize) {
    __shared__ int data[warpSize];
    data[threadIdx.x] = input[threadIdx.x];
    unsigned mask = threadIdx.x < 16 ? 0xFFFF : 0xFFFF0000; // 正确
    __syncwarp(mask);
    if (threadIdx.x == 0 || threadIdx.x == 16)
      output[threadIdx.x] = data[threadIdx.x + 1];
  }
}
__global__ void kernel_branches(int *input, int *output) {
  if (threadIdx.x < warpSize) {
    __shared__ int data[warpSize];
    data[threadIdx.x] = input[threadIdx.x];
    if (threadIdx.x < 16) {
      unsigned mask = 0xFFFF; // 正确
      __syncwarp(mask);
      output[threadIdx.x] = data[15 - threadIdx.x];
    } else {
      unsigned mask = 0xFFFF0000; // 正确
      __syncwarp(mask);
      output[threadIdx.x] = data[31 - threadIdx.x];
    }
  }
}
```

无效 warp 内建函数使用示例：

```cpp
  if (threadIdx.x < 4) { // 线程0, 1, 2, 3是活动的
    __all_sync(0b0000011, pred); // 错误，线程2, 3是活动的但在掩码中未设置
    __all_sync(0b0111111, pred); // 错误，线程4, 5不活动但在掩码中已设置
  }

  // 错误，参与线程有不同的且重叠的掩码
  __all_sync(threadIdx.x == 0 ? 1 : 0xFFFFFFFF, pred);
```

#### 5.2.7. Warp 矩阵函数 {#527-warp矩阵函数}

C++ warp 矩阵操作利用 Tensor Cell 来加速形式为 `D=A*B+C` 的矩阵计算。这些操作支持混合精度浮点数据。这需要 [warp](#132-线程束warps和simt) 中所有线程的合作。此外，只有当条件求值在整个 [warp](#132-线程束warps和simt) 中结果完全相同时，才允许在条件代码中执行这些操作，否则代码执行可能会挂起。

##### 5.2.7.1. 描述 {#5271-描述}

以下所有函数和类型都在 `awmma` 命名空间中定义。子字节操作被视为预览版功能，即它们的数据结构和 API 可能会发生变化，并且可能与未来版本不兼容。此额外功能在 `awmma::experimental` 命名空间中定义。

```cpp
template <typename Use, int m, int n, int k, typename T, typename Layout = void>
class fragment;
void load_matrix_sync(fragment<...> &a, const T *p, unsigned ldm);
void load_matrix_sync(fragment<...> &a, const T *p, unsigned ldm,
                      layout_t layout);
void store_matrix_sync(T *p, const fragment<...> &a, unsigned ldm,
                       layout_t layout);
void fill_fragment(fragment<...> &f, const T &in);
void mma_sync(fragment<...> &d, const fragment<...> &a, const fragment<...> &b,
              const fragment<...> &c, bool satf = false);
```

`fragment`

一个重载类，包含分布在 warp 中所有线程上的矩阵部分。矩阵元素映射到 `fragment` 内存的方式未指定，并且在未来架构中可能会更改。

只允许某些模板参数组合。第一个模板参数指定片段如何参与矩阵运算。`Use` 的可接受值为：

* `matrix_a`： 当片段用作第一个乘数 `A` 时，

* `matrix_b`： 当片段用作第二个乘数 `B` 时，

* `accumulator`： 当片段用作源或目标累加器（分别为 `C` 或 `D`）时。

`m`、`n` 和 `k` 描述参与乘加运算的 warp-wide 矩阵块的形状。每个块的维度取决于其角色。对于 `matrix_a`，块采用 `m × k` 维；对于 `matrix_b`，维度是 `k × n`，而 `accumulator` 块是 `m × n`。

数据类型 `T` 在用于乘数时可以是 `float`、`__half`、`__ppu_bfloat16`、`char` 或 `unsigned char`，用于累加器时可以是 `float`、`int` 或 `__half`。如 [元素类型和矩阵大小](#5273-元素类型和矩阵大小) 中所述，有限制地支持了累加器和乘数类型组合。布局参数必须为 `matrix_a` 和 `matrix_b` 片段指定。`row_major` 或 `col_major` 分别表示矩阵行或列中的元素在内存中是连续的。对于 `accumulator` 矩阵，`Layout` 参数应保留默认值 `void`。只有在加载或存储累加器时（如下所述），才需要指定行布局或列布局。

`load_matrix_sync`

等待所有 warp 通道到达 `load_matrix_sync`，然后从内存加载矩阵片段 `a`。`p` 必须是指向内存中矩阵第一个元素的 256 位对齐指针。`ldm` 描述连续行（对于行优先布局）或列（对于列优先布局）之间的步长（以元素为单位），并且对于 `__half` 元素类型必须是 8 的倍数，对于 `float` 元素类型必须是 4 的倍数（即，在这两种情况下都是 16 字节的倍数）。如果片段是 `accumulator`，则 `layout` 参数必须指定为 `mem_row_major` 或 `mem_col_major`。对于 `matrix_a` 和 `matrix_b` 片段，布局由片段的 `layout` 参数推断得出。`p`、`ldm`、`layout` 以及 `a` 的所有模板参数的值在 warp 中的所有线程中必须相同。warp 中的所有线程都必须调用此函数，否则结果未定义。

`store_matrix_sync`

等待所有 warp 通道到达 `store_matrix_sync`，然后将矩阵片段 `a` 存储到内存中。`p` 必须是指向内存中矩阵第一个元素的 256 位对齐指针。`ldm` 描述连续行（对于行优先布局）或列（对于列优先布局）之间的步长（以元素为单位），并且对于 `__half` 元素类型必须是 8 的倍数，对于 `float` 元素类型必须是 4 的倍数。（即，在这两种情况下都是 16 字节的倍数）。输出矩阵的布局必须指定为 `mem_row_major` 或 `mem_col_major`。`p`、`ldm`、`layout` 以及所有模板参数的值在 warp 中的所有线程中必须相同。

`fill_fragment`

用常量值 `in` 填充矩阵片段。由于矩阵元素到每个片段的映射未指定，因此通常由 warp 中的所有线程使用 `in` 的公共值调用此函数。

`mma_sync`

等待所有 warp 通道到达 `mma_sync`，然后执行 warp 同步矩阵乘加运算 `D=A*B+C`。也支持就地操作 `C=A*B+C`。`satf` 值和每个矩阵片段的模板参数必须在 warp 中的所有线程中相同。此外，片段 `A`、`B`、`C` 和 `D` 之间的模板参数 `m`、`n` 和 `k` 必须匹配。这个函数必须由 warp 中的所有线程调用，否则结果未定义。

如果 `satf`（饱和到有限值）模式为 `true`，则目标累加器具有以下额外的数值属性：

* 如果元素结果为+Infinity，则相应的累加器将包含 `+MAX_NORM`

* 如果元素结果为-Infinity，则相应的累加器将包含 `-MAX_NORM`

* 如果元素结果为 NaN，则相应的累加器将包含 `+0`

由于矩阵元素到每个线程的 `fragment` 的映射未指定，必须在调用 `store_matrix_sync` 后从内存（共享或全局）访问各个矩阵元素。在特殊情况下，当 warp 中的所有线程都将统一地对所有片段元素应用逐元素操作时，可以使用以下 `fragment` 类成员实现直接元素访问。

```cpp
enum fragment<Use, m, n, k, T, Layout>::num_elements;
T fragment<Use, m, n, k, T, Layout>::x[num_elements];
```

例如，以下代码将 `accumulator` 矩阵块元素缩小成一半。

```cpp
  awmma::fragment<awmma::accumulator, 16, 16, 16, float> frag;
  float scale = 0.5f; // warp中所有线程的相同值
  /*...*/
  for (int t = 0; t < frag.num_elements; t++)
    frag.x[t] *= scale;
```

##### 5.2.7.2. 替代浮点数 {#5272-替代浮点数}

Tensor Cell 支持替代浮点运算类型。

`__ppu_bfloat16`

这种数据格式是一种替代 fp16 格式，具有与 f32 相同的范围，但精度降低（7 位）。你可以直接使用 `hggc_bf16.h` 中提供的 `__ppu_bfloat16` 类型。带有 `__ppu_bfloat16` 数据类型的矩阵片段需要与 `float` 类型的累加器组合使用。支持的形状和操作与 `__half` 相同。

`tf32`

这种数据格式是 Tensor Cell 支持的特殊浮点格式，具有与 f32 相同的范围，但精度降低（>=10 位）。此格式的内部布局是实现定义的。要将此浮点格式与 awmma 操作一起使用，输入矩阵必须手动转换为 tf32 精度。

为了便于转换，提供了一个内置函数 `__float_to_tf32`。虽然该内置函数的输入和输出参数都是 `float` 类型，但输出将在数值上是 `tf32` 的。这种精度仅用于与 Tensor Cell 一起使用，如果与其他 `float` 类型操作混合使用，结果的精度和范围将是未定义的。

当输入矩阵（`matrix_a` 或 `matrix_b`）转换为 tf32 精度后，`precision::tf32` 精度的 `fragment` 和 `load_matrix_sync` 的 `float` 数据类型相结合，即可有效利用此功能。累加器片段都必须具有 `float` 数据类型。唯一支持的矩阵大小是 16x16x8（m-n-k）。

片段的元素表示为 `float`，因此从 `element_type<T>` 到 `storage_element_type<T>` 的映射是：

```text
precision::tf32 -> float
```

##### 5.2.7.3. 元素类型和矩阵大小 {#5273-元素类型和矩阵大小}

Tensor Cell 支持多种元素类型和矩阵大小。下表展示了支持的各种 `matrix_a`、`matrix_b` 和 `accumulator` 矩阵组合：

| 矩阵 A        | 矩阵 B        | 累加器 | 矩阵大小 (m-n-k) |
| ------------- | ------------- | ------ | ---------------- |
| __half        | __half        | float  | 16x16x16         |
| __half        | __half        | float  | 32x8x16          |
| __half        | __half        | float  | 8x32x16          |
| __half        | __half        | __half | 16x16x16         |
| __half        | __half        | __half | 32x8x16          |
| __half        | __half        | __half | 8x32x16          |
| unsigned char | unsigned char | int    | 16x16x16         |
| unsigned char | unsigned char | int    | 32x8x16          |
| unsigned char | unsigned char | int    | 8x32x16          |
| signed char   | signed char   | int    | 16x16x16         |
| signed char   | signed char   | int    | 32x8x16          |
| signed char   | signed char   | int    | 8x32x16          |

替代浮点支持：

| 矩阵 A          | 矩阵 B          | 累加器 | 矩阵大小 (m-n-k) |
| --------------- | --------------- | ------ | ---------------- |
| __ppu_bfloat16  | __ppu_bfloat16  | float  | 16x16x16         |
| __ppu_bfloat16  | __ppu_bfloat16  | float  | 32x8x16          |
| __ppu_bfloat16  | __ppu_bfloat16  | float  | 8x32x16          |
| precision::tf32 | precision::tf32 | float  | 16x16x8          |

##### 5.2.7.4. 示例 {#5274-示例}

以下代码在单个 warp 中实现 16x16x16 矩阵乘法。

```cpp
#include <hggc_mma.h>
__global__ void kernel(half *a, half *b, float *c) {
  // 声明片段
  awmma::fragment<awmma::matrix_a, 16, 16, 16, half, awmma::col_major> a_frag;
  awmma::fragment<awmma::matrix_b, 16, 16, 16, half, awmma::row_major> b_frag;
  awmma::fragment<awmma::accumulator, 16, 16, 16, float> c_frag;
  // 将输出初始化为零
  awmma::fill_fragment(c_frag, 0.0f);
  // 加载输入
  awmma::load_matrix_sync(a_frag, a, 16);
  awmma::load_matrix_sync(b_frag, b, 16);
  // 执行矩阵乘法
  awmma::mma_sync(c_frag, a_frag, b_frag, c_frag);
  // 存储输出
  awmma::store_matrix_sync(c, c_frag, 16, awmma::mem_row_major);
}
```

#### 5.2.8. 调试和诊断 {#528-调试和诊断}

##### 5.2.8.1. 断言 {#5281-断言}

```cpp
void assert(int expression);
```

如果 `expression` 等于零，`assert()` 宏会停止核函数执行。如果程序在调试中，则会触发断点，允许使用调试器检查设备的当前状态。否则，对于每个 `expression` 等于零的线程，在通过 `hggcDeviceSynchronize()`、`hggcStreamSynchronize()` 或 `hggcEventSynchronize()` 与主机同步后，都会向 stderr 打印一条消息。此消息的格式如下：

```cpp
<filename>:<line number> :<function> : block: [blockIdx.x,blockIdx.y,blockIdx.z], thread: [threadIdx.x,threadIdx.y,threadIdx.z] Assertion `<expression>` failed.
```

核函数执行被中止，并在主机程序中引发中断。`assert()` 宏会导致 HGGC 上下文损坏，导致任何后续 HGGC 调用或核函数调用以 `hggcErrorAssert` 失败。

如果 `expression` 不为零，则不影响核函数执行。

例如，以下来自源文件 `test.hg` 的程序：

```cpp
#include <assert.h>
__global__ void kernel(void) {
  // 不会有影响
  assert(1);
  // 将停止核函数执行
  assert(0);
}

int main(void) {
  kernel<<<1, 1>>>();
  hggcDeviceSynchronize(); // 等待 kernel 执行结束
  return 0;
}
```

将输出：

```text
test.hg:6 :void kernel() : block: [0,0,0], thread: [0,0,0] Assertion `0` failed.
```

断言用于调试目的。由于它们可能会影响性能，建议在生产代码中禁用它们。可以在编译时通过在包含 `assert.h` 或 `<cassert>` 之前定义 `NDEBUG` 预处理器宏，或使用编译器标志 `-DNDEBUG` 来禁用它们。请注意，表达式不应有副作用；否则，禁用断言将影响代码的功能。

##### 5.2.8.2. 断点函数 {#5282-断点函数}

可以从任何设备线程通过调用 `__brkpt()` 函数来暂停核函数的执行。

```cpp
void __brkpt();
```

### 5.3. HGGC 环境变量 {#53-hggc-环境变量}

本节列出了 HGGC 环境变量。

#### 5.3.1. 设备枚举与属性 {#531-设备枚举与属性}

##### 5.3.1.1. `HGGC_VISIBLE_DEVICES` {#5311-hggc_visible_devices}

通过此环境变量可以精确控制应用程序能够发现哪些 PPU 设备，并自定义它们的枚举顺序。未设置时所有 PPU 均可见；设为空字符串则屏蔽全部设备。

**取值格式**：以逗号分隔的 PPU 标识符列表。支持三种标识方式：

| 标识方式 | 格式                                                                  | 示例                     |
| -------- | --------------------------------------------------------------------- | ------------------------ |
| 整数序号 | `ppu-smi`（PPU System Management Interface）报告的从 0 开始的设备编号 | `0,2`                    |
| PPU UUID | `ppu-smi -L` 输出的完整或前缀 UUID                                    | `PPU-8932f937`           |
| MIG 实例 | `MIG-<UUID>/<PPU 实例 ID>/<计算实例 ID>`                              | `MIG-PPU-8932f937-…/1/2` |

**关于 UUID 前缀**：只要前缀在当前系统中能唯一标识目标设备，即可使用缩写形式。MIG 模式下仅支持枚举单个实例。

**无效标识符的截断行为**：列表按从左到右顺序解析。一旦遇到无效条目（如不存在的序号 `-1`），解析终止，后续的标识符将被忽略。例如 `HGGC_VISIBLE_DEVICES=0,2,-1,1` 的结果是仅设备 0 和 2 可见，设备 1 因排在无效项之后而被屏蔽。

**对 HGGC API 的影响**：设置此变量后，`hggcGetDeviceCount()` 仅返回可见设备数量，HGGC API 中的整数设备 ID 范围变为 `[0, 可见设备数−1]`。序号按列表中的出现顺序分配：例如，使用 `HGGC_VISIBLE_DEVICES=2,1` 时，调用 `hggcSetDevice(0)` 会将物理设备 2 设为当前设备，因为它在列表中首先出现并被分配序号 0；之后调用 `hggcGetDevice(&device_ordinal)` 也会将 `device_ordinal` 设为 0，对应物理设备 2。

**示例**：

```cpp
ppu-smi -L  # 查看系统中所有 PPU 及其 UUID
HGGC_VISIBLE_DEVICES=0,1
HGGC_VISIBLE_DEVICES=PPU-8932f937-d72c-4106-c12f-20bd9faed9f6
HGGC_VISIBLE_DEVICES=MIG-PPU-8932f937-d72c-4106-c12f-20bd9faed9f6/1/2
```

* * *

#### 5.3.2. 执行 {#532-执行}

##### 5.3.2.1. `HGGC_LAUNCH_BLOCKING` {#5321-hggc_launch_blocking}

控制核函数启动是否以同步模式运行。

默认情况下（`0`），所有核函数启动都是异步的——主机线程提交工作后立即继续执行，不等待设备完成。将此变量设为 `1` 会强制每次核函数启动在设备执行完毕后才返回控制权给 CPU。

同步模式的主要用途是调试：异步模式下，HGGC API 报告的错误往往延迟到后续的同步点才被捕获，难以定位到出错的具体调用。启用同步模式后，错误会在触发它的 API 调用处立即报告。需注意，同步模式会显著降低吞吐量，但有助于调试。

**可选值**：

- `1`：同步执行（每次核函数调用阻塞至完成）。

- `0`：异步执行（默认）。

**示例**：

```bash
HGGC_LAUNCH_BLOCKING=1
```

* * *

##### 5.3.2.2. `HGGC_DEVICE_MAX_CONNECTIONS` {#5322-hggc_device_max_connections}

控制并发计算和复制引擎连接（工作队列）数量，将两者都设置为指定值。

PPU 运行时将来自不同 HGGC 流的任务分配到有限数量的硬件工作队列中。当活跃流数量超过可用队列时，多个流会共享同一队列，导致原本无依赖的任务被串行执行（即产生**虚假依赖**）。增大此值可降低队列共享的概率，从而提升多流并行度。经验法则是将队列数设为不低于每个上下文中同时活跃的 HGGC 流数量。

**可选值**：`1` 到 `32`，默认 `8`（无 MPS 模式下）

**示例**：

```bash
HGGC_DEVICE_MAX_CONNECTIONS=16
```

* * *

#### 5.3.3. 模块加载 {#533-模块加载}

##### 5.3.3.1. `HGGC_MODULE_LOADING` {#5331-hggc_module_loading}

该环境变量影响 HGGC 运行时加载模块的方式，具体指如何初始化设备代码。

**可选值**：

- `DEFAULT`：默认行为，等效于 `1`。

- `1`：在程序初始化时完全加载 HGGC 模块和核函数。所有来自 HGBIN 或 FATBIN 文件的核函数和数据会在相应的 `hgModuleLoad*` 和 `hgLibraryLoad*` 驱动 API 调用时完全加载。启动时间较长， PPU 内存占用较高。核函数启动开销可预测。

- `2`：特定核函数的加载延迟到使用 `hgModuleGetFunction()` 或 `hgKernelGetFunction()` API 调用提取 HGGC 函数句柄 `HGfunc` 时。在这种情况下，HGBIN 中的数据会在加载 HGBIN 中第一个核函数或访问 HGBIN 中第一个变量时加载。驱动程序在首次调用核函数时加载所需代码；后续调用不会产生额外开销。这减少了启动时间和 PPU 内存占用。

**示例**：

```bash
HGGC_MODULE_LOADING=1
HGGC_MODULE_LOADING=2
```

* * *

#### 5.3.4. 图 {#533-图}

##### 5.3.4.1. `HGGC_DEVICE_GRAPH (实验特性)` {#5341-hggc_device_graph}

该环境变量会改变驱动实例化图的方式，并且影响后续提交到PPU上的命令队列的组织方式，会大幅加快图执行本身占用的时间，并且压缩图执行所占用的流数量，从而降低执行开销以及尽可能规避硬件队列上引入的隐式依赖。需要注意打开该环境变量在某些情况下可能会增加显存占用，用户可根据自己的实际需求打开该功能。

**可选值**：

- `0`：不使用Device Graph (默认行为)。

- `1`：有限的使用Device Graph, 图中只允许包含kernel/memcpy/memset/childGraph节点。

- `2`：尽可能使用Device Graph, 相比于设置为1, 图中还允许包含eventRecord/waitEvent节点。

**示例**：

```bash
HGGC_DEVICE_GRAPH=1
HGGC_DEVICE_GRAPH=2
```

### 5.4. 数学函数 {#54-数学函数}

#### 5.4.1. 浮点简介 {#541-浮点简介}

自从 1985 年采用 [IEEE-754 二进制浮点算术标准](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8766229) 以来，几乎所有主流计算系统都实现了该标准。IEEE-754 标准规定了浮点算术结果应如何近似。

为了获得准确的结果并以所需的精度实现最高性能，考虑浮点行为的许多方面非常重要。这在异构计算环境中尤为重要，因为操作是在不同类型的硬件上执行的。

##### 5.4.1.1. 浮点格式 {#5411-浮点格式}

浮点格式和功能在 [IEEE-754 标准](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8766229) 中定义。

该标准规定二进制浮点数据必须在三个字段上编码：

  * **符号**：一个位，用于指示正数或负数。

  * **指数**：以 2 为基数，用数值偏置编码的指数。

  * **尾数**（也称为 _mantissa_ 或 _fraction_）：编码数值的分数部分。

正常值的浮点编码关联的数值计算如下：

$$ (-1)^{\mathrm{sign}} \times 1.\mathrm{mantissa} \times 2^{\mathrm{exponent} - \mathrm{bias}} $$

对于非正规值，公式中缺少前导 `1`。

$$ (-1)^{\mathrm{sign}} \times 0.\mathrm{mantissa} \times 2^{\mathrm{1} - \mathrm{bias}} $$

单精度和双精度的指数分别偏置 $127$ 和 $1023$。分数的整数部分 $1.$ 是隐含的。

例如，值 $-192 = (-1)^1 \times 2^7 \times 1.5$，编码为负号、指数 $7$ 和分数部分 $0.5$。因此，指数 $7$ 对于 `float` 表示为位串值 `7 + 127 = 134 = 10000110`，对于 `double` 表示为 `7 + 1023 = 1030 = 10000111110`。尾数 `0.5 = 2^-1` 用二进制值表示，第一位为 `1`。

由于分数字段使用有限的位数，并非所有实数都能精确表示。例如，分数 $2 / 3$ 的数学值的二进制表示为 `0.10101010...`，在小数点后有无限位数。因此，$2 / 3$ 必须在表示为有限精度的浮点数之前进行舍入。IEEE-754 规定了舍入规则和模式。最常用的模式是 _向最近偶数舍入_。

##### 5.4.1.2. 正规值和非正规值 {#5412-正规值和非正规值}

任何可以用指数中至少一个位设置的浮点值都称为 _正规_ 值。

浮点正规值的一个重要方面是最小可表示非零浮点数 `FLT_MIN` 与零之间的巨大差距。这个差距比 `FLT_MIN` 与第二小可表示非零浮点数之间的差距大得多。

浮点 _非正规_ 数（也称为 _denormals_）的引入就是为了解决这个问题。非正规浮点值的指数中所有位都设置为零，尾数中至少有一个位设置。非正规数是 IEEE-754 浮点标准的必需部分。

非正规数允许精度逐渐损失，作为突然向零舍入的替代方案。然而，非正规数的计算成本更高。因此，不需要严格精度的应用程序可能会选择避免它们以提高性能。`hgcc` 编译器允许通过设置 `-ftz=true` 选项（刷新为零）来禁用非正规数，该选项也包含在 `--use_fast_math` 中。

##### 5.4.1.3. 特殊值 {#5413-特殊值}

IEEE-754 标准为浮点数定义了三种特殊值：

**零：**

  * 数学零。

  * 请注意，浮点零有两种可能的表示：`+0` 和 `-0`。这与整数零的表示不同。

  * `+0 == -0` 的计算结果为 `true`。

  * `0` 的编码中指数和尾数的所有位都设置为 `0`。

**无穷大：**

  * 浮点数的行为符合饱和算术，即超出可表示范围的运算结果为 `+Infinity` 或 `-Infinity`。

  * 无穷大的编码中指数所有位设置为 `1`，尾数所有位设置为 `0`。无穷大值恰好有两种编码。

  * 任何将有限数应用于无穷大的算术运算都会得到无穷大，除以零和乘以零除外，它们的结果为 NaN。

**非数字 (NaN)：**

  * NaN 是一个特殊符号，表示未定义或不可表示的值。常见示例包括 `0.0 / 0.0`、`sqrt(-1.0)` 或 `+Inf - Inf`。

  * NaN 的编码中指数所有位设置为 `1`，尾数中除全零外的任何位模式。有 $2^{\\mathrm{mantissa} + 1} - 2$ 种可能的编码。

  * 任何涉及 NaN 的算术运算都会得到 NaN。

  * 任何涉及 NaN 的比较运算都会得到 `false`，包括 `NaN == NaN`（非自反）。

##### 5.4.1.4. 舍入 {#5414-舍入}

IEEE-754 标准要求支持多种运算。这些包括算术运算，如加法、减法、乘法、除法、平方根、融合乘加、求余数、转换、缩放、符号和比较运算。对于给定的格式和舍入模式，这些运算的结果在所有标准实现中保证一致。

* * *

**舍入模式**

IEEE-754 标准定义了四种舍入模式：_向最近舍入_、_向正无穷舍入_、_向负无穷舍入_ 和 _向零舍入_。HGGC 支持所有四种模式。默认情况下，运算使用 _向最近舍入_。内建数学函数可用于为单个运算选择其他舍入模式。

| 舍入模式 | 解释                   |
| -------- | ---------------------- |
| `rn`     | 向最近舍入，舍入到偶数 |
| `rz`     | 向零舍入               |
| `ru`     | 向 $\infty$ 舍入       |
| `rd`     | 向 $-\infty$ 舍入      |

##### 5.4.1.5. 浮点数据类型 {#5415-浮点数据类型}

HGGC 支持 Bfloat16、半精度、单精度、双精度浮点数据类型。下表总结了 HGGC 中支持的浮点数据类型及其要求。

**表 4: 支持的浮点类型**
| 精度/名称 | 数据类型        | IEEE-754 | 头文件/内置     | 符号位 (Sign) | 指数位 (Exponent) | 尾数位 (Mantissa) | 总位宽 |
| --------- | --------------- | -------- | --------------- | ------------- | ----------------- | ----------------- | ------ |
| Bfloat16  | `__hg_bfloat16` | ❌        | `<hggc_bf16.h>` | 1             | 8                 | 7                 | 16     |
| 半精度    | `__half`        | ✅        | `<hggc_fp16.h>` | 1             | 5                 | 10                | 16     |
| 单精度    | `float`         | ✅        | 内置            | 1             | 8                 | 23                | 32     |
| 双精度    | `double`        | ✅        | 内置            | 1             | 11                | 52                | 64     |

**表 5 支持的浮点类型属性**
| 精度/名称 | 最大值             | 最小正值                     | 最小正非正规数 | 机器精度                       | 最小的正正则数 | 指数      |
| --------- | ------------------ | ---------------------------- | -------------- | ------------------------------ | -------------- | --------- |
| Bfloat16  | $\approx 2^{128}$  | $\approx 3.39 \cdot 10^{38}$ | $2^{-126}$     | $\approx 1.18 \cdot 10^{-38}$  | $2^{-133}$     | $2^{-7}$  |
| 半精度    | $\approx 2^{16}$   | $65504$                      | $2^{-14}$      | $\approx 6.1 \cdot 10^{-5}$    | $2^{-24}$      | $2^{-10}$ |
| 单精度    | $\approx 2^{128}$  | $\approx 3.39 \cdot 10^{38}$ | $2^{-126}$     | $\approx 1.18 \cdot 10^{-38}$  | $2^{-149}$     | $2^{-23}$ |
| 双精度    | $\approx 2^{1024}$ | $\approx 1.8 \cdot 10^{308}$ | $2^{-1022}$    | $\approx 2.22 \cdot 10^{-308}$ | $2^{-1074}$    | $2^{-52}$ |

#### 5.4.2. 函数精度 {#542-函数精度}

MATH 文档列出了在设备代码中支持的所有 C/C++ 标准库数学函数及其说明，以及所有内建函数（仅支持设备代码）。

下文对部分函数给出了以 **ULP**（Unit in the Last Place，最后位置单位）表示的精度界限。ULP 是衡量浮点近似与精确数学值之间距离的标准度量，其严格定义可参见 Jean-Michel Muller 的研究报告 *On the definition of ulp(x)*（RR-5504, LIP RR-2005-09, INRIA, LIP, 2005, pp.16），网址：https://hal.inria.fr/inria-00070503/document。

**与主机端 `math.h` 的行为差异**

设备端数学函数与主机端标准库在以下方面存在不同的行为约定：

| 差异项       | 主机端 (`math.h`)                                | 设备端 (HGGC)                                   |
| ------------ | ------------------------------------------------ | ----------------------------------------------- |
| 错误报告     | 通过 `errno` 或浮点异常通知调用方                | 既不设置 `errno` 也不触发异常；需调用方自行校验 |
| 指针参数校验 | 视实现而定，可能会做边界检查                     | 不做校验，调用方须确保指向合法已分配内存        |
| 未初始化参数 | 同属未定义行为（UB），现代编译器可能进行激进优化 | 内联优化可能利用未初始化值，导致不可预测结果    |

##### 5.4.2.1. 标准函数 {#5421-标准函数}

标准数学函数同时适用于主机代码和设备代码。下表中的误差界限针对设备端执行路径；当主机系统不提供某个函数的原生实现时，HGGC 库的实现也遵循相同的精度范围。

这些误差界限基于大规模随机和边界测试得出，覆盖面广但并非穷举式验证，因此应视为经验性上界而非数学证明。

###### 5.4.2.1.1. 单精度浮点函数 {#54211-单精度浮点函数}

基础算术（加法、乘法）严格遵循 IEEE-754 标准，误差不超过 0.5 ulp。

!!! tip "浮点取整性能提示"

    将单精度浮点操作数舍入为整数的推荐方法是使用 `rintf()` 而非 `roundf()`，结果仍然是单精度浮点数。`roundf()` 在设备上映射为 4 条指令序列，而 `rintf()` 映射为单条指令。`truncf()`、`ceilf()` 和 `floorf()` 也都映射为单条指令。

**表 6：单精度数学标准库函数的最大 ULP 误差**

最大误差表示为 HGGC 库函数返回的结果与根据向最近偶数舍入模式正确舍入的单精度结果之间以 ulp 为单位的差值的绝对值。

| 函数                   | 最大 ulp 误差                                                      |
| ---------------------- | ------------------------------------------------------------------ |
| x+y                    | 0 (IEEE-754 向最近偶数舍入)                                        |
| x*y                    | 0 (IEEE-754 向最近偶数舍入)                                        |
| x∕y                    | 当使用 -prec-div=true 编译时为 0<br> 否则，4 (全范围)              |
| 1∕x                    | 当使用 -prec-div=true 编译时为 0<br> 否则，1 (全范围)              |
| rsqrtf(x)              | 2 (全范围)                                                         |
| 1∕sqrtf(x)             | 仅适用于被编译器转换为 rsqrtf(x) 的 1∕sqrtf(x)。                   |
| sqrtf(x)               | 当使用 -prec-sqrt=true 编译时为 0<br> 否则，1                      |
| cbrtf(x)               | 1 (全范围)                                                         |
| rcbrtf(x)              | 1 (全范围)                                                         |
| hypotf(x,y)            | 3 (全范围)                                                         |
| rhypotf(x,y)           | 2 (全范围)                                                         |
| norm3df(x,y,z)         | 3 (全范围)                                                         |
| rnorm3df(x,y,z)        | 2 (全范围)                                                         |
| norm4df(x,y,z,t)       | 3 (全范围)                                                         |
| rnorm4df(x,y,z,t)      | 2 (全范围)                                                         |
| normf(dim,arr)         | 无法提供误差界限，因为使用了快速算法，由于舍入导致精度损失。       |
| rnormf(dim,arr)        | 无法提供误差界限，因为使用了快速算法，由于舍入导致精度损失。       |
| expf(x)                | 2 (全范围)                                                         |
| exp2f(x)               | 2 (全范围)                                                         |
| exp10f(x)              | 2 (全范围)                                                         |
| expm1f(x)              | 2 (全范围)                                                         |
| logf(x)                | 1 (全范围)                                                         |
| log2f(x)               | 1 (全范围)                                                         |
| log10f(x)              | 2 (全范围)                                                         |
| log1pf(x)              | 1 (全范围)                                                         |
| sinf(x)                | 2 (全范围)                                                         |
| cosf(x)                | 2 (全范围)                                                         |
| tanf(x)                | 4 (全范围)                                                         |
| sincosf(x,sptr,cptr)   | 2 (全范围)                                                         |
| sinpif(x)              | 1 (全范围)                                                         |
| cospif(x)              | 1 (全范围)                                                         |
| sincospif(x,sptr,cptr) | 1 (全范围)                                                         |
| asinf(x)               | 4 (全范围)                                                         |
| acosf(x)               | 4 (全范围)                                                         |
| atanf(x)               | 2 (全范围)                                                         |
| atan2f(y,x)            | 3 (全范围)                                                         |
| sinhf(x)               | 3 (全范围)                                                         |
| coshf(x)               | 2 (全范围)                                                         |
| tanhf(x)               | 3 (全范围)                                                         |
| asinhf(x)              | 3 (全范围)                                                         |
| acoshf(x)              | 4 (全范围)                                                         |
| atanhf(x)              | 3 (全范围)                                                         |
| powf(x,y)              | 10 (全范围)                                                        |
| erff(x)                | 2 (全范围)                                                         |
| erfcf(x)               | 5 (全范围)                                                         |
| erfinvf(x)             | 4 (全范围)                                                         |
| erfcinvf(x)            | 5 (全范围)                                                         |
| erfcxf(x)              | 4 (全范围)                                                         |
| normcdff(x)            | 5 (全范围)                                                         |
| normcdfinvf(x)         | 5 (全范围)                                                         |
| lgammaf(x)             | 6 (区间 -10.001… -2.264 之外；区间内更大)                          |
| tgammaf(x)             | 5 (全范围)                                                         |
| fmaf(x,y,z)            | 0 (全范围)                                                         |
| frexpf(x,exp)          | 0 (全范围)                                                         |
| ldexpf(x,exp)          | 0 (全范围)                                                         |
| scalbnf(x,n)           | 0 (全范围)                                                         |
| scalblnf(x,l)          | 0 (全范围)                                                         |
| logbf(x)               | 0 (全范围)                                                         |
| ilogbf(x)              | 0 (全范围)                                                         |
| j0f(x)                 | 9 当 \|x\| < 8 时<br> 否则，最大绝对误差为 2.2 x 10⁻⁶              |
| j1f(x)                 | 9 当 \|x\| < 8 时<br> 否则，最大绝对误差为 2.2 x 10⁻⁶              |
| jnf(n,x)               | 当 n = 128 时，最大绝对误差为 2.2 x 10⁻⁶                           |
| y0f(x)                 | 9 当 \|x\| < 8 时<br> 否则，最大绝对误差为 2.2 x 10⁻⁶              |
| y1f(x)                 | 9 当 \|x\| < 8 时<br> 否则，最大绝对误差为 2.2 x 10⁻⁶              |
| ynf(n,x)               | ceil(2 + 2.5n) 当 \|x\| < n 时<br> 否则，最大绝对误差为 2.2 x 10⁻⁶ |
| cyl_bessel_i0f(x)      | 8 (全范围)                                                         |
| cyl_bessel_i1f(x)      | 8 (全范围)                                                         |
| fmodf(x,y)             | 0 (全范围)                                                         |
| remainderf(x,y)        | 0 (全范围)                                                         |
| remquof(x,y,iptr)      | 0 (全范围)                                                         |
| modff(x,iptr)          | 0 (全范围)                                                         |
| fdimf(x,y)             | 0 (全范围)                                                         |
| truncf(x)              | 0 (全范围)                                                         |
| roundf(x)              | 0 (全范围)                                                         |
| rintf(x)               | 0 (全范围)                                                         |
| nearbyintf(x)          | 0 (全范围)                                                         |
| ceilf(x)               | 0 (全范围)                                                         |
| floorf(x)              | 0 (全范围)                                                         |
| lrintf(x)              | 0 (全范围)                                                         |
| lroundf(x)             | 0 (全范围)                                                         |
| llrintf(x)             | 0 (全范围)                                                         |
| llroundf(x)            | 0 (全范围)                                                         |

###### 5.4.2.1.2. 双精度浮点函数 {#54212-双精度浮点函数}

将双精度浮点操作数舍入为整数的推荐方法是使用 `rint()` 而非 `round()`，结果仍然是双精度浮点数。`round()` 在设备上映射为 5 条指令序列，而 `rint()` 映射为单条指令。`trunc()`、`ceil()` 和 `floor()` 也都映射为单条指令。

**表 7：双精度数学标准库函数的最大 ULP 误差**

最大误差表示为 HGGC 库函数返回的结果与根据向最近偶数舍入模式正确舍入的双精度结果之间以 ulp 为单位的差值的绝对值。

| 函数                  | 最大 ulp 误差                                                |
| --------------------- | ------------------------------------------------------------ |
| x+y                   | 0 (IEEE-754 向最近偶数舍入)                                  |
| x*y                   | 0 (IEEE-754 向最近偶数舍入)                                  |
| x∕y                   | 0 (IEEE-754 向最近偶数舍入)                                  |
| 1∕x                   | 0 (IEEE-754 向最近偶数舍入)                                  |
| sqrt(x)               | 0 (IEEE-754 向最近偶数舍入)                                  |
| rsqrt(x)              | 1 (全范围)                                                   |
| cbrt(x)               | 1 (全范围)                                                   |
| rcbrt(x)              | 1 (全范围)                                                   |
| hypot(x,y)            | 3 (全范围)                                                   |
| rhypot(x,y)           | 2 (全范围)                                                   |
| norm3d(x,y,z)         | 2 (全范围)                                                   |
| rnorm3d(x,y,z)        | 1 (全范围)                                                   |
| norm4d(x,y,z,t)       | 2 (全范围)                                                   |
| rnorm4d(x,y,z,t)      | 1 (全范围)                                                   |
| norm(dim,arr)         | 无法提供误差界限，因为使用了快速算法，由于舍入导致精度损失。 |
| rnorm(dim,arr)        | 无法提供误差界限，因为使用了快速算法，由于舍入导致精度损失。 |
| exp(x)                | 1 (全范围)                                                   |
| exp2(x)               | 1 (全范围)                                                   |
| exp10(x)              | 1 (全范围)                                                   |
| expm1(x)              | 1 (全范围)                                                   |
| log(x)                | 1 (全范围)                                                   |
| log2(x)               | 1 (全范围)                                                   |
| log10(x)              | 1 (全范围)                                                   |
| log1p(x)              | 1 (全范围)                                                   |
| sin(x)                | 2 (全范围)                                                   |
| cos(x)                | 2 (全范围)                                                   |
| tan(x)                | 2 (全范围)                                                   |
| sincos(x,sptr,cptr)   | 2 (全范围)                                                   |
| sinpi(x)              | 2 (全范围)                                                   |
| cospi(x)              | 2 (全范围)                                                   |
| sincospi(x,sptr,cptr) | 2 (全范围)                                                   |
| asin(x)               | 2 (全范围)                                                   |
| acos(x)               | 2 (全范围)                                                   |
| atan(x)               | 2 (全范围)                                                   |
| atan2(y,x)            | 2 (全范围)                                                   |
| sinh(x)               | 2 (全范围)                                                   |
| cosh(x)               | 2 (全范围)                                                   |
| tanh(x)               | 2 (全范围)                                                   |
| asinh(x)              | 3 (全范围)                                                   |
| acosh(x)              | 3 (全范围)                                                   |
| atanh(x)              | 2 (全范围)                                                   |
| pow(x,y)              | 2 (全范围)                                                   |
| erf(x)                | 2 (全范围)                                                   |
| erfc(x)               | 5 (全范围)                                                   |
| erfinv(x)             | 5 (全范围)                                                   |
| erfcinv(x)            | 7 (全范围)                                                   |
| erfcx(x)              | 6 (全范围)                                                   |
| normcdf(x)            | 5 (全范围)                                                   |
| normcdfinv(x)         | 9 (全范围)                                                   |
| lgamma(x)             | 4 (区间 -23.0001… -2.2637 之外；区间内更大)                  |
| tgamma(x)             | 10 (全范围)                                                  |
| fma(x,y,z)            | 0 (IEEE-754 向最近偶数舍入)                                  |
| frexp(x,exp)          | 0 (全范围)                                                   |
| ldexp(x,exp)          | 0 (全范围)                                                   |
| scalbn(x,n)           | 0 (全范围)                                                   |
| scalbln(x,l)          | 0 (全范围)                                                   |
| logb(x)               | 0 (全范围)                                                   |
| ilogb(x)              | 0 (全范围)                                                   |
| j0(x)                 | 7 当 \|x\| < 8 时<br> 否则，最大绝对误差为 5 x 10⁻¹²         |
| j1(x)                 | 8 当 \|x\| < 8 时<br> 否则，最大绝对误差为 5 x 10⁻¹²         |
| jn(n,x)               | 当 n = 128 时，最大绝对误差为 5 x 10⁻¹²                      |
| y0(x)                 | 7 当 \|x\| < 8 时<br> 否则，最大绝对误差为 5 x 10⁻¹²         |
| y1(x)                 | 7 当 \|x\| < 8 时<br> 否则，最大绝对误差为 5 x 10⁻¹²         |
| yn(n,x)               | 当 \|x\| > 1.5n 时，最大绝对误差为 5 x 10⁻¹²                 |
| cyl_bessel_i0(x)      | 8 (全范围)                                                   |
| cyl_bessel_i1(x)      | 8 (全范围)                                                   |
| fmod(x,y)             | 0 (全范围)                                                   |
| remainder(x,y)        | 0 (全范围)                                                   |
| remquo(x,y,iptr)      | 0 (全范围)                                                   |
| modf(x,iptr)          | 0 (全范围)                                                   |
| fdim(x,y)             | 0 (全范围)                                                   |
| trunc(x)              | 0 (全范围)                                                   |
| round(x)              | 0 (全范围)                                                   |
| rint(x)               | 0 (全范围)                                                   |
| nearbyint(x)          | 0 (全范围)                                                   |
| ceil(x)               | 0 (全范围)                                                   |
| floor(x)              | 0 (全范围)                                                   |
| lrint(x)              | 0 (全范围)                                                   |
| lround(x)             | 0 (全范围)                                                   |
| llrint(x)             | 0 (全范围)                                                   |
| llround(x)            | 0 (全范围)                                                   |


##### 5.4.2.2. 内建函数 {#5422-内建函数}

内建数学函数仅可在设备代码中调用。

HGGC 为部分标准数学函数提供了以 `__` 为前缀的内建版本（如 `__sinf(x)` 对应 `sinf(x)`）。内建版本通过减少原生指令数量换取更高的执行速度，代价是精度有所降低。编译选项 `-use_fast_math` 会全局地将下表所列函数替换为内建版本。需注意，这不仅影响精度，还可能改变边界条件（如 NaN、无穷大）的处理方式。

在性能敏感的代码中，建议逐个函数评估是否值得切换到内建版本，而非全局启用 `-use_fast_math`，以便在速度和数值稳定性之间取得适当平衡。

**表 8：受 -use_fast_math 影响的函数**

| 运算符/函数          | 设备函数               |
| -------------------- | ---------------------- |
| x∕y                  | __fdividef(x,y)        |
| sinf(x)              | __sinf(x)              |
| cosf(x)              | __cosf(x)              |
| tanf(x)              | __tanf(x)              |
| sincosf(x,sptr,cptr) | __sincosf(x,sptr,cptr) |
| logf(x)              | __logf(x)              |
| log2f(x)             | __log2f(x)             |
| log10f(x)            | __log10f(x)            |
| expf(x)              | __expf(x)              |
| exp10f(x)            | __exp10f(x)            |
| powf(x,y)            | __powf(x,y)            |
| tanhf(x)             | __tanhf(x)             |

###### 5.4.2.2.1. 单精度浮点函数 {#54221-单精度浮点函数}

`__fadd_[rn,rz,ru,rd]()` 和 `__fmul_[rn,rz,ru,rd]()` 映射到编译器从不合并为 FMAD 的加法和乘法运算。相比之下，由 '*' 和 '+' 运算符生成的加法和乘法经常会被合并为 FMAD。

带有 _rn 后缀的函数使用向最近偶数舍入模式运行。

带有 _rz 后缀的函数使用向零舍入模式运行。

带有 _ru 后缀的函数使用向上舍入（向正无穷大）模式运行。

带有 _rd 后缀的函数使用向下舍入（向负无穷大）模式运行。

浮点除法的精度取决于代码是使用 -prec-div=false 还是 -prec-div=true 编译的。当代码使用 -prec-div=false 编译时，常规除法 ∕ 运算符和 __fdividef(x,y) 具有相同的精度，但对于 2¹²⁶ < \|y\| < 2¹²⁸，__fdividef(x,y) 返回零结果，而 ∕ 运算符在表 21 所述的精度范围内返回正确结果。此外，对于 2¹²⁶ < \|y\| < 2¹²⁸，如果 x 是无穷大，__fdividef(x,y) 返回 NaN（由于无穷大乘以零的结果），而 ∕ 运算符返回无穷大。另一方面，当代码使用 -prec-div=true 编译或不带任何 -prec-div 选项编译时（因为其默认值为 true），∕ 运算符符合 IEEE 标准。

**表 9：单精度浮点内建函数**

（由 HGGC 运行时库支持，具有各自的误差界限）

| 函数                          | 误差界限                                                                                                   |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `__fadd_[rn,rz,ru,rd](x%2Cy)`   | 符合 IEEE 标准。                                                                                           |
| `__fsub_[rn,rz,ru,rd](x%2Cy)`   | 符合 IEEE 标准。                                                                                           |
| `__fmul_[rn,rz,ru,rd](x%2Cy)`   | 符合 IEEE 标准。                                                                                           |
| `__fmaf_[rn,rz,ru,rd](x%2Cy%2Cz)` | 符合 IEEE 标准。                                                                                           |
| `__frcp_[rn,rz,ru,rd](x)`     | 符合 IEEE 标准。                                                                                           |
| `__fsqrt_[rn,rz,ru,rd](x)`    | 符合 IEEE 标准。                                                                                           |
| `__frsqrt_rn(x)`              | 符合 IEEE 标准。                                                                                           |
| `__fdiv_[rn,rz,ru,rd](x%2Cy)`   | 符合 IEEE 标准。                                                                                           |
| `__fdividef(x,y)`             | 对于 \|y\| 在 [2⁻¹²⁶, 2¹²⁶] 范围内，最大 ulp 误差为 2。                                                    |
| `__expf(x)`                   | 最大 ulp 误差为 2 + floor(abs(1.173 * x))。                                                                |
| `__exp10f(x)`                 | 最大 ulp 误差为 2 + floor(abs(2.97 * x))。                                                                 |
| `__logf(x)`                   | 对于 x 在 [0.5, 2] 范围内，最大绝对误差为 2⁻²¹·⁴¹，否则，最大 ulp 误差为 3。                               |
| `__log2f(x)`                  | 对于 x 在 [0.5, 2] 范围内，最大绝对误差为 2⁻²²，否则，最大 ulp 误差为 2。                                  |
| `__log10f(x)`                 | 对于 x 在 [0.5, 2] 范围内，最大绝对误差为 2⁻²⁴，否则，最大 ulp 误差为 3。                                  |
| `__sinf(x)`                   | 对于 x 在 [−π, π] 范围内，最大绝对误差为 2⁻²¹·⁴¹，否则更大。                                               |
| `__cosf(x)`                   | 对于 x 在 [−π, π] 范围内，最大绝对误差为 2⁻²¹·¹⁹，否则更大。                                               |
| `__sincosf(x,sptr,cptr)`      | 与 `__sinf(x)` 和 `__cosf(x)` 相同。                                                                       |
| `__tanf(x)`                   | 源自其实现为 `__sinf(x) * (1/__cosf(x))`。                                                                 |
| `__powf(x, y)`                | 源自其实现为 `exp2f(y * __log2f(x))`。                                                                     |
| `__tanhf(x)`                  | 当前实现的最大相对误差为 2⁻¹¹。即使在 -ftz=true 编译器设置下，此快速内建函数的非正规结果也不会被刷新为零。 |

###### 5.4.2.2.2. 双精度浮点函数 {#54222-双精度浮点函数}

__dadd_rn() 和 __dmul_rn() 映射到编译器从不合并为 FMAD 的加法和乘法运算。相比之下，由 '*' 和 '+' 运算符生成的加法和乘法经常会被合并为 FMAD。

**表 10：双精度浮点内建函数**

（由 HGGC 运行时库支持，具有各自的误差界限）

| 函数                         | 误差界限         |
| ---------------------------- | ---------------- |
| `__dadd_[rn,rz,ru,rd](x%2Cy)`  | 符合 IEEE 标准。 |
| `__dsub_[rn,rz,ru,rd](x%2Cy)`  | 符合 IEEE 标准。 |
| `__dmul_[rn,rz,ru,rd](x%2Cy)`  | 符合 IEEE 标准。 |
| `__fma_[rn,rz,ru,rd](x%2Cy%2Cz)` | 符合 IEEE 标准。 |
| `__ddiv_[rn,rz,ru,rd](x%2Cy)`  | 符合 IEEE 标准。 |
| `__drcp_[rn,rz,ru,rd](x)`    | 符合 IEEE 标准。 |
| `__dsqrt_[rn,rz,ru,rd](x)`   | 符合 IEEE 标准。 |


### 5.5. 计算能力 {#55-计算能力}

PPU 的计算能力与实际的计算架构（ppu001/ppu0015）相关。

#### 5.5.1. 查询 PPU 的计算能力 {#551-查询ppu的计算能力}

可以通过 `ppu-smi` 实现 memory 等信息的查询。

```bash
ppu-smi --query-ppu=timestamp,index,name,compute_mode,memory.total,memory.used --format=csv
```


在运行时，可以通过 HGGC 运行时 API 的 `hggcDeviceGetAttribute()` 来进行查询。

```cpp
#include <hggc_runtime_api.h>

int computeCapabilityMajor, computeCapabilityMinor;
hggcDeviceGetAttribute(&computeCapabilityMajor, hggcDevAttrComputeCapabilityMajor, device_id);
hggcDeviceGetAttribute(&computeCapabilityMinor, hggcDevAttrComputeCapabilityMinor, device_id);


#include <hggc.h>

int computeCapabilityMajor, computeCapabilityMinor;
hggcDeviceGetAttribute(&computeCapabilityMajor, HG_DEVICE_ATTRIBUTE_COMPUTE_CAPABILITY_MAJOR, device_id);
hggcDeviceGetAttribute(&computeCapabilityMinor, HG_DEVICE_ATTRIBUTE_COMPUTE_CAPABILITY_MINOR, device_id);
```

#### 5.5.2. 可用特性 {#552-可用特性}
下表列出了编程时线程束（warp）/线程块（block）等的限制。

|                            | ppu001                             | ppu0015                            |
| -------------------------- | ---------------------------------- | ---------------------------------- |
| 线程束 warp 大小           | 32                                 | 32                                 |
| 最大线程数/CU              | 2048                               | 2048                               |
| 最大线程数/block           | 1024                               | 1024                               |
| 单 block 最大 (x,y,z) 范围 | (1024,1024,128)                    | (1024,1024,128)                    |
| 单 grid 最大 (x,y,z) 范围  | (2147483647,2147483647,2147483647) | (2147483647,2147483647,2147483647) |

在现代计算系统中，有效利用内存与最大化执行计算的功能单元的使用同样重要。异构系统有多重内存空间， PPU 还包含各种类型的可编程片上内存以及缓存。全局内存容量与硬件 ARCH 和产品信息相关，下表列出了除全局内存外的其他各级片上内存容量信息。

|                          | ppu001 | ppu0015 |
| ------------------------ | ------ | ------- |
| VREG 总量/CU             | 64K    | 64K     |
| 单 warp 最大 VREG 使用量 | 255    | 255     |
| 共享内存容量             | 256K   | 256K    |
| 共享内存 bank 数         | 32     | 32      |
| L1 缓存支持              | 是     | 是      |
| L2 缓存支持              | 是     | 是      |
| 常量内存支持             | 否     | 否      |
| LLC 缓存支持             | 是     | 是      |

共享内存是一种高速缓存内存，由同一线程块内的所有线程共享。由于共享内存比全局内存快得多，因此编程时经常使用，下表列出了不同 ARCH 上共享内存的信息。
| PPU ARCH | 共享内存容量 (KB) | 共享内存分组 |
| -------- | ----------------- | ------------ |
| ppu001   | 256 KB            | 2            |
| ppu0015  | 256 KB            | 4            |


使用 Tensor Cell 能更好利用整个 PPU 的计算能力。Tensor Cell 的功能可通过内联 TIX 的方式，在 HGGC 编译工具链中加以利用。下表列出了 Tensor Cell 加速所支持的输入数据类型。

|         | TF32 | F32 | BF16 | FP16 | FP8 | FP4 | INT8 |
| ------- | ---- | --- | ---- | ---- | --- | --- | ---- |
| ppu001  | Yes  | Yes | Yes  | Yes  |     |     | Yes  |
| ppu0015 | Yes  | Yes | Yes  | Yes  | Yes | Yes | Yes  |
