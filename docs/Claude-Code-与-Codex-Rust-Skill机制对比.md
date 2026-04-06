# Claude Code 与 Codex Rust Skill 机制对比

这份文档只看 `skill` 这一条链。

关注点只有四个：

- skill 列表什么时候给模型
- skill 正文什么时候给模型
- skill 是否进入独立调用链
- compact / recovery 是否特殊处理 skill

文件规范本身不是重点。

现在公开规范已经比较明确：

- skill 是一个目录
- 入口文件是 `SKILL.md`
- 可以附带 `scripts/`、`references/`、`assets/`

参考：

- `https://github.com/anthropics/skills`
- `https://agentskills.io/specification`

真正的差别在运行时。

## 1. Codex Rust：skill 先是 catalog，再是按需正文注入

### 1.1 loader 在做什么

Codex 的 `loader` 先把 skill 从多层来源扫出来，再按名称和作用域去重。

关键文件：

- `core/src/skills/loader.rs`
- `core/src/skills/manager.rs`

这里拿到的是 catalog，不是直接给模型的正文。

loader 关心的是：

- skill 从哪里来
- 多来源重名时保留哪一个
- 当前 cwd 下最终可见的 skill 集合是什么

### 1.2 默认给模型的是什么

Codex 默认不会把 `SKILL.md` 全文塞进上下文。

默认给模型的是一份 skills section，挂在 user instructions / project doc 里。

关键文件：

- `core/src/project_doc.rs`
- `core/src/skills/render.rs`

这份 section 包含：

- skill 名
- skill 描述
- skill 路径
- how-to-use 规则

所以 Codex 的默认行为是：

- 先把可用 skill catalog 给模型
- 让模型知道系统里有哪些 skill
- 但不提前支付 skill 正文的上下文成本

### 1.3 正文什么时候进上下文

Codex 主要走显式触发。

前端会识别输入中的 `$skill-name`，然后把它变成 `UserInput::Skill`。

关键文件：

- `tui/src/chatwidget.rs`
- `tui2/src/chatwidget.rs`

后面 `build_skill_injections(...)` 再去读取对应 `SKILL.md`，把正文注入当前 turn。

关键文件：

- `core/src/skills/injection.rs`

注入后的形式不是普通文本拼接，而是一个单独的 `<skill>...</skill>` 片段，里面带：

- `<name>`
- `<path>`
- skill 正文

这条链的含义是：

- listing 和正文是分开的
- 正文进入上下文有明确触发条件
- skill 注入是一次受控 include

### 1.4 skill 对后续轮次怎么生效

Codex 里的 skill 不是“只影响当前轮”。

一旦 skill 正文进了消息历史，后面的轮次当然还能继续看到它。
但这种持续性主要来自普通历史继承。

也就是说：

- 没有单独的 invoked skill 状态
- 没有专门围绕 skill 做 compact 保活
- 没有围绕 skill 单独做 resume 重建

skill 在 Codex 里更像一个受控的上下文输入源。

## 2. Claude Code：skill 已经进入主循环

### 2.1 loader 在做什么

Claude Code 的 loader 比 Codex 重很多。

关键文件：

- `src/skills/loadSkillsDir.ts`
- `src/commands.ts`

它不是简单列目录，而是在做运行时前置整理：

- 去重
- unconditional skill / conditional skill 分流
- `paths` frontmatter 激活条件
- dynamic skill
- MCP skill
- 进一步过滤哪些 skill 能进 `SkillTool`

这里还有两个具体约束：

- `getSkillToolCommands(...)` 不会把所有 prompt command 都给模型，还会按 `disableModelInvocation`、description、`whenToUse`、source 继续过滤
- skill listing 不是无限展开，`SkillTool` prompt 里明确把总预算压在上下文窗口约 1%，单条描述硬截到 250 字符

相关文件：

- `src/tools/SkillTool/prompt.ts`

这说明 Claude Code 在 listing 阶段就已经把 skill 当成受预算约束的运行时对象，而不是简单目录。

### 2.2 默认给模型的是什么

Claude Code 默认给模型的 skill 信息有两层：

1. `skill_listing`
2. `skill_discovery`

关键文件：

- `src/utils/attachments.ts`
- `src/utils/messages.ts`

`skill_listing` 的作用是告诉模型：

- 当前有哪些 skill 可以用

`skill_discovery` 的作用是告诉模型：

- 当前任务相关的 skill 是哪一部分

这两层不是总是一起出现。

几个关键细节：

- `skill_listing` 只有在当前 agent 真正拥有 `SkillTool` 时才会进上下文
- 如果实验性 skill search 开启，listing 会进一步收缩
- `skill_discovery` 不是普通附件展示，它最终会被转成 meta user message，直接参与本轮推理

