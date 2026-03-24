## 第三章：工具引擎——安全执行与四层沙箱 🔧

> **核心代码**：`openfang-runtime/src/tool_runner.rs`（3686 行）、`sandbox.rs`（607 行）、`tool_policy.rs`（478 行）、`shell_bleed.rs`（354 行）、`subprocess_sandbox.rs`（698 行）、`docker_sandbox.rs`（640 行）、`host_functions.rs`（668 行）
> **本章主轴**：工具执行链、策略决策、环境隔离、WASM 与 Docker 沙箱、Shell 泄露扫描
> **对标项目**：OpenClaw 的 Shell AST/混淆检测体系，ZeroClaw 的 canonicalize + symlink 防线与安全策略层

如果说第二章解释了 Agent 怎样思考，那么第三章解释的是：**当 Agent 决定动手时，OpenFang 怎样避免它把宿主机一起拖下水。** 这不是一个“给不给工具权限”的问题，而是一个多层问题：谁有资格调用、是否需要审批、参数是否带毒、命令会不会泄露环境变量、进程运行在哪一层隔离里、以及失败后如何把破坏面控制住。

---

### 3.1 深入解读

#### 设计张力 🎯

> **工具覆盖面** ←→ **隔离强度**
> 如果所有工具都只能跑在最重的 Docker 里，安全是高了，但延迟和复杂度会拖垮日常交互；如果都跑在宿主机 shell 里，体验是爽了，但一条被注入的 `curl | sh` 就足够出事故。OpenFang 选择的是分层执行：不是“一把锁”，而是一串闸门。

> **启发式拦截** ←→ **语义级解析**
> `check_taint_shell_exec()` 和 `shell_bleed.rs` 都偏启发式：靠模式、关键字、白名单做早拦截。它们实现快、成本低，但也天然容易被变种绕过。OpenClaw 则在 shell 解析上走得更重，代价是系统复杂度飙升。

> **声明式策略** ←→ **执行性策略**
> `ToolPolicy` 已经支持 deny-wins、glob、group、depth 限制，但“策略能表达什么”和“运行时真正拦了什么”仍不是一回事。第三章最大的工程现实，就是很多安全能力是半闭环的。



> **⚠️ 避坑指南 (Production Footgun)：为什么你需要了解执行污染？**
>
> 让我们来看一个非常真实的业务场景：用户让 Agent “写一个 Python 脚本来分析昨天的错误日志”。
>
> **在基于 Node.js/TS 编写的早阶框架中：**
> 很多开发者直接把 LLM 生成的代码扔进 `eval`，或者通过 `child_process.exec()` 执行：
> ```javascript
> // 💣 灾难性代码 (OpenClaw 早期的泥潭)
> const user_code = llm_response.code;
> const result = await exec(`python3 -c "${user_code}"`);
> ```
> 如果 `user_code` 里混入了 `import os; os.system("rm -rf /")`，或者 LLM 生成的代码没有正确转义双引号导致提前闭合闭包，你的宿主机就报废了。
>
> **在 OpenFang (Rust) 的 `tool_runner.rs` 和 `sandbox.rs` 中：**
> OpenFang 不再允许上述情况发生。它的 `shell_bleed.rs` 和 `check_taint_shell_exec()` 强制在执行前拦截（AST Taint/启发式检查），并且强制你申明这在**第几级沙箱**（如 Docker 或 WASM）运行，通过严格的 `std::process::Command` 并绑定工作区目录 (`workspace_root`) 来封堵漏洞。任何带有 `env` 泄露特征的尝试都会直接引发 `ToolError::BlockedByPolicy`。


#### 关键场景 🎬

第三章最需要面对的不是抽象的“工具安全”，而是 **一旦 Agent 真开始动手，宿主机和外部世界会怎样反咬系统一口**。至少有 4 类场景会把这一层的短板立即暴露出来：

1. **高风险 Shell 执行场景**
    这是 OpenClaw 施加产品现实压力最强的地方。真实世界里的危险命令不是只有 `rm -rf /`，而是混着 heredoc、wrapper、`env`、`bash -c`、变量展开、下载执行、命令混淆一起出现。只靠启发式规则通常能挡住最蠢的攻击，但挡不住最麻烦的变体。

