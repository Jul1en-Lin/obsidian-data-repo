# Codex 工作流更新-v3

> 相关笔记：[[AI/Codex 工作流更新-v2|Codex 工作流更新-v2]]

最近 OpenAI 的一篇博客： [Codex-maxxing for long-running work](https://openai.com/index/codex-maxxing-long-running-work/) 分享了关于长周期复杂任务的指南，并针对我已有的工作流作出一些更新。

长周期的复杂任务通常不是一次 prompt 改完代码就结束。它可能要经历调查、实现、预览、反馈、等 CI、继续修改、准备 PR、后续检查。该博客更像是总结了 Claude Code `goal` 和 `loop` 命令的思想 💭
![[Pasted image 20260624172022.png]]
![[Pasted image 20260624172044.png]]

## v2 现状

v2 从项目初始化开始： `setup-light` / `setup-full` 生成 `AGENTS.md`、`CLAUDE.md` 和 `docs/` 记忆文件，让 Agent 进入一个项目后，知道行为规范。它不太关注于任务线本身，比如预览页面谁来看，用户反馈放在哪里，PR review 几个小时后才来又怎么接着改，这些长期任务流程问题是 v2 未涉及的。

## 长时间复杂任务

OpenAI 这篇博客把 Codex 看成一个 persistent workspace：一件复杂任务有自己的线程、记忆、工具和审阅对象，重要的有以下几点。

**Durable thread：** 给长期任务一个固定会话，维护会话的上下文、旧决策、待办和用户反馈容易丢失。

**Memory：** 不只依赖聊天记录，而是把项目状态、决策、开放事项写进可以查看、可以 diff、可以复用的文件。对应到当前工作流里， `docs/` 就承担了 memory 的作用，跨项目 or 会话的个性要求则可以放进个人记忆或知识库。

**Steering：** Codex 工作时，用户可以继续补充 prompt。比如“先不要接短信登录”“这个错误提示太别扭”“等预览部署完再继续”。任务就不再是一次性指令，而是可以边做边调整的任务队列。

**Thread automation：** 定时任务，比如等待部署、看 review 是否更新、检查支持工单有没有新回复。

**Side panel：** 产物本身要进入审阅循环。Markdown、CSV、PDF、slides、本地页面预览都可以直接作为审阅对象，用户的评论继续变成下一轮任务上下文。

**Goals：** 目标要能有具体的量化标准：如要写清楚页面、接口、测试、异常情况分别达到什么结果。

这些点合在一起，就能把任务进化成一个工作循环：
<mark class="hltr-green-light">目标 -> 线程 -> 项目记忆 -> 调用工具 -> 产出 -> review -> 下一轮任务</mark>

这张图可以作为本次更新的总览
![[assets/codex-workflow-v3-long-running-task.png]]

## 三层模型

通过上述总结可以把工作流分成三层：

- **Thread：** 保存一条长期任务的上下文
- **Project memory：** 保存 repo 内的事实，也就是 `AGENTS.md` 和 `docs/*`
- **Execution surfaces：** 接触真实工作界面，比如 browser、Chrome、computer use、connectors、skills

而 v2 正好属于 Project memory 这一层，所以不需要做改进。另外，长期任务容易越做越多，所以边界要提前写清楚，例如可以配置相关 Hooks

**哪些任务适合长期线程？**

不是所有任务都要开长期线程。一次性命令、小改动、马上能验完的问题，按 v2 的普通流程就够。

更适合长期线程的场景：

- 会跨多轮修改
- 需要用户看预览或多次反馈
- 需要等待 CI、部署、review、第三方回复
- 需要跨工具处理，比如 GitHub + 本地代码 + 浏览器
- 后续可能沉淀成 skill
- 中断后还要接着做

## Prompt 模板

下次遇到复杂任务需要长时间工作流的场景，可以直接这样编写 prompt：
可以手动引用 `loop` `goal` 命令进一步约束 Agent，确保执行正确
```text
这是一个长期任务线程：<任务名>。

先读 AGENTS.md 和 docs/agent_workflow.md。
必要时再读 docs/project_status.md、docs/project_spec.md、docs/architecture.md。

目标：
- <目标 1>
- <目标 2>

验收标准：
1. <可以检查的结果>
2. <需要运行的检查>
3. <需要用户审阅的产物>

执行规则：
- 先调查，不要急着改代码
- 改前说明方案和影响范围
- 改后运行相关检查
- 涉及 UI 时，启动本地预览并给出地址
- 不要 push、发布、删除数据、发送消息，除非用户明确批准
- 任务中断前，更新 docs/project_status.md
```

这里的关键是验收标准要能检查。不要只写“完成登录功能”，而是写清楚哪些页面、哪些接口、哪些测试、哪些异常情况必须通过。


## Tools

此外还要说清楚，任务需要碰到哪些工具。可手动引用或在规范文档、prompt 中提示

常见选择：
- 本地网页预览：用 browser
- 需要用户登录态：用 Chrome
- 必须点桌面软件：用 computer use
- GitHub、Gmail、Slack、Calendar：用对应的 MCP
- 反复使用的流程：整理成 skill
- 需要隔一段时间回来查：用 thread automation

## checkpoint 的判断

v2 里 checkpoint 更像 commit 前的项目状态检查；这里 checkpoint 更像一次任务阶段记录。

需要 checkpoint 的情况包括：

- 当前阶段已经完成
- 任务要暂停
- 要交给另一个 Agent
- 后续需要接着做
- 出现新的风险或阻塞
- 行为、架构、依赖或项目状态发生变化

与之前的 checkpoint 思想不同，不是为了每次 commit 都按需更新记忆文档，而是为了让任务之后能接上。如果只是普通小改动，就不需要每次都更新一堆 docs。

## 复盘

v2 规定了 Agent 对项目的维护方式，v3 则是面对复杂长期任务时总结的思想


博客来源：
- [Codex-maxxing for long-running work](https://openai.com/index/codex-maxxing-long-running-work/)
