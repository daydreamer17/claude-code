# 前言 <del>（笔者的碎碎念）</del>
> 2026年3月31日,因为Anthropic公司的一次上传失误,泄露了当前市面上最强的coding agent-Claude Code CLI的源码,一时间引起AI 开发圈和工业界的"狂欢",正好笔者最近正在学习Agent开发和LLM Systems相关的知识,这样一个工业级别的顶级Agent应用无疑是一份学习多agent编排、上下文工程、skills和mcp集成等在agent开发中重要部分的最好材料,而对于如此的庞大代码量,光靠人工来一行行理解无疑是不现实的,所以怀着激动和兴奋的心情,我决定使用Codex和Claude Code这两个工具,为我逐步拆分分析这个项目的技术细节并记录在这个仓库当中,并考虑后续基于这个项目来进行跑通源码和二次开发

# Claude Code CLI 源码专题分析报告

> 本文按 11 个专题系统拆解 `src/` 目录中的 Claude Code CLI 源码快照，重点不是“罗列目录”，而是解释这套 coding agent CLI 为什么会长成现在的样子，它的主循环怎么运转，工具、权限、上下文、记忆、技能、多 Agent、Hooks 又是如何彼此耦合的。

## 阅读说明

- 本仓库根目录只有 `src/` 源码快照与说明文件，没有 `package.json`、锁文件、完整构建脚本，因此文中对技术栈和运行方式的判断，以源码导入、注释、控制流为依据。
- 文中所有“源码事实”都来自当前快照静态阅读；“最小必要组件”一节属于基于现有架构抽象出的精简模型，会明确标注为架构归纳，而不是仓库现成的发布裁剪方案。
- 当前 `src/` 快照最大文件是 `src/screens/REPL.tsx`，其次是 `src/main.tsx`。这两个文件已经足够说明：这个项目的复杂度核心不是“调用模型 API”，而是“围绕 agent 运行时构建出的完整终端产品”。

## 专题目录

1. 概述  
2. 系统主循环  
3. 上下文工程  
4. 工具系统  
5. 代码编辑策略  
6. 权限与安全  
7. 用户体验设计  
8. 最小必要组件  
9. Hooks 与可扩展性  
10. 多 Agent 架构  
11. 记忆与技能系统  

---

<details>
<summary><strong>1. 概述</strong></summary>

如果只用一句话概括这套源码：

> **Claude Code CLI 的本质不是“聊天模型 + 终端壳”，而是“以消息主循环为内核、以工具调用为执行面、以 REPL 为前台、以 Bridge/MCP/Remote 为扩展总线”的 agent runtime。**

它至少同时承担了五种角色：

- 一个本地交互式开发终端，负责 prompt 输入、消息流展示、权限确认、任务导航与恢复。
- 一个 LLM agent runtime，负责 system prompt、上下文拼装、tool use、tool result 回灌、压缩、预算与重试。
- 一个工具编排器，统一接入 Bash、文件读写、搜索、MCP、LSP、任务、技能、多 Agent 等能力。
- 一个远程会话客户端，支持 remote、teleport、assistant viewer、remote-control/bridge 等模式。
- 一个企业化客户端，内置 trust boundary、组织策略、远程托管设置、遥测、OAuth、权限模式与安全审计。

从分层看，这个项目大致可以分成六层：

- 入口层：`src/main.tsx`
- 交互层：`src/screens/REPL.tsx`、`src/components/`、`src/hooks/`、`src/ink/`
- 会话内核层：`src/QueryEngine.ts`、`src/query.ts`
- 能力层：`src/Tool.ts`、`src/tools.ts`、`src/commands.ts`
- 集成层：`src/services/`、`src/bridge/`、`src/remote/`、`src/server/`
- 基础设施层：`src/bootstrap/state.ts`、`src/state/`、`src/history.ts`、`src/utils/`、`src/migrations/`

这套结构里最重要的设计判断是：**所有外部形态最终都回收到统一的消息与工具协议上。**

不管你是：

- 在本地 REPL 里交互
- 用 `--print` 跑 headless/SDK
- 用 `--remote` 创建远程会话
- 用 `assistant` 做 viewer
- 用 `remote-control` 做桥接

最终都绕不开以下几个核心对象：

- `Message`
- `Tool`
- `Command`
- `ToolUseContext`
- `PermissionDecision`
- `Session/App State`

相关源码：

- `src/main.tsx`
- `src/screens/REPL.tsx`
- `src/QueryEngine.ts`
- `src/query.ts`
- `src/Tool.ts`
- `src/tools.ts`
- `src/commands.ts`

</details>

---

<details>
<summary><strong>2. 系统主循环</strong></summary>

系统主循环是理解整份代码的第一把钥匙。Claude Code 的运行不是“读输入 -> 调模型 -> 打印输出”这么简单，而是一条多阶段的状态机。

### 2.1 启动与预热

`src/main.tsx` 在最开头就做了明显的启动并行化：

- 先执行 `profileCheckpoint('main_tsx_entry')`
- 提前触发 `startMdmRawRead()`
- 提前触发 `startKeychainPrefetch()`

这说明作者把冷启动耗时拆成了“可并行的外部 I/O”和“后续模块求值”两段，尽量让系统设置读取、Keychain 读取与后面的 import 开销重叠。

`src/entrypoints/init.ts` 则做第二轮初始化：

- 启用配置系统
- 先应用“安全环境变量”
- 提前配置 CA 证书、mTLS、代理
- 预连接 Anthropic API
- 之后再按需初始化 OpenTelemetry、1P logging、GrowthBook
- 注册 LSP、team cleanup、scratchpad 等清理逻辑

这类写法非常像一套成熟客户端，而不是一次性脚本。

### 2.2 模式分流

`main.tsx` 是真正的总调度器，不只是 Commander 参数注册器。它在 runtime 上支持多种会话形态：

- 本地交互式 REPL
- 非交互 `--print`
- `--remote`
- `--teleport`
- `ssh`
- `assistant`
- `remote-control` / `rc`
- auto / plan / bypassPermissions 等权限模式

也就是说，`main.tsx` 实际做的是“运行时容器切换”。CLI 参数不是在切换某个小功能，而是在切换整条会话执行路径。

### 2.3 Setup 阶段

在进入主会话前，`src/interactiveHelpers.tsx` 会插入一段 setup 流程：

- onboarding
- trust dialog
- MCP server approval
- `CLAUDE.md` 外部 include 警告
- API key 审批
- bypass permission 风险确认
- auto mode opt-in
- channel / dev-channel 相关确认

