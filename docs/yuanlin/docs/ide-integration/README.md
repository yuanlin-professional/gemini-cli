# docs/ide-integration/ - IDE 集成文档

## 概述

`docs/ide-integration/` 目录描述 Gemini CLI 与 IDE（集成开发环境）的集成方案。包括 IDE Companion 扩展的技术规范和集成指南。

## 目录结构

```
ide-integration/
├── index.md                 # IDE 集成概述
└── ide-companion-spec.md    # IDE Companion 扩展技术规范
```

## 核心组件

| 文档 | 描述 |
|------|------|
| `index.md` | IDE 集成的概念和支持的 IDE 列表 |
| `ide-companion-spec.md` | IDE Companion 扩展的通信协议和 API 规范 |

## 依赖关系

### 内部引用

- 与 `packages/vscode-ide-companion/` VSCode 扩展包关联
