## 第四章：记忆、技能与自主 Agent 🧠

> **核心代码**：`openfang-memory/`（约 3900 行）、`openfang-skills/`（约 3400 行）、`openfang-hands/`（约 1500 行）
> **关联代码**：`openfang-kernel/src/background.rs`、`openfang-kernel/src/kernel.rs`、`openfang-types/src/agent.rs`
> **本章主轴**：长期记忆如何落地、Skill 如何被加载并注入能力、Hand 如何从模板变成后台运行的 Agent

这一章讨论的是 OpenFang 从“会回答问题”走向“能持续工作”的那部分骨架。这里真正关键的不是一句“支持长期记忆”或“支持自主 Agent”，而是三条具体链路：记忆如何写进 SQLite、技能如何被注册和冻结、Hand 如何被解析成 AgentManifest 并交给后台执行器运行。

---

### 4.1 深入解读

#### 设计张力 🎯

> **记忆广度** ←→ **检索可控性**
> 存得越多，理论上越聪明；但只要检索失控，LLM 拿到的就是更大一团噪声。OpenFang 的现实做法不是复杂 taxonomy，而是 SQLite 上的结构化存储、语义存储、会话压缩和简单衰减。

> **Skill 生态开放性** ←→ **加载安全性**
> Skill 越容易导入，生态越快繁殖；但每多一份 prompt-only 技能，就多一份提示注入和能力越权的风险。OpenFang 在这里更偏保守：先做格式转换和 prompt 注入扫描，再决定是否接纳。

> **自主运行** ←→ **成本失控**
> `ScheduleMode` 和 Hands 确实让 Agent 不再只能被动聊天，但“定义了可自主”不等于“所有资源约束都已经闭环”。第四章最需要避免的误读，就是把声明式能力直接写成生产级自治能力。

#### 关键场景 🎬

第四章如果只被理解成“记忆、技能、Hands 三件套介绍”，其实还不够。它真正服务的是 4 类长期演进场景：

1. **跨会话长期陪伴场景**
	当 Agent 不再只服务单轮对话，而是要跨天、跨渠道、跨任务记住用户、记住历史决策、记住工作上下文时，session、semantic memory、structured state 就必须形成一套能持续工作的底盘。

2. **技能生态持续扩张场景**
	一旦系统开始大量吸收外部 skill、兼容 OpenClaw 资产、允许 workspace override，真正困难的就不再是“能不能加载”，而是来源优先级、冲突治理、提示注入和生态边界能不能守住。

3. **从模板 Agent 走向长期后台 Agent 场景**
	Hands、`ScheduleMode`、`BackgroundExecutor` 把 OpenFang 从“用户发一句、Agent 回一句”推向了“Agent 可以周期运行、自主触发、持续占用资源”的另一类系统。

4. **资源受限的自治场景**
	长期记忆、自主调度、技能扩张这些能力一旦组合起来，最容易失控的不是功能，而是预算、并发、调度表达力和状态治理。如果这些能力只是声明上存在、执行上分散，系统就会在长期运行里暴露短板。

所以，第四章真正讨论的不是“OpenFang 支不支持记忆和技能”，而是：**OpenFang 是否已经具备把长期状态、技能生态和后台自治组织成一个可持续运行系统的底盘。**

#### 代码走读 📖

##### 1. `MemorySubstrate`：统一入口不是一个数据库表，而是六个子系统

`substrate.rs` 里的 `MemorySubstrate`（L28）不是单一 store，而是把 6 个子模块组合到一个统一接口后面；其打开入口 `open()` 在 L40：

| 子系统 | 字段 | 作用 |
|--------|------|------|
| 结构化存储 | `structured` | Agent 的 KV 状态、持久化 AgentEntry |
| 语义记忆 | `semantic` | 文本记忆与向量召回 |
| 知识图谱 | `knowledge` | 实体与关系 |
| 会话存储 | `sessions` | 聊天 session、canonical session、压缩摘要 |
| 记忆整合 | `consolidation` | 老记忆衰减 |
| 使用账本 | `usage` | LLM 费用、tokens、tool_calls 统计 |

