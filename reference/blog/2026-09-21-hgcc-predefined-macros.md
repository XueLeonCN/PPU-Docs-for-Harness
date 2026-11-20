---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=43"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=43"
source_title: "hgcc 预定义宏使用介绍"
source_id: "blog-43"
source_kind: "technical-blog"
source_date: "2026-09-21"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "c8e93e967cf8ea961a1b922fc2610329a396015b26d1934603e265e9ddb16120"
tags:
  - "hgcc"
  - "predefined-macros"
  - "__HGGC__"
  - "__HGGC_ARCH__"
  - "feature-macro"
  - "version-macro"
  - "RDC"
  - "DEBUG"
  - "conditional-compilation"
---
# hgcc 预定义宏使用介绍 { #hgcc-predefined-macros }

hgcc 会对同一份 `.hg` 源文件执行多次编译——主机端编译一次，每个目标架构再各编译一次——而预定义宏在不同编译端的取值并不相同。「宏已定义却未生效」「同一段代码在主机端与设备端行为不一致」这类问题，多数源于此。本文先介绍编译流程与查看宏的方法，再按宏逐个展开：`__HGGC__` / `__HGGCCC__` 的可移植写法、`__HGGC_ARCH__` 的取值与四条使用约束（其中一条会导致静默出错）、特性测试宏与版本宏的用法，以及 `__HGGCCC_RDC__` 与 `__HGGCCC_DEBUG__` 的条件编译用法。

!!! note
    后续示例基于 hgcc `2.2.0`、Ubuntu 24.04、主机编译器 `gcc 13.3.0`。

## 1. 背景介绍 { #multiple-passes }

`.hg` 文件同时包含主机端代码与设备端代码，会经历多次编译。hgcc 的编译流程中，设备端编译先对源文件做一次预处理，主机端编译再对同一份源文件做一次预处理——同一份源文件被预处理了两次；指定多个目标架构时，设备端还需按架构分别编译。

因此，同一个宏在不同编译端的定义与取值可能不同，源代码一旦依赖这种取值，就容易产生宏未按预期生效、两端行为不一致的问题。这也是本文对 hgcc 预定义宏做详细说明的原因：第 2 节介绍查看宏取值的方法，第 3 节给出完整清单，并按宏逐个展开定义条件、取值与使用约束。

`--dryrun` 会输出 hgcc 内部真实执行的命令。从输出中筛出所有的 `-D`（`-arch` 取默认值 `ppu_10`）：

```bash
hgcc --dryrun empty.hg -o empty 2>&1 | tr ' ' '\n' | grep -E '^-[DU]' | sort -u
```

主机端与设备端各自获得的宏定义如下：

| 宏 | 主机端 | 设备端 |
| :--- | :---: | :---: |
| `-D__HGGC__` | 定义 | 定义 |
| `-D__HGGCCC__` | 定义 | 定义 |
| `-D__HGGCCC_VER_MAJOR__=2` | 定义 | 定义 |
| `-D__HGGCCC_VER_MINOR__=2` | 定义 | 定义 |
| `-D__HGGCCC_VER_BUILD__=0` | 定义 | 定义 |
| **`-D__HGGC_ARCH__=100`** | 未定义 | 定义 |

表中唯一的差异是 `__HGGC_ARCH__`：它仅在设备端编译中定义，第 3.2 节将完整讨论它带来的影响。其余宏在两端都定义——版本宏、`__HGGC__`、`__HGGCCC__` 因此可以在主机代码中直接使用。

**`--dryrun` 属于调试手段，不是稳定接口。** 它输出的具体步骤可能随版本迭代变化，可以用来解释现象，但不应写入构建脚本。

## 2. 查看宏的取值 { #inspect-macros }

排查宏相关的问题，第一步是导出预处理器定义。`-E`（`--preprocess`）在 hgcc 中对应 C/C++ 预处理阶段，默认输出文件名形如 `a.hggci`。`gcc` 中常用的 `-dM` 不能直接传给 hgcc：

```text
$ hgcc -E -dM empty.hg
hgcc error: unknown argument '-dM', did you mean '-M'?
```

`-dM` 是主机编译器/预处理器的选项，需要通过 `-Xcompiler`（`--compiler-options`）透传：

```bash
hgcc -E -Xcompiler -dM empty.hg > macros.txt
wc -l macros.txt        # 2389
grep __HGGC macros.txt
```

hgcc 2.2.0 上共导出 2389 行 `#define`，但导出的只是设备端编译的结果——其中包含 `#define __HGGC_ARCH__ 100`，而主机端编译没有该宏。

结合上面两条命令，排查方式归纳为三种：

| 目的 | 命令 |
| :--- | :--- |
| 列出**设备端**的全部宏 | `hgcc -E -Xcompiler -dM x.hg` |
| 查看 hgcc **注入**了哪些 `-D`（可区分编译端） | `hgcc --dryrun x.hg -o x 2>&1 \| tr ' ' '\n' \| grep '^-D'` |
| 确定**某一编译端**中某个宏的实际取值 | 在源码中写 `#pragma message` |

前两种方式给出的是按编译端汇总的清单，第三种方式则能精确到某一行代码是在哪一端被编译的——`#pragma message` 以 warning 形式输出，并标明所属的编译端：

```cpp
// which_pass.hg
#ifdef __HGGC_ARCH__
#pragma message("DEVICE PASS, __HGGC_ARCH__ present")
#else
#pragma message("HOST PASS, __HGGC_ARCH__ absent")
#endif
__global__ void k() {}
int main() { return 0; }
```

