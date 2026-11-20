---
source_url: "https://developer.t-head.cn/blogDetail?bizType=BLOG&id=38"
source_api_url: "https://developer.t-head.cn/api/developer/content/detail?bizType=BLOG&id=38"
source_title: "Asight Systems MCP 使用实践：通过 AI agent 分析 asys 报告"
source_id: "blog-38"
source_kind: "technical-blog"
source_date: "2026-09-29"
source_version: null
source_format: "MARKDOWN"
source_authority: "official"
publisher: "平头哥开发者网站"
captured_at: "2026-11-20T00:00:00+08:00"
source_sha256: "c266998c5988eb1024a101b80def1b6c25a1f893538b11232cd0d7656f9c90f8"
tags:
  - "Asight Systems"
  - "MCP"
  - "AI-agent"
  - "Qoder"
  - "M890"
  - "810E"
  - "training-performance"
  - "Asight Compute"
  - "trace-analysis"
---
# Asight Systems MCP 使用实践：通过 AI agent 分析 asys 报告

> 本文以一份多机训练性能对比报告为例，以`Qoder`为例介绍如何用 AI agent 配合 Asight Systems MCP，通过自然语言提问完成从报告加载到跨报告对比的完整分析流程。

---

## 开篇 {#开篇}

分析 asys 性能报告是性能优化工作中的常见任务。Asight Systems 提供了多种分析和跟踪导出功能——GUI 有 Stats System View 支持饼图和表格展示，命令行有 `asys stats` 支持导出 CSV/xlsx，还内置了报告对比功能。但实际使用中，面对一份新报告，仍然存在一些门槛：需要知道该跑哪条统计规则、需要手动框定统计区间、需要自己解读表格里的数字意味着什么。一次完整的性能分析往往需要半天。

Asight Systems MCP 把 asys 底层的报告查询、统计规则、Insight 标注能力开放给 AI Agent，用户通过自然语言提问即可驱动 Agent 自动完成分析。本文以一份真实的多机训练性能对比报告为例，展示完整的配置和使用流程。

---

## 1. 配置 Asight Systems MCP {#1-配置-asight-systems-mcp}

### 1.1 前置条件 {#11-前置条件}

- 本机已安装 Asight Systems GUI（Windows / macOS）
- 本机有一份或多份 `.asysrep` 报告文件
- 本机已安装Qoder

### 1.2 启动 GUI 内置 MCP Server {#12-启动-gui-内置-mcp-server}

打开 Asight Systems GUI 后，右下角状态图标显示 MCP Server 运行状态（绿色 = 运行中，灰色 = 端口占用）。通过菜单 `Tools → asys MCP Server` 打开配置窗口，勾选"允许局域网访问"。

![MCP 配置窗口](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763956471/248f56cbbb8bf95987c6119fae557d04/mcp_server_dialog.png)

如果是远端 Linux 无 GUI 环境，也可以用 `asys_mcp_server` 命令行工具起 HTTP 服务，支持 stdio 和 http 两种模式，详见官方文档。

### 1.3 在Qoder里添加连接器 {#13-在Qoder里添加连接器}

打开Qoder → 在左侧导航栏选择扩展 → 打开连接器页签 → 选择添加连接器 → 选择JSON页签，粘贴配置：

```json
{
  "mcpServers": {
    "asight-systems-gui": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:28080/mcp"
    }
  }
}
```

保存后确认连接状态显示为"已连接"即可。如果本机有多块网卡，IP 地址需要根据实际网络情况调整。

![Qoder连接器配置](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1791601970066/f7a92b0b8782c764812d263b065783e6/qoderwork_mcp_settings.png)

![连接器配置详情](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1791601970693/7f7e2e5c2180949f9a1ce0fd7ca93757/qoderwork_mcp_settings_config.png)

### 1.4 验证 {#14-验证}

新开一个Qoder任务，输入"请列出 asys MCP 支持的分析能力"。Agent 会通过 MCP 返回支持的统计规则和分析功能列表，看到列表即接入成功。

---

## 2. MCP 的价值 {#2-mcp-的价值}

