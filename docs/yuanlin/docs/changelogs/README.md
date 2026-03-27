# docs/changelogs/ - 变更日志

## 概述

`docs/changelogs/` 目录包含 Gemini CLI 的版本变更日志，记录每个版本的新功能、改进和修复。变更日志按发布渠道分为稳定版（latest）和预览版（preview）。

## 目录结构

```
changelogs/
├── index.md        # 变更日志索引（历史版本列表）
├── latest.md       # 最新稳定版变更日志
└── preview.md      # 预览版变更日志
```

## 核心组件

| 文档 | 描述 |
|------|------|
| `index.md` | 所有版本变更日志的索引和导航 |
| `latest.md` | 最新稳定版本的详细变更内容和亮点 |
| `preview.md` | 预览版的变更内容（实验性功能） |

## 依赖关系

### 内部引用

- 由 `.gemini/skills/docs-changelog/` 技能自动生成和更新
- 使用 `references/latest_template.md` 和 `references/preview_template.md` 模板
