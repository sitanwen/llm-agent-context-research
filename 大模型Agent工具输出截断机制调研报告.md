# 大模型 Agent 工具输出截断/外置机制调研报告

**调研日期：2026-09-08**

## 1. 调研目的

客户反馈在使用 LCM（Lossless Claw）时，某次 Tool Output 达到约 77KB，触发了截断/外置机制，后续 Agent 基于不完整信息继续执行，最终导致任务跑偏。

本报告主要回答三个问题：

1. 对大 Tool Output 做截断、限流或外置，是否属于正常的上下文保护机制？
2. OpenAI Codex、Anthropic Claude Code 是否也采用类似机制？
3. 对当前问题，更合理的优化方向是什么？

---

## 2. 调研结论

**对大 Tool Output 进行截断、限流或外置，是主流 Agent Runtime 常见的上下文保护机制，本身不应直接定性为故障。**

原因是 Tool Result 最终也是模型下一轮输入上下文的一部分。若单次工具输出过大，会直接消耗 Context Window，挤占 System Prompt、Skill、历史消息和后续推理空间。

从公开实现看：

- **Lossless Claw**：对超过阈值的大内容进行外置，生成 `file_id`，后续通过 `lcm_grep` / `lcm_describe` 按需检索。
- **OpenAI Codex**：对模型可见的 Tool Output 应用 `TruncationPolicy`，源码中明确区分 `raw_output` 和经过截断后的 model-facing output。
- **Claude Code**：对 MCP Tool Output 设置默认最大 token 限制；超大结果可持久化到磁盘，并在对话中用文件引用替代，同时官方建议工具作者采用分页等方式控制大输出。

因此，本次约 77KB Tool Output 被保护性截断/外置，本身属于合理机制。

**真正需要优化的是：当 Agent 得知 Tool Result 已被截断后，不能继续把当前可见内容当成完整结果使用，而应先完成检索补全，再进入下一业务步骤。**

---

## 3. 为什么 Tool Output 也需要上下文管理

Agent 每一轮真正发送给 LLM 的输入，并不只有用户和助手的历史消息，还包含工具调用和工具返回结果：

```text
System Prompt
+ Skill / Instructions
+ 历史 User / Assistant 消息
+ Tool Call
+ Tool Result
+ 当前任务状态
```

因此 Tool Result 也是 Context 的一部分。

如果客户模型上下文窗口约为 200K tokens，那么 System Prompt、Skill、历史上下文、Tool Result、后续推理都要共享这 200K 预算。单次 Tool Output 过大，就可能造成：

- 历史信息被挤压；
- 更频繁触发上下文压缩；
- 延迟和 token 消耗增加；
- 极端情况下超过模型 Context Window；
- Agent 后续推理空间不足。

所以，**限制单次 Tool Output 的大小，本质上是 Context Budget 管理。**

---

## 4. Lossless Claw 的处理方式

Lossless Claw 官方架构文档中有专门的 **Large file handling** 机制。

公开文档说明：

- 超过 `largeFileTokenThreshold` 的大内容会生成唯一 `file_id`；
- 完整内容写入 `largeFilesDir`；
- 原始大内容在消息中被替换为紧凑引用；
- 后续可通过 `lcm_describe` 获取限定范围内容；
- 也可通过 `lcm_grep(scope="files")` 搜索外置文件；
- 当前公开文档中的默认阈值为 **25,000 tokens**。

官方文档：

- Lossless Claw Architecture  
  https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/architecture.md

- Lossless Claw Configuration  
  https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/configuration.md

其设计目标很明确：**避免单个大文件/大结果占满整个上下文，同时保持内容仍然可访问。**

---

## 5. OpenAI Codex 的处理方式

OpenAI Codex 的开源代码中，对 Tool Output 有明确的模型侧截断策略。

在 `ExecCommandToolOutput` 中可以看到以下字段：

```text
raw_output
truncation_policy
max_output_tokens
original_token_count
```

其中 `raw_output` 是工具真实产生的原始输出，而发送给模型的内容会经过 `TruncationPolicy`。

源码还明确存在：

```text
Warning: truncated output
```

以及：

```text
truncated_output_with_policy(...)
```

相关官方源码：

- Codex `context.rs`  
  https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/context.rs

- Codex `tools/mod.rs`  
  https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/mod.rs

