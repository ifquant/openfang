# OpenFang 完全解析教程（第二部分）—— 核心执行引擎剖析

> **深潜 OpenFang 的心脏：2,942 行的 Agent Loop、44 个子模块的精妙设计、以及如何让大模型在极端边界条件下仍能稳定运行。**

---

## 第四章：Agent Loop —— OpenFang 的心脏

如果说 OpenFang 的架构是"骨架"，那么 **Agent Loop** 就是"心脏"。它决定了系统如何思考、如何决策、如何应对大模型的各种"犯蠢"情况。

### 4.1 单次推理的完整生命周期

```
User Input
    │
    ├─→ [1] 记忆回忆 (Memory Recall)
    │   └─→ 向量搜索 + 全文搜索 + 最近对话
    │
    ├─→ [2] 上下文预算检查 (Context Budget Guard)
    │   └─→ 计算可用 Token 空间
    │
    ├─→ [3] 提示词构建 (Prompt Building)
    │   ├─→ System Prompt
    │   ├─→ 人格与角色设定
    │   ├─→ 任务指令
    │   ├─→ 记忆片段 (Retrieved Chunks)
    │   └─→ 历史对话
    │
    ├─→ [4] 循环生成 (Agent Loop)
    │   │
    │   │  当迭代次数 < 50 AND 未收到 stop_reason = "end_turn":
    │   │
    │   ├─→ [4a] LLM 调用 (Think Phase)
    │   │   ├─→ 速率限制检查 (Rate Limit Guard)
    │   │   ├─→ 重试机制 (Exponential Backoff)
    │   │   ├─→ Token 流式接收
    │   │   └─→ 流解析（JSON、函数调用等）
    │   │
    │   ├─→ [4b] 工具调用识别与修复 (Tool Call Recovery)
    │   │   ├─→ 工具名修复（别名标准化）
    │   │   ├─→ 参数提取与清洗
    │   │   └─→ Schema 校验
    │   │
    │   ├─→ [4c] 工具执行 (Act Phase)
    │   │   ├─→ 工具授权检查 (Policy)
    │   │   ├─→ 工具隔离执行（WASM/进程沙盒）
    │   │   ├─→ 超时控制 (120s)
    │   │   └─→ 输出大小截断
    │   │
    │   └─→ [4d] 观察反馈 (Observe Phase)
    │       └─→ 将工具结果作为下一轮输入
    │
    ├─→ [5] 响应清理 (Reply Directives)
    │   ├─→ 提取 [[silent]] 标记
    │   ├─→ 提取 [[no_reply]] 标记
    │   └─→ 提取自定义指令
    │
    └─→ [6] 会话保存
        ├─→ 新消息持久化
        ├─→ 向量索引更新
        └─→ Token 成本计算
```

---

### 4.2 代码架构详解

打开 `openfang-runtime/src/agent_loop.rs`，这个 2,942 行的文件可以分为几个关键函数：

#### **入口函数：`run_agent_loop()`**

```rust
// openfang-runtime/src/agent_loop.rs:L59
pub async fn run_agent_loop(
    manifest: &AgentManifest,
    user_message: &str,
    session: &mut Session,                          // ① 当前对话会话
    memory: &MemorySubstrate,                       // ② 向量库 + RAG
    driver: Arc<dyn LlmDriver>,                     // ③ LLM 驱动抽象
    available_tools: &[ToolDefinition],             // ④ 工具列表
    kernel: Option<Arc<dyn KernelHandle>>,          // ⑤ 事件总线句柄
    skill_registry: Option<&SkillRegistry>,         // ⑥ 技能注册表
    mcp_connections: Option<&tokio::sync::Mutex<Vec<McpConnection>>>,  // ⑦ MCP 服务器
    web_ctx: Option<&WebToolsContext>,              // ⑧ 网络工具（爬虫、搜索）
    browser_ctx: Option<&crate::browser::BrowserManager>,  // ⑨ 浏览器自动化
    embedding_driver: Option<&(dyn EmbeddingDriver + Send + Sync)>,  // ⑩ 嵌入模型
    workspace_root: Option<&Path>,                  // ⑪ 工作区根目录
    on_phase: Option<&PhaseCallback>,               // ⑫ 生命周期回调
    // ... 更多参数
) -> OpenFangResult<AgentLoopResult>
```

**参数解读**：这个函数接收了整个 OpenFang 系统的"血液"。每一个参数都代表系统的一个子模块：

