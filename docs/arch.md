# OpenClaw Lark/Feishu Plugin — 架构图与流程图

> 本文档以 Mermaid 图表形式展示项目的整体架构、模块关系和核心业务流程。
> 配合 [overview.md](./overview.md) 和 [core.md](./core.md) 阅读效果更佳。

---

## 1. 整体分层架构

```mermaid
graph TB
    subgraph External["外部系统"]
        FeishuAPI["飞书 Open API<br/>(REST)"]
        FeishuWS["飞书 WebSocket<br/>(事件推送)"]
        MCPEndpoint["MCP 端点<br/>(文档操作)"]
        FeishuUser["飞书用户"]
    end

    subgraph Host["OpenClaw 宿主平台"]
        PluginAPI["PluginApi<br/>registerChannel / registerTool / on"]
        AgentRuntime["Agent Runtime<br/>LLM 推理 + 工具调用"]
        PluginSDK["Plugin SDK<br/>ChannelPlugin / ReplyDispatcher"]
    end

    subgraph Plugin["openclaw-lark 插件"]
        Entry["index.ts<br/>插件入口"]

        subgraph ChannelLayer["渠道适配层 (channel/)"]
            PluginDef["plugin.ts<br/>ChannelPlugin 定义"]
            Monitor["monitor.ts<br/>WebSocket 监控"]
            EventHandlers["event-handlers.ts<br/>事件路由"]
            ChatQueue["chat-queue.ts<br/>会话队列"]
            AbortDetect["abort-detect.ts<br/>中断检测"]
            Onboarding["onboarding.ts<br/>配置向导"]
        end

        subgraph MessagingLayer["消息处理层 (messaging/)"]
            subgraph Inbound["入站 (inbound/)"]
                Handler["handler.ts<br/>七阶段流水线"]
                Parse["parse.ts<br/>事件解析"]
                Enrich["enrich.ts<br/>消息增强"]
                Gate["gate.ts<br/>策略门控"]
                Dispatch["dispatch.ts<br/>Agent 分发"]
                Dedup["dedup.ts<br/>消息去重"]
            end
            subgraph Outbound["出站 (outbound/)"]
                Send["send.ts<br/>消息/卡片发送"]
                Deliver["deliver.ts<br/>高层发送封装"]
                Media["media.ts<br/>媒体上传"]
            end
            Converters["converters/<br/>18种消息类型转换"]
        end

        subgraph CardLayer["卡片渲染层 (card/)"]
            ReplyDisp["reply-dispatcher.ts<br/>回复分发器工厂"]
            StreamCtrl["streaming-card-controller.ts<br/>流式卡片状态机"]
            Builder["builder.ts<br/>卡片构建"]
            FlushCtrl["flush-controller.ts<br/>刷新节流"]
            CardKit["cardkit.ts<br/>CardKit 2.0 API"]
        end

        subgraph ToolsLayer["工具注册层 (tools/)"]
            OAPITools["oapi/<br/>30+ OAPI 工具"]
            MCPTools["mcp/doc/<br/>3个文档工具"]
            OAuthTools["oauth.ts<br/>OAuth 授权工具"]
        end

        subgraph CoreLayer["核心基础设施层 (core/)"]
            LarkClient["lark-client.ts<br/>SDK 客户端管理"]
            Accounts["accounts.ts<br/>多账号管理"]
            TokenStore["token-store.ts<br/>令牌安全存储"]
            ConfigSchema["config-schema.ts<br/>配置 Schema"]
            ScopeManager["scope-manager.ts<br/>权限管理"]
            Logger["lark-logger.ts<br/>日志"]
        end

        Commands["commands/<br/>诊断/健康检查/授权"]
    end

    %% 外部 → 插件
    FeishuUser -->|发送消息| FeishuWS
    FeishuWS --> Monitor
    FeishuAPI <--> LarkClient
    MCPEndpoint <--> MCPTools

    %% 宿主 ↔ 插件
    PluginAPI --> Entry
    Entry --> PluginDef
    Entry --> OAPITools
    Entry --> MCPTools
    Entry --> OAuthTools
    Entry --> Commands
    AgentRuntime <--> Dispatch
    AgentRuntime <--> OAPITools
    AgentRuntime <--> MCPTools

    %% 插件内部
    Monitor --> EventHandlers
    EventHandlers --> ChatQueue
    ChatQueue --> Handler
    Handler --> Parse
    Handler --> Enrich
    Handler --> Gate
    Handler --> Dispatch
    Parse --> Converters
    Dispatch --> ReplyDisp
    ReplyDisp --> StreamCtrl
    StreamCtrl --> FlushCtrl
    StreamCtrl --> Builder
    StreamCtrl --> CardKit
    CardKit --> Send
    Builder --> Send
    Send --> FeishuAPI
    Deliver --> Send
    Media --> FeishuAPI

    %% 核心层依赖
    Monitor --> LarkClient
    LarkClient --> Accounts
    OAPITools --> LarkClient
    OAuthTools --> TokenStore
    Handler --> Accounts

    classDef external fill:#e8f4fd,stroke:#4a90d9
    classDef host fill:#f0e6ff,stroke:#8b5cf6
    classDef channel fill:#fef3c7,stroke:#d97706
    classDef messaging fill:#d1fae5,stroke:#059669
    classDef card fill:#fce7f3,stroke:#db2777
    classDef tools fill:#fed7aa,stroke:#ea580c
    classDef core fill:#e2e8f0,stroke:#64748b

    class FeishuAPI,FeishuWS,MCPEndpoint,FeishuUser external
    class PluginAPI,AgentRuntime,PluginSDK host
    class PluginDef,Monitor,EventHandlers,ChatQueue,AbortDetect,Onboarding channel
    class Handler,Parse,Enrich,Gate,Dispatch,Dedup,Send,Deliver,Media,Converters messaging
    class ReplyDisp,StreamCtrl,Builder,FlushCtrl,CardKit card
    class OAPITools,MCPTools,OAuthTools tools
    class LarkClient,Accounts,TokenStore,ConfigSchema,ScopeManager,Logger core
```

