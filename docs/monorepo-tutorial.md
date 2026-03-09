# Pi Monorepo Tutorial

这份文档面向第一次接触 `pi-mono` 的开发者，目标不是重复每个 package README 的 API 细节，而是先回答这几个更重要的问题：

1. 这个 monorepo 到底在解决什么问题？
2. 每个 package 在整体架构里扮演什么角色？
3. 一条用户请求是如何在不同包之间流动的？
4. 仓库里反复出现的关键概念分别是什么意思？
5. 想继续深入源码，应该先看哪里？

如果你已经看过根目录的 [README](../README.md)，这份 tutorial 可以作为它的“架构版导读”。

## 1. 仓库总览：它是什么？

`pi-mono` 是一个围绕“可构建、可扩展的 AI agent”展开的 monorepo。它不是单一产品，而是一组可以相互组合的基础设施：

- `packages/ai`：统一的多模型、多 provider LLM API
- `packages/agent`：带工具调用和事件流的 agent runtime
- `packages/tui`：终端 UI 基础组件
- `packages/web-ui`：Web chat UI 组件
- `packages/coding-agent`：交互式 coding agent CLI
- `packages/mom`：把 agent 接到 Slack 的 bot
- `packages/pods`：在 GPU pod 上部署和管理 vLLM 的 CLI

从抽象层次看，它大致可以分成四层：

```text
应用层
├─ packages/coding-agent   交互式终端 coding agent
├─ packages/mom            Slack bot
└─ packages/pods           GPU/vLLM 部署与测试 CLI

界面层
├─ packages/tui            终端 UI 基础设施
└─ packages/web-ui         Web chat UI 组件与存储

运行时层
└─ packages/agent          Agent loop、工具执行、事件流

模型接入层
└─ packages/ai             多 provider LLM 抽象与统一流式接口
```

这是理解整个仓库最重要的一点：**上层 package 通常不是“重新实现一套 agent”，而是在复用下层 package 的能力。**

## 2. Monorepo 结构怎么读？

仓库根目录最值得先看的文件有：

- [`README.md`](../README.md)：仓库入口，说明有哪些 package
- [`CONTRIBUTING.md`](../CONTRIBUTING.md)：贡献要求
- [`AGENTS.md`](../AGENTS.md)：面向 agent/human 的开发规则
- `package.json`：workspace、根脚本、构建顺序
- `tsconfig.base.json`、`biome.json`：TypeScript 和格式/检查规则

工作区由 npm workspaces 管理，根 `package.json` 会串起各个子包。`build` 脚本按依赖顺序构建：

```text
tui → ai → agent → coding-agent → mom → web-ui → pods
```

这个顺序本身就暴露了仓库的依赖关系：底层基础设施先构建，上层应用后构建。

## 3. 每个 package 的职责

### 3.1 `packages/ai`：统一 LLM API

`@mariozechner/pi-ai` 是整个仓库的模型接入层。它做的不是“只包装某一家 API”，而是把不同 provider 的差异压平到统一接口里。

它重点解决四类问题：

- **Provider 抽象**：OpenAI、Anthropic、Google、Bedrock、OpenRouter 等统一到一个入口
- **Model 注册与查询**：维护支持工具调用的模型列表
- **统一流式事件**：把不同 provider 的 streaming 响应转换为统一事件
- **工具调用与上下文**：用统一的数据结构表达 messages、tools、tool results

建议先看这些文件：

- `packages/ai/src/index.ts`：公共导出总表
- `packages/ai/src/types.ts`：核心类型
- `packages/ai/src/stream.ts`：统一流式入口
- `packages/ai/src/models.ts` / `models.generated.ts`：模型注册
- `packages/ai/src/providers/`：各 provider 实现
- `packages/ai/src/env-api-keys.ts`：环境变量/API key 解析

#### 关键概念：Model / Provider / API

在这个仓库里，这三个概念经常一起出现，但并不完全相同：

