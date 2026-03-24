## 第九章：终端交互层——CLI、TUI 与 MCP 外壳

> **核心代码**：`crates/openfang-cli/src/main.rs`（6307 行）、`tui/mod.rs`（2278 行）、`tui/event.rs`（2600+ 行）、`launcher.rs`（600 行）、`mcp.rs`（426 行）、`tui/screens/welcome.rs`、`tui/screens/wizard.rs`
> **本章命题**：如果前六章讲的是 OpenFang 的内核、记忆、工具、安全与接入面，那么这一章讲的是：**人类究竟如何真正操作这个系统**。

很多框架会把 CLI 写成一层很薄的参数壳，把 TUI 写成几个漂亮页面，把 MCP server 当成顺手放进去的小工具。`openfang-cli` 不是这种结构。它实际上承担了三种角色：

1. **系统总入口**：把 `init`、`start`、`doctor`、`agent`、`models`、`memory`、`security`、`cron` 等命令统一编排成一个人类可用的控制面。
2. **交互式控制台**：通过 Ratatui 把 daemon 或 in-process kernel 的能力组织成一个带启动阶段、事件分发、19 个页面的终端操作系统外壳。
3. **IDE/外部工具桥**：通过 `openfang mcp` 把运行中的 agent 暴露成 stdio JSON-RPC 工具；`MCP` 的术语说明可回看[附录 A：术语表与核心概念索引](openfang_tutorial_appendix_a_glossary.md)。

这意味着 `openfang-cli` 不是附属品，而是 OpenFang 面向操作员、开发者、自动化脚本和 IDE 生态的第一层产品化界面。

---



### 9.0 视觉化图解：TUI 的控制面透视

> **Day 2 调试体验体验铆点**：当 Agent 在后台长驻执行时出错了，我们该如何干预他？OpenFang 的 `Ratatui` 终端界面不仅是“好看”，更是一个强大的进程监视器。
> 想象你在命令行输入 `openfang tui`，你看到的并不是一堆滚动的日志瀑布流，而是像 `htop` 一样的多维监控大盘：

```text
┌─ OpenFang TUI (Daemon Connected) ───────────────────────────────┐
│ [1] Dashboard │ [2] Agents │ [3] Logs │ [4] Tools │ [5] Audit   │
├─────────────────────────────────────────────────────────────────┤
│ Active Agents (2 running)                                       │
│ 🟢 Agent-Alpha (Scraping Task)   [CPU: 12%] [Mem: 24MB]        │
│ 🟡 Agent-Beta  (Waiting Approval)[CPU:  0%] [Mem: 18MB]        │
│                                                                 │
│> Select Action: [K]ill  [P]ause  [A]pprove Tool Call            │
├─────────────────────────────────────────────────────────────────┤
│ Live Event Stream (Event Bus)                                   │
│ 14:02:01 [WARN] Agent-Beta requested `rm -rf /` -> Blocked.     │
│ 14:02:03 [INFO] Sandbox Layer 2 denied cross-dir execution.     │
└─────────────────────────────────────────────────────────────────┘
```
*这个控制台直接连通了 `openfang-api` 和内核的 EventBus，让你在高压状态下有了“切断电源”和“批准权限”的视觉触手。*


### 9.1 深入解读

#### 设计张力

这一章的核心张力有三组：

1. **daemon 优先 vs 单进程兜底**
   OpenFang 希望长期运行在 daemon 模式下，但又不想让单次命令、MCP server、快速聊天在没有 daemon 时完全瘫痪。因此 CLI 大量地方都先探测 daemon，再决定是否回退到 in-process kernel。

2. **命令面极宽 vs 用户认知负担**
   `main.rs` 提供了极宽的命令面，但 `launcher.rs`、`welcome.rs`、`wizard.rs` 又试图把它重新压回“先能开始使用”的低认知路径。

