---
kind: logging_system
name: 可观测性/追踪系统（Trace Sink 架构与结构化日志）
category: logging_system
scope:
    - '**'
source_files:
    - packages/core/src/observability/sink.ts
    - packages/core/src/observability/batching.ts
    - packages/core/src/observability/composite.ts
    - packages/core/src/observability/processors.ts
    - packages/core/src/observability/records.ts
    - packages/core/src/observability/file-store.ts
    - packages/core/src/observability/in-memory-store.ts
    - packages/core/src/observability/store-exporter.ts
    - packages/core/src/observability/legacy-callback.ts
    - packages/core/src/utils/trace.ts
    - packages/otel/src/exporter.ts
    - packages/otel/src/mapping.ts
    - packages/core/src/index.ts
---

## 1. 系统概览

本仓库没有传统意义上的“应用日志框架”（如 Winston、Pino、Bunyan），而是实现了一套**零依赖的 OpenTelemetry 风格追踪（trace）系统**，位于 `packages/core/src/observability/`。该系统的核心抽象是 `TraceSink`：所有业务模块通过 `emit(record)` 同步、非阻塞地投递结构化的 `TraceRecord`，再由 sink 链异步批量导出到外部后端。

- **记录模型**：`records.ts` 定义 schema v2 的 `TraceRecord = SpanStartRecord | SpanEventRecord | SpanEndRecord`，包含 `runId / attempt / traceId / spanId / sequence / timestampUnixMs` 等关联字段，以及按 `SpanKind`（`run|routing|agent|task|llm|tool|plan|consensus|checkpoint|callback`）和 `SpanEventName`（`retry_scheduled|budget_exhausted|first_chunk|approval_decision|stream_chunk|...`）分类的事件。
- **sink 接口**：`sink.ts` 定义 `TraceSink`（`emit / forceFlush / shutdown / getStats?`）、`TraceExporter`（`export(records, signal) -> ExportResult`）以及统一的 `FlushResult`（`ok|partial|timeout|error`）和诊断码体系（`sink_emit_failed|queue_full|record_too_large|export_failed|export_timeout|flush_timeout|shutdown_failed|emit_after_shutdown`）。
- **OTel 适配层**：`packages/otel/` 提供 `createOtelTraceSink()` / `createOtelTraceExporter()`，把 OMA 的 `TraceRecord` 映射为 OTLP span 并导出到任意 OTel collector。

## 2. 关键文件与包

| 路径 | 作用 |
|---|---|
| `packages/core/src/observability/sink.ts` | `TraceSink` / `TraceExporter` / `ObservabilityConfig` / `DiagnosticReporter` 核心接口与诊断器 |
| `packages/core/src/observability/batching.ts` | `BatchingTraceSink`：有界队列 + 定时批处理 + 指数退避重试 + 优先级调度（span_end > span_start > stream_chunk） |
| `packages/core/src/observability/composite.ts` | `CompositeSink`：扇出多 sink，任一子 sink 失败不影响其他 |
| `packages/core/src/observability/processors.ts` | `FilteringSink`（同步过滤）+ `SensitiveDataProcessor`（凭据脱敏、reasoning 内容移除、属性白名单、截断） |
| `packages/core/src/observability/file-store.ts` | `FileTraceStore`：本地 JSON 文件持久化 store |
| `packages/core/src/observability/in-memory-store.ts` | `InMemoryTraceStore`：内存 store |
| `packages/core/src/observability/store-exporter.ts` | `TraceStoreExporter`：从 store 导出为 `TraceRecord[]` |
| `packages/core/src/observability/legacy-callback.ts` | `LegacyCallbackTraceSink`：兼容旧版 `onTrace` 回调 |
| `packages/core/src/utils/trace.ts` | `emitTrace()`（吞掉异常保证可观测不中断主流程）、`generateRunId` / `generateSpanId` |
| `packages/otel/src/exporter.ts` | `OTelTraceExporter` / `OTelTraceSink`，基于 `@opentelemetry/sdk-trace-base` |
| `packages/core/src/index.ts` | 将 observability 公共 API 重新导出给消费者 |

## 3. 架构与设计约定

- **分层 sink 管道**：典型链路为 `AgentRunner → CompositeSink → [FilteringSink → SensitiveDataProcessor] → BatchingTraceSink → TraceExporter (OTel/File/...)`。每个组件都是 `TraceSink` 装饰器，职责单一。
- **零阻塞 emit**：`emit` 必须是同步且非阻塞的；所有 I/O、批处理、重试都在后台 worker/timer 中完成。`BatchingTraceSink` 使用 `scheduledDelayMs`（默认 5s）聚合批次，`maxBatchRecords=512`、`maxQueueRecords=2048`、`maxQueueBytes=16MB` 做内存上限保护。
- **幂等 flush/shutdown**：`forceFlush` 等待 acceptance watermark 到达目标 id；`shutdown` 幂等且会先 flush 再调用 exporter.shutdown。`CompositeSink` 对子 sink 的 flush/shutdown 统一加 30s 超时保护。
- **降级与诊断**：`DiagnosticReporter` 以固定间隔 rate-limit 输出诊断（默认 `console.warn`），支持 `silent` 模式并通过 `onDiagnostic` 回调上报；诊断消息不含 payload，避免递归进入追踪。
- **隐私边界**：`SensitiveDataProcessor` 默认关闭 prompt/completion/toolInput/toolOutput 捕获（`'none'`），仅保留 `code-only` 的错误信息；自动移除匹配 `authorization|api_key|password|secret|access_token|...` 的属性键，以及 `reasoning/thinking/chain_of_thought/signed_reasoning.*content` 字段。
- **运行期注入**：上层模块（如 `agent/runner.ts`）通过 `TraceRuntime` / `TraceSpan` 参数注入追踪上下文，而不是全局单例；`emitTrace` 包装用户回调，确保可观测代码崩溃不会中断业务执行。

## 4. 约定与约束

- **记录必须可 JSON 序列化**：`BatchingTraceSink.byteSize` 用 `JSON.stringify` 估算大小，不可序列化的 record 会被丢弃并上报 `record_too_large`。
- **sink 必须容忍异常**：`CompositeSink.emit` 捕获子 sink 抛出的异常并计入 `failed`；`emitTrace` 吞掉 async callback 的 rejection。
- **禁止在诊断中携带敏感数据**：`TelemetryDiagnostic.message` 被明确注释为 “Fixed, payload-free message. Never contains a TraceRecord or raw error.”
- **schema 版本锁定**：`TraceRecordBase.schemaVersion` 固定为 `2`，事件名集合在 `records.ts` 中枚举，新增事件需显式扩展 union 类型。
- **OTel 映射版本**：`packages/otel` 暴露 `OM_SCHEMA_VERSION` / `OTEL_GENAI_SEMCONV_VERSION`，用于向后兼容不同版本的语义约定。
- **示例与 CLI 中的 console.log**：示例脚本（如 `packages/core/examples/...`）仍使用原生 `console.log` / `console.error` 打印人类可读输出；这些不属于可观测性追踪管线，仅用于调试展示。

## 5. 适用性判断

本仓库不存在传统“应用日志系统”，但实现了完整、生产级的**结构化追踪/可观测性 sink 架构**，涵盖记录模型、sink 管道、批处理、隐私脱敏、错误恢复、OTel 导出与本地存储，属于 logging_system 范畴中的“结构化日志/追踪系统”。