- **Provider**：服务提供方，如 OpenAI、Anthropic、Google
- **Model**：具体模型，如 `gpt-4o-mini`、`claude-sonnet-*`
- **API**：接入方式或协议形态，如 OpenAI responses、chat completions、Anthropic stream

也就是说，这里不是“写死某个 HTTP 接口”，而是先抽象 provider 和 model，再映射到底层 API。

#### 关键概念：Context

`pi-ai` 中非常重要的概念是 `Context`。它通常包含：

- `systemPrompt`
- `messages`
- `tools`

这使得上下文天然可以：

- 直接传给 LLM
- 序列化保存
- 在不同模型/不同 provider 之间 handoff

因此 `packages/ai` 不只是“发请求”，它也是会话上下文的标准化层。

#### 关键概念：流式事件

`stream()` 是理解 `pi-ai` 的核心入口之一。统一后，调用方不必关心 provider 原始协议差异，而是消费统一事件，例如：

- `start`
- `text_start` / `text_delta` / `text_end`
- `thinking_start` / `thinking_delta` / `thinking_end`
- `toolcall_start` / `toolcall_delta` / `toolcall_end`
- `done`
- `error`

这也是为什么上层的 `agent`、`coding-agent`、`web-ui` 都能建立一致的“流式体验”。

### 3.2 `packages/agent`：agent runtime

`@mariozechner/pi-agent-core` 建立在 `pi-ai` 之上，负责把“LLM 调一次”升级成“可以执行多轮工具调用的 agent loop”。

如果说 `pi-ai` 负责“模型如何说话”，那 `pi-agent-core` 负责“agent 如何工作”。

建议先看：

- `packages/agent/src/agent.ts`
- `packages/agent/src/agent-loop.ts`
- `packages/agent/src/types.ts`
- `packages/agent/src/proxy.ts`
- `packages/agent/README.md`

#### 它解决了什么问题？

- 管理 agent state
- 维护消息上下文
- 处理工具调用和工具结果
- 在 UI 层可消费的粒度上发出事件
- 支持 steering / follow-up 这样的多消息队列机制

#### 关键概念：AgentMessage vs LLM Message

这是整个仓库里最重要的概念之一。

`AgentMessage` 是更通用的运行时消息类型，它不仅能表达标准 LLM 消息，也能表达应用自定义消息。  
但 LLM 实际只认识有限几类消息（如 `user`、`assistant`、`toolResult`）。

因此 agent 会经历类似这样的处理链路：

```text
AgentMessage[]
  → transformContext()
  → convertToLlm()
  → Message[]
  → stream()/complete()
```

这里：

- `transformContext()`：用于裁剪、注入外部上下文、做 compaction 前处理
- `convertToLlm()`：把应用层消息转成 LLM 真正能理解的消息

这层分离非常关键，因为它允许 UI 或应用保存更丰富的会话信息，但只把必要部分发送给模型。

#### 关键概念：Event-driven agent

`Agent` 不是只返回“最终答案”，而是持续发出事件。常见事件包括：

