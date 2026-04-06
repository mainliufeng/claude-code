# Claude Code 与 Codex Rust MCP Tool 机制对比

这份文档只看 `MCP tool`。

不讨论：

- MCP prompts
- MCP resources
- MCP skills
- MCP auth UI

只看一条链：

- 模型看见什么
- tool call 怎么被路由
- 权限怎么挂进去
- 断线和错误怎么处理
- 最后在产品上会长成什么样

## 1. Claude Code：先把 MCP tool 变成内部 Tool，再走现有 tool 系统

Claude Code 的 MCP tool 核心入口在：

- `src/services/mcp/client.ts`
- `src/tools/MCPTool/MCPTool.ts`

它的做法不是在主循环里保留一个“外部 MCP tool”类型。

它先做 `tools/list`，然后把远端 MCP tool 包装成 Claude Code 内部的 `Tool` 对象。

### 1.1 包装发生在什么时候

包装发生在 `fetchToolsForClient(...)`。

这里会对每个 MCP tool 做一层本地映射，补上：

- `name`
- `description / prompt`
- `inputJSONSchema`
- `mcpInfo`
- `isReadOnly`
- `isDestructive`
- `isOpenWorld`
- `checkPermissions`
- `call`

也就是说，在 Claude Code 里，MCP tool 在进入主循环之前，就已经被“Tool 化”了。

模型看到的不是一层独立的 MCP 协议对象。
模型看到的是普通 tool，只不过这个 tool 背后会回调某个 MCP server。

### 1.2 调用时怎么执行

执行不是走单独的 MCP router，而是直接走这个动态生成的 `Tool.call(...)`。

这条调用链里会做：

- `ensureConnectedClient(...)`
- `callMCPToolWithUrlElicitationRetry(...)`
- progress 上报
- session 失效重试
- MCP auth 错误转 `needs-auth`
- tool result / meta 组装

所以 Claude Code 的设计不是“tool call 时再识别这是不是 MCP”。
而是“在工具池里它就已经是普通 tool 了”。

### 1.3 权限是怎么接进去的

MCP tool 自己实现了 `checkPermissions()`，返回 `passthrough`。

这意味着：

- 它直接接进现有 tool 权限框架
- 不需要单独再建一套 MCP 批准系统
- 用户看到的批准体验，更容易和 bash / edit / write 这类内建 tool 保持一致

从实现上看，这种统一感来自“它被提前抹平”为内部 `Tool`。

### 1.4 产品上会长成什么样

这套设计带来的直接效果是：

- MCP tool 更像系统自带工具
- 权限提示、进度、结果渲染更统一
- 断线重连和 auth 状态更容易被产品层自动兜底

用户感受到的不是“我正在经过一个 MCP 协议桥接层”。
用户感受到的是“系统里多了一批工具”。

代价也很明确：

- `client.ts` 会变得很厚
- 连接、鉴权、重试、tool 映射、结果处理都堆在一起
- MCP 协议细节更容易渗进产品层

## 2. Codex Rust：先保留 MCP tool 身份，在路由层识别成独立 payload

Codex Rust 的核心入口在：

- `core/src/mcp_connection_manager.rs`
- `core/src/tools/router.rs`
- `core/src/tools/handlers/mcp.rs`
- `core/src/mcp_tool_call.rs`

它的做法和 Claude Code 不一样。

它不会先把 MCP tool 完全抹平成普通 tool 对象。
它先保留一层明确的 MCP 身份。

### 2.1 tool 名怎么进系统

`McpConnectionManager` 先把所有 MCP tool 聚合起来，再统一命名成：

- `mcp__server__tool`

这里还会做 sanitize 和长度控制，保证最终 tool 名能合法进入 Responses API。

这一步说明 Codex 不是在“包装成普通 tool 后忘掉来源”。
它在名字层面就保留了来源。

### 2.1.1 这里不是变成一个工具，而是变成一张全局工具表

这一步最容易误解。

`McpConnectionManager` 不是把所有 MCP tool 合成一个“大网关工具”。

它做的是：

1. 每个 server 单独 `tools/list`
2. 把每个 tool 记成一条 `ToolInfo`
   - `server_name`
   - `tool_name`
   - `tool` 原始 schema
3. 遍历所有 server，把这些 `ToolInfo` 聚合进一张总表
4. 给每条记录分配一个全局唯一的名字

最后得到的是：

- `mcp__github__create_issue`
- `mcp__github__search_code`
- `mcp__notion__search`

这种很多个工具。

所以这里的“聚合”不是“合并成一个工具”，而是“收集成一张统一索引表”。

### 2.1.2 这张表是怎么生成的

主线都在 `core/src/mcp_connection_manager.rs`：

1. 初始化每个 MCP server client
2. 对每个 server 跑 `tools/list`
3. 先按配置过滤
   - `enabled_tools`
   - `disabled_tools`
