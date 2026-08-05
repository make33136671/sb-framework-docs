# Telegram SDK 设计主文档

> 状态：**正式设计 v1 草案（WIP）**。全局需求收敛中，随讨论修订。
> 仓库归属：并入 `sb-framework-chat`；命名照 `naming-conventions.md` §3：`Sb.Framework.Chat.*`。
> 优先级定义：**P0 必须·紧迫**（Phase 1 缺了跑不起来）/ **P1 必须·重要**（完整系统必需，Phase 2）/ **P2 可选·增强**（按需）。
> **技术细节见《08-Telegram-SDK-技术设计.md》**（接口签名 / 存储表 / 时序 / 配置）。

## 0. 定位

基于 sb-framework 家族 + Telegram 双通道（Bot API + MTProto 个人号）的**统一全能消息开发 SDK**。设计风格参考盛派（Senparc）微信 SDK。

服务**两套独立系统**（商家入驻平台 + 公司内部系统），作为两者共享的统一地基。

## 1. 需求约束（已确认）

| 维度 | 约束 |
|------|------|
| 通道 | 个人号（MTProto）+ Bot（Bot API）双通道，一套全能 SDK |
| 账号开放 | SDK 全开放双通道；权限控制由上层系统按租户/角色落地 |
| 系统形态 | 两套独立系统，功能有重叠，共享同一 SDK 地基 |
| 商家平台 | **对外公网 SaaS**（外部商家自助入驻） |
| 场景一（内部） | 业务员手机登录 → 同步消息到内网 → 网页回复 → 防撤回；≤100 业务员 |
| 复刻范围 | 阶段化；**未完成功能必须有能力矩阵文档** |
| 商城载体 | **Telegram Web Apps 与平台 H5 双载体都支持**（SDK 中立） |
| 防撤回 | 收到即存，撤回标记保留内容；审查仅需查看撤回消息+内容（处理归 OA） |
| 登录/认证 | SDK 覆盖所有方式（验证码+2FA/二维码/导入 session），业务再选型 |
| 消息存储 | 复用框架 DB，独立几张表 |
| 设计风格 | 参考盛派微信 SDK |

> ⚠️ 封号风险（Userbot）由业务方自行权衡，SDK 按 API 允许范围实现。

## 2. 现有代码评估与处置（已确认）

| 组件 | 决策 |
|------|------|
| `SB.Chat` 抽象层（IChatBot/IChatClient） | ❌ 弃用（抽象太薄、协议泄漏） |
| `TelegramBotConnector` | 🧩 吸收为 `Sb.Framework.Chat.Telegram` 内部 BotApiAdapter |
| `TelegramClientConnector` | ❌ 弃用，重写为生产级 MtProtoAdapter |
| `AddTelegramChat` | ❌ 删除 |
| 命名/版本 | 一并修正（§3 规范），版本 0.1.0-preview.1 起步 |

## 3. 系统架构（两系统一地基）

```
┌─ 商家入驻系统（对外公网 SaaS）      ┌─ 公司内部系统（内部合规）
│   多租户、自助入驻、配置后台        │   业务员同步、防撤回、网页回复、OA
│   商城/小程序、支付、计费           │   审查、部门隔离
└──────────────┬────────────────────┴──────────────┐
               │         功能重叠区（SDK 负责）：    │
               ▼  账号接入、收发、Handler、媒体、    │
        统一 Telegram SDK（共享地基）               │
        账号容器 / 消息模型 / MessageHandler /      │
        媒体 / 存储原语 / 双通道适配器              │
               ▼
        Telegram（Bot API / MTProto）
```

## 4. SDK vs 平台边界（最终裁定）

**原则：SDK = 通道能力地基（通用、与业务无关）；平台 = 业务服务（内容、配置、决策）。SDK 不读业务配置、不记业务状态、不做业务决策。**

### SDK 拥有（`Sb.Framework.Chat` + 通道模块）
账号接入/生命周期/凭证、消息收发与归一化模型、MessageHandler 管道、ChatContext/防撤回机制、AccountContainer（OwnerId/TenantId、Capabilities）、媒体下载/存储原语、双通道适配器、webhook 集中路由、内容承载（Web App 按钮/initData 校验/H5 入口）、支付通道能力、菜单/键盘构建发送。

### 平台层拥有（商家系统 + 内部系统各自实现）
自动回复规则引擎与内容、菜单/命令配置存储、客户标签/分组/粉丝语义、商城（商品/订单/购物流程）、支付业务编排（订单/退款）、计费/配额/分析/告警、业务员组织/审查流程/OA、网页 UI/配置后台。

### 多租户
SDK 账号模型保留 `OwnerId/TenantId` 判别字段 + `Capabilities`；租户隔离/计费/配置由平台实现。