- `agent_start` / `agent_end`
- `turn_start` / `turn_end`
- `message_start` / `message_update` / `message_end`
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end`

这使得所有 UI 层都可以用同一种方式订阅 agent 的生命周期。

#### 关键概念：Steering 和 Follow-up

在 `pi-coding-agent` 或其他交互式 UI 里，用户不一定只能“等 agent 完全结束再继续说话”。  
仓库通过两种队列机制表达这种交互：

- **Steering**：打断/改道，通常在当前工具执行阶段后尽快插入
- **Follow-up**：等当前工作彻底完成后再追加

这是一个非常“agent runtime”而不是“普通聊天应用”的设计点。

### 3.3 `packages/tui`：终端 UI 基础设施

`@mariozechner/pi-tui` 是终端 UI 库，不是专门服务于 coding agent 的私有实现。它被设计成可以构建泛化的交互式 CLI。

建议先看：

- `packages/tui/src/index.ts`
- `packages/tui/src/tui.ts`
- `packages/tui/src/components/`
- `packages/tui/src/terminal.ts`
- `packages/tui/src/keybindings.ts`

#### 它的核心价值

- 差量渲染，减少不必要刷新
- 同步输出，降低闪烁
- 支持 overlay/dialog 等交互模式
- 提供常用组件：`Editor`、`Input`、`Markdown`、`SelectList` 等
- 支持终端图片协议

#### 关键概念：Component / Focus / Overlay

`pi-tui` 的抽象非常朴素：

- 组件通过 `render(width)` 生成行数组
- 组件可选实现 `handleInput`
- TUI 容器负责焦点、重绘、overlay 管理

这让上层可以实现复杂交互，但仍保持较低耦合。

对于 `pi-coding-agent` 来说，`pi-tui` 提供的是“可组合的交互界面原语”。

### 3.4 `packages/web-ui`：Web chat UI 组件

`@mariozechner/pi-web-ui` 是与 `pi-tui` 对应的 Web 界面层。它把 `agent` 的事件流包装成可直接落地的前端组件与存储系统。

建议先看：

- `packages/web-ui/src/index.ts`
- `packages/web-ui/src/ChatPanel.ts`
- `packages/web-ui/src/components/AgentInterface.ts`
- `packages/web-ui/src/storage/`
- `packages/web-ui/src/tools/`

#### 它提供什么？

- `ChatPanel`：高层 chat UI
- `AgentInterface`：更底层、可自定义布局的聊天组件
- `AppStorage` + IndexedDB backend：本地会话与设置存储
- 附件处理、artifact 渲染、REPL/document extraction 等工具

#### 关键概念：默认消息转换

`web-ui` 不仅是“显示消息”，它还扩展了消息类型，比如：

- 带附件的用户消息
- artifact 消息

因此它也需要自己的 `convertToLlm` 策略（如 `defaultConvertToLlm`），把 UI 专属消息过滤或转换成 LLM 可处理的消息。

这再次印证了一个核心设计：  
**应用层消息可以比 LLM 消息更丰富，但真正发给模型前必须标准化。**

#### 关键概念：Storage 是一等公民

`web-ui` 把以下内容都当成正式的前端状态基础设施：

- 设置
- provider keys
- sessions
- custom providers

这说明它不是一个“只会渲染”的 UI 包，而是一个“可独立构建完整浏览器 AI 应用”的包。

### 3.5 `packages/coding-agent`：交互式 coding agent

`@mariozechner/pi-coding-agent` 是仓库里最“产品化”的部分之一，也是很多人第一次接触这个仓库时实际在使用的包。

建议先看：

- `packages/coding-agent/src/index.ts`
- `packages/coding-agent/src/main.ts`
- `packages/coding-agent/src/config.ts`
- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/extensions/`
- `packages/coding-agent/src/core/tools/`
- `packages/coding-agent/src/modes/`

#### 它是什么？

它是一个最小但高度可扩展的 terminal coding harness。  
默认提供基础 coding tools（如读写文件、编辑、bash），然后通过扩展系统把更多能力交给用户自己组合。

#### 四种运行模式

README 中明确说明它支持四种模式：

- **interactive**：交互式终端模式
- **print**：一次性文本输出
- **JSON**：事件流输出，方便机器消费
- **RPC / SDK**：进程或程序化集成

这说明它并不是“只有一个 CLI 表面”，而是同一个 runtime 可以挂接多种交互方式。

#### 关键概念：Session

`coding-agent` 的核心不是简单的聊天记录，而是一个可分叉、可压缩、可恢复的 session 系统。

相关关键能力包括：

- 自动保存
- `resume`
- tree view
- fork
- compaction

#### 关键概念：Session Tree

与很多聊天产品只保留线性历史不同，这里会话是树状的。  
每条 entry 带 `id` 和 `parentId`，所以你可以：

- 回到任意历史节点继续
- 在同一个 session 文件里保留多条分支
- 对某个分支做 fork

