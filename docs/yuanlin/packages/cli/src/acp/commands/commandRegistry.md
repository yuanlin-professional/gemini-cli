# commandRegistry.ts

## 概述

`commandRegistry.ts` 是 Gemini CLI ACP 命令系统中的**命令注册表**实现文件。它提供了一个简洁的 `CommandRegistry` 类，用于集中管理所有 ACP 斜杠命令的注册、查询和遍历。

`CommandRegistry` 是一个基于 `Map<string, Command>` 的轻量级注册表容器，支持：
- 命令注册（含重复检测和子命令递归注册）
- 按名称查找命令
- 获取所有已注册命令列表

该类是 `CommandHandler` 的核心内部组件，在 `CommandHandler` 构造时被创建并填充命令。

## 架构图（Mermaid）

```mermaid
graph TB
    subgraph 命令处理器
        CH[CommandHandler<br/>命令处理器]
    end

    subgraph 命令注册表
        CR[CommandRegistry]
        commands[commands: Map&lt;string, Command&gt;<br/>命令存储]
        register[register<br/>注册命令]
        get[get<br/>按名称查询]
        getAllCommands[getAllCommands<br/>获取所有命令]
    end

    subgraph 已注册命令（示例）
        MemoryCmd[MemoryCommand<br/>name: memory]
        ExtCmd[ExtensionsCommand<br/>name: extensions]
        InitCmd[InitCommand<br/>name: init]
        RestoreCmd[RestoreCommand<br/>name: restore]
    end

    subgraph 子命令递归注册
        SubCmd1[子命令 A]
        SubCmd2[子命令 B]
    end

    subgraph 日志
        debugLogger[debugLogger<br/>调试日志]
    end

    CH --> CR
    CR --> commands
    CR --> register
    CR --> get
    CR --> getAllCommands

    register --> MemoryCmd
    register --> ExtCmd
    register --> InitCmd
    register --> RestoreCmd

    MemoryCmd -.->|subCommands| SubCmd1
    MemoryCmd -.->|subCommands| SubCmd2
    register -.->|递归注册子命令| SubCmd1
    register -.->|递归注册子命令| SubCmd2

    register -.->|重复注册警告| debugLogger
```

## 核心组件

### `CommandRegistry` 类

#### 私有属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `commands` | `Map<string, Command>`（`readonly`） | 命令名称到命令实例的映射表 |

#### `register(command: Command): void`

注册一个命令到注册表中。

**处理流程：**
1. **重复检测**：检查命令名称是否已存在于 `commands` 映射中
   - 如果已存在，通过 `debugLogger.warn` 输出警告日志并跳过注册
   - 如果不存在，将命令以 `name -> Command` 的形式存入 Map
2. **递归注册子命令**：遍历命令的 `subCommands` 数组（如果存在），对每个子命令递归调用 `register`

**重要特性 -- 扁平化注册：**

子命令会被递归地注册到**同一个** `commands` Map 中，与父命令处于同一层级。这意味着：
- `MemoryCommand`（name: "memory"）和其子命令（如 name: "add"）都在同一个 Map 中
- 可以直接通过 `get("add")` 查找到子命令，而不需要先找到父命令再遍历子命令
- 但这也意味着子命令的 name 必须全局唯一，否则会触发重复注册警告

#### `get(commandName: string): Command | undefined`

按命令名称查找已注册的命令。

**参数：**
- `commandName` -- 要查找的命令名称

**返回值：** 对应的 `Command` 实例，或 `undefined`（未找到）

#### `getAllCommands(): Command[]`

返回所有已注册命令的数组副本。

**注意：** 由于子命令也被扁平注册，返回的数组中会包含父命令和子命令。在 `CommandHandler.parseSlashCommand` 中，通过检查命令的 `subCommands` 属性来实现层级遍历，而不是依赖注册表的层级结构。

## 依赖关系

### 内部依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `@google/gemini-cli-core` | `debugLogger` | 调试日志输出（用于重复注册警告） |
| `./types.js` | `Command`（类型） | 命令接口类型定义 |

### 外部依赖

无外部第三方依赖。

## 关键实现细节

### 1. 基于 Map 的存储

使用原生 `Map<string, Command>` 作为存储容器，提供 O(1) 的按名称查找性能。`Map` 还保证了迭代顺序与插入顺序一致，这意味着 `getAllCommands()` 返回的命令顺序与注册顺序相同。

### 2. 只读保护

`commands` 属性使用 `readonly` 修饰符声明，防止在类内部意外替换整个 Map 实例（但 Map 内容仍可修改）。

### 3. 幂等注册

`register` 方法对重复注册进行了检测并跳过，而非抛出异常。这是一种宽容的设计策略：
- 不会因为意外的重复注册导致应用崩溃
- 通过日志警告帮助开发者发现问题
- 先注册的命令优先生效

### 4. 子命令扁平化策略

子命令通过递归调用 `register` 被扁平化注册到同一个 Map 中。这种设计简化了命令查找逻辑，但也带来了命名冲突的风险。在实际使用中，`CommandHandler.parseSlashCommand` 通过层级遍历（检查 `subCommands` 属性）来正确解析多级命令，而非依赖注册表的扁平结构。

### 5. 空值安全

`register` 方法中使用 `command.subCommands ?? []` 处理 `subCommands` 可能为 `undefined` 的情况，确保在没有子命令时不会抛出错误。

### 6. 与 CommandHandler 的协作

`CommandRegistry` 是 `CommandHandler` 的内部组件，其生命周期完全由 `CommandHandler` 管理：
- 在 `CommandHandler.createRegistry()` 静态方法中创建并填充
- `CommandHandler.getAvailableCommands()` 调用 `getAllCommands()` 获取命令列表
- `CommandHandler.parseSlashCommand()` 调用 `getAllCommands()` 获取顶层命令列表进行匹配
