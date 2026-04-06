# 测试策略与开发用 CLI

这份文档补的是 Go agent core 的验证路径。

目标不是把产品 CLI 补回来，而是定义一套：

- 能长期支撑核心库演进的测试分层
- 能快速复现 turn 行为的开发用入口
- 能稳定做回归的 transcript / golden 测试机制

## 1. 先给结论

建议做一个非常薄的内部 CLI，但把它严格定位为：

- `dev harness`
- `test harness`
- `replay / inspect / run` 入口

不把它当产品形态，不把核心逻辑写回 CLI。

也就是说：

- `agentcore` 仍然是主角
- CLI 只是 `agentcore` 的一个测试外壳

## 2. 为什么只做库还不够测

如果只有库接口，没有一个可执行入口，会有几类问题很难验证：

- 一次 turn 的完整链路
  `context -> model -> tool call -> tool result -> compact -> 下一轮`
- skill 注入是否符合预期
- tool result 写回 transcript 后，compact 是否正确处理
- MCP bridge 和本地支持模块是否真的协同工作
- approvals / sandbox / policy 是否在正确位置触发
- session resume / transcript replay 是否可复现

这些都不是单一包的单元行为，而是集成行为。

## 3. 建议的测试分层

### 3.1 单元测试

目的：

- 验证局部规则
- 提高重构安全性
- 快速定位纯逻辑错误

建议覆盖这些包：

- `agentcore/context`
  上下文拼装、来源合并、顺序稳定性
- `agentcore/compact`
  tool result 裁剪、rolling summary、hard budget compact
- `agentcore/tooling`
  tool registry、tool routing、并发/串行策略
- `agentcore/approval`
  allow / deny / ask 决策
- `agentcore/session`
  turn、checkpoint、resume 状态变化
- `support/skillfs`
  skill 扫描、frontmatter 解析、注入构造
- `support/toolpatch`
  patch 应用、diff 结果、冲突处理

这层尽量不用真实模型、真实 bash、真实 MCP。

## 3.2 集成测试

目的：

- 验证核心库和支持模块的编排关系
- 验证一个 turn 是否走完

建议方式：

- 直接调用 `agentcore.Engine`
- 注入 fake / test double：
  - `ModelProvider`
  - `SessionStore`
  - `ArtifactStore`
  - `ApprovalManager`
  - `MCPBridge`
- 只在必要处接真实实现：
  - skill loader
  - file patcher
  - transcript builder

建议重点验证：

- 模型返回 tool call 后能否正确路由
- tool result 是否正确回写为下一轮上下文
- compact 前后消息顺序是否稳定
- memory 提炼是否只保留结构化事实
- 子 agent / task 返回后是否能继续主链路

## 3.3 端到端测试

目的：

- 从真实入口跑完整回路
- 回归测试 prompt、tools、skills、compact 的协同行为

这层最适合通过开发用 CLI 做。

## 4. 开发用 CLI 应该长什么样

建议单独放一个很薄的包，例如：

```text
/cmd/agentdev
```

或者：

```text
/internal/devharness
/cmd/agentdev
```

职责只有三类：

- `run`
  给一段输入、一个 workspace、一个 config，跑一次真实 turn
- `replay`
  回放 fixture / transcript / recorded model events
- `inspect`
  打印 prompt、context sources、tools、compact 前后差异

不要做：

- 产品级配置系统
- 正式 TUI
- 正式用户交互体验
- 复杂命令体系

## 5. 推荐命令形态

### 5.1 `agentdev run`

用于本地跑一次或几次 turn。

建议参数：

- `--prompt`
- `--workspace`
- `--session`
- `--model`
- `--skill-root`
- `--toolset`
- `--dump-transcript`

适合做：

- 快速人工验证
- 复现 bug
- 看真实 tool call 链

### 5.2 `agentdev replay`

用于回放固定输入和固定模型事件。

建议参数：

- `--fixture`
- `--update-golden`

适合做：

- 稳定回归测试
- compact / memory / skill 注入验证
- prompt 结构变化检测

### 5.3 `agentdev inspect`

用于只看过程，不跑真实外部副作用。

建议输出：

- system / developer / user prompt 片段
- 注入了哪些 skill
- 暴露了哪些 tools
- compact 前后的消息数量与 token 估算
- 最终发送给模型的结构化请求

## 6. 最有价值的测试资产：fixture + golden

建议把大部分复杂行为测试做成 fixture 驱动：

```text
/testdata/fixtures/basic-turn/
/testdata/fixtures/skill-injection/
/testdata/fixtures/tool-result-compaction/
/testdata/fixtures/subagent-return/
/testdata/fixtures/session-resume/
```

每个 fixture 可包含：

- `input.json`
- `workspace/`
- `skills/`
- `model_events.json`
- `expected_transcript.json`
- `expected_final.txt`
- `expected_prompt.json`

这样做有两个好处：

- 行为可回放
- 回归差异可直接做文本比较

## 7. 哪些地方最适合做 golden 测试

优先级最高的是：

1. prompt 组装结果
   因为上下文、skill、tool、compact 很容易相互影响
2. tool result 回流后的 transcript
   因为它直接决定下一轮模型能看到什么
3. compact 后历史
   因为这是最容易“没报错但行为悄悄变坏”的部分
4. session resume 后的首轮输入
   因为恢复链最容易丢 skill / memory / summary

## 8. 模型测试怎么做

建议分三层：

### 8.1 Stub model

用于包级单元测试。

只返回固定事件：

- 文本输出
- 单个 tool call
- 多个 tool call
- 错误 / 中断

### 8.2 Recorded model

用于集成和 replay 测试。

把历史请求和响应录成 fixture，然后稳定回放。

这层最适合验证：

- prompt 是否变化
- tool schema 是否变化
- compact 后输入是否变化

### 8.3 Real model smoke test

只做少量 smoke test，不作为主要回归手段。

用途：

- 验证 provider adapter 没坏
- 验证 streaming / tool calling / auth 没坏

不要把这层当主回归集，因为：

- 不稳定
- 成本高
- 输出易漂移

## 9. 支持模块怎么测

你的架构里，支持模块和核心分离，所以测试也要分开。

### 9.1 skill

重点测：

- skill 扫描优先级
- frontmatter 解析
- 相对路径引用
- 注入文本生成

### 9.2 文件系统

重点测：

- 本地文件实现
- overlay / mock 文件实现
- patch/edit 结果
- 路径边界和软链处理

### 9.3 bash / command runner

重点测：

- stdout / stderr 捕获
- timeout
- cancel
- exit code 传播

### 9.4 MCP bridge

重点测：

- tools/resources/prompts 发现
- tool call 往返
- 连接断开重试
- schema 转换稳定性

## 10. 推荐的验证顺序

最实用的推进方式是：

1. 先把核心库单元测试补齐
2. 再给 `Engine` 做 fake-model 集成测试
3. 再做一个极薄的 `agentdev`
4. 再用 `agentdev replay` 搭 golden 回归
5. 最后才加少量真实模型 smoke test

这个顺序能避免一开始就被真实模型和复杂外部依赖拖死。

## 11. 和当前设计的关系

这个开发用 CLI 不改变前面的核心边界：

- `agentcore` 仍然只依赖接口
- `support` 仍然提供 skill、tool、fs、bash、mcp 等实现
- `storage` 仍然负责 transcript / artifact / memory 持久化

CLI 只是把这些东西装起来，形成一个最小可运行外壳。

## 12. 最短结论

是的，写好核心库后，最好补一个很薄的开发用 CLI。

但它应该是：

- 测试入口
- 调试入口
- replay 入口

而不是产品 CLI。
