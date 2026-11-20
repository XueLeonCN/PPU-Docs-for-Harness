---
source_url: "https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=48"
source_api_url: "https://developer.t-head.cn/api/developer/document/project/chapters?projectId=48"
source_title: "平头哥真武 M890 架构白皮书"
source_id: "project-48"
source_kind: "architecture-whitepaper"
source_date: "2026-09-16"
source_version: "T-Head SAIL SDK v2.2.0"
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "829ad6247023751e7ac507fbc1e92f241000a835af34d993251eab85bece8eb7"
tags:
  - "M890"
  - "architecture"
  - "GPGPU"
  - "CE"
  - "CU"
  - "TFU"
  - "AIU"
  - "TSM"
  - "cache"
  - "PCIe"
  - "video"
  - "ICN"
  - "SAIL"
---
# 平头哥真武 M890 架构简介

## 版本历史

| 版本 | 说明 | 日期 |
| :---: | :---: | :---: |
| V1.0 | 产品发布 | 2026-09-16 |

## 目录

- 版本历史
- 图表目录
- 术语与缩略语
- 1.&nbsp;概述
- 2.&nbsp;真武 M890 架构总览
- 3.&nbsp;真武 M890 创新特性
  - 3.1.&nbsp;39 个并行计算单元提供充足算力
  - 3.2.&nbsp;支持 ECC
  - 3.3.&nbsp;3 级缓存架构
  - 3.4.&nbsp;AI 数据移动加速
  - 3.5.&nbsp;大型片上用户管理内存
  - 3.6.&nbsp;PCIe 5.0 接口
  - 3.7.&nbsp;数据中心的虚拟化支持
- 4.&nbsp;真武 M890 计算微架构（CE）
- 5.&nbsp;AI 加速
- 6.&nbsp;视频流处理
- 7.&nbsp;芯片间互联网络（ICN）
- 8.&nbsp;完整软件栈
  - 8.1.&nbsp;驱动
  - 8.2.&nbsp;编译器
  - 8.3.&nbsp;计算/通信库
  - 8.4.&nbsp;视频编解码器
  - 8.5.&nbsp;调试和分析工具
    - 8.5.1.&nbsp;PPU-SMI
    - 8.5.2.&nbsp;Asight Systems
    - 8.5.3.&nbsp;Asight Compute
    - 8.5.4.&nbsp;PPU-GDB
  - 8.6.&nbsp;推理引擎
- 9.&nbsp;结语

## 图表目录

- 图表 2-1 真武 M890 芯片架构示意
- 图表 2-2 真武 M890 硬件配置规格
- 图表 7-1 基于环的芯片间网络（OAM8）
- 图表 7-2 ICN64 配置，每个真武 M890 都能以 800GB/s 的速率与任意真武 M890 进行通信
- 图表 7-3 多达 1024 个真武 M890 芯片通过两级 ICN Switch 进行互联
- 图表 8-1 T-HEAD SAIL 软件栈

## 术语与缩略语

|**缩略语**|**英文全名**|**中文解释**|
| :---: | :---: | :---: |
|AI|Artificial Intelligence|人工智能|
|AIU|AI accelerator in CU|AI 数据加速器|
|BLAS|Basic Linear Algebra Subroutine|基础线性代数子程序库|
|CE|Compute Engine|计算引擎|
|CI|Compute Instance|计算实例|
|CP|Command Processor|命令处理单元|
|CSP|Cloud Service Provider|云服务提供商|
|CU|Compute Unit|计算单元|
|DDR|Double Data Rate|双倍数据速率|
|DP|Dot Product|点积|
|FMA|Fused Multiply-Add|融合乘加运算|
|GD|Global Dispatch|全局线程分发|
|GPGPU|General Purpose GPU|通用图形处理器|
|IALU|Integer ALU|整数算术逻辑单元|
|LLC|Last Level Cache|最后一级缓存|
|LSU|Load/Store Unit|存取单元|
|MMA|Matrix Multiply-Add|矩阵乘加运算|
|MPS|Multi-Process Service|多进程服务|
|QoS|Quality of Service|服务质量|
|SALU|Scalar ALU|标量算术逻辑单元|
|SFU|Special Function Unit|特殊功能单元|
|SMI|System Management Interface|系统管理接口|
|SoC|System on Chip|系统级芯片|
|TFU|Tensor Function Unit|张量计算核|
|TSM|Thread block Shared Memory|线程块共享存储|
|VALU|Vector ALU，包括 Integer ALU (IALU) 和 Floating-Point ALU (FALU)|向量算术逻辑单元|

