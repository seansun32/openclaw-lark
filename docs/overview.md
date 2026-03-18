# OpenClaw Lark/Feishu Plugin — 项目架构概览

## 项目简介

`@larksuite/openclaw-lark` 是 OpenClaw 平台的官方飞书/Lark 渠道插件，由飞书开放平台团队开发维护。它将 OpenClaw Agent 无缝接入飞书工作空间，支持消息收发、文档、多维表格、日历、任务等丰富能力。

- **运行时**: Node.js >= 22, TypeScript (ESM)
- **许可证**: MIT
- **核心依赖**: `@larksuiteoapi/node-sdk`（飞书 SDK）、`openclaw`（宿主平台）、`zod` / `@sinclair/typebox`（校验）

---

## 目录结构

```
openclaw-lark/
├── index.ts                    # 插件入口：注册渠道、工具、命令、事件钩子
├── package.json
├── openclaw.plugin.json        # OpenClaw 插件清单
├── tsconfig.json
├── eslint.config.js
├── bin/
│   └── openclaw-lark.js        # CLI 可执行入口
├── skills/                     # AI 技能定义（Skill Prompt）
│   ├── feishu-bitable/         #   多维表格操作技能
│   ├── feishu-calendar/        #   日历管理技能
│   ├── feishu-channel-rules/   #   频道规则/配置技能
│   ├── feishu-create-doc/      #   创建文档技能
│   ├── feishu-fetch-doc/       #   获取文档技能
│   ├── feishu-im-read/         #   消息读取技能
│   ├── feishu-task/            #   任务管理技能
│   ├── feishu-troubleshoot/    #   故障排查技能
│   └── feishu-update-doc/      #   更新文档技能
├── src/
│   ├── core/                   # 核心基础设施层
│   ├── channel/                # 渠道适配层
│   ├── messaging/              # 消息处理层（入站/出站/转换器）
│   ├── card/                   # 卡片构建与流式渲染层
│   ├── tools/                  # 工具注册层（OAPI / MCP / OAuth）
│   └── commands/               # 聊天命令与 CLI 命令
└── docs/                       # 文档
```

---

## 整体架构

