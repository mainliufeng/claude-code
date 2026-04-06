# Claude Code 与 Codex Rust 架构对比

这份文档把当前仓库 Claude Code 的架构，和 `/home/liufeng/Code/self/codex/codex-rs` 的 Rust 实现放在一起比较。

重点不是评判谁更好，而是看：

- 两者的共同抽象是什么
- 相同问题分别怎么落地
- 如果你要抽一个公共核心，哪些边界是稳定的

如果只想看 `skill` 这一条链，另见：

- [Claude-Code-与-Codex-Rust-Skill机制对比.md](/home/liufeng/Code/source/claude-code/docs/Claude-Code-与-Codex-Rust-Skill机制对比.md)

如果只想看 `MCP tool` 这一条链，另见：

- [Claude-Code-与-Codex-Rust-MCP-Tool机制对比.md](/home/liufeng/Code/source/claude-code/docs/Claude-Code-与-Codex-Rust-MCP-Tool机制对比.md)

## 1. 一句话对比

- Claude Code：更像“REPL 产品外壳驱动的 agent runtime”
- Codex Rust：更像“核心库驱动、前端可替换的 agent 平台”

## 2. 模块映射

| 主题 | Claude Code | Codex Rust | 观察 |
| --- | --- | --- | --- |
| 主循环 | `src/query.ts` | `core/src/codex.rs` + `tasks/regular.rs` | 两边都有中心编排器；Codex 更库化 |
| 模型适配 | `src/services/api/claude.ts` | `core/src/client_common.rs` + `codex-api/` | 两边都把工具和上下文编译成模型请求 |
| 历史管理 | `messages + attachments + compact boundary` | `ContextManager` | Claude 更分散，Codex 更集中 |
| tools | `Tool.ts / tools.ts / toolExecution.ts` | `spec / router / orchestrator / registry` | Codex 的执行管线分层更清晰 |
| compact | `budget + microcompact + full compact` | `compact task + replace_history` | Claude 更细粒度，Codex 更粗粒度 |
| skills | `skill listing/discovery + SkillTool` | `SkillsManager + explicit injection` | Claude 更引导式，Codex 更显式 |
| session/runtime | `REPL + AppStateStore` | `SessionState + SessionTask + TUI` | Claude 更 UI 驱动，Codex 更 runtime 驱动 |
| MCP | REPL runtime 里合并 | `mcp_connection_manager` 聚合 | Codex 更像基础设施层 |
| sandbox/approval | hooks + permission + tool execution | `ToolOrchestrator + execpolicy + sandbox crates` | Codex 更系统工具化 |

## 3. 主循环差异

### Claude Code

特点是：

- `query.ts` 中心化很强
- attachment、compact、tool execution、continuation 都在一条很厚的链上
- 更像产品逻辑逐步长大后的编排中心

### Codex Rust

特点是：

- `run_turn` 仍然中心化
- 但 `Prompt`、`ContextManager`、`ToolRouter`、`ToolOrchestrator`、`SessionTask` 边界更明显
- 更容易把同一核心挂到 `tui/exec/cli` 多前端上

## 4. 上下文系统差异

### Claude Code

上下文的特点是：

- attachment 很重
- skill listing / discovery 很重
- compact 不是单次 summarize，而是多阶段收缩
- tool result 是上下文系统的核心组成部分

### Codex Rust

上下文的特点是：

- `ContextManager` 更集中
- skill / user instructions 更直接转成 `ResponseItem::Message`
- compact 更像显式重写历史
- attachment 不是主角

## 5. 工具体系差异

### Claude Code

强项在：

- 面向产品交互的灵活性
- tool 和 attachment、memory、UI 的整合很深

代价是：

- 工具执行链和上下文链更黏

### Codex Rust

强项在：

- router / orchestrator / registry 分层清晰
- approval + sandbox + retry 统一放在执行编排层
- 更适合抽成核心库

代价是：

- 上下文和产品侧引导层不如 Claude Code 那么丰富

## 6. skills 差异

这是两边一个很关键的不同点。

### Claude Code

- 先通过 `skill_listing / skill_discovery` 告诉模型有哪些 skill
- 真调用 `SkillTool` 后再注入正文
- compact 后通过 invoked skills 状态保活

### Codex Rust

- skills 先由 `SkillsManager` 按 config layer stack 加载和缓存
- 只有显式 mention 的 skill 才通过 `build_skill_injections` 直接注入正文
- 更接近“显式 skill include”

## 7. 如果要抽一个公共核心，哪些边界最稳定

我认为两边共同稳定的核心抽象有这些：

1. `Session / Thread`
   负责 turn、历史、任务、恢复
2. `Prompt`
   统一承接模型输入
3. `Tool Router / Registry`
   统一承接能力暴露和派发
4. `Tool Execution Orchestrator`
   承接 approval / sandbox / retry
5. `History Manager`
   负责规范化、裁剪、compact
6. `Skill Manager`
   负责 skill discover/load/inject

## 8. 哪边更适合做库

如果问题是“哪边更像可以抽出来做核心 agent 库的底盘”，我的判断是：

