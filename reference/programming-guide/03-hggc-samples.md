---
source_url: "https://developer.t-head.cn/docs_center/doc_detail/index.html?projectId=39&chapterId=1043"
source_api_url: "https://developer.t-head.cn/api/developer/document/chapter/content?projectId=39&chapterId=1043"
source_title: "T-Head SAIL HGGC 示例程序"
source_id: "chapter-1043"
source_kind: "programming-guide"
source_date: null
source_version: "T-Head SAIL SDK v2.2.1"
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "35bb8618157f9c4e94b9f6f564652fc9567416ee2cda0cbcee060fe6f8599a97"
tags:
  - "samples"
  - "build"
  - "GEMM"
  - "GEMV"
  - "convolution"
  - "sparse"
  - "graph"
  - "cooperative-group"
  - "AIU"
  - "tensor-cell"
  - "benchmark"
---
# T-Head SAIL HGGC 示例程序

## 1. 概述

HGGC-Samples 是面向 HGGC 开发者的示例程序集，展示了 T-Head SAIL 软件工具包的各项特性。当前版本支持 SAIL 2.2.1。

相较于 SAIL 2.2.0，新增 `acdnn_softmax_bf16`（BF16 Softmax 正确性校验）示例。

本版本同时包含 SAIL 2.2.0 对既有示例的更新：

- `atomic_intrinsics`：在静态编译和 HGRTC 运行时编译两条路径中补充 `unsigned long long` 原子操作，校验 global/shared memory 的并发最终值及操作返回的旧值。
- `graph_memory_nodes`：补充 `hggcGraphUpload`，覆盖显式 API 构图和流捕获构图的直接 launch、upload 后 launch 四条路径，并使图内存分配使用实际选定设备。
- `aiu_async_copy`、`aiu_gemm`、`bf16_tensor_cell_gemm`、`imma_tensor_cell_gemm`、`tf32_tensor_cell_gemm`、`aiu_throughput`、`transpose`：完成 ppu001/ppu0015 双架构适配，按架构选择 TIX 指令、AIU descriptor 和数据布局。

### 1.1. 获取 HGGC 示例程序

有关 T-Head SAIL 软件工具包的系统要求和安装说明，请参阅 [T-Head SAIL SDK 安装指南](/docs_center/doc_detail/index.html?projectId=39&chapterId=1039)。

使用 git 克隆 HGGC 示例程序仓库（命令见下方）。

`git clone git@github.com:t-head/hggc-samples.git`

如果不使用 git，最简便的方式是点击仓库页面上的“Download ZIP”按钮下载当前版本的压缩包，解压后即可使用这些示例程序。

### 1.2. 构建 HGGC 示例程序

HGGC 示例程序使用 CMake 构建。请按照以下 Linux 构建说明操作。

确保已安装 CMake（3.20 或更高版本）。如有必要，可通过包管理器安装：

例如： `sudo apt install cmake`

进入克隆的仓库根目录并创建构建目录：

```bash
mkdir build && cd build
```

使用 CMake 配置项目：

```bash
cmake ..
```

默认情况下，HGGC 示例程序会同时为 ppu001（`--gpu-architecture=ppu_10`）和 ppu0015（`--gpu-architecture=ppu_15`）两种 PPU 架构构建。如需仅面向某一种架构构建，可在配置阶段设置 `CMAKE_HG_ARCHITECTURES`：

```bash
cmake -DCMAKE_HG_ARCHITECTURES=ppu_10 ..
```

构建示例程序：

```bash
make -j$(nproc)
```

从构建目录中各自的子目录运行示例程序。也可以从示例程序仓库的任意子目录，或从单个示例程序目录中执行上述流程。

### 1.3. 架构相关 TIX 实现

使用架构相关 TIX 指令的示例在设备编译时选择 ppu001 的 `tc01`/旧版 3D AIU descriptor 或 ppu0015 的 `tc02`/2D tile AIU descriptor 实现。支持范围以各示例 README 的**支持的 PPU 架构**章节为准。

### 1.4. 使用测试脚本运行所有示例程序

需要注意的是，HGGC 示例程序**并非** HGGC 的验证套件。它们不覆盖边界情况，不完整覆盖运行时和驱动 API，也不用于性能基准测试。我们提供了 `run_tests.py` 脚本，可运行所有示例程序作为快速完整性检查。

该 Python3 脚本会在你指定的子目录中查找所有 `.out` 可执行文件（自动排除 CMake 构建系统生成的临时文件），并将应用程序名称与 `test_args.json` 中指定的命令行参数进行匹配。它接受以下命令行参数：

| 开关 | 用途 | 示例 |
| --- | --- | --- |
| --dir | 指定递归搜索可执行文件的根目录 | --dir ./build |
| --config | 可执行文件参数的 JSON 配置文件（存在 `test_args.json` 时默认加载该文件） | --config test_args.json |
| --no-config | 不自动加载 `test_args.json` | --no-config |
| --output | 测试结果的输出目录（stdout 保存为 .txt 文件，目录不存在时自动创建） | --output ./test |
| --parallel | 并行执行的应用程序数量 | --parallel 4 |
| --arch | 目标 PPU 架构，用于架构感知的跳过判断 | --arch ppu0015 |

应用程序配置从 `test_args.json` 中加载，并与可执行文件名称进行匹配。

加载 `test_args.json` 后，脚本会根据 `ppu-smi -q` 的输出识别目标 PPU 架构，并跳过 `skip_arch` 列表中包含该架构的示例程序。你可以通过 `--arch` 覆盖自动识别的架构。当可用设备数量不足时，要求多张 PPU 的示例程序也会被自动跳过。

脚本成功时返回 0，测试失败时返回遇到的第一个非零错误码。如有失败的示例程序，也会打印一份汇总列表。

配置示例：

```json
{
    "multi_device_collab": {
        "min_ppus": 2
    }
}
```

#### 1.4.1. 示例用法

以下是构建并测试所有示例程序的一组示例命令。

首先，构建：

```bash
mkdir build && cd build
cmake ..
make -j$(nproc)
```

然后运行测试脚本：

```bash
# 基本用法（存在 test_args.json 时自动加载）
python3 run_tests.py --dir ./build --output ./test

# 显式指定配置文件
python3 run_tests.py --dir ./build --output ./test --config test_args.json

# 并行运行
python3 run_tests.py --dir ./build --output ./test --parallel 4

# 禁用配置加载
python3 run_tests.py --dir ./build --output ./test --no-config

# 覆盖跳过判断使用的目标架构
python3 run_tests.py --dir ./build --output ./test --arch ppu0015
```

如果所有应用程序均运行成功，你将看到类似如下的输出：

```text
Test Summary:
Ran N test runs for N executables.
All test runs passed!
```

如果部分示例程序失败，你将看到类似如下的输出：

```text
Test Summary:
Ran N test runs for N executables.
Failed runs (2):
  acdnn_conv_activation: Failed (code 1)
  multi_device_collab: Failed (code 1)
```

你可以查看输出目录中的 stdout 日志（通常为 `APM_<application_name>.txt` 或 `APM_<application_name>.run<n>.txt`），从输出日志中判断可能出现的问题。如果你认为某个示例程序在你的系统上出现了不合理的失败，请在示例程序仓库中提交 issue。

## 2. 示例程序列表

### 2.1. 入门

本目录收录 HGGC 入门级示例，按主题归类后的全部子目录如下。部分目录在同一 sample 下提供 Runtime API、HGRTC、Driver API 等多种构建变体。

#### 2.1.1. assert - 设备端断言不变量校验

##### 2.1.1.1. 描述

本示例演示如何将设备端 `assert()` 用作核函数输出的不变量校验工具：

1. **前缀最大值核函数**产生单调非递减的输出；
2. **校验核函数**断言相邻元素满足 `array[i] <= array[i+1]`；
3. 主机端运行两种场景：正确输出（断言不触发）与故意破坏的输出（断言触发，主机检测到错误状态）。

相比简单的“线程 ID 越界”演示，本示例展示了 `assert()` 在真实开发中作为调试工具的实际用法。提供两个构建目标：

- **运行时 API 实现**（`assert.hg`）：kernel 与 host 代码静态编译。
- **HGRTC 运行时编译实现**（`assert_runtime_compile.cpp` + `assert_kernel.hg`）：使用 libHGRTC 即时编译 + 驱动 API 加载执行。

静态 `assert` 目标保留 Release 优化，仅在 HG 编译选项中使用 `-UNDEBUG`。编译期 `NDEBUG` 检查确保设备断言未被禁用。

两种实现均要求正常场景不出现断言错误，破坏输出的场景分别返回 `hggcErrorAssert` 或 `HGGC_ERROR_ASSERT`。预期断言触发后，示例直接退出进程，不继续使用错误上下文，也不重置共享设备。

##### 2.1.1.2. 关键概念

断言（Assert）、运行时编译（Runtime Compilation）。

##### 2.1.1.3. 支持的操作系统

Linux

##### 2.1.1.4. 支持的 CPU 架构

x86_64

##### 2.1.1.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.1.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize, hggcGetErrorString, hggcGetLastError

##### 2.1.1.7. HGGC 驱动 API
hgMemAlloc, hgMemFree, hgMemcpyHtoD, hgModuleGetFunction, hgLaunchKernel, hgCtxSynchronize

##### 2.1.1.8. 构建/运行所需依赖
HGRTC（运行时编译实现需要）

##### 2.1.1.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.2. async_event_timing - 异步 API

##### 2.1.2.1. 描述

本示例演示了如何使用 HGGC 事件（Events）进行 PPU 计时，以及如何实现 CPU 和 PPU 的并行执行。事件被插入到 HGGC 调用流（Stream）中。由于 HGGC 流调用是异步的，CPU 可以在 PPU 执行任务（包括主机与设备之间的 DMA 内存拷贝）的同时进行计算。CPU 可以查询 HGGC 事件来判断 PPU 是否已完成任务。

##### 2.1.2.2. 关键概念

异步数据传输（Asynchronous Data Transfers）、HGGC 流和事件（Streams and Events）。

##### 2.1.2.3. 支持的操作系统

Linux

##### 2.1.2.4. 支持的 CPU 架构

x86_64

##### 2.1.2.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.2.6. HGGC 运行时 API
hggcProfilerStop, hggcMalloc, hggcMemcpyAsync, hggcFree, hggcMallocHost, hggcProfilerStart, hggcDeviceSynchronize, hggcEventRecord, hggcFreeHost, hggcMemset, hggcEventDestroy, hggcEventQuery, hggcEventElapsedTime, hggcGetDeviceProperties, hggcEventCreate

##### 2.1.2.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.3. atomic_intrinsics - 简单原子操作 {#atomic_intrinsics}

##### 2.1.3.1. 描述

本示例通过静态编译和 HGRTC 运行时编译两条路径演示 32 位原子操作，以及 `unsigned long long` 的 Add/And/Or/Xor/Min/Max/Exch/CAS，覆盖 64 位数据的高 32 位和 global/shared memory。并发测试检查最终值，Exch 检查结果是否属于合法写入集合；单线程测试检查每次操作返回的旧值、最终值，以及 CAS 成功和失败时的行为，不假设并发线程的执行次序。32 位 OR 操作使用 unsigned 位移。

两条实现路径：

- **运行时 API 实现**（`static.hg` + `atomic_kernel.hgh`）：基于 HGGC 运行时 API，演示全局内存原子指令的最简使用。
- **HGRTC 运行时编译实现**（`runtime_compile.cpp` + `atomic_kernel_rtc.hg`）：使用 HGRTC 在程序启动时即时编译 kernel 源，通过驱动 API 加载执行，演示同一组原子内置函数在运行时编译路径上的等价行为。

##### 2.1.3.2. 关键概念

原子操作内置函数（Atomic Intrinsics）、ULL、global/shared memory、并发最终值、顺序 old-value、运行时编译（Runtime Compilation）。

##### 2.1.3.3. 支持的操作系统

Linux

##### 2.1.3.4. 支持的 CPU 架构

x86_64

##### 2.1.3.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.3.6. HGGC 运行时 API
hggcStreamCreateWithFlags, hggcFree, hggcMallocHost, hggcFreeHost, hggcStreamSynchronize, hggcMalloc, hggcMemcpyAsync, hggcStreamDestroy

##### 2.1.3.7. HGGC 驱动 API
hgMemcpyDtoH, hgLaunchKernel, hgMemcpyHtoD, hgCtxSynchronize, hgMemAlloc, hgMemFree, hgModuleGetFunction, hgModuleLoadData, hgModuleUnload

##### 2.1.3.8. HGRTC API
hgrtcCreateProgram, hgrtcCompileProgram, hgrtcGetProgramLogSize, hgrtcGetProgramLog, hgrtcGetHGBINSize, hgrtcGetHGBIN

##### 2.1.3.9. 构建/运行所需依赖
HGRTC（运行时编译实现需要）

##### 2.1.3.10. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。最低要求 SAIL 2.2.0。

#### 2.1.4. block_sort_merge - Block 排序 + 双调归并

##### 2.1.4.1. 描述

本示例演示在 PPU 上的两阶段并行排序：

1. **块内排序（Block sort）**：每个线程块使用奇偶换位排序网络（Odd-Even Transposition Network）在共享内存中对一段元素进行排序。
2. **双调归并（Bitonic merge）**：相邻的有序块通过共享内存中的双调归并网络进行合并。

该排序处理键值对（Key-Value Pairs），并以 `std::stable_sort` 作为 CPU 参考，验证排序的稳定性（相等的键保持原始值顺序）。

##### 2.1.4.2. 关键概念

并行排序（Parallel Sorting）、奇偶换位排序（Odd-Even Transposition Sort）、双调归并（Bitonic Merge）、键值稳定性（Key-Value Stability）、共享内存（Shared Memory）。

##### 2.1.4.3. 支持的操作系统

Linux

##### 2.1.4.4. 支持的 CPU 架构

x86_64

##### 2.1.4.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.4.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcDeviceSynchronize

