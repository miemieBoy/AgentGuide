# chrome-devtools-mcp 接入 DSH 技术方案

| 项 | 内容 |
|---|---|
| 版本 | v2.1（含实跑验证结果） |
| 日期 | 2026-11-05 |
| 状态 | **已执行并验证通过** |
| 环境 | Windows / DSH desktop profile / Chrome stable / Node v24.6.0 |

---

## 0. 变更摘要

| 版本 | 接入方式 | 关键状态 |
|---|---|---|
| v1.1 | 手写 `cordis.patch.yml` | 语法用错（裸 `id` 会被静默丢弃） |
| v2.0 | 改用 `dsh-mcp-manager` 图形界面 | 方案层面正确 |
| **v2.1** | **同 v2.0，补实跑结果** | **已装好、已连通、32 个工具可用** |

v2.1 新增的事实（均为本机实测，不是推演）：

1. `npx` 必须写**绝对路径 `npx.cmd`**，裸 `npx` 在 Windows 下必然启动失败（R1 已发生）
2. 实际注册工具数 = **32 个**
3. 冒烟四项全部通过（导航 / JS 执行 / 无障碍快照 / Network）
4. 持久化层级确认：**profile 级**，非全机级
5. 与 DSH 自带 `web_fetch` 的 A/B 对比结果

---

## 1. 背景与目标

**问题**：让 agent 观察并操作一个真实浏览器，用于「理解某个 Web 功能如何实现 → 产出行为规格文档」。

| 目标 | 非目标 |
|---|---|
| 读 DOM、样式、组件树 | 批量采集数据（用 Scrapling/脚本） |
| 看 Network 请求与响应 | 无人值守调度 |
| 看 Console、执行 JS | 跨浏览器（Firefox/Safari） |
| 截图、录性能 trace | 桌面客户端逆向 |
| 点击、填表、导航 | 高稳定性生产任务 |

---

## 2. 方案架构

```
┌─────────────────────────────────────────────────────┐
│ DSH 桌面端（Electron 主进程）                          │
│                                                     │
│  agent ──► ctx.tools                                │
│              │  工具名: mcp__chrome-devtools__<tool>  │
│              ▼                                      │
│  @deepseek-ai/dsh-mcp-client（官方插件，stdio 桥接）    │
│              │  spawn + JSON-RPC over stdin/stdout   │
└──────────────┼──────────────────────────────────────┘
               ▼
    npx.cmd chrome-devtools-mcp@latest   （独立子进程）
               │  CDP / WebSocket
               ▼
          Chrome 实例（--isolated，临时 profile）
```

管理器的位置（不参与运行时，只写配置）：

```
设置 → 内置插件 → MCP
        ├─ 为每台服务器生成一个纯配置 bundle
        │     ~/.dsh/mcp-servers/<id>/{package.json, cordis.patch.yml}
        └─ 调 Plugin Manager 装进 profile 的 bundles
```

---

## 3. 环境与前置条件（实测）

| 项 | 状态 | 值 |
|---|---|---|
| Node.js | ✅ | v24.6.0 |
| pnpm / npx | ✅ | `D:\node\node-v24.6.0-win-x64\node-v24.6.0-win-x64\` |
| git | ✅ | `D:\git\Git\cmd\git.exe` |
| Chrome | ✅ | `C:\Program Files\Google\Chrome\Application\chrome.exe` |
| `dsh-mcp-manager` | ✅ | `@local/dsh-mcp-manager` v1.2.0 |
| MCP 服务器 bundle | ✅ | `@local/dsh-mcp-chrome-devtools` v1.0.0 |

### 安装记录

```powershell
# 1) 管理器插件
dsh plugin --profile desktop add github:JokerAn/dsh-mcp-manager
# → + @local/dsh-mcp-manager 1.2.0  (18s)

