# OpenClaw Lark/Feishu Plugin — 核心入口、启动流程与业务流程详解

> 本文档是 [overview.md](./overview.md) 的深入补充，聚焦于核心入口文件、完整启动流程、核心业务流程的详细分析，以及各核心模块的职责说明。

---

## 1. 核心入口文件

项目有两个层级的入口文件：

### 1.1 插件入口 — `index.ts`

位于项目根目录，是整个插件的**唯一注册入口**，被 OpenClaw 宿主平台加载。

**核心职责：**

```typescript
const plugin = {
  id: 'openclaw-lark',
  name: 'Feishu',
  description: 'Lark/Feishu channel plugin with im/doc/wiki/drive/task/calendar tools',
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    // 1. 注入运行时
    LarkClient.setRuntime(api.runtime);

    // 2. 注册飞书渠道（ChannelPlugin 接口）
    api.registerChannel({ plugin: feishuPlugin });

    // 3. 注册 OAPI 工具族（日历、任务、多维表格、搜索、云文档等）
    registerOapiTools(api);

    // 4. 注册 MCP 文档工具（创建、获取、更新文档）
    registerFeishuMcpDocTools(api);

    // 5. 注册 OAuth 工具（设备授权流、批量授权）
    registerFeishuOAuthTool(api);
    registerFeishuOAuthBatchAuthTool(api);

    // 6. 注册工具调用追踪钩子（before/after tool call）
    api.on('before_tool_call', ...);
    api.on('after_tool_call', ...);

    // 7. 注册 CLI 命令（feishu-diagnose）
    api.registerCli(...);

    // 8. 注册聊天命令（/feishu_diagnose, /feishu_doctor, /feishu_auth, /feishu）
    registerCommands(api);

    // 9. 多账号安全检查
    emitSecurityWarnings(api.config, api.logger);
  },
};
export default plugin;
```

此外，`index.ts` 还通过 `export` 暴露了大量公共 API（消息发送、媒体上传、表情回应、群聊管理、消息解析等），供外部消费者使用。

### 1.2 渠道定义 — `src/channel/plugin.ts`

实现 OpenClaw 的 `ChannelPlugin<LarkAccount>` 接口，是飞书渠道在宿主平台中的**完整声明**。

**关键结构：**

| 属性 | 职责 |
|------|------|
| `meta` | 渠道元数据：id=`feishu`、label、别名 `lark`、排序等 |
| `pairing` | 用户配对逻辑：ID 标准化、审批通知、触发 onboarding |
| `capabilities` | 能力声明：支持 direct/group 聊天、媒体、表情、线程、流式 |
| `agentPrompt` | AI Agent 提示词：飞书目标语法、表情名称规范等 |
| `groups` | 群组工具策略解析 |
| `configSchema` | 配置 JSON Schema |
| `config` | 账号管理：列出账号、解析账号、默认账号、启用/禁用/删除 |
| `security` | 安全警告收集 |
| `setup` | 初始化配置适配 |
| `onboarding` | 交互式配置向导适配器 |
| `messaging` | 消息目标标准化与解析 |
| `directory` | 通讯录目录（联系人、群组列表） |
| `outbound` | 出站消息适配器 |
| `threading` | 线程上下文构建 |
| `actions` | 消息动作（删除、撤回等） |
| `status` | 运行状态快照与连通性探测 |
| **`gateway`** | **网关入口：启动/停止账号的 WebSocket 监听** |

`gateway.startAccount` 是渠道实际启动的入口——它调用 `monitorFeishuProvider()` 开始 WebSocket 监听。

---

## 2. 完整启动流程