这段 setup 很重要，因为它说明 Claude Code 把“工作区信任”单独建模成了系统边界，而不是把一切都塞进工具权限判断里。

### 2.4 交互主屏与消息驱动

如果是交互模式，主界面最终落到 `src/screens/REPL.tsx`。这个文件是全仓库最大的文件，说明它承担的不是单纯视图层，而是前台运行时协调器：

- prompt 输入与补全
- 流式消息展示
- tool progress / permission dialog
- remote / bridge / ssh / direct connect
- task、teammate、notification、survey、history、restore

REPL 收到用户输入后，并不是直接发给模型，而是会先进入一整条预处理链路：命令、Hook、上下文、工具可用性、状态刷新、队列处理、UI 同步。

### 2.5 QueryEngine 与 query.ts

会话真正的核心是 `src/QueryEngine.ts` 与 `src/query.ts`。

它们的职责分工很清楚：

- `QueryEngine` 管“跨 turn 的会话状态”
- `query.ts` 管“单次 turn 的状态机循环”

`QueryEngine.submitMessage()` 负责：

- 维护 `mutableMessages`
- 拼装 `ProcessUserInputContext`
- 获取 system prompt 与 user/system context
- 根据配置准备 tools、commands、MCP clients、thinkingConfig、budget 等
- 调用 `query()` 进入单回合主循环

`query.ts` 负责：

- 标准化消息
- 调用 `services/api/claude.ts` 发起流式模型请求
- 解析 streaming event
- 识别 `tool_use`
- 调度工具执行
- 把 `tool_result` 再喂回模型
- 处理 auto compact、reactive compact、microcompact、task budget、thinking block 规则、stop hooks 等

### 2.6 工具调用回路

工具调用不是附带逻辑，而是主循环的一部分。标准回路是：

```text
用户输入
-> QueryEngine.submitMessage()
-> query.ts 构造 API 请求
-> services/api/claude.ts 流式返回 assistant blocks
-> 出现 tool_use
-> services/tools/toolOrchestration.ts 调度工具
-> tools/* 执行
-> 生成 tool_result
-> 回灌到消息列表
-> query.ts 继续下一轮 assistant 生成
```

`toolOrchestration.ts` 还有一个很关键的设计：

- 工具先按 `tool.isConcurrencySafe(input)` 分批
- 并发安全的读类工具可以批量并发
- 非并发安全的写类工具严格串行
- 并发批次结束后再顺序回放 context modifier

这避免了最常见的 agent 失控问题：多个工具同时写上下文，导致状态与结果顺序错乱。

### 2.7 单轮 Turn 的细粒度时序

如果把 `query.ts` 与 `services/api/claude.ts` 合在一起看，一次 turn 实际更接近下面这张时序表：

| 阶段 | 主要代码位置 | 关键动作 |
| --- | --- | --- |
| 请求构造 | `QueryEngine.ts`、`query.ts`、`services/api/claude.ts` | 组装 system/user context、tools、betas、`cache_control`、thinking、task budget |
| 流式采样 | `services/api/claude.ts` | 解析 streaming event，持续产出 assistant block、`tool_use`、`server_tool_use` |
| 可恢复错误暂缓 | `query.ts` | `prompt_too_long`、media size、`max_output_tokens` 先 withheld，不立即抛给 UI |
| 工具执行 | `toolOrchestration.ts` | 并发安全工具并发执行，写类工具串行执行，结果标准化成 `tool_result` |
| 恢复重试 | `query.ts` | 先尝试 collapse drain，再试 reactive compact，再试 output token recovery |
| 收尾 | `query.ts`、post/stop hooks | 运行 post-sampling hooks、补齐中断结果、决定是否写出 stop failure |

这张表能说明一个关键事实：Claude Code 的 turn 并不是“一次 API 调用”，而是“请求构造 + 流式解释 + 工具回路 + 恢复策略 + 收尾钩子”的复合状态机。

### 2.8 会话收尾与恢复

系统不是“一轮即焚”的。它有很强的恢复能力：

- transcript 落盘
- session metadata 维护
- file history 快照
- cost / attribution 状态保存
- `resume`、`continue`、`fork-session`、`rewind-files`
- teleport / remote resume

这说明主循环从设计一开始就是“长寿命会话”，不是一次性请求。

相关源码：

- `src/main.tsx`
- `src/interactiveHelpers.tsx`
- `src/screens/REPL.tsx`
- `src/QueryEngine.ts`
- `src/query.ts`
- `src/services/api/claude.ts`
- `src/services/tools/toolOrchestration.ts`

</details>

---

<details>
<summary><strong>3. 上下文工程</strong></summary>

Claude Code 的上下文工程不是“把系统提示词写长一点”，而是一整套上下文采集、拼装、缓存、裁剪、压缩与预算控制机制。

### 3.1 System Context 与 User Context

`src/context.ts` 把上下文分成两类：

- `getSystemContext()`
- `getUserContext()`

`getSystemContext()` 主要负责环境性上下文，例如：

- git status 快照
- 当前 branch
- main branch
- recent commits
- 可选的 cache breaker 注入

`getUserContext()` 则更偏用户与工作目录相关上下文，例如：

- 自动发现的 `CLAUDE.md`
- `currentDate`
- 额外目录中的 context 文件

这意味着 Claude Code 从一开始就把“环境上下文”和“用户工作上下文”区分开，而不是混成一坨系统提示词。

### 3.2 `CLAUDE.md` 与项目上下文注入

项目级上下文的一个核心来源是 `CLAUDE.md`。  
其特点不是简单“读文件”，而是带规则的装配：

- 会自动发现工作目录内相关文件
- `--add-dir` 可以显式附加更多目录
- `--bare` 会关闭自动发现，但保留显式提供的目录
- 外部 include 会进入单独的 warning / approval 流程

这说明项目上下文被当作“受信任配置的一部分”，而不是普通 prompt 附件。

### 3.3 记忆上下文也属于系统提示词工程

`src/memdir/memdir.ts` 与 `src/constants/prompts.ts` 展示出 memory prompt 是如何进入系统提示词的：

- `loadMemoryPrompt()` 根据 auto memory / team memory / assistant daily log 模式返回不同提示段
- `constants/prompts.ts` 通过 `systemPromptSection('memory', ...)` 把 memory section 接入 system prompt
- `QueryEngine.ts` 在某些自定义 prompt 场景下还会单独注入 memory mechanics prompt

也就是说，memory 不只是文件系统上的持久化，它还是系统提示词的一部分。

