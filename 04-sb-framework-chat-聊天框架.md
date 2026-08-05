# 04 sb-framework-chat 聊天框架分析

> ⚠️ **本架构已被取代**：`SB.Chat / SB.Chat.Telegram` 经评审确定**弃用**，统一 SDK 见
> 《[07-Telegram-SDK-设计草案.md](./07-Telegram-SDK-设计草案.md)》（决策）与
> 《[08-Telegram-SDK-技术设计.md](./08-Telegram-SDK-技术设计.md)》（技术设计）。
> 本文保留为旧实现（当前仓库代码）的分析记录，仅作历史参考，**不要据此开发新功能**。

> 位置：`D:\projects\saibo\framework\sb-framework-chat`
> **定位**：不是可运行的应用，而是 sb-framework 家族中的 **IM 平台抽象层组件**，采用「契约 + 适配器」二分结构，以独立 NuGet 包形式被派生应用消费，与后端核心**无代码耦合**（仅共享治理规范与 NuGet feed）。

## 1. 解决方案结构

`sb-framework-chat.sln`（3 个项目，全部 `net8.0`）：

| 项目 | 类型 | 角色 |
|------|------|------|
| `SB.Chat` | classlib（可打包） | IM 抽象层（协议中立） |
| `SB.Chat.Telegram` | classlib（可打包） | Telegram 适配器 |
| `SB.Chat.Tests` | test（不可打包） | xUnit 单元测试 |

当前版本 `0.1.0-preview.1`。技术债：工程命名 `SB.Chat` 不符合 Saibo §3 规范（应为 `Sb.Framework.Chat.*`），计划 v0.2.0 重命名。

## 2. SB.Chat（抽象层）

纯抽象层，无 `Program.cs`。核心接口 + 模型：

### 2.1 `IChatBot`（Bot 账号连接器，`SB.Chat\IChatBot.cs:9`）

同时支持两种接收模式：
- **Polling**：`StartPollingAsync()` / `StopPollingAsync()`
- **Webhook**：`SetWebhookAsync(url)` / `DeleteWebhookAsync()` / `ProcessWebhookAsync(payload)`

连接管理：`ConnectAsync` / `DisconnectAsync` / `IsConnected`。
发送：`SendTextAsync`、`SendMenuAsync`（按钮菜单）、`EditMessageAsync`。
回调：`AnswerCallbackAsync`。
媒体：`DownloadMediaAsync`、`GetUserAvatarAsync`。
事件（标准 `EventHandler<T>`）：`OnMessage`、`OnCallback`、`OnGroupEvent`。

### 2.2 `IChatClient`（个人号 / MTProto 连接器，`SB.Chat\IChatClient.cs:8`）

最小化接口：`Connect` / `Disconnect` / `DownloadMedia` / `OnMessage`。

### 2.3 核心模型（`SB.Chat\Models\`）

| 模型 | 说明 |
|------|------|
| `ChatMessageEvent` | 消息事件（含 `RawData` 保留原始对象、`Media` 列表、`IsPrivateChat`） |
| `ChatCallbackEvent` | 内联按钮回调 |
| `ChatGroupEvent` + `ChatGroupEventType` | 群事件枚举：BotAdded/BotRemoved/MemberJoined/MemberLeft |
| `ChatMediaInfo` | 媒体元信息（FileId/Size/MimeType/MediaType） |
| `ChatMessageType` | 枚举：Text/Image/File/Audio/Video/Sticker |
| `MenuButton` | 按钮（`text`、`callbackData`） |

### 2.4 抽象层纯净性约束

- CI `lint:naming`（`.gitlab-ci.yml:15-25`）用正则禁止派生业务类型（`ChatRequirement|ChatBatch|...`）污染抽象层。
- `CONTRIBUTING.md:9-15`「三不准」：抽象层不得依赖具体 IM 协议。

## 3. SB.Chat.Telegram（适配器）

同时提供 **Bot（官方 Bot API）** 和 **个人号（MTProto）** 两条接入路径。

> 引用方式：同解决方案内用 **NuGet PackageReference**（`SB.Chat` 0.1.0-preview.1）而非 ProjectReference —— 强调「框架库」与「应用」边界，模拟派生项目真实消费方式。

### 3.1 `TelegramBotConnector : IChatBot`（`TelegramBotConnector.cs:16`）

