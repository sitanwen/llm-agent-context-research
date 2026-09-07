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

## 7. 三种方案对比

| 产品 | 对大 Tool Output 的处理 | 完整结果处理 | 后续获取方式 |
|---|---|---|---|
| Lossless Claw | 超阈值外置，Context 中保留紧凑引用 | 保存到磁盘 | `lcm_grep` / `lcm_describe` |
| OpenAI Codex | 对 model-facing Tool Output 应用 `TruncationPolicy` | Runtime 保留 raw output 等状态 | Agent 后续继续通过工具获取所需信息 |
| Claude Code | MCP Tool Output 有默认 token 上限，大结果可持久化 | 文件/外部结果 | 文件读取、分页或二次查询 |

共同点非常明显：

> **都不会默认允许任意大的 Tool Output 无限制进入 LLM Context。**

---

## 8. 本次任务为什么会跑偏

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

## 9. 建议优化方案

### 9.1 短期：增加截断后的强制校验

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

### 9.2 中期：优化 Tool 输出

对容易产生几十 KB 甚至更大结果的 Tool 做源头优化：

1. **增加查询条件**：让后端直接过滤，而不是全量返回后再让 LLM 搜。
2. **字段裁剪**：只返回当前任务需要的字段。
3. **分页 / Cursor**：大列表分批获取。
4. **Top-K / 关键词搜索**：服务端先检索，再返回最相关记录。
5. **Result ID**：完整结果留在服务端/磁盘，只返回句柄，需要时二次查询。

核心原则：

> **让数据源负责“找数据”，让 LLM 负责“判断数据”。**

### 9.3 阈值可以调，但不应作为根治方案

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

## 10. 建议对客户的最终表述

> 经调研，主流 Coding Agent / Agent Runtime 普遍会对超大 Tool Output 进行限制、截断或外置。OpenAI Codex 在源码中对 model-facing Tool Output 使用 TruncationPolicy；Claude Code 对 MCP Tool Output 设置默认 token 上限，并支持大结果持久化、文件引用和分页；Lossless Claw 的大文件外置 + 按需检索机制与上述方案设计方向一致。
>
> 因此，本次 LCM 对约 77KB Tool Output 执行保护不建议定性为上下文引擎故障。真正需要优化的是截断后的流程控制：当 Agent 得知 Tool Result 不完整后，应先完成信息检索补全，再进入后续业务判断。同时，可从 Tool 侧增加过滤、字段裁剪和分页，从源头减少无效大输出。
>
> LCM 阈值可以根据客户模型实际 Context Window 做适度调优，但不建议通过无限提高阈值来规避该机制，否则会增加 Context Window 被大 Tool Output 占满的风险。

---

## 11. 官方参考资料

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