```
OpenClaw 宿主平台启动
    │
    ├─① 加载 openclaw.plugin.json，发现插件 "openclaw-lark"
    │
    ├─② 导入 index.ts，调用 plugin.register(api)
    │   │
    │   ├── LarkClient.setRuntime(api.runtime)        ← 注入运行时单例
    │   ├── api.registerChannel(feishuPlugin)          ← 注册渠道定义
    │   ├── registerOapiTools(api)                     ← 注册 30+ OAPI 工具
    │   │   ├── 日历（4个工具）、任务（4个）、多维表格（5个）
    │   │   ├── 搜索、云文档、知识库、电子表格
    │   │   ├── IM（用户态 + 机器人态）、群聊、用户
    │   │   └── 每个工具通过 api.registerTool() 注册
    │   ├── registerFeishuMcpDocTools(api)              ← 注册 3 个 MCP 文档工具
    │   ├── registerFeishuOAuthTool(api)                ← 注册 OAuth 设备授权工具
    │   ├── registerFeishuOAuthBatchAuthTool(api)       ← 注册批量授权工具
    │   ├── api.on('before_tool_call', ...)             ← 工具调用前日志
    │   ├── api.on('after_tool_call', ...)              ← 工具调用后日志
    │   ├── api.registerCli(feishu-diagnose)            ← CLI 诊断命令
    │   ├── registerCommands(api)                       ← 聊天命令
    │   └── emitSecurityWarnings(...)                   ← 安全检查
    │
    ├─③ OpenClaw 调用 gateway.startAccount(ctx) 启动渠道
    │   │
    │   ├── 读取账号配置 getLarkAccount(cfg, accountId)
    │   ├── 设置运行状态 ctx.setStatus(...)
    │   │
    │   └── monitorFeishuProvider(opts)
    │       │
    │       ├── LarkClient.setGlobalConfig(cfg)       ← 保存全局配置
    │       │
    │       ├─ 单账号模式（指定 accountId）:
    │       │   └── monitorSingleAccount(...)
    │       │
    │       └─ 多账号模式（未指定 accountId）:
    │           └── Promise.all(accounts.map(monitorSingleAccount))
    │
    └─④ monitorSingleAccount(account) 实际启动
        │
        ├── 创建 MessageDedup 实例（消息去重）
        ├── LarkClient.fromAccount(account)            ← 获取/创建客户端实例
        ├── 挂载 messageDedup 到 LarkClient
        │
        └── lark.startWS(handlers, abortSignal)
            │
            ├── lark.probe()                           ← 调用 bot/v3/info 获取机器人身份
            │   └── 缓存 botOpenId, botName
            │
            ├── new EventDispatcher(encryptKey, verificationToken)
            │   └── dispatcher.register(handlers)
            │       ├── 'im.message.receive_v1'        → handleMessageEvent
            │       ├── 'im.message.reaction.created_v1' → handleReactionEvent
            │       ├── 'im.chat.member.bot.added_v1'  → handleBotMembershipEvent
            │       ├── 'im.chat.member.bot.deleted_v1'→ handleBotMembershipEvent
            │       └── 'card.action.trigger'          → handleCardActionEvent
            │
            ├── new WSClient(appId, appSecret, domain)
            │   └── 打 patch: 将 card 类型消息伪装为 event 类型
            │
            └── wsClient.start(eventDispatcher)        ← WebSocket 连接建立
                └── Promise 挂起，等待 abortSignal     ← 持续运行
```

### 启动关键点

1. **运行时注入**: `LarkClient.setRuntime()` 在 `register()` 阶段保存 OpenClaw 运行时单例，后续所有模块通过 `LarkClient.runtime` 访问。

2. **延迟加载**: `gateway.startAccount` 使用 `await import('./monitor.js')` 动态导入，避免在未启用飞书渠道时加载 WebSocket 相关代码。

3. **客户端缓存**: `LarkClient.fromAccount()` 按 `accountId` 缓存实例，凭证变更时自动销毁重建。SDK 客户端（`Lark.Client`）在 `get sdk()` 时惰性创建。

4. **WebSocket 补丁**: 飞书 SDK 的 `handleEventData` 只处理 `type="event"`，卡片回调是 `type="card"` 会被丢弃。启动时打 monkey-patch 将 card 类型改为 event，使 EventDispatcher 能正常路由卡片交互事件。

5. **多账号并行**: 当配置了多个飞书账号时，`monitorFeishuProvider` 使用 `Promise.all` 并行启动所有账号的 WebSocket 监听。

