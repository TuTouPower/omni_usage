<!-- omni_powers: scratch/code-review-tasks-2026-07-13 -->

# 代码审查 Task List（2026-07-13）

**来源**：综合 `code-review-2026-07-13.md`（原报告）+ `code-review-my-take-2026-07-13.md` + `code-review-response-2026-07-13.md`，并逐条核对代码与 spec 原文。
**原则**：只列真正需要改的；已核对事实才列入；需要设计决策的归"待澄清"不直接派任务。
**优先级**：P0 = 破坏业务不变量/基础契约失效且用户可见；P1 = 违反约定有潜在风险；P2 = 技术债/可读性；待澄清 = 需产品/设计决策才能定方向。

---

## P0 — 必须修

### P0-1. refresh-service 失败不写 stale observation

- **事实**：[refresh-service.ts:285-289](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/refresh-service.ts#L285-L289) 失败分支只 `runtimeStore.updateState({status:"failed", error, lastSuccess})`，未向 observation-store 插入 `stale:true` 行。
- **依据**：`domain.md` 不变量 2 + `observation-store.md`「采集失败保留上次成功观测，挂 `stale:true`+`lastError`，绝不覆盖删除」。
- **动作**：
    1. 在 refresh-service 失败分支，按 `(provider, sourceInstanceId, accountId, metricId)` 查询上次成功观测。
    2. 复制为新行，`stale:true` + `last_error` + `observed_at=now`，插入 observation-store。
    3. 若无上次成功观测（首次即失败），则不插——UI 应显示"无数据"而非"stale"。
- **风险**：需要 observation-store 提供"按 instance 取最新观测列表"的查询接口；可能要新增 `mark_stale(instanceId, lastError)`。
- **验收**：连接器采集失败后，UI 上该连接器的账号显示 stale 标记 + 错误信息，而非凭空消失或假装新鲜。

### P0-2. CPA 单账号失败不产 stale 观测

- **事实**：[cpa/connector.ts:514-525](file:///d:/Kar/Code/omni_usage/connectors/cpa/connector.ts#L514-L525) per-account try/catch 只 `ctx.log.warn` + `continue`，无 stale 观测产出。
- **依据**：`domain.md` 不变量 5「单账号失败只让那一行 stale，同 provider 其他账号照常刷新」。
- **难点**：CPA 脚本无状态，不知道"上次成功观测"是什么；宿主无法仅凭本次返回的 observations 判断"哪个账号失败了"。
- **动作（需先选方案）**：
    - 方案 A：扩展脚本协议，允许脚本返回 `{ observations: [...], failed_accounts: [{provider, accountId, error}] }`，宿主据此为失败账号插 stale 行。
    - 方案 B：宿主对比"上次成功的账号集合"与"本次成功的账号集合"，为消失的账号插 stale（无需改脚本协议，但宿主要维护账号集合状态）。
    - **推荐方案 A**——更显式、更符合"脚本负责发现、宿主负责盖章"的契约。
- **风险**：改脚本协议涉及 `ScriptObservation` 返回类型 + runtime 沙箱适配 + 其他连接器兼容。
- **验收**：CPA 单账号失败时，UI 上该账号显示 stale + 错误信息，同 provider 其他账号正常显示新鲜数据。

### P0-3. PopupView 账号子行写 disabled 越层

- **事实**：[PopupView.tsx:491-525](file:///d:/Kar/Code/omni_usage/src/renderer/views/PopupView.tsx#L491-L525) `disable_account` 在账号子行调 `add_account_override("disabled", ...)` + `config.save`。
- **依据**：`domain.md` 不变量 8「破坏性操作只出现在"行即数据源"层级，账号子行只做显示调整」+ 不变量 8 后半「CPA 账号存在性由远端 CPA-Manager 决定 → 本地只能隐藏」。
- **依赖**：此任务与"待澄清-2"（`disabled` 字段去留）强耦合。若决定删除 `disabled` 字段，本任务自动简化为"删 disable_account 函数 + UI 按钮"。
- **动作（若保留 disabled 语义）**：
    1. 对直连账号：`disabled` 应作用于连接器实例层级（禁用整个数据源），不是账号子行。移到设置页"行即数据源"行。
    2. 对 CPA 账号：移除 `disabled` 能力，只保留 `hidden`。
- **动作（若删除 disabled 字段）**：删 `disable_account` + `add_account_override("disabled", ...)` + `account-overrides.ts` 的 disabled 分支 + types.ts 的 disabled 字段。
- **验收**：账号子行只能 hidden / 改标签，不能 disable；CPA 账号子行尤其不能被本地禁用。

### P0-4. session partition 命名错 + 体系冲突

- **事实**：
    - [session-manager.ts:9](file:///d:/Kar/Code/omni_usage/src/main/core/session/session-manager.ts#L9) `SESSION_LOGIN_PARTITION = "persist:session-login"` → 拼成 `persist:session-login:<instance_id>`（按实例）
    - [auth-ipc.ts:13](file:///d:/Kar/Code/omni_usage/src/main/ipc/auth-ipc.ts#L13) `MIMO_LOGIN_PARTITION = "persist:mimo-login"` → 拼成 `persist:mimo-login:<instanceId>`（按实例 + 硬编码 provider）
    - 两套命名规则，都按 instance_id 而非 provider
- **依据**：`connector-session.md:21`「每个需登录 provider 一个独立持久化分区 `persist:<provider>-login`」——明确按 provider，同 provider 多实例共享 partition（合理：同一网站登录态可复用，减少重复登录）。
- **动作**：
    1. 统一 partition 命名为 `persist:<provider>-login`（provider 从 manifest 取，不硬编码）。
    2. 删除 `MIMO_LOGIN_PARTITION` 常量，auth-ipc 改用 session-manager 提供的统一函数 `get_session_login_partition(provider)`。
    3. session-manager 的 `get_session_login_partition` 签名从 `(instance_id)` 改为 `(provider)`。
- **风险**：已有按 instance 分的 partition 里的 cookie 会失效，用户需重新登录一次。可接受（首次升级成本）。
- **验收**：同 provider 多实例共享 partition；新增 session 连接器不改宿主代码；partition 命名规则单一。

### P0-5. cookie jar 回退拼装违禁

- **事实**：[session-manager.ts:104-110](file:///d:/Kar/Code/omni_usage/src/main/core/session/session-manager.ts#L104-L110) 在 `captured_cookie` 为 null 时回退到 `select_session_cookies(session, login_origin, request.cookie_names)` 从 cookie jar 拼装。auth-ipc.ts:147-184 亦有类似回退。
- **依据**：`connector-session.md:22`「通过 webRequest 捕获浏览器**实际发出**的目标接口请求头（尤其 Cookie）——**不从 cookie jar 猜拼**」。
- **动作**：删除 `select_session_cookies` 回退分支；`captured_cookie` 为 null 时直接 `resolve({saved:false})` 并提示"未捕获到 Cookie"。
- **风险**：原本靠回退登录成功的用户需要重新登录。可接受（回退本就是掩盖真 bug）。
- **验收**：登录仅靠 webRequest 捕获实际请求头；未捕获到即失败，不猜拼。

---

## P1 — 近期修

### P1-1. scheduler-orchestrator 新接口用废弃词 plugins

- **事实**：[scheduler-orchestrator.ts:10-18](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/scheduler-orchestrator.ts#L10-L18) 新建内部接口 `ConnectorListConfig { plugins: ... }`。
- **依据**：`domain.md §5`「新代码一律用统一词 connector」+ 此接口是新建内部接口，无兼容理由用 `plugins`。
- **动作**：`plugins` → `connectors`，同步改 `startAll/rebuild/reconcile` 签名 + 所有调用方。
- **验收**：新建内部接口无 `plugin` 词。（注：`config-store.ts` 的 `is_plugin_healthy`/`prune_invalid_plugins` 处理旧 `plugins[]` 字段，属兼容代码，**不改**。）

### P1-2. executor 仍发废弃 name 字段

- **事实**：[tier1-poll-executor.ts:70](file:///d:/Kar/Code/omni_usage/src/main/core/connector/tier1-poll-executor.ts#L70) 和 [probe-executor.ts:105](file:///d:/Kar/Code/omni_usage/src/main/core/connector/probe-executor.ts#L105) 仍发 `name: "Usage"`（同时已发 `raw_label`+`normalized_label`）。
- **依据**：`conventions.md §5` 步骤 3「应发 `raw_label`+`normalized_label`，`name` 已废弃」。
- **动作**：先 grep 确认无消费者读 `name` 字段（observation schema、UI、测试），再删除。
- **验收**：observation 无 `name` 字段；schema 不再接受 `name`。

### P1-3. config-ipc 用户文案用"插件"

- **事实**：[config-ipc.ts:110,162,198,223](file:///d:/Kar/Code/omni_usage/src/main/ipc/config-ipc.ts#L110-L223) 用户可见文案「未知的插件实例」「插件不存在」「源插件不存在」「复制插件失败」。
- **依据**：`domain.md §5`「术语中英一律统一」+ 用户面向文案更应统一。
- **动作**：插件 → 连接器（4 处文案）。
- **验收**：用户可见文案无"插件"。

### P1-4. MiMo 硬编码 cookie 名和登录域名

- **事实**：[auth-ipc.ts](file:///d:/Kar/Code/omni_usage/src/main/ipc/auth-ipc.ts) `trySilentCookieRefresh` 硬编码 MiMo cookie 名，`ALLOWED_LOGIN_DOMAINS` 写死。
- **依据**：`architecture.md §5`「连接器是声明式的」+ 新增 session 连接器应只加 manifest 不改宿主。
- **动作**：
    1. manifest schema 增加 `loginDomains[]` 字段（若未有）。
    2. cookie_names 已在 manifest（确认）。
    3. auth-ipc 从 manifest 读 cookie_names + loginDomains，删除硬编码。
- **风险**：需确认 manifest schema 是否已支持；若不支持要先扩 schema + 补已有连接器 manifest。
- **验收**：新增 session 连接器（如 Kimi/OpenCode Go）不改 auth-ipc 宿主代码。

---

## P2 — 技术债批量处理

### P2-1. camelCase 函数命名统一为 snake_case

- **事实**：[logger.ts:108,112,198](file:///d:/Kar/Code/omni_usage/src/shared/lib/logger.ts#L108-L198)、[preload/index.ts:346](file:///d:/Kar/Code/omni_usage/src/preload/index.ts#L346)、[provider-usage.ts:92-353](file:///d:/Kar/Code/omni_usage/src/renderer/lib/provider-usage.ts#L92-L353) 等混用 camelCase 与 snake_case。
- **依据**：`conventions.md §1`。
- **动作**：分模块批量 rename（logger → preload → renderer/lib → 其他）。建议分多个 PR，每 PR 一个模块。
- **注意**：React 组件文件名与组件名保持 PascalCase（conventions §1 例外）。
- **验收**：全库无 camelCase 函数/变量（React 组件除外）。

### P2-2. paths.ts 路径常量集中

- **事实**：[paths.ts](file:///d:/Kar/Code/omni_usage/src/main/core/paths.ts) 仅 `getConfigPath`；`secrets.vault`/`vault.key`/`observations.db`/`snapshot-cache`/`logs` 散落 [file-vault-backend.ts](file:///d:/Kar/Code/omni_usage/src/main/core/vault/file-vault-backend.ts) 与 [index.ts:114,118](file:///d:/Kar/Code/omni_usage/src/main/index.ts#L114-L118)。
- **依据**：`platform-services.md`「paths 集中 userData 下文件路径常量」。
- **动作**：把所有路径常量收口到 paths.ts，调用方改引用。
- **验收**：路径常量单一定义点。

### P2-3. toDTO 命名 + PluginSnapshotDTO 用废弃词

- **事实**：[ipc/helpers.ts:30](file:///d:/Kar/Code/omni_usage/src/main/ipc/helpers.ts#L30) `toDTO(state): PluginSnapshotDTO`——名字太泛 + 返回类型用废弃词。
- **动作**：`toDTO` → `state_to_snapshot_dto`（或更具体的业务名）；`PluginSnapshotDTO` → `ConnectorSnapshotDTO`（同步改 shared/types）。
- **验收**：命名清晰无废弃词。

### P2-4. net-client do_request 与 get_raw 重复

- **事实**：[net-client.ts:143-234](file:///d:/Kar/Code/omni_usage/src/main/core/connector/net-client.ts#L143-L323) 两段重复 URL→`assert_safe_connector_host`→`apply_auth`→AbortController/超时→`statusCode>=400` 前奏。
- **动作**：抽 `build_request_context(manifest, endpoint, vault)` 辅助函数，两处共用。
- **验收**：无重复前奏逻辑。

### P2-5. connector-ipc 重复 cpa 分支

- **事实**：[connector-ipc.ts:19-27,29-44,70-84](file:///d:/Kar/Code/omni_usage/src/main/ipc/connector-ipc.ts#L19-L84) 三处各自分支 `definition.manifest.id === "cpa"`/capabilities。
- **动作**：抽 CPA 专属 IPC handler 或用 capabilities 驱动的单一分发。
- **验收**：`cpa` 判断点单一。

---

## 待澄清 — 需产品/设计决策

### 待澄清-1. CONFIG_EXPORT 加密 spec 冲突

- **事实**：`secret-vault.md:45`「密钥部分必须用户输入口令，scrypt 派生密钥后 AES-GCM 加密成 bundle」；`config-store.md:35`「密钥脱敏为 `***REDACTED***`」；实现（config-ipc.ts）跟 config-store.md。
- **决策点**：导出配置时，密钥部分要支持加密导出（带口令）还是只脱敏导出（不带密钥）？
- **影响**：若选加密导出，需实现口令输入 UI + scrypt + AES-GCM bundle；若选脱敏导出，改 secret-vault.md 与 config-store.md 一致即可。
- **动作**：先统一两个 spec 文件，再决定实现方向。
- 用户决策：直接导出，权限完全开放给用户，不脱敏，不加密。

### 待澄清-2. accountOverrides.disabled / accountLabels 字段去留

- **事实**：代码有 `accountOverrides.disabled` 和 `accountLabels`，`config-store.md` AppConfiguration 列表未列；config-store.md:13 自己承认 accountOverrides 不在 Zod schema（load 时被静默剥掉）。
- **决策点**：
    - 保留：补进 Zod schema + 补 spec（AppConfiguration 字段列表）。
    - 删除：删 types + account-overrides.ts 相关分支 + UI（disable_account 按钮）。
- **联动**：若删除 `disabled`，P0-3 自动简化为"删代码"；若保留，P0-3 按原方案改。
- **动作**：产品决策后，再派 P0-3 或新任务。
- 用户决策：**保留 disabled 语义）**：1. 对直连账号：`disabled` 应作用于连接器实例层级（禁用整个数据源），不是账号子行。移到设置页"行即数据源"行。2. 对 CPA 账号：移除 `disabled` 能力，只保留 `hidden`。

### 待澄清-3. LocalAPI 网关端点

- **事实**：`platform-services.md`「可选网关 `/v1/<provider>/...`，默认关闭」；实现未做。
- **决策点**：要不要做？大概率不做（PRD「明确不做：不做通用开放代理」已暗示）。
- **动作**：若不做，从 spec 删"可选网关"段或标注"暂不实现"；若做，另开 task。
- 用户决策：不做

### 待澄清-4. 关闭动作第三选项

- **事实**：`window-management.md:46` 罗列「隐藏 vs 退出 vs 最小化到托盘」；实现 [settings-close-action.ts](file:///d:/Kar/Code/omni_usage/src/main/core/settings-close-action.ts) 仅 `"hide" | "proceed"`。
- **决策点**：要不要"最小化到托盘"？
- **动作**：若不做，spec 标注"暂不实现"；若做，另开 enhancement task。

---

## 执行顺序建议

1. **先决策待澄清-2**（`disabled` 字段去留）——决定 P0-3 的形态。
2. **P0-1 / P0-2 / P0-4 / P0-5 并行**——彼此独立，可同时推进。
3. **P0-3**——待澄清-2 决策后执行。
4. **P1-1 ~ P1-4**——可并行，互不依赖。
5. **P2 批量**——攒一起 refactor，避免零散改动。
6. **待澄清-1 / 待澄清-3 / 待澄清-4**——spec 决策，不阻塞代码修复。

## 不改项（已核对，非问题）

- **`account_id: "default"`**（tier1-poll-executor / probe-executor）— 单账号直连场景的合理兜底，非硬违规。若未来 API 能返回真实账号 ID 再换。
- **`is_plugin_healthy` / `prune_invalid_plugins`**（config-store.ts）— 处理旧 `plugins[]` 字段的兼容代码，命名合理。
- **`apply_auth` Feature Envy** — NetClient 本职就是发安全 HTTP 请求，认证是其固有职责，非 Feature Envy。
- **`secrets-store.ts` Middle Man** — 有意的边界接口（依赖倒置 + 封装变化 + 安全边界），非 Middle Man。
- **文件名 kebab-case** — TS/Node 生态事实标准且全库统一，应改 `conventions.md §1` 而非改代码。
