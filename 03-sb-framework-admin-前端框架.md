# 03 sb-framework-admin 前端框架深度分析

> 位置：`D:\projects\saibo\framework\sb-framework-admin`
> 这是 **Vben Admin v5.5.9**（`vbenjs/vue-vben-admin`）的赛博科技定制 fork，作为所有派生应用的前端基座。唯一应用 `apps/web-antd`（Ant Design Vue），端口 6012。

## 1. 工程形态

### 1.1 环境要求

| 项 | 要求 |
|----|------|
| Node | >=20.12.0（`.node-version`: 22.1.0） |
| pnpm | >=10.0.0（实测 `packageManager: pnpm@10.22.0`） |
| 语言 | `"type": "module"`（全 ESM） |

### 1.2 pnpm 配置要点

- `.npmrc`：`public-hoist-pattern`（lefthook/eslint/prettier/postcss 等）、`strict-peer-dependencies=false`、`auto-install-peers=true`、`dedupe-peer-dependents=true`。
- `package.json`：`overrides`（vue/pinia/esbuild 等强制版本）、`neverBuiltDependencies: ["canvas","node-gyp"]`、`preinstall: only-allow pnpm`、`postinstall: pnpm -r run stub --if-present`。
- **pnpm catalog**（`pnpm-workspace.yaml`）是版本统一核心机制，子包用 `"xxx": "catalog:"` 引用。

### 1.3 monorepo 工作区（pnpm-workspace.yaml）

```
internal/*  internal/lint-configs/*  packages/*  packages/@core/base/*
packages/@core/ui-kit/*  packages/@core/forward/*  packages/@core/*
packages/effects/*  packages/business/*  apps/*  scripts/*  docs  playground
```

### 1.4 turbo 任务管道（turbo.json）

- `build`: `dependsOn: ["^build"]`，输出 `dist/**`
- `dev`: 无依赖、`cache: false`、`persistent: true`
- `typecheck`: 仅输出校验；还有 `test:e2e`、`preview`、`build:analyze`
- globalDependencies: `pnpm-lock.yaml`、`**/.env.*local`、`**/tsconfig*.json`、`internal/*` 源码

### 1.5 常用脚本

```bash
pnpm dev            # 启动开发（antd 端口 6012）
pnpm build          # 构建
pnpm lint / format  # ESLint / Prettier
pnpm check          # 循环依赖 + 依赖 + 类型 + cspell
pnpm test:unit      # vitest
pnpm commit         # czg 交互式提交
```

### 1.6 代码规范 & AI 协作约定（重要）

- **`AGENTS.md`**：AI agent 指南。monorepo 依赖规则：`@vben-core/*` 不能依赖 `@vben/*`；`@vben-core/shared` 不能依赖任何 `@vben/*`。
- **`CLAUDE.md`**：端口约定（6012、`/storage`→6010、`/api`→mock 5320）+ 六大高频坑。
- **`AI_Conventions.md`**（484 行，最权威规范，主源在 `sb-api/docs/`）：toast/UI/颜色/时间/后端规范/后台任务/系统设置 19 种 DisplayType/CloudConfig/备份/品牌/安装向导/构建/提交 + 自研可复用资产清单。
- **`CONTRIBUTING.md`**：基座「三不准」（不准派生业务、不准硬编码颜色/品牌、不准文字链操作）。
- **CI**：`lint:naming` 用 grep 查硬编码十六进制色 + 禁止 `views/projects|requirements|tasks|mind-maps` 派生业务路由进入基座。

## 2. apps/web-antd（唯一应用）

### 2.1 启动链

```
index.html → src/main.ts (initApplication: 组装 namespace → initPreferences → 动态 import bootstrap)
→ src/bootstrap.ts:
    initComponentAdapter (antd 表单组件注册)
    initSetupVbenForm (表单 schema 配置)
    fetch /storage/app-info.json (后端安装向导生成的品牌配置) → window.__APP_INFO__ → updatePreferences
    setupI18n → initStores → registerAccessDirective → router → 动态标题
→ src/app.vue (antd ConfigProvider + darkAlgorithm/defaultAlgorithm/compactAlgorithm)
```

