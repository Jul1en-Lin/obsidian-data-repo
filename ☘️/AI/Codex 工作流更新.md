# Codex 工作流更新
> 相关笔记：[[AI|AI 知识总结]]


# 工作流框架

这篇博客旨在为从 Claude 转到 Codex 的工作者提供一些注意事项与 Tips。

Claude code 的工作方式与 Codex 还是有些许不一样，Claude code 更适合长对话的上下文，将模糊的想法转为可实现的东西，而 Codex 更专注于任务的执行，所以需要稍微改变一下我们的使用习惯。

先介绍下我的 Claude 工作流，大致为：先沉淀想法`brainstorm.md`​ → 生成规格文档`project_spec.md` → 拆架构与里程碑 → issue 化 bug → 研究报告驱动修复 → commit 同步 changelog/status，维护项目的变更日志与进度。

首先最明显的改变即是 `CLAUDE.md`​ 改为 `AGENTS.md`，但他们维护重心都是要求要短、准确、实用，不要写成又长又泛的“愿景文档”。

我的长期维护文件也有变化，新增了`decisions.md`

```text
.
├── AGENTS.md                  # Codex 的长期项目指令，替代 CLAUDE.md
├── docs/
│   ├── brainstorm.md          # 想法池，偏发散
│   ├── project_spec.md        # 产品规格，偏稳定
│   ├── architecture.md        # 架构设计
│   ├── project_status.md      # 当前进度与里程碑
│   ├── changelog.md           # 用户可读/开发可读变更记录
│   ├── decisions.md           # 重要技术决策记录
│   └── bugs/
│       └── issue-xxx-research.md  # 定位、写研究报告，再修 bug
```

我之前的工作流有一个问题，即每次commit会把变化放在 `project_status`​ 和 `changelog`​，时间长了这部分记录会非常长，这类决策后面会反复影响实现。`decisions.md`​ 专门记录技术取舍，比放进 `changelog` 更清除

在自己整理完想法文档后，建议不要让 Codex 直接让它根据对话结果自由生成项目规格，更加稳健的是执行以下指令

> 阅读 docs/brainstorm.md。
>
> 请不要修改代码。只做需求整理，生成 docs/project_spec.md。
>
> 要求：
>
> 1. 区分「已确定需求」「待确认问题」「暂不实现」
> 2. 每个核心功能必须包含：用户场景、输入、输出、边界条件、失败情况
> 3. 不要脑补未在 brainstorm 中出现的功能
> 4. 如果发现冲突，写入「待确认问题」，不要自行决定
> 5. 如果有不确定的则向用户提问，直至你有信心能完成最初的 MVP

# Bug 工作流

我在 Claude 中处理 Bug 的工作流为：复杂 bug → 写研究报告research report → 提issue → 基于研究报告修复，这套工作流同样可适用于 Codex

**Step 1：创建研究报告**

> 阅读 GitHub issue #23 和相关代码。
>
> 第一阶段：只调查，不修改代码。  
> 请生成 docs/bugs/issue-23-research.md，内容包括：
>
> 1. 现象复述
> 2. 最小复现路径
> 3. 相关文件和调用链
> 4. 根因假设，按可能性排序
> 5. 推荐修复方案
> 6. 需要新增/修改的测试
> 7. 风险点
>
> 完成后停止，不要修复。

**Step 2：基于研究报告修复 bug**

> 基于 docs/bugs/issue-23-research.md 修复 bug。
>
> 要求：
>
> 1. 先实现最小修复，不做无关重构
> 2. 添加或更新测试
> 3. 运行相关测试
> 4. 更新 docs/changelog.md 和 docs/project_status.md
> 5. 最后给出变更摘要、测试结果、风险说明

当然，可以将这些可复用的prompt封装成skill或者专属于你的命令行，这都是可行的。

# 维护记忆文档

上述提到的维护长期文件，我们可以将这套规则写入 Codex 的记忆文档中，我们每次初始化记忆文档时，Codex 就会自动帮我们维护这些长期文件，这对长时间的工作是非常重要。

同样我们也可以创建相关的skill

以在 Windows PowerShell 为例，创建这个目录：