```text
$ hgcc -c which_pass.hg -o which_pass.o
which_pass.hg:2:9: warning: DEVICE PASS, __HGGC_ARCH__ present [-W#pragma-messages]
1 warning generated when compiling for ppu001.
which_pass.hg:4:9: warning: HOST PASS, __HGGC_ARCH__ absent [-W#pragma-messages]
1 warning generated when compiling for host.
```

末尾的 `when compiling for ppu001` / `when compiling for host` 即编译端标识，后文多处输出都依靠它定位。

## 3. 预定义宏详解 { #macro-list }

掌握了查看方法，先看宏的全貌。下表列出 hgcc 的全部预定义宏。「启用条件」列是它们的定义条件，右两列是 hgcc 2.2.0 的取值——除 `__HGGC_ARCH__` 仅在设备端编译中定义、取值随 `-arch` 变化外，其余宏在条件满足时两端都定义为 `1`（版本宏为对应整数），条件不满足时两端都不定义：

| 宏 | 启用条件 | 主机端 | 设备端 |
| :--- | :--- | :---: | :---: |
| `__HGGC__` | 无条件，编译 C/C++/HGGC 源文件时定义 | `1` | `1` |
| `__HGGCCC__` | 无条件，编译 HGGC 源文件时定义 | `1` | `1` |
| `__HGGC_ARCH__` | 无条件，但仅在设备端编译中生效 | 未定义 | `100`（随 `-arch` 变化） |
| `__HGGCCC_EXTENDED_LAMBDA__` | `--expt-extended-lambda` / `--extended-lambda` | `1` | `1` |
| `__HGGCCC_RELAXED_CONSTEXPR__` | `--expt-relaxed-constexpr` | `1` | `1` |
| `__HGGCCC_VER_MAJOR__` | 无条件，hgcc 主版本号 | `2` | `2` |
| `__HGGCCC_VER_MINOR__` | 无条件，hgcc 次版本号 | `2` | `2` |
| `__HGGCCC_VER_BUILD__` | 无条件，hgcc 构建版本号 | `0` | `0` |
| `__HGGCCC_RDC__` | `-rdc=true` / `-dc`（`-dlto` 单独使用同样触发） | `1` | `1` |
| `__HGGCCC_DEBUG__` | `-G` | `1` | `1` |

其中 `__HGGC_ARCH__` 与 `__HGGCCC_DEBUG__` 分别记载于 hgcc 文档的架构宏小节与编程指南的调试选项小节。下面按宏逐个展开其定义条件与用法。

### 3.1. `__HGGC__` / `__HGGCCC__`：判断是否由 hgcc 编译 { #hggc-hggccc }

这两个宏用来编写「既可由 hgcc 编译、也可由 g++ 直接编译」的公共头文件。它们的定义条件分别是：`__HGGC__` 在「编译 C/C++/HGGC 源文件时」定义，`__HGGCCC__` 在「编译 HGGC 源文件时」定义。

最常见的用途是为设备限定符提供可移植定义，使同一个头文件在纯主机编译时也能通过编译：

```cpp
// portable.h —— 既可由 hgcc 编译，也可由 g++ 单独编译
#ifdef __HGGCCC__
  #define XPU_HD  __host__ __device__
#else
  #define XPU_HD               // g++ 不支持 __device__，退化为普通函数
#endif

XPU_HD inline int clamp01(int x) { return x < 0 ? 0 : (x > 1 ? 1 : x); }
```

#### 3.1.1. `.cpp` 文件不注入任何宏 { #cpp-files }

定义条件中的「C/C++ 源文件」容易引起误解：`.cpp` 交给 hgcc 编译并不会得到这些宏。

hgcc 处理 `.cpp` 时，只是原样转发给 `g++`，既不注入任何 `-D`，也不添加 SDK 的 include 路径。`.cpp` 的实际编译命令只有：

```bash
g++ -x c++ hostonly.cpp -c -w -fpermissive -m64 -o hostonly.o
```

于是一个只使用 Runtime API 的纯主机文件会直接编译失败：

```cpp
// hostonly.cpp
#include <stdio.h>
#include <hggc_runtime.h>
int main() { int n = 0; hggcGetDeviceCount(&n); printf("%d\n", n); return 0; }
```

```text
$ hgcc -c hostonly.cpp -o hostonly.o
hostonly.cpp:2:10: fatal error: hggc_runtime.h: No such file or directory
    2 | #include <hggc_runtime.h>
      |          ^~~~~~~~~~~~~~~~
compilation terminated.
```

两种解决方式：

```bash
# 方法 1：显式指定 include 路径（此时仍然没有任何 __HGGC* 宏）
hgcc -c hostonly.cpp -o hostonly.o -I$PPU_SDK/include

# 方法 2：强制按 HGGC 源文件处理（此时才会执行完整的两端编译并注入全部宏）
hgcc -x hg -c hostonly.cpp -o hostonly.o
```

`-x` 只接受 `c`、`c++`、`hg` 三个值。

综上，需要 `__HGGC*` 宏或 Runtime API 头文件的源文件，扩展名应使用 `.hg`，或显式指定 `-x hg`。

### 3.2. `__HGGC_ARCH__`：仅在设备端编译中定义的宏 { #hggc-arch }

`__HGGC_ARCH__` 是所有预定义宏中唯一区分编译端的宏，也是唯一带有成文使用约束的宏，因此单独用一节的篇幅展开。

#### 3.2.1. 取值与作用域 { #arch-basics }