## 5. 完整功能清单（必须 / 紧迫 / 阶段）

### 5.1 SDK 核心 `Sb.Framework.Chat`（全部必须）

| 模块 | P0 | P1 | P2 |
|------|----|----|----|
| 消息模型 | 文本/图片 + 类型注册表 | 视频/语音/文件 | — |
| MessageHandler | 分发基类 + 管道钩子 + 未知类型兜底 | 中间件链增强 | — |
| ChatContext/Container | 会话上下文 + 防撤回标记 | — | — |
| AccountContainer | OwnerId/TenantId、Capabilities、缓存策略 | 多账号批量管理 | — |
| 存储 | account/session/peer/message/media/sync_state | — | — |
| 出站发送 | 文本/图片 | 编辑/删除 | 批量群发 |
| 内容承载（双载体） | — | Web App 按钮+initData 校验 / H5 入口 | — |
| 支付抽象 | — | Payments/Stars 适配 | 对账 |

### 5.2 通道适配器

**`Sb.Framework.Chat.Telegram`（Bot，必须）**
| P0 | P1 | P2 |
|----|----|----|
| webhook/polling、图文收发、命令、回调、菜单 | 内联模式、Web App、支付 | 批量/定时 |

**`Sb.Framework.Chat.Telegram.MtProto`（个人号，必须）**
| P0 | P1 | P2 |
|----|----|----|
| 会话生命周期+认证流程、Update 流、历史补拉、**防撤回**、图文收发 | 完整媒体下载、同步增强 | 多设备 |

### 5.3 商家平台（对外公网 SaaS，完整系统清单）

| 功能 | 必须 | 紧迫 | 阶段 |
|------|------|------|------|
| 商家自助入驻（token/私人号接入） | ✅ 必须 | P0 | 1 |
| 账号管理（多账号/权限/停用） | ✅ 必须 | P0 | 1 |
| 消息路由 → 租户配置/Handler | ✅ 必须 | P0 | 1 |
| 自动回复（关键词/命令/关注回复） | ✅ 必须 | P1 | 2 |
| 客户管理（会话/参与者） | ✅ 必须 | P1 | 2 |
| 菜单配置（Reply/Inline Keyboard） | ✅ 必须 | P1 | 2 |
| 商城（Web App / H5 双载体） | ✅ 必须 | P1 | 2 |
| 支付接入（Stars/Payments） | ✅ 必须 | P1 | 2 |
| 素材管理 | ◻ 可选 | P2 | 3 |
| 群发/通知 | ◻ 可选 | P2 | 3 |
| 计费/配额 | ◻ 可选 | P2 | 3 |
| 数据分析/告警 | ◻ 可选 | P2 | 3 |

### 5.4 内部系统（场景一）

| 功能 | 必须 | 紧迫 | 阶段 |
|------|------|------|------|
| 业务员账号接入（MTProto） | ✅ 必须 | P0 | 1 |
| 消息同步 + 防撤回 | ✅ 必须 | P0 | 1 |
| 网页回复（图文） | ✅ 必须 | P0 | 1 |
| 撤回消息查看 + 内容检索 | ✅ 必须 | P0 | 1 |
| OA 对接 / 审查增强 | ◻ 可选 | P2 | 3 |

### 5.5 低成本高收益增强（已采纳全部 6 项）

| 增强 | 内容 | 阶段 | 接口/细节 |
|------|------|------|-----------|
| 统一通知推送 `INotificationService` | 业务系统一行式主动发送（通知/告警） | P1 | 见技术设计 §7.1 |
| 内置发送队列 + 频率控制 | 按账号限速、重试、保序（防 Telegram 限速） | P1 | §7.2 |
| Mock 适配器 | 内存收发，离线开发与单测 | P1 | §7.3 |
| 内置统一 Webhook 端点 | 一个 Controller 服务所有 Bot | P1 | §7.4 |
| 账号健康度 → /health | 连接状态/重连/队列积压接入框架健康检查 | P1 | §7.5 |
| 通用消息中间件 | 敏感词/审计/关键词回复，可插拔 | P2 | §7.6 |

## 6. Phase 1 与最终目标

**Phase 1（P0 全集）**：
- SDK 核心：图文收发 + MessageHandler + 防撤回载体 + 存储
- Bot 通道基础（webhook/polling/命令/回调/菜单）
- MtProto 通道基础（会话/认证/Update 流/历史补拉/防撤回/图文收发）
- 商家平台最小闭环：自助入驻 + 账号管理 + 消息路由
- 内部系统最小闭环：账号接入 + 同步防撤回 + 网页回复 + 撤回查看

**复刻范围阶段路线图（已确认）**：