---

## 3. 核心业务流程

### 3.1 入站消息处理流程（七阶段流水线）

当飞书用户发送消息后，完整处理链路如下：

```
飞书 WebSocket 事件到达
    │
    ├─ event-handlers.ts: handleMessageEvent(ctx, data)
    │   │
    │   ├── 事件所有权验证 isEventOwnershipValid()
    │   │   └── 校验 event.app_id 是否匹配当前账号
    │   │
    │   ├── 消息去重 messageDedup.tryRecord(msgId, accountId)
    │   │   └── 过滤 WebSocket 重连导致的重复消息
    │   │
    │   ├── 过期检查 isMessageExpired(create_time)
    │   │   └── 丢弃重连回放的过期消息
    │   │
    │   ├── 中断快速路径 (Abort Fast-Path)
    │   │   └── 若消息是中断指令且有活跃的流式回复 → 立即 abortCard()
    │   │
    │   └── enqueueFeishuChatTask(accountId, chatId, threadId, task)
    │       └── 同一会话的消息串行处理，不同会话并行
    │
    └─ handler.ts: handleFeishuMessage(params) — 七阶段流水线
        │
        ├─ 阶段1: 账号解析
        │   ├── getLarkAccount(cfg, accountId)
        │   └── 构造 accountScopedCfg（per-account 配置隔离）
        │
        ├─ 阶段2: 事件解析
        │   └── parseMessageEvent(event, botOpenId, opts)
        │       ├── 提取 MessageContext（消息内容、发送者、聊天类型等）
        │       └── merge_forward 类型就地展开
        │
        ├─ 阶段3: 轻量增强（gate 之前）
        │   └── resolveSenderInfo(ctx, account)
        │       ├── 解析发送者名称
        │       └── 追踪权限错误
        │
        ├─ 阶段4: 策略门控
        │   └── checkMessageGate(ctx, accountFeishuCfg, ...)
        │       ├── @提及检查：群聊中是否 @了机器人
        │       ├── 白名单检查：发送者是否在允许列表中
        │       └── 权限策略：DM / 群组策略判定
        │       └─ 不通过 → 记录历史条目后直接 return
        │
        ├─ 阶段5: 用户名缓存预热
        │   └── prefetchUserNames(ctx, account)
        │       └── 批量预取发送者和 @提及用户的名称
        │
        ├─ 阶段6: 重量级内容解析（并行执行）
        │   ├── resolveMedia(ctx, ...)       ← 下载图片/文件到本地
        │   ├── resolveQuotedContent(ctx, ...)← 解析引用消息内容
        │   └── substituteMediaPaths(...)     ← 替换飞书 file-key 为本地路径
        │
        ├─ 阶段7a: 命令授权
        │   └── resolveSenderCommandAuthorization(...)
        │       └── SDK 访问组命令门控系统
        │
        └─ 阶段7b: 分发到 Agent
            └── dispatchToAgent(params)
                ├── buildDispatchContext(params) ← 路由/会话/事件上下文
                ├── resolveThreadSessionKey()   ← 线程会话隔离
                ├── buildMessageBody()          ← 消息正文 + 引用
                ├── buildEnvelopeWithHistory()   ← 群聊历史上下文
                ├── buildInboundPayload()        ← SDK 入站载荷
                │
                ├─ /feishu 命令拦截
                │   └── /feishu_doctor, /feishu_auth, /feishu_start, /feishu → i18n 卡片回复
                │
                ├─ 系统命令（/new, /reset 等）
                │   └── dispatchSystemCommand() → 纯文本回复路径
                │
                └─ 普通消息
                    └── dispatchNormalMessage() → 流式卡片回复路径
```

### 3.2 出站回复流程（流式卡片模式）

Agent 生成回复后的回复分发流程：

