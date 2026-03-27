# config.ts

## 概述

`config.ts` 是 Gemini CLI 遥测系统的配置解析与合并模块。它负责从三个不同优先级的来源（命令行参数、环境变量、用户设置文件）中解析遥测配置，并将它们合并为最终的 `TelemetrySettings` 对象。该模块体现了典型的"配置层叠"（configuration cascading）设计模式，高优先级来源会覆盖低优先级来源的值。

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph 配置来源（优先级从高到低）
        ARGV[命令行参数<br/>argv / TelemetryArgOverrides]
        ENV[环境变量<br/>env / Record]
        SETTINGS[用户设置文件<br/>settings / TelemetrySettings]
    end

    subgraph 解析函数
        PBF[parseBooleanEnvFlag<br/>解析布尔环境变量]
        PTV[parseTelemetryTargetValue<br/>解析遥测目标值]
        RTS[resolveTelemetrySettings<br/>合并所有配置源]
    end

    subgraph 输出
        TS[TelemetrySettings<br/>最终遥测配置对象]
    end

    ARGV --> RTS
    ENV --> RTS
    SETTINGS --> RTS
    RTS --> PBF
    RTS --> PTV
    RTS --> TS

    subgraph 配置项
        E[enabled<br/>是否启用遥测]
        T[target<br/>遥测目标 local/gcp]
        OE[otlpEndpoint<br/>OTLP 端点地址]
        OP[otlpProtocol<br/>OTLP 协议 grpc/http]
        LP[logPrompts<br/>是否记录提示]
        OF[outfile<br/>输出文件路径]
        UC[useCollector<br/>是否使用收集器]
        UA[useCliAuth<br/>是否使用CLI认证]
    end

    TS --> E
    TS --> T
    TS --> OE
    TS --> OP
    TS --> LP
    TS --> OF
    TS --> UC
    TS --> UA
