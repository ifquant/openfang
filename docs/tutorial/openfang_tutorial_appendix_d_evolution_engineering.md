## 附录 D：演化工程规格——裂变蓝图、闭环模板与样板设计稿

> **定位**：本附录不是第十章的重复版，而是它的工程规格层。
> 第十章本体负责回答“为什么要演化、应该怎样理解”；本附录负责回答“如果现在真的要做，该按什么结构落地”。

> **版本锚点**：本附录基于 OpenFang 代码快照 `2026-03-07`。
> 如果上游 crate 结构已变化，例如 `tool_runner` 已独立出 `policy_gate`、`agent_loop` 已拆出 `recovery_plane`，请对照最新源码重新校准本文中的文件裂法、调用形态与验证边界。

---

## D.0 怎么使用这份附录

这份附录只服务两类读者：

1. 已经读完第十章本体，准备把判断压成 backlog 或设计任务的人。
2. 正在实现某个演化样板，需要查规格、模板和验收口径的人。

它不是第一次阅读第十章时的主入口，也不是当前源码说明。更准确地说，它是一份：

- **设计规格书**：把教程中的判断压成结构、模板和执行口径。
- **工程手册**：方便查阅，不负责再次解释“为什么重要”。
- **候选路线图**：描述未来可落地的形态，不等于当前仓库已经实现的事实。

为避免误读，下面内容统一遵守 3 条规则：

1. 凡是当前仓库中尚不存在的 Rust 结构、模块名、文件裂法，默认都属于**设计提案，非当前源码**。
2. 凡是讨论“应该怎样拆”，默认都表示**未来候选演化方向**，不是对现状的描述。
3. 本附录尽量不重复第十章本体的动机解释，只保留进入实施所需的最小前提。

---

## D.1 工程决策准则：什么时候优先内求，什么时候优先外生

本节不再重复第十章本体的动机，只提供**排产时直接可用的裁决口径**。

### D.1.1 总原则

先判断当前瓶颈到底是：

1. **内部承载力不足**
2. **外部生态吸纳力不足**

再决定这一轮优先 `内求` 还是优先 `外生`。如果两类信号同时成立，默认按“先补最短板承载力，再扩大外部吸纳面”的顺序执行。

### D.1.2 四条裁决规则

#### 规则 1：如果系统已经知道该接什么，但一接就乱，就优先内求

典型信号包括：

1. 接入后爆炸半径过大。
2. 治理边界不清。
3. 主循环、启动链、总装配点过于肥厚。
4. 局部改动无法局部验证。

这时外部价值不是没有，而是系统还没有资格稳定承接它。

#### 规则 2：如果系统内部已经足够稳，但外部价值迟迟进不来，就优先外生

典型信号包括：

1. 外部生态已经存在成熟能力。
2. 兼容层已经有现实支点。
3. 现有内部实现明显重复造轮子。
4. 接入成本明显高于能力收益。

这时继续纯内生细拆，会让系统越来越完整，但越来越封闭。

#### 规则 3：如果外部生态正在繁荣，就把“支持它”本身视为内生建设目标

当外部能力足够活跃时，外生线不会停留在“接进来”，而会倒逼内求线长出稳定支持结构：

1. `compat`
2. `registry`
3. `verify`
4. `reload`
5. `governance surface`

也就是说，外生越成功，越会反推内部继续生长器官。

#### 规则 4：如果外部生态还很弱，就把“自己先长成生态核心”视为第一任务

这时内求不是保守，而是播种：

1. 先把内部能力做成稳定母体。
2. 先把接口做成未来可外接的形状。
3. 先把治理边界做成未来可扩生态的形状。

### D.1.3 四象限判断表

真正的工程裁决不是二选一，而是下面这张判断表：

| 当前状态 | 优先动作 |
|---------|---------|
| 内部脆弱 + 外部繁荣 | 先补最短板内核支撑，再立即建立对应生态 surface |
| 内部稳健 + 外部繁荣 | 优先外生，把接入和共生规模快速做起来 |
| 内部脆弱 + 外部孱弱 | 优先内求，把自己做成种子生态 |
| 内部稳健 + 外部孱弱 | 选择少数高价值方向做原生突破，同时保留未来外接接口 |

### D.1.4 这会怎样影响“如何拆分”

未来不应再只问“哪个模块最值得拆”，而要同时问两组问题：

**内求问题**：

1. 这个节点是不是当前的承载瓶颈？
2. 拆了之后能不能明显降低爆炸半径？
3. 拆了之后能不能形成新的稳定自由度？

**外生问题**：

1. 这个位置是不是当前最值得长出生态 surface 的地方？
2. 这里有没有正在繁荣的外部能力值得吸纳？
3. 这里的兼容、导入、治理、热更新是否已经值得独立成面？

因此，“拆分”本身要分成两种：

1. **内求式拆分**：把一个粗骨架拆成更细内部自由度。
2. **外生式拆分**：把一个临时兼容入口提升成稳定生态表面。

### D.1.5 这会怎样影响开发计划

如果把上面的准则真正落到排产里，那么任务排序不应再只按“模块重要性”决定，而应先过三步：

1. 标注它属于 `内求`、`外生`，还是双线耦合。
2. 判断它当前面对的是“内部脆弱”还是“外部繁荣/孱弱”。
3. 最后才决定它进入 backlog 的先后顺序。

按照这个口径，当前最自然的外生 / 双线耦合任务包括：

1. `Markdown Surface`
2. `Shell Surface`
3. `OpenClaw Plugin Surface`
4. `MCP Surface`

它们都不是单纯“再接一个接口”，而会反推内部治理与承载结构继续生长。

---

## D.2 闭环模板层：如何把“演化”变成可分发、可验收、可止损的工程流程

本节默认读者已经接受第十章本体给出的 7 步闭环。这里不再解释方法论，只给执行时真正要填的三层模板：

1. **任务分发模板**：问题如何进入 backlog。
2. **验收模板**：变化做完后按什么口径过线。
3. **止损模板**：出现风险后按什么边界及时停下。

### D.2.1 闭环与四平面的关系

这一小节只负责做映射，避免实施时把“闭环步骤”和“能力分层”混成两套体系：

1. **观察 / 判型** -> 对应观察平面。
2. **选线 / 选样板 / 执行** -> 对应执行平面。
3. **验证与治理** -> 对应验证平面 + 治理平面。
4. **沉淀** -> 贯穿四个平面，把一次尝试变成长期演化资产。

因此，四平面不是新增概念，而是对同一条闭环的横切视图。实施时，优先按闭环步骤排任务，再用四平面检查能力缺口。

### D.2.2 任务分发模板

每一个进入 evolution backlog 的任务，在立项时都至少要写清下面 8 个字段：

1. **Signal Source**：问题来自 audit、test failure、runtime anomaly、Markdown drift，还是外部生态机会。
2. **Problem Type**：这是承载力问题、生态吸纳问题，还是双线耦合问题。
3. **Primary Line**：偏内求、偏外生，还是双线耦合。
4. **Candidate Surface / Sample**：准备挂到哪个样板上，例如 `Recovery Plane`、`Markdown Surface`、`MCP Surface`。
5. **Blast Radius**：只影响局部模块、协调层，还是可能触及治理层。
6. **Expected Artifact**：产出是 policy 调整、结构裂变、compat 支持、验证补丁，还是治理规则更新。
7. **Rollback Shape**：失败后如何退回，退回到旧路径、旧配置，还是旧 verdict。
8. **Success Signal**：什么信号算这次任务真的完成，而不是只是把报错压下去。