```
dispatchNormalMessage(dc, ctxPayload, ...)
    │
    ├── createFeishuReplyDispatcher(params) — 回复分发器工厂
    │   │
    │   ├── 解析回复模式 resolveReplyMode(feishuCfg, chatType)
    │   │   └── auto / streaming / static → expandAutoMode()
    │   │
    │   ├── 创建 StreamingCardController（流式模式时）
    │   │   └── 内部持有: FlushController + UnavailableGuard + ImageResolver
    │   │
    │   ├── 创建打字指示器 (Typing Indicator)
    │   │   └── 通过 addReaction/removeReaction 实现
    │   │
    │   └── createReplyDispatcherWithTyping() — SDK 回复调度器
    │       ├── onReplyStart → 显示打字指示器
    │       ├── deliver     → 卡片/文本投递
    │       ├── onError     → 错误处理
    │       ├── onIdle      → 完成卡片终态
    │       └── onCleanup   → 清理资源
    │
    ├── registerActiveDispatcher(queueKey, {abortCard, abortController})
    │   └── 注册活跃分发器，支持中断快速路径
    │
    └── core.channel.reply.dispatchReplyFromConfig(ctx, cfg, dispatcher, replyOptions)
        │
        └── SDK 调度流程（内部调用 LLM + 工具）
            │
            ├── onReasoningStream(payload) ← 思考过程流式回调
            │   └── controller.onReasoningStream()
            │       └── 累积思考文本 → throttledCardUpdate()
            │
            ├── onPartialReply(payload) ← 回复文本流式回调
            │   └── controller.onPartialReply()
            │       ├── 检测回复边界（文本长度缩短 = 新回复）
            │       ├── 累积流式文本
            │       └── throttledCardUpdate() → FlushController
            │           └── performFlush()
            │               ├── CardKit 模式: streamCardContent() ← 打字机效果
            │               └── IM 降级模式: updateCardFeishu()   ← 卡片 patch
            │
            ├── deliver(payload) ← 完整回复片段回调
            │   └── controller.onDeliver()
            │       └── 累积 completedText 用于终态卡片
            │
            └── onIdle() ← SDK 空闲回调
                └── controller.onIdle()
                    ├── setCardStreamingMode(false)   ← 关闭流式模式
                    ├── resolveImagesAwait()           ← 等待图片解析
                    ├── buildCardContent('complete')   ← 构建终态卡片
                    └── updateCardKitCard()            ← 最终更新
```

### 3.3 流式卡片状态机

`StreamingCardController` 使用显式状态机管理卡片生命周期：

```
                    ┌──────────┐
                    │   idle   │
                    └────┬─────┘
                         │ ensureCardCreated()
                    ┌────▼─────┐
                    │ creating │
                    └────┬─────┘
                    ┌────▼──────┐
               ┌────┤ streaming ├────┐
               │    └─────┬─────┘    │
               │          │          │
          ┌────▼────┐ ┌───▼───┐ ┌───▼──────┐
          │ aborted │ │comple-│ │terminated│
          │         │ │ ted   │ │          │
          └─────────┘ └───────┘ └──────────┘

    创建失败时:
                    ┌──────────┐
                    │ creating │
                    └────┬─────┘
                    ┌────▼───────────┐
                    │creation_failed │ → 降级为静态文本发送
                    └────────────────┘
```

**状态转换规则：**

| 当前状态 | 可转换到 | 触发条件 |
|---------|---------|---------|
| `idle` | `creating` | 首次需要发送卡片 |
| `creating` | `streaming` | CardKit 或 IM 卡片创建成功 |
| `creating` | `creation_failed` | 卡片创建失败 |
| `creating` | `aborted` / `terminated` | 用户中断 / 消息不可用 |
| `streaming` | `completed` | 回复正常完成 |
| `streaming` | `aborted` | 用户发送中断指令 |
| `streaming` | `terminated` | 消息被撤回/删除 |

### 3.4 工具调用流程

当 Agent 决定调用飞书工具时：