这非常适合 coding/agent 工作流，因为实验、回退、另起分支是高频操作。

#### 关键概念：Compaction

Agent 工作很容易撞上 context window 上限，所以 `coding-agent` 内置 compaction 逻辑：

- 保留近期关键上下文
- 汇总较旧的上下文
- 尽量不丢失任务连续性

你可以把它理解为“把一个长期 agent session 保持在模型能接受的上下文预算内”。

#### 关键概念：Extension / Skill / Prompt Template / Theme / Pi Package

这是 `coding-agent` 与其他类似工具差异最大的部分。

- **Extension**：运行时扩展，可注册工具、UI、命令、钩子
- **Skill**：给模型看的任务能力片段或工作流模板
- **Prompt Template**：可复用的 prompt 模板
- **Theme**：终端 UI 主题
- **Pi Package**：把上述资源打包分发的方式

换句话说，`coding-agent` 并不强调“官方内置一切”，而是强调“保持核心最小，让能力通过扩展系统长出来”。

### 3.6 `packages/mom`：Slack bot 应用

`@mariozechner/pi-mom` 是把 agent 带入团队沟通场景的应用层包。

建议先看：

- `packages/mom/README.md`
- `packages/mom/src/`
- `packages/mom/src/tools/`

#### 它的定位

Mom 是一个运行在 Slack 上、具备 bash/文件能力的 agent，并且强调：

- 自管理
- 工作空间持久化
- 每个频道单独上下文
- 通过日志和 memory 文件保留长期记忆

#### 关键概念：工作区就是记忆载体

Mom 的数据目录中会保存：

- 全量日志 `log.jsonl`
- 给 LLM 看的上下文 `context.jsonl`
- 附件
- scratch 空间
- 自己创建的技能/工具
- MEMORY.md

这让它不是“一个纯在线 bot”，而是“一个拥有本地持久工作区的 agent”。

#### 为什么这很重要？

因为这延续了整个仓库的共同设计倾向：  
**尽量把 agent 的状态做成显式、可检查、可持久化的文件或结构，而不是藏在不可见的运行时里。**

### 3.7 `packages/pods`：vLLM / GPU pod 管理

`pi` 在这个包里更偏向“基础设施工具”而不是“通用聊天应用”。

建议先看：

- `packages/pods/README.md`
- `packages/pods/src/cli.ts`
- `packages/pods/src/commands/`
- `packages/pods/src/model-configs.ts`
- `packages/pods/src/ssh.ts`

#### 它解决的问题

- 配置远端 pod
- 启动/停止模型
- 管理 vLLM 参数
- 在已部署模型上测试 agent
- 统一 OpenAI-compatible 接口接入

#### 为什么它在这个 monorepo 里？

因为这个仓库并不只关心“agent runtime”，也关心“agent 使用的模型如何被部署和接入”。  
`pods` 让这个生态覆盖到了运行环境层。

## 4. 一条请求如何在仓库里流动？

这是理解架构的最佳方式。以 `coding-agent` 为例，一条用户请求通常遵循下面的路径：

```text
用户在 interactive TUI 输入请求
  ↓
packages/coding-agent
  - 把输入写入 session
  - 装配 system prompt / skills / tools / settings
  ↓
packages/agent
  - Agent 接收 prompt
  - 触发 turn
  - 调用 LLM
  - 处理 tool calls / tool results
  - 发出事件
  ↓
packages/ai
  - 选择 provider/model
  - 发送流式请求
  - 统一返回 text/thinking/toolcall 等事件
  ↓
packages/agent
  - 汇总事件、更新状态
  ↓
packages/tui
  - 把事件渲染为消息、工具输出、状态变化
```

如果是 Web 应用，最后一层就从 `tui` 换成 `web-ui`；  
如果是 Slack bot，则是 `mom` 在应用层消费这些能力。

## 5. 仓库反复出现的关键概念

下面这些概念贯穿多个 package，是读源码时必须建立的共同语言。

