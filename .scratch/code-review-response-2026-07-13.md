<!-- omni_powers: scratch/code-review-response-2026-07-13 -->

# 对《代码审查报告（2026-07-13）》的复核意见

**复核范围**：`.scratch/code-review-2026-07-13.md` 中 Standards / Spec 两轴共 23 项指控。  
**复核方法**：抽样核对被引用的代码与 `docs/omni_powers/op_blueprint/` 原文。  
**结论**：报告抓到的问题**大体属实**，但部分条目把 spec 冲突、可选功能、已知不一致等同于“实现错误”，会人为抬高优先级。建议对 5 处降级处理。

---

## 一、建议降级或改判的 5 项

### 1. `CONFIG_EXPORT` 未加密 → 改判为“spec 冲突”

原报告依据 `secret-vault.md`：

> “配置可明文导出，**密钥部分必须用户口令**，scrypt 派生密钥后 AES-GCM 加密成 bundle。”

但 `config-store.md` 边界段明确写：

> “导入导出见 `ipc.md`（`CONFIG_EXPORT`/`CONFIG_IMPORT`，**密钥脱敏为 `\***REDACTED**\*`**）与 `secret-vault.md`。”

当前 `config-ipc.ts` 输出 `***REDACTED***` 与 `config-store.md` 的描述一致。因此这不是单方面实现错误，而是 **`secret-vault.md` 与 `config-store.md` 对同一段行为描述不一致**。应作为 spec 冲突处理，而非直接判实现错误。

---

### 2. `accountOverrides.disabled` / `accountLabels` → 不应叫 scope creep

原报告把这两项列为“(b) Scope creep”，理由是 `config-store.md` 的 AppConfiguration 字段列表未列它们。

但 `config-store.md` 第 13 行已记录为 **已知不一致**：

> “`accountOverrides` 在 TS 接口里有，但**不在 Zod schema** —— load 时会被静默剥掉（已知不一致）。”

`accountLabels` 同样属于半拉子实现。把它们称为 scope creep（范围蔓延）带有“未经规划乱加功能”的暗示，更像是 **遗留技术债 / spec 与实现不一致**。建议改列为“(a) 已知不一致或未完成”。

---

### 3. LocalAPI 网关端点缺失 → 从“缺失”降级为“可选功能未启用”

`platform-services.md` 原文：

> “**可选**网关 `/v1/<provider>/...` —— 转发到 manifest 声明的固定上游域名（白名单），**默认关闭**。”

“可选且默认关闭”不是硬性 spec，没实现不应等同于“缺失/未完整实现”。建议移到 enhancement 或已知限制清单，避免占用高优先级。

---

### 4. `config-store.ts` 的 `is_plugin_healthy` / `prune_invalid_plugins` → 不算“新代码引入废弃词”

`domain.md §5` 规定“新代码一律用统一词”，但也承认：

> “代码里仍残留 `plugin`（IPC `connector` 别名、config `plugins[]` 字段、`preload` 的 `plugin` 别名）为兼容包袱。”

这两个函数处理的对象正是 config 里的旧 `plugins[]` 字段，命名带 `plugin` 是兼容迁移代码的自然选择，不是新代码推广废弃词。真正应保留在硬违规里的是 `scheduler-orchestrator.ts` 中**新建**的 `ConnectorListConfig { plugins: ... }` 接口。

---

### 5. 关闭动作缺第三选项 → 从“(a) 缺失”降为 enhancement

`window-management.md` 第 46 行只是罗列：

> “纯函数决定设置窗关闭行为（隐藏 vs 退出 vs 最小化到托盘），可测。”

文档没有明确三个选项都必须已实现。当前 `hide/proceed` 已覆盖“运行中关闭设置窗”和“退出流程”两个核心场景。“最小化到托盘”应作为后续 enhancement，与 invariant 违规分开级别。

---

## 二、认同无需改判的部分

以下指控经核对，与 blueprint 一致，判定准确。

| #   | 问题                                       | 依据                                                       | 位置                                                |
| --- | ------------------------------------------ | ---------------------------------------------------------- | --------------------------------------------------- |
| 1   | camelCase 函数命名违反 `conventions.md §1` | `formatTimestamp` / `shouldLog` / `formatMeta` 等          | `src/shared/lib/logger.ts`                          |
| 2   | 新建接口仍用 `plugins` 废弃词              | `ConnectorListConfig { plugins: ... }`                     | `src/main/core/scheduler/scheduler-orchestrator.ts` |
| 3   | `account_id: "default"` 占位               | 违反 `domain.md` invariant 3                               | `tier1-poll-executor.ts`、`probe-executor.ts`       |
| 4   | executor 仍发已废弃的 `name` 字段          | 违反 `conventions.md §5` 步骤 3                            | `tier1-poll-executor.ts`、`probe-executor.ts`       |
| 5   | session partition 命名错误                 | `connector-session.md` 要求 `persist:<provider>-login`     | `src/main/core/session/session-manager.ts`          |
| 6   | cookie jar 回退拼装违禁                    | `connector-session.md` 明确“不从 cookie jar 猜拼”          | `session-manager.ts`、`auth-ipc.ts`                 |
| 7   | refresh-service 失败不写 stale observation | `observation-store.md` 要求失败挂 `stale:true`+`lastError` | `src/main/core/scheduler/refresh-service.ts`        |
| 8   | CPA 单账号失败不产 stale                   | 违反 `domain.md` invariant 5                               | `connectors/cpa/connector.ts`                       |
| 9   | PopupView 账号子行写 `disabled`            | 违反 `domain.md` invariant 8 的所有权层级                  | `src/renderer/views/PopupView.tsx`                  |

---

## 三、Severity 重排建议

### 高（会破坏业务不变量，应尽快修）

1. CPA 单账号失败未发 stale（invariant 5）。
2. refresh-service 失败不写 stale observation（observation-store.md 核心行为）。
3. PopupView 账号子行写 `disabled`（invariant 8，破坏性操作越层）。
4. `account_id: "default"` 占位（invariant 3，会导致历史/隐藏/标签错位）。
5. session partition 命名错误 + cookie jar 回退拼装（session 能力基础契约失效）。

### 中（违反约定，但风险局部）

6. 新建接口仍用 `plugins` 废弃词。
7. executor 仍发 `name` 字段。
8. 全库 camelCase 函数命名历史债（建议批量 refactor，不是单点修）。

### 低 / 需先澄清 spec

9. `CONFIG_EXPORT` 加密：先解决 `secret-vault.md` 与 `config-store.md` 冲突。
10. `accountOverrides.disabled` / `accountLabels`：决定是补进 Zod schema 还是彻底移除。
11. LocalAPI 网关：可选功能，明确是否真的要开。
12. settings close 第三选项：enhancement。

---

## 四、总体结论

原报告的问题发现能力较强，23 项中大部分都有文档或代码支撑。主要偏差在于：**把“spec 打架/可选/已知不一致”也打包进了“(a) 缺失/未完整实现”和“(b) scope creep”**，会让阅读者高估严重性。建议按上表重排后，再进入修复阶段。
