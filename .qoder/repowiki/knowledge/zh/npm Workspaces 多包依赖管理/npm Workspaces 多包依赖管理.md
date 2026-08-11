---
kind: dependency_management
name: npm Workspaces 多包依赖管理
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - packages/core/package.json
    - packages/otel/package.json
    - packages/create-oma-app/package.json
    - package-lock.json
    - .nvmrc
---

## 1. 使用的系统/方法

本仓库采用 **npm workspaces** 作为多包（monorepo）依赖管理的核心机制。根 `package.json` 通过 `workspaces: ["packages/*"]` 声明三个子包：`@open-multi-agent/core`、`@open-multi-agent/otel`、`create-oma-app`，所有依赖解析与安装由 npm 在 workspace 内统一处理，无需 pnpm/yarn。

Node.js 版本通过 `.nvmrc` 锁定为 `22`，各包的 `engines.node` 字段要求 `>=20.0.0`，形成“最低兼容 + 开发固定”的双层约束。

## 2. 关键文件

- `package.json`（根）：workspace 聚合、顶层脚本（`build`、`test`、`lint` 等通过 `--workspaces --if-present` 分发到子包）、`engines.node >=20.0.0`。
- `packages/core/package.json`：核心库 `@open-multi-agent/core`，声明运行时依赖（`@anthropic-ai/sdk`、`openai`、`zod`）、可选的 peerDependencies（AWS Bedrock、Google GenAI、MCP SDK、`ai`），以及 devDependencies 中的测试/类型工具。
- `packages/otel/package.json`：可选 OpenTelemetry 适配器，依赖 `@open-multi-agent/core ^1.11.0`，peerDependency 为 `@opentelemetry/api ^1.9.0`。
- `packages/create-oma-app/package.json`：脚手架 CLI，仅含 devDependencies，无运行时依赖。
- `package-lock.json`：npm 锁文件，锁定 workspace 内所有包的精确版本。
- `.nvmrc`：固定开发 Node 版本为 22。

## 3. 架构与约定

### 包间依赖关系
- `@open-multi-agent/otel` 通过 workspace 协议引用 `@open-multi-agent/core`（`^1.11.0`），体现“核心库 + 可选扩展”的分层设计。
- `create-oma-app` 是独立 CLI，不依赖 core，仅用于生成项目模板。

### 运行时 vs 可选依赖
- `core` 将 LLM 厂商 SDK（Bedrock、Google GenAI、MCP SDK、`ai`）声明为 **peerDependencies** 并标记 `optional: true`，使核心包保持最小体积，使用者按需安装对应 provider。
- 运行时必需依赖（Anthropic SDK、OpenAI SDK、Zod）放在 `dependencies` 中。

### 发布配置
- 每个包均设置 `publishConfig.access: "public"`，表明这些包会发布到 npm。
- 每个包都定义 `prepublishOnly: "npm run build"`，确保发布前执行 TypeScript 编译。
- 通过 `files` 字段精确控制发布产物（仅 `dist`、`README.md`、`LICENSE` 等）。

### 构建与测试脚本
- 根脚本使用 `npm run <cmd> --workspaces --if-present` 模式，实现跨包统一入口。
- 示例校验脚本（`scripts/example-catalog.test.mjs`、`scripts/validate-example-catalog.mjs`）在 `pretest` 钩子中运行，保证示例目录清单一致性。

## 4. 约定与约束

- **版本范围策略**：外部依赖普遍使用 `^` 语义化版本范围（如 `zod ^3.23.0`、`typescript ^5.6.0`、`vitest ^2.1.0`），允许小版本升级；包间依赖（如 otel → core）也使用 `^` 范围而非精确锁定。
- **Node 版本约束**：根与所有包均声明 `engines.node >=20.0.0`，`.nvmrc` 固定开发环境为 22，CI 应遵循此约束。
- **ESM 优先**：所有包设置 `"type": "module"`，输出 ESM 格式（`main` 指向 `dist/index.js`，`exports` 提供条件导出）。
- **无 vendoring**：未发现 `vendor/` 或第三方源码内联，全部依赖通过 npm registry 获取并由 `package-lock.json` 锁定。
- **无私有注册表配置**：未检出 `.npmrc`、`pnpm-workspace.yaml` 或自定义 registry 配置，默认使用 npm 官方源。
- **peerDependencies 作为插件点**：通过 `peerDependenciesMeta.optional` 将可选功能（MCP、Bedrock、Google GenAI、`ai`）解耦，使用者自行决定安装哪些 provider。
- **工作区脚本约定**：所有子包需暴露标准脚本名（`build`、`test`、`lint`、`prepublishOnly`），以便根脚本通过 `--if-present` 安全调用。