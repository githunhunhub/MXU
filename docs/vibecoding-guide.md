# MXU Vibe Coding 指南

本文档为 AI 辅助开发（Vibe Coding）提供项目深度参考，帮助 AI 快速理解架构、模式和约定。

---

## 1. 技术栈速查

| 层面 | 技术 | 版本 |
|------|------|------|
| 前端框架 | React | 19.1 |
| 语言 | TypeScript | 5.8 (strict mode) |
| 构建工具 | Vite | 7.0 |
| CSS | Tailwind CSS | 4.1 (v4, `@tailwindcss/vite` 插件) |
| 状态管理 | Zustand | 5.0 |
| 国际化 | i18next + react-i18next | 25.x / 16.x |
| 后端框架 | Tauri | 2.x |
| 后端语言 | Rust | 2021 edition |
| Web 服务 | Axum | 0.7 |
| 包管理 | pnpm | 10.28 |
| 格式化 | Prettier | 3.5 (src), cargo fmt (Rust) |
| Git Hooks | Husky + lint-staged | 9.x / 16.x |

---

## 2. 项目架构全景

```
┌─────────────────────────────────────────────────────────────────┐
│                    MXU (MaaFramework PI V2 Client)               │
├──────────────────────────────┬──────────────────────────────────┤
│       Frontend (React)       │       Backend (Tauri/Rust)       │
├──────────────────────────────┼──────────────────────────────────┤
│  src/                        │  src-tauri/src/                  │
│  ├── components/ (38 files)  │  ├── main.rs (binary entry)      │
│  ├── stores/ (Zustand)       │  ├── lib.rs (app builder)        │
│  ├── services/ (13 files)    │  ├── web_server.rs (Axum HTTP/WS)│
│  ├── types/ (4 files)        │  ├── mxu_actions.rs (Custom Act) │
│  ├── utils/ (18 files)       │  ├── screenshot_service.rs       │
│  ├── themes/                 │  ├── dummy_controller.rs         │
│  ├── i18n/                   │  ├── tray.rs                     │
│  ├── hooks/                  │  ├── commands/ (14 files)        │
│  └── App.tsx (1985 lines)    │  └── webview2/ (3 files)         │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 3. 核心概念：ProjectInterface V2

MXU 的核心是解析和渲染 MaaFramework 的 `interface.json`（PI V2 协议）。

### 3.1 interface.json 数据流

```
interface.json (本地文件 / HTTP)
    │
    ▼
interfaceLoader.ts ──── 解析、验证 interface_version === 2、处理 import 合并
    │
    ▼
appStore (Zustand) ──── 持有 projectInterface, instances, 选项值
    │
    ├── pipelineOverride.ts ──── 从用户选项生成 pipeline_override JSON
    ├── specialTasks.ts ──────── MXU 内置虚拟任务（Custom Action）
    └── maaService.ts ────────── 封装 MaaFramework API 调用
    │
    ▼
