# web-fetch.ts

## 概述

`web-fetch.ts` 实现了 Gemini CLI 的 **WebFetch 工具**，允许 LLM 从互联网上获取网页内容并进行处理。该工具支持两种运行模式：

1. **标准模式**（默认）：通过 Gemini API 的 grounding 功能处理 URL，包含引文和来源追踪；当 API 调用失败时自动降级到回退模式。
2. **实验模式**（Direct Web Fetch）：直接通过 HTTP 请求获取 URL 内容，支持多种内容类型（HTML、文本、JSON、图片、视频、PDF）。

该工具具备完善的安全机制（私有 IP 阻止、速率限制）、错误恢复（重试 + 回退）和资源管理（内容大小限制、智能预算分配）。

文件路径：`packages/core/src/tools/web-fetch.ts`

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph WebFetch 工具
        WFT["WebFetchTool<br/>工具声明类"]
        WFI["WebFetchToolInvocation<br/>工具调用类"]
        WFT -->|createInvocation| WFI
    end

    subgraph 执行模式
        WFI -->|execute()| 模式判断{"getDirectWebFetch()"}
        模式判断 -->|true| 实验模式["executeExperimental()<br/>直接 HTTP 抓取"]
        模式判断 -->|false| 标准模式["标准模式<br/>Gemini API Grounding"]
    end

    subgraph 标准模式流程
        标准模式 -->|成功| 引文处理["Grounding 引文处理<br/>+ 来源列表"]
        标准模式 -->|失败| 回退模式["executeFallback()<br/>HTTP 抓取 + LLM 处理"]
    end

    subgraph 实验模式流程
        实验模式 -->|文本/JSON/Markdown| 文本输出["文本截断输出"]
        实验模式 -->|HTML| HTML转换["html-to-text 转换"]
        实验模式 -->|图片/视频/PDF| 二进制输出["Base64 内联数据"]
    end

    subgraph 安全机制
        私有IP检查["isBlockedHost()<br/>私有 IP/localhost 阻止"]
        速率限制["checkRateLimit()<br/>每域名每分钟 10 次"]
        大小限制["readResponseWithLimit()<br/>10MB 响应体限制"]
    end

    WFI --> 私有IP检查
    WFI --> 速率限制
    WFI --> 大小限制

    subgraph 辅助函数
        解析["parsePrompt()<br/>URL 提取与验证"]
        标准化["normalizeUrl()<br/>URL 标准化"]
        GitHub转换["convertGithubUrlToRaw()<br/>GitHub blob→raw"]
        XML清理["sanitizeXml()<br/>XML 特殊字符转义"]
    end

    WFI --> 解析
    WFI --> 标准化
    WFI --> GitHub转换

    subgraph 外部服务
        Gemini["GeminiClient<br/>Gemini API"]
        HTTP["fetch API<br/>HTTP 请求"]
    end

    标准模式 -->|generateContent| Gemini
    回退模式 -->|generateContent| Gemini
    实验模式 --> HTTP
    回退模式 --> HTTP
```

## 核心组件

### 1. 常量定义

| 常量 | 值 | 说明 |
|------|---|------|
| `URL_FETCH_TIMEOUT_MS` | `10000` (10秒) | HTTP 请求超时时间 |
| `MAX_CONTENT_LENGTH` | `250000` (250K字符) | 传给 LLM 的最大内容长度 |
| `MAX_EXPERIMENTAL_FETCH_SIZE` | `10 * 1024 * 1024` (10MB) | HTTP 响应体最大大小 |
| `USER_AGENT` | `Mozilla/5.0 (compatible; Google-Gemini-CLI/1.0; ...)` | 自定义 User-Agent |
| `TRUNCATION_WARNING` | `'\n\n... [Content truncated due to size limit] ...'` | 截断警告文本 |
| `RATE_LIMIT_WINDOW_MS` | `60000` (1分钟) | 速率限制窗口 |
| `MAX_REQUESTS_PER_WINDOW` | `10` | 每域名每窗口最大请求数 |

### 2. 速率限制机制

#### `hostRequestHistory: LRUCache<string, number[]>`
使用 LRU 缓存（容量 1000）存储每个域名的请求时间戳历史。

#### `checkRateLimit(url: string): { allowed, waitTimeMs? }`
- 解析 URL 获取 hostname。
- 清理过期的时间戳（超出窗口期）。
- 如果当前窗口内请求数 >= 10，返回 `allowed: false` 和需要等待的毫秒数。
- 否则记录新时间戳，返回 `allowed: true`。

### 3. 辅助函数

#### `normalizeUrl(urlStr: string): string`
URL 标准化：
- hostname 转小写
- 移除末尾斜杠（根路径 `/` 除外）
- 移除默认端口（HTTP 80、HTTPS 443）

#### `parsePrompt(text: string): { validUrls, errors }`
从提示文本中提取 URL：
- 按空白字符分词。
- 识别包含 `://` 的 token。
- 使用 `new URL()` 验证格式。
- 只允许 `http:` 和 `https:` 协议。
- 返回有效 URL 列表和错误信息列表。