`__HGGC_ARCH__` 表示设备代码面向的 PPU 架构，其值与 `-arch=ppu_<version>` 的关系是 `__HGGC_ARCH__ = <version> * 10`。它仅在设备端编译中定义，主机端编译中不存在，因此可以用来区分主机代码与设备代码。

`-arch` 与 `__HGGC_ARCH__` 的对应关系：

| `-arch` | `__HGGC_ARCH__` | 编译端标识（`compiling for …`） |
| :--- | :--- | :--- |
| `ppu_10`（默认） | `100` | `ppu001` |
| `ppu_15` | `150` | `ppu0015` |
| `vm_10` | `100` | `ppu001` |
| `vm_15` | `150` | `ppu0015` |

该表有三点需要说明。

第一，**它的作用范围是整个翻译单元，而非单个函数**。设备端编译时整个翻译单元都能看到 `__HGGC_ARCH__`，主机端编译时整个翻译单元都看不到——主机端预处理 `__global__` 函数体时同样看不到，3.2.4 节的报错正源于此。也正因为它作用于整个翻译单元，可以出现在文件作用域的预处理指令中，用来控制 `static` 变量的定义位置：

```cpp
#ifdef __HGGC_ARCH__
  static __device__ int var2; // 正确，声明仅在设备编译期间可见
#else
  static int var3;            // 正确，声明仅在主机编译期间可见
#endif
```

第二，**真实架构与虚拟架构共用同一套编号**。真实架构 `ppu_XX` 编译为目标硬件可直接执行的 hgbin 二进制代码；虚拟架构 `vm_XX` 编译为 LLVM bitcode 中间表示，运行时由驱动程序即时编译（JIT）为目标硬件的 hgbin。两者按同一个 `<version> * 10` 公式取值，因此 `vm_10` 与 `ppu_10` 都得到 `100`，编译端标识也相同。据此，`__HGGC_ARCH__` 无法区分真实架构与虚拟架构。

第三，**宏的值反映的是编译目标，而非运行时的实际硬件**。依赖它做能力判断时，应将其理解为「该设备代码所面向的架构等级」。

#### 3.2.2. 区分主机路径与设备路径 { #host-vs-device-path }

区分编译端最直接的用途，是让同一个 `__host__ __device__` 函数在主机端与设备端分别选用不同实现。

```cpp
// hostdev.hg
#include <stdio.h>

// 一份源码，两套实现，由 __HGGC_ARCH__ 决定使用哪一套
__host__ __device__ int which_path(int x) {
#ifdef __HGGC_ARCH__
    return x * 10 + 1;   // 设备路径
#else
    return x * 10 + 2;   // 主机路径
#endif
}

__host__ __device__ const char *path_name() {
#ifdef __HGGC_ARCH__
    return "device";
#else
    return "host";
#endif
}

__global__ void k(int *out) {
    out[0] = which_path(7);
#ifdef __HGGC_ARCH__
    out[1] = __HGGC_ARCH__;
#else
    out[1] = -1;
#endif
    printf("[device] path_name()=%s which_path(7)=%d\n", path_name(), out[0]);
}

int main() {
    printf("[host]   path_name()=%s which_path(7)=%d\n", path_name(), which_path(7));
    int *d, h[2] = {-9, -9};
    hggcMalloc((void **)&d, 2 * sizeof(int));
    k<<<1, 1>>>(d);
    hggcDeviceSynchronize();
    hggcMemcpy(h, d, 2 * sizeof(int), hggcMemcpyDeviceToHost);
    printf("[host]   kernel wrote which_path=%d, __HGGC_ARCH__=%d\n", h[0], h[1]);
    hggcFree(d);
    return 0;
}
```

```text
$ hgcc hostdev.hg -o hostdev
$ ./hostdev
[host]   path_name()=host which_path(7)=72
[device] path_name()=device which_path(7)=71
[host]   kernel wrote which_path=71, __HGGC_ARCH__=100
```

同一个函数，主机侧调用返回 72，kernel 内调用返回 71；`__HGGC_ARCH__` 在设备端取值为 `100`。

更常见的场景是 CPU 参考实现与 PPU 实现共用一个函数体，只在需要设备内建函数的地方分支：

```cpp
__host__ __device__ float fast_rsqrt(float x) {
#ifdef __HGGC_ARCH__
    return rsqrtf(x);          // 设备端内建，走硬件近似指令
#else
    return 1.0f / sqrtf(x);    // 主机端常规计算，作为对照基线
#endif
}
```

#### 3.2.3. 按架构选实现 { #arch-selection }

除了区分主机与设备，宏的取值本身也可以作为分支条件，按架构选择实现：

```cpp
#if __HGGC_ARCH__ >= 150
    // ppu0015 专属优化路径（如 FP8 / 预取提示）
#else
    // ppu001 通用路径
#endif
```

多架构编译时各编译端分别取到什么值，可以用 `#pragma message` 打印：

```text
$ hgcc -arch=ppu_10,ppu_15 -c pm.hg -o pm_comma.o
pm.hg:8:9: warning: >>> pass with __HGGC_ARCH__ == 100 [-W#pragma-messages]
3 warnings generated when compiling for ppu001.
pm.hg:10:9: warning: >>> pass with __HGGC_ARCH__ == 150 [-W#pragma-messages]
3 warnings generated when compiling for ppu0015.
pm.hg:5:55: note: '#pragma message: HOST PASS: __HGGC_ARCH__ not defined'
```

**一次 `-c` 触发三次编译**：两次设备端（`ppu001` 取 100、`ppu0015` 取 150）加一次主机端（未定义）。

`-arch` 支持以下几种写法：