只要这 8 个字段写不清，这个任务就还不应该进入执行态。

### D.2.3 四类任务的最小实例

为了避免模板空转，下面给出 4 类当前最关键任务的最小实例：

| 任务 | Signal Source | Primary Line | Expected Artifact | Approval Level |
|------|---------------|-------------|------------------|----------------|
| `Markdown Surface` | 教程、身份文件、知识资产长期漂移 | 偏外生 | 分类器、policy、结构校验 | 自动建议补丁或低风险自动写入 |
| `Shell Surface` | shell 使用频繁但审计和分类不足 | 双线耦合 | 命令分类、observe/mutate policy、升级门 | 观察类可自动，改写类需更强治理 |
| `OpenClaw Plugin Surface` | 外部插件生态繁荣，兼容入口已存在但生命周期不够稳 | 偏外生 | intake / convert / lifecycle 边界 | 导入与校验可自动，启用与高权限运行需治理 |
| `MCP Surface` | 远程工具协议接入收益高，但接入、过滤、重载、allowlist 还偏粗 | 偏外生 | admission、tool projection、reload、agent scope policy | 连接建立与工具投影需治理记录 |

### D.2.4 验收模板：四张标准验收卡

演化任务不能只写“做了什么”，还必须在开工前声明“怎么证明没做坏”。这里固定 4 张标准验收卡。

#### 验收卡 A：结构验收卡

适用对象：`Policy Gate`、`Recovery Plane`、`Integration Init`、`Governance Plane` 这类偏内求的结构任务。

至少回答下面 5 个问题：

1. 新边界有没有比旧边界更清楚，而不是只是换了文件位置。
2. 新接口有没有稳定的 context / verdict / policy 形状。
3. 旧路径是否还能渐进共存，而不是强制一次性迁移。
4. 爆炸半径是否比原来更小，局部验证是否更容易。
5. 新结构是否真的形成了新的独立自由度，而不是把耦合换个地方继续藏。

#### 验收卡 B：行为验收卡

适用对象：`Shell Surface`、`MCP Surface`、`OpenClaw Plugin Surface` 这类接入与执行任务。

至少回答下面 5 个问题：

1. 旧行为是否保持兼容，不会因为新 surface 出现明显倒退。
2. 新接入路径是否能被稳定分类、过滤、投影，而不是裸接入。
3. 失败路径是否清楚，可见、可测、可诊断。
4. 健康状态、重试、降级是否有明确语义。
5. 真实使用时，行为收益是否超过新增治理成本。

#### 验收卡 C：资源验收卡

适用对象：所有会消耗 token、CPU、连接数、子进程、网络预算的任务。

至少回答下面 5 个问题：

1. 新路径是否引入明显不可控的资源放大。
2. 预算是否能被量化，而不是只凭感觉判断“应该够用”。
3. 失败重试是否存在上限，而不是隐性自旋。
4. 外部连接或工具调用是否有 timeout / quota / backoff 约束。
5. 演化任务本身是否值得它消耗掉的预算。

#### 验收卡 D：审计验收卡

适用对象：所有会写文件、改配置、接外部生态、触发高风险工具的任务。

至少回答下面 5 个问题：

1. 谁发起了这次任务，最初信号是什么。
2. 它最后被判成了内求、外生，还是双线耦合。
3. 它走了哪一条样板路径，哪些验证门通过或失败。
4. 它最终触发了哪些治理动作，例如 approval、deny、quota stop、hard stop。
5. 事后是否能从记录中回放这次任务为什么成立、为什么失败。

### D.2.5 止损模板：四张标准止损卡

验证解决的是“能不能成立”，治理解决的是“出了问题怎么不失控”。这里固定 4 张标准止损卡。

#### 止损卡 A：低风险文本卡

适用对象：`Markdown Surface`、教程、计划、身份文件、说明文档一类低风险文本任务。

默认口径：

1. 可自动生成建议补丁。
2. 局部结构校验通过后，可在限定目录内自动写入。
3. 一旦 link integrity、frontmatter、保留字段校验失败，立即退回建议模式。
4. 不允许越界触碰代码、发布、密钥、迁移路径。

#### 止损卡 B：中风险兼容卡

适用对象：`OpenClaw Plugin Surface`、`MCP Surface`、外部格式导入、兼容转换、配置 merge。

默认口径：

1. intake / convert / verify 可自动执行。
2. enable / expose / reconnect / cross-agent grant 需要更强治理记录。
3. 兼容失败时优先降级，不优先强推替换旧路径。
4. 一旦健康恶化或投影失真，应自动退回到更窄 scope。

#### 止损卡 C：高风险执行卡

适用对象：`Shell Surface` 里的 mutate 类动作、批量改写、连接外部执行世界的高权限操作。

默认口径：

1. 观察类动作与改写类动作必须分级。
2. 改写类动作默认要求审批或更高等级授权。
3. 连续失败、预算超限、目标漂移时必须立刻停止。
4. 所有高风险执行都必须留下可回放的 audit trail。

#### 止损卡 D：跨层核心卡

适用对象：`agent_loop`、`boot_with_config`、`OpenFangKernel`、治理层边界本身。

默认口径：

1. 不允许一次性全量替换。
2. 必须先建立新壳、新 verdict、新上下文，再做渐进迁移。
3. 必须同时具备结构验收卡、资源验收卡、审计验收卡。
4. 一旦出现状态不一致、旧路径回退失败、治理语义漂移，必须立即停止扩散。

### D.2.6 这三层模板真正解决了什么

模板层真正解决的不是“文档写得更完整”，而是把演化 backlog 与普通功能 backlog 区分开来。

以后进入演化闭环的任务，不再只是“要不要做”，而会天然带着三层边界：

1. **怎么分发**
2. **怎么验收**
3. **怎么止损**

只有这三层同时成立，第十章里的“可进化”才会从未来宣言变成工程秩序。

---

## D.3 自由度地图详表

这一节承接第十章本体中已经压缩掉的工程细节。第十章本体只保留两个判断：

1. OpenFang 已经拥有一批质量不错的演化自由度。
2. 当前最限制进一步并发演化的，不是模块数量不够，而是少数几个总装配点仍然过粗。

如果需要更细地判断“哪些模块已适合并发演化、哪些模块更适合先继续拆薄”，请看下面这张详表。

### D.3.1 当前版本的演化自由度地图

