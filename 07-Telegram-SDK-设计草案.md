# Telegram SDK 设计草案（WIP）

> 状态：**草案，持续修订中**。全局项目需求尚未描述完，本文档随讨论更新。
> 位置：家族文档库 `framework/docs/`（非 sb-framework-chat 仓库内，命名/仓库归属待定）。

## 0. 定位

基于 sb-framework 家族 + Telegram 双通道（Bot API + MTProto 个人号）的**统一全能消息开发 SDK**。设计风格参考盛派（Senparc）微信 SDK：强类型消息模型 + MessageHandler 分发 + 容器/缓存 + 模块化。

服务**两套独立系统**（商家入驻平台 + 公司内部系统），作为两者共享的统一地基。

## 1. 需求约束汇总（截至当前）

| 维度 | 约束 |
|------|------|
| 通道 | **个人号（MTProto）+ Bot（Bot API）双通道，做成一套全能 SDK** |
| 账号开放范围 | SDK 阶段**全开放双通道**；实际业务通过权限控制（谁能用个人号/支付/Mini App 由上层落地） |
| 系统形态 | **两套独立系统**：商家入驻（外部多租户 SaaS）+ 公司内部（内部合规），**功能有重叠，共享同一 SDK 地基** |
| 场景一（内部个人号） | 业务员手机登录 → 系统同步消息到内网 → 网页端以业务员身份回复 → 防撤回；≤100 业务员 |
| 场景二（商家平台） | Telegram 版微信公众号平台：商家自助填 token/私人号，平台提供服务（商城/小程序/支付/通知） |
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

## 3. 统一架构（盛派风格）+ 两系统一地基

```
┌─ 商家入驻系统（外部 SaaS）       ┌─ 公司内部系统（内部合规）
│   多租户、自助接入、配置后台     │   业务员同步、防撤回、网页回复、OA
│   商城/小程序、支付、计费        │   审查、部门隔离
└──────────────┬─────────────────┴──────────────┐
               │          功能重叠区：           │
               ▼  账号接入、收发、Handler、媒体、 │
        统一 Telegram SDK（共享地基）            │
        账号容器 / 消息模型 / MessageHandler /   │
        媒体 / 存储原语 / 双通道适配器           │
               ▼
        Telegram（Bot API / MTProto）
```

**功能重叠区 = SDK 负责**：账号接入与容器、消息收发、归一化模型、MessageHandler 管道、媒体、存储原语、双通道适配器、webhook 集中管理。

**各自独有 = 系统层负责**：商家系统（租户隔离、自助入驻、配置后台、商城/小程序、支付、计费）；内部系统（业务员组织、防撤回审查、网页回复、OA 对接）。

## 4. 多租户边界（已确认）

- **SDK 不内置完整多租户逻辑**，但账号模型保留 **`OwnerId`/`TenantId` 判别字段**：
  - 内部系统 = 单一 owner（或按部门细分）
  - 商家系统 = 每个商家一个 owner
- 租户的业务隔离、计费、配置由各自系统实现，SDK 只做账号/消息路由的通用承载。
- **权限控制（谁能用个人号 / 支付 / Mini App）由上层系统按租户/角色落地**；SDK 暴露能力/属性层（如 `AccountBag.Capabilities`）供业务读取决策，SDK 自身全开放、不内置业务权限。

## 5. 场景二：商家入驻平台（多租户 SaaS）

```
商家A → 填 Bot token ─┐
商家B → 填 Bot token ─┼─→ 平台（统一账号管理 + 消息路由 + 服务能力）
商家C → 填私人号  ───┘           ↓
                         商家各自的客户（终端用户）
```

**微信公众平台能力 → Telegram 映射（平台层提供）：**

| 微信公众平台 | Telegram 对应物 | 平台要提供的 |
|-------------|----------------|-------------|
| AppID/Secret 接入 | Bot token 接入 | 商家自助接入页 |
| 自动回复（关注/关键词） | 命令 + 文本匹配回复 | 自动回复配置后台 |
| 自定义菜单 | ReplyKeyboard / Inline Keyboard | 菜单配置后台 |
| 小程序 | **Telegram Web Apps（Mini Apps）** | 商城/服务的 H5 载体 |
| 微信支付 | **Bot Payments API / Stars** | 商城支付 |
| 模板消息/群发 | Bot 主动消息（start 后随时可发） | 通知推送 |
| 粉丝管理 | 会话参与者 | 客户列表 |
| 网页授权 | Web App initData 鉴权 | 商城登录态 |