| 参数 | 来自 Crate | 职责 |
|------|-----------|------|
| `manifest` | openfang-types | Agent 的配置清单 |
| `memory` | openfang-memory | 记忆检索与存储 |
| `driver` | openfang-runtime | LLM API 抽象 |
| `kernel` | openfang-kernel | 事件总线、审批、调度 |
| `web_ctx` | openfang-runtime | 网络工具（搜索、爬虫） |
| `browser_ctx` | openfang-runtime | Playwright 浏览器 |

这个函数的签名本身就体现了 **依赖注入** 的设计模式：所有外部资源都通过参数传入，而不是全局状态。这使得 Loop 本身可以被独立单元测试。

---

#### **第一步：记忆回忆 (Memory Recall)**

```rust
// L116-140
let memories = if let Some(emb) = embedding_driver {
    match emb.embed_one(user_message).await {
        Ok(query_vec) => {
            debug!("Using vector recall (dims={})", query_vec.len());
            memory
                .recall_with_embedding_async(
                    &session.agent_id,
                    &query_vec,
                    10,  // Top-10 最相关的记忆片段
                )
                .await?
        }
        Err(_) => {
            // 如果向量模型失败，降级为全文搜索
            memory.recall_with_text(&session.agent_id, user_message, 10).await?
        }
    }
} else {
    // 如果没有嵌入模型，使用纯全文搜索
    memory.recall_with_text(&session.agent_id, user_message, 10).await?
};
```

**设计智慧**：
1. **首选向量搜索** —— 语义理解强，能找到"相似意思"的历史对话
2. **降级全文搜索** —— 当嵌入模型出问题时，仍能工作
3. **没有外部调用** —— 记忆系统是本地数据库或向量库，不会因为网络超时而卡住

---

#### **第二步：上下文预算检查 (Context Budget Guard)**

```rust
// L144-160
let context_window = context_window_tokens.unwrap_or(DEFAULT_CONTEXT_WINDOW);
let mut context = apply_context_guard(
    &session.messages,
    &memories,
    available_tools,
    manifest.instructions.as_ref(),
    context_window,
);

debug!(
    "Context budget: {} tokens available, {} used for memories",
    context.available_tokens, context.memories_tokens_used
);
```

**这解决了什么问题？**

大模型的上下文窗口有限（GPT-4 是 128K tokens，Claude 是 200K tokens）。如果历史对话太长，新消息会被"淹没"在历史里。Context Budget Guard 会自动：

1. **计算可用 Token 数**：总窗口 - 系统提示词 - 工具定义 - 响应预留
2. **智能截断历史**：保留最近的对话，摘要化更早的对话
3. **动态调整**：如果添加了新工具定义，自动收缩历史窗口

这就像一个智能的"会议室管理员"，确保永远有足够的空间让新发言人讲话。

---

#### **第三步：Agent Loop —— 思考-行动-观察循环**

这是最关键的部分，分为 50 次迭代：

```rust
// L162-200
const MAX_ITERATIONS: u32 = 50;
let mut iterations = 0;
let mut accumulated_cost = 0.0;

loop {
    iterations += 1;
    if iterations > MAX_ITERATIONS {
        warn!("Agent exceeded maximum iterations ({MAX_ITERATIONS})");
        break;
    }

    // === PHASE 1: THINK (调用 LLM) ===
    
    if let Some(cb) = &on_phase {
        cb(LoopPhase::Thinking);  // 通知 UI："正在思考中..."
    }

    let completion_request = CompletionRequest {
        model: manifest.model.clone(),
        messages: context.messages.clone(),
        tools: context.tools.clone(),
        temperature: manifest.temperature.unwrap_or(0.7),
        max_tokens: context.available_tokens as u32,
        // ... 其他参数
    };

    // 速率限制 + 重试逻辑
    let cooldown_verdict = cooldown.check_and_update(&driver_id, &model).await;
    if let CooldownVerdict::BackoffRequired { retry_after } = cooldown_verdict {
        tokio::time::sleep(Duration::from_secs(retry_after)).await;
    }

    let response = match driver.complete(&completion_request).await {
        Ok(resp) => resp,
        Err(e) => {
            // 错误处理与降级：见后续章节
            handle_llm_error(e, &mut context, &driver).await?
        }
    };

    let stop_reason = response.stop_reason.clone();
    context.messages.push(Message {
        role: Role::Assistant,
        content: response.content.clone(),
    });

    accumulated_cost += response.usage.total_cost_usd();

    // === PHASE 2: ACT (执行工具) ===

    let tool_calls = extract_tool_calls(&response.content)?;
    if tool_calls.is_empty() {
        // Agent 没有调用任何工具，直接返回回复
        debug!("Agent produced final response");
        break;
    }

    for tool_call in tool_calls {
        if let Some(cb) = &on_phase {
            cb(LoopPhase::ToolUse {
                tool_name: tool_call.name.clone(),
            });
        }

        let tool_output = tool_runner::execute_tool(
            &tool_call,
            available_tools,
            workspace_root,
            // ... 其他参数
        )
        .await?;

        // === PHASE 3: OBSERVE (反馈) ===
        
        context.messages.push(Message {
            role: Role::User,
            content: MessageContent::Text(format!(
                "[Tool Result] {}: {}",
                tool_call.name, tool_output
            )),
        });
    }

    // 如果 LLM 主动说"我完成了"，就退出循环
    if stop_reason == StopReason::EndTurn {
        break;
    }
}
```

