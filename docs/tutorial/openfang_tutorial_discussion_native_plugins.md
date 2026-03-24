# 实现方案：OpenFang 如何支持 OpenClaw 官方插件

> 这篇文档不再以“讨论”为主，而是直接回答一个实现问题：
> **OpenFang 如果要支持 OpenClaw 官方插件，需要先支持什么、宿主要补什么、第一阶段不支持什么。**

这里的重点不是追求“最原生的 JS 执行器”，而是先把 **OpenClaw 官方插件合同** 接进 OpenFang。只要合同接不住，插件就谈不上真正支持。

关于 OpenClaw 插件本身的接口定义、运行时宿主环境与装载语义，本文不再重复作为主说明展开，统一参考 OpenClaw 教程的 [第十一章：插件系统：官方合同、宿主接口与微内核装配线](../openclaw_tutorial/openclaw_tutorial_chapter_11.md)。

如果要找一份足够接近真实兼容压力的参考样例，优先看该章里的复杂插件样例与逐项拆解。那一段比最小插件例子更适合作为 OpenFang 的兼容验收基线，因为它同时覆盖了配置、日志、runtime、tool、command、hook、service 这几类能力。

---

## 本文现在真正解决什么问题？

如果只把这篇文档写成“Rust 怎么跑 JS”，价值其实不大。当前真正有价值的问题是：

**OpenFang 要兼容的到底是什么，以及这种兼容应该被放在系统的哪个位置？**

因为这里牵扯的不是单纯性能，而是 5 个要同时满足的目标：

1. **兼容 OpenClaw 生态的速度**：能不能尽量少改现有 JS/TS 插件就跑起来。
2. **宿主安全边界**：插件能看到什么、能碰什么、崩了以后能炸到哪里。
3. **运行时成本**：启动延迟、内存占用、序列化开销、并发隔离成本。
4. **生态语义一致性**：project-wins、schema 校验、slot arbitration、diagnostics 这些 OpenClaw 生态行为，要不要保留。
5. **长期演进性**：未来是只兼容 JS，还是把 Python / Go / WASM 也一起纳进来。

所以这篇文档不是为了追求“最酷的技术栈”，而是为了给 OpenFang 定一份**插件支持实现范围**。

### 关键场景

真正需要面对的，是下面 5 类场景：

1. **快速收编 OpenClaw 现有插件场景**
   如果目标是先兼容已有 JS/TS 插件，最重要的是迁移阻力，而不是理论最优性能。

2. **高风险第三方插件市场场景**
   一旦 OpenFang 真开放 FangHub 或扩展市场，插件就不再是“自己写的脚本”，而是可能来自陌生开发者。此时安全边界优先级会瞬间压过性能。

3. **高并发、短生命周期工具调用场景**
   如果每次 agent tool call 都要临时拉起插件执行环境，启动成本和回收成本就会变成主导指标。

4. **深宿主扩展场景**
   一旦插件开始触碰 route、gateway、memory provider、channel adapter，问题就不再是“跑不跑得起来”，而是“该不该让它碰这些边界”。

5. **长期生态统一场景**
   如果 OpenFang 未来不想只服务 JS，而想把插件层统一抽象成更宽的执行接口，那“JS 原生兼容”就不应成为唯一设计目标。

---

## 先把对象说清楚：OpenFang 在参考哪一份 OpenClaw 插件定义？

这一点先固定：

**OpenFang 兼容 OpenClaw 插件时，关于“插件是什么、插件能调什么接口、插件运行时看到什么宿主环境”，统一以 OpenClaw 教程第十一章为参考基线。**

因此本文不再试图成为第二份 OpenClaw 插件说明书，而是只回答两个问题：

1. OpenFang 打算兼容那一章里的哪些内容
2. OpenFang 需要怎样映射那一章定义的宿主接口与治理语义

从兼容实施角度看，OpenClaw 第十一章中的两个样例承担不同角色：

1. 最小插件样例：用来验证 OpenFang 是否吃得下官方合同
2. 复杂插件样例：用来验证 OpenFang 是否真的提供了足够的宿主接口与生命周期支持

## OpenClaw 插件生态到底包含什么？

如果只把 OpenClaw 插件理解为“给 Agent 多几个 tool”，后面所有兼容设计都会失真。根据 OpenClaw 官方插件文档以及 `openclaw/src/plugins/registry.ts`、`openclaw/src/plugin-sdk/index.ts`，它实际上是一套已经成形的微内核插件生态。

### 0. 先把官方合同说清楚：OpenClaw 插件不是“任意 JS 包”

从官方文档看，OpenClaw 插件机制至少包含下面这些刚性约束：

1. **插件必须带 `openclaw.plugin.json`**，配置 schema、uiHints、skills 目录等元数据都从这里进入宿主。
2. **插件可以导出函数或对象**：要么直接导出 `(api) => {}`，要么导出 `{ id, name, configSchema, register(api) {} }` 这一类对象形态。
3. **插件目录不等于单入口**：一个包可以通过 `package.json.openclaw.extensions` 暴露多个扩展入口，形成一个 package pack。
4. **插件不是只注册 tools**：官方支持 RPC methods、HTTP routes、tools、CLI commands、background services、context engines、skills、providers、channels、auto-reply commands。
5. **插件是 in-process trusted code**：OpenClaw 设计里，插件默认与 Gateway 同进程加载，因此“插件能做什么”在官方语义里本来就是宿主级能力问题，而不是沙箱默认问题。

这 5 条很关键，因为它们说明用户说的“让 OpenFang 直接支持 OpenClaw 插件”，实际目标不是“能 import 某些 JS 文件”，而是：

**让 OpenFang 能识别、裁决、配置并承接这套官方插件合同。**

### 1. 它兼容的不是单一工具点，而是多维扩展点

当前 `PluginRegistry` 的正式落点至少包括：

1. `tools`
2. `hooks`
3. `channels`
4. `providers`
5. `gatewayHandlers`
6. `httpRoutes`
7. `cliRegistrars`
8. `services`
9. `commands`
10. `diagnostics`
11. `plugins` 元记录本身

这意味着 OpenClaw 插件的语义单位不是一个函数，而是一个可以挂载到不同系统边界位置的扩展包。

### 2. 它有自己的 SDK 与宿主假设

`openclaw/plugin-sdk` 暴露出来的并不只是 `registerTool()`。官方文档已经把 SDK 入口拆成 `openclaw/plugin-sdk/core`、`openclaw/plugin-sdk/compat` 以及 channel-specific subpaths，同时保留旧的 `openclaw/plugin-sdk` 单体入口兼容。

这意味着 OpenFang 若想“直接支持 OpenClaw 插件”，要兼容的并不是一个模糊的 npm 包导入点，而是一组**可被插件源码真实 import 的宿主模块路径**。

除此之外，它同时把 channel adapter、gateway handler、provider auth、webhook、group policy、pairing、command auth、thread binding、runtime group policy 等一整套宿主能力面导给插件作者。

这件事很关键，因为它说明 OpenClaw 生态不是“任意 JS 包都算插件”，而是**依赖 OpenClaw 宿主 API 的 JS/TS 包生态**。

### 2.5 先把接口面说具体：插件到底拿到什么 `api`？