| 写法 | 结果 |
| :--- | :--- |
| `-arch=ppu_10` | 设备端编译 1 次（`ppu001`） |
| `-arch=ppu_10,ppu_15` | 逗号列表，设备端编译 2 次 |
| `-arch=ppu_10 -arch=ppu_15` | 重复指定，等价于逗号列表 |
| `-arch=all` / `all-ppu` / `all-vm` | **三者的设备端编译完全相同**（`ppu001` + `ppu0015`） |
| `--gpu-architecture=all-ppu` | 长选项形式 |

多架构编译的代价体现在目标文件体积上：`-arch=all` 约 174 KB（178080 字节），单架构约 38 KB（38816 字节）。在 CMake 工程中，架构列表由 `CMAKE_HG_ARCHITECTURES` 控制——HGGC 示例工程默认同时为 ppu001（`--gpu-architecture=ppu_10`）和 ppu0015（`--gpu-architecture=ppu_15`）两种架构构建，如需仅面向某一种架构，可在配置阶段设置：

```bash
cmake -DCMAKE_HG_ARCHITECTURES=ppu_10 -B build
```

在第一代硬件上运行时，实际被选中的 image 如下：

| `-arch` | 运行结果 |
| :--- | :--- |
| 默认 / `ppu_10` / `ppu_10,ppu_15` / `all` / `all-ppu` | `__HGGC_ARCH__=100`，`err=no error` |
| **`ppu_15`** | kernel **未执行**，`err=invalid argument` |
| **`all-vm`** | kernel **未执行**，`err=operation not supported` |

fat binary 在运行时按当前设备选择 image：同时编译两个架构时，在第一代硬件上选中 `ppu_10` image；表中第二行只生成了 `ppu_15` image，在第一代硬件上没有可运行的 image。

第三行的 `all-vm` 需要单独说明。虚拟架构产物是 LLVM bitcode，由驱动程序在运行时 JIT 编译为 hgbin，其行为取决于驱动而非编译器，上表中的 `operation not supported` 即出现在 JIT 未生效的环境。编译期这一侧的行为是确定的：`-arch=all-vm` 的设备端编译产物与 `-arch=all` / `all-ppu` 完全相同，并没有生成一套独立的「虚拟架构专用」image。

**架构不匹配是静默失败**：`hggcDeviceSynchronize()` 仍然返回 `no error`，只有 `hggcGetLastError()` 才报出 `invalid argument`。若错误检查只调用前者，架构不匹配就表现为 kernel 静默不执行——启动 kernel 之后必须检查 `hggcGetLastError()`。

#### 3.2.4. 只能用于预处理指令 { #arch-without-ifdef }

无论按编译端分支还是按架构分支，`__HGGC_ARCH__` 始终只出现在预处理指令里。直接把它用在表达式中一律不可行，写在 `__global__` 函数体内也不例外——主机端编译会预处理整个文件，kernel 的函数体也在其中。

```cpp
// bad.hg
__global__ void k(int *o) { o[0] = __HGGC_ARCH__; }   // 没有 #ifdef 保护
int main() { return 0; }
```

```text
$ hgcc bad.hg -o bad
bad.hg:2:42: error: use of undeclared identifier '__HGGC_ARCH__'
1 error generated when compiling for host.
```

报错发生在 `compiling for host`：代码写在 `__global__` 函数体内，报错的却是主机端编译——这正是 3.2.1 节所述「作用范围是整个翻译单元而非单个函数」的直接体现。

因此，`__HGGC_ARCH__` 只能用于预处理指令（`#ifdef` / `#if` / `#elif`），或出现在已被 `#ifdef __HGGC_ARCH__` 保护的代码中。

```cpp
// 正确
__global__ void k(int *o) {
#ifdef __HGGC_ARCH__
    o[0] = __HGGC_ARCH__;
#else
    o[0] = -1;
#endif
}
```

#### 3.2.5. 与 `__launch_bounds__` 配合时的陷阱 { #arch-launch-bounds }

由架构宏推导的常量如果进一步用于 `__launch_bounds__`，还会引出一个更隐蔽的陷阱。先看按架构定制启动界限的写法：

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

错误出在下一步：`kernel<<<blocks, MY_KERNEL_MAX_THREADS>>>()` 直接复用由架构宏推导的常量。主机代码中 `__HGGC_ARCH__` 未定义，会取 `#else` 分支的值。

而 `#if __HGGC_ARCH__ >= 100` 这种写法不会像 3.2.4 节那样报错：预处理指令中，未定义的标识符被当作 `0`，于是主机端编译时无提示地进入 `#else` 分支。结果：

- 设备端：`__launch_bounds__(512, 3)`，kernel 编译成最大 512 线程；
- 主机端：`MY_KERNEL_MAX_THREADS` 是 **256**，`<<<blocks, 256>>>` 只用了一半的线程。

设备端取到较大界限、启动配置用了较小值，这个方向的错误只是性能损失；反过来（设备端界限较小、启动配置超出界限）就会在运行期失败——`__launch_bounds__` 的上限在运行时是强制的。下面的输出即对应这一情形：`__launch_bounds__` 限定 256 线程，程序依次以 256、1024、512 线程启动，超出界限的两次启动均被拒绝：

```text
launch 256 (within bounds): err=no error
launch 1024 (EXCEEDS bounds 256): err=701 too many resources requested for launch
launch 512 (EXCEEDS bounds 256): err=701 too many resources requested for launch
```

正确的做法是把启动用的常量与 `__HGGC_ARCH__` 解耦，让主机端和设备端看到同一个值：