3. **统一控制面 vs 后端能力不对称**
   TUI 试图把 daemon 与 in-process kernel 都包装成统一后端，但 `event.rs` 明确表明两种模式并不完全等价：有些页面在 daemon 模式下能拿到真实 API 数据，在 in-process 模式下只能给空列表、默认值或“暂不支持”。

#### 关键场景 🎬

第九章如果只被理解成“CLI/TUI/MCP 功能介绍”，还不够。它真正要替 OpenFang 回答的是 4 类操作层场景：

1. **本地开发者日常控制场景**
   这里最重要的不是命令多，而是 `init / start / status / doctor / chat` 能不能组成一条稳定的最小控制闭环。

2. **长期 daemon 运维场景**
   当系统不是一次性脚本，而是长驻 daemon + dashboard + TUI 控制面时，CLI/TUI 的价值就从“方便”升级成“第一层运维入口”。

3. **无 daemon 的快速兜底场景**
   in-process 模式的意义从来不是功能对等，而是让本地快速验证、单次聊天、最小调试不至于因为 daemon 缺席而完全断路。

4. **IDE / 外部自动化桥接场景**
   `openfang mcp` 不是彩蛋，而是把 OpenFang 的 agent 控制面重新包装给编辑器、外部工具与脚本生态的桥。它决定了 OpenFang 是否真的能进入更大的工作流里。

所以，这一章真正关心的不是“界面层是否漂亮”，而是：**OpenFang 是否已经拥有一层足够稳定、足够统一、又允许多入口共存的人类操作面。**

#### 9.1.1 `main.rs`：不是参数解析器，而是 OpenFang 的总入口路由器

`main.rs` 顶部注释已经把运行哲学说得很清楚：当 daemon 在运行时，CLI 通过 HTTP 与它交互；否则命令会 boot 一个 in-process kernel 来做单次操作。这一个前提决定了整个 crate 的形态不是“shell wrapper”，而是**模式分发器**。

从 `Commands` 枚举能确认，这个入口面覆盖了几乎整个系统控制面：

- `init` / `start` / `stop`
- `agent` / `workflow` / `trigger`
- `skill` / `channel` / `hand`
- `chat` / `message`
- `status` / `health` / `doctor`
- `models` / `memory` / `security`
- `approvals` / `cron` / `sessions` / `logs`
- `mcp` / `dashboard` / `tui`
- `vault` / `integrations` / `configure` / `reset` / `uninstall`

这一层最值得注意的，不是命令很多，而是它们大多不是本地小动作，而是系统能力的文本化暴露。也就是说，CLI 在 OpenFang 里承担的是“人类控制平面”的职责。

在 `main.rs` L800-L827 附近，CLI 还定义了一个很产品化的判断：

- 如果没有子命令，且 stdout 是 TTY，就走 launcher
- 如果显式传了 `tui`，直接进入全屏交互模式
- 如果显式传了 `mcp`，进入 stdio MCP server

这说明 OpenFang 把“无参数直接启动”视作用户体验问题来处理，而不是简单打印 `--help` 了事。

对应的总分发里，`Commands::Tui` 在 L841 进入 `tui::run(cli.config)`，`Commands::Mcp` 在 L909 进入 `mcp::run_mcp_server(cli.config)`。这两个入口分支几乎可以看成是 `openfang-cli` 里的两个“子产品”：一个给终端操作者，一个给 IDE / MCP 客户端。

#### 9.1.2 `find_daemon()`：整套 CLI 的模式切换枢纽

CLI 对 daemon 的探测被集中在 `find_daemon()`（`main.rs` L1011-L1025）里。它的行为很具体：

1. 从 `~/.openfang` 读取 daemon 信息
2. 规范化监听地址，把 `0.0.0.0` 替换为 `127.0.0.1`
3. 访问 `/api/health`
4. 成功则返回 base URL，否则视为 daemon 不可用

这个函数之所以关键，是因为它不是只被 `status` 或 `start` 使用，而是在整份 `main.rs` 中大量复用。换句话说，OpenFang 的 CLI 不会先抽象一个“transport layer”再往下传，而是以 `find_daemon()` 为模式分叉点，在不同命令里决定：