如果后面要谈“支持 OpenClaw 插件”，首先要知道插件 register 时实际拿到什么。按当前实现，宿主传给插件的 `api` 至少包含下面几类内容：

1. **插件自身元信息**：`id`、`name`、`version`、`description`、`source`
2. **宿主配置面**：`config`、`pluginConfig`
3. **运行时帮助器**：`runtime`
4. **日志能力**：`logger`
5. **注册接口**：`registerTool`、`registerHook`、`registerHttpHandler`、`registerHttpRoute`、`registerChannel`、`registerProvider`、`registerGatewayMethod`、`registerCli`、`registerService`、`registerCommand`
6. **辅助接口**：`resolvePath`、`on(...)`

这意味着 OpenFang 未来若声称“支持官方插件”，最少不能只模拟 `registerTool()`；它至少要回答：

1. 哪些 `register*` 接口第一阶段真的可用
2. 哪些接口会被降级成受控版本
3. 哪些接口即使存在同名兼容层，也必须明确返回“不支持”

### 2.6 再把外部环境说具体：插件运行时能看到什么宿主能力？

除了 `register*` 接口，OpenClaw 还通过 `api.runtime` 给插件暴露了一组已经分好命名空间的宿主环境。这个点非常关键，因为它决定了“插件系统的外部环境”到底是什么，而不是一句模糊的“宿主能力”。

按当前 runtime 类型定义，插件至少能看到这些命名空间：

1. `runtime.version`
2. `runtime.config`
3. `runtime.system`
4. `runtime.media`
5. `runtime.tts`
6. `runtime.tools`
7. `runtime.channel`
8. `runtime.logging`
9. `runtime.state`

可以把它们理解成下面这张更容易落地的宿主环境表：

| 运行时命名空间 | OpenClaw 暴露的外部环境 | 对 OpenFang 的意义 |
|------|------|------|
| `config` | 读取/写入配置 | 说明插件不只是读自己 config，还可能触碰宿主配置流程 |
| `system` | 系统事件、带超时的命令执行、原生依赖提示 | 说明插件可触达系统级执行面，必须纳入安全边界设计 |
| `media` | 远程媒体读取、MIME 检测、图片处理、音频兼容性判断 | 说明插件可直接依赖宿主媒体处理能力 |
| `tts` | Telephony TTS helper | 说明有些插件依赖的是宿主能力，而不是自己内置所有逻辑 |
| `tools` | Memory tool factory、memory CLI 注册 | 说明插件可以复用宿主已有工具资产 |
| `channel` | 文本切分、reply dispatch、routing、pairing、session、mentions、reactions、group policy、debounce、commands，以及 Discord/Slack/Telegram/Signal/iMessage/WhatsApp/Line 等 channel helper | 说明很多插件默认假定自己运行在一个已经有成熟消息通道基础设施的宿主里 |
| `logging` | verbose 判断、child logger | 说明插件日志本身也是宿主治理的一部分 |
| `state` | state dir 解析 | 说明插件默认可以使用宿主的状态目录约定 |

这里最值得强调的不是“功能很多”，而是：

**OpenClaw 插件的外部环境并不只是一个 `plugin-sdk` 包，而是一整套已经被宿主整理好的 runtime helper surface。**

所以 OpenFang 后面的兼容工作至少要分成两层：

1. **注册接口兼容**：插件能不能把自己的能力挂进宿主
2. **运行时环境兼容**：插件需要的宿主 helper 能不能被提供出来

如果只做前者，不做后者，那么大量真实插件即使能被加载，也会因为 `api.runtime` 缺面而无法真正工作。

### 3. 它有一整套装载与裁决语义

OpenClaw 当前已经稳定形成了这些装载语义：

1. **固定 discovery precedence**：`plugins.load.paths` -> workspace extensions -> global extensions -> bundled extensions
2. **Project-Wins / source precedence**：同身份包冲突时，高优先级来源先赢
3. **边界文件读取**：装载入口必须留在插件 root 内
4. **格式兼容**：TS / ESM / CJS 混部交给 `jiti`
5. **Config Schema 校验**：不合格配置直接进入 error
6. **allow / deny / provenance**：插件来源、加载路径、启用状态、安装来源都是可裁决对象
7. **单槽位裁决**：如 `memory`、`contextEngine` 这类独占槽位必须先选主
8. **严格未知项错误**：未知 plugin id、未知 channel key、未知 slot 都不是 warning，而是 error
9. **生命周期分层**：`loaded` / `disabled` / `error`
10. **诊断可见性**：冲突、kind mismatch、id mismatch 都会留下 diagnostics

OpenFang 如果未来只说“可以运行 OpenClaw 插件”，却不兼容这些语义，最终得到的更像“能执行一些 JS 包”，而不是“兼容 OpenClaw 插件生态”。

---

## 真正的兼容目标：OpenFang 到底要兼容到哪一层？

这一步必须先下定义，不然后面会把“导入包”误写成“生态兼容完成”。

更合理的兼容目标至少应当拆成四层：

### Level 0：包发现与元数据兼容

最弱的一层兼容，是让 OpenFang 识别 OpenClaw 官方插件包结构，读出：

1. 插件 id / name / version
2. 插件 kind
3. config schema
4. `openclaw.plugin.json` 中的 uiHints / skills / manifest metadata
5. `package.json.openclaw.extensions` 中的 package-pack 入口
6. 入口文件与导出形态
7. 运行时类型

这层只解决“能不能看懂它是什么”，还没有开始运行。

### Level 1：执行兼容

第二层是把插件真正跑起来，不管是：

1. Node 子进程
2. 托管 Node worker
3. 宿主内 JS runtime

这层解决的是“能不能执行”，但仍不等于“系统语义兼容”。

### Level 2：宿主 API 兼容

第三层才是最难的：OpenFang 要不要给插件一个足够接近 `openclaw/plugin-sdk` 的 API 面？

如果答案是否，那么它最多只能兼容一小批“只注册 tool”的插件。

如果答案是是，就必须回答：

1. 哪些 API 可以原样模拟
2. 哪些 API 只能降级映射
3. 哪些 API 根本不该开放
4. `openclaw/plugin-sdk` 与其 subpaths 要不要都模拟

### Level 3：生态语义兼容

最强的一层兼容，不是“插件能跑”，而是以下这些语义也成立：

1. project-wins
2. schema 校验
3. enabled / disabled / error 状态
4. diagnostics
5. 单槽位裁决
6. 安装与升级后的行为一致性
7. discovery precedence、allow / deny、install provenance 行为一致

只有到了这一层，才接近“OpenFang 兼容 OpenClaw 插件生态”这句话本来的含义。

---

## 先看 OpenFang 当前已经站在哪条线上

如果回到源码层，OpenFang 今天对“插件 / 扩展执行”的现实落点其实相当明确：它已经是一个**异构执行外壳**，但还不是一个“OpenClaw 插件生态兼容层”。

### 1. OpenClaw 兼容当前主要落在 Node 子进程层