---

## 2. 模块依赖关系图

```mermaid
graph LR
    subgraph EntryPoint
        index["index.ts"]
    end

    subgraph Channel
        plugin["channel/plugin"]
        monitor["channel/monitor"]
        eventHandlers["channel/event-handlers"]
        chatQueue["channel/chat-queue"]
        abortDetect["channel/abort-detect"]
        onboarding["channel/onboarding"]
        directory["channel/directory"]
        probe["channel/probe"]
        configAdapter["channel/config-adapter"]
    end

    subgraph Core
        larkClient["core/lark-client"]
        accounts["core/accounts"]
        tokenStore["core/token-store"]
        configSchema["core/config-schema"]
        scopeManager["core/scope-manager"]
        uatClient["core/uat-client"]
        agentConfig["core/agent-config"]
        securityCheck["core/security-check"]
    end

    subgraph Messaging
        handler["inbound/handler"]
        parse["inbound/parse"]
        enrich["inbound/enrich"]
        gate["inbound/gate"]
        dispatch["inbound/dispatch"]
        mention["inbound/mention"]
        dedup["inbound/dedup"]
        send["outbound/send"]
        deliver["outbound/deliver"]
        media["outbound/media"]
        converters["converters/*"]
    end

    subgraph Card
        replyDispatcher["card/reply-dispatcher"]
        streamingCtrl["card/streaming-card-controller"]
        builder["card/builder"]
        flushCtrl["card/flush-controller"]
        cardkit["card/cardkit"]
        replyMode["card/reply-mode"]
        imageResolver["card/image-resolver"]
    end

    subgraph Tools
        oapiIndex["tools/oapi/index"]
        mcpDoc["tools/mcp/doc"]
        oauth["tools/oauth"]
        autoAuth["tools/auto-auth"]
    end

    subgraph Cmds
        commands["commands/index"]
        diagnose["commands/diagnose"]
        doctor["commands/doctor"]
    end

    %% index.ts 依赖
    index --> plugin
    index --> larkClient
    index --> oapiIndex
    index --> mcpDoc
    index --> oauth
    index --> commands
    index --> securityCheck

    %% channel 层依赖
    plugin --> accounts
    plugin --> larkClient
    plugin --> send
    plugin --> onboarding
    plugin --> configAdapter
    plugin --> configSchema
    plugin --> directory
    monitor --> larkClient
    monitor --> dedup
    monitor --> eventHandlers
    eventHandlers --> handler
    eventHandlers --> autoAuth
    eventHandlers --> chatQueue
    eventHandlers --> abortDetect

    %% messaging inbound 依赖
    handler --> parse
    handler --> enrich
    handler --> gate
    handler --> dispatch
    handler --> accounts
    handler --> larkClient
    parse --> converters
    dispatch --> replyDispatcher
    dispatch --> mention
    dispatch --> chatQueue

    %% card 层依赖
    replyDispatcher --> streamingCtrl
    replyDispatcher --> replyMode
    replyDispatcher --> send
    replyDispatcher --> accounts
    streamingCtrl --> flushCtrl
    streamingCtrl --> builder
    streamingCtrl --> cardkit
    streamingCtrl --> imageResolver
    streamingCtrl --> send
    cardkit --> larkClient

    %% tools 依赖
    oapiIndex --> larkClient
    mcpDoc --> larkClient
    oauth --> tokenStore
    oauth --> uatClient
    autoAuth --> tokenStore

    %% core 内部依赖
    larkClient --> accounts
    uatClient --> tokenStore

    classDef entry fill:#fbbf24,stroke:#92400e,stroke-width:2px
    classDef core fill:#e2e8f0,stroke:#64748b
    classDef channel fill:#fef3c7,stroke:#d97706
    classDef msg fill:#d1fae5,stroke:#059669
    classDef card fill:#fce7f3,stroke:#db2777
    classDef tools fill:#fed7aa,stroke:#ea580c

    class index entry
    class larkClient,accounts,tokenStore,configSchema,scopeManager,uatClient,agentConfig,securityCheck core
    class plugin,monitor,eventHandlers,chatQueue,abortDetect,onboarding,directory,probe,configAdapter channel
    class handler,parse,enrich,gate,dispatch,mention,dedup,send,deliver,media,converters msg
    class replyDispatcher,streamingCtrl,builder,flushCtrl,cardkit,replyMode,imageResolver card
    class oapiIndex,mcpDoc,oauth,autoAuth tools
```