`tools/mod.rs` 中明确在“发送给模型”之前调用截断逻辑，即先对模型可见的 Tool Output 做预算控制，再序列化进上下文。

### 反面案例

Codex 官方 GitHub 上有一个典型 issue：某条执行路径允许约 **497KB shell output** 进入模型，约贡献 **138K tokens**，导致请求极度膨胀。

官方 Issue：

https://github.com/openai/codex/issues/42367

这个案例反过来说明：**不控制 Tool Output 本身就是上下文稳定性风险。**

---

## 6. Claude Code 的处理方式

Claude Code 官方 MCP 文档明确说明，为避免 Tool Output 淹没 conversation context，会对 MCP Tool Output 做大小控制。

公开文档当前说明：

- Tool Output 超过 **10,000 tokens** 时会显示警告；
- 默认最大值为 **25,000 tokens**；
- 可以通过 `MAX_MCP_OUTPUT_TOKENS` 调整；
- 对于大文本工具，工具作者可以通过 `anthropic/maxResultSizeChars` 声明更大的结果预算；
- 超过默认持久化阈值的结果可以保存到磁盘，并在会话中替换为文件引用；
- 官方还明确建议，对于经常产生大结果的 MCP Server，可以采用**分页**等方式。

官方文档：

- Claude Code — MCP output limits and warnings  
  https://code.claude.com/docs/en/mcp

因此 Claude Code 的思路同样是：

```text
Tool 产生大结果
    ↓
限制直接进入模型的内容
    ↓
必要时持久化/文件引用
    ↓
需要时再读取
```

这与 LCM 的“大结果外置 + 按需检索”属于同一类设计思想。

---

## 7. DeepSeek Harness 的双层工具输出治理

DeepSeek Harness（DSH）没有只使用一种“超长就截断”的规则，而是把工具结果治理拆成两个独立阶段：

```text
工具刚执行完成
    ↓
spill-policy：超大结果全文外置，模型只接收受限预览和引用
    ↓
会话继续增长并达到 compaction 压力
    ↓
compaction-tool-result-pruner：裁剪历史中仍然过大的 tool/result
    ↓
重新计算请求大小，必要时再执行摘要压缩
```

### 7.1 `compaction-tool-result-pruner`

官方默认配置为：

```yaml
thresholdChars: 8192
headChars: 4096
tailChars: 1024
```

当一个工具结果的文本超过 8,192 个 Unicode 码点时，模型后续看到的内容变为：

```text
前 4,096 个码点

[... tool result middle pruned ...]

后 1,024 个码点
```

这里的单位是 Unicode 码点，不是 token，也不是 UTF-8 字节。不同语言和内容的 token 密度不同，因此完成裁剪后仍要通过统一 token meter 重新判断请求是否已经降到安全范围。

该 pruner 只在 compaction 的压力或确认溢出条件成立后运行，不会在每次工具刚返回时立即执行。它会在 append-only session log 中保留完整原始事件，并追加一条较短的替换事件，因此适合回放和排障；但官方能力没有承诺模型可以直接通过该 pruner 的省略标记取回中间内容，若业务需要按需恢复，还需要配套的检索或 spill 引用。

它的优势是确定、快速、不调用模型。它的风险也很明确：只保留头尾，不理解中间哪些内容更重要。如果关键错误正好位于日志中间，模型可见结果中就没有这段信息。

官方说明：

- DeepSeek Harness `compaction-tool-result-pruner`
  https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/README.zh.md

- DeepSeek Harness Compaction 子系统
  https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md

### 7.2 `spill-policy`

`spill-policy` 工作在工具执行后的结果管线。当纯文本结果超过配置的 `maxInlineBytes` 时：

1. 将完整文本保存到 session-scoped spill store；
2. 在 `maxInlineBytes` 预算内生成头尾预览；
3. 在预览后附加 locator 和读取提示；
4. 使用这份受限内容替换模型可见的工具结果。

`maxInlineBytes` 按 UTF-8 字节计算。源码中该配置是可选项，未配置时 policy 不启用；官方设计材料中出现的 `50000` 是一种部署配置示例，不应表述为插件内部对所有部署都生效的默认值。

Spill 与直接截断的最大区别是：**被省略的全文仍然有稳定引用，模型可以在需要时使用 `read`、`grep` 或其他后端提供的 retrieval hint 查找。**

