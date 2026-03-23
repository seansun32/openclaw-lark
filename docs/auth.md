# OpenClaw Lark — 认证鉴权机制详解

## 目录

1. [总览](#1-总览)
2. [OAuth 2.0 设备授权流程](#2-oauth-20-设备授权流程)
3. [令牌存储与加密](#3-令牌存储与加密)
4. [令牌生命周期管理](#4-令牌生命周期管理)
5. [三级权限检查体系](#5-三级权限检查体系)
6. [ToolClient 统一调用链路](#6-toolclient-统一调用链路)
7. [Owner 访问控制策略](#7-owner-访问控制策略)
8. [自动授权机制](#8-自动授权机制)
9. [消息访问控制门控](#9-消息访问控制门控)
10. [多账号安全隔离](#10-多账号安全隔离)
11. [错误码与错误类型](#11-错误码与错误类型)
12. [安全设计总结](#12-安全设计总结)

---

## 1. 总览

本项目的认证鉴权体系围绕**飞书 OAuth 2.0** 构建，核心目标是让 AI Agent 能安全地代表用户调用飞书 Open API。整体架构分为以下层次：

```
用户发起 API 调用
    |
    v
ToolClient.invoke  ← 统一入口
    |
    +-- 1. App Scope 检查（应用是否开通了所需权限）
    +-- 2. Owner 身份检查（是否为应用所有者）
    +-- 3. User Scope 检查（用户是否授权了所需权限）
    |
    v
callWithUAT  ← 自动刷新 + 重试
    |
    v
飞书 Open API
```

**关键设计原则：**

- **Token 对 AI 不可见**：UAT 从不出现在工具返回值中，AI 层无法获取原始令牌
- **Fail-Close 安全策略**：Owner 查询失败时默认拒绝，而非放行
- **增量授权**：飞书 OAuth 支持累积 scope，只需请求缺失的权限
- **自动恢复**：授权失败时自动触发 Device Flow，无需 AI 介入

**核心源文件：**

| 文件 | 职责 |
|------|------|
| `core/device-flow.ts` | OAuth 2.0 设备授权流程（RFC 8628） |
| `core/token-store.ts` | 跨平台令牌加密存储 |
| `core/uat-client.ts` | UAT 获取、刷新、重试 |
| `core/tool-client.ts` | 统一 API 调用入口，scope 预检 |
| `core/scope-manager.ts` | 三级权限查询与检查 |
| `core/app-scope-checker.ts` | 应用已开通权限查询（带缓存） |
| `core/owner-policy.ts` | Owner 访问控制策略 |
| `core/auth-errors.ts` | 统一错误类型定义 |
| `core/security-check.ts` | 多账号隔离检测 |
| `tools/oauth.ts` | OAuth 工具注册与 Device Flow 编排 |
| `tools/auto-auth.ts` | 工具层自动授权处理（防抖+合并） |
| `messaging/inbound/gate.ts` | 消息访问控制门控 |

---

## 2. OAuth 2.0 设备授权流程

> 源码：`core/device-flow.ts`、`tools/oauth.ts`

采用 **OAuth 2.0 Device Authorization Grant（RFC 8628）**，适用于无浏览器交互的 CLI/Bot 场景。

### 2.1 端点解析

根据 `brand` 配置自动选择端点：

| Brand | 设备授权端点 | Token 端点 |
|-------|-------------|-----------|
| feishu | `accounts.feishu.cn/oauth/v1/device_authorization` | `open.feishu.cn/open-apis/authen/v2/oauth/token` |
| lark | `accounts.larksuite.com/oauth/v1/device_authorization` | `open.larksuite.com/open-apis/authen/v2/oauth/token` |
| 自定义域名 | 智能推导：`open.X` → `accounts.X` | `{domain}/open-apis/authen/v2/oauth/token` |

### 2.2 两步流程

**步骤一：请求设备码** — `requestDeviceAuthorization`

```
POST /oauth/v1/device_authorization
Authorization: Basic base64(appId:appSecret)
Content-Type: application/x-www-form-urlencoded

client_id={appId}&scope={scope} offline_access
```

- 使用 **HTTP Basic Auth**（`appId:appSecret`）进行机密客户端认证
- 自动追加 `offline_access` scope 以获取 refresh_token
- 返回 `deviceCode`、`userCode`、`verificationUri`、`expiresIn`、`interval`

**步骤二：轮询令牌** — `pollDeviceToken`

```
POST /open-apis/authen/v2/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code
&device_code={deviceCode}&client_id={appId}&client_secret={appSecret}
```

轮询策略：
- `authorization_pending` → 继续轮询
- `slow_down` → 间隔 +5s（上限 60s）
- `access_denied` → 终止，用户拒绝
- `expired_token` → 终止，设备码过期
- 安全上限：最多 200 次轮询
- 支持 `AbortSignal` 外部取消

### 2.3 身份校验（防群聊劫持）

授权完成后，调用 `/authen/v1/user_info` 验证实际授权用户的 `open_id` 是否与发起人 `senderOpenId` 一致：

```typescript
// oauth.ts — verifyTokenIdentity
const identity = await verifyTokenIdentity(brand, accessToken, senderOpenId);
if (!identity.valid) {
  // 身份不匹配 → 拒绝，更新卡片为"身份不匹配"
}
```

**目的**：防止群聊中其他用户点击授权链接后，错误的 UAT 被绑定到 owner 身份。

### 2.4 授权卡片交互

整个 Device Flow 通过飞书卡片向用户展示：

1. 发送授权卡片（含授权链接 + 二维码）
2. 后台异步轮询 token 端点
3. 授权成功 → 更新卡片为"授权成功"
4. 授权失败/超时 → 更新卡片为对应状态
5. 成功后发送合成消息通知 AI 自动重试之前的操作

---

## 3. 令牌存储与加密

> 源码：`core/token-store.ts`

### 3.1 存储数据结构

```typescript
interface StoredUAToken {
  userOpenId: string;      // 用户 open_id
  appId: string;           // 应用 ID
  accessToken: string;     // 访问令牌
  refreshToken: string;    // 刷新令牌
  expiresAt: number;       // access_token 过期时间（Unix ms）
  refreshExpiresAt: number;// refresh_token 过期时间（Unix ms）
  scope: string;           // 授权的 scope（空格分隔）
  grantedAt: number;       // 授权时间（Unix ms）
}
```

存储键：`Service = "openclaw-feishu-uat"`，`Account = "{appId}:{userOpenId}"`

### 3.2 跨平台后端

| 平台 | 后端 | 存储位置 | 加密方式 |
|------|------|---------|---------|
| macOS | Keychain Access | 系统钥匙串 | OS 原生加密 |
| Linux | 加密文件 | `$XDG_DATA_HOME/openclaw-feishu-uat/` | AES-256-GCM |
| Windows | 加密文件 | `%LOCALAPPDATA%\openclaw-feishu-uat\` | AES-256-GCM |

### 3.3 AES-256-GCM 加密细节（Linux/Windows）

**Master Key 管理：**
- 首次运行时生成 32 字节随机密钥，写入 `master.key`
- 文件权限 `0o600`（仅 owner 读写），目录权限 `0o700`

**加密格式：** `[12字节 IV][16字节 Auth Tag][密文]`

```typescript
function encryptData(plaintext: string, key: Buffer): Buffer {
  const iv = randomBytes(12);           // GCM 推荐 IV 长度
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const enc = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  return Buffer.concat([iv, cipher.getAuthTag(), enc]);
}
```

解密时校验 Auth Tag，任何篡改都会导致解密失败返回 `null`。

### 3.4 令牌状态判定

```typescript
function tokenStatus(token: StoredUAToken): 'valid' | 'needs_refresh' | 'expired' {
  const now = Date.now();
  if (now < token.expiresAt - 5 * 60 * 1000) return 'valid';       // 还有 5 分钟以上
  if (now < token.refreshExpiresAt)           return 'needs_refresh'; // 可刷新
  return 'expired';                                                   // 全部过期
}
```

**提前刷新窗口**：access_token 过期前 5 分钟即标记为 `needs_refresh`。

---

## 4. 令牌生命周期管理

> 源码：`core/uat-client.ts`

### 4.1 获取有效令牌 — getValidAccessToken

```
读取 Keychain → 检查状态
  ├─ valid → 直接返回 accessToken
  ├─ needs_refresh → refreshWithLock → 返回新 accessToken
  └─ expired → 删除令牌 → 抛 NeedAuthorizationError
```

### 4.2 Per-User 刷新锁

refresh_token 是**单次消费**的：一旦使用，旧的立即失效。并发刷新会导致第二个请求使用已消费的 token 而失败。

```typescript
const refreshLocks = new Map<string, Promise<StoredUAToken | null>>();

async function refreshWithLock(opts, stored): Promise<StoredUAToken | null> {
  const key = `${opts.appId}:${opts.userOpenId}`;
  const existing = refreshLocks.get(key);
  if (existing) {
    await existing;                    // 等待已有的刷新完成
    return getStoredToken(...);        // 重新读取最新令牌
  }
  const promise = doRefreshToken(opts, stored);
  refreshLocks.set(key, promise);
  try { return await promise; }
  finally { refreshLocks.delete(key); }
}
```

### 4.3 刷新重试策略

```
调用 token 端点 → 检查响应
  ├─ 成功 → 保存新令牌（refresh_token 已轮换）
  ├─ code=20050（服务端瞬时错误）→ 重试一次 → 仍失败则清除令牌
  └─ 其他错误（20026/20037/20064/20073）→ 直接清除令牌，强制重新授权
```

### 4.4 API 调用自动重试 — callWithUAT

```typescript
async function callWithUAT<T>(opts, apiCall): Promise<T> {
  const accessToken = await getValidAccessToken(opts);
  try {
    return await apiCall(accessToken);
  } catch (err) {
    const code = err?.code;
    if (TOKEN_RETRY_CODES.has(code)) {   // 99991668 或 99991677
      // 刷新令牌后重试一次
      const refreshed = await refreshWithLock(opts, stored);
      return await apiCall(refreshed.accessToken);
    }
    throw err;
  }
}
```

---

## 5. 三级权限检查体系

> 源码：`core/scope-manager.ts`、`core/app-scope-checker.ts`

### 5.1 三级模型

```
Level 1: Required Scopes  ← API 需要什么权限？
    来源：tool-scopes.ts（手动维护的静态映射）
    示例：feishu_calendar_event.create → ["calendar:calendar.event:create", ...]

Level 2: App Granted Scopes  ← 应用开通了吗？
    来源：GET /open-apis/application/v6/applications/{appId}
    缓存：30 秒 TTL 内存缓存
    前提：应用需有 application:application:self_manage 权限

Level 3: User Granted Scopes  ← 用户授权了吗？
    来源：OAuth token 的 scope 字段
    检查：token.scope 是否包含所有 Required Scopes
```

### 5.2 App Scope 查询与缓存

```typescript
// app-scope-checker.ts
async function getAppGrantedScopes(sdk, appId, tokenType?): Promise<string[]> {
  // 1. 检查 30s 缓存
  // 2. 调 API 获取 app.scopes
  // 3. 按 tokenType 过滤（user/tenant）
  // 4. 写入缓存并返回
}
```

特殊处理：
- 查询失败（无 `self_manage` 权限）→ 抛 `AppScopeCheckFailedError`
- HTTP 400/403 → 同上
- 其他网络错误 → 返回空数组，退回服务端判断

### 5.3 Scope 检查函数

```typescript
// 检查应用是否开通了所有必需 scope
checkAppScopes(toolAction, appGrantedScopes): boolean

// 检查用户是否授权了所有必需 scope
checkUserScopes(toolAction, userGrantedScopes): boolean

// 计算缺失的 scope
getMissingAppScopes(toolAction, appGrantedScopes): string[]
getMissingUserScopes(toolAction, userGrantedScopes): string[]
```

---

## 6. ToolClient 统一调用链路

> 源码：`core/tool-client.ts`

`ToolClient.invoke` 是所有工具调用飞书 API 的唯一入口，完整链路如下：

```
invoke(toolAction, fn, options)
  │
  ├─ 1. 检查旧版插件是否已禁用
  │
  ├─ 2. 获取 Required Scopes（从 tool-scopes.ts）
  │
  ├─ 3. 决定 token 类型（user / tenant）
  │
  ├─ 4. App Scope 检查
  │     ├─ UAT 模式：额外检查 offline_access
  │     ├─ 缺失 → 抛 AppScopeMissingError
  │     └─ 查询失败 → appScopeVerified=false，跳过本地预检
  │
  ├─ 5a. TAT 路径（tenant 身份）
  │     └─ 直接调用 fn(sdk)
  │
  └─ 5b. UAT 路径（user 身份）
        ├─ 解析 userOpenId（ticket > appOwner fallback）
        ├─ Owner 身份检查 → assertOwnerAccessStrict
        ├─ 读取已存储 token → 无则抛 UserAuthRequiredError
        ├─ User Scope 预检 → 缺失则抛 UserAuthRequiredError
        └─ callWithUAT(fn) → 自动刷新 + 重试
              ├─ 服务端 99991672 → AppScopeMissingError（清缓存）
              └─ 服务端 99991679 → UserScopeInsufficientError
```

---

## 7. Owner 访问控制策略

> 源码：`core/owner-policy.ts`、`core/app-owner-fallback.ts`

### 7.1 Fail-Close 策略

```typescript
async function assertOwnerAccessStrict(account, sdk, userOpenId): Promise<void> {
  const ownerOpenId = await getAppOwnerFallback(account, sdk);

  if (!ownerOpenId) {
    throw new OwnerAccessDeniedError(userOpenId, 'unknown');  // 查不到 → 拒绝
  }
  if (ownerOpenId !== userOpenId) {
    throw new OwnerAccessDeniedError(userOpenId, ownerOpenId); // 不匹配 → 拒绝
  }
}
```

### 7.2 Owner 判定规则

通过 `getAppInfo` 从应用信息 API 获取：

```
owner_type == 2（企业内成员）→ 使用 owner.owner_id
其他                        → 回退到 creator_id
```

### 7.3 安全细节

- **Owner ID 不对外暴露**：`OwnerAccessDeniedError` 的用户侧消息不包含 `appOwnerId`
- **应用场景**：OAuth 授权发起、批量授权、UAT 调用前检查

---

## 8. 自动授权机制

> 源码：`tools/auto-auth.ts`

当工具调用遇到授权问题时，**直接在工具层自动处理**，不让 AI 判断：

### 8.1 错误分发策略

| 错误类型 | 处理方式 |
|---------|---------|
| `UserAuthRequiredError`（appScopeVerified=true） | 直接发起 OAuth Device Flow |
| `UserScopeInsufficientError` | 用 missingScopes 发起增量 OAuth |
| `AppScopeMissingError` | 发送权限引导卡片 → 等用户确认 → 清缓存 → 发起 OAuth |
| 其他（`AppScopeCheckFailedError` 等） | 回退到标准错误处理 |

### 8.2 防抖与 Scope 合并

多个工具调用可能在短时间内并发触发授权。采用两阶段缓冲：

```
Phase 1: collecting（收集阶段）
  ├─ 50ms 防抖窗口（用户授权 150ms）
  ├─ 合并所有请求的 scope（集合并集）
  └─ 定时器到期 → 进入 executing

Phase 2: executing（执行阶段）
  ├─ 执行 executeAuthorize（发卡片 + 轮询）
  ├─ 后续到达的请求复用同一结果
  └─ 新 scope 到达时触发 500ms 延迟刷新（更新卡片内容）
```

**缓冲区 Key 规则**：
- 用户授权：`user:{accountId}:{senderOpenId}:{messageId}`
- 应用授权：`app:{accountId}:{chatId}:{messageId}`

**冷却期**：执行完毕后 entry 保留 30 秒，防止后续串行工具调用创建重复卡片。

### 8.3 AppScopeMissing 特殊流程

```
应用权限缺失
  → 发送权限引导卡片（含开放平台链接）
  → 用户在开放平台开通权限后点击"已完成"
  → 更新卡片为"处理中"
  → invalidateAppScopeCache 清缓存
  → 重新校验应用权限
  → 发送合成消息通知 AI
  → 发起 OAuth Device Flow
```

---

## 9. 消息访问控制门控

> 源码：`messaging/inbound/gate.ts`

所有入站消息在进入七阶段流水线的第④阶段时，经过 `checkMessageGate` 策略检查。

### 9.1 群聊门控（三层）

**Layer 1 — 群组级准入**（SDK `resolveGroupPolicy`）：

| groupPolicy | 行为 |
|------------|------|
| `open` | 任何群组通过 |
| `allowlist` | 仅配置的群组 ID 通过 |
| `disabled` | 全部群组拒绝 |

**Layer 2 — 发送者级过滤**：

```
globalGroupAllowFrom + perGroupAllowFrom → 合并列表
  ├─ groupPolicy="open" → 任何发送者
  ├─ groupPolicy="allowlist" → 检查合并后的 allowFrom 列表
  └─ groupPolicy="disabled" → 拒绝所有发送者
```

**Layer 3 — @提醒要求**（SDK `resolveRequireMention`）：

优先级：per-group > default("*") > requireMentionOverride > true（默认需要 @）

未 @bot 的消息不处理，但会记录到历史（供后续上下文使用）。

### 9.2 私聊门控

| dmPolicy | 行为 |
|---------|------|
| `disabled` | 拒绝所有私聊 |
| `open` | 允许所有私聊 |
| `allowlist` | 检查 configAllowFrom + storeAllowFrom |
| `pairing`（默认） | 检查允许列表；未配对用户 → 发送配对请求卡片 |

配对允许列表持久化在 SDK runtime 中，跨重启保留。

---

## 10. 多账号安全隔离

> 源码：`core/security-check.ts`

### 10.1 隔离状态检测

```typescript
type IsolationStatus =
  | { mode: 'not-applicable' }       // 单账号或同 appId
  | { mode: 'isolated' }             // 不同 agent，完全隔离
  | { mode: 'shared-explicit' }      // 显式共享同一 agent
  | { mode: 'shared-implicit' }      // 隐式共享（危险！）
```

检测逻辑：多个账号 + 不同 appId + 无 binding → `shared-implicit`（有数据泄露风险）

### 10.2 dmScope 会话隔离

不同 bot 与同一用户的私聊默认共享 session。需设置：

```
session.dmScope = "per-account-channel-peer"
```

使每个 bot 的私聊会话独立，避免消息串混。

### 10.3 配置模型

```typescript
// config-schema.ts 中的认证相关字段
FeishuAccountConfigSchema = {
  appId: string,              // 应用 ID
  appSecret: string,          // 应用密钥
  encryptKey: string,         // 事件签名验证密钥
  verificationToken: string,  // 事件签名验证 token
  uat: {
    enabled: boolean,         // 是否启用 UAT
    allowedScopes: string[],  // 允许的 scope 白名单
    blockedScopes: string[],  // 禁止的 scope 黑名单
  },
  dmPolicy: 'open' | 'pairing' | 'allowlist' | 'disabled',
  allowFrom: string[],        // 私聊允许列表
  groupPolicy: 'open' | 'allowlist' | 'disabled',
  groupAllowFrom: string[],   // 群聊发送者允许列表
}
```

---

## 11. 错误码与错误类型

> 源码：`core/auth-errors.ts`

### 11.1 飞书错误码常量

| 常量 | 错误码 | 含义 | 处理方式 |
|------|--------|------|---------|
| `APP_SCOPE_MISSING` | 99991672 | 应用 scope 不足 | 清缓存 → AppScopeMissingError |
| `USER_SCOPE_INSUFFICIENT` | 99991679 | 用户 scope 不足 | UserScopeInsufficientError |
| `TOKEN_INVALID` | 99991668 | access_token 无效 | 刷新 + 重试 |
| `TOKEN_EXPIRED` | 99991677 | access_token 过期 | 刷新 + 重试 |
| `REFRESH_TOKEN_INVALID` | 20026 | refresh_token 非法 | 清除令牌 |
| `REFRESH_TOKEN_EXPIRED` | 20037 | refresh_token 过期 | 清除令牌 |
| `REFRESH_TOKEN_REVOKED` | 20064 | refresh_token 被吊销 | 清除令牌 |
| `REFRESH_TOKEN_ALREADY_USED` | 20073 | refresh_token 已消费 | 清除令牌 |
| `REFRESH_SERVER_ERROR` | 20050 | 服务端瞬时错误 | 重试一次 |

### 11.2 自定义错误类

| 错误类 | 触发场景 | 自动授权行为 |
|--------|---------|------------|
| `NeedAuthorizationError` | 无 token 或刷新失败 | 触发 Device Flow |
| `AppScopeCheckFailedError` | 无法查询应用权限 | 提示管理员开通 self_manage |
| `AppScopeMissingError` | 应用未开通所需 scope | 发送权限引导卡片 |
| `UserAuthRequiredError` | 用户未授权或 scope 不足 | 发起 OAuth 授权 |
| `UserScopeInsufficientError` | 服务端报 99991679 | 增量 OAuth 授权 |
| `OwnerAccessDeniedError` | 非 owner 用户 | 拒绝，不触发授权 |

---

## 12. 安全设计总结

| 安全特性 | 实现方式 |
|---------|---------|
| 令牌加密存储 | macOS Keychain + Linux/Windows AES-256-GCM |
| Token 对 AI 不可见 | 工具返回值中从不包含原始令牌 |
| 令牌自动刷新 | 提前 5 分钟刷新 + per-user 锁防并发 |
| OAuth 身份校验 | 授权完成后 `/authen/v1/user_info` 验证 open_id |
| Owner 访问控制 | Fail-Close：查询失败 → 拒绝 |
| Owner ID 不泄露 | 错误响应不包含 appOwnerId |
| 三级 Scope 检查 | API required → App granted → User granted |
| Scope 缓存 | 30s TTL + 手动失效 |
| 增量授权 | 只请求缺失的 scope，飞书 OAuth 累积授权 |
| 自动授权防抖 | 50ms 防抖 + 30s 冷却期，防止重复卡片 |
| 多账号隔离 | Agent binding + per-account-channel-peer dmScope |
| 群聊三层门控 | 群组准入 → 发送者过滤 → @提醒要求 |
| 私聊配对机制 | pairing 模式自动发送配对请求 |
| refresh_token 单次消费 | per-user 锁确保不并发刷新 |
| 日志脱敏 | `maskToken` 只显示最后 4 位 |