### 5.1 Tool

Tool 不只是一个函数名，它通常包含：

- 名称
- 描述
- 参数 schema（通常用 TypeBox）
- 执行逻辑

Tool 的意义在这个仓库里非常核心，因为几乎所有 agentic workflow 都依赖“模型先产出 tool call，再由 runtime 执行”。

### 5.2 Event Stream

无论是模型流式响应，还是 agent 执行过程，事件流都是主干。

你可以大致把它分成两层：

- **LLM 事件**：来自 `pi-ai` 的 text/tool/thinking/usage 等
- **Agent 事件**：来自 `pi-agent-core` 的 turn/message/tool_execution 等

界面层依赖这些事件实现渐进式更新，而不是等最终结果一次性渲染。

### 5.3 Context 和 Serialization

仓库非常强调 context 的结构化与可序列化。这带来几个直接好处：

- 可以保存 session
- 可以在模型间 handoff
- 可以做 compaction
- 可以让应用层在不同 transport/UI 之间复用

### 5.4 Custom Message Types

仓库没有把消息系统设计成僵硬的三种角色，而是允许应用层定义更多消息角色。  
这样：

- UI 可以有自己的消息类型
- 业务应用可以保存更多上下文
- 发送给 LLM 之前再统一收敛

这是一种很重要的“运行时消息模型 > LLM 消息模型”的设计。

### 5.5 Session

在 `coding-agent` 和 `mom` 里，session 都不是一个简单数组，而是带结构、带状态的长期工作单元。

它通常承载：

- 对话历史
- 工具结果
- 统计信息
- 分支信息
- 压缩摘要

### 5.6 Compaction

Agent 不是一次性问答，而是长流程工作。  
只要工作足够长，就一定会碰到 context budget 问题。  
因此 compaction 不是附属功能，而是 agent 产品要可持续工作的基础设施。

## 6. 包之间的依赖关系

可以用一个更具体的方式记住各包关系：

```text
@mariozechner/pi-ai
  └─ 为所有上层提供统一 LLM 能力

@mariozechner/pi-agent-core
  └─ 基于 pi-ai 构建 agent runtime

@mariozechner/pi-tui
  └─ 为终端交互提供 UI 原语

@mariozechner/pi-web-ui
  └─ 为浏览器交互提供 UI、存储和工具展示

@mariozechner/pi-coding-agent
  └─ 组合 pi-agent-core + pi-tui + 自己的 session/extensions/tools

@mariozechner/pi-mom
  └─ 组合 pi-coding-agent / pi-agent-core / pi-ai，接入 Slack 场景

@mariozechner/pi-pods
  └─ 提供模型部署与远端运行环境管理，补足 agent 运行的基础设施链路
```

## 7. 如果你想开始读源码，推荐顺序是什么？

### 路线 A：想理解“核心架构”

1. 根 `README.md`
2. `packages/ai/README.md`
3. `packages/agent/README.md`
4. `packages/ai/src/types.ts`
5. `packages/ai/src/stream.ts`
6. `packages/agent/src/types.ts`
7. `packages/agent/src/agent.ts`
8. `packages/agent/src/agent-loop.ts`

这条路线能帮你建立“模型层 + runtime 层”的骨架。

### 路线 B：想理解“终端 coding agent 是怎么做出来的”

1. `packages/coding-agent/README.md`
2. `packages/tui/README.md`
3. `packages/coding-agent/src/main.ts`
4. `packages/coding-agent/src/core/agent-session.ts`
5. `packages/coding-agent/src/core/session-manager.ts`
6. `packages/coding-agent/src/core/extensions/`
7. `packages/coding-agent/src/core/tools/`
8. `packages/coding-agent/src/modes/interactive/`

### 路线 C：想理解“Web 端怎么接”

1. `packages/web-ui/README.md`
2. `packages/web-ui/src/ChatPanel.ts`
3. `packages/web-ui/src/components/AgentInterface.ts`
4. `packages/web-ui/src/components/Messages.ts`
5. `packages/web-ui/src/storage/`
6. `packages/web-ui/src/tools/`