基于 **Telegram.Bot 22.4.2** 官方 SDK：
- `new TelegramBotConnector(token)`，`ConnectAsync` 内 `GetMe()` 验活
- **Polling**：`StartReceiving(...)` + CancellationTokenSource 控制；`AllowedUpdates` 限定 Message/CallbackQuery/MyChatMember/ChatMember
- **Webhook**：直调 Bot API；`ProcessWebhookAsync` 用 `JsonSerializer.Deserialize<Update>(payload, JsonBotAPI.Options)` 解析
- **统一分发**：`ProcessUpdate`（Polling 与 Webhook 共用）按 `MyChatMember → CallbackQuery → Message` 路由到事件
- **媒体抽取** `ExtractMedia`：覆盖 photo/animation/document/audio/voice/video/video_note/sticker
- 所有公有方法前置 `EnsureConnected()`，未连接抛 `InvalidOperationException`

### 3.2 `TelegramClientConnector : IChatClient`（`TelegramClientConnector.cs:10`）

基于 **WTelegramClient 4.2.1**（MTProto 客户端），用于「Telegram 个人号连接器，被动采集群组消息」：
- `new TelegramClientConnector(apiId, apiHash, phone, sessionPath)`，凭据经 `ConfigProvider` 回调（`TelegramClientConnector.cs:77`）注入
- `HandleUpdates` 处理 `UpdateNewMessage` / `UpdateNewChannelMessage`，自动忽略自己消息
- ⚠️ `DownloadMediaAsync` 是简化实现（access_hash/file_reference 缺失），已知半成品

### 3.3 DI 扩展（空壳）

`AddTelegramChat(this IServiceCollection)` 当前为空壳——连接器因每账号凭据不同，通过 `new` 直接创建、不走 DI 容器（`TelegramChatExtensions.cs:12-14`）。预留挂载点。

## 4. SB.Chat.Tests（测试）

- **框架**：xUnit 2.5.3 + Microsoft.NET.Test.Sdk 17.8.0 + coverlet 覆盖率。
- **策略亮点**：不触网。`TelegramBotConnectorWebhookTests` 用反射调私有 `ProcessUpdate` + Telegram.Bot 内置 `JsonBotAPI.Options` 反序列化真实 JSON payload，完整覆盖 Update→事件映射（text/photo/audio/video/sticker/callback/群成员变化）。
- CI 中测试 `allow_failure: true`。

## 5. 架构模式

```
派生 App (消费者)  ── NuGet: SB.Chat ──▶  SB.Chat (IChatBot/IChatClient/Models) ← 协议中立抽象
                                                       ▲
                                    NuGet: SB.Chat.Telegram
                                                       │
                        SB.Chat.Telegram (TelegramBotConnector + TelegramClientConnector)
未来: SB.Chat.Discord / Slack / WeChat ...
```

- **契约与实现分离**：`SB.Chat` 零第三方依赖，适配器独立打包、独立演进。
- **可扩展点已留**：`AddTelegramChat`（空壳）+「新增 IM 适配器」指南（CONTRIBUTING.md）。

## 6. 部署与配置

- **无 appsettings.json / Dockerfile**：类库不持有运行时配置。凭据由派生应用管理，经构造函数注入：
  - Bot：`new TelegramBotConnector(token)`
  - 个人号：`new TelegramClientConnector(apiId, apiHash, phone, sessionPath)`
- **CI（GitLab，5 阶段）**：validate（lint:naming）→ build → test（allow_failure）→ package（只 pack SB.Chat + SB.Chat.Telegram）→ publish（仅 tag `vX.Y.Z` 触发，push 到 GitLab project-level NuGet feed）。

## 7. 关键代码位置速查

| 内容 | 位置 |
|------|------|
| Bot 接口 | `SB.Chat\IChatBot.cs:9` |
| Client 接口 | `SB.Chat\IChatClient.cs:8` |
| Telegram Bot 适配 | `SB.Chat.Telegram\TelegramBotConnector.cs:16`（Polling:47 / Webhook:82 / 分发:196 / 媒体:298） |
| MTProto 适配 | `SB.Chat.Telegram\TelegramClientConnector.cs:10`（ConfigProvider:77） |
| DI 扩展空壳 | `SB.Chat.Telegram\Extensions\TelegramChatExtensions.cs:15` |
| 抽象纯净性守卫 | `.gitlab-ci.yml:15-25` |
| 引用方式 | `SB.Chat.Telegram\SB.Chat.Telegram.csproj:24` |