## 1. 概述

通用图形处理器（GPGPU）作为大模型时代的算力引擎，已经广泛部署于智能数据中心，用于加速以人工智能（AI）为代表的各种高并行度、大规模工作负载。GPGPU 在 AI 领域的广泛采用使得训练万亿（Trillion）参数的大模型成为可能，同时也为高并发、实时性的大模型推理业务提供了必不可少的部署平台。为此，平头哥推出了具有自主知识产权的第二代 GPGPU 架构真武 M890。

平头哥真武 M890 架构的设计目标是为当前数据中心的主流 AI 训练和推理应用提供（1）具有竞争力的性能表现和（2）在编程使用上兼容主流 AI 生态的 GPGPU 产品。真武 M890 架构支持标量、向量和张量三类流水线，提供了完备的通用处理器指令集，可满足各种计算场景。并且，为了实现极致 AI 性能优化，真武 M890 架构还包括了一系列的创新特性，如张量数据读取加速、线程块共享内存访问合并、全局线程调度优化等。

平头哥为真武 M890 芯片的可扩展性提供了充足的硬件支撑。真武 M890 的芯片间互联网络（ICN）可为芯片间通信提供 800GB/s 的双向带宽，支持原生读写指令以及基于 DMA 的片间数据搬移。用户可以通过多种方式构建算力集群：（1）在同一个节点（Node）内的 8 个真武 M890 芯片可以通过直连 ICN 进行高速低延迟的通信。（2）通过平头哥自主研发的 ICN Switch 芯片，多个真武 M890 芯片之间可以实现全相连拓扑，例如 ICN8 或 ICN64 的产品所提供的；（3）进一步地，通过二级 ICN Switch 网络，多达 1024 个真武 M890 芯片可以互联形成一个 Exascale AI 计算引擎。

软件平台是芯片和用户应用之间必不可少的桥接。平头哥提供了完整的 T-HEAD SAIL SDK，为用户基于真武 M890 芯片的开发全过程保驾护航。T-HEAD SAIL SDK 提供运行时、驱动、编译器、调试器、性能分析工具和计算库（如 acBLAS、acDNN、PCCL、视频 SDK 等），从而使得用户能够轻松高效地开发真武 M890 应用程序。

## 2. 真武 M890 架构总览

真武 M890 是平头哥针对大规模智能数据中心的第二代 GPGPU 架构。真武 M890 的设计目标是满足 AI 训练和推理工作负载对计算能力的迫切需求，多达 1024 个真武 M890 处理器可以通过 ICN 连接在一起，形成一台 Exascale 算力的 AI 加速引擎。

如图表 2-1 所示，真武 M890 架构包含众多功能模块，其中包括：

- 10 个计算引擎（CE），每个 CE 有 4 个计算单元（CU）。CU 支持多种数据类型，包括 FP8, FP4 (with micro-scales), INT8, INT16、FP16、BF16、INT32、FP32，使得真武 M890 适用于不同数据精度要求下的 AI 训练和推理任务。

- CU 中的 L1，CE 中的 L2，以及和 Memory Controller（MC）相连的 LLC 是真武 M890 的数据缓存单元，他们利用数据局部性（Locality）提升计算性能。

- 真武 M890 中实现了 4 个视频解码器、2 个视频编码器和 4 个独立的图像解码器，以适应各种格式和类型的 AI 工作负载所需的高吞吐量数据流。

