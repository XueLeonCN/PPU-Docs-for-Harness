---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=45"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=45"
source_title: "如何控制 asys 跟踪采集的报告大小"
source_id: "blog-45"
source_kind: "technical-blog"
source_date: "2026-09-29"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "c2f3775190a9c7828cc20b536f44f4a6f6baab5a189ca85a50e735d80cf07aad"
tags:
  - "Asight Systems"
  - "asys"
  - "trace"
  - "Profiler-API"
  - "HGTX"
  - "NVTX"
  - "report-size"
  - "rank-filter"
  - "attach"
---
# 如何控制 asys 跟踪采集的报告大小

> 从采集到加载，全方位拆解 asys 报告体积控制策略

## 1. 为什么需要控制报告大小 {#1-为什么需要控制报告大小}

使用 asys 采集跟踪数据时，默认行为是从应用程序启动的那一刻开始采集，直到程序结束。对于小型应用来说，这没有什么问题。但当你面对大语言模型（LLM）推理或训练场景时，情况就完全不同了——模型加载、权重初始化、CUDA 上下文创建等过程会产生大量跟踪事件，而真正需要分析的推理迭代阶段却被淹没在海量数据中。

一份未经控制的 asysrep 报告动辄几个 GB，带来的问题是多方面的：

- **加载缓慢**：GUI 打开报告需要数分钟甚至更久
- **查找困难**：在漫长的时间线中定位到感兴趣的迭代区间如同大海捞针
- **分析卡顿**：报告过大导致交互操作（缩放、搜索、过滤）响应迟缓
- **传输不便**：将报告从服务器拷贝到分析机器耗时漫长

好在 asys 提供了丰富的采集控制手段，可以从多个维度限制报告大小。下面逐一介绍每种方法的特点和用法。

---

## 2. 事件触发跟踪采集 {#2-事件触发跟踪采集}

**核心思路**：让应用程序自己决定什么时候开始和停止采集，只抓取真正关心的代码段。以大模型推理为例，模型加载和初始化阶段通常占总跟踪事件的 50%-80%，通过在代码中插入事件标记，asys 只记录标记之间的代码段数据，这部分初始化数据完全不写入报告。配合 `repeat-shutdown:N` 还能控制采集次数——10 次推理迭代只采 2 次，数据量再减 80%。这种从代码层面精确控制采集范围的方式，是所有方法中数据缩减比最高的。

asys 默认从应用启动就开始采集，整个生命周期内的所有 API 调用、kernel 执行、内存操作都会被记录。但对于大模型应用，启动阶段包含模型加载、权重初始化、CUDA 上下文创建等大量操作，这些阶段的跟踪数据并非性能分析的重点关注对象，却往往占据报告总量的大部分。通过在代码中插入事件标记，asys 只在标记之间采集数据，其余阶段的跟踪事件完全不写入报告。

asys 支持通过 `--capture-range`（`-c`）选项指定触发源，提供两种触发方式：

| `--capture-range` 选项取值 | 触发事件说明 |
| --- | --- |
| `hggcProfilerApi` | 使用 `hggcProfilerStart` / `cudaProfilerStart` 触发开始；`hggcProfilerStop` / `cudaProfilerStop` 触发结束 |
| `hgtx` | 使用 HGTX / NVTX range 触发开始和结束 |
| `none` | 默认值，不使用事件触发采样 |

### 2.1. 使用 Profiler API 触发 {#21-使用-profiler-api-触发}

在代码中插入 `cudaProfilerStart()` / `cudaProfilerStop()` 调用，asys 遇到这两个 API 时自动启停采集。

**PyTorch 示例**：

```python
import torch

model = MyModel().cuda()
optimizer = torch.optim.Adam(model.parameters())
data = load_dataset()

# 预热阶段（不采集）
for _ in range(3):
    model(data)

# 开始跟踪采集
torch.cuda.profiler.start()
for step in range(10):
    output = model(data)
    loss = criterion(output, target)
    loss.backward()
    optimizer.step()
torch.cuda.profiler.stop()
```

对应的采集命令：

```bash
asys profile -t hggc -c hggcProfilerApi python train.py
```

`torch.cuda.profiler.start()` 底层调用的就是 `cudaProfilerStart()`，asys 会在这行代码执行时开始跟踪抓取，在 `torch.cuda.profiler.stop()` 时停止。预热阶段的跟踪数据完全不会被记录。

**C/C++ 示例**（原生 CUDA 应用）：

```c
cudaProfilerStart(); // 采集开始
DoProcess();
cudaProfilerStop();  // 采集结束
DoOtherProcess();
```