- 走 daemon HTTP
- 或 boot 本地 kernel

这是一种非常直接的工程风格：不优雅，但清晰，而且在本地开发者工具里往往更可靠。

#### 9.1.3 `launcher.rs`：无子命令时的轻量操作壳

`launcher.rs` 不是完整 TUI，而是一个 one-shot 启动器。它处理的是“用户直接输入 `openfang` 后该看到什么”的问题。

它有几个很有产品意味的判断：

- `is_first_run()`：如果 `~/.openfang/config.toml` 不存在，判定为首次运行
- `has_openclaw()`：如果首次运行且本机存在 `~/.openclaw`，说明很可能是迁移用户
- `detect_provider()`：优先探测是否已有 API key 环境变量

然后根据首次运行与否切两套菜单：

- **首次运行菜单**：把 `Get started` 放在第一位
- **回访用户菜单**：把 `Chat`、`Dashboard`、`Terminal UI` 这些高频动作放前面，把 `Settings` 放后面

这说明 OpenFang 的 CLI 不只是开发者工具，也在努力承担 onboarding 责任。它想解决的问题不是“怎么把所有命令列出来”，而是“用户第一次打开时，怎么尽快进入正确路径”。

另一个值得注意的点是：`launcher.rs` 也会在后台线程中探测 daemon，并根据结果显示不同状态。这和后面的 welcome screen 形成了呼应，说明“是否连 daemon”是整个 CLI/TUI 共同关心的第一层状态。

#### 9.1.4 TUI 的主骨架：Boot Phase → Main Phase

`tui/mod.rs` 顶部注释写的是：`Phase::Boot (Welcome/Wizard) -> Phase::Main with 19 tabs`。继续往下看 `Tab` 枚举会发现，实际有 **19 个 Tab**：

- Dashboard
- Agents
- Chat
- Sessions
- Workflows
- Triggers
- Memory
- Channels
- Skills
- Hands
- Extensions
- Templates
- Peers
- Comms
- Security
- Audit
- Usage
- Settings
- Logs

这本身就是一个非常有价值的信号：**注释已经落后于实现。** 这通常意味着 TUI 的能力面增长很快，文档/注释还没完全跟上。

结构上，`App` 里最关键的字段有三组：

1. **阶段状态**：`phase`、`active_tab`
2. **后端状态**：`backend`、`chat_target`
3. **页面状态**：每个 screen 各自一份 state，如 `dashboard`、`agents`、`channels`、`usage`、`logs` 等

这让 `tui/mod.rs` 更像一个小型前端应用的 state container，而不是传统 terminal menu 程序。

它的核心抽象是 `Backend`：

- `Daemon { base_url }`
- `InProcess { kernel }`
- `None`

然后通过 `to_ref()` 折叠成可传给后台线程的 `BackendRef`。这个设计非常关键，因为它决定了 TUI 不是绑死在 HTTP API 上，而是试图给 daemon 和本地 kernel 提供一个统一的读写界面。

#### 9.1.5 `event.rs`：真正把 TUI 跑起来的是事件总线，不是页面绘制

如果只看 `tui/mod.rs`，会以为核心复杂度在页面状态；但再看 `tui/event.rs`，会发现真正让这个 TUI 成为“系统控制台”的，是事件系统。

`AppEvent` 统一承载了几类事件：

- 键盘输入与 Tick
- LLM 流式输出
- kernel 启动完成/失败
- daemon 探测结果
- 各个页面的数据加载结果
- 各种副作用动作的完成结果，如 skill 安装、trigger 删除、approval 操作、workflow 运行等

`spawn_event_thread()` 则把 `crossterm` 输入与固定节拍 tick 串到同一通道里。这里还有一个非常现实的兼容细节：只转发 `KeyEventKind::Press`，因为 Windows 会额外发送 Release/Repeat，容易造成双重输入。这种处理非常“脏活”，但正是 TUI 可信赖的基础。