```bash
mkdir "$HOME\.agents\skills\codex-agents-init"
mkdir "$HOME\.agents\skills\codex-agents-init\scripts"
```

​`~/.codex`​ 主要是 Codex 配置和全局 `AGENTS.md`​；用户级 Skill 的推荐位置是 `$HOME/.agents/skills`。

然后创建SKILL.md：

```bash
notepad "$HOME\.agents\skills\codex-agents-init\SKILL.md"
```

把下面内容完整粘进去👇：

> ---
>
> name: codex-agents-init  
> description: Initialize or update a repository AGENTS.md for Codex with long-term documentation maintenance rules. Use this when the user asks to initialize AGENTS.md, set up Codex project instructions, add docs/project_spec.md architecture/status/changelog rules, or create a document-driven Codex workflow.
>
> ---
>
> # Codex AGENTS.md Initialization Skill
>
> Use this skill when the user asks to initialize or update Codex project instructions, especially with phrases like:
>
> - 初始化 AGENTS.md
> - 初始化 agents
> - 创建 Codex 项目规范
> - 加上长期文件维护规则
> - 初始化 docs 工作流
> - 添加 project_spec / architecture / project_status / changelog 维护规范
> - set up AGENTS.md for this repo
> - initialize Codex instructions
>
> ## Goal
>
> Create or update the current repository so that Codex has durable project instructions and long-term documentation maintenance rules.
>
> The expected repository structure is:
>
> ```text
> .
> ├── AGENTS.md
> └── docs/
>     ├── brainstorm.md
>     ├── project_spec.md
>     ├── architecture.md
>     ├── project_status.md
>     ├── changelog.md
>     ├── decisions.md
>     └── bugs/
> ```
>
> **Required behavior**
>
> When invoked:
>
> 1. Locate the repository root.
>
>    - Prefer `git rev-parse --show-toplevel`.
>    - If that fails, use the current working directory.
> 2. Create `AGENTS.md` if it does not exist.
> 3. If `AGENTS.md` exists, preserve existing content.
>
>    - Do not delete existing project-specific rules.
>    - Do not overwrite the whole file.
>    - Append or replace only the managed block marked by:
>
> ```text
> <!-- BEGIN: codex-long-term-docs -->
> <!-- END: codex-long-term-docs -->
> ```
>
> 4. Ensure these files/directories exist:
>
>    - ​`docs/brainstorm.md`
>    - ​`docs/project_spec.md`
>    - ​`docs/architecture.md`
>    - ​`docs/project_status.md`
>    - ​`docs/changelog.md`
>    - ​`docs/decisions.md`
>    - ​`docs/bugs/`
> 5. Do not overwrite existing documentation files.
>
>    - If a file already exists, leave its content unchanged.
>    - If a file is missing, create it with a minimal useful template.
> 6. Add long-term maintenance rules to `AGENTS.md`.
>
> The rules must state:
>
> - ​`docs/brainstorm.md` is an idea pool and must not be treated as final requirements.
> - ​`docs/project_spec.md` is the source of truth for product behavior.
> - ​`docs/architecture.md` is the source of truth for system design and module boundaries.
> - ​`docs/project_status.md` tracks current milestone, progress, blockers, and next actions.
> - ​`docs/changelog.md` records user-visible or developer-visible behavior changes.
> - ​`docs/decisions.md` records important technical decisions and rationale.
> - ​`docs/bugs/` stores complex bug investigation reports.
> - For complex bugs, first create a research report under `docs/bugs/`, then implement the fix in a separate step unless the user explicitly asks otherwise.
> - Before finishing implementation tasks, update `docs/changelog.md`​ and `docs/project_status.md` when relevant.
> - Do not perform broad rewrites or unrelated refactors during initialization.
>
> 7. Prefer using the bundled script:
>
> ```bash
> powershell -ExecutionPolicy Bypass -File .agents/skills/codex-agents-init/scripts/init-codex-agents.ps1
> ```
>
> If the skill is installed in the user-level skill directory, run the script from that location instead:
>
> ```bash
> powershell -ExecutionPolicy Bypass -File "$HOME\.agents\skills\codex-agents-init\scripts\init-codex-agents.ps1"
> ```
>
> 8. After running, summarize:
>
>    - Created files
>    - Skipped existing files
>    - Updated `AGENTS.md`
>    - Any errors or manual follow-up needed
>
> **Safety rules**
>
> - Never delete existing documentation.
> - Never overwrite an existing `AGENTS.md`.
> - Never make unrelated code changes.
> - Never commit automatically unless the user explicitly asks.
> - If shell execution is unavailable, perform the same changes manually.