**这个循环的设计之妙**：

1. **异常优雅的结束条件**：
   - Agent 主动说"我完成了" → 退出
   - 没有工具调用 → 返回最后一条回复
   - 达到 50 次迭代 → 强制退出（防止无限循环）
   - LLM 返回错误 → 重试或降级

2. **生命周期回调**：通过 `on_phase` 回调，UI 可以显示"正在思考"、"正在执行 Shell 命令"等进度提示

3. **成本追踪**：每次 LLM 调用都累加成本，最后返回给 Kernel 进行计费

---

### 4.3 容错机制：OpenFang vs ZeroClaw

这是两个框架最大的差异所在。让我们来看看 OpenFang 如何处理大模型的各种"犯蠢"情况。

#### **问题 1：工具名识别错误**

**场景**：大模型调用了 `shell` 但实际定义是 `execute_command`

```rust
// openfang-runtime/src/agent_loop.rs:L640
fn extract_tool_calls(content: &MessageContent) -> Result<Vec<ToolCall>> {
    // 首先尝试标准 JSON 解析
    if let Ok(calls) = parse_native_tool_calls(content) {
        return Ok(calls);
    }

    // 如果标准解析失败，尝试一系列降级策略
    
    // 策略 1：MiniMax (MiniMax 模型特殊格式)
    if let Ok(calls) = parse_minimax_format(content) {
        debug!("Recovered tool calls using MiniMax format");
        return Ok(calls);
    }

    // 策略 2：DeepSeek 格式
    if let Ok(calls) = parse_deepseek_format(content) {
        debug!("Recovered tool calls using DeepSeek format");
        return Ok(calls);
    }

    // 策略 3：GLM 格式
    if let Ok(calls) = parse_glm_format(content) {
        debug!("Recovered tool calls using GLM format");
        return Ok(calls);
    }

    // 策略 4：正则表达式回收
    if let Ok(calls) = regex_recover_tool_calls(content) {
        debug!("Recovered tool calls using regex");
        return Ok(calls);
    }

    // 都失败了，返回空列表（让 Agent 继续生成）
    Ok(vec![])
}

// 工具名修复
fn normalize_tool_name(raw_name: &str) -> String {
    // 27 个别名 → 5 个规范名
    match raw_name.to_lowercase().as_str() {
        "shell" | "bash" | "sh" | "execute" | "run_command" 
        | "execute_command" | "cmd" | "command" => "shell".to_string(),
        
        "search" | "web_search" | "google" | "bing" | "web_query"
        | "query_web" | "find_online" => "search".to_string(),
        
        // ... 更多别名映射
    }
}
```

**对比 ZeroClaw** —— ZeroClaw 也有类似的容错，但 OpenFang 将这些策略**显式编码**在 openfang-runtime 里，而 ZeroClaw 把大部分容错硬编码在 Agent Loop 里。两者的差异是：

| 维度 | ZeroClaw | OpenFang |
|------|----------|----------|
| 工具别名数量 | 27 个 | 27 个（相同） |
| 降级策略数量 | 6 级 | 4 级（更精简） |
| 方式 | 硬编码在 loop_.rs | 模块化的 parser（tool_runner.rs） |
| 扩展性 | 改 loop 需重编 50% 模块 | 改 tool_runner 仅重编 openfang-runtime |

---

#### **问题 2：参数提取失败**