DSH 还处理了两个容易忽略的问题：

- 预览和引用提示共同计入 `maxInlineBytes`，避免“预览已经用满预算，后面再追加路径”导致替换结果重新超限；
- 保存失败时保持原工具调用成功并返回原始内容，避免存储故障把正常工具调用变成错误。

第二点适合保证工具可用性，但对 Context Window 并不安全。如果在 OpenClaw 中参考该方案，spill 保存失败后仍应执行最后一道 provider-bound 安全裁剪，并记录告警，避免完整超长结果直接进入模型。

官方说明：

- DeepSeek Harness Spill 子系统
  https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/spill.md

- DeepSeek Harness `spill-policy` 源码
  https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill-policy/src/index.ts

### 7.3 两个机制不能混为同一条规则

| 机制 | 触发时间 | 单位 | 完整内容 | 模型看到什么 |
|---|---|---|---|---|
| `spill-policy` | 工具执行完成后 | UTF-8 bytes | 保存到 spill store | 头尾预览 + 引用 + 读取提示 |
| `compaction-tool-result-pruner` | compaction 压力/溢出成立后 | Unicode 码点 | 原事件留在 append-only log | 前 4096 + 省略标记 + 后 1024（默认） |

因此，“超过 8192 个字符，保留 4096 + 1024”属于 pruner 默认值，不是 spill-policy 的默认配置。对于客户遇到的单次 Tool Output 突然膨胀问题，应优先参考 spill；pruner 负责清理已进入历史或未被 spill 覆盖的结果。

### 7.4 对 OpenClaw/LCM 的适配建议

推荐落点是 OpenClaw 的 model-facing 工具结果边界，即工具结果写入当前工具循环、transcript 和 LCM fresh tail 之前：

```text
ToolExecutionResult
    ↓
统一序列化并计算 bytes/tokens
    ↓
超阈值：保存全文并生成 ToolResultRef
    ↓
模型消息、OpenClaw transcript、LCM ingest 接收“预览 + 引用”
    ↓
LCM 后续对旧历史继续摘要或 pruner
```

只在 LCM assembly 阶段把旧结果替换成 stub 仍然不够，因为最新超大结果可能位于受保护的 fresh tail，并已经在同一轮工具循环中发送给模型。只使用 Context Mode 的执行前路由也不够，因为普通工具、第三方 MCP 和未命中规则的自定义工具仍可能返回大结果。

一个可落地的 spill 记录至少应包含：

```text
spillId / locator
toolName
toolCallId
sessionId
contentType
originalBytes
estimatedTokens
sha256
createdAt
retrievalHint
```

读取接口必须支持分页、搜索和返回上限，防止出现：

```text
读取 spill 全文
→ 再次产生超大 Tool Result
→ 再次 spill
→ Agent 循环读取
```

对 JSON、表格、测试报告等结构化结果，优先提供字段过滤、分页、Top-K 和查询接口；头尾预览是兜底，不应代替结构化查询。

---

## 8. 四种方案对比

| 产品 | 对大 Tool Output 的处理 | 完整结果处理 | 后续获取方式 |
|---|---|---|---|
| Lossless Claw | 超阈值外置，Context 中保留紧凑引用 | 保存到磁盘 | `lcm_grep` / `lcm_describe` |
| OpenAI Codex | 对 model-facing Tool Output 应用 `TruncationPolicy` | Runtime 保留 raw output 等状态 | Agent 后续继续通过工具获取所需信息 |
| Claude Code | MCP Tool Output 有默认 token 上限，大结果可持久化 | 文件/外部结果 | 文件读取、分页或二次查询 |
| DeepSeek Harness | 工具完成后 spill，压缩时再 pruner | Spill store + append-only 原事件 | locator/retrieval hint；原事件用于回放检查 |

共同点非常明显：

> **都不会默认允许任意大的 Tool Output 无限制进入 LLM Context。**

---

## 9. 本次任务为什么会跑偏

本次问题可以拆成两个阶段。

### 阶段一：LCM 正常保护

```text
Tool 返回约 77KB
        ↓
LCM 判断结果过大
        ↓
执行截断/外置
        ↓
Context 中只保留部分结果/引用
        ↓
提示 Agent 使用 lcm_grep 获取完整信息
```

这一阶段属于正常机制。

