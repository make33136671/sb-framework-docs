# Chat.Realtime 通信保障层升级方案

> 状态:已实现并入库(Sb.Framework.Chat.Realtime 0.1.0-preview.1 + @sb/realtime-core)
> 本文为方案归档:设计目标、组件划分、保障机制与接入方式,与代码现状同步。

## 一、背景与目标

Telegram SDK(Sb.Framework.Chat.*)解决了"通道接入",但没有解决"面向终端用户的实时聊天"所需的保障能力。
本方案新增独立的实时通讯层,目标:

1. **消息可靠送达**:发送→落库→推送→确认的状态机,离线补投
2. **在线状态**:多端心跳、在线/离线判定、批量查询
3. **连接保障**:限流(防恶意连接)、心跳超时清理、连接广播
4. **前端 SDK**:SignalR 封装,断线重连 + 断路器 + Token 自动续期,消除旧 OpenIM monkey-patch
5. **可插拔**:认证/用户资料/事件汇由业务系统实现,框架提供默认兜底

## 二、后端(Sb.Framework.Chat.Realtime)

| 组件 | 职责 |
|------|------|
| `Hub/RealtimeHub` | SignalR Hub:认证、收发消息、会话、已读、在线 |
| `Auth/IRealtimeAuthProvider` | 业务接入认证(默认实现:userId 非空即放行,兜底) |
| `Auth/IRealtimeUserProvider` | 业务提供用户资料(默认:仅透传 userId) |
| `Presence/PresenceService` | 在线状态:多连接聚合、心跳超时判定、批量 IsOnline |
| `Messages/MessageService` | 消息发送/落库/类型注册表(MessageTypeRegistry)/系统消息 |
| `Messages/MessageDeliveryService` | 投递与状态回执(发送中/已送达/已读) |
| `Messages/RealtimeMessage` | 统一消息模型(contentType 区分业务类型与 sys.* 系统消息) |
| `Conversations/ConversationService` | 会话列表/成员/未读数 |
| `ReadState/ReadStateService` | 已读水位 |
| `Media/MediaService` | 聊天媒体 |
| `Storage/Rt*` | rt_conversations / rt_conversation_members / rt_messages / rt_media / rt_read_state 实体 + RealtimeModuleInitializer |
| `Filters/ConnectionRateLimitFilter` | Hub 连接/调用限流 |
| `Services/ConnectionBroadcastService` | 上下线广播 |
| `Services/HeartbeatCleanupService` | 后台心跳超时清理 |
| `Events/RealtimeEventBus` + `IRealtimeEventSink` | 事件总线(业务订阅;未实现时 NullEventSink 兜底) |
| `Endpoints/RealtimeHealthEndpoint` | /realtime-health 健康端点 |
| `Options/RealtimeOptions` | 心跳超时/限流阈值等配置 |
| `Extensions/SbRealtimeExtensions` | `AddSbRealtime()` 一键装配 |

测试:`Sb.Framework.Chat.Realtime.Tests`(Presence 9 用例全过)。

## 三、前端(sb-framework-chat-js)

| 包 | 内容 |
|----|------|
| `@sb/realtime-core` | RealtimeClient(SignalR 连接管理)、AuthService(登录/TokenRefresher 续期)、MessageService、ConversationService、UploadService、EventManager、ReconnectManager(指数退避重连)、CircuitBreaker(断路器)、SessionManager |
| `@sb/chat-ui` | Shadow DOM 隔离的可插拔聊天 UI:ChatUI/MessageListRenderer/InputArea/PluginManager,文本/图片/表情插件,i18n |

派生项目引用方式见 `sb-framework-chat-js/README.md`(本地 file: 依赖)。

## 四、保障机制要点

1. **心跳与离线判定**:PresenceService 以连接为单位聚合,HeartbeatCleanupService 周期扫描超时(RealtimeOptions.HeartbeatTimeout,默认 60s)
2. **限流**:ConnectionRateLimitFilter 在 Hub 层拦截,防连接风暴
3. **重连**:前端 ReconnectManager 指数退避;CircuitBreaker 在服务端不可用时快速失败避免雪崩
4. **Token 续期**:TokenRefresher 在过期前自动刷新,401 触发重建连接
5. **消息幂等与状态**:MessageService 落库先行,投递状态机可查询,离线消息由会话拉取兜底

## 五、接入方式

```csharp
// 派生项目 API 端
builder.AddSbRealtime();   // 装配(可替换 IRealtimeAuthProvider / IRealtimeUserProvider / IRealtimeEventSink)
```

前端:`new RealtimeClient({ baseURL, wsAddr })` → `login(userId, token)` → 事件订阅/收发。

## 六、待办

- 真实环境联调(多端在线、弱网重连、媒体上传)
- 已读回执与会话未读数的 UI 打通(chat-ui)
- 离线推送通道(如需)