```cpp
#define LAUNCH_BLOCK_SIZE 256          // 主机、设备都看到同一个值
__global__ void __launch_bounds__(LAUNCH_BLOCK_SIZE) k(float *p);
// 主机端： k<<<blocks, LAUNCH_BLOCK_SIZE>>>(p);
```

另有一条约束：`__launch_bounds__` 与 `__maxnreg__` 不能用于同一个 kernel，两种书写顺序都报错：

```text
both.hg:1:55: error: The __launch_bounds__ and __maxnreg__ qualifiers cannot be applied to the same kernel
1 error generated when compiling for host.
```

分离编译下 `__launch_bounds__` 还有一条跨编译单元的约束：限制了寄存器数量的 kernel，其调用的所有函数也必须遵守该限制，否则会在设备链接阶段失败。详见相关阅读 [hgcc 分离编译（RDC 模式）](/blogDetail?bizType=BLOG&id=42) 第 7.5 节。

#### 3.2.6. 四条使用约束 { #arch-constraints }

文档将四条约束分为两组——约束 1–2 为分离编译约束，约束 3–4 为类型与实例化约束——并在末尾声明：「编译器不能保证会为上述不支持的 `__HGGC_ARCH__` 用途生成诊断信息。」这四条约束中后果最严重的一条，编译器不会给出任何诊断信息。

##### 约束 1：外部链接实体的存在性不应依赖 `__HGGC_ARCH__` { #arch-constraint-1 }

不要将 `__device__` 全局变量或函数的定义置于架构条件之下。下面把 `__device__` 变量限定为仅在设备端编译中存在：

```cpp
// c1a.hg
#ifdef __HGGC_ARCH__
__device__ int only_in_device_pass = 1;
#endif
```

```text
$ hgcc -rdc=true -c c1a.hg -o c1a.o
/tmp/c1a-host-5ZTqzz.cpp: In function 'void __hggc_register_globals(void**)':
/tmp/c1a-host-5ZTqzz.cpp:14:40: error: '::only_in_device_pass' has not been declared
hgcc error: Exited with error code 1
```

报出这一错误的不是前端检查，而是 hgcc 自动生成的主机侧注册桩（`__hggc_register_globals`）：设备端定义了该变量，注册代码就要引用它，但主机端看不到，于是 g++ 报错。这是附带触发的诊断，并非针对该约束设计。绕过它之后，设备链接阶段给出的报错同样不指向问题根源。**这条约束缺少针对性的诊断**，只能依靠代码审查规避。

##### 约束 2：不得在头文件中通过 `__HGGC_ARCH__` 改变共享实体的行为 { #arch-constraint-2 }

典型的反例是：`a.h` 中有一个模板函数，函数体以 `#if __HGGC_ARCH__ >= 150` 分支；`a.hg` 与 `b.hg` 都包含该头文件，却以不同的 `-arch` 编译。

```cpp
// a.h
template <typename T>
__device__ T pick(T x) {
#if __HGGC_ARCH__ >= 150
    return x * 100;   // "ppu_15 tuned"
#else
    return x * 1;     // "ppu_10 baseline"
#endif
}

// a.hg
#include "a.h"
__device__ int from_a() { return pick(3); }

// b.hg
#include "a.h"
extern __device__ int from_a();
__device__ int from_b() { return pick(4); }
__global__ void k(int *o) { o[0] = from_a() + from_b(); }
```

架构不一致时，链接会直接失败：

```text
$ hgcc --gpu-architecture ppu_10 -dc a.hg
$ hgcc --gpu-architecture ppu_15 -dc b.hg
$ hgcc --gpu-architecture ppu_15 a.o b.o -o m7app
hglld: error: undefined symbol: from_a()
>>> referenced by b.o:(k(int*))
>>> did you mean: from_b()
hglink: error: HGVM_ERROR_COMPILATION
hgcc error: Exited with error code 1
```

换用第一代硬件可运行的 `ppu_10` 做最终链接，则由主机链接器报出 `undefined reference to __hggcRegisterLinkedBinary_<hash>`。该 hash 与架构绑定，架构不一致时链接必然失败。

架构一致时这层保护不存在。在同一架构下用 `-D` 给模板函数体制造差异，链接仍然成功，且两个版本各自保留、各自计算得到不同的结果，既没有报错也没有合并为一份。这违反了单一定义规则（ODR），编译器没有义务保证任何行为。

##### 约束 3：类型签名不应依赖 `__HGGC_ARCH__` { #arch-constraint-3 }

下面这段代码把类型本身绑定到了架构宏上：

```cpp
#if !defined(__HGGC_ARCH__)
typedef int type;
#else
typedef double type;
#endif
__device__ type var;
__global__ void kernel(type x) {}
```

这段代码在非 RDC / RDC、仅编译 / 编译并链接四种组合下都能通过，没有任何警告与报错。`-Wall` 无法帮助定位：hgcc 不接受该选项，可用的警告选项是 `-Wcross-execution-space-call` / `-Wmissing-launch-bounds` / `-Wdefault-stream-launch` / `-Wall-warnings` 这类专用名称。

后果是静默出错。由 kernel 向 `var` 写入 `(type)3.5`（设备端为 `double`），主机侧用 `hggcMemcpyFromSymbol` 读回：

```text
$ ./c3app
[host] sizeof(type)=4  (host pass thinks the __device__ var is this big)
[host] launch err=no error
[host] memcpyFromSymbol err=no error
[host] read back as sizeof=4 -> value printed as double: 0.000000 / as int: 0
```

主机侧目标文件中 `var` 的 ELF 符号大小是 4 字节：

```text
$ readelf -sW c3app | grep var
    14: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    5 _ZL3var
```

