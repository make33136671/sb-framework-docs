# Telegram SDK 技术设计文档

> 状态：**正式技术设计 v1**。基于决策文档《07-Telegram-SDK-设计草案.md》。
> 命名照 `naming-conventions.md` §3：`Sb.Framework.Chat.*`，仓库并入 `sb-framework-chat`。
> 依赖框架：`Sb.Framework.Core / Models`（版本下限 `[0.4.6,)`）。

## 1. 目标

统一全能 Telegram 消息开发 SDK：Bot API + MTProto 个人号双通道，供"商家平台（对外 SaaS）"与"内部系统（合规工具）"共享。设计风格参考盛派微信 SDK。

## 2. 包结构与依赖

```
Sb.Framework.Chat                 ← 核心（无第三方协议依赖；依赖 Sb.Framework.Core/Models）
Sb.Framework.Chat.Telegram        ← Bot 通道（依赖 Chat + Telegram.Bot 22.x）
Sb.Framework.Chat.Telegram.MtProto← 个人号通道（依赖 Chat + WTelegramClient）
```

依赖方向（单向，防循环）：
```
Sb.Framework.Core/Models ← Sb.Framework.Chat ← {Telegram, MtProto} ← 业务系统
```

## 3. 核心接口设计

### 3.1 统一消息模型（`Sb.Framework.Chat.Model`）

```csharp
public enum MessageDirection { Incoming, Outgoing }

public abstract class TelegramRequestMessageBase
{
    public string AccountId { get; set; } = "";
    public string PeerId { get; set; } = "";          // 会话方 id（客户/群）
    public long MessageId { get; set; }               // 通道内消息 id（需与 PeerId 组合定位）
    public MessageDirection Direction { get; set; }
    public DateTime Timestamp { get; set; }           // AppTime.UtcNow（SB0001）
    public bool IsWithdrawn { get; set; }             // 撤回标记（收到即存，delete 时置位）
    public string? ReplyToMessageId { get; set; }
}

public sealed class RequestMessageText : TelegramRequestMessageBase
{
    public string Content { get; set; } = "";
}

public sealed class RequestMessageImage : TelegramRequestMessageBase
{
    public ImageMedia? Media { get; set; }
}

public sealed class RequestMessageCallback : TelegramRequestMessageBase
{
    public string CallbackData { get; set; } = "";
    public string? CallbackId { get; set; }
}

public sealed class RequestMessageGroupEvent : TelegramRequestMessageBase
{
    public GroupEventType EventType { get; set; }     // BotAdded/Removed/MemberJoined/Left
}
```

**媒体抽象 + 注册表（扩展点，后续加视频/语音/文件不改核心）**：

```csharp
public abstract class MediaBase
{
    public string MediaId { get; set; } = "";
    public long Size { get; set; }
    public string? MimeType { get; set; }
}
public sealed class ImageMedia : MediaBase { public string? LocalUrl { get; set; } }
public sealed class VideoMedia : MediaBase { public string? LocalUrl { get; set; } }
public sealed class VoiceMedia  : MediaBase { }
public sealed class FileMedia   : MediaBase { public string? FileName { get; set; } }
public sealed class StickerMedia: MediaBase { }

public enum RequestMessageType { Text, Image, Video, Voice, File, Sticker, Callback, GroupEvent, Service, Unknown }

public interface IMessageTypeResolver { RequestMessageType Resolve(TelegramRequestMessageBase message); }
```

**响应消息（出站）**：

```csharp
public abstract class TelegramResponseMessageBase { public string? PeerId { get; set; } }
public sealed class ResponseMessageText  : TelegramResponseMessageBase { public string Content { get; set; } = ""; }
public sealed class ResponseMessageImage : TelegramResponseMessageBase { public ImageMedia Media { get; set; } = null!; }
public sealed class ResponseMessageCallbackAnswer : TelegramResponseMessageBase { public string? Text { get; set; } public bool ShowAlert { get; set; } }
```

### 3.2 MessageHandler（业务入口）