### 3.4 Prompt Section Cache 与缓存友好设计

从 `constants/prompts.ts`、`services/api/claude.ts` 与 `bootstrap/state.ts` 能看出，Claude Code 对 prompt cache 非常敏感：

- system prompt 被拆成 section
- 某些 section 走缓存，某些 section 刻意不缓存
- 日期变化等会尽量通过尾部提醒而不是破坏前缀缓存
- `cache_control`、`cache_reference`、prompt cache 1h TTL、header latch 都有专门处理

这背后的思想是：**prompt 不是纯文本，它是昂贵缓存资源。**

### 3.5 上下文压缩不是单点功能，而是多层机制叠加

`query.ts` 和 `services/compact/` 中至少有几类压缩机制：

- auto compact
- reactive compact
- microcompact
- cached microcompact
- snip
- token budget continuation

尤其 `microCompact.ts` 值得注意：

- 它不是简单删历史消息
- 它会针对 tool result 做更细粒度的清理
- 会维护 `cache_edits` 状态，尽量兼容 prompt cache
- 对 images/documents/thinking/tool_use 都做不同 token 估算逻辑

这已经属于“上下文工程系统化优化”，而不是零散的截断逻辑。

### 3.6 Thinking、Tool Search 与上下文预算耦合

在 Claude Code 里，thinking 不是一个单独选项，它与上下文预算、缓存、压缩强绑定：

- `query.ts` 对 thinking block 有严格规则约束
- `services/api/claude.ts` 会根据模型能力和配置附加 thinking 相关 beta header
- Tool Search / deferred tool loading 会影响初始 prompt 大小
- advisor、structured output、task budget 都会进一步影响请求体结构

这说明上下文工程在这里已经不是“写 prompt 的技巧”，而是一整套 runtime policy。

### 3.7 可恢复错误为什么要先 withheld

`query.ts` 有一个非常工程化的细节：某些错误在 streaming 阶段并不会立刻显示给用户，而是先“暂缓暴露”。典型包括：

- `prompt_too_long`
- media size 相关错误
- `max_output_tokens`

它们会先被放进 `assistantMessages`，但不立刻 `yield` 到界面上。随后系统会按顺序尝试：

- `contextCollapse.recoverFromOverflow()`
- `reactiveCompact.tryReactiveCompact()`
- `max_output_tokens` 提升重试
- 注入 “Resume directly” 的 meta message 做续写恢复

这样做的好处是：

- 用户不会先看到一个“其实可恢复”的错误
- stop hooks 不会在错误消息上继续注入额外上下文，避免死循环
- 压缩和恢复逻辑可以在同一 turn 内完成，而不用把失败暴露成一次新的用户交互

这是非常典型的 agent runtime 设计，而不是普通 API client 设计。

### 3.8 历史系统其实也是上下文减载层

`src/history.ts` 不只是“命令历史”，它其实承担了一部分上下文卸载职责：

- 全局历史写到 `history.jsonl`
- 读取时只抽取当前 project
- 当前 session 的记录会优先返回，避免并发会话互相污染 Up-arrow 历史
- pasted text / image 在历史里只保留 `[Pasted text #N]`、`[Image #N]` 引用

其中最值得注意的是 pasted content 的处理：

- 小文本直接 inline 存到历史项里
- 大文本先算 hash，再异步写到 paste store
- 真正恢复历史项时再懒加载全文

这解决了两个问题：

- 历史文件不会因为一次粘贴几百行代码而快速膨胀
- 终端交互层还能保留“这段内容曾经被贴过”的用户心智模型

也就是说，历史系统同时服务于：

- 交互体验
- 上下文治理
- 持久化恢复

### 3.9 文件路径还能触发动态技能上下文

`FileWriteTool`、`FileEditTool` 都会在编辑路径命中时触发：

- `discoverSkillDirsForPaths()`
- `addSkillDirectories()`
- `activateConditionalSkillsForPaths()`

这意味着“上下文”不仅来自文本，还来自**当前被编辑的文件路径**，并能够反向激活技能系统。

相关源码：

- `src/context.ts`
- `src/constants/prompts.ts`
- `src/QueryEngine.ts`
- `src/query.ts`
- `src/services/api/claude.ts`
- `src/services/compact/microCompact.ts`
- `src/memdir/memdir.ts`
- `src/history.ts`
- `src/utils/claudemd.ts`

</details>

---

<details>
<summary><strong>4. 工具系统</strong></summary>

工具系统是 Claude Code 的核心执行面。模型之所以能“像 agent 一样工作”，不是因为模型自己会操作文件，而是因为这套工具协议足够完整。

### 4.1 工具协议定义

`src/Tool.ts` 定义了统一工具接口。一个成熟工具至少具备这些维度：

- `name`
- `inputSchema`
- `description`
- `prompt`
- `call`
- `checkPermissions`
- `validateInput`
- `isReadOnly`
- `isDestructive`
- `isConcurrencySafe`
- `maxResultSizeChars`
- `shouldDefer`
- `alwaysLoad`

这说明工具在 Claude Code 里不是“函数调用封装”，而是具备：

- 输入约束
- 展示协议
- 权限语义
- 并发语义
- 上下文预算语义

的完整执行对象。

### 4.2 `buildTool()`：默认值体现了安全立场

`buildTool()` 会给工具补默认值，而且默认策略偏保守：

- `isConcurrencySafe` 默认 `false`
- `isReadOnly` 默认 `false`
- `isDestructive` 默认 `false`
- `checkPermissions` 默认为 allow，但仍会走通用权限系统

这类默认值的意义在于：如果一个工具实现没有明确声明自己的语义，系统宁愿把它当成“潜在写操作”或“不可并发”处理。

### 4.3 工具注册中心

`src/tools.ts` 是整个工具池的装配中心。它会按环境、feature gate、用户类型动态挂载工具：

- Bash / PowerShell
- Read / Edit / Write / NotebookEdit
- Grep / Glob / WebFetch / WebSearch / WebBrowser
- Skill / Agent / Team / SendMessage
- Todo / Task / Plan / Worktree
- MCP Resource 相关工具
- 各类实验性或内部工具

这意味着系统里“能让模型看到哪些工具”是动态可配置的，不是编译时写死列表。

### 4.4 工具分层

大体可把工具分成五类：

- 文件与本地执行类：`BashTool`、`FileReadTool`、`FileEditTool`、`FileWriteTool`
- 搜索与外部信息类：`GlobTool`、`GrepTool`、`WebFetchTool`、`WebSearchTool`
- 运行时控制类：`TodoWriteTool`、`Task*Tool`、`EnterPlanModeTool`、`ExitPlanModeTool`
- 扩展接入类：`MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool`、`LSPTool`
- Agent 编排类：`AgentTool`、`TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool`