---

## 3. 插件启动流程

```mermaid
sequenceDiagram
    participant Host as OpenClaw 宿主平台
    participant Entry as index.ts
    participant LC as LarkClient
    participant Monitor as monitor.ts
    participant WS as 飞书 WebSocket
    participant Bot as 飞书 Bot API

    Note over Host: 平台启动，扫描插件

    Host->>Entry: 加载 openclaw.plugin.json
    Host->>Entry: import default plugin
    Host->>Entry: plugin.register(api)

    activate Entry
    Entry->>LC: LarkClient.setRuntime(api.runtime)
    Entry->>Host: api.registerChannel(feishuPlugin)
    Entry->>Host: registerOapiTools(api) — 30+ 工具
    Entry->>Host: registerFeishuMcpDocTools(api) — 3 个文档工具
    Entry->>Host: registerFeishuOAuthTool(api)
    Entry->>Host: registerFeishuOAuthBatchAuthTool(api)
    Entry->>Host: api.on('before_tool_call', ...)
    Entry->>Host: api.on('after_tool_call', ...)
    Entry->>Host: api.registerCli('feishu-diagnose')
    Entry->>Host: registerCommands(api) — 聊天命令
    Entry->>Entry: emitSecurityWarnings()
    deactivate Entry

    Note over Host: 启动渠道网关

    Host->>Monitor: gateway.startAccount(ctx)

    activate Monitor
    Monitor->>LC: LarkClient.setGlobalConfig(cfg)
    Monitor->>LC: LarkClient.fromAccount(account)
    LC-->>Monitor: larkClient 实例（缓存）

    Monitor->>Monitor: new MessageDedup()
    Monitor->>LC: lark.startWS(handlers, abortSignal)

    activate LC
    LC->>Bot: GET /open-apis/bot/v3/info (probe)
    Bot-->>LC: botOpenId, botName

    LC->>LC: new EventDispatcher()
    LC->>LC: dispatcher.register(handlers)
    Note over LC: 注册 5 类事件处理器

    LC->>WS: new WSClient().start()
    Note over WS: WebSocket 连接建立
    WS-->>LC: 持续接收事件

    LC-->>Monitor: Promise 挂起（等待 abortSignal）
    deactivate LC
    deactivate Monitor
```