# 2) MCP 服务器 bundle（本地目录，走 link:）
dsh plugin --profile desktop add C:\Users\User\.dsh\mcp-servers\chrome-devtools
# → + @local/dsh-mcp-chrome-devtools 1.0.0  (375ms)
```

### 备份

```
C:\Users\User\.dsh\profiles\desktop\package.json.bak-20261105    # 装管理器前
C:\Users\User\.dsh\profiles\desktop\cordis.patch.yml.bak-20261105
C:\Users\User\.dsh\profiles\desktop\package.json.bak2-20261105   # 装 MCP 前
```

---

## 4. 实施步骤与结果

| 步骤 | 动作 | 结果 |
|---|---|---|
| **S0** | 重启 DSH → 设置 → 内置插件 → MCP | ✅ 标签页存在，`托管目录: C:\Users\User\.dsh\mcp-servers` |
| **S1** | 添加服务器（STDIO） | ⚠️ 首次失败 → 见 §5，改成 `npx.cmd` 后 ✅ |
| **S2** | 工具注册 | ✅ **32 个** `mcp__chrome-devtools__*` |
| **S3** | 冒烟：导航 + 读 DOM | ✅ |
| **S4** | 冒烟：Network | ✅ |
| **S5** | token 增量 | ⬜ 待记录 |

### S3/S4 实测明细

| 调用 | 结果 |
|---|---|
| `list_pages` | `1: about:blank` |
| `navigate_page` → example.com | 标题变为 `Example Domain` |
| `evaluate_script` | `{"title":"Example Domain","lang":"en"}` |
| `take_snapshot` | 7 个节点，含 iana.org 链接 |
| `list_network_requests` | 2 条：`/`、`/s.js`，均 200 |

---

## 5. 关键修正：`npx` 必须写绝对路径

**这是 v2.0 低估的一点。** v2.0 写的是「找不到再改绝对路径」，实测结论是：**必须一开始就用绝对路径**。

首次用 `"command": "npx"` 的结果：

```
mcp-client(chrome-devtools): server is disconnected
```

原因：Windows 下子进程 spawn 裸 `npx` 解析不到 `npx.cmd`。而该命令本身是可用的（手动跑 `npx.cmd -y chrome-devtools-mcp@latest --help` 正常输出参数表）。

修正：

```json
"command": "D:\\node\\node-v24.6.0-win-x64\\node-v24.6.0-win-x64\\npx.cmd"
```

改完**未重启 DSH 即生效**。

### 管理器表单对应填法

| 字段 | 值 |
|---|---|
| 显示名 | `Chrome DevTools` |
| serverName | `chrome-devtools`（`[A-Za-z0-9_-]{1,32}`，决定工具前缀） |
| 启动命令 | `D:\node\node-v24.6.0-win-x64\node-v24.6.0-win-x64\npx.cmd` |
| 参数 | `-y` / `chrome-devtools-mcp@latest` / `--isolated=true` / `--viewport=1440x900` |
| 环境变量 | 留空 |
| 工作目录 | 留空 |

---

## 6. 三套预设

改的是同一个卡片的「参数」。

| 预设 | 参数 | 适用 |
|---|---|---|
| **A 隔离+有界面**（推荐） | `-y chrome-devtools-mcp@latest --isolated=true --viewport=1440x900` | 日常调试 |
| B 无头 | `-y ... --headless=true --isolated=true --viewport=1920x1080` | CI / 无弹窗 |
| C 接管已登录 Chrome | `-y ... --browserUrl=http://127.0.0.1:9222` | 需要登录态 |