```
Agent 决定使用工具（如 feishu_calendar_event）
    │
    ├── api.on('before_tool_call') ← 日志记录
    │
    ├── 工具执行
    │   │
    │   ├─ OAPI 工具路径:
    │   │   ├── 解析参数 → OAuth UAT 令牌获取（token-store.ts）
    │   │   ├── LarkClient.fromCfg(cfg, accountId).sdk
    │   │   └── 调用飞书 Open API（REST）
    │   │
    │   └─ MCP 工具路径:
    │       ├── 通过 MCP 协议发送请求
    │       └── MCP 端点处理并返回结果
    │
    ├── api.on('after_tool_call') ← 日志记录（成功/失败 + 耗时）
    │
    └── 结果返回给 Agent → 继续推理或生成回复
```

### 3.5 消息队列与并发控制

```
chat-queue.ts: enqueueFeishuChatTask()
    │
    ├── 队列键 = accountId:chatId[:threadId]
    │   └── 同一会话（含线程）内消息串行处理
    │
    ├── 不同会话的消息完全并行
    │
    ├── registerActiveDispatcher(queueKey, dispatcher)
    │   └── 记录当前活跃的流式回复分发器
    │
    └── 中断快速路径 (Abort Fast-Path)
        ├── 用户发送中断文本（如 "停" / "stop"）
        ├── abort-detect.ts: isLikelyAbortText() 识别
        ├── 在入队之前即检查 → 立即终止当前流式卡片
        └── abortController.abort() + abortCard()
```

---

## 4. 核心模块职责详解

### 4.1 `src/core/` — 核心基础设施层

#### `lark-client.ts` — 飞书 SDK 客户端管理器

**核心类: `LarkClient`**

- **单例管理**: 按 `accountId` 缓存实例（`Map<string, LarkClient>`），凭证变更时自动重建
- **三种工厂方法**:
  - `fromCfg(cfg, accountId)` — 从配置解析账号后创建
  - `fromAccount(account)` — 从已解析的账号创建（有缓存）
  - `fromCredentials(credentials)` — 临时实例（不缓存，用于诊断）
- **惰性 SDK 客户端**: `get sdk()` 首次访问时创建 `Lark.Client`
- **WebSocket 管理**: `startWS()` 启动事件监听，`disconnect()` / `dispose()` 清理
- **机器人身份**: `probe()` 调用 `bot/v3/info` API 获取 `botOpenId` / `botName`
- **全局单例**: `runtime`（OpenClaw 运行时）和 `globalConfig`（全局配置）

#### `accounts.ts` — 多账号管理

- **配置合并**: `getLarkAccount()` 将顶层 `channels.feishu` 配置与 `accounts[id]` 覆盖合并
- **默认账号**: 未指定 accountId 时使用 `DEFAULT_ACCOUNT_ID`
- **启用判定**: `enabled` 显式设置时尊重配置，否则由 `configured`（有 appId + appSecret）推导

#### `token-store.ts` — OAuth 令牌安全存储

- **跨平台后端**: macOS 使用 `security` 命令操作 Keychain；Linux/Windows 使用 AES-256-GCM 加密文件
- **令牌结构**: `StoredUAToken` 包含 accessToken、refreshToken、过期时间、scope、授权时间
- **安全日志**: `maskToken()` 只显示末尾 4 位字符

#### `config-schema.ts` — 配置 Schema

定义 `FEISHU_CONFIG_JSON_SCHEMA`，用于 OpenClaw 配置校验和 IDE 提示。

#### `scope-manager.ts` / `tool-scopes.ts` — 权限管理

管理飞书应用和用户的 API 权限范围，建立工具与所需权限的映射关系。

#### `security-check.ts` — 安全检查

在多账号场景下检查配置安全性，发出警告（如凭证暴露、权限过宽等）。

### 4.2 `src/channel/` — 渠道适配层

#### `plugin.ts` — 渠道插件定义（上文已详述）

#### `monitor.ts` — WebSocket 监控

- `monitorSingleAccount()`: 单账号启动逻辑——创建去重器、获取客户端、注册事件处理器、启动 WebSocket
- `monitorFeishuProvider()`: 入口函数，支持单账号和多账号并行启动
- WebSocket 通过 AbortSignal 控制生命周期

#### `event-handlers.ts` — 事件处理器

从 `monitor.ts` 中提取的事件处理逻辑，处理四类飞书事件：