更重要的是，`event.rs` 把数据面刷新普遍做成后台线程，比如：

- `spawn_fetch_dashboard()`
- `spawn_fetch_channels()`
- `spawn_fetch_workflows()`
- `spawn_fetch_sessions()`
- `spawn_fetch_memory_agents()`
- `spawn_fetch_skills()`
- `spawn_fetch_hands()`
- `spawn_fetch_extensions()`
- `spawn_fetch_usage()`
- `spawn_fetch_models()`
- `spawn_fetch_peers()`
- `spawn_fetch_logs()`
- `spawn_fetch_comms()`

这说明 OpenFang TUI 的模型不是“用户按一次键，同步卡住获取数据”，而是“页面切换或动作触发后异步刷新，结果通过统一事件回流到状态机”。

#### 9.1.6 daemon / in-process 双后端：统一得不彻底，但很实用

第九章最关键的工程观察之一，是 `event.rs` 大量 `spawn_fetch_*` 函数都长成这样：

```rust
match backend {
    BackendRef::Daemon(base_url) => { ... HTTP API ... }
    BackendRef::InProcess(kernel) => { ... 本地数据或空实现 ... }
}
```

这背后有两个现实：

1. **OpenFang 确实在认真做双模式统一**
   例如 `spawn_fetch_dashboard()` 在 daemon 模式会拉 `/api/status` 和 `/api/audit/recent`，而 in-process 模式至少还能从 `kernel.registry.count()` 构造出最小 Dashboard 数据。

2. **两种模式并没有完全对齐**
   很多页面在 in-process 分支里直接返回：
   - 空列表
   - 默认值
   - “not available in in-process mode”

比如：

- `spawn_fetch_channels()` 的 in-process 分支直接给空列表
- `spawn_test_channel()` 在 in-process 模式直接返回“不支持”
- `spawn_fetch_workflows()` / `spawn_fetch_workflow_runs()` 在 in-process 模式目前基本为空
- `spawn_fetch_mcp_servers()`、`spawn_fetch_template_providers()`、`spawn_fetch_audit()`、`spawn_fetch_models()` 等也有大量 daemon-only 倾向

这带来的真实结论不是“设计不好”，而是：**OpenFang 的 TUI 目前本质上是 daemon-first，in-process 是为了兜底与快速单机可用，而不是功能完全对等的第二实现。**

#### 9.1.7 Welcome / Wizard：不是装饰层，而是启动阶段状态机

`welcome.rs` 和 `wizard.rs` 共同构成了 TUI 的 Boot Phase。

`WelcomeState` 处理几个关键行为：

- daemon 后台探测完成后重建菜单
- 如果 daemon 可用，菜单包含 `Connect to daemon`
- 始终保留 `Quick in-process chat`
- 提供 `Setup wizard`
- 通过双击 `Ctrl+C` 才真正退出，避免误触

这个 welcome screen 不是简单首页，它承担的是**模式选择器**的职责：

- 连 daemon
- 本地 quick chat
- 进入配置向导
- 退出

`WizardState` 则把首次配置压成一个 3 步流程：

1. 选 provider
2. 录入或复用 API key
3. 选 model 并落盘

其中有两个很实用的策略：

- 如果 provider 是本地型（如 `ollama`、`vllm`、`lmstudio`），直接跳过 API key
- 如果环境变量里已经存在 key，则不强迫再次录入

最终它会把配置写入 `~/.openfang/config.toml`，并创建 `agents/`、`data/` 等目录。这说明 wizard 的目标不是“演示 UI”，而是让用户在一次终端交互里拿到一个可启动的最小系统。

#### 9.1.8 `mcp.rs`：CLI 里其实藏着一个 stdio 协议桥

第六章讲了 OpenFang 的 MCP 集成面，但 `openfang-cli/src/mcp.rs` 解决的是另一个问题：**如何把 OpenFang 的 agent 变成外部 MCP 客户端可调用的工具。**

这里的设计很直接：