##### 2.1.4.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.5. clock - 设备端时钟微基准

##### 2.1.5.1. 描述

本示例使用设备端 `clock()` 内置函数，按 block 测量**合并访问（coalesced）**与**跨步访问（strided）**两种说明性访存模式的时钟周期开销。合并访问循环重复读取各线程对应的元素；跨步访问在 block 大小的区域内循环，包含重复地址。这些模式不构成独立带宽基准，速度排名不作为正确性判据。

每次核函数完成后，主机将全部累加输出与独立的高精度 CPU 参考结果逐元素比较，使用 `abs_error <= 1e-4 + 1e-5 * abs(reference)` 判据，并拒绝非有限值或未写入的输出。全部 block 的计时必须有效且非零。周期比仅作为 `INFO` 输出，`Result: PASS/FAIL` 和退出码由数据、计时有效性及 API 执行是否成功决定，不取决于哪种模式更快。

两个构建目标共享相同的算法逻辑，演示不同的编译/加载路径：

- **运行时 API 实现**（`clock.hg`）：kernel 与 host 代码在同一文件中静态编译。
- **HGRTC 运行时编译实现**（`clock_runtime_compile.cpp` + `clock_kernel.hg`）：kernel 源在程序启动时由 libHGRTC 即时编译，通过驱动 API 加载执行。

##### 2.1.5.2. 关键概念

设备端时钟测量（On-Device Clock）、HGGC 运行时 API、HGRTC 运行时编译、性能优化策略（Performance Strategies）。

##### 2.1.5.3. 支持的操作系统

Linux

##### 2.1.5.4. 支持的 CPU 架构

x86_64

##### 2.1.5.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.5.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.1.5.7. HGGC 驱动 API
hgMemAlloc, hgMemFree, hgMemcpyHtoD, hgMemcpyDtoH, hgModuleGetFunction, hgLaunchKernel, hgCtxSynchronize

##### 2.1.5.8. 构建/运行所需依赖
HGRTC（运行时编译实现需要）

##### 2.1.5.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.6. cooperative_sync - 协作同步原语

##### 2.1.6.1. 描述

本示例演示两种 HGGC 协作同步原语：到达等待屏障（Arrive-Wait Barrier）下的向量归一化，以及协作组（Cooperative Groups）下的块内基本用法。两条路径共享设备发现与命令行解析，通过 `--mode=barrier|groups` 子命令切换两种同步原语；默认 `groups` 模式。

##### 2.1.6.2. 关键概念

到达等待屏障（Arrive-Wait Barrier）、协作组（Cooperative Groups）。

##### 2.1.6.3. 支持的操作系统

Linux

##### 2.1.6.4. 支持的 CPU 架构

x86_64

##### 2.1.6.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.6.6. HGGC 运行时 API
hggcStreamCreateWithFlags, hggcFree, hggcDeviceGetAttribute, hggcMallocHost, hggcFreeHost, hggcStreamSynchronize, hggcLaunchCooperativeKernel, hggcMalloc, hggcOccupancyMaxActiveBlocksPerMultiprocessor, hggcMemcpyAsync, hggcOccupancyMaxPotentialBlockSize, hggcDeviceSynchronize, hggcGetErrorString, hggcStreamDestroy

##### 2.1.6.7. 构建/运行所需依赖
CPP11, MBCG（`--mode=barrier` 路径需要）

##### 2.1.6.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

##### 2.1.6.9. 模式说明

###### 2.1.6.8.1. --mode=barrier
到达等待屏障实现：所有线程通过到达—等待原语完成阶段同步，演示设备端协作屏障下的向量归一化。

###### 2.1.6.8.2. --mode=groups（默认）
协作组实现：使用 `cooperative_groups` 提供的 thread_block / thread_block_tile / coalesced_threads 等抽象，展示协作组在线程块内的基本用法。

#### 2.1.7. cpp_template_kernels - C++ 模板内核

##### 2.1.7.1. 描述

本示例通过一个**基于策略（Policy-based）的并行归约**，演示 C++ 模板在 PPU 核函数中的高级用法：

- **模板模板参数**：将归约策略（SumOp / MaxOp）作为参数注入核函数；
- **类型安全的动态共享内存包装**：通过 `DynamicSharedBuffer<T>` 实现；
- **模板特化**：为不同数据类型提供单位元（如 `INT_MIN` / `-FLT_MAX`）；
- **多类型 × 多操作组合分发**：一次性验证 `{float, int}` × `{Sum, Max}` 四种组合。

##### 2.1.7.2. 关键概念

C++ 模板（Templates）。

##### 2.1.7.3. 支持的操作系统

Linux

##### 2.1.7.4. 支持的 CPU 架构

x86_64

##### 2.1.7.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.7.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcGetDeviceProperties, hggcFree, hggcDeviceSynchronize

##### 2.1.7.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.8. device_diagnostics - 设备端诊断输出

##### 2.1.8.1. 描述

本示例是一个基础的 HGGC 运行时 API 示例，演示了如何在设备代码中使用 printf 函数。

##### 2.1.8.2. 关键概念

调试（Debugging）。

##### 2.1.8.3. 支持的操作系统

Linux

##### 2.1.8.4. 支持的 CPU 架构

x86_64

##### 2.1.8.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.8.6. HGGC 运行时 API
hggcGetDeviceProperties, hggcDeviceSynchronize, hggcGetDevice

##### 2.1.8.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.9. driver_runtime_interop - 简单驱动-运行时交互

##### 2.1.9.1. 描述

一个简单的示例，演示了 HGGC 驱动 API 和运行时 API 如何协同工作，加载向量加法核函数的 hggc fatbinary 并执行向量加法。

##### 2.1.9.2. 关键概念

HGGC 驱动 API、HGGC 运行时 API、向量加法。

##### 2.1.9.3. 支持的操作系统

Linux

##### 2.1.9.4. 支持的 CPU 架构

x86_64

##### 2.1.9.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.9.6. HGGC 运行时 API
hggcStreamCreateWithFlags, hggcFree, hggcMallocHost, hggcFreeHost, hggcStreamSynchronize, hggcMalloc, hggcMemcpyAsync, hggcStreamDestroy

##### 2.1.9.7. HGGC 驱动 API
hgLaunchKernel, hgModuleLoadData, hgCtxDestroy, hgModuleUnload, hgModuleGetFunction, hgCtxCreate, hgInit

##### 2.1.9.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.10. fp16_dot_product - FP16 标量乘积

##### 2.1.10.1. 描述

计算两个 FP16 数值向量的标量乘积（点积）。

##### 2.1.10.2. 关键概念

HGGC 运行时 API。

##### 2.1.10.3. 支持的操作系统

Linux

##### 2.1.10.4. 支持的 CPU 架构

x86_64

##### 2.1.10.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.10.6. HGGC 运行时 API
hggcMemcpy, hggcFree, hggcMallocHost, hggcFreeHost, hggcMalloc, hggcGetDeviceProperties

##### 2.1.10.7. 构建/运行所需依赖
FP16

##### 2.1.10.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.11. matrix_mul - 矩阵乘法

##### 2.1.11.1. 描述

本示例演示 PPU 上带偏置的分块 GEMM（`C = alpha * A * B + beta * C`），通过三种 API 路径实现：

- **运行时 API 实现**（`matrix_mul.hg`）：单文件包含 kernel 与主机端，使用事件进行计时。
- **HGRTC 运行时编译实现**（`runtime_compile.cpp`）：使用 HGRTC 对 kernel 源进行 JIT 编译。
- **驱动 API 实现**（`driver_api.cpp`）：加载离线 fatbin，并基于占用率选择 tile 尺寸。

所有实现均采用固定随机种子生成输入，并按相对误差容差与 CPU 参考三重循环计算结果进行校验。

##### 2.1.11.2. 关键概念

分块 GEMM（Tiled GEMM）、共享内存（Shared Memory）、偏置项（Bias Term）、CPU 参考校验、HGGC 运行时 API、HGGC 驱动 API、运行时编译（Runtime Compilation）。

##### 2.1.11.3. 支持的操作系统

Linux

##### 2.1.11.4. 支持的 CPU 架构

x86_64

##### 2.1.11.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.11.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcEventCreate, hggcEventRecord, hggcEventSynchronize, hggcEventElapsedTime, hggcEventDestroy, hggcDeviceSynchronize

##### 2.1.11.7. HGGC 驱动 API
hgInit, hgCtxCreate, hgCtxDestroy, hgModuleLoadData, hgModuleGetFunction, hgMemAlloc, hgMemFree, hgMemcpyHtoD, hgMemcpyDtoH, hgLaunchKernel, hgOccupancyMaxPotentialBlockSize

##### 2.1.11.8. 构建/运行所需依赖
HGRTC（运行时编译实现需要）

##### 2.1.11.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.12. mpi_ppu_dispatch - 简单 MPI

##### 2.1.12.1. 描述

本示例演示 MPI 与 PPU 结合完成一维温度梯度计算。温度分布通过 `MPI_Scatter` 分发到各 MPI rank，每个 rank 在 PPU 上计算中心差分梯度，随后通过 `MPI_Gather` 收集到根进程进行统计与 CPU 参考校验。

流程：

1. 根进程生成一维温度分布（线性梯度 + 随机扰动）。
2. `MPI_Scatter` 将各分段分发到所有 rank。
3. 每个 rank 在 PPU 上计算梯度：`out[i] = (T[i+1] - T[i-1]) / 2`。
4. `MPI_Gather` 将梯度收集回根进程。
5. 根进程报告最小/最大/均值统计，并与 CPU 参考结果进行校验。

##### 2.1.12.2. 关键概念

MPI 集成、PPU 核函数、中心差分（Central Difference）、Scatter/Gather、温度梯度（Temperature Gradient）。

##### 2.1.12.3. 支持的操作系统

Linux

##### 2.1.12.4. 支持的 CPU 架构

x86_64

##### 2.1.12.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.12.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcGetLastError

##### 2.1.12.7. 构建/运行所需依赖
MPI

##### 2.1.12.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.13. multi_device_collab - 多设备协作

##### 2.1.13.1. 描述

本示例演示在多个 HGGC 设备上协作工作的两条典型路径：基于 HGGC 上下文管理与多线程访问的多 PPU 异步派发，以及在两块支持点对点（P2P）能力的 PPU 之间通过 P2P 拷贝、P2P 寻址和统一虚拟内存寻址（UVA）完成跨设备核函数计算。两条路径共享设备枚举与拓扑探测，通过 `--mode=multi_dispatch|p2p` 子命令切换；缺省时若存在至少一对 P2P 可达 PPU 自动选择 `p2p`，否则回退到 `multi_dispatch`。

##### 2.1.13.2. 关键概念

性能优化策略（Performance Strategies）、异步数据传输（Asynchronous Data Transfers）、HGGC 流和事件（Streams and Events）、统一虚拟地址空间（Unified Virtual Address Space）、点对点数据传输（Peer to Peer Data Transfers）、多线程（Multithreading）、多 PPU。

##### 2.1.13.3. 支持的操作系统

Linux

##### 2.1.13.4. 支持的 CPU 架构

x86_64

##### 2.1.13.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.13.6. HGGC 运行时 API
hggcStreamDestroy, hggcFree, hggcMallocHost, hggcSetDevice, hggcFreeHost, hggcStreamSynchronize, hggcMalloc, hggcMemcpyAsync, hggcStreamCreate, hggcGetDeviceCount, hggcMemcpy, hggcEventCreateWithFlags, hggcEventSynchronize, hggcDeviceDisablePeerAccess, hggcDeviceSynchronize, hggcEventRecord, hggcGetDeviceProperties, hggcDeviceEnablePeerAccess, hggcEventDestroy, hggcEventElapsedTime, hggcDeviceCanAccessPeer

##### 2.1.13.7. 构建/运行所需依赖
only-64-bit

##### 2.1.13.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

##### 2.1.13.9. 模式说明

###### 2.1.13.8.1. --mode=multi_dispatch
多 PPU 异步派发：使用 HGGC 上下文管理和多线程访问，在多个 PPU 上运行 HGGC 核函数。

###### 2.1.13.8.2. --mode=p2p
跨设备 P2P 协作：在两块相同型号的 PPU 之间通过点对点（P2P）拷贝、P2P 寻址和统一虚拟内存寻址（UVA）完成跨设备 kernel 计算。

#### 2.1.14. occupancy_calculator - 占用率计算器

##### 2.1.14.1. 描述

本示例演示了 HGGC 占用率计算器和基于占用率的启动配置器 API 的基本用法，通过启动配置器启动核函数，并测量其与手动配置启动之间的利用率差异。

##### 2.1.14.2. 关键概念

占用率计算器（Occupancy Calculator）。

##### 2.1.14.3. 支持的操作系统

Linux

##### 2.1.14.4. 支持的 CPU 架构

x86_64

##### 2.1.14.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.14.6. HGGC 运行时 API
hggcMemcpy, hggcFree, hggcDeviceSynchronize, hggcEventRecord, hggcGetDevice, hggcMalloc, hggcEventElapsedTime, hggcOccupancyMaxActiveBlocksPerMultiprocessor, hggcGetDeviceProperties, hggcOccupancyMaxPotentialBlockSize, hggcEventCreate

##### 2.1.14.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.15. openmp_ppu_dispatch - HGGC OpenMP

##### 2.1.15.1. 描述

本示例演示使用 OpenMP 开发多 PPU 应用程序。每个 OpenMP 线程绑定一个 PPU 设备，并使用缩放平移核函数（`out[i] = in[i] * scale + shift`）处理浮点数组的一个切片。结果通过带容差的比较与 CPU 参考结果进行校验。

##### 2.1.15.2. 关键概念

OpenMP 集成、多设备派发（Multi-Device Dispatch）、数组缩放（Array Scaling）、CPU 参考校验。

##### 2.1.15.3. 支持的操作系统

Linux

##### 2.1.15.4. 支持的 CPU 架构

