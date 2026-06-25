# Codex 工作流更新-v2
> 相关笔记：[[AI/Codex 工作流更新|Codex 工作流更新]]

# 背景

这次主要是在整理 Codex / Claude Code / 其他 Agent 之间的项目初始化工作流。
之前有两个入口，一个是偏 Claude 的 `setup`，主要维护 `CLAUDE.md`；另一个是偏 Codex 的 `codex-agents-init`，主要维护 `AGENTS.md`。刚开始看起来只是名字不同，但实际用的时候会发现，这两个入口很容易记混，而且 Agent 自己也不一定能稳定判断当前到底运行在 Claude Code、Codex，还是其他兼容环境里。

所以这次调整的重点在于统一入口：以后都走 `setup` 系列，只是根据项目复杂度分成 `setup-light` 和 `setup-full`。

# 优化

最后没有继续保留默认 `setup`，也不再使用 `codex-agents-init` 这个名字，而是拆成两个更明确的入口：`setup-light` 和 `setup-full`。
`setup-light` 保留做小项目、脚本、demo、短期实验，只保留最基本的项目状态和 Agent 工作流记录：

```text
.
├── AGENTS.md
├── CLAUDE.md
└── docs/
    ├── project_status.md
    └── agent_workflow.md
```

`setup-full` 适合长期项目、产品型项目、多 Agent 接力，需要完整一点的项目记忆：

```text
.
├── AGENTS.md
├── CLAUDE.md
└── docs/
    ├── brainstorm.md
    ├── project_spec.md
    ├── architecture.md
    ├── project_status.md
    ├── changelog.md
    ├── decisions.md
    ├── agent_workflow.md
    └── bugs/
```

这里比较重要的是 `AGENTS.md` 和 `CLAUDE.md` 的关系。Codex 原生读取 `AGENTS.md`，Claude Code 则通过 `CLAUDE.md` 里的 `@AGENTS.md` 间接读取同一份规则。也就是说真正维护的只有 `AGENTS.md`，`CLAUDE.md` 只是桥接文件，避免两边各写一套规则，后面越改越不一致。

# setup 现在负责什么

新的 setup 系列不再只服务 Claude，而是作为通用的项目初始化入口。它会创建或更新 `AGENTS.md`，写入 `CLAUDE.md` 的桥接内容，按 light / full 创建对应的 `docs/` 文件，同时迁移旧的 `codex-long-term-docs` block，移除旧的 `claude-long-term-docs` block。

这里需要注意的是，它只替换 managed block，不会整文件覆盖。已经存在的用户规则、已有的 docs 文件，都应该保留下来。`setup-light` 实际调用的是 `--profile light`，`setup-full` 调用的是 `--profile full`。

`codex-agents-init` 目录已经删掉了，主要是避免以后又出现两个入口。后续如果继续改初始化逻辑，优先改 `setup-light` / `setup-full` 的 skill，尽量不要重新造一个新名字。

# AGENTS.md 的定位

这次我不想让 `AGENTS.md` 变成一份很长的操作手册。它应该告诉 Agent 必须遵守什么，以及更详细的 workflow 去哪里读。checkpoint、commit、bug、handoff 的细节步骤，放到 `docs/agent_workflow.md` 会更合适。

所以 `AGENTS.md` 里保留的是文件职责：`docs/brainstorm.md` 是想法池，不能当最终需求；`docs/project_spec.md` 是产品行为的事实来源；`docs/architecture.md` 是系统设计和模块边界；`docs/project_status.md` 记录当前里程碑、进度、阻塞和下一步；`docs/changelog.md` 记录有意义的用户可见或开发可见变化；`docs/decisions.md` 记录重要技术决策；复杂 bug 则先进入 `docs/bugs/` 做研究报告。

这次新增了一条比较关键的规则：

```text
Before any git commit, check whether the change requires a checkpoint.
```

意思是提交前不能只想着 `git commit`，要先判断这次改动会不会影响项目长期记忆。如果影响产品行为、架构、依赖、项目进度、阻塞、下一步，或者涉及长时间任务交接，就要先更新相关 docs。

