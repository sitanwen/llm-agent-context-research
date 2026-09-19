# Lossless Claw 的价值、复杂度与 Codex / Claude Code 上下文管理对比

> 更新日期：2026-09-20  
> 讨论范围：OpenClaw legacy、Lossless Claw、Context Mode、Codex、Claude Code，以及超长工具输出治理  
> 结论口径：基于公开文档、正式源码和本次事故日志；不以 GitHub star 数代替工程验证

## 1. 执行结论

Lossless Claw（LCM）不能简单评价为“差”或“没有价值”。它试图解决的是一个比普通自动压缩更难的问题：**不仅让当前请求继续执行，还要长期保存原始消息、建立分层摘要，并允许 Agent 以后找回被压缩的细节。**

它的问题也很明确：LCM 作为第三方 Context Engine 插入 OpenClaw 后，同时参与消息持久化、摘要、组装、压缩、检索、大文件外置、子 Agent 作用域和 session 生命周期。这带来了较大的状态面和兼容面，容易与 OpenClaw 自己的 transcript、prompt budget、tool loop、prompt lock 和升级节奏发生耦合。

因此应按客户目标选型：

| 客户第一目标 | 优先方案 | 原因 |
|---|---|---|
| 主流程不能中断、故障面最小 | **OpenClaw 原生 `legacy` Context Engine** | 上下文组装、压缩、重试和最终模型请求由同一宿主管理，少一套外部状态和契约边界 |
| 长时间运行且需要找回早期原始细节 | **评估 LCM** | SQLite 持久化、DAG 摘要和检索/展开能力强于一次性摘要 |
| 主要问题是工具输出太大 | **先治理工具输出** | 采用源头分页/筛选、spill、头尾预览和宿主截断；不必仅为此引入完整 LCM |
| 希望减少文件、日志和 API 数据进入上下文 | **Context Mode 作为辅助层** | 适合事前路由和沙箱内检索；当前不宜单独承担 OpenClaw 主 Context Engine 的压缩职责 |

最重要的判断是：

> **如果客户把“流程不能中断”放在第一位，应优先让 OpenClaw 原生 legacy 承担压缩。**这不是说 LCM 没有价值，而是说明该客户当前更重视低故障面，而不是可追溯的长期记忆。

## 2. 为什么 LCM 看起来“设计臃肿”

LCM 的链路大致如下：

```text
OpenClaw transcript
        ↓ 导入/对账
LCM SQLite 原始消息
        ↓
leaf summary
        ↓
condensed summary DAG
        ↓
summary + fresh tail 组装
        ↓
OpenClaw prompt preflight / tool loop
        ↓
provider-bound request
```

它还要处理：

- transcript 与数据库的 bootstrap、reconciliation 和幂等写入；
- 后台压缩债务、强制压缩和 summarizer 失败；
- fresh tail 的保护与预算控制；
- 大文件外置、摘要、检索和展开；
- 主会话、delegated session 和子 Agent 的作用域；
- rotation、prompt lock 和活动 session 的并发安全；
- OpenClaw 不同版本的 Context Engine 契约变化。

所以“臃肿”可以拆成两类。

### 2.1 必要复杂度

如果产品目标确实是“原始历史不丢失、摘要可追溯、可以按需展开”，以下组件基本不可避免：

- 原始消息持久化；
- 摘要节点及其来源关系；
- 多层摘要合并；
- 当前轮 assembly；
- 搜索和展开接口；
- token 预算、失败恢复和数据完整性校验。

这部分复杂度换来了普通滑窗摘要没有的能力，不能全部视为设计错误。

### 2.2 集成复杂度

更值得质疑的是插件与宿主之间的重复职责：

- OpenClaw 有一份 transcript，LCM 又有一份消息数据库；
- LCM 计算 assembly 预算，OpenClaw 还要计算完整 prompt；
- LCM 负责 compact，OpenClaw 负责 provider overflow recovery；
- LCM 想轮转 session，OpenClaw 掌握活动 prompt lock；
- LCM 外置工具结果，OpenClaw 也有 tool-result trimming；
- 两边都可能对“当前权威消息”做判断。

本次多起故障主要来自这类边界：不是 DAG 摘要概念本身错误，而是**两个控制平面必须在消息权威、预算口径、锁和生命周期上严格一致**。