4. 再做名字规范化
   - 原始名先按 `mcp__{server}__{tool}` 拼出来
   - 再做字符 sanitize
   - 超长时截断并追加 hash
   - 处理撞名

最后 `list_all_tools()` 返回的是：

- key：模型最终看到的 function tool 名
- value：这个名字背后对应的 `ToolInfo`

这张表同时解决了 3 个问题：

- 不同 server 的同名 tool 冲突
- 模型 function name 合法性
- 调用时的路由信息保留

### 2.1.3 这张表怎么进入模型的 tool 列表

这一层不在 `McpConnectionManager` 里，而在 `core/src/tools/spec.rs`。

运行一轮模型请求前，`core/src/codex.rs` 会先拿到 `list_all_tools()` 的结果，然后交给 `ToolRouter::from_config(...)`。

后面 `build_specs(...)` 会把总表里的每一项都单独转成一个 `ToolSpec::Function`。

也就是说，模型看到的是很多个普通 function tool：

- `mcp__github__create_issue`
- `mcp__github__search_code`
- `mcp__notion__search`

不是一个叫 “mcp” 的总入口工具。

这一步的本质是：

- 对模型接口来说，MCP tool 被展开成很多普通 function tool
- 对运行时内部来说，这些工具名还能再反解回 MCP 的 `server + tool`

### 2.2 调用时怎么识别

模型返回 function call 后，`ToolRouter` 会先解析 tool name。

如果名字能被拆成：

- server
- tool

那它不会把它当成本地 function tool，而是构造成：

- `ToolPayload::Mcp`

这一步很关键。

在 Codex 里，MCP tool 在路由层依然是 MCP tool。
它没有在更早阶段被完全抹平。

### 2.3 调用时怎么执行

后面 `McpHandler` 收到 `ToolPayload::Mcp`，再交给 `handle_mcp_tool_call(...)`。

这一层会做：

- 发 `McpToolCallBegin`
- `Session.call_tool(server, tool, arguments)`
- 发 `McpToolCallEnd`
- 返回 `McpToolCallOutput`

所以它的调用链是显式分层的：

- 连接管理层
- 路由层
- MCP handler
- session 调用层
- event 层

### 2.3.1 工具调用具体怎么落到 MCP server

这条链可以直接写成：

1. 模型返回 function call  
   例如 `mcp__github__create_issue`
2. `ToolRouter` 解析这个名字
3. 如果能反解回 MCP tool，就构造成  
   `ToolPayload::Mcp { server, tool, raw_arguments }`
4. `McpHandler` 接手
5. `handle_mcp_tool_call(...)` 发 begin 事件
6. `Session.call_tool(server, tool, arguments)`
7. `McpConnectionManager.call_tool(server, tool, arguments)`
8. 最终发到对应 MCP server 的 `call_tool`
9. 返回结果后再发 end 事件，并包装成 `McpToolCallOutput`

所以 Codex 这里其实是“两次变形”：

- 第一次：为了让模型能调用，把 MCP tool 展平成 function tool
- 第二次：为了真正执行，再把 function tool 名还原成 MCP 的 `server + tool`

这也是它和 Claude Code 很不一样的地方。

### 2.4 权限是怎么接进去的

这一段可以直接说白。

Codex 对 MCP tool 的权限，不是“每次调用前都问用户要不要授权”，而是“先决定这个 tool 能不能出现在工具列表里”。

先有一层暴露控制：

- 每个 MCP server 配置里可以写 `enabled_tools`
- 也可以写 `disabled_tools`
- `McpConnectionManager` 在聚合 tool 时先按这两项过滤

所以第一层控制是：

- 哪些 MCP tool 能暴露给模型
- 哪些 MCP tool 在进入系统前就被去掉

但进入执行阶段后，当前这条调用链里没有看到统一的逐次授权流。

原因是：

- `ToolRegistry` 只有在 handler 的 `is_mutating()` 返回 `true` 时，才会走 `tool_call_gate`
- `ToolHandler` 默认 `is_mutating = false`
- `McpHandler` 没有覆写这层

所以当前实际行为可以直接理解成：

- MCP tool 先经过配置过滤
- 只要已经进入工具列表，模型就可以直接调用
- 不会像 shell / apply_patch 那样，在每次调用前先停下来等一次用户授权

所以 Codex 这里不是完全没有控制。

它有控制，但控制点放在：

- server 接入
- tool 暴露过滤

而不是放在：

- 每次调用前 ask user

### 2.5 产品上会长成什么样

这套设计带来的直接效果是：

- 用户更容易知道这是 MCP tool call
- 更容易知道具体是哪个 server、哪个 tool 在执行
- begin / end / result 事件更清楚
- 排障和可观测性更直接