设备侧按 8 字节 `double` 写、主机侧按 4 字节 `int` 注册与读取，读回全 0，全程 `no error`。

`__HGGC_ARCH__` 可以改变函数的计算方式，但不能改变跨越主机/设备边界实体的类型与布局——`sizeof`、函数签名、变量存在性同样不得依赖该宏。

##### 约束 4：模板实例化不应依赖 `__HGGC_ARCH__` { #arch-constraint-4 }

模板实参与显式实例化列表同样不得依赖 `__HGGC_ARCH__`。其机理与约束 3 相同：由宏决定的模板实参让两端实例化出不同的类型，以这些类型声明的跨边界实体随之大小与布局不一致，同样没有任何诊断：

```cpp
// 反例：模板实参依赖架构宏
template <typename T> struct Accumulator { T sum; };

#if __HGGC_ARCH__ >= 150
  using Acc = Accumulator<double>;   // 设备端编译实例化的版本，8 字节
#else
  using Acc = Accumulator<int>;      // 主机端编译看到的版本，4 字节
#endif
__device__ Acc total;
```

主机端与设备端各自持有类型与大小都不一致的 `total`，后果与约束 3 完全一致：静默出错。

### 3.3. 特性测试宏 { #feature-macros }

hgcc 提供两个特性测试宏，在启用对应选项时定义：

- `__HGGCCC_EXTENDED_LAMBDA__`：支持扩展 lambda，由 `--expt-extended-lambda` 或 `--extended-lambda` 启用；
- `__HGGCCC_RELAXED_CONSTEXPR__`：支持宽松 constexpr 函数，由 `--expt-relaxed-constexpr` 启用。

这两个宏的作用是在不同工具链上保持可编译：`--expt-*` 属于实验性选项，不能假定下游用户必然启用。编写库代码时可用这两个宏提供双路径——启用时使用扩展写法，未启用时使用通用写法，两者结果一致。

#### 3.3.1. `__HGGCCC_EXTENDED_LAMBDA__`：extended lambda 与 functor 双路径 { #extended-lambda }

以 elementwise 的 ReLU 为例，两条路径共用同一个 kernel 模板，区别只在启动时传入的操作对象：

```cpp
// lambda.hg
// 通用 elementwise kernel：既接受 functor，也接受 extended __device__ lambda
template <typename Op>
__global__ void applyElementwise(Op op, float *buf, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) buf[i] = op(buf[i]);
}

#ifdef __HGGCCC_EXTENDED_LAMBDA__
static const char *kBuildPath = "extended-lambda";
static void launch_relu(float *d_buf, int n, int blocks) {
    applyElementwise<<<blocks, 256>>>([] __device__(float x) { return fmaxf(x, 0.f); },
                                      d_buf, n);
}
#else
static const char *kBuildPath = "functor-fallback";
struct ReluFunctor {
    __device__ float operator()(float x) const { return fmaxf(x, 0.f); }
};
static void launch_relu(float *d_buf, int n, int blocks) {
    applyElementwise<<<blocks, 256>>>(ReluFunctor(), d_buf, n);
}
#endif
```

分别按两种方式编译运行：

```text
$ hgcc lambda.hg -o lambda_functor
$ ./lambda_functor
build path : functor-fallback
relu sum   : 2655.0   negatives left: 0

$ hgcc --expt-extended-lambda lambda.hg -o lambda_ext
$ ./lambda_ext
build path : extended-lambda
relu sum   : 2655.0   negatives left: 0
```

两条路径数值结果完全一致。未加该选项而直接使用扩展 lambda 时，诊断信息会明确指出需要添加的选项：

```text
lambda_force.hg:3:5: error: lambda cannot be used in the template argument type of a __global__ function template instantiation, unless the lambda is an 'extended lambda' and the flag --expt-extended-lambda is specified
```

#### 3.3.2. `__HGGCCC_RELAXED_CONSTEXPR__`：实际行为宽于选项语义 { #relaxed-constexpr }

标准的双路径写法如下：

```cpp
// relaxed.hg
// 一个普通的（主机）constexpr 函数——没有 __device__ 标注
constexpr int poly(int x) { return 3 * x * x + 2 * x + 1; }

#ifdef __HGGCCC_RELAXED_CONSTEXPR__
__device__ int device_poly(int x) { return poly(x); }   // 跨执行空间调用
#else
__device__ constexpr int poly_dev(int x) { return 3 * x * x + 2 * x + 1; }
__device__ int device_poly(int x) { return poly_dev(x); }
#endif
```

无论是否指定 `--expt-relaxed-constexpr`，两条路径在主机端与设备端的结果都是 86，说明这套条件编译按预期生效。

但在 hgcc 2.2.0 中，跨执行空间调用 `constexpr` 函数并未被严格检查，也不会在编译期报错：即使未启用 `--expt-relaxed-constexpr`，去掉 `#ifdef` 保护、直接从 `__device__` 函数调用主机 `constexpr` 函数，编译仍能通过、运行正确，`-Wcross-execution-space-call` 不产生任何警告。

跨执行空间检查本身是生效的，只是不覆盖 `constexpr` 函数：把 `constexpr` 去掉换成普通主机函数，立即报错：

```text
hostcall.hg:2:33: error: reference to __host__ function 'plain_host' in __device__ function
```

因此，「不加 `--expt-relaxed-constexpr` 就不能从设备代码调用主机 `constexpr` 函数」这一前提在 hgcc 2.2.0 上并不成立。

能否使用跨执行空间调用，应以 `__HGGCCC_RELAXED_CONSTEXPR__` 的定义为准，而非以当前版本未报错为准；`#ifdef` 双路径写法应当保留。