x86_64

##### 2.1.15.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.15.6. HGGC 运行时 API
hggcGetDeviceCount, hggcGetDeviceProperties, hggcSetDevice, hggcGetDevice, hggcMalloc, hggcMemcpy, hggcFree

##### 2.1.15.7. 构建/运行所需依赖
OpenMP

##### 2.1.15.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.16. project_template - GEMV 项目模板

##### 2.1.16.1. 描述

一个基于 **GEMV（矩阵-向量乘 `y = A · x`）** 的 HGGC 项目模板，可作为创建新 HGGC 项目的起点。本示例还演示：

- **混合编译**：`.hg` 核函数 + `.cpp` 主机参考实现，分别编译后链接为单个可执行文件；
- **共享内存分块**：将向量 `x` 分块加载到共享内存，供同一 block 内所有线程复用；
- **多 block 并行**：每个线程负责输出向量 `y` 的一行，grid 自动覆盖整个 M 维；
- **完整的主机端流水线**：设备查询 → 设备内存分配 → H→D / D→H 数据传输 → 核函数启动 → 计时 → CPU 校验。

##### 2.1.16.2. 关键概念

共享内存与 `__syncthreads()` 同步（Shared Memory & `__syncthreads()`）、Block/线程索引（Block/Thread Indexing）、设备内存分配与同步（Device Memory Allocation & Synchronization）、主机/设备协作与 CPU 参考校验（Host/Device Collaboration & CPU Reference Verification）。

##### 2.1.16.3. 支持的操作系统

Linux

##### 2.1.16.4. 支持的 CPU 架构

x86_64

##### 2.1.16.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.16.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcDeviceSynchronize

##### 2.1.16.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.17. stream_callback - 简单 HGGC 回调

##### 2.1.17.1. 描述

本示例通过 `hggcStreamAddCallback` 演示异步流回调。多个工作线程（C++11 `std::thread`）在各自独立的流上启动 PPU 核函数；每个流完成时触发主机回调，回调校验结果并对一个 `std::atomic` 计数器发出信号。本示例使用标准 C++ 线程，而非自定义线程封装。

##### 2.1.17.2. 关键概念

流回调（Stream Callbacks）、异步执行（Asynchronous Execution）、多线程工作负载（Multi-Threaded Workloads）、std::thread、std::atomic。

##### 2.1.17.3. 支持的操作系统

Linux

##### 2.1.17.4. 支持的 CPU 架构

x86_64

##### 2.1.17.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.17.6. HGGC 运行时 API
hggcGetDeviceCount, hggcSetDevice, hggcStreamCreate, hggcStreamDestroy, hggcMalloc, hggcFree, hggcHostAlloc, hggcFreeHost, hggcMemcpyAsync, hggcStreamAddCallback

##### 2.1.17.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.18. streams_concurrency - 流级并发与计算-拷贝重叠

##### 2.1.18.1. 描述

本示例演示 HGGC 流（Streams）级并发的三类典型用法：使用 HGGC 流重叠核函数执行与主机—PPU 设备之间的内存拷贝（并复用 HGGC 固定通用主机内存特性），使用 HGGC 流并发执行多个核函数，以及使用 HGGC 流实现核函数执行与设备数据拷贝的重叠。三条路径共享设备发现与命令行解析，通过 `--mode=streams|hyperq|multicopy` 子命令切换；缺省运行 `streams` 路径。

##### 2.1.18.2. 关键概念

异步数据传输（Asynchronous Data Transfers）、HGGC 流和事件（Streams and Events）、HGGC 系统集成、性能优化策略（Performance Strategies）、计算与拷贝重叠（Overlap Compute and Copy）、PPU 性能。

##### 2.1.18.3. 支持的操作系统

Linux

##### 2.1.18.4. 支持的 CPU 架构

x86_64

##### 2.1.18.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.18.6. HGGC 运行时 API
hggcMemcpy, hggcSetDeviceFlags, hggcSetDevice, hggcEventDestroy, hggcStreamCreate, hggcMallocHost, hggcEventCreateWithFlags, hggcFreeHost, hggcMemcpyAsync, hggcGetDeviceCount, hggcStreamDestroy, hggcMemset, hggcEventElapsedTime, hggcHostAlloc, hggcFree, hggcEventSynchronize, hggcEventRecord, hggcMalloc, hggcGetDeviceProperties, hggcEventCreate, hggcDeviceSynchronize, hggcDeviceGetAttribute

##### 2.1.18.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

##### 2.1.18.8. 模式说明

###### 2.1.18.7.1. --mode=streams（默认）
使用 HGGC 流重叠核函数执行与主机—PPU 设备之间的内存拷贝，并复用 HGGC 固定通用主机内存特性。

###### 2.1.18.7.2. --mode=hyperq
使用 HGGC 流并发执行多个核函数。

###### 2.1.18.7.3. --mode=multicopy
使用 HGGC 流实现核函数执行与设备数据拷贝的重叠。

#### 2.1.19. unified_memory_models - 统一内存编程模型对比

##### 2.1.19.1. 描述

本示例对比演示 HGGC 上的两种统一内存编程模型：使用零拷贝（Zero MemCopy），核函数直接读写固定的系统内存；以及在单个 PPU 上使用 OpenMP 和流（Streams）与统一内存（Unified Memory）协同工作。两条路径共享设备发现与命令行解析，通过 `--mode=zero_copy|managed_streams` 子命令切换；缺省时若设备具备 managed memory 能力则自动选择 `managed_streams`，否则回退到 `zero_copy`。

##### 2.1.19.2. 关键概念

性能优化策略（Performance Strategies）、固定系统分页内存（Pinned System Paged Memory）、向量加法、HGGC 系统集成、OpenMP、ACBLAS、多线程（Multithreading）、统一内存（Unified Memory）、HGGC 流和事件（Streams and Events）。

##### 2.1.19.3. 支持的操作系统

Linux

##### 2.1.19.4. 支持的 CPU 架构

x86_64

##### 2.1.19.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.19.6. HGGC 运行时 API
hggcHostAlloc, hggcSetDeviceFlags, hggcSetDevice, hggcGetDeviceCount, hggcHostGetDevicePointer, hggcDeviceSynchronize, hggcFreeHost, hggcGetDeviceProperties, hggcStreamDestroy, hggcFree, hggcMallocManaged, hggcStreamAttachMemAsync, hggcStreamSynchronize, hggcStreamCreate

##### 2.1.19.7. 构建/运行所需依赖
OpenMP, UVM, ACBLAS（`--mode=managed_streams` 路径需要）

##### 2.1.19.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

##### 2.1.19.9. 模式说明

###### 2.1.19.8.1. --mode=zero_copy
使用零拷贝（Zero MemCopy），核函数可以直接读写固定的系统内存。

###### 2.1.19.8.2. --mode=managed_streams
在单个 PPU 上使用 OpenMP 和流（Streams）与统一内存（Unified Memory）协同工作。

#### 2.1.20. vector_add - 向量加法

##### 2.1.20.1. 描述

本示例演示 PPU 上的加权向量线性组合（`out[i] = alpha * x[i] + beta * y[i] + gamma`），通过四种 API 路径实现：

- **运行时 API 实现**（`vector_add.hg`）：单文件包含 kernel 与主机端。
- **HGRTC 运行时编译实现**（`runtime_compile.cpp`）：使用 HGRTC 对 kernel 源进行 JIT 编译。
- **驱动 API 实现**（`driver_api.cpp`）：通过驱动 API 加载离线 fatbin。
- **多设备 VMM 实现**（`mmap_multidevice.cpp`）：在支持 P2P 的设备之间进行条带式虚拟地址映射。

所有实现均采用确定性正弦输入，并按相对误差容差进行校验。

##### 2.1.20.2. 关键概念

向量运算（Vector Operations）、HGGC 运行时 API、HGGC 驱动 API、运行时编译（Runtime Compilation）、虚拟内存管理（VMM）、多设备映射（Multi-Device Mapping）。

##### 2.1.20.3. 支持的操作系统

Linux

##### 2.1.20.4. 支持的 CPU 架构

x86_64

##### 2.1.20.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.20.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcGetLastError

##### 2.1.20.7. HGGC 驱动 API
hgInit, hgCtxCreate, hgCtxDestroy, hgModuleLoadData, hgModuleGetFunction, hgMemAlloc, hgMemFree, hgMemcpyHtoD, hgMemcpyDtoH, hgLaunchKernel

##### 2.1.20.8. 构建/运行所需依赖
HGRTC（运行时编译实现需要）；支持多设备 P2P 的真武 PPU（多设备 VMM 实现需要，至少 2 张可互通的设备）

##### 2.1.20.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.1.21. warp_vote_ops - 简单投票内置函数

##### 2.1.21.1. 描述

一个简单的程序，演示了如何在 HGGC 核函数中使用投票（Vote）内置指令（__any_sync、__all_sync）。

##### 2.1.21.2. 关键概念

投票内置函数（Vote Intrinsics）。

##### 2.1.21.3. 支持的操作系统

Linux

##### 2.1.21.4. 支持的 CPU 架构

x86_64

##### 2.1.21.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.1.21.6. HGGC 运行时 API
hggcMemcpy, hggcFree, hggcDeviceSynchronize, hggcMalloc, hggcGetDeviceProperties

##### 2.1.21.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

### 2.2. 实用工具

本目录收录用于查询设备属性与系统拓扑的工具类示例。部分目录在同一 sample 下提供 Runtime API、Driver API 等多种构建变体。

#### 2.2.1. device_query - 设备查询

##### 2.2.1.1. 描述

本示例查询并按功能类别（计算、内存、调度、纹理限制、特性）展示 PPU 设备属性，提供两种 API 路径：

- **运行时 API 实现**（`device_query.cpp`）：使用 `DeviceReporter` 类，以分组属性报告和框线绘制格式输出。
- **驱动 API 实现**（`device_query_drv.cpp`）：使用 `hgDeviceGetAttribute` 逐属性查询，并以分段的键值对形式输出。

两个版本还会枚举设备之间的 P2P 拓扑。

##### 2.2.1.2. 关键概念

HGGC 运行时 API、HGGC 驱动 API、设备查询（Device Query）、属性检查（Property Inspection）、P2P 拓扑。

##### 2.2.1.3. 支持的操作系统

Linux

##### 2.2.1.4. 支持的 CPU 架构

x86_64

##### 2.2.1.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.2.1.6. HGGC 运行时 API
hggcGetDeviceCount, hggcGetDeviceProperties, hggcSetDevice, hggcDriverGetVersion, hggcRuntimeGetVersion, hggcDeviceGetAttribute, hggcDeviceCanAccessPeer, hggcGetErrorString

##### 2.2.1.7. HGGC 驱动 API
hgInit, hgDeviceGetCount, hgDeviceGetName, hgDeviceTotalMem, hgDeviceGetAttribute, hgDriverGetVersion, hgDeviceCanAccessPeer

##### 2.2.1.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.2.2. topology_query - 拓扑查询

##### 2.2.2.1. 描述

本示例生成一份结构化的多 PPU 系统拓扑报告，包含四个部分：

1. **设备概览表** — 每个 PPU 的名称、计算单元数量、内存容量与时钟频率；
2. **P2P 连通性矩阵** — 以 ASCII 矩阵可视化展示设备间的访问能力与原子操作支持；
3. **性能分组** — 按 P2P 性能等级分组，辅助选择最优的设备配对；
4. **主机 ↔ 设备属性** — 主机的原子操作、UVA 与托管内存支持状态。

用于在多设备工作负载分配前理解系统互连拓扑，为数据放置与迁移策略提供决策依据。

##### 2.2.2.2. 关键概念

性能优化策略（Performance Strategies）、多 PPU。

##### 2.2.2.3. 支持的操作系统

Linux

##### 2.2.2.4. 支持的 CPU 架构

x86_64

##### 2.2.2.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.2.2.6. HGGC 运行时 API
hggcGetDeviceCount, hggcGetDeviceProperties, hggcDeviceGetAttribute, hggcDeviceGetP2PAttribute

##### 2.2.2.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

### 2.3. 算法与技术

#### 2.3.1. convolution_separable - HGGC 可分离卷积

##### 2.3.1.1. 描述

本示例以高斯模糊为例，演示应用于图像处理的**可分离卷积（Separable Convolution）**：

- 将二维卷积分解为水平 + 垂直两趟一维卷积（复杂度从 O(N²K²) 降至 O(N²K)）；
- 行卷积核函数：每个 block 将 TILE_W 个像素 + 左右 halo 加载到共享内存；
- 列卷积核函数：转置布局确保垂直方向的合并读取；
- 一维核权重存放于 `__constant__` 内存以便广播；
- 高斯核在主机端由 sigma 参数生成，而非随机值。

##### 2.3.1.2. 关键概念

图像处理（Image Processing）、数据并行算法（Data Parallel Algorithms）。

##### 2.3.1.3. 支持的操作系统

Linux

##### 2.3.1.4. 支持的 CPU 架构

x86_64

##### 2.3.1.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.1.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize, hggcMemcpyToSymbol

##### 2.3.1.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.2. ctx_management - 多上下文生命周期管理

##### 2.3.2.1. 描述

本示例演示 HGGC Driver API 的**多上下文（Multi-Context）管理**机制：

1. **创建多个上下文** — 在同一设备上创建 N 个独立 `HGcontext`。
2. **上下文迁移** — 工作线程通过 `hgCtxPushCurrent` / `hgCtxPopCurrent` 绑定/解绑上下文。
3. **上下文共享** — 多个线程轮流使用同一上下文（序列化访问）。
4. **HGRTC 编译** — 每个上下文内独立编译和加载 kernel 模块。
5. **正确销毁** — `hgModuleUnload` + `hgCtxDestroy` 清理。

展示的核心 API：
- `hgCtxCreate` — 创建浮动上下文。
- `hgCtxPushCurrent` — 将上下文绑定到当前线程。
- `hgCtxPopCurrent` — 从当前线程解绑上下文（变为浮动）。
- `hgCtxSynchronize` — 等待上下文中所有工作完成。
- `hgCtxDestroy` — 销毁上下文。