在 TUI 里，这种边界是显式存在的：

- `McpToolCallBegin`
- `McpToolCallEnd`
- 专门的 MCP history cell

所以 Codex 更像是在告诉用户：

- 这是一个外部能力调用
- 系统知道它来自 MCP
- 这条链有自己的事件身份

## 3. 真正的差别在哪里

差别不是“都能不能调 MCP tool”。

差别在系统把 MCP tool 放在哪一层。

### 3.1 Claude Code

Claude Code 把 MCP tool 放在“工具池构建阶段”就抹平了。

这意味着：

- 后面越往下走，它越像普通 tool
- 统一体验更容易做
- 自动重试、权限、进度、渲染也更容易复用

### 3.2 Codex Rust

Codex 把 MCP tool 放在“路由阶段”才识别成独立 payload。

这意味着：

- 后面越往下走，它越保留 MCP 身份
- 事件边界和调试边界更清楚
- core 的分层也更稳

## 4. 只看 MCP tool，本身有哪些相关功能

把范围再收窄一点，只看 MCP tool，本质上要回答 6 个问题：

- tool 怎么进入系统
- tool 怎么暴露给模型
- 权限怎么处理
- 连接、重连、认证怎么处理
- tool 调用过程怎么被展示和记录
- tool 列表变化时系统怎么更新

下面分别看两边的实现。

### 4.1 Claude Code

Claude Code 对 MCP tool 的处理，核心是“尽早 Tool 化”。

#### 4.1.1 tool 注册和模型暴露

在 `fetchToolsForClient(...)` 里，每个远端 MCP tool 会先被包装成内部 `Tool`。

包装时会补上这些和运行时相关的属性：

- `name`
- `mcpInfo`
- `isMcp`
- `inputJSONSchema`
- `searchHint`
- `alwaysLoad`
- `isConcurrencySafe`
- `isReadOnly`
- `isDestructive`
- `isOpenWorld`
- `isSearchOrReadCommand`
- `checkPermissions`
- `call`

这意味着它不是只把 schema 转一下。

它实际上把一个远端 MCP tool，扩成了 Claude Code 内部完整的 Tool 对象。

这一步完成以后，后面的 tool search、defer loading、permissions、tool UI、tool result，都可以直接复用现有 Tool 系统。

#### 4.1.2 权限

Claude Code 这里至少有两层权限。

第一层是 server 级别：

- server 能不能被接进来
- project MCP server 是 `approved / pending / rejected`
- 可以被 settings 和 policy 提前禁用

这层在 MCP config / utils 里处理。

第二层是 tool 级别：

- 每个 MCP tool 都有 `checkPermissions()`
- 当前返回的是 `passthrough`
- 并且会附带把这个 `mcp__server__tool` 加进本地规则的 suggestion

也就是说，MCP tool 本身没有单独发明一套批准模型。

它是被接进 Claude Code 现有 tool 权限系统里跑的。

这点很重要。

在 Claude Code 里，MCP tool 的调用体验之所以和 bash、read、edit 比较像，不是因为它“本来就是普通 tool”，而是因为它已经被提前包装成内部 `Tool`，所以自然复用了现有的：

- tool permission 检查
- 规则建议
- auto mode 判定
- UI 审批交互

再加上它还会从 MCP annotations 里读：

- `readOnlyHint`
- `destructiveHint`
- `openWorldHint`

这些信息会继续影响权限感知、自动模式和产品提示。

#### 4.1.3 连接、重连、认证

Claude Code 这一层做得很产品化。

在每次真正调用 tool 前，会先：

- `ensureConnectedClient(...)`

调用过程中又有几层补偿逻辑：

- session 失效时清 cache，重建连接
- 最多做一轮 session retry
- 处理 URL elicitation，必要时让用户完成一个 URL 打开流程，再 retry
- auth 问题会把 server 状态转成 `needs-auth`

所以它不是“失败就把错误扔出去”。

它会尽量把：

- 重连
- retry
- needs-auth
- 用户交互补偿

都包进 MCP tool 的 `call()` 里。

从实现上看，Claude Code 这里是明显存在 per-call 补偿层的：

- 调用前先确保连接
- 会话失效时清理 cache 并重连
- 允许一次 session retry
- 需要 URL elicitation 时把用户交互嵌进调用链
- auth 问题会落成 `needs-auth`

所以它不是“连接管理层负责连上就完了”，而是 MCP tool 本身的运行时壳层已经承担了相当一部分产品补偿逻辑。

#### 4.1.4 进度、结果、展示

因为 MCP tool 已经是内部 Tool，所以它天然会接进 Claude Code 的普通 tool 交互层。

调用时会显式发：

- started
- completed

的 progress 事件。

tool result 也会走现有的：

- tool result message
- truncation
- collapse
- structured content / metadata 挂载