所以 Claude Code 在正文注入之前，已经先做了一层“能力暴露”，又做了一层“任务导向”。

### 2.3 正文什么时候进上下文

正文通常不直接跟着 listing 一起进来。

Claude Code 让模型先走 `SkillTool`。

关键文件：

- `src/tools/SkillTool/SkillTool.ts`

`SkillTool` 做的不是“读一下 `SKILL.md`”这么简单。

它至少承担四类动作：

1. 合法化  
   检查 skill 当前是否存在、是否允许被模型调用。

2. 运行时校验  
   接权限判断、避免重复运行、处理本地 skill 和远程 skill 的差异。

3. skill 展开  
   执行 prompt skill，把 skill 内容转成 `newMessages` 注入当前链路。

4. 副作用写回  
   skill 展开之后，可能还会改 allowed tools、模型覆盖等上下文条件。

所以 `SkillTool` 不是资料读取器。
它是一个运行时包装器。

这也是 Claude Code 和 Codex 最大的边界差别之一：

- Codex 的 skill 正文注入更像 include
- Claude Code 的 skill 正文注入是一次正式调用

### 2.4 compact 为什么要特殊处理 skill

Claude Code 的 compact 对 skill 有专门逻辑。

关键文件：

- `src/services/compact/compact.ts`
- `src/services/compact/postCompactCleanup.ts`

第一步，compact 前会主动剥掉：

- `skill_listing`
- `skill_discovery`

原因很明确：

- 这两类信息本来就会重新生成
- 喂给 summarizer 只会污染摘要
- 还会额外浪费 token

第二步，compact 后不会重置 `sentSkillNames`。

这不是遗漏，是明确设计。

原因在注释里写得很清楚：

- 重发整份 `skill_listing` 大概是 4K token 量级
- 这对用户没新增信息
- 但会制造 prompt cache miss

第三步，compact 后不会保整份 listing，而是只保真正被调用过的 skill。

对应做法是：

- 把 invoked skill 收成 `invoked_skills` attachment
- attachment 里保留 skill 名、路径、正文、调用时间、agent

关键文件：

- `src/bootstrap/state.ts`
- `src/utils/attachments.ts`
- `src/utils/messages.ts`

这说明 Claude Code 在 compact 阶段并不是“压完再说”。
它在主动区分：

- 哪些 skill 信息是可再生的提示层
- 哪些 skill 信息是必须保住的运行时状态

### 2.5 recovery 为什么也要单独处理 skill

恢复会话时，Claude Code 不只反序列化普通消息。

它会额外扫描历史中的 `invoked_skills` attachment，把这些 skill 重新装回内存状态。

关键文件：

- `src/utils/conversationRecovery.ts`

恢复链里两个动作很关键：

1. 看到 `invoked_skills`  
   调 `addInvokedSkill(...)`，把 skill 名、路径、正文重新装回进程内 `invokedSkills` map。

2. 看到历史里已有 `skill_listing`  
   调 `suppressNextSkillListing()`，避免恢复后立刻重复打一遍 listing。

这一步的含义很直接：

- Claude Code 里的 skill 不只是“历史里出现过的文本”
- 它是可以被恢复的运行时对象

## 3. 这两套实现到底差在哪

如果只看 `SKILL.md`，差别不大。

如果看运行时，差别很大。

Codex 的抽象链是：

- loader / catalog
- listing
- explicit injection

Claude Code 的抽象链是：

- loader / activation
- listing / discovery
- SkillTool 调用
- invoked skill state
- compact / recovery 专门处理

所以两边不是“都支持 skill，只是注入方式不同”。

更准确的说法是：

- Codex 把 skill 放在上下文装配层附近
- Claude Code 把 skill 推进了提示层、调用层、状态层

这也是为什么 Claude Code 的 skill 复杂度不在文件本身，而在主循环里。

## 4. 代码入口索引

### Claude Code

- `src/skills/loadSkillsDir.ts`
- `src/commands.ts`
- `src/utils/attachments.ts`
- `src/utils/messages.ts`
- `src/tools/SkillTool/SkillTool.ts`
- `src/tools/SkillTool/prompt.ts`
- `src/bootstrap/state.ts`
- `src/services/compact/compact.ts`
- `src/services/compact/postCompactCleanup.ts`
- `src/utils/conversationRecovery.ts`

### Codex Rust

- `core/src/skills/loader.rs`
- `core/src/skills/manager.rs`
- `core/src/project_doc.rs`
- `core/src/skills/render.rs`
- `core/src/skills/injection.rs`
- `tui/src/chatwidget.rs`
- `tui2/src/chatwidget.rs`