maa_core.rs (Rust) ──── 管理 MaaFramework 实例、控制器、资源、任务
mxu_actions.rs (Rust) ── 实现 Custom Action 回调
```

### 3.2 关键 PI V2 实体

| 实体 | 作用 | 前端类型 |
|------|------|----------|
| `controller` | 设备连接方式（Adb/Win32/WlRoots/PlayCover/Gamepad） | `ControllerItem` |
| `resource` | 资源包（含路径、控制器过滤器、hash） | `ResourceItem` |
| `task` | 可执行任务（入口点、选项、pipeline_override） | `TaskItem` |
| `option` | 选项定义（select/checkbox/switch/input） | `OptionDefinition` |
| `preset` | 预设配置（v2.3.0） | `PresetItem` |
| `group` | 任务分组（v2.4.0） | `GroupItem` |
| `import` | 导入外部 PI 文件（v2.2.0） | `string[]` |
| `global_option` | 全局选项（v2.3.0） | `string[]` |
| `agent` | 子进程 Agent 配置 | `AgentConfig` |

### 3.3 Pipeline Override 生成

PI V2 选项通过 `pipelineOverride.ts` 转换为 MaaFramework 可理解的 JSON：

```
Override 优先级（低→高）：
global_option < resource.option < controller.option < task.option
```

- **select**: 生成 `{ "pipeline_name": { "override_key": "selected_value" } }`
- **switch**: 生成 `{ "pipeline_name": { "override_key": true/false } }` + case 分支
- **checkbox**: 多选，每个勾选项生成独立 override
- **input**: 占位符替换 `{param_name}` → 实际值，支持 int/bool 类型转换
- 所有 override 最终以 JSON Array 格式发给 MaaFramework

### 3.4 MXU 特殊任务系统

7 个内置 Custom Action 任务，不依赖游戏画面：

| 任务 | Action | 用途 |
|------|--------|------|
| `MXU_SLEEP` | `MXU_SLEEP_ACTION` | 倒计时等待 |
| `MXU_WAITUNTIL` | `MXU_WAITUNTIL_ACTION` | 等到指定时间 |
| `MXU_LAUNCH` | `MXU_LAUNCH_ACTION` | 启动外部程序 |
| `MXU_WEBHOOK` | `MXU_WEBHOOK_ACTION` | 发送 HTTP Webhook |
| `MXU_NOTIFY` | `MXU_NOTIFY_ACTION` | 系统通知 |
| `MXU_KILLPROC` | `MXU_KILLPROC_ACTION` | 终止进程 |
| `MXU_POWER` | `MXU_POWER_ACTION` | 关机/重启/睡眠/熄屏 |

**设计模式**：
- 每个任务 `target: [0, 0, 1, 1]`（1x1 像素），`skipScreenshot: true`
- 选项通过 `pipeline_override.custom_action_param` 传参
- 前端深度合并（deep merge）所有 override 后包装为 Array 格式

**添加新特殊任务**：参见 `docs/add-special-task.md`

---

## 4. 前端架构详解

### 4.1 目录约定

```
src/
├── components/
│   ├── index.ts              ← 统一导出（Barrel Export 模式）
│   ├── app/                  ← App 级浮层（LoadingScreen, VersionWarning...）
│   ├── connection/           ← 连接逻辑（hooks + callbackCache）
│   ├── settings/             ← 设置页各 Section
│   ├── toolbar/              ← 工具栏按钮
│   ├── ui/                   ← 原子 UI 组件（Tooltip 等，基于 Radix UI）
│   ├── listItemShared.tsx    ← 共享列表项工具（右键菜单、内联编辑）
│   └── [30+ top-level .tsx]  ← 业务组件
├── stores/
│   ├── appStore.ts           ← 唯一 Zustand Store（~2200 lines）
│   ├── types.ts              ← AppState 接口定义（~500 lines）
│   └── helpers.ts            ← 纯函数辅助（ID 生成、选项校验）
├── services/
│   ├── index.ts              ← Barrel Export
│   ├── maaService.ts         ← MaaFramework API 封装（1156 lines）
│   ├── configService.ts      ← 配置持久化（读写 mxu.json）
│   ├── interfaceLoader.ts    ← PI V2 接口加载与合并
│   ├── updateService.ts      ← 自动更新（⚠️ 核心，需极度审慎）
│   ├── wsService.ts          ← WebSocket 服务
│   ├── cacheService.ts       ← 缓存管理
│   ├── contentResolver.ts    ← 内容解析（文件/URL/文本检测）
│   ├── proxyService.ts       ← 代理设置
│   ├── scheduleService.ts    ← 定时执行
│   ├── taskMonitor.ts        ← 任务监控
│   ├── taskStopService.ts    ← 任务停止
│   └── appearanceStorage.ts  ← WebUI 外观持久化
├── types/
│   ├── interface.ts          ← PI V2 协议类型（321 lines）
│   ├── specialTasks.ts       ← MXU 特殊任务定义（686 lines）
│   ├── config.ts             ← mxu.json 配置类型（238 lines）
│   └── maa.ts                ← MaaFramework 运行时类型（196 lines）
├── utils/
│   ├── index.ts              ← Barrel Export
│   ├── logger.ts             ← 统一日志（loglevel，模块化）
│   ├── backendApi.ts         ← 浏览器 HTTP API 工具
│   ├── pipelineOverride.ts   ← Pipeline Override 生成（321 lines）
│   ├── optionHelpers.ts      ← 选项值辅助函数
│   ├── paths.ts              ← 路径工具（跨平台数据目录）
│   ├── windowUtils.ts        ← 窗口操作工具
│   ├── jsonc.ts              ← JSONC 解析
│   ├── color.ts              ← 颜色工具
│   ├── cdkCrypto.ts          ← CDK 加解密（XOR + Base64）
│   ├── resourcePath.ts       ← 资源路径解析
│   ├── piEnv.ts              ← PI 环境变量
│   ├── taskSegmentation.ts   ← 任务分段（用于 Dashboard 控制器分配）
│   ├── logStdout.ts          ← 日志输出到 stdout
│   ├── runtimeLogPersistence.ts ← 运行时日志持久化
│   ├── tabExportImport.ts    ← 标签页导入导出
│   └── useExportLogs.ts      ← 日志导出 hook
├── hooks/
│   └── useIsMobile.ts        ← 移动端检测
├── themes/
│   ├── index.ts              ← 主题管理器
│   ├── types.ts              ← 主题类型
│   └── presets/              ← 主题预设 JSON
│       ├── light.json
│       ├── dark.json
│       └── accents/          ← 强调色预设（9 个内置 + 自定义）
├── i18n/
│   ├── index.ts              ← i18next 配置
│   └── locales/              ← 语言包（zh-CN, zh-TW, en-US, ja-JP, ko-KR）
└── App.tsx                   ← 主应用组件（1985 lines）
```

### 4.2 Zustand Store 模式

**单一 Store 架构**：所有状态集中在 `appStore.ts`，通过 `create<AppState>()` + `subscribeWithSelector` 中间件。

**状态域划分**：
- **Theme/Appearance**: `theme`, `accentColor`, `language`, `backgroundImage`
- **Interface**: `projectInterface`, `interfaceTranslations`, `basePath`, `dataPath`
- **Instances**: `instances[]`, `activeInstanceId`, `nextInstanceNumber`
- **Runtime**: `instanceConnectionStatus`, `instanceResourceLoaded`, `instanceTaskStatus`
- **Task Status**: `instanceTaskRunStatus` (idle/pending/running/succeeded/failed)
- **UI State**: `showAddTaskPanel`, `animatingTaskIds`, `closingTabIds`
- **Update**: `updateInfo`, `downloadStatus`, `downloadProgress`, `installStatus`
- **Config**: `saveDraw`, `devMode`, `tcpCompatMode`, `webServerEnabled`

**配置持久化**：手动通过 `configService.ts` + debounce，**不用** Zustand 的 `persist` 中间件。WebUI 模式用 `localStorage`，Tauri 模式用 `mxu.json` 文件。

**消费模式**：
```tsx
// ✅ 正确：使用 selector 精确订阅
const theme = useAppStore((s) => s.theme);
const accentColor = useAppStore((s) => s.accentColor);