这就是为什么在产品上，MCP tool 很容易看起来和 bash、read、edit 是同一类东西。

#### 4.1.5 列表刷新

Claude Code 对 MCP tool 还有一个很产品化的处理：

- 监听 `tools/list_changed`

收到通知后，会：

- 清掉 `fetchToolsForClient` cache
- 重新 `tools/list`
- 更新 app state 里的 tool 列表

所以 MCP tool 不是“启动时扫一次就定死”。

它可以热刷新。

### 4.2 Codex Rust

Codex Rust 对 MCP tool 的处理，核心是“保留 MCP 身份，再把它接进统一 tool runtime”。

#### 4.2.1 tool 注册和模型暴露

`McpConnectionManager` 先做的是收集和命名：

- 对每个 server 跑 `tools/list`
- 先按 `enabled_tools / disabled_tools` 过滤
- 再做全局命名
- 最后得到一张全局工具表

然后在 `build_specs(...)` 里，把这张表里的每个 MCP tool 单独转成一个 `ToolSpec::Function`。

同时还会给每个名字统一注册到 `McpHandler`。

所以模型看到的是很多普通 function tool。

但 runtime 内部并没有忘掉这是 MCP tool。

#### 4.2.2 权限和变更性

Codex 这里先把两件事分开了：

- 工具能不能暴露
- 工具是不是 mutating，需要走 gate

第一件事现在主要靠 MCP server config：

- `enabled_tools`
- `disabled_tools`

这是 server/tool 维度的显式过滤。

所以 Codex 不是完全没有权限控制。

它有一层“能不能暴露给模型”的控制。

只是这层控制发生在：

- server 配置
- tool 过滤

而不是发生在“每次调用前先向用户申请一次权限”。

第二件事在当前这条调用链里，并没有像 shell、apply_patch 那样专门挂进同一套 mutating gate。

原因很直接：

- `ToolRegistry` 默认 `is_mutating = false`
- `McpHandler` 没有覆写这一层

所以从当前实现看，Codex 的 MCP tool 权限模型可以直接理解成：

- 先控制“哪些 MCP tool 能暴露出来”
- 但不控制“每次调用前要不要再问一次用户”

也就是说：

- Codex 的 MCP tool 当前有“暴露控制”
- 但没有明显接进统一的“逐次 ask user 审批流”
- 一旦某个 MCP tool 已经进入工具列表，模型就可以直接调用，不会像 shell / apply_patch 那样先停下来等授权

再说白一点：

- 不是完全没控制
- 而是控制点放在“接不接进来、暴露不暴露”
- 不是放在“调用前逐次授权”

#### 4.2.3 连接和调用

Codex 的 `McpConnectionManager` 负责：

- 初始化 client
- 维护每个 server 的连接
- 聚合 tools
- 真正执行 `call_tool`

运行一轮时，模型如果返回 `mcp__server__tool` 这样的 function call：

- `ToolRouter` 先把它解析成 `ToolPayload::Mcp`
- 再交给 `McpHandler`
- `McpHandler` 再交给 `handle_mcp_tool_call(...)`
- 最后 `Session.call_tool(...)`
- 再落到 `McpConnectionManager.call_tool(...)`

所以 Codex 这里不是在某个动态 Tool 对象里把所有事情包完。

它是：

- manager 管连接
- router 管识别
- handler 管执行
- session 管事件和回流

#### 4.2.3.1 重试和失败补偿

从当前这条 MCP tool 主链看，Codex 没有像 Claude Code 那样，在每个 MCP tool 调用外面包一层很厚的 per-call 自动补偿壳。

当前能明确看到的是：

- server 启动阶段有 startup 管理
- tool 调用阶段有 timeout
- 调用失败会进入 `McpToolCallEnd`
- 错误结果会原样回到 session / UI / model 回流链

但没有明显看到这些 Claude Code 风格的行为：

- 每次调用前 `ensureConnectedClient`
- session expired 后主动 retry
- needs-auth 状态切换后再自动补一轮调用
- URL elicitation 这类交互式重试

所以 Codex 这里的思路更像：

- 连接管理和调用分层
- 失败要被明确记录成事件
- 重试不是 MCP tool 外壳的默认职责

这也解释了为什么 Codex 的实现更像 runtime，而不是产品化壳层。

#### 4.2.4 进度、结果、展示

Codex 对 MCP tool 的展示不是“假装它就是普通 tool”。

它会发专门的事件：

- `McpToolCallBegin`
- `McpToolCallEnd`

TUI 也有单独的 MCP call 渲染路径。

所以在产品上，用户更容易明确知道：

- 这是一条 MCP 调用
- 来自哪个 server
- 调了哪个 tool
- 什么时候开始
- 什么时候结束

这条链的可观测性更强。

#### 4.2.5 列表变化

从当前这条 core 实现看，Codex 的重点放在：