- Codex Rust 更像现成的核心库架构

原因是：

- `core/` 作为 workspace 主体已经很明显
- TUI/exec/cli 都在挂这套 core
- tool / session / compact / config / sandbox 的边界更系统化

而 Claude Code 更像：

- 一个功能很强的 agent 产品
- 但 REPL/runtime、attachment、state、tool/UI 的耦合更深

## 9. 哪边更适合研究产品形态

如果问题是“哪边更适合研究 coding agent 产品最终体验怎么做”，我的判断是：

- Claude Code 更值得看产品形态

原因是：

- skill listing / discovery
- attachment 桥接层
- memory / compact / task / UI / remote 的整合
- 面向交互体验的细节非常多

## 10. 最后的结论

最粗暴但也最准确的总结是：

- Claude Code 更像“产品完成度很高的 agent app”
- Codex Rust 更像“工程边界更适合抽核的 agent runtime”

如果你要做：

- 产品复刻：Claude Code 更值得参考
- 核心库设计：Codex Rust 更值得参考
- 最终方案：大概率应该参考 Codex Rust 的核心边界，再吸收 Claude Code 的产品层经验

## 11. 哪个对上下文缓存更友好

这个问题不能只回答一句“谁更好”，因为至少有两个维度：

- 面向模型提供方的 prompt cache 优化
- 面向系统内部的上下文结构稳定性

### 11.1 如果说的是 prompt cache 工程优化

我的判断是：

- Claude Code 更友好

原因不是它“更简单”，而是它明显做了更多缓存导向的工程优化。

从代码上看，Claude Code 对缓存友好的设计是成体系的：

- [src/tools.ts](/home/liufeng/Code/source/claude-code/src/tools.ts)
  明确为了 prompt cache 稳定性，保持 built-in tools 的顺序和连续性。
- [src/utils/api.ts](/home/liufeng/Code/source/claude-code/src/utils/api.ts)
  把 tool schema 的稳定部分和每次请求变化部分分开，避免不必要地打坏缓存键。
- [src/services/api/claude.ts](/home/liufeng/Code/source/claude-code/src/services/api/claude.ts)
  明确处理 `prompt_caching_scope`、`defer_loading`、tool search、tool hash、cache marker。
- [src/services/compact/microCompact.ts](/home/liufeng/Code/source/claude-code/src/services/compact/microCompact.ts)
  有专门的 cached microcompact 路径，通过 `cache_edits` / `cache_reference` 清理旧 tool result，而不是粗暴改写整段历史。
- [src/query.ts](/home/liufeng/Code/source/claude-code/src/query.ts)
  把 microcompact 放在主循环里，直接参与缓存友好的上下文治理。

也就是说，Claude Code 不是“顺便支持缓存”，而是把：

- tool 暴露
- 请求编译
- compact
- 历史裁剪

都按“尽量保住已缓存前缀”这个目标在设计。

### 11.2 如果说的是结构上更容易形成稳定上下文

这个维度下，Codex Rust 有优势，但不是因为它缓存优化更强，而是因为它结构更规整。

从代码上看：

- [/home/liufeng/Code/self/codex/codex-rs/core/src/client.rs](/home/liufeng/Code/self/codex/codex-rs/core/src/client.rs)
  明确给 Responses 请求设置 `prompt_cache_key`。
- [/home/liufeng/Code/self/codex/codex-rs/codex-api/src/requests/responses.rs](/home/liufeng/Code/self/codex/codex-rs/codex-api/src/requests/responses.rs)
  `ResponsesRequestBuilder` 把 `prompt_cache_key` 当成正式请求字段。
- [/home/liufeng/Code/self/codex/codex-rs/core/src/compact.rs](/home/liufeng/Code/self/codex/codex-rs/core/src/compact.rs)
  明确在 compact 时从历史前部裁剪，以保留前缀缓存。

但 Codex Rust 这套更像：

- 保持 `Prompt` 结构集中
- 保持 `ContextManager / compact / request builder` 的边界干净
- 用更规整的历史重写方式维持输入稳定

它确实更利于形成稳定请求，但没有看到像 Claude Code 那样大规模、细粒度地围绕 provider prompt cache 做专门优化。

### 11.3 所以结论要拆开说

- 如果你问“哪个对模型侧 prompt cache 更友好”，答案是 Claude Code。
- 如果你问“哪个更容易抽出一个结构稳定、便于后续做缓存优化的核心”，答案更偏向 Codex Rust。

### 11.4 对你的目标意味着什么

如果你的目标是：

- 语言用 Go
- 业务向 Claude Code 看齐
- 抽象向 Codex Rust 看齐

那缓存这一块最合理的策略不是二选一，而是组合：

- 核心结构学 Codex Rust
  让 `Prompt / Context / Compact / RequestBuilder` 的职责干净
- 缓存优化学 Claude Code
  让 tool schema、历史裁剪、compact、prefix 保持都围绕缓存命中率来设计

一句话总结：

- Claude Code 更“缓存工程化”
- Codex Rust 更“缓存结构化”
