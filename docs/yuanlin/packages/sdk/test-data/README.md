# test-data (sdk)

## 概述

此目录存放 `packages/sdk/` 下 SDK 模块测试所需的测试数据文件。包含多种 JSON 格式的测试场景数据，涵盖代理（agent）指令加载、会话恢复、技能（skill）加载以及工具（tool）调用的成功与错误处理等场景。同时包含一个 `skills/` 子目录，存放技能相关的测试资源。

## 目录结构

```
test-data/
├── agent-async-instructions.json
├── agent-dynamic-instructions.json
├── agent-resume-session.json
├── agent-static-instructions.json
├── skill-dir-success.json
├── skill-root-success.json
├── skills/
│   └── pirate-skill/
│       └── SKILL.md
├── tool-catchall-error.json
├── tool-error-recovery.json
└── tool-success.json
```