---

## 4. 入站消息处理流程（七阶段流水线）

```mermaid
flowchart TD
    Start([飞书用户发送消息]) --> WSEvent[WebSocket 事件到达]

    WSEvent --> EventHandler["event-handlers.ts<br/>handleMessageEvent()"]

    EventHandler --> OwnerCheck{事件所有权校验<br/>app_id 匹配?}
    OwnerCheck -->|不匹配| Discard1([丢弃])
    OwnerCheck -->|匹配| DedupCheck{消息去重<br/>tryRecord()}

    DedupCheck -->|重复| Discard2([跳过])
    DedupCheck -->|新消息| ExpiryCheck{过期检查<br/>isMessageExpired()}

    ExpiryCheck -->|过期| Discard3([丢弃])
    ExpiryCheck -->|有效| AbortCheck{中断快速路径<br/>isLikelyAbortText()?}

    AbortCheck -->|是中断指令| AbortFast["立即中断活跃回复<br/>abortController.abort()<br/>abortCard()"]
    AbortCheck -->|否| Enqueue

    AbortFast --> Enqueue["enqueueFeishuChatTask()<br/>加入会话串行队列"]
    Enqueue --> Pipeline

    subgraph Pipeline["handler.ts — 七阶段流水线"]
        direction TB
        S1["① 账号解析<br/>getLarkAccount()<br/>构造 accountScopedCfg"]
        S2["② 事件解析<br/>parseMessageEvent()<br/>→ MessageContext"]
        S3["③ 轻量增强<br/>resolveSenderInfo()<br/>解析发送者名称"]
        S4{"④ 策略门控<br/>checkMessageGate()"}
        S5["⑤ 用户名预热<br/>prefetchUserNames()"]
        S6["⑥ 重量级解析（并行）"]
        S7["⑦ 分发到 Agent<br/>dispatchToAgent()"]

        S1 --> S2 --> S3 --> S4
        S4 -->|拒绝| GateReject([记录历史 → 返回])
        S4 -->|通过| S5 --> S6 --> S7
    end

    subgraph ParallelResolve["阶段⑥ 并行执行"]
        ResolveMedia["resolveMedia()<br/>下载图片/文件"]
        ResolveQuote["resolveQuotedContent()<br/>解析引用消息"]
        SubstitutePaths["substituteMediaPaths()<br/>替换 file-key"]
    end

    S6 --> ParallelResolve

    subgraph DispatchRouting["阶段⑦ 分发路由"]
        IsFeishuCmd{/feishu_* 命令?}
        IsSysCmd{系统命令?<br/>/new /reset 等}
        IsAbortMsg{中断消息?}

        I18nCard["i18n 卡片回复"]
        SysDispatch["dispatchSystemCommand()<br/>纯文本回复"]
        NormalDispatch["dispatchNormalMessage()<br/>流式卡片回复"]

        IsFeishuCmd -->|是| I18nCard
        IsFeishuCmd -->|否| IsSysCmd
        IsSysCmd -->|是| SysDispatch
        IsSysCmd -->|否| IsAbortMsg
        IsAbortMsg -->|是| SysDispatch
        IsAbortMsg -->|否| NormalDispatch
    end

    S7 --> DispatchRouting

    classDef stage fill:#dbeafe,stroke:#3b82f6
    classDef reject fill:#fee2e2,stroke:#ef4444
    classDef success fill:#d1fae5,stroke:#10b981
    classDef parallel fill:#fef3c7,stroke:#f59e0b

    class S1,S2,S3,S5,S7 stage
    class Discard1,Discard2,Discard3,GateReject reject
    class NormalDispatch,SysDispatch,I18nCard success
    class ResolveMedia,ResolveQuote,SubstitutePaths parallel
```

---

## 5. 出站回复流程（流式卡片模式）