插件采用**分层架构**，从下至上依次为：

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw 宿主平台                      │
│              (PluginApi / ChannelPlugin 接口)              │
├─────────────────────────────────────────────────────────┤
│                   index.ts  插件入口                      │
│         注册渠道 / 工具 / 命令 / 事件钩子 / 安全检查          │
├──────────┬──────────┬──────────┬──────────┬──────────────┤
│  channel │ messaging│   card   │  tools   │  commands    │
│  渠道适配 │ 消息处理  │ 卡片渲染  │ 工具注册  │  聊天/CLI命令 │
├──────────┴──────────┴──────────┴──────────┴──────────────┤
│                     core  核心基础设施                     │
│   LarkClient / TokenStore / 权限 / 配置 / 日志 / 安全      │
├─────────────────────────────────────────────────────────┤
│              @larksuiteoapi/node-sdk (飞书 SDK)           │
└─────────────────────────────────────────────────────────┘
```

---

## 核心模块详解

### 1. `src/core/` — 核心基础设施层

| 文件 | 职责 |
|------|------|
| `lark-client.ts` | 飞书 SDK 客户端管理器，按 accountId 缓存实例，管理 WebSocket 连接、EventDispatcher 生命周期和机器人身份 |
| `token-store.ts` | OAuth 令牌持久化存储，跨平台支持（macOS Keychain / Linux+Windows AES-256-GCM 加密文件） |
| `uat-client.ts` | 用户访问令牌（UAT）客户端，处理令牌刷新和续期 |
| `agent-config.ts` | 读取 Agent 级别配置（身份、技能、工具、子代理），桥接 SDK Agent 基础设施与飞书调度层 |
| `scope-manager.ts` | 权限范围管理，追踪和验证应用/用户所需的 API 权限 |
| `tool-scopes.ts` | 工具与权限范围的映射关系 |
| `config-schema.ts` | 插件配置 Schema 定义（Zod / TypeBox） |
| `security-check.ts` | 多账号安全检查与警告 |
| `lark-logger.ts` | 统一日志工具 |
| `domains.ts` | 飞书/Lark 域名管理（国际版 vs 国内版） |
| `device-flow.ts` | OAuth 设备授权流程实现 |

### 2. `src/channel/` — 渠道适配层

| 文件 | 职责 |
|------|------|
| `plugin.ts` | 实现 OpenClaw `ChannelPlugin` 接口，定义能力声明（聊天、媒体、表情、线程、流式）、配置 Schema、安全策略 |
| `monitor.ts` | 渠道连接监控，WebSocket 心跳与重连 |
| `onboarding.ts` | `openclaw setup` 交互式配置向导（凭证、域名、群组策略、DM 白名单） |
| `onboarding-config.ts` | 引导流程的配置逻辑 |
| `config-adapter.ts` | 配置格式适配 |
| `event-handlers.ts` | 飞书事件分发处理 |
| `directory.ts` | 渠道目录管理 |
| `probe.ts` | 连通性探测（验证凭证、权限、网络） |
| `chat-queue.ts` | 聊天消息队列管理 |
| `abort-detect.ts` | 中断检测机制 |
| `types.ts` | 渠道层类型定义 |

### 3. `src/messaging/` — 消息处理层

消息处理分为三个子模块：

#### 3.1 `inbound/` — 入站消息处理（七阶段流水线）

```
接收事件 → 账号解析 → 消息解析 → 消息增强 → 策略门控 → 内容解析 → Agent 分发
```

| 文件 | 职责 |
|------|------|
| `handler.ts` | 主处理器，编排七阶段流水线 |
| `parse.ts` / `parse-io.ts` | 事件解析，提取消息结构 |
| `enrich.ts` | 消息增强（发送者信息等） |
| `gate.ts` / `gate-effects.ts` | 策略门控（@提及检查、白名单、权限） |
| `policy.ts` | 权限策略判定 |
| `permission.ts` | 权限检查 |
| `dispatch.ts` / `dispatch-builders.ts` / `dispatch-commands.ts` / `dispatch-context.ts` | Agent 分发与命令路由 |
| `mention.ts` | @提及解析与格式化 |
| `media-resolver.ts` | 媒体内容解析（图片、文件下载） |
| `dedup.ts` | 消息去重（过期检查） |
| `user-name-cache.ts` | 用户名缓存预热 |
| `reaction-handler.ts` | 表情回应处理 |

#### 3.2 `outbound/` — 出站消息发送

| 文件 | 职责 |
|------|------|
| `send.ts` | 底层消息/卡片发送，支持文本、富文本、交互卡片、@提及、i18n |
| `deliver.ts` | 高层发送封装（`sendTextLark`, `sendCardLark`, `sendMediaLark`） |
| `media.ts` | 媒体文件上传与发送（图片、文件、音频） |
| `media-url-utils.ts` | 媒体 URL 处理 |
| `fetch.ts` | 消息获取 |
| `forward.ts` | 消息转发 |
| `chat-manage.ts` | 群聊管理（成员增删、信息更新） |
| `reactions.ts` | 表情回应发送/删除/列表 |
| `actions.ts` | 消息操作动作 |
| `typing.ts` | 打字状态指示 |
| `outbound.ts` | 出站通道数据类型 |

#### 3.3 `converters/` — 消息类型转换器

将飞书各种消息类型转换为统一格式：

`text` / `post` / `image` / `file` / `audio` / `video` / `sticker` / `location` / `calendar` / `todo` / `vote` / `hongbao` / `share` / `merge-forward` / `video-chat` / `system` / `interactive`（卡片）/ `unknown`

### 4. `src/card/` — 卡片构建与流式渲染层

| 文件 | 职责 |
|------|------|
| `streaming-card-controller.ts` | 流式卡片控制器，管理完整生命周期状态机：`idle → creating → streaming → completed/aborted/terminated` |
| `builder.ts` | 卡片构建工具，支持 thinking / streaming / complete / confirm 等状态 |
| `flush-controller.ts` | 刷新节流控制，优化流式更新频率 |
| `reply-dispatcher.ts` | 回复分发器 |
| `reply-mode.ts` | 回复模式管理 |
| `cardkit.ts` | CardKit 2.0 封装 |
| `image-resolver.ts` | 异步图片加载与解析 |
| `markdown-style.ts` | Markdown 样式处理 |
| `unavailable-guard.ts` | 消息可用性检测守卫 |

### 5. `src/tools/` — 工具注册层

工具分为三类注册方式：

#### 5.1 `oapi/` — 飞书 Open API 工具

通过飞书 Open API 直接调用，由 `registerOapiTools()` 统一注册：

| 子目录 | 能力 |
|--------|------|
| `calendar/` | 日历管理（日历 CRUD、事件管理、参与者、忙闲查询） |
| `task/` | 任务管理（任务 CRUD、任务列表、子任务、评论） |
| `bitable/` | 多维表格（应用、表、字段、记录、视图 CRUD） |
| `drive/` | 云文档/云空间（文件操作、文档评论、媒体） |
| `wiki/` | 知识库（空间、节点管理） |
| `sheets/` | 电子表格操作 |
| `im/` | 即时通讯（消息读取、资源获取、用户名解析） |
| `chat/` | 群聊管理（群信息、成员） |
| `search/` | 搜索（文档搜索） |
| `common/` | 通用（用户搜索、用户信息） |

#### 5.2 `mcp/` — MCP（Model Context Protocol）工具

通过 MCP 协议调用，用于文档操作：
- `doc/create.ts` — 创建文档
- `doc/fetch.ts` — 获取文档
- `doc/update.ts` — 更新文档

#### 5.3 OAuth 工具

| 文件 | 职责 |
|------|------|
| `oauth.ts` | UAT 设备流授权工具 |
| `oauth-batch-auth.ts` | 批量权限授权工具 |
| `oauth-cards.ts` | OAuth 授权卡片 UI |
| `auto-auth.ts` | 自动授权逻辑 |
| `onboarding-auth.ts` | 引导流程中的授权 |

### 6. `src/commands/` — 命令注册

| 文件 | 职责 |
|------|------|
| `index.ts` | 注册聊天命令（`/feishu_diagnose`, `/feishu_doctor`, `/feishu_auth`, `/feishu`），支持 i18n |
| `diagnose.ts` | 诊断命令实现（配置检查、连通性测试、消息链路追踪） |
| `doctor.ts` | 健康检查命令 |
| `auth.ts` | 授权命令 |
| `locale.ts` | 国际化/语言环境 |

### 7. `skills/` — AI 技能定义

每个技能目录包含 `SKILL.md` 文件，定义 Agent 执行特定飞书操作时的 Prompt 指引：

- **feishu-bitable** — 多维表格操作指引（含字段属性、记录值、示例等参考文档）
- **feishu-calendar** — 日历事件管理指引
- **feishu-channel-rules** — 频道规则配置指引（含 Markdown 语法参考）
- **feishu-create-doc** / **feishu-update-doc** / **feishu-fetch-doc** — 文档 CRUD 指引
- **feishu-im-read** — 消息历史读取指引
- **feishu-task** — 任务管理指引
- **feishu-troubleshoot** — 故障排查指引

---

## 关键架构模式

### 多账号支持

每个飞书账号拥有独立的配置和凭证。`LarkClient` 按 `accountId` 缓存客户端实例，入站消息处理器构建账号作用域配置以确保隔离。

### 七阶段消息处理流水线

入站消息经过严格的处理链路：账号解析 → 事件解析 → 轻量增强 → 策略门控 → 用户缓存预热 → 重量级内容解析 → Agent 分发。

### 流式卡片状态机

流式回复采用显式状态机管理卡片生命周期（`idle → creating → streaming → terminal`），配合 `FlushController` 节流、`UnavailableGuard` 可用性检测、`ImageResolver` 异步图片加载。

### 双轨工具注册

工具分为 OAPI（直接调用飞书 Open API）和 MCP（通过 Model Context Protocol）两种注册方式，适配不同的集成模式。

### 跨平台安全存储

OAuth 令牌存储使用 OS 原生凭证服务（macOS Keychain）或 AES-256-GCM 加密文件（Linux/Windows），避免明文存储。

### 插件 SDK 集成

通过 OpenClaw 的 `ChannelPlugin` 接口实现标准化集成，包括能力声明、配置 Schema、引导向导、事件钩子等。

---

## 数据流概览

### 入站（用户 → Agent）

```
飞书用户发送消息
    ↓
飞书 WebSocket / Event → LarkClient (EventDispatcher)
    ↓
channel/event-handlers.ts → 路由到 handler
    ↓
messaging/inbound/handler.ts → 七阶段流水线
    ↓
messaging/converters/* → 统一消息格式
    ↓
OpenClaw Agent 处理
```

### 出站（Agent → 用户）

```
OpenClaw Agent 生成回复
    ↓
card/streaming-card-controller.ts → 流式卡片状态管理
    ↓
card/builder.ts → 构建卡片 JSON
    ↓
messaging/outbound/send.ts → 调用飞书 API 发送
    ↓
飞书用户收到消息/卡片
```

### 工具调用（Agent → 飞书 API）

```
Agent 决定使用工具
    ↓
tools/oapi/* 或 tools/mcp/* → 调用对应 API
    ↓
core/lark-client.ts → SDK 客户端发起请求
    ↓
飞书 Open API / MCP 端点
    ↓
结果返回给 Agent
```