2. **多租户 / 多 Agent 共享宿主机场景**
    当多个 Hands、多个 agent、多个 workflow 共享同一台宿主机时，工具层就不再是“某次调用失败”的问题，而是隔离边界是否真的成立：工作目录、环境变量、挂载路径、审批、能力声明，任何一层漏了，都会让一个 agent 的破坏面扩散到整个系统。

3. **浏览器 / 文件 / 网络混合工具链场景**
    真实自动化任务很少只打一种工具。更常见的是“搜索网页 → 读文件 → 执行 shell → 再写回文件”。这时工具风险会沿着多步链条传播，第三章的价值就在于判断 OpenFang 现在有没有足够强的跨工具污染控制与策略闭环。

4. **跨平台执行场景**
    安全策略如果只在 Unix 心智里成立，就不算真的成立。Windows shell wrapper、环境继承、路径解析和引号语义都会把一套在 macOS/Linux 上看起来正确的策略打穿。所以第三章也天然是“产品现实压力”和“运行时底线压力”同时交汇的一章。

所以，本章真正要回答的是：**当 Agent 从“会思考”跨到“会动手”时，OpenFang 是否真的有能力把破坏面关在可控范围内。**

#### 代码走读 📖

##### 1. `execute_tool()`：OpenFang 的工具总闸门

`tool_runner.rs` 里最重要的入口是 `execute_tool()`（L101），而 shell 污点前置检查 `check_taint_shell_exec()` 在同文件更前面的 L25。它的函数签名已经说明问题：这个入口同时接收 `kernel`、`allowed_tools`、`skill_registry`、`web_ctx`、`browser_ctx`、`allowed_env_vars`、`workspace_root`、`exec_policy`、`docker_config`、`process_manager`。也就是说，**OpenFang 没有把“执行工具”当成一个单纯函数，而是当成一个要借用整套内核上下文的安全边界。**

它的执行顺序可以压缩成 4 层：

| 阶段 | 代码位置 | 作用 |
|------|----------|------|
| 1. Capability 校验 | `execute_tool()` 开头 | 如果 `tool_name` 不在 `allowed_tools` 里，直接拒绝 |
| 2. Approval 校验 | 同函数前半段 | 如果 `kernel.requires_approval(tool_name)` 为真，则请求人工审批 |
| 3. 特定工具安全前置检查 | `web_fetch` / `shell_exec` 分支 | URL 污点检查、shell allowlist、taint 检查 |
| 4. 具体工具实现 | `match tool_name` | 进入文件、网络、shell、agent、memory 等各分支 |

其中第二层审批很实用：OpenFang 不是把完整原始输入发给人，而是构造一条 `summary = format!("{}: {}", tool_name, truncate(input, 200))` 的摘要。这说明系统在审批设计上已经考虑“让人能看懂”，而不是只考虑“技术上可拦截”。

##### 2. `shell_exec`：真正的重点不在 shell，而在 shell 之前的三重安检

在 `execute_tool()` 里，`shell_exec` 分支不是一上来就执行命令，而是先过三道关：

1. `validate_command_allowlist(command, policy)`
2. 检查是否 `ExecSecurityMode::Full`
3. 若不是 Full，则调用 `check_taint_shell_exec(command)`

这里可以看出 OpenFang 的一个关键设计：**`exec_policy` 的模式不仅决定“能不能执行”，还决定“是不是跳过 taint 检查”。** 也就是说，`Full` 模式不是“更宽松一点”，而是明确进入“你自己承担风险”的互信区。

`check_taint_shell_exec()` 的规则很短，但很具体。它会查找这些模式：