这说明 OpenFang 的“记忆”从一开始就不是一个抽象概念，而是直接跟 Agent 持久化、会话压缩、成本治理绑在一套 SQLite substrate 上。

`open()` 里还顺手做了两件基础工程工作：

- `PRAGMA journal_mode=WAL`
- `PRAGMA busy_timeout=5000`

也就是说，这套记忆底盘默认就是按“多读多写的本地数据库”来配置的，而不是只当临时演示存储。

##### 2. `SemanticStore`：真正的设计不是“向量搜索”，而是“向量搜索失败时还能工作”

`semantic.rs` 的核心函数是两个：`remember_with_embedding()`（L44）与 `recall_with_embedding()`（L95）。

- `remember_with_embedding()`
- `recall_with_embedding()`

`remember_with_embedding()` 会把文本、source、scope、metadata 以及可选 embedding 一起写入 `memories` 表。这里没有什么 fancy taxonomy，重点是 **embedding 是可选列**。这让它天然支持两种模式：

- 在线 embedding 模式
- 纯文本 fallback 模式

`recall_with_embedding()` 的流程很值得拆开：

1. 如果没有 query embedding，就对 `content LIKE '%query%'` 做文本检索
2. 如果有 query embedding，就先扩大候选集：`fetch_limit = max(limit * 10, 100)`
3. 从 SQLite 取出候选片段
4. 如果片段里有 embedding，就用 `cosine_similarity()` 做重排
5. 最终裁剪到 `limit`

这比“直接上向量库”更朴素，但也更稳。它的关键价值在于：**embedding 缺失时不会让整条检索链直接失效。**

不过这里也有明确边界：

- 没有 category taxonomy
- 没有 entropy 过滤
- 没有 TF-IDF 或倒排索引中间层
- 没有更复杂的 rerank pipeline

所以它是一个很实用的 Phase 1/2 设计，但还不是“工业级记忆检索栈”的终局形态。

##### 3. `ConsolidationEngine`：OpenFang 不是没有 consolidation，但当前只有“衰减”，还没有“合并”

这点很关键，因为当前章节原文基本没把它写清楚。`consolidation.rs` 里的 `ConsolidationEngine` 在 L14，`consolidate()` 在 L27，已经是真实现，不是空壳：

- 找出 `accessed_at < 7 days` 的旧记忆
- 用 `1 - decay_rate` 乘到 `confidence`
- 最低保留到 `0.1`
- 返回 `ConsolidationReport`

但它也明确写死了：

```rust
memories_merged: 0 // Phase 1: no merging
```

也就是说，OpenFang 当前的 consolidation 更准确地说应该叫：

- **confidence decay engine**

而不是“真正的记忆合并器”。如果教程把它写成复杂摘要与聚合系统，就会说过头。

##### 4. `SessionStore`：跨渠道长期记忆的关键，其实是 canonical session

第四章里最容易被忽略的其实不是 semantic store，而是 `session.rs`。这里实现了一个很重要的结构：**每个 Agent 一份 canonical session**。相关落点包括 `store_llm_summary()`（L322）、`append_canonical()`（L415）和 `write_jsonl_mirror()`（L533）。

它的行为是：

- 各渠道的消息都可以汇入同一个 canonical session
- 当消息超过阈值（默认 `100`）时做压缩
- 保留最近 `50` 条窗口消息
- 把更早的消息压成 `compacted_summary`

`append_canonical()` 当前的压缩实现不是再调用 LLM，而是**本地字符串摘要**：把旧消息按 `User:` / `Assistant:` / `System:` 拼接起来，再把整体裁到约 `4000` 字符。

这意味着 OpenFang 当前有两套不同层级的“会话压缩”能力：