| 自由度等级 | 模块 | 当前判断 | 继续拆分的可能 |
|-----------|------|----------|----------------|
| **高自由度** | `openfang-types` | 纯共享语言层、几乎无业务逻辑，边界非常清晰 | 可继续把高频演化的类型簇拆成更稳定的 capability / tool / event 子层 |
| **高自由度** | `openfang-memory` 子后端（`structured` / `semantic` / `knowledge` / `session` / `consolidation`） | 已经具备后端分层，适合单独演化检索、卫生、会话策略 | 可继续把 memory hygiene、recall policy、compaction policy 从后端实现中再独立出来 |
| **高自由度** | `openfang-skills` 子系统（`loader` / `verify` / `registry` / `openclaw_compat` / `marketplace`） | 导入、加载、校验、兼容、分发职责分离良好 | 可继续拆出 runtime adapter 层，把 Python / Node / WASM skill 执行进一步去耦 |
| **高自由度** | `openfang-channels` 各 adapter | 每个渠道天然是单独演化面，改动爆炸半径较小 | 可继续把每个 adapter 再拆成 ingress / normalize / delivery / retry 子层 |
| **高自由度** | `workflow` / `triggers` | 规则边界、数据结构、验证目标都很清晰 | 可继续把 DSL、planner、executor、history 分层，形成更细工作流自由度 |
| **高自由度** | `audit` / `metering` / `background` | 都属于职责明确的局部治理引擎 | 可继续把事件模型、策略模型、存储模型进一步拆开，便于独立升级 |
| **高自由度** | `tool_policy` / `session_repair` / `context_overflow` / `process_manager` | 这些 runtime 小模块已经很像独立演化原语 | 可继续把规则库、启发式、预算参数、恢复策略表从代码逻辑中再抽离 |
| **中自由度** | `openfang-api`（`routes` / `middleware` / `ws` / `stream_chunker`） | 已有分层，但仍共用同一套 kernel state 和服务语义 | 可继续把 read API、control API、stream API、admin API 拆成更清楚的服务面 |
| **中自由度** | `openfang-channels::bridge` / `router` | 是跨平台统一语义层，天然比单 adapter 更耦合 | 可继续拆成 normalize、debounce、idempotency、routing policy 四层 |
| **中自由度** | `tool_runner` | 已经是独立执行面，但仍把 resolve / validate / sandbox_select / execute / postprocess 强绑定在一条链上 | 可继续拆成 tool planning、policy check、sandbox dispatch、result shaping 四段 |
| **中自由度** | `sandbox` / `subprocess_sandbox` / `docker_sandbox` | 执行边界清楚，但运行时策略和宿主调用仍有汇聚点 | 可继续抽成统一 capability ABI、host call policy、resource budget 层 |
| **中自由度** | `routing` / `provider_health` / `drivers` 周边 | 语义相近，但还没完全收成统一“模型选择自由度” | 可继续把模型选择、冷却、重试、fallback、成本优化拆成独立策略单元 |
| **低自由度** | `OpenFangKernel` 总装配 | 是全系统总线板，不是适合高频演化的小自由度 | 未来应继续把 boot、registry wiring、service composition、lifecycle hooks 拆薄 |
| **低自由度** | `boot_with_config` 启动链 | 初始化顺序和依赖注入高度集中，局部改动容易牵全局 | 可继续拆成 config load、driver init、service init、background init、extension merge 几段 |
| **低自由度** | `agent_loop` 主循环 | 当前是最明显的超级编排点之一，依赖面很广 | 应继续拆成 planning、memory recall、tool-use orchestration、repair、overflow recovery、continuation manager |
| **低自由度** | 跨子系统统一状态汇聚点 | 例如 kernel 中同时挂载 memory / skills / browser / api / extensions / channels 的总持有结构 | 应继续把共享状态按 read model、control model、execution model 分面，减少中心化持有 |

### D.3.2 这张地图背后的拓扑：浅层分层，层内平坦，跨层受限

OpenFang 的自由度结构，不应该被理解成纯平面的模块清单，也不应该走向一座层层抽象、很难落地的高塔。更合理的结构是：

**浅层分层 + 层内平坦 + 跨层受限。**

也就是说：

1. **全局上要有层次**，否则自由度会彼此打架，系统很难收敛。
2. **每一层内部要尽量平坦**，否则 AI 很难高效并发演化局部模块。
3. **跨层修改必须更谨慎**，因为跨层意味着更大的爆炸半径和更高的验证成本。

如果把 OpenFang 当前和未来的演化结构理解成一个统一拓扑，我会把它压成三层：

1. **自由度层**：最适合被演化 agent 直接修改的局部模块，例如 `tool_policy`、`session_repair`、`context_overflow`、各 channel adapter、memory policy、workflow DSL。
2. **协调层**：负责把多个自由度编排起来的中层骨架，例如 `tool_runner`、`bridge/router`、`agent_loop`、`boot_with_config` 的各段初始化流程。
3. **治理层**：负责预算、审计、审批、认证、风险边界和回滚能力，例如 `audit`、`metering`、`approval`、`auth`，以及未来要明确化的 `Governance Plane`。

这条拓扑的实践含义非常直接：

1. 能在自由度层内解决的问题，就不要上推到协调层。
2. 能在协调层内解决的问题，就不要直接碰治理层。
3. 治理层不应吞掉执行细节，但必须约束执行边界。

### D.3.3 高自由度模块：已经适合并发演化的主战场

高自由度模块的共同特征是：职责单纯、输入输出清晰、验证目标稳定、改动后不必重验整套系统。更重要的是，它们大多天然落在**自由度层**，因此最适合保持层内平坦，并作为并发演化的主战场。

这类模块天然支持：

1. 独立观察问题。
2. 独立生成补丁。
3. 独立跑测试或局部验证。
4. 独立回滚。

但即便已经是高自由度，也不意味着不能继续拆。相反，高自由度模块往往最值得继续细化，因为它们的边界已经足够稳，适合从“一个模块”继续裂变成“一个模块内部的多个小自由度”。

例如：

1. `openfang-memory` 可以继续裂成 recall、hygiene、consolidation、retention 四种演化节奏。
2. `openfang-skills` 可以继续裂成 import compatibility、runtime execution、security verify、marketplace policy 四类自由度。
3. `workflow` 可以继续裂成 DSL、planner、executor、history 四层。

### D.3.4 中自由度模块：下一阶段最值得动刀的区域

中自由度模块的问题不是完全没边界，而是：**已经有结构，但局部改动还容易牵动横向语义。** 从拓扑上说，它们大多属于**协调层**，因此比自由度层更需要边界纪律。

它们往往是下一阶段最值得动刀的地方，因为再往前拆一层，就有机会进入高自由度队列。

这一类最典型的几个点是：

1. `openfang-api`：已经拆成 routes、middleware、ws 等层，但仍然围绕同一 kernel in-process 语义运转。
2. `bridge` / `router`：已经比 adapter 层更抽象，但仍把多个跨平台责任捆在一起。
3. `tool_runner`：是一条很强的执行链，但还没有彻底分成可独立进化的策略段。
4. `sandbox` 家族：已经是边界系统，但还可以继续把 capability ABI、resource budget、host call policy 拆开。

所以中自由度模块的核心任务，不是大改重写，而是继续做**职责内裂**。

### D.3.5 低自由度模块：当前最限制自我演化的粗颗粒区域

低自由度模块并不是“不重要”，恰恰相反，它们往往是系统最核心的骨架。但也正因为太核心、太集中，所以暂时不适合被高频并发演化。从拓扑上看，它们是协调层与治理层仍然纠缠、尚未分面的地方。

当前最典型的低自由度区域有两个：