**场景**：大模型调用了 `shell` 但参数是错误的 JSON：
```json
{
  "name": "shell",
  "arguments": {
    "command": "ls /home"  // 正确
  }
}

// vs

{
  "name": "shell",
  "arguments": "ls /home"  // 错误！应该是对象
}
```

```rust
// openfang-runtime/src/agent_loop.rs:L720
fn extract_tool_args(raw_args: &Value) -> Result<HashMap<String, Value>> {
    // 如果是对象，直接返回
    if let Some(obj) = raw_args.as_object() {
        return Ok(obj.iter()
            .map(|(k, v)| (k.clone(), v.clone()))
            .collect());
    }

    // 如果是字符串（Agent 把参数作为字符串而不是 JSON），尝试解析
    if let Some(s) = raw_args.as_str() {
        // 尝试作为 JSON 解析
        if let Ok(parsed) = serde_json::from_str::<serde_json::Value>(s) {
            if let Some(obj) = parsed.as_object() {
                return Ok(obj.iter()
                    .map(|(k, v)| (k.clone(), v.clone()))
                    .collect());
            }
        }

        // 如果是 Shell 工具且参数是字符串，直接使用 "command" 字段
        if raw_args.is_string() {
            return Ok(HashMap::from([
                ("command".to_string(), Value::String(s.to_string()))
            ]));
        }
    }

    bail!("Cannot extract tool arguments from: {:?}", raw_args)
}
```

**设计用意**：很多开源模型（如 Qwen、GLM）倾向于返回字符串格式的参数，而不是严格的 JSON 对象。这个修复函数让 OpenFang 能够自动兼容多种输出格式。

---

#### **问题 3：上下文溢出 (Context Overflow)**

**场景**：Agent 调用了太多工具，生成了大量上下文，即将超过 Token 限制。

```rust
// openfang-runtime/src/context_overflow.rs
pub async fn recover_from_overflow(
    context: &mut Context,
    stage: RecoveryStage,
) -> Result<()> {
    match stage {
        // 第一级：丢弃最早的 10 条历史消息
        RecoveryStage::DropOldHistory => {
            if context.messages.len() > 10 {
                let to_remove = context.messages.len() - 10;
                context.messages.drain(0..to_remove);
                info!("Dropped {} old messages to recover {} tokens",
                    to_remove, to_remove * 200);  // 估计每条 200 tokens
            }
        }

        // 第二级：摘要化最早的 5 条消息
        RecoveryStage::SummarizeHistory => {
            let first_messages = &context.messages[0..5];
            let summary = create_summary(first_messages).await?;
            context.messages.drain(0..5);
            context.messages.insert(0, Message {
                role: Role::System,
                content: MessageContent::Text(format!(
                    "[Historical Summary] {}",
                    summary
                )),
            });
            info!("Summarized 5 old messages, saved ~500 tokens");
        }

        // 第三级：缩小工具定义（移除某些不需要的工具）
        RecoveryStage::ReduceTools => {
            let original_tool_count = context.tools.len();
            context.tools.retain(|t| t.category == "essential");
            info!("Reduced tools from {} to {}", 
                original_tool_count, context.tools.len());
        }

        // 第四级：强制截断（最后手段）
        RecoveryStage::ForceTruncate => {
            let max_tokens = context.available_tokens;
            truncate_to_fit(context, max_tokens)?;
            warn!("Force truncated context to {} tokens", max_tokens);
        }
    }

    Ok(())
}
```

**执行流程**：

```
记忆 + 历史消息 + 工具定义 = 总 Token 数

如果 > 上下文窗口：
  ├─ 尝试第一级：丢弃旧消息
  ├─ 如果还是超：尝试第二级：摘要化
  ├─ 如果还是超：尝试第三级：移除非必要工具
  └─ 如果还是超：第四级：强制截断并继续运行
```

ZeroClaw 也有类似机制，但 OpenFang 的特色是**分离出独立的 context_overflow.rs 模块**，让这个逻辑可以被独立测试和优化。

---

### 4.4 速率限制与重试策略

这是生产级系统必须有的功能。当调用 OpenAI API 时，经常会因为频率限制而收到 429 错误。

