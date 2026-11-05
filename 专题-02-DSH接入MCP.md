# 专题：在 DSH 里接入 MCP

> 适用范围：DeepSeek Harness 桌面端（Windows）。沉淀自 2026-11-05 的一次真实接入。
> 本文讲**通用方法论**；具体案例见 [`chrome-devtools-mcp-DSH-接入方案.md`](./chrome-devtools-mcp-DSH-接入方案.md)。

---

## 一、一句话理解 MCP

MCP（Model Context Protocol）是**给 AI 装外设的接口**：外部程序按协议暴露一组「工具」，agent 像调本地函数一样调它们。

```
agent ──► 工具调用 ──► MCP 客户端（在 agent 进程内）
                            │ stdio / HTTP
                            ▼
                       MCP 服务器（独立进程）
                            │
                            ▼
                       外部世界（浏览器、数据库、SaaS）
```

关键认知：**MCP 服务器不是"装进" agent 的，它是"外部进程"，agent 只是知道怎么启动它、怎么跟它说话。**

---

## 二、DSH 里最容易搞混的四层

很多人卡住，是因为把「插件」「bundle」「profile」「MCP 服务器」混成一个东西。

| 概念 | 是什么 | 例子 |
|---|---|---|
| **插件（plugin）** | 一个 JS 模块，导出 `apply(ctx)`，跑在 DSH 进程**内** | `dsh-mcp-client`、`dsh-tool-pwsh` |
| **bundle** | 一个包，自带 `cordis.patch.yml`，声明「我要挂哪些插件行」 | `@local/dsh-mcp-chrome-devtools` |
| **profile** | 一套配置档案（= 一个文件夹），决定这台机器上用哪些 bundle | `~/.dsh/profiles/desktop` |
| **MCP 服务器** | **独立进程**，不属于上述任何一层，被插件 spawn 出来 | `chrome-devtools-mcp` |

**串联方式**：

```
profile.package.json 的 dsh.profile.bundles
        └─ 每个 bundle 贡献若干「插件行」
                └─ 其中一行是 @deepseek-ai/dsh-mcp-client（唯一真正懂 MCP 的）
                        └─ 它按 config 去 spawn 一个 MCP 服务器进程
                                └─ 服务器暴露的工具 → mcp__<serverName>__<tool>
```

所以：**MCP 服务器不需要"安装到 DSH"，只需要"有一条配置告诉 DSH 怎么启动它"。**

---

## 三、DSH 配置的三层模型

| 层 | 位置 | 作用范围 | 用途 |
|---|---|---|---|
| 功能包默认层 | npm 包内置 | 该 bundle 的所有使用者 | 出厂默认 |
| **用户 patch 层** | `~/.dsh/profiles/<name>/cordis.patch.yml` | 单个 profile | 日常自定义 |
| **全机 patch 层** | `~/.dsh/cordis.patch.yml` | 本机所有 profile | 全局通用配置 |
| 启动 overlay | 任意文件 + `--patch <file>` | 本次启动 | 试验、CI |

> 对应 Claude Code 的概念：profile 层 ≈ `--scope user`；全机层 ≈ 更强；`--patch` ≈ `--mcp-config` 一次性注入。
> 差别在于**DSH 没有"项目级"**——它不按工作目录分，而是按 profile / Agent 预设分。

---

## 四、三条接入路线

| 路线 | 怎么做 | 优点 | 缺点 | 何时用 |
|---|---|---|---|---|
| **A. 图形管理器** | 装 `dsh-mcp-manager` 插件 → 设置→内置插件→MCP→添加 | 可视化、可开关、可改参、不碰 YAML | 第三方代码，以你权限运行 | **默认选它** |
| **B. 手写配置** | 往 patch 层写 `- insert:` 行 | 官方原生、可纳入版本管理 | 语法易错、改完要重启 | 管理器不可用 / 要进 Git |
| **C. CLI 装 bundle** | `dsh plugin --profile <p> add <spec>` | 一条命令、脚本化 | 需要自己生成 bundle 文件 | 批量/自动化部署 |

### 路线 A 的最小步骤

```
1. 插件页 → + 添加插件 → 粘贴 https://github.com/JokerAn/dsh-mcp-manager → 安装
2. 完全退出 DSH（含托盘）→ 重开
3. 设置 → 内置插件 → MCP → + 添加 → STDIO → 填命令/参数 → 保存
4. 保存即生效，无需重启
```

### 路线 B 的唯一正确语法

```yaml
# ✅ 新增条目必须包在 insert 里
- insert:
    - id: my-server
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: my-server
        transport: stdio
        command: npx
        args: ["-y", "some-mcp-server@latest"]
```

