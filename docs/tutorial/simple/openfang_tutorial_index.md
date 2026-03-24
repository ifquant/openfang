# OpenFang 完整教程系列 —— 目录与导航

> 从零开始理解 OpenFang：一个 137K 行代码的 Agent 操作系统，如何通过模块化、自主代理、纵深防御，将 Agent 从"被动聊天机器人"进化成"24/7 自主员工"。

---

## 📚 教程全景

本教程系列分为 **3 大部分，9 个章节**，共 15,000+ 字的深度解析。

```
OpenFang 完全解析教程
│
├─ 第一部分：架构哲学与设计对比
│  ├─ 第零章：哲学转折点 —— 从"框架"到"操作系统"
│  │  ├─ 0.1 ZeroClaw 的单体架构
│  │  ├─ 0.2 OpenFang 的微内核架构
│  │  └─ 0.3 为什么这个进化是必需的
│  │
│  └─ 第一章：13 个 Crates 的微内核架构
│     ├─ 1.1 分层逻辑与依赖关系
│     ├─ 1.2 13 个 Crates 的职责清单
│     └─ 1.3 核心接口与 Trait 设计
│
├─ 第二部分：核心执行引擎深潜
│  ├─ 第四章：Agent Loop —— OpenFang 的心脏
│  │  ├─ 4.1 单次推理的完整生命周期
│  │  ├─ 4.2 代码架构详解
│  │  ├─ 4.3 容错机制：OpenFang vs ZeroClaw
│  │  └─ 4.4 速率限制与重试策略
│  │
│  └─ 第五章：44 个专业子模块详解
│     ├─ 5.1 工具执行与沙盒
│     ├─ 5.2 网络工具（搜索、爬虫、缓存）
│     ├─ 5.3 媒体与多模态
│     └─ 5.4 其他关键模块
│
├─ 第三部分：Hands 与生态完整性
│  ├─ 第七章：Hands —— OpenFang 的杀手级创新
│  │  ├─ 7.1 什么是 Hands？
│  │  ├─ 7.2 Lead Hand 深潜
│  │  ├─ 7.3 Collector Hand：OSINT 监控
│  │  └─ 7.4 Browser Hand：Web 自动化 + 支付审批
│  │
│  ├─ 第八章：生态与扩展系统
│  │  ├─ 8.1 OpenFang 的完整生态
│  │  └─ 8.2 如何编写自定义 Hand
│  │
│  └─ 第九章：ZeroClaw vs OpenFang —— 终极对比
│     ├─ 9.1 能力矩阵
│     ├─ 9.2 使用场景决策树
│     └─ 9.3 混合部署方案
│
└─ 附录
   ├─ 参考文献
   └─ 快速参考卡片
```

---

## 🎯 阅读指南

### 📍 按学习阶段阅读

#### **初级：快速上手（15 分钟）**

1. 阅读第零章 (0.1-0.2 节)：理解 ZeroClaw vs OpenFang 的根本区别
2. 阅读第一章 (1.1 节)：了解 13 个 Crates 的基本分层
3. 快速浏览第九章 (9.2 节)：选择是否适合你的场景

**产出**：了解"OpenFang 是什么"和"它为什么存在"

---

#### **中级：深入理解（1-2 小时）**

1. 第一章全文：理解完整的 13 Crates 体系和 Trait 设计
2. 第四章 (4.1-4.3 节)：掌握 Agent Loop 的执行流程和容错机制
3. 第七章 (7.1-7.2 节)：了解 Hands 的概念和 Lead Hand 的实现

**产出**：能够解释 OpenFang 的核心创新，知道如何编写一个 Hand

---

#### **高级：生产部署（2-4 小时）**

1. 第二部分全文：掌握 44 个运行时模块的细节
2. 第七、八章全文：深入 7 个预置 Hands，学会编写自定义 Hand
3. 第九章 (9.3 节)：设计适合自己的混合部署方案

**产出**：能够部署生产级 OpenFang 系统，扩展新功能

---

### 📍 按使用场景阅读