### 2.2 环境变量

| 文件 | 关键值 |
|------|--------|
| `.env` | `VITE_APP_TITLE=管理系统`、`VITE_APP_NAMESPACE=vben-web-antd` |
| `.env.development` | `VITE_PORT=6012`、`VITE_GLOB_API_URL=http://localhost:6010/admin`、`VITE_NITRO_MOCK=false` |
| `.env.production` | `VITE_GLOB_API_URL=/admin`、`VITE_ROUTER_HISTORY=hash`、`VITE_ARCHIVER=true` |

> `apiURL` 经 `useAppConfig` 读取：dev 读 `import.meta.env.VITE_GLOB_API_URL`，prod 读 `window._VBEN_ADMIN_PRO_APP_CONF_`（由 vite 插件注入）。

### 2.3 Vite 代理

```ts
'/api':      → http://localhost:5320/api (mock，ws: true)
'/storage':  → http://localhost:6010 (后端品牌资源)
```

### 2.4 前端目录分层

```
src/
  adapter/      组件适配层（component、form、vxe-table）
  api/          统一请求封装（request.ts + core/ 各领域模块）
  bootstrap.ts  应用装配
  components/   自研业务组件（TimeTag 等）
  composables/  usePermission
  directives/   v-permission
  layouts/      basic.vue / auth.vue
  locales/      i18n（zh-CN、en-US）
  router/       index / guard / access / routes
  store/        auth.ts（登录/登出/用户信息）
  types/        admin.ts（AdminType 枚举）
  utils/        permission.ts（PermissionUtil）
  views/        _core + 业务模块
```

## 3. packages/ 内部包职责

### 3.1 `packages/@core/*`（最底层，禁放业务逻辑）

| 包 | 职责 |
|----|------|
| `@core/base/design` | 设计 token（shadcn 体系 `--primary/--muted/--destructive/--card/--popover/--border/--radius`）+ scss-bem |
| `@core/base/icons` | 基础图标 |
| `@core/base/shared` | 最底层工具：cache、color、constants、global-state、store、utils（cn/date/diff/tree/merge/nprogress/to...） |
| `@core/base/typings` | 全库类型 + **RouteMeta 扩展**（authority/hideInMenu/keepAlive/iframeSrc/badge 等） |
| `@core/composables` | use-is-mobile / use-namespace / use-layout-style 等 |
| `@core/preferences` | 偏好设置核心（config / preferencesManager / update-css-variables） |
| `@core/ui-kit/*` | form-ui、menu-ui、popup-ui、shadcn-ui（24 组件）、tabs-ui、layout-ui |

### 3.2 `packages/effects/*`（@vben/*，应用直接引用）

| 包 | 职责 |
|----|------|
| `@vben/access` | 权限路由生成 `generateAccessible`（backend/frontend/mixed 三模式）+ `v-access` 指令 + `hasAccessByRoles/Codes` |
| `@vben/common-ui` | Page/ColPage/CountTo/EllipsisText/JsonViewer/Tree/ApiComponent/Captcha 等 |
| `@vben/hooks` | use-app-config（apiURL）、use-design-tokens（antd/naive/element 三套 token 映射）、use-watermark、use-tabs、use-refresh、use-pagination |
| `@vben/layouts` | BasicLayout、Authentication 布局、widgets（breadcrumb/search/lock/preferences/user-dropdown/theme-toggle） |
| `@vben/plugins` | echarts、motion、**vxe-table**（`useVbenVxeGrid`、格式化、CellImage/CellLink） |
| `@vben/request` | **HTTP 请求核心**（RequestClient）|

### 3.3 业务/基础包（packages/*）

| 包 | 职责 |
|----|------|
| `@vben/constants` | `LOGIN_PATH`、`SUPPORT_LANGUAGES` |
| `@vben/icons` | IconifyIcon + 本地 svg |
| `@vben/locales` | vue-i18n 实例 + `setupI18n`/`$t` |
| `@vben/preferences` | `defineOverridesPreferences` + re-export |
| `@vben/stores` | Pinia stores：access/user/tabbar/timezone/sound-settings（**SecureLS AES 加密持久化**） |
| `@vben/styles` | 全局样式（含 antd 专用 `@vben/styles/antd`） |
| `@vben/types` | UserInfo 扩展、RouteMeta 扩展、全局类型 |
| `@vben/utils` | generate-menus、generate-routes-backend、generate-routes-frontend、merge-route-modules |