其中最核心的不是单个工具，而是这几个共同构成的“工作闭环”：

- 读取上下文
- 运行命令
- 修改文件
- 维护任务清单
- 调起子 Agent

### 4.5 Tool Search 与 Deferred Loading

Claude Code 没有把所有工具一股脑塞进 prompt。  
`Tool.ts` 和 `services/api/claude.ts` 里都能看到：

- 工具可以 `shouldDefer`
- 也可以 `alwaysLoad`
- Tool Search 打开后，部分工具先延迟加载

这个设计解决的是典型问题：工具越多，system prompt 越膨胀，模型越容易迷路。

### 4.6 MCP 工具与本地工具是同一协议面

MCP 并不是“插件外设”，而是直接映射为工具：

- 有统一 tool name
- 有 input schema
- 有 tool result
- 可以参与权限与显示系统

这也是 Claude Code 能够把 MCP 真正纳入 agent 主循环的前提。

### 4.7 模型 API 协议层并不是薄封装

`src/services/api/claude.ts` 很值得单独拿出来看，因为这里不是“调 SDK”那么简单，而是在做一层完整的请求协议编排。比较关键的细节有：

- 根据 query 类型合并 beta headers，例如 advisor、effort、task budgets、structured outputs、fast mode、AFK、cache editing、context management
- Tool Search 打开时，只把已发现的 deferred tools 真正发给模型，避免一次性声明过多工具
- 在模型切换或恢复场景下，对消息做二次清洗，例如去掉不兼容的 tool-search 字段、修补 `tool_use` / `tool_result` 配对
- 对超过 API 限制的 media 项做截断与裁剪，而不是把难以恢复的错误直接抛给用户

更重要的是，它对 prompt cache 非常敏感：

- `cache_control` 和 `cache_reference` 是一等请求字段
- 某些 beta header 会“sticky latch”在 session 内保持稳定
- 这样中途切换 fast mode、AFK、cache editing 等特性时，不会反复打爆服务器侧 prompt cache

这说明它不是一个“模型 API 适配层”，而是一个“面向长会话 agent 的请求编排器”。

### 4.8 MCP 传输矩阵体现了集成层成熟度

`src/services/mcp/client.ts` 展示出的 MCP 能力已经远超“起个子进程读写 stdio”。它至少统一处理了这些传输：

- `stdio`
- `sse`
- `http` / streamable HTTP
- `ws`
- `sse-ide`
- `ws-ide`
- `claudeai-proxy`
- linked in-process transport

其中还有一些非常“工业化”的实现细节：

- SSE 的 `eventSourceInit.fetch` 不走普通超时包装，因为事件流本来就是长连接
- HTTP / SSE / proxy 都带独立 auth provider、proxy、TLS、step-up detection
- stdio transport 会提前接管 `stderr`，避免 MCP 进程错误直接污染终端 UI
- Chrome MCP / Computer Use MCP 会以内嵌 in-process server 方式运行，避免额外拉起重型子进程

连接建立后，MCP 服务器暴露出来的：

- tools
- prompts
- resources
- skill-like resources

都会被统一折叠进 Claude Code 自己的 `Tool` / `Command` / resource tool 体系。这就是为什么 MCP 在这套源码里不是外挂，而是原生总线。

相关源码：

- `src/Tool.ts`
- `src/tools.ts`
- `src/services/api/claude.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/mcp/client.ts`

</details>

---

<details>
<summary><strong>5. 代码编辑策略</strong></summary>

这一节很关键，因为 Claude Code 的“写代码”能力并不是简单的“模型吐一段 patch”，而是通过一套严格的本地编辑协议约束出来的。

### 5.1 FileEdit 与 FileWrite 是两条不同语义路径

`FileEditTool` 与 `FileWriteTool` 分工明确：

- `FileEditTool` 适合“在已有内容上做精确替换”
- `FileWriteTool` 适合“创建新文件或整体重写文件”

这两种语义拆分非常合理。很多 agent 工具把“编辑”和“覆盖写入”混在一起，会放大误改风险；Claude Code 明确将二者分离。

### 5.2 FileEdit 的核心策略：精确匹配而不是模糊改写

`FileEditTool` 的输入核心是：

- `file_path`
- `old_string`
- `new_string`
- `replace_all`

它的目标是做“可验证的局部替换”，而不是让模型自由重写文件。其验证过程包括：

- 检查 `old_string !== new_string`
- 检查目标路径是否被 deny
- 检查文件是否存在
- 对不存在文件的场景做更细分的错误提示
- 用相似路径、cwd 提示帮助模型纠错

这套机制显著降低了“模型以为自己改到了某行，其实根本没改到”的风险。

### 5.3 FileWrite 的核心策略：已有文件必须先读后写

`FileWriteTool` 的一个非常强的约束是：

- 如果文件已存在，但当前会话没有读过它，就拒绝直接写入
- 如果读过之后该文件又被修改过，也拒绝写入

对应逻辑来自：

- `readFileState`
- 文件 `mtime`
- “read-before-write” 检查

这相当于给 agent 引入了一种轻量的 optimistic concurrency control。它避免了两个危险场景：

- 模型盲写未知文件
- 模型覆盖用户或格式化器刚刚修改过的内容

### 5.4 编辑工具会和更多系统联动

代码编辑不是孤立的文件写入，它还会联动：

- `fileHistoryTrackEdit()` 记录文件历史
- `diagnosticTracker.beforeFileEdited()` 处理诊断状态
- `clearDeliveredDiagnosticsForFile()` 清空旧诊断
- `notifyVscodeFileUpdated()` 通知外部环境
- `fetchSingleFileGitDiff()` 获取差异
- `countLinesChanged()` 记录改动规模

也就是说，编辑工具已经被嵌入到“编辑后生态”里，而不是只做写盘。

### 5.5 编辑路径还能激活技能系统

`FileWriteTool` 和 `FileEditTool` 都会根据被写入路径：

- 发现 skill dir
- 加载 skill dir
- 激活 conditional skills

这说明“编辑某个目录里的文件”在 Claude Code 中本身就是一个上下文信号，会改变模型后续可调用的技能面。

### 5.6 编辑工具里的安全细节

代码里能直接看到一些很实在的安全防护：