```csharp
public abstract class TelegramMessageHandler<TContext> where TContext : ChatContext
{
    protected TelegramMessageHandler(TContext context, TelegramMessageHandlerOptions options);

    public Task<TelegramResponseMessageBase?> RunAsync(CancellationToken ct = default);

    // 按类型分发（默认返回 null = 不回）
    protected virtual Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText message, CancellationToken ct) => DefaultAsync();
    protected virtual Task<TelegramResponseMessageBase?> OnImageRequestAsync(RequestMessageImage message, CancellationToken ct) => DefaultAsync();
    protected virtual Task<TelegramResponseMessageBase?> OnCallbackRequestAsync(RequestMessageCallback message, CancellationToken ct) => DefaultAsync();
    protected virtual Task<TelegramResponseMessageBase?> OnGroupEventRequestAsync(RequestMessageGroupEvent message, CancellationToken ct) => DefaultAsync();

    // 未知/自定义消息类型兜底（可注册自定义 RequestMessage + 对应处理器）
    protected virtual Task<TelegramResponseMessageBase?> OnUnknownRequestAsync(TelegramRequestMessageBase message, CancellationToken ct) => DefaultAsync();

    // 管道钩子（中间件式扩展）
    protected virtual Task OnExecutingAsync(ChatContext context, TelegramRequestMessageBase request, CancellationToken ct) => Task.CompletedTask;
    protected virtual Task OnExecutedAsync(ChatContext context, TelegramRequestMessageBase request, TelegramResponseMessageBase? response, CancellationToken ct) => Task.CompletedTask;

    protected Task<TelegramResponseMessageBase?> DefaultAsync() => Task.FromResult<TelegramResponseMessageBase?>(null);
}
```

业务用法（同一 Handler 服务双通道，不感知通道）：
```csharp
public class CustomMessageHandler : TelegramMessageHandler<ChatContext>
{
    protected override async Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText message, CancellationToken ct)
    {
        // 存库、触发 OA、回复……
        return new ResponseMessageText { PeerId = message.PeerId, Content = "收到" };
    }
}
```

### 3.3 ChatContext / MessageContainer（会话上下文 + 防撤回载体）

```csharp
public class ChatContext
{
    public string AccountId { get; init; } = "";
    public string PeerId { get; init; } = "";
    public MessageContainer Messages { get; } = new();
    public object? SessionData { get; set; }              // 业务状态
    public DateTime LastActiveTime { get; set; }
}

public class MessageContainer
{
    public int Capacity { get; init; } = 50;              // 内存有界（完整数据在 DB）
    public void Add(TelegramRequestMessageBase message);  // 收到即入（防撤回根基）
    public void MarkWithdrawn(long messageId);            // 撤回只标记，不移除
    public IReadOnlyList<TelegramRequestMessageBase> Snapshot();
}
```

### 3.4 AccountContainer / AccountBag（账号与凭证容器）

```csharp
public enum AccountType { Bot, Personal }
public enum AccountSessionState { Created, AwaitingCode, Awaiting2FA, Ready, Disconnected, Invalidated }

public class AccountBag
{
    public string AccountId { get; init; } = "";
    public string? OwnerId { get; set; }                 // 多租户判别（商家=商家id / 内部=部门id）
    public AccountType Type { get; init; }
    public AccountSessionState SessionState { get; set; }
    public HashSet<string> Capabilities { get; } = new(); // 能力标记，供业务层权限判断
    public DateTime LastActiveTime { get; set; }
    // 凭证（实现可插拔，不直接暴露明文）
    internal AccountCredential Credential { get; set; }
}

public static class AccountContainer
{
    public static AccountBag GetOrCreate(AccountOptions options);
    public static AccountBag? Get(string accountId);
    public static void Remove(string accountId);
    public static event EventHandler<AccountStateChangedEventArgs>? StateChanged;  // 失效通知
}
```

认证流程（覆盖所有登录方式，业务选型）：
```
手机号+验证码+2FA / 二维码 / 导入现有 session —— 均归一为 AccountSessionState 状态机
Created → AwaitingCode → Awaiting2FA → Ready → (断线) Disconnected → RetryUtil 重连 → Ready
                                              ↘ Invalidated（手机登出/异地登录）→ 事件通知业务层
```

### 3.5 缓存策略（可插拔，桥接框架 ICacheService）

```csharp
public interface IMessageCacheStrategy
{
    T? Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan? expiration = null);
    void Remove(string key);
}
// 默认实现：桥接 SB.Framework.Core 的 ICacheService（Null/Memory/Redis 自动跟随）
```

### 3.6 通道适配器（SDK 内部，不暴露给业务）

```csharp
public interface ITelegramAdapter
{
    AccountType SupportedAccountType { get; }

    Task ConnectAsync(AccountBag account, CancellationToken ct);
    Task DisconnectAsync(AccountBag account);

    IAsyncEnumerable<TelegramRequestMessageBase> PullAsync(CancellationToken ct);

    Task SendAsync(TelegramResponseMessageBase response, CancellationToken ct);
    Task<Stream> DownloadMediaAsync(MediaReference media, CancellationToken ct);

    bool Supports(string capability);
}
```

### 3.7 内容承载（双载体，SDK 中立）