// ✅ 正确：多字段用 useShallow
const { theme, accentColor } = useAppStore(
  useShallow((s) => ({ theme: s.theme, accentColor: s.accentColor }))
);
```

### 4.3 组件模式

**懒加载**：重型组件通过 `React.lazy` + `import()` 延迟加载：
```tsx
const LazySettingsPage = lazy(async () => {
  const module = await import('@/components/SettingsPage');
  return { default: module.SettingsPage };
});
```

**Drag & Drop**：使用 `@dnd-kit/sortable`，仅垂直拖拽：
```tsx
const { attributes, listeners, setNodeRef, transform } = useSortable({ id });
```

**右键菜单**：自定义 `ContextMenu` + `useContextMenu` hook，存储菜单状态和位置。

**表单控件**：`FormControls.tsx` 导出统一的 `SwitchButton`, `TextInput`, `NumberInput`, `FileInput`, `TimeInput` + 复合 Field 包装器。

**CSS 策略**：
- Tailwind CSS utility-first
- `clsx` 条件类名组合
- CSS 自定义属性（`--color-bg-primary`, `--color-accent` 等）由主题系统动态注入
- 自定义动画（task pulse, card breathing, page slide, bell shake 等）

### 4.4 双模式运行

MXU 同时支持 **Tauri Desktop** 和 **Browser WebUI** 两种模式：

| 特性 | Tauri 模式 | WebUI 模式 |
|------|-----------|-----------|
| 后端调用 | `invoke()` Tauri 命令 | HTTP REST API (`apiGet/apiPost`) |
| 配置存储 | `mxu.json` 文件 | `localStorage` + 后端缓存 |
| 外观设置 | `mxu.json` settings | `localStorage` (WebUI) / 后端缓存 |
| 实时事件 | Tauri `listen()` 事件 | WebSocket |
| 截图 | 后端缓存直接 base64 | HTTP GET `/api/maa/instances/{id}/screenshot` |

**检测环境**：`isTauri()` 函数检查 `window.__TAURI__`。

---

## 5. 后端架构详解

### 5.1 Rust 源文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.rs` | 75 | 二进制入口（CLI 解析、WebView2 检测、管理员提权） |
| `lib.rs` | 331 | Tauri App 构建（插件注册、共享状态、命令路由） |
| `web_server.rs` | 1288 | Axum HTTP + WebSocket 服务器（WebUI 后端） |
| `ws_broadcast.rs` | 61 | WebSocket 广播通道 |
| `tray.rs` | 188 | 系统托盘图标和菜单 |
| `mxu_actions.rs` | 937 | 7 个 MXU Custom Action 实现 |
| `dummy_controller.rs` | 145 | 空控制器（返回纯黑 PNG） |
| `screenshot_service.rs` | 226 | 实例截图循环服务 |
| `commands/maa_core.rs` | 1203 | MaaFramework 核心操作（22 个命令） |
| `commands/maa_agent.rs` | 849 | Agent 子进程管理 |
| `commands/file_ops.rs` | 773 | 文件 I/O、日志导出（zip） |
| `commands/download.rs` | 581 | 流式下载 + 进度报告 |
| `commands/system.rs` | 906 | CLI、提权、自启动、进程操作 |
| `commands/update.rs` | 539 | 增量/全量更新逻辑 |
| `commands/app_config.rs` | 438 | 配置加载（JSONC 解析） |
| `commands/types.rs` | 430 | 所有数据类型和状态结构 |
| `commands/utils.rs` | 279 | 路径辅助、事件发射器 |
| `commands/state.rs` | 165 | 实例状态查询 |
| `commands/tray.rs` | 28 | 托盘命令包装 |
| `commands/mod.rs` | 40 | 模块声明 |
| `webview2/detection.rs` | 188 | 注册表检测 WebView2 |
| `webview2/install.rs` | 556 | 下载并解压 WebView2 |
| `webview2/dialog.rs` | 253 | 原生 Win32 进度/错误对话框 |