- 真武 M890 芯片通过 PCIe gen5.0 x16 接口连接到主机 CPU，通过 ICN 链路连接到其他真武 M890 芯片。ICN 提供 8 个高速端口，每个端口提供 100GB/s 的双向带宽。

- 所有 CE、视频编解码器和 ICN 都连接到高速真武 M890 片上互联网络（Fabric），他们的读写请求将被 Fabric 路由到 LLC。若 LLC 中没有对应数据，则会 MC 访问内存。

- CP、GD 用于真武 M890 任务调度。CP 是 host CPU 向真武 M890 下发任务的软硬件接口。GD 用于把线程块（Thread Block）分发到 CE 和 CU。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790766739069/37ca39e8b85560d9a9b6921f0ef1a947/fig2-1-m890-chip-arch.png)

图表 2-1 真武 M890 芯片架构示意

平头哥为基于真武 M890 的产品提供了 PCIe 和 OAM（OCP 加速模块）两种板型。图表 2-2 显示了真武 M890 在 OAM 和 PCIe 板型上的硬件配置规格。

图表 2-2 真武 M890 硬件配置规格

| 特性 | 真武 M890 OAM | 真武 M530 PCIe |
| :---: | :---: | :---: |
| CEs/M890 | 10 | 8 |
| CUs/M890 | 39 | 30 |
| CUs/CE | 4 | 4 |
| TSM 大小/CU | 256KiB | 256KiB |
| TSM 大小/M890 | 9.75MiB | 7.5MiB |
| 编码器 | 2 | 0 |
| 解码器 | 4 | 4 |
| ICN 端口/M890 | 8 | 8 |

## 3. 真武 M890 创新特性

### 3.1. 39 个并行计算单元提供充足算力

真武 M890 全芯片设计有 10 个 CE，每个 CE 配置有 4 个 CU，总共 39 个 CU。

### 3.2. 支持 ECC

配备大容量内存，满足大多数训练场景中数据集和模型权重，以及推理过程中长序列场景下 KVCache 的存储需求。

### 3.3. 3 级缓存架构

真武 M890 中的三级缓存结构为每个 CU 内有 L1 一级数据缓存；此外，每个 CE 都有 L2 二级缓存，由 4 个 CU 共享；最后一级高速缓存（LLC）由 10 个 CE 共享。大型 3 级片上缓存与编译器优化的高效缓存策略相结合，可以过滤大多数内存访问，减少数据移动开销和功耗。

### 3.4. AI 数据移动加速

专用的 AI 数据移动单元（即 AIU）用于从全局内存读取复杂多维矩阵/张量数据块到片上内存。通过将地址计算和数据加载卸载到 AIU 上，使得 ALU 资源可以专注于计算而非访存。AIU 还为卷积应用定制了硬件加速功能。

### 3.5. 大型片上用户管理内存

总计 9.75MB 的 CU 局部内存（即 TSM）使用户能够使用 AIU 和异步张量读取功能轻松隐藏较长的矩阵/张量加载延迟，这对于充分利用张量核的巨大吞吐量至关重要。

### 3.6. PCIe 5.0 接口

真武 M890 通过 PCIe 5.0 x16 接口连接到主机 CPU，为 CPU 与真武 M890 之间提供高速的数据传输通道。

### 3.7. 数据中心的虚拟化支持

真武 M890 支持多进程服务（MPS），最多支持 7 个用户进程独立地创建 M890 上下文（Context），并按需划分 CE 资源，从而支持在真武 M890 上灵活高效地部署多个并行的用户应用。

## 4. 真武 M890 计算微架构（CE）

真武 M890 的 CE 包括 4 个 CU 和一个 L2 缓存。当线程块到达 CE 时，会被进一步分解为线程束，并分派到相应的 CU 进行运算。

- CE 中包含一个由 4 个 CU 共享的 L2 缓存。在典型的 AI 应用中，线程块之间往往存在频繁的数据共享，这个 L2 可以提供线程块之间的数据共享，避免重复从更远的 LLC 或内存中读取数据，减轻 Fabric 的访存压力。