- **本地摘要压缩**：`append_canonical()` 自动触发
- **LLM 摘要写回**：`store_llm_summary()` 把更高质量摘要覆盖回 canonical session

另外，`write_jsonl_mirror()` 还会把会话镜像到磁盘 JSONL 文件。它不是主存储，但对调试和审计很实用。这一层比“向量搜索”更接近用户实际感知到的长期记忆行为。

##### 5. `SkillRegistry`：它是加载注册表，不是文件监听式热更新守护进程

`openfang-skills/src/registry.rs` 的 `SkillRegistry` 在 L13，关键入口 `freeze()` 在 L46、`load_bundled()` 在 L61、`load_all()` 在 L106。它的内部结构很简单：

| 字段 | 作用 |
|------|------|
| `skills: HashMap<String, InstalledSkill>` | 按 skill name 存已安装技能 |
| `skills_dir: PathBuf` | 技能目录 |
| `frozen: bool` | Stable 模式下禁止新技能加载 |

当前实现的关键行为有 5 个：

1. `load_bundled()`：加载编译期内置技能
2. `load_all()`：从目录扫描技能子目录，读取 `skill.toml`
3. 如果目录里没有 `skill.toml`，但有 `SKILL.md`，就走 `openclaw_compat::convert_skillmd()` 自动转换
4. 对 prompt 内容做 `SkillVerifier::scan_prompt_content()` 提示注入扫描，critical 级别直接 block
5. `freeze()` 后不允许继续动态加载

这说明它的强项是：

- 兼容 OpenClaw 的 `SKILL.md`
- prompt 注入防线前置到加载期
- workspace skills 可以覆盖 bundled/global 技能

但它**不是**下面这些东西：

- 不是基于 `notify` 的实时文件监听热重载器
- 没有依赖图求解
- 没有版本冲突仲裁
- 没有祖先链循环检测

所以第四章里“技能包热更新”这句话应该收紧为：**可加载、可转换、可覆盖、可冻结的技能注册表**。

##### 6. `openclaw_compat.rs`：OpenFang 真正在吃 OpenClaw 生态红利的地方

OpenFang 的兼容层不是一句口号，而是实打实的格式转换器。`openclaw_compat.rs` 的 `convert_skillmd()` 在 L164，做了这些事：

- 识别目录下是否存在 `SKILL.md`
- 解析 YAML frontmatter + Markdown body
- 提取 `requires.bins`、`requires.env`、commands 等元数据
- 把 OpenClaw 命令名映射成 OpenFang 工具名
- 生成 OpenFang `SkillManifest`
- 把 Markdown body 作为 `prompt_context`

这意味着 OpenFang 不是“兼容 OpenClaw 运行时”，而更像是：**把 OpenClaw 的 prompt-skill 资产翻译成自己的 manifest + prompt_context 体系。** 这是生态复用，但不是二进制兼容。

##### 7. `HandRegistry`：Hands 本质上是“带 UI 配置与依赖检查的 Agent 模板”

`openfang-hands/src/registry.rs` 的 `HandRegistry` 在 L39，有两个核心存储；相关入口包括 `activate()`（L87）、`set_agent()`（L144）、`check_requirements()`（L175）、`check_settings_availability()`（L194）：

| 字段 | 类型 | 作用 |
|------|------|------|
| `definitions` | `HashMap<String, HandDefinition>` | 所有已知 hand 定义 |
| `instances` | `DashMap<Uuid, HandInstance>` | 当前活跃实例 |

它做的主要事情并不是调度，而是：

- `load_bundled()` 载入 HAND.toml 定义
- `check_requirements()` 检查 binary / env / api key 依赖
- `check_settings_availability()` 检查某个 setting option 当前是否可用
- `activate()` 创建 `HandInstance`
- `set_agent()` 把后续 spawn 出来的 agent_id 回填到 hand instance

所以 Hands 的第一层定位应该是：