- `McpBackend::Daemon`：如果 daemon 存在，就通过 HTTP API 代理
- `McpBackend::InProcess`：否则本地 boot 一个 kernel，再直接调它

`run_mcp_server()` 循环读取 stdio，使用 Content-Length framing。`read_message()` 里还有一个很务实的安全边界：如果 MCP 消息超过 10MB，会主动丢弃并报错，避免 OOM 与流错位。

协议层它只支持必要的最小 JSON-RPC 路径：

- `initialize`
- `notifications/initialized`
- `tools/list`
- `tools/call`

而 `tools/list` 做的事情也很清楚：把 agent 暴露为 `openfang_agent_{name}` 工具。之后 `tools/call` 会把 tool name 反解析回 agent，再把 `message` 参数转成一次真实 agent message。

这说明 `mcp.rs` 不是“OpenFang 自己的 MCP runtime”，而是**把 OpenFang 控制面重新包装成 MCP-compatible tool surface 的桥接壳。**

#### 9.1.9 这一层的真正价值：把系统能力重新包装成“人类可操作性”

如果把前几章理解为“OpenFang 会做什么”，那第九章讲的是“人怎么稳定地让它做这些事”。

`openfang-cli` 的最大价值不在于命令多，也不在于 TUI 漂亮，而在于它完成了三次重新包装：

1. 把内核能力包装成命令
2. 把命令能力包装成交互状态机
3. 把 agent 能力包装成 MCP 工具

这三次包装之后，OpenFang 才真正从“Rust workspace”变成“可操作系统”。

---

### 9.2 压力测试 🧪

#### 思想实验 1：daemon 在 TUI 运行中途消失

场景：用户通过 daemon 模式进入 TUI，打开了 Dashboard、Workflows、Usage 等页面，随后 daemon 被外部停止。

从当前实现看，TUI 的很多 `spawn_fetch_*` 路径都显式依赖 HTTP API。一旦 daemon 消失，结果大概率是：

- 某次刷新失败
- 页面局部停留在旧状态
- 通过 `FetchError` 或局部空结果反馈失败

优点是：这种失败通常是**页面级**的，而不是整个 TUI 崩溃。

风险是：当前很多 fetch 逻辑各写一套，错误反馈是否一致并不完全有保障。换句话说，系统已经具备“局部失败”方向，但尚未形成完全统一的降级语义。

#### 思想实验 2：用户在 in-process 模式下期待完整 TUI 能力

场景：用户没有启动 daemon，直接进入 TUI，希望像 daemon 模式一样看 Channels、Workflows、Audit、Templates、MCP server 等全套信息。

实际代码表明，这种期待当前会部分落空。很多 `BackendRef::InProcess` 分支只是：

- 返回空列表
- 返回默认摘要
- 明示“不支持”

因此当前最真实的行为边界是：

- **in-process 模式适合快速本地聊天与最小控制**
- **daemon 模式才是完整 TUI 体验的主路径**

这不是 bug，而是当前产品分层的真实现状，但文档和界面最好明确说出来，否则用户会误判为“页面坏了”。

#### 思想实验 3：MCP 客户端发送超大消息

场景：某个 MCP 客户端给 `openfang mcp` 发送一个超大的 JSON-RPC 包。

`mcp.rs` 的 `read_message()` 明确设置了 `MAX_MCP_MESSAGE_SIZE = 10MB`，超过后会：

- 主动读掉消息体，避免流错位
- 返回 `InvalidData` 错误

这是一种非常务实的保护方式。它没有试图做复杂流控，但至少先把最危险的“单条消息撑爆进程”场景挡住了。

#### 思想实验 4：首次启动用户根本没有 API key

场景：用户是全新安装，没有任何 provider key，也没有 daemon。

`launcher.rs`、`welcome.rs`、`wizard.rs` 的组合实际上已经给了一个相对合理的落地路径：

- launcher 检测 first run
- welcome 允许直接进入 setup wizard
- wizard 提供 provider → key → model 的三步配置
- 本地 provider 可以跳过 key