```bash
asys profile -t hggc -c hggcProfilerApi hggc_test
```

### 2.2. 使用 HGTX/NVTX Range 触发 {#22-使用-hgtxnvtx-range-触发}

Profiler API 需要显式调用 start/stop，适合标注一段连续代码。但如果想在循环中每次迭代都触发采集，用 NVTX range 更方便——在代码中用 `nvtxRangePush` / `nvtxRangePop` 标记区域，asys 根据 range 名称自动启停采集。

**PyTorch 示例**：

```python
import torch

model = MyModel().cuda()
data = load_dataset()

# 预热
for _ in range(3):
    model(data)

# 用 NVTX range 标注推理迭代
for step in range(10):
    torch.cuda.nvtx.range_push("decode_step")
    output = model(data)
    torch.cuda.nvtx.range_pop()
```

对应的采集命令：

```bash
asys profile -t hggc,hgtx -c hgtx -p decode_step python inference.py
```

- `-c hgtx`：指定使用 HGTX / NVTX 作为事件触发源
- `-p decode_step`：指定匹配名称为 `decode_step` 的 NVTX range，range 开始时启动抓取，range 结束时停止抓取

也可以用 Python 上下文管理器写法，效果相同：

```python
with torch.cuda.nvtx.range("decode_step"):
    output = model(data)
```

如果应用本身不使用 PyTorch，可以 `pip install nvtx` 安装独立的 NVTX 包，提供 `nvtx.annotate` 上下文管理器和装饰器等写法，asys 同样识别，只要 range 名称匹配即可。

`-p`（`--hgtx-capture`）选项还支持多种匹配方式：

| `--hgtx-capture` 选项取值 | 匹配方式说明 |
| --- | --- |
| `range@domain` | 匹配指定 domain 下名称为 range 的时间范围 |
| `range` | 匹配默认 domain 下名称为 range 的时间范围 |
| `range@*` | 匹配任意 domain 下名称为 range 的时间范围 |

**C/C++ 示例**（原生 CUDA 应用）：

```c
for (int index = 0; index < 5; ++index) {
    nvtxRangePushA("DoProcess"); // range 开始
    DoProcess();
    nvtxRangePop();              // range 结束
}
```

```bash
asys profile -t hggc,hgtx -c hgtx -p DoProcess hggc_test
```

---

## 3. 延迟启动 + 定时采集 {#3-延迟启动-定时采集}

**核心思路**：跳过应用初始化阶段，只采集感兴趣的时间窗口，不需要修改应用代码。大模型应用从启动到进入推理状态通常需要数分钟，`--delay` 跳过这段初始化期，`--duration` 限定只采集分析者关心的推理时间窗口。例如一个总运行 5 分钟的应用，用 `--delay 180 --duration 30` 只采集 30 秒数据，报告体积相比全量采集可以缩小到 1/10 以下。

事件触发采集需要在代码中插入标记，但有些场景下不方便改代码——比如第三方框架启动的应用，或者二进制程序。asys 提供 `--delay`（`-y`）和 `--duration`（`-d`）两个选项来从时间维度控制采集范围，无需修改应用代码。

`--delay` 指定应用启动后延迟多少秒开始采集，`--duration` 指定采集持续多少秒后自动停止。两者配合，可以在应用运行过程中精确框选一个时间窗口。

```bash
asys profile -t hggc --delay 2 --duration 3 python test_linear.py
```

- `--delay 2`：应用启动后，延时 2 秒再开始采集
- `--duration 3`：采集 3 秒后自动停止应用并生成报告

采集过程的执行流程如下：

![延迟启动与定时采集示意图](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763958012/8ab34a31200b7e174628b2c94582c203/duration_3_1.png)

蓝色条代表应用执行的全程，上方绿色段则是实际被采集的 3 秒跟踪窗口。可以看到，应用启动初期的 2 秒被完全跳过。

**适用场景**：明确知道目标行为发生在应用启动后某个时间点，比如模型加载完成后推理才开始，跳过加载阶段可以显著减小报告体积。

---

## 4. 交互式采集手动控制启停 {#4-交互式采集手动控制启停}

**核心思路**：将"启动应用"和"开始采集"解耦，手动控制采集时机，按需采集。分析人员可以在应用进入稳定状态后（模型加载完成、GPU 利用率稳定）才执行 `asys start`，采集到需要的数据后立即 `asys stop` 生成报告，不感兴趣的阶段完全不采集。与 `--delay` 相比，不需要预估延迟时间，适合启动时间不确定的场景。一次应用运行中可以多次 start/stop 生成多份小报告，避免重复启动应用的开销。