- **产品化配置模板**
- **依赖检查面板**
- **运行中实例注册表**

而不是一个单独的自治调度器。

##### 8. `resolve_settings()` + `activate_hand()`：Hand 变成真实 Agent 的过程

Hands 最关键的真正实现不只在 hands crate，还在 `kernel.rs` 的 `activate_hand()`（L2674）链路里；而设置解析函数 `resolve_settings()` 在 `openfang-hands/src/lib.rs` L191。过程是：

1. 从 `hand_registry` 取到 `HandDefinition`
2. 调 `hand_registry.activate()` 先生成一个 HandInstance
3. 根据 hand 定义构造 `AgentManifest`
4. 若 hand 的 provider/model 是 `default`，就继承 kernel 默认模型
5. 将 hand 声明的 tools、skills、mcp_servers 写入 manifest
6. 若 hand 包含 `shell_exec`，直接给它 `ExecSecurityMode::Full`，并把 timeout 放宽到 300 秒
7. `resolve_settings()` 把用户在 UI 里选的 provider/binary/env 映射成 prompt block 与 allowed env vars
8. 如果 hand 自带 skill content，则把它注入到 system prompt 的 `Reference Knowledge`
9. 最后调用 `spawn_agent(manifest)` 真正生成 Agent
10. 再用 `set_agent()` 把 agent_id 回写到 HandInstance

这条链路说明 Hands 并不是另一套 Agent 系统。它是 **AgentManifest 的产品化生成器**。

##### 9. `ScheduleMode` + `BackgroundExecutor`：这里真的有实现，但比原文写得更保守

`ScheduleMode` 在 `agent.rs` L221 定义了四态，后台执行器 `BackgroundExecutor` 在 `background.rs` L21，真正起 loop 的入口 `start_agent()` 在 L48，全局并发上限 `MAX_CONCURRENT_BG_LLM` 在 L18：

- `Reactive`
- `Periodic { cron: String }`
- `Proactive { conditions: Vec<String> }`
- `Continuous { check_interval_secs }`

和原文最大的差别在于：它不是“严格 Linux Cron + 无限自主思考”，而是通过 `background.rs` 的 `BackgroundExecutor` 落成一套更保守的后台机制：

| 模式 | 实现方式 |
|------|----------|
| `Reactive` | 不启动后台任务 |
| `Continuous` | 固定间隔 self-prompt；若上次未完成则跳过本轮 |
| `Periodic` | 把简化的 `every 5m / every 1h` 表达式转成秒，再做定时 self-prompt |
| `Proactive` | 不建独立 loop，而是把条件解析为 `TriggerPattern`，由 trigger engine 驱动 |

还有三个很实际的约束：

- 全局并发背景 LLM 调用上限 `MAX_CONCURRENT_BG_LLM = 5`
- 每个 Agent 都有 `busy` 标志，前一轮没跑完就跳过当前 tick
- 通过 `watch::Receiver<bool>` 响应 Supervisor 的 shutdown 信号

所以 OpenFang 确实已经有“后台 Agent 执行器”，但要写准确：

- `Periodic` 当前是**简化 cron 解析**，不是完整 cron 语法
- `Proactive` 当前只支持有限条件格式，如 `event:system`、`memory:key`
- 这更像“可控的后台 self-prompt 机制”，而不是无限自治框架

##### 10. `ResourceQuota`：字段很完整，但执行闭环仍是分散的

`ResourceQuota` 的默认值依然很重要：

- `max_memory_bytes = 256 MB`
- `max_cpu_time_ms = 30_000`
- `max_tool_calls_per_minute = 60`
- `max_llm_tokens_per_hour = 1_000_000`
- `max_network_bytes_per_hour = 100 MB`
- `max_cost_per_hour_usd = 1.0`

但第四章必须跟第五章保持一致：这些字段不是全部都由同一个调度器强制执行。当前实现更像：