### 5.2 关键 Rust 依赖

```toml
maa-framework = { version = "1", features = ["dynamic"] }  # MaaFramework 动态链接
tauri = { version = "2", features = ["image-png", "image-ico", "tray-icon"] }
axum = { version = "0.7", features = ["ws"] }              # HTTP + WebSocket
tokio = { version = "1", features = ["rt", "sync", "net", "rt-multi-thread"] }
reqwest = { version = "0.12", features = ["stream", "blocking", "json"] }
winsafe = { git = "...", features = ["advapi", "kernel", "shell", "gui", "gdi", "user"] }
```

### 5.3 初始化流程 (`main.rs` → `lib.rs`)

```
main.rs:
1. CLI 解析 (--autostart, -i, -q)
2. WebView2 检测/安装（仅 Windows）
3. 管理员提权（Release 构建，ShellExecuteEx runas）

lib.rs:
1. 注册 Tauri 插件（opener, fs, dialog, http, process, global-shortcut, autostart, log）
2. 创建共享状态（MaaState, AppConfigState, WsBroadcast）
3. 加载 interface.json + 配置文件
4. 可选启动 Axum Web 服务器
5. Windows: 移除原生标题栏（自定义 Chrome）
6. 异步清理 cache/old/
7. 预加载 MaaFramework DLL
8. 注册 ~60 个 Tauri 命令（8 个分类）
9. 窗口事件：关闭→最小化到托盘，销毁→杀 Agent 子进程
```

