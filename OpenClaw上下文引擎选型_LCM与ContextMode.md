# OpenClaw 上下文引擎选型：LCM、Context Mode 与 Tool Result Spill

> 调研日期：2026-09-15
> 调研目标：比较 LCM 与 Context Mode 谁更适合担任 OpenClaw 主 Context Engine，并确定超长工具输出的治理方案

## 1. 结论

当前建议：

1. **LCM 继续作为 OpenClaw 主 Context Engine。**它实际实现了原始消息持久化、分层摘要、预算内组装、压缩和历史展开，覆盖了 OpenClaw Context Engine 的主要职责。
2. **Context Mode 用于工具执行优化和可检索记忆。**它擅长将大文件、日志和 API 数据放在上下文外处理，只返回筛选结果，从源头减慢上下文增长。
3. **在 OpenClaw 工具结果边界新增 spill-policy。**对于未经过 Context Mode 的普通工具和第三方 MCP 结果，保存全文，仅把受限预览、引用和读取提示送入模型。
4. **在 LCM 压缩阶段增加 tool-result pruner。**清理已经进入历史、没有被 spill 覆盖的超长工具结果。

推荐架构：

```text
Context Mode 式工具路由：执行前避免产生无用大输出
                    ↓
OpenClaw spill：工具完成后立即外置超长结果
                    ↓
LCM pruner：压缩时裁剪历史中的大 Tool Result
                    ↓
LCM summary + assemble：整理整个长会话
                    ↓
OpenClaw：校验完整 provider-bound prompt 是否在预算内
```

## 2. 为什么不能只比较“谁节省的 token 更多”

OpenClaw 对 Context Engine 的要求不只是减少工具输出。官方接口包含：

- `ingest()`：保存或索引会话消息；
- `assemble()`：每次模型调用前，返回符合 token budget 的有序消息；
- `compact()`：窗口已满或执行 `/compact` 时，实际总结或缩短旧历史；
- `afterTurn()` / `maintain()`：在轮次结束后持久化和维护状态；
- 可选的父子 Agent 上下文生命周期。

因此选型要回答的是：会话已经很长以后，谁能决定模型下一轮看见哪些消息，并让 overflow recovery 真正改变下一次请求。

官方接口：[OpenClaw Context Engine](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context-engine.md)

## 3. 为什么 LCM 更符合主 Context Engine 的职责

LCM 的工作链路是：

```text
保存所有原始消息
→ 将旧消息生成 leaf summary
→ 多个摘要继续合并成更高层摘要
→ 每轮组合“摘要 + 最近原始消息”
→ 按宿主 token budget 生成 assembly
→ 需要细节时搜索或展开原始内容
```

LCM 不只是保存“记忆”。它的 `assemble()` 会决定当前模型输入包含哪些历史；`compact()` 会实际运行摘要和合并；数据库可用时声明 `ownsCompaction: true`，由自身承担 `/compact` 和 overflow recovery。

这不等于 LCM 不会出错。LCM 0.15.6 仍需要解决或验证：

- degraded-live/fresh-tail 过大；
- LCM assembly 预算与 OpenClaw 完整 prompt 预算不一致；
- 普通工具结果增长过快；
- summarizer 超时、限流或失败；
- OpenClaw 是否把 assembly 作为最终权威消息；
- session rotation 与 prompt lock 的并发安全。

“LCM 更符合职责”的准确含义是：**它已经实现了存储、压缩、组装和找回这条完整链路。**最终稳定性仍要通过你们固定版本的长流程测试验证。