预设 C 前置（**必须先完全退出 Chrome**）：

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 --user-data-dir="C:\Users\User\.chrome-debug"
```

硬约束：Chrome 较新版本禁止在默认 profile 上开调试端口；若 Chrome 在运行，带参启动会被既有实例接管。

---

## 7. 为什么不用 DSH 自带的 `web_fetch`

DSH 自带 `web_fetch`（HTTP 抓取 + 正文提取）与 `web_search`。用同一条 SPA 链接做过 A/B：

| | `web_fetch` | chrome-devtools |
|---|---|---|
| 返回 | **仅一行 `<title>`** | 完整榜单 50 条（名称/分类/星主/价格/描述） |
| HTTP 状态 | 200（**不报错**） | 200 |
| 可见请求 | 1 | **81** |
| DOM 结构 | ❌ | ✅ 500+ 节点 |

结论：`web_fetch` 拿的是「服务器首包 HTML」，浏览器拿的是「JS 跑完后的 DOM」。

| 场景 | 选型 |
|---|---|
| 静态站、SSR、API 文档、文章页 | `web_fetch`（快、便宜） |
| SPA、无限滚动、点击才出内容 | 浏览器 |
| 需要登录态 | 浏览器 |
| 要抓接口/请求体/响应头 | 浏览器 |
| 截图、复现 bug | 浏览器 |

> 危险点：`web_fetch` 对 SPA 返回 200 + 空壳，**不会报错**，容易误以为抓到了。

---

## 8. 持久化与作用域

| 问题 | 答案 |
|---|---|
| 重启后还在吗 | ✅ 在。落盘于 profile 的 `package.json`（dependencies + bundles）+ `pnpm-lock.yaml` + bundle 目录 |
| 新会话能用吗 | ✅ 能。属 **bundle 层**，不是会话级临时插件 |
| 是全局的吗 | ⚠️ 仅对 **desktop profile 内所有会话** 全局；**不是**本机所有 profile |
| 全机层在哪 | `~/.dsh/cordis.patch.yml`（当前不存在） |

会失效的情况：`dsh plugin remove`、删除 bundle 目录、`--from-default-profile` 重建 profile、DSH 大版本重置。

> **不要两头都配**：全机层与 profile 层同时声明同一个 `serverName` 会触发命名冲突报错。

---

## 9. token 成本

理论上限 30+ 个工具，**实测 32 个**，schema 进入每一次模型请求。这是唯一实质成本。

| 手段 | 说明 |
|---|---|
| 卡片开关 | 关掉即消失，token 归零 |
| 按类裁剪 | `--categoryPerformance=false` / `--categoryEmulation=false` 等 |
| 先量后裁 | profile 的 `performanceUsage: detailed` 已开启 |

---

## 10. 安全边界

> 官方警告：接入后 agent 可读、改、执行浏览器内任意内容；连接已登录会话即等于以你身份行动。

| 风险 | 缓解 |
|---|---|
| Prompt injection（页面内容不可信，本工具**不做**清洗） | 用预设 A 隔离 profile 访问不可信页面；页面内指令一律不执行 |
| 登录态越权（预设 C） | 仅明确任务时启用，用完退出该 Chrome |
| 调试端口对本机开放 | 不常驻；不在该浏览器登录敏感站点 |
| 第三方插件以用户权限运行（未逐行审计 86KB `index.js`） | 只本地使用，不授予额外凭据 |
| MCP 调用是否进 DSH 审批流程 | **未实测**，待确认 |

---

## 11. 验收标准

- [x] MCP 标签页存在，管理器插件已启用
- [x] 卡片状态「已连接」
- [x] 工具以 `mcp__chrome-devtools__<name>` 出现（32 个）
- [x] 导航 / JS 执行 / 快照 / Network 全部通过
- [ ] token 增量已记录
- [ ] 关开关后工具消失（未测）
- [ ] MCP 调用是否进审批流程（未测）

---

## 12. 回滚

| 级别 | 操作 | 影响 |
|---|---|---|
| L1 | 卡片开关关闭 | 工具消失，配置保留 |
| L2 | 卡片垃圾桶 / 删 `~/.dsh/mcp-servers/<id>/` | 服务器移除 |
| L3 | 用 `.bak` 覆盖 `package.json` 与 `cordis.patch.yml`，重启 | 回到未装状态 |

---

## 13. 风险登记

| 编号 | 风险 | 状态 | 缓解 |
|---|---|---|---|
| R1 | Windows 裸 `npx` 启动失败 | **已发生、已解决** | 用 `npx.cmd` 绝对路径 |
| R2 | 首次 npx 下载超时 | 未发生 | 先手动 `--help` 预热 |
| R3 | token 增量超预期 | 待测 | 关开关 / 按类裁剪 |
| R4 | 页面注入攻击 | — | §10 |
| R5 | 上游 breaking change | — | 固定版本号 |
| R6 | 管理器插件与后续 DSH 版本不兼容 | — | 回退 §14 手写方案 |

---

## 14. 备用方案：手写配置

### 14.1 语法必须用 `insert`

```yaml
# ❌ 错误：裸 id 被当作「覆盖已有条目」，Loader 警告 entry "<id>" not found 后静默丢弃
- id: mcp-chrome-devtools
  name: "@deepseek-ai/dsh-mcp-client"
  config: { ... }

