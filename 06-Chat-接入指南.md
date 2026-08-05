# 06 Telegram 接入指南（已更新）

> ⚠️ **旧《SB.Chat 接入指南》已废弃**：`SB.Chat / SB.Chat.Telegram` 架构经评审确定弃用，
> 由统一 SDK **`Sb.Framework.Chat.*`** 取代（参考盛派微信 SDK 风格，双通道 Bot + 个人号）。
> 本文档为新的接入指引，具体接口以技术设计为准。

## 1. 当前状态

- **旧接入方式**（`new TelegramBotConnector(token)` / `new TelegramClientConnector(...)`）**不可再用**——旧层将被删除。
- **新接入方式**：通过统一 SDK 的装配扩展，见下。
- 完整决策与接口见《[07-Telegram-SDK-设计草案.md](./07-Telegram-SDK-设计草案.md)》与《[08-Telegram-SDK-技术设计.md](./08-Telegram-SDK-技术设计.md)》。

## 2. 新 SDK 接入方式（设计态，实现后生效）

```xml
<PackageReference Include="Sb.Framework.WebApi" Version="0.4.14" />
<PackageReference Include="Sb.Framework.Chat" Version="0.1.0-preview.1" />
<PackageReference Include="Sb.Framework.Chat.Telegram" Version="0.1.0-preview.1" />        <!-- Bot 通道 -->
<PackageReference Include="Sb.Framework.Chat.Telegram.MtProto" Version="0.1.0-preview.1" /> <!-- 个人号通道 -->
```

```csharp
builder.AddSbFramework();
builder.AddTelegramChat();           // SDK 核心装配
builder.AddTelegramBotAdapter();     // Bot 通道（按需）
builder.AddTelegramMtProtoAdapter(); // 个人号通道（按需）
```

## 3. 业务代码形态（盛派风格）

```csharp
public class CustomMessageHandler : TelegramMessageHandler<ChatContext>
{
    protected override Task<TelegramResponseMessageBase?> OnTextRequestAsync(RequestMessageText message, CancellationToken ct)
    {
        // 同一 Handler 服务 Bot 与个人号，不感知通道
        return Task.FromResult<TelegramResponseMessageBase?>(new ResponseMessageText { Content = "收到" });
    }
}
```

## 4. 防撤回（个人号通道）

收到即存 → 撤回时标记 `withdrawn=1`、内容保留。审查检索：`chat_message WHERE withdrawn=1`。
> 上限：系统离线期间"发出即撤"抓不到内容。

## 5. 与框架的关系

- SDK 是框架基座模块（仓库平级、依赖下级），复用框架存储/缓存/配置/日志/DI。
- 通用工具（`RetryUtil` 等）抽象进 `SB.Framework.Core/Util`。

> 具体接口签名、存储表、时序、配置以《08-Telegram-SDK-技术设计.md》为准。
