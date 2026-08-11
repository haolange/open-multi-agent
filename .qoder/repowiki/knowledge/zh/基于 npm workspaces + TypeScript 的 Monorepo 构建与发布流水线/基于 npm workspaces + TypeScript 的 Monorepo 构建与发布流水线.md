---
kind: build_system
name: 基于 npm workspaces + TypeScript 的 Monorepo 构建与发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - .github/workflows/ci.yml
    - .github/workflows/release-smoke.yml
    - packages/core/package.json
    - packages/core/tsconfig.json
    - packages/create-oma-app/package.json
    - packages/otel/package.json
    - scripts/observability-tarball-smoke.mjs
---

## 1. 使用的系统与工具

- **包管理**：npm workspaces（根 `package.json` 中声明 `workspaces: ["packages/*"]`），通过 `npm ci` / `npm run build --workspaces --if-present` 统一编排。
- **编译**：每个子包独立使用 TypeScript（`tsc`，目标 ES2022、模块 ESNext、输出到 `dist/`，开启 declaration/sourceMap），无额外打包器（如 esbuild/vite）。`prepublishOnly` 钩子保证发布前自动构建。
- **测试**：Vitest（`vitest run` / `vitest` watch / `--coverage`），覆盖率由 `@vitest/coverage-v8` 生成，CI 上传至 Codecov。
- **CI**：GitHub Actions（`.github/workflows/ci.yml`、`release-smoke.yml`、`provider-canary.yml`），在 push/pull_request to `main` 上触发 lint、多 Node 版本矩阵测试、覆盖率上报、包产物校验、示例冒烟测试、scaffold e2e。
- **发布后验证**：`release-smoke.yml` 在 GitHub Release published 事件后，从真实 npm registry 拉取 `create-oma-app@latest` 和 `@open-multi-agent/core`，运行 no-key demo 并检查 Markdown/JSON/HTML 报告；同时验证 `@open-multi-agent/otel` 与已发布 core 的配对兼容性。

## 2. 关键文件

- 根编排：`package.json`（workspaces、顶层 scripts）、`.github/workflows/ci.yml`、`.github/workflows/release-smoke.yml`
- 子包构建配置：`packages/core/package.json`、`packages/core/tsconfig.json`、`packages/create-oma-app/package.json`、`packages/otel/package.json`
- 包产物断言脚本：`scripts/observability-tarball-smoke.mjs`（构造临时消费者，验证 tarball 内容、exports 可导入、不污染依赖图）
- 示例/评估冒烟入口：`scripts/example-catalog.test.mjs`、`scripts/validate-example-catalog.mjs`、`scripts/eval-example-smoke.mjs`、`scripts/observability-example-smoke.mjs`

## 3. 架构与约定

### 包结构与导出契约
- 每个子包将源码放在 `src/`，编译产物输出到 `dist/`，并通过 `files` 字段精确声明要随 npm 包发布的文件。
- `core` 通过 `exports` 字段暴露多个子路径入口（`.`、`./observability`、`./observability/file`、`./acp`、`./process`、`./mcp`、`./ai-sdk`、`./classifiers`、`./eval`、`./eval/file`），并在 CI 中以 `npm pack --dry-run` 白名单校验这些 `.js` / `.d.ts` 必须出现在 tarball 中。
- `core` 还通过 `bin` 提供 CLI `oma`（`dist/cli/oma.js`），CI 会执行 `node dist/cli/oma.js help` 并断言包含 `oma eval run` 与 `oma eval gate` 命令。
- `create-oma-app` 暴露 `dist/index.js` 作为脚手架入口，并通过 `template/`、`templates/` 目录分发模板；CI 校验其 tarball 必须包含 `dist/`、`template/`、`templates/` 及若干样板文件。
- `otel` 仅暴露 `dist/index.js`，并以 `peerDependencies` 声明 `@opentelemetry/api`，以 `dependencies` 声明对 `@open-multi-agent/core` 的 caret 范围（CI 强制该范围为 `^X.Y.Z` 形式）。

### 版本与依赖约束
- 根 `engines.node >= 20.0.0`，CI 使用 Node 20/22/24 矩阵测试，lint/coverage/package 固定使用 Node 22。
- `otel` 的 `@open-multi-agent/core` 依赖必须为精确的 caret 范围（CI 正则校验 `^[0-9]+\.[0-9]+\.[0-9]+$`），用于“最低兼容 core 版本”测试——CI 会 `npm pack` 该最小版本并安装进干净消费者。
- `create-oma-app` 的模板中的 `@open-multi-agent/core` 版本号必须与当前仓库 `packages/core/package.json` 的版本完全一致（CI 逐一对比 `templates/*/package.json` 中的 pinned 版本）。

### 构建产物质量门禁
- **tarball 内容白名单**：`core` 只允许 `dist/`、`README.md`、`LICENSE`、`package.json`；`create-oma-app` 额外允许 `template/`、`templates/`；`otel` 同 core。任何不在白名单的文件都会导致 CI 失败。
- **入口可导入性**：CI 在 `packages/core` 下直接 `import('./dist/...')` 所有导出路径；`packages/otel` 同样校验 `dist/index.js` 可 import。
- **零网络/零 API Key 冒烟**：`test:eval-examples`、`test:observability-examples`、`bench:observability:ci` 在 CI 中运行，确保示例无需模型调用即可通过。
- **静态依赖图分析**：`observability-tarball-smoke.mjs` 解析 `dist/index.js`、`dist/observability/index.js`、`dist/eval/index.js` 的 import 语句，断言它们不会“急切地”引入 `@opentelemetry/*`、`node:fs` 或具体 file-store 实现；而 `observability/file.js` 与 `eval/file.js` 则必须引入 `node:fs`。

## 4. 约定与约束

| 约定 | 来源/强制执行方式 |
|---|---|
| 新增导出必须在 `core` 的 `exports` 与 `files` 中声明，否则 CI 的 tarball 白名单会失败 | `.github/workflows/ci.yml` 的 `npm pack --dry-run` 断言 |
| CLI 命令名变更需同步更新 CI 中对 `oma help` 输出的 grep 断言 | CI 步骤 `node dist/cli/oma.js help \| grep -q "oma eval run"` |
| `create-oma-app` 模板引用的 core 版本必须与仓库当前 core 版本一致 | CI 遍历 `templates/*/package.json` 对比 `jq .version` |
| OTel 对 core 的依赖必须是 `^X.Y.Z` 形式的 caret 范围 | CI 正则 `^[0-9]+\.[0-9]+\.[0-9]+$` 校验 |
| 新增模板文件必须加入 CI 的 tarball 缺失清单，否则 release-smoke 会报错 | CI 中 `missing=""` 累加未命中文件的逻辑 |
| 发布后 smoke test 要求 `npx create-oma-app@latest` 生成的项目能跑通 no-key demo 并产出 markdown/json/html 三份报告 | `.github/workflows/release-smoke.yml` 的 `grep -q` 断言 |
| 所有子包遵循 `tsc` 编译到 `dist/`、`prepublishOnly: tsc` 的统一模式 | 各 `package.json` scripts 一致 |

## 5. 补充说明

- 仓库没有 Makefile / Dockerfile / Shell 构建脚本；所有构建、测试、打包、发布验证均通过 npm scripts + GitHub Actions YAML + Node 脚本完成。
- 不存在跨平台交叉编译；Node 版本矩阵仅在 Linux runner (`ubuntu-latest`) 上执行。
- 覆盖率数据通过 `codecov-action@v5` 的 OIDC 模式上传，路径硬编码为 `./packages/core/coverage/lcov.info`。