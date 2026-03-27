# packages/test-utils/src/fixtures

## 概述

测试夹具目录，包含预定义的测试 Agent 定义，用于集成测试和评估测试中快速创建标准化的 Agent 配置。

## 目录结构

```
fixtures/
└── agents.ts    # 预定义测试 Agent
```

## 核心组件

### TestAgent 接口

```typescript
interface TestAgent {
    name: string;          // Agent 唯一名称
    definition: string;    // 完整的 YAML/Markdown 定义
    path: string;          // 标准保存路径（.gemini/agents/{name}.md）
    asFile(): Record<string, string>;  // 便捷转换为文件映射
}
```

### TEST_AGENTS 预定义 Agent

| Agent | 名称 | 描述 | 工具 |
|-------|------|------|------|
| `DOCS_AGENT` | docs-agent | 文档更新专家 | read_file, write_file |
| `TESTING_AGENT` | testing-agent | 测试编写专家 | read_file, write_file |

每个 Agent 的 `asFile()` 方法返回 `{ ".gemini/agents/{name}.md": definition }`，便于直接传入测试文件系统构建器。

## 依赖关系

无外部依赖。