- 每个 CU 有一个 L1 缓存和一个线程块共享内存（即 TSM），TSM 用于线程块范围内的线程间数据共享、通信和同步。

- CU 中的存储资源还包括标量和向量寄存器（SREG、VREG）。

- CU 是由平头哥完全自主设计的处理器核心，其中指令执行的起点是线程调度器：它从指令缓存读取指令，并根据指令类型分发到向量运算单元（VALU）、张量运算单元（TFU）、标量运算单元（SALU）、特殊函数单元（SFU）以及读写指令单元（LSU）中的某一种。真武 M890 使用了多种优化技术以提高线程级并行（TLP）和指令级并行（ILP）水平，允许同时执行 IALU、FALU、LSU、SFU 和 TFU。

- 每个 CU 包括 4 个 LSU 单元，VALU 和 SFU 的比例是 4：1，对于 FP16/BF16 输入数据格式，Tensor 算力和 FP32 VALU 的算力比例是 32：1。

- 真武 M890 支持单指令多线程的编程模式（SIMT）。

## 5. AI 加速

真武 M890 的 TFU 支持多种精度，以满足各种 AI 模型对计算精度的需求。TFU 的基本组成部分是点积（DP）单位。DP 单元的排列方式使得输入数据可以在它们之间实现共享，而无需对同一个寄存器进行多次访问。此外，允许输入矩阵 A 和 B 具有与累积矩阵 C 和输出矩阵 D 不同的精度。

为进一步解决内存墙问题，真武 M890 AI 加速单元（AIU）实现了异步张量读取功能，即在没有 warp 调度器和 LSU 的参与下，用单条 AIU 指令将待运算的多维张量从全局 Memory 搬移到局部 TSM。AIU 根据 AIU 指令中包含的信息，如源多维矩阵/张量内存基址和布局、TSM 基址、张量起始位置、内存布局等，自动计算出要获取的每一个数据元素的地址。AIU 异步张量读取功能可以实现 TFU 指令的运算功能和数据加载的软件流水（Software Pipelining），大大提高了张量核的实际利用效率，根据平头哥对真武 M890 芯片的实际测试，TFU 的利用率在 GEMM 程序中可以高达 95%（M=1280, N=2048）。

## 6. 视频流处理

基于视频的 AI 模型正受到越来越多的关注。为了达到最高的 AI 模型训练和推理的吞吐量，需要保证向 AI 模型提供的原始视频和图像数据不断流。真武 M890 能够解码各种标准的视频和图像流，包括 H.265、H264、VP9、VP8、WEBP、MPEG-2、JPEG、MJPEG、AV1、AVS、AVS+、AVS2 等。真武 M890 芯片具有 4 个视频解码器核，支持 8K 30FPS 和 4K 120FPS 的解码速率，匹配大多数 AI 模型训练和推理场景。还有 4 个用于图像的独立 JPEG 解码器核。JPEG 解码器核支持 YUV444、YUV422、YUV420 和 YUV400 输入，最大图像分辨率为 32Kx32K；此外，JPEG 解码器核还可以解码 MJPEG 视频，其性能为 4K@120FPS。

真武 M890 还具有视频生成 AI 工作负载的编码能力，例如 AI 直播、从文本生成智能视频和人脸生成器。真武 M890 可以压缩任何标准的原始视频流，如支持 H.265、H264、JPEG、AV1 等。真武 M890 的 2 个编码器核共支持 16x1080P@30FPS，它的引入不仅可以节省大量的 CPU 计算资源，而且可以大大减少真武 M890 和 CPU 之间的数据传输量。

## 7. 芯片间互联网络（ICN）

现代的 AI 模型的参数量已增长到万亿规模（1 Trillion），其计算量和内存容量快速增长，需要大规模的集群计算能力。因此，GPGPU 芯片间高速、高效、可扩展的芯片间通信至关重要。

ICN，即芯片间网络，是平头哥自主研发的第二代高带宽、低延迟、无损的 M890-to-M890 互连，它支持内存读写语义以及 DMA 搬运，实现了 Packet 纠错和重传机制，能够保证点对点的传输可靠性。

