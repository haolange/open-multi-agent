---
kind: frontend_style
name: OMA Run Viewer — 内联 CSS + 设计令牌驱动的离线单文件仪表盘
category: frontend_style
scope:
    - '**'
source_files:
    - packages/core/src/dashboard/render-run-viewer.ts
    - packages/core/src/dashboard/render-team-run-dashboard.ts
    - packages/core/src/dashboard/layout-tasks.ts
    - packages/core/src/dashboard/layout-waterfall.ts
    - packages/core/src/dashboard/run-viewer-model.ts
    - packages/core/tests/dashboard-render.test.ts
    - packages/create-oma-app/template/src/index.ts
    - packages/create-oma-app/template/src/report.ts
---

## 1. 系统/方法

本仓库没有独立的样式工程（无 `*.css`、`*.scss`、Tailwind、PostCSS 等构建产物），而是通过一个**纯函数渲染器**在运行时生成完整的离线 HTML 文档。核心入口为 `packages/core/src/dashboard/render-run-viewer.ts`，它返回一段自包含的 `<!doctype html>` 字符串：
- 所有样式以 `<style>` 标签内联注入，不依赖任何外部 CSS 资源。
- 交互逻辑以 IIFE 形式的原生 JavaScript 内联注入到 `<script type="application/json" id="oma-data">` 之后的 `<script>` 中，仅读取 JSON payload 进行 DOM 操作。
- 通过 CSP `default-src 'none'` 与 `style-src 'unsafe-inline'; script-src 'unsafe-inline'` 严格限制网络加载，确保生成的报告可离线打开且不受 CDN 影响。
- 测试 `packages/core/tests/dashboard-render.test.ts` 显式断言输出不包含 `cdn.tailwindcss.com`、`fonts.googleapis.com` 以及任何远程 `<script src=...>` / `<link href=...>`，从而强制“零外部依赖”约束。

该方案服务于 Open Multi Agent 的 **Run Viewer**（DAG 视图 + Waterfall 时序图 + 详情面板），由 `renderTeamRunDashboard` 作为向后兼容包装暴露给上层，并被 `create-oma-app` 模板写入磁盘报告。

## 2. 关键文件

- `packages/core/src/dashboard/render-run-viewer.ts` — 统一渲染器，组装模型、布局、JSON payload 并拼接最终 HTML 字符串。
- `packages/core/src/dashboard/layout-tasks.ts` — DAG 节点尺寸常量（`DAG_NODE_WIDTH` / `DAG_NODE_HEIGHT`）。
- `packages/core/src/dashboard/layout-waterfall.ts` — Waterfall 行布局计算。
- `packages/core/src/dashboard/run-viewer-model.ts` — 将运行结果转换为 viewer 内部模型。
- `packages/core/src/dashboard/render-team-run-dashboard.ts` — 旧版 `TeamRunResult` 适配层，转发到 `renderRunViewer`。
- `packages/core/tests/dashboard-render.test.ts` — 安全与样式契约测试（XSS 转义、敏感字段脱敏、CSP、无远程资源、类名存在性）。
- `packages/create-oma-app/template/src/index.ts`、`report.ts` — 调用 `renderTeamRunDashboard` 并将结果写入 `.html` 报告。

## 3. 架构与约定

### 设计令牌（Design Tokens）
样式集中在 `<style>` 顶部 `:root` 块中，使用 CSS 自定义属性定义暗色主题：
- 语义色：`--ink`、`--muted`、`--faint`、`--void`、`--deck`、`--panel`、`--line`、`--line-strong`。
- 强调色：`--mint`、`--cyan`、`--amber`、`--coral`、`--violet`，分别对应 task、agent、tool、routing、llm 等 span kind。
- 排版令牌：`--display`、`--body`、`--mono` 三套字体栈；`--radius`、`--shadow` 等视觉变量。
- 组件通过 `class="kind-{llm|tool|task|agent|routing}"` 与 `class="status-{ok|error|failed|timeout|budget_exhausted|rejected|suspended|in_progress|pending}"` 组合选择器映射到令牌。