**对 SDK 的直接影响**：账号模型带 `OwnerId/TenantId`；集中消息路由（消息 → 识别账号 → 路由到租户配置/Handler）；服务配置数据模型（`chat_tenant` / `chat_tenant_config`：自动回复规则、菜单、命令）——平台后台的存储基础。

## 6. 盛派模式映射

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

## 7. 包结构（已定：并入 sb-framework-chat，命名照规范 §3）

```
Sb.Framework.Chat                ← 统一核心（无第三方依赖）
│   ├─ Model/ 消息模型 + MessageType 注册表
│   ├─ MessageHandler/ 分发基类 + 管道钩子
│   ├─ Context/  ChatContext / MessageContainer（有界 + 防撤回标记）
│   ├─ Account/  AccountContainer / AccountBag（含 OwnerId/TenantId、Capabilities）
│   ├─ Cache/    IMessageCacheStrategy → 桥接框架 ICacheService
│   └─ Storage/  统一实体 + 表声明（IModuleInitializer）
Sb.Framework.Chat.Telegram        ← Bot 通道模块（吸收现有 TelegramBotConnector）
Sb.Framework.Chat.Telegram.MtProto ← 个人号通道模块（生产化 WTelegramClient）
```

- 旧 `SB.Chat` / `SB.Chat.Telegram` **删除/改名**，规范债一次还清
- 未来扩展：`Sb.Framework.Chat.Discord` / `.Slack`
- 版本从 `0.1.0-preview.1` 起步（规范 §5 SemVer）

## 8. 核心设计要点

### 8.1 统一消息模型
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

### 8.2 MessageHandler（业务入口）
```csharp
public class CustomMessageHandler : TelegramMessageHandler<ChatContext, TelegramRequestMessageBase, TelegramResponseMessageBase>
{
    protected override Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnImageRequestAsync(RequestMessageImage msg) { ... }
    protected override Task<TelegramResponseMessageBase?> OnCallbackRequestAsync(RequestMessageCallback msg) { ... }
}
```
同一套 Handler 同时服务 bot 与个人号，业务不感知通道。

### 8.3 防撤回（容器/上下文机制）
```
收到消息 → 立即进 MessageContainer + 落库（含媒体字节，撤回前下载）
收到 delete 更新 → container.MarkWithdrawn(id) + 库中 withdrawn=1，内容保留
```
上限：系统离线期间"发出即撤"抓不到内容（写入产品说明）。

### 8.4 账号容器
`AccountBag { AccountId, OwnerId/TenantId, AccountType(Bot|Personal), Credential, SessionState, Capabilities, LastActive }`
Bot：token；Personal：MTProto 会话（加密存储）+ 认证状态机 + 断线重连。缓存策略可插拔。

### 8.5 通道适配器（内部）
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

### 8.6 存储（统一表，复用框架 DB）
`chat_account`(含 owner) / `chat_session`(加密) / `chat_peer` / `chat_message`(withdrawn) / `chat_media` / `chat_sync_state` / `chat_tenant_config`(平台层)

## 9. 能力矩阵初始版（FEATURES.md 雏形）

| 能力 | Bot | Personal |
|------|-----|----------|
| 收发文本/图片 | ✅ Phase1 | ✅ Phase1 |
| 撤回标记+内容保留 | — | ✅ Phase1 |
| 命令/回调/菜单 | ✅ | — |
| Mini Apps / Web Apps | ✅ | — |
| 支付（Stars/Payments API） | 🚧 | — |
| 历史补拉/会话同步 | — | 🚧 |
| 视频/语音/文件 | 🚧 | 🚧 |
| 已读/在线/表情反应 | ❌ | ❌ |
| 多设备/会话多开 | ❌ | ❌ |

> 功能缺口文档化是硬要求——"以为支持了其实没有"是禁忌。

## 10. 待确认 / 未决问题

- ✅ 已确认：多租户边界（SDK 带 OwnerId/TenantId，不内置租户逻辑）；账号全开放、权限上层落地；仓库并入 sb-framework-chat；命名 `Sb.Framework.Chat.*`（规范 §3）
- ⏳ 全局项目剩余需求（用户继续描述中）
- ⏳ 商家平台：对外公网 vs 内部多业务线；商城载体（Telegram Web Apps vs 平台 H5）；平台后台功能范围
- ⏳ 登录认证方式的具体选型（业务阶段）
- ⏳ 复刻范围后续阶段清单
- ⏳ 网页回复层的职责边界（SDK 提供 API，网页 UI 归业务层）

## 11. 下一步

- 继续全局需求描述，本文档随讨论更新
- 需求收敛后：细化接口签名 → 定稿 → 落实现（先 MtProtoAdapter 生产化，含会话/同步/防撤回）