```csharp
// Telegram Web Apps（Mini Apps）
public static class TelegramWebAppUtil
{
    public static TelegramResponseMessageBase BuildWebAppButton(string peerId, string appUrl, string? buttonText = null);
    public static WebAppInitData ValidateInitData(string initData, string botToken);  // HMAC-SHA256 校验
    public static string GetUserJson(WebAppInitData data);
}
// 平台 H5 入口（deep link / 身份绑定）
public interface IContentEntryProvider { string BuildEntryUrl(string accountId, string peerId, string appPath); }
```

### 3.8 支付抽象（通道能力，业务编排在平台）

```csharp
public interface ITelegramPaymentProvider
{
    Task<PaymentInvoice> CreateInvoiceAsync(CreateInvoiceRequest request, CancellationToken ct); // Stars/Payments API
    Task<PaymentResult> HandlePreCheckoutAsync(TelegramRequestMessageBase message, CancellationToken ct);
}
```

## 4. 存储设计（复用框架 DB，`IModuleInitializer` 建表，表前缀 `chat_`）

```csharp
// 模块装配（SDK 核心提供）
public class ChatModuleInitializer : IModuleInitializer
{
    public string ModuleName => "Chat";
    public string ModuleKey => "chat";
    public int InitOrder => 100;
    public Type[] GetEntityTypes() => new[]
    {
        typeof(ChatAccount), typeof(ChatSession), typeof(ChatPeer),
        typeof(ChatMessage), typeof(ChatMedia), typeof(ChatSyncState)
    };
}
```

| 表 | 关键字段 | 说明 |
|----|---------|------|
| `chat_account` | id, owner_id, account_type(Bot/Personal), name, state, capabilities, last_active, created_time | 账号登记 |
| `chat_session` | id, account_id, provider(MtProto), **encrypted_session**, auth_state, phone, created_time | 会话凭证（**加密存储**） |
| `chat_peer` | id, account_id, peer_id, peer_type, title, avatar, last_message_time | 会话方 |
| `chat_message` | id, account_id, peer_id, channel_message_id, direction, **withdrawn**, message_type, content_json, media_ids, timestamp | 消息全量（防撤回根基） |
| `chat_media` | id, account_id, message_id, media_type, local_path, file_ref, mime_type, size, md5, created_time | 媒体文件（撤回前已落盘） |
| `chat_sync_state` | id, account_id, update_cursor, last_sync_time | 更新游标（断点续传） |

**索引**：`chat_message(account_id, peer_id, timestamp)`、`chat_message(withdrawn)`、`chat_message(account_id, channel_message_id)` 唯一。
**撤回检索**（内部系统审查）：`SELECT * FROM chat_message WHERE account_id=? AND withdrawn=1`。
平台层配置表（如 `chat_tenant_config`）由平台系统自行声明，SDK 不内置。

## 5. 通道适配器技术细节

### 5.1 BotApiAdapter（`Sb.Framework.Chat.Telegram`）

- **接收**：webhook 或 polling（可配），`AllowedUpdates` 限定 Message/CallbackQuery/MyChatMember/ChatMember
- **归一化**：`Update` → `RequestMessageText/Image/Callback/GroupEvent`（吸收现有 TelegramBotConnector 的 ProcessUpdate 分发逻辑）
- **出站**：`SendAsync` → `SendTextAsync / SendPhotoAsync / SendMenuAsync / AnswerCallbackAsync`
- **能力**：`commands / callbacks / menus / web_app / payments / inline`

### 5.2 MtProtoAdapter（`Sb.Framework.Chat.Telegram.MtProto`）

- **会话生命周期**：认证状态机 + `ISessionStore`（加密）+ `RetryUtil` 断线重连
- **Update 流**：`UpdateNewMessage / UpdateNewChannelMessage / UpdateEditMessage / UpdateDeleteMessages` → 归一化；**含出站消息（Direction=Outgoing）**——防撤回关键
- **历史补拉**：`getDialogs + getHistory` 按 `chat_sync_state.update_cursor` 断点续传，弥补离线缺口
- **媒体**：`DownloadMediaAsync` 用真实 `InputFileLocation(access_hash + file_reference)`（重写旧半成品）
- **能力**：`full_client / anti_withdrawal / history_sync / media_download`

### 5.3 防撤回时序（核心）

```
业务员手机发违规消息
  → Telegram 出站 UpdateNewMessage（out=true）流入 MtProtoAdapter
  → 归一化 RequestMessageText{ Direction=Outgoing }
  → ① Add 进 MessageContainer
  → ② 落库 chat_message（content_json）
  → ③ 若是图片：撤回前下载字节 → 落 chat_media
  →（业务员撤回）
  → Telegram UpdateDeleteMessages
  → ④ container.MarkWithdrawn(id) + chat_message.withdrawn=1（内容保留）
→ 审查：查 withdrawn=1 即可看到原文
```