- 对 UNC 路径避免触发 Windows SMB/NTLM 泄漏
- 对 team memory 文件写入做 secret guard
- 路径匹配与 hook 可见路径统一走绝对路径展开
- 对大文件编辑做 1 GiB 上限保护，避免 OOM

这些都不是“理论安全”，而是针对实际桌面环境和企业环境的工程化防护。

### 5.7 写入流程其实是带一致性语义的原子替换

`FileWriteTool` 的实现还有几个很容易被忽略但非常关键的细节：

- 在进入 read-modify-write 临界区前，先确保目录存在，并在需要时先做 file history backup
- 真正写入前会重新读取磁盘元数据，与 `readFileState` 对比，避免 stale write
- Windows 上如果 `mtime` 变化但内容没变，还会做内容回退比对，减少误报
- 写入时不会偷偷“继承旧文件换行风格”，而是尊重模型这次明确给出的内容和行尾

这一点很重要。很多 agent 会为了“看起来更稳”而自动保留旧行尾或随手做格式修复，但那会把“模型的显式输出”和“工具层的隐式修正”混在一起。Claude Code 这里的选择更严格：**工具负责一致性，不负责替模型重新解释文本。**

写完之后，它还会继续把变更传播到外部生态：

- 通知 LSP `didChange` / `didSave`
- 通知 VS Code 做 diff 展示
- 更新 `readFileState`
- 触发 file history 和 diagnostics 联动

`FileEditTool` 这边也有类似的精确化思路：

- 用 `findActualString()` 处理引号归一化
- 多处命中但 `replace_all=false` 时直接拒绝
- 不接受“没先读文件就直接 edit”

这些约束放在一起，才构成了真正可控的代码编辑协议。

### 5.8 这套编辑策略的总体思路

可以把 Claude Code 的编辑策略概括成一句话：

> **宁愿强制多走一步“先读、再比对、再写”，也不允许模型在无上下文、无版本感知的情况下自由覆盖文件。**

这也是它比很多“单次 patch agent”更稳定的根本原因之一。

相关源码：

- `src/tools/FileEditTool/FileEditTool.ts`
- `src/tools/FileWriteTool/FileWriteTool.ts`
- `src/utils/fileHistory.ts`
- `src/utils/fileRead.js`
- `src/utils/gitDiff.js`
- `src/services/lsp/manager.js`

</details>

---

<details>
<summary><strong>6. 权限与安全</strong></summary>

Claude Code 的权限与安全不是“出个确认弹窗”那么简单，而是多层边界叠加。

### 6.1 第一层边界：Workspace Trust

从 `interactiveHelpers.tsx` 可以看出，trust dialog 与工具权限是两套不同逻辑：

- trust 解决“这个工作区是不是可信”
- tool permission 解决“这个动作要不要允许”

这两者不能混为一谈。  
即使是 bypassPermissions 模式，也不代表可以跳过 workspace trust。

### 6.2 第二层边界：Permission Mode

源码中可以直接看到多种 permission mode：

- `default`
- `plan`
- `bypassPermissions`
- `auto`

它们不是纯 UI 开关，而会影响：

- 是否弹出权限确认
- 是否允许 classifier 自动放行
- 是否允许 plan mode 进入/退出
- 是否剥离危险权限

### 6.3 第三层边界：工具级权限

每个工具都可以实现自己的：

- `validateInput()`
- `checkPermissions()`
- `preparePermissionMatcher()`

例如文件写入类工具，会结合：

- 工具输入路径
- 全局 permission context
- allow / deny / ask 规则
- 文件系统 write permission 逻辑

这让权限系统既能统一，又能保留工具自己的语义判断。

### 6.4 `useCanUseTool()`：权限判定真正的入口

`src/hooks/useCanUseTool.tsx` 是权限决策主入口。它做的不只是弹窗，而是一条多阶段流水线：

- 先跑静态规则判断
- 如果 classifier 开启，可能先走 classifier
- 如果是 coordinator / swarm worker，还会先走自动检查
- 如果还不能决策，才进入交互式 permission flow
- 交互式 flow 又可能接本地用户、bridge 回应、channel 回应

这意味着权限系统在 Claude Code 里是一条独立的 orchestration pipeline。

### 6.5 Interactive Permission Flow：不仅 allow/deny，还能修改输入

`interactiveHandler.ts` 的设计非常成熟。它支持：

- 用户主动交互
- abort
- allow
- reject
- `recheckPermission()`
- remote/bridge 的并发竞态处理
- classifier grace period
- `updatedInput`
- `updatedPermissions`

这说明 permission prompt 并不只是“二元确认框”，而是“可协商的工具调用修改点”。

### 6.6 Remote Permission Bridge

`src/remote/remotePermissionBridge.ts` 很有代表性。它处理了一种复杂场景：

- tool use 发生在远端 CCR 容器
- 本地 CLI 没有真实 assistant message
- 本地甚至未必加载了该工具

为了解决这个问题，源码专门构造了：

- synthetic assistant message
- tool stub

这让远程权限请求依然能进入本地统一权限 UI，而不需要要求两端的工具集合完全一致。

### 6.7 企业安全能力

从初始化与服务层还能看出更多企业化安全能力：

- 组织策略限制 `policyLimits`
- 远程托管配置 `remoteManagedSettings`
- OAuth 与 token refresh
- Keychain 预取
- CA 证书与 mTLS
- upstream proxy
- trusted device

这类模块说明 Claude Code 的安全模型不是“单机用户工具”，而是默认要运行在企业受控环境中。

### 6.8 代码里的实际防御意识

除了高层权限模型，源码里还藏着不少底层防护：

- UNC path 导致的 NTLM 凭据泄漏防护
- stale file 写入检测
- deny rule 在 tool 可见性与调用时都生效
- 远程控制与本地桥接都带 session / token 约束

整体观感很明确：**安全不是附属功能，而是 runtime 的内生约束。**

### 6.9 权限决策其实是分层矩阵

如果把权限链路压缩成一个判断矩阵，大致可以这样理解：

| 层次 | 代表实现 | 解决的问题 |
| --- | --- | --- |
| 工作区信任 | `interactiveHelpers.tsx` | 当前项目是否进入可信边界 |
| 模式与策略 | permission mode、`policyLimits` | 当前 session 是否允许 auto / bypass / plan 等行为 |
| 工具前置校验 | `validateInput()`、`checkPermissions()` | 路径、参数、文件状态、危险输入是否成立 |
| 统一编排入口 | `useCanUseTool.tsx` | 静态规则、classifier、swarm、interactive flow 如何串起来 |
| 远程桥接 | `interactiveHandler.ts`、`remotePermissionBridge.ts` | 本地用户如何审批远端工具调用，甚至审批本地未加载的远端工具 |