每个真武 M890 芯片有 8 个独立的 ICN 链路/端口。每个端口提供 100GB/s 的双向带宽，8 个端口共计提供 800GB/s 的总带宽用于对等设备和共享内存访问，因而大大降低了 AI 训练和推理中通信的开销，帮助用户实现算力的强扩展性（Strong Scaling）。

目前，真武 M890 提供基于（1）ICN Link 直连和（2）ICN Switch 组网的两种产品形态：

1. ICN Link 直连

- Scale-UP: 真武 M890 芯片（OAM 或 PCIE 版本）之间通过 ICN 直连。如图表 7-1 基于环的芯片间网络（OAM8）所示，四种不同的颜色的环型连接使得 OAM8 中的每个真武 M890 芯片都能以 P2P 的方式读/写任意一个真武 M890 芯片，多 Hop 的访问 Packet 会被中间经过的真武 M890 转发，因而确保了大数据量集合通信的性能能以 800GB/s 的满速率进行。这种配置具有一定非对称性，具体体现在访问不同真武 M890 会有延迟的差异。但从图中可以看出，从每一个真武 M890 出发访问任意一个真武 M890 只需要经过最多 1 次中转，因而这种方式也保证了延迟敏感场景下通信任务的性能。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790766739231/dd29ba89230febcf7ad3cc0221f01599/fig7-1-ring-network-oam8.png)

图表 7-1 基于环的芯片间网络（OAM8）

- Scale-Out: OAM8 系统支持网卡对真武 M890 的直接内存访问，因而每个真武 M890 可以通过 PCIe Switch 与 RDMA/Ethernet 网卡连接，无需大量 CPU 资源开销。通过这种方式，可以构建更大规模的真武 M890 计算系统。其局限在于 PCIe 带宽成为了 Scale-Out 的速率瓶颈。

2. 基于 ICN Switch 组网

- 通过将真武 M890 与平头哥自主研发的 ICN Switch 进行互联（ICN Switch 支持与真武 M890 相同的 ICN 通信协议），可以构建出 64 卡真武 M890 互联的 Rack-Scale 产品，从而为用户提供充足的 AI 计算能力。

- 如图表 7-2 ICN64 配置，每个真武 M890 都能以 800GB/s 的速率与任意真武 M890 进行通信所示，以 ICN64 产品为例，该 ICN Switch 网络采用全对称的单级交换结构，任意两个真武 M890 之间的通信都经过 M890-ICN Switch-M890 的路径。ICN64 消除了 OAM8 中访问不同真武 M890 的延迟差异，因而能够为真武 M890 的用户带来更好的性能表现和更简洁的应用设计。此外，这种结合了 ICN Switch 的产品在多租户场景下能够在每个租户的真武 M890 Partition 内部以完整的 800GB/s 的带宽进行通信，而不会造成带宽损失。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790766739387/4e320f148b920a48f66cfd1e2709a9e3/fig7-2-icn64-topology.png)

图表 7-2 ICN64 配置，每个真武 M890 都能以 800GB/s 的速率与任意真武 M890 进行通信

3. ICN 1024

- 如图表 7-3 多达 1024 所示，通过二级 ICN Switch 互联网络，多达 1024 个真武 M890 芯片可以通过 ICN 进行高速互联。其中每个真武 M890 集群（最多支持 32 卡真武 M890）内部通过一级 ICN Switch 组成全相连结构，所有的一级 ICN Switch 再通过二级 ICN Switch 组成分通道的全相连结构。因此，每个真武 M890 可以通过三次跳转，访问整个系统中的任意一个真武 M890。此外，ICN 在从一级 ICN Switch 到二级 ICN Switch 的路径选择上提供了适应性路由（Adaptive Routing）的功能，为流量均衡问题提供调优选项。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790766739549/9db8c6ac0f6ecfd3a3cef1cbe60328e5/fig7-3-icn1024-topology.png)

图表 7-3 多达 1024 个真武 M890 芯片通过两级 ICN Switch 进行互联

## 8. 完整软件栈