```rust
// openfang-runtime/src/auth_cooldown.rs
pub struct ProviderCooldown {
    cooldowns: DashMap<String, CooldownState>,
}

pub enum CooldownVerdict {
    OK,
    BackoffRequired { retry_after: u64 },
}

impl ProviderCooldown {
    pub async fn check_and_update(
        &self,
        provider: &str,
        model: &str,
    ) -> CooldownVerdict {
        let key = format!("{}:{}", provider, model);
        
        if let Some(mut entry) = self.cooldowns.get_mut(&key) {
            if Instant::now() < entry.until {
                // 仍在冷却期
                let remaining = entry.until.elapsed().as_secs();
                return CooldownVerdict::BackoffRequired {
                    retry_after: remaining,
                };
            } else {
                // 冷却期已过，清除
                drop(entry);
                self.cooldowns.remove(&key);
            }
        }

        CooldownVerdict::OK
    }

    pub fn mark_rate_limited(
        &self,
        provider: &str,
        model: &str,
        retry_after: u64,
    ) {
        let key = format!("{}:{}", provider, model);
        self.cooldowns.insert(
            key,
            CooldownState {
                until: Instant::now() + Duration::from_secs(retry_after),
            },
        );
    }
}
```

**执行流程**：

```
Agent 调用 LLM
    │
    ├─ 收到 200 OK → 继续
    │
    ├─ 收到 429 Too Many Requests
    │   └─ 标记该 (provider, model) 对进入冷却期
    │   └─ 返回 BackoffRequired { retry_after: 60 }
    │
    ├─ 睡眠 60 秒
    │
    └─ 重新尝试
```

**对比 ZeroClaw**：ZeroClaw 有基础的重试机制，但 OpenFang 用 DashMap 实现了**无锁并发**的速率限制。多个 Agent 线程可以同时检查和更新冷却状态，无需全局锁。

---

## 第五章：44 个专业子模块详解

openfang-runtime 的神奇之处在于它包含了 44 个精心设计的子模块，每一个都解决一个具体的问题。

### 5.1 工具执行与沙盒

#### **`tool_runner.rs` —— 工具调用的中枢**

```rust
pub async fn execute_tool(
    tool_call: &ToolCall,
    available_tools: &[ToolDefinition],
    workspace_root: Option<&Path>,
    // ... 其他参数
) -> Result<ToolOutput> {
    // 第一步：验证工具是否存在
    let tool_def = available_tools
        .iter()
        .find(|t| t.name == tool_call.name)
        .ok_or_else(|| anyhow!("Tool not found: {}", tool_call.name))?;

    // 第二步：授权检查
    check_tool_policy(&tool_call, &tool_def).await?;

    // 第三步：参数验证
    let validated_args = validate_against_schema(
        &tool_call.arguments,
        &tool_def.parameters_schema,
    )?;

    // 第四步：根据工具类型分发执行
    match tool_def.handler {
        ToolHandler::Native => {
            // 内置工具（Shell、Web 搜索等），直接调用
            execute_native_tool(&tool_call.name, &validated_args).await
        }

        ToolHandler::Wasm { module_path } => {
            // WASM 沙盒工具，通过 Wasmtime 执行
            execute_wasm_tool(&module_path, &validated_args).await
        }

        ToolHandler::Docker { image } => {
            // Docker 沙盒工具，启动容器执行
            execute_docker_tool(&image, &validated_args).await
        }

        ToolHandler::Remote { endpoint } => {
            // 远程工具，通过 HTTP 调用
            execute_remote_tool(&endpoint, &validated_args).await
        }
    }
}
```

**设计亮点**：

1. **多层沙盒策略**：
   - WASM：轻量级、跨平台、Fuel 计量
   - Docker：重量级、完全隔离、适合需要 Python/Node 环境的工具
   - 远程：给 Agent 调用其他服务的能力

2. **参数校验**：使用 JSON Schema 严格验证参数，防止注入攻击

3. **错误处理**：每个沙盒类型都有独立的错误处理和恢复逻辑

---

#### **`subprocess_sandbox.rs` —— 进程级隔离**

```rust
pub async fn execute_in_subprocess_sandbox(
    command: &str,
    args: Vec<String>,
    config: &SubprocessConfig,
) -> Result<String> {
    // 设置隔离环境
    let mut cmd = tokio::process::Command::new(&command);
    
    // 清空环境变量（防止 API Key 泄露）
    cmd.env_clear();
    
    // 只注入允许的环境变量
    for allowed_var in &config.allowed_env_vars {
        if let Ok(val) = std::env::var(allowed_var) {
            cmd.env(allowed_var, val);
        }
    }
    
    // 设置工作目录
    if let Some(workdir) = &config.workspace_root {
        cmd.current_dir(workdir);
    }
    
    // 设置超时
    let output = tokio::time::timeout(
        Duration::from_secs(config.timeout_secs),
        cmd.output(),
    ).await??;
    
    // 检查输出大小（防止 Token 爆炸）
    let stdout = String::from_utf8(output.stdout)?;
    if stdout.len() > config.max_output_bytes {
        // 截断并添加 "...（更多内容已被截断）"
        let truncated = truncate_output(&stdout, config.max_output_bytes);
        return Ok(truncated);
    }
    
    Ok(stdout)
}
```