#### `convertGithubUrlToRaw(urlStr: string): string`
将 GitHub blob URL 转换为 raw.githubusercontent.com URL：
- 检测 `github.com` 域名和 `/blob/` 路径段。
- 将域名替换为 `raw.githubusercontent.com`。
- 从路径中移除 `/blob/` 段。

#### `sanitizeXml(text: string): string`
XML 特殊字符转义：`&`, `<`, `>`, `"`, `'`。

### 4. Grounding 元数据接口

用于处理 Gemini API 返回的 grounding（根据）信息：

- **`GroundingChunkWeb`** — 网页来源（URI + 标题）
- **`GroundingChunkItem`** — 包含 `web` 字段的来源项
- **`GroundingSupportSegment`** — 文本中被引用的片段（起止索引）
- **`GroundingSupportItem`** — 引用支持项（片段 + 来源索引列表）

配套类型守卫函数：`isGroundingChunkItem()`, `isGroundingSupportItem()`。

### 5. `WebFetchToolParams` 接口

```typescript
interface WebFetchToolParams {
  prompt?: string;  // 标准模式：包含 URL 和处理指令的提示文本
  url?: string;     // 实验模式：直接要抓取的 URL
}
```

### 6. `WebFetchToolInvocation` 类

继承 `BaseToolInvocation<WebFetchToolParams, ToolResult>`。

#### 构造函数
- 接收 `AgentLoopContext`（提供 config 和 geminiClient 访问）。
- 设置 `respectsAutoEdit = true`（AUTO_EDIT 模式下自动跳过确认）。
- 动态获取审批模式：`() => this.context.config.getApprovalMode()`。

#### `execute(signal: AbortSignal): Promise<ToolResult>`
主入口方法，根据配置选择执行模式：
- `config.getDirectWebFetch()` 为 `true` → `executeExperimental()`
- 否则 → 标准模式执行流程

#### 标准模式执行流程

1. 从 prompt 中解析 URL：`parsePrompt(userPrompt)`。
2. 过滤和验证 URL：`filterAndValidateUrls(validUrls)`
   - 排除私有/本地主机
   - 排除超过速率限制的域名
3. 如果所有 URL 都被跳过，返回错误。
4. 构建 sanitized prompt 并调用 Gemini API：
   ```
   geminiClient.generateContent({ model: 'web-fetch' }, ...)
   ```
5. 处理成功响应：
   - **引文插入**：将 grounding supports 中的引用标记（如 `[1]`、`[2]`）插入到响应文本的对应位置。插入使用从后向前的顺序，避免索引偏移。
   - **来源列表**：将 grounding chunks 格式化为来源列表追加到响应末尾。
   - **跳过警告**：如果有被跳过的 URL，在响应前添加警告。
6. 如果 API 调用失败 → 降级到 `executeFallback()`。

#### `executeFallback(urls, signal): Promise<ToolResult>` — 回退模式

1. 对每个 URL 调用 `executeFallbackForUrl()` 逐个抓取。
2. 如果全部失败，返回聚合错误。
3. **智能预算分配（水填充算法）**：
   - 将成功的内容按大小排序。
   - 从最小的开始，每个 URL 获得 `remainingBudget / remainingUrls` 的公平份额。
   - 较小的内容使用较少预算，剩余预算分配给较大的内容。
   - 总预算为 `MAX_CONTENT_LENGTH`（250K 字符）。
4. 将内容包装在 `<source>` XML 标签中。
5. 调用 Gemini API（model: `'web-fetch-fallback'`）处理聚合内容。

#### `executeFallbackForUrl(urlStr, signal): Promise<string>` — 单 URL 回退抓取

1. 转换 GitHub blob URL。
2. 检查是否为阻止的主机。
3. 使用 `retryWithBackoff` + `fetchWithTimeout` 发起 HTTP 请求。
4. 使用 `readResponseWithLimit` 读取响应（10MB 限制）。
5. 如果是 HTML 内容，使用 `html-to-text` 库转换；否则直接使用原始文本。
6. 截断到 `MAX_CONTENT_LENGTH`。

#### `executeExperimental(signal): Promise<ToolResult>` — 实验模式

1. 验证和解析 URL。
2. 转换 GitHub blob URL。
3. 检查阻止主机。
4. 使用 `retryWithBackoff` + `fetchWithTimeout` 发起请求，Accept 头优先请求 Markdown。
5. 使用 `readResponseWithLimit` 读取响应。
6. 根据 Content-Type 分别处理：
   - **`text/markdown`、`text/plain`、`application/json`** → 截断后直接返回文本。
   - **`text/html`** → 使用 `html-to-text` 转换（保留链接 href，使用 baseUrl）。
   - **`image/*`、`video/*`、`application/pdf`** → Base64 编码后作为 `inlineData` 返回。
   - **其他类型** → 尝试作为文本处理。
7. HTTP 4xx/5xx → 返回包含 headers 和截断响应体的错误信息。

#### `readResponseWithLimit(response, limit): Promise<Buffer>` — 安全响应读取

1. 先检查 `Content-Length` 头（如果有的话）。
2. 使用流式读取（`ReadableStream.getReader()`）。
3. 逐块累加大小，超过限制时取消读取并抛出错误。
4. 最后在 `finally` 中释放 reader lock。

#### `isBlockedHost(urlStr: string): boolean` — 主机安全检查

- 阻止 `localhost` 和 `127.0.0.1`。
- 调用 `isPrivateIp()` 检查是否为私有 IP 地址。
- URL 解析失败时默认阻止。

#### `filterAndValidateUrls(urls): { toFetch, skipped }` — URL 批量过滤

1. 使用 `normalizeUrl` 去重。
2. 检查每个 URL 是否为阻止主机。
3. 检查速率限制。
4. 记录被跳过的 URL 及原因。

#### `getConfirmationDetails()` — 确认详情

构建 `ToolInfoConfirmationDetails`，展示要抓取的 URL 列表和 prompt，供用户确认。

#### `handleRetry(attempt, error, delayMs)` — 重试处理

发射重试事件并记录遥测日志。

### 7. `WebFetchTool` 类

继承 `BaseDeclarativeTool<WebFetchToolParams, ToolResult>`。

#### 属性
- 名称：`WEB_FETCH_TOOL_NAME`
- 显示名：`WEB_FETCH_DISPLAY_NAME`
- Kind：`Kind.Fetch`（只读）
- `isOutputMarkdown`: `true`
- `canUpdateOutput`: `false`

#### `validateToolParamValues(params)` — 参数验证

根据模式进行不同验证：
- **实验模式**：`url` 参数必须存在且为有效 URL。
- **标准模式**：`prompt` 参数不可为空，必须包含至少一个有效 URL，不可包含格式错误的 URL。

#### `getSchema(modelId?)` — 动态 Schema

- **实验模式**：返回简化的 schema，只有 `url` 参数（required），描述强调"直接抓取"。
- **标准模式**：使用 `resolveToolDeclaration` 返回标准 schema。

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 说明 |
|------|---------|------|
| `./tools.js` | `ToolConfirmationOutcome`, `BaseDeclarativeTool`, `BaseToolInvocation`, `Kind`, `ToolCallConfirmationDetails`, `ToolInvocation`, `ToolResult`, `PolicyUpdateOptions` | 工具基类和类型 |
| `../confirmation-bus/message-bus.js` | `MessageBus` (类型) | 消息总线 |
| `./tool-error.js` | `ToolErrorType` | 工具错误类型 |
| `../utils/errors.js` | `getErrorMessage` | 安全的错误信息提取 |
| `../utils/partUtils.js` | `getResponseText` | 从 Gemini 响应中提取文本 |
| `../utils/fetch.js` | `fetchWithTimeout`, `isPrivateIp` | 带超时的 fetch 和私有 IP 检测 |
| `../utils/textUtils.js` | `truncateString` | 字符串截断 |
| `../telemetry/index.js` | `logWebFetchFallbackAttempt`, `WebFetchFallbackAttemptEvent`, `logNetworkRetryAttempt`, `NetworkRetryAttemptEvent` | 遥测事件 |
| `../telemetry/llmRole.js` | `LlmRole` | LLM 角色枚举（`UTILITY_TOOL`） |
| `./tool-names.js` | `WEB_FETCH_TOOL_NAME`, `WEB_FETCH_DISPLAY_NAME` | 工具名称常量 |
| `../utils/debugLogger.js` | `debugLogger` | 调试日志 |
| `../utils/events.js` | `coreEvents` | 核心事件发射器 |
| `../utils/retry.js` | `retryWithBackoff`, `getRetryErrorType` | 重试工具 |
| `./definitions/coreTools.js` | `WEB_FETCH_DEFINITION` | 工具 schema 定义 |
| `./definitions/resolver.js` | `resolveToolDeclaration` | schema 解析器 |
| `../config/agent-loop-context.js` | `AgentLoopContext` (类型) | Agent 循环上下文 |

### 外部依赖

| 包名 | 导入内容 | 说明 |
|------|---------|------|
| `html-to-text` | `convert` | HTML 转纯文本库 |
| `mnemonist` | `LRUCache` | LRU 缓存实现，用于速率限制 |

## 关键实现细节

1. **双模式架构**：工具支持标准模式（通过 Gemini API 的 grounding 功能）和实验模式（直接 HTTP 抓取）。标准模式利用 Google 的 web grounding 能力提供引文和来源追踪，实验模式则更简单直接，支持更多内容类型（包括二进制内容）。

2. **三级降级策略**：
   - 首选：Gemini API grounding（标准模式）
   - 降级：HTTP 抓取 + LLM 处理（回退模式）
   - 最终：返回详细的错误信息

3. **水填充预算分配算法**：当回退模式处理多个 URL 时，使用水填充算法公平分配 250K 字符的总预算。较小的内容获得它所需的空间，剩余预算重新分配给较大的内容。这避免了简单平均分配导致大内容被过度截断的问题。

4. **速率限制的 LRU 策略**：使用 `mnemonist` 的 LRU Cache 存储每域名的请求历史，缓存容量 1000 个域名。每个域名每分钟最多 10 次请求。LRU 策略确保内存使用有界，长时间不访问的域名会被自动淘汰。

5. **安全防护层**：
   - **私有 IP 阻止**：禁止访问 `localhost`、`127.0.0.1` 以及私有 IP 地址，防止 SSRF 攻击。
   - **协议白名单**：只允许 `http:` 和 `https:` 协议。
   - **XML 注入防护**：所有用户输入在嵌入 XML 标签前都经过 `sanitizeXml()` 转义。
   - **响应大小限制**：10MB 的响应体限制，流式读取时实时检查。

6. **GitHub URL 智能转换**：自动将 `github.com/.../blob/...` URL 转换为 `raw.githubusercontent.com/...` 格式，使得 GitHub 上的文件可以直接获取原始内容。

7. **引文处理的逆序插入**：处理 grounding supports 时，将引用标记按位置从后向前插入到响应文本中。这是因为从前向后插入会导致后续插入点的索引偏移。

8. **实验模式的 Accept 头优化**：实验模式的 HTTP 请求使用精心设计的 Accept 头，优先请求 `text/markdown`、然后是 `text/plain` 和 `application/json`，最后才是 `text/html`。这使得支持内容协商的服务器能返回更适合 LLM 处理的格式。

9. **Content-Type 感知处理**：实验模式根据响应的 Content-Type 采取不同处理策略：
   - 文本类型直接返回
   - HTML 使用 `html-to-text` 转换
   - 二进制类型（图片、视频、PDF）转为 Base64 内联数据
   - 未知类型回退到文本处理

10. **`respectsAutoEdit` 快速通道**：WebFetch 工具声明 `respectsAutoEdit = true`，在 AUTO_EDIT 审批模式下自动跳过用户确认，因为网页抓取是只读操作。

11. **遥测集成**：工具在关键路径上记录遥测事件：
    - `WebFetchFallbackAttemptEvent` — 记录回退触发原因（`primary_failed`、`private_ip_skipped`）
    - `NetworkRetryAttemptEvent` — 记录网络重试详情

12. **动态 Schema 切换**：`getSchema()` 根据 `getDirectWebFetch()` 配置动态返回不同的 schema。实验模式使用简化的 `url` 参数 schema，标准模式使用完整的 `prompt` 参数 schema。