1. `OpenFangKernel` 总装配层。
2. `agent_loop` 主循环。

它们的共同问题不是设计错误，而是承担了太多“总协调”职责：

1. 依赖面太宽。
2. 状态持有太集中。
3. 局部改动难以局部验证。
4. 任何一次拆动都容易牵连多个子系统。

因此，这里的真正目标不是“让 AI 直接去改这些大文件”，而是先继续把这些大节点往下裂变。

### D.3.6 从自由度地图推导出的原则

如果把这张地图翻译成一句更硬的工程原则，那就是：

**未来要提升 OpenFang 的自我演化能力，不能只盯内部裂变，也不能只盯外部生态，而要同时推进两件事：继续把低自由度节点拆薄，把外部高价值生态表面稳定成一等接口。**

换句话说，OpenFang 的未来演化路线里，模块细化本身是一等目标，生态表面稳定化也是一等目标。

因为只有当系统同时拥有越来越多的小自由度，以及越来越稳的环境交换接口，后续我们才可能把不同演化任务并发分派给不同 agent，并在总体上得到一种高并发、低爆炸半径、持续收敛的进化过程。

## D.4 全量裂变蓝图

这一节承接第十章本体中已经压缩掉的裂变方向规格。教程本体只保留三类区域的拆分原则；如果你需要更细地看“每一类模块该沿什么职责边界裂”，看这里。

### D.4.1 高自由度模块：继续细化为更稳定的局部自由度

高自由度模块不是“已经完成”，而是说它们已经具备继续细分的好基础。对未来自我演化系统来说，这类模块最适合优先进入“多 agent 并发维护”的轨道，而且应尽量继续留在自由度层内部裂变，而不是轻易上卷成新的协调层。

当前最值得继续细化的方向包括：

1. `openfang-types` -> capability / tool / event / agent / config 五组稳定协议簇
2. `openfang-memory` -> recall / hygiene / compaction / retention 四组记忆策略
3. `openfang-skills` -> import / registry / runtime adapter / security verify / marketplace 五条演化线
4. `openfang-channels` adapter -> ingress / normalize / delivery / platform quirks 四段式
5. `workflow` / `triggers` -> definition / planner / executor / history / policy 五层
6. `audit` / `metering` / `background` -> observation / budgeting / autonomy scheduling 三类治理自由度
7. `tool_policy`、`session_repair`、`context_overflow`、`process_manager` -> rule tables / heuristics / budgets / recovery modes 参数化策略单元
8. `openfang-wire` -> wire schema / peer transport / registry / routing policy / trust 五层协议簇
9. `openfang-hands` -> capability bundles / activation workflow / requirement check / lifecycle policy / outcome model
10. `openfang-extensions` -> discovery / credentialing / install-render / health-recovery / lifecycle governance 五段安装闭环

这类模块的共同目标，不是“拆得更漂亮”，而是让未来的演化任务可以从“改整个 crate”退化为“只改某一种局部策略或子协议”。

### D.4.2 中自由度模块：继续做职责内裂

中自由度模块的问题通常不是没有结构，而是一个模块里还混着 2-4 种不同节奏的职责。因此这里最合理的拆法不是大重构，而是先做职责内裂，把协调层里过于肥厚的节点压回更平坦的局部自由度网络。

当前最值得继续裂的方向包括：

1. `openfang-api` -> read API / control API / stream API / admin API 四种服务自由度
2. `openfang-channels::bridge` / `router` -> ingress normalize / debounce-idempotency / routing policy / delivery-ack 四段渠道管线
3. `tool_runner` -> tool planning / policy gate / execution dispatch / result shaping / audit hook 五段执行自由度
4. `sandbox` 家族 -> capability ABI / host policy layer / runtime backend layer 三层统一边界
5. `routing` / `provider_health` / `drivers` -> model selection / health scoring / retry-fallback / cost optimizer 四组模型选择自由度

中自由度模块的核心目标，不是追求一次性纯净架构，而是让原本“一条粗链”变成“几段可独立验证的协调面”。

### D.4.3 低自由度核心：把总协调者拆成次级协调平面

低自由度核心最需要的不是“多加点规则”，而是把总协调角色拆成多个次级协调角色。也就是说，先把一个大脑裂变成几个彼此协作的小脑，再谈让演化 agent 去碰它们。

当前最重要的裂变方向包括：

1. `OpenFangKernel` -> `BootAssembler` / `ServiceRegistry` / `ExecutionFacade` / `GovernancePlane` / `IntegrationHub`
2. `boot_with_config` -> config load & normalize / runtime infra init / governance init / execution init / integration init 五段启动管线
3. `agent_loop` -> input preparation / memory recall / planning / tool orchestration / recovery / commit 六个执行自由度
4. 跨子系统统一状态 -> read model / control model / execution model 三种状态视图

这一步的真正目标，不是“让 AI 直接去改大文件”，而是先让这些大文件退化成几个更稳定的协调平面。

### D.4.4 统一拆分纪律

无论是中自由度还是低自由度核心，后续都不应该走“先画一个完美架构图、再一次性重构”的路线。真正可演化的拆分纪律应该是：

1. 每次只拆出一个新自由度。
2. 新自由度必须有明确输入输出契约。
3. 新自由度必须能单独验证。
4. 新自由度拆出来后，旧总线板要立刻变薄，而不是再包一层名字继续堆逻辑。

也就是说，拆分本身也必须符合第十章前面定义的演化原则：小步、可验证、可回滚、可积累。

## D.5 执行顺序与论证

这一节承接第十章本体中已经压缩掉的首轮执行顺序论证。教程本体只保留一句原则：

**先拆最容易形成局部验证闭环的点，再拆真正的总装配核心。**

如果需要更细地讨论“为什么第一刀是 `Policy Gate`，为什么 `Governance Plane` 应该排在第五”，请看下面这份展开版。

### D.5.1 为什么首轮执行顺序不能平均发力

如果按工程现实来排，第一批拆分不应该平均发力，而应该遵守一个很硬的顺序：

**先拆最容易形成局部验证闭环的点，再拆真正的总装配核心。**

原因很直接：如果一开始就去重做 `agent_loop` 或 `OpenFangKernel` 全局骨架，OpenFang 很容易在“为了更可进化”这件事上先把自己拆坏。更稳的路线，是先在中自由度层建立 2-3 个成功样板，再把这种拆分方法推广到低自由度核心。

### D.5.2 第一批实际落地目标

基于当前代码结构，我建议第一批实际落地目标按下面顺序推进。这个顺序本质上也是按层级与边界排出来的：先在自由度层和协调层交界处建立样板，再慢慢推进到协调层核心，最后触碰治理层分面。

#### D.5.2.1 第一顺位：`tool_runner` 先裂出 `Policy Gate`

这是第一批里最应该先动手的点。原因不是它最核心，而是它最容易形成“拆一个点、立刻见效”的正反馈。更重要的是，它是一刀典型的**协调层内裂变**：先把安全判定从执行派发里剥出来，但不跨层打乱整体工具系统。

当前 `execute_tool()` 里同时承担：

1. capability enforcement
2. approval gate
3. taint / shell / net 风险检查
4. 真正的执行派发
5. 结果回包装