## 4. internal/ 内部工具

`lint-configs`（commitlint/eslint 18 配置组合/prettier/stylelint）、`node-utils`、`tailwind-config`、`tsconfig`（base/web/web-app/library/node）、`vite-config`（defineConfig 插件集 + loadAndConvertEnv）。

## 5. 技术栈与权限模型

| 类别 | 选型 |
|------|------|
| 框架 | Vue 3.5 + Vite 7 + TS 5.9 |
| 状态/路由 | Pinia 3（SecureLS AES 加密持久化）+ Vue Router 4（prod hash 模式） |
| HTTP | axios 1.10（@vben/request RequestClient） |
| UI | Ant Design Vue 4.2 + 自研 shadcn 组件 + vxe-table 4.17 |
| i18n | vue-i18n 11 |
| 样式 | Tailwind 3.4 + 双 token 体系 |

### 权限模型（mixed 混合模式）

```
preferences.app.accessMode = 'mixed'
┌─ 前端角色过滤 (frontend) ─────────────────┐
│ meta.authority 角色判断 + 按钮级 v-permission │
└──────────────────────────────────────────┘
┌─ 后端菜单驱动 (backend) ───────────────────┐
│ GET /admin/Menu/All → RouteRecordStringComponent[] │
│ → generateAccessible('mixed', ...) 动态挂路由 │
└──────────────────────────────────────────┘
```

- **角色模型**：`AdminType` 枚举 `SystemAdmin=0 / RegularAdmin=1 / SiteOwner=2 / CustomerService=3`；数据范围 `all/self/subordinate`。
- **按钮权限**：`usePermission()` + `v-permission="'xxx:yyy'"`（无权限移除 DOM）。
- **页面组件映射**：`pageMap = import.meta.glob('../views/**/*.vue')` + layoutMap。

## 6. 与后端对接方式（★ 派生项目写接口必读）

### 6.1 请求封装（`apps/web-antd/src/api/request.ts`）

**baseURL**：dev `http://localhost:6010/admin`，prod `/admin`（不含 `/admin` 前缀，后端自动加）。

**API 语义**：

| 写法 | 行为 |
|------|------|
| `api.get(url, {params})` | 静默查询（无 loading/toast），返回解包后的 `data` |
| `api.post(url, data)` | 带 loading + 成功 toast（`code===0` 自动 `message.success`），失败抛异常 |
| `api.post(url, data, {quiet:true})` | 静默 POST（登录/上传） |
| `api.post(url, data, {silent:true})` | 完全静默，调用方自己 try/catch |
| `api.raw.post(...)` | 原始请求（refreshToken/logout 用 `_rawClient`） |

**请求拦截器**：
- Content-Type：普通对象 → `application/x-www-form-urlencoded`；`FormData` → 删除 Content-Type（让浏览器自动加 multipart boundary）—— 后端全用 `[FromForm]`
- 注入 `Authorization: Bearer <token>` + `Accept-Language: preferences.app.locale`

**响应拦截器（按顺序）**：
1. `defaultResponseInterceptor`：`{code,data}` body 解包，`successCode:0`
2. `authenticateResponseInterceptor`：401 队列式刷新 token，失败 → 登出/弹登录过期
3. **503 + `type:'setup_required'` → 跳 `#/setup`**
4. **403 + `type:'license_invalid'` → 跳 `/auth/license`**
5. `errorMessageResponseInterceptor`：网络/超时/HTTP 错误 → 自动 `message.error`

### 6.2 登录流程

```
POST /Auth/Login {username,password,totpCode} (quiet)
→ accessToken → accessStore.setAccessToken
→ GET /Auth/GetInfo → userStore.setUserInfo
→ 跳 userInfo.homePath || '/profile'
支持 TOTP（Google 验证器绑定 /Auth/GetGoogleCodeQR）
```