- 初始化时聚合
- 调用时路由

没有像 Claude Code 那样，在前台 runtime 里明显做一层 `tools/list_changed -> 热刷新 app state` 的产品化壳层。

所以它的 MCP tool 管理更像 core service，而不是 UI runtime service。

## 5. 进度、状态、结果，这两边到底怎么抽象

这部分最容易被一句“都能调用 MCP tool”掩盖掉。

如果只看 runtime 内部数据结构，两边差异其实很大。

### 5.1 Claude Code：MCP tool 在运行时里基本被当成普通 Tool

Claude Code 的核心抽象还是 `Tool`。

MCP tool 在进入系统时，就已经被包装成内部 `Tool`，所以它在运行时里会复用普通 tool 的这些公共结构：

- `Tool`
- `ToolResult`
- `ProgressMessage`
- `tool_result` user message

也就是说，Claude Code 这里的 MCP tool 和普通 tool，在主数据结构上基本是同构的。

区别主要体现在两处：

- progress data 的具体类型是 `mcp_progress`
- result 上允许额外挂 `mcpMeta`

#### 5.1.1 进度怎么抽象

Claude Code 的进度抽象是统一的 `ProgressMessage`。

MCP tool 只是往这个统一结构里塞一种特定 data：

- `type: 'mcp_progress'`
- `status: 'started' | 'completed' | 'failed'`
- `serverName`
- `toolName`
- 可选 `progress`
- 可选 `total`
- 可选 `progressMessage`
- 可选 `elapsedTimeMs`

所以它的思路不是“为 MCP 单独发 begin/end 事件”，而是：

- 仍然用统一 progress message
- 只是 message payload 换成 MCP 专用字段

UI 层也是这么消费的：

- `MCPTool/UI.tsx` 读一串 `ProgressMessage<MCPProgress>`
- 决定显示 `Running...`
- 或百分比进度条
- 或完成/失败后的状态

这意味着在 Claude Code 里，MCP tool 的状态可见性是“嵌进统一 tool 进度系统”的。

#### 5.1.2 结果怎么抽象

Claude Code 的普通 tool 返回的是 `ToolResult<T>`：

- `data`
- `newMessages`
- `contextModifier`
- 可选 `mcpMeta`

MCP tool 返回时，本体结果仍然放在：

- `data: mcpResult.content`

如果 MCP server 还返回了 `_meta` 或 `structuredContent`，则再挂到：

- `mcpMeta._meta`
- `mcpMeta.structuredContent`

而真正进入会话历史的，仍然是普通的 `tool_result` user message。

也就是说：

- 会话主链里，MCP tool 的结果还是普通 `tool_result`
- MCP 专属的协议元数据放在 message sidecar 上
- 这些 sidecar 不一定直接发回模型，但会保留给 UI、SDK、存储和 hooks

这也是为什么 Claude Code 可以一边维持统一消息模型，一边又不丢掉 MCP 的结构化结果。

#### 5.1.3 内部可见性是什么样

Claude Code 的可见性不是“显式 MCP begin/end 事件流”，而是三层叠在一起：

1. `Tool` 对象本身带 `isMcp / mcpInfo`
2. progress message 里带 `mcp_progress`
3. result sidecar 里带 `mcpMeta`

所以它的 MCP 状态是“分散地附着在普通 tool 结构上”。

这套设计的好处是：

- 普通 tool UI 很容易复用
- 存储和消息模型不需要为 MCP 另起一套主结构

代价是：

- MCP 的运行时身份没有被抬成顶层事件类型
- 想看清楚完整 MCP 生命周期，要同时看 tool、progress、message sidecar 三层

### 5.2 Codex Rust：MCP tool 在运行时里保留独立类型

Codex Rust 这里就不是这样。

它在运行时内部没有把 MCP tool 完全抹平成普通 tool。

它至少有三层是独立出来的：

- `ToolPayload::Mcp`
- `ToolOutput::Mcp`
- `McpToolCallBegin / McpToolCallEnd`

这意味着 Codex 的 MCP tool 状态可见性是“显式 MCP 生命周期”。

#### 5.2.1 调用时的输入结构

在 router 里，一旦识别出 `mcp__server__tool`，就不是普通 function payload，而是：

- `ToolPayload::Mcp { server, tool, raw_arguments }`

这里已经把 MCP 身份单独建模了。

而普通 tool 用的是：

- `ToolPayload::Function`
- `ToolPayload::Custom`
- `ToolPayload::LocalShell`

所以从调用输入结构上，Codex 就已经把 MCP 和普通 tool 区分开了。

#### 5.2.2 进度和状态怎么抽象

Codex 的 MCP tool 状态，不走统一 progress message。

它是单独的顶层事件：

- `McpToolCallBegin`
- `McpToolCallEnd`