这说明 `tool_runner` 已经是一条完整执行链，但安全门和执行门仍然贴得过近。

第一刀最合理的裂法是：

1. 保留 `execute_tool()` 作为总入口。
2. 先抽出一个独立的 `policy_gate` 模块。
3. 把 capability、approval、exec policy、taint precheck 都收进这个 gate。
4. 让 `execute_tool()` 只负责“先过 gate，再 dispatch”。

它最关键的价值，是把 OpenFang 的拆分方法学先做出第一个成功样板。

#### D.5.2.2 第二顺位：`bridge` 裂出 `Debounce / Idempotency`

第二个应当落地的是 `openfang-channels::bridge`。理由很明确：它足够重要，但还没危险到像主循环那样牵一发动全身。这同样是一刀协调层内裂变，只是它针对的是跨平台统一语义节点。

这里最值得单独裂出的，是：

1. 碎消息聚合
2. 重复消息去重
3. 重放保护
4. 时间窗缓冲

也就是说，第二刀不是把 bridge 全拆，而是先抽出一个独立的 `debounce_idempotency` 层，挂在 normalize 之后、route 之前。

#### D.5.2.3 第三顺位：`agent_loop` 先裂出 `Recovery Plane`

到了第三步，才值得去碰主循环。但此时仍然不应该整块改 `run_agent_loop()`，而应该只先挑最适合独立化的一段。

最应该先裂出的，不是 planning，也不是 recall，而是 **Recovery Plane**。原因有三个：

1. 相关逻辑已经部分模块化存在，例如 `session_repair`、`context_overflow`、`loop_guard`、`llm_errors`。
2. 它的职责天然偏“失败后治理”，边界比 planning 更清楚。
3. 一旦独立出来，后续所有 agent loop 的演化都会更安全，因为失败路径先被收整了。

#### D.5.2.4 第四顺位：`boot_with_config` 裂出 `Integration Init`

第四步再去碰启动链。原因是启动链虽然粗，但它天然不适合最先做实验，因为每次改动都要重新验证 boot 全局路径。

不过在 `boot_with_config()` 里面，有一块非常适合作为第一刀：**Integration Init**。

从现有启动顺序看，skills、hands、extensions、MCP server merge、health monitor、web tools 这些外围集成逻辑已经形成了一簇相对独立的初始化片段。它们与 memory / driver / audit / metering 这类基础设施不同，更接近“外围系统装配层”。

#### D.5.2.5 第五顺位：`OpenFangKernel` 裂出 `Governance Plane`

只有在前面几步都跑通之后，才建议去正式动 `OpenFangKernel` 本体。而且第一刀也不该是大规模 service registry 重组，而应该是最“收得住”的那一层：**Governance Plane**。

这一刀的意义，不只是拆模块，而是第一次明确把治理层从总协调对象里分面出来。

### D.5.3 为什么这个顺序是对的

这一批顺序不是拍脑袋排的，而是按下面这条梯度排出来的：

1. **先中自由度，后低自由度**
2. **先局部策略链，后全局装配链**
3. **先失败治理链，后主执行骨架**
4. **先建立拆分成功样板，再碰最大核心**
5. **先层内裂变，后跨层分面**

如果按这个顺序推进，OpenFang 会先获得 5 次质量不错的“拆分演化胜利”，而不是一上来就在最危险的核心层豪赌一次大重构。

### D.5.4 首轮执行顺序（最终版）

第一批真正建议进入 backlog 的顺序，可以明确写成：

1. `tool_runner`：裂出 `Policy Gate`
2. `bridge`：裂出 `Debounce / Idempotency`
3. `agent_loop`：裂出 `Recovery Plane`
4. `boot_with_config`：裂出 `Integration Init`
5. `OpenFangKernel`：裂出 `Governance Plane`

这 5 步如果都能做成，OpenFang 的自我演化能力会发生一个非常关键的变化：

它不再只是“有很多模块”，而是开始真正拥有一批**可被持续拆分、持续验证、持续并发演化的结构样板**。

### D.5.5 把首轮样板放回“内求，外生”双线开发计划里

如果现在用“内求，外生”这条双线方法论重新看前面的执行顺序，那么一个更成熟的开发计划应该写成：

1. **内求线，先建立内部结构样板**
	`Policy Gate`、`Debounce / Idempotency`、`Recovery Plane`、`Integration Init`、`Governance Plane`
2. **外生线，紧接着建立生态表面样板**
	`Markdown Surface`、`Shell Surface`、`OpenClaw Plugin Surface`、`MCP Surface`
3. **双线汇合，形成真实工作流**
	观察 -> 判断当前环境偏“内求”还是偏“外生” -> 选择合适样板 -> 执行 -> 验证 -> 治理 -> 沉淀

这份顺序表真正要避免的是两种偏差：

1. 只会往内部无限拆，却不知道为什么拆。
2. 只会往外部不断接，却没有足够结构来承载。

真正更稳的路线应该是：

**内求负责把系统做成好土壤，外生负责把系统接到繁荣世界；两条线一起，系统才真的会长。**

## D.6 / D.7 样板拆解详表（已归档）

> **⚠️ 文档更新提示：**
> 在早期的教程草案中，本附录包含了 9 个详尽的工程样板拆解表（包括 `Policy Gate` 的 5 类边界、`Memory Store` 的物理表结构、以及 `MCP Surface` 等 4 个生态接入面的字段级设计口径）。
>
> 考虑到 OpenFang 的演进速度极快，这些“过度具体且硬编码”的实现细节极易与最新的 `main` 分支代码产生脱节（即所谓的“代码-文档漂移”效应）。因此，从 2.0 版教程开始，这 9 个样板的详细打样表已被移出主教程，转为以 **Issue / RFC 提案** 的形式在代码库中进行动态维护。
>
> 想要获取这 9 个子系统的最新拆分方案，请查阅 OpenFang 官方代码仓库中的 `rfcs/` 目录或直接追踪对应的 `T-architecture` 核心里程碑工单。本附录的目标即止步于**给出整体拆解模板与时序蓝图（D.2 ~ D.5）**，不再赘述具体的实现伪代码。

**本附录至此结束。**
# OpenFang 工业级进化蓝图：吸收 OpenClaw 工程精髓的 5 阶段演进计划

OpenClaw 的核心价值在于其“**脱壳而出、直面现实**”的工程壳层（Infra 层、渠道层、路由层）；而 OpenFang 的核心价值在于其“**云原生、强一致、严密权限**”的内核中枢（13 个 Crate 构建的系统骨架）。

将 OpenClaw 的工程精髓“移植”并“Rust 化”到 OpenFang 中，是让 OpenFang 从一个“结构优秀的内核”走向“生产级工业标准 OS”的必由之路。以下是基于对两套源码解构后制定的**详细 5 阶段逐步落实计划**。

---

## 阶段一：筑基与免疫系统（Execution & Safety Foundation）

**目标**：解决大模型生成危险指令的问题，将 OpenClaw 中最坚固的 Shell 拦截与 Safe Bin 机制下沉到 OpenFang。

**改动位置**：`crates/openfang-runtime` (工具执行沙箱) 与 `crates/openfang-kernel` (准入裁决)