具体的 checkpoint / commit / bug / handoff 流程则放到 `docs/agent_workflow.md`。这样 `AGENTS.md` 不会越来越长，Agent 真要执行的时候也有地方继续读。

# checkpoint 的作用

之前的 `/update-docs-and-commit` 是有用的，因为它能让 AI 长时间自主工作时留下一个项目状态记录。但是这个命令是会在每次提交都会更新文档，对小项目来说没问题，但是对于一些长期的项目，还需要进一步的优化

所以后面改成 `/checkpoint`。

`/checkpoint` 的逻辑是：先读当前项目记忆，优先看 `docs/agent_workflow.md`，再看 `git status` 和 diff，判断这次状态是否值得长期保存。如果值得保存，就更新 `docs/project_status.md`，按需更新 `docs/changelog.md`、`docs/decisions.md`，复杂问题则补 `docs/bugs/` 报告，最后再 commit 一次作为恢复点。

它的作用更偏向状态判断：让 AI 在一个任务阶段结束时判断，现在这个状态以后还需不需要恢复。如果只是普通小改动，可以正常提交；但如果这次改动影响行为、架构、依赖、进度、阻塞、交接上下文或未解决风险，就应该先 checkpoint。 我觉得 AI 对能力越强，对他的 Hermes 应该适当放宽一些，让 AI 自主决定提交～

当前规则是：

```text
The agent may propose a checkpoint, but must create one when changes affect behavior, architecture, dependencies, project status, blockers, next actions, long-running task progress, handoff context, or unresolved risk.
Commit only the checkpoint-relevant files and current work.
Do not push unless the user explicitly asks or the current task grants push/publish authorization.
```

这里默认不 push。这样可以让 AI 自主维护状态，但不会未经授权把内容发布出去。

# setup skill 排版重构

后来又对 `setup-light` 和 `setup-full` 的 `SKILL.md` 做了一次排版重构。

之前的问题是模板内容太靠前，Agent 一进来先看到一大段生成内容，反而不容易立刻知道自己应该怎么执行。后面改成先说明 profile 和事实来源，再写执行流程和安全规则，最后才放模板。

新的主结构是：

```text
Profile
Sources Of Truth
Workflow
Safety Rules
Minimal Templates
Managed Block
```

这样做的好处是，`Profile` 先告诉 Agent light / full 分别适合什么项目；`Sources Of Truth` 明确 `AGENTS.md`、`CLAUDE.md`、`SKILL.md` 的职责；`Workflow` 放在前面，Agent 不需要先读完大段模板；`Safety Rules` 提前强调不能覆盖用户已有内容、不能自动 commit、不能做无关修改。

`Minimal Templates` 和 `Managed Block` 继续保留在 `SKILL.md` 中，脚本仍然从 skill 中抽取模板和 managed block。这样以后只需要改 skill，就能更新生成结果，不需要在脚本里再重复维护一份模板。

# 复盘

这次更新的核心是把工作流拆清楚。初始化规则交给 `setup-light` 和 `setup-full`，长期项目记忆放进 `AGENTS.md` 和 `docs/`，Claude Code 通过 `@AGENTS.md` 复用同一套规则，细流程放进 `docs/agent_workflow.md`，长时间自主工作再靠 `/checkpoint` 建立恢复点。

我觉得这里比较重要的一点是：`AGENTS.md` 只负责放最核心的规则和文件职责，尽量的精简化，如果需更详细的说明可给出对应文件的索引，让 AI 按需寻找。比如一开始我规定 `AGENTS.md` 里放着具体的 workflow 流程，具体到每一步都写上去了，那我们就可以将这些具体步骤放到 `docs/agent_workflow.md`，在 `AGENTS.md` 中只需给出对应的文档索引，模型就会去主动找，这样后面维护起来更清楚。

这次还有一个经验：工作流本身也可以让 AI 参与 review，减少一次性拍定方案的风险。最开始只是想合并 Claude 和 Codex 的初始化入口，后来让 AI 从批判角度检查，发现 `AGENTS.md` 承载了太多流程说明；再继续拆流程，又发现小项目不需要完整 docs，所以才拆成 light 和 full；最后又发现脚本不应该重复维护大量模板内容，于是改成从 `SKILL.md` 里抽取模板。