#### **"我想在 IoT 设备上跑 Agent"**

→ 主要读 ZeroClaw 文档，参考第零章了解何时适合 OpenFang

#### **"我要构建企业内部 Agent 中台"**

→ 第一、四、七、八、九章（跳过第五章的细节）

#### **"我是 Agent 框架开发者，想学习最佳实践"**

→ 第一、二、四、五章（详细阅读所有代码示例）

#### **"我想写自定义 Hands"**

→ 第七、八章（重点）+ 第一章了解 Traits

#### **"我要做技术选型决策"**

→ 第零章 + 第九章（skip 其他部分）

---

## 🔑 核心要点速记

### 三句话理解 OpenFang

1. **架构**：13 个 Crates 的微内核，而非单体
2. **创新**：Hands —— 24/7 自主运行的代理
3. **优势**：模块化（易维护）+ 自主性（无人值守）+ 安全（纵深防御）

### 与 ZeroClaw 的本质区别

| 维度 | ZeroClaw | OpenFang |
|------|----------|----------|
| **架构** | 单体 48K 行 | 微内核 13 Crates 137K 行 |
| **运行模式** | 被动（API 触发） | 主动（事件驱动 + Hands） |
| **目标用户** | 个人/创业 | 企业/生产 |
| **核心竞争力** | 极致性能 + 容错深度 | 模块化 + 自主代理 |

### 13 个 Crates 快速记忆

```
底层：openfang-types (Traits) + openfang-wire (序列化)
     ↓
中层：openfang-memory (RAG) + openfang-runtime (Agent Loop) + openfang-kernel (事件总线)
     ↓
上层：openfang-hands (自主代理) + openfang-api (REST API) + openfang-cli (终端)
     ↓
外围：openfang-channels (40+ IM) + openfang-skills (技能库) + openfang-extensions (WASM)
     ↓
辅助：openfang-migrate (数据迁移) + openfang-desktop (GUI)
```

### 7 个 Hands 的职责

| Hand | 何时运行 | 主要工具 |
|------|---------|--------|
| **Lead** | 每日 6 AM | 网络搜索、爬虫、AI 评分 |
| **Collector** | 24/7 | 变更检测、OSINT、NLP |
| **Researcher** | 手动触发 | 多源搜索、信息交叉、APA 格式 |
| **Predictor** | 每日 2 次 | 数据聚合、统计推理 |
| **Clip** | 手动触发 | YouTube DL、FFmpeg、字幕生成 |
| **Twitter** | 每日 3 次 | 内容生成、发布 API、监控 |
| **Browser** | 按需 | Playwright、支付审批 |

### 4 层错误恢复（Agent Loop）

```
第一层：标准 JSON 解析
  ↓ (失败)
第二层：各厂商特殊格式 (MiniMax/DeepSeek/GLM)
  ↓ (失败)
第三层：正则表达式回收
  ↓ (失败)
第四层：返回空列表，让 Agent 重新生成
```

### 14 层防御体系（简化版）

```
输入验证 → 类型检查 → 权限检查 → 审批流程 → 工具白名单
  ↓
沙盒隔离（WASM/Docker）→ Fuel 计量 → 超时控制 → 内存限制
  ↓
审计链（Merkle Hash）→ 速率限制 → 密钥零化
```

---

## 📖 章节内容速览

### 第一部分：架构与设计

#### 第零章 总结

**问题**：为什么要从单体演进为微内核？

**答案**：
- 编译隔离：改一个 Crate 不触发整个项目重编
- 团队扩展：3 个小组可以并行开发 3 个 Crates
- 安全隔离：大模型的错误被沙盒限制，不会传播
- 模块替换：不满意记忆系统？换一个 openfang-memory 实现

**ZeroClaw 的代价**：单体导致的"牵一发动全身"在企业使用中越来越明显。

#### 第一章 总结

**13 个 Crates 的分工**：

1. **基础层**（2 个）
   - openfang-types：零依赖的 Traits 与类型
   - openfang-wire：序列化协议