- **Step 1.1: 静态特征拦截字典 (Command Obfuscation Pattern Ban)**
  - **OpenClaw 来源**：`exec-approvals-analysis.ts`
  - **OpenFang 落地**：在 `openfang-runtime` 中新增 `shell_analyzer` 模块。用 Rust 实现一套纯静态的词法解析器，在命令送入 OS 前，正则匹配并硬拦截 `$()`、反引号、`&` 注入等混淆特征。
- **Step 1.2: Wrapper 链递归解包 (Wrapper Unwrapping)**
  - **OpenClaw 来源**：`exec-command-resolution.ts`
  - **OpenFang 落地**：剥离 `nohup env BASH_ENV=... zsh -lc "npm run test"` 这种洋葱结构，提取出最核心的执行主体 `npm`。
- **Step 1.3: 分段 Safe Bin 与单引号重写 (Safe Bin & Argument Escaping)**
  - **OpenClaw 来源**：`exec-safe-bin-policy.ts`
  - **OpenFang 落地**：对于允许自动执行的命令（如 `cat`, `ls`），在 `openfang-runtime` 构筑一层中间件，将参数强制用 Shell Quote 重新包裹（例如使用 Rust 的 `shell-escape` crate），防止利用参数漏洞逃逸。

## 阶段二：感官与触角扩展（Channels & Multi-Modal Routing）

**目标**：打破 OpenFang 目前可能受限的 I/O 边界，将 OpenClaw 成熟的多渠道接入和流式控制吸收进来。

**改动位置**：`crates/openfang-api` (REST/WS 接驳) 与新建 `crates/openfang-channel` (或由 kernel 承担适配抽象)

- **Step 2.1: 协议适配与抹平整型 (Protocol Adapter)**
  - **OpenClaw 来源**：第八章（多渠道适配）
  - **OpenFang 落地**：在 `openfang-api` 前构筑统一的 `ChannelAdapter` Trait。不论是 Telegram、Slack 还是内部 WS，都解析成统一的 `AgentMessage` (文本、图片、引用、发送人鉴权)。
- **Step 2.2: SSE 流式响应蓄水池与截断器 (Chunking & UTF-8 Safety)**
  - **OpenClaw 来源**：第二章（流式输出处理）
  - **OpenFang 落地**：大模型吐出的 Stream 可能是非法的 UTF-8 字节切片。在 `openfang-kernel` 吐出 Event 给前端时，使用 Rust 的 `std::str::from_utf8` 加 buffer 机制，确保只有完整的 Emoji 或多字节字符才被推送，防止前端乱码。
- **Step 2.3: 会话锚定路由与幂等去重 (Idempotent Routing)**
  - **OpenClaw 来源**：`routing` 模块
  - **OpenFang 落地**：引入 Message ID 缓存池，处理外部 IM（如 WhatsApp）的重复 webhook 投递；同时基于回话指针（Session Key）正确路由长期离线后重连的对话。

## 阶段三：长治久安的运维骨架（State, Migration & Cost）

**目标**：确保 OpenFang 能在无人值守的服务器上长期运行，版本升级不丢零碎状态。

**改动位置**：`crates/openfang-memory` (持久化基底) 与 `crates/openfang-kernel`

- **Step 3.1: 扁平化状态的原子迁移 (Atomic State Migrations)**
  - **OpenClaw 来源**：`state-migrations.ts`
  - **OpenFang 落地**：在核心启动链路中加入严肃的 Migration Pipeline。由于 OpenFang 已有 SQLite，这里不仅是 DB migration，还包括配置目录 `.openfang/state` 中 JSON 结构破坏性变更的升级机制。
- **Step 3.2: 会话级 Token 计费与限流 (Cost Usage & Metering)**
  - **OpenClaw 来源**：`session-cost-usage.ts`
  - **OpenFang 落地**：在 `openfang-kernel` 现有的 MeteringEngine 中扩展“流式解析能力”。利用 Rust 的 Zero-copy 解析器在落盘 JSONL 审计日志的同时，精确计算 input/output/custom tools 的金额账单。

## 阶段四：物理互联与发现网络（Discovery & Lifecycle）

**目标**：让 OpenFang 去中心化，能在局域网（家庭/办公室）内无感知互联。

**改动位置**：`crates/openfang-wire` (P2P 协议) 或新建 `crates/openfang-net`

- **Step 4.1: mDNS / Bonjour 局域网广播**
  - **OpenClaw 来源**：`bonjour.ts`
  - **OpenFang 落地**：集成 `mdns` 或 `zeroconf` 库，让运行在树莓派或 NAS 上的 OpenFang Daemon 自动在局域网广播 `_openfang._tcp` 服务。
- **Step 4.2: 进程哨兵与僵尸树清理 (Process Respawn)**
  - **OpenClaw 来源**：`restart.ts`, `process-respawn.ts`
  - **OpenFang 落地**：如果 OpenFang 启动了第三方插件子进程（尤其是那些用不同语言编写的、容易遗留僵尸的），需要在 Rust 这边维护类似 PID namespace 或严密的 PTY 会话树，主进程退出或重启时发送精确的 SIGTERM/SIGKILL。

## 阶段五：生态扩展面重塑（Extension API Boundary）

**目标**：将 OpenClaw 的插件灵活性用 Rust 的高性能 FFI 或 WASM/Node JS 引擎包等方式安全接入。

**改动位置**：`crates/openfang-extensions`

- **Step 5.1: 官方插件合同与宿主 API 桥接 (Plugin Manifest & API)**
  - **OpenClaw 来源**：第十一章（微内核装配线）
  - **OpenFang 落地**：通过 JSON-RPC 或直接内嵌 `deno_core` / `boa_engine`（正如之前探讨的 Native Plugins），暴露标准的 `openfang.api.*` 给插件侧，但将实质的执行拦截把控在 Rust 的 `openfang-kernel` 内。

---

## 实施建议 (Actionable Advice)

从工程落地的优先级来看，不应该“全面铺开”。建议的**着手路线**是：

1.  **先做阶段三的 Step 3.2 (计费与限流)**：这会直接完善 OpenFang 现有的 `MeteringEngine` 核算面，补齐其云原生管理的基本盘。
2.  **次做阶段一 (执行安全边界)**：提取 OpenClaw 的词法规则并在 Rust 中写出高性能版本，这是立竿见影的亮点，可以直接替换现有的脆弱正则或粗放校验。
3.  **再做阶段二 (渠道接入规范化)**：当需要将这个引擎推向生产给不同 IM 用时，抹平渠道差异最为迫切。
# OpenFang 工业级进化蓝图：从“9面体”到“1000面体”的工程折叠矩阵

之前的“5阶段计划”如果说是一个只画了主骨架的“9面正方体”，那么真正的工业级 Agent OS（如我们在 OpenClaw 中所见，要在数十种极端环境下保持韧性），必须是一个充满容错、退化、限流边界的“1000面多面体”。

本节不再按简单的“模块 A 加个功能 B”来罗列，而是将整个 OpenFang 系统掷入一个包含 **控制流 (Control Flow)**、**数据流 (Data Flow)**、**时间轴 (Temporal)** 与 **异常面 (Failure Surface)** 的四维空间，把 OpenClaw 的工程经验拆解为横向的、极度饱满的实施面。

