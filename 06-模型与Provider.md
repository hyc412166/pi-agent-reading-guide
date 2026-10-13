# 06｜模型与 Provider：从模型名到真实请求

[返回总目录](README.md) · [上一章](05-Agent核心循环.md) · [下一章](07-工具系统.md)

本章回答一个常见疑问：“我选了某个模型，究竟是谁拿凭证、选 API、发请求、把返回变成 Agent 事件？”当前源码把这些责任分为目录、Provider、具体 API 实现、应用运行时四层。
不要把 `provider`（服务商身份）与 `api`（请求协议）当成同一字段。
[核心类型](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts)

```mermaid
flowchart LR
  A[Agent.streamFunction] --> B[ModelRuntime.streamSimple]
  B --> C[每次请求解析认证和选实体模型]
  C --> D[pi-ai Models 集合]
  D --> E[Provider: id/catalog/auth/API 路由]
  E --> F[api/*: 转请求与解析流]
  F --> G[AssistantMessageEventStream]
  G --> A
```

## 一个具体请求

假设会话选中 `provider="openai"` 的聊天模型。
模型目录对象指出 `api="openai-responses"`、`baseUrl`、是否支持 reasoning、上下文窗口、最大输出和价格。
`ModelRuntime.streamSimple()` 在实际消费流时准备 Provider 与请求认证；Provider 再按模型的 `api` 找具体 `ProviderStreams`，OpenAI Responses 实现将规范化 transcript 映射到对应 HTTP 请求，再把上游流转换成 `text_delta`、`toolcall_delta` 等统一事件。
换一个 Provider 可能共用同一种 API 协议；同一个 Provider 也可提供多种 API。
[运行时请求入口](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts) · [Provider 工厂](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) · [OpenAI Provider](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/providers/openai.ts)

## 先区分三类数据

| 名词 | 字段或职责 | 源码 |
|---|---|---|
| `Model<Api>` | 聊天模型的 `provider`、`api`、`id`、`baseUrl`、`reasoning`、`contextWindow`、`maxTokens`、`input`、`cost`，还可有兼容和采样参数。 | [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) |
| `ImageModel` / `ClassifierModel` | 独立模型类型；分别走 `generateImages()` / `classify()`，不可当聊天模型流式调用。 | [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) |
| `Provider` | 持有目录、认证策略、按 API 选择实现，并可动态刷新模型目录。 | [models.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) |
| `Context` | 公开入口可接受旧式 `systemPrompt` / `tools` 快捷字段，但它们在 `normalizeContext()` 处折进首条 system 消息。 | [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) |
| `TranscriptContext` | 真正传给 Provider 的规范化上下文，只有 `messages`；类型 brand 防止误把原始 Context 传下去。 | [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) |
| `AssistantMessage` | 最终响应的内容块、模型/协议、用量、结束原因、错误文字；一次响应可同时含文本、思考和工具调用。 | [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) |