前面两种方式都需要在命令行预先配置好采集参数，采集过程是自动的。但有些场景下，分析人员需要在应用运行过程中观察状态，择机开始和停止采集——比如等模型加载完成后看 GPU 利用率稳定了再开始采集，或者在一次运行中分别采集 prefill 阶段和 decode 阶段的数据。

asys 提供了一组交互式子命令来实现这种灵活控制：

| 子命令 | 功能 |
| --- | --- |
| `asys launch` | 启动应用，但不立即采集 |
| `asys start` | 开始跟踪采集 |
| `asys stop` | 停止采集并生成报告 |
| `asys shutdown` | 停止应用 |
| `asys sessions list` | 查看当前采集会话列表 |

### 4.1. 典型用法：先启动应用，抓取两次跟踪 {#41-典型用法先启动应用抓取两次跟踪}

```bash
asys launch -t hggc hggc_test      # 启动应用
asys start -o test_report1        # 开始第一次采集
asys stop                          # 停止采集，生成 report1
asys start -o test_report2        # 开始第二次采集
asys stop                          # 停止采集，生成 report2
asys shutdown                      # 停止应用
```

整个过程中，应用一直在运行，而采集只在需要时进行：

![交互式采集流程](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763957616/31f1e739c491b532a3d39ae14504bc1b/asys_launch_1.png)

蓝色条代表应用执行的全过程，两次绿色段是实际采集的时间窗口。可以看到，应用启动与采集开始之间有间隔，两次采集之间也有间隔——这些间隙期间的数据不会被记录到报告中。

`asys stop` 生成报告后应用继续运行，可以再次输入 `asys start` 开始下一轮采集，灵活地按需多次抓取不同阶段的数据。

### 4.2. 查看采集会话 {#42-查看采集会话}

当多个采集过程同时存在时，可以通过 `asys sessions list` 查看所有活跃的会话：

```bash
asys sessions list
```

```
ID        TIME   STATE          LAUNCH  NAME
126485    00:44  Collecting     1       profile-26485-cuda_test
426548    00:34  WaitForStart   1       [default]
626656    00:18  Collecting     1       test
```

每个会话有独立的 ID、状态和名称，方便在复杂场景下管理多个采集过程。

**适用场景**：应用启动时间长或启动条件复杂，需要在应用运行到特定状态后手动触发采集。也适用于在一次应用运行中多次采集不同阶段的跟踪数据，避免重复启动应用的开销。

---

## 5. 报告轮转采集 {#5-报告轮转采集}

**核心思路**：长时间持续采集，但将数据分散到多个小报告中，只保留最近几份。每份报告只覆盖 `--duration` 指定的短时间段（如 30 秒），单份报告体积可控；通过 `--trace-rotation` 限制保留份数，旧报告自动删除。对于运行数小时的推理服务，原本几十 GB 的单一大文件被拆分成若干个几十 MB 的小文件，分析时只需加载最新的一份。

对于持续运行的推理服务，无法用 `--duration` 只采一次就停——服务不会停。如果直接全量采集几小时的数据，报告会膨胀到几十甚至上百 GB。asys 的报告轮转模式通过 `--trace-rotation` 配合 `--duration` 实现持续采集 + 自动轮转：

- 每隔 `--duration` 指定的时间生成一份报告
- 仅保留最近的 N 份报告，旧报告自动删除
- 通过 `--output` 指定报告名模板，用 `%t` 避免重名

```bash
asys profile -t hggc --duration 30 --trace-rotation 3 --output test_report_%t hggc_test
```

- `--duration 30`：每 30 秒生成一份报告
- `--trace-rotation 3`：保留最近 3 份报告
- `--output test_report_%t`：报告名含时间戳，如 `test_report_08_09_33.asysrep`

轮转采集的工作方式如下图：

![报告轮转采集](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763957793/6fc3ae9a5280473874e76c869485daee/duration_30_1.png)

橙色虚线框的报告已过时被删除，绿色实线框的是当前保留的报告。整个过程在应用持续运行期间自动进行。

**适用场景**：持续运行的推理服务，需要周期性采集跟踪数据来监控性能变化。每份报告保持较小体积，便于单独分析。

---

## 6. 限制报告最大尺寸 {#6-限制报告最大尺寸}

