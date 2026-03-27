# packages/sdk/examples

## 概述

SDK 使用示例目录，展示了如何使用 `@google/gemini-cli-sdk` 创建 Agent、定义自定义工具和管理会话。

## 目录结构

```
examples/
├── simple.ts            # 基础示例：创建 Agent + 自定义工具
└── session-context.ts   # 会话上下文使用示例
```

## 核心组件

### simple.ts

展示了 SDK 最基本的使用方式：

1. 使用 `tool()` 工厂函数定义一个加法工具，包含 Zod 参数 Schema
2. 创建 `GeminiCliAgent` 实例，传入系统指令和工具列表
3. 调用 `agent.sendStream()` 发送提示词，遍历流式响应

```typescript
const myTool = tool({
    name: 'add',
    description: 'Add two numbers.',
    inputSchema: z.object({
        a: z.number(),
        b: z.number(),
    }),
}, async ({ a, b }) => {
    return { result: a + b };
});

const agent = new GeminiCliAgent({
    instructions: 'Make sure to always talk like a pirate.',
    tools: [myTool],
});

for await (const chunk of agent.sendStream('add 5 + 6')) {
    console.log(JSON.stringify(chunk, null, 2));
}
```

## 依赖关系

### 内部依赖
- `../src/index.ts` - SDK 核心模块（GeminiCliAgent, tool, z）