2. **执行层**（5 个）
   - openfang-runtime：2,942 行 Agent Loop
   - openfang-memory：向量库 + RAG
   - openfang-kernel：事件总线与调度
   - openfang-hands：7 个预置自主代理
   - openfang-skills：技能库系统

3. **接入层**（3 个）
   - openfang-api：140+ REST/WS 接口
   - openfang-channels：40+ 通讯渠道
   - openfang-extensions：WASM 沙盒

4. **应用层**（2 个）
   - openfang-cli：交互式终端
   - openfang-desktop：Tauri 桌面应用

5. **辅助层**（1 个）
   - openfang-migrate：数据迁移

**关键洞察**：单向依赖 + Trait 抽象 = 完美的可扩展性

---

### 第二部分：执行引擎

#### 第四章 总结

**Agent Loop 的 6 个阶段**：

1. **记忆回忆**：向量搜索 → 全文搜索（降级链）
2. **上下文预算**：自动截断历史确保不超过 Token 限制
3. **提示词构建**：动态组合系统提示、记忆、历史、工具定义
4. **Agent Loop**（最多 50 迭代）：
   - Think：调用 LLM
   - Act：执行工具（最多 4 层沙盒）
   - Observe：反馈给 Agent
5. **响应清理**：提取 [[silent]] 等指令
6. **会话保存**：持久化 + 向量索引

**容错机制的对比**：
- ZeroClaw：6 级解析链（更深）
- OpenFang：4 级解析链（更简洁，但可扩展）

**两者都有**：27 个工具别名、参数修复、URL 包装、Schema 清洗

#### 第五章 总结

**44 个运行时模块的分类**：

1. **工具执行**（4 个）：tool_runner, subprocess_sandbox, docker_sandbox, wasm_sandbox
2. **网络工具**（5 个）：web_search, web_fetch, web_cache, web_content, web_fetch
3. **媒体处理**（3 个）：media_understanding, image_gen, tts
4. **错误恢复**（4 个）：context_overflow, llm_errors, retry, session_repair
5. **优化加速**（3 个）：context_budget, compactor, web_cache
6. **其他**（25 个）：审计、认证、MCP、Playwright、Python 运行时等

**设计智慧**：每个模块都可被独立替换或删除

---

### 第三部分：Hands 与生态

#### 第七章 总结

**Hands 的革命性**：

| 特性 | ZeroClaw | OpenFang |
|------|----------|----------|
| 被动等待用户输入 | ✅ 是 | ❌ 否 |
| 24/7 自主运行 | ❌ 否 | ✅ 是（Hands）|
| 时间表驱动 | ❌ 仅基础 Cron | ✅ 完整 Cron + 事件驱动 |
| 预置生产级任务 | ❌ 无 | ✅ 7 个 Hands |

**Lead Hand 案例**：
- 清单：HAND.toml（声明依赖、时间表、参数）
- 专业知识：SKILL.md（ICP 评分框架、搜索策略）
- System Prompt：500+ 字的操作手册
- 审批规则：敏感操作需人工确认
- 执行流程：搜索 → 爬虫 → 提取 → 评分 → 去重 → 报告

**支付审批的三层确认**（Browser Hand）：
1. Agent 试图执行支付操作
2. 系统发送通知："花 $99.99 确认？"
3. 用户确认后，再次确认："这不可逆，再次确认？"
4. 仅在用户二次确认后才执行

#### 第八章 总结

**如何扩展 OpenFang**：

1. **编写 Rust 结构体**：实现 Hand trait
2. **编写清单**：HAND.toml（声明配置、时间表、工具）
3. **编写专业知识**：SKILL.md（领域知识注入）
4. **注册到 Kernel**：hand_registry.register()
5. **通过 CLI 启用**：openfang hand activate xxx

**可扩展点**：
- 新 Channel（Slack 插件、Teams 集成）
- 新 Skill（行业知识库）
- 新 Hand（自定义自主任务）
- 新 Tool（WASM 或 Docker）
- 新 Provider（支持新 LLM）