`openfang-skills/src/openclaw_compat.rs` 已经能识别带 `package.json`、`index.js` / `dist/index.js` 的 OpenClaw 技能目录，并把它们转换成 OpenFang 的 `SkillManifest`，且运行时会直接标成 `SkillRuntime::Node`。这说明 OpenFang 对 OpenClaw 生态的第一选择不是“重写插件”，而是**先收编包形态与入口约定**。

但要注意，当前这层更接近“skill 级包兼容”，还不是对 OpenClaw 官方插件合同的完整兼容，因为它还没有把 `openclaw.plugin.json`、`package.json.openclaw.extensions`、官方导出形态、slot 与配置布局一起纳入导入语义。

进一步看 `openfang-skills/src/loader.rs`，Node 技能执行实际是 `tokio::process::Command::new(node)` 拉起子进程，通过 stdin / stdout 交换 JSON，同时会 `env_clear()`，只保留最小必要环境变量。这一层已经体现出明确的现实判断：

1. 兼容现有 JS/TS 生态，比“立即做宿主内执行”更优先。
2. 当前的安全边界主要靠**进程隔离 + 环境净化**，不是靠宿主内 capability isolate。

### 2. OpenFang 已经有几块能接住兼容层的宿主落点

如果从 workspace 分工看，未来的 OpenClaw 兼容大概率不会只落在一个 crate：

1. `openfang-skills`：最接近当前 Node skill / OpenClaw compat 入口
2. `openfang-extensions`：最接近“安装、注册、凭证、市场化分发”
3. `openfang-kernel`：最接近 hooks、service、background、workflow、approval 这类宿主能力面
4. `openfang-api` / `openfang-channels` / `openfang-cli`：分别对应 route、channel、command 等外层接入位

这意味着 OpenFang 若要兼容 OpenClaw 插件生态，最终会是一个**跨 crate 的兼容层设计**，而不是只在技能加载器里多写一个 `if runtime == node`。

### 3. OpenFang 已经明确知道 Node 运行时的风险边界

`openfang-skills/src/verify.rs` 会对 `SkillRuntime::Node` 直接给出 “Node.js runtime has broad filesystem and network access” 的告警。这一点很关键，因为它说明 OpenFang 并没有把 Node 兼容误当成“已经安全解决”，而是明确把它定义为一种**高兼容、但边界偏宽**的现实方案。

此外，`openfang-kernel/src/whatsapp_gateway.rs` 也在用托管 Node 进程来承接 WhatsApp Web gateway，并负责安装依赖、拉起 `node index.js`、崩溃重启。这进一步证明：OpenFang 当前不是拒绝 Node，而是在**受控地使用 Node 作为现实生态桥**。

### 4. 宿主内可控执行，当前真正成熟的是 WASM 这条线

OpenFang 的 `openfang-runtime/src/sandbox.rs` 已经不是概念设计，而是一个基于 Wasmtime 的 deny-by-default WASM sandbox：默认不给文件系统、网络、凭证访问，所有 host call 都要走 capability 检查。更重要的是，`openfang-kernel/tests/wasm_agent_integration_test.rs` 已经覆盖了真实 WASM agent 的执行链。

也就是说，OpenFang 现在真正“宿主内 + 能力边界优先”的执行基础，并不在 JS 原生插件，而在 WASM agent 路线上。

### 5. 现状还没有进入“OpenClaw 插件语义兼容”阶段

这点也要写死，不然结论会飘：当前工作区里能看到 Wasmtime、Node 子进程兼容、OpenClaw skill conversion、托管 Node gateway，但看不到一个正式的：

1. `openclaw.plugin.json` manifest importer
2. `package.json.openclaw.extensions` package-pack importer
3. OpenClaw plugin registry mirror
4. discovery precedence + allow / deny + provenance 裁决层
5. OpenClaw plugin diagnostics surface
6. 对应 `openclaw/plugin-sdk` 与其 subpaths 的 OpenFang 兼容 API

甚至在 `openfang-skills/src/loader.rs` 里，`SkillRuntime::Wasm` 对技能层仍然明确返回 `WASM skill runtime not yet implemented`。这说明 OpenFang 虽然已经有 WASM agent 执行能力，但“技能市场层的统一原生 runtime”本身还没有完全收口。

所以，OpenFang 今天离“能跑一部分 OpenClaw JS 包”更近，离“兼容 OpenClaw 官方插件机制”还有一个完整中间层要补。

---

## 技术方案总览：在现有基础上，OpenFang 应该怎么兼容 OpenClaw 插件 API？

如果把目标说得更工程一些，可以把方案压成一句话：

**先用 Node bridge 承接 OpenClaw 插件执行，再在 OpenFang 内部补一层“插件宿主兼容层”，把 OpenClaw 的 `api` 和 `api.runtime` 映射到 OpenFang 现有 crate 的能力面上。**

这条路线不是拍脑袋选出来的，而是直接顺着 OpenFang 现有代码长出来的：

1. `openfang-skills` 已经能识别并执行 Node 技能。
2. `openfang-kernel` 已经在现实场景里托管 Node 进程。
3. `openfang-runtime` 已经有 deny-by-default 的 capability runtime，可以作为长期收口方向。
4. `openfang-kernel` 的 `EventBus`、`CapabilityManager`、config reload 机制、`openfang-memory` 的统一 substrate，已经提供了宿主治理所需的几块基础骨架。

所以最现实的技术路线不是“重写 OpenClaw 插件”，而是补 5 层兼容结构：

1. **合同导入层**：识别 manifest、package pack、入口与导出形态。
2. **SDK 兼容层**：在 Node 侧提供 `openclaw/plugin-sdk` 兼容入口。
3. **宿主 API 映射层**：把 `register*` 和 `api.runtime.*` 映射到 OpenFang crate。
4. **治理层**：状态、诊断、allow/deny、config schema、slot 仲裁。
5. **执行层**：短期 Node bridge，中期受控内嵌，长期 capability runtime 收口。

## 第一层：合同导入器，不先执行，先把插件“看懂”

这层最适合落在 `openfang-extensions` + `openfang-skills` 的交界处。它至少要做 6 件事：

1. 读取 `openclaw.plugin.json`
2. 解析 `package.json.openclaw.extensions`
3. 识别函数式导出 / 对象式导出
4. 生成 package pack 的多入口记录
5. 提取 config schema、uiHints、skills metadata
6. 生成导入诊断：缺 manifest、入口越界、未知扩展点、schema 缺失、import path 需求

这一层的产物不应只是“能不能运行”，而应是一个正式的 `PluginCandidate` / `PluginImportReport`。也就是说，给一个 OpenClaw 插件目录，OpenFang 先能回答：

1. 这个插件有没有满足官方合同
2. 它需要哪些宿主 API
3. 它声明了哪些扩展点
4. 它在哪一层被拒绝或放行

## 第二层：不要先做 Rust 原生 SDK，先做 Node 侧 shim

OpenClaw 插件源码真实依赖的是：

1. `openclaw/plugin-sdk`
2. `openclaw/plugin-sdk/core`
3. `openclaw/plugin-sdk/compat`
4. 若干 channel-specific subpaths

在现有基础上，OpenFang 最现实的做法不是立刻在 Rust 里重做整套 JS 运行时，而是先提供一个 **Node 侧 shim 包**，例如：

