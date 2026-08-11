---
kind: configuration_system
name: 配置系统：环境变量 + JSON 配置文件 + CLI 参数三层加载
category: configuration_system
scope:
    - '**'
source_files:
    - packages/core/src/cli/oma.ts
    - packages/core/src/types.ts
    - packages/create-oma-app/template/src/runtime.ts
    - packages/core/examples/cookbook/adaptive-customer-support.ts
    - packages/core/examples/basics/structured-input.ts
    - packages/create-oma-app/templates/demo/src/index.ts
---

## 1. 整体方案

本仓库没有引入 dotenv、config 等第三方配置库，而是采用 **纯 Node.js `process.env` + JSON 文件 + CLI argv** 的三层组合方式完成运行时配置加载与合并。核心思路是：
- 应用/示例代码通过直接读取 `process.env` 获取密钥、模型名、baseURL 等运行期开关；
- CLI（`oma`）通过解析命令行参数并读取 JSON 文件（`team.json`、`orchestrator.json`、`coordinator.json`、`tasks.json`、评估集/评分器/gate 策略等）作为声明式配置源；
- `create-oma-app` 模板提供一个轻量 `loadEnv()` 函数，从 `.env` 文件按行解析键值对后写入 `process.env`，供模板应用启动时调用。

该设计刻意保持零依赖：`@open-multi-agent/core` 的运行时依赖只有 `zod`，不包含任何配置框架。

## 2. 关键文件与位置

| 职责 | 文件 | 说明 |
|---|---|---|
| CLI 入口与 JSON 配置加载 | `packages/core/src/cli/oma.ts` | 实现 `oma run / task / eval / dashboard / provider` 子命令，解析 argv、读取 JSON、校验 TeamConfig/OrchestratorConfig/CoordinatorConfig/TaskSpecs |
| 核心类型定义（含配置接口） | `packages/core/src/types.ts` | 定义 `AgentConfig`、`OrchestratorConfig`、`CoordinatorConfig`、`TeamConfig`、`ToolUseContext.credentials` 等所有可配置结构 |
| 模板应用环境加载 | `packages/create-oma-app/template/src/runtime.ts` | 提供 `loadEnv(path)` 和 `resolveRuntime()`，从 `.env` 注入 `process.env` 并推断 Ollama / OpenAI 兼容后端 |
| 示例中的 env 使用模式 | `packages/core/examples/**/*.ts` | 多处示例通过 `process.env.OMA_MODEL`、`process.env.OPENAI_BASE_URL`、`process.env.OMA_PROVIDER`、`process.env.TICKET_SCENARIO`、`process.env.FORCE_FAIL` 等控制行为 |
| 提供者密钥约定表 | `packages/core/src/cli/oma.ts` 中 `PROVIDER_REFERENCE` | 集中列出每个 provider 对应的 `apiKeyEnv` 数组，如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GEMINI_API_KEY`、`AWS_ACCESS_KEY_ID` 等 |

## 3. 架构与加载顺序

### 3.1 CLI 层（`oma` 命令）
CLI 的 `main()` 先调用 `parseArgs(process.argv)` 将 `--key=value` 和 `--flag` 解析为 `{ _, flags, kv }`，再根据子命令分支：
- `oma run --goal ... --team <team.json> [--orchestrator <orch.json>] [--coordinator <coord.json>]`：依次读取 team.json → 可选 orchestrator.json → 可选 coordinator.json，并通过 `mergeOrchestrator({}, ...orchParts)` 浅合并多个 OrchestratorConfig 片段。
- `oma task --file <tasks.json> [--team <team.json>]`：读取 tasks.json，支持内嵌 `team`、`orchestrator` 字段，也可用 `--team` 覆盖。
- `oma eval run --set <evalset.json> --target <target.mjs> [--scorers <scorers.mjs>] [--gate <gate.json>] [--baseline <report.json>]`：全部通过路径参数传入 JSON/JS 模块。
- `oma dashboard --trace-store <traces.ndjson> --run-id <id>`：读取 FileTraceStore 输出。

JSON 配置在 CLI 中经过严格的 `asTeamConfig` / `asOrchestratorPartial` / `asCoordinatorPartial` / `asTaskSpecs` 校验，例如：
- `team.json` 必须包含非空 `name` 和至少一个 agent；agent 必须含 `name` 与 `model`；
- 显式拒绝 `sharedMemoryStore` 字段（SDK-only 对象无法 JSON 序列化），要求改用 `sharedMemory: true` 或 TS 中构造 MemoryStore；
- tasks 数组每项必须含 `title`、`description`，且 `metadata` 会经 `validateTaskMetadata` 校验。

### 3.2 运行时 env 层
示例与应用代码直接读取 `process.env`，常见约定：
- `OMA_MODEL`：默认模型名（如 `claude-sonnet-4-6`、`gpt-5.4-mini`），被多处示例用作 `??` 回退；
- `OMA_PROVIDER`：切换 provider（如 `copilot` vs `openai`）；
- `OPENAI_BASE_URL`：OpenAI 兼容后端地址（Ollama、vLLM、LM Studio、DeepSeek 等）；
- `TICKET_SCENARIO`、`FORCE_FAIL`、`EXAMPLE_PROVIDER`：示例/测试场景开关；
- 各 provider 自有密钥变量：`ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MINIMAX_API_KEY`、`MIMO_API_KEY`、`DEEPSEEK_API_KEY`、`ARK_API_KEY`、`HUNYUAN_API_KEY`、`QINIU_API_KEY`、`GITHUB_COPILOT_TOKEN`、`GITHUB_TOKEN`、`AZURE_OPENAI_API_KEY`、`AZURE_OPENAI_ENDPOINT`、`AZURE_OPENAI_DEPLOYMENT`、`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_REGION`。

这些约定由 `PROVIDER_REFERENCE` 集中声明，并被 `oma provider template <provider>` 用于生成带占位符的 `env` 块。

### 3.3 模板应用的 `.env` 加载
`create-oma-app` 模板的 `runtime.ts` 提供 `loadEnv(path = '.env')`：逐行读取文件，跳过注释与空行，按 `=` 分割 key/value，去除首尾引号，仅当 `process.env[key]` 未设置时才写入——因此外部已设置的 `process.env` 优先于 `.env`。`resolveRuntime()` 会先调用 `loadEnv()`，然后根据 `OMA_RUNTIME === 'ollama'` 走 Ollama 探测路径，否则要求 `OPENAI_API_KEY` 存在并返回 cloud 配置。

## 4. 约定与约束

### 4.1 环境变量命名约定（描述性）
- 以 `OMA_` 前缀表示框架级开关：`OMA_MODEL`、`OMA_PROVIDER`、`OMA_RUNTIME`；
- 以 provider 标准名称暴露密钥：`OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、`GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MINIMAX_API_KEY`、`MIMO_API_KEY`、`DEEPSEEK_API_KEY`、`ARK_API_KEY`、`HUNYUAN_API_KEY`、`QINIU_API_KEY`、`GITHUB_COPILOT_TOKEN`、`GITHUB_TOKEN`、`AZURE_OPENAI_*`、`AWS_*`；
- 以 `*_BASE_URL` 后缀指定自定义 endpoint：`OPENAI_BASE_URL`、`MIMO_BASE_URL`、`HUNYUAN_BASE_URL`、`OLLAMA_HOST`。