- `curl `
- `wget `
- `| sh`
- `| bash`
- `base64 -d`
- `$(curl`
- `` `curl``
- `eval `

然后给该命令打上 `TaintLabel::ExternalNetwork`，再让它去撞 `TaintSink::shell_exec()`。这说明当前 taint 防御更像“**对已知高风险组合做定点打击**”，而不是“理解 shell 语义后再判断风险”。这也是它和 OpenClaw 最大的差距来源。

##### 3. `tool_shell_exec()`：真正执行时的安全细节比表面更务实

`tool_shell_exec()` 在 `tool_runner.rs` L1386 启动，它真正执行前还会依赖 `subprocess_sandbox.rs` 里的 `sandbox_command()`（L40）与 `validate_command_allowlist()`（L142）。这里有几个工程味很重的小设计：

| 细节 | 实现 | 含义 |
|------|------|------|
| 工作目录绑定 | `cmd.current_dir(ws)` | 把生成文件尽量限制在 Agent workspace |
| 环境隔离 | `sandbox_command(&mut cmd, allowed_env)` | 清空继承环境，只回灌白名单变量 |
| Windows shell 适配 | 优先 Git Bash `sh -c`，再回退 `cmd /C` | 避免 `cmd.exe` 的 `%` 展开和引号问题 |
| 输出上限 | `max_output = 100_000` | 防止工具结果把内存和上下文冲爆 |
| 运行超时 | `tokio::time::timeout(...)` | 避免长时间阻塞 |

尤其是 Windows 适配这一点很值得写出来。很多项目会把 shell 执行当成“Unix-only 心智模型”，OpenFang 这里至少意识到了 Windows 上 `cmd.exe` 的引号与 `%` 展开是另一类风险源。

##### 4. `subprocess_sandbox.rs`：最朴素但最容易被忽视的一层

这个文件不是炫技型沙箱，但却非常关键，因为 `shell_exec` 默认最常走到这里。它做了两类事：

**第一类：环境白名单**

`sandbox_command()` 会先 `env_clear()`，再只加回：

- 通用安全变量：`PATH`、`HOME`、`TMPDIR`、`LANG`、`TERM` 等
- Windows 特有安全变量
- 调用方显式允许的 `allowed_env_vars`

这个做法很朴素，但意义很大：**默认不继承宿主环境**，这样即使 Agent 生成了脚本，也不至于天然就能看到所有 API key。

**第二类：执行 allowlist**

`validate_executable_path()` 只做 `..` 拦截，属于基础防线；更关键的是 `validate_command_allowlist()`。它基于 `ExecPolicy` 和 `ExecSecurityMode` 做命令级模式控制，说明 OpenFang 已经意识到“不是所有 shell 都应该一视同仁”。

不过这里也暴露了它的边界：当前还是“按命令字符串判断”，不是“按 AST + argv 结构判断”。

##### 5. `validate_path()` 与 `resolve_file_path()`：这里比当前章节原文更值得较真

`tool_runner.rs` 里最直白的路径防线是 `validate_path()`（L1192）与 `resolve_file_path()`（L1202）：

- `validate_path(path)`：遍历 path components，拒绝 `ParentDir`
- `resolve_file_path(raw_path, workspace_root)`：如果有 workspace sandbox，就交给 `workspace_sandbox::resolve_sandbox_path()`；否则退回 `validate_path()`

这说明当前系统的核心假设是：

1. 有 workspace sandbox 时，把越界控制交给更上层封装
2. 没有 sandbox 时，至少先堵住显式 `..`

但这也意味着它暂时没有在这一层统一做：

- `canonicalize()` 真路径归一化
- symlink 拒绝或展开校验
- hardlink 风险识别

这不是说 OpenFang 完全没有文件边界，而是说**它当前的路径安全更偏“字面路径拦截”，还没有到“真实文件系统拓扑审计”的强度。**

##### 6. `ToolPolicy`：deny-wins、group 展开、depth 惩罚三件套

`tool_policy.rs` 的 `resolve_tool_access()`（L76）是第三章里最容易被低估的函数之一。它的顺序非常明确：

1. 先检查 subagent 深度上限
2. 展开 group，把命中的组名转成 `@group_name`
3. 先跑 agent 级 deny
4. 再跑 global deny
5. 若存在 allow 规则，则必须至少命中一个 allow，否则隐式拒绝
6. 如果压根没有规则，则默认允许

这正是典型的 **deny-wins + explicit-allow + implicit-deny** 模型。

更关键的是深度惩罚模型：

- 所有 `depth > 0` 的 subagent 一律剥夺 `cron_create`、`schedule_delete`、`process_start` 等管理能力
- 当 `depth >= max_depth - 1` 时，还会额外剥夺 `agent_spawn` 和 `agent_kill`

这说明 OpenFang 的策略层不只是“工具能不能用”，还在显式防范深层代理链条继续扩张。这一点跟第二章的递归控制逻辑是呼应的。

##### 7. `shell_bleed.rs`：这是真正有新意的小模块

`shell_bleed.rs` 做的不是拦截命令执行，而是**扫描脚本文件本身会不会把环境变量泄露回 LLM 上下文**。这比普通的命令黑名单更细一层。

它的流程在 `shell_bleed.rs` 的几个关键落点上很清楚：`MAX_SCRIPT_SIZE`（L67）、`extract_script_path()`（L78）、`extract_env_var_refs()`（L183）。整体是：

1. `extract_script_path(command)` 从命令中猜出脚本文件路径
2. 如果文件不可读或超过 `MAX_SCRIPT_SIZE = 100 KB`，直接放弃扫描
3. 跳过注释行
4. 用 `extract_env_var_refs(line)` 识别 `$VAR`、`${VAR}`、`os.environ[...]`、`os.getenv(...)`、`process.env.VAR` 等模式
5. 过滤 `SAFE_VARS` 白名单
6. 对名字里含 `key`、`secret`、`token`、`password`、`credential`、`auth` 的变量发出 `ShellBleedWarning`

这块的设计价值不在“绝对安全”，而在它看到了一个很多框架没看到的问题：**即使 shell 命令本身安全，脚本输出也可能把宿主 secrets 回流给模型。**

##### 8. `WasmSandbox`：指令级 CPU 预算 + 墙钟超时

`sandbox.rs` 用的是 Wasmtime，不是只包一层超时。`SandboxConfig` 结构体在 L35，默认实现 `impl Default for SandboxConfig` 在 L47。它的默认值是：

```rust
SandboxConfig {
    fuel_limit: 1_000_000,
    max_memory_bytes: 16 * 1024 * 1024,
    timeout_secs: None,
    capabilities: Vec::new(),
}
```

真正重要的是执行模型：

- `Config::consume_fuel(true)` 开启指令燃料计费
- `Config::epoch_interruption(true)` 开启 epoch 中断
- `spawn_blocking` 把 CPU 密集型 WASM 从 Tokio executor 上挪开
- `store.set_fuel(config.fuel_limit)` 设置计算预算
- `store.set_epoch_deadline(1)` + watchdog 线程实现墙钟超时
- 不引入 WASI，只通过 `host_call` / `host_log` 暴露能力受控接口

这说明 OpenFang 的 WASM 沙箱不是“仅供演示的插件容器”，而是具备了三个关键的工业要素：

1. 指令级预算
2. 时间级中断
3. capability-based host ABI

##### 9. `docker_sandbox.rs`：Docker 在这里不是默认路径，而是重隔离后备层

`docker_sandbox.rs` 的实现比很多项目认真。`create_sandbox()` 在 L98，`exec_in_sandbox()` 在 L180。`create_sandbox()` 里至少做了这些事情：

- 校验 image 名合法字符
- 容器名只允许字母数字和 `-`
- 设置内存、CPU、PID 上限
- `--cap-drop ALL`
- `--security-opt no-new-privileges`
- 可选 `--cap-add` 白名单回补
- 只读 root filesystem
- 指定网络模式
- tmpfs 挂载
- workspace 只读挂载到容器内工作目录

然后 `exec_in_sandbox()` 再做：

- 命令模式校验，拒绝 `` ` ``、`$(`、`${`
- `docker exec ... sh -c command`
- 超时控制
- stdout/stderr 截断到 `50_000` 字节