##### 2.3.2.2. 关键概念

Driver API 上下文管理、多线程上下文迁移、hgCtxPushCurrent / hgCtxPopCurrent、HGRTC 运行时编译。

##### 2.3.2.3. 支持的操作系统

Linux

##### 2.3.2.4. 支持的 CPU 架构

x86_64

##### 2.3.2.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.2.6. HGGC 驱动 API
hgInit, hgDeviceGet, hgDeviceGetName, hgDeviceGetAttribute, hgDeviceGetCount, hgCtxCreate, hgCtxPushCurrent, hgCtxPopCurrent, hgCtxSynchronize, hgCtxDestroy, hgModuleLoadData, hgModuleGetFunction, hgModuleUnload, hgLaunchKernel, hgMemAlloc, hgMemFree, hgMemcpyDtoH

##### 2.3.2.7. 构建/运行所需依赖

HGRTC, C++11 (std::thread)

##### 2.3.2.8. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.3. histogram - HGGC 直方图

##### 2.3.3.1. 描述

本示例通过**三种逐步优化的策略**实现 256-bin 字节直方图，展示原子操作与内存私有化对吞吐的影响：

1. **全局 atomicAdd**（基线）— 每个字节直接原子累加到全局内存，争用最高；
2. **共享内存私有化** — 每个 block 在共享内存中维护私有直方图，最后合并到全局；
3. **Warp 级私有化** — 每个 warp 拥有独立子直方图，消除 warp 间争用，吞吐最高。

三种策略在同一数据集上运行以便对比，分别输出各自的吞吐（GB/s），并与 CPU 参考实现校验正确性。

##### 2.3.3.2. 关键概念

图像处理（Image Processing）、数据并行算法（Data Parallel Algorithms）。

##### 2.3.3.3. 支持的操作系统

Linux

##### 2.3.3.4. 支持的 CPU 架构

x86_64

##### 2.3.3.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.3.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize, atomicAdd

##### 2.3.3.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.4. inline_tix - TIX 内联汇编指令展示

##### 2.3.4.1. 描述

本示例演示通过 `asm()` 语句在 HGGC 核函数中嵌入多类 TIX 指令，每条指令配以一个实际用例：

1. **ppu.fma.rtte.f32** — 融合乘加，用于 Horner 法多项式求值（单次舍入精度优势）；
2. **ppu.popc.b32** — 位计数，计算汉明重量；
3. **ppu.mul.wide.s32** — 宽乘法，32×32 → 64 位全精度乘积；
4. **ppu.bfe.u32** — 位域提取，从任意位置提取指定长度的位片段。

每个测试用例均与 CPU 参考实现校验。提供两种构建模式：

- **运行时 API 实现**（`inline_tix.hg`）：静态编译，4 个测试核函数与 host 在同一文件；
- **HGRTC 运行时编译实现**（`inline_tix_hgrtc.cpp` + `inline_tix_kernel.hg`）：HGRTC 即时编译 + 驱动 API。

##### 2.3.4.2. 关键概念

性能优化策略（Performance Strategies）、TIX 汇编（TIX Assembly）、HGGC 驱动 API、运行时编译（Runtime Compilation）。

##### 2.3.4.3. 支持的操作系统

Linux

##### 2.3.4.4. 支持的 CPU 架构

x86_64

##### 2.3.4.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.4.6. HGGC 运行时 API
hggcMemcpy, hggcFree, hggcDeviceSynchronize, hggcMalloc

##### 2.3.4.7. HGGC 驱动 API
hgMemcpyDtoH, hgMemcpyHtoD, hgLaunchKernel, hgCtxSynchronize, hgMemAlloc, hgMemFree, hgModuleGetFunction

##### 2.3.4.8. 构建/运行所需依赖
HGRTC（运行时编译实现需要）

##### 2.3.4.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.5. radix_sort_thrust - Thrust 数据并行算法流水线

##### 2.3.5.1. 描述

本示例演示使用 Thrust 库构建完整的数据处理流水线，展示多个 Thrust 原语的组合使用：

1. **sort** — 基数排序（Thrust 的经典强项）；
2. **unique** — 移除连续重复元素；
3. **transform** — 应用一元函子（对每个元素求平方）；
4. **reduce** — 全局归约（求和）；
5. **inclusive_scan** — 前缀和。

每一步独立计时，展示 Thrust 作为可组合数据并行工具集的能力。

##### 2.3.5.2. 关键概念

数据并行算法（Data-Parallel Algorithms）、性能优化策略（Performance Strategies）。

##### 2.3.5.3. 支持的操作系统

Linux

##### 2.3.5.4. 支持的 CPU 架构

x86_64

##### 2.3.5.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.5.6. HGGC 运行时 API
hggcEventCreate, hggcEventRecord, hggcEventSynchronize, hggcEventElapsedTime, hggcEventDestroy

##### 2.3.5.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.6. scan - 多 Block 独占前缀和（Blelloch 算法）

##### 2.3.6.1. 描述

本示例采用经典的 Blelloch 工作高效算法实现任意长度的**独占前缀和（exclusive prefix sum）**：

1. **上扫（归约阶段）**：自底向上构建部分和树；
2. **下扫（分发阶段）**：自顶向下分发前缀和。

多 block 策略采用三趟执行：

- 第 1 趟：每个 block 独立扫描其分配的数据块，并写出 block 总和；
- 第 2 趟：对 block 总和数组做前缀和（单 block）；
- 第 3 趟：将 block 前缀加到每个元素上。

支持非 2 的幂次输入长度，并在多种规模下校验正确性。

##### 2.3.6.2. 关键概念

数据并行算法（Data-Parallel Algorithms）、性能优化策略（Performance Strategies）。

##### 2.3.6.3. 支持的操作系统

Linux

##### 2.3.6.4. 支持的 CPU 架构

x86_64

##### 2.3.6.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.6.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.3.6.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.7. spmv - CSR 稀疏矩阵-向量乘

##### 2.3.7.1. 描述

本示例实现 **CSR 格式稀疏矩阵-向量乘（y = A * x）**，展示两种 PPU 并行策略在不规则数据结构上的差异：

1. **Scalar（每线程一行）** — 每个线程独立遍历一行的非零元素求内积。实现最简单，但行长度不均匀时存在负载不均衡。
2. **Vector（每 warp 一行）** — 一个 warp（32 线程）协作处理一行：各 lane 以 stride 方式读取非零元素，最后 warp shuffle 归约求和。适合长行、变长行（如幂律图）。

教学要点：
- **间接寻址**：`col_idx[j]` 导致不规则内存访问模式。
- **负载不均衡**：不同行的非零数差异巨大。
- **Warp 协作归约**：`__shfl_down_sync` 在 SpMV 中的实际应用。

测试矩阵为合成的带状稀疏矩阵（对角占优），带宽可配置。

##### 2.3.7.2. 关键概念

稀疏矩阵（Sparse Matrix）、CSR 格式、不规则并行（Irregular Parallelism）、负载均衡、Warp 协作归约。

##### 2.3.7.3. 支持的操作系统

Linux

##### 2.3.7.4. 支持的 CPU 架构

x86_64

##### 2.3.7.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.7.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize, __shfl_down_sync

##### 2.3.7.7. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.8. stream_compact - 流压缩（条件过滤）

##### 2.3.8.1. 描述

本示例演示 **流压缩（Stream Compaction）** —— 从输入数组中提取满足谓词的元素到紧凑的输出数组中。以"提取素数"为实际用例，展示三种渐进优化策略：

1. **Naive atomic** — 满足条件的线程用 `atomicAdd` 获取输出位置，直接 scatter。实现最简单，但输出无序。
2. **Ballot + popc** — warp 内用 `__ballot_sync` 收集谓词结果，`__popc` 计算 warp 内前缀偏移，block 级用 shared atomic 协调，最后全局 atomic 获取 block 基址。输出 block 内有序。
3. **Two-pass scan** — 第一遍标记 + block 级 exclusive prefix sum 计算写入偏移；第二遍按偏移 scatter。输出完全有序（保留原始相对顺序）。

三种策略在同一数据集 [0, 65536) 上运行，分别计时并与 CPU 参考实现验证正确性。

##### 2.3.8.2. 关键概念

流压缩（Stream Compaction）、前缀和（Prefix Sum）、Warp Ballot、原子操作（Atomic Operations）、条件过滤。

##### 2.3.8.3. 支持的操作系统

Linux

##### 2.3.8.4. 支持的 CPU 架构

x86_64

##### 2.3.8.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.8.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize, atomicAdd, __ballot_sync, __popc, __shfl_sync

##### 2.3.8.7. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.9. stream_ordered_allocation - 流有序分配

##### 2.3.9.1. 描述

本示例深入演示 HGGC **流有序内存池（Stream-Ordered Memory Pools）**的完整生命周期与机制，通过三种分配策略的性能对比揭示内存池复用的价值：

1. **hggcMalloc（同步）** — 基线：每次迭代同步分配/释放，开销最高；
2. **hggcMallocAsync（threshold=0）** — 异步分配，但内存在释放后立即归还操作系统，无池复用；
3. **hggcMallocAsync（threshold=MAX）** — 异步分配 + 高释放阈值，内存保留在池中供后续分配复用。

演示的内存池管理机制：

- `hggcDeviceGetDefaultMemPool` — 获取默认内存池；
- `hggcMemPoolSetAttribute(hggcMemPoolAttrReleaseThreshold)` — 控制释放策略；
- `hggcMemPoolTrimTo` — 手动裁剪内存池，归还未使用内存；
- **地址复用观测** — 在池复用模式下，连续迭代分配到相同地址。

输出格式化的性能对比表，直观展示池复用带来的分配开销降低。

##### 2.3.9.2. 关键概念

性能优化策略（Performance Strategies）。

##### 2.3.9.3. 支持的操作系统

Linux

##### 2.3.9.4. 支持的 CPU 架构

x86_64

##### 2.3.9.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.9.6. HGGC 运行时 API
hggcMallocAsync, hggcFreeAsync, hggcDeviceGetDefaultMemPool, hggcMemPoolSetAttribute, hggcMemPoolTrimTo, hggcStreamCreateWithFlags, hggcStreamSynchronize, hggcDeviceGetAttribute, hggcDeviceSynchronize, hggcFree, hggcMalloc, hggcMemcpy, hggcMemcpyAsync, hggcMemset, hggcMemsetAsync, hggcStreamDestroy

##### 2.3.9.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.10. warp_bitonic_sort - Warp Shuffle 双调排序

##### 2.3.10.1. 描述

本示例实现了一种**纯寄存器**的双调排序（Bitonic Sort），使用 `__shfl_xor_sync` 在 warp 内完成 32 元素排序，无需共享内存。然后通过共享内存中的双调合并网络将各 warp 的有序序列合并为 block 级有序输出。

两阶段设计：
1. **Warp-level sort**（Phase 1）：每个 lane 持有一个元素，通过 5 轮 shuffle-XOR 比较交换完成 32 元素双调排序，全程在寄存器中完成。
2. **Block-level merge**（Phase 2）：将 8 个已排序 warp（共 256 元素）通过共享内存双调合并网络合并为一个有序块。

与传统实现的区别：
- 传统方法：所有数据加载到 shared memory → 全部在 shared memory 做比较交换。
- 本方法：warp 内部排序零 shared memory 开销 → 仅跨 warp 合并时使用 shared memory。

##### 2.3.10.2. 关键概念

双调排序（Bitonic Sort）、Warp Shuffle（__shfl_xor_sync）、排序网络（Sorting Networks）、寄存器级优化。

##### 2.3.10.3. 支持的操作系统

Linux

##### 2.3.10.4. 支持的 CPU 架构

x86_64

##### 2.3.10.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.10.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.3.10.7. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.3.11. warp_redux - 硬件 Warp 级归约（ppu.redux.sync）

##### 2.3.11.1. 描述

本示例演示 TIX 的 `ppu.redux.sync` 指令——一条硬件指令即可完成整个 warp 的归约运算，无需多轮 shuffle 迭代。

每种归约操作同时用两种方式实现并交叉验证：
- **ppu.redux.sync**（单指令，硬件加速）
- **__shfl_down_sync 循环**（传统 5 轮 shuffle 方式）

展示的操作：
1. **ppu.redux.sync.add.s32** — warp 求和。
2. **ppu.redux.sync.min.s32** — warp 最小值。
3. **ppu.redux.sync.max.s32** — warp 最大值。
4. **ppu.redux.sync.xor.b32** — warp 异或（奇偶校验）。

##### 2.3.11.2. 关键概念

Warp 级归约（Warp-Level Reduction）、TIX 内联汇编、ppu.redux.sync、硬件加速原语、__shfl_down_sync 对比。

##### 2.3.11.3. 支持的操作系统

Linux

##### 2.3.11.4. 支持的 CPU 架构

x86_64

##### 2.3.11.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.3.11.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.3.11.7. TIX 指令
ppu.redux.sync.add.s32, ppu.redux.sync.min.s32, ppu.redux.sync.max.s32, ppu.redux.sync.xor.b32

##### 2.3.11.8. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

### 2.4. HGGC 特性

#### 2.4.1. aiu_async_copy - 双架构 AIU 异步张量拷贝 + ldmatrix.swzl 解地址

##### 2.4.1.1. 描述

本示例展示架构对应的 `ppu.cp.async.aiu` 全局→共享异步张量拷贝，与 `ppu.tc01/tc02.ldmatrix.swzl` 解地址/寄存器加载流水线：

1. 4 个 warp 的 lane 0 各使用 ppu001 的 3D descriptor 或 ppu0015 的 2D tile descriptor，复制 16×1×64 tile 到独立 128B 对齐的共享内存分区。
2. `commit_group`、`wait_group 0` 和 `__syncthreads` 确保完成与 block 可见性。
3. 每个 warp 执行 4 次架构对应的 swizzle load，每线程累计 32 个 b16 元素。
4. 按 fragment 映射写回原位置，CPU 使用 `uint16_t` 逐位置精确比较全部 4096 个元素，不排序。

