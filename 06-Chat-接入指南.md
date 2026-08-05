# 06 Chat 接入指南

> 在派生应用中接入 `SB.Chat` 抽象层与 `SB.Chat.Telegram` 适配器，实现 Telegram Bot / 个人号消息收发。

## 1. 接入流程

```
1. 引用 NuGet 包（SB.Chat + SB.Chat.Telegram）
2. 创建连接器实例（new，不走 DI —— 每账号凭据不同）
3. 订阅事件（OnMessage / OnCallback / OnGroupEvent）
4. 启动（Polling 或 Webhook）
5. 把聊天逻辑编排为派生业务（供 PM/ERP/CRM 使用）
```

## 2. 引用包

```xml
<ItemGroup>
  <PackageReference Include="SB.Chat" Version="0.1.0-preview.1" />
  <PackageReference Include="SB.Chat.Telegram" Version="0.1.0-preview.1" />
</ItemGroup>
```

## 3. 创建连接器

### 3.1 Telegram Bot（官方 Bot API）

```csharp
using SB.Chat.Telegram;

var bot = new TelegramBotConnector(token);   // token 由 @BotFather 创建
await bot.ConnectAsync();                     // 内部 GetMe() 验活
```

**Polling 模式**（推荐起步）：

```csharp
bot.OnMessage += async (sender, e) =>
{
    // e.ChatId / e.Message / e.MessageType / e.Media / e.IsPrivateChat / e.RawData
    await bot.SendTextAsync(e.ChatId, $"收到: {e.Message}");
    if (e.Media is { } media)
    {
        // media.FileId / Size / MimeType / MediaType
        var bytes = await bot.DownloadMediaAsync(e.ChatId, media.FileId);
    }
};

bot.OnCallback += async (sender, e) =>
{
    // 内联按钮回调：e.CallbackData / e.ChatId / e.MessageId
    await bot.AnswerCallbackAsync(e.ChatId, e.CallbackData);
};

bot.OnGroupEvent += async (sender, e) =>
{
    // e.EventType: BotAdded/BotRemoved/MemberJoined/MemberLeft
};

await bot.StartPollingAsync();
```

**Webhook 模式**（生产推荐）：

```csharp
await bot.SetWebhookAsync("https://your-domain.com/webhook/telegram");
// 收到 POST 时：
await bot.ProcessWebhookAsync(payloadBody);   // 分发到 OnMessage/OnCallback/OnGroupEvent
// 停用：await bot.DeleteWebhookAsync();
```

### 3.2 Telegram 个人号（MTProto，被动采集群消息）

```csharp
using WTelegramClient;

var client = new TelegramClientConnector(apiId, apiHash, phone, sessionPath);
client.OnMessage += async (sender, e) => { /* 处理群消息 */ };
await client.ConnectAsync();
```

> ⚠️ 个人号 `DownloadMediaAsync` 是简化实现（缺 access_hash/file_reference），大文件媒体下载可能受限。

## 4. 事件模型速查

| 事件 | 类型 | 关键字段 |
|------|------|----------|
| `OnMessage` | `ChatMessageEvent` | ChatId、Message、MessageType(Text/Image/File/Audio/Video/Sticker)、Media[]、IsPrivateChat、RawData |
| `OnCallback` | `ChatCallbackEvent` | ChatId、MessageId、CallbackData |
| `OnGroupEvent` | `ChatGroupEvent` | ChatId、EventType(BotAdded/BotRemoved/MemberJoined/MemberLeft) |

## 5. 菜单按钮

```csharp
var buttons = new[]
{
    new MenuButton("开始", "start"),
    new MenuButton("帮助", "help"),
};
await bot.SendMenuAsync(chatId, "请选择:", buttons);
// 用户点击 → OnCallback（CallbackData 为 "start"/"help"）
```

## 6. 与 SB Framework 业务层结合（推荐模式）

将聊天编排为派生业务服务，供 Controller / 后台任务调用：

```csharp
// Sb.Apps.<App>.Services.Chat
[AutoDependencyInjection]
public class MyChatService
{
    private readonly IChatBot _bot;

    // 连接器可单例持有（框架内长生命周期对象）
    public MyChatService()
    {
        _bot = new TelegramBotConnector(Configuration["Telegram:BotToken"]!);
        _bot.OnMessage += HandleMessageAsync;
    }

    private async void HandleMessageAsync(object? s, ChatMessageEvent e) { ... }

    public Task SendProjectUpdateAsync(long chatId, string text)
        => _bot.SendTextAsync(chatId, text);
}
```

> 连接器不走 DI 容器（`AddTelegramChat` 目前是空壳），通过 `new` 创建并自行管理生命周期。

## 7. 新增 IM 适配器（扩展框架）

参照 `SB.Chat.Telegram` 模式（CONTRIBUTING.md:25-31）：

1. 新建 `SB.Chat.<Platform>` classlib，`PackageReference` 引 `SB.Chat`
2. 实现 `IChatBot`（和/或 `IChatClient`）
3. 独立打包、独立发版（GitLab 打 `vX.Y.Z` tag → 自动 publish NuGet）

## 8. 约定与注意事项

- **抽象层纯净性**：不要在 `SB.Chat` 中引入派生业务类型（CI 会失败）。
- **凭据管理**：token/apiId/apiHash/phone 由派生应用配置管理，经构造函数注入，勿硬编码。
- **版本现状**：`SB.Chat` 工程命名待 v0.2.0 规范化为 `Sb.Framework.Chat.*`（升级时注意包名变更）。
- **测试**：连接器逻辑可离线测试（参照 `SB.Chat.Tests` 用反射 + `JsonBotAPI.Options` 反序列化真实 payload，不触网）。