begin 里有：

- `call_id`
- `invocation`

end 里有：

- `call_id`
- `invocation`
- `duration`
- `result`

TUI 直接消费这两个事件：

- begin 时创建一个活动中的 `McpToolCallCell`
- end 时把这个 cell 标成完成态

所以 Codex 的状态可见性是：

- 顶层事件级别可见
- UI 里独立 cell 可见
- `server/tool/duration/result` 一路保留

不是嵌在统一 progress data 里的一个变体。

#### 5.2.3 结果怎么抽象

Codex 的 `ToolOutput` 本身就分成：

- `ToolOutput::Function`
- `ToolOutput::Mcp`

其中 MCP 结果不是普通字符串，而是：

- `Result<CallToolResult, String>`

最后再转成：

- `ResponseInputItem::McpToolCallOutput`

也就是说，Codex 对 MCP result 的态度是：

- 它不是普通 function output 的一个字段变体
- 它本身就是一类独立 output

这也是为什么 TUI 能针对它做单独渲染，比如：

- 文本 block
- image block
- embedded resource
- resource link

这些都可以沿着 MCP 结果结构直接展示。

#### 5.2.4 内部可见性是什么样

Codex 的可见性是集中式的。

你想知道一条 MCP 调用发生了什么，看这几层就够了：

1. `ToolPayload::Mcp`
2. `McpToolCallBegin`
3. `McpToolCallEnd`
4. `ToolOutput::Mcp`
5. `ResponseInputItem::McpToolCallOutput`

这条链是连续的。

所以它的优势是：

- 生命周期清楚
- 类型清楚
- 事件清楚
- 调试和观测更直接

代价是：

- MCP tool 和普通 tool 在 runtime 里不是完全同构
- 前端要单独理解 MCP begin/end/result 这套事件和输出

## 6. compact / transcript / replay 里，MCP tool 结果怎么存、怎么保留、怎么压缩

这一段只看一个问题：

- MCP tool 在运行时里很丰富
- 但落到会话历史、transcript、replay、compact 之后，还剩下什么

这件事很重要。

因为很多系统在实时 UI 里看起来很强，但一旦进入：

- 持久化
- 恢复会话
- 回放
- 压缩上下文

就会把原来那套丰富的 MCP 结构改写成更普通的历史记录。

两边在这里的处理差异很明显。

### 6.1 Claude Code：运行时里先 Tool 化，落盘时也尽量沿用同一条消息主链

Claude Code 的 MCP tool 结果，主链仍然是普通 `tool_result` message。

也就是说，对 transcript 和后续会话来说，MCP tool 结果首先不是一类单独的“顶层 MCP 历史对象”，而是：

- 普通 `tool_result`
- 再额外挂 `mcpMeta`

所以它的存储方式可以拆成两层：

第一层是主消息内容：

- `tool_result.content`

第二层是 sidecar：

- `mcpMeta._meta`
- `mcpMeta.structuredContent`

这意味着 Claude Code 的落盘思路是：

- 会话主链继续保持统一消息模型
- MCP 的协议细节不另起一套 transcript 结构
- 但也不直接丢掉，而是挂在 sidecar 上一起存

这也是为什么它恢复会话时，MCP tool 结果还是沿着普通消息恢复，而不是另跑一条独立 MCP replay 链。

#### 6.1.1 transcript 里实际保留了什么

Claude Code 的 transcript/会话存储会把消息对象整体序列化。

这意味着对 MCP tool 来说，真正被保留下来的主要是：

- `tool_result` 这条消息
- `toolUseResult`
- `mcpMeta`

但有一个点要单独看：

- progress message 不会一起落盘

也就是说，Claude Code 的：

- `mcp_progress`
- started/completed/failed 这种实时进度

主要是 UI 运行时状态，不是 transcript 主体。

所以会话恢复之后，你通常还能拿到：

- 结果
- 结构化 sidecar

但不会原样拿回一整条实时进度时间线。

#### 6.1.2 大结果怎么保留

Claude Code 在这一层做得比较重。

它不只是依赖普通 compact，还专门给 MCP 输出做了单独存储路径。

可以粗暴理解成两套：

- 普通大文本 tool result：走 `toolResultStorage`
- MCP 大结果、二进制结果：走 `mcpOutputStorage`

所以它处理 MCP tool 输出时，不是简单“太大了就截断”。

而是会做：

- 大文本输出落盘
- 二进制内容单独持久化
- 会话里保留引用或精简后的表示

这说明 Claude Code 在这里的目标是：

- 保住产品可读性
- 保住会话可恢复性
- 但不把超大 MCP 输出一直塞在 prompt 主链里

#### 6.1.3 compact 时怎么处理

Claude Code 对 MCP tool 结果的 compact 不是“完全单独一套”，而是“两段式”。