**核心思路**：给报告文件设一个硬上限，达到上限自动停止采集，作为报告体积的兜底保障。asys 在写入跟踪数据时实时监控报告文件大小，一旦达到设定的上限就立即停止采集，无论应用跟踪事件有多密集、采集时间有多长，最终报告大小都被控制在设定值附近。这在应用行为未知、无法预估 `--duration` 该设多少的场景中特别有用——应用可能在某些阶段突然产生大量事件（如集合通信阶段），设一个 200 MiB 上限可以确保报告始终可控。

前面的方法都是通过控制采集范围来间接控制报告大小，但实际环境中数据密度难以预估——同样 30 秒的采集，包含大量 kernel launch 的训练阶段可能产生几百 MB 数据，而推理阶段可能只有几十 MB。`--max-report-size` 选项直接从文件大小维度设硬上限，单位为 MiB，无论采集时间多长，报告文件不会超过设定的上限太多：

```bash
asys profile -t hggc --max-report-size 200 python test_linear.py
```

当报告文件超过 200 MiB 后，asys 自动停止跟踪采集。

**适用场景**：对报告大小有明确约束的场景，比如需要通过邮件或聊天工具发送报告，或分析机器磁盘空间有限。也适合作为其他采集策略的兜底保障——即使时间估算失误，报告也不会失控。

---

## 7. 限制采集的进程范围 {#7-限制采集的进程范围}

**核心思路**：多卡分布式训练或推理中，只采集部分进程的跟踪数据，减少进程数等于成比例减少数据量。报告体积与被采集的进程数近似成正比，8 卡分布式推理默认采集 8 个进程的数据；只采集其中 1 个进程，报告体积直接减少到 1/8。

在大规模分布式训练或推理场景中，一台机器上可能有 8 个甚至更多进程同时运行。默认情况下 asys 会采集所有进程的跟踪数据到同一份报告中，进程数越多，报告体积越大——8 卡推理时 8 个进程的 HGGC API 调用、kernel 执行、内存操作全部记录到同一份文件中，报告体积是单进程的近 8 倍。asys 提供了两种进程过滤方式，只采集需要的进程。

### 7.1. 按 PPU 设备过滤 {#71-按-ppu-设备过滤}

通过 `--process-ppu-include` 指定只采集使用特定 PPU 设备的进程：

```bash
asys profile --process-ppu-include 0,2 python test_linear.py
```

只采集使用 PPU 设备 0 和设备 2 的进程的跟踪数据。

### 7.2. 按 Rank ID 过滤 {#72-按-rank-id-过滤}

分布式计算中，每个参与计算的进程会被分配一个全局唯一的整数编号，称为 **Rank**。Rank 是 MPI（Message Passing Interface）标准中的概念，用于在进程间通信中标识每个进程的身份。不同的分布式启动框架通过不同的环境变量传递 Rank 信息：

| 启动框架 | Rank 环境变量 | 说明 |
| --- | --- | --- |
| mpirun（OpenMPI） | `OMPI_COMM_WORLD_RANK` | MPI 通信域中的全局进程编号 |
| srun（Slurm） | `SLURM_PROCID` | Slurm 调度器分配的任务编号 |
| torchrun（PyTorch） | `RANK` | PyTorch DDP 中的全局 Rank |
| PMI（Process Management Interface） | `PMI_RANK` | PMI 框架下的进程编号 |

asys 默认自动检测上述环境变量来获取进程的 Rank ID。通过 `--process-rank-include` 可以指定只采集特定 Rank 的进程：

```bash
# 只采集 Rank 0 和 Rank 2 的进程
asys profile --process-rank-include 0,2 python test_linear.py
```

如果应用使用非标准的 Rank 环境变量，可以通过 `--process-rank-env` 指定：

```bash
# 指定通过 PROCESS_RANK_ID 环境变量获取 Rank
asys profile --process-rank-include 0,2 --process-rank-env=PROCESS_RANK_ID python test_linear.py
```

**适用场景**：多卡分布式场景下，只需分析代表性卡的数据，不需要采集所有卡。比如 8 卡训练只需采集 Rank 0 的数据，报告体积可以缩小到 1/8。

---

## 8. Attach 方式采集 {#8-attach-方式采集}

**核心思路**：不通过 asys 启动应用，而是直接附着到已运行的进程上采集，只抓取附着后的短时段数据。Attach 天然跳过应用启动和初始化阶段——这些阶段早已完成，事件不会出现在报告中。配合 `--duration` 限定采集时长（如 10 秒），每次只生成小报告；还可以反复 attach 补充采集，不需要重启应用。相比从应用启动开始全量采集，报告通常可以缩小一到两个数量级。

前面所有方式都要求通过 asys 来启动应用。但有些场景下，应用已经在运行了（比如线上推理服务），不能重启。asys 的 `attach` 子命令支持直接附着到已运行的进程上采集跟踪数据：

