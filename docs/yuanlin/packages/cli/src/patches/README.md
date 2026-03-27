# patches (运行时补丁)

## 概述

`patches` 目录包含对第三方依赖库的运行时补丁（monkey-patch / 模块替换）。这些补丁通过构建配置（如 bundler alias）将原始第三方模块替换为自定义实现，用于修复特定环境下的兼容性问题。

## 目录结构

```
patches/
└── is-in-ci.ts   # 替换 is-in-ci 包，强制返回 false
```

## 架构图

```mermaid
graph LR
    subgraph 构建时替换
        Bundler[打包器 / 模块别名]
    end

    subgraph 原始模块
        IsInCI_Orig[is-in-ci 原始包<br/>检测 CI 环境]
    end

    subgraph 补丁模块
        IsInCI_Patch[is-in-ci.ts 补丁<br/>始终返回 false]
    end

    subgraph 消费方
        Ink[ink 库<br/>终端 React 渲染]
    end

    Bundler -->|别名替换| IsInCI_Patch
    IsInCI_Orig -.->|被替换| Bundler
    Ink -->|导入 is-in-ci| Bundler
    Bundler -->|实际加载| IsInCI_Patch
```

## 核心组件

### `is-in-ci.ts` — CI 环境检测补丁

**问题背景**：`ink`（Gemini CLI 的终端 UI 框架）依赖 `is-in-ci` 包检测是否在 CI 环境中运行。在 CI 环境下，`ink` 会跳过交互式 UI 渲染，导致 CLI 在某些被误判为 CI 的环境中无法正常显示界面。

**补丁策略**：用一个始终返回 `false` 的模块替换 `is-in-ci`，确保 `ink` 在任何环境下都正常渲染交互式 UI。

**安全性**：此补丁是安全的，因为 `is-in-ci` 仅在 `ink`（交互式代码路径）中使用，不影响 CLI 的非交互式/自动化模式。

参考：[Issue #1563](https://github.com/google-gemini/gemini-cli/issues/1563)

```typescript
// 补丁核心代码
const isInCi = false;
export default isInCi;
```

## 依赖关系

| 依赖方向 | 目标 | 说明 |
|---------|------|------|
| `ink` → `is-in-ci` | 被替换的依赖 | 原始 CI 检测逻辑被此补丁覆盖 |
| 构建配置 | 打包器别名 | 通过模块别名机制将 `is-in-ci` 指向此补丁文件 |

## 数据流

```mermaid
sequenceDiagram
    participant Ink as ink 终端渲染库
    participant Bundler as 打包器/模块解析
    participant Patch as is-in-ci.ts 补丁

    Ink->>Bundler: import isInCi from 'is-in-ci'
    Bundler->>Patch: 别名重定向到 patches/is-in-ci.ts
    Patch-->>Ink: false（永远不是 CI）
    Ink->>Ink: 正常渲染交互式 UI
```