### 响应式策略
- 基于 CSS Grid 的 shell 布局（`.shell` 5 行网格：masthead / summary / toolbar / workspace / details）。
- 两个断点：`@media (max-width: 940px)` 将双栏改为上下堆叠、工具栏换行；`@media (max-width: 520px)` 进一步压缩 masthead、隐藏路由摘要网格列。
- 通过 `prefers-reduced-motion` 控制过渡动画开关。

### 安全与可访问性
- CSP 禁止一切默认源，仅允许内联样式与脚本，图片仅允许 data URI。
- 用户输入经 `escapeJsonForHtmlScript` 转义后嵌入 `<script>`，并通过 `redactSensitiveObject` 脱敏后再序列化到 JSON payload。
- 测试断言 HTML 中不会出现 `onerror=`、`svg onload` 等 XSS 模式，且敏感值被替换为 `[redacted]`。
- 使用 ARIA 角色（`tablist`、`tab`、`tabpanel`、`role="status"`）、`aria-live` 区域与键盘导航（Home/End/ArrowLeft/Right/Escape）。

### 模块边界
- `render-run-viewer.ts` 是纯函数，不读写文件系统或发起网络请求，便于单元测试。
- 布局计算（`layout-tasks.ts`、`layout-waterfall.ts`）与模型转换（`run-viewer-model.ts`）解耦，渲染器只负责拼装。
- 对外仅暴露 `renderTeamRunDashboard` 与 `escapeJsonForHtmlScript`，其余实现细节对消费者不可见。

## 4. 约定与约束

### 观察到的约定
- 所有 UI 文本、HTML 结构与样式都内联在渲染器中，不引用外部 CSS/JS 资源。
- 颜色、字体、间距等视觉变量集中声明在 `:root`，组件通过 class 组合而非新增 token 扩展。
- Span 类型通过 `kind-*` 类名区分，状态通过 `status-*` 类名区分，二者正交。
- 运行时数据一律通过 `<script type="application/json" id="oma-data">` 传递，前端 JS 仅做单向读取。
- 报告标题固定为 `Open Multi Agent / Post-run artifact`，品牌前缀 `//` 由 CSS `::before` 伪元素注入。

### 明确执行的规则（由测试强制）
- 禁止引用远程资源：输出不得包含 `cdn.tailwindcss.com`、`fonts.googleapis.com`、任何 `<script src=...>` 或 `<link href=...>`（`dashboard-render.test.ts` 断言）。
- 禁止未转义的脚本终止符：嵌入 JSON 中的 `</script` 必须转义，防止 XSS（`escapeJsonForHtmlScript` + 测试用例）。
- 敏感字段必须脱敏：`OPENAI_API_KEY`、`Authorization: Bearer ...` 等出现在 HTML 与 JSON payload 中会被替换为 `[redacted]`。
- 任务 description/result 不得进入 JSON payload，仅保留在 HTML 展示侧（测试断言 `description` 与 `result` 字段缺失）。
- 必须输出受限 CSP：`default-src 'none'` 必须存在于 HTML 头部（测试断言）。
- 必须包含特定 DOM 结构：`id="waterfallTab"`、`id="dagTab"`、`id="details"`、`class="masthead-primary"`、`class="run-context"`、`id="routingSummary"`（用于下游装饰与测试）。

### 约束来源
- 代码注释：`render-run-viewer.ts` 首行注释 “Pure, self-contained HTML renderer for one OMA run (no filesystem or network I/O)” 定义了模块职责边界。
- 测试用例：`packages/core/tests/dashboard-render.test.ts` 是样式/安全契约的唯一权威来源，任何破坏这些断言的变更都会导致 CI 失败。
- 模板消费方：`create-oma-app` 模板直接写入 `.html` 文件并可选用 `decorateDashboard` 二次包装，因此渲染器输出的 HTML 结构被视为稳定契约。