### 5.4 Tauri 命令分组

| 分组 | 命令数 | 文件 | 职责 |
|------|--------|------|------|
| MaaCore | 22 | `maa_core.rs` | MaaFramework 生命周期管理 |
| Agent | ~10 | `maa_agent.rs` | 子进程 Agent 管理 |
| FileOps | ~8 | `file_ops.rs` | 文件读写、日志导出、zip |
| Download | ~3 | `download.rs` | HTTP 流式下载 |
| Update | ~5 | `update.rs` | 增量/全量更新 |
| System | ~10 | `system.rs` | CLI、提权、自启动、进程 |
| Config | ~3 | `app_config.rs` | JSONC 配置加载 |
| State | ~3 | `state.rs` | 实例状态查询 |

### 5.5 Capabilities (权限)

`src-tauri/capabilities/default.json` 配置了前端可访问的系统权限：
- **核心**: window 管理（标题、大小、位置、拖拽）
- **文件系统**: 全路径读写（`**/*` 通配）
- **网络**: HTTP 请求（`https://**`, `http://**`）
- **对话框**: 文件选择
- **进程**: 重启、退出
- **全局快捷键**: 注册/注销
- **自启动**: 启用/禁用/检查
- **URL 打开**: 全路径

---

## 6. 开发约定

### 6.1 路径别名

```typescript
// tsconfig.json + vite.config.ts
"@/*" → "./src/*"

// 使用示例
import { useAppStore } from '@/stores/appStore';
import { maaService } from '@/services/maaService';
```

### 6.2 国际化

```typescript
// ❌ 硬编码（禁止）
<Button>开始</Button>

// ✅ 使用 i18n
import { useTranslation } from 'react-i18next';
const { t } = useTranslation();
<Button>{t('start')}</Button>
```

- 所有面向用户文本必须在 `src/i18n/locales/` 中定义
- 新增文本需同步 5 种语言包：`zh-CN`, `zh-TW`, `en-US`, `ja-JP`, `ko-KR`
- interface.json 的翻译使用 `getInterfaceLangKey()` 映射（`zh-CN` → `zh_cn`）

### 6.3 组件文件组织

- 每个子目录有 `index.ts` 做 Barrel Export
- `src/components/index.ts` 聚合所有公共导出
- 业务组件直接放在 `src/components/` 根目录
- 按功能领域分子目录：`connection/`, `settings/`, `toolbar/`, `ui/`, `app/`

### 6.4 日志系统

```typescript
import { loggers } from '@/utils/logger';

const log = loggers.moduleName;  // 模块化日志器
log.debug('调试信息');
log.info('关键信息');
log.warn('警告');
log.error('错误');
```

基于 `loglevel`，Dev 模式默认 `trace`，Production 默认 `debug`。文件日志自动写入 `debug/` 目录。

### 6.5 配置文件结构

MXU 配置存储为 `config/mxu-{projectName}.json`（有 PI 时）或 `config/mxu.json`（无 PI 时）。