```mermaid
sequenceDiagram
    participant Agent as OpenClaw Agent
    participant Dispatch as dispatch.ts
    participant Factory as reply-dispatcher.ts
    participant Ctrl as StreamingCardController
    participant Flush as FlushController
    participant CK as CardKit API
    participant IM as 飞书 IM API
    participant User as 飞书用户

    Dispatch->>Factory: createFeishuReplyDispatcher()
    Factory->>Factory: resolveReplyMode() → streaming
    Factory->>Ctrl: new StreamingCardController(deps)
    Factory-->>Dispatch: {dispatcher, replyOptions, ...}

    Dispatch->>Agent: dispatchReplyFromConfig()
    Note over Agent: LLM 开始推理

    Agent->>Ctrl: onReasoningStream({text: "思考中..."})
    Ctrl->>Ctrl: ensureCardCreated()

    activate Ctrl
    Ctrl->>CK: createCardEntity() → cardId
    Ctrl->>IM: sendCardByCardId(cardId) → messageId
    Note over Ctrl: idle → creating → streaming
    deactivate Ctrl

    Ctrl->>Flush: throttledUpdate()
    Flush->>CK: streamCardContent(思考文本)
    CK-->>User: 💭 卡片显示思考过程

    loop 流式文本回调
        Agent->>Ctrl: onPartialReply({text: "部分文本..."})
        Ctrl->>Ctrl: 累积 accumulatedText
        Ctrl->>Flush: throttledUpdate()
        Flush->>CK: streamCardContent(累积文本)
        CK-->>User: 卡片实时更新（打字机效果）
    end

    Agent->>Ctrl: deliver({text: "完整回复片段"})
    Ctrl->>Ctrl: 累积 completedText

    Note over Agent: 推理完成

    Agent->>Dispatch: dispatcher.waitForIdle()
    Dispatch->>Ctrl: markFullyComplete()
    Dispatch->>Ctrl: onIdle()

    activate Ctrl
    Ctrl->>CK: setCardStreamingMode(false)
    Ctrl->>Ctrl: resolveImagesAwait()
    Ctrl->>Ctrl: buildCardContent('complete')
    Ctrl->>CK: updateCardKitCard(终态卡片)
    Note over Ctrl: streaming → completed
    deactivate Ctrl

    CK-->>User: ✅ 最终卡片（含耗时统计）
```

---

## 6. 流式卡片状态机

```mermaid
stateDiagram-v2
    [*] --> idle

    idle --> creating: ensureCardCreated()

    creating --> streaming: CardKit/IM 卡片创建成功
    creating --> creation_failed: 卡片创建失败
    creating --> aborted: 用户中断
    creating --> terminated: 消息不可用

    streaming --> completed: onIdle() 正常完成
    streaming --> aborted: abortCard() 用户中断
    streaming --> terminated: UnavailableGuard 检测

    creation_failed --> [*]: 降级为静态文本发送

    completed --> [*]
    aborted --> [*]
    terminated --> [*]

    note right of idle: 初始状态
    note right of creating: 正在创建 CardKit 实体\n和 IM 消息
    note right of streaming: 流式推送文本\nCardKit 打字机效果
    note right of completed: 终态卡片已更新
    note right of aborted: 显示已中断卡片
    note right of terminated: 消息被撤回/删除
    note right of creation_failed: 降级路径:\n走静态 deliver 发送
```

---

## 7. 消息队列与并发控制

```mermaid
flowchart LR
    subgraph Events["WebSocket 事件流"]
        E1["消息 A<br/>chat_1"]
        E2["消息 B<br/>chat_1"]
        E3["消息 C<br/>chat_2"]
        E4["消息 D<br/>chat_1<br/>thread_1"]
        E5["中断 ✋<br/>chat_1"]
    end

    subgraph Queues["会话队列（串行）"]
        Q1["队列: acct:chat_1<br/>───────────────<br/>A → B → ✋"]
        Q2["队列: acct:chat_2<br/>───────────────<br/>C"]
        Q3["队列: acct:chat_1:thread_1<br/>───────────────<br/>D"]
    end

    subgraph Processing["并行处理"]
        P1["处理 A<br/>(流式回复中)"]
        P2["处理 C"]
        P3["处理 D"]
    end

    E1 --> Q1
    E2 --> Q1
    E5 --> Q1
    E3 --> Q2
    E4 --> Q3

    Q1 --> P1
    Q2 --> P2
    Q3 --> P3

    subgraph AbortPath["中断快速路径"]
        AbortCheck["isLikelyAbortText()?"]
        AbortAction["abortController.abort()<br/>abortCard()"]
    end

    E5 -.->|入队前检查| AbortCheck
    AbortCheck -.->|命中| AbortAction
    AbortAction -.->|立即终止| P1

    classDef queue fill:#e0e7ff,stroke:#6366f1
    classDef proc fill:#d1fae5,stroke:#10b981
    classDef abort fill:#fee2e2,stroke:#ef4444

    class Q1,Q2,Q3 queue
    class P1,P2,P3 proc
    class AbortCheck,AbortAction abort
```