这使它更像“隔离执行器”，而不是“帮你随便起个容器”。但也要实话实说：它仍然是把命令作为字符串扔给 `sh -c`，只是先做了少量危险模式过滤。

#### 架构决策档案

**ADR-003：多层闸门优先于单点沙箱**

- **背景**：工具调用的风险来自多个阶段，不只是最终进程落在哪运行
- **决策**：先 capability、再 approval、再 taint、再环境隔离、再具体沙箱
- **正面后果**：即使某一层不完美，仍能靠其他层做补偿
- **负面后果**：用户看到“命令为什么没跑”时，错误可能来自 5 个位置，诊断体验会比较碎

**ADR-004：WASM 走能力受控 ABI，而不是直接开 WASI**

- **背景**：WASI 给得太多，第三方 skill 很容易碰到文件、网络、系统资源
- **决策**：不开放通用 WASI，只暴露 `host_call` / `host_log`
- **正面后果**：权限面更小，host 能逐次 capability 校验
- **负面后果**：插件生态的自由度下降，宿主 ABI 设计压力上升

**ADR-005：Shell Bleed 选择轻量预警，而不是强制阻断**

- **背景**：很多脚本会碰环境变量，但不一定都危险
- **决策**：生成 warning 并前置到结果里，而不是直接拒绝执行
- **正面后果**：误伤少，用户体验柔和
- **负面后果**：对于真正恶意脚本，这不是硬防线，只是提示灯