1. `@openfang/openclaw-plugin-sdk`
2. 通过 alias / loader / `NODE_PATH` / 运行目录注入，把 `openclaw/plugin-sdk*` 解析到 OpenFang shim
3. shim 内部把 `register*` 调用收集成声明，把 `runtime.*` 调用桥接到 OpenFang host

这一步的关键价值是：

1. 让现有 OpenClaw 插件源码尽量少改甚至不改
2. 把兼容复杂度集中在宿主桥接层，而不是分散到每个插件里
3. 让 OpenFang 能逐步补齐接口，而不是一次性做完全部 JS 宿主面

## 第三层：`register*` API 在 OpenFang 中如何落地？

这一层是整个方案的核心。OpenClaw 插件 `api` 暴露的注册接口，不能笼统地说“兼容”或“不兼容”，而是必须逐项映射到 OpenFang 现有 crate。

| OpenClaw API | OpenFang 当前最接近的落点 | 兼容策略 |
|------|------|------|
| `registerTool` | `openfang-skills` / tool runtime | **优先支持**。先映射为 OpenFang skill/tool registration，是 MVP 的核心能力。 |
| `registerCommand` | `openfang-api` command surface / `openfang-channels` command handling / `openfang-cli` | **受限支持**。先支持聊天命令或 API 命令面，不要求第一阶段完整复制所有 CLI 交互。 |
| `registerHook` / `on(...)` | `openfang-kernel` `EventBus` | **部分支持**。先提供事件订阅桥，把 OpenClaw hook 事件映射到 OpenFang event bus，未映射事件显式报不支持。 |
| `registerService` | `openfang-kernel` + `openfang-hands` 后台任务面 | **优先支持**。映射为受控后台任务，纳入生命周期、日志和停止回调管理。 |
| `registerCli` | `openfang-cli` | **延后支持**。第一阶段不要求完整兼容，除非某插件强依赖管理命令。 |
| `registerChannel` | `openfang-channels` | **后置支持**。结构上可映射，但宿主成本高，不应进入 MVP。 |
| `registerProvider` | provider catalog / auth profile / provider URL reload | **谨慎支持**。先只做 provider metadata / auth adapter 映射，不直接开放深宿主认证与模型调度插桩。 |
| `registerGatewayMethod` | `openfang-api` / transport edge | **默认不支持**。这是深宿主能力，应显式拒绝或只对白名单方法开放。 |
| `registerHttpHandler` / `registerHttpRoute` | `openfang-api` HTTP surface | **默认不支持**。第一阶段不开放动态 HTTP route 注入。 |

这张表背后的原则非常明确：

1. **tool / service / hook / command** 是最接近 OpenFang 现有骨架的。
2. **channel / provider / gateway / http** 是需要更深宿主治理的。
3. 兼容优先级不应按“OpenClaw 有没有这个接口”来定，而应按“OpenFang 现在有没有稳定承接面”来定。

## 第四层：`api.runtime` 在 OpenFang 中如何落地？

真正难的部分其实不只是 `register*`，而是插件运行时默认看到的那一大块 `api.runtime`。如果不把这层拆清楚，很多插件会出现“注册成功但实际跑不起来”的假兼容。

| OpenClaw runtime | OpenFang 当前基础 | 兼容策略 |
|------|------|------|
| `runtime.config` | `openfang-kernel` config + config reload | **部分支持**。先支持只读加载和插件命名空间配置读取；写配置应受控，不直接开放任意持久化写回。 |
| `runtime.system` | `openfang-skills` Node 子进程执行、`CapabilityManager`、WASM host-call 模型 | **部分支持**。`runCommandWithTimeout` 可通过受控 shell/tool execution 暴露，但必须挂 capability 审批。 |
| `runtime.media` | OpenFang 现有媒体/附件处理链，必要时由宿主桥补充 | **渐进支持**。先补最常用的拉取媒体、MIME 检测、基础图像处理接口。 |
| `runtime.tts` | 当前无明确 1:1 宿主面 | **暂不支持或单独桥接**。若插件强依赖，需独立实现 telephony/TTS bridge。 |
| `runtime.tools` | `openfang-memory` substrate + kernel tool registry | **部分支持**。先映射 memory get/search 一类高价值 helper。 |
| `runtime.channel.*` | `openfang-channels` + `openfang-api` + kernel channel reload | **最复杂**。第一阶段只应提供极少数通用 helper，不应直接承诺 Discord/Telegram/WhatsApp 全套 helper。 |
| `runtime.logging` | `tracing` / host logger hierarchy | **优先支持**。很容易桥接，而且对诊断至关重要。 |
| `runtime.state` | OpenFang home/data dir + state path conventions | **优先支持**。提供插件状态目录解析是低成本高收益能力。 |

这里有个容易被低估的现实点：

**OpenClaw `api.runtime` 的体量远大于 MVP 所需体量。**

因此 OpenFang 不应一上来就承诺“全部 runtime helper 兼容”，而应区分三档：

1. **必须先支持**：`logging`、`state`、基础 `config`、一部分 `system`、一部分 `tools`
2. **可渐进支持**：`media`、少量通用 `channel` helper
3. **暂不支持**：大部分 channel-specific helper、完整 TTS、深 provider auth helper

## 第五层：治理语义必须在 Rust 宿主里，而不是只靠 Node shim

OpenClaw 插件的真正难点不在 JS 代码执行，而在宿主治理。这里必须明确：

**allow / deny / precedence / diagnostics / slot arbitration 这些能力，应该在 OpenFang Rust 宿主侧实现，而不是寄希望于 Node shim 自己维护。**

基于当前代码，最接近的承接面是：

1. `openfang-kernel` 的 config reload / extensions reload 机制：承接启停、重载与 restart-required 判断
2. `openfang-kernel` 的 `EventBus`：承接 hook / lifecycle signal
3. `openfang-kernel` 的 `CapabilityManager`：承接 `runtime.system` 等危险宿主能力的准入
4. `openfang-memory` 的统一 substrate：承接 memory slot 的长期宿主实现

这一层应新增的不是“再写一个加载器”，而是一个正式的 `PluginHostRegistry`，至少管理：

1. discovered / validated / loaded / disabled / error 状态
2. source precedence
3. allow / deny
4. 配置校验结果
5. unsupported feature diagnostics
6. 单槽位选主结果

## 建议的兼容架构：按现有 crate 拆分职责

为了避免方案继续停留在抽象层，下面给一版按 crate 划分的建议：

1. `openfang-extensions`
   负责插件发现、安装、manifest 读取、package-pack 解析、来源记录。

2. `openfang-skills`
   负责 Node bridge 执行器、OpenClaw JS 插件入口启动、stdin/stdout 或 RPC 桥。

3. `openfang-kernel`
   负责 `PluginHostRegistry`、状态机、诊断、事件总线、后台 service 生命周期、capability 审批。

4. `openfang-api`
   负责受控 command surface，以及未来若有必要的极少数 gateway/RPC 映射入口。

5. `openfang-channels`
   负责 channel plugin 的后置映射，不进入 MVP 主线。

6. `openfang-memory`
   负责 memory / contextEngine 这类独占 slot 的长期宿主实现；第一阶段只作为 runtime helper 来源，不开放 1:1 slot 兼容。