### 3.4. 版本宏：编译期检查 hgcc 版本 { #version-macros }

`__HGGCCC_VER_MAJOR__` / `__HGGCCC_VER_MINOR__` / `__HGGCCC_VER_BUILD__` 分别是 hgcc 的主、次、构建版本号（整数）。三种典型用法——`#error` 前置检查、`#if` 版本分支、`static_assert` 编译期断言：

```cpp
// ver.hg
#if !defined(__HGGCCC_VER_MAJOR__)
#error "__HGGCCC_VER_MAJOR__ is not defined -- not compiling with hgcc?"
#endif

#if __HGGCCC_VER_MAJOR__ > 2 || (__HGGCCC_VER_MAJOR__ == 2 && __HGGCCC_VER_MINOR__ >= 2)
#pragma message("GATE: hgcc >= 2.2 -- new code path enabled")
#define HAVE_NEW_FEATURE 1
#else
#pragma message("GATE: hgcc < 2.2 -- legacy code path")
#define HAVE_NEW_FEATURE 0
#endif

static_assert(__HGGCCC_VER_MAJOR__ * 100 + __HGGCCC_VER_MINOR__ >= 202,
              "this project requires hgcc >= 2.2");
```

```text
$ hgcc ver.hg -o ver
ver.hg:8:9: warning: GATE: hgcc >= 2.2 -- new code path enabled [-W#pragma-messages]
$ ./ver
[host]   hgcc version macros: 2.2.0   HAVE_NEW_FEATURE=1
[device] hgcc version macros: 2.2.0   HAVE_NEW_FEATURE=1 (err=no error)
```

两端取值一致，都是 2.2.0，与 `hgcc --version` 的 `Release version 2.2.0` 相同。

`static_assert` 在编译期即触发——将下限设为 9：

```text
verfail.hg:1:1: error: static_assert failed due to requirement '2 >= 9' "requires hgcc 9.x (intentional failure)"
static_assert(__HGGCCC_VER_MAJOR__ >= 9, "requires hgcc 9.x (intentional failure)");
1 error generated when compiling for host.
```

报错发生在 `compiling for host`，且诊断信息中宏已被替换为字面量 `2`——版本宏在主机编译期间即可用，适合用于工具链下限检查。

上述 `static_assert` 中 `主版本号 * 100 + 次版本号` 的编码方式在次版本号小于 100 时是正确的；一旦次版本号达到或超过 100，就会与下一个主版本号重叠（`2.100` 与 `3.0` 同样编码为 300），比较结果随之出错。更稳妥的做法是把版本比较收敛为一个宏，也避免在多处书写长表达式：

```cpp
#define HGCC_VERSION_AT_LEAST(maj, min) \
    (__HGGCCC_VER_MAJOR__ > (maj) ||    \
     (__HGGCCC_VER_MAJOR__ == (maj) && __HGGCCC_VER_MINOR__ >= (min)))

#if !HGCC_VERSION_AT_LEAST(2, 2)
#error "this project requires hgcc >= 2.2"
#endif
```

### 3.5. `__HGGCCC_RDC__` 与 `__HGGCCC_DEBUG__`：构建模式的条件编译 { #rdc-debug-macros }

前文的宏描述的都是编译器与目标架构的信息，最后两个宏描述的则是构建模式：`__HGGCCC_RDC__` 对应分离编译，`__HGGCCC_DEBUG__` 对应 Debug 构建。两者都适合用作条件编译的开关。

#### 3.5.1. `__HGGCCC_RDC__`：分离编译的编译期检查 { #rdc-macro }

`__HGGCCC_RDC__` 在以可重定位设备代码模式编译时定义。默认不定义，指定 `-rdc=true`（编译阶段可简写为 `-dc`）后主机端和设备端都定义为 `1`；`-dlto` 单独使用时同样会定义它。

非 RDC 模式下引用其他编译单元的 `extern __device__` 变量只会产生一条警告：

```text
warning: extern declaration of the entity 'g_counter' is treated as a static definition
```

此时每个编译单元各持一份私有副本，程序能正常运行，但结果是错误的。在文件头部加上 `#error` 检查，即可将这种情况转为编译错误：

```cpp
// device_globals.hg —— 这个文件必须在 RDC 下编译
#ifndef __HGGCCC_RDC__
#error "device_globals.hg requires separate compilation; add -rdc=true (or -dc) to hgcc"
#endif

extern __device__ int g_counter;
```

RDC 的完整用法、性能代价、`-dlto`、分步链接与设备端静态库，见相关阅读 [hgcc 分离编译（RDC 模式）](/blogDetail?bizType=BLOG&id=42)。

#### 3.5.2. `__HGGCCC_DEBUG__`：区分 Debug 与 Release 构建 { #debug-macro }

`-G` 除了为设备代码生成调试信息，还会定义 `__HGGCCC_DEBUG__` 宏：

```cpp
// dbg.hg
__global__ void k(int *o) {
#ifdef __HGGCCC_DEBUG__
    o[0] = 1000 + __HGGCCC_DEBUG__;
#else
    o[0] = -1;
#endif
}
```

```text
$ hgcc dbg.hg -o dbg_noG && ./dbg_noG
host side: __HGGCCC_DEBUG__ NOT defined
device side returned: -1 (err=no error)

$ hgcc -G dbg.hg -o dbg_G && ./dbg_G
host side: __HGGCCC_DEBUG__ defined, value=1
device side returned: 1001 (err=no error)
```