# ✅ 正确
- insert:
    - id: mcp-chrome-devtools
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: chrome-devtools
        transport: stdio
        command: npx
        args: ["-y", "chrome-devtools-mcp@latest", "--isolated=true", "--viewport=1440x900"]
        env: {}
        cwd: !!js process.cwd()
```

> 依据：`dsh-mcp-manager` bundle 注释——「Rows must be wrapped in `insert:`; a bare new id is treated as an override of an existing row, warns `entry "<id>" not found`, and is silently dropped.」

### 14.2 三个落点

| 范围 | 文件 | 生效于 |
|---|---|---|
| 临时 | 任意文件 + `dsh web --patch <file>` | 本次启动 |
| 单 profile | `~/.dsh/profiles/desktop/cordis.patch.yml` | 仅 desktop |
| 全机 | `~/.dsh/cordis.patch.yml` | 所有 profile（需新建） |

### 14.3 官方最佳实践（`docs/user/guide/mcp-memory.zh.md`）

1. DSH 不负责下载服务器，stdio 类要先自行安装
2. 密钥放 `config.env`，不写明文
3. 不要覆盖已有 patch 文件，要合并
4. 先用 `--patch` 试，稳定后落盘
5. `serverName` 唯一（`[A-Za-z0-9_-]{1,32}`）
6. 只桥接 tools，Resources/Prompts 不支持
7. 验证用新会话，不必重启 Host
8. stdio 会剥离 `DSH_*` 与类凭据变量，需显式 `env`

---

## 15. 附录

### A. 关键路径

| 用途 | 路径 |
|---|---|
| DSH 配置根 | `C:\Users\User\.dsh` |
| profile 目录 | `...\profiles\desktop` |
| 用户 patch 层 | `...\profiles\desktop\cordis.patch.yml` |
| 管理器插件 | `...\profiles\desktop\node_modules\@local\dsh-mcp-manager` |
| MCP bundle | `C:\Users\User\.dsh\mcp-servers\chrome-devtools\` |
| 管理器日志 | `...\profiles\desktop\.plugin-manager\logs\` |
| Chrome | `C:\Program Files\Google\Chrome\Application\chrome.exe` |

### B. 参考资料

- [DSH 官方：连接第三方 MCP 服务](https://deepseek-harness.github.io/deepseek-harness/guide/mcp-memory.md)
- [dsh-mcp-manager](https://github.com/JokerAn/dsh-mcp-manager)
- [Chrome DevTools for agents](https://developer.chrome.com/docs/devtools/agents/get-started)
- [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp)

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-11-05 | 初稿：三套预设、实施步骤、验收、回滚 |
| v1.1 | 2026-11-05 | 补全实测前置条件；加 DSH 侧事实与路径附录 |
| v2.0 | 2026-11-05 | 改用 `dsh-mcp-manager`；修正 `insert` 语法 |
| v2.1 | 2026-11-05 | **实跑完成**：`npx.cmd` 改为强制要求；补 32 工具/冒烟结果；新增 §7 web_fetch 对比、§8 持久化与作用域 |