官方资料：[Lossless Claw README](https://github.com/Martian-Engineering/lossless-claw/blob/main/README.md)、[LCM 配置文档](https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/configuration.md)

## 4. Context Mode 在 OpenClaw 中实际做了什么

Context Mode 的突出能力是工具数据处理：

```text
普通方式：read 10万行日志 → 10万行进入模型上下文

Context Mode：ctx_execute_file 读取完整日志
             → 在外部程序中搜索/统计
             → stdout 只返回几十行结论
```

其 OpenClaw 适配还会：

- 在 `before_tool_call` 阻止、修改或放行工具调用；
- 在 `after_tool_call` 把事件记录到 SQLite；
- compaction 前生成 resume snapshot；
- compaction 后向 system context 注入恢复信息。

但是，截至本报告日期，官方仓库 `main` 分支中的 Context Engine 实现是：

```ts
info: { ownsCompaction: false },

async assemble({ messages }) {
  return { messages, estimatedTokens: 0 };
},

async compact() {
  return { ok: true, compacted: false };
}
```

这表示：

- `assemble()` 原样返回历史，没有按预算选择或总结消息；
- `estimatedTokens` 固定为 0，不是实际模型输入估算；
- `compact()` 是 no-op，并明确返回没有压缩；
- resume snapshot 用于帮助 Agent 恢复任务状态，不等于把完整长历史压缩成模型输入。

源码证据：[Context Mode OpenClaw plugin.ts](https://github.com/mksglu/context-mode/blob/main/src/adapters/openclaw/plugin.ts#L858-L882)

OpenClaw 官方文档说明，`ownsCompaction: false` 不会自动切换到 legacy Context Engine。非自主管理压缩的引擎需要在 `compact()` 中调用 `delegateCompactionToRuntime(...)`；活动引擎使用 no-op `compact()` 会使正常 `/compact` 和 overflow recovery 路径不完整。

因此，Context Mode 当前更适合作为工具数据控制和恢复增强方案。若要选为主 Context Engine，需要先确认准备部署的发布版本是否已经实现压缩委托，并用真实长会话验证。其部分文档写有“owning compaction”，但当前源码为 `ownsCompaction: false`，这个文档/实现差异本身也是选型风险。

官方资料：[Context Mode 仓库](https://github.com/mksglu/context-mode)、[OpenClaw 适配说明](https://github.com/mksglu/context-mode/blob/main/docs/adapters/openclaw.md)

## 5. 两者能力对比

| 维度 | LCM | Context Mode 当前 OpenClaw 适配 |
|---|---|---|
| 历史太长后实际压缩 | DAG 分层摘要 | Context Engine 的 `compact()` 未压缩 |
| 每轮按预算组装历史 | 摘要 + fresh tail | `assemble()` 原样返回消息 |
| 压缩后细节找回 | 摘要引用、搜索和展开 | FTS5/BM25 事件搜索 |
| 工具大输出事前预防 | 不是主要强项 | `ctx_*` 外部执行、筛选和索引 |
| 绕过工具路由后的兜底 | 需要 spill/pruner | 当前普通 OpenClaw tool output 不能统一改写 |
| 用户纠偏和任务连续性 | 依赖摘要质量与 fresh tail | 依赖事件快照和恢复注入 |
| 作为主 Context Engine | 职责较完整 | 需补 compact 委托和预算组装验证 |

仓库 star 数不能代替上述行为测试。一个工具输出优化项目可能更受欢迎，但不代表它已经完整承担 OpenClaw Context Engine 的契约。

## 6. 为什么 LCM 之外还必须研究 spill

工具返回结果是下一次模型请求的一部分。单条结果过大时，会挤占 system prompt、tool schemas、历史消息和模型输出空间，并且最新结果通常位于 LCM 不能随意删除的 fresh tail。

Context Mode 的执行前路由也不能覆盖所有工具：

- 模型可能继续调用普通 `read` 或 shell；
- 第三方 MCP 和自定义工具名可能没有命中规则；
- 工具执行前不一定能预测结果大小；
- 某个工具自身已经返回了完整大对象；
- OpenClaw 普通 `after_tool_call` hook 可以观察结果，但不保证能改写同一轮模型真正看到的输出。

因此需要宿主级 spill：无论结果来自哪个工具，都在进入模型消息、transcript 和 LCM fresh tail 前执行统一大小检查和外置。

## 7. DeepSeek Harness 提供的参考方案

DeepSeek Harness 将工具输出治理拆成两个独立机制。

### 7.1 `spill-policy`：工具返回后立即外置

纯文本结果超过配置的 `maxInlineBytes` 后：

1. 完整文本保存到 session-scoped spill store；
2. 在同一字节预算内生成头尾预览；
3. 附加 locator 和 retrieval hint；
4. 模型只接收预览和引用。

它按照 UTF-8 字节计算。`maxInlineBytes` 未配置时不启用。官方材料中出现的 `50000` 是配置示例，不能写成所有部署统一的内部默认值。

官方资料：[DSH Spill 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/spill.md)、[spill-policy 源码](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill-policy/src/index.ts)

### 7.2 `compaction-tool-result-pruner`：压缩时裁剪历史结果

官方默认配置：

```yaml
thresholdChars: 8192
headChars: 4096
tailChars: 1024
```

超过 8192 个 Unicode 码点的历史工具结果，模型可见内容变为：

```text
前4096个码点
[... tool result middle pruned ...]
后1024个码点
```

它只在 compaction 压力或确认溢出后运行，不会在每次工具刚返回时执行。它不调用模型，也不判断中间内容的重要性。完整原始事件保留在 append-only session log 中用于回放，但按需找回仍需要可访问的检索能力。

官方资料：[DSH tool-result-pruner 中文说明](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/README.zh.md)

### 7.3 两者的区别

| 机制 | 何时运行 | 统计单位 | 完整内容位置 | 主要目的 |
|---|---|---|---|---|
| spill-policy | 工具执行后 | UTF-8 bytes | spill store | 防止大结果首次进入上下文 |
| tool-result-pruner | compaction 压力/溢出后 | Unicode 码点 | 原事件日志 | 清理已进入历史的超长结果 |

“8192/4096/1024”属于 pruner，不是 spill-policy 的默认规则。对于客户遇到的工具结果突然膨胀，应优先建设 spill；pruner 是第二道历史治理。

## 8. 建议的 OpenClaw/LCM Spill 设计

推荐处理位置：

```text
ToolExecutionResult
→ 统一序列化
→ 计算 provider-bound bytes/tokens
→ 超阈值则保存全文并生成引用
→ 用“预览 + 引用”替换模型可见结果
→ 写入 transcript 和 LCM ingest
```

每条引用至少保存：

```text
spillId / locator
sessionId
toolName / toolCallId
contentType
originalBytes / estimatedTokens
sha256
createdAt
retrievalHint
```

还需要满足：

- 预览、说明和引用共同计入输出上限；
- spill 保存失败时，对模型侧结果进行安全裁剪并告警；
- `read`、`grep` 支持分页和单次返回上限；
- JSON、列表和测试报告提供字段过滤、分页、Top-K；
- 防止“读取 spill 全文 → 再次 spill → 循环读取”；
- LCM 摘要器读取预览和元数据，需要细节时显式查询；
- 最终由 OpenClaw 校验完整 provider prompt，而不是只检查 LCM assembly。

## 9. 验证方案

建议比较三组：

1. LCM 0.15.6 基线；
2. Context Mode 作为 active Context Engine；
3. LCM 0.15.6 + OpenClaw host-level spill + LCM pruner。

测试场景必须包含：

- 大日志的关键错误位于中间；
- 超大 JSON/MCP 结果；
- 同一轮连续多次工具调用；
- 用户中途纠偏，继续执行数十轮；
- 主流程 done 后重新唤醒并创建子 Agent；
- `/compact`、provider overflow 和重复恢复；
- summarizer 超时、限流或失败；
- spill 文件搜索、分页、丢失和存储失败。

关键指标：任务完成率、纠偏保留率、overflow 自动恢复率、provider context overflow 次数、峰值 final prompt tokens、spill 细节找回成功率、延迟和 token 成本。

## 10. 最终建议

```text
主 Context Engine：LCM
工具事前优化：Context Mode 式 ctx_execute/ctx_execute_file
普通工具兜底：OpenClaw 宿主级 spill-policy
历史二次治理：LCM compaction-time tool-result pruner
最终验收：OpenClaw provider-bound prompt token check
```

Context Mode 若要成为主 Context Engine 候选，需要先证明：

1. `compact()` 已实现自己的压缩，或正确调用 `delegateCompactionToRuntime(...)`；
2. `assemble()` 返回真实 token 估算，并能在长历史和工具循环中生成预算内消息；
3. overflow recovery 后的下一次请求确实变小；
4. 用户纠偏、任务状态和父子 Agent 连续性达到或超过 LCM。

在这些证据完成前，LCM 更适合承担主 Context Engine；Context Mode 和 DSH spill/pruner 的思路适合补强工具输出治理。