这说明 OpenFang 的 CLI/TUI 已经开始认真处理“冷启动可用性”，而不只是把所有责任都推给 README。

---

### 9.3 改进雷达

#### 🔬 ZeroClaw：操作面更收敛，OpenFang 的优势是面广

ZeroClaw 的整体风格更保守，通常会把操作面收得更紧，优先保证系统路径的稳定与一致。相比之下，OpenFang 在 CLI/TUI 这一层展现出的特点是：

- 命令面很宽
- 页面面很宽
- daemon / in-process / MCP 三种入口同时存在

这给了它更强的产品外壳，但也意味着更多状态同步、能力对齐和注释文档维护压力。

#### 🚧 OpenClaw：脏活成熟度通常更高，但控制面未必更统一

OpenClaw 这类系统在启动引导、消息边角、UI 细节、异步错误打磨上往往更老练。OpenFang 当前在这章里的弱点，也主要集中在这些地方：

- daemon / in-process 能力不完全对等
- fetch 逻辑大量重复
- 注释与实现已有轻微脱节

但 OpenFang 有一个非常明确的优势：**它把 CLI、TUI、MCP 三层都挂在同一系统控制面之上。** 这让它不是几个离散工具，而是一个可组合的人类操作壳。

#### 当前最值得优先修的 5 个点

1. **TUI 注释存在过漂移**：文件头曾写 16 tabs，但实际已有 19 tabs，这正是典型需系统级收敛的代码文档漂移问题。
2. **in-process 模式能力矩阵不透明**：用户很难预判哪些页面会空、哪些能工作。
3. **`event.rs` 后台抓取重复样板过多**：大量 `spawn_fetch_*` 都在重复搭 client、拉接口、parse JSON。
4. **MCP bridge 缺少更显式的审计面**：当前 tool call 能转 agent message，但操作员可见性还不够强。
5. **launcher / welcome / wizard 三层启动体验有重叠**：都在做一部分“发现 daemon / 发现 provider / 引导下一步”的工作，未来可继续收敛。

#### 验证资产与质量保障

作为全系统外壳，CLI/TUI层的测试目标天然和其他层面不同，需要建立专用的验证防线：
1. **端到端集成测试**：重点保障 `find_daemon()` 切换逻辑的正确性，在 daemon 存活和意外宕机这两种不同的降级态中，必须确保能平滑回退而不会死锁。
2. **状态防漂移**：前面提到的 16/19 个 TUI tab 漂移就是一个典型症状，未来这里应该有 AST 级别的文档防漂移测试。
3. **MCP Stdio Bridge 回归面**：MCP 的 `run_mcp_server` 是核心工具输出口，它的 JSON-RPC 的解析和响应错误，必须作为 `openfang-api` 的强依赖节点被包含在回归测试之内。

---

### 9.4 行动项

| 改进项 | 影响力 | 难度 | 目标模块 |
| --- | --- | --- | --- |
| 为 TUI 明确展示当前后端能力矩阵，例如在页面顶部标明 `Daemon` / `InProcess` 并提示不可用功能 | 高 | 低 | `tui/mod.rs` + 各 screen 状态栏 |
| 收敛 `event.rs` 中重复的 daemon fetch 样板，提炼统一的请求/错误处理助手 | 高 | 中 | `tui/event.rs` |
| 把 in-process 分支逐步补齐到可用最小集，至少让 Channels / Workflows / Audit 不再全部空白 | 高 | 中高 | `tui/event.rs` + kernel 直连查询接口 |
| 建立防止文档与注释漂移的自动巡检系统（如 TUI Tab 数量曾有过 16 与 19 的漂移） | 中 | 低 | `tui/mod.rs` 注释与相关文档 |
| 为 `openfang mcp` 增加 tool call 审计与更清晰的 agent 映射展示 | 中高 | 中 | `mcp.rs` + 审计/日志面 |
| 收敛 launcher、welcome、wizard 的启动探测逻辑，减少重复的 provider/daemon 检测代码 | 中 | 中 | `launcher.rs` + `welcome.rs` + `wizard.rs` |