**安全特性**：

1. **环境变量清空** —— 防止 Agent 读取系统的 AWS_KEY、GITHUB_TOKEN 等敏感信息
2. **工作目录隔离** —— Agent 的 `cd ..` 无法逃脱到系统根目录
3. **超时保护** —— 即使 Shell 命令陷入死循环，120 秒后自动杀死进程
4. **输出截断** —— 一条 `cat /var/log/kern.log` 的输出可能是几 GB，OpenFang 会自动截断到合理大小

---

#### **`docker_sandbox.rs` —— Docker 隔离**

```rust
pub async fn execute_in_docker_sandbox(
    command: Vec<String>,
    docker_image: &str,
    config: &DockerSandboxConfig,
) -> Result<String> {
    // 启动 Docker 容器
    let container = docker_client
        .containers()
        .create(CreateContainerOptions {
            image: Some(docker_image),
            cmd: Some(command),
            host_config: Some(HostConfig {
                // 内存限制：最多 512MB
                memory: Some(512 * 1024 * 1024),
                // CPU 限制：最多 1 核
                cpu_quota: Some(100_000),
                cpu_period: Some(100_000),
                // 网络限制：只允许访问特定域名
                ...
            }),
            ..Default::default()
        })
        .await?;
    
    // 启动容器
    container.start().await?;
    
    // 等待执行完成（最多 120 秒）
    let result = tokio::time::timeout(
        Duration::from_secs(120),
        container.wait(),
    ).await??;
    
    // 获取日志
    let logs = container.logs(...).await?;
    
    // 清理（删除容器）
    container.remove().await?;
    
    Ok(logs)
}
```

**何时使用 Docker 沙盒？**

- 需要运行 Python 代码（Agent 生成 Python 脚本并执行）
- 需要运行 Node.js 代码
- 需要特定的 System 库（`librabbitmq` 等）
- 性能要求不高，隔离要求很高

---

### 5.2 网络工具

#### **`web_search.rs` —— 搜索引擎集成**

```rust
pub struct WebSearchEngine {
    google_api_key: String,
    bing_api_key: String,
    default_engine: SearchEngine,
}

impl WebSearchEngine {
    pub async fn search(&self, query: &str, count: usize) -> Result<Vec<SearchResult>> {
        match self.default_engine {
            SearchEngine::Google => self.search_google(query, count).await,
            SearchEngine::Bing => self.search_bing(query, count).await,
            SearchEngine::DuckDuckGo => self.search_duckduckgo(query).await,
        }
    }
}

pub struct SearchResult {
    pub title: String,
    pub url: String,
    pub snippet: String,
    pub source: String,  // "google", "bing", "local_index"
}
```

**特色**：支持多个搜索引擎，如果 Google 超时可以自动切换到 Bing。

---

#### **`web_fetch.rs` —— 网页爬取与解析**

```rust
pub async fn fetch_and_parse(
    url: &str,
    config: &FetchConfig,
) -> Result<FetchResult> {
    // 第一步：获取网页
    let client = reqwest::Client::new();
    let response = client.get(url)
        .timeout(Duration::from_secs(30))
        .send()
        .await?;

    let html = response.text().await?;

    // 第二步：提取文本内容（去除 JavaScript、CSS）
    let content = extract_content(&html)?;

    // 第三步：提取结构化数据（标题、段落、表格）
    let structured = parse_structure(&content)?;

    // 第四步：如果需要，运行 JavaScript（Playwright）
    if config.require_js {
        let content = run_with_browser(url).await?;
        return Ok(FetchResult {
            content,
            js_rendered: true,
        });
    }

    Ok(FetchResult {
        content: structured,
        js_rendered: false,
    })
}
```

**设计考量**：很多现代网站都是 JavaScript 渲染的（SPA），静态 HTTP GET 无法获取完整内容。OpenFang 提供了两种模式：

- **快速模式**（仅 HTTP GET）—— 适合静态内容、速度优先
- **JS 渲染模式**（Playwright）—— 适合动态网站、准确性优先

