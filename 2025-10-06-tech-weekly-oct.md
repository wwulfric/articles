---
title: 10月 Tech Weekly
date: 2025-10-11 00:00
categories: [技术]
tags: [weekly]
---

## SRE Weekly

### w41

[Managing Datadog with Terraform](https://www.datadoghq.com/blog/managing-datadog-with-terraform/): 使用 terraform，直接创建datadog 的监控、仪表盘；也可以在创建 IaaS 资源的时候，顺便创建对应的 datadog 监控。terraform 是纯文本描述的、通过声明式配置文件来管理和供应基础设施资源的工具，因此适合管理大型基础设施。

---

[Demystifying Automatic Instrumentation: How the Magic Actually Works](https://opentelemetry.io/blog/2025/demystifying-auto-instrumentation/): 介绍自动插桩的一些常用技术方案。即：猴子补丁、字节码插桩、编译时插桩、eBPF 和语言运行时 API。每种技术都利用了不同编程语言和运行时环境的独特特性，无需更改代码即可添加可观察性。

- 猴子补丁：常用于 js、python、ruby 等动态语言。比如 js 的 require-in-the-middle 库允许我们在模块加载时执行此替换，拦截模块加载过程，在应用程序使用之前修改导出的函数
- 字节码插桩：对于运行在虚拟机上的语言，字节码插桩提供了一种强大的方法。这种技术通过修改虚拟机加载编译后的字节码来实现，使我们能够在指令级别注入代码。典型示例如 Java 的 Instrumentation API 
- 编译时插桩：对于 Go 等静态编译语言，编译时 instrumentation 提供了一种不同的方法。它不是在运行时修改代码，而是在构建过程中使用抽象语法树（AST）操作转换源代码。缺点是，必须要嵌入到源码构建和 CI/CD 流程中
- eBPF 插桩：eBPF 代表了一种完全不同的自动插桩方法。它不是修改应用程序代码或字节码，而是在内核级别工作，将探针附加到运行应用程序中的函数入口和出口点。
- 语言运行时 API：一些语言提供内置的 instrumentation API，提供更集成的方案。PHP 8.0 中引入的 Observer API 就是这种方案的典型例子

---

[Amazon CloudWatch Application Signals new enhancements for application monitoring](https://aws.amazon.com/blogs/mt/amazon-cloudwatch-application-signals-new-enhancements-for-application-monitoring/): AWS CloudWatch 的一些更新，提到了他们使用大模型来做根因分析的一些实践；提到了 SLO/SLI 视图。

---

[PagerDuty H2 2025 Release: 150+ Customer-Driven Features, AI Agents, and More](https://www.pagerduty.com/blog/product/product-launch-2025-h2/): PagerDuty 的 H2 2025 功能更新，包含值班排程、事件管理；智能化方面包含

- [Shift Agent](https://support.pagerduty.com/main/docs/shift-agent)：排班助手，该系统在后台主动运行——监控日历、检测潜在冲突、识别合格替岗人员、协调排班变更
- [Scribe Agent](https://support.pagerduty.com/main/docs/scribe-agent)：转录助手，可以将 zoom 会议的内容转录到 im 中，还可以进一步进行分析，更新相关的事件状态和进度等
- SRE Agent：PagerDuty 的 SRE Agent 提供了具有人工监督的全面端到端事件自动化，可以运行诊断、呈现关键上下文、提供分析、建议修复操作，并在获得批准后自动运行这些操作。它还从每个事件中学习，为将来使用生成智能手册。
- [Insights Agent](https://support.pagerduty.com/main/docs/insights-agent): 基于 Incidents、Services 和 Teams 分析，提供主动建议和按需对话式洞察。该 AI 代理可以轻松揭示运营风险、趋势和模式，继而赋能数据驱动的决策，并构建更具弹性的运营。
- AI Orchestrations：将通过利用在历史事件和故障数据上训练的机器学习，建议可操作的事件编排规则，从而将工作从被动响应转变为主动出击。

除此之外还提供了 MCP 服务，并支持和一些工具集成。

---

[Keep stakeholders informed with Datadog Status Pages](https://www.datadoghq.com/blog/status-pages/) datadog 的状态页（status page）设计。

---

[AI-powered observability: Resolve incidents faster, reduce alert fatigue, and expand access](https://grafana.com/blog/2025/10/08/grafana-assistant-ga-assistant-investigations-preview/): Grafana Assistant 是 Grafana Cloud 中的一个 AI 驱动的智能体，它使用自然语言帮助你更快地查询、构建和排除故障。它简化了常见的工作流程，如编写 PromQL、LogQL 或 TraceQL 查询、创建仪表盘和执行引导式根因分析。Assistant investigations 是一个直接构建在 Grafana Assistant 中的 SRE agent。它通过分析你的可观测性堆栈、发现异常并连接整个系统中的信号，帮助你更快地找到根本原因。你将获得清晰、有指导意义的修复建议——并且由于它嵌入在 Assistant 中，它为解决复杂事件提供了无缝的端到端工作流程。

### w42

[Remediating CVE-2025-3248: How Dynatrace Application Security protects Agentic AI applications](https://www.dynatrace.com/news/blog/remediating-cve-2025-3248-how-dynatrace-application-security-protects-agentic-ai-applications/): 介绍了CVE-2025-3248 漏洞的攻防，攻击者可以将代码注入到 AI agent 中，例如 Langflow 中的 AI agent，从而导致远程代码执行和后门。Dynatrace 的云应用程序检测和响应 (CADR) 通过检测漏洞、识别错误配置和调查可疑活动来防御此类攻击。这种多层方法包括

- 运行时漏洞分析：支持在运行时检测 Python 漏洞，可以提供关于攻击者潜在攻击途径的即时信息
- 安全态势管理和深度日志调查：比如检查缺失的网络策略

以保护基于 agent 的 AI 环境。（话说 langchain 的问题还真多……）



## 编程语言 weekly

### w41

[Python 3.14更新](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html): Python 3.14 是 Python 编程语言的最新稳定发布版，包含多项针对语言、实现和标准库的改变。 最大的变化包括 [模板字符串字面值](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html#whatsnew314-template-string-literals), [标注的迟延求值](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html#whatsnew314-deferred-annotations)，以及在标准库中对 [子解释器](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html#whatsnew314-multiple-interpreters) 的支持。

个人比较看好[安全的 CPython 外部调试器接口](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html#whatsnew314-remote-debugging)和[自由线程](https://docs.python.org/zh-cn/3.14/whatsnew/3.14.html#whatsnew314-free-threaded-now-supported)这2个功能。预计开源社区会陆续带来相关功能更新和优化。

