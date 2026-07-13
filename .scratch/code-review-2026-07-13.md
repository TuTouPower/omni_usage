<!-- omni_powers: scratch/code-review-2026-07-13 -->

# 代码审查报告（2026-07-13）

**审查范围**：当前仓库全部源码（`src/`、`connectors/`、`scripts/`、`tests/` 抽样），非 git diff，仅当前状态。
**审查方法**：两轴并行（Standards + Spec），sub-agent 独立产出，不合并排序。
**标准来源**：`docs/omni_powers/op_blueprint/` 下 `conventions.md` / `architecture.md` / `domain.md` / `test.md` + Fowler smell baseline。
**Spec 来源**：`prd.md` + `spec_index.md` + `specs/*.md` + `domain.md §4` 不变量。

---

## Standards

### 硬性违规（违反文档化标准）

**1. `conventions.md §1`（函数须 `snake_case`）** — 全代码库广泛使用 camelCase 函数，工具未强制：

- [logger.ts](file:///d:/Kar/Code/omni_usage/src/shared/lib/logger.ts#L108-L198) — `formatTimestamp`/`shouldLog`/`formatMeta`（同文件 `serialize_meta`/`scrub_meta` 为 snake_case，自相矛盾）。
- [preload/index.ts:346](file:///d:/Kar/Code/omni_usage/src/preload/index.ts#L346) — `sanitizeLogField`。
- [provider-usage.ts](file:///d:/Kar/Code/omni_usage/src/renderer/lib/provider-usage.ts#L92-L353) — `compareProviders`/`latestTimestamp`/`worstStatus`/`toPeriod`/`hasValidQuota`（同文件又有 snake_case `build_provider_usage_groups`）。

**2. `domain.md §5`（新代码须用 `connector`，禁 `plugin`）** — 旧包袱允许残留，但**新**代码仍引入废弃词：

- [scheduler-orchestrator.ts:11-18](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/scheduler-orchestrator.ts#L11-L18) — 新接口 `ConnectorListConfig { plugins: ... }`。
- [config-store.ts:47,59,67](file:///d:/Kar/Code/omni_usage/src/main/core/config/config-store.ts#L47-L67) — 新函数 `is_plugin_healthy`/`prune_invalid_plugins`。
- [config-ipc.ts:110,162,198,223](file:///d:/Kar/Code/omni_usage/src/main/ipc/config-ipc.ts#L110-L223) — 新用户文案「未知的插件实例」「插件不存在」「源插件不存在」「复制插件失败」。

**3. `domain.md` 不变量 3（accountId 须服务商稳定，禁占位）** — [tier1-poll-executor.ts:67](file:///d:/Kar/Code/omni_usage/src/main/core/connector/tier1-poll-executor.ts#L67) 与 [probe-executor.ts:102](file:///d:/Kar/Code/omni_usage/src/main/core/connector/probe-executor.ts#L102) 用 `account_id: "default"`。

**4. `conventions.md §5` 步骤 3（应发 `raw_label`+`normalized_label`，`name` 已废弃）** — 同两个 executor 仍发 `name: "Usage"`：

- [tier1-poll-executor.ts:70](file:///d:/Kar/Code/omni_usage/src/main/core/connector/tier1-poll-executor.ts#L70)
- [probe-executor.ts:105](file:///d:/Kar/Code/omni_usage/src/main/core/connector/probe-executor.ts#L105)

### Smell baseline（判断项）

- **Mysterious Name** — [ipc/helpers.ts:30](file:///d:/Kar/Code/omni_usage/src/main/ipc/helpers.ts#L30) `toDTO(state): PluginSnapshotDTO`，名不达意。
- **Duplicated Code** — [net-client.ts:143-234](file:///d:/Kar/Code/omni_usage/src/main/core/connector/net-client.ts#L143-L323) `do_request` 与 `get_raw` 重复 URL→`assert_safe_connector_host`→`apply_auth`→AbortController/超时→`statusCode>=400` 前奏。
- **Feature Envy** — [net-client.ts:124-141](file:///d:/Kar/Code/omni_usage/src/main/core/connector/net-client.ts#L124-L141) `apply_auth` 深探 `manifest.poll?.request.auth`/`vault.get`/`keyFor`，更羡 Manifest+Vault。
- **Repeated Switches** — [connector-ipc.ts:19-27,29-44,70-84](file:///d:/Kar/Code/omni_usage/src/main/ipc/connector-ipc.ts#L19-L84) 三个函数各自再分支 `definition.manifest.id === "cpa"`/capabilities。
- **Middle Man** — [secrets-store.ts:14-30](file:///d:/Kar/Code/omni_usage/src/main/core/config/secrets-store.ts#L14-L30) `createSecretsStore` 每个方法 1:1 转发 `vault`，仅 `importAll` 有逻辑。

> 注：文件名 `conventions.md §1` 要求 snake_case，实际全库 kebab-case（`manifest-loader.ts`/`net-client.ts`/`host-io.ts`…）——虽是文档字面违规，但显然已是既定风格，只标一次不逐文件列。

---

## Spec

### (a) 缺失/未完整实现

1. **采集失败未写 stale 观测** — `observation-store.md` 要求「失败保留上次成功，挂 `stale:true`+`lastError`」。[refresh-service.ts:285-289](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/refresh-service.ts#L285-L289) 失败分支只置 runtime state="failed"+存 lastSuccess，未插 `stale:true` 行；observation-store 支持 stale 列但失败路径不填。
2. **CPA 单账号失败未发 stale** — `domain.md` 不变量 5「仅该账号红点 + stale」。[cpa/connector.ts:514-525](file:///d:/Kar/Code/omni_usage/connectors/cpa/connector.ts#L514-L525) per-account try/catch 仅 `warn` 跳过，无 stale 观测产出。
3. **LocalAPI 网关端点缺失** — `platform-services.md`「`/v1/<provider>/...` 网关（可选但应存在）」。[server.ts](file:///d:/Kar/Code/omni_usage/src/main/core/local-api/server.ts) 仅 `/v1/health`+`/v1/ingest`。
4. **LocalAPI token 未入 Vault** — spec「token 由宿主生成、存 Vault」。`server.ts` 生成 token 仅落本地，未入 vault。
5. **paths.ts 未集中** — spec「集中 userData 下文件路径常量」。[paths.ts](file:///d:/Kar/Code/omni_usage/src/main/core/paths.ts) 仅 `getConfigPath`；`secrets.vault`/`vault.key`/`observations.db`/`snapshot-cache`/`logs` 散落 `file-vault-backend.ts` 与 [index.ts:114,118](file:///d:/Kar/Code/omni_usage/src/main/index.ts#L114-L118) 内联拼接。
6. **CONFIG_EXPORT 未加密** — `secret-vault.md`「密钥部分须用户口令+scrypt+AES-GCM bundle」。[config-ipc.ts:234-237](file:///d:/Kar/Code/omni_usage/src/main/ipc/config-ipc.ts#L234-L237) 直接输出 `***REDACTED***` 占位，无口令/scrypt/AES-GCM。
7. **关闭动作缺第三选项** — `window-management.md`「隐藏 vs 退出 vs 最小化到托盘」。[settings-close-action.ts](file:///d:/Kar/Code/omni_usage/src/main/core/settings-close-action.ts) 仅 `"hide" | "proceed"`。

### (b) Scope creep

1. **`accountOverrides.disabled`** — `config-store.md` AppConfiguration 仅列 `hidden`。[types.ts:67-70](file:///d:/Kar/Code/omni_usage/src/main/core/config/types.ts#L67-L70) + `src/shared/types/config.ts` 同列 `hidden`+`disabled`；[account-overrides.ts:6](file:///d:/Kar/Code/omni_usage/src/renderer/lib/account-overrides.ts#L6) kind 接受 `"hidden"|"disabled"`；[PopupView.tsx:491-525](file:///d:/Kar/Code/omni_usage/src/renderer/views/PopupView.tsx#L491-L525) 在账号子行写 `disabled`。
2. **`accountLabels`** — [types.ts:101](file:///d:/Kar/Code/omni_usage/src/main/core/config/types.ts#L101) + [account-overrides.ts:42-68](file:///d:/Kar/Code/omni_usage/src/renderer/lib/account-overrides.ts#L42-L68) 提供 `set_account_label`，spec AppConfiguration 未列。
3. **MiMo 硬编码** — [auth-ipc.ts](file:///d:/Kar/Code/omni_usage/src/main/ipc/auth-ipc.ts) `trySilentCookieRefresh` 硬编码 MiMo cookie 名，`ALLOWED_LOGIN_DOMAINS` 写死，应来自 manifest。

### (c) 实现错误

1. **session partition 命名错** — `connector-session.md`「`persist:<provider>-login`」。[session-manager.ts:162](file:///d:/Kar/Code/omni_usage/src/main/core/session/session-manager.ts#L162) 用 `persist:session-login:<instance_id>`（按实例非按 provider）。
2. **partition 体系冲突** — `auth-ipc.ts` 用 `persist:mimo-login:<instanceId>`，与 `session-manager.ts` 的 `persist:session-login:` 不一致。
3. **cookie jar 回退违禁** — `connector-session.md`「不从 cookie jar 猜拼」。[session-manager.ts:105-110](file:///d:/Kar/Code/omni_usage/src/main/core/session/session-manager.ts#L105-L110) 与 `auth-ipc.ts:147-184` 均有 `select_session_cookies` 回退拼装。
4. **破坏性操作越层** — `domain.md` 不变量 8「破坏性操作只在『行即数据源』层级，账号子行只调显示」。[PopupView.tsx:491-525](file:///d:/Kar/Code/omni_usage/src/renderer/views/PopupView.tsx#L491-L525) 在账号子行写 `disabled` 破坏性覆盖。

---

## 汇总

| 轴        | 总数 | 最严重项                                                               |
| --------- | ---- | ---------------------------------------------------------------------- |
| Standards | 9    | `domain.md §5` 新代码仍引入废弃词 `plugin`（4 处硬违规）               |
| Spec      | 14   | invariant 8 破坏性操作越层 + invariant 5 CPA 单账号失败未发 stale 观测 |

两轴独立，不交叉排序。