---

#### **`web_cache.rs` —— 智能缓存**

```rust
pub struct WebCache {
    cache: Arc<DashMap<String, CachedPage>>,
    ttl: Duration,
}

pub struct CachedPage {
    content: String,
    timestamp: SystemTime,
    etag: Option<String>,  // HTTP ETag，用于检测更新
}

impl WebCache {
    pub async fn get_or_fetch(
        &self,
        url: &str,
        allow_stale: bool,
    ) -> Result<String> {
        // 查询缓存
        if let Some(cached) = self.cache.get(url) {
            if Instant::now() - cached.timestamp < self.ttl {
                // 缓存未过期，直接返回
                return Ok(cached.content.clone());
            }
            
            if allow_stale {
                // 缓存过期但允许使用旧数据（在线不可用时的降级）
                return Ok(cached.content.clone());
            }
        }

        // 缓存不可用或过期，重新爬取
        let content = fetch_and_parse(url).await?;
        self.cache.insert(url.to_string(), CachedPage {
            content: content.clone(),
            timestamp: SystemTime::now(),
            etag: extract_etag_from_headers(),
        });

        Ok(content)
    }
}
```

**用途**：防止频繁爬取同一个网页，既节省 Token（不用反复爬），也尊重网站（避免 DDoS）。

---

### 5.3 媒体与多模态

#### **`media_understanding.rs` —— 图像识别与分析**

```rust
pub struct MediaEngine {
    vision_model: Arc<dyn VisionProvider>,  // 图像识别模型
}

impl MediaEngine {
    pub async fn analyze_image(&self, image: &[u8]) -> Result<ImageAnalysis> {
        // 使用视觉 LLM（如 GPT-4V、Claude Vision）分析图像
        let description = self.vision_model.describe(image).await?;
        
        let analysis = ImageAnalysis {
            description,
            objects: extract_objects(&description),
            text: extract_text_from_image(&description),
            dominant_color: extract_color(&description),
        };

        Ok(analysis)
    }

    pub async fn image_to_text(&self, image: &[u8]) -> Result<String> {
        // OCR 应用
        self.vision_model.ocr(image).await
    }
}
```

**用途**：Agent 收到用户发来的图片时，自动用 Vision API 分析图片内容，而不是盲目处理。

---

#### **`tts.rs` —— 文本转语音**

```rust
pub struct TtsEngine {
    provider: Arc<dyn TextToSpeechProvider>,
}

impl TtsEngine {
    pub async fn synthesize(&self, text: &str, voice: &VoiceConfig) -> Result<Vec<u8>> {
        self.provider.synthesize(text, voice).await
    }
}

pub trait TextToSpeechProvider {
    async fn synthesize(&self, text: &str, voice: &VoiceConfig) -> Result<Vec<u8>>;
    async fn list_voices(&self) -> Result<Vec<Voice>>;
}
```

**支持的提供商**：
- OpenAI TTS（高质量，支持多种语言）
- Google Cloud TTS
- Azure Speech Services
- ElevenLabs（自然度最高）

---

### 5.4 其他关键模块

| 模块 | 职责 | 代码行数 |
|------|------|--------|
| `agent_loop.rs` | 核心推理循环 | ~2,942 |
| `tool_runner.rs` | 工具执行与分发 | ~600 |
| `subprocess_sandbox.rs` | 进程隔离 | ~300 |
| `docker_sandbox.rs` | 容器隔离 | ~400 |
| `llm_driver.rs` | LLM API 抽象 | ~800 |
| `prompt_builder.rs` | 提示词动态生成 | ~500 |
| `context_budget.rs` | Token 预算管理 | ~400 |
| `context_overflow.rs` | 溢出恢复 | ~300 |
| `memory_recall.rs` | 向量搜索与检索 | ~350 |
| `web_search.rs` | 搜索引擎集成 | ~400 |
| `web_fetch.rs` | 网页爬取与解析 | ~500 |
| `browser.rs` | Playwright 浏览器自动化 | ~600 |
| `mcp.rs` | Model Context Protocol | ~500 |
| 其他 (28 个模块) | 媒体、审计、缓存等 | ~3,000 |

---

## 第六章：与 ZeroClaw 的技术权衡

### 6.1 算法设计的对比

#### **记忆检索策略**