```bash
asys attach -t hggc --duration 10 -f true -o attach_report 94644,94923
```

- 同时附着到 PID 为 94644 和 94923 的两个进程
- 采集 10 秒后自动停止并生成报告
- 应用程序不受影响，继续运行

Attach 方式的工作流程：

![Attach 模式采集](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763957420/3660aed729781c89d9f523300127973d/asys_attach_1.png)

蓝色条代表应用持续运行的全过程，绿色段是两次 attach 采集的时间窗口。可以看到，attach 可以反复对同一个应用进行多次采集，每次只采集需要的那一小段数据。

**适用场景**：线上服务不能重启，需要在不中断业务的情况下采集跟踪数据。通过 `--duration` 控制采集时间，每次采集只生成很小的报告文件。

---

## 9. 高级打开功能——只查看报告的一部分 {#9-高级打开功能只查看报告的一部分}

**核心思路**：报告已经很大了，加载时只看感兴趣的部分，减少 GUI 内存占用和加载时间。高级打开功能不改变磁盘上的报告文件大小，但通过选择性加载减少 GUI 实际解析和渲染的数据量。一份 5 GB 的报告，如果只加载前 30% 且只勾选 HGGC + HGTX 事件类型，GUI 实际处理的数据量可能只有全量加载的 10%-15%，加载时间从数分钟缩短到几十秒。适合先用部分加载确认范围，再决定是否全量加载或重新采集。

前面七种方法都是在采集阶段控制报告大小。但如果已经采集了一份很大的报告，不想重新采集怎么办？Asight Systems GUI 提供了高级打开功能（`File` -> `Open Advanced`），支持按百分比只加载报告的一部分，从加载阶段减小实际处理的数据量：

![高级打开功能](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763958140/1bffa1d1e10a7a32dc253019f60f055d/open_advanced_1.png)

高级打开功能提供三方面控制：

1. **多报告同时显示**：可以在同一个 Timeline View 中打开多个报告对比
2. **按百分比加载**：通过滑块选择加载报告的百分比范围，比如只加载前 30% 或中间 30%-60% 的部分
3. **按事件类型加载**：勾选只加载特定类型的跟踪数据，比如只加载 HGGC 和 HGTX，不加载 CPU 调度和内存使用数据

加载事件类型和跟踪数据的对应关系：

| 加载事件类型 | 跟踪数据类型 |
| --- | --- |
| Metrics sampling | PPU Metrics / CPU Metrics |
| CPU schedule | CPU 线程调度 |
| Memory usage | 设备内存 / Pinned / Host / Heap 内存使用 |
| OS runtime | OSRT 跟踪和调用栈 |
| HGTX | HGTX 跟踪 |
| HGGC | HGGC API 跟踪和 PPU Activity |
| Backtrace | 各类调用栈采样 |

**适用场景**：已经有一份大报告，不想重新采集。通过只加载部分数据，可以快速定位到感兴趣的区间进行初步分析，确认范围后再做完整加载或重新采集。

---

## 10. 方法总结与选型指南 {#10-方法总结与选型指南}

| 方法 | 核心选项 | 控制维度 | 适用场景 |
| --- | --- | --- | --- |
| 事件触发采集 | `--capture-range` `-p` | 代码范围 | 精确采集特定代码段或迭代步 |
| 延迟启动 + 定时采集 | `--delay` `--duration` | 时间范围 | 跳过初始化，采集特定时间窗口 |
| 交互式采集 | `launch`/`start`/`stop` | 手动控制 | 需要人工判断采集时机 |
| 报告轮转 | `--trace-rotation` | 持续采集 + 保留数量 | 长时间运行的推理服务 |
| 限制报告尺寸 | `--max-report-size` | 文件大小 | 对报告大小有硬性约束 |
| 限制采集进程 | `--process-ppu-include` `--process-rank-include` | 进程范围 | 多卡分布式场景，只采部分卡 |
| Attach 模式 | `asys attach` | 采集方式 | 不能重启的线上服务 |
| 高级打开 | GUI `Open Advanced` | 加载范围 | 已有大报告的部分加载 |

实际使用中，这些方法可以组合使用。比如：对分布式推理服务，可以用 `--process-rank-include 0` 只采集 Rank 0 的数据，同时用 `--delay 60 --duration 30` 跳过加载阶段只采集 30 秒推理数据，再加上 `--max-report-size 200` 兜底防止报告过大。组合使用可以将几个 GB 的报告缩小到几十 MB 级别，让后续的加载和分析变得轻松很多。