软件生态系统对于真武 M890 产品的开发、调试和管理能力而言至关重要。平头哥 AI 软件栈的名称为 T-HEAD SAIL（Seed of AI Libraries），具有自主可控的产权，具备统一的编程接口，可以端到端支持业务落地和扩展。

T-HEAD SAIL 主要由驱动、运行时、各种库、工具、编译器工具链和推理引擎等组成，软件架构如图 8-1 所示。其主要特征包括：

- 友好的用户编程界面，支持用户定制算子开发。

- 高性能计算库可以充分发挥真武 M890 的计算能力，达到行业领先水平。

- 支持通过 ICN 高速协议构建大规模训练集群，具有优秀的计算性能和通信带宽。

- 基于 MLIR 生态的高性能推理引擎，为用户提供优秀的推理性能。

- 丰富的设备监控、性能分析和调试工具，使用户能够轻松调试应用程序并优化性能。

![](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790766739714/cf876e3dc94f51d6f8c770942f1cd0bc/fig8-1-sail-software-stack.png)

图表 8-1 T-HEAD SAIL 软件栈

### 8.1. 驱动

T-HEAD SAIL 驱动包括固件、内核驱动、用户驱动和运行时库。

- 固件运行在真武 M890 内部 MCU 上，主要负责动态电源管理、PCIE/ICN/D2D 物理层建链以及带外管理的功能。

- 内核驱动支持多种主流 Linux 操作系统，主要包括硬件初始化、进程管理、内存管理、中断管理、任务调度、故障上报和恢复等功能。

- 用户驱动和运行时支持丰富的运行时用户编程接口，主要包括设备管理、上下文管理、流管理、事件管理、内存管理、图管理、核函数启动等功能。

- 驱动支持自研 ICN 多卡互联、对等设备直接内存访问和多进程服务（MPS）等功能。

### 8.2. 编译器

T-HEAD SAIL 编译器驱动支持编译 C++扩展语法的混合主机和设备代码能力。Host 编译器支持 GCC、LLVM C++等主流编译器版本。Device 编译器支持 C99，C++11/14/17/20 语法规范，对真武 M890 特定架构的编程语义实现扩展。Device 编译器提供了完整的编译功能，支持编译器命令行编译工具，也提供静态编译库和动态编译库，以满足集成开发需求；对开源社区 triton、tilelang 等编译项目提供全面支持。除此之外，T-HEAD SAIL 编译器工具链还为开发人员提供了完整的编译器工具，包括调试器、内存检查器、PPU objdump 二进制工具等。

### 8.3. 计算/通信库

计算和通信加速库为各种应用场景提供了高性能的解决方案，将持续不断地针对关键 AI 场景进行更新迭代，目前包含以下面向不同领域的加速库：

**acBLAS：** 基本线性代数计算加速库，包括矩阵乘法及其他 blas level1/level2/level3 定义的标量、向量、矩阵操作；

**acDNN：** 深度神经网络原语计算加速库，包括卷积、BatchNorm、Pooling、Softmax、Activation、CTCLoss、LSTM、GRU 和 backend fusion；

**acSolver：** 矩阵的各类分解和求解方案加速库，目前仅用于密集实数矩阵；

**acFFT：** 正反向傅里叶变换加速库，包括 R2C/C2R/C2C/D2Z/Z2Z；

**acRand：** 伪随机数生成器，包括多种算法和数据分布的组合；

**acSparse：** 通用稀疏矩阵运算加速库，包括 spmm、spmv 等操作，支持多种稀疏格式；

**acext：** 低精度量化运算加速库，包括 a16w8、a16w4、a8w8、a8w4 等量化组合；

**PCCL：** 多真武 M890 卡间和多节点通信的通信加速库，包括 all-gather、all-reduce、broadcast、reduce、reduce-scatter、point-to-point send 和 receive over PCIe/ICN-Link。

### 8.4. 视频编解码器

视频 SDK 为主流视频编解码与图像处理接口提供了对应的兼容 API。它支持：