```jsonc
{
  "version": "1.0",
  "instances": [
    {
      "id": "xxx",
      "name": "实例 1",
      "controllerName": "Adb",
      "resourceName": "资源包",
      "savedDevice": { "adbDeviceName": "emulator-5554" },
      "tasks": [
        {
          "id": "xxx",
          "taskName": "task_entry",
          "enabled": true,
          "optionValues": { "option_key": { "type": "select", "caseName": "Value" } }
        }
      ],
      "schedulePolicies": [...],
      "preActions": [...]
    }
  ],
  "settings": {
    "theme": "dark",
    "accentColor": "emerald",
    "language": "zh-CN",
    "mirrorChyan": { "cdk": "", "channel": "stable" },
    "windowSize": { "width": 1000, "height": 618 }
  },
  "recentlyClosed": [...],
  "newTaskNames": [...],
  "customAccents": [...]
}
```

### 6.6 构建与开发命令

```bash
pnpm dev          # 启动 Vite 开发服务器 (port 1420)
pnpm build        # TypeScript 检查 + Vite 构建
pnpm tauri dev    # 启动 Tauri 开发模式（热重载）
pnpm tauri build  # 构建生产包
pnpm format       # Prettier 格式化
pnpm format:check # Prettier 检查
pnpm format:rust  # cargo fmt
pnpm format:all   # 前端 + Rust 格式化
```

### 6.7 Vite 配置要点

- **端口**: 固定 1420，`strictPort: true`
- **代理**: `/api` → `localhost:12701`（Axum 后端），`/api/ws` → WebSocket
- **Chunk 分割**: 手动分包（vendor-react, vendor-react-dom, vendor-markdown, vendor-utils, vendor-ui, vendor-dnd, vendor-tauri）
- **环境变量**: `__MXU_VERSION__` 从 package.json 读取
- **路径别名**: `@` → `./src`
- **局域网访问**: 从 mxu.json 读取 `allowLanAccess`，控制 `host: '0.0.0.0'`

### 6.8 安全注意事项

- **updateService.ts**: 更新逻辑极度敏感，失效会导致用户无法修复软件
- **CDK 加密**: 使用 XOR + Base64，内存中明文，持久化时加密
- **管理员提权**: Release 构建使用 `ShellExecuteEx` + `runas` 动词
- **WebView2**: 自动检测/安装，使用固定版本缓存

---

## 7. 常见开发场景指引

### 7.1 添加新的 PI V2 支持

1. 在 `src/types/interface.ts` 中添加新类型
2. 在 `src/utils/pipelineOverride.ts` 中处理新的 override 逻辑
3. 在 `src/components/` 中添加对应的 UI 渲染组件
4. 在 `OptionEditor.tsx` 中注册新的选项渲染器

### 7.2 添加新的 MXU 特殊任务

1. **Rust 端** (`mxu_actions.rs`): 实现 `FnAction` trait
2. **前端** (`specialTasks.ts`): 注册 `TaskItem` 定义
3. **i18n**: 在所有语言包中添加任务标签
4. **详细步骤**: 参见 `docs/add-special-task.md`

### 7.3 添加新的 Tauri 命令

1. 在 `src-tauri/src/commands/` 对应文件中定义 `#[tauri::command]`
2. 在 `lib.rs` 的 `invoke_handler` 中注册
3. 在 `src/services/maaService.ts` 中添加前端调用封装
4. 在 `src-tauri/capabilities/default.json` 中检查是否需要新权限

### 7.4 添加新的 Service

1. 创建 `src/services/newService.ts`
2. 在 `src/services/index.ts` 中导出
3. 遵循现有模式：单例导出、`loggers.xxx` 日志、`isTauri()` 双模式

### 7.5 添加新的设置项

1. 在 `src/types/config.ts` 的 `AppSettings` 中添加字段（带默认值注释）
2. 在 `src/stores/types.ts` 的 `AppState` 中添加状态 + setter
3. 在 `src/stores/appStore.ts` 中实现
4. 在 `src/components/settings/` 对应 Section 中添加 UI
5. 在 `configService.ts` 的序列化/反序列化中处理

### 7.6 添加新的 UI 组件

1. 检查 `src/components/ui/` 是否有可复用的原子组件
2. 检查 `src/components/` 中是否有相似组件可参考
3. 使用 Tailwind CSS utility classes
4. 使用 `clsx` 组合条件类名
5. 所有面向用户的文本使用 `t()` 国际化