### 6.3 安装向导（无认证 fetch 直连）

`setupApiURL = apiURL.replace(/\/admin\/?$/, '/api')`，`fetch` 直连 `Setup/Status`、`Setup/TestDatabase`、`Setup/TestRedis`、`Setup/Install`。路由守卫最高优先级检测安装状态。

### 6.4 枚举/字典

- 前端枚举模式：`AdminType` + `AdminTypeLabels` + getter 函数（`types/admin.ts`）。
- 后端系统设置：19 种 `displayType` → `views/system-setting/SettingRender.vue` 映射 antd 组件，选项配置存 `dataContentConfig` JSON。

### 6.5 文件上传（智能上传体系）

`GET /Upload/Config` 拉后端配置 → 三组件 `SmartUpload`（按大小自动选直传/分片）/`DirectUpload`/`ChunkUpload`；分片：`ChunkInit`(instant 秒传/new/resume) → `ChunkUpload`(XHR 进度 + Bearer) → `ChunkMerge`；MD5 用 spark-md5。

## 7. 新项目前端开发约定

### 7.1 目录组织

```
views/<module>/
  api.ts            # 该模块的 API 函数
  List.vue          # 列表页（vxe-grid + formOptions 查询表单）
  Form.vue          # 表单（vben-form + drawer）
  (+ Tree.vue / table-data.ts)
```

### 7.2 命名

- views 目录：kebab-case 复数；组件文件/目录：PascalCase.vue；API 函数 `getXxx/postXxx`；常量 SCREAMING_SNAKE_CASE。
- 颜色：shadcn `hsl(var(--muted))` / antd `var(--ant-color-*)`，**禁止 `#xxx`/`rgb()`**（CI 检查）。
- 时间：统一 `TimeTag`/`TimeProgress`（按 appTimeZone 格式化，后端 UTC）。
- toast：不写与 API 配对的手动 message（拦截器统一弹）；纯前端操作/校验/silent 例外。

### 7.3 表格标准（vxe-grid）

```ts
const [gridApi, gridOptions] = useVbenVxeGrid({
  formOptions: { schema: [查询表单字段] },
  gridOptions: {
    columns,
    proxyConfig: { ajax: { query: (params) => getList(params) } }, // 返回 {items,total}
  },
});
```

### 7.4 新增页面/菜单标准流程

1. `views/<module>/` 建 List.vue + Form.vue + api.ts
2. **菜单注册走后端**：后端 `IModuleInitializer.GetMenus()` → `SB_Menu` 表 → 前端 `/Menu/All` 动态生成（无需手改前端路由）
3. 按钮权限：后端菜单项 `authCode` + 前端 `v-permission`
4. 词包：`locales/langs/zh-CN|en-US/*.json`
5. 复用组件：跨页 ≥2 处抽到 `components/`，同步更新 `AI_Conventions.md §13.1`

### 7.5 新增 API 模块

```ts
import { api } from '#/api/request';

export async function getList(params: any) {
  return api.get('/Xxx/List', { params });
}
export async function postAdd(data: any) {
  return api.post('/Xxx/Add', data);
}
```

> URL 不需要写 `/admin` 前缀（baseURL 已含）；FormData 自动处理 Content-Type。

## 8. 关键扩展点速查

| 需求 | 改哪里 |
|------|--------|
| 品牌名/Logo | 后端 `wwwroot/storage/app-info.json`（前端 bootstrap 自动读取） |
| 默认首页 | `apps/web-antd/src/preferences.ts` `defaultHomePath` |
| 请求 baseURL | `.env.development/.env.production` `VITE_GLOB_API_URL` |
| 登录接口 | `apps/web-antd/src/api/core/admin.ts` |
| 菜单接口 | `apps/web-antd/src/api/core/menu.ts` |
| 上传配置 | `api/core/upload.ts` + `components/Upload/` |
| 权限码表 | `apps/web-antd/src/utils/permission.ts`（硬编码表） |
| 主题色 | `packages/@core/base/design/src/design-tokens/default.css` |
| 时区 | `packages/stores/src/modules/timezone.ts` 的 `setTimezoneHandler` |