**ZeroClaw**：
```rust
// src/memory/recall.rs
fn recall_memories(query: &str) -> Vec<Memory> {
    // 方案 A：向量搜索（如果有嵌入模型）
    // 方案 B：全文搜索（BM25 或简单 contains）
    // 方案 C：重要性排序（手动标记）
    
    // 选择一种执行，没有 fallback
}
```

**OpenFang**：
```rust
// crates/openfang-runtime/src/agent_loop.rs:L116
let memories = if let Some(emb) = embedding_driver {
    match emb.embed_one(user_message).await {
        Ok(query_vec) => {
            // 方案 A：向量搜索
            memory.recall_with_embedding_async(...).await?
        }
        Err(_) => {
            // 降级到方案 B
            memory.recall_with_text(...).await?
        }
    }
} else {
    // 直接方案 B
    memory.recall_with_text(...).await?
};
```

**区别**：OpenFang 有显式的 fallback 链，单点故障不会导致整个 Agent 崩溃。

---

#### **工具调用的容错级别**

**ZeroClaw**（来自 Engineering Report）：
- ✅ 6 级解析链（JSON → MiniMax → DeepSeek → GLM → Grok → Perl 正则）
- ✅ 27 个工具名别名
- ✅ 7 种 Shell 参数别名
- ✅ URL 自动包装
- ✅ Schema 清洗（$ref 展开、循环检测）

**OpenFang**（来自 agent_loop.rs）：
- ✅ 4 级解析链（Native JSON → MiniMax → DeepSeek → GLM → 正则回收）
- ✅ 27 个工具名别名（相同）
- ⚠️ 7 种 Shell 别名（可能较少）
- ✅ URL 自动包装
- ✅ Schema 清洗（完整的 JSON Schema 验证）

**结论**：ZeroClaw 在"容错深度"上略胜一筹（6 级 vs 4 级）。但 OpenFang 的设计更**可维护**——解析逻辑分散在多个模块，改一个解析器不会影响其他部分。

---

### 6.2 并发与性能

| 维度 | ZeroClaw | OpenFang |
|------|----------|----------|
| **单次 Agent 推理速度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **多 Agent 并发（10 个同时运行）** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **内存占用（闲置）** | 5MB | 40MB |
| **内存占用（10 个 Agent 活跃）** | 150MB | 320MB（但分片锁不争用） |

**为什么 OpenFang 的并发更好？**

ZeroClaw 使用 `Arc<Mutex<AppState>>`（全局互斥锁），意味着：
- Agent A 在读取内存库时，Agent B 必须等待
- Agent C 在执行 Shell 时，Agent D 必须等待
- 10 个 Agent 实际上是"虚假并发"（并发启动，但执行时串行化）

OpenFang 使用 `DashMap`（分片 HashMap + 细粒度锁），意味着：
- Agent A 读内存库，Agent B 同时可以执行工具（不同的锁）
- Agent C 修改 Session 数据，Agent D 同时可以读取工具定义
- 真正的**并发执行**

在 Web 服务场景（数百个用户同时聊天），这个差异决定了系统是否能扩展到 1000+ req/s。

---

## 总结：核心执行引擎的设计美学

### 🎯 OpenFang 的三层优雅设计

1. **Agent Loop 的不变性**
   - 2,942 行的单一职责：接收消息 → 生成回复
   - 所有扩展都通过参数注入，而不是全局状态污染

2. **44 个子模块的正交设计**
   - 每个模块解决一个具体问题（容错、沙盒、缓存、web 工具等）
   - 模块间通过 Traits 通信，无硬依赖
   - 任何模块都可以被独立替换或删除

3. **容错的递进策略**
   - 第一选择：标准 JSON
   - 第二选择：各个大模型的特殊格式
   - 第三选择：正则表达式回收
   - 最后手段：返回空列表让 Agent 重新生成

### 🏛️ 工业级与艺术的碰撞

OpenFang 的代码不只是"能用"，更追求**完整性**和**优雅性**：

- ✅ 不依赖 Python（Rust 直接编译）
- ✅ 无运行时垃圾回收（确定性性能）
- ✅ 内存安全（编译期保证，无 Segfault）
- ✅ 并发安全（编译期强制无数据竞争）
- ✅ 极限规模（单个二进制运行 1000+ Agent 无压力）

---

## 下一步阅读

- **第三部分**：7 个预制 Hands 的实现剖析（Lead、Researcher、Browser 等）
- **第四部分**：事件总线与 Kernel 的协调机制
- **第五部分**：生产部署与监控