- Token 限额主要由 scheduler 那边计
- 成本限额主要由 metering 那边计
- 其他维度更多还是 manifest/schema 级声明

所以它们是很好的**资源治理接口定义**，但还不是统一自治熔断内核。

#### 架构决策档案

**ADR-004：把长期记忆分成 substrate、session、semantic 三层，而不是只做“聊天历史”**

- **背景**：只靠原始会话日志，无法支撑跨渠道上下文、检索式记忆和结构化状态
- **决策**：用 `MemorySubstrate` 统一聚合 structured / semantic / sessions / knowledge / consolidation / usage
- **正面后果**：所有长期状态都能共享一个 SQLite 基底，接口一致
- **负面后果**：概念面更多，新读者很容易把“session 压缩”和“semantic memory”混为一谈

**ADR-005：Skill 以 manifest 注册表组织，而不是动态代码注入**

- **背景**：如果技能只是随便一段代码，安全边界和工具描述都很难统一
- **决策**：要求 skill 最终变成 `SkillManifest`，再进 `SkillRegistry`
- **正面后果**：便于列出工具、扫描 prompt、冻结 registry
- **负面后果**：灵活性不如真正的插件依赖系统，冲突求解也还没出现

**ADR-006：Hands 不是新 runtime，而是 curated manifest 生成器**

- **背景**：用户想要的是“直接打开一个能用的 Agent 包”，不是自己手写 manifest
- **决策**：Hand 负责定义、设置、依赖检查和 UI 展示，最终仍落成普通 AgentManifest
- **正面后果**：产品体验统一，复用现有 agent/runtime 基础设施
- **负面后果**：Hands 的能力上限仍受普通 Agent 能力边界约束

---

### 4.2 压力测试 🧪

#### 思想实验 1：Embedding 服务断开后，记忆检索会不会直接失明？

不会完全失明，但会明显降级。

因为 `SemanticStore` 允许没有 embedding 的记忆存在，`recall_with_embedding()` 在没有 query embedding 时会直接走 `LIKE` 检索。所以系统还能“记得一点东西”，只是从向量语义相似退回到字面匹配，召回质量会下降。

#### 思想实验 2：记忆越来越多，OpenFang 会自动帮你做智能归并吗？

当前答案是：**只会做衰减，不会真正合并。**

`ConsolidationEngine` 会下调旧记忆的 confidence，但 `memories_merged` 仍固定为 `0`。如果教程把它写成 nightly synthesis 或 taxonomy consolidation，就和源码不符。

更准确地说，这里的实现更接近“衰减与清理钩子”，而不是完整的知识整理流水线。它确实在 `openfang-memory/src/consolidation.rs` 里为长期运行预留了衰减入口，但目前还没有把多条旧记忆重新压缩成新摘要、再回写主存储的闭环。

#### 思想实验 3：`Continuous` 模式是不是就等于真正的 24/7 守护？

要谨慎。

OpenFang 当前确实会启动后台 loop，并定时给自己发 self-prompt，但它依然受这几层约束：

- 全局背景 LLM 并发上限 5
- busy 时跳 tick
- shutdown 信号可中断
- 调度表达式是简化版，不是完整 cron 平台

所以它是“后台 Agent”，但不是“无限自治体”。

#### 思想实验 4：两个用户都加载了同名 Skill，会发生复杂依赖冲突求解吗？

不会。当前模型更简单：后插入的同名 skill 覆盖前面的同名项，本质上是 HashMap overwrite 语义。它能做到 bundled 被 workspace override，但还没有版本图与依赖求解器。

#### 已知边界与限制

1. 没有 memory category taxonomy，只有 scope/filter 等较轻层级划分
2. consolidation 只有 decay，没有 merge/summarize pipeline
3. SkillRegistry 没有文件监听热重载，也没有依赖冲突求解
4. `Periodic` 是简化表达式，不是完整 Linux cron
5. `ResourceQuota` 字段丰富，但执行仍分散在多个子系统