asys 的统计系统功能完备，从算子汇总到报告对比都有覆盖。但实际分析一份报告时，流程仍然比较复杂：需要判断跑哪条规则、框定统计区间、解读表格数字、跨报告对比时还要手动对齐时间范围。Asight Systems MCP 把这些能力开放给 AI Agent，用户通过自然语言提问即可驱动 Agent 自动完成上述流程。

**自动选择规则**：asys 统计系统有 50+ 条规则，面对新报告时需要判断"先跑哪条、再跑哪条"。通过 MCP，直接用自然语言描述分析目标，Agent 会自动选择合适的规则执行。

**自动解读结果**：跑完规则拿到统计表格后，还需要判断"这些数字说明什么问题"。比如两份报告对比时，某个算子类型的平均耗时差了 2.5 倍——在统计表里只是一行数字。Agent 拿到结果后会自动解读，直接给出结论。Agent 还具备事件级下钻能力（PPU stream 跟踪、调用栈火焰图、依赖关系、HGGC Graph 拓扑等），能从统计异常一路追到具体的跟踪事件。

![调用栈火焰图](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763956334/4b968cf5ebad10faf2c453ce2ee90445/function_view_flamegraph.png)

**AI Insight 标注**：分析结论以 AI Insight 形式贴回 Timeline，High/Medium/Low 三色分级，支持绑定 Timeline 中的跟踪事件、时间戳或全局关联，可点击跳转。下次打开报告，标注还在。

![AI Insight 面板](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763956186/d268abbf6199450aae3ddc4e6f14a740/ai_insight_pane.png)

---

## 3. 分析实践：真武 M890 与真武 810E 训练性能对比 {#3-分析实践真武-m890-与真武-810e-训练性能对比}

> **背景**：Qwen2.5-VL VLA 预训练，真武 M890 64 卡相比真武 810E 128 卡慢 8.8%。两个平台各采集了 8 节点的 asysrep 报告。以下演示用 Agent 端到端分析性能差距的原因。

### 3.1 端到端分析 {#31-端到端分析}

**提问**：

> 路径 `/path/to/m890/` 下有真武 M890 的 8 份 asysrep 报告，路径 `/path/to/810e/` 下有真武 810E 的 8 份 asysrep 报告。真武 M890 64 卡比真武 810E 128 卡慢 8.8%，帮我分析性能差距的原因。

Agent 收到提问后，在 Asight Systems GUI 的打开所有 asysrep 报告，自动完成以下多轮分析。

**第 1 步 · 全节点算子类型汇总**

Agent 加载两个平台共 16 份报告，对每个平台的 8 份报告分别跑算子类型汇总聚合规则，得到平台整体的耗时分布：

```
真武 M890（64 卡聚合）
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 算子类型     │ 总耗时占比   │ 调用次数     │ 平均耗时     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Native       │ 38.0%        │ 1,925,043    │ 30.0 μs      │
│ PCCL         │ 28.5%        │ 2,304        │ 18.8 ms      │
│ GEMM         │ 28.4%        │ 243,456      │ 177.0 μs     │
│ FlashAttention│ 2.9%        │ 26,112       │ 171.0 μs     │
│ Memory       │ 1.5%         │ 438,872      │ 5.2 μs       │
└──────────────┴──────────────┴──────────────┴──────────────┘

真武 810E（128 卡聚合）
┌──────────────┬────────────────────────────┬──────────────┐
│ 算子类型     │ 总耗时占比   │ 调用次数     │ 平均耗时     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ PCCL         │ 49.8%        │ 4,608        │ 37.3 ms      │
│ GEMM         │ 34.1%        │ 486,912      │ 241.9 μs     │
│ Native       │ 12.0%        │ 3,456,306    │ 12.0 μs      │
│ FlashAttention│ 2.7%        │ 52,224       │ 179.7 μs     │
│ Memory       │ 1.0%         │ 764,522      │ 4.3 μs       │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

两个平台的分布特征差异明显：真武 M890 的第一大耗时来源是 Native（38.0%），而真武 810E 的 Native 仅占 12.0%，第一大来源是 PCCL 通信（49.8%）。真武 M890 的 PCCL 占比（28.5%）明显低于真武 810E（49.8%），这是因为真武 810E 有 128 张卡参与集合通信，AllReduce 调用次数是真武 M890 的 2 倍（3,456 vs 1,728），且每次 AllReduce 的参与方更多、单次通信耗时也更长。

**第 2 步 · 验证负载均衡**

全节点聚合数据代表整体特征，但需要确认单个节点的分布是否和聚合一致，才能确认负载均衡、排除异常节点。Agent 选取各平台的 node0 单独跑算子类型汇总，和第 2 步的聚合结果对比：

```
真武 M890（node0 vs 8 节点聚合）
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 算子类型     │ node0 占比   │ 聚合占比     │ 差值         │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Native       │ 37.8%        │ 38.0%        │ 0.2%         │
│ PCCL         │ 28.7%        │ 28.5%        │ 0.2%         │
│ GEMM         │ 28.3%        │ 28.4%        │ 0.1%         │
│ FlashAttention│ 2.9%        │ 2.9%         │ 0.0%         │
│ Memory       │ 1.5%         │ 1.5%         │ 0.0%         │
└────────────────────────────┴──────────────┴──────────────┘