---

## 维度一：执行面的极限折叠 (The Execution Surface)

在理想状态下，大模型给出一个命令，系统执行它并返回结果（这是 Demo 级，只有 1 个面）。但在真实的 OpenClaw/OpenFang 中，这一层包含多少个面？

### 1-A. 意图到原语的安全降级 (Intent-to-Primitive Degradation)

- **AST 级清洗（非单层正则）**：不能只拦截 `$()`，必须应对 Bash、Zsh、PowerShell（如果是跨平台）的变量展开 `eval $(echo rm -rf /)`、花括号展开 `{cat,/etc/passwd}`。在 Rust 中，需要构建或引入轻量级 Shell Parser 来还原真实执行意图。
- **Wrapper 穿透的深度限制**：如果遇到 `env A=1 sh -c "zsh -c 'npm run'"`，解包不能无限递归，必须设定最大深度（如 3 层），超出直接判阻。
- **Safe Bin 的参数白名单与互斥锁**：`git clone` 是安全的，但 `git clone --upload-pack='...'` 是危险的。每一个被放行的二进制文件，都必须在 Rust 层挂载一个对应的参数验证 Profile。

### 1-B. 运行时隔离与资源熔断 (Runtime Isolation & Circuit Breaking)

- **沙箱逃逸的物理防线**：进程不仅要被拦截，执行时还必须有 PTY（伪终端）隔离，确保 `stdin/stdout` 不会让终端挂起。
- **IO/内存/时间的暴力熔断**：当 `cat` 了一个 2GB 的日志文件时，不能等文件读完发给大模型才触发 `Context Exceeded`。必须在 Rust 层的 stdout 管道（Pipe）设置 `MaxBytes`，一旦超过直接发送 `SIGKILL` 截断进程，并返回“输出已被 OOM 防御截断”。
- **进程树清理 (Zombie Process Sweeper)**：当 OpenFang Kernel 认为一个 Task 超时并终止它时，如果它是一个生了子进程的服务（如由 npm 启动的 webpack），单纯 `kill PID` 会遗留孤儿。必须通过 Process Group ID (PGID) 实现一击全杀。

---

## 维度二：数据流与感官通道的噪音过滤 (I/O & Channel Surface)

模型不能被未处理的现实噪音淹没。

### 2-A. 输入塑形与去重 (Inbound Traffic Shaping)

- **高并发下的消息折叠**：用户在 WhatsApp 里 3 秒内发了 5 条消息：“嗨”、“帮我查个东西”、“不用查了”、“还是查吧”、“查今天的天气”。不能触发 5 次大模型思考。需要在 Rust 接入层设置一个 `Debounce/Accumulator Window`（例如 2.5 秒），将碎片消息打包成单次上下文。
- **多端时序颠倒修复**：如果同一个会话在桌面端和移动端同时产生输入，导致网络到达顺序与真实发生顺序不一致，必须依赖本地 Vector Clock 或递增 Sequence ID 进行序列重排序，否则模型会“精神分裂”。

### 2-B. 响应截流与协议投射 (Outbound Protocol Projection)

- **字符级的脏数据防御**：由于流式生成，大模型可能会吐一半的 UTF-8 字符（Emoji 被劈开）。`openfang-api` 在转发 SSE 时，必须包含 UTF-8 边界对齐逻辑。
- **通道降级策略**：如果模型生成了一个巨大的 Markdown 表格，但投递目标是 SMS 短信（不支持换行和表格），在出站前必须有 `downgrade_to_plain_text()` 的过滤钩子。

---

## 维度三：时间轴与记忆的压实 (Temporal & Memory Surface)

Agent 是在时间长河中运行的，它不能被自己的记忆撑死。

### 3-A. 记忆栈的代际淘汰 (Generational Memory Compaction)

- **滑动窗口与冷热分离**：SQLite 中不能无限叠加 Session Logs。必须实现类似 OpenClaw 的 Compaction 机制：当一轮对话超过 100 回合，系统自动触发一个旁路模型调用，将前 80 回合总结为一段高密度 `Semantic Summary`，并打上 `[COMPACTED]` 墓碑标记。
- **身份连续性（Identity Continuity）**：如果用户更换了设备（换了 MAC 地址或 Token），如何证明“手机上的我”和“电脑上的我”是同一个人？需要设计一套 `Canonical Master Key` 绑定机制，允许会话跨设备平滑迁移。

### 3-B. 运维时间线的无感延续 (Lifecycle Continuity)

- **状态的原子回滚（Atomic State Migration rollback）**：当 OpenFang 升级，执行 V1 跨 V2 的数据库 Schema 变更时，如果中途断电，重启后必须能用预留的 `rollback_log` 恢复到 V1。
- **配置热重载（Zero-downtime hot reload）**：修改了允许执行的工具白名单 `tools.yml`，不应当重启整个 Daemon。Kernel 必须监听 `fs::notify`，在不打断当前 Agent Loop 的情况下安全替换底部的能力图谱。

---

## 维度四：治理与审计面板 (Governance & Audit Surface)

系统不仅要能跑，还要能在事故发生后“定责”。

### 4-A. 预算控制层 (Metering & Quota Enforcer)

- **不仅仅是记账，是拦截**：计算出花费了多少 Token 是事后诸葛亮。更复杂的是“前置阻断”：在调用一个昂贵工具前，预估可能消耗的 Context 成本，如果超越了当前 Session 配额，直接拒绝执行并提示 Human-in-the-loop 充值或授权。
- **多租户隔离计费**：如果未来 OpenFang 被当作 ServerOS 使用，必须严格将底层 SQLite 的 Token表 按照 Tenant ID 隔离。

### 4-B. 不可变审计链 (Immutable Audit Trails)

- **环境快照绑定**：在记录一条危险命令的执行日志时，光记下 `rm -rf /` 是不够的。必须把当时的 `CWD`、`EnvHash`、调用这条命令是出于哪个模型的推理（Model Name + Prompt Hash）一并签名固化。任何在界面上审批通过的人类，也必须留下指纹。

---

## 如何逐步消化这 1000 个面？

扩充为“1000面体”后，这不再是一个“几周内写完的 PR”，而是一部漫长的**系统进化史**。

要将其融入刚才的 5 阶段模型中，我们在接下来的细化中会采取 **“切片穿刺法”(Slice Piercing)**：

- 当我们选择“**阶段一：筑基与免疫系统**”开工时，我们不仅要写一个简单的白名单，我们要**同时切穿**上述的 1-A（特征字典）、1-B（进程树清理、IO 截断）、4-B（环境快照绑定），把关于“执行”的这几十个面一次性在 Rust 里做透，形成一个铁桶般的模块。
- 然后我们再用同样的思维切穿“计费”、“多渠道”等下一片空间。
# OpenFang 工业级进化蓝图：吸收 OpenClaw 工程精髓的 5 阶段演进计划

OpenClaw 的核心价值在于其“**脱壳而出、直面现实**”的工程壳层（Infra 层、渠道层、路由层）；而 OpenFang 的核心价值在于其“**云原生、强一致、严密权限**”的内核中枢（13 个 Crate 构建的系统骨架）。