#### 第九章 总结

**使用场景决策树**：

```
ZeroClaw ✅ 如果：
  - 边缘设备（10ms 启动，5MB 内存）
  - 开源模型（容错最深）
  - 单人项目（代码简洁）

OpenFang ✅ 如果：
  - 企业生产（模块化、多人开发）
  - 24/7 自主（Hands）
  - 跨平台（WASM 沙盒）
  - 高并发（>100 req/s，DashMap 无锁）
  - Web 自动化（Browser Hand）
```

**混合方案**：OpenFang Kernel（协调）+ 多个 ZeroClaw 边缘节点（极速）

---

## 🚀 快速参考卡片

### 编译 & 测试

```bash
# 完整编译（13 个 Crates）
cargo build --workspace --release

# 运行测试（1,767 个）
cargo test --workspace

# Clippy 检查（零警告）
cargo clippy --workspace --all-targets -- -D warnings
```

### 启动 OpenFang

```bash
# 初始化配置
openfang init

# 启动守护进程
openfang start

# 进入交互式 Shell
openfang shell

# 启用某个 Hand
openfang hand activate lead

# 查看日志
openfang logs --tail 100
```

### 核心 API 端点

```bash
# 获取所有 Agent
GET /api/agents

# 启动某个 Agent
POST /api/agents/{agent_id}/run

# 查询向量知识库
POST /api/memory/search

# 获取事件流（实时）
GET /api/events?stream=true

# 请求审批
POST /api/approval/{action_id}/request

# 查看 Hand 状态
GET /api/hands/{hand_name}/status
```

### 关键 Traits（重点记忆）

```rust
pub trait Agent: Send + Sync {
    async fn think(&mut self, input: &UserMessage) -> Result<AgentThought>;
    async fn act(&mut self, tool_call: ToolCall) -> Result<ToolOutput>;
    async fn observe(&mut self, feedback: &ToolOutput) -> Result<()>;
}

pub trait Memory: Send + Sync {
    async fn push_short_term(&self, msg: Message) -> Result<()>;
    async fn search_long_term(&self, query: &str, k: usize) -> Result<Vec<Chunk>>;
    async fn append_turn(&self, turn: ConversationTurn) -> Result<()>;
}

pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, input: ToolInput) -> Result<ToolOutput>;
}

pub trait Hand {
    async fn execute(&mut self) -> Result<HandOutput>;
    fn name(&self) -> &str;
    fn version(&self) -> &str;
}
```

---

## 📚 相关文档

本教程系列推荐配合以下文档阅读：

1. **[zeroclaw_unified_plan.md](zeroclaw_unified_plan.md)** —— ZeroClaw 的完整教程计划
2. **[zeroclaw_tutorial_chapter_0.md](zeroclaw_tutorial_chapter_0.md) 至 chapter_8** —— ZeroClaw 详解
3. **[zeroclaw_vs_openfang_engineering_report.md](zeroclaw_vs_openfang_engineering_report.md)** —— 工程深度对比
4. **[zeroclaw_vs_openfang_gap_report.md](zeroclaw_vs_openfang_gap_report.md)** —— 功能差异分析

---

## ✅ 教程完成度

- [x] 第一部分：架构与设计（2 章）
- [x] 第二部分：执行引擎（2 章）
- [x] 第三部分：Hands 与生态（3 章）
- [ ] 第四部分：生产部署与监控（计划）
- [ ] 第五部分：常见问题与最佳实践（计划）

---

## 💬 教程反馈

如果你在阅读过程中有以下需求：

1. **更多代码示例** —— 想要看某个模块的完整实现？
2. **性能数据** —— 想要看 OpenFang vs ZeroClaw 的 benchmark？
3. **案例研究** —— 想要看某个企业如何部署 OpenFang？
4. **概念澄清** —— 对某个 Crate 或 Hand 的职责不清楚？

请随时告诉我，我会扩展相应的章节！

---

**祝你阅读愉快，Happy Coding! 🚀**