---

然后创建脚本：

```bash
notepad "$HOME\.agents\skills\codex-agents-init\scripts\init-codex-agents.ps1"
```

在脚本里粘贴下面这份：

```text
$ErrorActionPreference = "Stop"

function Get-RepoRoot {
  try {
    $root = git rev-parse --show-toplevel 2>$null
    if ($LASTEXITCODE -eq 0 -and $root) {
      return $root.Trim()
    }
  } catch {
    # Fallback below.
  }

  return (Get-Location).Path
}

$RepoRoot = Get-RepoRoot
Set-Location $RepoRoot

$Created = New-Object System.Collections.Generic.List[string]
$Skipped = New-Object System.Collections.Generic.List[string]
$Updated = New-Object System.Collections.Generic.List[string]

function Ensure-Directory {
  param([string]$Path)

  if (!(Test-Path $Path)) {
    New-Item -ItemType Directory -Path $Path -Force | Out-Null
    $Created.Add($Path + "/")
  } else {
    $Skipped.Add($Path + "/")
  }
}

function Ensure-File {
  param(
    [string]$Path,
    [string]$Content
  )

  $Dir = Split-Path $Path

  if ($Dir -and !(Test-Path $Dir)) {
    New-Item -ItemType Directory -Path $Dir -Force | Out-Null
    $Created.Add($Dir + "/")
  }

  if (Test-Path $Path) {
    $Skipped.Add($Path)
    return
  }

  Set-Content -Path $Path -Value $Content -Encoding UTF8
  $Created.Add($Path)
}

$AgentsBlock = @"
<!-- BEGIN: codex-long-term-docs -->

## Long-term documentation workflow

This repository uses durable documentation files under `docs/` as project memory. Before modifying code, read the relevant documentation files and preserve their roles.

### Documentation sources of truth

- `docs/brainstorm.md`: rough ideas, pain points, and early product thinking. Do not treat this file as final requirements.
- `docs/project_spec.md`: source of truth for product behavior, user scenarios, accepted requirements, non-goals, edge cases, and failure behavior.
- `docs/architecture.md`: source of truth for system design, module boundaries, data flow, key abstractions, dependencies, and trade-offs.
- `docs/project_status.md`: source of truth for current milestone, progress, blockers, open questions, and next actions.
- `docs/changelog.md`: chronological record of user-visible or developer-visible behavior changes.
- `docs/decisions.md`: important technical decisions and their rationale.
- `docs/bugs/`: complex bug investigation reports.

### Before implementation

- Read `docs/project_spec.md`, `docs/architecture.md`, and `docs/project_status.md` when relevant.
- If `docs/brainstorm.md` conflicts with `docs/project_spec.md`, follow `docs/project_spec.md`.
- If requirements are ambiguous, record assumptions or questions in `docs/project_status.md` instead of silently guessing.
- Prefer small, coherent, reversible changes.

### Complex bug workflow

For complex bugs:

1. First create a research report under `docs/bugs/`.
2. The report should include:
   - Symptom summary
   - Minimal reproduction path
   - Related files and call chain
   - Root-cause hypotheses ranked by likelihood
   - Recommended fix
   - Required tests
   - Risks and rollback notes
3. Do not implement the fix in the same step unless explicitly asked.

### Before finishing a task

- Run relevant tests, or explain why they could not be run.
- Update `docs/changelog.md` when behavior changes, a feature is added, a user-visible bug is fixed, configuration changes, or a breaking change is introduced.
- Update `docs/project_status.md` when milestone progress, blockers, or next actions change.
- Do not update `docs/changelog.md` for pure formatting, comment-only changes, test-only refactors, or internal renames with no behavior change.
- Summarize changed files, tests run, and remaining risks.

<!-- END: codex-long-term-docs -->
"@

function Ensure-AgentsMd {
  $Path = "AGENTS.md"

  if (!(Test-Path $Path)) {
    Set-Content -Path $Path -Value "# AGENTS.md`n`n$AgentsBlock`n" -Encoding UTF8
    $Created.Add($Path)
    return
  }

  $Existing = Get-Content -Path $Path -Raw

  $Pattern = "(?s)<!-- BEGIN: codex-long-term-docs -->.*?<!-- END: codex-long-term-docs -->"

  if ($Existing -match $Pattern) {
    $NewContent = [regex]::Replace($Existing, $Pattern, $AgentsBlock.Trim())
    Set-Content -Path $Path -Value $NewContent -Encoding UTF8
    $Updated.Add($Path + " managed block")
    return
  }

  $Separator = if ($Existing.Trim().Length -eq 0) { "" } else { "`n`n" }
  Add-Content -Path $Path -Value ($Separator + $AgentsBlock)
  $Updated.Add($Path + " appended managed block")
}