真武 810E（node0 vs 8 节点聚合）
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 算子类型     │ node0 占比   │ 聚合占比     │ 差值         │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ PCCL         │ 49.4%        │ 49.8%        │ 0.4%         │
│ GEMM         │ 34.4%        │ 34.1%        │ 0.3%         │
│ Native       │ 12.1%        │ 12.0%        │ 0.1%         │
│ FlashAttention│ 2.7%        │ 2.7%         │ 0.0%         │
│ Memory       │ 1.0%         │ 1.0%         │ 0.0%         │
└──────────────┴──────────────┴──────────────┴──────────────┘

两个平台 node0 的算子类型占比和 8 节点聚合结果的差值都在 0.4% 以内，说明各节点间的算子分布高度一致，负载均衡，没有异常节点。后续可以选取 node0 作为代表节点做详细对比。
```

**第 3 步 · 算子级对比发现参数不匹配**

负载均衡确认后，Agent 选取各平台 node0 做算子级跨报告对比。但直接对比时发现两份报告的算子无法匹配——810E 的所有算子在 M890 侧均显示 0.0%：

```
┌────────────────────────────┬──────────────┬──────────────┬────────────────────────────────────────────┐
│ 算子                       │ 真武 810E    │ 真武 M890    │ 原因                                       │
────────────────────────────┼──────────────┼──────────────┼────────────────────────────────────────────
│ AllReduce w:128            │ 18.8%        │ 0.0%         │ M890 是 w:64，参数不同                     │
│ AllReduce c:490M w:128     │ 12.9%        │ 0.0%         │ 同上                                       │
│ GemmEx 1024x4096           │ 1.3%         │ 0.0%         │ M890 用 Matmul 4096x8192，API 和维度都不同 │
└────────────────────────────┴──────────────┴──────────────┴────────────────────────────────────────────┘

原因是两个平台并行度不同（810E 128 卡 vs M890 64 卡），AllReduce 的 worker 数参数不同；GEMM 的 BLAS API 也不同（GemmEx vs Matmul）。算子按完整参数名匹配，参数不同就无法对应。
```

**第 4 步 · 配置自定义算子规则归一化后重新对比**

Agent 配置两条自定义算子识别规则：将所有 AllReduce 归一化为同一名称（忽略 worker 数参数差异），将所有 FlashAttention 归一化为同一名称（忽略 seqlen 等参数差异）。Native 算子因两边使用相同的 PyTorch kernel 名，无需自定义规则即可天然匹配。重新跑算子级对比：

```
┌────────────────────────────┬────────────────────────────┬──────────────────────────────────────────┐
│ 算子                       │ 真武 810E 占比 / 平均      │ 真武 M890 占比 / 平均      │ 比值         │
────────────────────────────┼────────────────────────────┼────────────────────────────┼──────────────
│ AllReduce                  │ 46.2% / 45.6ms             │ 26.9% / 23.8ms             │ 0.5x         │
│ Native unrolled_elementwise│ 1.1% / 16.9μs              │ 3.8% / 52.8μs              │ 3.1x         │
│ Native elementwise         │ 1.0% / 19.0μs              │ 4.2% / 69.1μs              │ 3.6x         │
│ FlashAttention             │ 2.7% / 178μs               │ 2.9% / 171μs               │ 1.0x         │
└────────────────────────────────────────────────────────┴──────────────────────────────────────────┘

