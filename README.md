# SB Framework 家族框架文档

> 赛博科技（Saibo）自研开发框架家族 — 全家族架构分析 + 新项目开发指南
>
> 生成日期：2026-08-05 · 依据源码实测（非 README 旧描述）

## 文档目录

| 文档 | 内容 |
|------|------|
| [01-架构总览与技术栈.md](./01-架构总览与技术栈.md) | 家族整体架构、三大子项目关系、技术栈总览、核心设计理念 |
| [02-sb-framework-后端框架.md](./02-sb-framework-后端框架.md) | .NET 10 后端基座框架深度分析（Core/Models/Services/WebApi/插件/流程引擎） |
| [03-sb-framework-admin-前端框架.md](./03-sb-framework-admin-前端框架.md) | Vben Admin 定制前端基座深度分析（monorepo/权限/请求/页面开发） |
| [04-sb-framework-chat-聊天框架.md](./04-sb-framework-chat-聊天框架.md) | 旧 SB.Chat 架构分析（**已被 07/08 取代**，历史参考） |
| [05-新项目开发指南.md](./05-新项目开发指南.md) | **基于框架从零搭建新项目的完整操作指南（后端+前端）** |
| [06-Chat-接入指南.md](./06-Chat-接入指南.md) | Telegram 统一 SDK 接入指引（旧接入方式已废弃） |
| [07-Telegram-SDK-设计草案.md](./07-Telegram-SDK-设计草案.md) | **Telegram 统一 SDK 决策文档**（需求/边界/阶段/优先级，WIP） |
| [08-Telegram-SDK-技术设计.md](./08-Telegram-SDK-技术设计.md) | **Telegram 统一 SDK 技术设计**（接口签名/存储表/时序/配置） |

## 框架家族速览

```
D:\projects\saibo\framework\
├── sb-framework          ← .NET 10 后端基座（NuGet 分发，引包即用）
│     ├── SB.Framework.Core / Models / Services / WebApi   ← 5 个核心 NuGet 包
│     ├── SB.Feature.AccessToken / BackupReceive / LicenseManage  ← 插件
│     ├── SB.FlowEngine    ← 流程执行引擎
│     └── SB.Framework.Analyzers / DevHost
│
├── sb-framework-admin    ← Vue3 + TS 前端基座（Vben Admin v5.5.9 fork）
│     └── apps/web-antd   ← 唯一应用（Ant Design Vue，端口 6012）
│     └── packages/@core + packages/effects + internal/  ← 可复用库
│
└── sb-framework-chat     ← Telegram 统一 SDK（双通道 Bot + 个人号）
      ├── Sb.Framework.Chat              ← 统一核心（模型/MessageHandler/Account/存储）
      ├── Sb.Framework.Chat.Telegram     ← Bot 通道适配器
      └── Sb.Framework.Chat.Telegram.MtProto ← 个人号通道适配器（旧 SB.Chat 弃用）
```

## 关键结论速记

- **后端**：.NET 10 + FreeSql ORM + 双 JWT 认证 + Serilog + 插件化（`SB.Feature.*`）+ 声明式数据装配（`IModuleInitializer`）。核心理念是「两行代码装配，引包即启用」。
- **前端**：Vue 3.5 + Vite 7 + Ant Design Vue + Pinia + Vue Router + vxe-table，backend 菜单驱动 + mixed 权限模式，axios 请求封装含 token 刷新/安装向导/授权拦截。
- **Chat**：旧 `SB.Chat` 弃用，重构为统一 SDK `Sb.Framework.Chat.*`（盛派风格，双通道 Bot + 个人号），服务商家平台与内部系统两套系统（见 07/08）。
- **派生方式**：后端走 **NuGet 消费 + 子类覆盖 + 插件**；前端走 **fork + merge 同步**（基座漂移审计）。