### 路线 D：想理解“真实应用怎么接入 agent”

1. `packages/mom/README.md`
2. `packages/pods/README.md`

前者是团队协作/Slack 场景，后者是模型部署/运行环境场景。

## 8. 一个最实用的心智模型

如果要把这个 monorepo 压缩成一句话，可以这样记：

> `pi-ai` 负责和模型说话，`pi-agent-core` 负责让模型会工作，`pi-tui` / `pi-web-ui` 负责让人能和 agent 交互，`coding-agent` / `mom` / `pods` 负责把这些能力变成具体产品与工作流。

再换一种更工程化的表达：

- **`pi-ai`**：统一模型协议层
- **`pi-agent-core`**：agent 编排层
- **`pi-tui` / `pi-web-ui`**：交互展示层
- **`coding-agent` / `mom` / `pods`**：面向具体使用场景的应用层

## 9. 开发与验证工作流

根目录 README 已经给出常用命令：

```bash
npm install
npm run build
npm run check
./test.sh
./pi-test.sh
```

但需要注意两点：

1. 根 README 明确说明：`npm run check` 依赖先完成 `npm run build`
2. 这个仓库是多包结构，所以定位问题时要先弄清楚你改的是“底层基础库”还是“上层应用”

一个很常见的判断方式是：

- 只改 provider / model / stream：先看 `packages/ai`
- 只改 agent loop / tool handling / event semantics：先看 `packages/agent`
- 只改 terminal 交互：先看 `packages/tui` 和 `packages/coding-agent`
- 只改浏览器端消息展示/会话存储：先看 `packages/web-ui`
- 只改 Slack 机器人行为：先看 `packages/mom`
- 只改远端部署或 vLLM 管理：先看 `packages/pods`

## 10. 对这个仓库设计风格的总结

读完这些包后，你会发现它们共享一些明显的设计倾向：

- **强调最小核心**：核心保持克制，把扩展点留给外部
- **强调结构化状态**：messages、sessions、tools、events 都是显式结构
- **强调可组合**：runtime、UI、应用层分离
- **强调工具调用**：不仅支持聊天，更重视 agentic workflows
- **强调长期工作流**：branching、compaction、storage、memory 都在为长会话服务

这也是为什么它不是一个“单独的聊天 UI 仓库”，而更像一个 agent 平台的分层实现。

## 11. 下一步读什么？

如果你刚刚读完这份 tutorial，推荐按下面顺序继续：

1. 根 [`README.md`](../README.md)  
2. 你最关注的 package README  
3. 对应 package 的 `src/index.ts`  
4. 该 package 最核心的类型文件（通常是 `types.ts`）  
5. 该 package 的主流程文件（例如 `stream.ts`、`agent.ts`、`main.ts`、`ChatPanel.ts`）

这会让你先建立“接口和边界”，再进入“实现细节”，理解成本最低。

---

## 附：一张速查表

| 目标 | 先读哪里 |
| --- | --- |
| 想知道仓库整体做什么 | `README.md` + 本文档 |
| 想知道如何统一接不同模型 | `packages/ai` |
| 想知道 agent 如何循环执行工具 | `packages/agent` |
| 想知道终端 UI 怎么搭 | `packages/tui` |
| 想知道 coding agent 的产品能力 | `packages/coding-agent` |
| 想知道 Web 版聊天界面怎么实现 | `packages/web-ui` |
| 想知道 Slack bot 怎么长期记忆和执行任务 | `packages/mom` |
| 想知道模型如何部署到 GPU pod | `packages/pods` |

如果你在阅读过程中只记住一件事，请记住这句：

> 这个仓库的核心不是“某个 CLI 命令”，而是“一套可组合的 agent 基础设施”，CLI、Web UI、Slack bot、GPU 部署工具只是这套基础设施在不同场景下的具体落地。