```

## 核心组件

### 1. `parseBooleanEnvFlag(value: string | undefined): boolean | undefined`

布尔环境变量解析工具函数。

- **输入**：可选的字符串值
- **输出**：`boolean | undefined`
- **逻辑**：
  - 如果值为 `undefined`，返回 `undefined`（表示未设置）
  - 如果值为 `'true'` 或 `'1'`，返回 `true`
  - 其他所有值返回 `false`

### 2. `parseTelemetryTargetValue(value: string | TelemetryTarget | undefined): TelemetryTarget | undefined`

遥测目标值解析函数，将字符串或枚举值规范化为 `TelemetryTarget` 枚举。

- **输入**：字符串、`TelemetryTarget` 枚举值或 `undefined`
- **输出**：`TelemetryTarget | undefined`
- **支持的值**：
  - `TelemetryTarget.LOCAL` 或字符串 `'local'` -> `TelemetryTarget.LOCAL`
  - `TelemetryTarget.GCP` 或字符串 `'gcp'` -> `TelemetryTarget.GCP`
  - 其他值返回 `undefined`

### 3. `TelemetryArgOverrides` 接口

定义命令行参数中可以覆盖的遥测配置项：

```typescript
interface TelemetryArgOverrides {
  telemetry?: boolean;              // 是否启用遥测
  telemetryTarget?: string | TelemetryTarget;  // 遥测目标
  telemetryOtlpEndpoint?: string;   // OTLP 端点
  telemetryOtlpProtocol?: string;   // OTLP 协议
  telemetryLogPrompts?: boolean;    // 是否记录提示
  telemetryOutfile?: string;        // 输出文件
}
```

### 4. `resolveTelemetrySettings(options): Promise<TelemetrySettings>`

核心配置解析函数，按优先级从三个来源解析并合并遥测设置。

**参数**：
- `options.argv`：命令行参数覆盖（最高优先级）
- `options.env`：环境变量映射
- `options.settings`：用户设置文件中的配置（最低优先级）

**返回值**：`Promise<TelemetrySettings>` 对象，包含以下字段：

| 字段 | 来源优先级（argv -> env -> settings） | 环境变量名 |
|------|--------------------------------------|-----------|
| `enabled` | `argv.telemetry` -> `GEMINI_TELEMETRY_ENABLED` -> `settings.enabled` | `GEMINI_TELEMETRY_ENABLED` |
| `target` | `argv.telemetryTarget` -> `GEMINI_TELEMETRY_TARGET` -> `settings.target` | `GEMINI_TELEMETRY_TARGET` |
| `otlpEndpoint` | `argv.telemetryOtlpEndpoint` -> `GEMINI_TELEMETRY_OTLP_ENDPOINT` / `OTEL_EXPORTER_OTLP_ENDPOINT` -> `settings.otlpEndpoint` | `GEMINI_TELEMETRY_OTLP_ENDPOINT` 或 `OTEL_EXPORTER_OTLP_ENDPOINT` |
| `otlpProtocol` | `argv.telemetryOtlpProtocol` -> `GEMINI_TELEMETRY_OTLP_PROTOCOL` -> `settings.otlpProtocol` | `GEMINI_TELEMETRY_OTLP_PROTOCOL` |
| `logPrompts` | `argv.telemetryLogPrompts` -> `GEMINI_TELEMETRY_LOG_PROMPTS` -> `settings.logPrompts` | `GEMINI_TELEMETRY_LOG_PROMPTS` |
| `outfile` | `argv.telemetryOutfile` -> `GEMINI_TELEMETRY_OUTFILE` -> `settings.outfile` | `GEMINI_TELEMETRY_OUTFILE` |
| `useCollector` | 无 argv -> `GEMINI_TELEMETRY_USE_COLLECTOR` -> `settings.useCollector` | `GEMINI_TELEMETRY_USE_COLLECTOR` |
| `useCliAuth` | 无 argv -> `GEMINI_TELEMETRY_USE_CLI_AUTH` -> `settings.useCliAuth` | `GEMINI_TELEMETRY_USE_CLI_AUTH` |

**错误处理**：
- 当 `target` 的原始值已设置但无法解析为有效的 `TelemetryTarget` 时，抛出 `FatalConfigError`，提示有效值为 `local` 或 `gcp`
- 当 `otlpProtocol` 的原始值已设置但不是 `'grpc'` 或 `'http'` 时，抛出 `FatalConfigError`

## 依赖关系

### 内部依赖

| 依赖模块 | 导入内容 | 用途 |
|----------|----------|------|
| `../config/config.js` | `TelemetrySettings`（类型） | 遥测设置的类型定义 |
| `../utils/errors.js` | `FatalConfigError` | 配置验证失败时抛出的致命错误类 |
| `./index.js` | `TelemetryTarget` | 遥测目标枚举（LOCAL / GCP） |

### 外部依赖

无。此文件不依赖任何第三方包。

## 关键实现细节

1. **三层优先级配置合并**：使用 JavaScript 的空值合并运算符（`??`）实现了一个简洁的三层优先级配置合并策略。`argv`（命令行参数）优先级最高，其次是 `env`（环境变量），最后是 `settings`（用户设置文件）。这确保了用户可以通过命令行参数临时覆盖任何配置，同时也支持通过环境变量在 CI/CD 环境中灵活配置。

2. **OTLP 端点的双环境变量支持**：`otlpEndpoint` 字段同时支持 `GEMINI_TELEMETRY_OTLP_ENDPOINT`（Gemini 专有）和标准的 `OTEL_EXPORTER_OTLP_ENDPOINT`（OpenTelemetry 标准）环境变量，其中 Gemini 专有变量优先级更高。这体现了对 OpenTelemetry 生态的兼容性。

3. **严格的输入验证**：对 `target` 和 `otlpProtocol` 字段进行严格验证。如果提供了值但不在有效范围内，直接抛出 `FatalConfigError` 而非静默忽略，遵循了 "fail fast" 原则，防止无效配置导致运行时难以排查的问题。

4. **部分字段缺少 argv 覆盖**：`useCollector` 和 `useCliAuth` 字段没有对应的命令行参数覆盖，只能通过环境变量或设置文件配置。这可能是因为这些选项是高级配置，不需要暴露给普通用户。

5. **异步函数设计**：`resolveTelemetrySettings` 被声明为 `async` 函数，尽管当前实现中没有 `await` 操作。这可能是为了未来扩展预留的（例如可能需要从远程配置服务获取设置），也可能是为了与调用方的异步接口保持一致。

6. **类型安全的协议验证**：OTLP 协议验证使用了 `(['grpc', 'http'] as const).find()` 的模式，这不仅在运行时验证了值，还确保 TypeScript 能正确推断 `otlpProtocol` 的类型为 `'grpc' | 'http' | undefined`。