7. `openfang-runtime`
   负责长期 capability runtime 收口；短期不替代 Node bridge，但为中长期迁移提供 ABI 模型。

## MVP 应该怎么定义，才不会把方案做散？

如果站在“基于现有基础尽快兼容”的角度，一个现实的 MVP 应该限定为：

1. 能识别 `openclaw.plugin.json` 与 `package.json.openclaw.extensions`
2. 能启动 Node 插件，并解析函数式 / 对象式导出
3. 能提供 shim 版 `openclaw/plugin-sdk`
4. 能支持 `registerTool`、`registerService`、部分 `registerHook`、部分 `registerCommand`
5. 能提供 `logger`、`pluginConfig`、`runtime.state`、基础 `runtime.system`、基础 `runtime.tools`
6. 能输出 discovered / validated / loaded / disabled / error 与对应 diagnostics

MVP 明确不做：

1. `registerHttpRoute` / `registerHttpHandler`
2. `registerGatewayMethod`
3. 完整 `registerChannel`
4. 完整 `registerProvider`
5. `memory` / `contextEngine` slot 的 1:1 兼容
6. 大部分 channel-specific runtime helper

## 字段级兼容清单：先回答 OpenClaw 插件拿到的每个 `api` 字段，在 OpenFang 里怎么处理

上面的分层方案已经说明了方向，但如果要真正进入实现阶段，还必须再往下压一层：**把 OpenClaw 插件 `register(api)` 时拿到的每个字段逐项列出来，分别标成“直接支持 / 适配支持 / 暂不支持”。**

否则“兼容 OpenClaw 插件”这句话在工程上仍然过于宽泛，最后很容易出现：

1. manifest 能读
2. 插件也能启动
3. 但实际一进 `register(api)` 就发现缺字段、缺 helper、缺事件面

所以这里先给一版 MVP 视角下的字段级清单。

### 1. `api` 顶层字段兼容清单

| OpenClaw `api` 字段 | 语义 | OpenFang MVP 判断 | 建议落点 |
|------|------|------|------|
| `id` | 插件最终身份 | **直接支持** | `openfang-kernel::plugin_host::PluginDescriptor` |
| `name` | 展示名 | **直接支持** | `PluginDescriptor` |
| `version` | 插件版本 | **直接支持** | `PluginDescriptor` |
| `description` | 展示描述 | **直接支持** | `PluginDescriptor` |
| `source` | 来源路径 / 来源类型 | **直接支持** | `PluginDescriptor` + import report |
| `config` | 宿主全局配置视图 | **适配支持** | 只读 `HostConfigView`，不要直接把 OpenFang 内部配置结构裸露给插件 |
| `pluginConfig` | 插件命名空间配置 | **直接支持** | `PluginResolvedConfig` |
| `runtime` | 宿主 helper 集合 | **适配支持** | `PluginRuntimeBridge` |
| `logger` | 插件日志器 | **直接支持** | `PluginLoggerBridge` |
| `registerTool` | 注册工具 | **直接支持** | `ToolRegistrationSink` |
| `registerHook` | 注册通用 hook | **适配支持** | `HookRegistrationSink` + `EventBus` 映射 |
| `registerHttpHandler` | 注册 HTTP handler | **暂不支持** | 显式 diagnostic，MVP 不开放 |
| `registerHttpRoute` | 注册 HTTP route | **暂不支持** | 显式 diagnostic，MVP 不开放 |
| `registerChannel` | 注册 channel 插件 | **暂不支持** | 后置能力，不进 MVP |
| `registerGatewayMethod` | 注册网关方法 | **暂不支持** | 后置能力，不进 MVP |
| `registerCli` | 注册 CLI 命令 | **延后支持** | 第二阶段再接 `openfang-cli` |
| `registerService` | 注册后台服务 | **直接支持** | `ServiceRegistrationSink` + kernel lifecycle |
| `registerProvider` | 注册 provider | **暂不支持** | 先不把 provider 扩展引入 MVP |
| `registerCommand` | 注册聊天/命令面命令 | **适配支持** | `CommandRegistrationSink` |
| `resolvePath` | 解析插件根目录内路径 | **直接支持** | `PluginPathResolver` |
| `on(...)` | typed lifecycle hook | **适配支持** | `TypedHookRegistrationSink` + hook name mapper |

这里有 3 个实现原则必须先写死：

1. **`config` 必须是视图，不是内核结构体直通。** 否则插件会和 OpenFang 内部配置模型耦死。
2. **所有“暂不支持”的字段都必须注册 diagnostic，而不是静默忽略。** 否则插件作者无法判断是宿主 bug 还是宿主策略。
3. **`runtime` 和 `register*` 要分开建桥。** 一个负责“插件如何调用宿主 helper”，一个负责“插件如何把能力挂回宿主”。

### 2. `api.runtime` 字段兼容清单

前面已经按命名空间讲过一轮，这里再压成实现清单。重点不是解释用途，而是明确每一项在 MVP 里要不要进。

| OpenClaw `api.runtime` 字段 | OpenFang MVP 判断 | 建议落点 | 备注 |
|------|------|------|------|
| `version` | **直接支持** | `RuntimeMetadataBridge` | 返回宿主版本与兼容层版本 |
| `config` | **适配支持** | `RuntimeConfigBridge` | 先只读，写回需要能力审批 |
| `system` | **适配支持** | `RuntimeSystemBridge` | `runCommandWithTimeout` 必须走 capability gate |
| `media` | **部分支持** | `RuntimeMediaBridge` | 先补高频 helper，不承诺全量 API |
| `tts` | **暂不支持** | diagnostic stub | 除非有明确需求，否则先拒绝 |
| `tools` | **部分支持** | `RuntimeToolsBridge` | 先提供 memory get/search 类 helper |
| `channel` | **极少量支持** | `RuntimeChannelBridge` | 仅保留通用 helper，不承诺 channel-specific helpers |
| `logging` | **直接支持** | `RuntimeLoggingBridge` | child logger 与 verbose 判断都容易桥接 |
| `state` | **直接支持** | `RuntimeStateBridge` | 插件状态目录解析是低成本高收益能力 |

### 3. Hook 与命令面也需要单独列白名单

OpenClaw 的 `registerHook` 和 `on(...)` 不只是“有接口”这么简单，它们背后要求宿主真的存在对应生命周期事件。所以 OpenFang 不能只做一个泛型 `registerHook(eventName)`，还要先给出一版**允许映射的事件白名单**。

MVP 建议优先映射：

1. `before_tool_call`
2. `after_tool_call`
3. `message_received`
4. `message_sent`
5. `session_start`
6. `session_end`

MVP 默认不映射：

1. 与 Gateway 深绑定的事件
2. 与完整 channel adapter 深绑定的事件
3. 需要完整 LLM orchestration pipeline 细节的事件

同样，`registerCommand` 也要先限定在：

1. 聊天命令
2. API 侧受控命令

而不是一开始就把 CLI、HTTP、gateway 三条命令面全部混在一起。

## 反推代码结构：为了接住这张清单，OpenFang 代码里应该先长出哪些模块？

