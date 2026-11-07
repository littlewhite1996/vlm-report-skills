# 让知识来找你：多模态 AI 行业洞察助手

让大模型定期帮你读行业动态、筛出值得关注的变化，并把判断和原文放在一起。每周或每月收到一份可回查的多模态报告，你可以把时间花在理解变化和形成判断上。

这个仓库提供 **两份可复用的 Skill** 和 **一份原版 HTML 报告**。第一次可以从最近 7 天开始，以后按你自己的节奏续报。

**[下载两份 Skill 和原版示例（ZIP）](https://github.com/littlewhite1996/vlm-report-skills/archive/refs/heads/main.zip)** · **[查看原版 HTML 文件](sample/index.html)**

[![原版报告首页：本期主线、分类筛选和正文配图](assets/report-preview.webp)](sample/index.html)

## 每期帮你回答三个问题

- **发生了什么？** 从多模态模型、Agent、具身智能等方向筛出亮点，区分正式上线、灰测和待核实。
- **为什么值得关注？** 把相关更新串成几条主线，解释它们可能改变什么，再挑出值得优先读的内容。
- **我怎么核对？** 保留原文链接和相关配图；引用榜单或跑分时，说明评测条件与适用范围。

## 原版报告长什么样？

示例覆盖 **2026.10.29–11.06**：从 76 篇报道中整理出 33 条资讯、5 条主线和 Top 5。下载后双击 `sample/index.html`，即可离线查看；点开原文链接时需要联网。

<details>
<summary>展开：看看分类资讯流</summary>

[![原版报告的分类资讯流，含配图和来源链接](assets/report-filter.webp)](sample/index.html)

</details>

## 从第一期开始

1. 在仓库点 **Code → Download ZIP**。解压后，把[多模态报告 Skill](vlm-monthly-report/SKILL.md)和[HTML 配图 Skill](html-image-standards/SKILL.md)交给 Codex 或 Claude Code。
2. 说清你关心的方向、时间范围、信源和输出位置。可以直接用下面这条指令。
3. 先试跑一期。想固定每周或每月收到报告，再用你所用工具的定时功能运行同一流程；续报时从上期截止日接着做。

> 读取这两份 SKILL.md。总结过去 7 天的多模态模型动态，优先关注图像与视频生成、视觉理解和具身智能。提炼 3 条主线，每条说明发生了什么、为什么值得关注、证据在哪。区分正式上线、灰测和待核实，输出为单文件 HTML。

两份 Skill 分别负责调研写作和配图核验。生成新报告时通常需要联网；图片处理示例用到 Python、requests 和 Pillow，视觉审图可使用你自己的模型配置。