| 阶段 | 加入的功能 |
|------|-----------|
| Phase 1 | 图文收发（双通道）、命令/回调/菜单（Bot）、防撤回+内容保留（个人号）、历史补拉/会话同步（个人号）、Mini Apps / H5 双载体入口 |
| Phase 2 | 视频/语音/文件收发、消息编辑、引用增强；群聊/频道支持；转发；支付（Stars/Payments） |
| Phase 3 | 表情反应、已读回执（视需要） |
| 不规划 | 语音通话/视频通话（网页复刻成本极高，明确排除） |

**最终目标**：
- SDK 成为完整多通道消息开发框架（双通道、双内容载体、支付、中间件体系、能力矩阵自动维护）
- 商家平台长成"微信公众平台级别"完整服务（入驻/配置/商城/支付/素材/群发/计费/分析）
- 内部系统长成完整合规工具（审查/检索/告警/OA 深度集成）

## 7. 盛派模式映射

| 盛派 | 统一 SDK | 职责 |
|------|---------|------|
| RequestMessage/ResponseMessage | `TelegramRequestMessageBase` / `TelegramResponseMessageBase` | 双通道归一化消息 |
| MessageHandler | `TelegramMessageHandler<TContext,TRequest,TResponse>` | 业务入口，override `OnXxxRequestAsync` |
| MessageContext/Container | `ChatContext` / `MessageContainer` | 会话上下文 + 防撤回载体 |
| Container/ContainerBag | `AccountContainer` / `AccountBag` | 账号凭证容器，自动刷新重连 |
| IContainerCacheStrategy | `IMessageCacheStrategy` | 可插拔缓存（框架 ICacheService） |
| 模块化 | 核心包 + Bot / MtProto 通道模块 | 分层解耦 |
| 管道钩子 | `OnExecutingAsync` / `OnExecutedAsync` | 中间件式扩展 |
| 未知类型兜底 | `OnUnknownRequestAsync` | 自定义消息类型注册 |

## 8. 包结构（已定）

```
Sb.Framework.Chat                ← 统一核心（无第三方依赖）
│   ├─ Model/ 消息模型 + MessageType 注册表
│   ├─ MessageHandler/ 分发基类 + 管道钩子
│   ├─ Context/  ChatContext / MessageContainer（有界 + 防撤回标记）
│   ├─ Account/  AccountContainer / AccountBag（OwnerId/TenantId、Capabilities）
│   ├─ Cache/    IMessageCacheStrategy → 桥接框架 ICacheService
│   └─ Storage/  统一实体 + 表声明（IModuleInitializer）
Sb.Framework.Chat.Telegram        ← Bot 通道模块（吸收现有 TelegramBotConnector）
Sb.Framework.Chat.Telegram.MtProto ← 个人号通道模块（生产化 WTelegramClient）
```

未来扩展：`Sb.Framework.Chat.Discord` / `.Slack`。版本 `0.1.0-preview.1` 起步。

## 9. 核心设计要点

### 9.1 统一消息模型
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
`RequestMessageText` / `RequestMessageImage` 继承；媒体类型用注册表扩展。

### 9.2 MessageHandler
```csharp
public class CustomMessageHandler : TelegramMessageHandler<ChatContext, TelegramRequestMessageBase, TelegramResponseMessageBase>
{
    protected override Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnImageRequestAsync(RequestMessageImage msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnCallbackRequestAsync(RequestMessageCallback msg) { ... }
}
```
同一套 Handler 服务双通道，业务不感知通道。

### 9.3 防撤回（SDK 机制，审查是业务）
```
收到消息 → 立即进 MessageContainer + 落库（含媒体字节，撤回前下载）
收到 delete 更新 → container.MarkWithdrawn(id) + 库中 withdrawn=1，内容保留
```
上限：系统离线期间"发出即撤"抓不到内容（写入产品说明）。

### 9.4 账号容器
`AccountBag { AccountId, OwnerId/TenantId, AccountType(Bot|Personal), Credential, SessionState, Capabilities, LastActive }`

### 9.5 通道适配器
```csharp
interface ITelegramAdapter
{
    Task ConnectAsync(AccountBag account);
    Task DisconnectAsync(AccountBag account);
    IAsyncEnumerable<TelegramRequestMessageBase> PullAsync();
    Task SendAsync(TelegramResponseMessageBase response);
}
```

### 9.6 存储（复用框架 DB，`IModuleInitializer` 建表）
`chat_account`(owner) / `chat_session`(加密) / `chat_peer` / `chat_message`(withdrawn) / `chat_media` / `chat_sync_state`
平台层配置表由平台系统各自声明：`chat_tenant_config`（自动回复/菜单/命令）。

## 10. 能力矩阵（FEATURES.md 雏形，随版本演进）