其中最后一层尤其值得注意。`remotePermissionBridge.ts` 通过 synthetic assistant message 和 tool stub，把“远端工具调用、本地并不知道这个工具”的尴尬场景，重新纳入本地统一权限 UI。这个设计非常漂亮，因为它把“权限体验一致性”放在了“本地工具集合必须完全一致”之前。

相关源码：

- `src/interactiveHelpers.tsx`
- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`
- `src/remote/remotePermissionBridge.ts`
- `src/remote/RemoteSessionManager.ts`
- `src/utils/permissions/*`
- `src/services/policyLimits/`
- `src/services/remoteManagedSettings/`

</details>

---

<details>
<summary><strong>7. 用户体验设计</strong></summary>

Claude Code 的 UX 不是“美化命令行输出”，而是围绕长时间开发工作流做的一整套终端产品设计。

### 7.1 REPL 是主产品，不是附带界面

`src/screens/REPL.tsx` 是全仓库最大的文件，这本身就是一个信号：

- 产品主战场是交互式 REPL
- 非交互模式是能力分支，不是唯一中心

REPL 里集成了：

- message list
- prompt input
- spinner / progress
- permission dialog
- task list
- notification
- survey
- remote / bridge / ssh 状态
- IDE 集成

这不是“控制台聊天窗口”，而是一个完整的开发工作台。

### 7.2 React + Ink 的价值

从 `main.tsx`、`ink/`、`components/`、`hooks/` 的组织方式可以看到，这套 UI 采用的是 React 风格终端应用：

- 组件化渲染
- hook 化状态组织
- AppStateProvider
- KeybindingProvider
- 各类 dialog launcher

这带来的好处是：

- 权限对话框、设置页、resume 选择器、teleport wrapper 都可以复用统一交互框架
- REPL 与 setup screen 可以共享一套渲染机制
- terminal UI 不再是 if/else 拼字符串，而是真正的界面系统

### 7.3 终端产品化细节

源码里能看到很多真正的 UX 细节：

- `history.ts` 中 pasted text / image 用引用占位，而不是污染历史
- `dialogLaunchers.tsx` 将零散弹窗抽成薄启动器
- `statusline`、`theme`、`vim`、`voice`、`buddy`、`keybindings`
- `assistant history`、resume chooser、teleport mismatch dialog
- survey 与 frustration detection

这说明作者在意的不只是“模型答得对不对”，也在意用户是否能长时间舒服地用下去。

### 7.4 性能也是 UX 的一部分

从启动预热、lazy import、deferred prefetch、GrowthBook gate cache、system prompt section cache 等设计可以看出：

- UX 不是只看视觉
- CLI 首屏时间、滚动流畅度、响应速度、切换模式成本都被视为体验的一部分

这是一种很成熟的工程产品视角。

### 7.5 多运行模式下的一致体验

Claude Code 并没有因为运行模式增多而完全割裂体验：

- 本地 REPL
- remote viewer
- bridge
- ssh
- print/headless

虽然它们实现不同，但都尽量保留统一的消息语义、权限交互和命令集合。这种一致性对用户心智模型非常重要。

### 7.6 状态管理分层本身就是 UX 基础设施

如果只看界面层，很容易忽略 Claude Code 的另一层 UX 投入：状态管理。

这套代码至少分成了三层状态：

- `bootstrap/state.ts`：进程级与 session 级全局态，例如 `cwd`、`projectRoot`、`sessionId`、telemetry counters、最近 API 请求、prompt cache latches、invoked skills
- `state/AppStateStore.ts`：REPL 前台态，例如 settings、permission context、remote/bridge 状态、MCP clients、plugins、tasks、notifications、speculation
- `state/store.ts`：最底层的轻量 observable store，仅提供 `getState` / `setState` / `subscribe`

这个分层的好处是：

- 进程级不变量不会和 UI 局部状态混在一起
- Ink/React 层可以按需订阅，不需要引入更重的状态库
- REPL、dialog、remote viewer、权限弹窗能够共享一套一致的状态语义

换句话说，Claude Code 的“终端体验稳定”并不是因为组件写得漂亮，而是因为底层状态边界切得比较清楚。

相关源码：

- `src/screens/REPL.tsx`
- `src/components/`
- `src/ink/`
- `src/history.ts`
- `src/dialogLaunchers.tsx`
- `src/hooks/`
- `src/bootstrap/state.ts`
- `src/state/AppStateStore.ts`
- `src/state/store.ts`

</details>

---

<details>
<summary><strong>8. 最小必要组件</strong></summary>

说明：本节不是在描述仓库现成的“最小发行版”，而是在现有架构上抽象出一套**能够复现 Claude Code 核心闭环**的最小运行单元。

如果你要做一个“Claude Code 风格”的最小 agent CLI，真正不可缺的组件只有以下几类。

### 8.1 会话入口

最小系统首先需要一个入口层，负责：

- 解析用户输入
- 决定交互模式还是 headless 模式
- 初始化 cwd / sessionId / config

在当前源码里，这个角色由 `src/main.tsx` 承担。

### 8.2 消息与状态模型

你必须有一套稳定的消息对象模型，否则无法支撑：

- tool use / tool result
- assistant streaming
- resume / replay
- compact / snip / persistence

当前实现对应：

- `src/types/message.js`
- `src/bootstrap/state.ts`
- `src/state/store.ts`

### 8.3 Query 主循环

这是绝对核心。没有它，就只有一个普通聊天 CLI。  
最小 agent CLI 至少要有：

- system prompt 组装
- 消息标准化
- API streaming
- tool_use 检测
- tool_result 回灌
- 多轮迭代直到 stop

当前实现对应：

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/services/api/claude.ts`

### 8.4 最小工具集

如果目标是“能真的写代码”，最小工具集至少应包括：

- 文件读取
- 文件编辑或写入
- shell 执行
- 搜索

在当前项目中，大致对应：

- `FileReadTool`
- `FileEditTool` 或 `FileWriteTool`
- `BashTool`
- `GrepTool` / `GlobTool`

没有这些工具，模型再强也只能停留在建议层。

### 8.5 权限控制

只要有 shell 和文件写入，权限系统就不是可选项。最小版本至少要有：

- allow / deny / ask
- 路径级规则
- 对写操作和 shell 操作的区分

当前实现对应：

- `src/Tool.ts` 中的 `checkPermissions`
- `src/hooks/useCanUseTool.tsx`
- `src/utils/permissions/*`

### 8.6 最小上下文工程

一个能工作的 agent CLI 至少要注入：

- 当前工作目录
- 当前日期
- 基本 repo 信息
- 项目说明文件

否则模型的行为会非常漂浮。  
当前源码中这部分来自：

- `src/context.ts`
- `src/constants/prompts.ts`
- `src/utils/claudemd.ts`

### 8.7 最小持久化

哪怕不做复杂 resume，也至少要能保存：

- transcript
- session id
- 最近消息
- 文件修改记录

否则就无法支持真正的多轮协作。  
当前实现对应：

- `src/utils/sessionStorage.ts`
- `src/history.ts`
- `src/utils/fileHistory.ts`

### 8.8 可以删掉但不影响“核心 agent 闭环”的部分

如果只是做最小系统，这些都可以先不做：

- MCP
- bridge / remote / teleport
- 多 Agent
- memory / skills
- voice / buddy / survey
- LSP
- IDE integration

但一旦想把系统从“能跑”做成“能长期工作”，这些模块几乎都会重新长回来。Claude Code 现在的体量，本质上就是这些“非最小组件”长期积累的结果。

相关源码：

- `src/main.tsx`
- `src/QueryEngine.ts`
- `src/query.ts`
- `src/Tool.ts`
- `src/tools.ts`
- `src/context.ts`
- `src/utils/sessionStorage.ts`

</details>

---

<details>
<summary><strong>9. Hooks 与可扩展性</strong></summary>

Claude Code 的扩展性不是只靠插件。Hooks、skills、plugins、MCP、commands 共同组成了一个多层扩展系统。

### 9.1 Hooks 是生命周期级扩展点

`src/utils/hooks.ts` 直接定义了 Hooks 的定位：

> Hooks are user-defined shell commands that can be executed at various points in Claude Code's lifecycle.

这意味着 Hooks 不是 UI 菜单项，而是插在生命周期关键位置的执行点。

### 9.2 Hook 事件类型很丰富

`src/types/hooks.ts` 中能看到大量 hook event / response 类型，例如：

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `UserPromptSubmit`
- `SessionStart`
- `SessionEnd`
- `Setup`
- `PermissionDenied`
- `PermissionRequest`
- `Elicitation`
- `FileChanged`
- `CwdChanged`
- `SubagentStart`
- `SubagentStop`

这说明 hook 系统覆盖的不是单个动作，而是几乎整个 agent 生命周期。

### 9.3 Hook 不只是观察，还能反向干预系统

从 schema 可以看到，hook 响应不仅能“说一句话”，还能：

- 决定 `continue` / `stopReason`
- 在 `PreToolUse` 中返回 permission decision
- 修改 `updatedInput`
- 附加 `additionalContext`
- 为 `SessionStart` / `FileChanged` 注册 watch paths
- 对 permission request 直接给 allow / deny 决策

这说明 hook 不是 logging callback，而是**可拦截、可修改、可施加策略**的扩展点。

### 9.4 同步与异步 Hook

`src/types/hooks.ts` 还区分了：

- sync hook
- async hook

`src/utils/hooks.ts` 中还有：

- 后台执行
- async hook registry
- 超时控制
- 将 blocking error 回灌为系统提醒

也就是说，Hook 不仅支持即刻决策，还支持耗时任务和后台回唤。

### 9.5 Hook 执行后端并不单一

从 `utils/hooks.ts` 的 import 可以看到，Hook 执行至少支持：

- shell command
- agent hook
- HTTP hook
- callback hook

这让 Hook 成为了一个非常灵活的扩展面，可以接本地脚本、外部服务甚至 agent 本身。

### 9.6 Skills、Plugins、Commands、MCP 共同组成扩展层

除了 Hook，Claude Code 还有几条并行扩展路径：

- skills：prompt 型能力
- plugins：命令与技能注入
- commands：用户侧显式入口
- MCP：外部能力总线

`commands.ts` 里能够看到多种命令来源被统一装配：

- skill dir commands
- plugin skills
- bundled skills
- builtin plugin skills
- workflow commands
- MCP skill commands

这种设计很强，因为它让“扩展能力”尽量复用同一套装配与发现机制。

### 9.7 为什么这套可扩展性设计很有价值

很多 agent 系统的扩展方式非常碎：

- 一部分写死在 prompt
- 一部分靠插件
- 一部分靠外部 API

Claude Code 的优点在于，它把这些扩展尽量都拉回：

- 统一命令面
- 统一工具面
- 统一权限面
- 统一消息面

这让扩展能力仍然能受到主系统的调度、预算与安全约束。

相关源码：

- `src/utils/hooks.ts`
- `src/types/hooks.ts`
- `src/commands.ts`
- `src/commands/hooks/index.ts`
- `src/skills/loadSkillsDir.ts`
- `src/services/mcp/client.ts`

</details>

---

<details>
<summary><strong>10. 多 Agent 架构</strong></summary>

Claude Code 的多 Agent 设计不是“子任务递归调用”那么简单，而是一套带隔离、任务状态、权限语义、远程执行能力的子运行时系统。

### 10.1 AgentTool 是统一入口

`src/tools/AgentTool/AgentTool.tsx` 定义了多 Agent 的主入口。其输入 schema 支持：

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

这说明子 Agent 不是黑盒调用，而是可配置执行单元。

### 10.2 子 Agent 可以是同步、异步、后台任务

源码里能看到多种子 Agent 运行方式：

- 同步运行并把结果直接回到父回合
- 以 background task 形式运行
- 注册本地 async agent task
- 注册 remote agent task

这意味着 AgentTool 不是单一执行模型，而是统一入口 + 多种后端。

### 10.3 隔离策略是真正落地的

`AgentTool` 支持至少两种隔离模式：

- `worktree`
- `remote`

其中 `worktree` 的价值在于：

- 给子 Agent 独立工作副本
- 避免与主会话文件状态直接互踩

而 `remote` 则把子 Agent 发往远端 CCR 环境，形成更强隔离。

### 10.4 Team 与 Swarm

除了单个子 Agent，代码里还存在：

- `TeamCreateTool`
- `TeamDeleteTool`
- `SendMessageTool`
- swarm permission / leader bridge / worker handling
- coordinator mode

这表明系统支持的不是简单树状子任务，而是更复杂的 team/swarm 协作模型。

### 10.5 多 Agent 仍受统一权限与任务系统约束

多 Agent 并不是“脱离主系统另起一套”。  
它仍然会与这些模块耦合：

- permission mode
- app state
- task output file
- progress tracker
- session storage
- message tagging
- telemetry

换句话说，子 Agent 在架构上是“派生会话”，不是“普通函数调用”。

### 10.6 多 Agent 与验证工作流

`TodoWriteTool` 里甚至有一个很有意思的设计：

- 当主线程 agent 关闭了 3 个以上任务
- 且 todo 中没有 verification 项
- 系统会在 tool result 里明确 nudging，让它去调 verification agent

这说明 Claude Code 不只是“允许多 Agent”，而是在逐步把“验证 agent”“规划 agent”“工作 agent”做成结构化协作习惯。

### 10.7 这套多 Agent 架构的意义

它的价值不只是提升并行性，更关键的是：

- 让不同类型工作具备不同权限和隔离级别
- 让复杂任务可以显式拆分
- 让验证与执行从机制上分离

这比“让一个模型自己又写又审”更可靠。

相关源码：

- `src/tools/AgentTool/AgentTool.tsx`
- `src/tools/TeamCreateTool/`
- `src/tools/TeamDeleteTool/`
- `src/tools/SendMessageTool/`
- `src/coordinator/`
- `src/tasks/`
- `src/utils/swarm/`

</details>

---

<details>
<summary><strong>11. 记忆与技能系统</strong></summary>

记忆系统与技能系统是 Claude Code 里两个非常有辨识度的高阶层。前者负责跨会话持久化，后者负责把“可复用工作流”以 prompt 能力的形式组织起来。

### 11.1 记忆系统不是一个文件，而是一套文件型知识库

`src/memdir/memdir.ts` 展示出记忆系统的基本形态：

- 记忆目录存在于文件系统中
- 入口文件叫 `MEMORY.md`
- `MEMORY.md` 是索引，不是承载全部记忆内容的正文仓库
- 每条 memory 更推荐写成单独 topic file，再在 `MEMORY.md` 里挂指针

这意味着 Claude Code 的 memory 不是 KV store，而是**可读、可编辑、可版本化的 Markdown 知识库**。

### 11.2 记忆有明确类型学

从 `memoryTypes.ts` 与 `buildMemoryLines()` 的语义可以看出，系统对记忆内容有明确分类和约束：

- 应该保存什么
- 不应该保存什么
- 什么时候访问记忆
- 什么时候应该更新而不是新增

这避免了“什么都往 memory 里塞”，否则记忆会很快退化成噪声。

### 11.3 `MEMORY.md` 有严格的大小和形态约束

`memdir.ts` 中对入口文件有明确限制：

- 最大行数
- 最大字节数
- 超限时会截断并附带 warning

原因很直接：`MEMORY.md` 会进入系统提示词。  
它既是持久化结构，又是 prompt 预算结构，所以必须控制体积。

### 11.4 Assistant 长会话模式下，记忆策略还会切换

在 KAIROS / assistant 模式下，记忆不会始终直接写 `MEMORY.md`，而会转成：

- append-only daily log
- 夜间再蒸馏到 `MEMORY.md` 与 topic files

这个设计非常成熟，因为长寿命会话直接维护索引文件会导致：

- 高频重写
- 上下文污染
- 协作冲突

而 append-only log 更适合“先记录，再蒸馏”。

### 11.5 记忆会被接入系统提示词

记忆不是“有空再查”的外部存储，它直接参与模型行为：

- `constants/prompts.ts` 把 `loadMemoryPrompt()` 接到 system prompt sections
- `QueryEngine.ts` 在某些情形下还会额外注入 memory mechanics prompt

也就是说，记忆系统同时扮演了：

- 文件型持久化层
- 模型行为指导层

### 11.6 技能系统：本质是带元数据的 prompt command

从 `skills/loadSkillsDir.ts` 与 `SkillTool.ts` 可以看出，技能本质上是：

- Markdown 文件
- 带 frontmatter
- 最终装配成 `Command`

frontmatter 支持的能力很丰富：

- `description`
- `when_to_use`
- `allowed-tools`
- `model`
- `effort`
- `hooks`
- `paths`
- `arguments`
- `disable-model-invocation`

这说明 skill 不是静态提示词片段，而是带执行语义和装配语义的“prompt capability package”。

### 11.7 技能来源很多，而且被统一管理

技能至少有这些来源：

- 本地 skill dir
- bundled skills
- plugin skills
- builtin plugin skills
- MCP skills
- 动态发现 skill dirs

`commands.ts` 把这些来源统一装配后，再给：

- 用户侧 `/skills`
- 模型侧 `SkillTool`

这是一种非常聪明的复用设计。

### 11.8 SkillTool 并不是简单展开 prompt，它会走 forked sub-agent

`src/tools/SkillTool/SkillTool.ts` 中最有意思的一点是：

- 某些技能不是直接把文本插进当前上下文
- 而是会放到 forked sub-agent context 中执行

这意味着技能可以拥有：

- 自己的 agent
- 自己的 token budget
- 自己的进度上报
- 自己的消息收集

因此技能系统在 Claude Code 中并不是“prompt library”，而更像“轻量级 agent workflow package”。

### 11.9 记忆与技能的关系

这两个系统其实互补：

- 记忆回答“我长期知道什么”
- 技能回答“我长期会怎么做”

一个是持久化事实与偏好，另一个是持久化工作流与能力包。  
放在 coding agent 场景里，这两者一起决定了 agent 的“长期行为稳定性”。

相关源码：

- `src/memdir/memdir.ts`
- `src/memdir/paths.ts`
- `src/constants/prompts.ts`
- `src/skills/loadSkillsDir.ts`
- `src/skills/bundled/index.ts`
- `src/tools/SkillTool/SkillTool.ts`
- `src/commands/skills/index.ts`

</details>

---

## 总结

如果只把 Claude Code 看成一个“终端里的 Claude”，会低估这份源码的复杂度。  
更准确的判断应该是：

> **它是一套本地优先、消息驱动、工具编排型、强权限治理、可远程扩展的 agent 执行平台。**

从架构角度最值得关注的几件事是：

- 它把系统主循环做成了真正的多轮 agent 状态机。
- 它把工具、权限、上下文、压缩、预算、记忆、技能、多 Agent 都纳入了同一运行时。
- 它把 REPL 做成了真正的产品前台，而不是日志终端。
- 它把扩展与安全都设计成了一等公民。

这也是为什么它读起来更像三者叠加：

- 一个终端 IDE
- 一个 agent runtime
- 一个企业化开发客户端

而不是单纯的命令行聊天工具。