---


### 3.1.5 源码补充：WASM 沙箱的 Guest ABI 与中断模型
我们在第一章提到过 OpenFang 有个 100% Rust 隔离的 WASM 沙箱。那么它究竟是如何在代码里成立的？
翻开 `crates/openfang-runtime/src/sandbox.rs`：
这里实现了一套极度克制的 **Guest ABI（客户端二进制接口，见[附录 A：术语表与核心概念索引](openfang_tutorial_appendix_a_glossary.md)）**。当 Agent 生成了第三方恶意 WASM 插件时，常规环境可能会被读取整个文件系统。
- 但在 OpenFang 中，WASM 模块被强制且仅仅只能导出：`memory`、`alloc` 和 `execute` 三个基本入口！它想要的所有能力（查文件、发网络）必须通过 Host 层的 `host_call` 进行受控的 RPC 申请。
- 此外，`SandboxConfig` 里有两个关键字段：`fuel_limit_（CPU 指令燃烧预算，见[附录 A：术语表与核心概念索引](openfang_tutorial_appendix_a_glossary.md)）` 和 `timeout_secs`。通过 `wasmtime` 底层引擎，如果恶意插件企图执行 `while true {}` 死循环耗尽宿主资源，OpenFang 会在指令配额耗尽的瞬间发起 **Epoch-based 纪元中断**（见[附录 A：术语表与核心概念索引](openfang_tutorial_appendix_a_glossary.md)），以 OOM/Timeout 判决强制终止该插件，实现可靠的资源隔离。

### 3.1.6 源码补充：Shell Bleed 环境变量泄露扫描
再看另一个多数 Node.js 框架尚未系统性覆盖的盲区：大模型生成的执行脚本如果无意中把高危密码打印了出来怎么办？
翻看 `crates/openfang-runtime/src/shell_bleed.rs`。这是一个低调但关键的安全防线文件；`ShellBleed` 的定义可回看[附录 A：术语表与核心概念索引](openfang_tutorial_appendix_a_glossary.md)。
- 当 Agent 生成并执行一段 `bash run.sh` 时，OpenFang 不会直接抛给 OS。它会先将其送入 `shell_bleed` 检测引擎。
- 引擎内部硬编码了一份详尽的 `SAFE_VARS` 白名单。一旦在脚本文件或执行参数里检测到例如 `OPENAI_API_KEY` 或其它高危模式的变量解引用（如 `$KEY`），系统会触发 `ShellBleedWarning` 阻断，并提示：`"Potential environment variable leak in script"`。这就是 OpenFang 在环境变量泄露层面的纵深防御设计。

### 3.2 压力测试 🧪

#### 思想实验 1：`python3 -c "import os; print(os.environ['SECRET_KEY'])"` 会被 Shell Bleed 抓住吗？

**大概率不会。**

因为 `shell_bleed.rs` 的起点是 `extract_script_path(command)`，它要先从命令里提取出一个脚本文件路径。像 `python3 -c ...` 这种内联代码没有脚本文件，因此会直接返回 `None`，扫描流程根本不会启动。