将 OpenClaw 的工程精髓“移植”并“Rust 化”到 OpenFang 中，是让 OpenFang 从一个“结构优秀的内核”走向“生产级工业标准 OS”的必由之路。以下是基于对两套源码解构后制定的**详细 5 阶段逐步落实计划**。

---

## 阶段一：筑基与免疫系统（Execution & Safety Foundation）

**目标**：解决大模型生成危险指令的问题，将 OpenClaw 中最坚固的 Shell 拦截与 Safe Bin 机制下沉到 OpenFang。

**改动位置**：`crates/openfang-runtime` (工具执行沙箱) 与 `crates/openfang-kernel` (准入裁决)

- **Step 1.1: 静态特征拦截字典 (Command Obfuscation Pattern Ban)**
  - **OpenClaw 来源**：`exec-approvals-analysis.ts`
  - **OpenFang 落地**：在 `openfang-runtime` 中新增 `shell_analyzer` 模块。用 Rust 实现一套纯静态的词法解析器，在命令送入 OS 前，正则匹配并硬拦截 `$()`、反引号、`&` 注入等混淆特征。
- **Step 1.2: Wrapper 链递归解包 (Wrapper Unwrapping)**
  - **OpenClaw 来源**：`exec-command-resolution.ts`
  - **OpenFang 落地**：剥离 `nohup env BASH_ENV=... zsh -lc "npm run test"` 这种洋葱结构，提取出最核心的执行主体 `npm`。
- **Step 1.3: 分段 Safe Bin 与单引号重写 (Safe Bin & Argument Escaping)**
  - **OpenClaw 来源**：`exec-safe-bin-policy.ts`
  - **OpenFang 落地**：对于允许自动执行的命令（如 `cat`, `ls`），在 `openfang-runtime` 构筑一层中间件，将参数强制用 Shell Quote 重新包裹（例如使用 Rust 的 `shell-escape` crate），防止利用参数漏洞逃逸。

## 阶段二：感官与触角扩展（Channels & Multi-Modal Routing）

**目标**：打破 OpenFang 目前可能受限的 I/O 边界，将 OpenClaw 成熟的多渠道接入和流式控制吸收进来。

**改动位置**：`crates/openfang-api` (REST/WS 接驳) 与新建 `crates/openfang-channel` (或由 kernel 承担适配抽象)

- **Step 2.1: 协议适配与抹平整型 (Protocol Adapter)**
  - **OpenClaw 来源**：第八章（多渠道适配）
  - **OpenFang 落地**：在 `openfang-api` 前构筑统一的 `ChannelAdapter` Trait。不论是 Telegram、Slack 还是内部 WS，都解析成统一的 `AgentMessage` (文本、图片、引用、发送人鉴权)。
- **Step 2.2: SSE 流式响应蓄水池与截断器 (Chunking & UTF-8 Safety)**
  - **OpenClaw 来源**：第二章（流式输出处理）
  - **OpenFang 落地**：大模型吐出的 Stream 可能是非法的 UTF-8 字节切片。在 `openfang-kernel` 吐出 Event 给前端时，使用 Rust 的 `std::str::from_utf8` 加 buffer 机制，确保只有完整的 Emoji 或多字节字符才被推送，防止前端乱码。
- **Step 2.3: 会话锚定路由与幂等去重 (Idempotent Routing)**
  - **OpenClaw 来源**：`routing` 模块
  - **OpenFang 落地**：引入 Message ID 缓存池，处理外部 IM（如 WhatsApp）的重复 webhook 投递；同时基于回话指针（Session Key）正确路由长期离线后重连的对话。

## 阶段三：长治久安的运维骨架（State, Migration & Cost）

**目标**：确保 OpenFang 能在无人值守的服务器上长期运行，版本升级不丢零碎状态。

**改动位置**：`crates/openfang-memory` (持久化基底) 与 `crates/openfang-kernel`

- **Step 3.1: 扁平化状态的原子迁移 (Atomic State Migrations)**
  - **OpenClaw 来源**：`state-migrations.ts`
  - **OpenFang 落地**：在核心启动链路中加入严肃的 Migration Pipeline。由于 OpenFang 已有 SQLite，这里不仅是 DB migration，还包括配置目录 `.openfang/state` 中 JSON 结构破坏性变更的升级机制。
- **Step 3.2: 会话级 Token 计费与限流 (Cost Usage & Metering)**
  - **OpenClaw 来源**：`session-cost-usage.ts`
  - **OpenFang 落地**：在 `openfang-kernel` 现有的 MeteringEngine 中扩展“流式解析能力”。利用 Rust 的 Zero-copy 解析器在落盘 JSONL 审计日志的同时，精确计算 input/output/custom tools 的金额账单。

## 阶段四：物理互联与发现网络（Discovery & Lifecycle）

**目标**：让 OpenFang 去中心化，能在局域网（家庭/办公室）内无感知互联。

**改动位置**：`crates/openfang-wire` (P2P 协议) 或新建 `crates/openfang-net`

- **Step 4.1: mDNS / Bonjour 局域网广播**
  - **OpenClaw 来源**：`bonjour.ts`
  - **OpenFang 落地**：集成 `mdns` 或 `zeroconf` 库，让运行在树莓派或 NAS 上的 OpenFang Daemon 自动在局域网广播 `_openfang._tcp` 服务。
- **Step 4.2: 进程哨兵与僵尸树清理 (Process Respawn)**
  - **OpenClaw 来源**：`restart.ts`, `process-respawn.ts`
  - **OpenFang 落地**：如果 OpenFang 启动了第三方插件子进程（尤其是那些用不同语言编写的、容易遗留僵尸的），需要在 Rust 这边维护类似 PID namespace 或严密的 PTY 会话树，主进程退出或重启时发送精确的 SIGTERM/SIGKILL。

## 阶段五：生态扩展面重塑（Extension API Boundary）

**目标**：将 OpenClaw 的插件灵活性用 Rust 的高性能 FFI 或 WASM/Node JS 引擎包等方式安全接入。

**改动位置**：`crates/openfang-extensions`

- **Step 5.1: 官方插件合同与宿主 API 桥接 (Plugin Manifest & API)**
  - **OpenClaw 来源**：第十一章（微内核装配线）
  - **OpenFang 落地**：通过 JSON-RPC 或直接内嵌 `deno_core` / `boa_engine`（正如之前探讨的 Native Plugins），暴露标准的 `openfang.api.*` 给插件侧，但将实质的执行拦截把控在 Rust 的 `openfang-kernel` 内。

---

## 实施建议 (Actionable Advice)

从工程落地的优先级来看，不应该“全面铺开”。建议的**着手路线**是：

1.  **先做阶段三的 Step 3.2 (计费与限流)**：这会直接完善 OpenFang 现有的 `MeteringEngine` 核算面，补齐其云原生管理的基本盘。
2.  **次做阶段一 (执行安全边界)**：提取 OpenClaw 的词法规则并在 Rust 中写出高性能版本，这是立竿见影的亮点，可以直接替换现有的脆弱正则或粗放校验。
3.  **再做阶段二 (渠道接入规范化)**：当需要将这个引擎推向生产给不同 IM 用时，抹平渠道差异最为迫切。
