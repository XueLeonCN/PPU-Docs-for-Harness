# 真武 PPU 官方资料

本目录保存面向 Agent 检索的真武 PPU 官方技术资料。

```text
reference/
├── README.md
├── catalog.md
├── manifest.json
├── architecture/
├── programming-guide/
├── blog/
├── assets/
└── skills/
    └── zhenwu-ppu-operator-optimizer/
        └── SKILL.md
```

- `catalog.md`：按主题组织的详细摘要、关键内容和适用问题。
- `manifest.json`：机器可读索引，包含标题、路径、来源 URL、标签和完整性哈希。
- `architecture/`：官方架构白皮书。
- `programming-guide/`：官方 HGGC/TIX 编程与优化指南。
- `blog/`：官方技术博客。
- `assets/`：博客内嵌图表的本地副本。
- `skills/zhenwu-ppu-operator-optimizer/`：面向通用业务场景的真武算子开发与优化工作流。

## 来源约定

每条资料都包含：

- `source_url`：网站阅读链接；
- `source_api_url`：网站返回正文的接口；
- `source_authority: official`：官方来源标识；
- `publisher: 平头哥开发者网站`；
- `source_date`、`source_version`、`source_format`；
- `captured_at`：抓取时间；
- `source_sha256`：接口返回原始正文的 SHA-256；
- `tags`：检索标签。

`manifest.json` 另记录生成文件的 `output_sha256`。资料不保留个人作者信息。

## 正文处理

- 4 篇编程文档及 7 篇 Markdown 技术博客：技术正文直接使用网站接口返回的 Markdown 原文。
- 架构白皮书：保留全部技术正文，仅移除版权页、免责声明、保密等级表、PDF 页标记、作者列和第三方商标脚注。原始接口正文哈希记录在 `source_sha256`，清理后正文哈希记录在 `prepared_body_sha256`。
- HTML 技术博客：抽取官方网站提供的 `.docBody` 正文，转换为 Markdown，保留表格、代码、公式、图题和链接。
- KaTeX 公式使用网站内嵌的 TeX annotation；对来源 HTML 中被控制字符破坏的 `\times`、`\text` 等序列做无损恢复。
- `data:image/...` 内嵌图表解码到 `assets/blog/<id>/`，正文使用相对路径引用，避免 Markdown 被超长 base64 污染。

## 官方来源

- 架构白皮书：`projectId=48`
- 编程指南：`projectId=39`，`chapterId=1041..1044`
- 技术博客：`bizType=BLOG`

逐篇链接见 `manifest.json` 或各文件 frontmatter。