载荷为 64×1×64 的递增 b16 位模式，8192B，不做浮点运算。tc02 swizzle 地址为 `(smem_addr + channel_byte_offset) >> 4`；该拷贝样例交换 R1/R2 以统一 tc01 的写回映射。

##### 2.4.1.2. 关键概念

AIU 硬件单元（ppu.cp.async.aiu）、128B Swizzle 地址变换、ldmatrix.swzl 解地址、异步拷贝同步（commit_group/wait_group）、批量张量拷贝（Bulk Tensor Copy）。

##### 2.4.1.3. 支持的操作系统

Linux

##### 2.4.1.4. 支持的 CPU 架构

x86_64

##### 2.4.1.5. 支持的 PPU 架构

ppu001、ppu0015

设备编译时通过 `__HGGC_ARCH__` 100/150 选择 ppu001 的 3D AIU/tc01 或 ppu0015 的 2D tile AIU/tc02，CMake 对应 `ppu_10`/`ppu_15`；两条路径分别使用对应的 descriptor、地址单位与 fragment 映射。

##### 2.4.1.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.4.1.7. TIX 指令
ppu.cp.async.aiu.bulk.tensor.shared.global, ppu.cp.async.commit_group, ppu.cp.async.wait_group, ppu.tc01/tc02.ldmatrix.swzl.sync.aligned.m8n8.x4.shared.b16, ppu.cvta.to.shared.u32

##### 2.4.1.8. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.2. aiu_gemm - AIU GEMM: FP16 GEMM via AIU .swzl Bulk Tensor Copy

##### 2.4.2.1. 描述

本示例使用架构对应的 AIU `.swzl`（128B swizzle）搬运 A/B 矩阵，配合 `ldmatrix.swzl` 解地址加载，实现 FP16 矩阵乘法 D = A × B。ppu0015 的 swizzle load 地址以 16B 为单位，A/B 采用 tc02 的 `m8n8.x4`/`m8n8.x4.trans`，直接供 tc02 MMA 使用，不额外交换 fragment；ppu001 使用 3D descriptor、字节地址和 B 的 `m16n16.x1.trans`。

关键特性：
- **AIU .swzl** 批量搬运 16×64 tile（dim_c=64 → 128B swizzle）。
- **A 矩阵**：预排列为 16 M-rows × 64 K-cols，`ldmatrix.swzl.m8n8.x4.b16`（无 trans）加载。
- **B 矩阵**：预排列为 16 K-rows × 64 N-cols（原始 row-major，非转置），4 个 N-tile 打包，使用架构对应的 `ldmatrix.swzl.trans` → column-major fragment。
- **channel_offset**：A 选 K-tile，B 选 N-tile。
- **ppu.tc01/tc02.mma.m16n16k16.row.col** 累加计算，C/D 按各架构位置映射写回。
- 随机输入验证，相对误差 < 5%。

##### 2.4.2.2. 分块方案

```text
A tile: 16 M-rows x 64 K-cols (1 M-tile x 1 K-group)
B tile: 16 K-rows x 64 N-cols (1 K-tile x 4 N-tiles packed)

Each block outputs 16x64 (1 M-tile x 4 N-tiles)
Grid: (N/64, M/16)
```

##### 2.4.2.3. B 矩阵端到端布局

```text
gmem: B[k][n] row-major (KxN)
  -> pre-tile: B_tiled[r=K, c=N] remains row-major
  -> AIU .swzl: smem remains row-major (with swizzle)
  -> architecture-specific ldmatrix.swzl.trans: .trans transpose
  -> fragment: column-major (same n, adjacent k) -> MMA .col check
```

##### 2.4.2.4. 关键概念

AIU 硬件单元（ppu.cp.async.aiu）、128B Swizzle 批量搬运、ldmatrix.swzl 解地址、Tensor Cell MMA（ppu.tc01/tc02.mma）、FP16 矩阵乘法。

##### 2.4.2.5. 支持的操作系统

Linux

##### 2.4.2.6. 支持的 CPU 架构

x86_64

##### 2.4.2.7. 支持的 PPU 架构

ppu001、ppu0015

设备编译时通过 `__HGGC_ARCH__` 100/150 选择 ppu001 的 3D AIU/tc01 或 ppu0015 的 2D tile AIU/tc02，CMake 对应 `ppu_10`/`ppu_15`；两条路径分别使用对应的 descriptor、地址单位与 fragment 映射。

##### 2.4.2.8. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.4.2.9. TIX 指令
ppu.cp.async.aiu.bulk.tensor.shared.global, ppu.tc01/tc02.ldmatrix.swzl, ppu.tc01/tc02.mma.sync.aligned.m16n16k16

##### 2.4.2.10. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.3. bf16_tensor_cell_gemm - TIX Tensor Cell BF16 GEMM (Double-Buffer)

##### 2.4.3.1. 描述

本示例使用**纯 TIX 内联汇编**实现 BF16 矩阵乘法（D = A*B），演示 Tensor Cell 指令的底层用法与双缓冲异步流水线：

核心 TIX 指令：

- `ppu.cp.async.cg.shared.global` — 全局 → 共享异步拷贝（16 字节/线程）；
- `ppu.cp.async.commit_group` / `ppu.cp.async.wait_group` — 异步拷贝组管理；
- `ppu.tc01/tc02.ldmatrix.sync.aligned.m8n8.x4.b16` — A 矩阵共享 → 寄存器片段加载；
- 架构对应的 `ldmatrix.trans` — B 矩阵加载（`.trans` 转置生成列主序片段），tc01 用 `m16n16.x1.trans`，tc02 用 `m8n8.x4.trans`；
- `ppu.tc01/tc02.mma.sync.aligned.m16n16k16.row.col.f32.bf16.bf16.f32` — 16×16×16 MMA。

算法结构：

1. **序幕（Prologue）**：预取第一个 K-tile 到 shared buffer[0]；
2. **主循环**：预取下一个 K-tile 到 buffer[1-cur]，同时用 buffer[cur] 做 ldmatrix + mma；
3. **尾声（Epilogue）**：处理最后一个 K-tile，写回 f32 结果。

每线程寄存器片段布局（m16n16k16 bf16）：

- A 片段：4 个 `.f16x2` 寄存器（约束 `"r"`）；
- B 片段：4 个 `.f16x2` 寄存器（约束 `"r"`）；
- C/D 片段：8 个 `.f32` 寄存器（约束 `"f"`）。

ppu0015 的 A/B 行地址为 `row=lane%16`、`col=(lane/16)*8`，加载后直接供 tc02 MMA，不交换 A/B 中间寄存器。C/D 的 `R[i]` 坐标相对于 16×16 输出 tile：

| 架构 | row | col |
|---|---|---|
| tc01 | `lane/4+(i/4)*8` | `lane%4+(i%4)*4` |
| tc02 | `lane/4+((i%4)/2)*8` | `(lane%4)*2+(i%2)+(i/4)*8` |

`/` 为整数除法；tc02 每 lane 持相邻两列，R0/R1、R2/R3、R4/R5、R6/R7 对应左上、左下、右上、右下。AIU FP16、TF32、INT8 同架构采用相同 C/D 映射，不沿用 tc01 排列。

##### 2.4.3.2. 关键概念

矩阵乘法（Matrix Multiply）、AWMMA、Tensor Cell（Tensor Cells）。

##### 2.4.3.3. 支持的操作系统

Linux

##### 2.4.3.4. 支持的 CPU 架构

x86_64

##### 2.4.3.5. 支持的 PPU 架构

ppu001、ppu0015

通过 `__HGGC_ARCH__` 100/150 选择 tc01/tc02 指令与架构对应的 fragment 位置映射，CMake 对应 `ppu_10`/`ppu_15`，不使用其它架构宏代替。

##### 2.4.3.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize

##### 2.4.3.7. TIX 指令
ppu.cp.async.cg.shared.global, ppu.cp.async.commit_group, ppu.cp.async.wait_group, ppu.tc01/tc02.ldmatrix.sync.aligned, ppu.tc01/tc02.mma.sync.aligned.m16n16k16

##### 2.4.3.8. 构建/运行所需依赖
CPP11

##### 2.4.3.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.4. binary_partition_cg - Warp 级条件分流（Binary Partition）

##### 2.4.4.1. 描述

本示例演示 `cg::binary_partition()` 实现的 **Warp 级动态条件分流**：

给定一组浮点数，每个 warp 在运行时根据数值符号动态拆分为两组：

- **正数组**：组内归约求最大值（`cg::reduce` + `greater`）；
- **负数组**：组内归约求最小值（`cg::reduce` + `less`）。

每个子组的 leader 线程将结果原子写回全局内存，随后与 CPU 参考实现对比校验。

应用场景：

- 机器学习推理中的条件路由（MoE 门控）；
- 物理仿真中的粒子分类；
- 图算法中的异构邻居处理。

##### 2.4.4.2. 关键概念

协作组（Cooperative Groups）。

##### 2.4.4.3. 支持的操作系统

Linux

##### 2.4.4.4. 支持的 CPU 架构

x86_64

##### 2.4.4.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.4.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize, cg::binary_partition, cg::reduce

##### 2.4.4.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.5. global_to_shmem_async_copy - 异步拷贝同步策略对比

##### 2.4.5.1. 描述

本示例对比全局内存 → 共享内存异步拷贝的三种**同步策略**，展示 PPU 特有的 awbar 机制：

1. **Naive** — 显式加载（global → 寄存器 → shared）+ `__syncthreads()`；
2. **cp.async + commit_group/wait_group** — 经典异步拷贝流水线（TIX `ppu.cp.async` 指令族）；
3. **cp.async + awbar** — 到达-等待屏障，将异步拷贝完成与线程同步统一到同一原语中。

awbar 的独特之处：

- `ppu.cp.async.awbar.arrive` 自动将拷贝完成事件关联到屏障；
- 线程与异步操作共享同一计数器；
- `ppu.awbar.test_wait` 非阻塞轮询，支持多阶段奇偶切换；
- 比 commit_group/wait_group 更细粒度，无需全局组编号。

负载：分块向量归约（简单，聚焦于同步机制差异）。

##### 2.4.5.2. 关键概念

HGGC 运行时 API、线性代数（Linear Algebra）、CPP11 HGGC。

##### 2.4.5.3. 支持的操作系统

Linux

##### 2.4.5.4. 支持的 CPU 架构

x86_64

##### 2.4.5.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.5.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize

##### 2.4.5.7. TIX 指令
ppu.cp.async.ca.shared.global, ppu.cp.async.commit_group, ppu.cp.async.wait_group, ppu.cp.async.awbar.arrive, ppu.awbar.init, ppu.awbar.arrive, ppu.awbar.test_wait, ppu.cvta.to.shared.u32

##### 2.4.5.8. 构建/运行所需依赖
CPP11

##### 2.4.5.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.6. graph_conditional_nodes - Graph 条件循环迭代求解器

##### 2.4.6.1. 描述

本示例使用 HGGC 图条件节点实现一个**设备端收敛控制的迭代求解器**：用 Jacobi 方法求解三对角线性方程组，循环直至收敛。

图结构：

```text
init_kernel -> WHILE [ jacobi_step -> check_convergence ] -> done_kernel
```

关键机制：

- `hggcGraphConditionalHandleCreate` 创建条件句柄（默认值 1，确保至少执行一次）；
- `hggcGraphAddNode(hggcGraphNodeTypeConditional, hggcGraphCondTypeWhile)` 添加 while 条件节点；
- `hggcGraphSetConditional(handle, 0/1)` 在设备端控制循环终止；
- `hggcStreamBeginCaptureToGraph` 使用流捕获填充循环体。

与 `power_iteration_graph`（使用 `hggcGraphExecKernelNodeSetParams` 动态更新核函数参数）互补——本示例演示条件节点的设备端循环控制能力。

##### 2.4.6.2. 关键概念

HGGC 图（Graphs）。

##### 2.4.6.3. 支持的操作系统

Linux

##### 2.4.6.4. 支持的 CPU 架构

x86_64

##### 2.4.6.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.6.6. HGGC 运行时 API
hggcGraphCreate, hggcGraphAddNode, hggcGraphConditionalHandleCreate, hggcGraphSetConditional, hggcGraphInstantiate, hggcGraphLaunch, hggcGraphExecDestroy, hggcGraphDestroy, hggcStreamCreate, hggcStreamBeginCaptureToGraph, hggcStreamEndCapture, hggcDeviceSynchronize, hggcFree, hggcMalloc, hggcMemcpy, hggcStreamDestroy

##### 2.4.6.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.7. graph_memory_footprint - Graph 内存节点缓冲区复用

##### 2.4.7.1. 描述

本示例演示 HGGC 图内存节点如何**自动复用临时缓冲区**，降低多步计算流水线的峰值内存占用。

场景：一个 3 步计算流水线（`input*2 -> +1 -> *0.5`），每步需要不同大小的临时缓冲区。

对比：

- **无复用**：所有缓冲区同时存在 → 峰值 = buf1 + buf2 + buf3 = 16 KB；
- **图内存节点**：alloc/free 节点让运行时在上一步释放后复用物理内存 → 峰值 ≈ max(buf1, buf2, buf3) = 8 KB。

演示的 API：

- `hggcGraphAddMemAllocNode` — 在图内分配临时内存；
- `hggcGraphAddMemFreeNode` — 在图内释放（使运行时得以复用）；
- `hggcDeviceGetGraphMemAttribute` — 查询实际内存使用量；
- `hggcDeviceGraphMemTrim` — 释放所有未使用的图内存。

##### 2.4.7.2. 关键概念

HGGC 运行时 API、性能优化策略（Performance Strategies）、HGGC 图（Graphs）。