| 能力 | Bot | Personal |
|------|-----|----------|
| 收发文本/图片 | ✅ Phase1 | ✅ Phase1 |
| 撤回标记+内容保留 | — | ✅ Phase1 |
| 命令/回调/菜单 | ✅ | — |
| Mini Apps / Web Apps | ✅ | — |
| 支付（Stars/Payments） | 🚧 | — |
| 历史补拉/会话同步 | — | 🚧 |
| 视频/语音/文件 | 🚧 | 🚧 |
| 已读/在线/表情反应 | ❌ | ❌ |
| 多设备/会话多开 | ❌ | ❌ |

> 功能缺口文档化是硬要求——"以为支持了其实没有"是禁忌。

## 11. 未决问题

- ✅ 复刻范围阶段路线图：**已确认**（见 §6）
- ✅ 网页回复层职责边界：**已确认**（SDK 提供 API，网页 UI 归业务层）
- ✅ 登录认证方式选型：**不阻塞设计**（SDK 覆盖所有方式，业务实现阶段再选）
- ✅ 低成本高收益增强：**已确认全部 6 项采纳**（见 §5.5）
- ✅ 全局项目需求：**两个场景即全部需求，设计收敛完成**

## 12. 与 sb-framework 家族的关系（已规划）

**关系定位：仓库平级，依赖下级。**
- 仓库/分类平级：`sb-framework-chat` 与 `sb-framework` 是 `saibo/framework/` 兄弟仓库；`Sb.Framework.Chat` 与 `Sb.Framework.Core/Services/WebApi` 同为基座模块（规范 §3）
- 依赖下级：SDK 依赖框架核心（Core/Models）基础设施；框架绝不反向依赖 SDK（单向，防循环）
- 性质变化：旧 chat 仓库是协议中立独立库（不依赖框架）；新 SDK 是**框架依赖模块**（接存储/缓存/配置/日志/DI）

**依赖拓扑：**
```
sb-framework（Core/Models/Services/WebApi NuGet）
    ↑ 仅基础设施依赖
Sb.Framework.Chat（无第三方协议依赖）
    ↑
Sb.Framework.Chat.Telegram（+Telegram.Bot）/ Sb.Framework.Chat.Telegram.MtProto（+WTelegramClient）
    ↑ 按需引用
业务系统（商家平台/内部系统/PM/ERP/CRM）
```

**业务系统引用：**
```xml
<PackageReference Include="Sb.Framework.WebApi" Version="0.4.14" />
<PackageReference Include="Sb.Framework.Chat" Version="0.1.0-preview.1" />
<PackageReference Include="Sb.Framework.Chat.Telegram" Version="0.1.0-preview.1" />
<PackageReference Include="Sb.Framework.Chat.Telegram.MtProto" Version="0.1.0-preview.1" />
```
```csharp
builder.AddSbFramework();
builder.AddTelegramChat();          // SDK 核心装配
builder.AddTelegramBotAdapter();    // Bot 通道（按需）
builder.AddTelegramMtProtoAdapter();// 个人号通道（按需）
```

**重复功能抽象：SDK 复用框架基础设施，不复用框架业务抽象。**
FreeSql+IModuleInitializer 建表 / ICacheService / AppSettings+IOptions / Serilog / [AutoDependencyInjection] / AppTime(SB0001) / BaseBackgroundTask / Result —— 全部复用框架；消息模型/Handler 管道/通道适配器为 SDK 自有。

**版本治理：** SDK 版本独立，依赖框架包用版本下限（`[0.4.6,)`）；框架升级 → SDK 重新编译对齐（吸取 CHANGELOG 0.4.2/0.4.3 程序集版本事故教训）；同源发布 nuget.saibo.club。

### 12.1 抽象到 SB.Framework.Core 的通用工具（已规划）

SDK 需要的**通用型功能**抽象进 Core（`SB.Framework.Core/Util/`），SDK 与其它系统共用：

| 工具 | 职责 | 紧迫 |
|------|------|------|
| `RetryUtil` | 指数退避+抖动重试（重连/webhook 投递/HTTP/消息补偿） | P0 |
| `SecurityUtil.MaskSecret()` | 通用敏感值脱敏（token/手机号/API key/密码），扩展现有 SecurityUtil | P0 |
| `IdempotencyUtil` | 幂等去重（updateId/消息/Webhook 重复投递） | P1 |
| `RateLimiterUtil` | 统一限流入口（封装 System.Threading.RateLimiting） | P1 |
| `WebhookSignatureValidator` | HMAC 验签（平台 webhook 端点安全） | P1 |

**留在 SDK 的（Telegram 特定，不进 Core）**：消息模型、MessageHandler、ChatContext/防撤回、通道适配器、账号会话、`chat_*` 表实体。

## 13. 下一步

需求收敛 → 细化接口签名 → 定稿 → 落实现（先 MtProtoAdapter 生产化 + SDK 核心）。
