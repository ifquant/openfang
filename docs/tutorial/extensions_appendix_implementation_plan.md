# OpenFang Extensions 附录实施计划

> 日期：2026-03-07
> 范围：`openfang/crates/openfang-extensions/src/`
> 目标：判断 `openfang-extensions` 应并入第六章生态补充，还是独立成附录；并给出可直接落文的章节骨架。

---

## 1. 为什么这一块值得单独覆盖

`openfang-extensions` 只有 2.8K 行左右，但它不是边缘杂项，而是把 OpenFang 的 MCP 生态补成“可安装、可凭证管理、可 OAuth、可健康监控”的产品层。它回答的问题不是“内核支不支持 MCP”，而是：

1. 模板化的 MCP 集成从哪里来。
2. 安装状态如何持久化。
3. secrets 如何从 env / dotenv / vault / 交互输入中解析。
4. OAuth 型集成如何跑 PKCE。
5. 集成失败后如何做健康状态与重连退避。

第六章已经讲了 MCP 接入面，但没有讲“集成市场/安装层”。因此最合理的定位不是重写第六章，而是新增一个 **附录或第十章短章**，作为“生态分发与凭证治理层”。

---

## 2. 当前已确认的源码骨架

### 2.1 类型系统与模板模型

- `lib.rs`
  - `IntegrationCategory`：L57
  - `McpTransportTemplate`：L82
  - `RequiredEnvVar`：L95
  - `OAuthTemplate`：L116
  - `HealthCheckConfig`：L130
  - `IntegrationTemplate`：L148
  - `IntegrationStatus`：L182
  - `InstalledIntegration`：L209

结论：这个 crate 先定义了一套“集成模板 DSL”，再让安装器、注册表、健康监控围着它转，而不是散乱地拼几个 JSON/TOML 文件。

### 2.2 Bundled 模板与注册表

- `bundled.rs`
  - `bundled_integrations()`：L7
- `registry.rs`
  - `IntegrationRegistry`：L17
  - `new()`：L28
  - `load_bundled()`：L37
  - `load_installed()`：L55
  - `save_installed()`：L71
  - `search()`：L142
  - `list_all_info()`：L156
  - `to_mcp_configs()`：L178

结论：注册表不是只做浏览，它还承担了一个关键桥接职责：把已安装且启用的模板转成 `McpServerConfigEntry`，也就是把“市场层”重新折叠回内核可消费的 MCP 配置层。

### 2.3 一键安装流

- `installer.rs`
  - `install_integration()`：L36
  - `remove_integration()`：L133
  - `list_integrations()`：L146
  - `search_integrations()`：L195
  - `scaffold_integration()`：L222
  - `scaffold_skill()`：L264

结论：安装器的真实职责是 `template lookup -> credential resolution -> OAuth/provider choice -> 写 integrations.toml`。它更像一个产品安装向导，而不是简单的文件复制器。

### 2.4 凭证解析与保险箱

- `credentials.rs`
  - `CredentialResolver`：L17
  - `new()`：L28
  - `resolve()`：L48
  - `missing_credentials()`：L111
  - `store_in_vault()`：L119
- `vault.rs`
  - `CredentialVault`：L56
  - `new()`：L69
  - `init()`：L79
  - `unlock()`：L133
  - `init_with_key()`：L195
  - `unlock_with_key()`：L213
  - `resolve_master_key()`：L245
  - `save()`：L266
  - `load()`：L324
  - `derive_key()`：L401
  - `machine_fingerprint()`：L503

结论：OpenFang 在这里已经不是“把 token 塞进 env 就完了”。它明确做了 vault、keyring、zeroizing、主密钥解析与磁盘落密文，是 MCP 集成产品化的重要成熟度信号。

### 2.5 OAuth 与健康监控

- `oauth.rs`
  - `default_client_ids()`：L19
  - `resolve_client_ids()`：L30
  - `run_pkce_flow()`：L130
  - `open_browser()`：L291
- `health.rs`
  - `IntegrationHealth`：L15
  - `HealthMonitor`：L106
  - `register()`：L123
  - `report_ok()`：L135
  - `report_error()`：L142
  - `backoff_duration()`：L159
  - `should_reconnect()`：L166
  - `mark_reconnecting()`：L179

结论：这说明 `openfang-extensions` 不只是“模板商店”，而是把 OAuth onboarding 与失败后自愈策略也往里塞了，产品意味比表面更强。

---

## 3. 建议写法

### 方案 A：作为第六章补充小节

优点：

- 逻辑上仍属于网络/MCP 生态面。
- 不会继续膨胀主章节数量。

缺点：

- 第六章已经 570 行，继续塞会变重。
- 凭证保险箱、OAuth、安装器这些内容和 API/router/wire 并不是同一层。

### 方案 B：独立附录，标题建议

建议标题：

- `附录 B：MCP 集成市场——模板、保险箱与 OAuth 安装流`

优点：

- 更容易围绕“产品化生态层”组织叙事。
- 不会稀释第六章的 API/协议主线。
- 可以把 vault / credentials / PKCE / health 讲成一套闭环。

当前更推荐 **方案 B**。

---

## 4. 建议章节骨架

### B.1 深入解读

1. `IntegrationTemplate`：OpenFang 怎样把 MCP 集成描述成模板 DSL
2. `IntegrationRegistry`：bundled 模板、installed 状态与 `to_mcp_configs()` 桥接
3. `install_integration()`：一键安装流的真实步骤
4. `CredentialResolver` + `CredentialVault`：env / dotenv / vault / interactive 的多层凭证解析
5. `run_pkce_flow()`：OAuth 型集成的本地回调与浏览器流程
6. `HealthMonitor`：错误状态、指数退避与自动重连判断

### B.2 压力测试

1. `integrations.toml` 损坏会怎样
2. vault 锁住且 keyring 不可用时会怎样
3. OAuth 浏览器回调失败时的退路是什么
4. 集成持续失败时，健康监控是否会无限重连

### B.3 改进雷达

1. 安装层与 daemon hot-reload 的耦合边界是否够清晰
2. `IntegrationStatus` 目前较粗，Ready/Setup/Disabled/Error 之外缺少更细原因
3. vault 与操作系统 keyring 的失败语义是否足够一致
4. bundled 模板是否支持用户级覆盖与版本升级策略

### B.4 行动项

1. 给 `to_mcp_configs()` 增加来源/状态审计
2. 给安装流增加更清晰的失败分类
3. 补 vault 恢复与备份文档
4. 把健康监控结果暴露到 TUI / API

---

## 5. 下一步最小执行单元

如果下一轮直接落文，建议优先再读四个文件：

1. `vault.rs`：确认 AES-256-GCM、keyring、machine fingerprint 的具体边界
2. `credentials.rs`：确认 resolver 的优先级顺序
3. `oauth.rs`：确认 localhost callback 与 PKCE 的完整流程
4. `health.rs`：确认重连/backoff 是观测层还是执行层

完成这四个文件后，就足以直接写出附录初稿。