| 处理器 | 事件 | 逻辑 |
|--------|------|------|
| `handleMessageEvent` | `im.message.receive_v1` | 所有权验证 → 去重 → 过期检查 → 中断快速路径 → 入队处理 |
| `handleReactionEvent` | `im.message.reaction.created_v1` | 所有权验证 → 去重 → 过期检查 → 预解析上下文 → 入队处理 |
| `handleBotMembershipEvent` | `im.chat.member.bot.*` | 记录机器人加入/移出群聊的日志 |
| `handleCardActionEvent` | `card.action.trigger` | 卡片交互事件（如 OAuth 授权按钮）→ 委托给 auto-auth |

#### `chat-queue.ts` — 聊天消息队列

实现 per-chat 串行处理队列：
- 队列键 = `accountId:chatId[:threadId]`
- 同一会话内严格串行，避免并发回复冲突
- 维护活跃分发器注册表，支持中断快速路径

#### `abort-detect.ts` — 中断检测

- `extractRawTextFromEvent()`: 从原始事件中提取文本
- `isLikelyAbortText()`: 判断是否为中断指令（如 "停"、"stop" 等）

### 4.3 `src/messaging/inbound/` — 入站消息处理

#### `handler.ts` — 七阶段流水线主处理器（上文已详述）

#### `parse.ts` / `parse-io.ts` — 事件解析

将飞书原始事件解析为统一的 `MessageContext` 结构，处理各种消息类型（文本、富文本、图片、文件、合并转发等）。

#### `enrich.ts` — 消息增强

- `resolveSenderInfo()`: 轻量增强——解析发送者名称（gate 之前执行，确保日志可读）
- `prefetchUserNames()`: 批量预热用户名缓存（gate 之后执行）
- `resolveMedia()`: 下载媒体附件（图片/文件）到本地临时目录
- `resolveQuotedContent()`: 解析引用消息的完整内容
- `substituteMediaPaths()`: 将飞书 file-key 占位符替换为本地文件路径

#### `gate.ts` / `gate-effects.ts` — 策略门控

核心访问控制逻辑：
- 群聊场景：检查是否 @了机器人（requireMention 策略）
- 白名单检查：发送者是否在 allowFrom 列表中
- DM 策略：pairing / open / closed
- 群组策略：open / allowlist / mention_only

#### `dispatch.ts` — Agent 分发

将处理后的消息分发到 OpenClaw Agent：
- 系统命令路径（/new, /reset, /feishu_*）→ 纯文本回复
- 普通消息路径 → 流式卡片回复
- 中断消息 → 绕过卡片直接分发

### 4.4 `src/card/` — 卡片构建与流式渲染层

#### `reply-dispatcher.ts` — 回复分发器工厂

thin factory 函数，组装整个回复管线：
1. 解析回复模式（streaming / static / auto）
2. 创建 StreamingCardController 或静态发送器
3. 配置打字指示器（通过 Reaction 实现）
4. 组装 SDK 回复调度器的所有回调

#### `streaming-card-controller.ts` — 流式卡片控制器（上文已详述）

#### `builder.ts` — 卡片构建

根据卡片状态（thinking / streaming / complete / confirm）构建飞书卡片 JSON：
- 思考中状态：显示加载动画
- 流式状态：显示实时文本 + 思考过程折叠
- 完成状态：最终文本 + 耗时统计 + 页脚
- 错误/中断状态：错误提示 + 已生成内容

#### `flush-controller.ts` — 刷新节流

控制流式更新频率：
- CardKit 模式：~100ms 节流间隔
- IM patch 模式：~500ms 节流间隔
- 防止飞书 API 限流（230020 错误码）

#### `cardkit.ts` — CardKit 2.0 API 封装

封装飞书 CardKit 2.0 的底层操作：
- `createCardEntity()` — 创建卡片实体
- `sendCardByCardId()` — 通过 card_id 发送 IM 消息
- `streamCardContent()` — 流式推送内容（打字机效果）
- `updateCardKitCard()` — 更新卡片内容
- `setCardStreamingMode()` — 开关流式模式