第一段是输出落盘：

- 先把大结果从消息主链里挪出去
- 会话里留下引用、预览或精简表示

第二段才是普通 compact：

- 后续再和其他 `tool_result` 一起进入通用 compact 流程

这里一个很关键的事实是：

- Claude Code 有 MCP 专用输出存储
- 但 `microcompact` 里并没有明显把 MCP tool 单独列成一类“专门清空旧结果的 tool”

这意味着它对 MCP tool 的思路不是：

- 为 MCP 再做一套完全平行的 compact

而是：

- 先把 MCP 特有的大输出问题解决掉
- 然后回到统一消息压缩链

#### 6.1.4 replay / resume 时是什么感觉

Claude Code 的 replay/resume 更像：

- 读取统一 transcript
- 恢复普通消息链
- MCP 结果作为 `tool_result + mcpMeta` 一起回到会话里

所以它保留的是：

- MCP 结果内容
- MCP 结构化 sidecar

但不会把实时 progress 事件原样 replay 成一条独立 MCP 生命周期。

换句话说，Claude Code 在 replay 阶段优先保的是：

- 结果语义
- 消息连续性

不是：

- 完整的 MCP 调用事件时间线

### 6.2 Codex Rust：运行时里显式区分 MCP，落到历史里再逐步改写成普通历史记录

Codex 的 MCP runtime 内部很“硬”：

- `ToolPayload::Mcp`
- `ToolOutput::Mcp`
- `McpToolCallBegin`
- `McpToolCallEnd`
- `McpToolCallOutput`

但这个结构不会完整原样穿透到 transcript / replay / compact。

它在持久化和后续上下文里，会一步步改写成更普通的历史记录形式。

#### 6.2.1 运行时先保留独立 MCP 事件

实时执行阶段，Codex 对 MCP tool 的可见性很强：

- begin 事件
- end 事件
- 独立 output 类型
- TUI 单独渲染

这一层它明显比 Claude Code 更强调：

- 生命周期
- 类型边界
- 调用边界

#### 6.2.2 但进入历史后，会先被改写成普通 `FunctionCallOutput`

Codex 的关键点在这里：

- 运行时内部先得到的是 `ResponseInputItem::McpToolCallOutput`
- 但这个对象不会直接作为长期历史记录一直保存

写历史时，它会被改写成：

- `ResponseItem::FunctionCallOutput`

这一步就在 `response_input_to_response_item(...)` 里。

逻辑很直接：

- 如果 MCP 调用成功，就把 `CallToolResult` 转成 `FunctionCallOutputPayload`
- 如果 MCP 调用失败，就把错误字符串塞进 `FunctionCallOutputPayload`
- 最后统一写成 `ResponseItem::FunctionCallOutput`

也就是说，对后续会话、rollout 和 replay 来说，真正长期保留下来的不是完整 begin/end 生命周期，而是：

- 一条普通 `FunctionCallOutput`
- 里面装着已经从 MCP 结果转换过来的内容

所以 Codex 的特点是：

- 运行时内部强类型
- 持久化历史里更普通、更像一般 tool 结果

#### 6.2.3 transcript / rollout 里实际保留了什么

Codex 的 rollout/历史记录会保留结果项，但不会把 MCP 的 begin/end 事件整条持久化下来。

这意味着它在 replay 时，通常能保住：

- 这次工具调用的输出
- 这次调用对应的结果语义
- 如果原始 MCP 结果里能转成普通 `content` / `content_items`，这些内容也会保住

但不保住：

- begin/end 的实时 UI 生命周期
- 一整条 MCP 专属事件流
- “这条历史记录原来是独立 MCP output 类型”这层身份

所以如果你只看 replay 后的历史，Codex 会显得比实时运行时“更普通”。

#### 6.2.4 截断和压缩怎么做

Codex 在这一层更像 runtime-first 的设计。

它对 MCP 结果会先做：

- 从 `McpToolCallOutput` 改写成 `FunctionCallOutput`
- 超长截断

然后再送回模型和历史主链。

这件事从测试就能看出来：

- MCP tool 输出过长时，会先被截断后再送给模型

也就是说，Codex 更倾向于：

- 尽早把所有 tool 结果收敛到统一的历史结构里
- 让后面的历史、replay、compact 主要处理 `FunctionCallOutput`
- 然后再做模型上下文控制

而不是像 Claude Code 那样，先给 MCP 大输出专门找一条持久化旁路。

#### 6.2.5 compact 时会发生什么

Codex 的 compact 重点不是“保留 MCP 身份”，而是“把对话继续压回可用上下文”。

一旦进入 compact，MCP 结果和其他历史内容一样，会继续朝下面这个方向走：

- 先按普通 `FunctionCallOutput` 来处理
- 再被总结成文本摘要

所以在 compact 之后，Codex 对 MCP 的处理更像：

