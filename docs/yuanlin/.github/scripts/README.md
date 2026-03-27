# .github/scripts/

## 概述

该目录包含被 GitHub Actions 工作流调用的 **辅助脚本**，共 4 个文件。这些脚本主要用于 PR 分类（triage）、Issue 标签同步、回填等仓库治理自动化任务。脚本分为 Bash 脚本和 Node.js（CommonJS）脚本两种类型。

## 目录结构

```
.github/scripts/
├── pr-triage.sh                   # PR 分类脚本（Bash）：检查关联 Issue、同步标签
├── sync-maintainer-labels.cjs     # 维护者标签同步（Node.js）：递归遍历根 Issue 的所有子 Issue 并打标签
├── backfill-need-triage.cjs       # 回填 need-triage 标签（Node.js）
└── backfill-pr-notification.cjs   # 回填 PR 通知（Node.js）
```

## 架构图

```mermaid
graph LR
    subgraph GitHub Actions 工作流
        WF1[gemini-scheduled-pr-triage.yml]
        WF2[label-workstream-rollup.yml]
        WF3[其他调度工作流]
    end

    subgraph 脚本层
        PRT[pr-triage.sh<br/>PR 分类]
        SML[sync-maintainer-labels.cjs<br/>维护者标签同步]
        BNT[backfill-need-triage.cjs<br/>回填分类标签]
        BPN[backfill-pr-notification.cjs<br/>回填 PR 通知]
    end

    subgraph 外部服务
        GH_API[GitHub API<br/>gh CLI / Octokit]
    end

    WF1 --> PRT
    WF2 --> SML
    WF3 --> BNT
    WF3 --> BPN

    PRT --> GH_API
    SML --> GH_API
    BNT --> GH_API
    BPN --> GH_API
```

## 核心组件

### 1. pr-triage.sh -- PR 分类脚本

主要功能：
- **检查所有开放的 PR** 是否关联了 Issue
- 未关联 Issue 的非草稿 PR 会被添加 `status/need-issue` 标签
- 已关联 Issue 的 PR 会从关联 Issue 同步 `area/*`、`priority/*`、`help wanted` 等标签
- 使用 **平面字符串缓存**（兼容 Bash 3.2）避免重复查询 Issue 标签
- 输出需要评论提醒的 PR 列表到 `GITHUB_OUTPUT`

关键技术细节：
- 通过 `gh pr list` 批量获取所有 PR 数据，用 `jq` 提取关联 Issue
- 支持单个 PR 处理（`PR_NUMBER` 环境变量）和批量处理两种模式

### 2. sync-maintainer-labels.cjs -- 维护者标签同步

主要功能：
- 从预定义的 **根 Issue** 出发，递归发现所有子 Issue（BFS 广度优先搜索）
- 两种子 Issue 发现方式：GitHub 原生子 Issue 和 Markdown 任务列表引用
- 为公共仓库中的所有后代 Issue 添加 `maintainer only` 标签
- 同时移除 `status/need-triage` 标签
- 支持 `--dry-run` 模式
- 使用 **GraphQL API** 高效获取 Issue 数据，支持分页

核心常量：
- `ROOT_ISSUES`：3 个根 Issue（#15374, #15456, #15324）
- `PUBLIC_REPO`：`gemini-cli`
- `PRIVATE_REPO`：`maintainers-gemini-cli`

## 依赖关系

| 脚本 | 运行时依赖 | 调用方式 |
|------|-----------|---------|
| pr-triage.sh | `gh` CLI、`jq` | Bash 直接执行 |
| sync-maintainer-labels.cjs | `@octokit/rest` | `node` 执行 |
| backfill-need-triage.cjs | `@octokit/rest` | `node` 执行 |
| backfill-pr-notification.cjs | `@octokit/rest` | `node` 执行 |

## 数据流

```mermaid
sequenceDiagram
    participant WF as GitHub Actions 工作流
    participant PRT as pr-triage.sh
    participant SML as sync-maintainer-labels.cjs
    participant API as GitHub API

    Note over WF: 定时触发

    WF->>PRT: 执行 PR 分类
    PRT->>API: gh pr list（获取所有开放 PR）
    API-->>PRT: PR 列表 + 关联 Issue
    loop 每个 PR
        PRT->>API: gh issue view（获取 Issue 标签，带缓存）
        API-->>PRT: Issue 标签
        PRT->>API: gh pr edit（同步标签）
    end
    PRT-->>WF: 输出需要评论的 PR 列表

    WF->>SML: 执行维护者标签同步
    SML->>API: GraphQL 查询根 Issue
    loop BFS 遍历子 Issue
        SML->>API: GraphQL 查询子 Issue 数据
        API-->>SML: Issue 数据 + 子 Issue 列表
    end
    SML->>API: 添加 maintainer only 标签
    SML->>API: 移除 need-triage 标签
```