#### 验证资产与质量保障

第四章的核心组件都有着明确的内部结构，也应当具备对应的可测试面：
1. `MemorySubstrate` 和 `SemanticStore` 需要具备持久化拓扑与降级（无 embedding 时退化为 `LIKE` 匹配）的精准单元测试。
2. `SkillRegistry` 和 `openclaw_compat` 当前在 OpenFang 中已有大量加载转换逻辑，必须确保依赖解析、覆盖逻辑和恶意识别被覆盖。
3. `ConsolidationEngine` 尚未实现真实的记忆摘要与合并，但在未来引入该功能时，相关的回放、准确度与丢失率验证也是必须的，否则很容易制造出伪记忆。

---


### 4.2.5 源码补充：记忆的老化与衰减引擎 (Memory Decay & Consolidation)

> **源码事实**：`crates/openfang-memory/src/consolidation.rs` 已经实现了记忆衰减与回收路径。
> **工程含义**：这说明 OpenFang 不是单纯把记忆持续追加进存储，而是开始处理长期运行下的清理问题。
> **当前边界**：这里更准确的表述应当是“衰减与回收机制”，还不能等同于完整的记忆归并、摘要和长期知识整合流水线。

在大部分教程里，所谓“长期记忆”就是往数据库里不断 INSERT 数据。但真正的工业级 AI 架构师都会面临一个高壁垒痛点：**数据撑爆了怎么办？旧记忆和新事实冲突了怎么办？**

如果你去翻开 OpenFang 的 `crates/openfang-memory/src/consolidation.rs`，会看到一段相当具体的记忆衰减与合并逻辑：
- **`Memory consolidation and decay logic`（记忆合并与衰退逻辑）**。
- OpenFang 在底层设计了一个类似 **“艾宾浩斯遗忘曲线”** 的衰减机制。每一条长期 `Memory` 都会随着心跳周期的推移，其 `confidence`（置信度）按照预设的 `decay_rate`（衰减率）逐渐降低。
- 当旧有陈旧记忆的置信度跌破清理阈值，而新的记忆事实又被重新写入时，系统能依据时间戳和语义距离，在后台静默地"融合降级"或直接进入垃圾回收。这使得 OpenFang 在长期运行时具备了一条明确的衰减与回收路径，而不是完全依赖记忆持续累积。

### 4.2.6 架构弱点：OpenFang Memory 的两处关键瓶颈

> **源码事实**：这两处问题都能在当前实现里直接追溯到具体结构，一个是 `Arc<Mutex<rusqlite::Connection>>`，另一个是先按最近访问时间筛选、再在进程内做 O(N) 相似度计算。
> **工程含义**：它们不意味着这一层“不能用”，但决定了 OpenFang 的记忆系统目前更像可运行的底盘，而不是已经完成工业化扩展的长期记忆栈。
> **当前边界**：这里讨论的是结构性瓶颈与召回上界，不应直接外推成“系统整体不可用”或“记忆设计失败”。

我们在前面的分析中看到了 OpenFang 在防 OOM（Consolidation）以及系统隔离上的努力，但如果站在长期运行和高并发场景下审视，这一层仍有两处不能回避的结构性瓶颈。它们不意味着设计失败，却决定了后续迭代必须优先补哪里。

#### 🚨 缺陷 1：全局 SQLite 互斥锁死 (Mutex Bottleneck)
在 `openfang-memory/src/substrate.rs` 中，整个 `MemorySubstrate` 是基于一个全局的 `Arc<Mutex<rusqlite::Connection>>` 实现的。
- **意味着什么**：尽管开启了 `WAL`（预写日志），但由于顶层直接使用了 Rust 的强互斥锁 `Mutex`，**所有的读写操作在 Rust 进程层面被强制串行化了**。
- **承压场景**：当一个 Agent 正在发起一次需要全表扫描的大型语义检索，或者正在遍历旧记忆进行 Decay（衰减回收）时，其他 Agent 的记忆读取也会被这把全局锁串行化。在更高并发的多 Agent 场景里，这会逐渐显露为明显的吞吐瓶颈。