---

## 8. 关键文件速查表

| 需求 | 文件 |
|------|------|
| PI V2 类型定义 | `src/types/interface.ts` |
| MXU 特殊任务 | `src/types/specialTasks.ts` |
| 配置类型 | `src/types/config.ts` |
| MaaFramework 类型 | `src/types/maa.ts` |
| Zustand Store | `src/stores/appStore.ts` |
| Store 类型 | `src/stores/types.ts` |
| Store 辅助函数 | `src/stores/helpers.ts` |
| MaaFramework 服务 | `src/services/maaService.ts` |
| 配置持久化 | `src/services/configService.ts` |
| PI 加载器 | `src/services/interfaceLoader.ts` |
| Pipeline Override | `src/utils/pipelineOverride.ts` |
| 选项辅助 | `src/utils/optionHelpers.ts` |
| 路径工具 | `src/utils/paths.ts` |
| HTTP API 工具 | `src/utils/backendApi.ts` |
| 日志系统 | `src/utils/logger.ts` |
| 主题管理 | `src/themes/index.ts` |
| i18n 配置 | `src/i18n/index.ts` |
| 语言包 | `src/i18n/locales/*.ts` |
| 主应用 | `src/App.tsx` |
| Vite 配置 | `vite.config.ts` |
| Tauri 配置 | `src-tauri/tauri.conf.json` |
| Cargo 配置 | `src-tauri/Cargo.toml` |
| 权限配置 | `src-tauri/capabilities/default.json` |
| MaaFramework 命令 | `src-tauri/src/commands/maa_core.rs` |
| Custom Actions | `src-tauri/src/mxu_actions.rs` |
| Axum 服务器 | `src-tauri/src/web_server.rs` |
| App 构建 | `src-tauri/src/lib.rs` |
| 特殊任务开发 | `docs/add-special-task.md` |

---

## 9. 数据流图

```
用户操作
  │
  ▼
React 组件 ──useAppStore()──► Zustand Store
  │                               │
  │                               ├── projectInterface (PI V2 定义)
  │                               ├── instances[].tasks[].optionValues (用户选择)
  │                               └── runtime states (连接/任务状态)
  │
  ▼
maaService.ts
  │
  ├── Tauri: invoke('maa_xxx', params) ──► Rust 命令 ──► MaaFramework C 库
  │
  └── WebUI: apiGet/apiPost('/api/maa/...') ──► Axum 路由 ──► MaaFramework C 库
  │
  ▼
MaaFramework 回调
  │
  ├── Tauri: listen('maa-callback') ──► callbackCache.ts ──► Store 更新
  │
  └── WebSocket: wsService.onMaaCallback ──► callbackCache.ts ──► Store 更新
  │
  ▼
React 重新渲染（通过 Zustand selector 自动订阅）
```

---

## 10. 注意事项与陷阱

1. **配置保存防抖**: `configService.ts` 使用 debounce 保存，避免频繁写盘
2. **Self-save 追踪**: `markSelfSave()` / `consumeSelfSave()` 防止自己触发的 config-changed 事件导致 UI 重置
3. **Pipeline Override 深度合并**: MaaFramework 只做浅替换，MXU 在前端深度合并所有 override 后包装为 Array
4. **选项值类型安全**: `sanitizeOptionValues()` 校验持久化的选项值是否仍符合当前 PI 定义
5. **平台过滤**: `interfaceLoader.ts` 根据运行平台过滤控制器（移除非本平台的控制器类型）
6. **预加载**: MaaFramework DLL 在 `lib.rs` setup 阶段预加载，避免首次调用延迟
7. **WebView2**: 每次启动检测，使用固定版本缓存在 `cache/webview_data/`
8. **管理员权限**: Release 构建尝试提权，用户取消 UAC 则不提权继续运行
9. **截图流**: 由后端 `ScreenshotService` 驱动，前端仅订阅/取消订阅
10. **Dummy 控制器**: 用于执行不依赖游戏画面的 MXU 特殊任务（返回纯黑 PNG）