如果把上面的兼容清单当成约束条件，那么反推出来的代码结构其实会比“按功能随手加文件”清晰得多。因为每一层都已经有明确职责。

### 1. Rust 宿主侧先补一个插件宿主内核，而不是继续把逻辑塞进 skill loader

最先该长出来的不是另一个 `openclaw_compat.rs`，而是一组正式的 host-side 类型，建议挂在 `openfang-kernel`：

1. `plugin_host/mod.rs`
2. `plugin_host/registry.rs`
3. `plugin_host/descriptor.rs`
4. `plugin_host/diagnostics.rs`
5. `plugin_host/runtime_caps.rs`
6. `plugin_host/hook_mapper.rs`
7. `plugin_host/service_manager.rs`

这层至少要有下面这些核心结构：

```rust
pub struct PluginDescriptor {
   pub id: String,
   pub name: String,
   pub version: Option<String>,
   pub description: Option<String>,
   pub source_path: PathBuf,
   pub origin: PluginOrigin,
}

pub enum PluginLoadState {
   Discovered,
   Validated,
   Loaded,
   Disabled,
   Error,
}

pub struct PluginHostRegistry {
   pub descriptors: HashMap<String, PluginDescriptor>,
   pub states: HashMap<String, PluginLoadState>,
   pub diagnostics: Vec<PluginDiagnostic>,
   pub slot_bindings: HashMap<String, String>,
}
```

这组结构的意义是把“插件是什么、当前状态是什么、为什么失败”从执行器里拆出来，变成宿主治理层的一等公民。

### 2. `openfang-extensions` 负责导入，不负责运行

合同导入层不应该和 Node 执行器混在一起，建议放到 `openfang-extensions`：

1. `plugin_import/mod.rs`
2. `plugin_import/manifest.rs`
3. `plugin_import/package_pack.rs`
4. `plugin_import/entrypoint.rs`
5. `plugin_import/report.rs`

这层建议的核心结构如下：

```rust
pub struct PluginCandidate {
   pub descriptor: PluginDescriptor,
   pub manifest_path: PathBuf,
   pub entrypoints: Vec<PluginEntrypoint>,
   pub config_schema: Option<serde_json::Value>,
   pub required_sdk_imports: Vec<String>,
}

pub struct PluginImportReport {
   pub candidate: PluginCandidate,
   pub diagnostics: Vec<PluginDiagnostic>,
   pub unsupported_features: Vec<String>,
}
```

这一步做完以后，OpenFang 才能先回答“这个插件理论上能不能被当前宿主承接”，而不是上来就拉起 Node 进程。

### 3. `openfang-skills` 需要新增一个 OpenClaw 插件桥，而不是只复用现有 skill stdin/stdout 协议

现有 `openfang-skills` 的 Node 执行足以证明路线可行，但 OpenClaw 插件不只是“执行一个工具函数”。它至少还需要：

1. 注入 `openclaw/plugin-sdk*` shim
2. 接收插件发来的注册声明
3. 把 `runtime.*` 请求桥回 Rust host
4. 管理 service 生命周期与插件日志

因此这里建议新增一组桥接模块：

1. `openclaw_plugin_bridge/mod.rs`
2. `openclaw_plugin_bridge/launcher.rs`
3. `openclaw_plugin_bridge/rpc.rs`
4. `openclaw_plugin_bridge/registrations.rs`
5. `openclaw_plugin_bridge/runtime_calls.rs`

桥接协议建议不要继续沿用“工具请求一次，进程跑一次”的最小协议，而改成**长生命周期 plugin session**：

```text
Rust host
  -> launch plugin worker
  -> send init(api_surface, plugin_config)
  <- receive registrations(tools, hooks, services, commands)
  <- receive runtime_call(...)
  -> reply runtime_result(...)
  -> send stop()
```

只有这样，`registerService`、`on(...)`、child logger、状态目录这些能力才有稳定的会话语义。

### 4. Node 侧需要一个真正的 shim package，而不是在单个入口文件里硬编码

如果未来要兼容 `openclaw/plugin-sdk`、`openclaw/plugin-sdk/core`、`openclaw/plugin-sdk/compat` 以及若干 subpath，那么最好单独放一组 Node 包或虚拟包结构，例如：

1. `packages/openclaw-plugin-shim/package.json`
2. `packages/openclaw-plugin-shim/index.js`
3. `packages/openclaw-plugin-shim/core.js`
4. `packages/openclaw-plugin-shim/compat.js`
5. `packages/openclaw-plugin-shim/channel/*.js`

它至少要输出 4 类对象：

1. `createPluginApi()`
2. `createRuntimeBridge()`
3. `createRegistrationCollector()`
4. `createDiagnosticStub()`

其中最关键的是：**对暂不支持的接口，shim 必须返回可解释的失败，而不是假装成功。**

### 5. Hook、Command、Service 三类注册要在 Rust 侧拆成独立 sink

如果继续把所有注册项都塞进一个 `Vec<Registration>`，后面演进会很痛苦。更合理的是在 `openfang-kernel` 里拆成几种显式 sink：

1. `ToolRegistrationSink`
2. `HookRegistrationSink`
3. `CommandRegistrationSink`
4. `ServiceRegistrationSink`

这样做的原因不是“好看”，而是因为这 4 类对象后面的治理方式完全不同：

1. tool 需要注册到 tool runtime
2. hook 需要接 EventBus 和优先级排序
3. command 需要接 channel/API 命令面
4. service 需要接 start/stop 生命周期与重载

### 6. 反推后的 crate 职责边界应该保持稳定

综合上面这张清单，比较稳的职责边界是：

1. `openfang-extensions`：发现、读取、导入、安装、生成 import report
2. `openfang-skills`：Node plugin worker、shim 注入、桥接协议
3. `openfang-kernel`：注册表、状态机、diagnostics、hook/service/command/tool sinks、capability 审批
4. `openfang-api`：受控命令面与后续极少数 API/gateway 接口
5. `openfang-channels`：后置 channel 映射，不进 MVP
6. `openfang-memory`：长期 slot 与 memory helper 落点
7. `openfang-runtime`：长期 capability runtime 收口，不抢 MVP 的 Node bridge 主线

### 7. 第一批真正值得开始写的文件

如果下一步要从文档进入实现，建议第一批只开 6 个点，不要铺太大：

1. `openfang-extensions`：`plugin_import/report.rs`
2. `openfang-extensions`：`plugin_import/manifest.rs`
3. `openfang-kernel`：`plugin_host/registry.rs`
4. `openfang-kernel`：`plugin_host/diagnostics.rs`
5. `openfang-skills`：`openclaw_plugin_bridge/rpc.rs`
6. Node shim 包：`index.js` + `core.js`

这 6 个点一旦成形，OpenFang 就已经从“讨论兼容”进入“有正式宿主骨架可接插件”的阶段了。

## 最适合作为兼容验收基线的插件样例是什么？

前面已经把 OpenClaw 第十一章里的复杂插件样例定成参考基线，这里可以再说得更技术一点：

**最适合当 OpenFang 兼容验收基线的，不是“最小 hello tool 插件”，而是同时使用 `pluginConfig`、`logger`、`runtime.system`、`runtime.state`、`registerTool`、`registerCommand`、`registerHook`、`registerService` 的复合插件。**