##### 2.4.7.3. 支持的操作系统

Linux

##### 2.4.7.4. 支持的 CPU 架构

x86_64

##### 2.4.7.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.7.6. HGGC 运行时 API
hggcGraphCreate, hggcGraphAddMemAllocNode, hggcGraphAddMemFreeNode, hggcGraphAddKernelNode, hggcGraphInstantiate, hggcGraphLaunch, hggcDeviceGetGraphMemAttribute, hggcDeviceGraphMemTrim, hggcGraphExecDestroy, hggcGraphDestroy, hggcFree, hggcMalloc, hggcMemcpy, hggcStreamCreateWithFlags, hggcStreamDestroy, hggcStreamSynchronize

##### 2.4.7.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.8. graph_memory_nodes - Graph 构建方式对比（API vs Stream Capture） {#graph_memory_nodes}

##### 2.4.8.1. 描述

本示例演示构建 HGGC 图的**两种等价方式**，并验证它们产生相同的执行结果：

1. **图 API（显式构建）** — 手动创建节点并指定依赖关系；
   - `hggcGraphAddMemAllocNode` / `hggcGraphAddMemFreeNode` + `hggcGraphAddKernelNode`；
   - 更灵活，可精确控制依赖拓扑。
2. **流捕获（隐式构建）** — 在捕获模式下的普通 HGGC 代码自动生成图；
   - `hggcStreamBeginCapture` → 普通核函数启动 → `hggcStreamEndCapture`；
   - 代码更简洁，与非图代码一致。

计算：`output = input * 2 + 1`；实际图为 `MemAlloc -> scale(x2) -> bias(+1) -> MemFree`，没有 memcpy node。

显式 API 与 capture 各覆盖普通 launch、首次 launch 前显式 `hggcGraphUpload`，共四条独立路径。使用新实例、独立输出以及同一 non-default stream 的 upload/launch/synchronize，检查有限性、CPU 参考和路径间一致性；没有性能 PASS 门槛。`poolProps.location.id` 使用 `-device=N` 的实际选择。

##### 2.4.8.2. 关键概念

HGGC 图 API、流捕获（Stream Capture）、显式 upload、图内存节点、设备选择、四路径正确性比较。

##### 2.4.8.3. 支持的操作系统

Linux

##### 2.4.8.4. 支持的 CPU 架构

x86_64

##### 2.4.8.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.8.6. HGGC 运行时 API
hggcGraphCreate, hggcGraphAddKernelNode, hggcGraphAddMemAllocNode, hggcGraphAddMemFreeNode, hggcGraphInstantiate, hggcGraphUpload, hggcGraphLaunch, hggcGraphExecDestroy, hggcGraphDestroy, hggcStreamBeginCapture, hggcStreamEndCapture, hggcStreamCreateWithFlags, hggcFree, hggcFreeAsync, hggcMalloc, hggcMallocAsync, hggcMemcpy, hggcStreamDestroy, hggcStreamSynchronize

##### 2.4.8.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。最低要求 SAIL 2.2.0。

#### 2.4.9. imma_tensor_cell_gemm - TIX Tensor Cell INT8 GEMM (m16n16k32)

##### 2.4.9.1. 描述

本示例使用**纯 TIX 内联汇编**实现 INT8 矩阵乘法（D = A*B，s8 输入，s32 累加）。

与 bf16/tf32 版本对比：

| | bf16 | tf32 | **int8（本示例）** |
|---|---|---|---|
| MMA 形状 | m16n16k16 | m16n16k8 | **m16n16k32** |
| A/B 类型 | bf16 (2B) | tf32 (4B) | **s8 (1B)** |
| C/D 类型 | f32 | f32 | **s32** |
| 每 tile 的 K | 16 | 8 | **32** |
| 迭代次数 (K=256) | 16 | 32 | **8** |

核心 TIX 指令：

- `ppu.tc01/tc02.mma.sync.aligned.m16n16k32.row.col.s32.s8.s8.s32` — INT8 MMA；
- `ppu.cp.async.cg.shared.global` — 异步拷贝；
- `ppu.tc01/tc02.ldmatrix.sync.aligned.m8n8.x4.b16` — 片段加载。

精度特性：整数 MMA 计算无舍入误差，结果应与 CPU 参考精确匹配。

##### 2.4.9.2. 关键概念

矩阵乘法（Matrix Multiply）、AWMMA、Tensor Cell。

##### 2.4.9.3. 支持的操作系统

Linux

##### 2.4.9.4. 支持的 CPU 架构

x86_64

##### 2.4.9.5. 支持的 PPU 架构

ppu001、ppu0015

通过 `__HGGC_ARCH__` 100/150 选择 tc01/tc02 指令与架构对应的 fragment 位置映射，CMake 对应 `ppu_10`/`ppu_15`，不使用其它架构宏代替。

##### 2.4.9.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize

##### 2.4.9.7. TIX 指令
ppu.tc01/tc02.mma.sync.aligned.m16n16k32, ppu.cp.async.cg.shared.global, ppu.tc01/tc02.ldmatrix.sync.aligned, ppu.cvta.to.shared.u32

##### 2.4.9.8. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.10. new_delete - 设备端链表（new/delete + 虚函数）

##### 2.4.10.1. 描述

本示例演示在 PPU 设备端的 **C++ 动态内存管理与多态**：

在设备上构建一个**单向链表**，节点通过 `new` 在设备堆上分配，遍历时通过**虚函数**实现多态行为，并通过 `delete` 释放。

- **DataNode**：`contribute()` 返回 `value`；
- **DoubleNode**：`contribute()` 返回 `value * 2`（多态派生）。

演示的 C++ 特性：

- 设备端 `new` / `delete`（设备堆分配）；
- 虚函数（设备端虚分派）；
- 虚析构函数（正确释放派生类）；
- 指针遍历（链表遍历）；
- `hggcDeviceSetLimit(hggcLimitMallocHeapSize)` 配置设备堆大小。

##### 2.4.10.2. 关键概念

设备内存分配（Device Memory Allocation）、C++ 模板（Templates）。

##### 2.4.10.3. 支持的操作系统

Linux

##### 2.4.10.4. 支持的 CPU 架构

x86_64

##### 2.4.10.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.10.6. HGGC 运行时 API
hggcDeviceSetLimit, hggcMalloc, hggcMemset, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.4.10.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.11. power_iteration_graph - Graph Exec 参数动态更新（幂迭代法）

##### 2.4.11.1. 描述

本示例通过**幂迭代法（Power Iteration）**求矩阵最大特征值，展示 `hggcGraphExecKernelNodeSetParams` 在不重建 graph 的情况下**动态更新 kernel 参数**。

算法：
```text
v_new = A * v_old / ||A * v_old||
重复直到收敛，每步交换 v_old ↔ v_new
```

Graph 结构（通过 Stream Capture 构建）：
```text
matvec(A, src, dst) -> norm(dst) -> normalize(dst)
```

每次迭代后通过 `hggcGraphExecKernelNodeSetParams` 交换 src/dst 指针，无需重新实例化 graph。

与 `graph_conditional_nodes`（设备端 while 循环）的区别：本示例的循环在 host 端控制，graph exec 被反复 launch，但节点参数在每次 launch 前更新。

##### 2.4.11.2. 关键概念

Graph Exec 参数更新（hggcGraphExecKernelNodeSetParams）、幂迭代法、Stream Capture、迭代收敛。

##### 2.4.11.3. 支持的操作系统

Linux

##### 2.4.11.4. 支持的 CPU 架构

x86_64

##### 2.4.11.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.11.6. HGGC 运行时 API
hggcGraphExecKernelNodeSetParams, hggcGraphGetNodes, hggcStreamBeginCapture, hggcStreamEndCapture, hggcGraphInstantiate, hggcGraphLaunch, hggcGraphExecDestroy, hggcGraphDestroy, hggcFree, hggcMalloc, hggcMemcpy, hggcStreamCreateWithFlags, hggcStreamDestroy, hggcStreamSynchronize

##### 2.4.11.7. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.12. simple_hggc_graphs - Graph 节点类型全展示

##### 2.4.12.1. 描述

本示例在一个连贯的流水线中演示 HGGC 图的**所有核心节点类型**：

**演示 1：事件记录/等待节点**

- `hggcGraphAddEventRecordNode` — 在图内标记完成点；
- `hggcGraphAddEventWaitNode` — 等待来自另一个图的事件；
- 实现精确的跨图同步。

**演示 2：子图节点**

- `hggcGraphAddChildGraphNode` — 将一个完整的图作为单个节点嵌入父图；
- 实现模块化的图组合。

**演示 3：图克隆**

- `hggcGraphClone` — 复制已有图；
- `hggcGraphGetNodes` — 遍历克隆图的节点；
- 用于创建图变体（修改选定节点）。

此外还演示：核函数节点、主机回调节点（`hggcGraphAddHostNode`）。

##### 2.4.12.2. 关键概念

HGGC 图（Graphs）、流捕获（Stream Capture）。

##### 2.4.12.3. 支持的操作系统

Linux

##### 2.4.12.4. 支持的 CPU 架构

x86_64

##### 2.4.12.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.4.12.6. HGGC 运行时 API
hggcGraphCreate, hggcGraphAddKernelNode, hggcGraphAddEventRecordNode, hggcGraphAddEventWaitNode, hggcGraphAddHostNode, hggcGraphAddChildGraphNode, hggcGraphClone, hggcGraphGetNodes, hggcGraphInstantiate, hggcGraphLaunch, hggcGraphExecDestroy, hggcGraphDestroy, hggcGraphKernelNodeSetParams, hggcEventCreate, hggcEventDestroy, hggcFree, hggcMalloc, hggcMemcpy, hggcStreamCreateWithFlags, hggcStreamDestroy, hggcStreamSynchronize

##### 2.4.12.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.13. tf32_tensor_cell_gemm - TIX Tensor Cell TF32 GEMM (m16n16k8)

##### 2.4.13.1. 描述

本示例使用**纯 TIX 内联汇编**实现 TF32 精度矩阵乘法（D = A*B），演示 m16n16k8 形状的 Tensor Cell 指令用法。ppu0015 在 shared load 后交换 A1/A2，A 寄存器 i 对应 `row=lane/4+(i%2)*8`、`k=lane%4+(i/2)*4`，B 保持原加载顺序。主循环与 epilogue 使用同一加载路径及当前双缓冲索引。

与 bf16 版本（`bf16_tensor_cell_gemm`）的关键差异：

| | bf16 版本 | tf32 版本（本示例） |
|---|---|---|
| MMA 形状 | m16n16k**16** | m16n16k**8** |
| A/B 数据类型 | bf16 (2B, `.f16x2` 打包) | tf32/f32 (4B, `.b32`) |
| K-tile 大小 | 16 | 8 |
| A/B 加载方式 | ldmatrix | ppu.ld.shared.b32（逐元素） |
| C/D 布局 | 按 tc01/tc02 选择写回映射 | 同一架构下与 bf16 相同 |

核心 TIX 指令：

- `ppu.tc01/tc02.mma.sync.aligned.m16n16k8.row.col.f32.tf32.tf32.f32` — 16×16×8 MMA；
- `ppu.cp.async.cg.shared.global` — 异步拷贝；
- `ppu.ld.shared.b32` — 从共享内存加载 tf32 片段；
- `ppu.cvta.to.shared.u32` — 地址空间转换。

##### 2.4.13.2. 关键概念

矩阵乘法（Matrix Multiply）、AWMMA、Tensor Cell。

##### 2.4.13.3. 支持的操作系统

Linux

##### 2.4.13.4. 支持的 CPU 架构

x86_64

##### 2.4.13.5. 支持的 PPU 架构

ppu001、ppu0015

通过 `__HGGC_ARCH__` 100/150 选择 tc01/tc02 指令与架构对应的 fragment 位置映射，CMake 对应 `ppu_10`/`ppu_15`，不使用其它架构宏代替。

##### 2.4.13.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize

##### 2.4.13.7. TIX 指令
ppu.tc01/tc02.mma.sync.aligned.m16n16k8, ppu.cp.async.cg.shared.global, ppu.ld.shared.b32, ppu.cvta.to.shared.u32

##### 2.4.13.8. 构建/运行所需依赖
CPP11

##### 2.4.13.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.4.14. memcpy_batch_async - 批量异步 D2D 拷贝 {#memcpy_batch_async}

##### 2.4.14.1. 描述

在 `-device=N` 选定设备上用 non-default stream 提交 6 组独立 D2D 拷贝，长度为 13、257、1023、4099、8191、16387 字节。确定性输入、哨兵输出，属性起始索引 `{0,3}` 分组，源/目标位置提示使用实际设备。批内不依赖执行顺序，同步后逐字节校验，不以性能提升为 PASS 条件。

目标缓冲区使用 `hggcMemsetAsync` 预填哨兵值，并与批次入队到同一个 stream。不能用默认流的 `hggcMemset` 代替：以 `hggcStreamNonBlocking` 创建的流不参与默认流的隐式同步，混用可能让预填覆盖已完成的拷贝。

SAIL 2.2.1 的 rt2 声明为 9 参数并含 `failIdx`，rt3 为 8 参数且没有 `failIdx`；样例通过函数指针重载匹配，不强转源指针数组。仅当 `failIdx` 不再是初始哨兵值时，样例才打印该索引；rt3 不提供失败项索引，失败时不打印 `failIdx`。

##### 2.4.14.2. 关键概念

Batch Async Memcpy、D2D、属性分组、non-default stream、逐字节验证。

##### 2.4.14.3. 支持的平台

Linux、x86_64；ppu001、ppu0015 的源码目标。

##### 2.4.14.4. HGGC 运行时 API

hggcGetDeviceCount, hggcSetDevice, hggcDeviceGetAttribute, hggcStreamCreateWithFlags, hggcMalloc, hggcMemcpy, hggcMemsetAsync, hggcMemcpyBatchAsync, hggcStreamSynchronize, hggcFree, hggcStreamDestroy, hggcGetErrorString