### 4.2 强制约束（由代码验证）
- `team.json` 中不允许出现 `sharedMemoryStore` 字段，CLI 会抛出 `OmaValidationError` 并提示改用 SDK 路径（见 `asTeamConfig`）。
- `tasks.json` 的 `tasks[]` 每一项必须包含字符串类型的 `title` 与 `description`，否则抛错。
- `--meta key=value` 形式的元数据必须使用 `=` 分隔，否则会报 usage 错误。
- `--report` 重复选项只接受 `json`、`markdown`、`junit` 三者之一。
- `--repeats`、`--concurrency` 必须为正整数，否则抛 `OmaValidationError`。
- `--baseline` 必须搭配 `--gate` 使用，否则报错。
- `dashboard` 子命令要求同时提供 `--trace-store` 与 `--run-id`。
- `writeDashboardFile` 使用 `fs.writeFile(..., { flag: 'wx' })` 防止覆盖已存在的输出文件，若目标已存在则抛出 `dashboard_output_exists` 错误。
- `FileTraceStore.open` 失败时抛出 `trace_store_not_found` 错误。
- `classifyCliError` 将 `ENOENT` / `EACCES` 归类为 `io` 退出码 2，`Invalid JSON` 归类为 `validation`，其余异常归类为 `runtime` 退出码 3。

### 4.3 安全相关约定
- `ToolUseContext.credentials` 被文档明确标记为“secrets”，会被自动从 traces 和 dashboards 中脱敏（引用 `utils/redaction.ts`）；
- 每个 agent 的 `credentials` 互不继承、不被 coordinator 合并，遵循最小权限原则；
- 工具代码仍可读取 `process.env`，但 `credentials` 是推荐的 per-agent 密钥注入通道。

### 4.4 配置来源优先级
在 CLI 路径中，配置合并顺序为：
1. 空的 `OrchestratorConfig` 基线；
2. `team.json` 中内嵌的 `orchestrator`（若有）；
3. 通过 `--orchestrator` 指定的额外 JSON 文件；
4. 最终传给 `new OpenMultiAgent(mergedConfig)`。

对于模板应用，`.env` 仅在 `process.env[key]` 不存在时写入，因此进程级环境变量始终优先。

## 5. 总结

该仓库的配置系统以 **极简、无依赖** 为核心设计：不引入 dotenv/config 等中间件，而是让 CLI 负责 JSON 声明式配置的强校验与合并，让示例/应用代码直接消费 `process.env`，并由模板提供一份可复用的 `.env` 加载器。所有 provider 密钥名集中在 `PROVIDER_REFERENCE` 中声明，形成单一事实来源；CLI 对所有 JSON 配置进行 Zod 之外的手工校验，确保 `oma` 命令在 CI 中以确定性的退出码（0/1/2/3）表达成功、业务失败、用法错误与内部错误。