**上限（写进产品说明）**：系统离线期间"发出即撤"的消息抓不到内容——Update 流不在线就收不到。

## 6. 装配 / DI / 配置

### 6.1 装配扩展（业务系统调用）

```csharp
public static class TelegramChatExtensions
{
    public static IServiceCollection AddTelegramChat(this IServiceCollection services, Action<TelegramChatOptions>? configure = null);
    public static IServiceCollection AddTelegramBotAdapter(this IServiceCollection services);      // 按需
    public static IServiceCollection AddTelegramMtProtoAdapter(this IServiceCollection services);  // 按需
}
```

### 6.2 配置节（appsettings.json）

```jsonc
{
  "Chat": {
    "ContextCacheCapacity": 50,
    "MediaStoragePath": "/storage/chat/",
    "Accounts": [ { "AccountId": "...", "OwnerId": "...", "Type": "Bot|Personal" } ],
    "Bot": { "WebhookSecret": "...", "WebhookBaseUrl": "https://...", "PathPrefix": "/telegram/webhook" },
    "MtProto": {
      "ApiId": "...", "ApiHash": "...",
      "SessionStore": "Db",                        // Db | File | EncryptedFile
      "Reconnect": { "MaxRetries": 5, "BaseBackoffMs": 1000, "MaxBackoffMs": 30000 }
    }
  }
}
```

### 6.3 后台任务复用
连接维护/心跳走框架 `BaseBackgroundTask`；更新游标节流写库走后台任务。

## 7. 与框架关系 & Core 工具

- **仓库平级、依赖下级**：`Sb.Framework.Chat` 是基座模块，依赖 `Sb.Framework.Core/Models` 基础设施，框架不反向依赖。
- **复用框架基础设施**：FreeSql+IModuleInitializer 建表、ICacheService、AppSettings+IOptions、Serilog、[AutoDependencyInjection]、AppTime(SB0001)、BaseBackgroundTask、Result。
- **抽象到 Core 的通用工具**（SDK 与其它系统共用）：
  - `RetryUtil`（P0）：指数退避+抖动重试
  - `SecurityUtil.MaskSecret()`（P0）：通用敏感值脱敏
  - `IdempotencyUtil`（P1）：消息/Webhook 幂等去重
  - `RateLimiterUtil`（P1）：统一限流入口
  - `WebhookSignatureValidator`（P1）：HMAC 验签
- **留在 SDK 的领域内容**：消息模型、MessageHandler、ChatContext/防撤回、通道适配器、账号会话、`chat_*` 表实体。

## 8. 能力矩阵（FEATURES.md 雏形）

| 能力 | Bot | Personal | 阶段 |
|------|-----|----------|------|
| 收发文本/图片 | ✅ | ✅ | Phase1 |
| 命令/回调/菜单 | ✅ | — | Phase1 |
| 撤回标记+内容保留 | — | ✅ | Phase1 |
| 历史补拉/会话同步 | — | ✅ | Phase1 |
| Mini Apps / Web Apps | ✅ | — | Phase1 |
| 支付（Stars/Payments） | 🚧 | — | Phase2 |
| 视频/语音/文件 | 🚧 | 🚧 | Phase2 |
| 平台 H5 双载体入口 | ✅ | — | Phase1 |
| 已读/在线/表情反应 | ❌ | ❌ | 未规划 |
| 多设备/会话多开 | ❌ | ❌ | 未规划 |

> 功能缺口文档化是硬要求——"以为支持了其实没有"是禁忌。此表随版本演进维护于 SDK 仓库 `FEATURES.md`。

## 9. 实施顺序

1. **Core 工具先行**：`RetryUtil` / `SecurityUtil.MaskSecret` / `IdempotencyUtil` / `RateLimiterUtil` / `WebhookSignatureValidator`（进 sb-framework）
2. **SDK 核心** `Sb.Framework.Chat`：消息模型 → MessageHandler → ChatContext/Container → AccountContainer → 存储表
3. **MtProtoAdapter 生产化**（会话/同步/防撤回/媒体）
4. **BotApiAdapter**（吸收现有分发逻辑）
5. **业务平台层**（商家 SaaS / 内部系统，P0 最小闭环）

## 10. 风险与取舍

| 风险 | 对策 |
|------|------|
| Userbot 封号 | 业务方权衡；SDK 低频被动优先 |
| 防撤回离线缺口 | 写产品说明，不承诺 100% |
| MTProto session 泄露 | 加密存储（chat_session）+ 密钥管理 |
| 大规模多租户 webhook | 集中路由 + 按 token 识别账号 |
| 框架版本对齐 | 版本下限 + 重编译对齐（吸取 0.4.x 教训） |