AllReduce 比值 0.5x，810E 通信更多因为 128 卡。FlashAttention 比值 1.0x，两个平台基本一致。Native 算子差异巨大：同样的 elementwise kernel，真武 M890 平均耗时 69.1μs，是真武 810E（19.0μs）的 3.6 倍。GEMM 因 API 不同（GemmEx vs Matmul）仍无法匹配。这说明并非所有算子都有差异——性能差距集中在 Native kernel 上。
```

**Agent 结论**

> Agent 输出结论：真武 M890 训练慢的主要原因是 Native kernel 效率低。

```
真武 M890 训练慢的主要原因是 Native kernel 效率低。全节点聚合显示真武 M890 的 Native 占比 38.0%，是真武 810E（12.0%）的 3 倍多。算子级对比确认同样的 elementwise kernel，真武 M890 平均耗时 69.1μs，是真武 810E（19.0μs）的 3.6 倍。两个平台的算子分布高度一致（node0 与聚合差值 <0.4%），说明这是全局性的 kernel 效率问题而非个别节点异常。建议用 Asight Compute 对 elementwise kernel 做 kernel 级分析定位根因。
```

可以要求 Agent"把结论标注到时间线上"，Agent 会在 Timeline 上添加 Insight 标注，下次打开报告还能看到。
![MCP 配置窗口](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763956623/7cc85285aeb368e83969844ce41cad28/naitive_kernel_insight.png)

---

### 3.2 生成 Asight Compute 命令做 kernel 级分析 {#32-生成-asight-compute-命令做-kernel-级分析}

端到端分析定位到真武 M890 的 Native elementwise kernel 是主要瓶颈后，下一步是做 kernel 级性能分析。Asight Systems MCP 支持根据报告中的 kernel 执行记录，查看 kernel 启动参数并自动生成 Asight Compute 采集命令。

**提问**：

> 帮我在真武 M890 报告里找一个 elementwise kernel 的 kernel，并生成 Asight Compute 采集命令。

**Agent 动作**：Agent 在报告中搜索 elementwise 类型的 kernel 执行记录，找到 `elementwise_kernel<128, 4, CUDAFunctor_add>`（持续 60.3μs），查询其启动参数并生成 Asight Compute 命令。

**Agent 输出——kernel 启动参数**：

![AI Insight 面板](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763955805/47ce1975f077259f3c031434dc4b3050/agent_output6.png)

**Agent 输出——Asight Compute 命令**：

![AI Insight 面板](https://thead-marketing-public.oss-cn-hangzhou.aliyuncs.com/1790763955992/a2740ef7d254c6024ad09df58034b3c8/agent_output7.png)

Asight Compute 命令会对 `elementwise_kernel` 做单次 kernel 级采集，生成的报告可以用来分析指令吞吐、访存效率、占用率等指标，从而定位 3.6 倍耗时差异的根因。对真武 810E 用同样方式生成 Asight Compute 命令即可做跨平台 kernel 级对比。

---

## 4. 使用建议 {#4-使用建议}

1. **从概览到细节逐步下钻**：先让 Agent 给全局概览，再针对异常点追问细节，比一上来就问"为什么慢"效果更好。

2. **让 Agent 落 Insight**：分析完可以要求 Agent"把结论标注到时间线上"，结论就变成可点击的锚点，下次打开还在。

3. **跨报告对比时注意拓扑差异**：不同并行度的报告对比时，算子级对比会因参数不匹配而失败。可以让 Agent 配置自定义算子规则归一化参数差异，或者直接用算子类型汇总对比做粗粒度分析。

---

## 5. 结语 {#5-结语}

Asight Systems MCP 为 AI 分析 asysrep 报告提供了桥梁，使 Qoder 可以直接读取报告数据、执行统计规则、标注分析结论。通过自然语言提问，Agent 可以自动完成规则选择、多报告汇总、算子对比等分析流程，将问题粗筛和定位的效率大幅提升——以往需要人工逐条跑规则、逐表解读的工作，现在几分钟内就能得到方向性结论，帮助更快聚焦到需要深入排查的环节。