Ensure-AgentsMd

Ensure-File "docs/brainstorm.md" @"
# Brainstorm

This file stores rough ideas, pain points, feature intuition, and early product thinking.

Do not treat this file as final requirements.

## Raw ideas

-

## Pain points

-

## Core feature candidates

-

## Open questions

-
"@

Ensure-File "docs/project_spec.md" @"
# Project Specification

This file is the source of truth for product behavior.

## Product goal

-

## Users and scenarios

-

## Core requirements

-

## Non-goals

-

## Inputs and outputs

-

## Edge cases

-

## Failure behavior

-

## Acceptance criteria

-
"@

Ensure-File "docs/architecture.md" @"
# Architecture

This file is the source of truth for system design and module boundaries.

## Overview

-

## Module boundaries

-

## Data flow

-

## Key abstractions

-

## External dependencies

-

## Risks and trade-offs

-
"@

Ensure-File "docs/project_status.md" @"
# Project Status

## Current milestone

Milestone:

Status:

## Done

-

## In progress

-

## Blocked / Questions

-

## Next actions

1.
2.
3.
"@

Ensure-File "docs/changelog.md" @"
# Changelog

All notable changes to this project should be documented here.

## Unreleased

### Added

-

### Changed

-

### Fixed

-

### Removed

-
"@

Ensure-File "docs/decisions.md" @"
# Technical Decisions

This file records important technical decisions and their rationale.

## Template

### YYYY-MM-DD - Decision title

Status: Proposed / Accepted / Rejected / Superseded

Context:

Decision:

Alternatives considered:

Consequences:
"@

Ensure-Directory "docs/bugs"

Write-Host ""
Write-Host "Codex AGENTS.md documentation workflow initialized."
Write-Host "Repository root: $RepoRoot"
Write-Host ""

Write-Host "Created:"
if ($Created.Count -eq 0) {
  Write-Host "  - None"
} else {
  $Created | ForEach-Object { Write-Host "  - $_" }
}

Write-Host ""
Write-Host "Updated:"
if ($Updated.Count -eq 0) {
  Write-Host "  - None"
} else {
  $Updated | ForEach-Object { Write-Host "  - $_" }
}

Write-Host ""
Write-Host "Skipped existing:"
if ($Skipped.Count -eq 0) {
  Write-Host "  - None"
} else {
  $Skipped | ForEach-Object { Write-Host "  - $_" }
}
```

这样你的 Skill 结构就是：

```text
$HOME\.agents\skills\codex-agents-init\
├── SKILL.md
└── scripts\
    └── init-codex-agents.ps1
```

保存后，重启 Codex 桌面端,之后你在 Codex 里可以这样说：

> /codex-agents-init 初始化当前项目的 AGENTS.md，并加上长期文件维护规范。

或者直接说：

> 初始化 AGENTS.md，并创建 docs/project_spec.md、architecture.md、project_status.md、changelog.md 等长期维护文件。

# 结语

以上就是我目前的工作流的整体迁移，整体来说都是比较无缝衔接的，希望这篇帖子对你有所帮助~
