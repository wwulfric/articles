---
title: AGENTS.md 在我们的 Agent 评估中优于 Skills 翻译
date: 2026-02-09 06:43
categories: [技术]
tags: [ai, agent]
---

翻译自 Vercel 博客文章：[AGENTS.md outperforms skills in our agent evals](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)

## The problem we were trying to solve
## 我们试图解决的问题

AI 编码 Agent 经常依赖训练数据，这些数据可能会迅速过时，尤其是对于像 Next.js 这样快速发展的框架。例如，Next.js 16 引入了 `use cache`、`connection()` 和 `forbidden()` 等 API，而 Agent 在训练期间并未见过这些 API。这导致 Agent 生成不正确或过时的代码建议，或者推荐可能在项目当前版本中不存在的 API。

目标是通过让 Agent 访问版本匹配的文档来解决这个问题，以便代码推荐能够与项目的实际 Next.js 版本精确对齐。

## Two approaches for teaching agents framework knowledge
## 教授 Agent 框架知识的两种方法

我们比较了两种方法：

### Skills（技能）

模块化的、可打包的领域知识块（提示词、工具、文档），Agent 可以根据需要调用。

### AGENTS.md

项目根目录中的 Markdown 文件，为 Agent 提供持久的、始终可用的上下文。

## We started by betting on skills
## 我们最初押注在 Skills 上

Skills 为打包文档提供了一个理论上的抽象。我们假设 Skills 在效率和模块化方面会更有优势。

## Skills weren't being triggered reliably
## Skills 无法被可靠地触发

在实践中，Agent 经常无法调用 Skills（仅 44% 的使用率）。与基线相比没有可衡量的改进；有时甚至引入了额外的噪音（通过率停留在 53-58%）。

即使提供了明确的指令，Skills 方法也只能达到最高 79% 的通过率。在没有明确指令的情况下，在 56% 的情况下，Agent 根本没有调用 Skill。

## A compressed 8KB docs index in AGENTS.md achieved 100% on Next.js 16 API evals. Skills maxed at 79%.
## AGENTS.md 中的压缩 8KB 文档索引在 Next.js 16 API 评估中达到了 100%。Skills 最高 79%。

嵌入的压缩 8KB 文档索引在 AGENTS.md 中达到了 100% 的通过率。AGENTS.md 的内容总是存在的，因此 Agent 在每次交互时都能以零摩擦获得最新文档。

### Results table
### 结果表格

| 配置方式 | 通过率 | 备注 |
|---------|-------|------|
| 基线（无文档） | 53% | - |
| Skills（默认） | 53% | Skill 经常未被调用 |
| Skills（显式使用指令） | 79% | Agent 有时会使用 Skill |
| AGENTS.md（8KB 文档索引） | 100% | Agent 始终使用 |

**关键发现：**

- 在 56% 的测试中，Skill 从未被 Agent 调用
- AGENTS.md 作为持久上下文，总是被引用

## Here's what we learned and how to set it up
## 以下是我们学到的经验以及如何设置

### Why did AGENTS.md outperform?
### 为什么 AGENTS.md 表现更好？

**持久上下文：** AGENTS.md 在每一轮交互时都会被附加到 Agent 的上下文中。Agent 无法忽略它，并且总能以零摩擦获得最新文档的注入。

**无需决策点：** 与 Skills 不同（Agent 必须识别需求并触发 Skill），AGENTS.md 的内容始终存在，因此 Agent 不会忽略文档。这消除了一个关键的失败点——Agent 可能无法意识到需要查阅文档。

**简单性：** AGENTS.md 在概念上简单，不依赖 Agent 启发式或不断演进的 Skill 调用逻辑。

**实用性：** 虽然 Skills 对于打包跨项目的工作流很强大，但 AGENTS.md 为紧密范围的、版本化的文档提供了一致的准确性。

### How to set up AGENTS.md
### 如何设置 AGENTS.md

在项目根目录放置 AGENTS.md 文件，包含 API 摘要、自定义代码模式和版本注意事项。

- 可以是压缩的（即使 <8KB 也足够涵盖 Next.js 16 完整 API 索引）
- 确保包含常见 API 使用的清晰代码示例、注意事项以及项目的确切版本特性

**AGENTS.md 示例代码片段：**

```markdown
# Next.js 16 项目文档

## useCache()
- [新功能] 在服务器端缓存数据，用于 SSR。参见：[链接]

## connection()
- 处理 Next.js 16 中的客户端-服务器流式传输。

## forbidden()
- 处理受保护的路由和权限。

...更多文档...
```

### When to use Skills
### 何时使用 Skills

Skills 对于以下场景仍然有价值：

- **可复用的工作流：** 代码审查规则、可访问性审计等跨项目的标准化流程
- **复杂的多步骤任务：** 需要显式调用控制的地方
- **模块化和重用：** 当相同的能力需要在多个项目中使用时

### When to use AGENTS.md
### 何时使用 AGENTS.md

- **项目特定文档：** 版本特定的 API 文档，特别是对于新 API 或专有框架
- **始终需要的上下文：** 关键的项目知识，Agent 在每次交互时都应该知道
- **紧密范围的知识：** 可以压缩到小尺寸（<8KB）的文档

### Comparison table
### 对比表格

| 方法 | 优点 | 缺点 |
|-----|------|------|
| **AGENTS.md** | - 总是被注入，无法被忽略<br>- 易于维护<br>- 对版本特定/项目特定知识可靠 | - 每轮都消耗一些提示词 token<br>- 扩展到超大文档可能需要创造性的索引/摘要 |
| **Skills** | - 可复用、标准化<br>- 不总是在上下文中（节省 token）<br>- 适合工作流/专家配方 | - Agent 不能可靠调用<br>- 需要显式触发<br>- 可能被忽略，导致错误 |

## Conclusion
## 结论

Vercel 发现，对于关键的、版本匹配的框架文档，将压缩文档索引注入 AGENTS.md 产生了远好于当今 Agent 领域中使用 Skills 的结果（100% 通过率 vs 79% 最高通过率）。

AGENTS.md 的始终在线上下文胜过了许多文档驱动的编码场景中按需、由 Agent 调用的模型。Skills 对于模块化的、生态系统范围的自动化仍然有价值——根据知识的特定性和可重用性来选择。

## Further reading
## 延伸阅读

- [完整文章：AGENTS.md outperforms skills in our agent evals - Vercel 博客](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Cursor 社区讨论](https://forum.cursor.com/t/agents-md-outperforms-skills-in-our-agent-evals/150242)
- [FAQ：什么是 Skill，如何使用？](https://vercel.com/blog/agent-skills-explained-an-faq)
- [GitHub：agent-skills 仓库](https://github.com/vercel-labs/agent-skills)
- [完整指南：AGENTS.md and SKILLS.md](https://ai-blog-peach.vercel.app/blog/agents-md-skills-md)

---

**原文链接**：https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals
