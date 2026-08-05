# 02 sb-framework 后端框架深度分析

> 位置：`D:\projects\saibo\framework\sb-framework`
> 注意：README/docs 部分文档仍描述旧版（.NET 8、fork 模式、MasterKey），**本文以源码为准**（.NET 10 + 插件化 + AccessToken）。

## 1. 解决方案结构

`sb-framework.sln`（11 个工程，依赖拓扑）：

```
Analyzers (netstandard2.0, 以 analyzer 形式被所有包引用)
    ↑
Models (只引 FreeSql)          ← FlowEngine 只引 Core
    ↑
Core (引 Models + Analyzers)
    ↑
Services (引 Core + Models)
    ↑
WebApi (引 Services)           ← DevHost 引 WebApi + 2 个 Feature
```

| 工程 | NuGet PackageId | 版本 | 用途 |
|------|-----------------|------|------|
| SB.Framework.Core | `SB.Framework.Core`（变体 `.NoLicense`） | 0.4.6 | 原语：Auth、License、Util、Middleware、插件框架、BaseCrudService |
| SB.Framework.Models | `SB.Framework.Models` | 0.4.7 | 实体、DTO、枚举、`IEntity`/`IModuleInitializer`/`Result` |
| SB.Framework.Services | `SB.Framework.Services` | 0.4.13 | 应用服务 + 扩展点（`IFreeSqlConfigurator`/`IStartupSyncHook`） |
| SB.Framework.WebApi | `SB.Framework.WebApi` | 0.4.14 | Controllers + 中间件 + 一键装配 |
| SB.Framework.Analyzers | `SB.Framework.Analyzers` | 0.4.6 | Roslyn 编译期约束 SB0001/SB0002 |
| SB.Framework.DevHost | （不发包） | — | 本地调试启动壳 |
| SB.FlowEngine | `SB.FlowEngine` | 0.2.1 | 图执行引擎 |
| SB.Feature.AccessToken | `SB.Feature.AccessToken` | 0.2.3 | 统一访问令牌 |
| SB.Feature.BackupReceive | `SB.Feature.BackupReceive` | 0.1.1 | 云端备份接收端 |
| SB.Feature.LicenseManage | `SB.Feature.LicenseManage` | 0.1.1 | 授权签发端 |

> Feature 包只引 Core + Models（不引 Services/WebApi），通过 `FrameworkReference: Microsoft.AspNetCore.App` 拿 MVC 类型 —— 避免循环依赖的关键设计。

## 2. SB.Framework.Core 关键机制

### 2.1 全局配置

- `AppSettings`（POCO）：ProductCn/En、DbConfig、CacheConfig、PasswordConfig、JwtAdmin/JwtAPI、UploadConfig、RateLimitConfig、LicenseConfig、CloudConfig、DeployConfig。
- `AppGlobal.AppSettings` 进程级静态持有，`builder.Configuration.Get<AppSettings>()` 一次性赋值。
- `AppTime` 取代 `DateTime.Now`（时区可配，SB0001 强制）。

### 2.2 依赖注入约定

```csharp
[AutoDependencyInjection]          // 标记即 AddScoped
public class MyService : BaseCrudService<MyEntity> { ... }
```

- `ServiceSetup.RegisterServicesFromAssembly` 自动扫描注册。
- **覆盖机制**：子类继承也标了 `[AutoDependencyInjection]` 的基类 → 同时以基类类型注册（工厂 `sp.GetRequiredService(type)`），后注册覆盖先注册。
- **扫描顺序**：框架 Services → feature（字典序）→ 项目程序集（永远最后），保证派生子类覆盖一切。

### 2.3 认证/授权（双 JWT + API Key）

| Scheme | 用途 | 路由 |
|--------|------|------|
| `JwtAdmin` | 管理后台 | `/admin/*` |
| `JwtApi` | 客户端 API（app 端） | `/api/*`（含单点登录校验：查 IMemoryCache 比对 token + Member 状态） |

- **API Key 核心缝**：`IApiKeyAuthenticator.Resolve(context, scheme)` → `ApiKeyAuthResult`。实现由 `SB.Feature.AccessToken` 插件提供；未加载插件时静默退回 JWT。
- **FullAccess（god）key** 旁路 License（取代旧静态 MasterKey，可吊销/审计）。
- **Scope 闸门**：`ScopeAttribute` + `ScopeFilter`（默认 deny，least-privilege）。
- **WebSocket**：JWT 放 query `access_token`，仅 ws 升级请求取用。
- **Claims 扩展**：`Id()`/`Username()`/`UserRole()`/`UserType()`(Admin/Member)/`Role()`(Owner/Agent)。

### 2.4 数据访问层（FreeSql）

- `IFreeSql` 单例注册（MySql 默认 + PostgreSQL）。
- **仓储模式**：`BaseCrudService<T> : BaseRepository<T>`（约束 `T : class, IEntity`）。
- 通用 CRUD：`Detail(id)`、`Del(id)`（自动软删检测 `ISoftDelete`）、`ApplySorting`、`PagedResult`。
- `BaseRepositoryExtensions`：`GetUnitOfWork`/`ExecuteInTransaction`/`GetByIdOrThrow`/`GetPageList`/`WhereIf`/`InsertRange`。
- **全局软删过滤**：Admin/Menu/AdminRole/Member/LicenseClient 自动 `Apply("IsDeleted", ...)`。
- `fsql.UseJsonMap()`；`CodeFirst.SyncStructure` 自动建表（无迁移文件）。
- **慢查询监控**：`Aop.CommandAfter` 阈值默认 200ms → Warning 日志；`QueryCountMiddleware` 统计每请求 SQL 次数。

### 2.5 异常 / 日志 / 缓存

- `GlobalExceptionMiddleware`：未处理异常 → 500 JSON，`ExposeErrorDetails` 控制是否泄露（默认 false）。
- Serilog：NDJSON/CLEF 分级文件（`error-{Date}.ndjson` 等），启动前归档旧日志。
- `ICacheService` 三实现按 `CacheConfig.Enabled` + `Provider` 选择；Redis `AllowAdmin` 支持 `GetStats()` 诊断。

### 2.6 插件框架（★ 最核心复用机制）

三层架构：

| 层 | 接口/组件 | 实例化方式 | 职责 |
|----|-----------|-----------|------|
| 发现 | `FeatureScanner`（读 `.deps.json` 按 `SB.Feature.` 前缀） | — | 产出 `FeatureCatalog` |
| 清单/生命周期 | `IFeatureModule` | `Activator.CreateInstance`（不进 DI） | `ConfigureServices`/`Configure` |
| 数据装配 | `IModuleInitializer` | `GetUninitializedObject`（不调构造） | 建表/菜单/系统设置/种子 |

> 关键设计裁决：清单与数据声明**物理拆类**，因实例化语义不兼容（Activator vs GetUninitializedObject）。

## 3. SB.Framework.Models

纯数据层（只引 FreeSql），`GenerateDocumentationFile=true`。

**基础接口**：
- `IEntity`：`int Id`
- `ISoftDelete`：`bool IsDeleted`（实现后自动软删过滤）
- `IModuleInitializer`：`ModuleName`/`ModuleKey`/`InitOrder`/`GetEntityTypes()`/`GetSystemSettings()`/`GetMenus()`/`SeedData(fsql)`，全部默认实现
- `Result`：统一响应 `{code, message, data}`，`Success`/`Fail`/`Page`
- `MenuDefinition`：菜单声明（ParentPath/Name/Path/Component/Title/Icon/OrderNo/Redirect/HideInMenu/Children）

**核心实体**（`sb_*` 表，20+）：

| 领域 | 实体 |
|------|------|
| RBAC | Admin、AdminRole、AdminRoleMenu、Menu |
| C 端 | Member、MemberRealName |
| 配置 | SystemSetting、SystemSettingCategory |
| 邮件 | EmailConfig、EmailTemplate、EmailLog、EmailVerificationCode |
| 上传 | ChunkUpload、ChunkUploadPart |
| 备份 | DatabaseBackup、BackupRemoteRecord |
| 任务 | BackgroundTaskLog、DeployJob |
| 授权 | LicenseClient、LicenseRecord、LicenseRequest、LicenseVerifyLog |

示例 `Admin`：`[Table(Name="SB_Admin")]`，含 ParentId(树)、Username/Password、RoleId、Status、AuthenticatorKey(2FA)、AccountType(Human/Service)、IsDeleted。

**无 EF DbContext**：`DatabaseSeeder` 从各 `IModuleInitializer.GetEntityTypes()` 收集实体自动建表。

## 4. SB.Framework.Services

**3 个扩展点**（派生项目实现接口注册到 DI）：

| 接口 | 触发时机 | 用途 |
|------|----------|------|
| `IFreeSqlConfigurator` | Seed 阶段，按 `Order` 执行（基座 0，派生 100+） | 配置 FreeSql（GlobalFilter/Mapper/Converter） |
| `IStartupSyncHook` | 启动后（schema 同步 + Seed 完成后、HTTP 监听前），按 Order 串行 | 启动同步任务 |
| `IModuleInitializer` | 数据库装配 | 建表/菜单/设置/种子 |

**核心服务**（30+）：
- 认证：`AdminService`/`MemberService`/`ApiAuthService`/`PermissionService`/`RealNameService`
- 系统：`AppService`/`AppSettingsService`/`SetupService`（安装向导 916 行）/`SystemSettingService`/`DashboardService`
- 基础设施：`CommonService`/`FileUploadService`/`ChunkUploadService`/`SqlService`/`DatabaseBackupService`/`DeployJobService`/`LogViewerService`（含 AI 诊断 QueryDiag）
- 缓存：`ICacheService` + 三实现
- 邮件：`IEmailService` + `EmailProviderFactory` + GoogleSmtp/Resend + `VerificationCodeService`
- 后台任务：`IBackgroundTaskManager` + `BackgroundTaskManager`/`HostedService`/`RegistrationService`
- 授权客户端：`LicenseClientService`/`LicenseVerifyClient`
- 菜单：`MenuService`/`AdminRoleService`/`AdminRoleMenuService`
- 其他：`IQdrantDriver`/`ISpreadsheetReader`(NPOI)/`IRandomNameGenerator`

## 5. SB.Framework.WebApi

### 5.1 一键装配（`SbFrameworkExtensions.cs`）

**`AddSbFramework`** 装配顺序（`[F-*]` 为插件装配点）：
1. 读配置 → `AppGlobal.AppSettings`；归档旧日志；Serilog；UseSystemd
2. `[F-1]` FeatureScanner.Scan → FeatureCatalog
3. FeatureModuleActivator.ActivateAndSort（实例化清单 + 拓扑排序）
4. 收集 `Modules:Disabled` 黑名单
5. License DI（`#if !DISABLE_LICENSE`）
6. **`isInstalled` 门控**（`installed.lock`）：已安装 → FreeSql + Jwt + `[F-2]`业务服务 + 缓存/邮件/Qdrant/后台任务/限流/健康检查 + `[F-3]`feature DI；未安装 → 仅 Authentication + MemoryCache
7. `[F-4]` Controller 跨程序集发现（`mvc.AddApplicationPart(asm)`）
8. `[F-4b]` 移除被禁模块 Controller（`DisabledModuleControllerFilter`）
9. `AdminRoutePrefixConvention` + `ControllerExclusionConvention`
10. `AppTime.Initialize(timeZone)`

**`UseSbFramework`** 管线顺序：
```
HSTS(非Dev) → CORS → RateLimiter → SerilogRequestLogging
→ [License 中间件(已安装+非Dev, 主权闸门最外层)]
→ GlobalException → QueryCount → StaticFiles → SetupMiddleware(安装门控)
→ Routing → [ApiKeyMiddleware] → Authorization → SwaggerDoc
→ [F-5] feature 中间件 → HealthChecks(/health) → MapControllers → 默认路由 → RazorPages
→ SeedDatabase(已安装+非冒烟) → 启动日志
```

**冒烟模式**：`SB_SMOKE_TEST=1` 跳过 DB 迁移与后台任务，只探活 `/health`（部署预检，避免污染生产）。

### 5.2 Controllers（20 个）

- `ApiController` → `/api/[action]`（JwtApi，公开 API）
- 其余 19 个 → `admin/` 前缀（JwtAdmin）：App、Auth、Setup、AppSettings、Manager、Member、Menu、Database、BackgroundTask、LogViewer、Upload、Verification、EmailConfig、Dashboard、SystemSetting 等。
- 路由约定：`AdminRoutePrefixConvention` 自动加 `admin/` 前缀（绝对路由 `/` 或 `~/` 开头除外）。

### 5.3 中间件

| 中间件 | 作用 |
|--------|------|
| `SetupMiddleware` | 未安装时拦截 → API 503 `setup_required` / 浏览器 302 `/setup/index.html` |
| `ApiKeyMiddleware` | 全局把有效 sk_ key 设为归属账号身份，`/agent/*` 与 `/admin/*` 统一 Bearer。`ApiKeyDeny.Add(path, denyFor)` 可扩展禁止路径 |

### 5.4 配置结构（appsettings 关键节）

```
TimeZone / ProductCn / ProductEn / Debug / Serilog
DbConfig (DbType/Server/Port/Database/User/Password/AutoSyncStructure/SlowQueryThresholdMs)
CacheConfig (Enabled/Provider:Memory|Redis/Redis.Connection)
PasswordConfig (StorageMode: Encrypted|PlainText)
UploadConfig / RateLimitConfig / CorsConfig
JwtAdmin / JwtApi (Key/Issuer/Audience/Expires)
DeployConfig.Projects / ProjectAssemblyName / ExcludeControllers
PrimaryDomain / LicenseConfig.Key / CloudConfig.ServerUrl
Features:Disabled (黑名单) + Features:{FeatureName} (插件私有配置)
Modules:Disabled (核心模块黑名单)
```

## 6. SB.FlowEngine（流程引擎）

独立 NuGet 包，只依赖 Core，是前端 `@saibo/flow-engine` 的服务端对称实现——前后端同一张图跑出相同结果。

- **FlowGraph**：`Version`/`Nodes`/`Edges`/`Variables`/`Entry`；FlowNode 含 `ModuleId` + `Config`(Dictionary)；FlowEdge `Kind`("exec"|"data")。
- **双线模型**：执行指针沿 exec 线走，数据沿 data 线拉取求值（只读缓存）。
- **内置控制节点**：`if`/`loop`/`goto`/`label`/`varGet`/`varSet`/`wait`/`subFlow`。
- **业务节点接入**：实现 `IFlowNodeExecutor`（`ModuleId` + `ExecuteAsync(FlowNodeContext, ct)`）。
- **节点级控制**：`retry`/`retryIntervalMs`/`stopOnError`/`delayMs`；`MaxSteps` 防死循环；子流程深度上限。
- **边界**：引擎只做图执行 + 契约 + 接口 + 内置控制节点；业务节点/Controller/图持久化由业务层负责。

## 7. SB.Feature.* 插件（范式三件套）

统一范式：**Feature（清单薄壳）+ Initializer（声明式数据）+ Service（注入式逻辑）+ Controller + Options + Models**。

### 7.1 SB.Feature.AccessToken（v0.2.3）