#### 🚨 缺陷 2：“伪向量数据库”的盲区 (The Semantic Illusion)
在前面探讨 `semantic.rs` 的 `recall_with_embedding` 时，代码里有一段惊人的操作：
- 这个系统并没有使用真正的向量索引（如 `pgvector` HNSW 或 `sqlite-vss`），它执行的是一种**“假向量搜索”**：
- **核心问题**：它先直接抛弃大多数数据，**只把根据“最近访问时间”排序的前 N 条（默认几十到一百条）记录拉入内存**，然后在 Rust 进程中用一个 O(N) 的 for 循环进行 CPU 上的余弦相似度计算！
- **承压场景**：如果一个事实很重要，但它恰好发生在较早时间且最近没有被访问过，那么它可能在 SQL 预筛选阶段就因为 `LIMIT` 被排除，后面的相似度计算根本看不到它。这意味着当前实现更接近“最近访问优先 + 小窗口语义筛选”，而不是完整的全量向量召回。

因此第四章把 `SemanticStore` 定义为“带向量辅助的分层召回”会比“向量数据库”更准确。它有降级路径，也能在 embedding 不可用时继续工作，但召回上界仍然受 SQLite 预筛选和进程内 O(N) 计算约束。

### 4.3 改进雷达

#### 🔬 ZeroClaw：在记忆 taxonomy 与 nightly consolidation 上更重

ZeroClaw 在这一块比 OpenFang 更明确地把记忆分层：`Core`、`Daily`、`Conversation`、`Custom`。同时它还有 nightly consolidation 任务，会把每天运行结果沉淀回更核心的记忆层。

OpenFang 相比之下的特点是：

- substrate 统一得更彻底
- 但 taxonomy 更轻
- consolidation 目前更偏 decay，而不是 synthesize

这意味着 OpenFang 现在更像“通用底盘”，ZeroClaw 更像“已经带了一部分运行策略”的垂直产品。

#### 🚧 OpenClaw：插件发现与路径去重更成熟

OpenClaw 在插件/扩展发现上更成熟的点，不是它有复杂依赖求解，而是它对来源优先级、路径去重、symlink-aware realpath 处理更细。这个点很值得借鉴，但也要说准确：它强在 **discovery / dedup**，不是真正的依赖图求解器。

#### ⚠️ 差距清单

1. **记忆分类粒度不足**：OpenFang 还没有像 ZeroClaw 那样固定 taxonomy
2. **记忆整合偏弱**：只有衰减，没有真正的 nightly synthesis/merge
3. **Skill 冲突治理不足**：没有版本、依赖、来源冲突仲裁模型
4. **后台调度表达能力有限**：简化 periodic + 基础 proactive，离完整自治编排还远
5. **资源约束闭环不足**：quota 字段很完整，但自治运行时的统一仲裁仍缺位

---

### 4.4 行动项

| 改进项 | 影响力 | 难度 | 代码位置 |
|--------|-------|------|---------|
| **为 memory 引入显式 taxonomy**<br>增加 `Core / Daily / Conversation / User` 这类一级分类，别只靠 scope 和 metadata。 | ⭐⭐⭐⭐ | 中 | `openfang-types::memory` + `semantic.rs` / `substrate.rs` |
| **把 consolidation 从 decay 升级为 merge + summary**<br>保留衰减逻辑，同时引入真正的记忆归并与摘要沉淀。 | ⭐⭐⭐⭐⭐ | 中高 | `consolidation.rs` + `session.rs` |
| **给 SkillRegistry 增加来源优先级与冲突报告**<br>不仅覆盖，还要告诉用户是谁覆盖了谁、为什么。 | ⭐⭐⭐ | 中 | `openfang-skills/src/registry.rs` |
| **为 Skill 加入依赖与版本约束**<br>让技能生态从“HashMap 注册表”进化到可验证的依赖图。 | ⭐⭐⭐⭐ | 高 | `SkillManifest`、loader、registry |
| **把 `Periodic` 升级为更完整的 cron 解析与可观测调度**<br>当前 `every 5m` 足够简单，但不够表达复杂工作流。 | ⭐⭐⭐ | 中 | `background.rs` 的 `parse_cron_to_secs()` |
| **统一自治配额仲裁**<br>把后台 loop 的频率、token、成本、tool_calls 统一接进一个自治执行预算器。 | ⭐⭐⭐⭐⭐ | 高 | `background.rs`、scheduler、metering、supervisor |

