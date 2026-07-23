# MXU 开发者与 AI 指南 (AGENTS.md)

本文档为 MXU 项目的开发规范与架构指南，旨在为人类开发者及 AI 辅助工具提供统一的代码理解与协作标准。

## 1. 项目概述

**MXU** 是一款基于 [MaaFramework ProjectInterface V2](https://github.com/MaaXYZ/MaaFramework/blob/main/docs/zh_cn/3.3-ProjectInterfaceV2协议.md) 协议的通用图形界面客户端。

- **核心目标**：实现对 MaaFramework 生态项目的零配置/低配置自动化支持。
- **技术选型**：
  - **Frontend**: React 19 + TypeScript + Vite 7 + Tailwind CSS 4 + Zustand 5 + i18next
  - **Backend**: Tauri 2 (Rust) + Axum 0.7 (HTTP/WebSocket 服务)
  - **包管理**: pnpm 10
  - **格式化**: Prettier (TS) + cargo fmt (Rust)

## 2. 目录架构索引

```
src/
├── components/          # UI 组件库（~38 个组件）
│   ├── index.ts         # 统一导出（Barrel Export 模式）
│   ├── app/             # App 级浮层（LoadingScreen, VersionWarning...）
│   ├── connection/      # 连接逻辑（hooks + callbackCache）
│   ├── settings/        # 设置页各 Section
│   ├── toolbar/         # 工具栏按钮
│   ├── ui/              # 原子 UI 组件（基于 Radix UI）
│   └── [30+ .tsx]       # 业务组件（TaskItem, DashboardView, TabBar...）
├── stores/              # Zustand 单一 Store
│   ├── appStore.ts      # 主 Store（~2200 行）
│   ├── types.ts         # AppState 接口定义（~500 行）
│   └── helpers.ts       # 纯函数辅助（ID 生成、选项校验）
├── services/            # 核心业务逻辑层
│   ├── maaService.ts    # MaaFramework API 封装（Tauri invoke / HTTP REST）
│   ├── configService.ts # 配置持久化（mxu.json 读写，debounce）
│   ├── interfaceLoader.ts # PI V2 接口加载与 import 合并
│   ├── updateService.ts # 自动更新（⚠️ 极度审慎修改）
│   ├── wsService.ts     # WebSocket 服务
│   └── ...              # 其他服务（cache, proxy, schedule, taskMonitor 等）
├── types/               # TypeScript 类型定义
│   ├── interface.ts     # PI V2 协议类型（ControllerItem, TaskItem, OptionDefinition...）
│   ├── specialTasks.ts  # MXU 特殊任务定义（7 个 Custom Action）
│   ├── config.ts        # mxu.json 配置类型（MxuConfig, AppSettings...）
│   └── maa.ts           # MaaFramework 运行时类型（AdbDevice, ControllerConfig...）
├── utils/               # 通用工具函数
│   ├── pipelineOverride.ts  # Pipeline Override 生成（选项→JSON）
│   ├── optionHelpers.ts     # 选项值辅助函数
│   ├── backendApi.ts        # 浏览器 HTTP API 工具（apiGet/apiPost）
│   ├── paths.ts             # 路径工具（跨平台数据目录）
│   ├── logger.ts            # 统一日志（loglevel，模块化）
│   ├── windowUtils.ts       # 窗口操作工具
│   └── ...                  # 其他工具（jsonc, color, cdkCrypto 等）
├── themes/              # 主题系统（CSS 变量注入）
│   ├── index.ts         # 主题管理器
│   └── presets/         # 主题预设（light/dark + 9 个强调色）
├── i18n/                # 国际化
│   ├── index.ts         # i18next 配置
│   └── locales/         # 语言包（zh-CN, zh-TW, en-US, ja-JP, ko-KR）
├── hooks/               # 自定义 Hooks
│   └── useIsMobile.ts
└── App.tsx              # 主应用组件（~2000 行，路由、初始化、事件系统）

src-tauri/
├── src/
│   ├── main.rs          # 二进制入口（CLI、WebView2 检测、管理员提权）
│   ├── lib.rs           # Tauri App 构建（插件注册、~60 个命令路由）
│   ├── web_server.rs    # Axum HTTP + WebSocket 服务器（WebUI 后端）
│   ├── mxu_actions.rs   # 7 个 MXU Custom Action 实现
│   ├── screenshot_service.rs # 实例截图循环
│   ├── dummy_controller.rs   # 空控制器（返回纯黑 PNG）
│   ├── commands/        # Tauri 命令集（maa_core, maa_agent, file_ops, download, update, system, app_config, state）
│   └── webview2/        # WebView2 检测与安装
├── capabilities/default.json # Tauri 权限配置
├── Cargo.toml           # Rust 依赖
└── tauri.conf.json      # Tauri 配置
```

## 3. 核心架构模式

### 3.1 ProjectInterface V2 数据流

```
interface.json → interfaceLoader.ts → Zustand Store
                                        ├── pipelineOverride.ts → pipeline_override JSON
                                        ├── specialTasks.ts → MXU 内置虚拟任务
                                        └── maaService.ts → Tauri invoke / HTTP API
                                                                    ↓
                                                        maa_core.rs → MaaFramework C 库
```

### 3.2 双模式运行（Tauri Desktop + Browser WebUI）

| 特性     | Tauri 模式                       | WebUI 模式                   |
| -------- | -------------------------------- | ---------------------------- |
| 后端调用 | `invoke()` Tauri 命令            | HTTP REST (`apiGet/apiPost`) |
| 配置存储 | `mxu.json` 文件                  | `localStorage` + 后端缓存    |
| 实时事件 | Tauri `listen()`                 | WebSocket                    |
| 检测函数 | `isTauri()` (`window.__TAURI__`) | 同                           |

### 3.3 Zustand Store 模式

- **单一 Store**：所有状态集中在 `appStore.ts`，使用 `subscribeWithSelector` 中间件
- **配置持久化**：手动通过 `configService.ts` + debounce，**不用** Zustand 的 `persist`
- **消费方式**：精确 selector 订阅，多字段用 `useShallow`

```tsx
const theme = useAppStore((s) => s.theme);
const { theme, accent } = useAppStore(
  useShallow((s) => ({ theme: s.theme, accent: s.accentColor })),
);
```

### 3.4 Pipeline Override 生成

PI V2 选项通过 `src/utils/pipelineOverride.ts` 转换为 MaaFramework JSON：

- **优先级**（低→高）：`global_option` < `resource.option` < `controller.option` < `task.option`
- **select**: `{ "pipeline": { "key": "value" } }`
- **switch**: `{ "pipeline": { "key": true/false } }` + case 分支
- **checkbox**: 多选，每个勾选项独立 override
- **input**: 占位符替换 `{param}` → 值，支持 int/bool 类型转换
- 最终以 JSON Array 格式发送（前端深度合并后包装）

### 3.5 MXU 特殊任务系统

7 个内置 Custom Action 任务（`src/types/specialTasks.ts`）：

| 任务            | Action                 | 用途                |
| --------------- | ---------------------- | ------------------- |
| `MXU_SLEEP`     | `MXU_SLEEP_ACTION`     | 倒计时等待          |
| `MXU_WAITUNTIL` | `MXU_WAITUNTIL_ACTION` | 等到指定时间        |
| `MXU_LAUNCH`    | `MXU_LAUNCH_ACTION`    | 启动外部程序        |
| `MXU_WEBHOOK`   | `MXU_WEBHOOK_ACTION`   | 发送 HTTP Webhook   |
| `MXU_NOTIFY`    | `MXU_NOTIFY_ACTION`    | 系统通知            |
| `MXU_KILLPROC`  | `MXU_KILLPROC_ACTION`  | 终止进程            |
| `MXU_POWER`     | `MXU_POWER_ACTION`     | 关机/重启/睡眠/熄屏 |

设计：`target: [0,0,1,1]` + `skipScreenshot: true`，选项通过 `pipeline_override.custom_action_param` 传参。添加新任务参见 `docs/add-special-task.md`。

### 3.6 主题系统

- `src/themes/index.ts` 通过 CSS 自定义属性（`--color-bg-primary`, `--color-accent` 等）注入主题
- 支持 light/dark 模式 + 9 个内置强调色 + 自定义强调色
- Tailwind CSS v4 通过 `@theme` 映射 CSS 变量到 utility classes

### 3.7 组件模式

- **懒加载**：重型组件用 `React.lazy` + 动态 `import()`
- **Drag & Drop**：`@dnd-kit/sortable`，仅垂直拖拽
- **右键菜单**：自定义 `ContextMenu` + `useContextMenu` hook
- **表单控件**：`FormControls.tsx` 导出统一的 Switch/Text/Number/File/Time Input + Field 包装器
- **CSS**：Tailwind utility-first + `clsx` 条件类名 + CSS 自定义属性

## 4. 开发准则与最佳实践

### 4.1 路径别名

```typescript
"@/*" → "./src/*"  // tsconfig.json + vite.config.ts
```

### 4.2 国际化 (i18n)

- **严禁硬编码**：所有面向用户的文本必须在 `src/i18n/locales/` 中定义
- **5 种语言**：`zh-CN`, `zh-TW`, `en-US`, `ja-JP`, `ko-KR`，新增文本需同步更新
- **interface.json 翻译**：使用 `getInterfaceLangKey()` 映射（`zh-CN` → `zh_cn`）

### 4.3 代码质量

- **DRY 原则**：优先复用 `src/components/ui/` 原子组件和 `src/utils/` 工具函数
- **格式化**：Prettier (TS/TSX) + cargo fmt (Rust)，通过 `lint-staged` + Husky 在提交时自动执行
- **日志**：使用 `loggers.xxx` 模块化日志器（基于 loglevel），禁止 `console.log`

### 4.4 安全性

- **更新系统**：涉及 `updateService.ts` 及 Rust 端下载指令的修改需极度审慎
- **权限管理**：系统资源访问需检查 `src-tauri/capabilities/default.json`
- **Win32 API**：优先使用 WinSafe Nightly 封装的安全 API
- **CDK 加密**：使用 XOR + Base64，内存中明文，持久化时加密

### 4.5 状态管理

- **单一事实来源**：业务状态托管于 `src/stores/`，组件通过 Selector 消费
- **配置防抖保存**：`configService.ts` 使用 debounce，`markSelfSave()`/`consumeSelfSave()` 防止配置变更事件循环

### 4.6 安全注意事项

- `updateService.ts` 失效会导致用户无法修复软件
- WebView2 每次启动检测，使用固定版本缓存在 `cache/webview_data/`
- Release 构建尝试管理员提权（ShellExecuteEx runas），用户可取消
- 截图流由后端 `ScreenshotService` 驱动，前端仅订阅/取消订阅

## 5. 开发手册

以下手册提供特定开发场景的详细指引，**按需读取**：

| 场景              | 文档                                                 | 何时阅读                                |
| ----------------- | ---------------------------------------------------- | --------------------------------------- |
| 新增 MXU 特殊任务 | [docs/add-special-task.md](docs/add-special-task.md) | 添加基于 Custom Action 的内置功能任务时 |
| AI 深度开发参考   | [docs/vibecoding-guide.md](docs/vibecoding-guide.md) | 需要全面了解项目架构、模式、约定时      |

## 6. 关键文件速查

| 需求              | 文件                                  |
| ----------------- | ------------------------------------- |
| PI V2 类型定义    | `src/types/interface.ts`              |
| MXU 特殊任务      | `src/types/specialTasks.ts`           |
| 配置类型          | `src/types/config.ts`                 |
| MaaFramework 类型 | `src/types/maa.ts`                    |
| Zustand Store     | `src/stores/appStore.ts`              |
| Store 类型        | `src/stores/types.ts`                 |
| MaaFramework 服务 | `src/services/maaService.ts`          |
| 配置持久化        | `src/services/configService.ts`       |
| PI 加载器         | `src/services/interfaceLoader.ts`     |
| Pipeline Override | `src/utils/pipelineOverride.ts`       |
| HTTP API 工具     | `src/utils/backendApi.ts`             |
| 日志系统          | `src/utils/logger.ts`                 |
| 主题管理          | `src/themes/index.ts`                 |
| i18n 配置         | `src/i18n/index.ts`                   |
| 主应用            | `src/App.tsx`                         |
| Vite 配置         | `vite.config.ts`                      |
| Tauri 配置        | `src-tauri/tauri.conf.json`           |
| Cargo 配置        | `src-tauri/Cargo.toml`                |
| 权限配置          | `src-tauri/capabilities/default.json` |
| MaaFramework 命令 | `src-tauri/src/commands/maa_core.rs`  |
| Custom Actions    | `src-tauri/src/mxu_actions.rs`        |
| Axum 服务器       | `src-tauri/src/web_server.rs`         |
| App 构建          | `src-tauri/src/lib.rs`                |

## 7. 构建与开发命令

```bash
pnpm dev          # Vite 开发服务器 (port 1420)
pnpm build        # TypeScript 检查 + Vite 构建
pnpm tauri dev    # Tauri 开发模式（热重载）
pnpm tauri build  # 生产包构建
pnpm format       # Prettier 格式化
pnpm format:check # Prettier 检查
pnpm format:rust  # cargo fmt
```

## 8. 相关资源

- [MaaFramework Core](https://github.com/MaaXYZ/MaaFramework)
- [ProjectInterface V2 协议文档](https://github.com/MaaXYZ/MaaFramework/blob/main/docs/zh_cn/3.3-ProjectInterfaceV2协议.md)
- [Tauri V2 Documentation](https://tauri.app/v2/)
- [Zustand Guide](https://zustand-demo.pmnd.rs/)
- [WinSafe Nightly Documentation](https://rodrigocfd.github.io/winsafe/winsafe/)