这说明当前 Shell Bleed 防的是“文件脚本型泄露”，不是“内联代码型泄露”。

#### 思想实验 2：`check_taint_shell_exec()` 能挡住变形载荷吗？

不能太乐观。

它当前本质是对 8 个高风险字符串做 `contains()`。如果攻击者改成：

- 先下载再拼接执行
- 用别名或 wrapper 包裹 `curl`
- 用 heredoc、printf、xxd/od、变量展开改写载荷

那么当前 taint 前置检查很可能就看不出来。这也是为什么 OpenClaw 要走到 AST 和混淆检测那一步。

#### 思想实验 3：路径里没有 `..`，是不是就安全了？

不是。

如果路径经过 symlink、hardlink 或多层真实路径跳转，仅仅检查 `..` 组件并不能证明最终访问目标还在安全边界内。OpenFang 这里的实现更像“第一层字面拦截”，还没到 ZeroClaw 那种系统化 `canonicalize()` + symlink 拒绝级别。

#### 思想实验 4：WASM 死循环会不会拖死 Tokio？

按当前实现，不会直接拖死 Reactor。

因为 `WasmSandbox::execute()` 是 `spawn_blocking`，而不是在 async executor 上直接跑 CPU 密集逻辑；同时它还有 fuel limit 和 epoch interruption 双保险。所以这块的成熟度其实比 shell 防御更高。

#### 已知边界与限制

1. `shell_bleed.rs` 不覆盖 `-c` 内联代码、stdin 管道脚本等场景
2. `check_taint_shell_exec()` 仍是启发式字符串匹配，不是 shell 语义分析
3. 路径安全尚未统一纳入 `canonicalize()` / symlink 审计
4. Docker 沙箱仍通过 `sh -c` 执行命令，字符串风险没有根除
5. `ToolPolicy` 的表达能力强于运行时某些具体拦截点的落地能力

#### 验证资产与质量保障

第三章接下来最该补的，不只是更强的防线，还有**更像安全工程的验证资产**。

1. `ToolPolicy`、taint 预检、`shell_bleed`、workspace 边界、WASM fuel、Docker 隔离已经形成多层控制链，这意味着安全问题已经可以按层回归，而不是只能端到端碰运气。
2. 但真正缺位的验证面也很清楚：内联代码样本库、shell 变形载荷语料、`canonicalize()` / symlink 拓扑测试、容器 argv 模式回归，这些都还没有被系统化固定下来。
3. 这也是为什么第三章和第十章是直接相连的：未来只要谈“系统自动执行修复动作”，它就必须先站在第三章这种可审计、可解释、可回归的执行控制链上，而不是绕过它。

---

### 3.3 改进雷达

#### 🚧 OpenClaw：Shell 解析明显更重

OpenClaw 在 shell 安全上投入得比 OpenFang 更深。根据 `openclaw/src/infra/exec-approvals-analysis.ts` 与 `exec-obfuscation-detect.ts`：

- 有完整的 quote / escape 状态机
- 能拆 `&&`、`||`、`;` 但尽量尊重引号上下文
- 支持 heredoc 解析
- 有 16 类混淆攻击模式检测

因此，第三章里原本“OpenClaw 17 个脏活防线”这类判断方向是对的，甚至还算保守。和它相比，OpenFang 当前更像是：

- 工具级大闸门做得不错
- 但 shell 语义层做得还浅

#### 🔬 ZeroClaw：文件系统防线更严，尤其是 symlink

ZeroClaw 的优势不在 shell AST，而在文件/技能加载路径上更强势地使用：

- `canonicalize()` 真路径归一化
- `is_symlink()` 主动拒绝
- 工作区边界检查

这和 OpenFang 当前以 `validate_path()` + workspace sandbox 为主的做法形成鲜明对比。也就是说，**OpenFang 在“执行环境隔离”上不差，但在“文件系统拓扑安全”上还可以更硬。**

#### ⚠️ 差距清单