##### 2.4.14.5. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。最低要求 SAIL 2.2.0。

#### 2.4.15. array_roundtrip - 1D/2D/3D Array 往返验证 {#array_roundtrip}

##### 2.4.15.1. 描述

使用 int32 单平面 default array，依次验证 37 元素的 1D、37×11 的 2D、19×9×5 的 3D 分配、通道/形状/标志查询、完整往返、偏移子区域更新与未修改区域。2D 行距为 41 元素，3D 行距为 23 元素、切片为 11 行，检查主机 padding 哨兵。3D 两侧都有非零偏移。

Array 分配 extent 使用元素；1D/2D 的复制 X 偏移、width/count 和 pointer pitch 使用字节。含 array 的 3D `extent.width` 使用元素，而 `pos.x` 使用字节（须为元素大小的整数倍），pitch 使用字节，Y/Z 使用行/切片。本示例使用 1D ToArray/FromArray 接口演示一维 Array 往返；这些接口已标记 deprecated。

##### 2.4.15.2. 关键概念

Array、Channel Descriptor、Extent、Pitch、偏移、精确比较、padding 哨兵。

##### 2.4.15.3. 支持的平台

Linux、x86_64；ppu001、ppu0015 的源码目标。不涉及 texture、surface、layered、sparse。

##### 2.4.15.4. HGGC 运行时 API

hggcSetDevice, hggcMallocArray, hggcMalloc3DArray, hggcArrayGetInfo, hggcGetChannelDesc, hggcCreateChannelDesc, hggcMemcpyToArray, hggcMemcpyFromArray, hggcMemcpy2DToArray, hggcMemcpy2DFromArray, hggcMemcpy3D, hggcFreeArray

##### 2.4.15.5. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。最低要求 SAIL 2.2.0。

### 2.5. HGGC 库

#### 2.5.1. acdnn_conv_activation - ACDNN 卷积+ReLU 流水线

##### 2.5.1.1. 描述

本示例使用 ACDNN 库构建一个经典的 CNN 前向计算流水线：2D 卷积 + ReLU 激活。

演示流程：
1. 创建输入张量（NCHW, FP32）和卷积滤波器（KCRS），用随机数据初始化。
2. 设置 ACDNN 张量描述符、滤波器描述符、卷积描述符和激活描述符。
3. 查询卷积工作空间大小并执行卷积前向计算（`acdnnConvolutionForward`）。
4. 对卷积输出施加 ReLU 激活（`acdnnActivationForward`）。
5. 将结果拷回主机并打印。
6. 在主机端用朴素循环实现参考卷积+ReLU，验证真武 PPU 结果。

##### 2.5.1.2. 关键概念

深度学习（Deep Learning）、卷积（Convolution）、激活函数（Activation Function）、ACDNN 库。

##### 2.5.1.3. 命令行参数

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| -N=<int> | batch size | 1 |
| -C=<int> | 输入通道数 | 4 |
| -H=<int> | 输入高度 | 8 |
| -W=<int> | 输入宽度 | 8 |
| -K=<int> | 输出通道数 | 8 |
| -R=<int> | 卷积核高度 | 3 |
| -S=<int> | 卷积核宽度 | 3 |
| -device=<id> | 指定真武 PPU 设备 | 默认设备 |

##### 2.5.1.4. 支持的操作系统

Linux

##### 2.5.1.5. 支持的 CPU 架构

x86_64

##### 2.5.1.6. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.1.7. ACDNN API
acdnnCreate, acdnnDestroy, acdnnSetStream, acdnnCreateTensorDescriptor, acdnnSetTensor4dDescriptor, acdnnDestroyTensorDescriptor, acdnnCreateFilterDescriptor, acdnnSetFilter4dDescriptor, acdnnDestroyFilterDescriptor, acdnnCreateConvolutionDescriptor, acdnnSetConvolution2dDescriptor, acdnnDestroyConvolutionDescriptor, acdnnGetConvolutionForwardWorkspaceSize, acdnnConvolutionForward, acdnnCreateActivationDescriptor, acdnnSetActivationDescriptor, acdnnDestroyActivationDescriptor, acdnnActivationForward

##### 2.5.1.8. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcStreamCreate, hggcStreamDestroy, hggcDeviceSynchronize

##### 2.5.1.9. 构建/运行所需依赖

ACDNN

##### 2.5.1.10. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.2. acsolver_dn_lu_factorization - LU 分解深度演示

##### 2.5.2.1. 描述

本示例使用 acsolverDn 库对稠密矩阵执行带部分选主元的 LU 分解，并深入展示分解结果的内部结构。

与简单的 "分解→求解" 流程不同，本程序以程序方式生成测试矩阵，提取并打印 L、U 因子和主元序列，在主机端验证 P·A = L·U 的一致性，然后利用分解结果求解线性方程组 A·x = b 并与已知真解对比。

##### 2.5.2.2. 关键概念

线性代数（Linear Algebra）、LU 分解（LU Factorization）、ACSOLVER 库。

##### 2.5.2.3. 工作流程

1. 生成一个对角占优的随机矩阵 A 和已知解 x_true，计算 b = A*x_true。
2. 在真武 PPU 上执行 LU 分解（`acsolverDnDgetrf`），A 被紧凑的 L*U 存储覆盖。
3. 提取 L（单位下三角）和 U（上三角）因子，并显示主元序列。
4. 在主机端验证 P*A = L*U（重构一致性检查）。
5. 使用分解结果求解线性方程组 A*x = b（`acsolverDnDgetrs`）。
6. 与已知解 x_true 比较，计算相对残差 ||b - A*x|| / (||A||*||x||)。

##### 2.5.2.4. 命令行参数

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| -N=<int> | 矩阵阶数 | 5 |
| -seed=<int> | 随机种子 | 基于时间 |
| -device=<id> | 指定真武 PPU 设备 | 默认设备 |

##### 2.5.2.5. 支持的操作系统

Linux

##### 2.5.2.6. 支持的 CPU 架构

x86_64

##### 2.5.2.7. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.2.8. ACSOLVER API
acsolverDnCreate, acsolverDnDestroy, acsolverDnSetStream, acsolverDnDgetrf_bufferSize, acsolverDnDgetrf, acsolverDnDgetrs

##### 2.5.2.9. ACBLAS API
acblasCreate, acblasDestroy, acblasSetStream, acblasDgemm_v2

##### 2.5.2.10. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcMemset, hggcStreamCreate, hggcStreamDestroy, hggcDeviceSynchronize

##### 2.5.2.11. 构建/运行所需依赖

ACSOLVER, ACBLAS

##### 2.5.2.12. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.3. hg_jpeg - HGGCJPEG 简单示例

##### 2.5.3.1. 描述

本示例演示使用 hgjpeg 进行批量 JPEG 解码，并在设备端计算像素统计信息。JPEG 文件从目录中加载，通过 `hgjpegDecodeBatched` 批量解码，随后使用自定义 PPU 归约核函数计算每通道像素统计（最小值/最大值/均值），展示 hgjpeg 与核函数的互操作。

流程：

1. 从目录加载 JPEG 文件。
2. 通过 `hgjpegGetImageInfo` 查询图像元数据（尺寸、通道数、子采样）。
3. 通过 `hgjpegDecodeBatched` 批量解码。
4. 在设备端计算每通道像素统计（最小值/最大值/均值）。
5. 打印包含元数据与统计信息的汇总表。

##### 2.5.3.2. 关键概念

HGGCJPEG 库、批量解码（Batch Decoding）、像素统计（Pixel Statistics）、设备核函数归约（Device Kernel Reduction）、图像处理（Image Processing）。

##### 2.5.3.3. 支持的操作系统

Linux

##### 2.5.3.4. 支持的 CPU 架构

x86_64

##### 2.5.3.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.3.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcMemset, hggcFree, hggcDeviceSynchronize

##### 2.5.3.7. HGGCJPEG API
hgjpegCreateEx, hgjpegJpegStateCreate, hgjpegDecodeBatchedInitialize, hgjpegGetImageInfo, hgjpegDecodeBatched, hgjpegJpegStateDestroy, hgjpegDestroy

##### 2.5.3.8. 构建/运行所需依赖
HGGCJPEG

##### 2.5.3.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.4. hg_jpeg_encoder - HGGCJPEG 编码器

##### 2.5.4.1. 描述

本示例解码 JPEG 图像，并以多个质量等级（30/50/70/90）重新编码，以展示质量与体积之间的权衡。对每张图像，报告各质量等级下的编码体积以及原始文件大小，随后将最高质量的输出写入磁盘。

每张图像的处理流程：

1. 通过 `hgjpegDecode` 解码 JPEG，得到设备端像素数据。
2. 对每个质量等级 Q ∈ {30, 50, 70, 90}：
    - 通过 `hgjpegEncoderParamsSetQuality` 设置质量。
    - 通过 `hgjpegEncodeImage` 编码。
    - 通过 `hgjpegEncodeRetrieveBitstream` 获取码流大小。
3. 打印质量与体积对比表。
4. 将最高质量的输出写入磁盘。

##### 2.5.4.2. 关键概念

HGGCJPEG 库、JPEG 编码（JPEG Encoding）、质量-体积权衡（Quality-Size Tradeoff）、转码（Transcoding）、图像压缩（Image Compression）。

##### 2.5.4.3. 支持的操作系统

Linux

##### 2.5.4.4. 支持的 CPU 架构

x86_64

##### 2.5.4.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.4.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcDeviceSynchronize

##### 2.5.4.7. HGGCJPEG API
hgjpegCreate, hgjpegJpegStateCreate, hgjpegEncoderStateCreate, hgjpegEncoderParamsCreate, hgjpegEncoderParamsSetQuality, hgjpegEncoderParamsSetSamplingFactors, hgjpegGetImageInfo, hgjpegDecode, hgjpegEncodeImage, hgjpegEncodeRetrieveBitstream, hgjpegEncoderParamsDestroy, hgjpegEncoderStateDestroy, hgjpegJpegStateDestroy, hgjpegDestroy

##### 2.5.4.8. 构建/运行所需依赖
HGGCJPEG

##### 2.5.4.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.5. jit_lto - 使用 libhgJitLink 的 Saxpy

##### 2.5.5.1. 描述

本示例演示对两个分别编译的模块进行运行时 JIT 链接，使用混合输入类型：

- **模块 A（LTO IR）**：使用 `-dlto` 编译的 JIT kernel（`weighted_average`），调用外部设备函数（`blend`）。
- **模块 B（HGBIN）**：`blend` 设备函数的实现，编译为普通 HGBIN（不含 LTO）。

与纯 LTO 链接示例不同，本示例展示混合输入场景：JIT 编译的 kernel 链接到预编译的库函数，这是插件架构和可热更新 kernel 的常见模式。

算法：加权平均（`out[i] = w*x[i] + (1-w)*y[i]`），并与 CPU 参考结果进行校验。

##### 2.5.5.2. 关键概念

HGGC 驱动 API、运行时编译（HGRTC）、JIT 链接（hgJitLink）、LTO、LTO IR 与 HGBIN 混合链接。

##### 2.5.5.3. 支持的操作系统

Linux

##### 2.5.5.4. 支持的 CPU 架构

x86_64

##### 2.5.5.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.5.6. HGRTC API
hgrtcCreateProgram, hgrtcCompileProgram, hgrtcGetLTOIR, hgrtcGetHGBIN, hgrtcGetProgramLog, hgrtcDestroyProgram

##### 2.5.5.7. hgJitLink API
hgJitLinkCreate, hgJitLinkAddData, hgJitLinkComplete, hgJitLinkGetLinkedHgbin, hgJitLinkDestroy, hgJitLinkVersion

##### 2.5.5.8. HGGC 驱动 API
hgInit, hgDeviceGet, hgDeviceGetAttribute, hgCtxCreate, hgCtxDestroy, hgCtxSynchronize, hgModuleLoadData, hgModuleGetFunction, hgModuleUnload, hgMemAlloc, hgMemFree, hgMemcpyHtoD, hgMemcpyDtoH, hgLaunchKernel

##### 2.5.5.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.6. simple_acblas_lu - 简单 ACBLAS LU

##### 2.5.6.1. 描述

本示例使用 **BLAS Level 1/2/3 基本运算** 实现带部分选主元的 LU 分解，展示基础 BLAS 运算如何组合成更高层的线性代数算法。

本示例并非调用高层的 `getrfBatched`，而是逐步构建分解过程：

- `acblasDswap`：行交换（选主元）。
- `acblasDscal`：列缩放（归一化）。
- `acblasDger`：秩 1 更新（Schur 补）。
- `acblasDgemm`：验证 L*U = P*A。
- `acblasDtrsm`：三角求解（Ax=b 的前代/回代）。

算法：

```text
for k = 0..n-1:
  1. 查找主元（主机端列最大值搜索）
  2. acblasDswap —— 交换第 k 行与主元行
  3. acblasDscal —— 将对角线以下元素按 1/A[k,k] 缩放
  4. acblasDger  —— 对尾部子矩阵做秩 1 更新
```

验证：通过 `acblasDgemm` 计算 L*U，并与置换后的 P*A 比较。

线性求解：通过 `acblasDtrsm` 进行前代（Ly=Pb）与回代（Ux=y）。

##### 2.5.6.2. 关键概念

ACBLAS 库、LU 分解（LU Decomposition）、BLAS Level 1/2/3、部分选主元（Partial Pivoting）、Doolittle 分解（Doolittle Factorization）。

##### 2.5.6.3. 支持的操作系统

Linux

##### 2.5.6.4. 支持的 CPU 架构

x86_64

##### 2.5.6.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.6.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree

##### 2.5.6.7. ACBLAS API
acblasCreate, acblasDestroy, acblasDswap, acblasDscal, acblasDger, acblasDgemm, acblasDtrsm

##### 2.5.6.8. 构建/运行所需依赖
ACBLAS