### 4.5 `src/messaging/outbound/` — 出站消息发送

#### `send.ts` — 底层发送

- `sendMessageFeishu()`: 发送 Markdown 格式的 post 消息
- `sendCardFeishu()`: 发送交互卡片
- `updateCardFeishu()`: 更新已发送的卡片
- `editMessageFeishu()`: 编辑已发送的消息
- 支持线程回复、@提及、i18n 多语言

#### `deliver.ts` — 高层封装

- `sendTextLark()` / `sendCardLark()` / `sendMediaLark()`: 面向外部消费者的简洁 API

#### `media.ts` — 媒体处理

- 图片/文件/音频的上传和发送
- `uploadAndSendMediaLark()`: 一步完成上传+发送

### 4.6 `src/tools/` — 工具注册层

#### `oapi/index.ts` — OAPI 工具统一注册

调用各子模块的注册函数，注册 30+ 飞书 Open API 工具。每个工具通过 `api.registerTool()` 注册到 OpenClaw，Agent 可按需调用。

#### `mcp/doc/` — MCP 文档工具

通过 Model Context Protocol 实现文档操作，与 OAPI 工具互补。仅在账号配置启用时注册。

#### `oauth.ts` / `oauth-batch-auth.ts` — OAuth 工具

实现用户授权流程：
- 设备授权流（Device Flow）：无需浏览器的授权方式
- 批量授权：一次性申请所有应用所需权限

#### `auto-auth.ts` — 自动授权

处理卡片交互事件中的 OAuth 授权按钮点击，自动完成令牌获取和存储。

### 4.7 `src/messaging/converters/` — 消息类型转换器

将飞书各种消息类型转换为 OpenClaw 统一格式的 `MessageContext`：

- 基础类型：`text`, `post`, `image`, `file`, `audio`, `video`
- 社交类型：`sticker`, `location`, `todo`, `vote`, `hongbao`
- 复合类型：`share`, `merge-forward`, `video-chat`, `calendar`
- 交互类型：`interactive`（卡片消息，含 legacy 兼容）
- 系统类型：`system`, `unknown`

每个转换器实现 `ContentConverter` 接口，通过 `converters/index.ts` 统一注册和分发。

### 4.8 `src/commands/` — 命令模块

| 命令 | 实现 | 功能 |
|------|------|------|
| `/feishu_diagnose` | `diagnose.ts` | 配置检查、API 连通性测试、消息链路追踪 |
| `/feishu_doctor` | `doctor.ts` | 健康检查（权限状态、配置完整性） |
| `/feishu_auth` | `auth.ts` | 触发 OAuth 授权流程 |
| `/feishu` / `/feishu_help` | `index.ts` | 帮助信息 |
| `feishu-diagnose` (CLI) | `diagnose.ts` | CLI 版诊断命令（`openclaw feishu-diagnose`） |

所有聊天命令支持 i18n（`zh_cn` + `en_us`），通过卡片形式回复。

---

## 5. 关键设计决策总结

| 决策 | 原因 |
|------|------|
| 按 accountId 缓存 `LarkClient` 实例 | 多账号场景下避免重复创建 SDK 客户端和 WebSocket 连接 |
| 账号级配置隔离（`accountScopedCfg`） | 每个账号可独立配置策略（群组策略、白名单等），不互相干扰 |
| Gate 前执行轻量增强、Gate 后执行重量级解析 | 被拒绝的消息无需下载媒体/解析引用，节省资源 |
| 同会话串行队列 + 跨会话并行 | 避免同一会话中的并发回复冲突，同时保证多会话响应速度 |
| 中断快速路径在入队前执行 | 用户发送"停止"时无需等待队列中的前序任务完成 |
| CardKit 2.0 + IM patch 双路降级 | CardKit 流式更新最优，失败时自动降级到 IM 消息更新 |
| WebSocket card 类型 monkey-patch | 飞书 SDK 限制不支持 card 类型事件路由，通过补丁解决 |
| OAuth 令牌跨平台安全存储 | 使用 OS 原生凭证服务或加密文件，避免明文存储敏感信息 |