---

## 8. 工具调用流程

```mermaid
flowchart TD
    Agent([OpenClaw Agent]) -->|决定调用工具| ToolCall["工具调用请求"]

    ToolCall --> BeforeHook["api.on('before_tool_call')<br/>日志: 工具名 + 参数"]

    BeforeHook --> ToolType{工具类型}

    ToolType -->|OAPI| OAPIPath
    ToolType -->|MCP| MCPPath
    ToolType -->|OAuth| OAuthPath

    subgraph OAPIPath["OAPI 工具路径"]
        direction TB
        OAPIResolve["解析参数"]
        OAPIAuth["获取 OAuth UAT 令牌<br/>token-store.ts"]
        OAPIClient["LarkClient.sdk<br/>飞书 SDK 客户端"]
        OAPICall["调用飞书 Open API<br/>(REST)"]

        OAPIResolve --> OAPIAuth --> OAPIClient --> OAPICall
    end

    subgraph MCPPath["MCP 工具路径"]
        direction TB
        MCPResolve["解析参数"]
        MCPRequest["发送 MCP 请求"]
        MCPEndpoint["MCP 端点处理"]

        MCPResolve --> MCPRequest --> MCPEndpoint
    end

    subgraph OAuthPath["OAuth 工具路径"]
        direction TB
        DeviceFlow["设备授权流<br/>device-flow.ts"]
        TokenOp["令牌操作<br/>token-store.ts"]

        DeviceFlow --> TokenOp
    end

    OAPICall --> AfterHook
    MCPEndpoint --> AfterHook
    TokenOp --> AfterHook

    AfterHook["api.on('after_tool_call')<br/>日志: 成功/失败 + 耗时"]

    AfterHook --> Result["结果返回 Agent"]
    Result --> Agent

    classDef oapi fill:#dbeafe,stroke:#3b82f6
    classDef mcp fill:#fef3c7,stroke:#f59e0b
    classDef oauth fill:#fce7f3,stroke:#db2777

    class OAPIResolve,OAPIAuth,OAPIClient,OAPICall oapi
    class MCPResolve,MCPRequest,MCPEndpoint mcp
    class DeviceFlow,TokenOp oauth
```

---

## 9. 多账号架构

```mermaid
flowchart TB
    subgraph Config["OpenClaw 配置"]
        GlobalCfg["channels.feishu<br/>(顶层默认配置)"]
        AcctMap["accounts:<br/>  bot-a: {appId, appSecret, ...}<br/>  bot-b: {appId, appSecret, ...}"]
    end

    GlobalCfg --> Merge
    AcctMap --> Merge

    subgraph Merge["配置合并 (accounts.ts)"]
        MergeA["getLarkAccount('bot-a')<br/>= 默认 + bot-a 覆盖"]
        MergeB["getLarkAccount('bot-b')<br/>= 默认 + bot-b 覆盖"]
    end

    subgraph Clients["LarkClient 缓存"]
        ClientA["LarkClient('bot-a')<br/>SDK + WebSocket + botOpenId"]
        ClientB["LarkClient('bot-b')<br/>SDK + WebSocket + botOpenId"]
    end

    MergeA --> ClientA
    MergeB --> ClientB

    subgraph Monitor["并行 WebSocket 监听"]
        MonA["monitorSingleAccount('bot-a')"]
        MonB["monitorSingleAccount('bot-b')"]
    end

    ClientA --> MonA
    ClientB --> MonB

    subgraph EventProcess["事件处理"]
        OwnerA{app_id == bot-a?}
        OwnerB{app_id == bot-b?}

        ScopeA["accountScopedCfg<br/>(bot-a 配置隔离)"]
        ScopeB["accountScopedCfg<br/>(bot-b 配置隔离)"]
    end

    MonA --> OwnerA
    MonB --> OwnerB
    OwnerA -->|是| ScopeA
    OwnerB -->|是| ScopeB

    ScopeA --> Pipeline["七阶段流水线<br/>(各自独立策略)"]
    ScopeB --> Pipeline

    classDef config fill:#e0e7ff,stroke:#6366f1
    classDef client fill:#dbeafe,stroke:#3b82f6
    classDef monitor fill:#d1fae5,stroke:#10b981

    class GlobalCfg,AcctMap config
    class ClientA,ClientB client
    class MonA,MonB monitor
```