目录中的 `type` 可省略，省略表示 chat；image 和 classifier 需显式标明。
`api` 是协议标识，例如 `anthropic-messages`、`openai-responses` 或 `openai-completions`，不是 Provider 名。
`hasApi()` 只把聊天模型收窄到某协议；调用方若从混合目录选模型，应先按 `isModelType()` 检查用途。
[模型类型与收窄](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

## 目录从哪里来

`packages/ai/src/providers/*.ts` 定义内置 Provider，`*.models.ts` 给出模型条目，`models.generated.ts` 汇集生成目录。
`packages/ai/src/index.ts` 只导出无副作用核心；Provider 工厂和具体 API 实现从各自入口按需加载。
`createProvider()` 接受静态 `models` 与可选 `fetchModels()`；动态条目按“模型类型 + id”覆盖静态项，所以同名 chat 与 image 不会互相覆盖。
[Provider 例子](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/providers/anthropic.ts) · [工厂实现](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) · [核心入口](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/index.ts)

`ModelRuntime.create()` 装配内置 Provider、用户的 `models.json` 配置、模型目录存储和凭证存储。
创建时默认会读取本地目录；网络目录刷新还须显式 `allowModelNetwork=true` 且未设 `PI_OFFLINE`。
`refresh()` 重读配置、重组 Provider、刷新 `pi-ai` 目录和可用性快照。
目录刷新失败与“某 Provider 是否有可用认证”不是同一个状态；诊断时分别看刷新返回错误和 `getError()`。
[创建与刷新](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

`ModelRegistry` 是扩展可用的同步门面，内部持有同一个 `ModelRuntime`。
`getAll()` 看已知聊天模型，`getAvailable()` 看已刷新认证可用性快照，`find()` 定位某个目录条目；它没有维护第二份独立模型目录。
运行时内部直接用 `ModelRuntime`，扩展可以经 Registry 注册 Provider 或虚拟模型。
[门面源码](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-registry.ts)

“目录已知”不等于“请求一定成功”。
Provider 可能没有凭证、某模型的协议映射可能不存在，服务器也可能拒绝请求。
`createProvider()` 对未实现的 `model.api` 返回带错误终止的流，不猜测改走另一协议；真实认证由请求时再解析。
[协议分派](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

## 凭证什么时候取得

在 `ModelRuntime.prepareRequest()` 中，每次实际请求先查 Provider，再 `getAuth(model)`，合并认证给出的 headers 与本次请求 headers/env；如果认证给出替代 `baseUrl`，它创建请求模型副本，不改目录原对象。
`transformHeaders` 在这里执行后移除，避免 Provider 再执行一次。
流式方法返回的对象可延迟真正开始，所以“创建流对象”与“开始联网”并非同一时刻。
[运行时准备](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

`pi-ai` 的认证解析顺序是本次显式 API key、已存凭证、无已存凭证时的环境或环境外认证（如配置允许）。
**已存凭证拥有该 Provider**：若 OAuth 刷新失败或凭证类型不匹配，不会静默改用环境变量里的 key。
这样一个登录身份失效不会悄悄变成另一个身份。
`RuntimeCredentials` 可叠加当前进程的临时 key；文件持久化默认由 `AuthStorage` 管理。
[认证解析](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/auth/resolve.ts) · [临时覆盖](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/runtime-credentials.ts) · [文件存储](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/auth-storage.ts)

OAuth 刷新涉及并发与取消：源码在凭证存储锁内重新检查是否仍需刷新，并在刷新完成后持久化；等待锁可以取消，一旦刷新开始则给它独立超时，避免取消调用者时丢弃已经轮换的唯一 refresh token。
这个规则属于认证层，不是 Agent 回合重试。
[OAuth 刷新](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/auth/resolve.ts)

## Provider 与 API 实现如何合作

`createProvider()` 要求至少有 chat、image 或 classifier 中一种具体实现。
Chat 实现可以是所有聊天模型共用一个 `ProviderStreams`，也可以是按 `model.api` 的映射；图片与分类各有独立映射。
Provider 持有服务身份、认证策略、模型目录和操作路由，`api/*` 文件处理该协议的请求参数、响应解析、事件发射与协议特有细节。
[工厂契约](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) · [API 实现目录](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/api/openai-responses.ts)

例如 OpenAI Provider 注册静态聊天和分类模型，并为相应 API 接上具体实现；Anthropic Provider 将其模型接到 Anthropic Messages API。
模型 `reasoning` 和 `thinkingLevelMap` 描述能力；Agent 传通用思考等级，具体 API 实现再转为协议字段。
`getSupportedThinkingLevels()` / `clampThinkingLevel()` 负责按模型元数据求可用级别，不能只看 UI 上出现了某级就推断实际请求字段。
[OpenAI](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/providers/openai.ts) · [Anthropic](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/providers/anthropic.ts) · [等级处理](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

虚拟模型是**可选**的选择机制。
`ModelRuntime.streamSimple()` 如果发现虚拟模型，会先根据 transcript 解析到实体模型，然后用实体 Provider 重新准备请求；跨 Provider 路由时不会把原虚拟模型 Provider 的认证头、key 和环境直接发给目标服务。
其输出上限还会被目标模型上限约束。
[虚拟模型路径](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

## 流事件、失败和重试

`AssistantMessageEventStream` 的成功顺序是 `start`、零个或多个内容块事件、最终 `done`；建连前故障可以直接以 `error` 结束，开始后故障也以 `error` 结束。
`partial` 是随流增长的活对象，不是该时刻的深拷贝；若记录每个 delta 时的快照，应主动复制。
`toolcall_end` 带最终调用，之前的参数增量可能尚未完整。
[事件类型](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) · [事件流实现](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/utils/event-stream.ts)

`done.reason` 可为 `stop`、`length`、`toolUse` 或 `deferred`；`error.reason` 是 `error` 或 `aborted`。
其中 `length` 会让 Agent 拒绝执行本条回复里的工具调用；`deferred` 是特定 Provider 支持的异步响应能力，不代表普通 Agent 默认走延迟模式。
默认 Agent 核心及 AgentSession 不会仅因 `stopReason="deferred"` 自动轮询 handle；需由支持该能力的宿主显式管理。实验 Durable 的 GenerationTask 有独立 `poll` phase，通过 `fetchDeferred()` 取得结果。[核心循环](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/agent/src/agent-loop.ts) · [Durable 轮询](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/durable/src/harness/generation.ts)
[类型定义](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) · [Agent 处理](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/agent/src/agent-loop.ts)

具体 API 实现负责把供应商事件和 stop reason 归一化。
以 OpenAI Responses 与 Anthropic Messages 为例，二者都将真实请求包装进 `retryProviderRequest()`，并关闭 SDK 自带重试，避免两层重试倍增。
重试 helper 只对允许的请求错误工作，遵守 `x-should-retry`、常见暂时性 HTTP 状态、`Retry-After` 和 AbortSignal；默认最长服务器指定等待为 60 秒。
`maxRetries` 的默认值在 helper 自身为 0，实际调用次数要看各 API 调用时传入的值，不能笼统声称所有 Provider 自动重试两次。
[统一重试](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/utils/provider-retry.ts) · [OpenAI 接入](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/api/openai-responses.ts) · [Anthropic 接入](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/api/anthropic-messages.ts)

当认证或协议调用失败时，最终要检查响应的 `stopReason` 和 `errorMessage`；只看到流对象存在、收到 `start`、或目录里列出模型，都不等于请求成功。
图片生成和分类操作是另外两条单次调用路径，它们把故障转为各自的错误结果，而非聊天消息流。
[模型操作](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

## 把一次请求拆成可检查的五个对象

假设一次调用参数是 `streamSimple(model, context, {reasoning:"high"})`。
调试时依次检查以下对象，能避免把某层的推测当成下一层的事实。
[运行时入口](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

1. **目录模型**：`model.provider` 决定到哪家 Provider 找实现和认证，`model.api` 决定走哪种协议；`id` 是该 Provider 下的模型名。`contextWindow`、`maxTokens` 是目录元数据，不是本次实际使用量。[模型类型](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts)
2. **规范化上下文**：`normalizeContext()` 使 system 提示段和工具增删都在 transcript 中有位置。公开快捷字段只是入口便利形式；Provider 不再另外收到独立的 `systemPrompt` 或 `tools` 状态。[回放](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/utils/transcript.ts)
3. **认证结果**：它可能含 API key、headers、替代 base URL 与环境信息；认证结果按本次请求求得，不能只看目录模型的 `baseUrl` 推断最终网络目的地。[认证结果](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/auth/resolve.ts)
4. **协议请求**：Provider 根据 `model.api` 找到 `api/*` 实现。该实现决定字段名、图片/工具内容如何编码、流事件如何解释。不同协议不能靠把 `baseUrl` 改成同一个地址就等价使用。[分派](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)
5. **最终 assistant**：`responseModel` 可能记录上游实际回报的具体模型，`usage`、`stopReason`、`errorMessage` 与内容块共同描述结果；请求名与响应名可以不同。[响应类型](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts)

这五个对象之间有意保留差异。
例如目录允许一个 Provider 的多种 API；认证可以覆盖 URL 但不修改全局目录；响应可以报告不同的实际模型。
若把它们压成一个“当前模型字符串”，就解释不了跨 Provider 路由、OAuth 令牌更新和协议级错误。
[运行时准备](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

## `getAll()`、`getAvailable()` 与真实执行

模型目录回答“系统知道什么模型”。
可用性快照回答“上次刷新时哪些 Provider/模型有可工作的凭证”。
实际请求还会重新准备认证并受到服务器、网络、配额和模型能力影响。
按这条链读 UI 选项时，不应把下拉列表出现某项视为请求已成功。
[Registry 快照](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-registry.ts) · [请求准备](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

`ModelRuntime.create()` 的 `modelsPath=null` 表示不使用文件模型配置，并默认把目录存储放在内存；普通默认则读取 agent 目录的 `models.json`，目录缓存放在 `models-store.json`。
扩展注册原生 Provider、兼容配置及虚拟模型会经过运行时的组合逻辑。
添加一种服务并非仅在生成目录加一行：它还需认证行为和至少一种实际操作实现。
[创建流程](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts) · [Provider 工厂](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

Provider 的 `refreshModels()` 可以恢复已存动态目录，再在允许网络时获取新目录并发布；这与聊天响应的 `stream()` 完全是两类网络操作。
缓存的动态模型按“类型 + id”覆盖静态同名条目；刷新失败时需要看返回错误，不能仅凭仍能列出旧模型就断定刷新成功。
[动态模型合并](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts)

## 与 Agent 的边界再画一次

Agent 关心“何时再请求一次”和“是否执行工具”；它没有硬编码 OpenAI、Anthropic 等服务商。
`StreamFn` 接收某个 `Model<Api>`、规范化的 `TranscriptContext` 和通用请求选项，返回统一事件流。
编码 Agent 把自己的 `ModelRuntime.streamSimple()` 接进去，故运行时可以更新配置或注册扩展 Provider，而低层 Agent 循环不用因此改变。
[StreamFn](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/agent/src/types.ts) · [ModelRuntime](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

`ModelRegistry` 给扩展提供兼容的同步查找和注册入口。
如果扩展从 Registry 找到 `Model` 对象后保存很久，运行时目录已刷新时那个对象不一定代表最新元数据；发请求时应关注当前 runtime 解析与认证。
Registry 的 `getApiKeyForProvider()` 是只返回 key 的便利函数，它把认证异常折成 `undefined`；需要解释认证失败原因时应调用返回显式状态的接口。
[Registry 方法](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-registry.ts)

## 用事件时间线排查“卡住了”

```text
选择模型 → 创建懒流 → 开始消费流
→ 请求时认证 → Provider/API 发送请求
→ start → text_start → text_delta... → text_end
→ done 或 error → Agent.message_end → Agent.turn_end
```

若在 `start` 前就报错，优先检查目录模型、Provider 是否存在、认证和请求建立；若有 `start` 后报错，检查上游流、解析和取消。
若已经 `done` 但 Agent 仍运行，可能是 assistant 带工具调用、steering、follow-up 或 `finishTurn` 明确继续。
若 Agent 已 `agent_end` 但 UI 尚忙，先检查被等待的异步事件订阅者，再检查 AgentSession 的重试、压缩和收尾边界；底层循环结束不等于整个会话已经 settled。
每个结论都需要结合最终 `stopReason`，不能只凭一个中间事件。
[事件流](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) · [Agent 调度](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/agent/src/agent-loop.ts)

`text_delta` 和 `thinking_delta` 都是同一条 partial 消息内容的增量；`toolcall_delta` 的 JSON 片段可能在 `toolcall_end` 才成为完整参数。
Agent 只在最终 assistant 结束后收集工具调用并执行。
前端可以逐片渲染，执行器不能把尚未结束的参数片段当作一次完整调用。
[事件协议](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) · [循环取 toolCall](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/agent/src/agent-loop.ts)

## 关于“重试”的精确说法

`retryProviderRequest()` 只负责它包住的单次 Provider 请求，不负责重跑整个 Agent 任务，也不重新执行本轮工具。
是否重试、最多几次，由调用该 helper 的 API 实现传入 `maxRetries` 与错误类别共同决定。
请求若明确被取消，等待立即中断；如果上游给了超过允许上限的 `Retry-After`，helper 将该错误作为可见失败返回，而非悄悄睡很久。
[重试 helper](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/utils/provider-retry.ts)

`Models.stream()` 接受该 API 专属选项，`streamSimple()` 则接受通用选项并让具体实现翻译 reasoning 等设置；`complete()` / `completeSimple()` 只消费相应流并取 `result()`，不是额外发第二次模型请求。
`streamDeferred()` 与取消延迟响应的操作只在 Provider 明确实现时可用，不能视作聊天请求的普遍能力。
[pi-ai Models](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) · [编码 Agent 运行时](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts)

## 阅读时可自己验证的四个判断

1. 改 `provider` 而保持 `api` 不变时，认证来源和模型目录会发生什么变化？从 [models.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) 的 `requireChatProvider()` 与 [auth/resolve.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/auth/resolve.ts) 回答。
2. 改 `api` 而保持 `provider` 不变时，怎样确保该 Provider 真有对应实现？看 [createProvider()](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) 缺映射的错误流。
3. `models.json` 新增模型后，`getAll()` 与 `getAvailable()` 何时刷新？看 [ModelRuntime.refresh()](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts) 与 [ModelRegistry](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-registry.ts)。
4. OAuth 过期期间用户取消时，为何已经开始的 refresh 不立即丢弃？看 [refreshStoredOAuthCredential()](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/auth/resolve.ts) 的锁与持久化顺序。

## 建议按这条路线读源码

1. 先读 [types.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/types.ts) 中 `Model`、`ProviderStreams`、`Provider`、`Context` 和 `AssistantMessageEvent`，写下每个字段属于目录、请求还是响应。
2. 顺着 [model-runtime.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/src/core/model-runtime.ts) 的 `streamSimple → prepareRequest`，找出认证、headers 和替代 base URL 在哪一层处理。
3. 对照 [models.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/models.ts) 的 `createProvider()` 与 [providers/openai.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/providers/openai.ts)，解释 `provider` 与 `api` 怎样共同选实现。
4. 在 [api/openai-responses.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/api/openai-responses.ts) 与 [api/anthropic-messages.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/api/anthropic-messages.ts) 中分别找 `start`、`done`、`error` 的发射点，再回看 Agent 如何消费它们。
5. 用 [model-registry.test.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/test/model-registry.test.ts)、[model-runtime-auth-options.test.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/coding-agent/test/model-runtime-auth-options.test.ts) 和 [provider-retry.ts](https://github.com/earendil-works/pi/blob/6fb2e7815167e6b19006fc526d1a5d0f5f998787/packages/ai/src/utils/provider-retry.ts) 检查“目录可见”“认证可用”“请求成功”三种判断是否被混淆。
