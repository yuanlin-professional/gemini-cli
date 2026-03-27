# packages/core/src

## 概述

`packages/core/src` 是 Gemini CLI 的核心源码目录，包含了 CLI 工具的所有核心业务逻辑。该包基于 Google Gemini API（`@google/genai`）构建，提供了从 LLM 客户端通信、工具调用、策略引擎、沙箱执行到 MCP 协议支持的完整 AI Agent 框架。入口文件 `index.ts` 统一导出所有子模块的公共 API。

## 目录结构

```
src/
├── index.ts                 # 包入口，统一导出所有公共 API
├── index.test.ts            # 入口测试
├── __mocks__/               # 测试 mock 文件
├── mocks/                   # 额外 mock 模块
├── agent/                   # Agent 会话管理（AgentSession、事件翻译、内容工具）
├── agents/                  # Agent 加载器、类型定义、浏览器 Agent 等
├── availability/            # 模型可用性与策略辅助
├── billing/                 # 计费模块
├── code_assist/             # Code Assist 集成（OAuth2、服务端、管理员控制）
├── commands/                # CLI 命令（init、memory、restore、extensions）
├── config/                  # 配置管理（模型配置、常量、存储、注入服务、完整性校验）
├── confirmation-bus/        # 确认总线（用于工具执行前的用户确认）
├── core/                    # 核心 LLM 交互层（Client、GeminiChat、ContentGenerator、Prompts、Turn）
├── fallback/                # 回退处理逻辑
├── hooks/                   # 钩子系统（Agent 生命周期钩子）
├── ide/                     # IDE 集成（检测、客户端、安装器）
├── mcp/                     # MCP 协议支持（OAuth Provider、Token 存储）
├── output/                  # 输出格式化（JSON formatter、流式 JSON）
├── policy/                  # 策略引擎（策略类型、TOML 加载器、策略配置）
├── prompts/                 # Prompt 管理（MCP prompts）
├── resources/               # 资源注册表
├── routing/                 # 路由策略
├── safety/                  # 安全模块
├── sandbox/                 # 沙箱执行（Windows 沙箱管理器等）
├── scheduler/               # 调度器（任务调度、工具执行器、策略调度）
├── services/                # 服务层（文件发现、Git、Shell 执行、沙箱管理、会话等）
├── skills/                  # 技能系统（技能管理器、技能加载器、内置技能）
├── telemetry/               # 遥测与日志上报
├── test-utils/              # 测试工具（mock 配置、mock 工具等）
├── tools/                   # 工具定义（read-file、edit、shell、web-fetch、grep 等）
├── utils/                   # 通用工具函数（文件操作、错误处理、Git、路径、Session 等）
└── voice/                   # 语音响应格式化
```

## 核心组件

### 1. core/ — LLM 交互核心
- **`client.ts`**：核心客户端，编排与 Gemini API 的交互。负责会话管理、对话轮次（Turn）控制、内容压缩、循环检测、重试与回退逻辑。依赖 `GeminiChat`、`ContentGenerator`、`Turn`、`TokenLimits` 等。
- **`geminiChat.ts`**：封装与 Gemini API 的聊天通信。
- **`contentGenerator.ts`**：内容生成器抽象，支持日志记录（`loggingContentGenerator`）和录制（`recordingContentGenerator`）。
- **`prompts.ts`**：系统提示词（System Prompt）的构建与管理。
- **`turn.ts`**：对话轮次模型，定义压缩状态（CompressionStatus）和流事件类型。
- **`tokenLimits.ts`**：Token 额度与限制计算。

### 2. tools/ — 工具定义
提供 Agent 可调用的工具集合，包括文件读写（read-file、write-file、edit）、搜索（grep、glob、ripGrep）、Shell 执行（shell）、网络（web-fetch、web-search）、MCP 工具代理等。

### 3. policy/ — 策略引擎
基于 TOML 配置的策略引擎，用于控制工具执行权限和审批模式（如 yolo、read-only、sandbox-default 等）。

### 4. scheduler/ — 调度器
负责任务调度和工具执行编排，集成策略决策。

### 5. services/ — 服务层
提供文件发现、Git 操作、Shell 执行、沙箱管理、会话记录、上下文管理等基础服务。

### 6. config/ — 配置管理
管理模型配置、默认配置、注入服务、完整性校验和持久化存储。

## 依赖关系

- **外部依赖**：`@google/genai`（Google Gemini SDK）为核心 LLM 通信层。
- **内部依赖链**：`client.ts` -> `geminiChat.ts` -> `contentGenerator.ts`；`client.ts` 同时依赖 `policy/`（策略决策）、`services/`（环境服务）、`hooks/`（生命周期钩子）、`telemetry/`（遥测上报）、`routing/`（路由策略）、`availability/`（模型可用性）等。
- **导出方式**：`index.ts` 通过 `export *` 将所有子模块的公共 API 平铺导出，供上层 CLI 包（如 TUI）使用。