第 2 节导出的全量宏清单可以佐证这一点：指定 `-G` 时导出结果比不指定时恰好多出一行，即 `#define __HGGCCC_DEBUG__ 1`，`-G` 对预处理器的影响仅此一条。因此可以用它判断设备代码是否按 Debug 模式编译。`-G` 只在设备代码中插入调试信息，主机代码的调试信息由 `-g` 控制，两者相互独立。

有了这个宏，就可以只在 Debug 构建中保留开销较高的设备端检查：

```cpp
#ifdef __HGGCCC_DEBUG__
  #define DEV_CHECK(cond, msg)                                         \
      do { if (!(cond)) {                                              \
          printf("DEV_CHECK failed: %s @ block %d thread %d\n",        \
                 msg, blockIdx.x, threadIdx.x);                        \
      } } while (0)
#else
  #define DEV_CHECK(cond, msg) do { } while (0)
#endif

__global__ void scale(float *p, int n, float k) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    DEV_CHECK(n > 0, "n must be positive");
    if (i < n) p[i] *= k;
}
```

## 4. 速查表 { #cheatsheet }

**查看宏**

```bash
hgcc -E -Xcompiler -dM x.hg              # 全部宏（注意：导出的是设备端结果）
hgcc --dryrun x.hg -o x 2>&1 | tr ' ' '\n' | grep '^-D'   # hgcc 注入的 -D，可区分编译端
# 在源码中写 #pragma message(...)，根据输出末尾的 "compiling for ppu001 / for host" 定位所属编译端
```

**主机端与设备端**

| | 主机端 | 设备端 |
| :--- | :---: | :---: |
| `__HGGC__` / `__HGGCCC__` / 三个版本宏 | 定义 | 定义 |
| `__HGGCCC_RDC__` / `__HGGCCC_DEBUG__` / `__HGGCCC_EXTENDED_LAMBDA__` / `__HGGCCC_RELAXED_CONSTEXPR__`（开启对应选项后） | 定义 | 定义 |
| **`__HGGC_ARCH__`** | **未定义** | 定义 |

**常用条件编译写法**

```cpp
#ifdef __HGGC_ARCH__                 // 当前是设备端编译
#if __HGGC_ARCH__ >= 150             // 目标架构为 ppu_15 或更高（主机端未定义视作 0，进入 #else）
#ifndef __HGGCCC_RDC__               // 未开启 -rdc=true
#ifdef __HGGCCC_DEBUG__              // 已开启 -G
#ifdef __HGGCCC_EXTENDED_LAMBDA__    // 已开启 --expt-extended-lambda
#ifdef __HGGCCC__                    // 由 hgcc 编译 .hg，而非直接调用 g++
```

**架构值**

| `-arch` | `__HGGC_ARCH__` | 编译端标识 |
| :--- | :--- | :--- |
| `ppu_10`（默认）/ `vm_10` | `100` | `ppu001` |
| `ppu_15` / `vm_15` | `150` | `ppu0015` |

## 5. 总结 { #summary }

最后把全文的要点归纳为四条。

1. 判断某一行代码在哪一端被编译，用 `#pragma message` 加输出末尾的 `compiling for ppu001` / `for host` 标识；不要依据 `hgcc -E -Xcompiler -dM` 的导出结果推断，它导出的只是设备端结果。
2. 需要 `__HGGC*` 宏或 Runtime API 头文件的源文件，扩展名用 `.hg` 或显式指定 `-x hg`；`.cpp` 交给 hgcc 只会原样转发给 `g++`，既得不到任何宏，也得不到 SDK 的 include 路径。
3. `__HGGC_ARCH__` 有两条红线：只能用于预处理指令——它在主机端编译中不存在，`#if __HGGC_ARCH__ >= X` 会无提示地进入 `#else`，由它推导的常量不要用于启动配置；同时不得影响任何跨越主机/设备边界实体的类型、`sizeof`、函数签名或存在性，违反后没有任何诊断，直接表现为静默出错。
4. 工具链下限检查用 `HGCC_VERSION_AT_LEAST(maj, min)` 这类宏，不要手写 `主版本号 * 100 + 次版本号`，后者在次版本号达到 100 时失效。

这四条中前两条关乎工程组织，后两条关乎正确性。`__HGGC_ARCH__` 的四条约束之所以需要单独成节，根本原因在于文档末尾的声明：编译器不保证为不支持的用途生成诊断信息。

## 6. 延伸阅读 { #further-reading }

- 相关阅读：[hgcc 分离编译（RDC 模式）](/blogDetail?bizType=BLOG&id=42) —— `__HGGCCC_RDC__` 宏背后的完整机制
- [T-Head SAIL HGCC](https://developer.t-head.cn/docs_center/doc_detail/index.html?k=5vopartyng) —— 编译器驱动手册，「预定义宏」章节给出宏清单，编译阶段选项表说明 `-E` / `-dc` / `-G` 的语义
- [T-Head SAIL HGGC 编程指南](https://developer.t-head.cn/docs_center/doc_detail/index.html?k=8flckpzsbw) —— 「`__HGGC_ARCH__`」小节是第 3.2.6 节四条约束的出处；「HGGC 特性测试宏」与「调试选项」小节分别对应第 3.3 节与 `__HGGCCC_DEBUG__`；「启动界限」与「核函数启动和占用率」小节是第 3.2.5 节 `__launch_bounds__` 陷阱的背景
- [T-Head SAIL HGGC 示例程序](https://developer.t-head.cn/docs_center/doc_detail/index.html?k=4k38irnmyz) —— `CMAKE_HG_ARCHITECTURES` 与多架构构建
