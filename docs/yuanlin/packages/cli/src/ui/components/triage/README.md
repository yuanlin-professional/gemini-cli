# triage

## 概述

`triage` 目录包含问题分类相关的组件，用于辅助 Bug 报告的重复检测和 Issue 管理功能。

## 目录结构

```
triage/
├── TriageDuplicates.tsx  # Issue 重复检测展示组件
└── TriageIssues.tsx      # Issue 列表展示组件
```

## 架构图

```mermaid
graph LR
    BUG[/bug 命令] --> TD[TriageDuplicates 重复检测]
    BUG --> TI[TriageIssues Issue列表]
```

## 核心组件

| 组件 | 职责 |
|------|------|
| `TriageDuplicates` | 在提交 Bug 报告前展示可能的重复 Issue |
| `TriageIssues` | 展示相关 Issue 列表供用户参考 |

## 依赖关系

### 内部依赖
- `../../contexts/`: ConfigContext
- `../shared/`: 共享组件

### 外部依赖
- `ink`: 终端渲染