##### 2.5.6.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.7. simple_acfft - 简单 ACFFT

##### 2.5.7.1. 描述

本示例演示使用 ACFFT 进行频谱分析与频域滤波。将一个复合信号（多个正弦波之和 + 噪声）变换到频域，分析检测主导频率，使用理想矩形低通滤波器进行滤波，再通过逆 FFT 重建。

流程：

1. 在主机端构建复合信号（3 个已知频率的正弦波 + 白噪声）。
2. 通过 `acfftExecC2C` 执行正向 FFT。
3. 设备核函数：计算幅度谱。
4. 检测峰值频率分量（主机端搜索）。
5. 设备核函数：施加理想矩形低通滤波器。
6. 通过 `acfftExecC2C` 执行逆 FFT。
7. 与 CPU DFT 参考结果比较。

##### 2.5.7.2. 关键概念

ACFFT 库、频谱分析（Spectrum Analysis）、频域滤波（Frequency Domain Filtering）、低通滤波器（Low-Pass Filter）、DFT。

##### 2.5.7.3. 支持的操作系统

Linux

##### 2.5.7.4. 支持的 CPU 架构

x86_64

##### 2.5.7.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.5.7.6. HGGC 运行时 API
hggcMalloc, hggcMemcpy, hggcFree, hggcDeviceSynchronize

##### 2.5.7.7. ACFFT API
acfftPlan1d, acfftExecC2C, acfftDestroy

##### 2.5.7.8. 构建/运行所需依赖
ACFFT

##### 2.5.7.9. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.5.8. acdnn_softmax_bf16 - BF16 Softmax 正确性校验 {#acdnn_softmax_bf16}

##### 2.5.8.1. 描述与配置

本示例使用 ACDNN 库校验 BF16 softmax 的正确性，要求 SAIL 2.2.1 和对应 ACDNN 库。输入/输出采用完全打包的 NCHW 布局，数据类型为 `ACDNN_DATA_BF16`，计算模式为 `ACDNN_SOFTMAX_ACCURATE` / `ACDNN_SOFTMAX_MODE_CHANNEL`。主机 FP32 `alpha=1`、`beta=0`，每个 N/H/W 位置跨 C 归一化。

固定 N=2、H=3、W=5，依次运行 C=1/17/257：常量 -3.25、确定性非均匀输入、[-160,160] 较大正负输入，分别为 30/510/7710 个元素。输入用 round-to-nearest, ties-to-even 量化为 BF16 位模式；CPU 参考从实际上传的 BF16 数据恢复为 FP32，减通道最大值后用 double 累加稳定 softmax，不采用量化前 FP32 输入。

##### 2.5.8.2. 校验与运行

显式 non-default stream；使用页锁定主机缓冲区，H2D 初始化、BF16 NaN 哨兵填充、softmax 和 D2H 回传在同一流排队，不依赖与默认流的隐式顺序。`beta=0` 不应读取旧输出。同步后检查有限性、非负性、逐元素 `abs_error <= 5e-4 + 1e-2*abs(reference)`，以及每个 N/H/W 的通道概率和误差 `<=1e-2`。打印最大绝对误差、最大概率和误差、非法值与不匹配计数。API 错误或数值失败返回非零并停止，不放宽阈值、不自动重试、不静默跳过。

支持 Linux、x86_64，目标架构为 ppu001、ppu0015。

##### 2.5.8.3. ACDNN API

acdnnCreate, acdnnDestroy, acdnnSetStream, acdnnCreateTensorDescriptor, acdnnSetTensor4dDescriptor, acdnnDestroyTensorDescriptor, acdnnSoftmaxForward

##### 2.5.8.4. HGGC 运行时 API

hggcMalloc, hggcFree, hggcMallocHost, hggcFreeHost, hggcMemcpyAsync, hggcMemsetAsync, hggcStreamCreateWithFlags, hggcStreamSynchronize, hggcStreamDestroy

##### 2.5.8.5. 构建/运行所需依赖

HGGC::toolkit、HGGC::acdnn。

##### 2.5.8.6. 前置条件

请下载并安装适用您平台的 T-Head SAIL 软件工具包。最低要求 SAIL 2.2.1。

### 2.6. 性能

#### 2.6.1. aiu_throughput - AIU 与 cp.async 拷贝吞吐对比

##### 2.6.1.1. 描述

本示例对比两种 PPU 异步数据拷贝机制，将逐渐增大的 bf16 张量（2KB 到 32KB）从全局内存经共享内存搬运再写回：

- **cp.async**：每线程 4 字节异步拷贝，配合 commit/wait 同步。
- **AIU 批量拷贝**：每 warp 张量拷贝，配合 swizzle 与 ldmatrix.swzl 解地址加载。

测试五种张量规模（16×1×64 到 256×1×64 bf16），以展示吞吐随规模的扩展情况。两种方法均校验数据完整性（输出 == 输入），并报告 GB/s 及加速比。

##### 2.6.1.2. 关键概念

AIU 批量拷贝（AIU Bulk Copy）、cp.async、张量共享内存（Tensor Shared Memory）、ldmatrix.swzl、吞吐基准测试（Throughput Benchmarking）、扩展性分析（Scaling Analysis）。

##### 2.6.1.3. 支持的操作系统

Linux

##### 2.6.1.4. 支持的 CPU 架构

x86_64

##### 2.6.1.5. 支持的 PPU 架构

ppu001、ppu0015

设备编译时通过 `__HGGC_ARCH__` 100/150 选择 ppu001 的 3D AIU/tc01 或 ppu0015 的 2D tile AIU/tc02，CMake 对应 `ppu_10`/`ppu_15`；两条路径分别使用对应的 descriptor、地址单位与 fragment 映射。

##### 2.6.1.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcEventCreate, hggcEventRecord, hggcEventSynchronize, hggcEventElapsedTime, hggcEventDestroy, hggcDeviceSynchronize

##### 2.6.1.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.6.2. hggc_graphs_perf_scaling - HGGC 图性能扩展

##### 2.6.2.1. 描述

本示例测量 HGGC 图 API 操作（捕获、实例化、启动）随图拓扑规模的扩展表现。通过流捕获构建复杂度递增的并行链式图（10 到 200 个节点），随后对每个阶段分别计时，以展示其扩展行为。

##### 2.6.2.2. 关键概念

图捕获（Graph Capture）、图实例化（Graph Instantiation）、图启动（Graph Launch）、流捕获（Stream Capture）、性能扩展（Performance Scaling）、并行拓扑（Parallel Topology）。

##### 2.6.2.3. 支持的操作系统

Linux

##### 2.6.2.4. 支持的 CPU 架构

x86_64

##### 2.6.2.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.6.2.6. HGGC 运行时 API
hggcStreamCreate, hggcStreamDestroy, hggcStreamBeginCapture, hggcStreamEndCapture, hggcStreamSynchronize, hggcStreamWaitEvent, hggcEventCreate, hggcEventDestroy, hggcEventRecord, hggcGraphInstantiateWithFlags, hggcGraphLaunch, hggcGraphExecDestroy, hggcGraphDestroy

##### 2.6.2.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.6.3. mem_strategy_bench - 内存分配策略性能对比

##### 2.6.3.1. 描述

本示例针对一个简单的逐元素缩放核函数，对比 4 种 PPU 内存分配策略：

- **托管内存（Managed Memory）**：`hggcMallocManaged` + `hggcMemPrefetchAsync`，实现设备/主机间迁移。
- **零拷贝（Zero Copy）**：`hggcHostAlloc`（映射）+ `hggcHostGetDevicePointer`，供设备直接访问主机内存。
- **页锁定 + 异步（Pinned + Async）**：`hggcHostAlloc`（portable）+ `hggcMemcpyAsync`，实现分级传输。
- **可分页 + 同步（Pageable + Sync）**：`malloc` + `hggcMemcpy`，作为基线对比。

测试多种数据规模（256KB 到 4MB），以展示各策略随数据量的扩展表现。

##### 2.6.3.2. 关键概念

托管内存（Managed Memory）、零拷贝（Zero Copy）、页锁定内存（Pinned Memory）、异步内存拷贝（Async Memcpy）、内存分配策略（Memory Allocation Strategy）、性能基准测试（Performance Benchmarking）。

##### 2.6.3.3. 支持的操作系统

Linux

##### 2.6.3.4. 支持的 CPU 架构

x86_64

##### 2.6.3.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.6.3.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMallocManaged, hggcMemPrefetchAsync, hggcHostAlloc, hggcFreeHost, hggcHostGetDevicePointer, hggcMemcpy, hggcMemcpyAsync, hggcDeviceSynchronize

##### 2.6.3.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.6.4. smem_bank_conflict - 共享内存 Bank 冲突影响

##### 2.6.4.1. 描述

本示例以不同的访问步长（stride）测量共享内存读取吞吐，展示 bank 冲突如何降低性能。PPU 共享内存包含 32 个 bank（每个 4 字节）；当同一 warp 中的多个线程访问同一 bank 时，访问将被串行化。

测试步长从 1（无冲突）到 32（最大冲突），并报告每种访问模式的有效吞吐（GB/s）及相对效率。

##### 2.6.4.2. 关键概念

共享内存（Shared Memory）、Bank 冲突（Bank Conflict）、访问步长（Access Stride）、内存吞吐（Memory Throughput）、性能退化（Performance Degradation）。

##### 2.6.4.3. 支持的操作系统

Linux

##### 2.6.4.4. 支持的 CPU 架构

x86_64

##### 2.6.4.5. 支持的 PPU 架构

ppu001、ppu0015

##### 2.6.4.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcEventCreate, hggcEventRecord, hggcEventSynchronize, hggcEventElapsedTime, hggcEventDestroy, hggcDeviceSynchronize

##### 2.6.4.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

#### 2.6.5. transpose - 矩阵转置

##### 2.6.5.1. 描述

本示例针对 bf16 方阵对比三种矩阵转置方法：

- **原生实现（Native）**：使用 16×16 tile 的共享内存分块，并通过填充避免 bank 冲突。
- **AIU + ldmatrix 实现**：AIU 批量拷贝到 TSM + ldmatrix.swzl 解地址 + 转置写回（64×64 tile，每个 tile 4 个 warp）。
- **AIU pipeline 实现**：生产者-消费者双缓冲 + awbarrier 同步（64×64 tile，1 个生产者 warp + 4 个消费者 warp，cp.async.awbar.arrive 关联 AIU 拷贝）。

测试多种矩阵规模（64×64 到 16384×16384）以展示吞吐扩展。三种方法均校验正确性（输出 == 输入的转置）。

##### 2.6.5.2. 关键概念

矩阵转置（Matrix Transpose）、共享内存分块（Shared Memory Tiling）、AIU 批量拷贝（AIU Bulk Copy）、ldmatrix.swzl、Bank 冲突规避（Bank Conflict Avoidance）、吞吐基准测试（Throughput Benchmarking）。

##### 2.6.5.3. 支持的操作系统

Linux

##### 2.6.5.4. 支持的 CPU 架构

x86_64

##### 2.6.5.5. 支持的 PPU 架构

ppu001、ppu0015

设备编译时通过 `__HGGC_ARCH__` 100/150 选择 ppu001 的 3D AIU/tc01 或 ppu0015 的 2D tile AIU/tc02，CMake 对应 `ppu_10`/`ppu_15`；两条路径分别使用对应的 descriptor、地址单位与 fragment 映射。

##### 2.6.5.6. HGGC 运行时 API
hggcMalloc, hggcFree, hggcMemcpy, hggcEventCreate, hggcEventRecord, hggcEventSynchronize, hggcEventElapsedTime, hggcEventDestroy, hggcDeviceSynchronize

##### 2.6.5.7. 前置条件
请下载并安装适用您平台的 T-Head SAIL 软件工具包。

## 3. 依赖项

部分 HGGC 示例程序依赖第三方应用程序和/或库，或依赖 T-Head SAIL 软件工具包和驱动程序提供的特性，才能完成构建或执行。以下列出了这些依赖项。

如果某个示例程序所依赖的第三方库在系统上可用但未安装，该示例程序将在构建时自动跳过。

每个示例程序的依赖项列在其 README 的“依赖项”章节中。

### 3.1. 第三方依赖

以下第三方依赖项被部分 HGGC 示例程序所需。如果可用，这些依赖项通常已自动安装在系统上，或可通过系统包管理器（Linux）或第三方网站安装。

#### 3.1.1. 消息传递接口（MPI）

MPI（消息传递接口）是用于分布式进程间数据通信的 API。可通过 Linux 发行版的包管理器安装 MPI 编译器，也可从 Open MPI 等在线资源获取。

#### 3.1.2. OpenMP

OpenMP 是用于多进程编程的 API，可通过 Linux 发行版的包管理器安装，通常随 GCC 预装，也可从 OpenMP 官网获取。

### 3.2. HGGC 特性

部分 HGGC 示例程序展示了以下 HGGC 特性：

#### 3.2.1. 多块协作组

多块协作组（MBCG）扩展了协作组和 HGGC 编程模型，以表达线程块间的同步。

#### 3.2.2. 多设备协作组

多设备协作组扩展了协作组和 HGGC 编程模型，使在多个 PPU 上执行的线程块能够协作和同步。

#### 3.2.3. HGRTC

HGRTC（HGGC 运行时编译）是用于 HGGC C++ 的运行时编译库。

#### 3.2.4. 统一虚拟内存

UVM（统一虚拟内存）支持 CPU 和 PPU 无需显式复制即可访问同一块内存。UVM 仅在 Linux 系统上可用。

#### 3.2.5. 16 位浮点数

FP16 是 16 位浮点格式。1 位用于符号位，5 位用于指数，10 位用于尾数。

#### 3.2.6. C++11 HGGC

hgcc 支持 C++11 特性。

## 4. 常见问题

有关 HGGC 常见问题的解答，请参阅 [T-Head SAIL 软件工具包发布说明](/docs_center/doc_detail/index.html?projectId=39&chapterId=1038)。
