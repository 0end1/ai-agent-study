# AI Agent 学习系列 · 新篇章规划
## 《2026 企业 AI 实战手册：从企业数字化到 AI 落地》(Stanford Enterprise AI Playbook)

> 系列定位：承接《AI-Agents-in-Depth》(001-013) 的"原理与工程"视角，转向**企业数字化落地**视角。
> 书源：Stanford Digital Economy Lab《The Enterprise AI Playbook》(Elisa Pereira, Alvin Wang Graylin, Erik Brynjolfsson 著)，116 页 / 11 章 / 51 个企业 AI 部署案例 / 41 家组织 / 9 大行业 / 7 个国家。
> 已下载至本机，可用 pdfplumber 提取。

---

## 系列概览

这份手册不是技术教程，而是**企业 AI 落地的实证研究报告**——通过对 51 个真实部署案例的研究，回答一个核心问题：**企业把 AI 部署到生产环境后，到底发生了什么？**

9 条核心发现（Key Findings）奠定了整个系列的骨架：

1. **技术不是最难的**——77% 的难点是隐性的非技术成本（变革管理、数据质量、流程再造），61% 成功项目都经历过至少一次失败。
2. **时间差异来自组织而非技术**——相同用例在 A 公司几周、在 B 公司几年，差异在高层支持与组织流程。
3. **升级式模型（Escalation）效果更好**——AI 自主处理 80%+、人类只审例外，中位数生产率提升 71%，远超审批式（Approval）的 30%。
4. **高层支持是"行动"不是"批准"**——有效的支持者每周清障、连接业务与技术、绑定 OKR、创造允许失败的文化。
5. **职能部门是最常见的阻力源**——法务/HR/风控/合规占 35% 阻力，高于内部终端用户的 23%。
6. **减员常见但非必然**——45% 部署的最大成果是减员，但 55% 是"避免招聘/再部署/不减员"。
7. **AI 营收真实但罕见**——三种模式：能转化的个性化、能赢单的速度、内部工具产品化。
8. **Agentic AI 有效但多数公司还没用**——Agentic 中位数提升 71% vs 高自动化 40%，但仅占 20% 案例。
9. **脏数据不是拦路虎，只要绕开它设计**——真正的数据挑战是访问与存储，不是整洁度。

---

## 章节与规划文章对照表

| 文章序号 | 章节 | 章主题（英文原文） | 文章主题 |
|---------|------|------------------|---------|
| **014** | Chapter 1 | Why do AI business cases underestimate real investment? | 为什么 AI 业务案例总是低估真实投入？——决定成败的隐性成本 |
| **015** | Chapter 2 | How to cross the valley of death between deployment and ROI? | 如何跨过"部署到 ROI 的死亡之谷"？——什么让同类用例从几周拖成几年 |
| **016** | Chapter 3 | How much human oversight is optimal? | 多少人监督是最优的？——审批式 vs 升级式模型，71% vs 30% |
| **017** | Chapter 4 | What separates sponsors who drive results from those who just approve budgets? | 推动结果的高管 vs 只批预算的高管——有效高层支持的四个行动 |
| **018** | Chapter 5 | Where does fatal resistance come from? | 致命阻力从哪来？——法务/HR/风控 35% 阻力与破局 |
| **019** | Chapter 6 | When productivity gains are high, what happens to headcount? | 生产率飙升时裁员会怎样？——减员、转岗还是冻结招聘 |
| **020** | Chapter 7 | Where is AI opening doors that were previously closed? | AI 打开了哪些此前关闭的门？——从提效到新营收与新能力 |
| **021** | Chapter 8 | Is agentic AI generating real value? | Agentic AI 真的创造价值吗？——自主 AI 在哪行得通、哪用简单方案 |
| **022** | Chapter 9 | How clean does enterprise data actually need to be? | 企业数据到底要多干净？——真正的挑战是访问与存储而非整洁 |
| **023** | Chapter 10 | Does rigorous security protect the project or kill it? | 严格的安全是保护项目还是杀死它？ |
| **024** | Chapter 11 | When is foundation model choice not a commodity? | 什么时候基座模型不是"商品"？——开源 vs 闭源的决策 |
| **025** | Conclusion | The technology works. The challenge is everything else. | 结论：技术可行，难的是其他一切——51 个案例的启示与行动清单 |

**共 12 篇新文章（014-025）**，正好完成"企业数字化→AI 落地"从认知到行动的闭环。

---

## 文章结构模板（沿用现有系列）

每篇文章 1500-2500 字，结构：
1. **开篇引入**：类比/故事（真实案例）
2. **核心概念通俗解析**：章节核心发现
3. **Markdown 表格对比要点**：升级式 vs 审批式、有效 vs 无效赞助等
4. **代码/配置示例**：如适用（本系列偏管理，更多用决策清单/模板）
5. **实践建议与总结**：给决策者/工程师的行动清单

---

## 执行说明

- **书源**：已下载至 `/tmp/enterprise-ai-playbook.pdf`，可随时用 pdfplumber 提取各章正文。
- **每章页码**（PDF 页）：Ch1 p12 / Ch2 p19 / Ch3 p28 / Ch4 p35 / Ch5 p44 / Ch6 p51 / Ch7 p58 / Ch8 p68 / Ch9 p78 / Ch10 p86 / Ch11 p94 / Conclusion p105。
- **推送**：文章保存到 `ai-agent-study/articles/NNN-xxx.md`，README 追加新表，同步博客仓库 `0end1.github.io`。
- **日期**：沿用"实际运行日"而非预置日期，避免 Jekyll 未来日期问题。