统一访问令牌：账号即 token（密码 / API Key 两种认证），scoped key + FullAccess(god) 取代静态 MasterKey。

- `ApiKey`（表 `SB_ApiKey`）：AccountId、TokenHash(sha256)、TokenPrefix、Scopes(RBAC)、FullAccess、IpAllowlist、ExpiresAt、RateLimitPerMinute。明文只签发时回显一次。
- 8 个 Service：`ApiKeyAuthenticator`（核心缝实现）、`ApiKeyService`、`ApiKeyAuditService`、`ApiKeyRateLimiter`、`ApiKeyContext`、`ScopeFilter`、`ServiceAccountService`、`ApiKeyAuditCleanupService`。

### 7.2 SB.Feature.BackupReceive（v0.1.1）

服务端接收客户端 multipart 上传 → 落盘 `{StoragePath}` → 入库 `BackupRemoteRecord`。不暴露删除接口。

- `BackupReceiveService : BaseCrudService<BackupRemoteRecord>`（**要基类**范式）。

### 7.3 SB.Feature.LicenseManage（v0.1.1）

服务端授权签发端（管理客户/审批/签发/吊销）。

- `LicenseManageService`（789 行，**不继承 BaseCrudService**，直接 `IFreeSql.Select<>`）—— 覆盖「不要基类」范式。
- RSA 密钥目录优先级：`SB_LICENSE_KEY_DIR` 环境变量 > Options.KeyDirectory > ContentRoot。
- 典型部署在 **NoLicense 主包**上（自己不被授权束缚 + 同时能给下游签发）。

## 8. SB.Framework.Analyzers

netstandard2.0 Roslyn 分析器（DLL 显式打到 `analyzers/dotnet/cs/`，否则不生效）。

| 规则 | 内容 |
|------|------|
| SB0001 | 禁 `DateTime.Now`/`Today`/`UtcNow`，用 `AppTime.*`（附自动修复 CodeFix） |
| SB0002 | 禁 `[FromBody]`，除非标 `[AllowJsonBody]`；admin 默认 `[FromForm]`（防 Content-Type 不一致致参数 null） |
| SB0010 | 表前缀 `sb_` 检查（规划中，未实现） |

## 9. scripts 与 docs

- `scripts/`：`sb-build.sh`（构建）、`pack-push.sh`（推 NuGet）、`deploy-via-api.sh`（零宕机部署）
- `docs/`：`AI_Index.md`（索引）、`AI_Conventions.md`（写码必读）、`AI_Plugin_Architecture.md`（插件架构权威）、`AI_API_Reference.md`（24 个 Controller 全接口）、`AI_Downstream_Guide.md`（下游接入指南）、`naming-conventions.md`（命名规范）等。**均已按当前源码（.NET 10 / NuGet 消费 / AccessToken）校正**，可直接引用。

## 10. 新项目后端接入速查

```csharp
// 1. csproj 引用（配 NuGet 源 saibo-baget + nuget.org）
<PackageReference Include="SB.Framework.WebApi" Version="0.4.14" />

// 2. Program.cs 两行装配
builder.AddSbFramework();
app.UseSbFramework();

// 3. 业务实体
[Table(Name = "sb_<app>_<entity>")]
public class MyEntity : IEntity, ISoftDelete { public int Id {get;set;} public bool IsDeleted {get;set;} ... }

// 4. 业务服务（自动注册 + 继承基类获得通用 CRUD）
[AutoDependencyInjection]
public class MyEntityService : BaseCrudService<MyEntity> { ... }

// 5. 控制器（自动加 admin/ 前缀）
[ApiController, Authorize(AuthenticationSchemes = "JwtAdmin"), Route("[controller]/[action]")]
public class MyEntityController : ControllerBase { ... }

// 6. 模块数据装配（建表/菜单/设置/种子）
public class MyModuleInitializer : IModuleInitializer { public int InitOrder => 100; ... }
```