---

## 10. CardKit 降级策略

```mermaid
flowchart TD
    Start["ensureCardCreated()"]

    Start --> TryCardKit["尝试 CardKit 2.0 流程"]

    TryCardKit --> CreateEntity["createCardEntity()<br/>创建卡片实体"]
    CreateEntity --> CreateOK{创建成功?}

    CreateOK -->|成功| SendByCardId["sendCardByCardId()<br/>通过 card_id 发送 IM"]
    SendByCardId --> SendOK{发送成功?}

    SendOK -->|成功| CardKitStreaming["CardKit 流式模式 ✅<br/>streamCardContent()<br/>打字机效果"]

    CreateOK -->|失败| TryIMCard
    SendOK -->|失败| TryIMCard

    TryIMCard["降级: 尝试 IM 卡片"]
    TryIMCard --> SendCard["sendCardFeishu()<br/>发送普通交互卡片"]
    SendCard --> IMCardOK{发送成功?}

    IMCardOK -->|成功| IMPatchMode["IM Patch 模式 ⚠️<br/>updateCardFeishu()<br/>整卡更新"]

    IMCardOK -->|失败| CreationFailed["creation_failed 🔴<br/>降级为静态文本"]

    subgraph StreamingUpdate["流式更新阶段"]
        CKUpdate["CardKit: streamCardContent()"]
        CKFail{CardKit 更新失败?}
        CKUpdate --> CKFail
        CKFail -->|是| DisableCK["禁用 CardKit 流式<br/>cardKitCardId = null"]
        DisableCK --> IMUpdate["降级: updateCardFeishu()"]
        CKFail -->|否| CKUpdate

        RateLimit{API 限流 230020?}
        CKFail -->|限流| RateLimit
        RateLimit --> Skip["跳过本次更新"]
    end

    CardKitStreaming --> StreamingUpdate

    subgraph FinalUpdate["终态更新"]
        CloseStream["setCardStreamingMode(false)"]
        FinalCard["updateCardKitCard(终态卡片)"]
        CloseStream --> FinalCard
    end

    StreamingUpdate --> FinalUpdate

    classDef success fill:#d1fae5,stroke:#10b981
    classDef warn fill:#fef3c7,stroke:#f59e0b
    classDef error fill:#fee2e2,stroke:#ef4444
    classDef neutral fill:#f1f5f9,stroke:#94a3b8

    class CardKitStreaming success
    class IMPatchMode warn
    class CreationFailed error
    class Skip neutral
```

---

## 图表说明

| 图表 | 内容 |
|------|------|
| 图 1 | 整体分层架构：外部系统、宿主平台、插件各层及其交互 |
| 图 2 | 模块依赖关系：各源文件之间的 import 依赖拓扑 |
| 图 3 | 启动时序：从宿主平台加载插件到 WebSocket 连接建立 |
| 图 4 | 入站消息流程：事件验证 → 去重 → 七阶段流水线 → 分发路由 |
| 图 5 | 出站回复时序：Agent 推理 → 流式卡片创建 → 文本推送 → 终态更新 |
| 图 6 | 流式卡片状态机：6 种状态及其转换条件 |
| 图 7 | 消息队列：per-chat 串行 + 跨 chat 并行 + 中断快速路径 |
| 图 8 | 工具调用流程：OAPI / MCP / OAuth 三条路径 |
| 图 9 | 多账号架构：配置合并 → 客户端缓存 → 并行监听 → 配置隔离 |
| 图 10 | CardKit 降级策略：CardKit → IM Patch → 静态文本的三级降级 |
