# Telegram SDK 设计草案（WIP）

> 状态：**草案，持续修订中**。全局项目需求尚未描述完，本文档随讨论更新。
> 位置：家族文档库 `framework/docs/`（非 sb-framework-chat 仓库内，命名/仓库归属待定）。

## 0. 定位

基于 sb-framework 家族 + Telegram 双通道（Bot API + MTProto 个人号）的**统一全能消息开发 SDK**。设计风格参考盛派（Senparc）微信 SDK：强类型消息模型 + MessageHandler 分发 + 容器/缓存 + 模块化。

## 1. 需求约束汇总（截至当前）

| 维度 | 约束 |
|------|------|
| 通道 | **个人号（MTProto）+ Bot（Bot API）双通道，做成一套全能 SDK** |
| 规模 | ≤100 业务员（个人号侧，单机可扛） |
| 场景一（个人号） | 业务员手机登录 → 系统同步消息到内网 → 网页端以业务员身份回复 → 防撤回 |
| 复刻范围 | 阶段化；Phase 1 仅**图文收发**；**未完成功能要有文档清单（能力矩阵）** |
| 防撤回 | 收到即存，撤回标记保留内容；审查仅需"查看撤回消息+内容"（处理归 OA） |
| 登录/认证 | SDK 覆盖所有方式（手机号+验证码+2FA / 二维码 / 导入 session），业务再选型 |
| 消息存储 | 复用框架 DB，独立几张表（复杂才独立库） |
| 设计风格 | 参考盛派微信 SDK |

> ⚠️ 封号风险（Userbot）由业务方自行权衡，SDK 按 API 允许范围实现，不再重复提醒。

## 2. 现有代码评估与处置决策

| 组件 | 评估 | 决策 |
|------|------|------|
| `SB.Chat` 抽象层（IChatBot/IChatClient） | 抽象太薄、`RawData` 泄漏协议类型 | ❌ **弃用**（全能 SDK 自持统一模型） |
| `TelegramBotConnector` | 分发逻辑好（统一 ProcessUpdate、有测试） | 🧩 **吸收**为内部 `BotApiAdapter` |
| `TelegramClientConnector` | 半成品（fileId 模型不适用 MTProto、无会话/同步/防撤回） | ❌ **弃用**，重写为生产级 `MtProtoAdapter` |
| `AddTelegramChat` DI 扩展 | 空壳 | ❌ 删除 |
| 命名/版本 | `SB.Chat` 不符合规范、net8 落后 net10 | ⚠️ 顺带解决 |

## 3. 统一架构（盛派风格）

```
┌─ 业务层（网页/审查/OA/机器人业务）
├─ Telegram SDK（统一全能框架）
│   ├─ 统一账号抽象 TelegramAccount（BotAccount / PersonalAccount）
│   ├─ 统一消息管道 MessageHandler（按类型分发 + 管道钩子）
│   ├─ 统一消息模型 + 媒体注册表
│   ├─ 会话上下文 ChatContext / MessageContainer（防撤回载体）
│   ├─ 账号容器 AccountContainer / AccountBag（token / session 自动管理）
│   ├─ 缓存策略 IMessageCacheStrategy（桥接框架 ICacheService）
│   ├─ 存储（统一表，account_type 区分）
│   └─ 能力矩阵 FEATURES.md
├─ 内部通道适配器（不暴露）
│   ├─ BotApiAdapter（Telegram.Bot）
│   └─ MtProtoAdapter（WTelegramClient，生产化）
└─ sb-framework 基础设施（DI/配置/建表/日志）
```

## 4. 盛派模式映射

| 盛派 | 统一 SDK | 职责 |
|------|---------|------|
| RequestMessage/ResponseMessage | `TelegramRequestMessageBase` / `TelegramResponseMessageBase` | 双通道归一化消息 |
| MessageHandler | `TelegramMessageHandler<TContext,TRequest,TResponse>` | 业务入口，override `OnXxxRequestAsync` |
| MessageContext/Container | `ChatContext` / `MessageContainer` | 会话上下文 + **防撤回载体** |
| Container/ContainerBag | `AccountContainer` / `AccountBag` | 账号凭证容器，自动刷新重连 |
| IContainerCacheStrategy | `IMessageCacheStrategy` | 可插拔缓存（框架 ICacheService） |
| 模块化 | 核心包 + Bot / MtProto 通道模块 | 分层解耦 |
| 管道钩子 | `OnExecutingAsync` / `OnExecutedAsync` | 中间件式扩展 |
| 未知类型兜底 | `OnUnknownRequestAsync` | 自定义消息类型注册 |