---

### 4.5 交叉引用导读 🔗

- 如果你想看记忆与技能最终怎样被主循环消耗和修复，可回看 [openfang_tutorial_chapter_2.md](./openfang_tutorial_chapter_2.md)。
- 如果你想继续追后台 Hands、预算和调度如何进入治理回路，下一站是 [openfang_tutorial_chapter_5.md](./openfang_tutorial_chapter_5.md)。
- 如果你更关注这套长期状态能力在控制面上如何被人类观测和操作，可继续看 [openfang_tutorial_chapter_9.md](./openfang_tutorial_chapter_9.md)。

### 4.6 本章小结

如果把第四章放回整本教程的新框架里，最重要的判断也应该明确收口：

- **从产品现实压力看**，长期记忆、技能生态、后台 Hands 这些能力决定了 OpenFang 能不能从“会对话”进化成“会长期工作”的系统。
- **从运行时底线压力看**，OpenFang 已经具备相当扎实的底盘：`MemorySubstrate`、`SemanticStore`、`SessionStore`、`SkillRegistry`、`openclaw_compat`、`HandRegistry`、`BackgroundExecutor` 这些模块都不是口号，而是已经落成的真实实现。
- **从源码现状看**，真正的缺口并不在“有没有”，而在“闭环够不够硬”：memory taxonomy 还轻、consolidation 仍以 decay 为主、Skill 冲突治理还浅、后台调度表达能力有限、quota 执法仍分散在多子系统。
- **从下一步演进看**，第四章最值得补的不是继续扩写概念，而是把记忆分类、整合策略、skill 冲突报告、后台调度表达力和自治预算仲裁真正收成一条长期运行治理链。

因此，本章的最终结论不是“OpenFang 支持记忆、技能和 Hands”，而是：**OpenFang 已经拥有持续运行系统所需的长期状态底盘，但距离五星标准，还差把这些能力从并列模块推进为真正稳定的长期自治闭环。**

### 4.7 本章记住 3 件事

1. 第四章真正讨论的不是“功能列表”，而是 OpenFang 有没有为长期运行准备好记忆、技能、后台任务这三类状态底盘。
2. `MemorySubstrate`、`SemanticStore`、`SkillRegistry`、`HandRegistry` 和 `BackgroundExecutor` 已经落地，但很多治理能力仍停留在“可用”而非“闭环”。
3. 本章最关键的判断不是“有没有这些模块”，而是“它们的分类、冲突仲裁、调度表达力和预算执法是否已经足够硬”。

### 4.8 最容易误解的点

最容易误解的一点是：**既然 OpenFang 已经有记忆衰减、技能注册和后台 Hands，它就已经具备成熟的长期自治能力。**

并不是这样。当前实现更准确的状态是：长期运行所需的主要模块已经长出来了，但 taxonomy、merge/summarize、冲突仲裁、调度表达和统一预算仲裁还没有完全闭合，所以它更像扎实底盘，而不是已经完成工业化打磨的长期自治系统。

### 4.9 一个验证题

如果有人说“OpenFang 的记忆系统已经等价于完整向量数据库 + 长期知识库”，你会用本章里的哪三条源码证据去反驳这个判断？