原因很直接：

1. 它能同时测试合同层、API 层、runtime 层、生命周期层。
2. 它不要求 OpenFang 立刻开放最危险的 gateway/http/channel 深宿主面。
3. 它已经足够逼近真实插件兼容压力，不会让“hello world 能跑”误导决策。

如果 OpenFang 连这一类复合插件都无法承接，那么谈更深的 channel/provider/gateway 兼容没有意义。

---

## 兼容矩阵：OpenClaw 的每类插件落点，在 OpenFang 里应该映射到什么？

这一节比“选哪种 JS runtime”更重要，因为它决定了 OpenFang 未来兼容层是不是有清晰边界。

| OpenClaw 扩展点 | OpenClaw 当前语义 | OpenFang 最接近的落点 | 兼容判断 |
|------|------|------|------|
| `tools` | 给 agent 注册工具工厂 | `openfang-skills` / tool runtime | 最容易兼容 |
| `services` | 启动后台守护任务 | `openfang-hands` / `BackgroundExecutor` | 可映射，但需新适配层 |
| `hooks` | 接入生命周期事件 | `openfang-kernel` hooks / event bus | 可部分兼容 |
| `commands` | 聊天 / 命令面注册 | `openfang-channels` / CLI command surface | 可部分兼容 |
| `providers` | 模型与认证方式扩展 | model catalog / auth profile / provider registry | 中等难度 |
| `channels` | 新增完整聊天渠道 | `openfang-channels` | 高成本，不能自动兼容 |
| `gatewayHandlers` | 直接接管网关方法 | `openfang-api` / transport edge | 应谨慎开放 |
| `httpRoutes` | 新增 HTTP 路由 | `openfang-api` | 不应默认原样兼容 |
| `cliRegistrars` | 向 CLI 注入命令 | `openfang-cli` | 可兼容，但要受控 |
| `memory` kind | 独占记忆实现 | `openfang-memory` | 不能简单 1:1 硬兼容 |

这张表最重要的结论是：

1. **并不是所有 OpenClaw 插件点都值得 1:1 照搬。**
2. **最适合最先兼容的是 tools / services / hooks / 部分 commands。**
3. **channels / gatewayHandlers / memory provider 这类深宿主扩展，必须显式分级。**

### 为什么不能追求“全量原样兼容”？

因为 OpenFang 的系统边界与 OpenClaw 并不相同：

1. OpenClaw 更像 Node 宿主 + 插件总线
2. OpenFang 更像 Rust 中枢 + 多 crate 控制面 + 更强能力边界

如果直接把 OpenClaw 的 gateway / route / memory 插件语义原样搬进 OpenFang，等于把一部分本该受强约束治理的内核边界重新开放成动态注入点。这和 OpenFang 当前的系统哲学是冲突的。

所以更稳的目标不是“全兼容”，而是：

**兼容作者体验和高价值生态，不照搬全部宿主边界。**

---

## OpenFang 应该如何兼容：推荐的四层适配架构

如果要把兼容做成工程，而不是一句愿景，最合理的结构应该分四层。

## 当前文档的直接目标

从现在开始，这份文档把目标收紧成一句话：

**先让 OpenFang 支持一批真实可用的 OpenClaw 官方插件。**

这里的“支持”至少意味着 4 件事：

1. OpenFang 能识别官方插件包结构。
2. OpenFang 能加载并执行第一批受支持的插件类型。
3. OpenFang 能提供插件运行所需的最小宿主 API。
4. OpenFang 能给出与官方插件机制接近的配置、状态与诊断反馈。

如果做不到这 4 件事，就不应对外称为“支持 OpenClaw 插件”。

## MVP：第一阶段到底支持什么？

第一阶段不要追求“所有插件点都能跑”，而是明确支持下面这组最有价值、最容易落地的官方能力：

1. `tools`
2. `services`
3. 一小部分 `hooks`
4. 一小部分 `commands`

第一阶段明确不纳入默认支持范围：

1. `channels`
2. `gatewayHandlers`
3. `httpRoutes`
4. `providers` 的深宿主认证面
5. `memory` / `contextEngine` 这类独占 slot

这样做的原因很简单：

1. 这批插件点最接近 OpenFang 现有的 skills、background task、kernel hooks、command surfaces。
2. 这批插件点最适合先用受控 Node bridge 承接。
3. 这批插件点不会立刻把 OpenFang 的核心网关和内核边界重新开放成动态注入点。

## OpenFang 必须模拟的最小宿主环境

如果目标是“让官方 OpenClaw 插件尽量少改就能接入 OpenFang”，那 OpenFang 至少要模拟下面这些宿主能力。

### 1. 官方包合同

OpenFang 需要能识别并导入：

1. `openclaw.plugin.json`
2. `package.json.openclaw.extensions`
3. 函数式导出与对象式导出
4. package pack 多入口拆分

这一层解决的是“插件包能不能被看懂”。

### 2. 官方模块入口

OpenFang 需要在插件运行环境里接住至少这些 import path：

1. `openclaw/plugin-sdk`
2. `openclaw/plugin-sdk/core`
3. `openclaw/plugin-sdk/compat`

如果这些路径无法解析，那么很多真实插件连启动都做不到。

### 3. 官方配置合同

OpenFang 需要承接至少这些配置语义：

1. `plugins.entries.<id>.enabled`
2. `plugins.entries.<id>.config`
3. `plugins.allow`
4. `plugins.deny`
5. `plugins.load.paths`

这一层解决的是“插件能不能被宿主配置和治理”。

### 4. 官方注册面

OpenFang 至少要实现第一阶段支持范围内的 `api.register*` 族接口，包括：

1. tool 注册
2. service 注册
3. 受限 hook 注册
4. 受限 command 注册

这里的关键不是 API 名字完全一致，而是插件作者实际依赖的能力能被稳定映射。

### 5. 官方状态与诊断面

OpenFang 需要把插件状态变成宿主可见对象，而不是只有“跑起来”或“没跑起来”。最少要有：

1. discovered
2. validated
3. loaded
4. disabled
5. error

同时最少要能报告：

1. manifest 错误
2. schema 错误
3. id mismatch
4. import path 缺失
5. 不受支持扩展点
6. source precedence 冲突

## 推荐实现顺序

### Step 1：先做插件导入器，不急着跑插件

第一步不是执行，而是构建一个正式的 OpenClaw 插件导入器，至少完成：

1. manifest 读取
2. package pack 解析
3. 导出形态识别
4. 扩展点清单提取
5. 兼容性审计结果输出

这一阶段的结果应该是：

**给一个 OpenClaw 官方插件目录，OpenFang 能明确告诉你它是否可导入、为什么可导入、缺什么才能运行。**

### Step 2：先用 Node bridge 承接第一批插件

第一阶段执行层不要过度设计，直接复用 OpenFang 已有 Node 执行基础，目标是尽快把支持链路跑通：

1. 导入插件
2. 解析插件注册面
3. 将 tool / service / 部分 hook / 部分 command 映射到 OpenFang 宿主
4. 提供最小 `plugin-sdk` 兼容入口

这一步的重点是“支持插件”，不是“设计最先进的运行时”。