### 阶段二：Agent 后续处理不足

理论上应该：

```text
发现结果 truncated
        ↓
确认当前结果“不完整”
        ↓
调用 lcm_grep / lcm_describe
        ↓
检索当前业务步骤需要的信息
        ↓
信息补齐
        ↓
继续下一步
```

但实际问题表现为：

```text
发现结果 truncated
        ↓
Agent 仍直接使用当前局部信息
        ↓
进入下一业务判断
        ↓
错误/缺失信息继续向后传播
        ↓
任务跑偏
```

因此更准确的定性是：

> **LCM 的截断机制是问题的触发条件，但不是故障根因。根因更偏向 Agent/OpenClaw 在消费“不完整 Tool Result”时，缺少强制的信息补全和流程门禁。**

---

## 10. 建议优化方案

### 10.1 短期：增加截断后的强制校验

当检测到：

```text
truncated = true
或
Output truncated
或
存在 file_id / externalized result
```

应将当前 Tool Result 标记为“不完整”，并增加硬约束：

```text
Tool Result 被截断
        ↓
禁止直接进入下一业务步骤
        ↓
必须先执行 lcm_grep / lcm_describe
        ↓
补齐当前任务所需信息
        ↓
再继续流程
```

这是当前问题最直接的止血方案。

### 10.2 中期：优化 Tool 输出

对容易产生几十 KB 甚至更大结果的 Tool 做源头优化：

1. **增加查询条件**：让后端直接过滤，而不是全量返回后再让 LLM 搜。
2. **字段裁剪**：只返回当前任务需要的字段。
3. **分页 / Cursor**：大列表分批获取。
4. **Top-K / 关键词搜索**：服务端先检索，再返回最相关记录。
5. **Result ID**：完整结果留在服务端/磁盘，只返回句柄，需要时二次查询。

核心原则：

> **让数据源负责“找数据”，让 LLM 负责“判断数据”。**

### 10.3 阈值可以调，但不应作为根治方案

可以结合客户约 200K Context Window 做压测，适度提高 Tool Output 的内联阈值。

但不建议简单从 32KB 无限调大：

```text
32KB → 77KB 被截断
128KB → 当前 77KB 不截断
未来 300KB → 仍然出现相同问题
```

同时阈值越高，单次 Tool Output 越容易占用大量上下文预算。

因此建议：

> **阈值调整作为缓解措施；截断后的可靠恢复 + Tool 源头控量才是长期方案。**

---

## 11. 建议对客户的最终表述

> 经调研，主流 Coding Agent / Agent Runtime 普遍会对超大 Tool Output 进行限制、截断或外置。OpenAI Codex 在源码中对 model-facing Tool Output 使用 TruncationPolicy；Claude Code 对 MCP Tool Output 设置默认 token 上限，并支持大结果持久化、文件引用和分页；Lossless Claw 的大文件外置 + 按需检索机制与上述方案设计方向一致。
>
> 因此，本次 LCM 对约 77KB Tool Output 执行保护不建议定性为上下文引擎故障。真正需要优化的是截断后的流程控制：当 Agent 得知 Tool Result 不完整后，应先完成信息检索补全，再进入后续业务判断。同时，可从 Tool 侧增加过滤、字段裁剪和分页，从源头减少无效大输出。
>
> LCM 阈值可以根据客户模型实际 Context Window 做适度调优，但不建议通过无限提高阈值来规避该机制，否则会增加 Context Window 被大 Tool Output 占满的风险。

---

## 12. 官方参考资料

1. **Lossless Claw — Architecture / Large file handling**  
   https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/architecture.md

2. **Lossless Claw — Configuration**  
   https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/configuration.md

3. **OpenAI Codex — Tool Output Context / TruncationPolicy**  
   https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/context.rs

4. **OpenAI Codex — Exec output formatting and truncation**  
   https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/mod.rs

5. **OpenAI Codex Issue #42367 — Large shell output entering model context**  
   https://github.com/openai/codex/issues/42367

6. **Claude Code — MCP output limits and warnings**  
   https://code.claude.com/docs/en/mcp

7. **DeepSeek Harness — Tool result pruner**
   https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/README.zh.md

8. **DeepSeek Harness — Spill subsystem**
   https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/spill.md

9. **DeepSeek Harness — Spill policy source**
   https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill-policy/src/index.ts