1. **Shell 解析深度不足**：仍停留在模式匹配，而不是 AST 解析
2. **内联代码泄露盲区**：`shell_bleed` 无法看见 `-c`、管道脚本、here-string
3. **路径真实边界校验不足**：未统一做 canonicalize / symlink 防御
4. **策略与执行未完全闭环**：策略层可表达的风险控制，比具体拦截点更丰富
5. **Docker 命令仍是字符串执行**：虽然周边安全很多，但底层仍依赖 shell 解释

---

### 3.4 行动项

| 改进项 | 影响力 | 难度 | 代码位置 |
|--------|-------|------|---------|
| **把 `check_taint_shell_exec()` 升级为 AST 级分析**<br>至少先支持 heredoc、wrapper chain、argv 级拆解，再谈真正精细的 shell 风险控制。 | ⭐⭐⭐⭐⭐ | 高 | `tool_runner.rs` `check_taint_shell_exec()`（L25） |
| **把 `shell_bleed` 扩展到内联代码与 stdin 场景**<br>不仅扫脚本文件，还要扫 `python -c`、`node -e`、`bash -c` 的代码字符串。 | ⭐⭐⭐⭐ | 中 | `shell_bleed.rs` `extract_script_path()`（L78）与外层调用点 |
| **统一路径真实边界检查**<br>在文件读写、工具执行、技能加载路径上统一引入 `canonicalize()`、`symlink_metadata()` 与边界回验。 | ⭐⭐⭐⭐⭐ | 中高 | `tool_runner.rs`、workspace sandbox、技能加载路径 |
| **把 `ToolPolicy` 下沉到更细粒度命令策略**<br>不仅控制 `shell_exec`，还应控制具体二进制、参数类别、网络目标。 | ⭐⭐⭐⭐ | 高 | `tool_policy.rs` + `subprocess_sandbox.rs` |
| **为 Docker 执行引入 argv 模式而非 `sh -c` 字符串**<br>减少字符串解释层，把命令拆成参数数组传入容器。 | ⭐⭐⭐ | 中 | `docker_sandbox.rs` `exec_in_sandbox()`（L180） |
| **把 shell 拦截结果结构化**<br>将“命中了哪条规则、在哪一层被拒绝”暴露给 API/TUI，减少调试黑箱感。 | ⭐⭐⭐ | 低 | `tool_runner.rs` 的错误返回结构 |

---

### 3.5 交叉引用导读 🔗

- 如果你想看这些安全边界在主循环里怎样与容错、续尾和压缩机制相互咬合，可回看 [openfang_tutorial_chapter_2.md](./openfang_tutorial_chapter_2.md)。
- 如果你想理解被拒绝的执行最终如何进入审计、预算和治理链，建议连读 [openfang_tutorial_chapter_5.md](./openfang_tutorial_chapter_5.md)。
- 如果你更关注用户如何在 CLI/TUI 中感知这套安全体系的存在，可继续读 [openfang_tutorial_chapter_9.md](./openfang_tutorial_chapter_9.md)。

### 3.6 本章小结

如果把第三章放回整本教程的新框架里，结论其实非常清楚：

- **从产品现实压力看**，工具层永远是 OpenFang 最容易被真实世界打穿的地方，因为 shell、文件、网络、浏览器和外部进程不会按理想模型合作。
- **从运行时底线压力看**，OpenFang 已经有了比普通 Agent 框架更完整的一串闸门：`ToolPolicy`、approval、taint、workspace 边界、`shell_bleed`、WASM fuel、Docker 重隔离，这说明它不是在把“安全”寄托给单点沙箱奇迹。
- **从源码现状看**，真正还没五星的不是“有没有防线”，而是防线之间仍有几处薄弱连接：shell 解析深度不够、内联代码盲区、真实路径边界不够硬、Docker 仍靠字符串 shell 执行。
- **从下一步演进看**，第三章最值得补的不是再多列几个风险点，而是把 shell 语义分析、路径真实边界、命令策略颗粒度和结构化拒绝原因真正闭成统一执行防线。

因此，本章的最终结论不是“OpenFang 有四层沙箱”，而是：**OpenFang 已经搭起了多层工具安全骨架，但距离五星标准，还差把这些骨架从并列防线进一步闭成真正可审计、可解释、可跨平台成立的执行控制链。**