---

#### 源码补充：P2P 通讯与审计面

除了人类可见的 CLI/TUI 交互，第九章还值得补一个较少被文档正面展开的事实：操作层并不是悬在空中的壳，它直接连着 OpenFang 的节点通信与审计基础设施。

- `openfang-wire/src/peer.rs` 说明系统并不局限于单机会话控制。底层存在基于 TCP 的节点互联协议，消息采用长度头加 JSON frame 的方式封装，并在握手阶段加入 HMAC-SHA256 校验。这意味着控制面后面站着的是一个可扩展节点网络，而不是只能操作本地进程的前端壳。
- `openfang-runtime/src/audit.rs` 则解释了为什么 TUI 里的 `Audit` 页面不是普通日志查看器。这里实现的是一条面向高风险动作的 Merkle hash chain；它当前仍偏进程内，但已经具备了“操作是否被篡改”这一层验证语义。
- 因而，从教程结构上看，第九章真正负责的是把“人类如何操作系统”这件事讲清楚，同时把读者引回第六章和第五章：前者提供互联面，后者提供审计与治理面。

### 9.5 交叉引用导读 🔗

- **[第五章：Kernel 心脏——事件总线、精算引擎与编排三件套](./openfang_tutorial_chapter_5.md)**：第九章里看到的 Audit、预算展示和操作入口，背后依赖的就是第五章讲清楚的治理骨架。
- **[第六章：接入面与互联系统](./openfang_tutorial_chapter_6.md)**：如果你想把 CLI/TUI/MCP 的外壳进一步接回 API、A2A、OFP 和渠道漏斗，第六章是直接上游。
- **[第十章：未来之章——让 OpenFang 参与自己的进化](./openfang_tutorial_chapter_10.md)**：控制面之所以重要，不只是因为它让人能操作系统，更因为未来很多受控维护闭环都会从这里被观测、触发和审计。

### 9.6 本章小结

第九章补上的，不是一个“命令手册”，而是 OpenFang 的人类操作层真相。

从源码看，`openfang-cli` 至少已经形成了三层真实能力：

- `main.rs` 作为总入口路由器
- `tui/mod.rs` + `event.rs` 作为交互状态机与异步控制台
- `mcp.rs` 作为把 agent 包装成 MCP 工具的 stdio 协议桥

更重要的是，这三层都建立在同一个系统控制面之上。它们不是彼此无关的小工具，而是 OpenFang 操作系统外壳的不同呈现方式。

但也必须说清楚当前边界：TUI 仍然明显是 daemon-first，in-process 更像兜底模式；事件抓取层已经足够实用，但重复样板较多；MCP bridge 已经可用，但审计与可观测性还有继续补强空间。

但如果把这章放回整本教程的新框架里，结论还可以再更明确一点：

- **从产品现实压力看**，OpenFang 必须保住 daemon、TUI、MCP、quick chat 这几条入口共存，因为真实操作者不会只使用一种入口。
- **从运行时底线压力看**，CLI/TUI 层不能无限堆壳，必须继续减少重复 fetch、明确后端能力矩阵、收敛注释与实现漂移，否则操作层自己会变成新的复杂性来源。
- **从源码现状看**，`cmd_init()`、`cmd_start()`、`cmd_status()`、`cmd_doctor()`、`find_daemon()`、`run_mcp_server()` 已经构成了很实的第一代控制平面骨架。
- **从下一步演进看**，真正该追求的不是再多几个页面，而是把 daemon-first 的完整体验、in-process 的兜底语义、以及 MCP 的外部桥接统一成更透明的能力矩阵。

因此，第九章给出的最终判断是：**OpenFang 已经拥有真正的操作层，而不是只会跑 agent 的内核；接下来的重点，不是“有没有界面”，而是把这层界面的后端一致性、错误语义和长期维护性继续做扎实。**