## 5. 包结构（待定命名/仓库）

```
SB.Chat.Core                ← 统一核心（无第三方依赖）
│   ├─ Model/ 消息模型 + MessageType 注册表
│   ├─ MessageHandler/ 分发基类 + 管道钩子
│   ├─ Context/  ChatContext / MessageContainer（有界 + 防撤回标记）
│   ├─ Account/  AccountContainer / AccountBag
│   ├─ Cache/    IMessageCacheStrategy → 桥接框架 ICacheService
│   └─ Storage/  统一实体 + 表声明（IModuleInitializer）
SB.Chat.Telegram            ← Bot 通道模块（吸收现有 TelegramBotConnector）
SB.Chat.Telegram.MtProto    ← 个人号通道模块（生产化 WTelegramClient）
```

> 命名随规范债一起解决（`SB.Chat` → `Sb.Framework.Chat.*`），仓库归属待定。

## 6. 核心设计要点

### 6.1 统一消息模型
```csharp
public abstract class TelegramRequestMessageBase
{
    public string AccountId { get; set; }
    public string PeerId { get; set; }
    public long MessageId { get; set; }
    public bool IsOutgoing { get; set; }
    public DateTime Timestamp { get; set; }
    public bool IsWithdrawn { get; set; }
}
```
`RequestMessageText` / `RequestMessageImage` 继承；媒体类型用注册表扩展（视频/语音/文件后续挂载）。

### 6.2 MessageHandler（业务入口）
```csharp
public class CustomMessageHandler : TelegramMessageHandler<ChatContext, TelegramRequestMessageBase, TelegramResponseMessageBase>
{
    protected override Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnImageRequestAsync(RequestMessageImage msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnCallbackRequestAsync(RequestMessageCallback msg) { ... }
}
```
同一套 Handler 同时服务 bot 与个人号，业务不感知通道。

### 6.3 防撤回（容器/上下文机制）
```
收到消息 → 立即进 MessageContainer + 落库（含媒体字节，撤回前下载）
收到 delete 更新 → container.MarkWithdrawn(id) + 库中 withdrawn=1，内容保留
```
上限：系统离线期间"发出即撤"抓不到内容（写入产品说明）。

### 6.4 账号容器
`AccountBag { AccountId, AccountType(Bot|Personal), Credential, SessionState, LastActive }`
Bot：token；Personal：MTProto 会话（加密存储）+ 认证状态机 + 断线重连。缓存策略可插拔。

### 6.5 通道适配器（内部）
```csharp
interface ITelegramAdapter
{
    Task ConnectAsync(AccountBag account);
    Task DisconnectAsync(AccountBag account);
    IAsyncEnumerable<TelegramRequestMessageBase> PullAsync();
    Task SendAsync(TelegramResponseMessageBase response);
}
```
BotApiAdapter / MtProtoAdapter 各自把原始协议归一化为统一模型。

### 6.6 存储（统一表，复用框架 DB）
`chat_account` / `chat_session`(加密) / `chat_peer` / `chat_message`(withdrawn) / `chat_media` / `chat_sync_state`

## 7. 能力矩阵初始版（FEATURES.md 雏形）

| 能力 | Bot | Personal |
|------|-----|----------|
| 收发文本/图片 | ✅ Phase1 | ✅ Phase1 |
| 撤回标记+内容保留 | — | ✅ Phase1 |
| 命令/回调/菜单 | ✅ | — |
| 历史补拉/会话同步 | — | 🚧 |
| 视频/语音/文件 | 🚧 | 🚧 |
| 已读/在线/表情反应 | ❌ | ❌ |
| 多设备/会话多开 | ❌ | ❌ |

> 功能缺口文档化是硬要求——"以为支持了其实没有"是禁忌。

## 8. 待确认 / 未决问题

1. SDK 仓库归属与命名（并入 sb-framework-chat / 新仓库；`Sb.Framework.Chat.*` 规范）
2. 全局项目剩余需求（用户继续描述中）
3. 登录认证方式的具体选型（业务阶段）
4. 复刻范围后续阶段清单
5. 网页回复层的职责边界（SDK 提供 API，网页 UI 归业务层）

## 9. 下一步

- 继续全局需求描述，本文档随讨论更新
- 需求收敛后：细化接口签名 → 定稿 → 落实现（先 MtProtoAdapter 生产化，含会话/同步/防撤回）
