# UI-TARS Desktop 分层技术架构分析

> 基于源码阅读的架构梳理：每层的组件清单、各组件职责、以及组件间调用关系。

## 1. 仓库总览

| 目录 | 角色 |
|---|---|
| `apps/ui-tars` | Electron 桌面应用本体（main / preload / renderer 三进程） |
| `packages/ui-tars` | 核心 SDK 包族：sdk、action-parser、electron-ipc、shared、operators/*、utio、visualizer、cli |
| `packages/agent-infra` | Agent 基建：browser、browser-use、logger、mcp-client、search 等 |
| `multimodal/` | 内嵌二级 monorepo（tarko / agent-tars / gui-agent / omni-tars），**并行演进的新栈**，桌面 App 不直接依赖 |
| `packages/common` + `infra/` | 构建与发布工具链（pdk） |

工程形态：pnpm + turbo monorepo；桌面应用基于 Electron 34（electron-vite + electron-forge）。

## 2. 分层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│ L1 表现层（Renderer 进程）                                           │
│    React 19 页面 · useStore 镜像 · 类型化 RPC api.ts · IndexedDB     │
├─────────────────────────────────────────────────────────────────────┤
│ L2 桥接层（Preload 进程）                                            │
│    contextBridge: electron · zustandBridge · setting · platform     │
├─────────────────────────────────────────────────────────────────────┤
│ L3 应用层（Main 进程）                                               │
│    IPC 路由(7组) · runAgent 编排 · zustand 状态 · SettingStore      │
│    ProxyClient 远程代理 · JWT 设备认证                               │
├─────────────────────────────────────────────────────────────────────┤
│ L4 Agent 核心层（@ui-tars/sdk）                                      │
│    GUIAgent 主循环 · UITarsModel · actionParser · Operator 抽象      │
├─────────────────────────────────────────────────────────────────────┤
│ L5 Operator 执行层                                                   │
│    nut-js 桌面 · browser(CDP) · adb 安卓 · Remote 云端沙箱           │
├─────────────────────────────────────────────────────────────────────┤
│ L6 基础层                                                            │
│    @ui-tars/shared · electron-ipc · utio · visualizer · cli         │
└─────────────────────────────────────────────────────────────────────┘
        │                    │                        │
        ▼                    ▼                        ▼
   VLM API(OpenAI兼容)   本机 OS(nut-js)        云端沙箱(RDP/CDP)
```

## 3. 逐层组件与职责

### L1 表现层（Renderer 进程）

| 组件 | 位置 | 职责 |
|---|---|---|
| App / 路由 | `apps/ui-tars/src/renderer/src/App.tsx` | React 19 + react-router 7（HashRouter），路由 `/`（远程）、`/local`、`/free-remote`、`/widget` |
| useStore | `apps/ui-tars/src/renderer/src/hooks/useStore.ts` | zustand 镜像 hook：订阅主进程推送的 `subscribe` 事件，在渲染层重建同名 store（zutron 式，无 rxjs） |
| api 客户端 | `apps/ui-tars/src/renderer/src/api.ts` | `createClient<Router>` 生成类型安全 RPC 客户端，供页面直接调用主进程路由 |
| SessionManager | `apps/ui-tars/src/renderer/src/db/session.ts` | IndexedDB（`ui_tars_db`）：会话持久化 |
| ChatManager | `apps/ui-tars/src/renderer/src/db/chat.ts` | IndexedDB（`ui_tars_db_chat`）：聊天记录持久化 |

UI 技术栈：Radix UI / shadcn + Tailwind 4 + swr。

### L2 桥接层（Preload 进程）

组件：`apps/ui-tars/src/preload/index.ts`，通过 contextBridge 暴露 4 个通道：

- **electron** — 通用 ipcRenderer.invoke 封装（承载 RPC 客户端）
- **zustandBridge** — 订阅主进程 `subscribe` 事件，回传状态快照
- **setting** — 设置读写专用通道
- **platform** — 平台信息

### L3 应用层（Main 进程）

| 组件 | 位置 | 职责 |
|---|---|---|
| 启动编排 | `apps/ui-tars/src/main/main.ts` | Squirrel 更新 → ElectronStore.initRenderer → 权限检查 → 托盘 → UTIOService.appLaunched → 主窗口 → 注册全部 IPC |
| IPC 路由 | `apps/ui-tars/src/main/ipcRoutes/index.ts` | t.router 组合 7 组路由：agent / setting / remoteResource / screen / window / permission / browser |
| runAgent 服务 | `apps/ui-tars/src/main/services/runAgent.ts` | **核心编排**：按 settings 选择 Operator → 组装本地/远程模型配置 → `new GUIAgent(...)` 并 `run()`；`onData` 回调里做 SoM 标注（markClickPosition）并 setState |
| agent 路由 | `apps/ui-tars/src/main/ipcRoutes/agent.ts` | runAgent / pause / resume / stop / setInstructions / 历史管理；GUIAgentManager 单例持有 Agent 实例 |
| 全局状态 | `apps/ui-tars/src/main/store/create.ts` | zustand vanilla AppState；main.ts 中 `store.subscribe → ipcMain.emit('subscribe')` 推给渲染层 |
| 设置存储 | `apps/ui-tars/src/main/store/setting.ts` | electron-store 持久化（`ui_tars.setting`），onDidAnyChange 广播 `setting-updated` |
| 远程代理 | `apps/ui-tars/src/main/remote/proxyClient.ts` | 云端沙箱 HTTP 代理：allocResource / getSandboxInfo / getBrowserCDPUrl / getTimeBalance |
| 设备认证 | `apps/ui-tars/src/main/remote/auth.ts` | jose RS256 JWT + node-machine-id，RSA 密钥存 `~/.ui-tars-desktop/` |
| 其他路由 | `apps/ui-tars/src/main/ipcRoutes/` | setting.ts（openai SDK 探测模型可用性）、screen.ts、window.ts、permission.ts、browser.ts |

### L4 Agent 核心层（@ui-tars/sdk）

| 组件 | 位置 | 职责 |
|---|---|---|
| GUIAgent | `packages/ui-tars/sdk/src/GUIAgent.ts` | **主循环**：observe（asyncRetry 截图，Jimp 校验）→ think（model.invoke）→ act（operator.execute）；处理 pause/resume、user stop（AbortSignal）、最大循环（100）、连续截图失败（≥10 报错） |
| UITarsModel | `packages/ui-tars/sdk/src/Model.ts` | openai SDK 兼容端点，支持 Chat Completions 与 Responses API（previous_response_id 链式） |
| actionParser | `packages/ui-tars/action-parser/src/actionParser.ts` | 解析模型输出的 Thought/Action 文本为结构化 `PredictionParsed[]`，坐标 /1000 归一化换算 |
| 核心抽象 | `packages/ui-tars/sdk/src/core.ts` | Operator 抽象类（screenshot / execute + MANUAL.ACTION_SPACES）与 StatusEnum 状态机 |

### L5 Operator 执行层

| 组件 | 位置 | 职责 |
|---|---|---|
| NutJSElectronOperator | `apps/ui-tars/src/main/agent/operator.ts` | 继承 NutJSOperator，desktopCapturer 截图，nut-js 控制本机鼠标键盘 |
| 浏览器 Operator | `packages/ui-tars/operators/browser` | Default / Remote 浏览器 Operator（CDP 协议） |
| 安卓 Operator | `packages/ui-tars/operators/adb` | adb 控制安卓设备 |
| 远程 Operator | `apps/ui-tars/src/main/remote/operators.ts` | RemoteComputerOperator / createRemoteBrowserOperator：经 ProxyClient 对接云端沙箱 |

### L6 基础层

| 组件 | 位置 | 职责 |
|---|---|---|
| shared | `packages/ui-tars/shared` | 类型与常量底座：MAX_LOOP_COUNT=100、DEFAULT_FACTOR=1000、UITarsModelVersion 等 |
| electron-ipc | `packages/ui-tars/electron-ipc/src/types.ts` | tRPC 风格类型安全 IPC 基建：`RouterType` / `ClientFromRouter` / `ServerFromRouter` |
| utio | `packages/ui-tars/utio` | 遥测：appLaunched / sendInstruction / shareReport |
| visualizer | `packages/ui-tars/visualizer` | HTML 执行报告模板 |
| cli | `packages/ui-tars/cli` | 终端环境复用 SDK（无 Electron） |

## 4. 关键调用链

1. **命令下行**：React 页面 → api.ts（invoke）→ preload bridge → ipcRoutes/agent.ts → runAgent → `new GUIAgent().run(instructions, history, authHeaders)`
2. **Agent 循环**：GUIAgent → `operator.screenshot()` → `model.invoke()`（+ actionParser 解析）→ `operator.execute()`
3. **状态上行**：`onData` 回调 → setState（zustand）→ `store.subscribe` → ipcMain.emit('subscribe') → preload → useStore → React 重渲染
4. **设置流**：renderer 调 setting 路由 → SettingStore（electron-store 落盘）→ onDidAnyChange 广播 `setting-updated`
5. **持久化**：会话/聊天存渲染层 IndexedDB；应用设置存主进程 electron-store
6. **外部交互**：UITarsModel → OpenAI 兼容 VLM API；Remote Operator → ProxyClient → 云端沙箱（RDP/CDP）；nut-js → 本机 OS

## 5. 附注：multimodal 并行新栈

`multimodal/` 下的 tarko、agent-tars、gui-agent、omni-tars 是基于同一套 `@agent-infra/*` 基建的并行新栈（pnpm + pdk 二级 monorepo），通过 npm 发布共享。当前桌面 App 运行时**不依赖** `@agent-tars` / `@tarko` / `@gui-agent` / `@omni-tars`（源码零引用）。