## 3. LCM 的真正价值是什么

LCM 官方定义的核心能力包括：

1. 将每条消息持久化到 SQLite；
2. 把旧消息总结成 leaf summaries；
3. 把多个 summary 继续压成更高层节点，形成 DAG；
4. 每轮用“摘要 + 最近原始消息”组装上下文；
5. 通过 `lcm_grep`、`lcm_describe`、`lcm_expand` 搜索和恢复细节。

官方资料：[Lossless Claw README](https://github.com/Martian-Engineering/lossless-claw/blob/main/README.md)、[Architecture](https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/architecture.md)

这里的“lossless”不应理解成“每一次模型 prompt 都包含全部原文”。更准确的含义是：

```text
模型当前看到的是压缩视图
原始消息仍保存在持久层
摘要能够追溯到来源
需要时可以搜索或展开
```

它尤其适合：

- 一个 Agent 连续运行数天或数周；
- 业务要求审计“这个结论来自哪一轮”；
- `/new` 或会话迁移后仍希望保留结构化长期记忆；
- 早期细节不能只依赖一份不可回溯的总摘要；
- 希望模型按需检索旧内容，而不是每轮重放全部历史。

如果这些需求不存在，只需要“当前编码任务在一个窗口内稳定完成”，LCM 的收益会明显下降，而集成成本仍然存在。

## 4. 为什么 LCM 问题不少却仍在不断更新

持续更新同时说明两件事，不能只看其中一面。

### 4.1 正面含义：项目仍在解决真实工程问题

上下文管理不是单个摘要函数，而是持久化、并发、预算、工具循环、父子会话和 provider 差异的组合。活跃更新意味着维护者正在补齐契约、迁移、诊断和异常恢复，而不是放弃项目。

### 4.2 风险含义：它仍处于快速演进和兼容磨合期

OpenClaw 自身的 Context Engine 契约也在快速变化，例如：

- 从旧 JSONL 兼容路径转向 SQLite-backed session；
- assembly 的权威语义和 `promptAuthority`；
- durable turn fencing 与幂等 commit；
- `maintain()`、后台维护和资源生命周期；
- 子 Agent 创建、结束和 conversation scope；
- transcript rewrite 与 session target。

插件必须跟随宿主变化。更新频繁既是活跃维护的证据，也是生产用户需要固定版本、做 canary 和回归测试的信号。

### 4.3 客观评价

```text
更新多 ≠ 产品一定不稳定
更新多 ≠ 产品已经成熟稳定
```

正确做法是观察：是否有兼容矩阵、回归测试、故障隔离、数据库迁移、可观测性、稳定 release channel，以及你们关键场景的持续通过率。

## 5. OpenClaw legacy 怎么管理上下文

OpenClaw 内置 `legacy` 引擎的职责比较集中：

- transcript 持久化仍由 session manager 负责；
- `assemble()` 走宿主已有的 sanitize、validate 和 limit 流程；
- `compact()` 委托给 OpenClaw 内置摘要压缩；
- 对旧历史生成一份摘要，同时保留最近消息；
- `/compact`、自动压缩、provider overflow recovery 与最终请求都由同一宿主协调。

官方说明：[OpenClaw Context Engine：The legacy engine](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context-engine.md#the-legacy-engine)

legacy 的优势不是记忆能力更强，而是控制面更少：

```text
同一份 session 状态
→ 同一个宿主计算预算
→ 同一个宿主执行 compact
→ 同一个宿主重试
→ 同一个宿主发送最终模型请求
```

这降低了 assembly 与 raw transcript 不一致、session rotation 与 prompt lock 竞争、插件数据库与宿主 transcript 对账等风险。

它的代价是：

- 主要是一份压缩摘要，不是可追溯 DAG；
- 被摘要掉的细节不一定能自动恢复；
- 跨会话深层回忆能力不如 LCM；
- 摘要质量差时，同样可能遗失关键上下文。

因此，legacy 是“可靠、简单的默认压缩器”，不是“功能上全面胜过 LCM”。

## 6. OpenClaw 原生如何处理超长工具输出

OpenClaw 的工具结果限制面向标准 `toolResult` 消息，而不是只针对某几个具体工具。内置工具、MCP 和自定义工具只要最终被标准化成文本型 `toolResult`，通常都能进入这条限制链路。

当前配置入口是：

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        toolResultMaxChars: 12000
      }
    }
  }
}
```

未显式配置时，源码会按模型窗口选择自动上限；当前实现中可见 16K、32K、64K 字符档位。截断器优先保留头部；当尾部出现 error、exception、traceback、fatal、exit code、summary、result 等诊断信号时，会生成“头部 + 省略标记 + 尾部”。

官方源码：[tool-result-limits.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-result-limits.ts)、[tool-result-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/embedded-agent-runner/tool-result-truncation.ts)

但要特别注意：**OpenClaw 的原生截断不等于 spill。**它主要缩小模型可见结果，并不会自动为所有结果保存全文并返回可读取路径。若需要可恢复性，最好让工具生产者主动：

```text
保存全文到受控文件/对象存储
+ 返回安全路径或 artifact id
+ 返回头部/尾部预览
+ 给出 grep、范围读取或分页提示
```

OpenClaw 文档还明确说明：旧工具结果的内存裁剪不因切换 Context Engine 而停止。因此启用 LCM 不等于自动剥夺 OpenClaw 的 tool-result trimming；历史事故日志中 LCM 活跃时仍有多条工具结果被截断，也证明该路径至少在相应运行模式下仍执行。

参考：[Context Engine 与 compaction/memory 的关系](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context-engine.md#relationship-to-compaction-and-memory)

## 7. Context Mode 的正确定位

Context Mode 最擅长的是在数据进入主上下文前改变使用方式：

```text
不要 read 整个 huge.log
→ 在沙箱/子进程中 grep、解析、统计
→ 只把相关片段和结论返回模型
```

这对大日志、数据文件和 API 输出非常有效。但其当前 OpenClaw Context Engine 适配中，`assemble()` 原样返回消息，`compact()` 返回 `compacted: false`。OpenClaw 官方契约又明确说明：`ownsCompaction: false` 不会自动回退 legacy；活动引擎的 no-op `compact()` 会破坏正常 `/compact` 和 overflow recovery 路径。

源码与契约：

- [Context Mode OpenClaw plugin](https://github.com/mksglu/context-mode/blob/main/src/adapters/openclaw/plugin.ts)
- [OpenClaw ownsCompaction 契约](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context-engine.md#ownscompaction)

因此当前更合理的组合是：

```text
主 Context Engine：OpenClaw legacy 或经过验证的 LCM
辅助工具层：Context Mode
自研工具：自身实现分页、筛选和 spill
```

Context Mode 对未知自定义工具不会自动理解其语义；自研工具若要被严格治理，应在 Context Mode 路由中增加适配，或直接在工具实现中限制返回大小。

## 8. Codex 如何管理上下文

Codex 的关键特点不是“永远不会爆仓”，而是上下文管理由同一套 agent harness 端到端协调。

### 8.1 指令按层加载，而不是反复读取为普通工具输出

Codex 会从全局目录、仓库根目录到当前工作目录发现 `AGENTS.md` / `AGENTS.override.md`，按层合并并限制总大小。项目指令属于启动上下文的一部分，不需要先由普通 `read` 产生一个可能被外置的匿名 file id。

官方资料：[Codex AGENTS.md](https://developers.openai.com/docs/agent-configuration/agents-md)

### 8.2 工具输出有独立预算

Codex 配置公开了 `tool_output_token_limit`，用于限制每条工具输出存入会话的 token 数。OpenAI 的 Codex 模型指导还建议工具响应以约 10K tokens 为上限，超限时保留一半头部、一半尾部，并在中间显示截断标记。

官方资料：

- [Codex 配置示例：`tool_output_token_limit`](https://developers.openai.com/docs/config-file/config-sample)
- [Codex 模型指导：Tool Response Truncation](https://developers.openai.com/api/docs/guides/latest-model)

这条原则的重点不是某个固定数字，而是：**单条工具结果在写入会话前就受到明确限制，且头尾策略与模型训练分布一致。**

### 8.3 压缩是 harness 与模型/API 的一等能力

Codex App Server 提供 `thread/compact/start`，并通过 `contextCompaction` item 暴露压缩生命周期。Responses API 还支持 `/responses/compact` 或服务端 `context_management.compact_threshold`，返回可在后续请求继续使用的 opaque compaction item。

官方资料：

- [Codex App Server：thread compact](https://developers.openai.com/docs/app-server)
- [OpenAI Responses Compaction](https://developers.openai.com/api/docs/guides/compaction)
- [Compact conversation API](https://developers.openai.com/api/reference/resources/responses/methods/compact)

Codex 配置还区分模型上下文窗口和自动压缩触发上限，例如 `model_context_window`、`model_auto_compact_token_limit`。这使工具结果上限、压缩触发和模型窗口属于同一运行时的预算体系。

### 8.4 子 Agent 隔离上下文

子 Agent 使用独立上下文执行任务，再把精炼结果交还主任务。大量探索性读文件、搜索和命令输出不必全部堆进主对话。

官方资料：[Codex Subagents](https://developers.openai.com/docs/agent-configuration/subagents)

### 8.5 Codex 为什么体感更稳定

主要原因是：

1. 工具输出有独立上限；
2. 指令文件有确定的加载通道；
3. 压缩属于同一运行时和 API 的正式生命周期；
4. 子任务可隔离到新上下文；
5. 工作区文件和 Git 状态可以在压缩后重新读取，不要求所有细节永远留在 prompt；
6. 最终请求和压缩状态由同一个产品栈协调。

这不代表 Codex 绝不会遇到单轮超大输入、工具返回过大或压缩丢失细节；它只是减少了“插件 assembly 与宿主最终 prompt 不是同一对象”的集成风险。

## 9. Claude Code 如何管理上下文

Claude Code 同样不是依靠单一摘要机制，而是组合了自动压缩、输出外置、按需加载和上下文隔离。

### 9.1 自动压缩与手动压缩

Claude Code 在上下文接近容量时自动总结并继续，也支持 `/compact` 主动压缩。公开环境变量允许控制自动压缩：

- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`：调整自动压缩百分比，文档说明默认大约在 95%；
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW`：为自动压缩指定计算窗口；
- `DISABLE_AUTO_COMPACT`：只关闭自动压缩，仍保留手动 `/compact`；
- `DISABLE_COMPACT`：关闭自动和手动压缩。

官方资料：[Claude Code 环境变量](https://code.claude.com/docs/en/env-vars)、[Claude Code commands](https://code.claude.com/docs/en/commands)

### 9.2 大 shell 输出保存全文并返回路径

Claude Code 的 `BASH_MAX_OUTPUT_LENGTH` 明确规定：当 Bash 输出超过阈值时，**完整输出写入文件，Claude 收到路径和短预览**。这正是 LCM 大文件设计目前值得借鉴的地方：既控制当前 prompt，又保留一个模型能继续按范围读取的稳定 locator。

同一文档还提供：

- `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS`：限制文件读取的默认输出；
- `MAX_MCP_OUTPUT_TOKENS`：限制 MCP 响应，默认 25,000 tokens，并在超过 10,000 tokens 时警告；
- `TASK_MAX_OUTPUT_LENGTH`：限制子 Agent 输出，截断时把全文保存到磁盘并在响应中包含路径。

官方资料：[Claude Code 环境变量](https://code.claude.com/docs/en/env-vars)

### 9.3 指令与知识按用途分层

Claude Code 将上下文成本拆分：

- `CLAUDE.md`：启动时加载项目规则；
- Skills：启动时只加载描述，使用时再加载正文；
- MCP：工具名先进入上下文，完整 schema 可以延后；
- Hooks：默认在上下文外执行，只有返回内容时才增加上下文；
- Subagents：使用独立上下文，主会话只接收结果。

官方资料：[Claude Code features overview：context cost](https://code.claude.com/docs/en/features-overview)、[Claude Code memory](https://code.claude.com/docs/en/memory)

### 9.4 文件系统充当可恢复状态

Anthropic 对长任务的官方建议包含：在上下文切换前把进度和状态保存到 memory/文件；新窗口可以检查工作目录、进度文件、测试和 Git 历史后继续。也就是说，Claude Code 不要求把所有执行细节永久留在模型窗口中。

官方资料：[Anthropic prompting best practices：multi-window workflows](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-templates-and-variables)

### 9.5 Claude Code 为什么体感更稳定

```text
大输出先外置并返回真实路径
+ Read/MCP/Subagent 各有输出上限
+ CLAUDE.md/Skills/MCP schema 按层按需加载
+ 自动压缩由同一客户端管理
+ 子 Agent 隔离探索噪声
+ 文件与 Git 可在新窗口重新发现
```

它同样可能失败，例如单次用户输入超过窗口、第三方 MCP 不遵守约束、hook 把大量正文重新注入、代理网关报告错误窗口、压缩服务调用失败等。正确结论是“防线更完整且集成边界更少”，不是“永不爆仓”。

## 10. 三种方案的核心差异

| 维度 | OpenClaw legacy | Lossless Claw | Codex / Claude Code |
|---|---|---|---|
| 压缩控制者 | OpenClaw 自己 | LCM 插件，OpenClaw 负责最终运行 | 产品内置 harness |
| 历史保存 | OpenClaw session | SQLite 原始消息 + DAG | 产品 session + 工作区/线程状态 |
| 压缩结构 | 单摘要 + 最近消息 | 多层摘要 DAG + fresh tail | 内置 compaction item/summary |
| 压缩后细节找回 | 较弱 | 强，可 grep/describe/expand | 主要依赖重读文件、线程状态和工具 |
| 大工具输出 | 宿主截断，默认不保证全文路径 | 可外置 file id，路径/作用域体验仍在改进 | 输出限额；Claude Code 明确支持全文落盘 + 路径 |
| 子任务上下文隔离 | 取决于 OpenClaw | 需要正确处理 conversation scope | 产品内置独立子 Agent 上下文 |
| 集成故障面 | 最小 | 最大 | 内部统一，外部用户较少接触契约 |
| 可审计长期记忆 | 一般 | 最强 | 更偏任务连续性，不等同于 LCM DAG |

## 11. 对当前项目的推荐架构

不建议用一句“LCM 一定比 Context Mode 好”或“直接去掉 LCM”结束选型。建议保留两条经过验证的部署档位。

### 11.1 稳定优先档

适用于客户明确要求主流程不能因上下文插件中断：

```text
OpenClaw legacy 负责 context assembly / compact / retry
+ OpenClaw toolResultMaxChars
+ 自研工具源头分页、筛选和 spill
+ 全文保存到安全路径/对象存储
+ 头部 + 尾部 + 省略标记 + retrieval hint
+ Context Mode 可作为辅助工具层，但不占主 Context Engine 槽位
```

这条路线牺牲 LCM 的 DAG 回溯能力，换取更少的运行时状态和集成边界。

### 11.2 长期记忆优先档

适用于确实需要跨长会话可追溯记忆：

```text
经过版本验证的 OpenClaw + LCM
+ 工具输出在进入 fresh tail 前先 spill
+ 对 assembly、system、tools、final prompt 分层观测
+ summarizer 限流/超时/降级测试
+ 子 Agent scope 回归测试
+ instruction file 不按普通匿名大结果外置
+ 固定兼容版本并做长流程 canary
```

LCM 不应独自承担“大工具输出治理”。超长结果应首先在工具或宿主边界被控制，否则 fresh tail 会在摘要生效前先把窗口撑爆。

## 12. 对 LCM 的改进方向

### P0：正确性与不中断

- 统一 assembly budget 与完整 provider-bound prompt budget 的契约；
- 明确 canonical messages，避免 raw/preassembly 与 assembly 双重权威；
- 失败时优先走“已有摘要 + 尾窗”的安全 assembly；
- compact 无法继续降低 token 时，不能重复返回形式成功；
- session rewrite/rotation 必须通过 OpenClaw 提供的锁和 transcript API；
- 插件异常时保证 turn-local fallback 到 legacy 能继续回复。

### P1：工具输出

- 全文外置；
- 返回经过验证的安全路径或稳定 artifact locator；
- 默认提供 deterministic head/tail，而不是只返回头部；
- 在预览内加入原始大小、hash、MIME、截断比例和检索提示；
- 支持 line range、grep、分页和结构化字段过滤；
- `SKILL.md`、`AGENTS.md`、`CLAUDE.md` 等指令文件保留原始路径和特殊语义。

### P1：观测

每轮至少区分：

```text
raw transcript tokens
LCM stored tokens
assembly tokens
system prompt tokens
tool schema tokens
pending current-turn tokens
final provider-bound prompt tokens
promptAuthority
compaction outcome / fallback reason
```

不能再用两个不同统计对象相减后标成“节省 tokens”。

## 13. 本次已推动的上游改进

- [PR #1176：暴露经过验证的大工具输出路径](https://github.com/Martian-Engineering/lossless-claw/pull/1176)
- [PR #1189：delegated session 可访问自身 conversation scope](https://github.com/Martian-Engineering/lossless-claw/pull/1189)
- [PR #1190：保护指令文件读取，不按普通大结果破坏语义](https://github.com/Martian-Engineering/lossless-claw/pull/1190)

这些 PR 分别解决可恢复定位、子会话权限和指令文件语义问题。它们能改善 LCM 的工具输出体验，但不能替代预算统一、canonical assembly 和宿主级最终 prompt 校验。

## 14. 最终判断

LCM 的价值是真实的：它提供可追溯、可检索、可展开的长期上下文，不只是把旧聊天压成一份摘要。它之所以复杂，是因为目标比 legacy compaction 更大；其中一部分是必要复杂度，另一部分则来自与 OpenClaw 重叠的状态和控制权。

Codex 和 Claude Code 体感更稳定，并不是因为它们没有压缩，也不是因为模型窗口永远够用，而是因为它们把以下能力做成同一 harness 的内建机制：

```text
工具输出限制/外置
+ 指令按层或按需加载
+ 压缩触发
+ 最终请求预算
+ 子任务隔离
+ 文件系统恢复
```

对当前客户，建议明确两种承诺：

1. **若“流程不中断”优先：选择 OpenClaw legacy，并把 spill、头尾截断、分页和文件路径返回做好。**
2. **若“长期记忆可追溯”优先：保留 LCM，但必须接受更高的集成验证成本，并针对你们真实长流程建立固定版本与回归矩阵。**

这比用 star 数或单次 demo 判断上下文引擎优劣更可靠。

## 15. 参考资料

### OpenClaw / Lossless Claw / Context Mode

- [OpenClaw Context Engine](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context-engine.md)
- [OpenClaw Compaction](https://github.com/openclaw/openclaw/blob/main/docs/concepts/compaction.md)
- [OpenClaw Context](https://github.com/openclaw/openclaw/blob/main/docs/concepts/context.md)
- [OpenClaw tool-result-limits.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-result-limits.ts)
- [OpenClaw tool-result-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/embedded-agent-runner/tool-result-truncation.ts)
- [Lossless Claw README](https://github.com/Martian-Engineering/lossless-claw/blob/main/README.md)
- [Lossless Claw Architecture](https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/architecture.md)
- [Lossless Claw Configuration](https://github.com/Martian-Engineering/lossless-claw/blob/main/docs/configuration.md)
- [Context Mode OpenClaw plugin](https://github.com/mksglu/context-mode/blob/main/src/adapters/openclaw/plugin.ts)
- [Context Mode platform support](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md#openclaw)

### Codex / OpenAI

- [OpenAI Compaction Guide](https://developers.openai.com/api/docs/guides/compaction)
- [OpenAI Compact conversation API](https://developers.openai.com/api/reference/resources/responses/methods/compact)
- [Codex App Server](https://developers.openai.com/docs/app-server)
- [Codex configuration sample](https://developers.openai.com/docs/config-file/config-sample)
- [Codex AGENTS.md](https://developers.openai.com/docs/agent-configuration/agents-md)
- [Codex Subagents](https://developers.openai.com/docs/agent-configuration/subagents)
- [OpenAI model guidance：Tool Response Truncation](https://developers.openai.com/api/docs/guides/latest-model)

### Claude Code / Anthropic

- [Claude Code environment variables](https://code.claude.com/docs/en/env-vars)
- [Claude Code commands](https://code.claude.com/docs/en/commands)
- [Claude Code features overview](https://code.claude.com/docs/en/features-overview)
- [Claude Code memory](https://code.claude.com/docs/en/memory)
- [Claude Code hooks](https://code.claude.com/docs/en/hooks)
- [Anthropic prompting best practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-templates-and-variables)