- 保结果含义
- 不保那条独立的 MCP 生命周期结构

也就是说，Codex 的 MCP 独立性在实时 runtime 很强，但在 compact 后会明显下降。

#### 6.2.6 更关键的一点：compact 后，Codex 实际保留的历史非常少

Codex 的 compact 不是“把所有历史压短一点继续保留”。

它真正做的事情更接近：

1. 先跑一次 compact 任务，让模型生成一段总结
2. 然后重建一份新的 history
3. 这份新 history 里主要只保留：
   - 初始上下文
   - 少量最近的 user message
   - 一条 summary message

这里最重要的是：

- `collect_user_messages(...)` 只收集 user message
- `FunctionCallOutput` 不会被原样挑出来继续保留
- `McpToolCallOutput` 更不会以独立类型继续保留

也就是说，一旦进入 compact：

- 普通 tool 输出不会原样保留
- MCP tool 输出也不会原样保留
- 真正留下来的主要是“总结里有没有把它讲进去”

所以 Codex 的 compact 对 MCP tool 的真实效果是：

- MCP tool 的结果先进入运行时和历史
- 但 compact 之后，这些结果大多不会继续作为单独输出项存在
- 后面主要靠 summary 文本延续这些结果的语义

换句话说：

- Claude Code 压缩后，MCP 结果的消息痕迹更容易还在
- Codex 压缩后，MCP 结果通常不再作为独立输出项继续留在历史里
- Codex 后面更多依赖 compact summary 继续携带这些结果的含义

### 6.3 这两边的抽象差异到底体现在哪里

这件事可以直接压成一句：

- Claude Code：运行时和持久化主链都尽量沿用统一消息模型，MCP 信息作为 sidecar 保留下来
- Codex Rust：运行时先把 MCP 单独建模，进入历史和 compact 后再改写成普通历史结构

如果换成更白的话：

- Claude Code 更在意“结果落盘后还能保多少 MCP 细节”
- Codex 更在意“实时调用时能不能明确看出这是一次 MCP 调用”

### 6.4 所以 replay 之后，你还能看见什么

Claude Code 更像：

- 还能看见这是一次 `tool_result`
- 还能带着 `mcpMeta`
- 但实时 progress 已经不是主角

Codex 更像：

- 运行时你能明确看见这是 MCP begin/end/output
- replay 后主要剩结果，不太剩那条独立生命周期

这就是两边在“内部状态可见性”上的真正差异：

- Claude Code 把 MCP 状态融进统一消息主链
- Codex 把 MCP 状态抬成运行时事件，但不强求这套事件完整穿透到 compact/replay

## 7. 这会如何影响实际体验

如果只看“正常调用成功”这一个瞬间，两边体验差异不一定大。

真正有差异的是这些场景：

- 权限提示是不是完全像内建 tool
- UI 是不是明确告诉你这是 MCP tool
- 断线和 auth 问题是谁来兜底
- 出问题时用户更先看到“统一错误提示”还是“明确的 MCP 调用边界”

### 7.1 Claude Code 的实际体验

更像：

- 一批动态接进来的普通工具
- 系统尽量帮你自动恢复
- 权限、进度、结果展示统一

### 7.2 Codex Rust 的实际体验

更像：

- 一个有明确来源的外部工具调用
- begin/end 和 server/tool 维度都比较清楚
- 更方便调试和追踪

## 8. 一句话总结

- Claude Code：`MCP tool -> 先变内部 Tool -> 再执行`
- Codex Rust：`MCP tool -> 保持 MCP payload -> Router / Handler / Session 执行`

如果目标是统一产品体验，Claude Code 这条路更顺。

如果目标是保持 core 边界清楚、事件链清楚，Codex Rust 这条路更稳。

## 9. 一个更准确的简化判断

如果只保留最核心的一句，可以这么理解：

- 两边本质上都是“模型调工具，工具再调 MCP server”
- 两边也都是“一个 MCP tool 对应模型侧一个可调用 tool”

真正的差别不在这里。

真正的差别在运行时抽象：

- Claude Code 在工具池构建阶段就把 MCP tool 包装成内部 `Tool`
- Codex Rust 在模型侧也把 MCP tool 展平成普通 function tool，但运行时内部仍把它当成 `ToolPayload::Mcp`

所以你的理解基本是对的，但要补一层限定：

- 不是只有 Claude Code 才让模型“看到普通工具”
- Codex 也会把 MCP tool 注册成普通 function tool 给模型
- 只是到了运行时内部，Claude Code 更接近“不区分”，Codex 更接近“继续区分”

再压缩一点：

- Claude Code：模型侧和运行时侧都尽量把 MCP tool 当普通 tool 用
- Codex Rust：模型侧当普通 tool 暴露，运行时侧再还原成 MCP tool 处理