- 解码：JPEG、MJPEG、H.265、H264、VP9、AV1、AVS2、VP8、WEBP、MPEG-2、AVS 和 AVS+
- 编码：JPEG、H.265、H.264 和 AV1
- 图像处理：颜色转换、缩放和裁剪

### 8.5. 调试和分析工具

#### 8.5.1. PPU-SMI

PPU-SMI 是一个基于 HGML（真武 M890 管理库）的命令行工具，旨在帮助管理和监控真武 M890 设备。此实用程序允许管理员查询真武 M890 设备状态，并允许管理员使用适当的权限修改真武 M890 设备状态。PPU-SMI 可以将查询信息以 XML 或人类可读的纯文本形式报告给标准输出或文件。有关更多详细信息，请参阅 PPU-SMI 文档。

#### 8.5.2. Asight Systems

Asight Systems 是一款低开销的系统级的性能分析工具，用来采集系统各种事件，CPU 和真武 M890 的活动，API 执行时间以及相关调用栈，CPU/真武 M890 activity 关联关系等，在 Timeline View 上统一的可视化呈现出来。通过 Timeline View，开发人员可以方便地分析 CPU/真武 M890 的负载和关联关系，找到性能瓶颈，确保 CPU 和真武 M890 能够协调的工作，充分发挥并行度。有关更多的详细信息，请参阅 Asight Systems 用户文档。

#### 8.5.3. Asight Compute

Asight Compute 是一款 kernel 性能分析工具，通过采集真武 M890 硬件 perf counter，组合成为一系列性能指标，称为 metrics。GUI 通过各种维度，把这些 metrics 呈现出来，帮助用户深入分析和优化 kernel。有关更多的详细信息，请参阅 Asight Compute 用户文档。

#### 8.5.4. PPU-GDB

PPU-GDB 是真武 M890 调试工具，用于调试运行在 Linux 上的 PPU 应用程序。PPU-GDB 是 GNU 项目调试器 GDB 的扩展。该工具为开发人员提供了一种机制，用于调试在实际硬件上运行的 PPU 应用程序。这使开发人员可以在没有模拟和仿真环境所带来的潜在变化的情况下调试应用程序。

PPU-GDB 旨在为用户提供无缝调试环境，允许在同一应用程序中同时调试 PPU 和 CPU 代码。正如设备端 C 语言扩展编程是对主机 C 编程的自然扩展一样，用 PPU-GDB 调试是用 GDB 调试的自然扩展。现有的 GDB 调试功能本质上用于调试主机代码，并且提供了其他功能来支持调试 PPU 设备代码。

### 8.6. 推理引擎

T-HEAD SAIL 推理引擎通过高吞吐量、低延迟和低内存使用量为深度学习模型推理提供高性能。它支持 TensorFlow、PyTorch 训练模型（如 TensorFlow 保存模型和 PyTorch TorchScript 格式）以及 ONNX 模型格式。可以与 TensorFlow 和 PyTorch 进行集成，以及提供独立的 API 供集成使用。支持各种数据类型，如 INT8、FP16、TF32、FP32 和多种混合精度格式。

## 9. 结语

作为现代数据中心的基石，GPGPU 已被广泛用于计算加速。平头哥自主研发设计的真武 M890 架构具备先进的算力、灵活的扩展性和良好的软件生态。在真武 M890 的设计过程中，平头哥与 CSP 进行了深度合作，充分考虑了各种需求，使得真武 M890 更好地适应大型数据中心的部署特点。

真武 M890 芯片可以满足不同规模的计算需求。OAM8 提供了单个节点内的 8 个真武 M890 芯片以环形拓扑进行高速互联。ICN64 将真武 M890 芯片用平头哥 ICN Switch 进行互联，使得用户可以在 64 个真武 M890 芯片构建具有高速 ICN 链路，满足当前大规模 AI 训练和推理的算力和通信需求。通过二级 ICN Switch 互联网络，1024 个真武 M890 可以进一步互联成为大规模 AI 计算引擎，为将来持续增长的 AI 训练和推理任务提供高效可靠的算力基座。
