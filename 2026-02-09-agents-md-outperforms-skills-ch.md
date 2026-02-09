---
title: AGENTS.md 在 Agent 评估中优于 Skills 翻译
date: 2026-02-09 06:43
categories: [技术]
tags: [ai, agent]
---

翻译自 Vercel 博客文章：[AGENTS.md outperforms skills in our agent evals](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)

## 背景与问题

Vercel 构建了一个评估系统，用于测试编码 Agent 在使用 Next.js 16 API 方面的能力——包括 `use cache`、`connection()` 和 `forbidden()` 等新功能，这些特性是当前 LLM 训练数据中所没有的。

挑战在于：**如何在编码任务期间为 Agent 提供最新的、版本匹配的文档**，尤其是当模型知识落后于最新框架时？

## 评估的两种方法

Vercel 测试了两种为 Agent 提供框架知识的方法：

### 1. Skills（技能）

- 结构化的、可复用的提示词、工具和文档包
- Agent 应该能够识别何时需要帮助，并按需调用相关 Skill

### 2. AGENTS.md

- 放置在项目根目录的 Markdown 文件
- 其内容会自动注入到每一轮 Agent 交互中，为 Agent 提供持久的、有保证的上下文

Vercel 最初的假设是 Skills 在效率和模块化方面更有优势。

## 实验结果

| 配置方式 | 通过率 | 备注 |
|---------|-------|------|
| 基线（无文档） | 53% | - |
| Skills（默认） | 53% | Skill 经常未被调用 |
| Skills（显式使用指令） | 79% | Agent 有时会使用 Skill |
| AGENTS.md（8KB 文档索引） | 100% | Agent 始终使用 |

**关键发现：**

- 在 **56% 的测试中**，Skill 从未被 Agent 调用
- AGENTS.md 作为持久上下文，总是被引用

## 为什么 AGENTS.md 表现更优？

### 1. 自动上下文注入

AGENTS.md 在*每一轮*交互时都会被附加到 Agent 的上下文中。Agent 无法忽略它，并且总能以零摩擦获得最新文档的注入。

### 2. Skill 使用不足

在超过一半的情况下，Agent 未能自动调用 Skill。即使 Skill 可用，Agent 也并不总是意识到*需要*调用它们。

### 3. 指令敏感性

只有在*显式指令*的情况下，基于 Skill 的文档才达到 79% 的通过率——这表明 Agent 还不擅长自我诊断何时缺乏框架知识。

### 4. 噪音/干扰

Skills 不仅未能超越基线——有时还会使情况变糟，因为未使用的 Skill 可能会分散注意力或引入无关的上下文。

## 技术设置

### 如何配置 AGENTS.md

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

...更多文档...
```

### Skills 方法参考

- Skill 是一个目录结构（包含 SKILL.md、脚本、引用），可以使用 `npx skills add vercel-labs/agent-skills` 安装
- Skills 对于可复用的工作流（如代码审查规则、可访问性审计）效果很好，但对于核心项目 API 上下文来说可靠性较差，除非进行精心的提示工程

## 优劣对比

| 方法 | 优点 | 缺点 |
|-----|------|------|
| **AGENTS.md** | - 总是被注入，无法被忽略<br>- 易于维护<br>- 对版本特定/项目特定知识可靠 | - 每轮都消耗一些提示词 token<br>- 扩展到超大文档可能需要创造性的索引/摘要 |
| **Skills** | - 可复用、标准化<br>- 不总是在上下文中（节省 token）<br>- 适合工作流/专家配方 | - Agent 不能可靠调用<br>- 需要显式触发<br>- 可能被忽略，导致错误 |

## 社区见解

- 许多开发者都认同"直接附加文档"比 Skills 更可靠，因为 Agent 往往会忽略附加规则/Skills，除非被强制使用
- 一些人青睐 AGENTS.md，因为它在概念上简单，不依赖 Agent 启发式或不断演进的 Skill 调用逻辑
- "Skills"标准对于打包和重用领域知识仍然有价值，但目前，*关键项目知识*在始终注入的 Markdown 上下文中更安全

## 结论

Vercel 发现，对于*关键的、版本匹配的框架文档*，将压缩文档索引注入 AGENTS.md 产生了远好于当今 Agent 领域中使用 Skills 的结果（100% 通过率）。Skills 对于可复用的工作流类型知识仍然很好，但为了确保 Agent 不会遗漏关键上下文——通过 Markdown 注入是目前的最佳实践。

## 延伸阅读

- [完整文章：AGENTS.md outperforms skills in our agent evals - Vercel 博客](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Cursor 社区讨论](https://forum.cursor.com/t/agents-md-outperforms-skills-in-our-agent-evals/150242)
- [FAQ：什么是 Skill，如何使用？](https://vercel.com/blog/agent-skills-explained-an-faq)
- [GitHub：react-best-practices（含 AGENTS.md 示例）](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)

---

**原文链接**：https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals
