# fetch.ts

## 概述

`fetch.ts` 是 Gemini CLI 核心包中的网络请求工具模块，提供了安全增强的 HTTP 请求能力。该模块的核心关注点包括：

1. **SSRF（服务端请求伪造）防护**：通过一系列 IP 地址检测函数，阻止对私有网络、保留地址段的请求，防止 SSRF 攻击。
2. **超时控制**：提供带超时机制的 `fetchWithTimeout` 函数，以及通过 `undici` 库配置全局默认超时。
3. **代理支持**：支持全局 HTTP 代理配置和安全代理代理（ProxyAgent）的创建。
4. **自定义错误类型**：定义了 `FetchError` 和 `PrivateIpError` 两个专用错误类，便于上层代码进行精确的错误处理。

## 架构图（Mermaid）

```mermaid
flowchart TD
    subgraph 安全检测层
        A[URL / 主机名] --> B[sanitizeHostname<br/>清理IPv6括号]
        B --> C{isLoopbackHost?<br/>是否回环地址}
        C -->|是| D[允许访问<br/>localhost/127.0.0.1/::1]
        C -->|否| E[isAddressPrivate<br/>检查是否私有IP]
        E --> F{是否合法IP?}
        F -->|非IP格式| G[允许访问]
        F -->|是IP| H{IPv4-mapped IPv6?}
        H -->|是| I[解包为IPv4后递归检查]
        H -->|否| J{IANA Benchmark Range?<br/>198.18.0.0/15}
        J -->|是| K[阻止访问]
        J -->|否| L{addr.range()}
        L -->|unicast| M[允许访问]
        L -->|其他| N[阻止访问<br/>私有/保留/链路本地等]
    end

    subgraph 异步DNS解析检测
        O[isPrivateIpAsync] --> P[DNS lookup<br/>解析所有地址]
        P --> Q[逐一检查 isAddressPrivate]
    end

    subgraph 请求发送层
        R[fetchWithTimeout] --> S[创建 AbortController]
        S --> T[设置超时定时器]
        T --> U[调用 fetch]
        U --> V{成功?}
        V -->|是| W[返回 Response]
        V -->|超时| X[抛出 FetchError<br/>ETIMEDOUT]
        V -->|其他错误| Y[抛出 FetchError]
    end

    subgraph 全局配置层
        Z1[setGlobalProxy] --> Z2[setGlobalDispatcher<br/>ProxyAgent]
        Z3[模块初始化] --> Z4[setGlobalDispatcher<br/>Agent 默认超时]
    end

    style K fill:#F44336,color:#fff
    style N fill:#F44336,color:#fff
    style D fill:#4CAF50,color:#fff
    style G fill:#4CAF50,color:#fff
    style M fill:#4CAF50,color:#fff
```

## 核心组件

### 1. 错误类

#### `FetchError`

继承自 `Error`，用于包装所有网络请求相关错误。

| 属性 | 类型 | 说明 |
|---|---|---|
| `message` | `string` | 错误描述 |
| `code` | `string \| undefined` | 错误码（如 `'ETIMEDOUT'`） |
| `name` | `string` | 固定值 `'FetchError'` |

#### `PrivateIpError`

继承自 `Error`，用于表示访问私有网络被阻止。

| 属性 | 类型 | 说明 |
|---|---|---|
| `message` | `string` | 默认 `'Access to private network is blocked'` |
| `name` | `string` | 固定值 `'PrivateIpError'` |

### 2. 全局配置常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_HEADERS_TIMEOUT` | `300000`（5 分钟） | HTTP 响应头接收超时 |
| `DEFAULT_BODY_TIMEOUT` | `300000`（5 分钟） | HTTP 响应体接收超时 |
| `IANA_BENCHMARK_RANGE` | `198.18.0.0/15` | IANA 基准测试保留地址段 |

### 3. IP 地址安全检测函数

#### `sanitizeHostname(hostname: string): string`
清理主机名，去除 IPv6 地址的方括号包装（如 `[::1]` -> `::1`）。

#### `isLoopbackHost(hostname: string): boolean`
检查主机名是否为回环地址（`localhost`、`127.0.0.1`、`::1`）。回环地址在某些场景下（如本地开发/测试连接本地 MCP 服务器）被允许访问。

#### `isPrivateIp(url: string): boolean`
同步检查 URL 的主机名是否为私有 IP 地址（仅基于字面量，不进行 DNS 解析）。

#### `isAddressPrivate(address: string): boolean`
核心私有 IP 检测函数，判断逻辑如下：
1. `localhost` 直接判定为私有
2. 非有效 IP 格式，判定为非私有（允许域名通过）
3. IPv4-mapped IPv6 地址（`::ffff:x.x.x.x`）解包后递归检查
4. 处于 IANA 基准测试范围（`198.18.0.0/15`）的地址判定为私有
5. 使用 `ipaddr.js` 的 `range()` 方法判断：只有 `'unicast'` 范围被视为公网可达
6. 解析失败时保守地判定为私有（安全优先）

#### `isPrivateIpAsync(url: string): Promise<boolean>`
异步版本的私有 IP 检测。与同步版本的关键区别：
- 先通过 DNS 解析获取目标主机名的所有 IP 地址
- 逐一检查每个解析结果是否为私有 IP
- 允许回环地址通过（用于本地开发/测试）
- `TypeError` 异常（通常是无效 URL）被视为非私有

### 4. 网络请求函数

#### `fetchWithTimeout(url, timeout, options?): Promise<Response>`
带超时控制的 HTTP 请求函数。

特性：
- 使用 `AbortController` 实现超时中断
- 支持外部传入 `AbortSignal`：如果外部信号已中止，立即中止请求；否则监听外部信号的中止事件
- 超时错误被包装为 `FetchError`，错误码为 `'ETIMEDOUT'`
- 其他错误也被包装为 `FetchError`，保留原始错误的 `cause` 链

#### `createSafeProxyAgent(proxyUrl: string): ProxyAgent`
创建一个 `undici` 的 `ProxyAgent` 实例。

#### `setGlobalProxy(proxy: string): void`
设置全局 HTTP 代理，通过 `undici` 的 `setGlobalDispatcher` 将所有后续的 `fetch` 请求通过指定代理发送。同时配置默认的 headers 和 body 超时。

### 5. 模块初始化

模块加载时立即执行 `setGlobalDispatcher`，配置默认的 `Agent` 全局派发器，设置 5 分钟的 headers 和 body 超时。这确保了所有使用全局 `fetch` 的请求都有合理的超时设置，避免请求无限挂起。

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 用途 |
|---|---|---|
| `./errors.js` | `getErrorMessage`, `isNodeError` | 错误消息提取和 Node.js 错误类型判断 |

### 外部依赖

| 模块 | 导入内容 | 用途 |
|---|---|---|
| `node:url` | `URL` | URL 解析 |
| `node:dns/promises` | `lookup` | 异步 DNS 解析 |
| `undici` | `Agent`, `ProxyAgent`, `setGlobalDispatcher` | 高性能 HTTP 客户端库，提供代理和全局派发器支持 |
| `ipaddr.js` | `ipaddr` | IP 地址解析、范围判断、CIDR 匹配 |

## 关键实现细节

1. **SSRF 防护的多层设计**：
   - **同步层**（`isPrivateIp` / `isAddressPrivate`）：基于 URL 字面量进行快速检查，适用于已知 IP 地址的场景
   - **异步层**（`isPrivateIpAsync`）：通过 DNS 解析获取实际 IP 后检查，能防止 DNS rebinding 等高级攻击手段
   - **地址范围全覆盖**：利用 `ipaddr.js` 的 `range()` 方法，自动识别 `private`、`loopback`、`linkLocal`、`multicast`、`carrierGradeNat` 等所有非 `unicast` 范围

2. **IPv4-mapped IPv6 的处理**：一些系统会将 IPv4 地址表示为 IPv6 格式（如 `::ffff:192.168.1.1`）。模块通过 `isIPv4MappedAddress()` 检测并使用 `toIPv4Address()` 解包，然后递归检查底层的 IPv4 地址，防止通过 IPv6 包装绕过安全检查。

3. **IANA 基准测试地址段的特殊处理**：`198.18.0.0/15` 范围在 `ipaddr.js` 中被分类为 `'unicast'`（会被误判为公网地址），但实际上是 IANA 保留的基准测试范围，不应作为公网目标访问。模块通过额外的 `isBenchmarkAddress()` 检查来弥补这个库的分类不足。

4. **回环地址的差异化处理**：
   - 在 `isAddressPrivate` 中，`localhost` 被判定为私有（它确实是私有的）
   - 但在 `isPrivateIpAsync` 中，回环地址被显式允许通过。这是因为异步检查通常用于实际网络请求前的守卫，而本地开发/测试场景需要能够连接 `localhost` 上运行的服务

5. **AbortSignal 的组合处理**：`fetchWithTimeout` 同时处理内部超时信号和外部传入的 `AbortSignal`。通过在外部信号上注册 `abort` 事件监听器（`{ once: true }`），实现了两个信号的"或"逻辑——任一信号触发都会中止请求。

6. **保守的安全策略**：在 `isAddressPrivate` 中，如果 IP 解析失败（`catch` 块），默认返回 `true`（视为私有），这是"安全优先"的设计理念——宁可误阻也不要漏放。

7. **全局 Dispatcher 的运行时修改**：`setGlobalProxy` 通过调用 `setGlobalDispatcher` 改变全局 HTTP 行为。这意味着调用后所有使用全局 `fetch` 的代码都会受到影响，是一个全局副作用操作。
