# packages/devtools/src

## 概述

DevTools 服务端源码目录，包含核心的 DevTools 单例类和类型定义。负责启动 HTTP/WebSocket 服务器、管理 CLI 会话连接和日志数据。

## 目录结构

```
src/
├── index.ts    # DevTools 类 - 核心服务器实现
└── types.ts    # 日志类型定义
```

## 核心组件

### DevTools 类 (`index.ts`)

单例模式（`getInstance()`），继承 `EventEmitter`。核心功能：

- **日志管理**：网络日志上限 2000 条，控制台日志上限 5000 条，超出自动移除最早条目
- **网络日志**：支持增量更新（通过 id 匹配合并请求/响应/分块数据）
- **会话管理**：Map 结构维护活跃会话，支持心跳检测

### 类型定义 (`types.ts`)

- `NetworkLog` - 网络请求完整记录，包含分块数据（chunks）支持流式响应
- `ConsoleLogPayload` - 控制台日志负载（type + content）
- `InspectorConsoleLog` - 带元数据的控制台日志

## 依赖关系

### 外部依赖
- `ws` - WebSocket 实现
- `node:http` - HTTP 服务器
- `node:crypto` - UUID 生成
- `node:events` - EventEmitter

## 数据流

DevTools 的数据通过三个事件进行分发：
- `update` - 网络日志更新
- `console-update` - 控制台日志更新
- `session-update` - 会话列表变化