### Step 3：补齐生态语义

当第一批插件已经能跑以后，第二优先级不是换运行时，而是补宿主语义：

1. `enabled / disabled / error`
2. source precedence
3. allow / deny
4. `plugins.entries.<id>.config`
5. diagnostics

只有补到这一步，OpenFang 才算开始真正承接 OpenClaw 官方插件机制，而不只是“临时执行一些 JS 文件”。

### Step 4：再考虑宿主内运行时优化

只有当插件合同、宿主 API、状态语义已经稳定以后，再考虑把高价值插件迁到更强受控的宿主内执行层。否则只是把一个还没定型的兼容层更快地执行一遍。

## 明确的非目标

为了避免文档再次滑回“大而全讨论”，这里把当前非目标写清楚：

1. 不是第一阶段就完整兼容所有 OpenClaw 插件点。
2. 不是第一阶段就复刻 OpenClaw 的 in-process trusted-code 模型。
3. 不是第一阶段就把 Node bridge 全部替换成宿主内 JS runtime。
4. 不是第一阶段就开放 `channels`、`gatewayHandlers`、`httpRoutes`、独占 memory slot 这类深宿主能力。

## 结论

按照最新需求，OpenFang 文档的中心不应该再是“Rust 原生怎么跑插件”，而应该是：

**如何先把 OpenClaw 官方插件支持起来。**

更准确地说，当前最重要的实现顺序是：

1. 先支持官方插件包合同。
2. 先支持第一批高价值插件点。
3. 先补齐最小宿主 API、配置语义和诊断面。
4. 最后再优化执行器，而不是反过来。

如果这个顺序不倒，OpenFang 后续无论继续用 Node bridge，还是迁向更强受控的宿主内运行时，都会建立在一个已经可用的插件支持层之上。
---

## 路线选择表：OpenFang 到底该优先押哪一条？

| 路线 | 生态兼容 | 安全边界 | 运行时成本 | 工程复杂度 | 更像什么角色 |
|------|----------|----------|------------|------------|--------------|
| Node bridge / STDIO | 很强 | 中等偏强 | 较高 | 低 | 今天最稳的过渡层 |
| `deno_core` 嵌入 V8 | 强 | 强 | 低到中 | 高 | 中期主力路线 |
| `boa` 纯 Rust JS | 中低 | 强 | 中 | 中 | 轻量受限插件层 |
| WASM / Extism | 中低 | 很强 | 中低 | 很高 | 长期统一终局 |

这张表最重要的结论其实很克制：

1. **短期** 不应该急着否定 Node bridge。它仍然是 OpenFang 最稳、最容易落地、最适合快速吸纳现有生态的方案。
2. **中期** 最值得重点下注的是 `deno_core`。它最平衡：兼容 JS 生态的能力够强，宿主控制权也还在 Rust 手里。
3. **长期** 真正有体系价值的统一方向仍然是 WASM 化，因为它最符合 OpenFang 作为 Agent OS 的边界控制哲学。
4. **更关键的一句**：不应追求“让所有 OpenClaw 插件点都跑起来”，而应先决定“哪些插件点值得被 OpenFang 长期接纳”。

换句话说，最合理的路线不是三选一，而是：

1. **先用 Node bridge 收编高价值子集**
2. **再补 OpenClaw 生态语义兼容层**
3. **再用 `deno_core` 吃下高价值受控 JS 插件**
4. **最后把真正长期要留下来的插件接口收敛到 WASM / capability-based runtime**

---

## 为什么这个判断和 OpenFang 现状是连得上的

如果只看技术想象，`deno_core` 嵌入宿主当然很诱人；但如果看 OpenFang 现有代码，真正有价值的是它刚好填在**Node 兼容层**和**WASM 能力边界层**之间：

1. 向左，它延续了 OpenClaw / JS 生态，不会像直接切 WASM 一样立刻抬高作者迁移成本。
2. 向右，它比当前 Node 子进程模式更接近宿主内 capability execution，方向上和 OpenFang 现有 WASM sandbox 哲学一致。
3. 向下，它不会推翻现有 process manager、Node skill loader、托管 Node gateway 这些现实资产，而是给它们增加一条更强的“高价值插件内嵌路线”。
4. 向上，它只有在 OpenFang 先补齐 OpenClaw SDK 兼容层与生态语义兼容层后，才真正值得投入。

所以 `deno_core` 在这篇文章里之所以是中期主力候选，不是因为它听起来先进，而是因为它最像**现有 OpenFang 执行面的一次连续演化**。

---

## 已知边界与限制

这篇讨论如果要达到准五星，必须把边界讲明：

1. 它不是当前功能说明。OpenFang 现在并没有主线落地 OpenClaw plugin registry mirror，也没有主线落地 `deno_core` / `boa` / `extism` 运行时。
2. 它不是“全量 1:1 兼容宣言”。像 `channels`、`gatewayHandlers`、`memory provider` 这类深宿主扩展，不应被默认视为自动兼容对象。
3. 它不是性能实测报告。这里的运行时成本判断是架构级推断，不是基于本仓库 benchmark 的定量结论。
4. 它不是“一步到位替换 Node”的主张。当前 OpenClaw 兼容、Node skill loader、托管 Node gateway 都仍然有很强现实价值。
5. 它也不是“WASM 已经统一一切”的结论。源码能证明的是 WASM agent 线已经较成熟，但技能层统一 runtime 仍在收口中。
6. 真正的决策门槛不在能不能跑 JS，而在 OpenFang 是否准备长期维护一套 OpenClaw SDK 兼容层、一套生态语义兼容层，以及一套宿主内 JS capability runtime。

---

## 结论：OpenFang 兼容 OpenClaw 插件生态的真正破局点

如果我们要在“兼容 OpenClaw 插件生态”的前提下让 OpenFang 原生发力，最稳妥的结论不是“马上选一种 JS runtime”，而是先把目标拆开：

1. **先兼容高价值插件点**：tools / services / hooks / 部分 commands
2. **再兼容 OpenClaw 生态语义**：schema、status、diagnostics、source precedence、slot arbitration
3. **最后再把值得长期保留的执行面收口到宿主内 runtime 与 capability-based 边界**

如果把原生执行也放回这个框架里，那么更精确的判断是：

1. **从产品现实压力看**，OpenFang 现在最需要的不是立刻重写所有 OpenClaw 插件，而是低阻力收编其中高价值子集，因此 Node bridge 短期仍然最有现实价值。
2. **从生态兼容语义看**，真正缺的不是执行器，而是 OpenClaw plugin manifest / SDK / diagnostics / status model 的兼容层。
3. **从运行时底线压力看**，OpenFang 不能满足于永远依赖 Node 子进程，因此 `deno_core` 或更强的宿主内执行路线迟早要落地。
4. **从长期系统演进看**，真正该追求的不是“Rust 原生运行 JS”这件事本身，而是建立一个让插件永远受能力边界约束、且未来能统一多语言的执行层。

所以，这篇讨论最重要的结论不是“选哪门技术”，而是给 OpenFang 确认一个更清晰的三段式演进方向：

**先兼容生态语义，再优化执行方式，最后统一长期运行时。**