```yaml
# ❌ 裸 id = 「覆盖已有条目」，Loader 警告 not found 后**静默丢弃**
- id: my-server
  name: "@deepseek-ai/dsh-mcp-client"
```

**这是本专题最贵的一课**：写错不报错，你以为装上了，其实什么都没有。

---

## 五、五个坑（按踩到概率排序）

| # | 坑 | 现象 | 解 |
|---|---|---|---|
| 1 | **Windows 裸 `npx` 起不来** | `server is disconnected`，但命令行手动跑是好的 | 命令写**绝对路径** `...\npx.cmd` |
| 2 | **`insert` 语法** | 配置写了但工具不出现，无报错 | 必须 `- insert: [...]` |
| 3 | **token 常驻** | 每轮请求都变贵 | 工具 schema 全量进上下文；不用就关开关 / 用 `--category*=false` 裁剪 |
| 4 | **`serverName` 冲突** | 加载时明确报错 | 全机层与 profile 层**二选一**，别叠 |
| 5 | **`web_fetch` 假成功** | SPA 返回 200 + 空壳 | 见下节 |

---

## 六、该用浏览器还是 `web_fetch`

DSH 自带 `web_fetch`（HTTP 抓取 + 正文提取）。实测对比一条 SPA 榜单页：

| | `web_fetch` | 浏览器（MCP） |
|---|---|---|
| 返回 | 只有 `<title>` 一行 | 完整 50 条结构化内容 |
| 状态码 | 200（**不报错**） | 200 |
| 可见请求 | 1 | 81 |
| DOM | ❌ | ✅ |

**选型规则**：

```
静态 / SSR / 文章页 / 文档 ────────► web_fetch（快、便宜）
SPA / 无限滚动 / 点击才出内容 ─────► 浏览器
需要登录态 ───────────────────────► 浏览器
要看接口、请求体、响应头 ──────────► 浏览器
要截图 / 复现 bug ─────────────────► 浏览器
```

一句话：**`web_fetch` 拿的是「服务器发的第一包」，浏览器拿的是「页面自己长出来的样子」。**

---

## 七、验证清单（装完必做）

| 层 | 验证 | 通过标准 |
|---|---|---|
| 界面 | 设置→内置插件→MCP | 卡片存在，状态「已连接」 |
| 工具 | 问 agent「列出 mcp__ 开头的工具」 | 数量和预期一致 |
| 能力 | 打开一个 **SPA** 页面读内容 | 能读到渲染后内容（`web_fetch` 读不到） |
| 反证 | 打开日常已登录的站 | 显示未登录（证明隔离 profile 生效） |
| 成本 | 关掉卡片再问工具 | `mcp__` 工具全部消失 |

---

## 八、持久化速查

| 问题 | 答案 |
|---|---|
| 重启后还在吗 | ✅ 在（写在 profile 的 `package.json` + bundle 目录） |
| 新会话能用吗 | ✅ 能（bundle 层，不是会话级） |
| 全局吗 | 对**该 profile 的所有会话**是；对其他 profile 不是 |
| 怎么变全局 | 写 `~/.dsh/cordis.patch.yml`，同时删掉 profile 层那份 |

---

## 九、安全底线

1. MCP 服务器能读到什么，agent 就能读到什么——**页面内容是不可信输入**，绝不当作指令执行
2. 默认用隔离 profile（`--isolated`），不要让「登录态」和「不可信页面」出现在同一次会话
3. 调试端口（`--remote-debugging-port`）对本机任意程序开放，用完就关
4. 第三方插件 = 以你的权限运行的代码，装之前先想清楚信任边界

---

## 十、FAQ

**Q：装了管理器插件，还需要装 `dsh-mcp-client` 吗？**
不需要。它是 DSH 自带的，你只是"实例化"它。

**Q：为什么装完要重启 DSH，但加服务器不用？**
Host 侧插件代码在进程启动时加载；服务器是运行时被 spawn 的子进程，配置一改就重连。

**Q：能装多少个 MCP 服务器？**
技术上不限，但每个的工具 schema 都进每轮请求，实际按 token 预算决定。

**Q：CLI profile 和桌面 profile 是同一套吗？**
不是。它们是 `~/.dsh/profiles/` 下的不同文件夹，配置互不相通。

---

## 参考

- [DSH 官方：连接第三方 MCP 服务](https://deepseek-harness.github.io/deepseek-harness/guide/mcp-memory.md)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [dsh-mcp-manager](https://github.com/JokerAn/dsh-mcp-manager)
